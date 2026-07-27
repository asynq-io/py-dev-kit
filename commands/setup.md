---
description: Initialize the project with py-dev-kit defaults (CLAUDE.md + .claude/settings.json)
allowed-tools: Read, Write, Edit, Glob, Bash(mkdir:*), Bash(ls:*)
---

CLAUDE.md template: @${CLAUDE_PLUGIN_ROOT}/templates/CLAUDE.md

Settings template: @${CLAUDE_PLUGIN_ROOT}/templates/settings.json

Set up this project with the templates above:

1. **CLAUDE.md** — if the project root has no `CLAUDE.md`, write the template
   verbatim. If one exists, merge in the missing sections and keep all existing
   content; never delete or reorder what the user already has.
2. **.claude/settings.json** — if missing, write the template (create the
   `.claude/` directory if needed). If it exists, merge: add missing `allow`
   and `deny` entries, keep every existing key untouched. Do not add hooks —
   the plugin already provides them.
3. Report what was created or changed, and list anything skipped because it
   was already present.
