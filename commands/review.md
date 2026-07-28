---
description: Do a full code review of current changes + write test + fix loop
---
Perform a full code review of the current branch using `code-reviewer` agent. Categorize every finding as issue/nit. Spawn parallel sub-agents — one per issue — to apply the fixes. Each sub-agent must run tests and pre-commit before reporting back. Do NOT edit any third-party library internals. Once all agents complete, run the full verification suite one final time and give me a consolidated summary of what changed, with test counts and coverage.
