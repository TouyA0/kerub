# Kerub — Event Schema

| | |
|---|---|
| **Schema version** | 1 (`kerub.schema_version`) |
| **Status** | Draft |
| **Related** | [architecture.md](architecture.md) §4, [ipc-protocol.md](ipc-protocol.md), [privacy.md](privacy.md), [requirements.md](requirements.md) (FR-DET-003, FR-VIS-002, FR-VIS-005) |

Every sensor, the detection engine, the store, the agent and every export
use **one normalized format**. This document defines it.

**Source of truth.** The Rust types in `kerub-core` (`event` module) define
the schema; this document describes them. A change to one must update the
other in the same pull request.

---

## 1. Principles

1. **ECS first.** Kerub follows the
   [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/index.html)
   (ECS). When an ECS field exists for a piece of information, Kerub uses it,
   with its exact name and meaning. Events can therefore be sent to Elastic,
   Wazuh, Seraph or any ECS-aware SIEM without translation.
2. **Custom fields live under `kerub.*`.** Anything ECS does not cover goes
   into the `kerub` namespace, so it can never collide with a future ECS
   field.
3. **Collect what is needed, not everything.** A field is stored only if a
   rule, the UI or an export needs it. The raw original event
   (`event.original`) is **not** stored by default (see [privacy.md](privacy.md)).
4. **Nested JSON.** Fields are nested objects (`{"event": {"kind": …}}`),
   never dotted keys (`{"event.kind": …}`). This document writes paths with
   dots for readability only.

---

## 2. Three record types

| Record | Produced by | `event.kind` | Meaning |
|--------|-------------|--------------|---------|
| **Event** | Sensors | `event` | Something was observed |
| **Alert** | Detection engine | `alert` | A rule matched one or more events |
| **Action** | Action manager | `event` (with `kerub.action.*`) | Kerub changed, or simulated changing, the system |

All three share the common fields below and are stored, exported and shown
on the timeline the same way.

---

## 3. Common fields (always present)

| Field | Type | Description |
|-------|------|-------------|
| `@timestamp` | date | When the thing **happened** (source time), UTC |
| `event.created` | date | When Kerub **observed** it, UTC |
| `event.id` | keyword | UUID v7 (time-ordered), unique per record |
| `event.kind` | keyword | `event` or `alert` (§2) |
| `event.category` | keyword[] | ECS category (§5) |
| `event.type` | keyword[] | ECS type (§5) |
| `event.action` | keyword | What happened, in kebab-case (`logon-failed`, `ip-blocked`…) |
| `event.outcome` | keyword | `success`, `failure` or `unknown` |
| `event.module` | keyword | Kerub module that produced it (`logwatch`, `usbguard`…) |
| `event.dataset` | keyword | `kerub.<module>` |
| `event.severity` | integer | 0 to 4 (§6) |
| `message` | text | One plain-language sentence, shown in the UI |
| `host.name` | keyword | Machine name |
| `agent.type` | keyword | Always `kerub` |
| `agent.version` | keyword | Kerub version |
| `ecs.version` | keyword | ECS version Kerub follows, pinned in `kerub-core` |
| `kerub.schema_version` | integer | This document's version: `1` |

**Sensor-sourced events** also carry:

| Field | Type | Description |
|-------|------|-------------|
| `event.provider` | keyword | Original source, e.g. `Microsoft-Windows-Security-Auditing`, `Microsoft-Windows-Sysmon` |
| `event.code` | keyword | Original event ID, e.g. `4625` |

---

## 4. Field sets used by Kerub

Only the fields Kerub actually fills are listed. All are standard ECS unless
they start with `kerub.`.

| Field set | Fields used |
|-----------|-------------|
| `user.*` | `user.name`, `user.domain`, `user.id` (SID) |
| `group.*` | `group.name`, `group.id` |
| `source.*` / `destination.*` | `ip`, `port`, `address` |
| `network.*` | `network.transport`, `network.direction`, `network.type` (`ipv4`/`ipv6`) |
| `process.*` | `pid`, `executable`, `name`, `command_line`, `hash.sha256`, `parent.pid`, `parent.executable` |
| `file.*` | `path`, `name`, `hash.sha256`, `code_signature.exists`, `code_signature.trusted`, `code_signature.subject_name` |
| `rule.*` | `id`, `name`, `description`, `ruleset`, `version` |
| `threat.*` | `framework`, `technique.id`, `technique.name`, `tactic.name` |
| `kerub.logon.*` | `type` (Windows logon type) |
| `kerub.usb.*` | `vendor_id`, `product_id`, `serial`, `class`, `instance_id`, `decision` |
| `kerub.network.*` | `profile`, `interface`, `ssid`, `vpn_active` |
| `kerub.posture.*` | `check`, `result`, `expected`, `observed` |
| `kerub.alert.*` | `related_events` (event IDs), `recommendation` |
| `kerub.action.*` | `id`, `kind`, `state`, `mode`, `trigger`, `expires_at`, `reverts` |
| `kerub.truncated` | `true` when at least one field was truncated (§8) |

---

## 5. Mapping of Kerub event families

This table is the reference for sensor implementers. Each new event family
adds a row.

| Family | Source | `event.category` | `event.type` | `event.action` | Key fields |
|--------|--------|------------------|--------------|----------------|------------|
| Successful logon | Security 4624 | `authentication` | `start` | `logged-in` | `user.*`, `source.ip`, `kerub.logon.type` |
| Failed logon | Security 4625 | `authentication` | `start` | `logon-failed` | `user.*`, `source.ip`, `kerub.logon.type` |
| Account created | Security 4720 | `iam` | `user`, `creation` | `user-created` | `user.*` (target), `group.*` |
| Added to privileged group | Security 4732 | `iam` | `group`, `change` | `added-member-to-group` | `user.*`, `group.*` |
| Security log cleared | Security 1102 | `configuration` | `deletion` | `audit-log-cleared` | `user.*` |
| Process started | Sysmon 1 | `process` | `start` | `process-started` | `process.*`, `user.*` |
| Network connection | Sysmon 3 | `network` | `connection`, `start` | `connection-attempted` | `source.*`, `destination.*`, `network.*`, `process.executable` |
| Network changed | netwatch | `network` | `change` | `network-changed` | `kerub.network.*` |
| Device connected | devwatch | `host` | `device`, `creation` | `device-connected` | `kerub.usb.*` |
| Posture check | posture | `configuration` | `info` | `posture-checked` | `kerub.posture.*` |

Where ECS gives no obvious category (device arrivals, log clearing), the
choice above is Kerub's convention and must stay stable.

### 5.1 Alerts

Alerts add, on top of the common fields:

- `event.kind`: `alert`;
- `rule.*`: the rule that matched (`rule.ruleset` is `sigma` or `threshold`);
- `threat.*`: the MITRE ATT&CK mapping (`threat.framework` is
  `MITRE ATT&CK`) — mandatory for every alert (FR-DET-003);
- `kerub.alert.related_events`: IDs of the events that triggered it;
- `kerub.alert.recommendation`: what the user can do, in plain language;
- the most relevant fields of the triggering events (for example
  `source.ip`), copied for display without a lookup.

### 5.2 Actions

Actions add `kerub.action.*`:

| Field | Values |
|-------|--------|
| `kind` | `ip-block`, `device-allow`, `device-block`, `network-profile`, `setting-change`… |
| `state` | `planned`, `applied`, `simulated` (audit mode), `failed`, `reverted`, `expired` |
| `mode` | `audit` or `enforce` |
| `trigger` | ID of the alert, user request or policy that caused it |
| `expires_at` | Date, when the action is temporary |
| `reverts` | ID of the action this one undoes, if any |

Each state change of an action produces a new record, so the timeline shows
its whole life.

---

## 6. Severity

| `event.severity` | Label | Sigma level | UI behavior |
|------------------|-------|-------------|-------------|
| 0 | info | informational | Timeline only |
| 1 | low | low | Timeline, grouped notifications |
| 2 | medium | medium | Notification |
| 3 | high | high | Immediate notification |
| 4 | critical | critical | Immediate notification, stays until acknowledged |

Plain events usually have severity 0; the severity of an alert comes from
its rule.

---

## 7. Time and identifiers

- All dates are RFC 3339 strings in UTC with millisecond precision
  (`2026-09-26T08:15:02.431Z`).
- `@timestamp` comes from the source when available; otherwise it equals
  `event.created`.
- Record IDs are **UUID v7**: they sort by creation time, which keeps
  database indexes efficient.

---

## 8. Size limits and truncation

| Item | Limit |
|------|-------|
| `process.command_line` | 2 048 characters |
| `message` | 512 characters |
| Other string fields | 1 024 characters |
| Whole record, serialized | 16 KiB |

Truncated values end with `…` and set `kerub.truncated: true`. A record
still over the limit after truncation is dropped, counted, and reported as a
sensor error.

---

## 9. Sensitive fields

Some fields may contain personal or secret data. Handling rules are in
[privacy.md](privacy.md); in short:

| Field | Risk | Handling |
|-------|------|----------|
| `process.command_line` | May contain passwords or tokens | Best-effort redaction of known secret patterns; truncated |
| `user.name`, `user.id` | Personal data | Stored locally only; exported only when export is enabled |
| `destination.*`, `kerub.network.ssid` | Reveal activity and places | Retention limits apply |
| `event.original` | Everything above, unfiltered | Not stored by default |

---

## 10. Examples

**Failed logon (event):**

```json
{
  "@timestamp": "2026-09-26T08:15:01.912Z",
  "event": {
    "id": "01923f4e-9a1b-7c3d-8e2f-4a5b6c7d8e9f",
    "kind": "event",
    "category": ["authentication"],
    "type": ["start"],
    "action": "logon-failed",
    "outcome": "failure",
    "module": "logwatch",
    "dataset": "kerub.logwatch",
    "provider": "Microsoft-Windows-Security-Auditing",
    "code": "4625",
    "created": "2026-09-26T08:15:02.004Z",
    "severity": 0
  },
  "message": "Failed logon for user admin from 203.0.113.7.",
  "user": { "name": "admin", "domain": "DESKTOP-01" },
  "source": { "ip": "203.0.113.7" },
  "host": { "name": "DESKTOP-01" },
  "agent": { "type": "kerub", "version": "0.1.0" },
  "ecs": { "version": "<pinned>" },
  "kerub": { "schema_version": 1, "logon": { "type": 10 } }
}
```

**Brute-force alert:**

```json
{
  "@timestamp": "2026-09-26T08:15:02.431Z",
  "event": {
    "id": "01923f4e-9c20-7d11-a2b3-c4d5e6f70812",
    "kind": "alert",
    "category": ["intrusion_detection"],
    "type": ["indicator"],
    "action": "brute-force-detected",
    "outcome": "unknown",
    "module": "logwatch",
    "dataset": "kerub.logwatch",
    "created": "2026-09-26T08:15:02.431Z",
    "severity": 3
  },
  "message": "10 failed logons from 203.0.113.7 in 60 seconds.",
  "rule": {
    "id": "kerub-thr-0001",
    "name": "Logon brute force",
    "ruleset": "threshold",
    "version": "1"
  },
  "threat": {
    "framework": "MITRE ATT&CK",
    "technique": { "id": ["T1110"], "name": ["Brute Force"] },
    "tactic": { "name": ["Credential Access"] }
  },
  "source": { "ip": "203.0.113.7" },
  "host": { "name": "DESKTOP-01" },
  "agent": { "type": "kerub", "version": "0.1.0" },
  "ecs": { "version": "<pinned>" },
  "kerub": {
    "schema_version": 1,
    "alert": {
      "related_events": ["01923f4e-9a1b-7c3d-8e2f-4a5b6c7d8e9f"],
      "recommendation": "If you do not recognize this address, keep it blocked. Check that remote desktop is really needed on this machine."
    }
  }
}
```

**Resulting action (enforce mode):**

```json
{
  "@timestamp": "2026-09-26T08:15:02.510Z",
  "event": {
    "id": "01923f4e-9c6e-7f00-b1c2-d3e4f5a6b7c8",
    "kind": "event",
    "category": ["network"],
    "type": ["denied", "creation"],
    "action": "ip-blocked",
    "outcome": "success",
    "module": "respond",
    "dataset": "kerub.respond",
    "created": "2026-09-26T08:15:02.510Z",
    "severity": 2
  },
  "message": "Blocked 203.0.113.7 for 1 hour.",
  "source": { "ip": "203.0.113.7" },
  "host": { "name": "DESKTOP-01" },
  "agent": { "type": "kerub", "version": "0.1.0" },
  "ecs": { "version": "<pinned>" },
  "kerub": {
    "schema_version": 1,
    "action": {
      "id": "01923f4e-9c6e-7f00-b1c2-d3e4f5a6b7c8",
      "kind": "ip-block",
      "state": "applied",
      "mode": "enforce",
      "trigger": "01923f4e-9c20-7d11-a2b3-c4d5e6f70812",
      "expires_at": "2026-09-26T09:15:02.510Z"
    }
  }
}
```

---

## 11. Storage and export

- **Store:** each record is saved as its JSON document, with the fields used
  for filtering (`@timestamp`, `event.kind`, `event.category`,
  `event.severity`, `event.module`) also extracted into indexed columns.
- **Export:** JSON Lines, one record per line, exactly as defined here
  (FR-VIS-005).
- **IPC:** the timeline and events sent to the agent use the same records;
  the agent never builds its own format.

---

## 12. Versioning

- `kerub.schema_version` increases for any **breaking** change: removed or
  renamed field, changed meaning or type.
- Adding an optional field or a new event family is **not** breaking; it
  is listed in the CHANGELOG.
- Upgrading the pinned ECS version is a deliberate change reviewed like any
  other.