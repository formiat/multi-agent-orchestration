# Plan From Context

Create or refine a fix plan from the provided context and prompt.

The full context is in `$ARGUMENTS`. Treat it as the authoritative planning request. It may include:
- a newly discovered regression or side bug in the current branch;
- already established findings from local investigation;
- objections to the current `PLAN.md`;
- explicit implementation constraints, non-goals, or desired review points.

Summarize what needs to be planned before proceeding.

If `INVESTIGATION.md` exists in the repo root, read it before drafting the plan. Treat its confirmed findings and ruled-out hypotheses as input constraints for planning. If your new evidence contradicts `INVESTIGATION.md`, call that out explicitly in `PLAN.md` instead of silently ignoring it.

If `PLAN.md` already exists in the repo root, read it first and treat it as the current plan to review, refine, or partially rewrite rather than ignore.

## Step 2: Check current branch

Check the current branch with `git branch --show-current`.

- Continue on the current branch as-is.
- Do not create or switch branches automatically for this command unless the provided context explicitly asks for that.

If the current branch matches `task-<id>-...`, you may reuse that task id in `PLAN.md` and in the commit message. Otherwise treat this as a context-only planning request.

## Step 3: Triage and plan

Spawn an agent that does the following:

1. **Analyze the context** — restate the request, acceptance criteria, affected subsystems, explicit constraints, and edge cases from the prompt.
2. **Find related code** — search the codebase for modules, structs, functions, configs, tests, commits, and local materials relevant to the requested fix. Read the key files.
3. **Review existing planning state** — if `PLAN.md` already exists, review it critically before drafting updates. Preserve useful parts, remove wrong assumptions, and explicitly note what changed.
4. **Draft an implementation plan** — create or update `PLAN.md` in the repo root:
   - Write the plan in English
   - **Context**: the prompt/request that triggered this plan
   - **Investigation context**: if `INVESTIGATION.md` exists, briefly list the findings from it that affect the plan
   - **Current plan review**: if `PLAN.md` existed before, note what was wrong or incomplete in the previous version
   - **Affected components**: list of crates/modules that need changes
   - **Implementation steps**: numbered, concrete steps with file paths
   - **Testing strategy**: try to propose the maximum practical set of automated tests for the new and affected functionality; group all proposed automated tests into three categories: 1) no refactoring required, 2) light refactoring required, and 3) heavy refactoring required; list all three categories in the plan and explicitly mark category 1 as planned for implementation together with the main functional changes; for changes to existing functionality, still add new tests if current coverage is insufficient or adapt existing tests to the new logic; require tests to be self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state; use manual verification only as a supplement
   - **Risks and tradeoffs**: what can break or what needs special care
   - **Open questions**: anything that needs clarification before starting
5. **Commit the plan** — commit `PLAN.md` as a separate local commit. Never push commits automatically or without an explicit user command.
   Use this message:
   - if the current branch matches `task-<id>-...`: `[TASK-<id>] PLAN.md`
   - otherwise: `[context] PLAN.md`

After the plan is committed, show the user the plan summary.
