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
