# Feature Specification: CLI

- **Feature ID:** 015
- **Module:** Interface
- **Status:** Draft
- **Created:** 2026-11-08
- **Source:** `CONCEPT.md` §8, §9, §13
- **Depends on:** 001-014
- **Blocks:** none

## 1. Overview

The TypeScript command-line package gives operators and developers a scriptable
interface to GUM without writing code. The CLI uses the embedded SDK by default and
may target an HTTP endpoint when configured. It must be safe by default: destructive
or irreversible operations require explicit confirmation, and machine-readable
output is stable.

## 2. Scope

### In scope

- Commands for facts, revisions, dependencies, integrity, evidence, and admin
- Embedded database access by default; optional HTTP endpoint mode
- JSON and human-readable table output
- Exit codes suitable for automation
- Configuration discovery and explicit flags
- Dry-run and confirmation for destructive actions
- Diagnostic commands for engine and index health

### Out of scope

- Interactive TUI or dashboard
- Background daemon management beyond explicit commands
- Shell completion in the first release (may be added)
- Business logic duplication

## 3. User Scenarios & Testing

### P1 — Add and read a Fact

**Given** a configured database and namespace
**When** the operator runs `gum fact add --namespace project:shopflow --category architecture --statement "..."`
**Then** the Fact is created through the SDK and its id is printed
**And** `gum fact get <id>` returns the current Fact.

### P1 — Search facts

**Given** a populated store
**When** the operator runs `gum search "PostgreSQL" --namespace project:shopflow --limit 5`
**Then** concise results are printed with ids, statements, and health warnings.

### P1 — Destructive action requires confirmation

**Given** `gum fact delete <id>`
**When** the command would permanently delete data
**Then** the CLI requires `--yes` or an interactive confirmation
**And** refuses non-interactively without it.

### P2 — JSON output for automation

**Given** any read command
**When** `--format json` is supplied
**Then** stdout contains a single valid JSON document (or JSONL where documented)
**And** diagnostics go to stderr.

### P2 — Stale write surfaces non-zero exit

**Given** an update with `--expected-version 1` while current is 2
**When** the command runs
**Then** the CLI prints a structured `STALE_WRITE` error and exits non-zero
**And** does not prompt to force an unsafe overwrite.

### P2 — Dependency operations

**Given** two Facts
**When** the operator runs `gum dep add F002 F001 --reason "..."` and
`gum dep dependents F001`
**Then** the edge is created and reverse lookup shows `F002`.

### P3 — Integrity report

**Given** a store with flagged Facts
**When** the operator runs `gum integrity report --namespace ...`
**Then** the CLI prints counts by health state and stale assumptions.

### P3 — Import/export

**Given** a bundle path
**When** the operator runs `gum export --namespace ... --out bundle.gum` and
`gum import bundle.gum --dry-run`
**Then** export writes a validated bundle and dry-run reports what would change
without writing.

### P4 — HTTP mode

**Given** `--endpoint https://gum.internal` and a token
**When** any command runs
**Then** it calls the HTTP API instead of opening a local database.

## 4. Functional Requirements

- **FR-001:** The CLI MUST be a TypeScript package with a stable command grammar
  and built-in help for every command.
- **FR-002:** The CLI MUST use the SDK for embedded mode and the HTTP API for
  endpoint mode; it MUST NOT duplicate domain logic.
- **FR-003:** Commands MUST cover Fact CRUD/search/list, revisions, dependencies,
  integrity, reconciliation, evidence, namespaces, export/import, retention, and
  index rebuild.
- **FR-004:** Read commands MUST support `--format table|json|jsonl` and write only
  the requested data to stdout.
- **FR-005:** Errors MUST print a stable code, message, and safe details to stderr
  and exit non-zero according to a documented exit-code mapping.
- **FR-006:** Destructive commands MUST require `--yes` or interactive confirmation
  when a TTY is available; non-interactive runs without `--yes` MUST fail.
- **FR-007:** Destructive and maintenance commands MUST support `--dry-run` where
  meaningful.
- **FR-008:** The CLI MUST support configuration from flags, environment variables,
  and a config file with documented precedence (flags > env > file > defaults).
- **FR-009:** The CLI MUST never print secrets or full evidence content by default.
- **FR-010:** Health/status commands MUST report database version, migration version,
  index lag, pending operations, and namespace summary.
- **FR-011:** The CLI MUST handle SIGINT gracefully and must not leave partial writes.
- **FR-012:** Output MUST be deterministic for identical data and flags, excluding
  timestamps where the command explicitly reports current time.
- **FR-013:** The CLI MUST NOT start a frontend or long-lived server process unless
  the operator explicitly runs a serve command backed by the HTTP module (if provided).
- **FR-014:** Commands MUST validate arguments and reject unknown flags by default.
- **FR-015:** The CLI MUST provide a version command and must report incompatible
  database or bundle versions clearly.
- **FR-016:** Batch inputs SHOULD be supported via JSONL for add/import workflows.
- **FR-017:** Exit codes MUST distinguish usage error, validation error, not found,
  conflict/stale, authorization, and internal/database failure.

## 5. Key Entities

### Command Groups

`gum fact`, `gum search`, `gum revision`, `gum dep`, `gum integrity`, `gum reconcile`,
`gum evidence`, `gum namespace`, `gum export`, `gum import`, `gum retention`,
`gum index`, `gum status`, `gum version`.

### Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 2 | Usage or argument error |
| 3 | Validation error |
| 4 | Not found |
| 5 | Conflict or stale write |
| 6 | Authorization failure |
| 7 | Database or internal failure |

## 6. Edge Cases & Error Handling

- No database path configured → clear configuration error, no interactive guess.
- Database locked → `DB_BUSY` with retry guidance and non-zero exit.
- Unknown Fact or namespace → `NOT_FOUND`, non-zero exit.
- Piped output with TTY prompts → never prompt; require `--yes`.
- Broken pipe (`| head`) → exit without stack trace noise.
- Invalid bundle → report validation failures, write nothing.
- HTTP endpoint unreachable → network error with retry guidance.
- Interrupt during export → leave no partial bundle or clearly marked incomplete file.
- Unknown command or flag → usage help and exit code `2`.
- Version mismatch between CLI and database schema → refuse unsupported operations.

## 7. Non-Functional Requirements

- CLI startup SHOULD be fast enough for interactive use.
- Long operations MUST show progress on stderr when interactive, not stdout.
- JSON output MUST be machine-parseable with no decorative text.
- The CLI MUST work on Linux, macOS, and Windows terminals.
- The package MUST have no frontend dependencies.

## 8. Success Criteria

- All §9 operations are reachable through CLI commands.
- JSON output parses successfully in automation for every read command.
- Destructive commands cannot run non-interactively without explicit consent.
- Error exit codes are consistent with the documented mapping.
- `--dry-run` makes no writes.
- CLI and SDK produce identical domain outcomes for the same operation.

## 9. Open Questions

- Should the CLI bundle interactive prompts by default? Initial answer: yes, only
  when stdout/stderr are a TTY.
- Should batch mutate commands use a single transaction or per-item transactions?
  Initial answer: per-item with a final report, unless `--atomic` is explicit.
