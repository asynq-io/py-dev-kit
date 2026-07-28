---
description: Full feature workflow — plan, implement, review, test, fix until green
argument-hint: <feature description>
---

Implement the following feature end-to-end: $ARGUMENTS

Follow this workflow strictly, phase by phase. Do not skip or reorder phases.

## 1. Planning

Explore the relevant code first (use Serena tools for symbol lookup). Then present
a brief implementation plan: files to touch, approach, and any trade-offs. **Wait
for users approval before writing any code.**

## 2. Implementation

Implement the approved plan using the `code-style` skill.

## 3. Code review

Run the `code-reviewer` agent on the full changeset. Categorize every finding as
**issue** or **nit**.

## 4. Fix review findings

Spawn parallel sub-agents — one per issue — to apply the fixes. Give each
sub-agent the finding verbatim plus the affected file paths. Apply nits yourself
only when trivial and safe. If fixes touched the code substantially, re-run the
`code-reviewer` agent once more.

## 5. Write tests

Delegate to the `test-writer` agent: name the modules/features to cover and any
constraints from the plan. It follows the `pytest` skill conventions.

## 6. Verify

Run the full test suite and `pre-commit run --all`.

## 7. Fix until green

If anything fails, fix the code or tests (use the `pytest` skill for test
changes) and re-run step 6. Repeat until both tests and pre-commit pass — never
silence errors with `noqa`, `type: ignore`, skips, or weakened assertions.

## 8. Summary

Report: what was implemented, files changed, review findings fixed vs. deferred,
test counts and coverage, and final tests + pre-commit status.
