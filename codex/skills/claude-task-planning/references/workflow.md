# Workflow

This workflow adds task-scoped branch preflight, planning-specific prompt requirements, and the final reporting contract on top of the shared delegated-agent and Claude-provider references.

## Task-branch preflight

- Run `notion task view <task_id>` and use the task title only for branch preflight.
- Derive a short kebab-case description from the task title.
- The expected branch name is `task-<task_id>-<short-description>`.
- Read the current branch with `git branch --show-current`.
- If the current branch already belongs to the same task, stay on it even if the suffix differs.
- Otherwise run `git fetch origin develop`, then `git checkout --no-track -b task-<task_id>-<short-description> origin/develop`, then verify upstream is unset and clear it immediately with `git branch --unset-upstream` if needed.
- Commit `CLAUDE_SESSION.json` only after this branch preflight is complete.

## Planning inputs and existing-result reuse

- If `INVESTIGATION.md` exists in the repository root, study it before any planning review and treat it as required input for `PLAN.md`.
- Before sending the first new Claude message in this workflow run, inspect the current branch for an already-produced planning result from an earlier Claude round.
- If a non-empty `PLAN.md` relevant to the current task already exists, review it immediately.
- If it already meets the quality bar, stop and report without contacting Claude further.
- Otherwise keep the findings and use them as the first Claude inbox message instead of a fresh `/take-task <task_id>` bootstrap or generic planning request.

## Claude session matching

- Use the default Claude session discovery rules from `../../.shared/delegated-agent/providers/claude/session.md`.

## Session metadata discipline

- If an existing Claude session is reused and `CLAUDE_SESSION.json` does not exist yet, create and commit it before sending the first delegated request into that session.
- If a new Claude session is explicitly approved and bootstrapped, discover its real `session_uuid` immediately after bootstrap via the shared before/after discovery rules, create and commit `CLAUDE_SESSION.json` immediately, and only then continue normal waiting for outbox or other round results. Do not wait for turn completion before fixing the new session metadata.

## New-session bootstrap

- When the user explicitly approves creating a new Claude session, bootstrap with an English request that tells Claude to `Run /take-task <task_id>.`
- That bootstrap must tell Claude not to commit inbox/outbox, to commit planning results, never to push automatically or without an explicit user command, to write minimal status in English to `./.codex/outbox.md`, to keep resulting human-facing workflow artifacts/output in English by default, and to keep any code comments added or edited in project files in English only.
- If the existing `PLAN.md` already produced review findings, use that first consultative review message instead of the fresh bootstrap request.

## Planning-specific request requirements

- In addition to the shared Claude transport contract, every outer prompt for this workflow must also require Claude to:
- evaluate the request and planning scope as a colleague and explain mismatches or weak assumptions in outbox instead of silently diverging;
- read `INVESTIGATION.md` when it exists and account for its confirmed findings in `PLAN.md`;
- propose as many practical automated tests as possible for the new and affected functionality, grouped into three categories: `1)` no refactoring required, `2)` light refactoring required, and `3)` heavy refactoring required;
- list all three categories and explicitly mark category `1` as planned for implementation together with the main functional changes;
- for changes to existing functionality, still add tests when current coverage is insufficient or adapt existing tests to the new logic;
- keep planned tests self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state;
- make a commit after any project file changes and report what changed plus the commit hash in outbox.

## Wait condition

- Wait until Claude writes `PLAN.md`, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`.
- Treat dirty project-file changes, local commits, `PLAN.md`, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- If a local commit appears, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check.

## Independent verification focus

- Study the task, inspect `PLAN.md`, inspect `INVESTIGATION.md` too if it exists, and independently verify as many central claims as practical against the repository, local materials, current branch state, relevant tests, and git history.

## Review-loop focus

- Review the plan aggressively and skeptically.
- Look for flaws, missing pieces, weak assumptions, insufficient test-coverage expectations, inconsistencies, contradictions with `INVESTIGATION.md`, and anything important that Claude failed to notice on its own.
- For each review round, first clear `./.codex/inbox.md` locally with `truncate -s 0` and verify that the inbox size is `0`, then write a consultative review there that lists findings, risks, objections, investigation findings that the plan ignored or contradicted, and independently discovered facts or counterexamples, and ask Claude to either update `PLAN.md` or explain why it disagrees in outbox.
- Do not patch `PLAN.md` locally while reviewing; send findings back to Claude for evaluation.
- Keep local changes limited to workflow/support files such as `./.codex/inbox.md`, `./.codex/outbox.md`, and `CLAUDE_SESSION.json`.

## Deliverable

Report to the user:

- the final plan quality assessment;
- the workflow stop reason; if the stop reason was reaching the quality bar, state that explicitly and include the achieved quality assessment;
- what major issues were found and whether they were fixed or explicitly disputed;
- the last relevant Claude commit hash if Claude committed changes;
- whether the workflow stopped because Claude could not complete the work for any reason.
