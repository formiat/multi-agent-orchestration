# OpenCode Progress, Polling, and Retries

Apply these rules in every `opencode-*` workflow unless the workflow reference narrows the wait condition for a specific artifact.

## Progress signals

- After each new OpenCode request or review round is sent, sleep at least `2` minutes before the first progress check of any kind.
- Treat every OpenCode progress check as another interaction point. After any progress check, sleep at least `2` minutes before the next cheap progress check.
- After OpenCode starts working and is not blocked on approvals, leave it alone for an initial `5` minute cool-down.
- Keep the workflow batch-only. Do not open the interactive OpenCode TUI.
- Signs of ongoing work are limited to a still-running `opencode run` process and recent OpenCode-side activity timestamps such as the top-level numeric `updated` field for the one relevant current session object, looked up by UUID from `OPENCODE_SESSION.json`, plus fresh mtime changes on OpenCode log files under `$HOME/.local/share/opencode/log/`. Do not read the whole session list into model context during polling when the UUID is already known. Preferred pattern:
  `sid=$(jq -r '.session_id' OPENCODE_SESSION.json) && opencode session list --format json | jq -r --arg sid "$sid" '.[] | select(.id == $sid) | .updated'`
  Use export metadata only as a fallback when those are insufficient.
- Treat `./.codex/outbox.md` size/content, dirty worktree state, requested artifact files, and local commits as completion/result signals or signs of imminent completion, not as signs of ongoing work.
- Cheap progress/completion checks such as `ps`, the top-level numeric `updated` field for the one relevant current session object selected by UUID, fresh OpenCode log-file mtimes, outbox size/content, `git status --short`, `git log -1 --oneline`, and requested artifact presence/mtime may be checked at most once every `2` minutes.
- Do not use the absence of newly visible OpenCode chat history/transcript turns as a signal of failure or non-delivery.
- Do not inspect or rely on OpenCode stdout content during polling.
- If a completed OpenCode run is being diagnosed after failure, you may inspect its captured stderr/stdout once as a post-mortem artifact, but not as a normal progress channel.
- Before any export fallback, run only a narrow `rg`/`grep` check against the freshest `1-2` OpenCode log files under `$HOME/.local/share/opencode/log/` that are newer than the current request start time.
- Do not read raw log bodies during normal polling. Do not read the whole file by default and do not use broad searches that may return full prompt-bearing lines. Prefer compact extraction searches for service-cap markers such as `429`, `1308`, `1311`, `Usage limit reached`, `reset at`, and `subscription plan`.
- Read raw log lines only as an exceptional post-mortem fallback when the narrow marker check is inconclusive and a deeper diagnosis is still necessary.
- If those compact log checks confirm rate limits, quota exhaustion, plan-access denial, or another service-cap condition, stop immediately and report that state to the user. Do not retry and do not spend tokens on `opencode export`.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when cheaper signals and the compact fresh-log checks are still insufficient.
- Never read the full export repeatedly or by default. At most once every `5` minutes, create a new stripped JSON snapshot in `/tmp/` whose filename contains a timestamp, then inspect only `root.messages[-1]`; inspect `root.messages[-2]` only if `root.messages[-1]` is insufficient, or otherwise inspect only the minimal metadata slice needed for the current check.

## Hang detection and retries

- If no signs of work appear within `5` minutes after the current request was sent, or if previously observed work signals stop and remain absent for `5` minutes, do not kill OpenCode immediately. First confirm via a UUID-filtered lookup of the one relevant current session object from `opencode session list --format json` and fresh OpenCode log mtimes, or, only if needed, the latest export snapshot that OpenCode-side activity is also older than that same `5`-minute threshold.
- If there are signs of work, allow up to `20` minutes from the moment the current request was sent before treating the run as a likely hang.
- Maintain a consecutive restart counter for the current OpenCode work item.
- Before any retry, verify that there is no other still-running `opencode run` for the same `session_id`.
- If a run exits but the observed effect is that OpenCode did not use the repository-local `./.codex/inbox.md` and `./.codex/outbox.md` files as instructed, treat that as a failed attempt for the current work item, not as a completion signal.
- Only when both conditions hold, no signs of ongoing work and a stale OpenCode-side activity timestamp, treat the situation as a likely hang: terminate the stale `opencode run` process first when it is still running, then resend the exact same request.
- For any failure other than a service-cap condition, including likely hangs, crashes, file-routing failures, and mid-round exits without the requested result, resend the exact same request without narrowing or rewriting it. Retry up to five consecutive times.
- Reset the consecutive restart counter to `0` only after a successful completion that produces the requested result for that round. When the workflow expects both a commit and outbox, wait for both before treating the round as successful.
- If five consecutive restarts still do not produce the requested result, stop the workflow and report that OpenCode could not complete the work in the current scenario.
- If OpenCode stops because of rate limits, quota exhaustion, provider errors, or another service-cap error, stop immediately and report that state to the user. Do not continue the delegated work yourself.

## Result and review gating

- Do not start review when dirty project-file changes first appear. Wait up to `2` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, do not start review immediately. Wait up to `2` minutes for `./.codex/outbox.md` to be written before reviewing the commit without outbox.
- Outbox remains the preferred start signal for review. Review dirty changes or a commit without outbox only after the relevant `2`-minute grace period expires.

## Export fallback discipline

- When export is unavoidable, do not read from stdout and do not read from OpenCode's internal storage files directly.
- Do not treat the absence of your prompt in a user-visible OpenCode transcript as justification for retrying. Use outbox/artifacts/session-list/export policy instead.
- Before each export fallback, rerun the compact fresh-log check for service-cap markers and skip export entirely if that check already explains the failure.
- The allowed export cadence is at most one new snapshot every `5` minutes.
- Create each snapshot in `/tmp/` with a timestamped filename, for example `/tmp/opencode_export_<session_id>_<YYYYmmddTHHMMSS>.json`.
- Because `opencode export` prepends a non-JSON status line, strip that first line before treating the snapshot as JSON.
- When reading from the snapshot, inspect only `root.messages[-1]` first. Inspect `root.messages[-2]` only if `root.messages[-1]` is insufficient. Do not scan older messages and do not read the full export dump.
