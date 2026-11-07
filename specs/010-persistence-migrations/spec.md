# Feature Specification: Persistence & Migrations

- **Feature ID:** 010
- **Module:** Infrastructure
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §8, §8.1, §8.2, §10
- **Depends on:** none
- **Blocks:** all stateful modules

## 1. Overview

SQLite is GUM's authoritative source of truth. Persistence owns the schema,
database configuration, migrations, transaction boundaries, indexes, prepared
statements, and recovery behavior. SQLite WAL permits concurrent readers and one
writer; GUM must design around single-writer serialization rather than claim
unlimited concurrent writes.

Correctness requirements from the rest of the system depend on this module:
atomic multi-entity commits, immutable revisions, optimistic version checks,
referential integrity, durable queues, and crash recovery.

## 2. Scope

### In scope

- SQLite configuration (WAL, foreign keys, busy timeout)
- Schema for all durable entities
- Versioned SQL migrations
- Transaction API with commit/rollback and busy handling
- Indexes for reads, reverse-edge lookups, FTS5, and queues
- Prepared statement management
- Startup integrity checks and recovery
- Backup and checkpoint support

### Out of scope

- ANN index storage implementation (008)
- Domain semantics (001-009)
- Network database servers
- Cross-machine replication

## 3. User Scenarios & Testing

### P1 — Fresh database initialization

**Given** no database file
**When** the engine starts
**Then** GUM creates the database, applies all migrations in order, and records
the schema version
**And** enables WAL, foreign keys, and the configured busy timeout.

### P1 — Atomic multi-entity write

**Given** an operation that creates a Fact, a revision, and an audit event
**When** any step fails
**Then** the transaction rolls back completely
**And** no partial Fact or orphan revision exists.

### P1 — Crash recovery

**Given** an interrupted write or queued operation
**When** the process restarts
**Then** SQLite recovers committed transactions
**And** GUM resumes durable pending work without duplicating completed effects.

### P2 — Migration upgrade

**Given** an existing database at schema version N
**When** the engine starts with migrations through N+2
**Then** each migration runs transactionally in order and the recorded version is
updated only after success
**And** a failed migration leaves the database at the last successful version.

### P2 — Concurrent readers and a writer

**Given** multiple readers and one writer
**When** reads and a write overlap
**Then** readers observe a consistent committed snapshot and the writer commits
atomically
**And** a second writer waits within the busy timeout or fails with a retryable
`SQLITE_BUSY`-based error.

### P2 — Prepared statements

**Given** repeated Fact and dependency queries
**When** the engine executes them
**Then** reused prepared statements are used rather than re-parsing SQL.

### P3 — Backup and checkpoint

**Given** an active database
**When** a backup is requested
**Then** GUM produces a consistent copy using SQLite backup or WAL checkpoint
semantics without blocking readers for the entire operation.

### P3 — Integrity verification

**Given** a database file
**When** startup integrity or an explicit check runs
**Then** GUM reports foreign-key violations, missing revision chains, orphan edges,
and queue inconsistencies without silently repairing domain data.

## 4. Functional Requirements

- **FR-001:** SQLite MUST be opened with `journal_mode = WAL`,
  `foreign_keys = ON`, and a configurable `busy_timeout`.
- **FR-002:** The schema MUST include at minimum: `namespaces`, `facts`,
  `fact_revisions`, `dependencies`, `dependency_support_groups`, `evidence`,
  `integrity_events`, `embeddings`, `memory_operations`, `retention_policies`,
  `schema_migrations`, and FTS5 index tables.
- **FR-003:** Foreign keys MUST enforce referential integrity between revisions,
  dependencies, support groups, and evidence.
- **FR-004:** Indexes MUST exist for Fact identity/lookups, revision history,
  reverse dependency lookup, dependency validation state, evidence lookups, queue
  status, and health filtering.
- **FR-005:** Migrations MUST be versioned, ordered, forward-only, transactional,
  and recorded in `schema_migrations` with checksum and applied timestamp.
- **FR-006:** Migration files MUST be SQL resources packaged with the engine and
  applied automatically or explicitly according to configuration.
- **FR-007:** The transaction API MUST guarantee rollback on thrown errors and MUST
  not allow nested independent commits that break atomicity.
- **FR-008:** Write transactions MUST be short; long traversal or model work MUST
  occur outside the write transaction.
- **FR-009:** Optimistic version checks MUST execute inside the same transaction as
  the Fact update to prevent lost updates.
- **FR-010:** `SQLITE_BUSY` and lock contention MUST surface as retryable structured
  errors, never silent corruption or unbounded blocking.
- **FR-011:** Prepared statements MUST be cached per connection with bounded size.
- **FR-012:** The database path MUST be configurable; GUM MUST NOT rely on a
  process-global singleton connection.
- **FR-013:** Startup MUST verify schema version compatibility and refuse to run
  with a newer unknown schema.
- **FR-014:** Startup MUST recover or mark incomplete durable operations according
  to their idempotency keys.
- **FR-015:** Backup/checkpoint operations MUST be documented and MUST preserve
  transactional consistency.
- **FR-016:** FTS5 virtual tables MUST be maintained in the same write flow as Fact
  create/update/delete, or by a durable outbox with detectable lag.
- **FR-017:** The engine MUST NOT run long synchronous scans on the main event loop;
  heavy database work MUST be chunked or moved to workers.
- **FR-018:** Schema changes MUST be additive-compatible where feasible; destructive
  changes require a migration with explicit data handling.
- **FR-019:** All timestamps MUST be stored in UTC; database CHECK constraints SHOULD
  enforce valid enum values where practical.
- **FR-020:** Database errors MUST be mapped to structured domain errors with SQLite
  codes preserved for diagnostics.

## 5. Key Entities

### Schema Migration

| Field | Notes |
|---|---|
| `version` | Integer, primary key |
| `name` | Human-readable migration name |
| `checksum` | SQL file checksum |
| `applied_at` | Timestamp |

### Core Tables

- `namespaces`: namespace identity and configuration
- `facts`: current Fact state and pointers
- `fact_revisions`: immutable snapshots, unique `(fact_id, version)`
- `dependencies`: directed edges with assumptions and validation state
- `dependency_support_groups`: required/alternative groups
- `evidence`: provenance and verification metadata
- `integrity_events`: audit trail and health transitions
- `embeddings`: canonical vectors and model metadata
- `memory_operations`: durable queue and maintenance state
- `retention_policies`: namespace retention configuration

## 6. Edge Cases & Error Handling

- Database locked beyond timeout → `DB_BUSY`, caller may retry with backoff.
- Disk full or read-only database → startup/runtime error with no partial commit.
- Corrupt database file → refuse writes, report integrity failure, do not attempt
  destructive repair automatically.
- Migration checksum mismatch → refuse to run; require operator resolution.
- Unknown newer schema version → refuse to open; prevents downgrade corruption.
- Foreign-key violation from a domain bug → transaction rolls back; error is logged
  with entity ids.
- Interrupted backup → leave original database untouched; report failure.
- Duplicate `(fact_id, version)` → unique constraint rejects; transaction rolls back.
- WAL growth → checkpoint policy configurable; monitor size.
- Two engine processes writing the same file → serialize through SQLite; document
  that single-writer contention is expected.

## 7. Non-Functional Requirements

- Reads MUST use indexed access paths for hot queries.
- Write transactions SHOULD target low-millisecond duration for normal operations.
- Startup integrity checks MUST be bounded and must not delay service indefinitely
  on large stores.
- The engine MUST expose database health metrics: size, WAL size, busy events,
  migration version, pending operations.
- Backups MUST be restorable and verifiable.

## 8. Success Criteria

- Killing the process at any point during a write never corrupts committed data.
- A failed migration leaves the prior schema fully operational.
- Foreign keys prevent orphan revisions, edges, and evidence links.
- Two processes cannot both commit conflicting next versions for one Fact.
- Backup restored to a new path passes integrity checks and preserves all
  revision chains and dependency edges.
- Reverse-dependency queries use the reverse index, confirmed by query plans.

## 9. Open Questions

- Which migration tooling is used, or is the runner custom? Initial answer: a small
  custom runner over numbered SQL files.
- Should FTS5 maintenance be synchronous or outbox-based? Initial answer: outbox
  with durable jobs, matching vector indexing semantics.
- What is the default busy timeout? Decide in implementation; must be configurable.
