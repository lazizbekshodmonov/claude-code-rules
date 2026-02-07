# Git Rules

> Universal Git workflow rules for Claude Code. Apply to all projects regardless of tech stack.

## Branch Naming

Use descriptive, prefixed branch names:

```
feature/add-user-auth
feature/dashboard-charts
bugfix/fix-login-redirect
bugfix/null-pointer-on-submit
hotfix/critical-payment-error
refactor/extract-api-service
chore/update-dependencies
docs/add-api-documentation
```

| Prefix | Usage |
|--------|-------|
| `feature/` | New functionality |
| `bugfix/` | Non-critical bug fixes |
| `hotfix/` | Critical production fixes |
| `refactor/` | Code restructuring without behavior changes |
| `chore/` | Dependencies, configs, tooling |
| `docs/` | Documentation only |

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/) format:

```
type(scope): short description

[optional body]
[optional footer]
```

### Types

| Type | Usage |
|------|-------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `docs` | Documentation only |
| `style` | Formatting, missing semicolons (no logic change) |
| `test` | Adding or updating tests |
| `chore` | Build process, dependencies, tooling |
| `perf` | Performance improvements |
| `ci` | CI/CD configuration |
| `revert` | Reverting a previous commit |

### Examples

```bash
# ✅ Good commit messages
feat(auth): add Google OAuth login flow
fix(dashboard): resolve chart rendering on mobile
refactor(api): extract HTTP client into service layer
test(user): add unit tests for registration validation
docs(readme): update API endpoint documentation
chore(deps): upgrade Vue to 3.4.0

# ❌ Bad commit messages
fixed stuff
update
WIP
asdf
changes
```

### Rules

- Keep the subject line under **72 characters**.
- Use imperative mood: "add feature" not "added feature" or "adding feature."
- Do not end the subject line with a period.
- Separate subject from body with a blank line.
- Use the body to explain **what** and **why**, not how.

## Commit Scope

- Make **atomic commits** — each commit should represent one logical change.
- Do NOT combine unrelated changes in a single commit.

```bash
# ❌ Incorrect — mixed concerns
git commit -m "feat: add login page and fix header bug and update deps"

# ✅ Correct — atomic commits
git commit -m "feat(auth): add login page component"
git commit -m "fix(header): resolve responsive layout issue"
git commit -m "chore(deps): update Tailwind to v3.4"
```

## Pull Requests

- PR title must follow the same conventional commit format.
- PR description must include:
  - What was changed and why
  - How to test it
  - Screenshots (if UI changes)
  - Link to related issue/task
- Keep PRs small and focused — ideally under 400 lines changed.
- Never merge your own PR without at least one review approval.

## Protected Files

Never commit these files to Git:

```
.env
.env.*
*.pem
*.key
secrets/
credentials.json
```

Always verify `.gitignore` includes sensitive files before committing.

## Rules

- Always pull latest changes before starting work: `git pull origin main`
- Never force push to shared branches (`main`, `develop`, `staging`).
- Delete merged branches to keep the repository clean.
- Use `git rebase` for feature branches to maintain linear history (team preference).
- Tag releases with semantic versioning: `v1.0.0`, `v1.1.0`, `v2.0.0`.
