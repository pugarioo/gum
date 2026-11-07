# Feature Specification: Hybrid Retrieval

- **Feature ID:** 007
- **Module:** Retrieval
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §6, §12, §9 (Fact Management)
- **Depends on:** 001, 003, 008, 010
- **Blocks:** 012, 013, 014, 015

## 1. Overview

GUM owns retrieval. It fuses keyword, semantic, structured, and graph-aware signals
to return concise, dependency-aware results. Retrieval must never imply that a
high-ranked memory is verified. Dependency-health warnings are always surfaced
when material, even under tight token budgets.

Progressive disclosure is mandatory:
`SEARCH → SELECT → INSPECT → EXPAND`.

## 2. Scope

### In scope

- `search_facts` with keyword, semantic, hybrid, and structured filters
- SQLite FTS5 keyword search
- Embedding-based semantic search through the ANN adapter (008)
- Fusion and optional reranking
- Graph-aware expansion signals (prerequisites, dependents, impact)
- Progressive disclosure response shapes
- Token/result budgets and truncation reporting
- Namespace isolation and authorization

### Out of scope

- Embedding generation and ANN maintenance (008)
- Fact mutation (001, 002)
- Reconciliation decisions (006)
- Cross-namespace search without explicit authorization (017)
- Answer generation or summarization by a model

## 3. User Scenarios & Testing

### P1 — Keyword search

**Given** Facts containing `"PostgreSQL"` and unrelated Facts
**When** `search_facts` is called with query `"PostgreSQL"` and mode `keyword`
**Then** GUM uses FTS5 and returns matching Facts ranked by relevance
**And** each result contains id, statement, namespace, category, status,
dependency_health, score, and match reason
**And** archived Facts are excluded by default.

### P1 — Health warnings always survive budget limits

**Given** a `possibly_invalid` Fact that matches a query
**When** the caller sets a very tight token/result budget
**Then** GUM still returns or explicitly reports the material health warning
**And** MUST NOT silently omit the warning while returning the statement.

### P1 — Namespace isolation

**Given** Facts with similar statements in two namespaces
**When** the caller searches namespace A
**Then** no Fact from namespace B is returned, even at high semantic similarity.

### P2 — Hybrid semantic plus keyword

**Given** a query and an up-to-date vector index
**When** `search_facts` is called with mode `hybrid`
**Then** GUM fuses keyword and semantic candidates, deduplicates by Fact id, and
returns a deterministic combined ranking
**And** includes a `signals` explanation for each result.

### P2 — Graceful degradation

**Given** the vector index is unavailable or stale
**When** semantic or hybrid search is requested
**Then** GUM degrades to keyword search, includes an explicit `degraded` indicator,
and does not fail the entire query.

### P2 — Structured filters

**Given** Facts across categories and statuses
**When** `search_facts` includes namespace, category, status, and dependency-health
filters
**Then** every returned Fact matches all filters.

### P3 — Graph-aware expansion

**Given** a high-ranked Fact with dependencies
**When** `search_facts` is called with `graph_context: true` and `depth: 1`
**Then** GUM includes compact prerequisite/dependent summaries or impact signals
**And** does not expand beyond the requested depth.

### P3 — Progressive disclosure flow

**Given** a concise search result
**When** the caller calls `get_fact`
**Then** GUM returns full details for that Fact
**When** the caller calls `get_dependencies` or `explain_dependency`
**Then** GUM expands graph context on demand
**When** history or evidence is requested
**Then** GUM returns those records only on explicit request.

### P4 — Truncation visibility

**Given** more matching Facts than the result budget allows
**When** search returns
**Then** GUM sets `truncated: true`, reports the applied limits, and provides a
stable cursor or offset for continuation.

## 4. Functional Requirements

- **FR-001:** `search_facts` MUST support `keyword`, `semantic`, and `hybrid` modes;
  `hybrid` is the default when embeddings are available.
- **FR-002:** Keyword search MUST use SQLite FTS5 over Fact statements and selected
  searchable details, with a documented tokenizer configuration.
- **FR-003:** Semantic search MUST use the pluggable ANN index and canonical
  embeddings from 008; it MUST NOT compute vectors on the request path.
- **FR-004:** Hybrid search MUST fuse ranked candidate lists deterministically,
  deduplicate by Fact id, and expose per-result signal contributions.
- **FR-005:** Structured filters MUST support namespace, category, status,
  dependency health, and updated-at ranges.
- **FR-006:** Default search MUST exclude `archived` and `retracted` Facts unless
  explicitly requested.
- **FR-007:** Search results MUST be concise by default: id, statement, namespace,
  category, status, dependency_health, score, and match reason.
- **FR-008:** Dependency-health warnings MUST be surfaced for every material result
  and MUST NOT be removed by token or result budgets; if space is insufficient,
  GUM MUST return fewer results, not hide warnings.
- **FR-009:** Results MUST indicate when semantic search was degraded, when the
  index is stale beyond the configured lag threshold, and when graph context was
  truncated.
- **FR-010:** Graph-aware retrieval MUST be opt-in, depth-bounded, and MUST NOT
  recursively traverse the whole graph by default.
- **FR-011:** `search_facts` MUST never mutate Fact data or health state.
- **FR-012:** Search MUST be namespace-isolated and authorization-checked before
  querying any index.
- **FR-013:** Pagination MUST be deterministic with a stable tie-breaker (score,
  then Fact id) and MUST NOT repeat or skip results across pages.
- **FR-014:** The result budget MUST accept a maximum result count and an optional
  token budget; applied limits and truncation MUST be reported.
- **FR-015:** Optional reranking MUST use only configured local or authorized
  signals and MUST NOT silently replace warning visibility with relevance.
- **FR-016:** Retrieval MUST label Fact text and evidence as untrusted data, never
  as instructions.
- **FR-017:** FTS and ANN indexes MUST be updated in coordination with Fact writes;
  stale index reads MUST be detectable.
- **FR-018:** Empty queries MUST be rejected unless the caller explicitly requests
  a filtered listing.

## 5. Key Entities

### Search Request

| Field | Notes |
|---|---|
| `query` | Keyword or natural-language query |
| `mode` | `keyword`, `semantic`, `hybrid` |
| `namespace` | Required scope |
| `filters` | Category, status, health, dates |
| `graph_context` | Optional expansion flag |
| `depth` | Expansion depth |
| `limit` / `token_budget` | Budget controls |

### Search Result

| Field | Notes |
|---|---|
| `fact_id` | Stable Fact id |
| `statement` | Concise current statement |
| `namespace`, `category`, `status` | Classification |
| `dependency_health` | Always included |
| `score` | Fused relevance score |
| `match_reason` | Keyword, semantic, graph, filter |
| `signals` | Optional per-signal detail |
| `warning` | Material health/evidence warning |

### Retrieval Envelope

`results`, `truncated`, `applied_limits`, `degraded`, `index_lag_ms`, `next_cursor`.

## 6. Edge Cases & Error Handling

- Empty or whitespace query → `VALIDATION_ERROR` unless listing mode is explicit.
- FTS syntax characters in user query → safe escaping; no raw query injection.
- Vector index missing while semantic requested → degraded keyword fallback with
  warning; do not return empty success.
- Index behind database → include staleness indicator; optionally exclude
  not-yet-indexed Facts with an explicit note.
- Result budget too small for any full result → return a compact minimal payload
  that preserves ids and warnings, plus truncation metadata.
- Namespace unauthorized → `UNAUTHORIZED_SCOPE`, no partial results.
- Corrupt ANN ids referencing deleted Facts → filter and log integrity warning.
- duplicate candidates from keyword and semantic → one result per Fact.
- Archived Fact explicitly requested → include with status visible.

## 7. Non-Functional Requirements

- Keyword search MUST use indexed FTS5, not in-memory scans.
- Hybrid search target latency MUST be measured and bounded per namespace size.
- Result envelopes MUST be compact and token-aware.
- Search MUST support concurrent readers under SQLite WAL.
- Deterministic ordering MUST be reproducible for identical data and query.

## 8. Success Criteria

- A query matching only a `possibly_invalid` Fact always exposes the warning, even
  at the smallest allowed budget.
- Search never returns Facts outside the requested namespace.
- Hybrid results contain no duplicate Fact ids.
- Degraded semantic search is explicitly reported 100% of the time.
- Pagination over a fixed dataset returns each matching Fact exactly once.
- Search does not write to Fact, revision, or dependency tables.

## 9. Open Questions

- Which fusion algorithm is canonical? Initial answer: reciprocal rank fusion with
  configurable weights; reranking is a later additive layer.
- Should semantic search require an explicit mode or default automatically when
  embeddings exist? Initial answer: default to hybrid when healthy.
- What is the default stale-index threshold for warning users? Decide by benchmark.
