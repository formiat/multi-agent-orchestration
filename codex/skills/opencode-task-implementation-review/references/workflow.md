# Workflow

This workflow adds implementation-scope selection, OpenCode implementation/test requirements, and the final reporting contract on top of the shared delegated-agent and OpenCode-provider references.

## Scope selection and existing-result reuse

- Determine the implementation scope source before starting:
- if `PLAN.md` exists in the repository root and is non-empty, use it as the authoritative implementation plan;
- otherwise require a non-empty user implementation prompt/hint and use that as the authoritative starting scope;
- if neither exists, stop and tell the user.
- If `INVESTIGATION.md` exists in the repository root, study it before reviewing or steering the implementation and treat it as supplemental factual context behind the active scope source.
- Before sending the first new OpenCode message in this workflow run, inspect the current branch for an already-produced implementation result from an earlier OpenCode round.
- Review any existing implementation/test changes relevant to the active scope source immediately.
- If the existing implementation already meets the quality bar, stop and report without contacting OpenCode further.
- Otherwise keep the findings and use them as the first OpenCode request instead of a fresh implementation request.

## OpenCode session matching

- Use the default OpenCode session discovery rules from `../../.shared/delegated-agent/providers/opencode/session.md`.

## New-session bootstrap

- When the user explicitly approves creating a new OpenCode session, bootstrap with either the first actual implementation request written in English or, when existing implementation findings already exist, that first consultative review message written in English instead.
- Do not pass `./.codex/inbox.md` through `-f` or `--file`. Direct prompts in this workflow must instruct OpenCode to read `./.codex/inbox.md` from the current working directory.

## Implementation-specific request requirements

- In direct OpenCode prompts for this workflow, require OpenCode to:
- evaluate the request and, when present, `PLAN.md` as a colleague;
- when `PLAN.md` exists, treat it as the authoritative implementation plan;
- when `PLAN.md` does not exist, treat the prompt plus the explicit user implementation hint as the authoritative starting scope;
- when `PLAN.md` exists, pass through a user-supplied `implementation_hint` only when it adds something not already captured in `PLAN.md`;
- when `PLAN.md` does not exist, preserve the user's implementation prompt/hint as the operative scope instead of replacing it with newly invented planning prose;
- explain any mismatch or disagreement explicitly instead of silently diverging;
- evaluate any user-supplied `implementation_hint` explicitly and keep the implementation consistent with it when it does not conflict with the active scope source;
- explain any conflict with the hint explicitly instead of silently dropping it;
- read `INVESTIGATION.md` when it exists and keep the implementation consistent with its confirmed findings unless new evidence disproves them;
- add or update tests according to the repository test expectations and maximize practical automated coverage for new functionality;
- add tests when current coverage is insufficient or adapt existing tests when changing existing functionality;
- keep changed tests self-contained and portable: no machine-specific local DB files, migration directories/files, `$HOME` content, absolute paths, sibling repositories, mounted volumes, temp directories outside test control, or other preexisting local state; prefer in-memory data, inline fixtures, and mock/fake/stub dependencies;
- run changed tests before finishing;
- run `cargo fmt --all` after implementation changes;
- run `make clippy` after implementation changes and fix the reported issues before finishing, or explain explicitly why `make clippy` could not be run;
- read `./.codex/inbox.md` as the authoritative delegated request for the current round;
- read `./.codex/outbox.md` with the Read tool before resetting it, then clear it in place via truncation, and only then write the round result so the file contains only the latest response; never delete `./.codex/outbox.md`;
- make a commit after any project file changes and report what changed, what commands were run, and the commit hash in `./.codex/outbox.md`;
- never push commits automatically or without an explicit user command.

## Wait condition

- Wait until OpenCode finishes implementation, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`, or clearly reports that it is blocked there.
- Treat dirty project-file changes, local commits, implementation artifacts, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait up to `2` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, wait up to `2` minutes for `./.codex/outbox.md` before reviewing the commit without outbox.
- If outbox and cheaper external signals are insufficient, run only a narrow `rg`/`grep` marker check on the freshest `1-2` OpenCode log files newer than the current request start before any export. Do not read raw log bodies in the normal path.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when those fresh-log checks are inconclusive, at most once every `5` minutes, by writing a stripped timestamped snapshot under `/tmp/`, reading only `root.messages[-1]` first, and reading `root.messages[-2]` only if `root.messages[-1]` is insufficient.

## Independent verification focus

- Study the task, inspect `PLAN.md` when it exists, inspect `INVESTIGATION.md` if it exists, account for any user-supplied implementation hint/comment if present, review the latest OpenCode commit, and independently verify as many central implementation claims as practical against the repository, local materials, current branch state, relevant tests, and git history.

## Review-loop focus

- Focus review aggressively and skeptically on bugs, regressions, missing tests, mismatches with the active scope source, contradictions with `INVESTIGATION.md`, contradictions with any user-supplied implementation hint/comment, weak assumptions, and anything important that OpenCode failed to notice on its own.
- Do not patch project code locally while reviewing. You may inspect files, run builds, run tests, run formatting/lint checks, and update workflow support files, but implementation fixes must go back to OpenCode for evaluation.
- For each review round, send a direct English consultative prompt through `opencode run -s <session_id> "<prompt>"`.
- For those direct review-round prompts, do not use `-f` or `--file` for `./.codex/inbox.md`; instruct OpenCode in the prompt itself to read `./.codex/inbox.md` from the current working directory.
- The prompt must list findings, risks, objections, and independently discovered facts or counterexamples, and restate any user-supplied implementation hint/comment that the current implementation ignored or contradicted.
- Ask OpenCode to either update the implementation or explain why it disagrees in `./.codex/outbox.md`.
- Require OpenCode in review rounds to keep changed tests self-contained and portable, rerun those tests, then run `cargo fmt --all`, then `make clippy`, or explain explicitly why `make clippy` could not be run.

## Deliverable

Report to the user:

- what OpenCode implemented;
- what review findings remain, including any explicit disagreements;
- what was verified locally;
- the last relevant OpenCode commit hash;
- whether the workflow stopped because OpenCode could not complete the work for any reason.
