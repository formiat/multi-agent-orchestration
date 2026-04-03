# OpenCode Session and Transport

Apply these rules in every `opencode-*` workflow unless the workflow reference adds stricter session-discovery logic.

## Working files

- Session metadata lives in `./OPENCODE_SESSION.json`.
- The delegated request file is `./.codex/inbox.md`.
- The delegated response file is `./.codex/outbox.md`.
- Never commit `./.codex/inbox.md` or `./.codex/outbox.md`.
- Treat `opencode run -s <session_id> ...` as a batch continuation that uses the specified OpenCode session as execution context.
- Do not assume that `opencode run -s <session_id> ...` will append visible turns to the same interactive/UI transcript in a user-observable way.
- The absence of a newly visible chat message in OpenCode UI/history is not, by itself, evidence that the request was not sent or not executed.
- Do not use OpenCode stdout as a response channel, progress channel, or review channel. Ignore stdout content except for bare process liveness/exit behavior.
- Treat `opencode export <session_id>` as a strictly discouraged fallback only when outbox and cheaper external signals are insufficient.
- Never read the full export by default. When export is unavoidable, first write a fresh timestamped snapshot to `/tmp/`, then read only `root.messages[-1]`. Read `root.messages[-2]` only if `root.messages[-1]` is insufficient for the current check. Inside each message object, the role is at `message.info.role` and any text parts are under `message.parts[*].text`.
- Use `opencode session list --format json` for discovery. Once the current `session_id` is already known, do not keep reading the whole session list into model context during polling; instead extract only the one relevant session object by UUID, or ideally only its top-level numeric `updated` field. Preferred pattern:
  `sid=$(jq -r '.session_id' OPENCODE_SESSION.json) && opencode session list --format json | jq -r --arg sid "$sid" '.[] | select(.id == $sid) | .updated'`
  If the whole current-session object is needed, use:
  `sid=$(jq -r '.session_id' OPENCODE_SESSION.json) && opencode session list --format json | jq -c --arg sid "$sid" '.[] | select(.id == $sid)'`
- When `session_id` is known, treat only that single matching session object as relevant. Do not read unrelated session objects into the model during normal polling.
- Fresh mtimes on OpenCode log files under `$HOME/.local/share/opencode/log/` count as cheap provider-side activity signals. Before any export, use only a narrow `rg`/`grep` marker check on the freshest relevant logs rather than reading raw log bodies.

## Session reuse

- Before any OpenCode interaction, read the current git branch and check whether `OPENCODE_SESSION.json` already exists.
- If `OPENCODE_SESSION.json` exists, parse it and reuse `session_id`.
- If `OPENCODE_SESSION.json` does not exist, resolve the current Codex session name via the exact lookup `CODEX_THREAD_ID` -> matching `id` in `$HOME/.codex/session_index.jsonl` -> `thread_name`. Do not read Codex logs and do not broad-search `~/.codex` for this lookup.
- Search `opencode session list --format json` for candidate sessions whose `directory` equals the current repository root and whose `title` equals the current Codex session name. This full-list read is for discovery only.
- If there is exactly one unambiguous OpenCode session candidate, use that `session_id` for continuation.
- If there are zero or multiple candidates, ask the user what to do. Never create a new OpenCode session without explicit user approval.

## Session creation

- Only create a new OpenCode session after explicit user approval.
- When new-session approval is granted, create a titled batch session in the current repository using the workflow-specific `opencode run --title ...` bootstrap command or direct batch prompt.
- Determine the real `session_id` from `opencode session list --format json` as soon as it becomes available.
- As soon as the real OpenCode `session_id` is known, if `OPENCODE_SESSION.json` does not exist yet, create it immediately with `session_id`, `created_at`, and any workflow-specific fields that are already known or immediately derivable at that moment, then commit it immediately. Do not wait for the first OpenCode response.
- After `OPENCODE_SESSION.json` exists, use its UUID to query only the one relevant session object in subsequent checks instead of re-reading the whole list into model context.

## Request transport

- Prefer batch continuation via `opencode run -s <session_id> ...` once the session id is known.
- Never run more than one `opencode run` concurrently against the same `session_id`. Serialize all work and all retries per OpenCode session.
- For OpenCode, treat requested artifacts, `./.codex/outbox.md`, dirty worktree state, and local commits as completion/result signals. They are not signs of ongoing work. Do not use UI/transcript visibility as a success criterion.
- Use `--command <name> ...` when the workflow bootstrap maps directly to an existing OpenCode command such as `take-task`, `investigate-bug`, `investigate-from-context`, or `plan-from-context`.
- Use direct freeform prompts through `opencode run -s <session_id> "<prompt>"` for review rounds and for workflows with no dedicated command bootstrap.
- Before every OpenCode request, overwrite `./.codex/inbox.md`, clear `./.codex/outbox.md` with `truncate -s 0`, and immediately verify that the outbox size is `0`.
- After every local create, overwrite, truncation, or other modification of `./.codex/inbox.md` or `./.codex/outbox.md`, immediately run `sudo chmod -R 777 ./.codex/`. This is a local Codex-side hygiene step, not an instruction for OpenCode.
- When asking OpenCode itself to reset `./.codex/outbox.md`, explicitly require this sequence: first read `./.codex/outbox.md` with the Read tool, then clear it in place with truncation, then write the new result. Never tell OpenCode to delete the file and never use `rm -f` for outbox resets.
- `./.codex/inbox.md` is a working file in the repository, not an OpenCode CLI parameter. Do not pass it through `-f`, do not pass its contents as a synthetic file argument, and do not try to deliver it through attach-style CLI transport.
- For OpenCode workflows, the correct transport is: write `./.codex/inbox.md` locally, then run `opencode run ...` normally, and instruct OpenCode in the prompt or command bootstrap to read `./.codex/inbox.md` from the current working directory.
- Every outer OpenCode prompt or command bootstrap must explicitly tell OpenCode which file to read and which file to write.
- Every outer OpenCode prompt or command bootstrap must tell OpenCode to read `./.codex/outbox.md` with the Read tool before resetting it, then clear it in place via truncation, and only then write its answer so the file contains only the latest response. Never tell OpenCode to delete `./.codex/outbox.md`.
- Every direct OpenCode prompt must explicitly say never to push commits automatically or without an explicit user command.
- For direct prompts and command bootstraps, require a concise English status/summary in `./.codex/outbox.md` at the end and require commit hashes and executed verification commands to be reported there when relevant.
- If an OpenCode command such as `plan-from-context` or `investigate-from-context` already knows to read `./.codex/inbox.md`, prefer invoking that command plainly and do not duplicate the inbox delivery through CLI file attachments.
- Use `opencode export <session_id>` only as a strictly discouraged fallback to recover the latest assistant message or minimal session state when outbox and cheaper signals are insufficient.
- When using export, do it at most once every `5` minutes, always write the stripped JSON snapshot to a new timestamped file under `/tmp/`, and never read the whole session dump; first extract only `root.messages[-1]`, and read `root.messages[-2]` only if `root.messages[-1]` is insufficient, or otherwise extract the smallest metadata slice that answers the current question.
- If a reused OpenCode session reads from or writes to inbox/outbox files other than the repository-local `./.codex/inbox.md` and `./.codex/outbox.md`, stop reusing that session for this workflow.
- When this file-routing failure is identified, stop reusing the session immediately. Do not try to salvage it with more delegated work. Ask the user whether to create a fresh OpenCode session or choose another existing one.
