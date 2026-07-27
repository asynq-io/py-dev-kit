---
name: pytest
description: Use when writing or extending pytest tests for a Python project — adding coverage for a module or feature, testing a just-implemented change, or fixing failing tests. Enforces project conventions: function-only tests, anyio for async, fixtures, parametrization, high branch coverage.
---

# Writing tests

Produce tests that match the project's existing style exactly — study neighbouring
`tests/test_*.py` files before writing.

## Hard rules

- **Function tests only.** Never write test classes. Each test is a top-level `def`/`async def`.
- **Async tests** use the `@pytest.mark.anyio` decorator. Never call `asyncio` directly; the
  project standardises on `anyio`.
- **Fixtures** via `@pytest.fixture`. Put shared fixtures in `tests/conftest.py`; reuse the
  existing ones rather than re-creating them.
- **Parametrization** via `@pytest.mark.parametrize` for input/branch variations.
- **Lint/typing in tests**: check pyproject's per-file-ignores for `tests/*` — projects
  typically relax rules there (bare asserts, missing annotations). Match what the existing
  test files do; don't add stricter or looser style than the neighbours.
- Reuse the project's established test harness (e.g. app/client fixtures, factories, test
  doubles) instead of inventing new infrastructure.

## What good coverage means

- Aim for high **branch** coverage, not just line coverage. Cover error paths, edge inputs,
  and each `if`/`except` branch.
- Test **real business logic and behaviour** — public contracts, error handling, edge cases.
  Do NOT write dummy tests that only exist to bump the coverage number.

## Workflow

1. Read the module under test and 2-3 sibling test files to match structure and imports.
2. Write focused tests; parametrize variations instead of duplicating bodies.
3. Run the relevant subset: `./run_tests.sh` if present, otherwise
   `uv run pytest tests/test_<name>.py -v`.
4. Report uncovered branches you deliberately left out and why. Never add `# noqa` /
   `# type: ignore` to make a test pass — fix the test instead.
