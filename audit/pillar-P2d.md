# P2d — Secret Custody: apex-mcp

## Scope inspected

- `apex_mcp/config.py` — env var loading
- `apex_mcp/db.py` — connection kwargs
- `apex_mcp/tools/setup_tools.py` — requirement checks, masking
- `.mcp.json.example` — documented credential pattern
- `README.md` — setup instructions
- `launch-nexus.sh` — how nexus consumes the server

## What was found

Secrets are loaded from environment variables at import time and held as module-level globals (`config.py` lines 13-26). The documented configuration pattern stores `ORACLE_DB_PASS`, `ORACLE_WALLET_PASSWORD`, and workspace identifiers in `.mcp.json` (`README.md` lines 75-95, `.mcp.json.example` lines 7-16). This pattern is prone to accidental git commits and IDE/AI client log capture.

`apex_check_requirements` masks secrets as `val[:3] + ***`, which leaks almost the entire value for short secrets (`setup_tools.py` lines 243-252).

There is no integration with a secrets manager, no short-lived tokens, and no credential rotation support.

`launch-nexus.sh` fetches the workspace password by grepping `terraform.tfvars` inside an orchestrator container and exports it as `ORACLE_DB_PASS`. This couples apex-mcp secret custody to the orchestrator's state file and is outside the apex-mcp repo's control.

## What is done well

1. **No secrets are hardcoded in source files** — only placeholders in `.mcp.json.example`.
2. **Audit log path is configurable** via `APEX_MCP_AUDIT_DIR` (`audit.py` line 17).

## What could not be verified

- Whether consumers use OS keyrings or secret managers in practice.
- Whether `.mcp.json` is actually in `.gitignore` in consumer projects.

## Maturity score

**P2d: 1 / 5** — Basic env-var storage; no enterprise secret custody.
