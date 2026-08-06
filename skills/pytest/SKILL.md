---
name: pytest
description: Use when writing, extending, or fixing tests with pytest.
---

# Writing tests

Produce tests that match the project's existing style exactly — study neighbouring
`tests/test_*.py` files before writing.

## Hard rules

- **Function tests only.** Never write test classes. Each test is a top-level `def`/`async def`.
- **Async tests** use the `@pytest.mark.anyio` decorator. Never call `asyncio` directly; the
  project standardises on `anyio`.
- **Fixtures** via `@pytest.fixture`. Put shared fixtures in `tests/conftest.py`; reuse the
  existing ones rather than re-creating them. Never add unused fixtures arguments. Use `@pytest.mark.usefixtures` instead
- **Parametrization** via `@pytest.mark.parametrize` for input/branch variations.
- **Lint/typing in tests**: check pyproject's per-file-ignores for `tests/*` — projects
  typically relax rules there (bare asserts, missing annotations). Match what the existing
  test files do; don't add stricter or looser style than the neighbours.
- Reuse the project's established test harness (e.g. app/client fixtures, factories, test
  doubles) instead of inventing new infrastructure.
- Never add `# noqa` / `# type: ignore` to make a test pass — fix the test instead.

## What good coverage means

- Aim for high **branch** coverage, not just line coverage. Cover error paths, edge inputs,
  and each `if`/`except` branch.
- Test **real business logic and behaviour** — public contracts, error handling, edge cases.
  Do NOT write dummy tests that only exist to bump the coverage number.

## Workflow

1. Read the module under test and 2-3 sibling test files to match structure and imports.
2. Write focused tests; parametrize variations instead of duplicating bodies.
3. Run the relevant subset of tests.
4. Report uncovered branches you deliberately left out and why.
