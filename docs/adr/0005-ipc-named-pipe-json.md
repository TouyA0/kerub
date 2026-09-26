# ADR-0005: Use a named pipe with JSON messages for IPC

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | TM-IPC-1 to TM-IPC-6, SEC-IPC-001 to SEC-IPC-005, ADR-0001, [ipc-protocol.md](../ipc-protocol.md) |

## Context

The agent and the CLI talk to the service across trust boundary TB1. Both
ends must authenticate each other (TM-IPC-1, TM-IPC-2); the service must
resist malformed input (TM-IPC-3) and flooding (TM-IPC-4); every request
must be auditable (TM-IPC-5). The channel needs request/response **and**
server-pushed messages (alerts, approval prompts).

## Options considered

### Transport

- **Named pipe** — protected by a Windows security descriptor; the peer
  process ID is retrievable on both sides; first-instance creation prevents
  pipe squatting; remote clients can be rejected.
- **Local TCP / HTTP / WebSocket** — no native ACL, reachable by every
  local process and potentially by web pages (CSRF, DNS rebinding), port
  conflicts. Rejected.
- **COM / RPC (ALPC)** — powerful but complex and heavy for this need.
- **Shared memory** — no framing, no access control per message. Rejected.

### Encoding

- **JSON** — `serde`, human-readable (debugging, audit logging), easy to
  validate and fuzz.
- **Protobuf / gRPC** — schemas and code generation, extra toolchain, opaque
  binary format; overkill for a low message rate.
- **MessagePack / bincode** — compact, but not human-readable.

## Decision

We will use the named pipe **`\\.\pipe\kerub`** with **JSON messages**:

- The pipe server is created with `tokio`'s named pipe support, as **first
  instance**, **rejecting remote clients** (a Windows named pipe is
  otherwise reachable over the network through SMB), and with a security
  descriptor built in `kerub-platform` (the only place allowed to use
  `unsafe`).
- Framing: 4-byte little-endian length prefix, then one JSON object;
  maximum message size 64 KiB.
- Every message carries a protocol version, an ID and a type; one
  connection supports requests, responses and a subscription stream.
- Decoding is strict: `serde` with unknown fields denied, then per-type
  validation.
- Rust types in `kerub-ipc` are the single source of truth; TypeScript
  types for the agent front-end are generated from them.

Full specification: [ipc-protocol.md](../ipc-protocol.md).

## Consequences

**Positive**

- Strong local access control and peer identification from the OS.
- No network exposure, and no port opened.
- Messages readable in logs and easy to fuzz.

**Negative**

- Schema discipline relies on shared Rust types and strict decoding rather
  than a schema compiler.
- Building the security descriptor requires `unsafe` FFI code, confined to
  `kerub-platform`.

**Follow-up**

- Fuzz target for the decoder (SEC-IPC-003).
- ipc-protocol.md.
- Add "remote access to the pipe over SMB" to the threat model at its next
  review.