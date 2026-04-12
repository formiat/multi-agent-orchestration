# Workflow

Start working on task `TASK-<task_id>`.

## Step 1: Understand the task

1. Run:
   - `notion task view <task_id>`
2. Parse the output and extract:
   - task title;
   - acceptance criteria and scope;
   - mentioned modules, routes, configs, tests, or manual verification hints;
   - references to other tasks.
3. If related tasks are mentioned:
   - run `notion task view <related_id>` for each;
   - run `git log --all --grep="TASK-<related_id>"` to understand what was already implemented locally.
4. If `INVESTIGATION.md` exists in the repo root, read it before planning. If new evidence contradicts it, call that out explicitly in `PLAN.md` instead of silently ignoring it.

## Step 2: Ensure the correct task branch

1. Derive a short kebab-case description from the task title, normally 2-4 words.
2. Expected branch name:
   - `task-<task_id>-<short-description>`
3. Check current branch:
   - `git branch --show-current`
4. If the current branch already matches `task-<task_id>-...`, keep using it even if the suffix differs.
5. If the current branch does not belong to this task:
   - run `git fetch origin develop`
   - create the branch without tracking:
     - `git checkout --no-track -b task-<task_id>-<short-description> origin/develop`
   - verify no upstream is set; if needed, clear it with:
     - `git branch --unset-upstream`

Important:
- a newly created task branch must never track `origin/develop`.

## Step 3: Investigate the code

1. Re-read the task details and identify:
   - acceptance criteria;
   - affected subsystems;
   - likely edge cases;
   - required tests and manual checks.
2. Search the codebase for relevant modules, structs, functions, settings, migrations, configs, routes, and tests.
3. Read the key files directly rather than relying on guesses.
4. If the task involves detection, violations, or video processing, inspect relevant local data or tooling only for investigation/manual verification context. Do not plan machine-specific automated tests.

## Step 4: Draft the plan

Create or update `PLAN.md` in the repo root. Write it in English and include:

1. `Task`
   - `TASK-<task_id>` title and a short summary.
2. `Context`
   - only the task facts that materially affect implementation;
   - if `INVESTIGATION.md` exists, briefly list the findings that constrain the plan.
3. `Affected components`
   - crates/modules/files that likely need changes.
4. `Implementation plan`
   - numbered, concrete steps;
   - include file paths when known;
   - describe important technical decisions and alternatives when a choice is needed.
5. `Testing`
   - propose as many practical automated tests as possible for the affected and new functionality;
   - group all proposed automated tests into three categories: 1) no refactoring required, 2) light refactoring required, 3) heavy refactoring required;
   - list all three categories explicitly in the plan, and mark category 1 separately as planned for implementation together with the main functional changes;
   - manual verification only as a supplement;
   - keep test design portable and self-contained.
6. `Open questions`
   - anything that still needs clarification before implementation.

## Step 5: Commit the plan

1. Commit `PLAN.md` as a separate local commit:
   - `[TASK-<task_id>] PLAN.md`
2. Do not push automatically.

## Step 6: Report back

Summarize for the user:
- task title and short summary;
- chosen/current branch;
- main affected components;
- open questions;
- commit hash.
