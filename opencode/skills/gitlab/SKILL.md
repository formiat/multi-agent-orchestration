---
name: gitlab
description: Use when the user mentions GitLab, MR, merge request, push, or asks to publish/share code changes. Provides workflow rules for working with GitLab via glab CLI.
---

# GitLab Workflow

Use `glab` for all GitLab interactions.

**Push and MR creation are NOT automatic. Only do these when explicitly requested by user.**

## Push

Before push, ensure current branch is not accidentally tracking `origin/develop`.

```bash
# optional safety check
upstream=$(git rev-parse --abbrev-ref --symbolic-full-name @{upstream} 2>/dev/null || true)
if [ "$upstream" = "origin/develop" ]; then
  git branch --unset-upstream
fi

git push -u origin <branch>
```

`-u` here is expected: it sets upstream to `origin/<branch>` (not to `origin/develop`).

If push requires rebase — cancel, do not rebase. Inform the user instead.

## Create MR

```bash
glab mr create --title "[TASK-{ID}] Description" --target-branch develop --squash-before-merge --remove-source-branch --yes --description "$(cat <<'EOF'
https://www.notion.so/task-{TASK_ID}

## What Was Done
...
EOF
)"
```

- Target branch: always `develop`
- Enable squash on merge + auto-delete source branch
- MR title format: `[TASK-{TASK_ID}] {Description}`

### After Creating the MR: Update the Notion Task

Put the MR link into the **Merge Request -> BACKEND** section of the task.

**IMPORTANT:** Use only in-place `update_block` updates. Do **not** use `pull | sed | push`, because that rewrites all blocks and breaks link annotations.

Safe sequence:

```bash
# 1. Pull and save to a file (creates a sidecar with block IDs)
notion task pull {TASK_ID} --save-to /tmp/t{TASK_ID}.md

# 2. Update only the BACKEND block in place via the API
/home/formi/.local/share/pipx/venvs/notion-cli/bin/python3 - <<'EOF'
import os, json
from pathlib import Path
from notion_client import Client

task_id = "{TASK_ID}"
mr_url = "{MR_URL}"

sidecar = json.loads(Path(f"/tmp/.t{task_id}.md.blocks.json").read_text())
blocks = sidecar["blocks"]

# Find BACKEND block strictly after the "Merge Request" heading
in_mr_section = False
block_id = None
for b in blocks:
    if b["type"] == "heading_2" and b["text"] == "Merge Request":
        in_mr_section = True
        continue
    if in_mr_section and b["type"] == "heading_2":
        break  # left the section
    if in_mr_section and b["text"].startswith("BACKEND:"):
        block_id = b["id"]
        break

if not block_id:
    raise RuntimeError("BACKEND block not found in Merge Request section")

client = Client(auth=os.environ["NOTION_API_KEY"], base_url=os.environ["NOTION_BASE_URL"])
client.blocks.update(
    block_id=block_id,
    paragraph={"rich_text": [
        {"type": "text", "text": {"content": "BACKEND: "}},
        {"type": "text", "text": {"content": mr_url, "link": {"url": mr_url}}}
    ]}
)
print(f"Updated BACKEND → {mr_url}")
EOF
```

### Task Link in the MR Description

**Always** start the MR description with the Notion task link.

Link source priority:
1. **Full link** from `notion task view {ID}` (`URL:` field) is preferred because Notion may change numeric IDs and a composed link can become invalid.
2. **Composed link** `https://www.notion.so/task-{TASK_ID}` should be used only if the full link is unknown.

Example: if `notion task view 7687` returns `URL: https://www.notion.so/8f21a6c94b0d47e2a1c38f5d6b7e9a10`, use that exact URL at the top of the MR description instead of `https://www.notion.so/task-7687`.

## Update MR Description

**ALWAYS** read current description first before writing anything:

```bash
glab mr view
```

Extract and preserve:
- Notion link (usually at the top)
- Any other manually written content

Then compose and push updated description. **Never overwrite blindly** — losing the Notion link is not acceptable.

```bash
glab mr update --description "$(cat <<'EOF'
https://www.notion.so/...   ← preserve existing Notion link (full URL preferred over task-{ID} form)

## What Was Done
...
EOF
)"
```

When passing descriptions via heredoc, always use `<<'EOF'` (single-quoted) — backticks inside must NOT be escaped.

## Useful Commands

```bash
glab mr list
glab mr view
glab mr view <id>
glab pipeline list
glab pipeline status
```
