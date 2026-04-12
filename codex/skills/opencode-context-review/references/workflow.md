# Workflow

This workflow adds review-target semantics, OpenCode review-prompt requirements, and the final reporting contract on top of the shared delegated-agent and OpenCode-provider references.

## Review-target semantics and existing-result reuse

- Require a non-empty `review_hint`. Treat it as the authoritative description of the review target and desired focus.
- If `PLAN.md` or `INVESTIGATION.md` exist in the repository root and clearly relate to the same current-branch changes under review, study them as supporting context before the first substantive review pass.
- Before sending the first new OpenCode message in this workflow run, inspect whether there is already a substantive OpenCode review result for the same target.
- If that existing review already meets the quality bar, stop and report without contacting OpenCode further.
- Otherwise keep the findings and use them as the first OpenCode request instead of a fresh bootstrap.

## OpenCode session matching

- Use the default OpenCode session discovery rules from `../../.shared/delegated-agent/providers/opencode/session.md`.

## Session metadata discipline

- If an existing OpenCode session is reused and `OPENCODE_SESSION.json` does not exist yet, create and commit it before sending the first delegated request into that session.
- If a new OpenCode session is explicitly approved and bootstrapped, discover its real `session_id` immediately after bootstrap via the shared before/after discovery rules, create and commit `OPENCODE_SESSION.json` immediately, and only then continue normal waiting for outbox or other round results. Do not wait for turn completion before fixing the new session metadata.

## New-session bootstrap

- When the user explicitly approves creating a new OpenCode session, bootstrap with a direct English request that tells OpenCode to review the target described by `review_hint`.
- Do not pass `./.codex/inbox.md` through `-f` or `--file`. Direct prompts in this workflow must instruct OpenCode to read `./.codex/inbox.md` from the current working directory.
- That bootstrap must tell OpenCode to read `./.codex/inbox.md`, read `./.codex/outbox.md` with the Read tool before resetting it, clear `./.codex/outbox.md` in place via truncation rather than deletion, not to modify project files during review unless the user explicitly asked for fixes, to commit and report the hash only if it still changes project files, never to push automatically or without an explicit user command, and to write its findings in English to `./.codex/outbox.md`.

## Review-specific request requirements

- In direct OpenCode prompts for this workflow, require OpenCode to:
- review the target described by `review_hint` from local repository context, local git history/diffs, and explicitly user-provided local materials;
- avoid remote systems or external environments unless the user explicitly asked for them;
- present findings first, ordered by severity, with file/line references when possible;
- separate confirmed findings from open questions or uncertainty;
- avoid modifying project files during review unless the user explicitly asked for fixes;
- read `./.codex/inbox.md` as the authoritative delegated request for the current round;
- read `./.codex/outbox.md` with the Read tool before resetting it, then clear it in place via truncation, and only then write the round result so the file contains only the latest response; never delete `./.codex/outbox.md`;
- create a commit and report the commit hash in `./.codex/outbox.md` if project file changes are made anyway;
- never push commits automatically or without an explicit user command.

## Wait condition

- Wait until OpenCode writes a substantive review to `./.codex/outbox.md` or clearly reports that it is blocked there.
- If project-file changes appear anyway, treat them as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- If a local commit appears, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check.
- If outbox and cheaper external signals are insufficient, run only a narrow `rg`/`grep` marker check on the freshest `1-2` OpenCode log files newer than the current request start before any export. Do not read raw log bodies in the normal path.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when those fresh-log checks are inconclusive, at most once every `5` minutes, by writing a stripped timestamped snapshot under `/tmp/`, reading only `root.messages[-1]` first, and reading `root.messages[-2]` only if `root.messages[-1]` is insufficient.

## Independent verification focus

- Independently inspect the review target described by `review_hint`.
- Use the relevant local evidence for that target:
- `git show`, `git diff`, `git log`, and local files for commit/branch/diff review;
- repository files, tests, and current branch state for code-path review;
- any explicitly user-provided local materials for surrounding context.

## Review-loop focus

- Review OpenCode's review aggressively and skeptically.
- Look for missed bugs, false positives, weak evidence, missing edge cases, unsupported claims, or anything important that OpenCode failed to notice on its own.
- For each review round, send a direct English consultative prompt through `opencode run -s <session_id> "<prompt>"`.
- For those direct review-round prompts, do not use `-f` or `--file` for `./.codex/inbox.md`; instruct OpenCode in the prompt itself to read `./.codex/inbox.md` from the current working directory.
- The prompt must list findings, risks, objections, facts from the local repo or local git state that OpenCode ignored or contradicted, and independently discovered facts or counterexamples, then ask OpenCode to either update its review in `./.codex/outbox.md` or explain why it disagrees there.
- Do not edit project files locally as part of the review. Keep local changes limited to workflow/support files such as `OPENCODE_SESSION.json`.

## Deliverable

Report to the user:

- the workflow stop reason; if the stop reason was reaching the quality bar, state that explicitly and include the achieved quality assessment;
- the main review findings, or explicitly that no findings were found;
- what you independently verified yourself;
- any remaining disagreements with OpenCode;
- the last relevant OpenCode commit hash if OpenCode changed project files, or say that the review produced no project-file changes;
- whether the workflow stopped because OpenCode could not complete the work for any reason.
