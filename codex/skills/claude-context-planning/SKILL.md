---
name: claude-context-planning
description: Coordinate a persistent local Claude session to create or refine PLAN.md from current branch context and a freeform prompt instead of a task id, then review and iterate on the plan until it is good enough. Use when the user wants Claude to plan a side regression fix, re-plan from fresh local findings, or revise an existing PLAN.md based on current chat context. Typical inputs include request_context.
---

# Claude Context Planning

Read these files and follow them in order:

1. `../.shared/delegated-agent/common/basics.md`
2. `../.shared/delegated-agent/providers/claude/session.md`
3. `../.shared/delegated-agent/providers/claude/run-loop.md`
4. `references/workflow.md`

## Inputs

Extract these inputs from the user request:

- `request_context` (required)

The workflow-specific reference defines the plan artifact rules, context bootstrap, stricter Claude session matching, review loop, and final deliverable.
