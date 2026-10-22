# P4 — Supply Chain: apex-mcp

## Scope inspected

- `pyproject.toml` — dependencies
- `.github/dependabot.yml` — dependency alerts
- `apex_mcp/__init__.py` — version

## What was found

Runtime dependencies are specified with floating lower bounds:

```toml
dependencies = [
    "fastmcp>=3.0.0",
    "oracledb>=2.0.0",
]
```

There is no lockfile (`requirements-lock.txt`, `poetry.lock`, `uv.lock`). Builds are not reproducible. Dependabot is configured for pip and GitHub Actions, but there is no CI workflow to run `pip-audit` or `safety`.

No SBOM is generated. No artefact signing is configured.

## What is done well

1. **Dependabot is enabled** for both pip and GitHub Actions.
2. **MIT license** is clearly stated with no known copyleft conflict.

## What could not be verified

- Actual installed versions in consumer environments.
- Transitive dependency tree beyond direct deps.

## Maturity score

**P4: 2 / 5** — Partially present (dependabot), not enforced or measured.
