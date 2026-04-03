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
- If `CLAUDE_SESSION.json` does not exist, resolve the current Codex session name via the exact lookup `CODEX_THREAD_ID` -> matching `id` in `$HOME/.codex/session_index.jsonl` -> `thread_name`. Do not read Codex logs and do not broad-search `~/.codex` for this lookup.
- Search the current project's Claude logs for candidate sessions whose `customTitle` or `agentName` matches the current Codex session name. The workflow reference may replace or tighten this matching logic; follow any workflow-specific override when it is present.
- If there is exactly one unambiguous Claude session candidate, use that `session_uuid` for continuation.
- If there are zero or multiple candidates, ask the user what to do. Never create a new Claude session without explicit user approval.

## Session creation

- As soon as the real Claude `session_uuid` is known, if `CLAUDE_SESSION.json` does not exist yet, create it immediately with `session_uuid`, `created_at`, and any workflow-specific fields that are already known or immediately derivable at that moment, then commit it immediately. Do not wait for the first Claude response.
- Only create a new Claude session after explicit user approval.
- When new-session approval is granted, start Claude once in batch mode with `claude -p --dangerously-skip-permissions --permission-mode acceptEdits ...`, determine `session_uuid` from the project logs as soon as it becomes available, create and commit `CLAUDE_SESSION.json`, and only then wait for outbox or other external completion signals.

## Request transport

- `claude -p` without `--resume` is allowed only when the user explicitly approved creating a new Claude session for the current workflow run.
- Recommended first headless bootstrap command in a trusted local repository:
  `claude -p --dangerously-skip-permissions --permission-mode acceptEdits "<prompt>"`
- Recommended batch continuation command once the session UUID is known:
  `claude -p --resume <session_uuid> --dangerously-skip-permissions --permission-mode acceptEdits "<prompt>"`
- Use `claude -p -c "<prompt>"` when the explicit goal is to continue the current project conversation from the current directory without first resolving a session UUID.
- Important limitation: in batch mode, `claude -p --resume <value>` requires a real session UUID and does not accept a human-readable alias.
- If a real session UUID is needed beyond the standard discovery rules, find it under `$HOME/.claude/projects/<project_key>/` and then use `claude -p --resume <session_uuid> ...`.
- Before every Claude request, overwrite `./.codex/inbox.md`, clear `./.codex/outbox.md` with `truncate -s 0`, and immediately verify that the outbox size is `0`.
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
