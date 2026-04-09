# Workflow

This workflow adds task-scoped branch preflight, planning-specific OpenCode command usage, review-loop rules, and the final reporting contract on top of the shared delegated-agent and OpenCode-provider references.

## Task-branch preflight

- Run `notion task view <task_id>` and use the task title only for branch preflight.
- Derive a short kebab-case description from the task title.
- The expected branch name is `task-<task_id>-<short-description>`.
- Read the current branch with `git branch --show-current`.
- If the current branch already belongs to the same task, stay on it even if the suffix differs.
- Otherwise run `git fetch origin develop`, then `git checkout --no-track -b task-<task_id>-<short-description> origin/develop`, then verify upstream is unset and clear it immediately with `git branch --unset-upstream` if needed.
- Commit `OPENCODE_SESSION.json` only after this branch preflight is complete.

## Planning inputs and existing-result reuse

- If `INVESTIGATION.md` exists in the repository root, study it before any planning review and treat it as required input for `PLAN.md`.
- Before sending the first new OpenCode message in this workflow run, inspect the current branch for an already-produced planning result from an earlier OpenCode round.
- If a non-empty `PLAN.md` relevant to the current task already exists, review it immediately.
- If it already meets the quality bar, stop and report without contacting OpenCode further.
- Otherwise keep the findings and use them as the first OpenCode request instead of a fresh bootstrap.

## OpenCode session matching

- Use the default OpenCode session discovery rules from `../../.shared/delegated-agent/providers/opencode/session.md`.

## New-session bootstrap

- When the user explicitly approves creating a new OpenCode session, bootstrap with:
  `opencode run --title "<codex_session_name>" --command take-task <task_id>`
- Do not pass `./.codex/inbox.md` through `-f` or `--file`. The command must read `./.codex/inbox.md` from the current working directory on its own.
- If the existing `PLAN.md` already produced review findings, use a direct English review prompt instead of the fresh command bootstrap.

## Planning-specific request requirements

- In direct OpenCode prompts for this workflow, require OpenCode to:
- evaluate the request and planning scope as a colleague and explain mismatches or weak assumptions explicitly;
- read `INVESTIGATION.md` when it exists and account for its confirmed findings in `PLAN.md`;
- propose as many practical automated tests as possible for the new and affected functionality, grouped into three categories: `1)` no refactoring required, `2)` light refactoring required, and `3)` heavy refactoring required;
- list all three categories and explicitly mark category `1` as planned for implementation together with the main functional changes;
- for changes to existing functionality, still add tests when current coverage is insufficient or adapt existing tests to the new logic;
- keep planned tests self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state;
- read `./.codex/inbox.md` as the authoritative delegated request for the current round;
- read `./.codex/outbox.md` with the Read tool before resetting it, then clear it in place via truncation, and only then write the round result so the file contains only the latest response; never delete `./.codex/outbox.md`;
- make a commit after any project file changes and report what changed plus the commit hash in `./.codex/outbox.md`;
- never push commits automatically or without an explicit user command.

## Wait condition

- Wait until OpenCode writes `PLAN.md`, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`.
- Treat dirty project-file changes, local commits, `PLAN.md`, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- If a local commit appears, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check.
- If outbox and cheaper external signals are insufficient, run only a narrow `rg`/`grep` marker check on the freshest `1-2` OpenCode log files newer than the current request start before any export. Do not read raw log bodies in the normal path.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when those fresh-log checks are inconclusive, at most once every `5` minutes, by writing a stripped timestamped snapshot under `/tmp/`, reading only `root.messages[-1]` first, and reading `root.messages[-2]` only if `root.messages[-1]` is insufficient.

## Independent verification focus

- Study the task, inspect `PLAN.md`, inspect `INVESTIGATION.md` too if it exists, and independently verify as many central claims as practical against the repository, local materials, current branch state, relevant tests, and git history.

## Review-loop focus

- Review the plan aggressively and skeptically.
- Look for flaws, missing pieces, weak assumptions, insufficient test-coverage expectations, inconsistencies, contradictions with `INVESTIGATION.md`, and anything important that OpenCode failed to notice on its own.
- For each review round, send a direct English consultative prompt through `opencode run -s <session_id> "<prompt>"`.
- For those direct review-round prompts, do not use `-f` or `--file` for `./.codex/inbox.md`; instruct OpenCode in the prompt itself to read `./.codex/inbox.md` from the current working directory.
- The prompt must list findings, risks, objections, investigation findings that the plan ignored or contradicted, and independently discovered facts or counterexamples, then ask OpenCode to either update `PLAN.md` or explain why it disagrees in `./.codex/outbox.md`.
- Do not patch `PLAN.md` locally while reviewing; send findings back to OpenCode for evaluation.
- Keep local changes limited to workflow/support files such as `OPENCODE_SESSION.json`.

## Deliverable

Report to the user:

- the final plan quality assessment;
- the workflow stop reason; if the stop reason was reaching the quality bar, state that explicitly and include the achieved quality assessment;
- what major issues were found and whether they were fixed or explicitly disputed;
- the last relevant OpenCode commit hash if OpenCode committed changes;
- whether the workflow stopped because OpenCode could not complete the work for any reason.
