---
description: Read and analyze a Notion task and map acceptance criteria and affected code without changing files.
agent: build
---

# Study Task

Read and analyse Notion task TASK-$ARGUMENTS. No changes are made anywhere — no branches, no files, no commits.

## Step 1: Read the task

Run `notion task view $ARGUMENTS` to get the task description, requirements, and context. Parse the output.

If the task references other tasks (e.g., TASK-1234), fetch them too with `notion task view <id>`. For each referenced task, also check if it was already implemented: `git log --all --grep="TASK-<id>" --oneline`.

## Step 2: Analyse the codebase

Spawn an Explore agent that does the following:

1. **Identify acceptance criteria** — re-read the task output, extract what "done" means: required behaviors, endpoints, formats, role access, etc.
2. **Find related code** — search the codebase for modules, structs, functions, configs, and routes relevant to the task. Read the key files.
3. **Map affected components** — list every crate/module that would need changes and briefly explain why.
4. **Spot edge cases and open questions** — note anything ambiguous or missing in the task description.

## Step 3: Present findings

Output a structured summary:

- **Task**: title and short plain-language summary of what needs to be done
- **Acceptance criteria**: concrete list of what "done" looks like
- **Affected components**: crates/modules that are relevant, with file paths
- **Key findings**: important code discoveries (existing patterns to follow, constraints, gotchas)
- **Open questions**: anything that needs clarification before coding can start

Do not write PLAN.md. Do not create or modify any files. Do not create branches. Do not commit anything.
