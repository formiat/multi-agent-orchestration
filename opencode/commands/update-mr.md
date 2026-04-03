---
description: Create or update the GitLab MR for the current branch while preserving the Notion link and existing content.
agent: build
---

# Update MR

Create or update the GitLab Merge Request for the current branch.

## Step 1: Determine current branch and check for existing MR

```bash
git branch --show-current
glab mr list --source-branch <current-branch>
```

Also determine whether the branch has unpushed local commits. If it is ahead of its upstream, push the current branch before any MR create/update action, using the same safety rules as below. The MR must reflect the current local `HEAD`, not an older remote state.

## Step 2a: If MR exists — update description

**ALWAYS read the current description first before writing anything:**

```bash
glab mr view <id>
```

Extract and preserve from the existing description:
- Notion link (usually at the top — MUST be kept)
- Any other manually written content

Then compose the new description incorporating:
1. Preserved Notion link at the very top
2. Description of what was done (`## Что сделано`)
3. Any additional context from the current conversation
4. If the MR touches HTTP or web routes, a dedicated `## Затронутые роуты` section

Rules for `## Затронутые роуты`:
- Include the full list of touched routes when the list is reasonably small.
- If the MR touches many routes, or effectively all routes of a module, route patterns may be masked with `*` (for example `/api/camera-manager/*`).
- Prefer concrete routes over masked patterns whenever practical.
- Do not omit touched routes from the MR description just because they are "obvious from the diff".

Use `glab mr update <id> --description "..."` with a `<<'EOF'` heredoc (single-quoted to avoid backtick issues).

**Never overwrite blindly — losing the Notion link is not acceptable.**

## Step 2b: If no MR exists — create one

Before creating, get the full Notion link:

```bash
# Extract task ID from branch name (e.g. task-7812-... → 7812)
notion task view <TASK_ID>
# Use the URL: field from output (preferred over composed https://www.notion.so/task-{ID})
```

Create MR:

```bash
glab mr create \
  --title "[TASK-{ID}] {Description}" \
  --target-branch develop \
  --squash-before-merge \
  --remove-source-branch \
  --yes \
  --description "$(cat <<'EOF'
{NOTION_URL}

## Что сделано
...

## Затронутые роуты
...
EOF
)"
```

Rules:
- Target branch: always `develop`
- Enable squash on merge + auto-delete source branch
- MR title format: `[TASK-{TASK_ID}] {Description}`
- Description starts with full Notion URL (from `notion task view`, not composed form)
- If the MR touches HTTP or web routes, the description must contain a `## Затронутые роуты` section with the full list of touched routes.
- Masking routes with `*` is allowed only when the list is very large or the whole route surface of a module is affected.

### After creating MR — update Notion task

Put the MR link in the **Merge Request → BACKEND** block of the Notion task.

Use `update_block` in-place — NEVER use `pull | sed | push` (breaks link annotations):

```bash
# 1. Pull with sidecar (creates block ID map)
notion task pull {TASK_ID} --save-to /tmp/t{TASK_ID}.md

# 2. Update BACKEND block in-place
/home/formi/.local/share/pipx/venvs/notion-cli/bin/python3 - <<'EOF'
import os, json
from pathlib import Path
from notion_client import Client

task_id = "{TASK_ID}"
mr_url = "{MR_URL}"

sidecar = json.loads(Path(f"/tmp/.t{task_id}.md.blocks.json").read_text())
blocks = sidecar["blocks"]

in_mr_section = False
block_id = None
for b in blocks:
    if b["type"] == "heading_2" and b["text"] == "Merge Request":
        in_mr_section = True
        continue
    if in_mr_section and b["type"] == "heading_2":
        break
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

## Push safety check (if needed before MR creation or update)

```bash
upstream=$(git rev-parse --abbrev-ref --symbolic-full-name @{upstream} 2>/dev/null || true)
if [ "$upstream" = "origin/develop" ]; then
  git branch --unset-upstream
fi
git push -u origin <branch>
```

Use this push step in two cases:
- the branch is not yet on `origin`;
- the branch has unpushed local commits and is ahead of its upstream.

If push requires rebase — cancel, do not rebase. Inform the user instead.
