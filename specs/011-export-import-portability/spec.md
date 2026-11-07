# Feature Specification: Export & Import Portability

- **Feature ID:** 011
- **Module:** Portability
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §13, §10, §9 (Administration and Portability)
- **Depends on:** 001-005, 010, 016
- **Blocks:** 012, 013, 014, 015

## 1. Overview

GUM is Git-independent and portable. A named memory store can be exported with
Facts, revisions, dependencies, support groups, and evidence, then imported into
another GUM instance with integrity checks. Cross-machine live synchronization is
explicitly out of scope; portability means snapshot transfer with validation, not
implicit replication.

## 2. Scope

### In scope

- `export_memory` and `import_memory`
- Versioned bundle format with manifest and checksums
- Namespace-scoped or full-store export
- Facts, revisions, dependencies, support groups, evidence, metadata
- Optional inclusion of embeddings, events, and retention policies
- Import validation, conflict policy, and atomic staging
- Namespace remapping
- Backup-oriented export

### Out of scope

- Live cross-machine synchronization
- Identity merging between two divergent live stores
- Binary database file copying as the only backup method (SQLite backup is 010)
- Importing untrusted executable content

## 3. User Scenarios & Testing

### P1 — Round-trip export and import

**Given** a namespace with Facts, revisions, dependencies, support groups, and evidence
**When** `export_memory` writes a bundle and `import_memory` loads it into an empty
GUM instance
**Then** all entities are restored with identical ids, version chains, graph edges,
evidence provenance, and timestamps
**And** integrity checks pass.

### P1 — Corrupt bundle rejected

**Given** a bundle with a checksum mismatch or broken revision chain
**When** import is attempted
**Then** GUM rejects the entire import before writing domain data
**And** reports the specific validation failures.

### P1 — Atomic import

**Given** a valid bundle with many entities
**When** any insert fails mid-import
**Then** the import transaction rolls back completely
**And** the target store is unchanged.

### P2 — Conflict policy

**Given** a target store that already contains a Fact with the same id
**When** import uses `on_conflict = "reject"` (default)
**Then** the import is rejected with the conflicting ids
**When** import uses `on_conflict = "overwrite"`
**Then** GUM replaces the existing Fact only after validating the incoming chain
**And** records an audit event describing the replacement.

### P2 — Namespace remap

**Given** a bundle exported from `project:shopflow`
**When** import maps it to `project:shopflow-copy`
**Then** all internal identifiers remain stable and namespace references are
remapped consistently.

### P3 — Selective export

**Given** a large store
**When** export is limited to one namespace and a date range
**Then** only matching Facts and their required linked records are included
**And** the manifest declares the filter applied.

### P3 — Embeddings optional

**Given** an export without embeddings
**When** imported into a store with a different embedding model
**Then** Facts and graph data remain intact and semantic index jobs are enqueued
according to policy.

### P4 — Audit and retention portability

**Given** export includes audit events and retention policies
**When** imported
**Then** history is preserved as evidence and retention rules are applied explicitly,
not silently activated.

## 4. Functional Requirements

- **FR-001:** Export MUST produce a versioned, self-describing bundle with a
  manifest containing format version, GUM version, export time, filters, entity
  counts, and checksums.
- **FR-002:** Export MUST include Facts, revisions, dependencies, support groups,
  and evidence required to reconstruct the memory within the selected scope.
- **FR-003:** Revisions MUST preserve `previous_revision_id` chains and version
  numbers exactly.
- **FR-004:** Selective export MUST include all linked records needed to keep
  referential integrity; it MUST NOT emit dangling dependencies or evidence.
- **FR-005:** Import MUST validate bundle format version, manifest checksums, schema
  compatibility, enum values, referential integrity, revision chain continuity,
  and graph acyclicity before writing.
- **FR-006:** Import MUST run in a transaction or validated staging area so a
  failure leaves no partial domain data.
- **FR-007:** Import MUST support conflict policy `reject` (default) and `overwrite`;
  silent merge is forbidden.
- **FR-008:** `overwrite` MUST validate that incoming revisions form a valid chain
  and MUST record the replacement in the audit trail.
- **FR-009:** Import MUST support optional namespace remapping and MUST update all
  references consistently.
- **FR-010:** Identifiers MUST be preserved by default; if remapping ids is needed,
  it must be an explicit future operation, never silent.
- **FR-011:** Import MUST NOT import or execute scripts, embeddings as code, or
  evidence content as instructions.
- **FR-012:** Evidence provenance and fingerprints MUST be preserved unless the
  source content is excluded, in which case the exclusion MUST be declared.
- **FR-013:** Import of a bundle from a newer unsupported format version MUST be
  rejected with a clear error.
- **FR-014:** Import MUST report counts for imported, skipped, and rejected entities.
- **FR-015:** Export/import operations MUST be auditable and idempotent with respect
  to their operation ids.
- **FR-016:** Export MUST NOT include secrets, credentials, or unredacted sensitive
  fields; configured redaction MUST be applied.
- **FR-017:** The bundle format MUST support streaming for large stores and MUST NOT
  require loading everything into memory.

## 5. Key Entities

### Export Bundle

| Field | Notes |
|---|---|
| `manifest` | Format version, GUM version, filters, counts |
| `namespaces` | Included namespace metadata |
| `facts` | Current Fact records |
| `fact_revisions` | Full revision chains |
| `dependencies` | Edges and assumptions |
| `dependency_support_groups` | Required/alternative groups |
| `evidence` | Provenance records |
| `embeddings` | Optional canonical vectors and model metadata |
| `events` | Optional audit subset |
| `checksums` | Per-file or per-section digests |

### Import Report

`operation_id`, `imported`, `skipped`, `rejected`, `conflicts`, `warnings`,
`duration_ms`.

## 6. Edge Cases & Error Handling

- Bundle contains a dependency whose target Fact is outside the export scope →
  reject selection or include the target; never import a dangling edge.
- Duplicate Fact ids within a bundle → reject.
- Revision chain with a gap → reject.
- Cyclic dependency within the bundle → reject.
- Checksum mismatch → reject before writes.
- Target namespace unauthorized or missing → require explicit creation/authorization.
- Import interrupted → rollback or staged cleanup; no partial data.
- Bundle from newer format version → reject with actionable error.
- Evidence content missing but fingerprint present → import allowed only with
  explicit `allow_missing_evidence_content`, marked degraded.
- Very large bundle → stream and report progress; bound memory.

## 7. Non-Functional Requirements

- Import/export MUST be streaming and memory-bounded.
- Checksums MUST be deterministic.
- Import validation MUST complete in bounded time and report progress.
- The bundle MUST be portable across operating systems and Node.js versions.
- Sensitive data redaction MUST be configurable and documented.

## 8. Success Criteria

- Export/import round-trip preserves every Fact, revision, edge, and evidence record
  exactly.
- Corrupt or internally inconsistent bundles are rejected without any domain write.
- Namespace remapping leaves no cross-scope references.
- Importing the same bundle twice with `reject` is idempotent/fails cleanly without
  corruption.
- A bundle exported from a newer unsupported version is refused.
- No evidence text is ever executed or interpreted as an instruction.

## 9. Open Questions

- Should the bundle be a single JSON file, JSONL with attachments, or a zip-like
  container? Initial answer: a directory/zip container with JSONL sections and a
  manifest for streaming.
- Should import support dry-run? Recommended yes, via the same validation pipeline.
- Should embedding vectors be exported by default? Initial answer: optional,
  excluded by default due to size and model coupling.
