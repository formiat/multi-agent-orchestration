# Workflow

This workflow adds context-based investigation request rules, investigation-specific prompt requirements, and the final reporting contract on top of the shared delegated-agent and Claude-provider references.

## Existing-result reuse

- Before sending the first new Claude message in this workflow run, inspect the current branch for an already-produced investigation result from an earlier Claude round.
- If a non-empty `INVESTIGATION.md` relevant to the current context already exists, review it immediately.
- If it already meets the quality bar, stop and report without contacting Claude further.
- Otherwise keep the findings and use them as the first Claude inbox message for the current session.

## Claude session matching

- Use the default Claude session discovery rules from `../../.shared/delegated-agent/providers/claude/session.md`.

## Session metadata discipline

- If `CLAUDE_SESSION.json` already exists, reuse only its `session_uuid` and do not run session discovery by name in this workflow.
- If discovery is used and yields a deterministic existing-session winner, creating a new Claude session is forbidden.
- If an existing Claude session is reused and `CLAUDE_SESSION.json` does not exist yet, create and commit it before sending the first delegated request into that session.
- Creating a new Claude session/chat is strictly forbidden. If no deterministic existing session can be found by name, stop the workflow and report to the user.

## Investigation-specific request requirements

- In addition to the shared Claude transport contract, every outer prompt for this workflow must also require Claude to:
- evaluate the request and current evidence as a colleague and explain mismatches or weak assumptions in outbox instead of silently diverging;
- create or update `INVESTIGATION.md`;
- treat an existing `INVESTIGATION.md` as the current investigation state to refine or correct rather than ignore it;
- inspect the freeform prompt/context and any user-provided local materials described in the inbox request;
- inspect relevant existing tests where useful, identify coverage gaps, and record what tests a future fix should add or adapt;
- keep proposed future tests self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state;
- investigate only through local repository files and explicitly user-provided local materials;
- avoid `ssh`, `scp`, `sftp`, or remote `curl`/API calls unless the user explicitly asked for that;
- make a commit after any project file changes and report what changed plus the commit hash in outbox.

## Wait condition

- Wait until Claude writes `INVESTIGATION.md`, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`.
- Treat dirty project-file changes, local commits, `INVESTIGATION.md`, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- If a local commit appears, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check.

## Independent verification focus

- Inspect `INVESTIGATION.md` and independently verify as many central claims as practical against the repository, local materials, current branch state, relevant tests, git history, and the supplied context.
- Verify that facts, hypotheses, ruled-out explanations, and suspect commits are clearly separated.

## Review-loop focus

- Review the investigation aggressively and skeptically.
- Look for unsupported claims, missing evidence, contradictions, weak timelines, missing blame/history analysis, missing test-coverage implications, fix directions that are not grounded in the evidence, and anything important that Claude failed to discover on its own.
- For each review round, first clear `./.codex/inbox.md` locally with `truncate -s 0` and verify that the inbox size is `0`, then write a consultative review there that lists findings, risks, unsupported claims, objections, and independently discovered facts or counterexamples, and ask Claude to either update `INVESTIGATION.md` or explain why it disagrees in outbox.
- Do not patch `INVESTIGATION.md` locally while reviewing; send findings back to Claude for evaluation.
- Keep local changes limited to workflow/support files such as `./.codex/inbox.md`, `./.codex/outbox.md`, and `CLAUDE_SESSION.json`.

## Deliverable

Report to the user:

- the final investigation quality assessment;
- the workflow stop reason; if the stop reason was reaching the quality bar, state that explicitly and include the achieved quality assessment;
- the most likely root cause and strongest evidence;
- which hypotheses were ruled out and which remain open;
- the last relevant Claude commit hash if Claude committed changes;
- whether the workflow stopped because Claude could not complete the work for any reason.
