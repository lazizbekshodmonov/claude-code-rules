# Code Review Rules

> Backend code review rules for Claude Code. Apply to all Node.js backend projects.

## Reviewer Responsibilities

- Write **clear, detailed comments** explaining the issue and suggesting a solution.
- The reviewer must resolve their own comments — not the author.
- Do NOT approve a PR until the developer has addressed every comment.
- Be respectful and constructive — critique the code, not the person.

## Comment Types

Use prefixes to classify comments:

| Prefix | Meaning | Action Required |
|--------|---------|-----------------|
| `blocker:` | Critical issue, must be fixed | PR cannot be merged |
| `suggestion:` | Improvement recommendation | Author should consider |
| `nit:` | Minor style/preference issue | Optional to fix |
| `question:` | Need clarification | Author must respond |
| `idea:` | Future improvement suggestion | No action needed now |

### Example Comments

```
blocker: This endpoint has no authentication guard. Any unauthenticated
user can access admin data. Add @UseGuards(AuthGuard, RolesGuard).

suggestion: Consider using a database transaction here — if the order
creation succeeds but inventory update fails, we'll have inconsistent data.

nit: Missing return type annotation on this method.

question: Why are we using a raw query instead of the ORM here?
Is there a performance reason?

idea: We could add a caching layer here when traffic grows.
```

## Review Checklist

Before approving any PR, verify:

### Functionality
- [ ] Code does what the PR description says
- [ ] Edge cases are handled (null, empty, invalid input, not found)
- [ ] No regressions in existing functionality

### Code Quality
- [ ] Follows project coding standards (CLAUDE.md rules)
- [ ] No code duplication — reuses existing services/utilities
- [ ] Functions are small and single-purpose
- [ ] No `any` types (TypeScript projects)
- [ ] No `console.log` or debug code left behind

### API Design
- [ ] Correct HTTP methods and status codes
- [ ] Consistent response format
- [ ] Input validation on all endpoints
- [ ] Proper pagination for list endpoints

### Security
- [ ] Authentication and authorization on protected endpoints
- [ ] No hardcoded secrets or credentials
- [ ] Input is validated and sanitized
- [ ] No SQL injection vulnerabilities
- [ ] Rate limiting on sensitive endpoints

### Database
- [ ] Queries are optimized (no N+1, proper indexes)
- [ ] Transactions used for multi-table operations
- [ ] Migrations are reversible and tested

### Testing
- [ ] New/modified code has corresponding tests
- [ ] Tests cover happy path and error cases
- [ ] All tests pass
- [ ] No test data hardcoded with production values

### Documentation
- [ ] JSDoc on all new service methods
- [ ] Swagger decorators on all new endpoints
- [ ] Complex logic has inline comments explaining "why"

### Error Handling
- [ ] Custom error classes used (not generic Error)
- [ ] Errors logged with sufficient context
- [ ] No internal details leaked to client

## Author Responsibilities

- Keep PRs small and focused — under 400 lines changed.
- Write a clear PR description with context, changes, and testing steps.
- Respond to all review comments promptly.
- Do NOT resolve reviewer comments yourself — let the reviewer verify and resolve.
- If you disagree with a comment, explain your reasoning — don't silently ignore it.

## Rules

- Every PR needs at least **one approval** before merging.
- Self-merging without review is not allowed (except emergency hotfixes).
- Stale PRs (open for more than 5 days) should be addressed or closed.
- Review PRs within **24 hours** of being requested.
- Use "Request changes" for blockers, "Comment" for discussions, "Approve" only when everything is good.
