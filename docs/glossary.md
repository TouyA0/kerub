# Kerub — Glossary

Terms used across the Kerub documentation, in three groups: Kerub's own
vocabulary, security and Windows terms, and development terms. When a term
has a specific meaning in Kerub, that meaning wins over the general one.

**Rule:** when a document introduces a new term, it is added here in the
same pull request.

---

## 1. Kerub terms

| Term | Definition |
|------|------------|
| **Action** | A change Kerub makes (or simulates) in the system, such as blocking an IP address. Always journaled, reversible, and recorded as an action record. See [architecture.md](architecture.md) §5.3. |
| **Action manager** | The component that checks, journals, applies, expires and reverts actions. The only path by which Kerub changes the system. |
| **Agent** | `kerub-agent`, the tray application running in the user session. Displays and asks; never decides. |
| **Alert** | A record created when a detection rule matches. Carries a severity and a MITRE ATT&CK technique. |
| **Allow-list** | Devices the user has approved permanently. |
| **Audit log** | The hash-chained security log of everything Kerub detected, did or was asked to do. Distinct from the diagnostic log. |
| **Audit mode** | A module state in which Kerub detects and records what it *would* do, without changing the system. Every module starts in this mode. Opposite: enforce mode. |
| **Control style** | *Reactive* (an event triggers an action) or *declarative* (the system is continuously brought to a desired state). See architecture §4.2. |
| **Diagnostic log** | Technical log for troubleshooting. Never contains secrets. Distinct from the audit log. |
| **Emergency reset** | `kerub-cli emergency-reset` (or its PowerShell fallback): removes all of Kerub's enforcement without needing the service or the database. See [dev/recovery.md](dev/recovery.md). |
| **Enforce mode** | A module state in which Kerub actually applies its actions. |
| **Event** | A normalized record of something a sensor observed. See [event-schema.md](event-schema.md). |
| **Footprint manifest** | Registry record of every change Kerub made outside its own folders, with previous values. Lets the emergency reset work without the database. |
| **Kill switch** | The VPN guard feature: when enabled, all traffic outside the VPN tunnel is blocked, including when the VPN drops. |
| **Maintenance mode** | A time-limited suspension (≤ 60 min) of all enforcement, requiring elevation. |
| **Module** | An independently switchable feature (`netguard`, `vpnguard`, `usbguard`, `logwatch`, `posture`…). |
| **Never-block list** | Addresses Kerub never blocks: loopback, gateway, DNS servers, VPN servers. |
| **Personal security phrase** | A phrase chosen at setup and shown in every genuine Kerub prompt, so fake prompts can be recognized. Never shown while the service is unverified. |
| **Profile (network)** | The trust level assigned to a network: *home*, *public* or *work*. Unknown networks are *public*. |
| **Reconciliation** | At startup, comparing the action journal with what actually exists in the system, and fixing the differences. |
| **Record** | Any event, alert or action stored by Kerub. |
| **Responder (Kerub)** | A component that applies one kind of action (firewall, devices, settings) and knows how to undo it. Not to be confused with the Responder attack tool. |
| **Sensor** | A component that turns a Windows signal (log entry, network change, device arrival) into events. |
| **Service** | `kerub-svc`, the Windows service running as SYSTEM. The only privileged part of Kerub, and the only one that decides. |
| **Supervisor** | The component that runs module tasks, detects panics and restarts the affected module. |
| **Unlock password** | Kerub's own password, required for actions that weaken protection. Stored only as an Argon2id hash. |

---

## 2. Security and Windows terms

| Term | Definition |
|------|------------|
| **Argon2id** | A password hashing algorithm designed to be slow and memory-hard, which makes offline cracking expensive. |
| **Atomic Red Team** | An open-source library of small, safe simulations of attacker techniques, each mapped to MITRE ATT&CK. Used to test detection. |
| **Authenticode** | Microsoft's code-signing technology. Verifying a binary's Authenticode signature proves who published it and that it was not modified. |
| **BadUSB** | A malicious USB device that pretends to be a keyboard and types commands. |
| **Composite device** | A USB device that exposes several functions at once (for example keyboard + storage). An unusual combination is a classic BadUSB sign. |
| **BFE (Base Filtering Engine)** | The Windows service that manages WFP filters and loads persistent filters. |
| **BitLocker** | Windows full-disk encryption. |
| **Boot-time filter** | A WFP filter enforced from the start of boot, before services start. |
| **Captive portal** | The login page of a hotel, station or café Wi-Fi, which must be reached before the network works. |
| **CSP (Content Security Policy)** | Browser-side rules restricting what a web page may load and run. Protects the agent's UI against script injection. |
| **DACL** | Discretionary access control list: the list of who may do what with a Windows object (file, pipe, service, process). |
| **DLL hijacking** | Tricking a program into loading a malicious DLL, for example by placing it in a folder searched before System32. |
| **DPAPI** | Windows Data Protection API: encrypts secrets with keys tied to the user or the machine. |
| **ECS (Elastic Common Schema)** | A standard set of field names for security events, understood by most SIEMs. |
| **ELAM / PPL** | Early Launch Anti-Malware driver / Protected Process Light: Windows mechanisms that let approved anti-malware products protect themselves. Out of Kerub's reach (ADR-0006). |
| **ETW** | Event Tracing for Windows: the kernel and user-mode tracing system behind many Windows logs. |
| **Evil twin** | A rogue Wi-Fi access point imitating a legitimate network. |
| **EVTX** | The file format of Windows event logs. |
| **HID** | Human Interface Device: keyboards, mice and anything that declares itself as one. |
| **Integrity level** | Windows' trust level for processes (low, medium, high, system). A low-integrity process cannot write to medium-integrity objects. |
| **IPC** | Inter-process communication. In Kerub, the named pipe between the agent or CLI and the service. |
| **LLMNR / NBT-NS** | Legacy Windows name-resolution protocols. On a hostile network they let an attacker answer name lookups and capture credentials. |
| **LocalSystem (SYSTEM)** | The most privileged Windows account, used by `kerub-svc`. |
| **MITRE ATT&CK** | A public knowledge base of attacker tactics and techniques (for example T1110, Brute Force). |
| **Named pipe** | A Windows IPC channel with a name (`\\.\pipe\kerub`) and access control. Reachable over SMB unless remote clients are rejected. |
| **Persistent filter** | A WFP filter stored by Windows and re-applied after reboot, even if the program that created it is not running. |
| **Pipe squatting** | Creating a named pipe before the legitimate server, to impersonate it. |
| **Provider (WFP)** | An identifier that tags WFP objects as belonging to one product. Kerub has its own. |
| **Responder (tool)** | An attack tool that poisons LLMNR/NBT-NS to capture credentials. Used only in the isolated lab. |
| **Session 0** | The isolated Windows session where services run, with no user interface. |
| **SID** | Security identifier: the unique ID of a Windows user, group or account (`S-1-5-18` is SYSTEM). |
| **SIEM** | Security Information and Event Management: a central platform that collects and analyzes security events. |
| **Sigma** | An open, generic format for writing detection rules that can be converted to many SIEMs and engines. |
| **SMB** | Windows file-sharing protocol. Also carries remote access to named pipes. |
| **STRIDE** | Threat categories: Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. |
| **Sublayer (WFP)** | A container of WFP filters with its own priority. Kerub has its own sublayer. |
| **Sysmon** | A free Microsoft Sysinternals tool that logs detailed process, network and file activity. |
| **TPM** | Trusted Platform Module: a security chip used by BitLocker and Windows 11. |
| **UAC** | User Account Control: the Windows elevation prompt, shown on a separate secure desktop. |
| **WebView2** | The Microsoft Edge engine embedded in applications; renders the agent's UI. |
| **WFP** | Windows Filtering Platform: the Windows framework for filtering network traffic. See ADR-0003. |
| **WinRE** | Windows Recovery Environment: a separate recovery system used for offline repairs. |
| **WireGuard** | A modern VPN protocol. Used in the lab to test the kill switch. |

---

## 3. Development terms

| Term | Definition |
|------|------------|
| **ADR** | Architecture Decision Record: a short document recording one decision, its context, the options considered and its consequences. See [adr/](adr/). |
| **Crate** | A Rust compilation unit (library or binary). Kerub is split into several crates. |
| **Definition of Done** | The checklist every task must satisfy before it is complete. See [roadmap.md](roadmap.md) §1.2. |
| **Dogfooding** | Using your own product daily to find its problems. |
| **Fuzzing** | Feeding random or malformed inputs to a program to find crashes and bugs. |
| **MoSCoW** | Prioritization scheme: Must, Should, Could, Won't. |
| **Test double (fake)** | An in-memory replacement for a real component (for example the Windows platform), used in unit tests. |
| **Traceability** | The ability to follow a requirement to the tasks, code and tests that implement it, and back. |
| **`unsafe` (Rust)** | A block where the compiler's memory-safety guarantees are suspended, needed to call Windows APIs. Allowed only in `kerub-platform`. |
| **UUID v7** | A unique identifier that starts with a timestamp, so identifiers sort by creation time. |
| **Walking skeleton** | A minimal version of the whole system in which every layer exists and works end to end, before any real feature is built. |
| **Workspace (Cargo)** | A set of Rust crates built and versioned together. |