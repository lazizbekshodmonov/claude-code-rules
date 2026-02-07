# Planning Rules

> Universal planning mode rules for Claude Code. Apply to all projects regardless of tech stack.

## Mandatory Planning Mode

- When given a task, **always start in planning mode** before writing any code. Think through the approach, affected files, and potential side effects first.
- Save every task plan as a Markdown file in the `planning-claude-tasks/` folder at the project root.
- File naming convention: `YYYY-MM-DD-short-task-description.md` (e.g., `2026-02-07-add-user-auth.md`).
- Do NOT start writing or modifying code until the plan is saved and reviewed.
- Update the plan's status as the task progresses.
- This folder must be committed to Git so the team can track all AI-assisted task history.

## Plan File Template

Each plan file must include:

```markdown
# Task: [Short description]

## Date
YYYY-MM-DD

## Objective
What needs to be done and why.

## Affected Files
- List of files that will be created / modified / deleted

## Implementation Plan
1. Step-by-step breakdown of the approach
2. ...
3. ...

## Potential Risks / Side Effects
- Any risks, breaking changes, or dependencies to consider

## Subtasks (if parallelized)
| # | Subtask | Agent | Status |
|---|---------|-------|--------|
| 1 | Description | Agent 1 | ⬜ Planned |
| 2 | Description | Agent 2 | ⬜ Planned |

## Status
- [ ] Planned
- [ ] In Progress
- [ ] Completed
```

## Plan Complexity Levels

Scale the plan detail based on task complexity:

| Complexity | Files Affected | Plan Detail |
|-----------|---------------|-------------|
| Small | 1–3 files | Brief objective + file list + 3–5 steps |
| Medium | 4–10 files | Full template with risks and subtasks |
| Large | 10+ files | Full template + parallelization strategy + dependency graph |

## Rules

- Every task gets a plan — no exceptions, even for "quick fixes".
- Plans must be written **before** any code changes.
- If the scope changes mid-task, update the plan first, then continue coding.
- Completed plans serve as documentation — never delete them.
- Review past plans in `planning-claude-tasks/` before starting similar tasks to avoid repeating mistakes.
