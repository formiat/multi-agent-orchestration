# Workflow

This workflow adds implementation-scope selection, implementation/test-specific prompt requirements, and the final reporting contract on top of the shared delegated-agent and Claude-provider references.

## Scope selection and existing-result reuse

- Determine the implementation scope source before starting:
- if `PLAN.md` exists in the repository root and is non-empty, use it as the authoritative implementation plan;
- otherwise require a non-empty user implementation prompt/hint and use that as the authoritative starting scope;
- if neither exists, stop and tell the user.
- If `INVESTIGATION.md` exists in the repository root, study it before reviewing or steering the implementation and treat it as supplemental factual context behind the active scope source.
- Before sending the first new Claude message in this workflow run, inspect the current branch for an already-produced implementation result from an earlier Claude round.
- Review any existing implementation/test changes relevant to the active scope source immediately.
- If the existing implementation already meets the quality bar, stop and report without contacting Claude further.
- Otherwise keep the findings and use them as the first Claude inbox message instead of a fresh implementation request.

## Claude session matching

- Use the default Claude session discovery rules from `../../.shared/delegated-agent/providers/claude/session.md`.

## New-session bootstrap

- When the user explicitly approves creating a new Claude session, bootstrap with either the first actual implementation request written in English or, when existing implementation findings already exist, that first consultative review message written in English instead.

## Implementation-specific request requirements

- In addition to the shared Claude transport contract, every outer prompt for this workflow must also require Claude to:
- evaluate the request and, when present, `PLAN.md` as a colleague;
- when `PLAN.md` exists, treat it as the authoritative implementation plan;
- when `PLAN.md` does not exist, treat the inbox request plus the explicit user prompt/hint as the authoritative starting scope;
- when `PLAN.md` exists, pass through a user-supplied `implementation_hint` only when it adds something not already captured in `PLAN.md`;
- when `PLAN.md` does not exist, preserve the user's implementation prompt/hint as the operative scope instead of replacing it with newly invented planning prose;
- explain any mismatch or disagreement in outbox with technical reasoning instead of silently diverging;
- evaluate any user-supplied `implementation_hint` explicitly and keep the implementation consistent with it when it does not conflict with the active scope source;
- explain any conflict with the hint in outbox instead of silently dropping it;
- read `INVESTIGATION.md` when it exists and keep the implementation consistent with its confirmed findings unless new evidence disproves them;
- add or update tests according to the repository test expectations and maximize practical automated coverage for new functionality;
- add tests when current coverage is insufficient or adapt existing tests when changing existing functionality;
- keep changed tests self-contained and portable: no machine-specific local DB files, migration directories/files, `$HOME` content, absolute paths, sibling repositories, mounted volumes, temp directories outside test control, or other preexisting local state; prefer in-memory data, inline fixtures, and mock/fake/stub dependencies;
- run changed tests before finishing;
- run `cargo fmt --all` after implementation changes;
- run `make clippy` after implementation changes and fix the reported issues before finishing, or explain in outbox why `make clippy` could not be run;
- make a commit after any project file changes and report what changed, what commands were run, and the commit hash in outbox.

## Wait condition

- Wait until Claude finishes implementation, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`, or clearly reports that it is blocked there.
- Treat dirty project-file changes, local commits, implementation artifacts, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- If a local commit appears, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check.

## Independent verification focus

- Study the task, inspect `PLAN.md` when it exists, inspect `INVESTIGATION.md` if it exists, account for any user-supplied implementation hint/comment if present, review the latest Claude commit, and independently verify as many central implementation claims as practical against the repository, local materials, current branch state, relevant tests, and git history.

## Review-loop focus

- Focus review aggressively and skeptically on bugs, regressions, missing tests, mismatches with the active scope source, contradictions with `INVESTIGATION.md`, contradictions with any user-supplied implementation hint/comment, weak assumptions, and anything important that Claude failed to notice on its own.
- Do not patch project code locally while reviewing. You may inspect files, run builds, run tests, run formatting/lint checks, and update workflow support files, but implementation fixes must go back to Claude for evaluation.
- For each review round, first clear `./.codex/inbox.md` locally with `truncate -s 0` and verify that the inbox size is `0`, then write a consultative review there that lists findings, risks, objections, and independently discovered facts or counterexamples, and restate any user-supplied implementation hint/comment that the current implementation ignored or contradicted.
- Ask Claude to either update the implementation or explain why it disagrees in `./.codex/outbox.md`.
- Require Claude in review rounds to read from inbox, write to outbox, commit after changing project files, keep changed tests self-contained and portable, rerun those tests, then run `cargo fmt --all`, then `make clippy`, or explain in outbox why `make clippy` could not be run.

## Deliverable

Report to the user:

- the workflow stop reason; if the stop reason was reaching the quality bar, state that explicitly and include the achieved quality assessment;
- what Claude implemented;
- what review findings remain, including any explicit disagreements;
- what was verified locally;
- the last relevant Claude commit hash;
- whether the workflow stopped because Claude could not complete the work for any reason.
