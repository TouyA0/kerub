# ADR-0001: Use Rust for all Kerub binaries

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | TM-IPC-3, TM-SVC-1, TM-SVC-5, NFR-PERF-001, NFR-PERF-002, NFR-MAINT-001, FR-CORE-004 |

## Context

- `kerub-svc` runs as `LocalSystem`. Any memory-safety bug reachable from
  the IPC channel becomes a local privilege escalation (TM-IPC-3).
- Kerub needs broad access to Windows APIs: WFP, device management, event
  log subscriptions, DPAPI, ACLs, tokens, COM interfaces.
- Installation must be simple: self-contained binaries, no runtime to deploy
  (FR-CORE-004).
- Kerub runs permanently: idle footprint must be very low (NFR-PERF-001,
  NFR-PERF-002).
- The author knows Python, Go, Java, C and web technologies, and is willing
  to learn a new language to make the best long-term choice.

## Options considered

### C / C++

- Pros: direct access to every API, smallest footprint.
- Cons: memory-unsafe code running as SYSTEM. Unacceptable given TM-IPC-3.

### Java

- Pros: known by the author.
- Cons: JVM footprint, awkward Win32 access, heavy distribution.

### Python

- Pros: known by the author, fast prototyping.
- Cons: interpreter packaging, easy to tamper with, performance.

### Go

- Pros: memory-safe, single static binary, known by the author, simple.
- Cons: the standard Windows bindings cover only part of the API; WFP and
  COM interfaces need third-party or hand-written bindings; garbage
  collector.

### C# / .NET

- Pros: the best Windows integration available; official generator
  (CsWin32) for the rest of the API; close to Java, quick to learn.
- Cons: runtime and garbage-collector footprint (reduced but not removed by
  Native AOT, which has its own limitations); the natural UI options (WPF,
  WinUI) would replace the web front-end.

### Rust

- Pros: memory safety **without** garbage collector; `windows-rs`,
  maintained by Microsoft and generated from Windows metadata, covers the
  whole Windows API; minimal footprint; Tauri keeps a web front-end with a
  Rust back-end; strong tooling (Clippy, `cargo audit`, `cargo deny`,
  fuzzing); `unsafe` code can be forbidden per crate by the compiler.
- Cons: real learning curve (ownership, borrowing, lifetimes, async); slower
  development at first; long compile times; FFI calls to Windows still
  require `unsafe`.

## Decision

We will write `kerub-svc`, `kerub-agent` and `kerub-cli` in **Rust**
(stable channel, edition 2024), with these rules:

- The toolchain is pinned in `rust-toolchain.toml`; target
  `x86_64-pc-windows-msvc`. Upgrades are deliberate.
- **`unsafe` is allowed only in `kerub-platform`.** Every other crate
  declares `#![forbid(unsafe_code)]`. Every `unsafe` block carries a
  `// SAFETY:` comment, enforced by Clippy.
- Asynchronous runtime: `tokio`.
- `cargo fmt` and `cargo clippy` with warnings as errors are mandatory in CI.
- Panic strategy `unwind`, so that the supervisor can detect a panicking
  task and restart its module (SEC-SVC-004).
- Python remains allowed for development tooling only, never for shipped
  components.
- **Learning before building:** the author completes *The Rust Programming
  Language* and *Rustlings* before task T0.1, and must understand every line
  merged into the repository.

## Consequences

**Positive**

- Memory safety for the privileged service, with no garbage-collector
  overhead.
- Complete, official access to the Windows API.
- The compiler enforces where dangerous code may live.
- One language for the service, the agent back-end and the CLI.

**Negative**

- Several weeks of learning before productive development.
- Slower early progress and longer builds than Go.
- FFI wrappers in `kerub-platform` require careful review: this is where
  memory bugs remain possible.

**Follow-up**

- T0.1: workspace, `rust-toolchain.toml`, lint configuration, CI.