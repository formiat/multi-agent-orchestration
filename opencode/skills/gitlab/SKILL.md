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

## Что сделано
...
EOF
)"
```

- Target branch: always `develop`
- Enable squash on merge + auto-delete source branch
- MR title format: `[TASK-{TASK_ID}] {Description}`

### После создания MR: обновить Notion-таску

Положить ссылку на MR в раздел **Merge Request → BACKEND** таски.

**ВАЖНО:** Использовать только `update_block` in-place — НЕ использовать `pull | sed | push`, т.к. это делает полную замену всех блоков и ломает link-аннотации (ссылки становятся некликабельными).

Безопасный пайплайн:

```bash
# 1. Pull с сохранением в файл (создаёт sidecar с block ID)
notion task pull {TASK_ID} --save-to /tmp/t{TASK_ID}.md

# 2. Обновить только BACKEND-блок in-place через API
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

### Ссылка на задачу в описании MR

**Всегда** начинай описание MR со ссылки на задачу Notion.

Приоритет источников ссылки:
1. **Полноценная ссылка** из вывода `notion task view {ID}` (поле `URL:`) — предпочтительна, т.к. Notion иногда меняет числовые ID и композированная ссылка может инвалидироваться.
2. **Композированная ссылка** `https://www.notion.so/task-{TASK_ID}` — использовать только если полная ссылка неизвестна.

Пример: если `notion task view 7687` вернул `URL: https://www.notion.so/8f21a6c94b0d47e2a1c38f5d6b7e9a10`, то в начало описания MR кладём именно эту ссылку, а не `https://www.notion.so/task-7687`.

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

## Что сделано
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
