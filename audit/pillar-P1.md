# P1 — Architecture / Structure: apex-mcp

## Scope inspected

- `apex_mcp/server.py` — FastMCP server, tool registration, CLI args, resources
- `apex_mcp/db.py` — `ConnectionManager` singleton
- `apex_mcp/session.py` — `ImportSession` singleton
- `apex_mcp/ids.py` — ID generator
- `apex_mcp/config.py` — env var loading
- `apex_mcp/tools/*.py` — 17 tool modules
- `pyproject.toml`, `README.md`, `CLAUDE.md`

## What was found

The codebase is modular on the surface: tools are grouped by category into separate modules under `apex_mcp/tools/`, and the core concerns (DB, session, IDs, config, validation, utils) are separated. However, the runtime architecture relies on module-level singletons (`db = ConnectionManager.get()`, `session = ImportSession()`, `ids = IdGenerator()`) that are shared across all callers. This makes the HTTP/SSE transports inherently single-user and unsafe for concurrent clients.

Tool registration in `server.py` is a flat namespace with no role-based visibility. Every tool executes against the same Oracle connection profile, so there is no architectural boundary between a read-only observer and a destructive operator.

The lifecycle (`apex_connect` → `apex_create_app` → mutations → `apex_finalize_app`) is tracked in the global `ImportSession`. While this works for a single stdio client, it is a liability under the multi-tenant SaaS assumption.

## What is done well

1. **Dry-run mode** — `db.enable_dry_run()` and `apex_dry_run_preview()` allow operators to review generated PL/SQL before execution (`db.py` lines 230-242, `app_tools.py` lines 700-736).
2. **Batch mode** — `begin_batch` / `commit_batch` queues PL/SQL and can roll back on error, reducing round-trips and partial-state risk (`db.py` lines 302-363).
3. **Auto-reconnect on transient Oracle errors** — A defined set of ORA codes triggers retry with exponential back-off (`db.py` lines 21-40, 111-135).

## What could not be verified

- Actual runtime behaviour of FastMCP HTTP transport under concurrent load (no integration tests for concurrency).
- Whether any consumer deploys the server behind an authenticated reverse proxy.

## Maturity score

**P1: 2 / 5** — Partially present, documented only in README limitations, not enforced architecturally.
