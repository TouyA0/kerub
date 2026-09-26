# ADR-0006: No kernel-mode component

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | Threat model R1, R4, R5, T7, T9; ADR-0003 |

## Context

Many security products rely on kernel drivers: file-system minifilters,
WFP callouts, process and image-load callbacks, and ELAM drivers that allow
the product to run as a protected process (PPL). These give deeper
visibility, real-time blocking and tamper resistance — they would reduce
accepted risk R1.

Constraints:

- New kernel drivers must be signed through Microsoft's hardware developer
  program, which requires an EV code-signing certificate and a verified
  organization — not realistic for an individual student project.
- Running as a protected anti-malware process requires an ELAM driver,
  available only to members of the Microsoft Virus Initiative.
- A kernel bug crashes the whole machine or becomes a kernel-level
  vulnerability.
- Kernel development would consume most of the project's time.
- Test-signing mode on development machines lowers their security.

## Options considered

### Own kernel driver

- Pros: maximum visibility, real-time blocking, tamper resistance.
- Cons: every constraint above. Rejected.

### User mode only, building on OS facilities

- Pros: no crash risk for the system, simple signing, fast development.
- Cons: weaker self-protection; no real-time file I/O interception.

## Decision

Kerub contains **no kernel-mode code**. It builds on:

- **Telemetry:** Windows event logs, ETW, and **Sysmon** — a
  Microsoft-signed driver that provides kernel-level telemetry. Sysmon is
  optional but recommended; Kerub does not redistribute it, the user
  installs it.
- **Enforcement:** the WFP management API (ADR-0003) and OS policies such as
  device installation restrictions, which Windows enforces by itself.

## Consequences

**Positive**

- Kerub cannot crash the system.
- Simpler build and signing.
- The project stays achievable.

**Negative**

- Accepted risk R1: an administrator-level attacker can disable Kerub.
- No real-time file I/O blocking: ransomware detection is reactive
  (FR-DET-010).
- Rich telemetry depends on Sysmon being installed.

**Follow-up**

- Posture check and documentation for installing Sysmon with the
  recommended configuration (`configs/sysmon/`).
- Revisit only if the project is backed by an organization able to sign
  drivers.