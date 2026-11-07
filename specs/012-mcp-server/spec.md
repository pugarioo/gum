# Feature Specification: MCP Server

- **Feature ID:** 012
- **Module:** Interface
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §8, §8.4, §9, §10
- **Depends on:** 001-011, 016, 017
- **Blocks:** none

## 1. Overview

GUM exposes its capabilities to agent clients through an MCP server built on the
official TypeScript MCP SDK. The server is an adapter over the same domain services
used by the SDK, HTTP API, and CLI. It must not contain business logic and must not
start a frontend server.

The primary deployment is a local stdio server launched by an agent client and
terminated on disconnect.

## 2. Scope

### In scope

- stdio MCP server lifecycle
- Tool registration with input/output schemas
- Mapping all CONCEPT.md §9 operations to tools
- Structured errors
- Progressive disclosure responses and token-aware defaults
- stderr logging and stdout protocol purity
- Graceful shutdown

### Out of scope

- HTTP/SSE MCP transport (possible future work)
- Developer dashboard or UI
- Business logic duplication
- Background maintenance scheduling beyond exposed tools

## 3. User Scenarios & Testing

### P1 — Start and list tools

**Given** a configured database path
**When** an MCP client launches the server over stdio
**Then** the server initializes the engine, registers all supported tools, and
responds to `tools/list`
**And** writes logs only to stderr.

### P1 — Create and retrieve a Fact through MCP

**Given** a connected MCP client
**When** it calls `add_fact`, then `get_fact`
**Then** the tool responses reflect the same domain behavior as the SDK
**And** the Fact is persisted once, with one initial revision and audit event.

### P1 — Stale write error is structured

**Given** a Fact updated to version 2 by another client
**When** the MCP client calls `update_fact` with `expected_version: 1`
**Then** the response is a structured `STALE_WRITE` error with current version and
latest revision id
**And** no partial mutation occurs.

### P2 — Progressive disclosure defaults

**Given** a search with many candidates
**When** the client calls `search_facts`
**Then** the default response contains concise results with ids, statements, scope,
and health warnings
**And** details, history, and graph expansion require explicit follow-up calls.

### P2 — Tool schema validation

**Given** an invalid tool input
**When** the client invokes the tool
**Then** the server returns a validation error naming the offending field
**And** does not call the domain service.

### P2 — Namespace authorization

**Given** a client authorized for namespace A
**When** it calls a tool scoped to namespace B
**Then** the server returns `UNAUTHORIZED_SCOPE` and writes nothing.

### P3 — Graceful disconnect

**Given** a running server with pending durable operations
**When** the client disconnects
**Then** the server stops accepting requests, flushes safe state, closes the
database, and exits without corrupting pending jobs.

### P3 — Large result budget

**Given** a search or list request with a small token budget
**When** results exceed the budget
**Then** the server returns truncation metadata and never drops material health
warnings.

### P4 — Unsupported tool version

**Given** a client requests an unknown tool or unsupported schema version
**When** the server processes the request
**Then** it returns a structured unsupported-operation error without crashing.

## 4. Functional Requirements

- **FR-001:** The server MUST use the official TypeScript MCP SDK and communicate
  over stdio by default.
- **FR-002:** Every registered tool MUST have a Zod input schema and a documented
  output schema or response shape.
- **FR-003:** The server MUST expose the §9 operations: fact management, revision
  history, dependency graph, integrity/reconciliation, evidence, and administration.
- **FR-004:** Tools MUST delegate to shared domain services; the MCP layer MUST NOT
  implement persistence, graph traversal, or reconciliation logic.
- **FR-005:** The server MUST NOT start a Next.js, React, or any frontend server.
- **FR-006:** Protocol messages MUST be written to stdout only; diagnostics MUST go
  to stderr.
- **FR-007:** Errors MUST be structured with a stable code, message, and safe
  details: `VALIDATION_ERROR`, `NOT_FOUND`, `STALE_WRITE`, `STALE_RECONCILIATION`,
  `CYCLE_DETECTED`, `DEPENDENCY_MISSING`, `UNAUTHORIZED_SCOPE`,
  `GRAPH_INTEGRITY_ERROR`, `DB_BUSY`, `UNSUPPORTED_OPERATION`.
- **FR-008:** The server MUST validate inputs before invoking domain services and
  MUST reject unknown fields by default.
- **FR-009:** Default responses MUST be concise and token-aware; full history,
  evidence content, and graph expansion MUST require explicit parameters.
- **FR-010:** Health warnings MUST be included in search and fact responses whenever
  material, regardless of budget.
- **FR-011:** The server MUST enforce namespace authorization on every call.
- **FR-012:** The server MUST handle `SIGINT`/`SIGTERM` and client disconnect by
  closing cleanly without losing durable queued work.
- **FR-013:** The server MUST NOT promote retrieved text or evidence into system
  instructions or tool instructions.
- **FR-014:** Tool responses MUST be deterministic for identical inputs and state
  where domain behavior is deterministic.
- **FR-015:** The server MUST expose engine and index health through a tool without
  leaking secrets.
- **FR-016:** Tool names and shapes MUST match CONCEPT.md §9 exactly unless a
  documented versioned alias is introduced.
- **FR-017:** The server MUST support configuration through explicit config/env
  values and MUST NOT require a global installation.
- **FR-018:** Startup MUST fail clearly if the database cannot be opened or migrated.

## 5. Key Entities

### MCP Tool Registry

Grouped tools:

- Fact Management: `add_fact`, `update_fact`, `get_fact`, `delete_fact`,
  `search_facts`, `list_facts`
- Revision History: `get_revisions`, `get_revision`, `compare_revisions`,
  `inspect_history`
- Dependency Graph: `add_dependency`, `remove_dependency`, `get_dependencies`,
  `get_dependents`, `explain_dependency`, `inspect_graph`
- Integrity: `analyze_impact`, `validate_dependencies`, `get_affected_facts`,
  `reconcile_fact`, `get_integrity_report`
- Evidence: `attach_evidence`, `inspect_evidence`, `verify_evidence`, `list_evidence`
- Administration: `create_namespace`, `export_memory`, `import_memory`,
  `configure_retention`, `rebuild_index`

### MCP Error Envelope

`code`, `message`, `details` (safe), `retryable`, `request_id`.

## 6. Edge Cases & Error Handling

- Client sends malformed JSON-RPC → protocol-level error, server stays alive.
- Tool called before initialization completes → `NOT_READY`, retryable.
- Database locked → `DB_BUSY`, retryable with backoff guidance.
- Unknown namespace or Fact → `NOT_FOUND` with no data leakage.
- Oversized input → validation error before domain call.
- Tool handler throws unexpectedly → sanitized internal error, logged with request id,
  server remains alive.
- Client disconnects mid-call → handler completes or aborts safely; no partial write.
- stdout contamination from a dependency → must be caught in tests; logs via stderr.
- Two MCP clients share one database → SQLite single-writer semantics apply; errors
  are structured, not crashes.

## 7. Non-Functional Requirements

- Startup SHOULD be fast enough for on-demand client launch.
- Tool calls MUST not block the event loop on CPU-heavy embedding or long scans.
- Responses MUST respect client budgets and remain under configured maximum size.
- The server MUST be testable through an in-memory MCP transport harness.
- No secrets in logs or error details.

## 8. Success Criteria

- `tools/list` returns all specified tools with valid schemas.
- All tool calls produce identical domain outcomes to direct SDK calls.
- stdout contains only MCP protocol messages in integration tests.
- Invalid input never reaches domain services.
- Disconnect and restart preserve all committed data and pending durable jobs.
- No frontend process or HTTP listener is started by the stdio server.

## 9. Open Questions

- Should an HTTP/SSE MCP transport be added later? Not in current scope.
- Should tool responses support a `format: "compact" | "full"` flag in addition to
  explicit expansion parameters? Consider during implementation.
