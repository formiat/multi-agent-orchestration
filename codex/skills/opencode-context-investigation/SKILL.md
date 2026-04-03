---
name: opencode-context-investigation
description: Coordinate a persistent local OpenCode session to investigate a bug from current branch context and a freeform prompt instead of a task id, create or update INVESTIGATION.md, and review the investigation in a loop until it is good enough. Use when the user wants OpenCode to study a side regression, fresh local bug, or prompt-defined issue already discussed in chat. Typical inputs include request_context.
---

# OpenCode Context Investigation

Read these files and follow them in order:

1. `../.shared/delegated-agent/common/basics.md`
2. `../.shared/delegated-agent/providers/opencode/session.md`
3. `../.shared/delegated-agent/providers/opencode/run-loop.md`
4. `references/workflow.md`

## Inputs

Extract these inputs from the user request:

- `request_context` (required; freeform prompt/context)

The workflow-specific reference defines the context-investigation bootstrap, investigation artifact rules, review loop, and final deliverable.
