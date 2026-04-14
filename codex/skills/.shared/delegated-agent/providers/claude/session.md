# Claude Session and Transport

Apply these rules in every `claude-*` workflow unless the workflow reference adds stricter session-discovery logic.

## Working files

- Session metadata lives in `./CLAUDE_SESSION.json`.
- The delegated request file is `./.codex/inbox.md`.
- The delegated response file is `./.codex/outbox.md`.
- Never commit `./.codex/inbox.md` or `./.codex/outbox.md`.

## Session reuse

- Always run Claude from the current repository directory.
- Before any Claude interaction, read the current git branch and check whether `CLAUDE_SESSION.json` already exists.
- If `CLAUDE_SESSION.json` exists, parse it and reuse `session_uuid`.
- If `CLAUDE_SESSION.json` exists, session discovery by name is forbidden; do not scan project logs for alternative candidates in this case.
- If `CLAUDE_SESSION.json` does not exist, resolve the current Codex session name via the exact lookup `CODEX_THREAD_ID` -> matching `id` in `$HOME/.codex/session_index.jsonl` -> `thread_name`. Do not read Codex logs and do not broad-search `~/.codex` for this lookup.
- Search the current project's top-level Claude session logs (`$HOME/.claude/projects/<project_key>/*.jsonl`) for candidate sessions whose latest effective title matches the current Codex session name. Do not use sidechain/service subdirectories for this lookup.
- For Claude project `.jsonl` logs, extract names only from the real JSON keys `customTitle` for `type == "custom-title"` and `agentName` for `type == "agent-name"`. Do not treat missing generic keys such as `title`, `name`, or `value` as authoritative for this lookup.
- In each session file, compute one current effective title: prefer the latest `custom-title`; use the latest `agent-name` only when no `custom-title` exists in that session file.
- Match candidates only by exact equality against that single effective title (not by union of both fields).
- First collect only exact current-title matches. If there is exactly one exact-match candidate, use that `session_uuid` for continuation.
- If there are multiple exact-match candidates, choose the freshest exact-match candidate by session-file `mtime`. Treat this as the default deterministic tie-breaker rather than as ambiguity that requires a new session.
- If discovery produced a deterministic existing-session winner under these rules, creating a new Claude session is forbidden. Reuse the winner.
- Only if multiple exact-match candidates remain genuinely indistinguishable after the recency tie-breaker, or if the discovery output is clearly incomplete/corrupted, ask the user what to do.
- Never treat a parser/extraction failure as evidence that there are zero candidates. If the extraction script is suspect, fix or rerun discovery first. Never create a new Claude session without explicit user approval.
- If an existing Claude session is discovered and `CLAUDE_SESSION.json` does not exist yet, create and commit `CLAUDE_SESSION.json` immediately before sending the first delegated request into that existing session. Do not send the first work request and only then persist the reused `session_uuid`.

## Session creation

- As soon as the real Claude `session_uuid` is known, if `CLAUDE_SESSION.json` does not exist yet, create it immediately with `session_uuid`, `created_at`, and any workflow-specific fields that are already known or immediately derivable at that moment, then commit it immediately. Do not wait for the first Claude response.
- Only create a new Claude session after explicit user approval.
- When new-session approval is granted, start Claude once in batch mode with `claude -p --dangerously-skip-permissions --permission-mode acceptEdits ...`, determine `session_uuid` from the project logs as soon as it becomes available, create and commit `CLAUDE_SESSION.json`, and only then wait for outbox or other external completion signals.
- For a newly created Claude session, determine `session_uuid` by before/after project-log discovery rather than by loose name matching:
  1. Before bootstrap, snapshot the set of existing project session files under `$HOME/.claude/projects/<project_key>/*.jsonl`.
  2. Run the approved `claude -p ...` bootstrap command.
  3. After bootstrap starts, look again only under that same project directory.
  4. Prefer a newly appeared top-level session `.jsonl` file that was absent from the pre-bootstrap snapshot; its basename without `.jsonl` is the new `session_uuid`.
  5. If multiple new files appeared, choose the freshest by `mtime`.
  6. If no new file appeared, use the freshest top-level session file whose `mtime` advanced after bootstrap and whose current effective title (latest `custom-title`, else latest `agent-name`) still matches the current Codex thread name.
  7. Never identify a newly created Claude session by broad historical grep across old session files.
- For a newly created Claude session, persist and commit `CLAUDE_SESSION.json` immediately after the new `session_uuid` is discovered, even if the current Claude turn is still running. Do not wait for turn completion, outbox, commit output, or any other round result before fixing the new session metadata.

## Request transport

- Every outer Claude prompt must include the shared `Executor mandatory checklist` from `../../common/basics.md` as explicit requirements for the delegated run.
- `claude -p` without `--resume` is allowed only when the user explicitly approved creating a new Claude session for the current workflow run.
- Recommended first headless bootstrap command in a trusted local repository:
  `claude -p --dangerously-skip-permissions --permission-mode acceptEdits "<prompt>"`
- Recommended batch continuation command once the session UUID is known:
  `claude -p --resume <session_uuid> --dangerously-skip-permissions --permission-mode acceptEdits "<prompt>"`
- Use `claude -p -c "<prompt>"` when the explicit goal is to continue the current project conversation from the current directory without first resolving a session UUID.
- Important limitation: in batch mode, `claude -p --resume <value>` requires a real session UUID and does not accept a human-readable alias.
- If a real session UUID is needed beyond the standard discovery rules, find it under `$HOME/.claude/projects/<project_key>/` and then use `claude -p --resume <session_uuid> ...`.
- Before every Claude request, first clear `./.codex/inbox.md` locally with `truncate -s 0` and immediately verify that the inbox size is `0`, then write the new delegated request into `./.codex/inbox.md`. Also clear `./.codex/outbox.md` with `truncate -s 0` and immediately verify that the outbox size is `0`.
- After every local create, overwrite, truncation, or other modification of `./.codex/inbox.md` or `./.codex/outbox.md`, immediately run `sudo chmod -R 777 ./.codex/`. This is a local Codex-side hygiene step, not an instruction for Claude.
- Every outer Claude prompt must explicitly tell Claude which file to read and which file to write.
- Every outer Claude prompt must tell Claude to clear `./.codex/outbox.md` before writing its answer so the file contains only the latest response.
- Every outer Claude prompt must explicitly say never to push commits automatically or without an explicit user command.
- Treat requested artifacts, `./.codex/outbox.md`, dirty worktree state, and local commits as completion/result signals. They are not signs of ongoing work.
- Workflow references add artifact-specific requirements such as commits, tests, formatting, or review output shape.

## Shell-command hygiene

- Avoid sending Claude shell commands that contain `$()`, heredocs, multiline segments, or other constructs that often trigger extra confirmations. Rewrite them into simpler commands when possible.
- If you know in advance that the workflow will require a commit or another approved shell step, prefer simpler commands over compound ones when possible.
- Claude may ask for shell-command approval even after code changes are done, especially for compound commands such as `cd ... && git ...` or heredoc-based commit messages. Expect an extra confirmation step before the final commit or verification command.
