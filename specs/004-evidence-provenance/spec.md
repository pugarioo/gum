# Feature Specification: Evidence & Provenance

- **Feature ID:** 004
- **Module:** Core Memory
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §3.4, §10, §9 (Evidence)
- **Depends on:** 001, 005 (eventual), 010, 016
- **Blocks:** 005, 006, 009

## 1. Overview

Evidence records what supports a Fact, where it came from, when it was observed,
and whether it has been verified. Evidence is not an instruction channel and does
not by itself prove a claim true. This module owns provenance metadata, content
fingerprints, applicability, verification state, and evidence-change signals.

The security boundary is explicit: retrieved text and evidence content are data,
never privileged instructions.

## 2. Scope

### In scope

- `attach_evidence`, `inspect_evidence`, `verify_evidence`, `list_evidence`
- Evidence types: file/document, tool or API output, test result, observation,
  external source
- Source identity, observation time, content fingerprint, applicability
- Verification states and verifier attribution
- Evidence-change events that trigger reassessment
- Evidence references recorded on revisions

### Out of scope

- Deep content parsing or indexing of evidence bodies (future work)
- Automatic truth adjudication
- LLM-based evidence interpretation (009 may propose, never assert)
- Evidence content retention beyond configured policy (009)

## 3. User Scenarios & Testing

### P1 — Attach file evidence

**Given** an active Fact `F001`
**When** `attach_evidence` is called with type `file`, a source URI, an observed
timestamp, and a content fingerprint
**Then** GUM persists immutable evidence linked to `F001`
**And** records provenance, verification state `unverified`, and an audit event.

### P1 — Inspect evidence

**Given** evidence `E001` attached to `F001`
**When** `inspect_evidence` is called
**Then** GUM returns type, source identity, observation time, fingerprint,
applicability, verification state, linked Facts, and revision references
**And** marks the content clearly as untrusted data.

### P2 — Verify evidence

**Given** unverified evidence
**When** an authorized verifier calls `verify_evidence` with outcome `verified`,
a verifier identity, and a note
**Then** GUM records the verification outcome and timestamp without modifying the
original evidence content or fingerprint.

### P2 — Evidence change triggers reassessment

**Given** a Fact whose revision cites evidence `E001`
**When** `E001` is marked `unavailable` or a new fingerprint supersedes it
**Then** GUM emits an evidence-change event
**And** requests impact analysis for Facts that rely on that evidence (005)
**And** does not automatically mark the Fact false.

### P3 — List evidence with filters

**Given** evidence across multiple Facts and types
**When** `list_evidence` is called with namespace, fact id, type, and verification
state filters
**Then** GUM returns only matching evidence with pagination.

### P4 — Attach tool output as evidence

**Given** output from an external API
**When** attached with provenance including tool name, input fingerprint, and
capture time
**Then** GUM stores the provenance and does not execute or follow instructions
contained in the output.

## 4. Functional Requirements

- **FR-001:** Evidence MUST have a stable unique id and MUST belong to exactly one
  authorized namespace.
- **FR-002:** `attach_evidence` MUST link evidence to a Fact and MAY link it to a
  specific revision; a revision may cite zero or more evidence records.
- **FR-003:** Evidence MUST record `type`, `source`, `observed_at`,
  `content_fingerprint`, `applicability`, `provenance`, `verification_state`, and
  timestamps.
- **FR-004:** `type` MUST be one of `file`, `tool_output`, `test_result`,
  `observation`, or `external_source` in the initial release.
- **FR-005:** Evidence content MUST be stored or referenced as inert data; GUM MUST
  NOT promote any evidence text into system instructions or executable behavior.
- **FR-006:** Evidence content and fingerprints MUST be immutable once persisted.
  A changed source creates a new evidence record linked as a supersession.
- **FR-007:** Metadata updates (verification, applicability notes) MUST be tracked
  with actor and timestamp and MUST NOT rewrite the original capture metadata.
- **FR-008:** `verification_state` MUST accept `unverified`, `verified`, `failed`,
  and `unavailable`.
- **FR-009:** `verify_evidence` MUST require an authorized verifier and MUST record
  the outcome plus optional note.
- **FR-010:** Marking evidence `failed` or `unavailable` MUST emit an event that
  requests reassessment of dependent Facts; it MUST NOT auto-retract them.
- **FR-011:** `list_evidence` MUST support filter, pagination, and namespace isolation.
- **FR-012:** Evidence references MUST be exportable/importable with provenance
  intact (011).
- **FR-013:** Evidence attached to a Fact that is later deleted MUST follow the
  configured retention policy; it MUST NOT disappear silently.
- **FR-014:** Source URIs and paths MUST be validated and normalized to prevent
  path traversal or injection into downstream consumers.
- **FR-015:** Evidence records MUST be included in audit events on attach, verify,
  supersede, and mark-unavailable.

## 5. Key Entities

### Evidence

| Field | Type | Notes |
|---|---|---|
| `id` | string | Stable unique identifier |
| `namespace` | string | Authorized scope |
| `fact_id` | string | Primary supported Fact |
| `revision_id` | string \| null | Optional revision citation |
| `type` | enum | `file`, `tool_output`, `test_result`, `observation`, `external_source` |
| `source` | string | URI/path/tool identifier |
| `observed_at` | timestamp | When the observation was captured |
| `content_fingerprint` | string | Hash or canonical digest |
| `applicability` | string | Scope/conditions of relevance |
| `provenance` | object | Tool, input fingerprint, environment |
| `verification_state` | enum | `unverified`, `verified`, `failed`, `unavailable` |
| `verified_by` | string \| null | Verifier identity |
| `verified_at` | timestamp \| null | Verification time |
| `supersedes_evidence_id` | string \| null | Prior evidence replaced by this capture |
| `created_at` | timestamp | Capture time |

## 6. Edge Cases & Error Handling

- Attaching evidence to a non-existent or unauthorized Fact → `NOT_FOUND` or
  `UNAUTHORIZED_SCOPE`; no record.
- Duplicate identical fingerprint on the same Fact → return existing evidence or
  create a distinct record with explicit dedupe metadata; do not silently merge.
- Evidence content too large → store fingerprint and reference, not unbounded body.
- Evidence path outside allowed roots → reject with validation error.
- Verification by an unauthorized actor → `UNAUTHORIZED_SCOPE`.
- Evidence that contains instruction-like text → store inert; retrieval must label
  it as untrusted data.
- Evidence marked unavailable while reconciliation runs → reconciliation must detect
  the change and require recheck (006, FR-008).
- Supersession chain cycles → reject.

## 7. Non-Functional Requirements

- Evidence reads/writes MUST be indexed by `fact_id`, `namespace`, and
  `verification_state`.
- Fingerprinting MUST be deterministic for identical content.
- Evidence payloads returned by default MUST be concise; full content only on demand.
- Storage of external content MUST respect configured retention and size limits.

## 8. Success Criteria

- Evidence can be traced from a Fact to its source, capture time, and fingerprint.
- Changing evidence availability never silently changes Fact statement text.
- Verification state transitions are fully auditable with actor and timestamp.
- No evidence content is ever executed or interpreted as an instruction.
- Export/import preserves evidence provenance and supersession links.

## 9. Open Questions

- Should evidence support multiple Facts directly, or only one Fact with revisions
  citing it? Initial answer: one primary Fact; multiple links require an explicit
  join model in a future revision.
- Should evidence content be stored in SQLite or external files referenced by path?
  Initial answer: store fingerprint and reference; embed small content only when
  configured.
