# Feature Specification: Memory Lifecycle & Maintenance

- **Feature ID:** 009
- **Module:** Lifecycle
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §7, §11, §10, §9 (Administration)
- **Depends on:** 002, 003, 004, 005, 006, 010, 016
- **Blocks:** none
- **Blocked by:** none

## 1. Overview

GUM supports the full memory lifecycle: ingestion, classification, deduplication,
dependency suggestion, persistence, revision, integrity checks, reconciliation,
consolidation, archiving, expiration, deletion, and scheduled or triggered
maintenance.

Maintenance may be deterministic or model-assisted. Deterministic structural checks
must run without an LLM. Model-assisted operations may propose, but never assert
infallible truth or silently rewrite memory.

## 2. Scope

### In scope

- Ingestion and classification pipeline
- Duplicate and near-duplicate detection
- Consolidation preserving meaning, evidence, and graph links
- Proposed dependency suggestion and acceptance
- Scheduled and event-triggered maintenance
- Structural checks, contradiction reports, missing-link detection
- Retention policies: archive, expire, purge
- Durable, resumable maintenance operations

### Out of scope

- Automatic contradiction resolution without review
- Automatic dependency confirmation
- Cross-store synchronization
- Developer dashboard or UI for maintenance

## 3. User Scenarios & Testing

### P1 — Ingest and classify a Fact

**Given** raw input with a statement, namespace, and category candidate
**When** ingestion runs
**Then** GUM validates and normalizes the input, classifies it, checks for likely
duplicates, and persists it through Fact Management
**And** records the operation as durable and idempotent.

### P1 — Detect duplicates without auto-merge

**Given** two active Facts with near-identical statements in one namespace
**When** maintenance runs duplicate detection
**Then** GUM reports a duplicate candidate pair with confidence and rationale
**And** does not merge them without explicit authorization.

### P2 — Consolidate duplicates

**Given** an authorized merge decision for two duplicate Facts
**When** consolidation executes
**Then** GUM preserves both revision histories, combines evidence references,
rewires dependencies to the surviving Fact, records the merge event, and archives
or retracts the absorbed Fact
**And** the transaction is atomic.

### P2 — Proposed dependency suggestion

**Given** a model or heuristic suggests `F002` depends on `F001`
**When** the suggestion is stored
**Then** it is marked `proposed`, not confirmed
**And** it does not alter dependent health until accepted by an authorized actor.

### P2 — Structural maintenance without LLM

**Given** a populated store
**When** deterministic maintenance runs
**Then** GUM detects orphan edges, stale assumptions, malformed support groups,
missing evidence references, and expired items
**And** performs only policy-authorized repairs, logging every action.

### P3 — Contradiction detection

**Given** model-assisted contradiction detection is configured
**When** two Facts appear contradictory
**Then** GUM reports a contradiction candidate with evidence and uncertainty
**And** does not retract or rewrite either Fact automatically.

### P3 — Retention dry-run

**Given** a retention policy `archive facts inactive for 180 days`
**When** maintenance runs in dry-run mode
**Then** GUM reports exactly which Facts would be archived
**And** performs no writes.

### P4 — Scheduled maintenance

**Given** a configured interval
**When** the schedule fires
**Then** GUM enqueues maintenance jobs durably, runs them with concurrency limits,
and records outcomes.

## 4. Functional Requirements

- **FR-001:** Ingestion MUST validate, normalize, classify, deduplicate-check, and
  persist through the same domain services used by other interfaces.
- **FR-002:** Ingestion MUST be idempotent and MUST record an operation id to prevent
  duplicate processing.
- **FR-003:** Deduplication MUST produce candidates with confidence and rationale;
  it MUST NOT auto-merge by default.
- **FR-004:** Consolidation MUST preserve both Facts' revisions, evidence, and
  dependency links by re-linking them to the surviving Fact and recording a merge
  record.
- **FR-005:** Proposed dependencies MUST be distinguishable from confirmed/validated
  dependencies and MUST NOT trigger health penalties before acceptance.
- **FR-006:** Deterministic maintenance MUST run without an LLM and MUST include
  orphan-edge, cycle, stale-version, support-group, evidence-reference, and
  retention-expiry checks.
- **FR-007:** Model-assisted maintenance MUST be disabled by default, configured
  explicitly, and MUST output uncertainty and evidence rather than assertions.
- **FR-008:** Contradiction detection MUST NOT auto-resolve; it creates a review
  item or proposed operation.
- **FR-009:** Retention policies MUST be defined per namespace and support
  `archive`, `expire`, and `purge` actions with optional age, status, and last-use
  criteria.
- **FR-010:** Retention execution MUST support dry-run and report affected records.
- **FR-011:** Archival MUST preserve revisions, dependencies, evidence, and audit
  history unless a later purge policy explicitly removes them.
- **FR-012:** Purge MUST define dependency treatment (reject, detach, cascade) and
  MUST preserve tombstone audit events.
- **FR-013:** Expiration MUST follow the same dependency and audit rules as purge.
- **FR-014:** Maintenance jobs MUST be durable, resumable, idempotent, and
  concurrency-limited.
- **FR-015:** Interrupted maintenance MUST NOT leave partially consolidated Facts or
  half-rewired graphs; use atomic transactions per unit of work.
- **FR-016:** All maintenance actions MUST be auditable with actor, trigger,
  affected records, and outcome.
- **FR-017:** Missing supporting links SHOULD be reported as suggestions, never
  auto-attached.
- **FR-018:** Maintenance MUST respect namespace isolation and authorization.

## 5. Key Entities

### Memory Operation

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique operation id |
| `type` | enum | `ingest`, `dedupe`, `consolidate`, `retention`, `maintenance` |
| `status` | enum | `pending`, `running`, `done`, `failed`, `cancelled` |
| `idempotency_key` | string | Deduplication key |
| `progress` | object | Counts and cursor |
| `attempts` | integer | Retry count |
| `created_at`, `updated_at` | timestamps | Lifecycle |

### Consolidation Record

`surviving_fact_id`, `absorbed_fact_ids`, `revision_map`, `evidence_map`,
`dependency_rewrites`, `actor`, `created_at`.

### Retention Policy

`namespace`, `action` (`archive`, `expire`, `purge`), `criteria` (age, status,
last use), `dry_run`, `schedule`, `created_by`.

## 6. Edge Cases & Error Handling

- Consolidation where both Facts have conflicting revisions → preserve both chains
  and link them; never discard history.
- Consolidation introducing a cycle through rewired dependencies → abort and report.
- Purge of a Fact with dependents → require strategy; default reject.
- Deduplication false positive → merge requires authorization; reversible via audit.
- Model unavailable for semantic maintenance → deterministic checks still run.
- Maintenance overlapping with agent writes → optimistic versions and short
  transactions avoid clobbering.
- Retention policy matching everything → dry-run required before destructive action.
- Expired evidence referenced by active Facts → flag Facts, do not silently drop
  provenance.

## 7. Non-Functional Requirements

- Maintenance MUST run incrementally and be resumable after crash.
- Background work MUST not block foreground reads/writes beyond configured limits.
- Large consolidation batches MUST stream records and bound memory.
- Model-assisted jobs MUST be optional and isolated from the deterministic path.
- All operations MUST be observable through counts, durations, and failures.

## 8. Success Criteria

- Ingesting the same input twice produces one Fact and one completed operation.
- Consolidation preserves 100% of revisions, evidence references, and dependency
  links, with no dangling edges.
- No model-assisted suggestion changes Fact content without authorization.
- Interrupted maintenance restarts without duplicating merges or rewrites.
- Retention dry-run makes zero writes and matches actual execution set exactly.
- Deterministic checks complete with no model configured.

## 9. Open Questions

- What similarity threshold defines a duplicate candidate? Tune by benchmark;
  candidates are always advisory.
- Should consolidation survive as a redirect from absorbed Fact id to survivor?
  Initial answer: yes, store a merge redirect for external references.
- What is the default retention schedule? No destructive default; operators opt in.
