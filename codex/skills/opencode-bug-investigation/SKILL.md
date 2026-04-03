---
name: opencode-bug-investigation
description: Coordinate a persistent local OpenCode session to investigate a bug, create or update INVESTIGATION.md, and review the investigation in a loop until the root cause analysis is good enough. Use when the user wants OpenCode to study logs, traces, suspect commits, and code history while Codex reviews the findings. Typical inputs include task_id=TASK-1234.
---

# OpenCode Bug Investigation

Read these files and follow them in order:

1. `../.shared/delegated-agent/common/basics.md`
2. `../.shared/delegated-agent/providers/opencode/session.md`
3. `../.shared/delegated-agent/providers/opencode/run-loop.md`
4. `references/workflow.md`

## Inputs

Extract these inputs from the user request:

- `task_id` (required)

The workflow-specific reference defines the task-branch preflight, investigation artifact rules, review loop, and final deliverable.
