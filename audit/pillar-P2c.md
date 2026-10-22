# P2c — Command Execution / Code Generation Risks: apex-mcp

## Scope inspected

- `apex_mcp/server.py` — tool annotations, transport
- `apex_mcp/tools/sql_tools.py` — `apex_run_sql`
- `apex_mcp/tools/app_tools.py` — app create/delete/export
- `apex_mcp/tools/component_tools.py` — regions/items/buttons/processes/dynamic actions
- `apex_mcp/tools/shared_tools.py` — LOVs, auth schemes, app items/processes
- `apex_mcp/tools/generator_tools.py` — CRUD/dashboard/login generators
- `apex_mcp/tools/inspect_tools.py` — direct table UPDATE/DELETE
- `apex_mcp/tools/user_tools.py` — APEX user creation
- `apex_mcp/tools/ui_tools.py`, `visual_tools.py`, `chart_tools.py` — generated `EXECUTE IMMEDIATE`
- `apex_mcp/validators.py` — what is validated
- `apex_mcp/utils.py` — `_esc`, `_blk`, `_sql_to_varchar2`

## What was found

The most critical issue is `apex_run_sql`: it executes any statement that does not start with `SELECT` or `WITH` as a PL/SQL anonymous block, with no allowlist, no row-level authorization, and no separation between read and write paths (`sql_tools.py` lines 96-111). The `startswith` check is trivially bypassed by leading whitespace or comments.

PL/SQL generation across the tool suite uses string interpolation. `_esc()` only escapes single quotes; it does not validate identifiers, reject SQL comments, or use bind variables for dynamic table/column names. This creates prompt-injection and SQL-injection risk in generated APEX components.

Tool annotations (`destructiveHint`, `readOnlyHint`) are advisory only. The server does not enforce them; it depends entirely on the AI client to request user confirmation.

`apex_fix_permissions` can grant `SELECT, UPDATE, DELETE` on APEX internal tables to the current user (`setup_tools.py` lines 471-474), increasing privilege escalation risk.

`apex_export_app` writes full APEX exports to arbitrary filesystem paths (`app_tools.py` lines 561-564).

## What is done well

1. **Validators exist for page/app IDs, item names, table names, colors, chart types** (`validators.py`). `validate_table_name` restricts to Oracle identifier characters.
2. **Dry-run and batch rollback** provide a safety net for interactive use.
3. **`eval()` and `document.write()` warnings** in dynamic actions show awareness of unsafe JS patterns (`component_tools.py` lines 817-825).

## What could not be verified

- Whether any downstream consumer applies additional sandboxing or network segmentation.
- Real-world prompt-injection success rate against the generators.

## Maturity score

**P2c: 1 / 5** — Ad-hoc; arbitrary code execution path exists and is central to the product.
