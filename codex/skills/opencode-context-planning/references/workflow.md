# Workflow

This workflow adds context-based planning semantics, OpenCode command usage, review-loop rules, and the final reporting contract on top of the shared delegated-agent and OpenCode-provider references.

## Planning inputs and existing-result reuse

- If `INVESTIGATION.md` exists in the repository root, study it before drafting the plan. Treat its confirmed findings and ruled-out hypotheses as input constraints for planning.
- If `PLAN.md` already exists in the repository root, read it first and treat it as the current plan to review, refine, or partially rewrite rather than ignore.
- Before sending the first new OpenCode message in this workflow run, inspect whether the existing `PLAN.md` is already good enough for the supplied context.
- If it already meets the quality bar, stop and report without contacting OpenCode further.
- Otherwise keep the findings and use them as the first OpenCode request instead of a fresh bootstrap.

## OpenCode session matching

- Use the default OpenCode session discovery rules from `../../.shared/delegated-agent/providers/opencode/session.md`.

## New-session bootstrap

- When the user explicitly approves creating a new OpenCode session, bootstrap with:
  `opencode run --title "<codex_session_name>" --command plan-from-context`
- Do not pass `./.codex/inbox.md` through `-f` or `--file`. The command must read `./.codex/inbox.md` from the current working directory on its own.
- If the existing `PLAN.md` already produced review findings, use a direct English review prompt instead of the fresh command bootstrap.

## Context-planning-specific request requirements

- In direct OpenCode prompts for this workflow, require OpenCode to:
- evaluate the request, existing plan state, and context as a colleague and explain mismatches or weak assumptions explicitly;
- account for the freeform prompt/context explicitly in the plan;
- read `INVESTIGATION.md` when it exists and account for its confirmed findings in `PLAN.md`;
- preserve useful parts of an existing `PLAN.md`, remove wrong assumptions, and explicitly note what changed when rewriting it;
- propose as many practical automated tests as possible for the new and affected functionality, grouped into three categories: `1)` no refactoring required, `2)` light refactoring required, and `3)` heavy refactoring required;
- list all three categories and explicitly mark category `1` as planned for implementation together with the main functional changes;
- keep planned tests self-contained and portable rather than dependent on machine-specific local files, directories, or preexisting state;
- read `./.codex/inbox.md` as the authoritative delegated request for the current round;
- read `./.codex/outbox.md` with the Read tool before resetting it, then clear it in place via truncation, and only then write the round result so the file contains only the latest response; never delete `./.codex/outbox.md`;
- make a commit after any project file changes and report what changed plus the commit hash in `./.codex/outbox.md`;
- never push commits automatically or without an explicit user command.

## Wait condition

- Wait until OpenCode writes `PLAN.md`, creates a local commit without pushing it, and writes the round summary to `./.codex/outbox.md`.
- Treat dirty project-file changes, local commits, `PLAN.md`, and `./.codex/outbox.md` as completion/result signals, not as signs of ongoing work.
- If dirty project-file changes appear, wait up to `2` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, wait up to `2` minutes for `./.codex/outbox.md` before reviewing the commit without outbox.
- If outbox and cheaper external signals are insufficient, run only a narrow `rg`/`grep` marker check on the freshest `1-2` OpenCode log files newer than the current request start before any export. Do not read raw log bodies in the normal path.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when those fresh-log checks are inconclusive, at most once every `5` minutes, by writing a stripped timestamped snapshot under `/tmp/`, reading only `root.messages[-1]` first, and reading `root.messages[-2]` only if `root.messages[-1]` is insufficient.

## Independent verification focus

- Study `PLAN.md`, inspect `INVESTIGATION.md` too if it exists, compare both to the supplied prompt/context, and independently verify as many central claims as practical against the repository, local materials, current branch state, relevant tests, and git history.

## Review-loop focus

- Review the plan aggressively and skeptically.
- Look for flaws, missing pieces, weak assumptions, insufficient test-coverage expectations, inconsistencies, contradictions with `INVESTIGATION.md`, contradictions with the supplied context, and anything important that OpenCode failed to notice on its own.
- For each review round, send a direct English consultative prompt through `opencode run -s <session_id> "<prompt>"`.
- For those direct review-round prompts, do not use `-f` or `--file` for `./.codex/inbox.md`; instruct OpenCode in the prompt itself to read `./.codex/inbox.md` from the current working directory.
- The prompt must list findings, risks, objections, investigation findings or prompt facts that the plan ignored or contradicted, and independently discovered facts or counterexamples, then ask OpenCode to either update `PLAN.md` or explain why it disagrees in `./.codex/outbox.md`.
- Do not patch `PLAN.md` locally while reviewing; send findings back to OpenCode for evaluation.
- Keep local changes limited to workflow/support files such as `OPENCODE_SESSION.json`.

## Deliverable

Report to the user:

- the final plan quality assessment;
- what major issues were found and whether they were fixed or explicitly disputed;
- the last relevant OpenCode commit hash if OpenCode committed changes;
- whether the workflow stopped because OpenCode could not complete the work for any reason.
