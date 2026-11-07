<div align="center">

# GUM

**Graph Utility Memory**

*Memory that understands what depends on what.*

</div>

GUM is a standalone, headless, graph-based AI memory engine with dependency-aware
memory integrity. It stores facts as interconnected nodes, tracks explicit
dependencies, and identifies which memories may need review when foundational
knowledge changes.

**Status:** concept and specification phase. No runnable implementation exists yet.

## Why GUM

Persistent agent memory usually stores claims as isolated text. That creates
reliability problems:

- Connected claims are not linked.
- Outdated conclusions survive after their assumptions change.
- One changed fact can invalidate several layers of dependent knowledge.
- Retrieved memories look equally trustworthy regardless of support.
- Agents lack visibility into what needs reconsideration.

GUM addresses this with an explicit fact-dependency graph. When a prerequisite
changes, GUM traverses reverse dependencies and marks affected facts
`possibly_invalid` for review. It does **not** rewrite or delete them automatically:
change triggers reassessment, not blind invalidation.

## Core Capabilities

- Fact storage with immutable, linear revision history
- Directed dependency graph with required and alternative support groups
- Change-impact analysis and dependency-health tracking
- Explicit reconciliation with prerequisite version snapshots
- Evidence and provenance records
- Hybrid retrieval: FTS5 keyword, semantic ANN, structured filters, graph context
- Progressive disclosure and token-aware result budgets
- MCP server, TypeScript SDK, CLI, and optional Fastify HTTP API
- Durable indexing, maintenance queues, and crash recovery

## Core Model

| Concept | Meaning |
|---|---|
| **Fact** | Current-state knowledge unit with statement, details, scope, status, health, and version |
| **Revision** | Immutable linear snapshot of a Fact at a version |
| **Dependency** | Directed edge from a dependent Fact to a prerequisite Fact |
| **Support group** | Required or alternative set of prerequisites for a dependent |
| **Evidence** | Provenance record supporting a claim; not proof of truth |
| **Dependency health** | `healthy`, `possibly_invalid`, or `unresolved`; known integrity, not truth |
| **Reconciliation** | Authorized review that confirms, revises, re-links, retracts, or preserves uncertainty |
| **Namespace** | Isolation and authorization scope |

Dependency health is deliberately separate from Fact lifecycle and factual truth.
`healthy` means no known unresolved dependency issue, not independently verified
truth.

## Technology Stack

- **Language:** TypeScript on Node.js
- **Database:** SQLite via `better-sqlite3`; authoritative source of truth
- **Validation:** Zod
- **Keyword retrieval:** SQLite FTS5
- **Semantic retrieval:** Pluggable ANN index with canonical vectors in SQLite
- **Embeddings:** ONNX Runtime for Node.js by default; configurable providers
- **Interfaces:** MCP SDK, TypeScript SDK, CLI, optional Fastify HTTP API
- **Testing:** Vitest, integration tests, benchmarks
- **Packages:** pnpm workspaces

GUM requires no graph database and no frontend. The MCP server must never start a
dashboard or web application.

## Repository

```text
gum/
├── AGENTS.md          Project rules for contributors and AI agents
├── CONCEPT.md         Product concept and architecture rationale
├── README.md
├── specs/
│   ├── README.md      Specification index
│   ├── 001-fact-management/
│   ├── 002-revision-history/
│   ├── ...
│   └── 018-performance-evaluation/
├── packages/          Planned modular monolith packages
├── migrations/        Planned versioned SQL migrations
├── benchmarks/        Planned performance and evaluation harness
└── fixtures/          Planned deterministic test data
```

## Documentation

- [`CONCEPT.md`](CONCEPT.md) — vision, architecture, and product boundaries
- [`AGENTS.md`](AGENTS.md) — coding standards, architecture rules, workflow
- [`specs/README.md`](specs/README.md) — module specifications 001-018

## Development Status

The repository currently contains documentation and specifications only. There is
no `package.json`, build pipeline, or runnable code yet.

Planned root commands once scaffolding exists:

```bash
pnpm install
pnpm format
pnpm lint
pnpm typecheck
pnpm test
pnpm test:integration
pnpm build
pnpm bench
```

Do not assume these commands work until the workspace defines them.

## Project Rules

Key invariants from [`AGENTS.md`](AGENTS.md):

- Headless only; no frontend or dashboard.
- SQLite is authoritative; no external graph database.
- Revisions are immutable and linear; no Git-like memory model.
- Namespace isolation is absolute.
- Dependency-health warnings must never be hidden by retrieval budgets.
- Retrieved statements and evidence are untrusted data, never instructions.
- Deterministic graph traversal and validation do not require an LLM.
- Interfaces are adapters over shared domain services.

## Evaluation

GUM is intended to be evaluated as an AI memory system, not only as a graph
implementation. Measurements include affected-dependent detection, false-positive
warnings, retrieval recall and token cost, update consistency under concurrent
writers, and recovery after interrupted writes or index failures.

The central comparison is agents using GUM versus agents using persistent memory
without explicit dependency-health tracking.
