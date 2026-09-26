# ADR-0003: Use the Windows Filtering Platform for network enforcement

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | FR-NET-002 to FR-NET-007, FR-RSP-001, SEC-NET-001, SEC-RSP-002, TM-NET-1, TM-NET-2, TM-RSP-1, TM-RSP-2, ADR-0001, ADR-0006 |

## Context

Kerub must filter traffic by interface, address, port and application; keep
the VPN kill switch active if the service crashes and during boot
(FR-NET-007, SEC-NET-001); own every rule it creates so it can remove them
cleanly (SEC-RSP-002); and coexist with Windows Defender Firewall and VPN
clients (NFR-COMP-002).

## Options considered

### Windows Defender Firewall rules (COM `INetFwPolicy2` or `netsh advfirewall`)

- Pros: simple, well documented.
- Cons: Kerub's rules are mixed with user, policy and third-party rules and
  can be edited by any admin in the firewall console; no boot-time
  semantics; limited conditions; cleanup relies on naming conventions only.

### WFP management API from user mode

- Pros: Kerub's own **provider** and **sublayer**; filters can be
  **persistent** and **boot-time**; rich conditions (interface LUID, IP,
  port, application); every object tagged with Kerub's provider, so it can
  be enumerated and deleted precisely; changes grouped in transactions; this
  is how VPN clients implement kill switches.
- Cons: complex and sparsely documented API; mistakes can cut all
  connectivity, even across reboots.

### WFP callout driver or NDIS filter

- Kernel-mode code. Rejected by ADR-0006.

## Decision

We will enforce all network rules through the **WFP management API from
user mode**, called through `windows-rs` inside `kerub-platform`, with these
rules:

- A small safe wrapper owns the engine handle and transactions (released
  automatically when dropped); every change is made inside a transaction
  that is either committed or aborted as a whole.
- Every filter belongs to Kerub's provider and lives in Kerub's sublayer.
- **Kerub only adds restrictions.** It never uses "hard permit" filters to
  override other sublayers: a block from Windows Firewall or another product
  still applies.
- Kill-switch and profile filters are persistent and boot-time; temporary
  responses (IP blocks) are persistent with an expiry handled by the action
  manager.

**Identifiers** (generated once in task T0.1, never changed):

| Object | GUID |
|--------|------|
| Kerub provider | *to be generated in T0.1* |
| Kerub sublayer | *to be generated in T0.1* |

## Consequences

**Positive**

- Protection survives crashes and reboots.
- Exact ownership: uninstall and emergency reset delete everything Kerub
  created, and nothing else.
- Atomic changes: a half-applied rule set cannot exist.
- Fine-grained conditions and good performance.

**Negative**

- A faulty persistent or boot-time filter can leave the machine offline
  **even after a reboot**. Mitigations: all network development happens in
  the lab VM; [dev/recovery.md](../dev/recovery.md) and the emergency reset
  exist **before** the first filter is written.
- The wrapper is `unsafe` FFI code: it gets the most careful review in the
  project.
- Harder debugging (`netsh wfp show filters`, `netsh wfp show state`).

**Follow-up**

- T0.1: generate and record both GUIDs.
- Recovery procedure and emergency reset before any WFP task.