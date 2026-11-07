# Feature Specification: Dependency Graph

- **Feature ID:** 003
- **Module:** Core Memory
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §3.3, §4, §9 (Dependency Graph)
- **Depends on:** 001, 002, 010, 016
- **Blocks:** 005, 006, 007, 009

## 1. Overview

Facts are graph nodes; dependencies are directed edges. A dependency states that
a dependent Fact relies on a prerequisite Fact. This module owns edge creation,
removal, traversal, explanation, cycle prevention, support groups, and graph
inspection. It is the basis for change-impact analysis and graph-aware retrieval.

A dependency change is not a statement revision. It is its own auditable event.

## 2. Scope

### In scope

- `add_dependency`, `remove_dependency`
- `get_dependencies`, `get_dependents`, `explain_dependency`, `inspect_graph`
- Direct and reverse lookup, multi-hop traversal, depth bounds
- Cycle detection and prevention
- Dependency types: initially `DEPENDS_ON`
- Required and alternative support groups
- Assumed prerequisite version / canonical fingerprint
- Proposed versus confirmed dependency state

### Out of scope

- Integrity state propagation (005)
- Reconciliation outcomes (006)
- Automated semantic dependency discovery (009; may only propose)
- Cross-store or cross-machine graph synchronization (not in scope)

## 3. User Scenarios & Testing

### P1 — Add a dependency

**Given** Facts `F001` and `F002` in the same authorized namespace
**When** `add_dependency` is called with `from_fact = F002`, `to_fact = F001`,
`type = "DEPENDS_ON"`, a reason, and the current version of `F001`
**Then** GUM persists one directed edge `F002 -> F001`
**And** records author, timestamps, assumed version, and validation state
**And** writes an audit event.

### P1 — Prevent cycles

**Given** `F002 -> F001` and `F003 -> F002`
**When** adding `F001 -> F003`
**Then** GUM rejects the write with `CYCLE_DETECTED`
**And** the graph is unchanged.

### P1 — Reverse lookup finds dependents

**Given** a chain `F004 -> F003 -> F002 -> F001`
**When** `get_dependents` is called for `F001` with `depth = 2`
**Then** GUM returns `F002` and `F003` with path metadata
**And** does not return `F004` at depth 2.

### P2 — Remove a dependency

**Given** an existing edge
**When** `remove_dependency` is called
**Then** the edge is removed, support-group membership is updated, and an audit
event is written
**And** affected dependents are queued for integrity re-evaluation (005).

### P2 — Explain a dependency

**Given** `F002 -> F001`
**When** `explain_dependency` is called
**Then** GUM returns the reason, author, creation time, assumed prerequisite
version, current prerequisite version, validation state, and whether the
assumption is stale.

### P3 — Required support group

**Given** a dependent `F010` that requires both `F001` and `F002`
**When** the group is created with `mode = "required"`
**Then** both edges are members of the group
**And** 005 treats a substantive change to either prerequisite as relevant.

### P3 — Alternative support group

**Given** a dependent `F011` supported by either `F003` or `F004`
**When** the group is created with `mode = "alternative"`
**Then** 005 flags `F011` only when no independently sufficient support remains.

### P4 — Inspect graph health

**Given** a populated namespace
**When** `inspect_graph` is called
**Then** GUM returns node count, edge count, max depth, orphan edges, stale
assumptions, and support-group inconsistencies.

## 4. Functional Requirements

- **FR-001:** Every dependency MUST connect two existing, distinct Facts and MUST
  be stored as a separate edge, never inside Fact `details`.
- **FR-002:** Self-dependencies MUST be rejected.
- **FR-003:** Cross-namespace dependencies MUST be rejected by default; any future
  opt-in requires explicit policy from 017.
- **FR-004:** Duplicate active edges with the same `from_fact`, `to_fact`, and
  `type` MUST be rejected or return the existing edge idempotently.
- **FR-005:** Edge insertion MUST run cycle detection over the reverse path before
  commit and MUST reject any cycle.
- **FR-006:** Each edge MUST record `from_fact`, `to_fact`, `type`, `reason`,
  `assumed_version` (or canonical fingerprint), `validation_state`, `created_by`,
  and `created_at`.
- **FR-007:** `validation_state` MUST distinguish at least `proposed`, `confirmed`,
  and `invalidated`; confirmed dependencies are not automatically true facts.
- **FR-008:** `get_dependencies` and `get_dependents` MUST support direct lookup,
  bounded transitive traversal, and cycle-safe visited tracking.
- **FR-009:** Traversal MUST enforce a configurable maximum depth and return a
  truncation indicator when the bound is hit.
- **FR-010:** `explain_dependency` MUST report why the edge exists and whether the
  assumed prerequisite version differs from the current version.
- **FR-011:** `inspect_graph` MUST be read-only and MUST report graph statistics and
  consistency findings without mutating data.
- **FR-012:** Support groups MUST have a unique id, a mode of `required` or
  `alternative`, and zero or more member edges; an edge MAY belong to at most one
  group in the initial design.
- **FR-013:** Deleting a member edge MUST remove its support-group membership in the
  same transaction.
- **FR-014:** Removing an edge MUST trigger integrity re-evaluation for its direct
  dependents and MUST write an audit event.
- **FR-015:** Graph writes MUST be atomic with their audit events and MUST use
  indexed reverse-edge lookups, not full-graph scans in TypeScript.

## 5. Key Entities

### Dependency Edge

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique edge identifier |
| `from_fact` | string | Dependent Fact |
| `to_fact` | string | Prerequisite Fact |
| `type` | string | Initially `DEPENDS_ON` |
| `reason` | string | Why the dependency exists |
| `assumed_version` | integer | Prerequisite version relied upon |
| `assumed_fingerprint` | string | Optional canonical content fingerprint |
| `support_group_id` | string \| null | Optional group membership |
| `validation_state` | enum | `proposed`, `confirmed`, `invalidated` |
| `created_by` | string | Actor identity |
| `created_at` | timestamp | Creation time |

### Dependency Support Group

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique group identifier |
| `fact_id` | string | Dependent Fact |
| `mode` | enum | `required` or `alternative` |
| `created_at` | timestamp | Creation time |

## 6. Edge Cases & Error Handling

- Missing source or target Fact → `NOT_FOUND`; no partial edge.
- Edge to an archived or retracted Fact → allowed only with an explicit policy
  flag; default is rejection for retracted prerequisites.
- Cycle attempt through a long path → detected before insert; no orphan writes.
- Traversal encountering corrupt cycle already in data → return a graph-integrity
  warning and stop expanding that path.
- Massive fan-out → bounded results, token budget, and truncation flag.
- Confirmed edge whose prerequisite version advances → remains an edge; 005 marks
  dependent `possibly_invalid`, not remove the edge.
- Alternative group with all members invalid → dependent becomes `unresolved` or
  `possibly_invalid` per 005 rules.
- Edge removal while impact analysis runs → transaction isolation and version
  checks must prevent inconsistent reads.

## 7. Non-Functional Requirements

- Reverse-edge lookups MUST be indexed on `(to_fact, type)`.
- Traversal MUST avoid loading the whole graph into memory.
- Cycle detection MUST be correct under concurrent edge insertion; use a write
  transaction and re-check before commit.
- Graph operations MUST NOT depend on an LLM.
- Optional adjacency caches MUST be rebuildable from SQLite and invalidated on write.

## 8. Success Criteria

- Inserting an edge that closes any cycle is rejected 100% of the time.
- A 10,000-edge acyclic graph remains queryable with bounded depth without stack
  overflow or full-graph scans.
- `get_dependents(F, depth = 1)` returns exactly the direct reverse edges.
- Support-group semantics are preserved across export/import (011).
- Removing an edge never leaves a dangling support-group membership.

## 9. Open Questions

- Should multiple dependency types beyond `DEPENDS_ON` exist (e.g., `SUPPORTS`,
  `CONTRADICTS`)? Initial answer: only `DEPENDS_ON`; others are future work.
- Should failure of one required group member mark `possibly_invalid` immediately,
  or wait for reconciliation? Initial answer: mark immediately, reconcile later.
