# Claude Code Project Rules

A collection of battle-tested `CLAUDE.md` rule sets for different project types. These rules ensure Claude Code follows your team's coding standards, architecture patterns, and best practices consistently across all team members.

## Table of Contents

- [What is this?](#what-is-this)
- [Available Rule Sets](#available-rule-sets)
- [Shared (Universal)](#shared-universal)
- [Frontend Shared Rules](#frontend-shared-rules)
- [Backend Shared Rules (Node.js)](#backend-shared-rules-nodejs)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [How It Works](#how-it-works)
- [Repository Structure](#repository-structure)
- [Customization](#customization)
- [Team Setup Guide](#team-setup-guide)
- [FAQ](#faq)
- [Contributing](#contributing)

---

## What is this?

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) is Anthropic's CLI tool for AI-powered coding. It reads a special `CLAUDE.md` file from your project root and follows the rules defined in it — coding standards, architecture patterns, formatting rules, testing requirements, and more.

This repository provides **ready-to-use rule sets** for different tech stacks so you don't have to write them from scratch. Just copy the one that matches your project, customize it, and commit it to your repo.

### Benefits

- **Team consistency** — Every developer gets the same AI behavior
- **Zero onboarding** — New team members just `git clone` and start
- **Battle-tested** — Rules refined through real project usage
- **Customizable** — Use as a starting point and adapt to your needs

---

## Available Rule Sets

### Frontend

| Rule Set | Tech Stack | File | Status |
|----------|-----------|------|--------|
| Vue + TypeScript | Vue 3, TypeScript, Tailwind CSS | [`frontend/rules/vue-ts/CLAUDE.md`](frontend/rules/vue-ts/CLAUDE.md) | ✅ Ready |

### Backend

| Rule Set | Tech Stack | File | Status |
|----------|-----------|------|--------|
| NestJS | NestJS, TypeScript, TypeORM | [`backend/node-js/rules/nest-js/CLAUDE.md`](backend/node-js/rules/nest-js/CLAUDE.md) | ✅ Ready |
| Express | Express, TypeScript, TypeORM | [`backend/node-js/rules/express-js/CLAUDE.md`](backend/node-js/rules/express-js/CLAUDE.md) | 🚧 Coming soon |
| Java + Spring Boot | Java, Spring Boot | [`backend/java/rules/CLAUDE.md`](backend/java/rules/CLAUDE.md) | 🚧 Coming soon |
| Python + FastAPI | Python, FastAPI | [`backend/python/rules/CLAUDE.md`](backend/python/rules/CLAUDE.md) | 🚧 Coming soon |

### Mobile

| Rule Set | Tech Stack | File | Status |
|----------|-----------|------|--------|
| Flutter | Dart, Flutter, BLoC | [`mobile/rules/flutter/CLAUDE.md`](mobile/rules/flutter/CLAUDE.md) | 🚧 Coming soon |

> Don't see your stack? [Contribute](#contributing) a new rule set or open an issue!

---

## Shared (Universal)

The `shared/` directory contains rules that apply to **all stacks** — frontend, backend, and mobile.

> **Important:** Claude Code only reads `CLAUDE.md` files, not the `shared/` directory directly. Embed these rules into your project's `CLAUDE.md`.

| File | Description |
|------|-------------|
| [`shared/agent-rules.md`](shared/agent-rules.md) | Multi-agent execution, context limits, scoping strategies |
| [`shared/git-rules.md`](shared/git-rules.md) | Branch naming, commit messages (conventional commits), PR practices |
| [`shared/planning-rules.md`](shared/planning-rules.md) | Mandatory planning mode, plan file template, complexity levels |

---

## Frontend Shared Rules

The `frontend/shared/` directory contains **reusable frontend building blocks** — rules that are embedded into frontend `CLAUDE.md` files. These cover naming, testing, security, performance, and other frontend-specific conventions.

> **Important:** Claude Code only reads `CLAUDE.md` files, not the `shared/` directory directly. These rules are already included in the frontend rule sets (e.g., `vue-ts/CLAUDE.md`).

| File | Description |
|------|-------------|
| [`frontend/shared/naming-rules.md`](frontend/shared/naming-rules.md) | Casing conventions, boolean/function/component/event naming |
| [`frontend/shared/typescript-rules.md`](frontend/shared/typescript-rules.md) | Type file organization, declaration order, enums, avoid `any` |
| [`frontend/shared/component-rules.md`](frontend/shared/component-rules.md) | Single responsibility, page thin wrappers, UI library usage, naming |
| [`frontend/shared/api-rules.md`](frontend/shared/api-rules.md) | API service layer, HTTP client, error handling, request/response typing |
| [`frontend/shared/testing-rules.md`](frontend/shared/testing-rules.md) | Test types (unit/integration), naming conventions, 80% coverage minimum |
| [`frontend/shared/documentation-rules.md`](frontend/shared/documentation-rules.md) | JSDoc, block comments, inline comments, file-level docs |
| [`frontend/shared/code-review-rules.md`](frontend/shared/code-review-rules.md) | Reviewer/author responsibilities, comment types, review checklist |
| [`frontend/shared/performance-rules.md`](frontend/shared/performance-rules.md) | Rendering, lazy loading, image optimization, bundle size, memory |
| [`frontend/shared/security-rules.md`](frontend/shared/security-rules.md) | Secrets, input validation, auth, XSS prevention, dependency security |
| [`frontend/shared/accessibility-rules.md`](frontend/shared/accessibility-rules.md) | Semantic HTML, ARIA, keyboard navigation, color contrast |
| [`frontend/shared/css-rules.md`](frontend/shared/css-rules.md) | Tailwind preference, design tokens, scoped styles, nesting limits |
| [`frontend/shared/asset-rules.md`](frontend/shared/asset-rules.md) | Image optimization, SVG icon components, `currentColor`/`1em` standards |
| [`frontend/shared/code-organization-rules.md`](frontend/shared/code-organization-rules.md) | Index files, Prettier/ESLint setup, import order, arrow functions |

---

## Backend Shared Rules (Node.js)

### Universal (NestJS + Express)

The `backend/node-js/shared/` directory contains rules that apply to **both** NestJS and Express:

| File | Description |
|------|-------------|
| [`backend/node-js/shared/api-design-rules.md`](backend/node-js/shared/api-design-rules.md) | REST conventions, HTTP methods/status codes, pagination, versioning |
| [`backend/node-js/shared/naming-rules.md`](backend/node-js/shared/naming-rules.md) | File/class/method naming, DTO naming, casing conventions |
| [`backend/node-js/shared/code-review-rules.md`](backend/node-js/shared/code-review-rules.md) | Review checklist (API, security, DB, testing), comment types |
| [`backend/node-js/shared/documentation-rules.md`](backend/node-js/shared/documentation-rules.md) | Swagger/OpenAPI, JSDoc, DTO docs, env variable documentation |
| [`backend/node-js/shared/logging-rules.md`](backend/node-js/shared/logging-rules.md) | Structured logging, log levels, correlation ID, sensitive data masking |
| [`backend/node-js/shared/configuration-rules.md`](backend/node-js/shared/configuration-rules.md) | Env validation, config module, secrets management, feature flags |

### NestJS-Specific

The `backend/node-js/rules/nest-js/shared/` directory contains rules specific to **NestJS** (guards, pipes, interceptors, modules):

| File | Description |
|------|-------------|
| [`backend/node-js/rules/nest-js/shared/database-rules.md`](backend/node-js/rules/nest-js/shared/database-rules.md) | BaseEntity, custom Repository pattern, static Mapper classes, Pagination.of() |
| [`backend/node-js/rules/nest-js/shared/error-handling-rules.md`](backend/node-js/rules/nest-js/shared/error-handling-rules.md) | AppException, error code enums, i18n error messages, HttpExceptionFilter |
| [`backend/node-js/rules/nest-js/shared/security-rules.md`](backend/node-js/rules/nest-js/shared/security-rules.md) | JWT cookie auth, JwtWebGuard, RolesGuard, CompanyKeyGuard, custom decorators |
| [`backend/node-js/rules/nest-js/shared/testing-rules.md`](backend/node-js/rules/nest-js/shared/testing-rules.md) | Custom repository mocking, AppException testing, cookie-based e2e tests |
| [`backend/node-js/rules/nest-js/shared/performance-rules.md`](backend/node-js/rules/nest-js/shared/performance-rules.md) | LoggerMiddleware, QueryBuilder field selection, pagination, mapper performance |
| [`backend/node-js/rules/nest-js/shared/caching-rules.md`](backend/node-js/rules/nest-js/shared/caching-rules.md) | Custom RedisService with ioredis, OTP storage, rate limiting, cache patterns |

### Express-Specific

The `backend/node-js/rules/express-js/shared/` directory contains rules specific to **Express** (middleware, manual DI):

| File | Description |
|------|-------------|
| [`backend/node-js/rules/express-js/shared/database-rules.md`](backend/node-js/rules/express-js/shared/database-rules.md) | TypeORM + Express: DataSource setup, service layer, QueryBuilder |
| [`backend/node-js/rules/express-js/shared/error-handling-rules.md`](backend/node-js/rules/express-js/shared/error-handling-rules.md) | Error middleware, asyncHandler wrapper, Zod validation, custom errors |
| [`backend/node-js/rules/express-js/shared/security-rules.md`](backend/node-js/rules/express-js/shared/security-rules.md) | JWT middleware, authorize middleware, express-rate-limit, helmet |
| [`backend/node-js/rules/express-js/shared/testing-rules.md`](backend/node-js/rules/express-js/shared/testing-rules.md) | Supertest, mock repositories, test app factory, middleware testing |
| [`backend/node-js/rules/express-js/shared/performance-rules.md`](backend/node-js/rules/express-js/shared/performance-rules.md) | Compression, response timing middleware, Bull queues, streaming |
| [`backend/node-js/rules/express-js/shared/caching-rules.md`](backend/node-js/rules/express-js/shared/caching-rules.md) | Redis client, cache middleware, manual caching, cache invalidation |

---

## Quick Start

The fastest way to get started — just 3 commands:

```bash
# 1. Download the rule set you need (example: Vue + TypeScript)
curl -o CLAUDE.md https://raw.githubusercontent.com/lazizbekshodmonov/claude-code-rules/main/frontend/rules/vue-ts/CLAUDE.md

# 2. Commit to your project
git add CLAUDE.md
git commit -m "feat: add Claude Code project rules"

# 3. Start Claude Code — it automatically reads CLAUDE.md
claude
```

---

## Installation

### Method 1: Copy a single rule set

Best for: **Most projects.** Simple, no dependencies.

```bash
# Navigate to your project root
cd /path/to/your-project

# Copy the rule set you need
cp /path/to/claude-code-rules/frontend/rules/vue-ts/CLAUDE.md ./CLAUDE.md

# Commit
git add CLAUDE.md
git commit -m "feat: add Claude Code project rules"
```

### Method 2: Clone and pick

Best for: **Trying out different rule sets** or using rules across multiple projects.

```bash
# Clone this repository somewhere on your machine
git clone https://github.com/lazizbekshodmonov/claude-code-rules.git ~/claude-code-rules

# Copy to any project
cp ~/claude-code-rules/frontend/rules/vue-ts/CLAUDE.md /path/to/project-a/CLAUDE.md
```

### Method 3: Use as Git submodule

Best for: **Keeping rules in sync** across multiple projects.

```bash
# Add as submodule in your project
git submodule add https://github.com/lazizbekshodmonov/claude-code-rules.git .claude-rules

# Create a symlink to the rule set you need
ln -s .claude-rules/frontend/rules/vue-ts/CLAUDE.md ./CLAUDE.md

# Commit
git add .gitmodules .claude-rules CLAUDE.md
git commit -m "feat: add Claude Code rules as submodule"
```

To update rules later:

```bash
cd .claude-rules
git pull origin main
cd ..
git add .claude-rules
git commit -m "chore: update Claude Code rules"
```

---

## How It Works

```
┌─────────────────────────────────────────────────────┐
│                  Your Project                        │
│                                                      │
│   CLAUDE.md              ← Claude Code reads this    │
│                                                      │
│   planning-claude-tasks/ ← Task plans saved here     │
│   ├── 2026-02-07-add-auth.md                         │
│   └── 2026-02-08-refactor-dashboard.md               │
│                                                      │
│   src/                                               │
│   └── ...your code                                   │
└─────────────────────────────────────────────────────┘

Developer runs `claude` in terminal
        ↓
Claude Code reads CLAUDE.md automatically
        ↓
All rules are applied to every response
        ↓
Team gets consistent, high-quality code
```

### What Claude Code does with the rules:

1. **Reads `CLAUDE.md`** on startup — no configuration needed
2. **Follows coding standards** — naming, formatting, architecture
3. **Plans before coding** — saves plans to `planning-claude-tasks/`
4. **Writes tests** — unit + integration for every component/function
5. **Runs parallel agents** — splits large tasks for speed
6. **Documents code** — JSDoc for functions, block comments for components

---

## Repository Structure

```
claude-code-rules/
├── README.md
├── shared/                          ← Universal rules (all stacks)
│   ├── agent-rules.md
│   ├── git-rules.md
│   └── planning-rules.md
├── frontend/
│   ├── shared/                      ← Frontend-specific shared rules (13 files)
│   │   ├── naming-rules.md
│   │   ├── typescript-rules.md
│   │   ├── component-rules.md
│   │   ├── api-rules.md
│   │   ├── testing-rules.md
│   │   ├── documentation-rules.md
│   │   ├── code-review-rules.md
│   │   ├── performance-rules.md
│   │   ├── security-rules.md
│   │   ├── accessibility-rules.md
│   │   ├── css-rules.md
│   │   ├── asset-rules.md
│   │   └── code-organization-rules.md
│   └── rules/
│       └── vue-ts/
│           └── CLAUDE.md            ← Vue 3 + TypeScript rules (✅ Ready)
├── backend/
│   ├── node-js/
│   │   ├── shared/                  ← Universal Node.js rules (6 files)
│   │   └── rules/
│   │       ├── nest-js/
│   │       │   ├── shared/          ← NestJS-specific rules (6 files)
│   │       │   └── CLAUDE.md        ← NestJS rules (✅ Ready)
│   │       └── express-js/
│   │           ├── shared/          ← Express-specific rules (6 files)
│   │           └── CLAUDE.md        ← Express rules (🚧 Coming soon)
│   ├── java/
│   │   └── rules/
│   │       └── CLAUDE.md            ← Java + Spring Boot rules (🚧 Coming soon)
│   └── python/
│       └── rules/
│           └── CLAUDE.md            ← Python + FastAPI rules (🚧 Coming soon)
└── mobile/
    └── rules/
        └── flutter/
            └── CLAUDE.md            ← Flutter + Dart rules (🚧 Coming soon)
```

Each `CLAUDE.md` is a **self-contained** file. It includes everything Claude Code needs — coding standards, formatting rules, testing requirements, planning rules, and more.

---

## Customization

### Adding your own rules

Open the `CLAUDE.md` in your project and add rules at the bottom:

```markdown
## 8. Project-Specific Rules

### 8.1. API Configuration
- Base API URL is defined in `.env` as `VITE_API_BASE_URL`
- All API calls must go through `src/services/api.ts`

### 8.2. Authentication
- Auth tokens stored in Pinia store, not localStorage
- Use `useAuth` composable for all auth operations
```

### Combining rule sets

For monorepo or full-stack projects, you can combine rules. Use separate `CLAUDE.md` files in subdirectories:

```
monorepo/
├── CLAUDE.md              ← Shared rules (git, planning, general)
├── frontend/
│   └── CLAUDE.md          ← Vue + TypeScript rules
└── backend/
    └── CLAUDE.md          ← Node.js API rules
```

Claude Code reads `CLAUDE.md` from the current directory AND all parent directories, so shared rules in the root will always apply.

### Personal overrides

For personal preferences that shouldn't be shared with the team:

```bash
# Global personal rules (applies to all your projects)
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md << 'EOF'
# Personal Preferences
- Respond in Uzbek when I write in Uzbek
- Use concise explanations
EOF
```

---

## Team Setup Guide

### Step 1: Choose your rule set

Pick the rule set that matches your project's tech stack from the [Available Rule Sets](#available-rule-sets) table.

### Step 2: Add to your project

```bash
cp frontend/rules/vue-ts/CLAUDE.md ./CLAUDE.md
```

### Step 3: Commit and push

```bash
git add CLAUDE.md
git commit -m "feat: add Claude Code project rules"
git push
```

### Step 4: Notify your team

Share this with your team:

> Claude Code rules have been added to the project. Run `claude` in the project root — rules are applied automatically. No extra setup needed.

---

## FAQ

**Does every team member need to configure something?**
No. Once `CLAUDE.md` is committed to the repo, any team member who runs `claude` will automatically use the rules.

**Can I use different rules for different branches?**
Yes. `CLAUDE.md` is just a file — different branches can have different versions.

**What about Prettier and ESLint?**
Each `CLAUDE.md` includes instructions for Claude Code to automatically install and configure them if missing.

**Will Claude Code always follow these rules perfectly?**
It treats `CLAUDE.md` as strong guidance. For critical rules, linters (ESLint, Prettier) enforce them at the tool level.

**Can I use these rules with other AI coding tools?**
The `CLAUDE.md` format is specific to Claude Code, but you can symlink it for other tools (`ln -s CLAUDE.md CURSOR-RULES.md`).

**How do I update rules across all my projects?**
Use [Method 3 (submodule)](#method-3-use-as-git-submodule) and run `git pull` in the submodule directory.

**What if Claude Code hits context limit?**
The rules include a Context Limit Prevention section that limits sub-agents to 5–10 files and uses `/compact` or `/clear` when needed.

---

## Contributing

Contributions are welcome! Here's how to add a new rule set:

1. Fork this repository
2. Create a new folder under the appropriate category:
   - Frontend: `frontend/rules/<stack-name>/`
   - Backend: `backend/<runtime>/rules/<framework>/`
   - Mobile: `mobile/rules/<framework>/`
3. Add your `CLAUDE.md` with comprehensive rules
4. Update the [Available Rule Sets](#available-rule-sets) table in this README
5. Submit a Pull Request

### Rule set quality checklist:

- [ ] Covers coding standards (naming, formatting, file structure)
- [ ] Includes component/module architecture patterns
- [ ] Has testing requirements
- [ ] Includes planning mode rules
- [ ] Has documentation requirements
- [ ] Includes multi-agent execution guidelines
- [ ] Provides clear examples with incorrect / correct patterns

---

## License

MIT — use these rules freely in your projects.

---

**Star this repo if it helped your team!**
