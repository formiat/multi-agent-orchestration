---
name: take-task
description: "Start work on a task by id: read the Notion task, inspect related tasks and local commits, ensure the correct task branch exists, investigate the relevant code, write PLAN.md in English, and commit the plan as a separate local commit. Use when the user invokes $take-task or asks to begin a task such as TASK-1234."
---

# Take Task

Read `references/workflow.md` and follow it.

## Inputs

Extract these inputs from the user request:

- `task_id` (required; numeric part is enough, e.g. `7838`)

If `task_id` is missing, ask for it.

## Operating Rules

- Read the repository `AGENTS.md` before starting.
- If the repository defines extra startup context loading, do that first.
- Read all repository `CLAUDE.md` files before drafting the plan.
- Use `notion task view <task_id>` as the authoritative source for task title, body, and linked context.
- If the task text references other tasks, inspect them too and search local history with `git log --all --grep="TASK-<id>"` to understand prior related work.
- If `INVESTIGATION.md` exists in the repo root, read it before planning and treat its confirmed findings and ruled-out hypotheses as input constraints.
- If the current branch does not already belong to this task, create the expected task branch automatically from `origin/develop` without asking, and make sure it does not track `origin/develop`.
- Write `PLAN.md` in English.
- The plan must explicitly include as many practical automated tests as possible for the new and affected functionality. Group all proposed automated tests into three categories: 1) no refactoring required, 2) light refactoring required, and 3) heavy refactoring required. The plan must list all three categories, and category 1 must be explicitly marked as planned for implementation together with the main functional changes. For changes to existing functionality, still add tests if current coverage is insufficient or adapt existing tests to the new logic.
- Planned tests must be self-contained and portable. Do not rely on local DB files, migration directories/files, `$HOME`, absolute paths, sibling repositories, mounted volumes, or other machine-specific preexisting state. Prefer in-memory data, inline fixtures, and mock/fake/stub dependencies. Manual verification may complement tests, but should not replace reasonable automated coverage.
- If the task involves detection, violations, or video processing, you may inspect relevant local datasets or test infrastructure for investigation/manual verification context, but do not make automated tests depend on `/ds/...`, sibling repositories, local DB files, migration directories, absolute paths, or other machine-specific assets.
- Commit `PLAN.md` as a separate local commit. Never push automatically or without an explicit user command.

## Deliverable

Report to the user:

- the task title and short summary;
- the chosen/current branch;
- the main affected components from the plan;
- any open questions left in `PLAN.md`;
- the commit hash for `[TASK-<task_id>] PLAN.md`.
