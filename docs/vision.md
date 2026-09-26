# Kerub — Vision

> Kerub is an open-source, lightweight guardian for Windows workstations.
> It controls what gets plugged in, what the machine talks to, and watches
> for signs of intrusion — and it explains every decision it makes.

The name comes from *kerub* (plural *kerubim*), the Hebrew form of "cherub":
in the oldest texts, the cherubim guard the gate of Eden. Kerub guards the
gates of a workstation: its ports, its network and its logs.

---

## 1. The problem

A typical personal Windows machine is protected by Windows Defender and the
built-in firewall. They are good at what they do, but they leave gaps that
matter for an individual user:

- **Devices are trusted on insertion.** Any USB stick is mounted, and any
  device that declares itself a keyboard can type commands (BadUSB).
- **Public networks are hostile by default.** On a café or hotel Wi-Fi,
  legacy protocols (LLMNR, NetBIOS, SMB) expose the machine to credential
  theft, and nothing warns the user.
- **VPNs fail silently.** When the tunnel drops, traffic leaks in clear text,
  including DNS and IPv6.
- **Attacks leave traces nobody reads.** Brute-force attempts, new admin
  accounts or cleared logs are recorded by Windows, but no one looks at the
  event logs of a personal PC.
- **Security settings drift.** BitLocker off, UAC lowered, updates pending:
  the user rarely knows the actual security posture of their machine.

Existing answers are fragmented (a firewall front-end here, a VPN kill switch
there, Sysmon for experts only) or built for enterprises (EDRs that are
opaque, expensive and designed for fleets managed by a SOC).

## 2. Who Kerub is for

- **Primary users:** technically curious individuals on their own Windows
  10/11 machine — students, developers, remote workers, security
  enthusiasts. They want strong protection and want to understand it.
- **Secondary users:** less technical people sharing that machine. They must
  be protected without having to understand anything, and must not be able
  to weaken the protection by accident.
- **Not targeted (for now):** enterprise fleets and centrally managed
  endpoints.

## 3. What Kerub does

Kerub is organized in independent modules, each of which can be enabled,
disabled, or run in observation-only mode:

1. **Device control** — unknown USB storage is blocked until approved;
   new keyboards and HID devices must be confirmed.
2. **Network guard** — per-network profiles (home, public, work), VPN kill
   switch with DNS and IPv6 leak protection, hardening on public networks.
3. **Detection & response** — reads Windows and Sysmon logs, detects
   intrusion attempts and suspicious behavior, and responds (for example by
   temporarily blocking an IP address).
4. **Security posture** — audits the machine's configuration, computes a
   score, and offers reversible fixes.
5. **File checks** — inspects downloaded files (hash, signature, origin,
   reputation) before they are run.
6. **Visibility** — a timeline of everything Kerub saw and did, with plain
   explanations, and export to a SIEM.

Every detection is mapped to a MITRE ATT&CK technique.

## 4. Principles

These principles settle design disputes. When two options conflict, the one
that best respects them wins.

1. **Never lock the user out.** Every action is reversible, and a documented
   recovery procedure always exists.
2. **Explain, don't just forbid.** Every alert states what happened, why it
   matters, and what the user can do.
3. **Observe before enforcing.** Every new protection starts in audit mode;
   blocking is enabled only once false positives are understood.
4. **Least privilege, and defend itself.** Privileged code is kept minimal.
   Kerub must never become the weakest point of the machine it protects.
5. **Light and quiet.** Negligible resource usage. Alerts are rare, grouped
   and meaningful.
6. **Local and private by default.** No telemetry. Nothing leaves the
   machine without explicit opt-in.
7. **Complement, don't replace.** Kerub works alongside Windows Defender,
   never against it.
8. **Open and verifiable.** Open-source code, documented decisions,
   reproducible builds.

## 5. Non-goals

Kerub deliberately does **not**:

- act as an antivirus or ship a signature-based scanning engine;
- use a kernel driver;
- protect against an attacker who already has administrator or SYSTEM
  privileges (see [threat-model.md](threat-model.md));
- act as a VPN client — it only enforces that an existing VPN is used;
- manage fleets of machines or depend on a cloud backend (at least until
  v1.0).

## 6. Success criteria

Kerub is successful when:

- it runs daily on the author's own machine for 30 days without a lockout
  and with a false-positive rate the author finds acceptable;
- it installs and uninstalls without leaving any residue (rules, services,
  registry keys);
- its resource usage stays within the limits defined in
  [requirements.md](requirements.md);
- every detection rule is mapped to MITRE ATT&CK and tested against real
  attack samples;
- a reader can understand every design choice from the `docs/` folder alone.

## 7. Scope of v1.0

- Service, agent and CLI skeleton, with clean install and uninstall
- Network guard: profiles, VPN kill switch, public-network hardening
- Detection & response based on Windows logs
- Device control for USB storage and HID devices
- Security posture audit

File checks, advanced response playbooks and SIEM export come after v1.0.
The detailed plan lives in [roadmap.md](roadmap.md).

## 8. Ecosystem

Kerub emits events in the Elastic Common Schema (ECS), so they can be
forwarded to any SIEM — including [Seraph](https://github.com/TouyA0), the
author's self-hosted SOC workbench. Seraph investigates; Kerub guards.

## 9. Status

Design phase. See the other documents in `docs/`:

- [threat-model.md](threat-model.md) — what Kerub defends against, and what it doesn't
- [requirements.md](requirements.md) — functional and non-functional requirements
- [architecture.md](architecture.md) — how Kerub is built
- [adr/](adr/) — architecture decision records
- [roadmap.md](roadmap.md) — milestones and tasks