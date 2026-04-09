---
name: update-mr
description: Create or update GitLab MR for the current branch. Creates a new MR if none exists, or updates the description of the existing one. Preserves Notion link and other existing content. Use when the user invokes $update-mr or asks to sync the current branch with its GitLab MR.
---

# Update MR

Read `references/workflow.md` and follow it.

## Inputs

Extract these inputs from the user request:

- `extra_context` (optional; any additional current-conversation details that should be reflected in the MR description)

If `extra_context` is not given, do not ask.

## Operating Rules

- Read the repository `AGENTS.md` before starting.
- If the repository defines extra startup context loading, do that first.
- Strictly forbidden: do not modify any files under any circumstances as part of this workflow.
- Strictly forbidden: do not create commits under any circumstances as part of this workflow.
- Follow the current repository `CLAUDE.md` and local repo state when composing the MR description.
- Never overwrite an MR description blindly. If an MR already exists, always read its current description first and preserve the Notion link and any other manually written content.
- When updating an existing MR description, preserve the Notion link at the very top.
- When no MR exists, derive the task id from the current branch name if possible and fetch the full Notion URL via `notion task view <TASK_ID>` instead of composing a guessed URL.
- If the MR touches HTTP or web routes, the MR description must include the full list of touched routes.
- If the route list is very large or the MR effectively touches all routes of a module, route patterns may be masked with `*`, for example `/api/collage/*`.
- Prefer explicit concrete routes over masked patterns whenever the full list is still reasonably small.
- If the branch is not yet on `origin`, push it only as part of this explicitly invoked workflow and only after the upstream safety check from the workflow reference.
- If the current branch has unpushed local commits, push them as part of this explicitly invoked workflow before creating or updating the MR description.
- If push requires rebase or otherwise stops because the remote branch has diverged, stop and inform the user. Do not rebase as part of this workflow.
- After creating an MR, update the Notion task in the `Merge Request -> BACKEND` block using an in-place block update. Never use a `pull | sed | push` style rewrite that can break link annotations.
- Do not push any commits automatically unless this workflow explicitly requires pushing the current branch for MR creation/update and the user invoked `$update-mr`.

## Deliverable

Report to the user:

- whether an existing MR was updated or a new MR was created;
- the MR id and URL;
- whether the Notion task was updated;
- any push-related issue that prevented MR creation/update.
