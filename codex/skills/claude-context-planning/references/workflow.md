# Workflow

This workflow adds context-based planning bootstrap, stricter Claude session matching, planning-specific prompt requirements, and the final reporting contract on top of the shared delegated-agent and Claude-provider references.

## Planning inputs and existing-result reuse

- If `INVESTIGATION.md` exists in the repository root, study it before any planning review and treat it as required input for `PLAN.md`.
- If `PLAN.md` already exists in the repository root, study it before any new Claude planning round and treat it as the current plan to review and refine rather than ignore.
- Before sending the first new Claude message in this workflow run, inspect the current branch for an already-produced planning result from an earlier Claude round.
- If a non-empty `PLAN.md` relevant to the current context already exists, review it immediately.
- If it already meets the quality bar, stop and report without contacting Claude further.
- Otherwise keep the findings and use them as the first Claude inbox message instead of a fresh `/plan-from-context` bootstrap or generic planning request.

## Claude session matching override

- When searching the current project's Claude logs, do not grep raw text for historical mentions of the current Codex session name and do not load whole session logs into model context.
- Use local shell tooling to extract only a compact per-file summary: session UUID, latest `type == "custom-title"` value, latest `type == "agent-name"` value, and optional recency metadata.
- In each file, track the latest `custom-title` value and the latest `agent-name` value, then treat only those latest values as the session's current effective names.
- Match the current Codex session name only by exact equality against those latest values.
- Do not special-case or filter out custom names such as `--trash-*`; if that is the current exact name, it is a valid match.
- Use file mtime or other recency signals only as a tie-breaker after exact current-name matching, never before it.
- This override replaces the default session-discovery matching from `../../.shared/delegated-agent/providers/claude/session.md`.

## New-session bootstrap

- When the user explicitly approves creating a new Claude session, bootstrap with a English request that tells Claude to `Выполни /plan-from-context.` and includes the current prompt/context below it.
- That bootstrap must tell Claude not to commit inbox/outbox, to commit planning results, never to push automatically or without an explicit user command, to write minimal status in English to `./.codex/outbox.md`, to keep resulting human-facing workflow artifacts/output in English by default, and to keep any code comments added or edited in project files in English only.
- If the existing `PLAN.md` already produced review findings, use that first consultative review message instead of the fresh bootstrap request.

## Planning-specific request requirements

- In addition to the shared Claude transport contract, every outer prompt for this workflow must also require Claude to:
- evaluate the request, existing plan state, and context as a colleague and explain mismatches or weak assumptions in outbox instead of silently diverging;
- read `INVESTIGATION.md` when it exists and account for its confirmed findings in `PLAN.md`;
- read `PLAN.md` first when it exists and treat it as the current plan to review and refine rather than ignore;
- account for the freeform prompt/context from the inbox request explicitly in the plan;
- propose as many practical automated tests as possible for the new and affected functionality, grouped into three categories: `1)` no refactoring required, `2)` light refactoring required, and `3)` heavy refactoring required;
- list all three categories and explicitly mark category `1` as planned for implementation together with the main functional changes;
- for changes to existing functionality, still add tests when current coverage is insufficient or adapt existing tests to the new logic;
- keep planned tests self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state;
- make a commit after any project file changes and report what changed plus the commit hash in outbox.

## Wait condition

- Wait until Claude writes `PLAN.md`, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`.
- Treat dirty project-file changes, local commits, `PLAN.md`, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait up to `2` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, wait up to `2` minutes for `./.codex/outbox.md` before reviewing the commit without outbox.

## Independent verification focus

- Study `PLAN.md`, inspect `INVESTIGATION.md` too if it exists, compare both to the supplied prompt/context, and independently verify as many central claims as practical against the repository, local materials, current branch state, relevant tests, and git history.

## Review-loop focus

- Review the plan aggressively and skeptically.
- Look for flaws, missing pieces, weak assumptions, insufficient test-coverage expectations, inconsistencies, contradictions with `INVESTIGATION.md`, contradictions with the supplied prompt/context, and anything important that Claude failed to notice on its own.
- For each review round, overwrite `./.codex/inbox.md` with a consultative review that lists findings, risks, objections, investigation findings or prompt facts that the plan ignored or contradicted, and independently discovered facts or counterexamples, then ask Claude to either update `PLAN.md` or explain why it disagrees in outbox.
- Do not patch `PLAN.md` locally while reviewing; send findings back to Claude for evaluation.
- Keep local changes limited to workflow/support files such as `./.codex/inbox.md`, `./.codex/outbox.md`, and `CLAUDE_SESSION.json`.

## Deliverable

Report to the user:

- the final plan quality assessment;
- what major issues were found and whether they were fixed or explicitly disputed;
- the last relevant Claude commit hash if Claude committed changes;
- whether the workflow stopped because Claude could not complete the work for any reason.
