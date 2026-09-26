# ADR-0007: License Kerub under Apache-2.0

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | Vision principle 8 (open and verifiable) |

## Context

The repository is public. Goals: a portfolio project others can read, learn
from and reuse; compatibility with dependencies (Tauri, `windows-rs`,
`serde` — MIT or Apache-2.0; `tokio`, `rusqlite`, React — MIT; SQLite —
public domain); clear protection for users and contributors.

## Options considered

### MIT

- Pros: shortest, most permissive, universally known.
- Cons: no explicit patent grant.

### Dual MIT OR Apache-2.0

- Pros: the usual convention for Rust libraries; users pick either.
- Cons: mostly useful for libraries meant to be embedded in other
  projects; Kerub is an application.

### Apache-2.0

- Pros: permissive, explicit patent grant and patent retaliation clause,
  common in security and infrastructure projects, compatible with all
  current dependencies.
- Cons: longer text; modified files must carry change notices; a NOTICE
  file must be kept if one is added.

### GPL-3.0

- Pros: copyleft — distributed forks must stay open source.
- Cons: discourages reuse by companies; stronger obligations for
  contributors and users.

### AGPL-3.0

- Designed for network services; not relevant for a desktop tool.

## Decision

Kerub is licensed under **Apache-2.0**. The `LICENSE` file is copied
verbatim from the official Apache text. Source files may carry an SPDX
identifier (`// SPDX-License-Identifier: Apache-2.0`), and every crate
declares `license = "Apache-2.0"` in its `Cargo.toml`.

## Consequences

**Positive**

- Maximum reuse, patent protection, compatibility with every dependency.

**Negative**

- Anyone may build a closed-source product on Kerub.

**Follow-up**

- Add `LICENSE` at the repository root.
- `cargo deny` checks that every dependency uses a compatible license.
- State in CONTRIBUTING (later) that contributions are accepted under the
  same license.