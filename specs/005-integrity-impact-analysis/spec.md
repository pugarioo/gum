# Feature Specification: Integrity & Impact Analysis

- **Feature ID:** 005
- **Module:** Integrity Engine
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §5.1–§5.3, §5.5, §9 (Integrity and Reconciliation)
- **Depends on:** 001, 002, 003, 004, 010, 016
- **Blocks:** 006, 007, 009, 012
- **Blocked by:** none

## 1. Overview

When a Fact changes, GUM must find which other Facts relied on it, decide whether
their support assumptions still hold, and flag them for review when they do not.
This is the signature capability of GUM: **dependency-aware memory integrity**.

Change triggers reassessment, not automatic rewriting. GUM may mark a dependent
`possibly_invalid`; it never asserts that the dependent statement is false.
Dependency health measures known dependency integrity, not factual truth.

## 2. Scope

### In scope

- Substantive-versus-metadata change classification
- Reverse-dependency traversal (direct and transitive)
- Support-group evaluation (required and alternative)
- Assumed-version comparison
- Dependency-health transitions (`healthy`, `possibly_invalid`, `unresolved`)
- `analyze_impact`, `validate_dependencies`, `get_affected_facts`,
  `get_integrity_report`
- Durable deferred-work tracking and crash recovery

### Out of scope

- Human/agent reconciliation decisions (006)
- Semantic contradiction detection (009)
- Automated statement rewriting
- Graph edge mutation (003)

## 3. User Scenarios & Testing

### P1 — Cascading impact from a prerequisite change

**Given** `F002` depends on `F001`, `F003` depends on `F002`, `F004` depends on `F003`
**When** `F001` receives a substantive update from version 1 to 2
**Then** GUM marks `F002`, `F003`, and `F004` `possibly_invalid`
**And** records the path and the prerequisite version that triggered each warning
**And** does not modify their statements.

### P1 — Metadata-only change does not cascade

**Given** a Fact update that changes only non-substantive metadata
**When** the update commits
**Then** dependent Facts remain `healthy`
**And** no impact warnings are created.

### P1 — Analyze impact without writing

**Given** a proposed change to a prerequisite
**When** `analyze_impact` is called in dry-run mode
**Then** GUM returns affected Fact ids, paths, depths, support-group reasons, and
the current health of each affected Fact
**And** no health state changes are persisted.

### P2 — Required versus alternative support groups

**Given** dependent `F010` requires both `P1` and `P2`
**When** only `P1` changes substantively
**Then** `F010` becomes `possibly_invalid` because a required support changed.
**Given** dependent `F011` is supported by either `P3` or `P4`
**When** only `P3` changes
**Then** `F011` remains `healthy` while `P4` still satisfies alternative support.

### P2 — Version assumption is stale

**Given** an edge created against prerequisite version 2
**When** the prerequisite is at version 5
**Then** `validate_dependencies` reports the edge as stale
**And** the dependent is flagged unless reconciliation has already re-validated it.

### P2 — Evidence change triggers reassessment

**Given** a Fact supported by evidence that is marked unavailable
**When** the evidence-change event is processed
**Then** GUM evaluates affected Facts and flags them with an evidence reason
**And** does not retract them.

### P3 — Integrity report

**Given** a populated namespace
**When** `get_integrity_report` is called
**Then** GUM returns counts by health state, stale assumptions, unresolved
conflicts, support-group inconsistencies, and pending deferred checks.

### P4 — Crash during propagation

**Given** a change transaction commits and deferred propagation is queued
**When** the process crashes before propagation completes
**Then** restart resumes the durable deferred work
**And** no affected Fact is silently left unmarked.

## 4. Functional Requirements

- **FR-001:** GUM MUST classify each Fact update as `substantive` or `metadata`
  using deterministic rules; statement, details, dependency structure, and evidence
  applicability changes are substantive by default.
- **FR-002:** Substantive changes MUST trigger reverse-dependency impact analysis;
  metadata-only changes MUST NOT invalidate unrelated knowledge.
- **FR-003:** Traversal MUST start from the changed Fact, follow reverse edges, and
  support a configurable maximum depth with truncation reporting.
- **FR-004:** Traversal MUST be cycle-safe even if corrupt graph data contains a cycle.
- **FR-005:** For each edge, GUM MUST compare the assumed prerequisite state with
  the current state and record the reason for any warning.
- **FR-006:** Required support groups MUST flag the dependent if any member
  prerequisite changes substantively.
- **FR-007:** Alternative support groups MUST flag the dependent only when no
  independently sufficient support remains valid.
- **FR-008:** GUM MUST set `dependency_health = "possibly_invalid"` for flagged
  Facts and MUST NEVER rewrite Fact statements automatically.
- **FR-009:** `dependency_health = "unresolved"` MUST be used for insufficient,
  conflicting, or structurally broken support that requires investigation.
- **FR-010:** `analyze_impact` MUST support dry-run and must not mutate state.
- **FR-011:** `validate_dependencies` MUST detect orphan edges, stale assumptions,
  cycles, missing Facts, and support-group inconsistencies.
- **FR-012:** `get_affected_facts` MUST return direct and transitive affected Facts
  with path, depth, reason, and timestamp.
- **FR-013:** `get_integrity_report` MUST aggregate warnings without exposing
  unbounded raw logs.
- **FR-014:** Health updates, dependency-state updates, and audit events MUST commit
  atomically where they share a SQLite transaction.
- **FR-015:** Deferred impact work MUST be recorded durably in `memory_operations`
  or an equivalent queue with idempotent processing.
- **FR-016:** Repeated analysis MUST be idempotent and MUST NOT create duplicate
  warnings for the same (dependent, cause, version) tuple.
- **FR-017:** A reconciliation (006) or subsequent substantive change MAY clear or
  replace a warning; clearing MUST record why it was cleared.
- **FR-018:** The engine MUST NOT require an LLM for deterministic graph traversal
  or version comparison.

## 5. Key Entities

### Impact Analysis Result

| Field | Type | Notes |
|---|---|---|
| `root_fact_id` | string | Changed prerequisite |
| `affected_fact_id` | string | Flagged dependent |
| `path` | string[] | Traversal path |
| `depth` | integer | Distance from root |
| `reason` | string | Version, support-group, or evidence cause |
| `previous_health` | enum | Health before analysis |
| `new_health` | enum | Health after analysis |
| `created_at` | timestamp | Analysis time |

### Integrity Warning

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique warning identifier |
| `fact_id` | string | Dependent Fact |
| `cause_type` | enum | `version_change`, `support_change`, `evidence_change`, `structural` |
| `cause_fact_id` | string \| null | Prerequisite Fact |
| `cause_version` | integer \| null | Changed version |
| `state` | enum | `open`, `resolved`, `preserved` |
| `created_at` | timestamp | Creation time |
| `resolved_at` | timestamp \| null | Resolution time |

### Deferred Operation

Durable record of pending propagation, reindex, or reassessment work with status,
attempt count, and idempotency key.

## 6. Edge Cases & Error Handling

- Change to a Fact with no dependents → analysis completes without warnings.
- Deep chain or wide fan-out → bounded by depth/result budget with explicit
  truncation; truncated work MUST be enqueued, never dropped.
- Graph corruption (missing target Fact) → `unresolved` with structural reason
  and integrity report entry.
- Concurrent updates to the same prerequisite → each substantive change produces a
  distinct cause version; analysis must handle the latest committed state.
- Warning cleared then prerequisite changes again → warning reopens with new cause.
- Reconciliation in progress while analysis runs → version snapshot comparison
  determines whether the reconciliation is stale.
- Alternative support where one support is invalidated but another is `possibly_invalid`
  → dependent becomes `unresolved`, not silently healthy.

## 7. Non-Functional Requirements

- Impact analysis MUST use indexed reverse-edge queries and avoid full-graph scans.
- Writes MUST be short transactions; heavy traversal may be staged into durable
  operations processed by workers.
- Analysis of 10,000 dependents MUST complete with bounded memory and report progress.
- Deterministic structural checks MUST run without model inference.
- Health status MUST always be surfaced in retrieval results when material (007).

## 8. Success Criteria

- The four-Fact cascade example (`F001` → `F004`) flags exactly the affected chain.
- Metadata-only updates produce zero new `possibly_invalid` warnings.
- 100 repeated analyses of the same change produce one warning per cause tuple.
- Crash after commit but before propagation results in zero missed affected Facts
  after restart.
- `analyze_impact` dry-run leaves the database byte-for-byte unchanged.
- Alternative support prevents false-positive invalidation when another valid
  support remains.

## 9. Open Questions

- Should health transitions generate a revision of the dependent Fact? Initial
  answer: no; they generate integrity events only.
- What exact change classification rules apply to `category` and `confidence`?
  Initial answer: metadata, unless namespace policy declares them substantive.
