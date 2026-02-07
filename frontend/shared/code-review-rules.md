# Code Review Rules

> Universal code review standards for Claude Code. Apply to all projects regardless of tech stack.

## Reviewer Responsibilities

- Write **clear, detailed comments** explaining the issue and suggesting a solution.
- The reviewer must resolve their own comments — not the author.
- Do NOT approve a PR until the developer has reviewed or explained every comment.
- Be respectful and constructive — critique the code, not the person.

## Comment Types

Use prefixes to classify comments:

| Prefix | Meaning | Action Required |
|--------|---------|-----------------|
| `🔴 blocker:` | Critical issue, must be fixed | PR cannot be merged |
| `🟡 suggestion:` | Improvement recommendation | Author should consider |
| `🟢 nit:` | Minor style/preference issue | Optional to fix |
| `❓ question:` | Need clarification | Author must respond |
| `💡 idea:` | Future improvement suggestion | No action needed now |

### Example Comments

```
🔴 blocker: This API call has no error handling. If the request fails,
the entire page will crash. Please wrap in try-catch and show an error
message to the user.

🟡 suggestion: Consider extracting this logic into a composable —
it's used in 3 different components already.

🟢 nit: Extra blank line here.

❓ question: Why are we using setTimeout instead of nextTick here?
Is there a specific timing issue?

💡 idea: We could add pagination here in the future when the list
grows beyond 100 items.
```

## Review Checklist

Before approving any PR, verify:

### Functionality
- [ ] Code does what the PR description says
- [ ] Edge cases are handled (null, empty, error states)
- [ ] No regressions in existing functionality

### Code Quality
- [ ] Follows project coding standards (CLAUDE.md rules)
- [ ] No code duplication — reuses existing utilities/components
- [ ] Functions and components are small and single-purpose
- [ ] No `any` types (TypeScript projects)
- [ ] No `console.log` or debug code left behind

### Testing
- [ ] New/modified code has corresponding tests
- [ ] Tests cover happy path and edge cases
- [ ] All tests pass

### Documentation
- [ ] JSDoc comments on all new functions
- [ ] Block comments on all new components
- [ ] Complex logic has inline comments explaining "why"

### Security
- [ ] No hardcoded secrets or credentials
- [ ] User input is validated and sanitized
- [ ] No XSS vulnerabilities (`v-html`, `dangerouslySetInnerHTML`)

### Performance
- [ ] No unnecessary re-renders or computations
- [ ] Large lists use virtualization or pagination
- [ ] Images are optimized

## Author Responsibilities

- Keep PRs small and focused — under 400 lines changed.
- Write a clear PR description with context, changes, and testing steps.
- Respond to all review comments promptly.
- Do NOT resolve reviewer comments yourself — let the reviewer verify and resolve.
- If you disagree with a comment, explain your reasoning — don't silently ignore it.

## Rules

- Every PR needs at least **one approval** before merging.
- Self-merging without review is not allowed (except for emergency hotfixes, which must be reviewed post-merge).
- Stale PRs (open for more than 5 days without activity) should be addressed or closed.
- Review PRs within **24 hours** of being requested.
- Use "Request changes" for blockers, "Comment" for discussions, "Approve" only when everything is good.
