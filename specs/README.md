# GUM Feature Specifications

Specifications derived from `CONCEPT.md` for **GUM — Graph Utility Memory**.

## Layout

GUM is a single product with 18 cohesive modules, not a phased curriculum, so the
phase-number prefix from the specification skill is replaced with a flat module
number:

```
specs/
├── 001-fact-management/spec.md
├── 002-revision-history/spec.md
├── 003-dependency-graph/spec.md
├── ...
└── 018-performance-evaluation/spec.md
```

Each module directory contains one `spec.md`.

## Module Index

| ID | Module | Depends on |
|---|---|---|
| 001 | Fact Management | 010, 016, 017 |
| 002 | Revision History | 001, 010, 016 |
| 003 | Dependency Graph | 001, 002, 010, 016 |
| 004 | Evidence & Provenance | 001, 005, 010, 016 |
| 005 | Integrity & Impact Analysis | 001, 002, 003, 004, 010, 016 |
| 006 | Reconciliation | 002, 003, 004, 005, 016 |
| 007 | Hybrid Retrieval | 001, 003, 008, 010 |
| 008 | Embedding & Vector Index | 001, 002, 010 |
| 009 | Memory Lifecycle & Maintenance | 002, 003, 004, 005, 006, 010 |
| 010 | Persistence & Migrations | — |
| 011 | Export & Import Portability | 001-005, 010 |
| 012 | MCP Server | 001-011, 017 |
| 013 | TypeScript SDK | 001-011, 017 |
| 014 | HTTP API | 001-011, 013, 017 |
| 015 | CLI | 001-014 |
| 016 | Audit Events | 010 |
| 017 | Namespace & Security | 001, 010 |
| 018 | Performance & Evaluation | all modules |

## Priority Definitions

Every module uses the same P1-P4 scale for user scenarios:

- **P1:** Core path. Module is unusable or data is unsafe without it.
- **P2:** Required behavior for correctness, safety, or operator confidence.
- **P3:** Valuable enhancement; can ship after P1/P2 without data loss.
- **P4:** Future-facing or scale concern; explicitly deferred.

## Traceability

Each spec cites the `CONCEPT.md` sections it implements and lists cross-module
dependencies and out-of-scope boundaries. Interface names match CONCEPT.md §9.

## Status

All specs are **Draft** as of 2026-11-08. They describe required behavior, not an
implementation claim.
