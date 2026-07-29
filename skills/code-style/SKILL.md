---
name: code-style
description: Always use when writing, refactoring, or reviewing Python code.
---

# Code style
- **Always Follow PEP8**
- Follow SOLID principles.
- Match neighbouring modules' structure, patterns, and conventions — read them first.
- Type as precisely as possible: no `Any`; prefer concrete types, generics, `TypedDict`/`Protocol`, narrow unions.
- For concurrency related/async code use `anyio` skill.
- Self-documenting code over comments: clear names, small functions.
- Short docstrings on new public classes and functions; don't add them to existing code unless asked.
