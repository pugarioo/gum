# Feature Specification: Performance & Evaluation

- **Feature ID:** 018
- **Module:** Quality
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §12, §14
- **Depends on:** all modules
- **Blocks:** none

## 1. Overview

GUM targets low-overhead, local-first execution and must be evaluated as an AI
memory system, not only as a graph implementation. This module defines performance
budgets, benchmark coverage, worker isolation requirements, operational metrics, and
the evaluation methodology comparing GUM against persistent memory without explicit
dependency-health tracking.

Claims of performance must be measured, not presumed.

## 2. Scope

### In scope

- Benchmark suite and repeatable harness
- Latency, throughput, recall, storage, and memory measurements
- Graph traversal and propagation benchmarks
- Retrieval quality and token-cost evaluation
- Concurrency and crash-recovery tests
- Worker isolation for CPU-heavy work
- Evaluation against a dependency-unaware memory baseline
- Reporting and regression gates

### Out of scope

- Marketing benchmarks against unrelated products
- Distributed/cluster performance
- GPU acceleration
- Model quality research beyond configured embeddings

## 3. User Scenarios & Testing

### P1 — Fact read/write latency benchmark

**Given** a database with 10k, 100k, and 1M Facts
**When** the benchmark runs add/get/list operations
**Then** it reports p50, p95, and p99 latency plus throughput
**And** writes are short and do not block the event loop.

### P1 — Graph traversal benchmark

**Given** a synthetic acyclic graph with varying depth and fan-out
**When** direct, reverse, and transitive queries run
**Then** the harness reports query latency, rows examined, and memory use
**And** verifies indexes are used.

### P1 — Impact propagation benchmark

**Given** a chain and a wide fan-out graph
**When** a root prerequisite changes
**Then** the harness measures affected-Fact detection time, warning counts, and
false-positive rate against a labeled fixture.

### P1 — Retrieval quality evaluation

**Given** a labeled query set with expected relevant Facts
**When** keyword, semantic, and hybrid retrieval run
**Then** the harness reports recall@k, MRR, latency, and token cost per result set
**And** compares health-warning visibility under budgets.

### P2 — Concurrency benchmark

**Given** multiple concurrent readers and writers
**When** writes contend under SQLite WAL
**Then** the harness reports accepted writes, `STALE_WRITE` counts, busy retries,
and max latency
**And** verifies no lost updates.

### P2 — Crash recovery benchmark

**Given** writes and indexing jobs interrupted at controlled points
**When** the process restarts
**Then** the harness verifies committed data integrity, resumed jobs, and no
duplicate effects.

### P2 — Embedding and index lag

**Given** a stream of Fact updates
**When** embedding and indexing run
**Then** the harness reports embedding time per Fact, index lag distribution, and
recall before/after convergence.

### P3 — Storage overhead

**Given** a dataset with revisions, dependencies, and evidence metadata
**When** the harness measures storage
**Then** it reports bytes per Fact, per revision, per edge, and per vector
**And** attributes index and WAL overhead.

### P3 — End-to-end agent token cost

**Given** an agent task that uses GUM through MCP
**When** the task runs with progressive disclosure and budgets
**Then** the harness reports total tokens consumed for memory interaction.

### P4 — Baseline comparison

**Given** the same agent tasks and facts
**When** run with GUM versus persistent memory without dependency-health tracking
**Then** the evaluation reports task success, outdated-assumption decisions,
warning precision/recall, and token cost.

## 4. Functional Requirements

- **FR-001:** A repeatable benchmark harness MUST exist and MUST NOT require
  production data.
- **FR-002:** Benchmarks MUST cover Fact read/write latency and throughput at
  multiple database sizes.
- **FR-003:** Benchmarks MUST cover direct and reverse dependency lookup, bounded
  multi-hop traversal, and impact propagation.
- **FR-004:** Benchmarks MUST measure retrieval latency and recall for keyword,
  semantic, and hybrid modes.
- **FR-005:** The evaluation MUST measure detection of genuinely affected dependents
  and false-positive `possibly_invalid` warnings.
- **FR-006:** The evaluation MUST measure missed dependency edges and missed
  downstream impacts using labeled fixtures.
- **FR-007:** The harness MUST measure reconciliation correctness and cost.
- **FR-008:** The harness MUST measure retrieval token cost, storage overhead,
  embedding time, memory/RSS, and indexing lag.
- **FR-009:** Concurrency tests MUST verify update consistency under multiple agents
  and exactly-one-success semantics per base version.
- **FR-010:** Recovery tests MUST verify behavior after interrupted writes, index
  failures, and crash during export/import.
- **FR-011:** CPU-intensive embedding and large scans MUST run in worker threads or
  native backends and MUST NOT stall the main event loop.
- **FR-012:** Results MUST report environment details (hardware, Node version,
  SQLite version, model, dataset) for reproducibility.
- **FR-013:** Performance regressions beyond configured thresholds MUST fail CI or
  produce a blocking report.
- **FR-014:** Benchmarks MUST assert correctness as well as speed; a fast wrong answer
  is a failure.
- **FR-015:** Index usage for hot queries MUST be verified via SQLite query plans.
- **FR-016:** The primary comparison MUST be agents using GUM versus agents using
  persistent memory without explicit dependency-health tracking.
- **FR-017:** The desired outcome MUST prioritize fewer decisions based on outdated
  assumptions without excessive warning noise.
- **FR-018:** All reported numbers MUST be reproducible from committed fixtures and
  scripts.

## 5. Key Entities

### Benchmark Case

`id`, `dataset`, `operation`, `concurrency`, `iterations`, `metrics`, `environment`.

### Metrics

- Read/write latency p50/p95/p99, throughput
- Graph traversal time, rows examined
- Propagation time and affected counts
- Retrieval recall@k, MRR, latency, token cost
- False-positive and missed-impact rates
- Embedding time, index lag, storage bytes per entity
- Memory/RSS, WAL size
- Crash-recovery success and duplicate-effect counts

### Evaluation Report

Dataset, system under test, task set, success rate, outdated-assumption rate,
warning precision/recall, token cost, environment, commit hash.

## 6. Edge Cases & Error Handling

- Benchmark on slow CI with noisy neighbors → report variance and avoid hard
  microsecond gates; gate on correctness and broad thresholds.
- Dataset too large for CI → provide scaled fixture and nightly full run.
- Embedding model unavailable → semantic benchmarks skipped with explicit status,
  never silently reported as passing.
- False-positive warnings from intentionally ambiguous fixtures → label and report,
  do not hide.
- Crash tests on unsupported filesystems → document limitations.
- SQLite planner chooses a scan after schema change → regression test fails.
- Benchmark measures warm cache only → include cold-start and steady-state results.
- Results compared across different hardware → label as non-comparable.

## 7. Non-Functional Requirements

- Benchmarks MUST be deterministic where practical and report variance.
- The harness MUST clean up databases and temporary files.
- Measurements MUST not require network access except configured providers.
- Reports MUST be machine-readable (JSON) and human-readable.
- Performance claims in documentation MUST reference a recorded benchmark.

## 8. Success Criteria

- A single command runs the core benchmark suite and produces a report.
- Hot-path queries are confirmed indexed.
- No lost updates occur in concurrency tests.
- Crash and index-failure recovery tests pass without duplicate effects.
- Retrieval evaluation reports recall and token cost with health warnings visible.
- Baseline comparison demonstrates lower outdated-assumption decisions without
  unacceptable warning noise.
- Regressions are detected against a stored baseline.

## 9. Open Questions

- What are the concrete initial latency targets? Set after first baseline; avoid
  invented numbers.
- Which embedding model and ANN library form the reference benchmark? Decide with
  the implementation.
- Should quality evaluation use synthetic fixtures only, or also a published agent
  memory benchmark? Prefer a reproducible synthetic suite first, external benchmark
  later.
