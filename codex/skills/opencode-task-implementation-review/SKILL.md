---
name: opencode-task-implementation-review
description: Coordinate a persistent local OpenCode session to implement either from an existing PLAN.md or directly from an explicit user implementation prompt, then review OpenCode's commits and send review feedback in a loop until quality is acceptable. Use when the user wants OpenCode to do the implementation work while Codex performs code review and steers follow-up fixes. Typical inputs include an optional implementation_hint; at least one of PLAN.md or an explicit user implementation prompt must be available.
---

# OpenCode Task Implementation Review

Read these files and follow them in order:

1. `../.shared/delegated-agent/common/basics.md`
2. `../.shared/delegated-agent/providers/opencode/session.md`
3. `../.shared/delegated-agent/providers/opencode/run-loop.md`
4. `references/workflow.md`

## Inputs

Extract these inputs from the user request:

- `implementation_hint` (optional, but required when `PLAN.md` does not exist)

If `PLAN.md` exists, `implementation_hint` stays optional.
If `PLAN.md` does not exist, a non-empty user implementation prompt/hint is required and becomes the authoritative starting scope for the workflow.
The workflow-specific reference defines scope selection, implementation/test requirements, review loop, and final deliverable.
