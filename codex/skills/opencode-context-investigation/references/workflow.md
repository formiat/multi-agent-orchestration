# Workflow

This workflow adds context-based investigation semantics, OpenCode command usage, review-loop rules, and the final reporting contract on top of the shared delegated-agent and OpenCode-provider references.

## Existing-result reuse

- If `INVESTIGATION.md` already exists in the repo root, read it first and treat it as the current state of the investigation to refine or correct rather than ignore.
- Before sending the first new OpenCode message in this workflow run, inspect whether the existing `INVESTIGATION.md` is already good enough for the supplied context.
- If it already meets the quality bar, stop and report without contacting OpenCode further.
- Otherwise keep the findings and use them as the first OpenCode request instead of a fresh bootstrap.

## OpenCode session matching

- Use the default OpenCode session discovery rules from `../../.shared/delegated-agent/providers/opencode/session.md`.

## New-session bootstrap

- When the user explicitly approves creating a new OpenCode session, bootstrap with:
  `opencode run --title "<codex_session_name>" --command investigate-from-context`
- Do not pass `./.codex/inbox.md` through `-f` or `--file`. The command must read `./.codex/inbox.md` from the current working directory on its own.
- If the existing `INVESTIGATION.md` already produced review findings, use a direct English review prompt instead of the fresh command bootstrap.

## Context-investigation-specific request requirements

- In direct OpenCode prompts for this workflow, require OpenCode to:
- evaluate the request and current evidence as a colleague and explain mismatches or weak assumptions explicitly;
- create or update `INVESTIGATION.md`;
- inspect the freeform prompt/context and any user-provided local materials described there;
- inspect relevant existing tests where useful, identify coverage gaps, and record what tests a future fix should add or adapt;
- keep proposed future tests self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state;
- investigate only through local repository files and explicitly user-provided local materials;
- avoid `ssh`, `scp`, `sftp`, or remote `curl`/API calls unless the user explicitly asked for that;
- read `./.codex/inbox.md` as the authoritative delegated request for the current round;
- read `./.codex/outbox.md` with the Read tool before resetting it, then clear it in place via truncation, and only then write the round result so the file contains only the latest response; never delete `./.codex/outbox.md`;
- make a commit after any project file changes and report what changed plus the commit hash in `./.codex/outbox.md`;
- never push commits automatically or without an explicit user command.

## Wait condition

- Wait until OpenCode writes `INVESTIGATION.md`, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`.
- Treat dirty project-file changes, local commits, `INVESTIGATION.md`, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait up to `2` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, wait up to `2` minutes for `./.codex/outbox.md` before reviewing the commit without outbox.
- If outbox and cheaper external signals are insufficient, run only a narrow `rg`/`grep` marker check on the freshest `1-2` OpenCode log files newer than the current request start before any export. Do not read raw log bodies in the normal path.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when those fresh-log checks are inconclusive, at most once every `5` minutes, by writing a stripped timestamped snapshot under `/tmp/`, reading only `root.messages[-1]` first, and reading `root.messages[-2]` only if `root.messages[-1]` is insufficient.

## Independent verification focus

- Inspect `INVESTIGATION.md` and independently verify as many central claims as practical against the repository, local materials, current branch state, relevant tests, git history, and the supplied context.

## Review-loop focus

- Review the investigation aggressively and skeptically.
- Look for unsupported claims, missing evidence, contradictions, weak timelines, missing blame/history analysis, missing test-coverage implications, fix directions that are not grounded in the evidence, and anything important that OpenCode failed to discover on its own.
- For each review round, send a direct English consultative prompt through `opencode run -s <session_id> "<prompt>"`.
- For those direct review-round prompts, do not use `-f` or `--file` for `./.codex/inbox.md`; instruct OpenCode in the prompt itself to read `./.codex/inbox.md` from the current working directory.
- The prompt must list findings, risks, unsupported claims, objections, and independently discovered facts or counterexamples, then ask OpenCode to either update `INVESTIGATION.md` or explain why it disagrees in `./.codex/outbox.md`.
- Do not patch `INVESTIGATION.md` locally while reviewing; send findings back to OpenCode for evaluation.
- Keep local changes limited to workflow/support files such as `OPENCODE_SESSION.json`.

## Deliverable

Report to the user:

- the final investigation quality assessment;
- the most likely root cause and strongest evidence;
- which hypotheses were ruled out and which remain open;
- the last relevant OpenCode commit hash if OpenCode committed changes;
- whether the workflow stopped because OpenCode could not complete the work for any reason.
