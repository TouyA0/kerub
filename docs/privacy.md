# Kerub — Privacy and Data Handling

| | |
|---|---|
| **Status** | Draft |
| **Related** | [vision.md](vision.md) (principle 6), [threat-model.md](threat-model.md) (TM-ST-3, TM-EXT-1), [requirements.md](requirements.md) (NFR-PRIV-001 to 003, SEC-EXT-001), [event-schema.md](event-schema.md) §9 |

A security tool sees a lot: who logs on, which programs run, where the
machine connects. This document states exactly what Kerub collects, where it
keeps it, for how long, and what — if anything — ever leaves the machine.

**Rule for contributors:** any change that collects a new kind of data or
sends data anywhere must update this document in the same pull request.

---

## 1. Commitments

1. **No telemetry.** Kerub never reports usage, statistics or crashes to
   anyone (NFR-PRIV-001).
2. **Local by default.** Everything Kerub collects stays on the machine.
3. **Opt-in, feature by feature.** Nothing leaves the machine unless the
   user has explicitly enabled the specific feature that sends it
   (NFR-PRIV-002).
4. **Minimum necessary.** Kerub stores only what its rules, its interface or
   its exports need.
5. **Transparent and reversible.** Users can see, export and delete what
   Kerub has stored.

---

## 2. What Kerub collects

| Category | Examples of data | Source | Why |
|----------|------------------|--------|-----|
| Logon activity | User names, SIDs, logon types, source IP addresses, success or failure | Windows Security log | Detect brute force and account misuse |
| Account changes | Created accounts, group membership changes | Windows Security log | Detect privilege escalation and persistence |
| Process activity *(with Sysmon)* | Executable paths, command lines, hashes, parent processes | Sysmon | Detect malicious behavior |
| Network activity *(with Sysmon)* | Destination addresses and ports, protocol, originating program | Sysmon | Detect suspicious connections |
| Network context | Network name (SSID), interface, assigned profile, VPN state | Windows network APIs | Apply the right profile, enforce the kill switch |
| Devices | USB vendor and product IDs, serial numbers, device class, approval decisions | Windows device APIs | Device control and allow-list |
| Security posture | Results of configuration checks | Windows APIs, registry | Posture audit and score |
| Alerts and actions | What was detected, what Kerub did, when, why | Kerub | Timeline, reversibility, investigation |
| Kerub administration | Configuration versions, approvals, maintenance periods, IPC requests (client process, user SID) | Kerub | Accountability (audit log) |
| Diagnostics | Errors, timings, internal state | Kerub | Troubleshooting |

### What Kerub does **not** collect

- The **contents** of files, documents or messages.
- Keystrokes, screenshots, clipboard content, camera or microphone input.
- Web browsing history or page contents.
- Passwords: only an **Argon2id hash** of Kerub's own unlock password is
  stored (SEC-ST-003).
- The raw original of every event (`event.original`) — disabled by default.

Future modules that would change this list (for example DNS filtering,
which would see visited domain names) must update this section before they
are merged.

---

## 3. Where data is stored and who can read it

| Location | Content | Who can read |
|----------|---------|--------------|
| `C:\ProgramData\Kerub\data\kerub.db` | Events, alerts, actions, exceptions, device allow-list, configuration history | SYSTEM and Administrators |
| `C:\ProgramData\Kerub\audit\` | Security audit log | SYSTEM and Administrators |
| `C:\ProgramData\Kerub\logs\` | Diagnostic logs | SYSTEM and Administrators |
| `C:\ProgramData\Kerub\config\` | Configuration | SYSTEM and Administrators |
| `%LOCALAPPDATA%\Kerub\` | Agent display preferences, agent diagnostic log | The user |

Standard users see their machine's data **through the agent**, which only
shows what the service sends it; they cannot read the files directly.

Kerub does not encrypt its data at rest: that is the role of full-disk
encryption. Kerub's posture audit checks that **BitLocker** is enabled and
recommends it otherwise.

---

## 4. Retention

| Data | Default retention | Configurable |
|------|-------------------|--------------|
| Events | 30 days | Yes |
| Alerts and actions | 180 days | Yes |
| Security audit log | 180 days | Yes (minimum 30 days) |
| Diagnostic logs | Size-based rotation: 5 files × 10 MB | Yes |
| Device allow-list, configuration | Until changed or uninstalled | — |

The database is also capped in size (default 500 MB, NFR-PERF-005). When the
cap is reached, the **oldest events** are deleted first; alerts and actions
are kept as long as possible because they matter most for an investigation.

---

## 5. Minimization measures

- **Command lines** are truncated to 2 048 characters, and values that look
  like secrets are replaced by `[REDACTED]` — for example the value after
  `-p`, `--password`, `/p:`, `password=`, `token=` or `apikey=`, and
  well-known key formats. Redaction is best effort, not a guarantee.
- **Diagnostic logs** never contain event contents at the default log level,
  never contain passwords or secrets, and are kept short.
- **The Sysmon configuration** shipped with Kerub is tuned to exclude noisy,
  low-value events, which also reduces personal data collected.
- **Every stored field** must be justified by a rule, the interface or an
  export ([event-schema.md](event-schema.md) principle 3).

---

## 6. What can leave the machine

**By default: nothing.** The table lists every feature that can send data,
all disabled until the user enables them.

| Feature | What is sent | To whom | Notes |
|---------|--------------|---------|-------|
| File reputation (FR-FILE-002) | SHA-256 hash of the file — **never the file itself** | VirusTotal | A hash of a well-known file reveals that you have it; the hash of a unique personal document reveals nothing about its content |
| Phone notifications (FR-VIS-007) | Alert title and severity; details only if the user chooses | The ntfy server configured by the user (self-hosting recommended) | Detail level configurable |
| SIEM forwarding (FR-VIS-006) | Full records, as in [event-schema.md](event-schema.md) | The destination configured by the user | TLS required |
| Update checks (after v1.0) | IP address and Kerub version, implicitly, in the HTTPS request | GitHub (release metadata) | No update mechanism exists before v1.0 |

Every connection uses TLS with system certificate validation (SEC-EXT-001).
Enabling one of these features is recorded in the audit log.

### Diagnostic bundles

`kerub-cli diagnostics export` creates a bundle **locally**; Kerub never
sends it anywhere. The bundle may contain user names and file paths: review
it before sharing it with anyone. An anonymization option replaces user
names, machine names and IP addresses with placeholders.

---

## 7. User control

| Right | How |
|-------|-----|
| **See** | Everything Kerub stores is visible in the agent's timeline |
| **Export** | Timeline export as JSON Lines (FR-VIS-005) |
| **Delete** | "Purge history" (administrator, requires elevation): deletes events, alerts and actions. The audit log records *that* a purge happened, not what was purged |
| **Limit** | Disable modules or reduce retention periods |
| **Remove** | Uninstall deletes all Kerub data, unless the user asks to keep the logs |

---

## 8. Shared machines and organizations

- Kerub records activity from **every account** on the machine (logons,
  processes, connections). If other people use the machine, they should be
  told that Kerub is installed and what it records.
- When Kerub is deployed by an organization, the organization decides what
  is collected and is responsible for complying with data-protection law
  (for example the GDPR in the European Union).
- This section is practical guidance, not legal advice.

---

## 9. Changes to this document

Each change to this document is listed in the [CHANGELOG](../CHANGELOG.md).
A change that collects **more** data or sends data somewhere **new** must
also be mentioned in the release notes of the version that introduces it.