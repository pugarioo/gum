# Feature Specification: Reconciliation

- **Feature ID:** 006
- **Module:** Integrity Engine
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §5.4, §5.5, §9 (Integrity and Reconciliation)
- **Depends on:** 002, 003, 004, 005, 010, 016
- **Blocks:** 009, 012

## 1. Overview

Reconciliation is how an authorized agent or validator resolves a dependency-health
warning. It records **who/what checked the memory**, **which prerequisite versions
and evidence were considered**, and **the outcome**. A warning may only be cleared
after the relevant assumptions are explicitly reconsidered. Reconciliation never
pretends to establish objective truth.

## 2. Scope

### In scope

- `reconcile_fact` with outcomes: confirm, revise, update dependencies, attach
  evidence, retract, preserve uncertainty
- Snapshot of prerequisite versions and evidence considered
- Clearing, preserving, or escalating dependency-health warnings
- Stale-reconciliation detection
- Reconciliation audit records

### Out of scope

- Automatic reconciliation by an LLM without authorization
- Blind clearing based on retrieval rank or graph position
- Rewriting arbitrary Facts without revision semantics
- Cross-store conflict resolution (not in scope)

## 3. User Scenarios & Testing

### P1 — Confirm a dependent Fact as still applicable

**Given** `F002` is `possibly_invalid` because `F001` changed from version 1 to 2
**When** an authorized agent calls `reconcile_fact` with outcome `confirm`, having
re-read `F001` at version 2 and checked the cited evidence
**Then** GUM records the checked prerequisite versions and evidence
**And** updates the dependency edge assumption to version 2
**And** clears `F002` to `healthy` only if all required assumptions still hold
**And** writes an audit event.

### P1 — Stale reconciliation is rejected

**Given** a reconciliation review started when `F001` was at version 2
**When** `F001` changes to version 3 before the reconciliation commits
**Then** GUM rejects the reconciliation with `STALE_RECONCILIATION`
**And** `F002` remains flagged
**And** the agent must re-check against the new state.

### P2 — Revise the dependent statement

**Given** `F002` is flagged
**When** `reconcile_fact` is called with outcome `revise`, new statement/details,
and `expected_version` of `F002`
**Then** GUM creates a new revision through 002 and re-evaluates the dependency
health against the current prerequisites
**And** records the reconciliation outcome.

### P2 — Update dependency structure

**Given** a dependent relies on an outdated prerequisite
**When** reconciliation outcome `update_dependencies` adds, removes, or regroups
support edges
**Then** GUM validates graph invariants (no cycles, existing Facts), applies the
changes atomically, and re-runs impact analysis.

### P2 — Attach new evidence

**Given** a flagged Fact with no supporting evidence
**When** reconciliation attaches new evidence and confirms
**Then** the evidence is linked, the revision may cite it, and health is cleared
only if the prerequisite assumptions are also satisfied.

### P3 — Retract the dependent Fact

**Given** a dependent Fact that no longer applies
**When** reconciliation outcome is `retract`
**Then** status becomes `retracted`, a retraction revision is created, and reverse
dependents of the retracted Fact are reassessed.

### P3 — Preserve uncertainty

**Given** a flagged Fact that cannot yet be resolved
**When** reconciliation outcome is `preserve_uncertainty` with a note
**Then** `dependency_health` remains `unresolved` or `possibly_invalid`
**And** the warning is annotated with the reviewer and reason rather than cleared.

### P4 — Reconciliation history

**Given** multiple reconciliations of one Fact
**When** `get_integrity_report` or history inspection is requested
**Then** GUM lists each reconciliation event with actor, timestamp, checked
versions, evidence, and outcome.

## 4. Functional Requirements

- **FR-001:** `reconcile_fact` MUST require an authorized actor, the target Fact,
  `expected_version` of the target, and an explicit outcome.
- **FR-002:** Outcome MUST be one of `confirm`, `revise`, `update_dependencies`,
  `attach_evidence`, `retract`, or `preserve_uncertainty`.
- **FR-003:** Reconciliation MUST record the checked prerequisite Fact ids and
  versions, and the evidence ids considered.
- **FR-004:** A warning MUST be cleared only when the recorded prerequisite versions
  match the current versions at commit time and all required assumptions hold.
- **FR-005:** If any checked prerequisite or evidence changed between review and
  commit, GUM MUST reject with `STALE_RECONCILIATION` and require a re-check.
- **FR-006:** `confirm` MUST update the affected dependency edge assumed versions
  to the verified versions and store the verifying actor.
- **FR-007:** `revise` MUST delegate to the revision service with optimistic
  concurrency; it MUST NOT bypass revision history.
- **FR-008:** `update_dependencies` MUST enforce all graph invariants from 003 in
  the same transaction.
- **FR-009:** `retract` MUST create a retraction revision, set status `retracted`,
  and trigger impact analysis for reverse dependents.
- **FR-010:** `preserve_uncertainty` MUST keep the warning open or unresolved and
  MUST record the rationale; it MUST NOT silently suppress future warnings.
- **FR-011:** Reconciliation MUST be atomic: outcome record, health update, edge
  updates, revision, evidence links, and audit event commit together or not at all.
- **FR-012:** Reconciliation MUST NOT mark a Fact as true; it records a review
  outcome against stated assumptions.
- **FR-013:** A warning can reopen if a reviewed assumption changes again.
- **FR-014:** Reconciliation outcomes MUST be queryable and exportable.
- **FR-015:** Unauthorized or out-of-namespace actors MUST be rejected before any
  write.

## 5. Key Entities

### Reconciliation Record

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique record id |
| `fact_id` | string | Reconciled Fact |
| `outcome` | enum | See FR-002 |
| `actor` | string | Authorized reviewer |
| `expected_version` | integer | Target version checked |
| `checked_prerequisites` | object[] | Fact id + version pairs |
| `checked_evidence` | string[] | Evidence ids considered |
| `note` | string \| null | Rationale |
| `stale` | boolean | Whether commit-time assumptions changed |
| `created_at` | timestamp | Completion time |

### Health Transition

`previous_health`, `new_health`, `reason`, and matching warning id.

## 6. Edge Cases & Error Handling

- Target Fact missing or unauthorized → `NOT_FOUND` / `UNAUTHORIZED_SCOPE`.
- Reconciliation commit after prerequisite change → `STALE_RECONCILIATION`; no
  partial clear.
- `confirm` with no prerequisite snapshot → validation error; blind clearing is
  forbidden.
- `revise` with stale target version → `STALE_WRITE`.
- Dependency update introduces a cycle → `CYCLE_DETECTED`; no changes applied.
- Retracting an already retracted Fact → idempotent or explicit no-op event.
- Evidence becomes unavailable during review → stale detection; re-check required.
- Alternative support group where one support remains valid → confirm may clear
  based on the remaining valid support, with reason recorded.
- Two concurrent reconciliations of the same Fact → version check admits at most one.

## 7. Non-Functional Requirements

- Reconciliation writes MUST be short, atomic transactions.
- Review payloads MUST be concise enough for agent context budgets.
- All outcomes and assumption snapshots MUST be auditable.
- Deterministic confirmation MUST NOT require an LLM.

## 8. Success Criteria

- No warning can be cleared without recorded prerequisite versions and evidence.
- A prerequisite changing mid-review always produces a stale rejection.
- Revising through reconciliation preserves all prior revisions.
- Retraction propagates to reverse dependents.
- Every reconciliation has an actor, timestamp, and outcome traceable in audit.

## 9. Open Questions

- Should `confirm` require evidence, or is prerequisite version review sufficient?
  Initial answer: version review plus at least a rationale; evidence is optional
  but strongly recommended.
- Should bulk reconciliation of many dependents be supported? Future work; initial
  release is per-Fact.
