# Feature Specification: Revision History

- **Feature ID:** 002
- **Module:** Core Memory
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §3.2, §5.5, §9 (Revision History)
- **Depends on:** 001 Fact Management, 010 Persistence, 016 Audit Events
- **Blocks:** 003, 005, 006, 008

## 1. Overview

Every accepted Fact mutation produces an immutable revision snapshot. Revisions
are linear within a store, monotonic per Fact, and protected by optimistic
concurrency. Revision history answers **"How did this assertion change?"**; the
dependency graph answers **"What relies on this assertion?"**. These are separate
concerns and MUST NOT be conflated.

## 2. Scope

### In scope

- Immutable Fact revision snapshots
- Optimistic version checking (`expected_version`)
- `get_revisions`, `get_revision`, `compare_revisions`, `inspect_history`
- Change types: initial, correction, supersession, retraction, detail update
- Previous-revision links and author attribution
- Evidence references recorded with revisions

### Out of scope

- Branching, merging, or Git-like history (explicitly rejected by CONCEPT.md)
- Dependency-graph edges (003)
- Revision-based restore operations (possible future work)
- Retention-based purge execution (009)

## 3. User Scenarios & Testing

### P1 — Reject stale writes

**Given** Fact `F001` at version 3
**When** writer A updates with `expected_version: 3` and commits version 4
**And** writer B attempts an update with `expected_version: 3`
**Then** writer B receives `STALE_WRITE`
**And** no revision 5 or Fact mutation is created by B.

### P1 — Atomic revision and Fact update

**Given** a valid update request
**When** the transaction commits
**Then** a new revision with `version = current_version + 1`,
`previous_revision_id`, author, change type, and snapshot is written
**And** the Fact pointer and `updated_at` are updated in the same transaction
**And** an audit event is recorded.

### P2 — Inspect revision history

**Given** a Fact with five revisions
**When** `get_revisions` is called with pagination
**Then** revisions are returned newest-first by default with stable tie-breaking
**And** each revision includes id, fact id, version, change type, author, and timestamp
**And** full statement/details are included only for the requested page.

### P2 — Compare two revisions

**Given** revisions 2 and 4 of the same Fact
**When** `compare_revisions` is called
**Then** GUM returns statement changes, details changes, change types, authors,
timestamps, and evidence-reference differences
**And** the comparison is read-only.

### P3 — Retraction is a revision

**Given** an active Fact
**When** it is retracted
**Then** a revision with `change_type = "retraction"` is recorded, status becomes
`retracted`, and downstream impact analysis is triggered through 005.

### P4 — History inspection summary

**Given** a Fact with many revisions
**When** `inspect_history` is called
**Then** GUM returns a compact summary (count, version range, authors, change-type
counts, first/last timestamps) without loading all snapshots.

## 4. Functional Requirements

- **FR-001:** A Fact MUST NOT store a revision array; revisions MUST live in the
  Revision entity keyed by `fact_id`.
- **FR-002:** Every accepted update MUST require `expected_version`, and the write
  MUST fail with `STALE_WRITE` if it does not match the Fact's current version.
- **FR-003:** The revision version MUST be exactly `previous_version + 1`; gaps and
  duplicate versions for one Fact are forbidden.
- **FR-004:** Revisions MUST be immutable. No public API may edit or delete an
  accepted revision; only retention-driven purge (009) may remove data, and it
  must leave an auditable tombstone.
- **FR-005:** Each revision MUST record `revision_id`, `fact_id`,
  `previous_revision_id` (nullable for initial), `version`, `statement`,
  `details`, `change_type`, `author`, `created_at`, and `evidence_refs`.
- **FR-006:** `change_type` MUST be one of `initial`, `correction`,
  `supersession`, `retraction`, or `detail_update`.
- **FR-007:** Revision creation, Fact head update, dependency-version impact
  marking, and audit event MUST commit atomically when they share the transaction.
- **FR-008:** A non-substantive metadata-only edit MAY skip revision creation when
  policy explicitly classifies it as metadata; the default is to create a revision.
- **FR-009:** `get_revisions` MUST support pagination, ordering, and optional
  inclusion of full snapshots; it MUST NOT return unbounded history by default.
- **FR-010:** `get_revision` MUST return exactly one revision scoped to its Fact.
- **FR-011:** `compare_revisions` MUST reject revisions belonging to different Facts.
- **FR-012:** `inspect_history` MUST NOT mutate state and MUST honor result limits.
- **FR-013:** A stale writer MUST receive enough information to re-read and retry:
  current version, latest revision id, and updated_at.
- **FR-014:** The `author` MUST identify the acting agent, user, or process; anonymous
  writes MUST be rejected unless the namespace explicitly enables them.
- **FR-015:** Revisions MUST be included in export/import with chain integrity
  preserved (011).

## 5. Key Entities

### Fact Revision

| Field | Type | Notes |
|---|---|---|
| `revision_id` | string | Unique revision identifier |
| `fact_id` | string | Parent Fact |
| `previous_revision_id` | string \| null | Null only for initial revision |
| `version` | integer | Monotonic per Fact |
| `statement` | string | Historical snapshot |
| `details` | object | Historical snapshot |
| `change_type` | enum | See FR-006 |
| `author` | string | Agent/user/process identity |
| `created_at` | timestamp | Recording time |
| `evidence_refs` | string[] | Evidence IDs supplied with the change |

## 6. Edge Cases & Error Handling

- Two concurrent writers from the same base version → exactly one succeeds.
- Retrying after `STALE_WRITE` → caller must re-read; GUM must not auto-merge text.
- Revision chain missing a `previous_revision_id` (corruption) → integrity report
  and import rejection.
- Update with unchanged statement/details → allowed only as an explicit no-op or
  metadata update; must not create misleading change history.
- Update to a retracted Fact → only through 006 reconciliation.
- Very long statement/details → enforce configured size limits before persisting.
- Revision ID collisions → impossible by design; enforce unique constraint.

## 7. Non-Functional Requirements

- Version checks and writes MUST be atomic under WAL with one writer at a time.
- Revision reads MUST use indexes on `(fact_id, version)` and `revision_id`.
- History responses MUST support token/result budgets.
- Immutability MUST be enforced at the storage layer via append-only operations.

## 8. Success Criteria

- 100 concurrent writers starting from version 1 produce exactly one accepted
  version 2 and 99 `STALE_WRITE` errors.
- Every Fact has exactly one revision per version from 1 to `current_version`.
- `compare_revisions` output round-trips the differences without mutation.
- Revising a Fact preserves all prior revision snapshots bit-for-bit.
- Crash during a write leaves either the old state or the fully committed new state,
  never a Fact head without its revision.

## 9. Open Questions

- Should a `restore_revision` operation exist? Not in current scope; restoration
  is expressed as a new `correction` revision with copied content.
- Should metadata-only edits create revisions by default or be logged only as
  events? Initial answer: revisions by default for auditability.
