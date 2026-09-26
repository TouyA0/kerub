# Kerub — Requirements

| | |
|---|---|
| **Status** | Draft |
| **Related** | [vision.md](vision.md), [threat-model.md](threat-model.md), [roadmap.md](roadmap.md) |

## 0. Conventions

**Identifiers.** Every requirement has a stable ID. IDs are never reused; a
dropped requirement is marked *Withdrawn*, not deleted.

| Prefix | Meaning |
|--------|---------|
| `FR-<MODULE>-nnn` | Functional requirement — what Kerub does |
| `NFR-<CATEGORY>-nnn` | Non-functional requirement — how well it does it |
| `SEC-<AREA>-nnn` | Security requirement — derived from the [threat model](threat-model.md) |

**Priority (MoSCoW).** **M**ust, **S**hould, **C**ould. *Won't* items are
listed in the [vision](vision.md#5-non-goals).

**Release.** `1.0` = required for v1.0. `Later` = after v1.0.

**Traceability.** Roadmap tasks, tests and commit messages reference these
IDs (for example `feat(usb): block unknown storage [FR-DEV-001]`).

**Wording.** "Shall" = mandatory behavior. Acceptance criteria are detailed
in the roadmap task that implements the requirement.

---

## 1. Functional requirements

### 1.1 Core (`CORE`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-CORE-001 | Kerub shall run as a Windows service that starts automatically at boot. | M | 1.0 |
| FR-CORE-002 | The agent shall start at user logon and show a tray icon reflecting the global status: *protected*, *degraded* or *not protected*. | M | 1.0 |
| FR-CORE-003 | The CLI shall provide at least: status, module list, maintenance mode, and export of a diagnostic bundle. | M | 1.0 |
| FR-CORE-004 | Kerub shall install and uninstall through a single installer; uninstall shall remove every service, filter, policy and file it created. | M | 1.0 |
| FR-CORE-005 | Each module shall be independently enabled, disabled, and set to *audit* or *enforce* mode. | M | 1.0 |
| FR-CORE-006 | The service shall report the health of each module (running, degraded, failed) to the agent. | M | 1.0 |
| FR-CORE-007 | A maintenance mode shall suspend all enforcement for a limited duration; entering it requires elevation and is logged. | M | 1.0 |
| FR-CORE-008 | A first-run wizard shall set the unlock password, the personal security phrase, the initial network profile, and start every module in audit mode. | M | 1.0 |
| FR-CORE-009 | The service shall restart automatically after an unexpected termination. | M | 1.0 |

### 1.2 Network guard (`NET`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-NET-001 | Kerub shall detect network changes and assign each network a profile (*home*, *public*, *work*); unknown networks default to *public*. | M | 1.0 |
| FR-NET-002 | Kerub shall apply profile-specific filtering rules. | M | 1.0 |
| FR-NET-003 | Under the *public* profile, Kerub shall block unsolicited inbound connections and disable LLMNR, NBT-NS and SMB exposure. | M | 1.0 |
| FR-NET-004 | When the VPN kill switch is enabled, Kerub shall block all traffic that does not go through the designated VPN interface, except VPN server endpoints, DHCP and loopback. | M | 1.0 |
| FR-NET-005 | The kill switch shall prevent DNS queries outside the tunnel. | M | 1.0 |
| FR-NET-006 | The kill switch shall handle IPv6 explicitly (tunneled or blocked, never leaked). | M | 1.0 |
| FR-NET-007 | The kill switch shall remain enforced when the service is stopped or has crashed, and during boot. | M | 1.0 |
| FR-NET-008 | The user shall be able to grant a time-limited exception to reach a captive portal. | S | 1.0 |
| FR-NET-009 | Kerub shall inventory listening ports and alert when a new one opens. | S | Later |
| FR-NET-010 | Kerub shall detect evil twin access points impersonating a known network. | C | Later |
| FR-NET-011 | Kerub shall offer per-application outbound connection control. | C | Later |
| FR-NET-012 | Kerub shall filter DNS against malicious-domain blocklists. | C | Later |

### 1.3 Detection (`DET`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-DET-001 | Kerub shall collect events from the Windows Security log and, when installed, from Sysmon. | M | 1.0 |
| FR-DET-002 | Kerub shall support detection rules in Sigma format and threshold rules (*N* events within *T* seconds). | M | 1.0 |
| FR-DET-003 | Each rule shall declare a severity, a MITRE ATT&CK technique, a plain-language description and a recommended action. | M | 1.0 |
| FR-DET-004 | Kerub shall detect brute-force attempts on local and RDP logons. | M | 1.0 |
| FR-DET-005 | Kerub shall detect account creation and additions to privileged groups. | M | 1.0 |
| FR-DET-006 | Kerub shall detect clearing of the Security event log. | M | 1.0 |
| FR-DET-007 | Kerub shall verify that the audit policies its rules depend on are enabled, and warn otherwise. | M | 1.0 |
| FR-DET-008 | Kerub shall detect persistence mechanisms: new services, scheduled tasks, Run keys, WMI subscriptions. | S | Later |
| FR-DET-009 | Kerub shall detect suspicious PowerShell usage and abuse of built-in binaries (certutil, mshta, rundll32…). | S | Later |
| FR-DET-010 | Kerub shall detect ransomware behavior through canary files and mass file renaming. | C | Later |

### 1.4 Response (`RSP`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-RSP-001 | Kerub shall block a remote IP address for a configurable duration. | M | 1.0 |
| FR-RSP-002 | Kerub shall never block addresses on the never-block list (loopback, gateway, DNS servers, VPN servers). | M | 1.0 |
| FR-RSP-003 | Every action shall be journaled before execution with its undo information and expiry, and be undoable from the agent. | M | 1.0 |
| FR-RSP-004 | Automatic responses shall be rate-limited. | M | 1.0 |
| FR-RSP-005 | Kerub shall terminate or suspend a process. | S | Later |
| FR-RSP-006 | Kerub shall quarantine a file and restore it on request. | S | Later |
| FR-RSP-007 | Kerub shall isolate the machine from the network in one action, keeping Kerub's own required connectivity. | C | Later |

### 1.5 Device control (`DEV`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-DEV-001 | Kerub shall block any new USB mass-storage device until it is approved with the unlock password. | M | 1.0 |
| FR-DEV-002 | Approval shall be one-time or permanent; permanent approvals are stored in an allow-list managed from the agent. | M | 1.0 |
| FR-DEV-003 | Any new keyboard or other HID device shall require confirmation before it can send input. | M | 1.0 |
| FR-DEV-004 | Unknown storage devices shall be mountable in read-only mode. | S | Later |
| FR-DEV-005 | Kerub shall alert on new Bluetooth pairings. | C | Later |
| FR-DEV-006 | Kerub shall alert when an application accesses the camera or microphone. | C | Later |

### 1.6 Security posture (`POS`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-POS-001 | Kerub shall audit: antivirus status, firewall, BitLocker, Secure Boot, TPM, UAC level, pending updates, SMBv1, daily use of an admin account, automatic screen lock. | M | 1.0 |
| FR-POS-002 | Kerub shall compute a security score from the audit results. | M | 1.0 |
| FR-POS-003 | Each failed check shall come with a plain-language explanation of the risk. | M | 1.0 |
| FR-POS-004 | Each failed check shall offer a reversible one-click fix where technically possible. | S | Later |
| FR-POS-005 | Kerub shall report installed software with known vulnerabilities. | C | Later |

### 1.7 File checks (`FILE`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-FILE-001 | Kerub shall inspect new files in the Downloads folder: SHA-256, Authenticode signature, Mark of the Web. | S | Later |
| FR-FILE-002 | Kerub shall look up file hashes on VirusTotal when the user has opted in. | S | Later |
| FR-FILE-003 | Kerub shall scan files with YARA rules. | C | Later |
| FR-FILE-004 | Kerub shall run a file in Windows Sandbox on request. | C | Later |

### 1.8 Visibility (`VIS`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-VIS-001 | The agent shall show a searchable, filterable timeline of events, alerts and actions. | M | 1.0 |
| FR-VIS-002 | Each alert shall state what happened, why it matters, its severity, its MITRE technique, and the available actions. | M | 1.0 |
| FR-VIS-003 | Kerub shall send Windows notifications, grouped and rate-limited by severity. | M | 1.0 |
| FR-VIS-004 | The user shall be able to mark an alert as a false positive, creating a scoped exception. | S | 1.0 |
| FR-VIS-005 | Kerub shall export events as JSON following ECS. | S | 1.0 |
| FR-VIS-006 | Kerub shall forward events to a SIEM (syslog or HTTP). | C | Later |
| FR-VIS-007 | Kerub shall send notifications to a phone through a self-hosted service (e.g. ntfy). | C | Later |
| FR-VIS-008 | Kerub shall produce a weekly summary report. | C | Later |

### 1.9 Configuration and roles (`CFG`)

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| FR-CFG-001 | Only administrators shall change the policy; standard users can view status and answer low-impact prompts. | M | 1.0 |
| FR-CFG-002 | Every configuration change shall go through the service, be validated, versioned and logged. | M | 1.0 |
| FR-CFG-003 | Kerub shall offer global modes (*Normal*, *Travel*, *Paranoid*, *Quiet*). | C | Later |
| FR-CFG-004 | Configuration shall be exportable and importable. | C | Later |

---

## 2. Non-functional requirements

| ID | Requirement | Prio | Release |
|----|-------------|------|---------|
| NFR-PERF-001 | Service idle CPU usage shall stay below 1 % on average over 10 minutes. | M | 1.0 |
| NFR-PERF-002 | Idle memory shall stay below 50 MB for the service and 150 MB for the agent. | S | 1.0 |
| NFR-PERF-003 | 95 % of alerts shall be raised less than 2 seconds after the triggering event. | S | 1.0 |
| NFR-PERF-004 | Kerub shall not reduce network throughput by more than 5 %. | S | 1.0 |
| NFR-PERF-005 | The database shall be size-capped (default 500 MB) with a configurable retention (default 30 days). | M | 1.0 |
| NFR-REL-001 | Every enforcement shall be reversible through maintenance mode or the emergency reset script. | M | 1.0 |
| NFR-REL-002 | The failure of one module shall not affect the others. | M | 1.0 |
| NFR-REL-003 | At startup, the service shall reconcile the actual system state with its action journal. | M | 1.0 |
| NFR-USA-001 | Alert messages shall be in plain language; technical details are available on demand. | M | 1.0 |
| NFR-USA-002 | All UI strings shall be externalized in translation files; English by default, French provided. | M | 1.0 |
| NFR-USA-003 | Non-critical notifications shall be limited per hour and grouped to prevent alert fatigue. | M | 1.0 |
| NFR-USA-004 | The agent shall be fully usable with the keyboard and meet WCAG 2.1 AA contrast. | S | 1.0 |
| NFR-COMP-001 | Kerub shall support Windows 10 22H2 and Windows 11, x64. | M | 1.0 |
| NFR-COMP-002 | Kerub shall coexist with Windows Defender, third-party antivirus and VPN clients. | M | 1.0 |
| NFR-COMP-003 | Every feature not explicitly relying on an external service shall work offline. | M | 1.0 |
| NFR-COMP-004 | Kerub should support Windows on ARM64. | C | Later |
| NFR-PRIV-001 | Kerub shall contain no telemetry. | M | 1.0 |
| NFR-PRIV-002 | No data shall leave the machine without a per-feature opt-in. | M | 1.0 |
| NFR-PRIV-003 | Collected data and retention shall be documented in [privacy.md](privacy.md). | M | 1.0 |
| NFR-MAINT-001 | All Windows-specific calls shall be isolated in the `kerub-platform` crate, the only crate allowed to contain `unsafe` code. | M | 1.0 |
| NFR-MAINT-002 | Test coverage shall be at least 70 % for crates other than `kerub-platform`. | S | 1.0 |
| NFR-MAINT-003 | Every architecture decision shall be recorded as an ADR. | M | 1.0 |
| NFR-MAINT-004 | Build, tests, linters and security scanners shall pass in CI before any merge. | M | 1.0 |
| NFR-OBS-001 | Diagnostic logs shall be structured (JSON), rotated, and kept separate from the security audit log. | M | 1.0 |

---

## 3. Security requirements

Each requirement mitigates one or more threats from the
[threat model](threat-model.md).

| ID | Requirement | Mitigates | Prio | Release |
|----|-------------|-----------|------|---------|
| SEC-IPC-001 | The pipe DACL shall allow only SYSTEM and the interactive user; the service shall verify the client's image path and signature. | TM-IPC-1 | M | 1.0 |
| SEC-IPC-002 | The service shall create the pipe as first instance; the agent shall verify the server process identity before sending data. | TM-IPC-2 | M | 1.0 |
| SEC-IPC-003 | IPC messages shall be typed, schema-validated and size-limited; no request shall execute arbitrary commands; the parser shall be fuzzed. | TM-IPC-3 | M | 1.0 |
| SEC-IPC-004 | The IPC shall enforce connection limits, per-client rate limiting, timeouts, and back-off on failed password attempts. | TM-IPC-4, TM-IPC-6 | M | 1.0 |
| SEC-IPC-005 | Every IPC request shall be audit-logged with client PID, image path and user SID. | TM-IPC-5 | M | 1.0 |
| SEC-SVC-001 | Binaries shall be installed under Program Files with a quoted service path; DLLs shall be loaded from System32 only. | TM-SVC-1 | M | 1.0 |
| SEC-SVC-002 | Service and process DACLs shall deny stop, reconfigure and terminate rights to non-admins. | TM-SVC-2 | M | 1.0 |
| SEC-SVC-003 | The service shall declare its required privileges and drop all others. | TM-SVC-4 | M | 1.0 |
| SEC-SVC-004 | Every task shall run under a supervisor that detects panics and restarts the affected module. | TM-SVC-5 | M | 1.0 |
| SEC-SVC-005 | The agent shall alert when it loses contact with the service. | TM-SVC-3 | M | 1.0 |
| SEC-AG-001 | Every genuine prompt shall display the user's personal security phrase. | TM-AG-1 | M | 1.0 |
| SEC-AG-002 | Disabling a protection shall require UAC elevation. | TM-AG-1 | M | 1.0 |
| SEC-AG-003 | The agent shall hold no authority: every decision and verification happens in the service. | TM-AG-2 | M | 1.0 |
| SEC-AG-004 | Event data shall never be rendered as HTML; the UI shall enforce a strict CSP and load no remote content. | TM-AG-3 | M | 1.0 |
| SEC-ST-001 | The data directory shall be accessible only to SYSTEM and Administrators; the service is its only writer. | TM-ST-1, TM-ST-3 | M | 1.0 |
| SEC-ST-002 | The audit log shall be hash-chained. | TM-ST-2 | M | 1.0 |
| SEC-ST-003 | The unlock password shall be hashed with Argon2id; secrets shall be protected with machine-scope DPAPI. | TM-ST-4, TM-EXT-3 | M | 1.0 |
| SEC-DET-001 | Only trusted event channels and providers shall feed the detection engine. | TM-DET-1 | M | 1.0 |
| SEC-DET-002 | Event queues shall be bounded; dropped events shall be counted and alerted. | TM-DET-2 | M | 1.0 |
| SEC-RSP-001 | New modules and rules shall start in audit mode by default. | TM-RSP-1 | M | 1.0 |
| SEC-RSP-002 | Everything Kerub creates in the system shall carry Kerub's own identifiers and be removed on uninstall. | TM-RSP-2 | M | 1.0 |
| SEC-NET-001 | Kill-switch filters shall be persistent and active at boot. | TM-NET-1, TM-NET-2 | M | 1.0 |
| SEC-EXT-001 | External services shall be opt-in, receive hashes only, and use TLS with system certificate validation. | TM-EXT-1, TM-EXT-2 | M | Later |
| SEC-SC-001 | Every new dependency shall be justified; dependencies shall be pinned and scanned (cargo-audit, cargo-deny, Dependabot). | TM-SC-1 | M | 1.0 |
| SEC-SC-002 | No automatic update before v1.0; afterwards, updates shall be signed and verified before being applied. | TM-SC-2 | M | 1.0 |
| SEC-SC-003 | The repository shall use 2FA, signed commits, branch protection, and GitHub Actions pinned by commit SHA. | TM-SC-3 | M | 1.0 |
| SEC-SC-004 | Releases shall publish checksums and signed binaries. | TM-SC-4 | S | 1.0 |
| SEC-SC-005 | Every change shall receive a human review and a security review against the threat model before merge. | TM-SC-5 | M | 1.0 |