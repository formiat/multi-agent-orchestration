# Investigate From Context

Start investigating a bug from the provided context and prompt.

The full context is in `$ARGUMENTS`. Treat it as the authoritative request. It may include:
- the symptom and expected behavior;
- local file paths to logs, traces, dumps, or materials;
- suspect commits, branches, modules, or hypotheses;
- explicit constraints about what to inspect or avoid.

Summarize what needs to be investigated before proceeding.

If `INVESTIGATION.md` already exists in the repo root, read it first and treat it as the current state of the investigation to refine or correct rather than ignore.

If the provided context references tasks, inspect their local commit history with `git log --all --grep="TASK-<id>"` when relevant. Do not assume a task id is present.

Important safety scope for this command:
- Investigate only through local repository files and explicitly provided local materials.
- Do not access remote servers, remote complexes, or external environments on your own initiative.
- Do not use `ssh`, `scp`, `sftp`, or remote `curl`/API calls as part of the investigation unless the user explicitly asks for that.

## Step 2: Check current branch

Check the current branch with `git branch --show-current`.

- Continue on the current branch as-is.
- Do not create or switch branches automatically for this command unless the provided context explicitly asks for that.

If the current branch matches `task-<id>-...`, you may reuse that task id in `INVESTIGATION.md` and in the commit message. Otherwise treat this as a context-only investigation.

## Step 3: Investigate

Spawn an agent that does the following:

1. **Analyse the context** — restate the bug, expected behavior, scope, and the explicit questions from the prompt.
2. **Find relevant code and data** — search the codebase for modules, structs, functions, configs, traces, tests, and local materials relevant to the investigation. Read the key files.
3. **Inspect evidence** — study logs, traces, dumps, local materials, and reproduction assets mentioned in the prompt. Build a concrete timeline. Distinguish confirmed facts from hypotheses.
4. **Check history** — inspect suspect commits and older blame/history in relevant files to identify who and when likely introduced the regression.
5. **Draft the investigation** — create or update `INVESTIGATION.md` in the repo root:
   - Write the document in English
   - **Context**: the prompt/request that triggered this investigation
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
6. **Commit the investigation** — commit `INVESTIGATION.md` as a separate local commit. Never push commits automatically or without an explicit user command.
   Use this message:
   - if the current branch matches `task-<id>-...`: `[TASK-<id>] INVESTIGATION.md`
   - otherwise: `[context] INVESTIGATION.md`

After the investigation is committed, show the user a concise summary of the root cause hypothesis, strongest evidence, and open questions.
