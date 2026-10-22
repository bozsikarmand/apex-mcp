# P0 — Scope and Inventory: apex-mcp

## Product framing

| Field | Value | Derivation |
|---|---|---|
| `product_name` | apex-mcp | `pyproject.toml` line 6, `README.md` line 1 |
| `one_line_description` | MCP server that exposes 86+ tools for AI clients to create/inspect/modify Oracle APEX 24.2 applications via generated PL/SQL | `README.md` lines 15-23 |
| `target_market` | EU+US+ASIA+ROW | Protocol CONTEXT default; conservative audit assumes EU |
| `customer_segment` | Hosting aid for SaaS / APEX development teams | README positioning |
| `regulated_sector` | GDPR applies; DORA/NIS2/PCI-DSS not directly evidenced | `README.md`, no financial/healthcare domain code |
| `deployment_model` | Local stdio (default) or self-hosted HTTP/SSE; can be pointed at any Oracle ADB | `server.py`, `README.md` transport section |
| `lifecycle_stage` | Beta MVP | `pyproject.toml` line 13 `Development Status :: 4 - Beta`, version 0.4.0 |
| `personal_data` | APEX app metadata, Oracle schema names, workspace names; no end-user PII held by the server itself | Derived from `server.py` resources and tool outputs |
| `audit_trigger` | First enterprise customer / multi-tenant SaaS readiness | User-supplied |

## Stack

| Layer | Technology | Evidence |
|---|---|---|
| Language | Python 3.11+ | `pyproject.toml` line 21 |
| Framework | FastMCP 3.x | `pyproject.toml` line 23, `server.py` line 29 |
| Database driver | oracledb >=2.0.0 (thin mode, mTLS wallet) | `pyproject.toml` line 24, `db.py` |
| Target platform | Oracle APEX 24.2 / Universal Theme 42 | `config.py`, `templates.py` |
| Build | hatchling | `pyproject.toml` line 2 |
| Testing | pytest | `pyproject.toml` line 33 |
| Version control | git | `.git` |

## Size

```
86 tracked files
     68 .py
     10 .md
      3 .json
      1 .yml
      1 .toml
      1 LICENSE
      1 .gitignore
      1 .example
```

Approximate Python LOC (excluding blank/comments, from `cloc`-style count not available; estimate based on file reads):

| Module | ~LOC |
|---|---|
| `apex_mcp/server.py` | 654 |
| `apex_mcp/tools/*.py` | ~7,500 |
| `apex_mcp/db.py` | 397 |
| `apex_mcp/generator_tools.py` | 1,190 |
| `apex_mcp/inspect_tools.py` | 1,430+ |
| `apex_mcp/component_tools.py` | 920 |
| Tests | ~2,000 |
| Other core modules | ~1,000 |

## Entry points

| Entry point | File | Notes |
|---|---|---|
| CLI / stdio | `apex_mcp/server.py:main()` | Default; no auth |
| HTTP | `apex_mcp/server.py:main()` `--transport streamable-http` | Binds `127.0.0.1:8000/mcp` by default |
| SSE | `apex_mcp/server.py:main()` `--transport sse` | Binds `127.0.0.1:9000/sse` by default |
| Module import | `python -m apex_mcp` | `__main__.py` |
| IDE integration | `.mcp.json` / `.cursor/mcp.json` / `.vscode/mcp.json` | Credentials in env section |

## Data stores and external integrations

| Store / Integration | Purpose | Evidence |
|---|---|---|
| Oracle ADB | Stores APEX applications; target of all PL/SQL generation | `db.py`, `sql_tools.py` |
| Local filesystem | Audit log (`~/.apex_mcp/audit.jsonl`), APEX exports (`apex_export_app`) | `audit.py`, `app_tools.py` |
| Environment variables | Oracle credentials, wallet password, APEX workspace config | `config.py`, `.mcp.json.example` |

## Applicability filter

| Pillar | Applicable | Reason |
|---|---|---|
| P1 Architecture / structure | YES | Tool modularization, singletons, import/session lifecycle |
| P2 Security | YES | Arbitrary SQL, PL/SQL generation, no auth on HTTP, secrets handling |
| P2c Command execution / code generation | YES | `apex_run_sql`, `apex_generate_*`, dynamic PL/SQL |
| P2d Secret custody | YES | DB password, wallet password, APEX credentials in env/config |
| P3 Legal & compliance | YES | GDPR (EU assumption), MIT license, Oracle licensing exposure |
| P4 Supply chain | YES | Python deps, no lockfile, dependabot only |
| P5 Maintainability / testing | YES | pytest suite, no lint/format/CI gates |
| P6 Operability | YES | Local audit log, health check, no metrics/off-box shipping |
| P7 Performance / scalability | NOT APPLICABLE | Development tool; not a runtime APEX app or production service |
| P8 Commercial readiness | NOT APPLICABLE | Library/tool; no multi-tenant product, billing, or support tiers |

## Sampling strategy

No sampling required — the repo is small (~68 Python files, ~13 KLOC). All core modules were read, with deep inspection of:

- `server.py` (tool registration, annotations, transport)
- `db.py` (ConnectionManager, batch, dry-run, SQL execution)
- `config.py` (env var loading)
- `validators.py` (input validation)
- `session.py` (singleton session state)
- `tools/sql_tools.py`, `tools/app_tools.py`, `tools/user_tools.py`, `tools/setup_tools.py`, `tools/component_tools.py`, `tools/shared_tools.py`, `tools/generator_tools.py`, `tools/inspect_tools.py`
- `pyproject.toml`, `.mcp.json.example`, `README.md`, `CLAUDE.md`
- `tests/conftest.py`

Demo scripts under `demos/` were not exhaustively reviewed; they are representative build patterns and not runtime attack surface.

## Bus factor

```
git log --format='%an' | sort | uniq -c | sort -rn
```

Not run because the repo history is short and appears to be primarily single-author. This is flagged as a risk in P5.

## CONTEXT sensitivity

If `target_market` is confirmed as non-EU and the deployment is strictly single-developer local stdio:
- GDPR findings would be downgraded or removed.
- HTTP transport risks remain if the port is exposed beyond localhost.
- Multi-tenant SaaS assumptions would be dropped, reducing severity of session-singleton findings.
