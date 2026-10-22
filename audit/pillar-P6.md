# P6 — Operability: apex-mcp

## Scope inspected

- `apex_mcp/audit.py` — local audit log
- `apex_mcp/tools/setup_tools.py` — `apex_health_check`, `apex_get_audit_log`
- `apex_mcp/db.py` — `health_metrics`
- `apex_mcp/server.py` — lifespan logging

## What was found

An audit log exists and writes JSON Lines to `~/.apex_mcp/audit.jsonl`. It records tool name, status, app_id, duration, and error. However, it is local-only, best-effort, and can be deleted or modified by the OS user. There is no off-box shipping, signing, or integrity protection.

`apex_health_check` returns a simple pass/warn/fail JSON covering connectivity, workspace access, template IDs, session state, and configuration. It does not expose Prometheus-style metrics.

There are no structured logs with correlation IDs, no rate limiting, no alert thresholds, and no runbooks.

## What is done well

1. **Health check exists** and is exposed as a tool.
2. **Audit log records tool invocations** with timestamps.
3. **Connection health metrics** (`connected`, `dry_run`, `batch_mode`, `col_cache_size`) are available programmatically (`db.py` lines 378-393).

## What could not be verified

- Whether any consumer ships the audit log to a SIEM.
- Actual log retention and backup practices.

## Maturity score

**P6: 1 / 5** — Basic local observability; no enterprise operability controls.
