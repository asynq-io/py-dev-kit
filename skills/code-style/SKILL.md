---
name: code-style
description: Use when writing or modifying Python code — implementing features, refactoring, or reviewing style. Enforces project code style: PEP8, precise typing, anyio over asyncio, structured concurrency, self-documenting code.
---

# Code style
- **ALWAYS adhere to PEP8.**
- **Mimic the code structure and patterns used in the project** — study neighbouring
  modules before writing and match their conventions.
- **Add the most precise typing annotations possible** — avoid `Any`; prefer concrete
  types, generics, `TypedDict`/`Protocol`, and narrow unions.
- **Use async/await when feasible; always `anyio` over raw `asyncio`**, with
  structured concurrency — follow the `anyio` skill.
- **Prefer self-documenting code over comments** — clear names and small functions
  instead of explanatory comments.
- **Public classes and methods/functions should have short, meaningful docstrings.**
