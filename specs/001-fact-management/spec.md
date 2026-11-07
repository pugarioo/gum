# Feature Specification: Fact Management

- **Feature ID:** 001
- **Module:** Core Memory
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §3.1, §7, §9 (Fact Management)
- **Depends on:** 010 Persistence, 016 Audit Events, 017 Namespace & Security
- **Blocks:** 002, 003, 004, 005, 007

## 1. Overview

A Fact is the primary current-state unit of knowledge in GUM. This module owns
the Fact entity, its identity, namespace scope, classification, lifecycle status,
confidence, and current version. It provides create, read, list, and delete
operations. Statement revisions are delegated to 002; dependency relationships
are delegated to 003.

GUM must not treat Facts as free-floating text blobs. Every Fact belongs to a
namespace, has a stable identity, and exposes metadata that retrieval and
integrity services can reason about.

## 2. Scope

### In scope

- `add_fact`, `get_fact`, `list_facts`, `delete_fact`
- Fact identity, statement, structured `details`, namespace, category
- Lifecycle statuses: `active`, `retracted`, `archived`
- `confidence` as a qualified evidence-support assessment
- Initial version and derived dependency health
- Validation and namespace-scoped visibility

### Out of scope

- Revision creation, version conflict handling (002)
- Dependency edges and support groups (003)
- Retrieval ranking and indexing (007, 008)
- Automated purge and retention scheduling (009)
- Namespace creation and authorization policy (017)

## 3. User Scenarios & Testing

### P1 — Create and retrieve a Fact

**Given** an existing namespace `project:shopflow`
**When** an authorized agent calls `add_fact` with statement `"The project uses PostgreSQL."`, details `{ "orm": "Prisma" }`, and category `architecture`
**Then** GUM persists an `active` Fact with `current_version = 1`,
`dependency_health = "healthy"`, an initial revision, and an audit event
**And** `get_fact` returns the statement, details, category, status, confidence,
and version.

### P1 — Reject invalid or unscoped Facts

**Given** an unknown or unauthorized namespace
**When** `add_fact` is called
**Then** GUM rejects the request with a validation or authorization error
**And** no Fact, revision, or audit event is written.

### P2 — List Facts with structured filters

**Given** Facts across two namespaces and several categories
**When** `list_facts` is called with `namespace`, `category`, `status`, and pagination
**Then** GUM returns only Facts matching every supplied filter
**And** archived Facts are excluded unless `status: "archived"` is explicitly requested.

### P2 — Update delegates to revision semantics

**Given** an active Fact at version 3
**When** `update_fact` is called with `expected_version: 3`
**Then** Fact Management validates the Fact and delegates the write to 002
**And** version becomes 4 only after the revision transaction commits.

### P3 — Soft removal by archive

**Given** an active Fact that may still be referenced by dependents
**When** an authorized caller archives it
**Then** status becomes `archived`, the Fact disappears from default retrieval,
and revisions, dependencies, and evidence remain intact.

### P3 — Delete with dependency handling

**Given** a Fact with dependents
**When** `delete_fact` is called without a dependency strategy
**Then** GUM rejects the deletion and reports the blocking dependencies
**When** called with `dependency_strategy: "detach"` or `"cascade"`
**Then** GUM applies the declared strategy atomically and writes a tombstone audit event.

### P4 — Confidence is qualified, not truth

**Given** a Fact with attached evidence
**When** the caller sets `confidence`
**Then** GUM stores the assessment with its basis and does not label the Fact as
objectively true.

## 4. Functional Requirements

- **FR-001:** `add_fact` MUST require a non-empty `statement`, an existing
  authorized namespace, and a valid category.
- **FR-002:** Every Fact MUST have a stable unique identifier that does not change
  across updates, archiving, or export/import.
- **FR-003:** `details` MUST be a structured JSON object stored inline with the
  Fact; it MUST NOT create a separate memory entity.
- **FR-004:** New Facts MUST default to `status = "active"`,
  `dependency_health = "healthy"`, and `current_version = 1`.
- **FR-005:** Creating a Fact MUST atomically write the Fact, its initial
  revision, and an audit event.
- **FR-006:** `get_fact` MUST return the current statement, details, metadata,
  and current version, and MUST reveal the Fact only within authorized namespaces.
- **FR-007:** `list_facts` MUST support filtering by namespace, category, status,
  and dependency health, plus limit/offset pagination with a deterministic order.
- **FR-008:** `update_fact` MUST require an `expected_version` and MUST delegate
  version checks and revision creation to the module defined in 002.
- **FR-009:** `status` MUST accept only `active`, `retracted`, or `archived`;
  lifecycle status MUST remain independent of `dependency_health`.
- **FR-010:** `dependency_health` MUST accept only `healthy`, `possibly_invalid`,
  or `unresolved`, and MUST be written only by integrity/reconciliation services.
- **FR-011:** `confidence` MUST accept a bounded value plus an optional basis
  description; GUM MUST NOT present it as objective truth.
- **FR-012:** `delete_fact` MUST require an explicit dependency strategy when
  dependents exist; supported strategies are `reject` (default), `detach`, and
  `cascade`.
- **FR-013:** Hard deletion MUST remove or rewrite affected graph edges according
  to the chosen strategy and MUST preserve a tombstone audit event.
- **FR-014:** Archived Facts MUST be excluded from default retrieval while
  remaining inspectable with an explicit status filter.
- **FR-015:** All Fact writes MUST be schema-validated (Zod) before reaching
  persistence, and MUST reject unknown fields by default.

## 5. Key Entities

### Fact

| Field | Type | Notes |
|---|---|---|
| `id` | string | Stable unique identifier |
| `statement` | string | Current accepted assertion |
| `details` | object | Embedded structured context |
| `namespace` | string | Authorized scope |
| `category` | string | Knowledge classification |
| `status` | enum | `active`, `retracted`, `archived` |
| `dependency_health` | enum | `healthy`, `possibly_invalid`, `unresolved` |
| `confidence` | number + basis | Evidence-qualified assessment |
| `current_version` | integer | Monotonic; starts at 1 |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last accepted mutation time |

### Namespace reference

A Fact stores a namespace identifier; namespace lifecycle and access rules are
owned by 017.

## 6. Edge Cases & Error Handling

- Empty or whitespace-only statement → `VALIDATION_ERROR`.
- Unknown namespace or unauthorized scope → `UNAUTHORIZED_SCOPE`; no writes.
- Updating a `retracted` Fact → allowed only through reconciliation (006).
- Archiving an already archived Fact → idempotent success or explicit no-op event.
- Deleting a Fact that has evidence or revisions → must follow retention policy
  in 009; deletion is not permitted to orphan audit records silently.
- `cascade` deletion of a Fact with many transitive dependents → must be bounded
  and reported before commit; caller receives affected Fact IDs.
- Concurrent delete and update → optimistic version check decides; loser receives
  `STALE_WRITE`.
- Duplicate statement in the same namespace → allowed, but 009 may suggest
  consolidation; Fact Management MUST NOT auto-merge.

## 7. Non-Functional Requirements

- Fact reads MUST use indexed lookups, not full-table scans.
- Writes MUST be short, atomic SQLite transactions.
- Response payloads MUST exclude historical revisions unless explicitly requested.
- All timestamps MUST be stored in UTC with millisecond precision.

## 8. Success Criteria

- A Fact created in one namespace is never visible through another namespace.
- Creating 1,000 Facts produces exactly 1,000 Facts, 1,000 initial revisions, and
  1,000 creation events.
- A Fact survives process restart with identical id, statement, details, version,
  and timestamps.
- Deleting a Fact with dependents fails without an explicit strategy and leaves
  the graph unchanged.
- `list_facts` pagination returns every Fact exactly once with no duplicates.

## 9. Open Questions

- Should cross-namespace dependency or movement of a Fact between namespaces be
  supported? Initial answer: no, move requires export/import.
- Should `confidence` accept only a 0–1 number plus basis, or a qualified label
  set (`low`, `medium`, `high`) with basis? Decide before implementation.
