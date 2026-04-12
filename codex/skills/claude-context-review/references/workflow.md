# Workflow

This workflow adds review-target semantics, review-specific prompt requirements, and the final reporting contract on top of the shared delegated-agent and Claude-provider references.

## Review-target semantics and existing-result reuse

- Require a non-empty `review_hint`. Treat it as the authoritative description of the review target and desired focus.
- If `PLAN.md` or `INVESTIGATION.md` exist in the repository root and clearly relate to the same current-branch changes under review, study them as supporting context before the first substantive review pass.
- Before sending the first new Claude message in this workflow run, inspect whether there is already a substantive Claude review result for the same target.
- Start with a cheap check such as a non-empty `./.codex/outbox.md` from the current run.
- If needed, inspect the current Claude session state minimally to determine whether Claude already produced review findings for the current `review_hint`.
- If that existing review already meets the quality bar, stop and report without contacting Claude further.
- Otherwise keep the findings and use them as the first Claude inbox message instead of a fresh bootstrap request.

## Claude session matching

- Use the default Claude session discovery rules from `../../.shared/delegated-agent/providers/claude/session.md`.

## Session metadata discipline

- If an existing Claude session is reused and `CLAUDE_SESSION.json` does not exist yet, create and commit it before sending the first delegated request into that session.
- If a new Claude session is explicitly approved and bootstrapped, discover its real `session_uuid` immediately after bootstrap via the shared before/after discovery rules, create and commit `CLAUDE_SESSION.json` immediately, and only then continue normal waiting for outbox or other round results. Do not wait for turn completion before fixing the new session metadata.

## New-session bootstrap

- When the user explicitly approves creating a new Claude session, bootstrap with a English request that tells Claude to review the target described by `review_hint`.
- That bootstrap must tell Claude not to commit inbox/outbox, not to modify project files during review unless the user explicitly asked for fixes, to commit and report the hash only if it still changes project files, never to push automatically or without an explicit user command, to write its findings in English to `./.codex/outbox.md`, to keep resulting human-facing workflow artifacts/output in English by default, and to keep any code comments added or edited in project files in English only.
- If an existing review already produced follow-up findings, use that consultative feedback message instead of the fresh bootstrap request.

## Review-specific request requirements

- In addition to the shared Claude transport contract, every outer prompt for this workflow must also require Claude to:
- review the target described by `review_hint` from local repository context, local git history/diffs, and explicitly user-provided local materials;
- avoid remote systems or external environments unless the user explicitly asked for them;
- present findings first, ordered by severity, with file/line references when possible;
- separate confirmed findings from open questions or uncertainty;
- avoid modifying project files during review unless the user explicitly asked for fixes;
- create a commit and report the commit hash in outbox if project file changes are made anyway.

## Wait condition

- Wait until Claude writes a substantive review to `./.codex/outbox.md` or clearly reports that it is blocked there.
- If project-file changes appear anyway, treat them as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- If a local commit appears, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check.

## Independent verification focus

- Independently inspect the review target described by `review_hint`.
- Use the relevant local evidence for that target:
- `git show`, `git diff`, `git log`, and local files for commit/branch/diff review;
- repository files, tests, and current branch state for code-path review;
- any explicitly user-provided local materials for surrounding context.

## Review-loop focus

- Review Claude's review aggressively and skeptically.
- Look for missed bugs, false positives, weak evidence, missing edge cases, unsupported claims, or anything important that Claude failed to notice on its own.
- For each review round, first clear `./.codex/inbox.md` locally with `truncate -s 0` and verify that the inbox size is `0`, then write a consultative review there that lists findings, risks, objections, facts from the local repo or local git state that Claude ignored or contradicted, and independently discovered facts or counterexamples, and ask Claude to either update its review in outbox or explain why it disagrees.
- Do not edit project files locally as part of the review. Keep local changes limited to workflow/support files such as `./.codex/inbox.md`, `./.codex/outbox.md`, and `CLAUDE_SESSION.json`.

## Deliverable

Report to the user:

- the workflow stop reason; if the stop reason was reaching the quality bar, state that explicitly and include the achieved quality assessment;
- the main review findings, or explicitly that no findings were found;
- what you independently verified yourself;
- any remaining disagreements with Claude;
- the last relevant Claude commit hash if Claude changed project files, or say that the review produced no project-file changes;
- whether the workflow stopped because Claude could not complete the work for any reason.
