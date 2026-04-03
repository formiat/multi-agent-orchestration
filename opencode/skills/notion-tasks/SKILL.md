---
name: notion-tasks
description: Use when the user mentions Notion tasks, task IDs (like 7595 or TASK-7595), or asks to read/edit/create/comment on tasks. Provides knowledge of the `notion` CLI tool for task management.
---

# Notion CLI Task Management

You have access to the `notion` CLI tool for managing tasks in a Notion database.

## Reading Tasks

```bash
# List tasks (filterable)
notion task list --limit 10
notion task list --status "В Процессе"
notion task list --assignee "Name"

# View a task (rich formatted)
notion task view <id>

# Pull task as raw markdown (for editing)
notion task pull <id>
notion task pull <id> --save-to /tmp/task.md
```

Task IDs can be short numbers (7595) or full IDs (TASK-7595).

## Editing Tasks

### Quick property update

```bash
notion task update <id> --status "В Процессе"
notion task update <id> --assign "Name" --priority "High"
notion task update <id> --title "New title"
```

### Full content editing via pull/push

1. Pull to file: `notion task pull <id> --save-to /tmp/task.md`
2. Edit the file (frontmatter properties + body content)
3. Preview: `notion task push <id> --from /tmp/task.md --dry-run`
4. Apply: `notion task push <id> --from /tmp/task.md`

### Push from stdin (full replace, no diffing)

```bash
echo '---
title: Task title
status: Создана
assignee: Name
---

Body content here.' | notion task push <id> --from -
```

### Markdown format

```markdown
---
title: Fix login bug
status: In Progress
assignee: Alice
priority: High
due_date: 2026-03-01
---

Body content as markdown (headings, paragraphs, lists, code blocks, to-do items).

---

<!-- comments (read-only) -->
Comments below this marker are ignored on push.
```

## Creating Tasks

```bash
# From frontmatter + body
echo '---
title: New task
status: Создана
assignee: Name
priority: Medium
---

Task description.
- [ ] Step one
- [ ] Step two' | notion task create --from -

# Simple creation
notion task create --title "Task name" --status "Создана"

# From file
notion task create --from task.md

# CLI flags override frontmatter
notion task create --title "Override" --assign "Name" --from task.md
```

## Comments

```bash
notion task comment <id> "Comment text"
```

## Workflow Tips

- When asked to read a task, use `notion task pull <id>` for raw content or `notion task view <id>` for formatted display.
- When asked to edit a task, pull it first, make changes, then push. Always use `--dry-run` before applying unless the user says otherwise.
- When asked to create a task, use `notion task create --from -` with stdin to set all properties and body at once.
- Assignee names are matched by substring (case-insensitive). Use the name as it appears in `notion task list`.
- Common statuses: Создана, К Работе, В Процессе, На Проверке, Выполнена.
