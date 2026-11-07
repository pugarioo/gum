# Feature Specification: Embedding & Vector Index

- **Feature ID:** 008
- **Module:** Retrieval
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §6, §8.3, §12
- **Depends on:** 001, 002, 010, 016
- **Blocks:** 007

## 1. Overview

Semantic retrieval requires embeddings and an approximate-nearest-neighbor (ANN)
index. SQLite is the source of truth for canonical vectors and model metadata; the
ANN index is derived and rebuildable. A durable indexing queue/outbox coordinates
database writes with vector-index updates so crashes and index failures are
recoverable.

Embedding models may change. GUM must manage dimensions and re-embedding explicitly.

## 2. Scope

### In scope

- Canonical embedding storage in SQLite with model metadata
- Pluggable embedding providers, including local ONNX Runtime
- Pluggable ANN adapter interface
- Durable indexing outbox and worker processing
- Crash recovery and idempotent indexing
- `rebuild_index`
- Model change and re-embedding workflow
- Index lag reporting

### Out of scope

- Search fusion and ranking (007)
- Remote hosted embedding services without explicit configuration
- GPU/accelerator management
- Cross-store index replication

## 3. User Scenarios & Testing

### P1 — Embed and index a new Fact

**Given** a Fact is created or substantively updated
**When** the write commits
**Then** an indexing job is durably enqueued in the same transaction
**And** a worker computes the embedding, stores the canonical vector and model
metadata in SQLite, and updates the ANN index
**And** the job is marked complete idempotently.

### P1 — Crash during indexing

**Given** a Fact write committed and an indexing job is pending
**When** the process crashes before the ANN update
**Then** restart resumes pending jobs
**And** the ANN index converges to include the Fact without duplicates.

### P1 — Local embeddings by default

**Given** the default configuration
**When** embeddings are computed
**Then** GUM uses the configured local ONNX model in a worker or native backend
**And** does not block the Node.js event loop.

### P2 — Rebuild the index

**Given** canonical vectors in SQLite
**When** `rebuild_index` is called
**Then** GUM rebuilds the ANN index from canonical data with progress reporting
**And** search remains available in degraded mode until the rebuild completes.

### P2 — Change embedding model

**Given** existing embeddings from model A with dimension 384
**When** configuration switches to model B with dimension 768
**Then** GUM creates a new model generation, re-embeds Facts into the new
dimension, and never mixes model generations in one ANN query
**And** reports progress and completion.

### P3 — Index failure and retry

**Given** an ANN write fails transiently
**When** the worker retries
**Then** retries use bounded backoff and the job remains durable
**And** permanent failures are marked with a diagnostic status without corrupting
canonical vectors.

### P3 — Delete removes vectors

**Given** a Fact is deleted or archived according to policy
**When** the change commits
**Then** its vectors are removed from the ANN index or excluded from queries
**And** canonical vector rows follow retention policy.

### P4 — Provider parity

**Given** a configured remote or alternative embedding provider
**When** it satisfies the provider interface
**Then** GUM can use it with the same canonical storage and index workflow.

## 4. Functional Requirements

- **FR-001:** GUM MUST store canonical embedding vectors in SQLite with `fact_id`,
  `model_id`, `model_version`, `dimensions`, `vector`, and `created_at`.
- **FR-002:** Every substantive Fact create/update MUST enqueue exactly one
  idempotent indexing job in the same transaction as the Fact write.
- **FR-003:** Metadata-only changes MAY skip re-embedding unless they affect
  searchable content.
- **FR-004:** Embedding computation MUST run off the main event loop (worker thread
  or native backend).
- **FR-005:** The embedding provider MUST be an interface supporting at least local
  ONNX and configurable providers, with model id, version, and dimensions declared.
- **FR-006:** The ANN index MUST be treated as derived and must be fully rebuildable
  from canonical vectors without loss.
- **FR-007:** ANN writes MUST be idempotent by `(fact_id, model_generation)` and
  MUST tolerate duplicate deliveries.
- **FR-008:** The indexing outbox MUST persist status, attempt count, last error,
  and idempotency key, and MUST survive restart.
- **FR-009:** Failed jobs MUST retry with bounded exponential backoff; permanent
  failures MUST be visible in index health reporting.
- **FR-010:** `rebuild_index` MUST operate from canonical SQLite data, report
  progress, and avoid serving mixed old/new model generations.
- **FR-011:** A model change MUST require explicit re-embedding; GUM MUST NOT
  compare vectors from different model generations.
- **FR-012:** Index health MUST expose pending job count, oldest pending age, last
  successful index time, failure count, and active model generation.
- **FR-013:** Fact deletion MUST enqueue vector removal or immediately mark vectors
  ineligible for results.
- **FR-014:** Embedding input MUST be deterministic for a given Fact version and
  model (documented statement/details composition).
- **FR-015:** Vector dimensions MUST be validated against model metadata before
  storage or query.
- **FR-016:** The ANN adapter MUST be pluggable without changing domain services.
- **FR-017:** Index data MUST be namespace-filterable at query time, or results MUST
  be filtered post-query before exposure.
- **FR-018:** All index operations MUST be auditable at operation level, not per
  vector by default.

## 5. Key Entities

### Embedding

| Field | Type | Notes |
|---|---|---|
| `id` | string | Unique embedding id |
| `fact_id` | string | Source Fact |
| `fact_version` | integer | Embedded version |
| `model_id` | string | Provider/model identifier |
| `model_version` | string | Model release |
| `dimensions` | integer | Vector size |
| `vector` | blob/array | Canonical vector |
| `created_at` | timestamp | Computation time |

### Indexing Job

| Field | Type | Notes |
|---|---|---|
| `id` | string | Job id |
| `fact_id` | string | Target Fact |
| `fact_version` | integer | Target version |
| `operation` | enum | `upsert`, `delete`, `reembed` |
| `status` | enum | `pending`, `running`, `failed`, `done` |
| `attempts` | integer | Retry count |
| `last_error` | string \| null | Diagnostic |
| `idempotency_key` | string | Dedup key |
| `created_at`, `updated_at` | timestamps | Lifecycle |

### Model Generation

`model_id`, `model_version`, `dimensions`, `status` (`active`, `building`,
`retired`), and re-embedding progress.

## 6. Edge Cases & Error Handling

- Embedding provider unavailable → job remains pending with backoff; semantic
  search degrades (007).
- Model dimension mismatch → reject before writing; report configuration error.
- Duplicate indexing jobs for same fact version → idempotent collapse.
- Fact deleted before embedding completes → cancel or discard job and ensure no
  stale vector is queryable.
- Rebuild interrupted → resume from persisted progress; canonical vectors remain
  authoritative.
- Corrupt vector blob → mark job failed and exclude from index; do not crash search.
- Model switch mid-rebuild → old generation remains queryable until new generation
  is fully active, or semantic search is degraded; never mixed.
- Storage growth → retention policy for obsolete model generations (009).

## 7. Non-Functional Requirements

- Embedding work MUST NOT block request handling.
- Indexing lag MUST be measurable and bounded under normal load.
- Canonical vectors MUST survive process restart.
- ANN adapter operations MUST be atomic per job from the queue's perspective.
- Memory use during rebuild MUST be bounded; stream from SQLite.

## 8. Success Criteria

- Every active Fact with searchable content has one canonical vector for the active
  model generation.
- Killing the process mid-index results in zero lost jobs and no duplicate vectors.
- Rebuilding from canonical data produces equivalent search recall to the prior index.
- Switching models never returns cross-generation similarity scores.
- Semantic search remains available or explicitly degraded during maintenance.

## 9. Open Questions

- Which ANN library is the initial adapter? Decide by benchmark; the interface must
  remain library-neutral.
- Should embeddings be versioned per Fact revision instead of per Fact version?
  Initial answer: per Fact version, with model generation as a second key.
- Should re-embedding be automatic on model change or operator-triggered? Initial
  answer: operator-triggered with an explicit migration job.
