# Enterprise Readiness Audit Report — apex-mcp

## 1. Executive summary

apex-mcp is a FastMCP-based server that exposes 86+ tools for AI clients to build and modify Oracle APEX 24.2 applications. Under the strictest assumptions — multi-tenant SaaS, public internet exposure, GDPR — the current codebase is not enterprise-ready. The architecture is single-user, the HTTP/SSE transports have no authentication, and the flagship `apex_run_sql` tool allows arbitrary DML/DDL. Secrets are stored in local environment variables and `.mcp.json`. The maturity score is **28.8%** after proportional redistribution of N/A pillars. Two critical findings dominate the risk profile: unauthenticated HTTP access and arbitrary SQL execution. The shortest path to a sellable state requires adding transport authentication, separating read-only from privileged tools, integrating a secrets manager, and implementing GDPR data-handling documentation. Effort order of magnitude: 2–3 months of focused security and operability work.

## 2. Scope and method

**Examined:**
- All Python source files under `apex_mcp/` (core modules and 17 tool modules)
- `pyproject.toml`, `.github/dependabot.yml`, `.mcp.json.example`
- `README.md`, `CLAUDE.md`
- `tests/conftest.py` and test file inventory
- `launch-nexus.sh` (nexus integration entry point)

**Not examined:**
- Full execution of integration tests (no live Oracle connection available)
- Consumer deployments or reverse-proxy configurations
- Git history beyond basic inventory

**Method:** Static code review against the protocol's 0–5 rubric, with evidence recorded in `findings.jsonl`. Conservative defaults applied for GDPR and multi-tenant SaaS assumptions.

## 3. Threat model summary

Trust boundaries: AI client → MCP server → Oracle ADB/APEX. The AI client is treated as fully trusted, yet it holds plaintext credentials. The MCP server exposes all tools over unauthenticated HTTP/SSE and executes arbitrary SQL against the database. Assets at risk include Oracle credentials, APEX applications, database data, and audit integrity. Key threats: unauthenticated tool invocation (T-01), arbitrary SQL/DML (T-03), AI-driven destructive operations (T-02), credential leakage (T-05/T-06), and weak generated APEX auth (T-28).

## 4. Maturity scoring

**Pillars excluded:** P7 Performance (not runtime), P8 Commercial readiness (library/tool).
**Applicable weight total:** 84 (original 100 minus P7 8 and P8 8).

| Pillar | Raw weight | Redistributed weight | Score (0–5) | Weighted contribution |
|---|---:|---:|---:|---:|
| P1 Architecture | 15 | 17.86% | 2 | 7.14% |
| P2 Security | 20 | 23.81% | 1 | 4.76% |
| P3 Legal & compliance | 15 | 17.86% | 1 | 3.57% |
| P4 Supply chain | 8 | 9.52% | 2 | 1.90% |
| P5 Maintainability / testing | 14 | 16.67% | 2 | 3.33% |
| P6 Operability | 12 | 14.29% | 1 | 2.86% |
| **Total** | **84** | **100%** | — | **23.57%** |

The unredistributed raw maturity (using original weights that sum to 100) is **24.2%**. After proportional redistribution of the applicable 84 weight points across the remaining pillars, the effective maturity is:

Maturity % = (17.857×2 + 23.810×1 + 17.857×1 + 9.524×2 + 16.667×2 + 14.286×1) / 5
= 144.048 / 5
= **28.8%**

**Ceiling rules considered:**
- Open critical security findings present (2) → cap 60%. 28.8% is below cap; no reduction.
- No tested restore procedure → cap 70%. Not applicable to a development tool; no reduction.
- No legal blocker confirmed; no 50% cap applied.

## 5. Risk matrix

| Likelihood / Impact | Low | Medium | High | Critical |
|---|---|---|---|---|
| **Almost certain** | — | — | F-AM-P2c-004 (gen SQL injection) | F-AM-P2c-002 (arbitrary SQL) |
| **Likely** | — | F-AM-P6-002 (rate limiting) | F-AM-P2c-005 (session singleton) | F-AM-P2c-001 (HTTP no auth) |
| **Possible** | — | F-AM-P4-001 (floating deps)<br>F-AM-P5-001 (no CI)<br>F-AM-P6-001 (local audit) | F-AM-P2c-003 (destructive auto-approve)<br>F-AM-P2d-001 (credentials in config)<br>F-AM-P1-001 (no privilege separation)<br>F-AM-P3-001 (GDPR gaps) | — |
| **Unlikely** | — | F-AM-P2c-006 (export path) | F-AM-P2d-002 (weak masking) | — |

## 6. Detailed findings

See `audit/findings.jsonl` for machine-readable records. Summary by pillar:

### P2c — Command execution / code generation (critical/high)
- **F-AM-P2c-001** (critical): HTTP/SSE transport has no authentication.
- **F-AM-P2c-002** (critical): `apex_run_sql` executes arbitrary DML/DDL.
- **F-AM-P2c-003** (high): Tool annotations alone protect destructive operations.
- **F-AM-P2c-004** (high): Generated PL/SQL uses string interpolation with single-quote escaping only.
- **F-AM-P2c-005** (high): Single-user `ImportSession` singleton unsafe for concurrent HTTP clients.
- **F-AM-P2c-006** (medium): `apex_export_app` writes to arbitrary filesystem path.

### P2d — Secret custody
- **F-AM-P2d-001** (high): Credentials stored in `.mcp.json` / environment variables.
- **F-AM-P2d-002** (medium): Wallet password weakly masked in health checks.

### P1 — Architecture
- **F-AM-P1-001** (high): No separation between AI client trust boundary and database privileges.

### P3 — Legal / compliance
- **F-AM-P3-001** (high): No documented GDPR lawful basis, retention, deletion, or portability.

### P4 — Supply chain
- **F-AM-P4-001** (medium): Dependency versions float; no lockfile.

### P5 — Maintainability / testing
- **F-AM-P5-001** (medium): No linting, formatting, type checking, or CI security gates.

### P6 — Operability
- **F-AM-P6-001** (medium): Audit log local-only, no integrity or off-box shipping.
- **F-AM-P6-002** (medium): No rate limiting on HTTP/SSE endpoints.

## 7. Roadmap

| Lane | ID | Item | Acceptance criterion | Effort | Owner role | Dependencies |
|---|---|---|---|---|---|---|
| Stop the bleed | R-001 | Add auth to HTTP/SSE transport | Unauthenticated requests return 401; tokens rotate | M | Backend developer | — |
| Stop the bleed | R-002 | Restrict `apex_run_sql` to read-only or gated writes | DML/DDL rejected by default; privileged mode requires role + confirmation | M | Backend developer / DBA | R-001 |
| Quick wins | R-003 | Integrate secrets manager | Server connects without plaintext env password; `.mcp.json` removed from git | M | DevOps | — |
| Quick wins | R-004 | Add CI with lint, type check, bandit, pytest | PRs blocked on failure | M | DevOps | — |
| Quick wins | R-005 | Sanitize export path | Path restricted to configured directory | S | Backend developer | — |
| Medium-term | R-006 | Per-client session scoping | Concurrent HTTP clients do not share import state | L | Backend developer | R-001 |
| Medium-term | R-007 | Server-side confirmation for destructive tools | `apex_delete_*` require explicit confirm token | M | Backend developer | R-001 |
| Medium-term | R-008 | Bind variables / allowlist for generated SQL | Prompt-injection payloads rejected; generated code reviewed in dry-run | L | Backend developer | — |
| Strategic | R-009 | GDPR documentation and data lifecycle | Published privacy policy; retention + deletion + export implemented | L | Product / Legal / Backend | — |
| Strategic | R-010 | Append-only external audit trail | Audit events shipped to SIEM; deletion triggers alert | M | DevOps | R-003 |

## 8. What is done well

1. **Dry-run and batch modes** provide a safety net for interactive development (`db.py`, `app_tools.py`).
2. **Validators exist** for common inputs (page/app IDs, item names, table names, SQL prefixes).
3. **Auto-reconnect logic** handles transient Oracle connection failures robustly.

## 9. Open questions and recommended next step

1. What is the intended production deployment model for HTTP mode — localhost proxy, container, or exposed service?
2. Which secrets manager is already in use by the target customer base (OpenBao, AWS Secrets Manager, etc.)?
3. Does any consumer currently rely on `apex_run_sql` for DML/DDL, or can it be restricted?

**Recommended next step:** Implement transport authentication (R-001) and restrict `apex_run_sql` (R-002) before any further enterprise evaluation. These two changes alone remove the disqualifying risks.

## 10. Machine-readable summary

```json
{
  "maturity_pct": 28.8,
  "ceiling_applied": "none (already below 60% cap from critical findings)",
  "applicable_pillars": ["P1", "P2", "P3", "P4", "P5", "P6"],
  "na_pillars": ["P7", "P8"],
  "counts": {
    "critical": 2,
    "high": 6,
    "medium": 6,
    "low": 0,
    "info": 0
  },
  "top_risks": [
    "Unauthenticated HTTP/SSE transport exposes all tools",
    "apex_run_sql allows arbitrary SQL/DML/DDL",
    "Database credentials stored in .mcp.json / environment variables"
  ]
}
```
