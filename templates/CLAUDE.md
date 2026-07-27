## Project
- All dependencies and settings are specified in `pyproject.toml` file. Read it first
- `uv` is used for managing virtualenv and dependencies, always run commands with `uv run` prefix (mypy, ruff, pytest), except for the `pre-commit`.

## Commands
- `source .venv/bin/activate` - activate virtualenv
- `pre-commit run --all` - running pre-commit

## Workflow
- Prefer small, atomic changes
- After completing each step, run tests and pre-commit
- Fix errors from pytest/pre-commit. Try to actually fix the issue instead of adding exclusion rule or ignoring directive (# noqa, # type: ignore etc.). Sometimes it's ok to add ignore rules for `/tests` directories
- After finishing task, do a full code review on created changes, note all issues and comments, try to fix them, verify they pass pre-commit and tests

## Code style
- ALWAYS use `code-style` skill when writing python code

## Tests
- Use `pytest` skill for writing tests

## DO NOT
- Rewrite entire files without reason
- Change naming conventions
- Introduce frameworks/dependencies without request
- Add TODOs instead of implementing requested features
- Assume business logic
