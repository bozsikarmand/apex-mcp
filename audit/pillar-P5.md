# P5 — Maintainability / Testing: apex-mcp

## Scope inspected

- `tests/` — test suite
- `pyproject.toml` — pytest config
- `README.md` — testing instructions
- `.github/` — CI presence

## What was found

The repo contains 19 test files. `pytest.ini_options` defines `integration` and `slow` markers. The README claims 51 unit tests. `conftest.py` provides session reset fixtures.

However, there is no linter, formatter, or type checker configured. There is no CI workflow to enforce tests on PRs. The project appears to be maintained by a very small team (likely single author), creating bus-factor risk.

Tests require a live Oracle connection for integration coverage; without it, only basic validators and error paths can be exercised.

## What is done well

1. **Unit tests exist** for validators, tools, IDs, sessions, guards, exceptions, config, and audit.
2. **Clear pytest markers** separate integration tests from unit tests.
3. **Demo scripts** serve as informal acceptance tests for full app builds.

## What could not be verified

- Actual test pass rate (no live Oracle available during audit).
- Code coverage percentage.

## Maturity score

**P5: 2 / 5** — Tests exist but quality gates and CI are missing.
