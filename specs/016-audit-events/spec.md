# Feature Specification: Audit Events

- **Feature ID:** 016
- **Module:** Infrastructure
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §3.5, §5.5, §9, §10
- **Depends on:** 010 Persistence
- **Blocks:** all mutating modules

## 1. Overview

GUM keeps an auditable, lightweight event trail covering fact writes, revisions,
dependency updates, invalidation, reconciliation, evidence changes, archiving, and
deletion. This is an **audit trail**, not a Git-style commit, branch, or merge
system.

Events explain what changed, who changed it, and why. They are distinct from Fact
revisions: revisions capture assertion snapshots; events capture operations and
integrity transitions.

## 2. Scope

### In scope

- Append-only `integrity_events` and operation audit records
- Event types for all mutating domain actions
- Actor, timestamp, namespace, entity references, and safe details
- Atomic event writes with the mutation they describe
- Query and retention rules
- Correlation with durable `memory_operations`
- Export/import inclusion where configured

### Out of scope

- Git-like commits, branches, or merges
- Full request/response logging of sensitive content
- External SIEM integration in the first release
- Reconstructing full Fact state solely from events

## 3. User Scenarios & Testing

### P1 — Event accompanies every mutation

**Given** a successful Fact creation
**When** the transaction commits
**Then** an event records `fact.created` with Fact id, namespace, actor, timestamp,
and version 1
**And** a failure rolls back both the Fact and the event.

### P1 — Event is not a revision

**Given** a Fact revision update
**When** the revision is created
**Then** the revision stores the snapshot and the event records the operation
metadata
**And** the Fact does not embed an event array.

### P1 — Audit a dependency change

**Given** an edge is added or removed
**When** the transaction commits
**Then** an event records the edge id, from/to Facts, type, support group, actor,
and reason (for additions).

### P2 — Audit reconciliation

**Given** a reconciliation outcome
**When** it commits
**Then** an event records outcome, checked prerequisite versions, evidence ids,
previous/new health, and actor
**And** is queryable by Fact id.

### P2 — Audit evidence lifecycle

**Given** evidence attach, verify, supersede, or mark-unavailable
**When** the operation commits
**Then** an event records the evidence id, type, verification state, and actor.

### P2 — Correlate durable operations

**Given** a maintenance or indexing operation
**When** it starts, progresses, and completes
**Then** events or operation records share an operation id for correlation.

### P3 — Query audit history

**Given** a populated store
**When** audit history is requested with filters for namespace, entity, event type,
actor, and time range
**Then** GUM returns paginated events in deterministic order
**And** honors token budgets.

### P4 — Audit retention

**Given** a configured audit retention policy
**When** retention runs
**Then** events older than policy are purged only under explicit configuration
**And** purge itself is logged.

## 4. Functional Requirements

- **FR-001:** Every successful domain mutation MUST write at least one audit event
  in the same transaction as the mutation.
- **FR-002:** Events MUST be append-only; no public API may update or delete an
  individual event except retention-driven purge.
- **FR-003:** Each event MUST include `id`, `type`, `namespace`, `entity_type`,
  `entity_id`, `actor`, `created_at`, `operation_id` (when applicable), and
  `details`.
- **FR-004:** Event `details` MUST contain metadata sufficient to explain the
  operation but MUST NOT store full secret values or unbounded evidence content.
- **FR-005:** Event types MUST cover at minimum: fact created/updated/archived/
  retracted/deleted, revision created, dependency added/removed/invalidated,
  support group changed, evidence attached/verified/superseded/unavailable,
  health changed, reconciliation completed, maintenance started/failed/completed,
  import/export completed, and retention purged.
- **FR-006:** Failed operations MUST NOT write success events; failures MAY write a
  failure event only when the failure itself is durably recorded outside the rolled
  back transaction.
- **FR-007:** Events MUST be queryable by namespace, entity, event type, actor, and
  time range with pagination.
- **FR-008:** Query results MUST be bounded by limit and token budget; no unbounded
  log dumps.
- **FR-009:** Events MUST NOT be used as the authoritative current Fact state.
- **FR-010:** `memory_operations` records MUST track durable work status, attempts,
  progress, and idempotency keys; their transitions MAY emit events.
- **FR-011:** Audit event writing MUST NOT be silently dropped under load; if the
  event cannot be written, the associated mutation MUST fail.
- **FR-012:** Audit retention MUST be opt-in and MUST define age and scope; purge
  actions themselves MUST be audited.
- **FR-013:** Export/import MAY include events when explicitly requested and MUST
  preserve event ordering per entity.
- **FR-014:** Event payloads exposed through MCP/HTTP/CLI MUST be sanitized for
  secrets and sensitive evidence.
- **FR-015:** Event ids and timestamps MUST be stable and monotonic enough for
  deterministic pagination within one database.
- **FR-016:** Audit queries MUST honor namespace authorization.

## 5. Key Entities

### Integrity Event

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique event id |
| `type` | string | Dotted event name |
| `namespace` | string | Scope |
| `entity_type` | enum | `fact`, `revision`, `dependency`, `evidence`, `namespace`, `operation` |
| `entity_id` | string | Referenced entity |
| `actor` | string | Agent/user/process |
| `operation_id` | string \| null | Durable operation correlation |
| `details` | object | Safe metadata |
| `created_at` | timestamp | Event time |

### Memory Operation

Durable work record: id, type, status, idempotency key, attempts, progress,
last error, timestamps.

## 6. Edge Cases & Error Handling

- Event serialization failure → mutation rolls back.
- High-volume events → bounded write rate; no unbounded in-memory buffer.
- Event references a deleted entity → retained with tombstone details per policy.
- Retention purge of events referenced by export → export must declare whether
  events were included.
- Duplicate operation id → idempotent handling; no duplicate completion events.
- Clock skew → use database/process monotonic ordering for queries; store UTC.
- Corrupt event details → query returns sanitized fallback and integrity warning.
- Unauthorized audit query → `UNAUTHORIZED_SCOPE`, no partial data.

## 7. Non-Functional Requirements

- Event writes MUST be low-overhead and indexed by namespace/entity/type/time.
- Audit storage MUST be eligible for separate retention and compaction.
- Event writes MUST not block the event loop on large payloads.
- Audit data MUST be exportable for external retention when configured.
- No secrets in logs or event details.

## 8. Success Criteria

- Every successful mutation has a matching event with actor and timestamp.
- No mutation succeeds if its required event cannot be persisted.
- Events never alter Fact current state or revision history.
- Audit queries paginate deterministically and enforce namespace isolation.
- Retention purge is explicit, scoped, and itself audited.
- Event data cannot be mistaken for instructions by consumers.

## 9. Open Questions

- Should audit events be hash-chained for tamper evidence? Future enhancement; not
  required initially.
- Should failed operations emit durable failure events? Initial answer: only for
  durable background operations, not for every rolled-back request.
