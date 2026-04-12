# OpenCode Progress, Polling, and Retries

Apply these rules in every `opencode-*` workflow unless the workflow reference narrows the wait condition for a specific artifact.

## Progress signals

- Apply the shared delegated-agent state machine from `common/basics.md`. For OpenCode, provider-side work signals are specifically the still-running `opencode run` process, the `updated` field of the one relevant current session object, fresh OpenCode log-file mtimes, and, when relevant, live child verification/build/test processes.
- After each new OpenCode request or review round is sent, sleep at least `3` minutes before the first progress check of any kind.
- Treat every OpenCode progress check as another interaction point. After any progress check, sleep at least `3` minutes before the next cheap progress check.
- Cadence exception: if OpenCode already exited and the required round result is still missing (for example no outbox and no required commit/artifact), switch to immediate diagnosis without another `3`-minute wait.
- Do not infer hidden/background executors without direct process evidence. Treat fresh session/log events only as provider activity, not as proof of a separate worker process.
- If `opencode run` exits but session/log activity remains fresh for this round, keep the round alive and continue normal cadence polling on outbox plus provider activity signals.
- Classify immediate early-exit failure only when `opencode run` exited, required outputs are missing, and provider-side activity is stale or absent for this round.
- After OpenCode starts working and is not blocked on approvals, leave it alone for an initial `5` minute cool-down.
- Keep the workflow batch-only. Do not open the interactive OpenCode TUI.
- Signs of ongoing work are limited to a still-running `opencode run` process and recent OpenCode-side activity timestamps such as the top-level numeric `updated` field for the one relevant current session object, looked up by UUID from `OPENCODE_SESSION.json`, plus fresh mtime changes on OpenCode log files under `$HOME/.local/share/opencode/log/`. Do not read the whole session list into model context during polling when the UUID is already known. Preferred pattern:
  `sid=$(jq -r '.session_id' OPENCODE_SESSION.json) && opencode session list --format json | jq -r --arg sid "$sid" '.[] | select(.id == $sid) | .updated'`
  Use export metadata only as a fallback when those are insufficient.
- During long verification/finalization phases, also treat live child processes such as `cargo`, `rustc`, `clippy-driver`, `make`, `pytest`, or similar commands spawned under the OpenCode run as additional signs of ongoing work.
- Treat `./.codex/outbox.md` size/content, dirty worktree state, requested artifact files, and local commits as completion/result signals or signs of imminent completion, not as signs of ongoing work.
- Cheap progress/completion checks such as `ps`, the top-level numeric `updated` field for the one relevant current session object selected by UUID, fresh OpenCode log-file mtimes, outbox size/content, `git status --short`, `git log -1 --oneline`, and requested artifact presence/mtime may be checked at most once every `3` minutes.
- Do not use the absence of newly visible OpenCode chat history/transcript turns as a signal of failure or non-delivery.
- Do not inspect or rely on OpenCode stdout content during polling while the run is still active.
- After each completed `opencode run`, keep outbox as the primary result signal. Inspect its captured stdout/stderr once only when outbox is still empty, missing, or insufficient to explain the terminal state before deciding retry, stop reason, or success classification. Do not read the full stdout dump by default; prefer a compact marker search or minimal terminal slice for decisive phrases such as `Usage limit reached`, `rate limit`, `quota`, `subscription plan`, `permission denied`, provider errors, or other explicit terminal markers.
- Before any export fallback, run only a narrow `rg`/`grep` check against the freshest `1-2` OpenCode log files under `$HOME/.local/share/opencode/log/` that are newer than the current request start time.
- Do not read raw log bodies during normal polling. Do not read the whole file by default and do not use broad searches that may return full prompt-bearing lines. Prefer compact extraction searches for service-cap markers such as `429`, `1308`, `1311`, `Usage limit reached`, `reset at`, and `subscription plan`.
- Read raw log lines only as an exceptional post-mortem fallback when the narrow marker check is inconclusive and a deeper diagnosis is still necessary.
- If those compact log checks confirm rate limits, quota exhaustion, plan-access denial, or another service-cap condition, stop immediately and report that state to the user. Do not retry and do not spend tokens on `opencode export`.
- If the completed run's stdout/stderr already confirms a service-cap condition, stop immediately and report it. Do not retry, do not wait for more signals, and do not spend tokens on `opencode export`.
- Use `opencode export <session_id>` only as a strictly discouraged fallback when cheaper signals and the compact fresh-log checks are still insufficient.
- Never read the full export repeatedly or by default. At most once every `5` minutes, create a new stripped JSON snapshot in `/tmp/` whose filename contains a timestamp, then inspect only `root.messages[-1]`; inspect `root.messages[-2]` only if `root.messages[-1]` is insufficient, or otherwise inspect only the minimal metadata slice needed for the current check.

## Hang detection and retries

- If no signs of work appear within `5` minutes after the current request was sent, or if previously observed work signals stop and remain absent for `5` minutes, do not kill OpenCode immediately. First confirm via a UUID-filtered lookup of the one relevant current session object from `opencode session list --format json` and fresh OpenCode log mtimes, or, only if needed, the latest export snapshot that OpenCode-side activity is also older than that same `5`-minute threshold.
- If there are signs of work, allow up to `30` minutes from the moment the current request was sent before treating the run as a likely hang.
- Maintain a consecutive restart counter for the current OpenCode work item.
- Before any retry, verify that there is no other still-running `opencode run` for the same `session_id`. Use a self-filtering process query pattern such as `pgrep -a -f "[o]pencode run"` (and narrow by session when needed) to avoid matching the probe command itself.
- After any interrupted local wait/poll step or turn-abort, re-check the actual `opencode run` process list before concluding that OpenCode exited, hung, or needs a retry.
- After any interrupted local wait/poll step or turn-abort, also clear stale local wait helpers (for example leftover `sleep` processes started by the polling loop) before resuming checks.
- If a run exits but the observed effect is that OpenCode did not use the repository-local `./.codex/inbox.md` and `./.codex/outbox.md` files as instructed, treat that as a failed attempt for the current work item, not as a completion signal.
- Only when both conditions hold, no signs of ongoing work and a stale OpenCode-side activity timestamp, treat the situation as a likely hang: terminate the stale `opencode run` process first when it is still running, then resend the exact same request.
- For any failure other than a service-cap condition, including likely hangs, crashes, file-routing failures, and mid-round exits without the requested result, resend the exact same request without narrowing or rewriting it. Retry up to five consecutive times.
- Early-exit handling: if OpenCode process exit is confirmed and required outputs for the round are missing, classify immediately as failed attempt and retry immediately under the same retry budget only when provider-side activity for the round is also stale/absent.
- Reset the consecutive restart counter to `0` only after a successful completion that produces the requested result for that round. When the workflow expects both a commit and outbox, wait for both before treating the round as successful.
- If five consecutive restarts still do not produce the requested result, stop the workflow and report that OpenCode could not complete the work in the current scenario.
- If OpenCode stops because of rate limits, quota exhaustion, provider errors, or another service-cap error, stop immediately and report that state to the user. Do not continue the delegated work yourself.

## Result and review gating

- Do not start review when dirty project-file changes first appear. Wait up to `3` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, do not start review immediately. Wait up to `3` minutes for `./.codex/outbox.md` to be written before reviewing the commit without outbox.
- Outbox remains the preferred start signal for review. Review dirty changes or a commit without outbox only after the relevant `3`-minute grace period expires.

## Export fallback discipline

- When export is unavoidable, do not read from stdout and do not read from OpenCode's internal storage files directly.
- Do not treat the absence of your prompt in a user-visible OpenCode transcript as justification for retrying. Use outbox/artifacts/session-list/export policy instead.
- Before each export fallback, rerun the compact fresh-log check for service-cap markers and skip export entirely if that check already explains the failure.
- The allowed export cadence is at most one new snapshot every `5` minutes.
- Create each snapshot in `/tmp/` with a timestamped filename, for example `/tmp/opencode_export_<session_id>_<YYYYmmddTHHMMSS>.json`.
- Because `opencode export` prepends a non-JSON status line, strip that first line before treating the snapshot as JSON.
- When reading from the snapshot, inspect only `root.messages[-1]` first. Inspect `root.messages[-2]` only if `root.messages[-1]` is insufficient. Do not scan older messages and do not read the full export dump.
