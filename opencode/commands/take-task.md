---
description: Start a task by task id: read Notion, ensure task branch, and write PLAN.md.
agent: build
---

# Take Task

Start working on task TASK-$ARGUMENTS.

Before doing substantive work:
- read `./.codex/inbox.md` and treat it as the authoritative delegated request/additional constraints for this run;
- before writing your final response, clear `./.codex/outbox.md` so that file contains only the latest response;
- do not rely on stdout for the final handoff; write the final English summary to `./.codex/outbox.md`.

## Step 1: Understand the task

Run `notion task view $ARGUMENTS` to get the task description, requirements, and context. Parse the output and summarize what needs to be done.

If the task description references other tasks (e.g., TASK-1234), fetch their details too using `notion task view <id>`. Those related tasks may already be implemented — search for their commits with `git log --all --grep="TASK-<related-task-id>"` to understand what was done and how, which can inform the current implementation.

If `INVESTIGATION.md` exists in the repo root, read it before drafting the plan. Treat its confirmed findings and ruled-out hypotheses as input constraints for planning. If your new evidence contradicts `INVESTIGATION.md`, call that out explicitly in `PLAN.md` instead of silently ignoring it.

## Step 2: Check current branch

Derive a short kebab-case description (2-4 words) from the task title. The expected branch name is `task-$ARGUMENTS-<short-description>`.

Check the current branch with `git branch --show-current`. Extract the task id from the current branch name if it matches a `task-<id>-...` pattern.

- If the current branch already belongs to TASK-$ARGUMENTS, continue on the current branch as-is, even if its suffix differs from the newly derived short description.
- If the current branch task id is missing or does not match TASK-$ARGUMENTS, create the expected branch automatically without asking the user:
  1. Run `git fetch origin develop`.
  2. Create the branch without tracking: `git checkout --no-track -b task-$ARGUMENTS-<short-description> origin/develop`.
  3. Verify upstream is not set. If upstream exists, immediately clear it: `git branch --unset-upstream`.

Important: a newly created task branch must **never** be tied to `origin/develop` as upstream.

## Step 3: Triage and plan

Spawn an agent that does the following:

1. **Analyze the task** — re-read the notion task output, identify acceptance criteria, affected subsystems, and edge cases.
2. **Find related code** — search the codebase for modules, structs, functions, configs, and tests related to the task. Read the key files.
3. **Check related local data/manual-verification assets** — if the task involves detection, violations, or video processing, you may inspect relevant local datasets or test infrastructure for investigation/manual verification context, but do not make automated tests depend on `/ds/...`, sibling repositories, local DB files, migration directories, absolute paths, or other machine-specific assets.
4. **Draft an implementation plan** — write a `PLAN.md` file in the repo root with:
   - Write the plan in English
   - **Task**: TASK-$ARGUMENTS title and summary
   - **Investigation context**: if `INVESTIGATION.md` exists, briefly list the findings from it that affect the plan
   - **Affected components**: list of crates/modules that need changes
   - **Implementation steps**: numbered, concrete steps with file paths
   - **Testing strategy**: try to propose the maximum practical set of automated tests for the new and affected functionality; group all proposed automated tests into three categories: 1) no refactoring required, 2) light refactoring required, and 3) heavy refactoring required; list all three categories in the plan and explicitly mark category 1 as planned for implementation together with the main functional changes; for changes to existing functionality, still add new tests if current coverage is insufficient or adapt existing tests to the new logic; require tests to be self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state; use manual verification only as a supplement
   - **Open questions**: anything that needs clarification before starting
5. **Commit the plan** — commit `PLAN.md` as a separate local commit with message: `[TASK-$ARGUMENTS] PLAN.md`. Never push commits automatically or without an explicit user command.

After the plan is committed, write a concise English plan summary to `./.codex/outbox.md`.
