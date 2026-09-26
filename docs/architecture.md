# Kerub — Architecture

| | |
|---|---|
| **Status** | Draft |
| **Related** | [vision.md](vision.md), [threat-model.md](threat-model.md), [requirements.md](requirements.md), [ipc-protocol.md](ipc-protocol.md), [event-schema.md](event-schema.md), [adr/](adr/) |

This document describes how Kerub is built. It goes from the widest view
(Kerub in its environment) to the narrowest (components inside the service),
then covers data flows, lifecycle, storage and the repository layout.

---

## 1. Key decisions

These decisions shape everything else. Each links to its ADR when one exists.

| # | Decision | Why | Record |
|---|----------|-----|--------|
| D1 | Rust for all Kerub binaries | Memory safety without garbage collector, complete Windows API through `windows-rs`, minimal footprint | [ADR-0001](adr/0001-use-rust.md) |
| D2 | Tauri v2 with a React front-end for the agent UI | Native window through WebView2, Rust back-end, polished web UI | [ADR-0002](adr/0002-use-tauri-react.md) |
| D3 | Windows Filtering Platform for all network enforcement | Persistent and boot-time filters, own sublayer, clean removal | [ADR-0003](adr/0003-use-wfp.md) |
| D4 | SQLite for local storage | Embedded, transactional, no server | [ADR-0004](adr/0004-use-sqlite.md) |
| D5 | Named pipe + JSON messages for agent ↔ service | Local-only, ACL-protected, easy to validate and fuzz | [ADR-0005](adr/0005-ipc-named-pipe-json.md) |
| D6 | No kernel driver | Signing requirements and risk are out of proportion for this project | [ADR-0006](adr/0006-no-kernel-driver.md) |
| D7 | Three processes with privilege separation | Only the service is privileged; the UI holds no authority | this document, §3 |
| D8 | Windows-specific code isolated in the `kerub-platform` crate, the only crate allowed to use `unsafe` | Testable logic, one place to audit dangerous code | this document, §7 |
| D9 | Every action is journaled before execution, with undo and expiry | Reversibility, crash recovery, clean uninstall | this document, §5.3 |
| D10 | Two separate logs: diagnostic log and security audit log | Different audiences, different integrity guarantees | this document, §6 |

---

## 2. Context — Kerub in its environment

```mermaid
flowchart TB
    U(("User"))
    A(("Administrator"))
    K["<b>Kerub</b><br/>Windows workstation guardian"]
    OS["Windows<br/>WFP, device manager, event logs,<br/>Defender, UAC"]
    SYS["Sysmon<br/>(optional)"]
    DEV["USB / HID devices"]
    NET["Networks<br/>home, public, VPN"]
    EXT["Opt-in external services<br/>VirusTotal, ntfy, SIEM / Seraph"]

    U -->|"sees alerts, approves devices"| K
    A -->|"installs, configures policy"| K
    K -->|"enforces through"| OS
    OS -->|"events"| K
    SYS -->|"events"| K
    DEV --> OS
    NET --> OS
    K -.->|"opt-in"| EXT
```

Kerub never talks to hardware or the network directly: it observes and acts
through documented Windows APIs.

---

## 3. Containers — the three processes

```mermaid
flowchart LR
    subgraph USER["User session — medium integrity"]
        AG["<b>kerub-agent</b><br/>Tauri app: tray icon,<br/>dashboard, prompts"]
        CLI["<b>kerub-cli</b><br/>admin tool<br/>(run elevated)"]
    end
    subgraph SYS["Session 0 — LocalSystem"]
        SVC["<b>kerub-svc</b><br/>Windows service:<br/>sensors, detection,<br/>policy, response"]
        DATA[("ProgramData\Kerub<br/>config, database,<br/>audit log, diagnostics")]
    end
    AG <-->|"\\.\pipe\kerub<br/>JSON messages"| SVC
    CLI <-->|"\\.\pipe\kerub"| SVC
    SVC --> DATA
```

| Container | Runs as | Responsibilities | Must never |
|-----------|---------|------------------|------------|
| `kerub-svc` | `LocalSystem`, session 0, starts at boot | Collects events, detects, decides, acts, stores, serves the IPC | Show UI; trust the agent's decisions |
| `kerub-agent` | Interactive user, starts at logon | Displays status, timeline and alerts; asks the user for approvals; forwards requests | Make security decisions; verify passwords; write configuration; run elevated (first-run wizard excepted) |
| `kerub-cli` | Administrator, elevated on demand | Status, maintenance mode, diagnostics, emergency reset | Bypass the service for normal operations (emergency reset excepted) |

**Golden rule: the agent asks, the service decides.** Any request coming
through the pipe is treated as untrusted input, whoever sends it.

**Administrative actions from the agent** (restart the service, maintenance
mode) launch `kerub-cli` through UAC, with fixed arguments. The agent never
runs arbitrary programs and never runs elevated — with one exception: the
first-run wizard is an elevated instance of the agent, started once, whose
only administrative request is `setup.initialize` (SEC-IPC-007).

---

## 4. Components — inside `kerub-svc`

### 4.1 Overview

```mermaid
flowchart LR
    subgraph SENSE["Sense"]
        S1["logwatch<br/>(Security log, Sysmon)"]
        S2["netwatch<br/>(network changes)"]
        S3["devwatch<br/>(device arrivals)"]
        S4["posture<br/>(periodic checks)"]
    end
    BUS(["Event bus<br/>bounded, fan-out"])
    DET["Detection engine<br/>Sigma + threshold rules"]
    POL["Policy<br/>profile, mode,<br/>exceptions"]
    ACT["Action manager<br/>journal, never-block list,<br/>rate limits, expiry"]
    subgraph RESPOND["Respond"]
        R1["firewall<br/>(WFP)"]
        R2["devices"]
        R3["system settings"]
    end
    ST[("Store<br/>SQLite")]
    AUD[("Audit log<br/>hash-chained")]
    IPC["IPC server"]

    S1 & S2 & S3 & S4 --> BUS
    BUS --> DET
    BUS --> POL
    DET -->|"alerts"| POL
    POL -->|"actions"| ACT
    ACT --> R1 & R2 & R3
    BUS --> ST
    DET --> ST
    ACT --> ST
    ACT --> AUD
    IPC <--> POL
    IPC <--> ST
    ST -->|"subscriptions"| IPC
```

### 4.2 Two control styles

Kerub acts in two different ways, and both go through the action manager.

- **Reactive** — an event triggers a detection, the detection raises an
  alert, the policy turns the alert into actions.
  *Example: 10 failed logons from one IP → alert T1110 → block the IP for 1 h.*
- **Declarative** — the policy defines a *desired state*, and a controller
  continuously brings the system to that state (like a Kubernetes
  controller).
  *Example: current network is "public" → the public-profile filters must
  exist → create what is missing, remove what is extra.*

Network profiles, the VPN kill switch and device policies are declarative.
Detections are reactive.

### 4.3 Crates and responsibilities

The code is a Cargo workspace. Splitting it into crates enforces the
dependency rules at compile time and lets every crate except
`kerub-platform` declare `#![forbid(unsafe_code)]`.

| Crate | Responsibility |
|-------|----------------|
| `kerub-core` | Event types (ECS subset, see [event-schema.md](event-schema.md)), action types, event bus, module contract and supervisor |
| `kerub-config` | Embedded defaults, loading, validation, versioning, hot reload |
| `kerub-detect` | Rule loading, Sigma matching, threshold correlation, alert creation |
| `kerub-policy` | Turns alerts and desired states into actions according to profile, mode and exceptions |
| `kerub-respond` | Action manager and responders (firewall, devices, settings) |
| `kerub-store` | SQLite access, migrations, retention |
| `kerub-auditlog` | Append-only, hash-chained security log |
| `kerub-ipc` | Message types, framing, pipe server and client, peer verification — shared by the three binaries |
| `kerub-secret` | Argon2id hashing, DPAPI |
| `kerub-modules` | One module per feature: `netguard`, `vpnguard`, `usbguard`, `logwatch`, `posture`, later `filewatch` |
| `kerub-platform` | Traits abstracting Windows, their Windows implementations and their test doubles |

**Dependency rules:**

- Crates depend inward: `kerub-modules` may use `detect`, `policy`,
  `respond` and the `platform` traits; nothing depends on `kerub-modules`
  except the service binary.
- Only `kerub-platform` depends on the `windows` crate and contains
  `unsafe` code.
- The agent and the CLI depend only on `kerub-core` types and `kerub-ipc`.

**Runtime and conventions:**

- Asynchronous runtime: `tokio`.
- Errors: typed errors in library crates, errors with context in binaries.
- Logging: `tracing`, JSON output, file rotation.

### 4.4 Main contracts (sketch)

```rust
/// One independently switchable feature.
pub trait Module: Send + Sync {
    fn name(&self) -> &'static str;
    /// Must return quickly; long-running work is spawned as supervised tasks.
    async fn start(&self, deps: Deps) -> Result<()>;
    async fn stop(&self) -> Result<()>;
    /// Running, Degraded or Failed, with a reason.
    fn health(&self) -> Health;
}

/// Turns a Windows signal into normalized events.
pub trait Sensor: Send {
    async fn run(self, events: mpsc::Sender<Event>) -> Result<()>;
}

/// Applies one kind of action and knows how to undo it.
pub trait Responder: Send + Sync {
    fn kind(&self) -> ActionKind;
    async fn apply(&self, action: &Action) -> Result<Undo>;
    async fn revert(&self, undo: &Undo) -> Result<()>;
    /// What actually exists in the system, for reconciliation.
    async fn observed(&self) -> Result<Vec<Observed>>;
}
```

These are sketches: exact signatures (including how the traits are made
object-safe) are decided when each crate is implemented.

---

## 5. Key flows

### 5.1 Unknown USB storage device

```mermaid
sequenceDiagram
    participant D as USB device
    participant W as Windows
    participant S as kerub-svc
    participant A as kerub-agent
    participant U as User

    D->>W: plugged in
    Note over W: Installation blocked by the device policy<br/>Kerub applied beforehand
    W-->>S: device arrival notification
    S->>S: unknown storage → pending approval
    S-->>A: approval request (subscription)
    A->>U: prompt showing the personal security phrase
    U->>A: unlock password
    A->>S: ApproveDevice(instanceId, password, once|always)
    S->>S: rate limit, Argon2id check, audit log
    S->>W: allow this device instance
    S-->>A: result
```

The device is blocked **before** Kerub even sees it: Windows enforces the
installation policy on its own, so a BadUSB device cannot win a race against
the service.

### 5.2 Brute-force attempt

```mermaid
sequenceDiagram
    participant L as Security log
    participant SE as logwatch
    participant DE as Detection
    participant P as Policy
    participant AM as Action manager
    participant FW as Firewall responder (WFP)

    L-->>SE: event 4625 ×10 from 203.0.113.7
    SE->>DE: normalized events
    DE->>P: alert "Brute force" (T1110, high)
    P->>AM: action: block 203.0.113.7 for 1 h
    AM->>AM: never-block list, rate limit
    AM->>AM: write journal entry (undo + expiry)
    AM->>FW: add filter in Kerub's sublayer
    FW-->>AM: filter id
    AM->>AM: mark journal entry applied, audit log
```

In **audit mode**, the policy still produces the action, but the action
manager records it as *simulated* and stops there.

### 5.3 Action lifecycle

Every change Kerub makes to the system follows the same path:

1. **Plan** — the policy produces an action.
2. **Check** — never-block list, rate limits, module mode.
3. **Journal** — the action is written to the store *before* execution,
   with the information needed to undo it and its expiry.
4. **Apply** — the responder executes it.
5. **Confirm** — the journal entry is marked applied (or failed) and an
   audit-log entry is written.
6. **Expire or revert** — on expiry, user request or uninstall, the
   responder reverts it using the stored undo information.

When Kerub changes an existing system setting (for example disabling
LLMNR), the **previous value** is part of the undo information, so the
machine returns exactly to its prior state.

**Footprint manifest.** Every change Kerub makes outside its own folders
(WFP objects, device policies, system settings with their previous values)
is also recorded in a small manifest in the registry,
`HKLM\SOFTWARE\Kerub\Footprint` (SYSTEM and Administrators only). The
manifest is independent of the database, so the emergency reset (§8.4) can
undo everything even if the database is missing or damaged.

---

## 6. State, storage and logs

### 6.1 On-disk layout

| Path | Content | Access |
|------|---------|--------|
| `C:\Program Files\Kerub\` | Binaries | Admin write, everyone read |
| `C:\ProgramData\Kerub\config\` | `policy.yaml` (current configuration) | SYSTEM + Administrators |
| `C:\ProgramData\Kerub\data\kerub.db` | SQLite database | SYSTEM + Administrators |
| `C:\ProgramData\Kerub\audit\` | Hash-chained security audit log | SYSTEM + Administrators |
| `C:\ProgramData\Kerub\logs\` | Diagnostic logs (JSON, rotated) | SYSTEM + Administrators |
| `%LOCALAPPDATA%\Kerub\` | Agent UI preferences and agent logs (nothing sensitive) | Current user |

### 6.2 Database

Main tables: `events`, `alerts`, `actions` (journal), `exceptions`,
`devices` (allow-list), `config_versions`. Schema changes go through
numbered migrations in `crates/kerub-store/migrations`. Retention and size
cap follow NFR-PERF-005.

### 6.3 Configuration

Three layers, the last one wins:

1. **Defaults** embedded in the binary;
2. **Policy file** `policy.yaml`, written only by the service;
3. **Changes** requested through the agent or CLI, validated by the service,
   then written as a new policy version.

Writes are atomic (write to a temporary file, then rename). Every version is
kept in `config_versions`. If the current file is invalid at startup, the
service falls back to the last known good version and raises an alert.

### 6.4 Two logs, two purposes

| | Diagnostic log | Security audit log |
|---|---|---|
| **Audience** | Developer, troubleshooting | User, investigator, SIEM |
| **Content** | Errors, timings, internal state | Alerts, actions, approvals, config changes, IPC requests |
| **Integrity** | Plain rotated JSON | Hash-chained, verified at startup |
| **Retention** | Short (size-based rotation) | Long (retention policy) |

Important security events are also written to the Windows Event Log
(Application, source `Kerub`) so that any existing log collector can pick
them up.

---

## 7. Platform layer

All Windows-specific code lives in the `kerub-platform` crate, behind
traits:

| Trait | Wraps |
|-------|-------|
| `Firewall` | WFP: provider, sublayer, filters |
| `Devices` | Device notifications, installation restrictions, enable/disable |
| `EventLogs` | Subscriptions to event channels |
| `Network` | Interfaces, profiles, network change notifications |
| `Security` | ACLs, tokens, process identity, Authenticode verification |
| `Secrets` | DPAPI |
| `Settings` | Registry and system settings, with read-before-write |

Real implementations are compiled only on Windows (`#[cfg(windows)]`);
in-memory test doubles live in a `fake` module used by unit tests of the
other crates.

Rules for this crate:

- It is the **only** crate allowed to contain `unsafe` code; every other
  crate declares `#![forbid(unsafe_code)]`.
- Every `unsafe` block carries a `// SAFETY:` comment explaining why it is
  sound.
- Every Windows return code is checked.
- It contains no business logic.
- The service restricts the DLL search path to System32 at startup
  (`SetDefaultDllDirectories`).

---

## 8. Lifecycle

### 8.1 Service startup

1. Restrict the DLL search path.
2. Load and validate configuration (fallback to last known good).
3. Open the store and run migrations.
4. Verify the audit-log hash chain; alert if broken.
5. **Reconcile**: compare the action journal with what actually exists in
   the system; revert expired actions, re-apply missing ones, remove
   unknown Kerub-owned artifacts.
6. Start modules under the supervisor.
7. Start the IPC server.
8. Report *ready*.

### 8.2 Service shutdown

Stop the IPC server, stop modules with a timeout, flush the store.
**Persistent protections stay in place** (kill switch, device policies):
stopping the service does not open the gates.

### 8.3 Uninstall

Stop the service, revert every journaled action, delete Kerub's WFP
provider and sublayer, restore previous system settings, remove the service
and binaries. Logs are kept only if the user asks for it.

### 8.4 Emergency reset

`kerub-cli emergency-reset` works **without the service**: it reads the
footprint manifest (§5.3) and removes every Kerub-owned filter and policy
directly. A PowerShell fallback exists in
`scripts/emergency-reset.ps1`. Procedure: [dev/recovery.md](dev/recovery.md).

---

## 9. Identifiers

Fixed once, never changed:

| Item | Value |
|------|-------|
| Service name | `KerubSvc` |
| Pipe | `\\.\pipe\kerub` |
| Event Log source | `Kerub` |
| WFP provider GUID | generated in task T0.1, recorded in ADR-0003 |
| WFP sublayer GUID | generated in task T0.1, recorded in ADR-0003 |

---

## 10. Repository layout

```
kerub/
├── Cargo.toml               workspace definition
├── Cargo.lock
├── rust-toolchain.toml      pinned Rust version
├── justfile                 build, test, lint, package tasks
├── apps/
│   ├── kerub-svc/           service binary
│   ├── kerub-agent/         Tauri app
│   │   ├── src-tauri/       Rust side
│   │   └── ui/              React + TypeScript front-end
│   └── kerub-cli/           admin tool
├── crates/                  libraries, see §4.3
├── rules/
│   ├── sigma/               Sigma detection rules
│   └── threshold/           threshold rules
├── configs/
│   ├── default-policy.yaml
│   └── sysmon/              recommended Sysmon configuration
├── fuzz/                    fuzz targets
├── scripts/                 dev install, emergency reset
├── installer/               MSI definition
├── tests/
│   ├── e2e/                 end-to-end tests, run in the lab VM
│   └── fixtures/evtx/       recorded attack logs
└── docs/
```

---

## 11. Testing strategy

| Level | Where | What |
|-------|-------|------|
| Unit | CI and dev machine | All logic, with the platform test doubles |
| Fuzz | CI, Linux job | IPC decoder, rule parser, config parser (platform-independent code) |
| Detection | CI | Rules replayed against recorded EVTX attack samples |
| Integration | Lab VM only (ignored by default) | Real WFP, devices, event logs |
| End-to-end | Lab VM only | Installed service + agent + simulated attacks (Atomic Red Team) |

Integration and end-to-end tests **never** run on the development machine.

---

## 12. Build and release

- Target: `x86_64-pc-windows-msvc`; the Rust version is pinned in
  `rust-toolchain.toml`.
- The `justfile` drives build, tests, lint and packaging; the workspace
  version in `Cargo.toml` is the single source of the version number.
- CI (GitHub Actions, Windows runner) runs `cargo fmt --check`,
  `cargo clippy` with warnings as errors, `cargo test`, `cargo audit`,
  `cargo deny`, and the front-end linter on every pull request.
- Releases publish binaries, the installer and SHA-256 checksums; signing is
  added before v1.0 (SEC-SC-004).