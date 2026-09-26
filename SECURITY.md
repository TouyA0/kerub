# Security Policy

Kerub is a security tool whose service runs with SYSTEM privileges. A
vulnerability in Kerub could harm the very machines it is meant to protect,
so security reports are taken seriously and handled privately.

## Supported versions

| Version | Supported |
|---------|-----------|
| Design phase (no release yet) | Reports on the design are welcome |
| Latest release, once published | ✅ Security fixes |
| Older releases | ❌ Please upgrade |

## Reporting a vulnerability

**Do not open a public issue, pull request or discussion for a security
problem.**

Report it privately through GitHub:
**[Report a vulnerability](https://github.com/TouyA0/kerub/security/advisories/new)**
(repository *Security* tab → *Report a vulnerability*).

Please include:

- the affected version or commit;
- the component (`kerub-svc`, `kerub-agent`, `kerub-cli`, installer,
  scripts, documentation);
- a description of the issue and its impact;
- the steps to reproduce it, ideally a minimal proof of concept;
- the relevant threat or requirement ID from the
  [threat model](docs/threat-model.md) or the
  [requirements](docs/requirements.md), if you know it;
- whether you would like to be credited, and under which name.

## What happens next

Kerub is maintained by one person on their own time; these are targets, not
guarantees.

| Step | Target |
|------|--------|
| Acknowledgement of your report | 7 days |
| Initial assessment and severity | 14 days |
| Fix for a critical or high-severity issue | 30 days |
| Fix for other issues | Next planned release |

You will be kept informed at each step. Once a fix is available, a GitHub
security advisory is published, crediting you unless you prefer to stay
anonymous.

**Coordinated disclosure:** please allow up to **90 days** from your report
before publishing details, or less if we agree on it together.

## Scope

**In scope**

- Privilege escalation, authentication or authorization bypass, including
  through the IPC channel.
- Ways to disable, bypass or tamper with a Kerub protection without
  administrator rights.
- Remote reachability of any Kerub component.
- Leaks through the VPN kill switch (traffic, DNS, IPv6).
- Exposure of secrets or personal data beyond what [privacy.md](docs/privacy.md)
  describes.
- Memory-safety issues, including in `unsafe` code.
- Security mistakes in the documentation that could mislead users (for
  example in the recovery procedures).

**Out of scope**

- The accepted risks listed in the
  [threat model](docs/threat-model.md#6-accepted-risks) — for example an
  attacker who already has administrator rights disabling Kerub — unless
  you found a way to make them significantly worse.
- Vulnerabilities in third-party dependencies: please report them to the
  upstream project, and let us know if Kerub is affected in a specific way.
- Social engineering, and physical attacks outside the threat model.
- Findings from automated scanners without a demonstrated impact.

## Safe harbor

Good-faith security research is welcome. When testing:

- only test on machines and networks **you own or are explicitly authorized
  to test**;
- do not access, modify or destroy other people's data;
- do not degrade services used by others;
- give us reasonable time to fix the issue before disclosure.

Research carried out within these rules will not be pursued in any way by
the Kerub project.

There is no bug bounty program.

## Security design

Kerub's security design is documented in the open:

- [Threat model](docs/threat-model.md) — threats, mitigations, accepted risks
- [Security requirements](docs/requirements.md#3-security-requirements)
- [IPC protocol](docs/ipc-protocol.md) — how the privileged service
  authenticates its clients
- [Architecture decisions](docs/adr/)