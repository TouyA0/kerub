# ADR-0004: Use SQLite through rusqlite for local storage

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | FR-RSP-003, FR-VIS-001, FR-CFG-002, NFR-PERF-005, NFR-REL-003, TM-ST-5, ADR-0001 |

## Context

The service stores events, alerts, the action journal, exceptions, the
device allow-list and configuration versions. The journal must be
transactional (journal before acting, FR-RSP-003). The timeline needs
filtering and search (FR-VIS-001). Only the service writes. Storage must be
size-capped with retention (NFR-PERF-005).

## Options considered

### Flat files (JSON / JSON Lines)

- Pros: simplest, human-readable.
- Cons: no transactions, no queries, manual indexing.

### redb (embedded key-value store, pure Rust)

- Pros: transactional, pure Rust, stable.
- Cons: every query written by hand; no ad-hoc inspection.

### sqlx (async SQL toolkit)

- Pros: async API, compile-time checked queries.
- Cons: heavier, macros need a database or offline metadata at build time;
  async brings little for a single local writer.

### SQLite through rusqlite, bundled

- Pros: SQL queries, ACID transactions, universal tooling for inspection,
  single file; the `bundled` feature compiles a pinned SQLite version into
  the binary, with no system dependency.
- Cons: SQLite is C code compiled into Kerub; synchronous API.

### External database server

- Out of proportion for a single-machine tool.

## Decision

We will use **SQLite through `rusqlite` with the `bundled` feature**, with
these rules:

- The store owns the only connection, on a dedicated thread; async code
  talks to it through a channel. This matches the single-writer design.
- WAL journal mode, foreign keys enabled, busy timeout set.
- Numbered SQL migrations in `crates/kerub-store/migrations`, embedded in
  the binary and applied at startup in a transaction.
- The hash-chained **security audit log is not stored in SQLite**: it is a
  separate append-only file with its own integrity model (architecture
  §6.4).

## Consequences

**Positive**

- Transactions make the action journal and reconciliation reliable.
- Timeline queries are simple SQL.
- The database can be inspected with standard SQLite tools during
  development.

**Negative**

- SQLite is C code inside a Rust binary. Accepted: it is one of the most
  thoroughly tested libraries in existence; its version is tracked by
  Dependabot through `libsqlite3-sys`.
- The database is readable by administrators; encryption at rest is out of
  scope and relies on BitLocker (documented in privacy.md).

**Follow-up**

- Retention job and size cap (NFR-PERF-005).
- Migration tests in CI.