## Project
- All dependencies and settings are specified in `pyproject.toml` file.
- If `uv` is used in project always run python related commands with `uv run` prefix, e.g:
`uv run pytest`, `uv run python -C ...`, `uv run ruff`

## Commands
- `pre-commit run --all` - running pre-commit run after finishing applying changes

## Workflow
- Prefer small, atomic changes when possible
- Run `pre-commit` and tests
- Fix errors errors

## Tooling
- Use `serena_mcp` tools for code navigation and symbol lookup instead of grep
- Use `context7` for library documentation lookup

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
- Over-engineer the solution
