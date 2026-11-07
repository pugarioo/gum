# Feature Specification: TypeScript SDK

- **Feature ID:** 013
- **Module:** Interface
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §8, §8.4, §9
- **Depends on:** 001-011, 016, 017
- **Blocks:** 014, 015

## 1. Overview

The TypeScript SDK is the embedded programmatic interface to GUM. It opens a local
SQLite-backed engine, exposes typed domain operations, and surfaces structured
errors. All other interfaces (MCP, HTTP, CLI) call the same domain services; the
SDK is the reference consumer of those services.

## 2. Scope

### In scope

- Package exports: engine factory, typed models, error classes
- Embedded local engine lifecycle (`open`, `close`)
- Operations parity with CONCEPT.md §9
- Zod-backed runtime validation with inferred TypeScript types
- Configuration (database path, embedding provider, budgets, migrations)
- Structured errors and retry hints
- Safe concurrency within one process and documented multi-process behavior

### Out of scope

- Remote transport client (HTTP module in 014 may wrap it)
- Browser runtime support
- ORM or query builder exposure
- Automatic cross-process coordination beyond SQLite

## 3. User Scenarios & Testing

### P1 — Open an embedded engine

**Given** a database path
**When** the caller invokes the SDK factory
**Then** the engine opens the database, applies migrations according to config, and
returns a typed client
**And** no global singleton is created.

### P1 — Add and read a Fact

**Given** an open client and an existing namespace
**When** the caller calls `addFact` and `getFact`
**Then** the returned objects are fully typed and reflect the persisted state
**And** validation errors are raised before any write.

### P1 — Structured stale-write error

**Given** a Fact at version 2
**When** the caller updates with `expectedVersion: 1`
**Then** the SDK throws a `StaleWriteError` containing current version and latest
revision id
**And** no mutation occurs.

### P2 — Full operation coverage

**Given** the SDK
**When** a consumer inspects the client
**Then** every operation in CONCEPT.md §9 is available with typed inputs/outputs
**And** progressive disclosure options are exposed.

### P2 — Close and reopen

**Given** a client with committed data
**When** `close()` is called and a new client opens the same path
**Then** all Facts, revisions, dependencies, and evidence are intact
**And** pending durable jobs resume.

### P2 — Configuration

**Given** custom configuration for search budgets, embedding provider, and
migration policy
**When** the engine initializes
**Then** the configuration is validated and applied
**And** invalid configuration fails fast with a clear error.

### P3 — Two clients in one process

**Given** two SDK clients opened against the same database path
**When** both write concurrently
**Then** SQLite serialization and optimistic checks prevent lost updates
**And** at most one conflicting write succeeds.

### P3 — Abort signal

**Given** a long-running search or export
**When** the caller aborts via `AbortSignal`
**Then** the operation stops cooperatively and releases resources
**And** no partial domain mutation remains.

### P4 — Typed error taxonomy

**Given** each documented error condition
**When** it occurs
**Then** the SDK raises a specific error subclass with stable `code`, `retryable`,
and safe details.

## 4. Functional Requirements

- **FR-001:** The package MUST export a factory that opens an embedded engine with a
  required database path and optional typed configuration.
- **FR-002:** The SDK MUST export typed Fact, Revision, Dependency, Evidence,
  Namespace, IntegrityReport, and SearchResult models.
- **FR-003:** Runtime schemas MUST use Zod and TypeScript types MUST be inferred from
  them; duplicate hand-written model types are forbidden.
- **FR-004:** The SDK MUST expose every operation in CONCEPT.md §9 through
  domain-oriented methods.
- **FR-005:** All mutating methods MUST accept explicit concurrency parameters where
  required (`expectedVersion`, `expectedRevision`, reconciliation snapshots).
- **FR-006:** The SDK MUST accept an `AbortSignal` for operations that can be long
  running.
- **FR-007:** Errors MUST be subclasses or structured objects with stable codes and
  retryability, covering validation, not found, stale write/reconciliation, cycle,
  authorization, graph integrity, database busy, and unsupported operation.
- **FR-008:** The engine MUST NOT be a process-global singleton; multiple engines
  with distinct paths MUST be supported.
- **FR-009:** `close()` MUST be idempotent and MUST stop accepting new operations.
- **FR-010:** Migrations MUST run on open by default, configurable to explicit
  migration mode for controlled deployments.
- **FR-011:** The SDK MUST NOT require a frontend or network server.
- **FR-012:** The SDK MUST expose a health/status method with database version,
  migration version, index lag, pending operations, and namespace list summary.
- **FR-013:** Configuration MUST validate database path, busy timeout, budgets,
  embedding provider, model metadata, and migration mode.
- **FR-014:** The SDK MUST document single-writer SQLite behavior and must not
  promise unlimited concurrent writes.
- **FR-015:** The SDK MUST preserve structured error cause chains for diagnostics.
- **FR-016:** The SDK MUST NOT leak internal SQL errors without mapping, while
  retaining SQLite codes in non-sensitive diagnostics.
- **FR-017:** API naming MUST use idiomatic TypeScript (`camelCase`) while mapping
  one-to-one to MCP/HTTP operations.
- **FR-018:** The package MUST be ESM-first with documented Node.js version support
  and no browser bundle requirement.

## 5. Key Entities

### GumEngineConfig

`databasePath`, `busyTimeoutMs`, `migrations` (`auto` | `manual`), `embedding`
(provider, model, dimensions), `retrieval` (default limits, token budget),
`logging`, `namespaceAuthorization`.

### GumClient

Typed methods for facts, revisions, dependencies, integrity, evidence, and admin
operations; `close()`, `status()`.

### Error Classes

`GumValidationError`, `NotFoundError`, `StaleWriteError`,
`StaleReconciliationError`, `CycleDetectedError`, `DependencyMissingError`,
`UnauthorizedScopeError`, `GraphIntegrityError`, `DatabaseBusyError`,
`UnsupportedOperationError`.

## 6. Edge Cases & Error Handling

- Open with missing directory → create only when configured; otherwise clear error.
- Open with incompatible newer schema → refuse and explain.
- Invalid path permissions → startup error with no partial migration.
- Use after `close()` → `EngineClosedError`.
- Duplicate method calls with same idempotency key → return prior result where
  supported.
- Validation failure → no domain call.
- Concurrent close during an operation → operation completes or aborts cleanly.
- Abort after commit point → operation returns success; abort only before commit.
- Embedding provider misconfigured → semantic features degrade; engine still opens
  if configured to allow degradation.

## 7. Non-Functional Requirements

- The SDK MUST not block the event loop on heavy work; use workers.
- Type definitions MUST be published with the package.
- API surface MUST be stable and versioned by semantic versioning.
- Errors and logs MUST not expose secrets or full evidence content by default.
- Documentation MUST include embedded usage, error handling, and concurrency rules.

## 8. Success Criteria

- A consumer can add, search, inspect, and reconcile Facts with typed APIs without
  importing internal modules.
- Every documented error condition maps to a stable typed error.
- Opening two engines in one process does not cross-contaminate state.
- Type tests fail if a model field is renamed without updating schemas.
- Reopening after `close()` resumes durable pending work.
- No network listener or frontend is started by the SDK.

## 9. Open Questions

- Should the SDK expose a lower-level transaction API for advanced consumers?
  Initial answer: no; provide domain operations only.
- Should the SDK include an in-memory SQLite option for tests? Recommended yes,
  clearly marked non-persistent.
