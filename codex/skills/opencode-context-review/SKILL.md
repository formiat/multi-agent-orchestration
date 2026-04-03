---
name: opencode-context-review
description: Coordinate a persistent local OpenCode session to review a branch, commit, MR, diff, or other repository entity from current context and a freeform prompt-hint, then independently review OpenCode's findings and iterate until the review quality is good enough. Use when the user wants Codex and local OpenCode to perform a joint review of any locally inspectable target. Typical inputs include a required review_hint.
---

# OpenCode Context Review

Read these files and follow them in order:

1. `../.shared/delegated-agent/common/basics.md`
2. `../.shared/delegated-agent/providers/opencode/session.md`
3. `../.shared/delegated-agent/providers/opencode/run-loop.md`
4. `references/workflow.md`

## Inputs

Extract these inputs from the user request:

- `review_hint` (required; freeform prompt/comment that identifies the review target and any desired focus)

The workflow-specific reference defines the review target semantics, review-output contract, refinement loop, and final deliverable.
