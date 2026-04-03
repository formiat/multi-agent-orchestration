# Investigate Bug

Start investigating bug for task TASK-$ARGUMENTS.

## Step 1: Understand the task

Run `notion task view $ARGUMENTS` to get the task description, symptom, requirements, and context. Parse the output and summarize what needs to be investigated.

If the task description references other tasks (e.g., TASK-1234), fetch their details too using `notion task view <id>`. For related bugfix tasks, also inspect their commits with `git log --all --grep="TASK-<related-task-id>"` to understand what was changed and whether it could be relevant.

If `INVESTIGATION.md` already exists in the repo root, read it first and treat it as the current state of the investigation to refine or correct rather than ignore.

Important safety scope for this command:
- Investigate only through local repository files and explicitly user-provided local materials such as logs, traces, dumps, or attachments.
- Do not access remote servers, remote complexes, or external environments on your own initiative.
- Do not use `ssh`, `scp`, `sftp`, or remote `curl`/API calls as part of the investigation unless the user explicitly asks for that.

## Step 2: Check current branch

Derive a short kebab-case description (2-4 words) from the task title. The expected branch name is `task-$ARGUMENTS-<short-description>`.

Check the current branch with `git branch --show-current`. Extract the task id from the current branch name if it matches a `task-<id>-...` pattern.

- If the current branch already belongs to TASK-$ARGUMENTS, continue on the current branch as-is, even if its suffix differs from the newly derived short description.
- If the current branch task id is missing or does not match TASK-$ARGUMENTS, create the expected branch automatically without asking the user:
  1. Run `git fetch origin develop`.
  2. Create the branch without tracking: `git checkout --no-track -b task-$ARGUMENTS-<short-description> origin/develop`.
  3. Verify upstream is not set. If upstream exists, immediately clear it: `git branch --unset-upstream`.

Important: a newly created task branch must **never** be tied to `origin/develop` as upstream.

## Step 3: Investigate

Spawn an agent that does the following:

1. **Analyse the symptom** — re-read the notion task output, identify the observable bug, expected behavior, scope, and possible subsystems involved.
2. **Find relevant code and data** — search the codebase for modules, structs, functions, configs, traces, and tests relevant to the bug. Read the key files.
3. **Inspect evidence** — study available logs, traces, dumps, local materials, and reproduction assets. Build a concrete timeline. Distinguish confirmed facts from hypotheses.
4. **Check history** — inspect recent suspect commits and older blame/history in relevant files to identify who and when likely introduced the regression.
5. **Draft the investigation** — create or update `INVESTIGATION.md` in the repo root:
   - Write the document in English
   - **Task**: TASK-$ARGUMENTS title and short summary
   - **Symptom**: what is wrong and where it is observed
   - **Expected behavior**: what should happen instead
   - **Evidence**: concrete logs, traces, files, code references, and facts
   - **Timeline**: ordered sequence of the relevant events
   - **Ruled out hypotheses**: explanations that were checked and rejected
   - **Main hypotheses**: remaining plausible explanations, ranked by confidence
   - **Suspect commits**: relevant historical changes and why they matter
   - **Most likely root cause**: current best-supported conclusion
   - **Testing implications**: relevant existing tests, current coverage gaps, and what tests a future fix should add or adapt; future tests should be self-contained and portable rather than dependent on machine-specific local files, directories, or environments
   - **Fix directions**: possible repair strategies and tradeoffs
   - **Open questions**: what is still unknown
6. **Commit the investigation** — commit `INVESTIGATION.md` as a separate local commit with message: `[TASK-$ARGUMENTS] INVESTIGATION.md`. Never push commits automatically or without an explicit user command.

After the investigation is committed, show the user a concise summary of the root cause hypothesis, strongest evidence, and open questions.
