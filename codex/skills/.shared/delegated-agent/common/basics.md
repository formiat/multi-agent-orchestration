# Delegated Agent Basics

These delegated-agent workflows keep Codex as the orchestrator/reviewer and use a local delegated agent session as the implementation, planning, investigation, or review worker. Follow these rules in every delegated-agent workflow unless the workflow reference tightens or overrides them.

## Preflight and interaction style

- Read the repository `AGENTS.md` before starting.
- If the repository defines extra startup context loading, do that first.
- Start in headless batch mode and keep the workflow batch-only. Do not use the provider's interactive TUI.
- Keep delegated-agent messages lean and non-duplicative. Include only the operative request, explicit user constraints or local context that the delegated agent cannot already derive, and persistent workflow requirements that still matter for the current round.
- When delegating an autonomous phase, prefer strict completion conditions so less monitoring is needed. Ask the delegated agent to finish the requested work, run the required verification, create a commit when project files changed, report the commit hash and executed verification commands, and then stop.
- In every delegated-agent interaction, use a consultative peer-to-peer tone. Present findings, risks, contradictions, and counterexamples as observations for the delegated agent to evaluate; ask it to either agree and update the artifact/result or disagree and explain why. Do not use one-way order-giving style.
- After every delegated-agent interaction, including the initial request, each review round, and each progress check, sleep at least `2` minutes before the next polling action, stdout inspection, session-history read, `git status`, `git log`, or other progress check.
- Cheap progress signals may be checked at most once every `2` minutes.

## Language defaults

- Write delegated-agent requests/prompts in English unless the workflow reference or the user explicitly requires another language.
- Require human-facing workflow artifacts/output to stay in English by default unless the user explicitly asked for another language or the target artifact is inherently not English.
- Require all code comments that the delegated agent adds or edits in project files to remain in English only.

## Review posture

- Do not trust the delegated agent's work or self-report by default. Independently verify as much as practical against the repository, local materials, git history, tests, and current branch state, then feed the findings back into review rounds.
- In review cycles, avoid monitoring while the delegated agent is editing. Treat dirty project-file changes, local commits, requested artifacts, and outbox writes as completion/result signals, not as signs of ongoing work.
- In code- or artifact-producing workflows, if dirty project-file changes appear, wait up to `2` minutes for a local commit before reviewing uncommitted changes. Review uncommitted changes only if that grace period expires without a commit.
- After a local commit appears in a code- or artifact-producing workflow, wait up to `2` minutes for `./.codex/outbox.md` to be written before reviewing the commit without outbox. Start review earlier only when outbox is already present.
- In review-only workflows whose main deliverable is outbox, wait for outbox first. If project-file changes appear anyway, apply the same `2`-minute commit grace and then the same `2`-minute outbox-after-commit grace before reviewing.
- Prefer one consolidated review pass over many tiny interruptions.
- Stop the workflow only when one of these conditions holds: quality reaches at least `8/10`; the disagreement is irreconcilable; five consecutive non-service-cap failures/restarts have occurred; or a service-cap condition blocks continuation. Do not stop for other reasons.
- If the delegated agent cannot complete the assigned work for any reason, surface that state to the user instead of silently converting the workflow into solo Codex work.
