# Agent Rules

> Universal multi-agent execution and context limit prevention rules for Claude Code. Apply to all projects regardless of tech stack.

## Parallel Multi-Agent Execution

- When a task is **large or complex** (involves multiple modules, 3+ files, or independent subtasks), Claude Code **must** break it into smaller subtasks and run multiple sub-agents in parallel using the `Task` tool.
- Do NOT execute large tasks sequentially when subtasks are independent of each other.

### When to Parallelize

- Creating multiple components/modules that don't depend on each other
- Writing tests for several files simultaneously
- Refactoring across multiple modules
- Generating types, services, and components for a new feature at once
- Adding documentation/comments to multiple files

### When NOT to Parallelize

- Tasks with strict sequential dependencies (B depends on A's output)
- Single file changes
- Database migrations that must run in order
- Tasks requiring shared state between steps

### How It Should Work

```
Large Task: "Build a user profile page with avatar, stats, and settings"

Sequential ❌ (slow):
  1. Build Avatar → wait
  2. Build Stats → wait
  3. Build Settings → wait
  4. Write tests → wait

Parallel ✅ (fast):
  Agent 1 → Avatar component + tests
  Agent 2 → Stats component + tests
  Agent 3 → Settings component + tests
  Main    → Assembles page after agents finish
```

### Post-Parallel Checklist

After all parallel agents finish, the main agent must:
1. Verify all subtasks are complete
2. Run the full test suite to check for conflicts
3. Ensure modules integrate correctly together
4. Update the planning file with final status

### Document Subtasks

Document the parallelization strategy in the task's planning file:

```markdown
## Subtasks
| # | Subtask | Agent | Status |
|---|---------|-------|--------|
| 1 | Avatar component + tests | Agent 1 | ✅ Done |
| 2 | Stats component + tests | Agent 2 | ✅ Done |
| 3 | Settings component + tests | Agent 3 | ✅ Done |
| 4 | Integration + final test run | Main | ✅ Done |
```

## Context Limit Prevention

> **CRITICAL:** Each sub-agent has a limited context window. To prevent `Context limit reached` errors, follow these strict limits.

### File Limits

- Each sub-agent must handle **no more than 5–10 files** per task.
- If more files need changes, split into additional sequential batches.
- Do NOT read entire large files when only a small section needs editing — use targeted reads (specific line ranges).

### Scope Estimation

Before starting parallel agents, **estimate the scope** of each subtask:
- Less than 5 files, each under 200 lines → ✅ Safe for one agent
- 5–10 files or files up to 500 lines → ⚠️ Borderline, monitor closely
- More than 10 files or files over 500 lines → ❌ Must split further

### Recovery Strategy

If an agent hits the context limit mid-task:
1. Use `/compact` to compress the context and continue
2. If still insufficient, use `/clear` and restart only the remaining unfinished work
3. Document which files were completed and which are still pending in the planning file

### Scoping Example

```
Large Task: "Add JSDoc comments to all 40 components"

❌ Bad — 2 agents, 20 files each (will hit context limit):
  Agent 1 → 20 components
  Agent 2 → 20 components

✅ Good — 8 agents, 5 files each (stays within limits):
  Agent 1 → Components A–E (5 files)
  Agent 2 → Components F–J (5 files)
  Agent 3 → Components K–O (5 files)
  ...and so on
```

### Rules

- Prefer **more agents with fewer files** over fewer agents with many files.
- After each batch completes, verify results before starting the next batch.
- If a task requires reading shared context (e.g., types file used by many components), include it in each agent's scope but count it toward the file limit.
- Each sub-agent must follow **all project rules** (planning, testing, code style).
