# GUM — Graph Utility Memory

**Product Concept & Technical Architecture — Full System Design**  
**Project name:** GUM (Graph Utility Memory)  
**Category:** Open-source, dependency-aware AI memory infrastructure  
**Tagline:** *Memory that understands what depends on what.*  
**Status:** Product concept and proposed architecture; not an implementation claim.

---

## 1. Product Vision

**GUM is a standalone, graph-based AI memory engine** that stores, retrieves, organizes, and maintains interconnected factual knowledge across agent tasks and sessions.

Its defining capability is **dependency-aware memory integrity**. Rather than treating memories as isolated records, GUM models facts as interconnected nodes. A fact can depend on other facts, evidence, or assumptions. When foundational knowledge changes, GUM identifies potentially affected memories, evaluates their dependency health, and supports their reconciliation.

GUM owns its memory infrastructure:

- Persistent fact storage and immutable revision history
- Explicit fact-dependency graph management
- Change-impact analysis and integrity propagation
- Evidence and provenance tracking
- Hybrid memory retrieval and progressive disclosure
- Reconciliation, consolidation, and retention controls
- MCP and TypeScript SDK interfaces

GUM is **not Git-based** and does **not require** another memory provider or a dedicated graph database. It is a **headless engine**; a developer dashboard is **not part of the current product scope**.

## 2. The Problem

Persistent memory helps AI agents maintain context, but creates reliability risks:

1. **Isolated knowledge.** Logically connected claims are stored as unrelated text records.
2. **Outdated assumptions.** A remembered conclusion can persist after its supporting assumptions change.
3. **Cascading inconsistencies.** One changed fact can affect several layers of dependent knowledge.
4. **Uncertain reliability.** A retrieved memory may be relevant but unsupported or no longer applicable.
5. **Poor change awareness.** Agents may not know which other memories need reconsideration after an update.
6. **Retrieval inefficiency.** Agents may receive unnecessary context without visibility into dependency warnings.

**Example:** An agent remembers that a project uses MongoDB (F001), uses Mongoose because of that choice (F002), requires a MongoDB container (F003), and has a deployment procedure based on that container (F004). If the database changes to PostgreSQL, changing F001 alone is insufficient. GUM must identify downstream memories that *may* no longer apply.

GUM does not assert that these dependents are automatically false: changing a prerequisite triggers **reassessment**, not blind deletion or rewriting.

## 3. Core Memory Model

### 3.1 Fact

A **Fact** is the primary, current-state unit of knowledge.

| Field | Purpose |
|---|---|
| `id` | Stable, unique fact identifier |
| `statement` | Current accepted assertion |
| `details` | Embedded structured context; not a separate memory entity |
| `namespace` | Scope, workspace, or project |
| `category` | Knowledge classification |
| `status` | Lifecycle state (e.g., `active`, `retracted`, `archived`) |
| `dependency_health` | Integrity state derived from dependency evaluation |
| `confidence` | Qualified evidence-support assessment, not objective truth |
| `current_version` | Monotonic version for concurrency and dependency tracking |
| `created_at`, `updated_at` | Lifecycle timestamps |

A Fact **does not store an array of its revisions**. Its current statement and details are available directly; historical snapshots live in the Revision model.

Illustrative logical model:

```json
{
  "id": "F001",
  "statement": "The project uses PostgreSQL.",
  "details": { "orm": "Prisma", "purpose": "Primary database" },
  "namespace": "project:shopflow",
  "category": "architecture",
  "status": "active",
  "dependency_health": "healthy",
  "current_version": 3
}
```

### 3.2 Fact Revision

A **Fact Revision** is an immutable snapshot associated with exactly one Fact.

| Field | Purpose |
|---|---|
| `revision_id` | Unique revision identifier |
| `fact_id` | Parent Fact identifier (revision → fact) |
| `previous_revision_id` | Previous accepted revision, if any |
| `version` | Monotonically increasing per-fact version |
| `statement`, `details` | Historical snapshot |
| `change_type` | E.g., correction, supersession, retraction, detail update |
| `author` | Agent, user, or process responsible |
| `created_at` | Recording timestamp |
| `evidence_refs` | Associated evidence identifiers, if supplied |

Revisions are **linear within a memory store**. A Fact's update must check its expected version and atomically create the next revision and update the current Fact. Concurrent writers cannot silently produce two accepted successors from the same base version. A stale writer receives a conflict and must re-read and retry or reconcile.

A later revision is the **currently accepted record**, not proof of objective truth. Updates can also distinguish an earlier mistake from a previously correct statement that is no longer applicable.

### 3.3 Dependency

A **Dependency** connects a dependent Fact to a prerequisite Fact:

`F002 --DEPENDS_ON--> F001`

Each relationship records:

- `from_fact`: dependent fact
- `to_fact`: prerequisite fact
- `type`: initially `DEPENDS_ON`
- `reason`: why the dependency exists
- `assumed_version` or canonical fingerprint: which prerequisite state was relied upon
- Creation and validation metadata
- Optional support-group membership

Dependencies are separate graph edges, not embedded inside the Fact's `details`.

For composite reasoning, GUM supports **required groups** (multiple prerequisites must hold) and **alternative support groups** (an independently sufficient support can remain valid). This avoids treating every change to one of several independent supports as automatically decisive.

### 3.4 Evidence

Evidence records support a claim and may reference:

- Local files and documents
- Tool or API output
- Test results
- Explicitly confirmed observations
- Other source material, with provenance

Evidence metadata includes source identity, observation time, relevant content fingerprint, applicability, and verification state. Evidence changing or becoming unavailable signals a need to reassess; it does not automatically establish that an assertion is false.

### 3.5 Integrity and Change Events

GUM keeps auditable, lightweight events covering fact writes, revisions, dependency updates, invalidation, reconciliation, evidence changes, archiving, and deletion. This is an **audit trail**, not a Git-style commit or branching system.

## 4. Dependency Graph Engine

Facts are nodes, dependencies are directed edges. The graph engine supports:

- Direct and reverse dependency lookup
- Multi-hop traversal and impact analysis
- Graph consistency checks
- Dependency explanations
- Cycle detection and prevention
- Required and alternative supports
- Efficient graph indexing and selective expansion

The **revision history** and **fact-dependency graph** are distinct:

- *Revision history* answers: **How did this assertion change?**
- *Dependency graph* answers: **What relies on this assertion?**

A change to a Fact increments its version. Dependency edges retain the assumed prerequisite version so GUM can explain what changed and why a dependent Fact was flagged.

The initial relationship semantics are explicit. Semantic dependency discovery may suggest edges, but it cannot guarantee that all correct dependencies have been found.

## 5. Memory Integrity and Reconciliation

### 5.1 Change Detection

When a Fact changes, GUM distinguishes substantive assertion changes from non-substantive metadata changes. Dependency-aware impact logic is triggered according to the change's relevant scope. Metadata-only edits need not invalidate unrelated knowledge.

### 5.2 Cascading Impact Analysis

GUM traverses **reverse dependencies** from the modified prerequisite to locate direct and transitive dependents. It then evaluates support groups, assumed versions, and existing warnings to determine which require review.

### 5.3 Dependency Health

| State | Meaning |
|---|---|
| `healthy` | No **known unresolved** dependency issues |
| `possibly_invalid` | A prerequisite or relevant support has changed |
| `unresolved` | Insufficient or conflicting support requires investigation |

These statuses measure **known dependency integrity**, not factual truth. `healthy` does not mean independently verified. Fact lifecycle (`active`, `retracted`, `archived`) remains separate from dependency health.

### 5.4 Reconciliation

An authorized agent or validator may:

- Confirm the current Fact remains applicable
- Revise the statement/details
- Update the dependency or support-group structure
- Attach new evidence
- Retract the Fact
- Preserve uncertainty and the warning

Reconciliation records **who/what checked the memory**, against **which prerequisite versions and evidence**, and the **outcome**. A warning can only be cleared after relevant assumptions are reconsidered; a change during reconciliation requires rechecking.

### 5.5 Atomicity and Propagation

The fact update, revision snapshot, affected dependency state, and associated events are committed atomically where they share the same SQLite transaction. Any deferred work must be recorded durably so interruptions cannot silently lose pending integrity checks.

## 6. Native Hybrid Retrieval

GUM provides its own retrieval infrastructure, independent of other memory providers.

**Retrieval signals:**

1. **Keyword:** SQLite FTS5 across statements and selected details.
2. **Semantic:** Embeddings and an approximate-nearest-neighbor index.
3. **Structured:** Namespace, type, lifecycle and health filters.
4. **Graph-aware:** Relevant prerequisites, dependents, and impact signals.

Results are fused and optionally reranked using the most relevant signals. Dependency-health status is always surfaced where material; results should never appear confidently verified just because they ranked highly.

**Progressive disclosure:** `SEARCH → SELECT → INSPECT → EXPAND`.

- `search_facts` returns concise statements, IDs, scope, and important health warnings.
- `get_fact` returns a selected Fact with details.
- `get_dependencies` and `explain_dependency` expand graph context only when requested.
- History and evidence are available on demand.

Retrieval honors token/result budgets, limits expansion depth, and explicitly indicates truncated or omitted information. A retrieval budget must not silently remove material warnings.

## 7. Full Memory Lifecycle

GUM supports ingestion, classification, deduplication, dependency suggestion, persistence, revision, integrity checks, reconciliation, consolidation, archiving, expiration, deletion, and scheduled/triggered maintenance.

- **Deduplication and consolidation** must preserve meaning, evidence, and graph links.
- **Proposed dependencies** are distinguishable from user-confirmed or validated dependencies.
- **Autonomous semantic maintenance** may use configured models or agents; deterministic graph traversal does not require an LLM.
- **Retention and deletion** must define treatment of linked dependencies, audit history, and evidence records.

## 8. Technology Stack

GUM is **TypeScript-first**, uses **Node.js**, and is designed as a **headless modular monolith**.

| Layer | Technology / approach |
|---|---|
| Core engine | TypeScript on Node.js |
| Authoritative database | SQLite |
| Database driver | `better-sqlite3` |
| Runtime validation | Zod |
| Graph operations | Indexed SQLite tables, recursive CTEs, selective TypeScript adjacency caches |
| Keyword retrieval | SQLite FTS5 |
| Semantic retrieval | Pluggable native ANN/vector index |
| Embeddings | ONNX Runtime for Node.js (local), plus configurable providers |
| MCP interface | Official TypeScript MCP SDK |
| HTTP integration | Fastify, optional/headless |
| Embedded SDK | TypeScript package |
| CLI | TypeScript command-line package |
| Testing | Vitest, integration tests, and performance benchmarks |
| CPU-intensive work | Node.js worker threads or native libraries |
| Package management | pnpm |
| Database migrations | Versioned SQL migrations |

**No developer dashboard is included in the current scope.** Running the MCP server must not start a Next.js or other frontend server.

### 8.1 SQLite as Source of Truth

SQLite stores:

- `facts`
- `fact_revisions`
- `dependencies`
- `dependency_support_groups`
- `evidence`
- `integrity_events`
- `namespaces`
- `embeddings` and indexing metadata
- `memory_operations` and maintenance state

Recommended database controls include WAL, foreign-key enforcement, indexed reverse-dependency lookups, prepared statements, short write transactions, and optimistic version checks.

SQLite WAL allows concurrent readers and a writer, but SQLite still allows only **one active writer at a time**. GUM should design around this rather than claim unlimited concurrent writes.

### 8.2 Graph Processing

Use recursive SQL and reverse-edge indexes for authoritative traversal. Optional in-memory adjacency caches accelerate frequent graph queries but must be invalidated/rebuilt from persisted data.

Dedicated graph infrastructure such as Neo4j is not required by the proposed graph model.

### 8.3 Vector Index Consistency

Store canonical embedding vectors and model metadata durably in SQLite. Treat the ANN index as **derived/rebuildable**. A persisted indexing queue or outbox coordinates database writes with vector-index updates, allowing crash recovery and reindexing.

Changing embedding models must explicitly manage vector dimensions and re-embedding.

### 8.4 Runtime and Deployment

- A local **stdio MCP server** can be launched by an agent client and exit on disconnect.
- The same engine can run embedded through the TypeScript SDK.
- An optional **Fastify HTTP service** can serve networked consumers.
- Large synchronous database work or CPU-heavy embedding calculations must not stall the main Node.js event loop; use workers or native backends as warranted.
- No frontend is required.

## 9. Interfaces and Operations

All interfaces call the same domain services; the MCP server is an adapter, not the location of core business logic.

### Fact Management

`add_fact`, `update_fact`, `get_fact`, `delete_fact`, `search_facts`, `list_facts`

### Revision History

`get_revisions`, `get_revision`, `compare_revisions`, `inspect_history`

### Dependency Graph

`add_dependency`, `remove_dependency`, `get_dependencies`, `get_dependents`, `explain_dependency`, `inspect_graph`

### Integrity and Reconciliation

`analyze_impact`, `validate_dependencies`, `get_affected_facts`, `reconcile_fact`, `get_integrity_report`

### Evidence

`attach_evidence`, `inspect_evidence`, `verify_evidence`, `list_evidence`

### Administration and Portability

`create_namespace`, `export_memory`, `import_memory`, `configure_retention`, `rebuild_index`

The SDK and MCP interfaces expose structured errors for stale writes, missing dependencies, cyclic relationships, unauthorized scopes, and unresolved graph integrity issues.

## 10. Reliability, Security, and Concurrency

The full system requires:

- Atomic database transactions for multi-entity changes
- Immutable accepted revision snapshots
- Optimistic concurrency on Fact updates
- Referential integrity and dependency cycle prevention
- Durable invalidation/reconciliation state
- Evidence provenance and clear untrusted-input boundaries
- Namespace isolation and authorization
- Input/schema validation
- Auditable changes with configured retention and deletion
- Backup, import/export integrity checks, and crash recovery
- No promotion of retrieved text or evidence into privileged instructions

**Security boundary:** Retrieval relevance, graph position, and source presence never independently authorize a memory as an instruction or prove a claim true.

## 11. Autonomous Maintenance

Configurable maintenance processes may detect changed evidence, pending dependency-health warnings, duplicate or contradictory claims, missing supporting links, and memories eligible for consolidation, archival, or reconsideration.

Deterministic structural checks should execute without LLM inference. Contradiction detection, dependency suggestion, and semantic fact reconciliation may require a configured model or an authorized agent and must present uncertainty and evidence, not pretend to be infallible.

## 12. Performance Principles

GUM targets low-overhead, local-first execution through:

- Indexed SQLite access and FTS5 instead of scanning all facts in TypeScript
- Efficient reverse-edge lookup and bounded graph expansion
- Atomic, short write transactions
- Native ANN indexing and local ONNX embeddings
- Compact, progressively expanded retrieval responses
- Durable, incremental vector-index synchronization
- Worker isolation for CPU-heavy operations
- Measured optimizations rather than presumed runtime superiority

Performance measurements should include fact read/write latency, graph traversal time, propagation time, retrieval latency and recall, embedding time, memory/RSS use, indexing lag, storage overhead, and end-to-end agent token consumption.

## 13. Portability and Multi-Agent Operation

GUM is **Git-independent**. It supports named memory stores/namespaces, exports and imports facts with their revisions, dependencies, and evidence, and can be accessed by multiple authorized agents.

Single-database concurrency is governed by SQLite transactions and optimistic write checks. Cross-machine live synchronization, if added, must explicitly address identity, ordering, conflict resolution, and data integrity; it is not implicitly provided by SQLite or MCP.

## 14. Evaluation

GUM should be evaluated as an **AI memory system**, not only as a graph implementation. Core metrics include:

- Relevant-memory recall and task success
- Detection of genuinely affected dependents
- False-positive `possibly_invalid` warnings
- Missed dependency edges and missed downstream impacts
- Correctness and cost of reconciliation
- Retrieval token cost, latency, and storage overhead
- Update consistency under concurrent agents
- Recovery after interrupted writes and index failures

The key comparison is between agents using GUM and agents using persistent memory **without explicit dependency-health tracking**. The desired outcome is fewer decisions based on outdated supporting assumptions without excessive warning noise.

## 15. Product Identity and Scope

**GUM — Graph Utility Memory** is an independent, full-featured, graph-based memory engine for AI agents. Its signature is the **fact dependency graph** and the integrity/reconciliation process built around it.

It is **not**:

- Git for AI memory
- A Git-like commit, branch, or merge system
- A memory debugger layered over another provider
- Merely an MCP wrapper around an existing memory service
- A dashboard-first application

Its central principle remains:

> **An AI agent should not only remember what it knows. Its memory system should understand what that knowledge depends on.**

---

**Architecture summary:** TypeScript · Node.js · SQLite · `better-sqlite3` · Zod · FTS5 · pluggable ANN search · ONNX Runtime · MCP · TypeScript SDK · optional Fastify HTTP API · Vitest · pnpm.  
**UI:** Headless; no developer dashboard currently planned.  
**History:** Immutable linear Fact revisions, independent of Git.  
**Graph:** Explicit dependency edges, change-impact propagation, support-aware reconciliation.  
**Scope:** Full product concept; not an MVP-only specification.
