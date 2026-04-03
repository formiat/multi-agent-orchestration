---
description: Read and analyze a GitLab MR, its comments, and affected code without changing anything.
agent: build
---

# Study MR

Read and analyse GitLab Merge Request $ARGUMENTS. No changes are made anywhere — no commits, no files, no comments.

If $ARGUMENTS is empty, use the MR for the current branch: `glab mr view`.

## Step 1: Read the MR

```bash
glab mr view $ARGUMENTS
glab mr diff $ARGUMENTS
glab mr note list $ARGUMENTS
```

Parse: title, description, author, target branch, changed files, CI status.

If the description contains a Notion task URL or TASK-ID, fetch it:
```bash
notion task view <id>
```

Read all review comments and discussion threads from `glab mr note list`.
Note which comments are resolved vs open, who left them, and what they concern.

## Step 2: Analyse changed files

For each changed file in the diff:
- Read the full file from the codebase (not just the diff) to understand context.
- Note what the change does within the broader module.

Spawn an Explore agent if the change spans many files or requires deep codebase traversal.

## Step 3: Present findings

Output a structured summary:

- **MR**: title, author, target branch, CI status
- **Linked task**: TASK-ID and its acceptance criteria (if found)
- **What changes**: plain-language description of what the MR does, file by file
- **Affected components**: crates/modules touched and why
- **Key observations**: patterns used, architectural choices, notable decisions
- **Open questions**: anything unclear or potentially problematic
- **Review comments**: open threads, key reviewer concerns, resolved vs unresolved

Do not post any comments. Do not modify any files. Do not create branches. Do not run any writes.
