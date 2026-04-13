# Multi-Agent Orchestration Workflows

This repository is a workflow library for running Codex as the primary orchestrator while delegating bounded planning, investigation, implementation, and review work to local Claude or OpenCode sessions.

The center of gravity is [`codex/skills`](./codex/skills): those skills define how Codex should open a task, discover or reuse delegated sessions, exchange work through inbox/outbox files, review delegated output, and decide when a workflow is complete. The `claude/` and `opencode/` trees complement that layer with provider-side commands and skill material that the delegated agents execute once Codex has handed work off.

## What This Repository Optimizes For

- Codex stays in charge of orchestration, validation, and final reporting.
- Delegated agents work in batch mode rather than interactive TUI mode.
- Planning and investigation artifacts are explicit and reviewable.
- Local commits are used as durable workflow milestones.
- Review loops are skeptical by default: delegated output is inspected, challenged, and iterated on until the quality bar is met.
- Task and branch handling are standardized so workflows can resume safely.

## Repository Layout

```text
.
├── codex/
│   └── skills/
│       ├── .shared/                    # shared transport, language, and review rules
│       ├── take-task/                  # direct Codex task intake and planning workflow
│       ├── update-mr/                  # direct Codex GitLab MR sync workflow
│       ├── claude-*/                   # Codex orchestration for delegated Claude workflows
│       └── opencode-*/                 # Codex orchestration for delegated OpenCode workflows
├── claude/
│   ├── commands/                       # provider-side Claude command bootstraps
│   └── skills/                         # provider-side Claude support material
└── opencode/
    ├── commands/                       # provider-side OpenCode command bootstraps
    └── skills/                         # provider-side OpenCode support material
```

## How Codex-Orchestrated Workflows Operate

All Codex workflow skills follow the same high-level pattern.

1. **Task or context intake**
   Codex identifies the active scope from a task id, current branch context, or an explicit review/implementation hint.

2. **Branch preflight**
   Task-oriented workflows normalize branch handling around:
   `task-<task_id>-<short-description>`
   If the current branch already belongs to the same task, it is reused. Otherwise a fresh branch is created from `origin/develop` without inheriting an upstream from `origin/develop`.

3. **Session discovery or reuse**
   Delegated workflows do not blindly create new sessions. Codex first tries to reuse an existing local Claude or OpenCode session tied to the current repository and Codex thread, and only creates a new delegated session with explicit approval.

4. **Inbox/outbox transport**
   Delegated requests are passed through repository-local workflow files:
   - `./.codex/inbox.md`
   - `./.codex/outbox.md`

   Codex writes the operative request into inbox, clears outbox, and instructs the delegated agent to read inbox and write back only the latest round result into outbox.

5. **Artifact and commit checkpoints**
   Workflows rely on durable local signals instead of chat-like streaming:
   - `PLAN.md`
   - `INVESTIGATION.md`
   - local commits
   - `CLAUDE_SESSION.json`
   - `OPENCODE_SESSION.json`
   - `./.codex/outbox.md`

   These signals are treated as completion markers for a round, not as evidence of ongoing work.

6. **Independent review loop**
   Codex does not trust delegated output by default. It re-reads the repository, checks the current branch state, evaluates artifacts, verifies claims, and sends review findings back as consultative follow-up rounds until the workflow is good enough or explicitly blocked.

## Workflow Model

These skills are not a generic task runner. They encode a specific orchestration pattern:

- **Orchestrator/reviewer -> delegated worker**
  Codex owns orchestration and review. Claude or OpenCode owns bounded execution. The separation is deliberate: one side acts, the other side challenges and validates.

- **Artifact-driven workflow**
  State is not kept only in memory or chat history. It lives in durable artifacts such as `PLAN.md`, `INVESTIGATION.md`, `CLAUDE_SESSION.json`, `OPENCODE_SESSION.json`, `./.codex/inbox.md`, and `./.codex/outbox.md`.

- **Mailbox-style file IPC**
  Request and response move through inbox/outbox files rather than a native agent API. That keeps the handoff explicit, auditable, and easy to resume.

- **Persistent-session orchestration**
  Delegated Claude sessions are reused through `session_uuid` and `claude -p --resume`, and OpenCode sessions are reused through their session metadata. The workflow is designed for continuity, not one-shot prompts.

- **Human-in-the-loop automation**
  New sessions require explicit approval. Pushes are gated. Quality gates are manual where they need to be, even when the delegated worker can automate the heavy lifting.

- **Cyclic state machine, not a simple DAG**
  The workflow is intentionally iterative. It loops through implement -> review -> revise, with retries and stop criteria such as “quality reached” or “disagreement is irreconcilable.” This is closer to a controlled state machine than a one-pass pipeline.

What this buys us:

- roles are separated correctly, so execution and skepticism do not collapse into one agent;
- workflow state is visible in files, not hidden in ephemeral chat;
- the system supports review loops instead of single-shot generation;
- guardrails are explicit: do not push automatically, do not create new sessions without approval, run tests and validation, and keep asking until the work is actually good enough;
- persistence through session files makes the process resumable rather than disposable.

## Provider-Agnostic Orchestration Lifecycle

The same control algorithm can be applied to Claude, OpenCode, or another delegated executor. The transport and session APIs may differ, but the orchestration state machine should stay consistent.

### Round Lifecycle (macro algorithm)

1. **Initialize run**
   Record `run_id`, workflow type, target output for this round (`PLAN.md`, `INVESTIGATION.md`, review output, or implementation commit), round success criteria, required artifacts, and forbidden side effects (for example `auto-push`).

2. **Bind executor session**
   Reuse session metadata when valid. If metadata is missing, discover reusable sessions. If discovery is ambiguous (none or multiple matches), stop for human decision unless explicit approval to create a new session is already given.

3. **Dispatch request**
   Serialize the request into provider transport (`inbox`, batch prompt, or command envelope), then record `dispatch_time`, `attempt_no`, and `request_fingerprint` (used to enforce exact-same-request retries).

4. **Cooldown window**
   Wait for a fixed window (for example 3 to 10 minutes) unless there is an immediate startup failure. This avoids noisy polling while a long executor step is still active.

5. **Monitor with cheap-first signals**
   Poll low-cost signals first (`git status`, artifact timestamps, process/session heartbeat). Escalate to expensive probes (`outbox`, provider stdout export, full session export) only when cheap signals are inconclusive. Persist every probe as a run-log event.

6. **Classify round state**
   Use explicit states:
   - `in_progress`
   - `completed_success`
   - `completed_blocked`
   - `failed_silent_exit`
   - `failed_likely_hang`
   - `failed_service_cap` (rate-limit/quota/provider cap)
   - `failed_protocol` (artifact contract violation)

7. **Apply state-specific reaction**
   - `completed_success`: continue to orchestrator-side independent verification.
   - `completed_blocked`: surface blocker and decide next action.
   - `failed_likely_hang`: terminate stale process/session, then resend the identical request.
   - `failed_silent_exit`: resend the identical request.
   - `failed_service_cap`: stop immediately and report; do not silently continue with scope drift.
   - `failed_protocol`: stop or request correction round with an explicit contract error report.

8. **Enforce retry policy**
   Track `consecutive_restart_counter`, retry only with the same `request_fingerprint`, reset counter after successful completion, and stop as unrecoverable after a defined cap (for example 5 restarts).

9. **Post-process successful round**
   Orchestrator independently validates artifacts, diff quality, test signals, and scope adherence. If quality is insufficient, create a feedback round. If quality passes threshold (for example >= 8/10), finish workflow.

10. **Finalize workflow stop reason**
   Use terminal classifications:
   - `done_quality_reached`
   - `done_irreconcilable_disagreement`
   - `stopped_service_cap`
   - `stopped_retry_exhausted`
   - `stopped_human_decision_required`
   - `stopped_external_blocker`
   - `stopped_protocol_violation`
   - `stopped_cancelled_by_user`

### State Machine (compact transition map)

| State | Event / Condition | Guard | Action | Next State |
| --- | --- | --- | --- | --- |
| `INIT` | run created | inputs valid | initialize run metadata and target artifact | `SESSION_BINDING` |
| `SESSION_BINDING` | metadata exists | session ref valid | bind session | `DISPATCH_PREP` |
| `SESSION_BINDING` | metadata missing | exactly one discovered session | bind discovered session | `DISPATCH_PREP` |
| `SESSION_BINDING` | metadata missing | zero or multiple sessions | request human decision | `WAIT_HUMAN` |
| `WAIT_HUMAN` | human approved create | `approved=true` | create session, persist metadata | `DISPATCH_PREP` |
| `WAIT_HUMAN` | human selected session | chosen session valid | bind selected session | `DISPATCH_PREP` |
| `WAIT_HUMAN` | human aborted | n/a | stop | `STOPPED_HUMAN_DECISION_REQUIRED` |
| `DISPATCH_PREP` | request ready | n/a | build payload, save fingerprint, increment attempt | `DISPATCHED` |
| `DISPATCHED` | batch call sent | n/a | store dispatch time | `COOLDOWN` |
| `COOLDOWN` | cooldown elapsed | no fatal startup error | start polling loop | `POLL_CHEAP` |
| `COOLDOWN` | provider startup failure | service-cap/quota/rate-limit | mark terminal reason | `STOPPED_SERVICE_CAP` |
| `POLL_CHEAP` | progress signals present | n/a | log progress event | `IN_PROGRESS` |
| `POLL_CHEAP` | signals inconclusive | n/a | escalate to expensive probe | `POLL_EXPENSIVE` |
| `POLL_CHEAP` | process gone and outputs missing | required outputs absent | classify failure | `FAILED_SILENT_EXIT` |
| `IN_PROGRESS` | required outputs present | artifact contract satisfied | collect outputs | `ROUND_SUCCESS` |
| `IN_PROGRESS` | prolonged silence | stale session/export too | classify failure | `FAILED_LIKELY_HANG` |
| `IN_PROGRESS` | explicit blocker | n/a | capture blocker | `ROUND_BLOCKED` |
| `IN_PROGRESS` | provider cap error | n/a | capture provider failure | `STOPPED_SERVICE_CAP` |
| `IN_PROGRESS` | output contract violated | malformed/missing required sections | capture protocol error | `FAILED_PROTOCOL` |
| `POLL_EXPENSIVE` | outputs complete | artifact contract satisfied | collect outputs | `ROUND_SUCCESS` |
| `POLL_EXPENSIVE` | explicit blocker | n/a | capture blocker | `ROUND_BLOCKED` |
| `POLL_EXPENSIVE` | stale and no outputs | n/a | classify failure | `FAILED_LIKELY_HANG` |
| `POLL_EXPENSIVE` | process ended, no outputs | n/a | classify failure | `FAILED_SILENT_EXIT` |
| `POLL_EXPENSIVE` | output contract violated | malformed/missing required sections | capture protocol error | `FAILED_PROTOCOL` |
| `FAILED_LIKELY_HANG` | retries left | restarts < limit | terminate stale session/process and resend identical request | `DISPATCH_PREP` |
| `FAILED_SILENT_EXIT` | retries left | restarts < limit | resend identical request | `DISPATCH_PREP` |
| `FAILED_LIKELY_HANG` | retries exhausted | restarts >= limit | mark terminal | `STOPPED_RETRY_EXHAUSTED` |
| `FAILED_SILENT_EXIT` | retries exhausted | restarts >= limit | mark terminal | `STOPPED_RETRY_EXHAUSTED` |
| `FAILED_PROTOCOL` | contract mismatch confirmed | not safely auto-recoverable | stop and report protocol failure | `STOPPED_PROTOCOL_VIOLATION` |
| `ROUND_BLOCKED` | can be unblocked by human | n/a | report blocker and wait | `WAIT_HUMAN` |
| `ROUND_BLOCKED` | cannot unblock now | n/a | stop with blocker reason | `STOPPED_EXTERNAL_BLOCKER` |
| `ROUND_SUCCESS` | verification started | n/a | run independent orchestrator verification | `ORCH_VERIFY` |
| `ORCH_VERIFY` | quality gate passed | score >= threshold | finalize | `DONE_QUALITY_REACHED` |
| `ORCH_VERIFY` | quality gate failed | retry loop allowed | send feedback request | `DISPATCH_PREP` |
| `ORCH_VERIFY` | irreconcilable disagreement | n/a | finalize | `DONE_IRRECONCILABLE_DISAGREEMENT` |

### Additional Requirements That Prevent Drift

- **Idempotent retries:** retries must be byte-equivalent or semantically equivalent to the original request; no silent scope expansion.
- **Output contract versioning:** each workflow should declare expected artifact schema/sections so protocol failures are machine-detectable.
- **Cancellation path:** user/operator cancel must be first-class and mapped to terminal `stopped_cancelled_by_user`.
- **Observability baseline:** each run should keep an append-only event log with timestamps, state transitions, retry causes, and terminal reason.
- **Orchestrator independence:** a delegated self-report is never treated as sufficient evidence without repository-level verification.

## Core Codex Workflow Skills

### Direct Codex workflows

These workflows are run directly by Codex and do not depend on a delegated Claude/OpenCode session.

| Skill | Purpose | Typical outputs |
| --- | --- | --- |
| `take-task` | Start work from a task id, inspect task context and related local history, ensure the correct task branch, draft `PLAN.md`, and commit the plan. | `PLAN.md`, task branch, local plan commit |
| `update-mr` | Create or update the GitLab MR for the current branch, preserve the task link in the MR description, push the branch when the workflow explicitly requires it, and update the task's MR backlink. | updated or new GitLab MR, updated task backend MR link |

### Delegated Claude workflows

These skills keep Codex in the orchestration role while Claude performs bounded autonomous work in a reusable local session.

| Skill | Purpose | Main artifact or result |
| --- | --- | --- |
| `claude-task-planning` | Produce or refine a task-scoped `PLAN.md` from a concrete task id. | reviewed `PLAN.md` |
| `claude-context-planning` | Produce or refine `PLAN.md` from current branch context and a freeform prompt instead of a task id. | reviewed `PLAN.md` |
| `claude-bug-investigation` | Investigate a task-scoped bug using logs, history, and code context. | reviewed `INVESTIGATION.md` |
| `claude-context-investigation` | Investigate a prompt-defined or branch-defined issue without a formal task id. | reviewed `INVESTIGATION.md` |
| `claude-context-review` | Review a branch, commit, MR, diff, or other local review target and iterate on findings. | structured review in outbox |
| `claude-task-implementation-review` | Let Claude implement from `PLAN.md` or an explicit implementation prompt, then have Codex review and steer fix rounds. | implementation commit(s), tests, reviewed outbox summary |

### Delegated OpenCode workflows

These mirror the Claude orchestration model, but use OpenCode-specific session discovery and transport rules.

| Skill | Purpose | Main artifact or result |
| --- | --- | --- |
| `opencode-task-planning` | Produce or refine a task-scoped `PLAN.md` from a concrete task id. | reviewed `PLAN.md` |
| `opencode-context-planning` | Produce or refine `PLAN.md` from branch context and a freeform prompt. | reviewed `PLAN.md` |
| `opencode-bug-investigation` | Investigate a task-scoped bug and build a defensible root-cause artifact. | reviewed `INVESTIGATION.md` |
| `opencode-context-investigation` | Investigate a branch-local or prompt-defined issue without a task id. | reviewed `INVESTIGATION.md` |
| `opencode-context-review` | Review a local target such as a branch, commit, or diff and iterate on findings. | structured review in outbox |
| `opencode-task-implementation-review` | Let OpenCode implement from `PLAN.md` or a direct implementation request while Codex handles review. | implementation commit(s), tests, reviewed outbox summary |

## Shared Workflow Guarantees

The orchestration skills in `codex/skills/.shared` impose common rules across providers:

- Delegated work stays batch-oriented.
- The delegated agent should not push automatically.
- Inbox and outbox are treated as disposable workflow transport files and must not be committed.
- Human-facing workflow artifacts default to English.
- Code comments added in project files should remain in English.
- Review cycles should be evidence-driven, not trust-driven.
- Codex should wait for stable artifacts and commits instead of over-polling.
- If a delegated session cannot complete the assigned work, the workflow should report that explicitly rather than silently collapsing into solo Codex work.

## Important Workflow Artifacts

### Planning and investigation artifacts

- `PLAN.md`
- `INVESTIGATION.md`

These files are the main durable outputs for planning and bug-analysis workflows. They are intentionally reviewable and commit-friendly so the next workflow round can reuse them.

### Session metadata

- `CLAUDE_SESSION.json`
- `OPENCODE_SESSION.json`

These files pin the delegated provider session that Codex should resume for future rounds. Session reuse is a first-class behavior in this repository; a workflow should continue an existing session when it can do so safely.

### Transport files

- `./.codex/inbox.md`
- `./.codex/outbox.md`

These files are the handoff channel between Codex and the delegated agent. The inbox carries the current round request. The outbox carries the delegated agent's latest answer, summary, verification report, or review result.

## Why the Codex Skills Matter Most

The `claude/` and `opencode/` directories are useful, but they are not the orchestration policy layer. The real workflow behavior lives in the Codex skills:

- they decide when a task branch is created or reused;
- they decide whether an existing delegated result should be reviewed immediately;
- they define when to stop, when to iterate, and when to escalate;
- they define the quality bar for plans, investigations, reviews, and implementations;
- they define how Codex validates delegated output instead of simply relaying it.

If you want to understand how this repository actually runs multi-agent work, start with `codex/skills/.shared`, then read `take-task`, `update-mr`, and the delegated workflow families.

## Suggested Reading Order

1. `codex/skills/.shared/delegated-agent/common/basics.md`
2. `codex/skills/take-task/SKILL.md`
3. `codex/skills/update-mr/SKILL.md`
4. One provider session file:
   - `codex/skills/.shared/delegated-agent/providers/claude/session.md`
   - `codex/skills/.shared/delegated-agent/providers/opencode/session.md`
5. One task-scoped delegated workflow:
   - `codex/skills/claude-task-planning/references/workflow.md`
   - `codex/skills/opencode-task-planning/references/workflow.md`
6. One review or implementation-review workflow for the quality loop semantics.

## Typical Use Cases

- Open a task from the task tracker, create the right branch, and generate a reviewed plan.
- Reuse a delegated session instead of starting over when planning or investigation already exists.
- Send bug analysis to Claude or OpenCode, then have Codex independently review the result before accepting it.
- Delegate implementation but keep code-review authority in Codex.
- Keep GitLab MR descriptions aligned with the current branch state and task metadata.

## Summary

This repository is not a collection of isolated prompts. It is a structured orchestration layer for long-running local agent workflows where Codex remains the control plane, artifacts are explicit, and delegated work is always subject to a real review loop.
