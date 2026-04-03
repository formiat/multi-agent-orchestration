# Claude Progress, Polling, and Retries

Apply these rules in every `claude-*` workflow unless the workflow reference narrows the wait condition for a specific artifact.

## Progress signals

- After each new Claude request or review round is sent, sleep at least `2` minutes before the first progress check of any kind.
- Treat every Claude progress check as another interaction point. After any progress check, sleep at least `2` minutes before the next cheap progress check.
- After Claude starts working and is not blocked on approvals, leave it alone for an initial `5` minute cool-down.
- Keep the workflow batch-only. Do not open Claude TUI.
- Signs of ongoing work are limited to a still-running Claude batch process and recent Claude-side activity such as the session `.jsonl` file `mtime` or the timestamp of the latest `.jsonl` entry when that path is known.
- When the Claude session `.jsonl` path is already known, treat that file's modification time as another cheap progress signal before reading any `.jsonl` lines.
- Treat `./.codex/outbox.md` size/content, dirty worktree state, requested artifact files, and local commits as completion/result signals or signs of imminent completion, not as signs of ongoing work.
- Cheap progress/completion checks such as `ps`, `.jsonl` file `mtime`, outbox size/content, `git status --short`, `git log -1 --oneline`, and requested artifact presence/mtime may be checked at most once every `2` minutes.
- Use the Claude session `.jsonl` log only when those cheaper signals are insufficient.
- Read `.jsonl` at most once every `5` minutes.
- When `.jsonl` inspection is unavoidable, first read only the last `1` line.
- Read the penultimate line only if the last line is insufficient for the current check.
- Check whether Claude completed the round by inspecting the outbox size first.
- If batch stdout is silent for a while, do not treat that by itself as a hang. `claude -p` often prints nothing until completion.
- If batch stdout is noisy, truncated, or visually corrupted, treat the Claude `.jsonl` session log as the source of truth for the last assistant message, tool calls and tool results, whether Claude actually finished the turn, and commit hashes or build results that may not be legible in stdout.

## Hang detection and retries

- If no signs of work appear within `5` minutes after the current request was sent, or if previously observed work signals stop and remain absent for `5` minutes, do not kill Claude immediately. First check the Claude `.jsonl` file modification time when that path is known. If its `mtime` is recent, treat that as a sign of work and do not kill the process.
- If there are signs of work, allow up to `20` minutes from the moment the current request was sent before treating the run as a likely hang.
- Only if external signals are stale and the `.jsonl` `mtime` is also stale, inspect the latest Claude `.jsonl` entry minimally and confirm that it is also older than the same threshold.
- Maintain a consecutive restart counter for the current Claude work item.
- Only when both conditions hold, no signs of ongoing work and a stale latest `.jsonl` entry, treat the situation as a likely hang: terminate the stale Claude process/session first, then resend the exact same request.
- For any failure other than a service-cap condition, including likely hangs, crashes, and mid-round exits without the requested result, resend the exact same inbox contents and the same outer prompt template without narrowing or rewriting them. Retry up to five consecutive times.
- Reset the consecutive restart counter to `0` only after a successful Claude completion that produces the requested result for that round. When the workflow expects both a commit and outbox, wait for both before treating the round as successful.
- If five consecutive restarts still do not produce the requested result, stop the workflow and report that Claude could not complete the work in the current scenario.
- If Claude stops because of rate limits, quota exhaustion, or another service-cap error, stop immediately and report that state to the user. Do not continue the delegated work yourself.

## Result and review gating

- Do not start review when dirty project-file changes first appear. Wait up to `2` minutes for a local commit before reviewing uncommitted changes.
- If a local commit appears, do not start review immediately. Wait up to `2` minutes for `./.codex/outbox.md` to be written before reviewing the commit without outbox.
- Outbox remains the preferred start signal for review. Review dirty changes or a commit without outbox only after the relevant `2`-minute grace period expires.

## Operational discipline

- Do not leave stale Claude sessions or stale `claude -p` processes running in parallel unless there is a reason. Old leftover sessions can keep consuming CPU and make it harder to understand which session is actually doing work.
- Optimize for low-observation workflows. Do not continuously watch batch output during long autonomous work phases; continuous polling adds little signal and wastes tokens.
- Use sparse or increasing polling intervals rather than frequent checks. A practical default is first rely on work signals only, then make the first substantive hang check around `5` minutes, treat `5` minutes without work signals as a likely hang after confirmation, and otherwise allow up to `20` minutes from request start before restarting when work signals continue.

## JSONL fallback discipline

- Treat the Claude `.jsonl` log as a fallback analogous to OpenCode export, not as a default reading channel.
- Prefer checking the `.jsonl` file modification time before reading any `.jsonl` content.
- The allowed `.jsonl` read cadence is at most once every `5` minutes.
- When `.jsonl` is read, inspect only the latest line first.
- Inspect the penultimate line only if the latest line is insufficient.
- Do not scan older lines and do not load the full `.jsonl` log into model context.
