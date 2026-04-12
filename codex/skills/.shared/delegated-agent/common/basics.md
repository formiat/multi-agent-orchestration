# Delegated Agent Basics

These delegated-agent workflows keep Codex as the orchestrator/reviewer and use a local delegated agent session as the implementation, planning, investigation, or review worker. Follow these rules in every delegated-agent workflow unless the workflow reference tightens or overrides them.

## Preflight and interaction style

- Read the repository `AGENTS.md` before starting.
- If the repository defines extra startup context loading, do that first.
- Start in headless batch mode and keep the workflow batch-only. Do not use the provider's interactive TUI.
- Keep delegated-agent messages lean and non-duplicative. Include only the operative request, explicit user constraints or local context that the delegated agent cannot already derive, and persistent workflow requirements that still matter for the current round.
- When delegating an autonomous phase, prefer strict completion conditions so less monitoring is needed. Ask the delegated agent to finish the requested work, run the required verification, create a commit when project files changed, report the commit hash and executed verification commands, and then stop.
- In every delegated-agent interaction, use a consultative peer-to-peer tone. Present findings, risks, contradictions, and counterexamples as observations for the delegated agent to evaluate; ask it to either agree and update the artifact/result or disagree and explain why. Do not use one-way order-giving style.
- After every delegated-agent interaction, including the initial request, each review round, and each progress check, sleep at least `3` minutes before the next polling action, stdout inspection, session-history read, `git status`, `git log`, or other progress check.
- Terminal-state exception: when the executor has already exited without the required round result, or when a service-cap marker is already confirmed, skip the normal polling cooldown and branch immediately into diagnosis/stop-or-retry handling.
- Cheap progress signals may be checked at most once every `3` minutes.

## Language defaults

- Write delegated-agent requests/prompts in English unless the workflow reference or the user explicitly requires another language.
- Require human-facing workflow artifacts/output to stay in English by default unless the user explicitly asked for another language or the target artifact is inherently not English.
- Require all code comments that the delegated agent adds or edits in project files to remain in English only.

## Review posture

- Model the delegated executor with a small state machine. Stage 1 is analysis/thinking; its work signals are a live executor process and fresh provider-side activity such as log or session-activity timestamps. Stage 2 is result production; its result signals are dirty project files, local commits, and `./.codex/outbox.md`. Stage 2 may include editing, verification/finalization such as `fmt`/tests/build/clippy/commit, and the final outbox write. In normal completion, outbox is written and then the executor exits.
- Keep work signals and result signals separate. Dirty files, commits, and outbox are not signs of ongoing work; they are signs of produced result, imminent completion, or completion.
- Keep exactly one live delegated-agent executor process per work item and provider session. Do not start a second batch run for the same work item as a hedge, speculative retry, or overlapping poll. Before any retry or replacement run, first verify that the previous provider process for that same work item/session is no longer running, or terminate it explicitly.
- Long verification/finalization phases are normal. During `fmt`, tests, build, clippy, or commit creation, provider-side logs may be quiet for a while. When relevant, treat live child processes such as `cargo`, `rustc`, `clippy-driver`, `make`, `pytest`, or similar build/test commands as additional signs of ongoing work rather than as hangs.
- Distinguish these terminal states: successful completion, crashed/exited without the requested result, service cap, and likely hang. A live-but-unproductive loop still counts as a likely-hang scenario once the retry thresholds below are met.
- Treat a crashed/exited process with missing expected outputs (for example missing outbox and missing required commit/artifact for the current round) as an immediate failed attempt. Do not keep waiting under polling cadence once this terminal state is confirmed.
- After each delegated-agent batch process exits, keep `./.codex/outbox.md` as the primary result channel. Inspect that run's captured stdout only if outbox is still empty, missing, or clearly insufficient to explain the terminal state. Even then, do not read the full stdout dump into model context by default because it may be noisy and token-expensive; instead extract only a narrow terminal slice or run a compact marker search for decisive keywords such as service-cap, rate-limit, quota, permission denial, usage limit, or other explicit stop/failure markers. Use this post-exit stdout check only after process completion, not as a normal progress channel while the agent is still running.
- After any interrupted turn, aborted local wait command, or other orchestration-side interruption, re-check the real executor-process state before drawing conclusions or issuing a retry. Never assume that a previously observed PID has already exited just because the local orchestration step was interrupted.
- After any interrupted turn or aborted local wait command, also check and terminate stale local polling helpers such as `sleep` timers from the interrupted check loop before resuming orchestration.
- Do not trust the delegated agent's work or self-report by default. Independently verify as much as practical against the repository, local materials, git history, tests, and current branch state, then feed the findings back into review rounds.
- In review cycles, avoid monitoring while the delegated agent is editing. Treat dirty project-file changes, local commits, requested artifacts, and outbox writes as completion/result signals, not as signs of ongoing work.
- In code- or artifact-producing workflows, if dirty project-file changes appear, wait exactly `3` minutes before the next check for a local commit; review uncommitted changes only if the commit is still missing after that check.
- After a local commit appears in a code- or artifact-producing workflow, wait exactly `3` minutes before the next check for `./.codex/outbox.md`; review the commit without outbox only if outbox is still missing after that check. Start review earlier when outbox is already present.
- In review-only workflows whose main deliverable is outbox, wait for outbox first. If project-file changes appear anyway, apply the same `3`-minute commit grace and then the same `3`-minute outbox-after-commit grace before reviewing.
- Prefer one consolidated review pass over many tiny interruptions.
- Stop the workflow only when one of these conditions holds: quality reaches at least `8/10`; the disagreement is irreconcilable; five consecutive non-service-cap failures/restarts have occurred; or a service-cap condition blocks continuation. Do not stop for other reasons.
- If the delegated agent cannot complete the assigned work for any reason, surface that state to the user instead of silently converting the workflow into solo Codex work.
