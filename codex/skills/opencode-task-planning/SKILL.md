---
name: opencode-task-planning
description: Coordinate a persistent local OpenCode session to create PLAN.md for a task, then review and iterate on the plan until it is good enough. Use when the user wants OpenCode to write the implementation plan, save it to PLAN.md, commit it, and then receive Codex review in a loop. Typical inputs include task_id=TASK-1234.
---

# OpenCode Task Planning

Read these files and follow them in order:

1. `../.shared/delegated-agent/common/basics.md`
2. `../.shared/delegated-agent/providers/opencode/session.md`
3. `../.shared/delegated-agent/providers/opencode/run-loop.md`
4. `references/workflow.md`

## Inputs

Extract these inputs from the user request:

- `task_id` (required)

The workflow-specific reference defines the task-branch preflight, planning artifact rules, review loop, and final deliverable.
