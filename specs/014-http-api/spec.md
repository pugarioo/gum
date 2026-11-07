# Feature Specification: HTTP API

- **Feature ID:** 014
- **Module:** Interface
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §8, §8.4, §9, §10
- **Depends on:** 001-011, 013, 016, 017
- **Blocks:** 015
- **Blocked by:** none

## 1. Overview

An optional Fastify HTTP service exposes GUM to networked consumers. It is
headless, disabled by default in local/stdio deployments, and an adapter over the
same domain services as the SDK and MCP server. Starting it must never start a
frontend application.

Because HTTP exposes memory beyond a local process, authentication, namespace
authorization, rate limits, and safe error mapping are mandatory.

## 2. Scope

### In scope

- Optional Fastify server with explicit enablement
- JSON endpoints for all §9 operations
- Request/response validation
- Authentication and namespace authorization hooks
- Structured errors and HTTP status mapping
- Health/readiness endpoints
- Streaming for large export/import where practical

### Out of scope

- Developer dashboard or UI
- Live cross-machine synchronization
- Browser CORS defaults for arbitrary origins
- GraphQL
- Multi-tenant billing or user management

## 3. User Scenarios & Testing

### P1 — Start only when enabled

**Given** default configuration
**When** the engine starts
**Then** no HTTP listener is created.
**Given** HTTP is explicitly enabled with a port and auth configuration
**When** the service starts
**Then** it listens on the configured interface and exposes validated endpoints.

### P1 — Fact CRUD over HTTP

**Given** an authorized token scoped to namespace A
**When** the client calls `POST /v1/facts` and `GET /v1/facts/:id`
**Then** the same domain behavior as the SDK applies
**And** responses use stable JSON field names and status codes.

### P1 — Unauthorized scope denied

**Given** a token for namespace A
**When** it requests a Fact in namespace B
**Then** the service returns `403` with `UNAUTHORIZED_SCOPE`
**And** no data is leaked in the body.

### P2 — Stale write conflict

**Given** a Fact at version 2
**When** the client sends an update with `expectedVersion: 1`
**Then** the service returns `409` with `STALE_WRITE`, current version, and latest
revision id.

### P2 — Validation failure

**Given** an invalid or unknown-field request body
**When** it is submitted
**Then** the service returns `400` with field-level validation details
**And** does not call the domain service.

### P2 — Search budgets

**Given** a search request with `limit` and `tokenBudget`
**When** results exceed the budget
**Then** the response includes truncation metadata and preserves material health
warnings.

### P3 — Health and readiness

**Given** a running service
**When** `GET /health` is called
**Then** it returns process liveness, database status, migration version, index lag,
and pending operation counts
**And** readiness fails when the database is unavailable.

### P3 — Rate limiting

**Given** a configured request limit
**When** a client exceeds it
**Then** the service returns `429` with retry guidance
**And** does not execute the operation.

### P4 — Export streaming

**Given** a large namespace export
**When** the client requests export
**Then** the service streams a bundle with manifest and checksums rather than
loading all data into memory.

## 4. Functional Requirements

- **FR-001:** The HTTP service MUST be optional and disabled by default; enabling it
  MUST require explicit configuration.
- **FR-002:** The service MUST use Fastify and call shared domain services; it MUST
  NOT duplicate business logic.
- **FR-003:** The service MUST NOT serve or start any frontend or dashboard.
- **FR-004:** Endpoints MUST cover all §9 operations under a versioned path (`/v1`).
- **FR-005:** Every request and response MUST be schema-validated; unknown fields
  MUST be rejected by default.
- **FR-006:** Authentication MUST be required for non-health endpoints in networked
  mode; no anonymous writes.
- **FR-007:** Namespace authorization MUST be enforced for every operation, including
  search, export, and graph traversal.
- **FR-008:** HTTP status mapping MUST be stable: `400` validation, `401`
  unauthenticated, `403` unauthorized scope, `404` not found, `409` stale/conflict,
  `422` graph/domain rule violation, `429` rate limited, `503` unavailable.
- **FR-009:** Error bodies MUST include a stable code, message, safe details, and
  request id; they MUST NOT expose SQL or internal paths.
- **FR-010:** Health and readiness endpoints MUST be unauthenticated only when
  explicitly configured; readiness MUST reflect database and migration state.
- **FR-011:** The service MUST enforce request size and token/result budgets.
- **FR-012:** The service MUST support request ids and structured access logs to
  stderr or the configured logger, never to protocol responses.
- **FR-013:** Long-running export/import SHOULD support streaming or chunked
  responses; interruption MUST leave no partial domain writes.
- **FR-014:** The service MUST apply rate limits per identity and namespace.
- **FR-015:** The service MUST NOT promote retrieved text or evidence into
  instructions or executable behavior.
- **FR-016:** TLS termination MAY be delegated to a reverse proxy, but the service
  MUST document the trust boundary for forwarded headers.
- **FR-017:** The service MUST expose engine status without leaking secrets or full
  evidence content.
- **FR-018:** Graceful shutdown MUST stop accepting new requests, finish in-flight
  work within a timeout, and close the database safely.

## 5. Key Entities

### HTTP Endpoint Groups

- `/v1/facts` — create, read, update, delete, list, search
- `/v1/facts/:id/revisions` — history and comparison
- `/v1/dependencies` — add, remove, inspect, explain, dependents
- `/v1/integrity` — impact, validation, affected facts, report
- `/v1/reconciliations` — reconcile Fact
- `/v1/evidence` — attach, inspect, verify, list
- `/v1/namespaces` — create/list
- `/v1/admin` — export, import, retention, rebuild index
- `/health`, `/ready`

### Error Envelope

`code`, `message`, `details`, `requestId`, `retryable`.

## 6. Edge Cases & Error Handling

- Missing or invalid token → `401`, no domain call.
- Token valid but namespace unauthorized → `403`.
- Request body too large → `413` with limit details.
- Malformed JSON → `400`.
- Database busy/locked → `503` with retry hint.
- Domain conflict (stale, cycle, reconciliation) → `409` or `422` with code.
- Unknown route → `404` JSON error, no HTML.
- Slow client during streaming export → backpressure handling; do not buffer entire
  export.
- Service disabled → no port open; attempting to connect fails normally.
- Duplicate idempotency key → return prior result where supported.
- Proxy headers untrusted → ignore unless explicitly configured trust.

## 7. Non-Functional Requirements

- Request handling MUST be non-blocking; heavy work delegated to workers.
- Search and list responses MUST respect token/result budgets.
- Access logs MUST be structured and redact secrets and sensitive content.
- The service MUST support graceful restart without corrupting the database.
- Startup MUST fail fast on invalid auth or schema configuration.
- Documentation MUST cover deployment, auth, and trust boundaries.

## 8. Success Criteria

- Default startup opens no HTTP port.
- All §9 operations are reachable over HTTP with documented schemas.
- Cross-namespace access always returns `403` and no data.
- Error codes are stable across SDK, MCP, and HTTP for the same domain condition.
- Service passes concurrency tests under SQLite single-writer semantics.
- No frontend assets or dashboard routes exist.

## 9. Open Questions

- What authentication mechanism is canonical? Initial answer: pluggable bearer-token
  verifier with namespace claims; OAuth is deployment-specific.
- Should the HTTP API be a separate package or a subpath export? Initial answer:
  separate optional package to keep the core headless.
- Should SSE/WebSocket be added for live events? Not in current scope.
