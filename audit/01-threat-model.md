# P1 — Threat Model: apex-mcp

## Data flow and trust boundaries

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Trust boundary 1: AI client / IDE                                           │
│  - Claude Code, Cursor, VS Code, OpenAI Agents SDK, Gemini CLI              │
│  - Holds .mcp.json with Oracle credentials (env section)                    │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ MCP (stdio / HTTP / SSE)
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Trust boundary 2: apex-mcp server process                                   │
│  - Single-user ImportSession singleton                                      │
│  - Reads DB_USER, DB_PASS, WALLET_PASS, WORKSPACE_ID from env               │
│  - Generates PL/SQL and executes via oracledb                               │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ oracledb thin + mTLS wallet
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Trust boundary 3: Oracle ADB / APEX workspace                               │
│  - wwv_flow_imp_page / wwv_flow_imp_shared APIs                             │
│  - APEX application metadata tables (WWV_FLOW_*)                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Assets worth protecting (ranked)

1. **Oracle database credentials** — DB password, wallet password, workspace schema access. Loss = full APEX workspace compromise.
2. **APEX applications under construction** — Business logic, data models, auth schemes; destructive tools can drop apps/pages/components.
3. **Database data** — `apex_run_sql` can read arbitrary rows and execute DML/DDL once connected.
4. **Audit trail integrity** — Local `~/.apex_mcp/audit.jsonl` can be deleted/modified by the OS user.
5. **MCP session state** — Single-user singleton; concurrent users corrupt each other's import sessions.

## Attack surface

| Surface | Location | Risk |
|---|---|---|
| stdio transport | Local AI client | Low if client is trusted; credential exposure to client process |
| HTTP transport (`--transport streamable-http`) | `server.py:647` | No auth, no TLS by default; any process on host can call tools |
| SSE transport (`--transport sse`) | `server.py:650` | Same as HTTP |
| `apex_run_sql` tool | `tools/sql_tools.py:72` | Arbitrary SELECT/DML/DDL with only a `startswith` check |
| `apex_create_user` | `tools/user_tools.py:9` | Creates APEX workspace users; password passed in clear |
| `apex_fix_permissions` | `tools/setup_tools.py:436` | Self-grants on APEX internal tables if run as privileged user |
| `apex_export_app` | `tools/app_tools.py:479` | Writes full APEX export to arbitrary filesystem path |
| `apex_generate_rest_endpoints` | `tools/devops_tools.py` | Creates ORDS endpoints; not inspected in depth |
| MCP Resources | `server.py:400-518` | Expose config (with DB user/DSN), session, schema, apps |
| `.mcp.json` | Project root | Example file contains plaintext credential placeholders |

## STRIDE-classified threats

| ID | Threat | STRIDE | Boundary | Current control (if any) |
|---|---|---|---|---|
| T-01 | Attacker on same host calls HTTP/SSE tools without authentication | Spoofing | AI client → Server | Binds to `127.0.0.1` by default |
| T-02 | Malicious/compromised AI client instructs destructive operations via MCP | Elevation / Tampering | AI client → Server | Tool annotations (`destructiveHint`) only |
| T-03 | `apex_run_sql` executes arbitrary DML/DDL, dropping data or apps | Tampering / Elevation | Server → Oracle | `startswith` SELECT/WITH check bypassable |
| T-04 | AI-generated SQL/PL/SQL injected into generated APEX components | Injection | Server → Oracle | `_esc()` only escapes single quotes; no bind variables for dynamic identifiers |
| T-05 | Credentials in `.mcp.json` committed to source control | Information Disclosure | Host filesystem | `.mcp.json.example` documents the pattern |
| T-06 | Wallet password / DB password exposed in process env or logs | Information Disclosure | Server process | Env vars loaded at import; `apex_check_requirements` masks first 3 chars only |
| T-07 | Single-user session singleton corrupted by concurrent HTTP clients | Tampering / Denial | AI client → Server | Documented limitation only |
| T-08 | APEX export written to attacker-controlled path | Tampering / Info Disclosure | Server → Filesystem | No path validation |
| T-09 | Audit log on local disk deleted/modified by host user | Repudiation | Server → Filesystem | Best-effort append-only, no off-box ship |
| T-10 | Dependency confusion / supply-chain compromise of `fastmcp`/`oracledb` | Tampering | Build/Runtime | `>=` pinning, no lockfile |
| T-28 (APEX-specific) | APEX app generated without authorization schemes or with weak auth, exposing created apps | Elevation / Tampering | Generated APEX app → End users | `apex_add_auth_scheme` exists but auth is not enforced by default |

## Key assumptions

- The AI client is treated as fully trusted in the current design.
- HTTP/SSE transport is intended for localhost reverse-proxy scenarios.
- The connected Oracle user has broad APEX API privileges (WWV_FLOW_IMP, APEX_UTIL, internal table DML).
