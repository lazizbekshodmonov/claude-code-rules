# 📋 Claude Code Project Rules

A collection of battle-tested `CLAUDE.md` rule sets for different project types. These rules ensure Claude Code follows your team's coding standards, architecture patterns, and best practices consistently across all team members.

## 📖 Table of Contents

- [What is this?](#what-is-this)
- [Available Rule Sets](#available-rule-sets)
- [Quick Start](#quick-start)
- [Installation](#installation)
  - [Method 1: Copy a single rule set](#method-1-copy-a-single-rule-set)
  - [Method 2: Clone and pick](#method-2-clone-and-pick)
  - [Method 3: Use as Git submodule](#method-3-use-as-git-submodule)
- [How It Works](#how-it-works)
- [Repository Structure](#repository-structure)
- [Customization](#customization)
  - [Adding your own rules](#adding-your-own-rules)
  - [Combining rule sets](#combining-rule-sets)
  - [Personal overrides](#personal-overrides)
- [Team Setup Guide](#team-setup-guide)
- [Planning & Task Tracking](#planning--task-tracking)
- [FAQ](#faq)
- [Contributing](#contributing)

---

## What is this?

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) is Anthropic's CLI tool for AI-powered coding. It reads a special `CLAUDE.md` file from your project root and follows the rules defined in it — coding standards, architecture patterns, formatting rules, testing requirements, and more.

This repository provides **ready-to-use rule sets** for different tech stacks so you don't have to write them from scratch. Just copy the one that matches your project, customize it, and commit it to your repo. Every team member using Claude Code will automatically follow the same rules.

### Benefits

- **Team consistency** — Every developer gets the same AI behavior
- **Zero onboarding** — New team members just `git clone` and start
- **Battle-tested** — Rules refined through real project usage
- **Customizable** — Use as a starting point and adapt to your needs

---

## Available Rule Sets

| Rule Set | Tech Stack | File |
|----------|-----------|------|
| Vue + TypeScript | Vue 3, TypeScript, Tailwind CSS | [`rules/vue-ts/CLAUDE.md`](frontend/rules/vue-ts/CLAUDE.md) |
| React + TypeScript | React, TypeScript, Tailwind CSS | [`rules/react-ts/CLAUDE.md`](frontend/rules/react-ts/CLAUDE.md) |
| Nuxt 3 | Nuxt 3, Vue 3, TypeScript | [`rules/nuxt3/CLAUDE.md`](frontend/rules/nuxt3/CLAUDE.md) |
| Next.js | Next.js, React, TypeScript | [`rules/nextjs/CLAUDE.md`](frontend/rules/nextjs/CLAUDE.md) |
| Node.js API | Express/Fastify, TypeScript, Prisma | [`rules/node-api/CLAUDE.md`](frontend/rules/node-api/CLAUDE.md) |
| Flutter | Dart, Flutter, BLoC | [`rules/flutter/CLAUDE.md`](frontend/rules/flutter/CLAUDE.md) |

> 💡 Don't see your stack? [Contribute](#contributing) a new rule set or open an issue!

---

## Quick Start

The fastest way to get started — just 3 commands:

```bash
# 1. Download the rule set you need (example: Vue + TypeScript)
curl -o CLAUDE.md https://raw.githubusercontent.com/YOUR_USERNAME/claude-code-rules/main/rules/vue-ts/CLAUDE.md

# 2. Commit to your project
git add CLAUDE.md
git commit -m "feat: add Claude Code project rules"

# 3. Start Claude Code — it automatically reads CLAUDE.md
claude
```

That's it. Claude Code will now follow all the rules defined in the file.

---

## Installation

### Method 1: Copy a single rule set

Best for: **Most projects.** Simple, no dependencies.

```bash
# Navigate to your project root
cd /path/to/your-project

# Copy the rule set you need
cp /path/to/claude-code-rules/rules/vue-ts/CLAUDE.md ./CLAUDE.md

# Commit
git add CLAUDE.md
git commit -m "feat: add Claude Code project rules"
```

### Method 2: Clone and pick

Best for: **Trying out different rule sets** or using rules across multiple projects.

```bash
# Clone this repository somewhere on your machine
git clone https://github.com/YOUR_USERNAME/claude-code-rules.git ~/claude-code-rules

# Copy to any project
cp ~/claude-code-rules/rules/vue-ts/CLAUDE.md /path/to/project-a/CLAUDE.md
cp ~/claude-code-rules/rules/react-ts/CLAUDE.md /path/to/project-b/CLAUDE.md
cp ~/claude-code-rules/rules/nuxt3/CLAUDE.md /path/to/project-c/CLAUDE.md
```

### Method 3: Use as Git submodule

Best for: **Keeping rules in sync** across multiple projects. When the main rules repo is updated, all projects can pull the latest version.

```bash
# Add as submodule in your project
git submodule add https://github.com/YOUR_USERNAME/claude-code-rules.git .claude-rules

# Create a symlink to the rule set you need
ln -s .claude-rules/rules/vue-ts/CLAUDE.md ./CLAUDE.md

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
3. **Installs tools if missing** — sets up Prettier, ESLint automatically
4. **Plans before coding** — saves plans to `planning-claude-tasks/`
5. **Writes tests** — unit + integration for every component/function
6. **Runs parallel agents** — splits large tasks for speed
7. **Documents code** — JSDoc for functions, block comments for components

---

## Repository Structure

```
claude-code-rules/
├── README.md
└── rules/
    ├── vue-ts/
    │   └── CLAUDE.md        ← Vue 3 + TypeScript rules
    ├── react-ts/
    │   └── CLAUDE.md        ← React + TypeScript rules
    ├── nuxt3/
    │   └── CLAUDE.md        ← Nuxt 3 rules
    ├── nextjs/
    │   └── CLAUDE.md        ← Next.js rules
    ├── node-api/
    │   └── CLAUDE.md        ← Node.js API rules
    └── flutter/
        └── CLAUDE.md        ← Flutter / Dart rules
```

Each `CLAUDE.md` is a **self-contained** file. It includes everything Claude Code needs — coding standards, formatting rules, Prettier/ESLint configs to be auto-installed, testing requirements, planning rules, and more. No extra config files needed.

---

## Customization

### Adding your own rules

Open the `CLAUDE.md` in your project and add rules at the bottom:

```markdown
## 8. Project-Specific Rules

### 8.1. API Configuration
- Base API URL is defined in `.env` as `VITE_API_BASE_URL`
- All API calls must go through `src/services/api.ts`
- Use `useApi` composable for reactive API calls

### 8.2. Authentication
- Auth tokens stored in Pinia store, not localStorage
- Use `useAuth` composable for all auth operations
- Protected routes defined in `router/guards.ts`
```

### Combining rule sets

For monorepo or full-stack projects, you can combine rules:

```bash
# Copy frontend rules to CLAUDE.md
cat rules/vue-ts/CLAUDE.md > ./CLAUDE.md

# Append backend rules
echo -e "\n\n---\n" >> ./CLAUDE.md
cat rules/node-api/CLAUDE.md >> ./CLAUDE.md
```

Or use separate `CLAUDE.md` files in subdirectories:

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
- Show terminal commands with comments
EOF
```

---

## Team Setup Guide

### Step 1: Choose your rule set

Pick the rule set that matches your project's tech stack from the [Available Rule Sets](#available-rule-sets) table.

### Step 2: Add to your project

```bash
cp rules/vue-ts/CLAUDE.md ./CLAUDE.md
```

### Step 3: Create planning directory

```bash
mkdir -p planning-claude-tasks
touch planning-claude-tasks/.gitkeep
```

### Step 4: Commit and push

```bash
git add CLAUDE.md planning-claude-tasks/
git commit -m "feat: add Claude Code project rules"
git push
```

### Step 5: Notify your team

Share this with your team:

> ✅ Claude Code rules have been added to the project.
>
> **To get started:**
> 1. `git pull` to get the latest changes
> 2. Run `claude` in the project root — rules are applied automatically
> 3. Claude Code will auto-install Prettier/ESLint if they are missing
>
> **No extra setup needed.** Claude Code reads `CLAUDE.md` on startup.

---

## Planning & Task Tracking

All rule sets include **mandatory planning mode**. Before writing any code, Claude Code creates a plan in `planning-claude-tasks/`:

```
planning-claude-tasks/
├── 2026-02-07-add-user-auth.md
├── 2026-02-07-refactor-dashboard.md
└── 2026-02-08-fix-api-error-handling.md
```

Each plan file includes:
- Objective and scope
- List of affected files
- Step-by-step implementation plan
- Potential risks and side effects
- Status tracking (Planned → In Progress → Completed)
- Subtask breakdown for parallel agent execution

This gives your team **full visibility** into all AI-assisted development work.

---

## FAQ

### Does every team member need to configure something?

**No.** Once `CLAUDE.md` is committed to the repo, any team member who runs `claude` in the project root will automatically use the rules. Zero configuration needed.

### Can I use different rules for different branches?

**Yes.** `CLAUDE.md` is just a file in your repo. You can have different rules in different branches, and Claude Code will read whichever version is checked out.

### What about Prettier and ESLint?

Each `CLAUDE.md` includes instructions for Claude Code to **automatically install and configure** Prettier and ESLint if they're missing from the project. No need to include config files separately — everything is inside the `CLAUDE.md`.

### Will Claude Code always follow these rules perfectly?

Claude Code treats `CLAUDE.md` as strong guidance. For critical rules, the `CLAUDE.md` instructs Claude Code to also set up linters (ESLint, Prettier) that enforce rules at the tool level.

### Can I use these rules with other AI coding tools?

The `CLAUDE.md` format is specific to Claude Code. However, there is an emerging standard called [AGENT.md](https://ampcode.com/AGENT.md) that aims to create a universal format. You can symlink the same file for multiple tools:

```bash
ln -s CLAUDE.md CURSOR-RULES.md
ln -s CLAUDE.md .github/copilot-instructions.md
```

### How do I update rules across all my projects?

If using [Method 3 (submodule)](#method-3-use-as-git-submodule):

```bash
cd .claude-rules && git pull origin main && cd ..
git add .claude-rules && git commit -m "chore: update rules"
```

If using Method 1 or 2, you'll need to manually copy the updated file.

### What if Claude Code hits context limit?

The rules include a **Context Limit Prevention** section that instructs Claude Code to limit each sub-agent to 5–10 files maximum and use `/compact` or `/clear` when needed.

---

## Contributing

Contributions are welcome! Here's how to add a new rule set:

1. Fork this repository
2. Create a new folder in `rules/` with your stack name (e.g., `rules/svelte-ts/`)
3. Add your `CLAUDE.md` with comprehensive rules
4. Update the [Available Rule Sets](#available-rule-sets) table in this README
5. Submit a Pull Request

### Rule set quality checklist:

- [ ] Covers coding standards (naming, formatting, file structure)
- [ ] Includes component/module architecture patterns
- [ ] Has testing requirements
- [ ] Includes planning mode rules
- [ ] Has JSDoc / documentation requirements
- [ ] Includes multi-agent execution guidelines
- [ ] Contains Prettier/ESLint auto-install instructions
- [ ] Provides clear examples with ❌ / ✅ patterns

---

## License

MIT — use these rules freely in your projects.

---

**⭐ Star this repo if it helped your team!**
