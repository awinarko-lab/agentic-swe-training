# Git Worktree + Claude Code: Separate Workspaces for AI

> A guide to using Git Worktree as a safe and isolated workflow when collaborating with Claude Code — including handling database sharing issues, environment variables, and day-to-day worktree management.

---

## Table of Contents

1. [Why Isolation Is Needed](#1-why-isolation-is-needed)
2. [Git Worktree — Core Concepts](#2-git-worktree--core-concepts)
3. [Setting Up Worktrees for Claude Code](#3-setting-up-worktrees-for-claude-code)
4. [Common Issues and How to Solve Them](#4-common-issues-and-how-to-solve-them)
   - [.env Not Automatically Copied](#41-env-not-automatically-copied)
   - [Database Sharing Between Worktrees](#42-database-sharing-between-worktrees)
5. [Daily Workflow](#5-daily-workflow)
6. [Helper Scripts](#6-helper-scripts)
7. [Advanced Tips](#7-advanced-tips)
8. [Worktrees: The Engine Behind Parallel Agents](#8-worktrees-the-engine-behind-parallel-agents)

---

## 1. Why Isolation Is Needed

When Claude Code works on a task, it moves fast — actively reading, writing, and modifying files. Without clear isolation, several problems can arise:

**Conflicts with your active work.** If Claude works on a task in the same branch you're working on, changes will get mixed together. It's hard to tell which code is yours and which is Claude's.

**Claude gets confused by unfinished changes.** Half-finished files or uninstalled dependencies can cause Claude to make wrong decisions about the current state of the code.

**No clear checkpoint.** Without a separate branch, rolling back if Claude makes a mistake becomes complicated.

**Git Worktree** solves all of this: one repository, multiple active working directories at the same time, each on a different branch. Claude works in its own directory and you keep working in yours — without interfering with each other.

---

## 2. Git Worktree — Core Concepts

### How Is It Different from a Regular Clone?

```
Regular clone:
~/projects/calculator-app/        ← repo A (full .git/)
~/projects/calculator-app-copy/   ← repo B (full .git/)
→ Two separate repos, no automatic synchronization

Git Worktree:
~/projects/calculator-app/            ← main worktree (has .git/)
~/projects/calculator-app-issue-12/   ← linked worktree (only a pointer to .git/)
→ One .git/, two working directories — branches stay connected
```

Advantages of worktree over clone:

- All branches, tags, and history are available in every worktree
- Commits in a worktree are immediately visible from other worktrees
- No need for `git remote` or `git pull` between worktrees
- Much more disk-efficient (no duplication of `.git/`)

### Basic Commands

```bash
# Add a new worktree with a new branch
git worktree add <path> -b <branch-name>

# Add a worktree from an existing branch
git worktree add <path> <existing-branch>

# List all active worktrees
git worktree list

# Remove a worktree (when done)
git worktree remove <path>

# Force remove (there are uncommitted files)
git worktree remove <path> --force

# Prune worktrees whose directories were manually deleted
git worktree prune
```

---

## 3. Setting Up Worktrees for Claude Code

### Recommended Directory Structure

Place all worktrees outside the main repo directory, in a single parent folder:

```
~/projects/
├── calculator-app/              ← main repo (branch: main)
├── calculator-app-issue-12/     ← worktree for issue #12
├── calculator-app-issue-15/     ← worktree for issue #15
└── calculator-app-feat-memory/  ← worktree for a new feature
```

Avoid placing worktrees inside the main repo directory — this can confuse tools that scan directories (linters, test runners, etc.).

### Creating a Worktree for a Claude Task

```bash
# Make sure you're in the main repo
cd ~/projects/calculator-app

# Create a worktree + new branch at the same time
git worktree add ../calculator-app-issue-12 -b fix/issue-12-division-by-zero

# Enter the worktree
cd ../calculator-app-issue-12

# Run Claude Code here
claude
```

Inside Claude Code:

```
> "Read src/Calculator.ts. Fix the bug in the divide() method that
   produces Infinity when dividing by a number < 1e-15.
   Throw CalculatorError with a descriptive message.
   Include 3 unit tests."
```

### Verify Active Worktrees

```bash
git worktree list
# /home/bruce/projects/calculator-app           abc1234 [main]
# /home/bruce/projects/calculator-app-issue-12  def5678 [fix/issue-12-division-by-zero]
```

---

## 4. Common Issues and How to Solve Them

### 4.1 `.env` Not Automatically Copied

This is the most frequently encountered issue. `.env` (and its variants like `.env.local`, `.env.development`) are usually in `.gitignore` — meaning they **don't get copied when a worktree is created**. Claude Code running in a new worktree will immediately error because environment variables are unavailable.

**Diagnosing the issue:**

```bash
cd ../calculator-app-issue-12
ls -la .env*
# → No output. Files don't exist.

# Check if they're in .gitignore
cat .gitignore | grep .env
# .env
# .env.local
# .env*.local
```

**Solution 1: Manual copy (simplest)**

```bash
# Copy from main repo to the new worktree
cp ~/projects/calculator-app/.env ~/projects/calculator-app-issue-12/.env

# If there are multiple .env files
cp ~/projects/calculator-app/.env* ~/projects/calculator-app-issue-12/
```

**Solution 2: Symlink (always in sync with main)**

Use a symlink so that changes to `.env` in the main repo automatically apply to all worktrees:

```bash
# Create a symlink to the main repo's .env
ln -s ~/projects/calculator-app/.env ~/projects/calculator-app-issue-12/.env

# Verify
ls -la ~/projects/calculator-app-issue-12/.env
# lrwxrwxrwx ... .env -> /home/bruce/projects/calculator-app/.env
```

> **Caution with symlinks:** If Claude Code modifies `.env` in its worktree, the change will directly affect the main repo as well because they point to the same file. To avoid this, use a regular copy if Claude's task might change the configuration.

**Solution 3: Automated script when creating a worktree**

Create a shell function that handles `.env` automatically after a worktree is created:

```bash
# In ~/.bashrc or ~/.zshrc
function claude-task() {
  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"

  # Create worktree
  git worktree add "$worktree_dir" -b "$branch"

  # Copy all existing .env files
  for env_file in .env .env.local .env.development .env.test; do
    if [ -f "$env_file" ]; then
      cp "$env_file" "${worktree_dir}/${env_file}"
      echo "✓ Copied ${env_file}"
    fi
  done

  # Install dependencies if there's a package.json
  if [ -f "package.json" ]; then
    echo "→ Installing dependencies..."
    (cd "$worktree_dir" && npm install --silent)
    echo "✓ Dependencies installed"
  fi

  # Install dependencies if there's a composer.json
  if [ -f "composer.json" ]; then
    echo "→ Installing PHP dependencies..."
    (cd "$worktree_dir" && composer install --quiet)
    echo "✓ Composer dependencies installed"
  fi

  echo ""
  echo "✓ Worktree ready at: $worktree_dir"
  echo "✓ Branch: $branch"
  echo ""
  echo "Run:"
  echo "  cd $worktree_dir && claude"
}
```

Usage:

```bash
cd ~/projects/calculator-app
claude-task 12
# ✓ Copied .env
# ✓ Copied .env.local
# → Installing dependencies...
# ✓ Dependencies installed
# ✓ Worktree ready at: ../calculator-app-issue-12
# ✓ Branch: task/issue-12
```

---

### 4.2 Database Sharing Between Worktrees

This is a subtler and potentially more dangerous issue. By default, **all worktrees share the same database** because they all read the same `DATABASE_URL` from `.env`.

#### Dangerous Scenario

Imagine Bruce is working on a feature in the main repo, and Claude Code is working on an issue in a separate worktree:

```
main repo (Bruce):
→ Developing a new feature, has test data in the DB

calculator-app-issue-12 (Claude):
→ Claude runs a migration to fix issue #12
→ Migration alters the calculation_history table schema
→ BOOM: schema changed, Bruce's code in main repo is now broken
```

Or even worse:

```
calculator-app-issue-15 (Claude):
→ Claude runs a seeder to set up test data
→ Your production/staging data gets overwritten
```

#### Strategy 1: Separate Database per Worktree (Recommended)

Create a dedicated database for each Claude worktree. This is the safest approach.

```bash
# Create a new database for the issue-12 worktree
createdb calculator_app_issue_12

# Or for PostgreSQL via psql
psql -c "CREATE DATABASE calculator_app_issue_12;"
```

When creating a worktree, override `DATABASE_URL` in its `.env` file:

```bash
function claude-task() {
  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"
  local db_name="${repo_name//-/_}_issue_${issue}"  # calculator_app_issue_12

  # Create worktree
  git worktree add "$worktree_dir" -b "$branch"

  # Copy .env
  cp .env "${worktree_dir}/.env"

  # Override DATABASE_URL with the new database
  # For PostgreSQL
  sed -i "s|DATABASE_URL=.*|DATABASE_URL=postgresql://bruce:***@localhost:5432/${db_name}|" \
    "${worktree_dir}/.env"

  # Create the database
  createdb "$db_name" 2>/dev/null && echo "✓ Database '$db_name' created" \
    || echo "⚠ Database '$db_name' already exists, reusing"

  # Run migrations on the new database
  echo "→ Running migrations on ${db_name}..."
  (cd "$worktree_dir" && npm run migrate 2>/dev/null \
    || npx prisma migrate deploy 2>/dev/null \
    || echo "⚠ No recognized migration runner found, run manually")

  echo "✓ Worktree ready at: $worktree_dir"
}
```

#### Strategy 2: Same Database, But Tell Claude

If creating a new database is too heavy, at least document this in `CLAUDE.md` so Claude doesn't carelessly run migrations or seeders:

```markdown
# CLAUDE.md

## ⚠ Warning: Database Sharing

This project uses a database that is shared with other worktrees.

### What Claude MAY do:
- SELECT queries to read data
- INSERT/UPDATE existing test data
- Create migration files (but don't run them)

### What Claude MUST NOT do without explicit confirmation:
- Run `npm run migrate` or `npx prisma migrate`
- Run seeders that delete or overwrite data
- DROP TABLE or ALTER TABLE directly
- Delete existing data

If you need to run a migration, ask Bruce first.
```

#### Strategy 3: Docker Compose per Worktree

For full isolation, run a separate database instance via Docker:

```yaml
# docker-compose.worktree.yml
# Place this file in the worktree
version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: calculator_worktree
      POSTGRES_USER: bruce
      POSTGRES_PASSWORD: pass
    ports:
      - "0:5432"  # Random port, avoid conflicts
    volumes:
      - worktree_db:/var/lib/postgresql/data

volumes:
  worktree_db:
```

```bash
# Inside the worktree
docker compose -f docker-compose.worktree.yml up -d

# Find the assigned port
DB_PORT=$(docker compose -f docker-compose.worktree.yml port db 5432 | cut -d: -f2)
echo "Database running on port: $DB_PORT"

# Update .env with the correct port
sed -i "s|DATABASE_URL=.*|DATABASE_URL=postgresql://bruce:***@localhost:${DB_PORT}/calculator_worktree|" .env
```

#### Strategy 4: Table Prefix (for very specific cases)

For some frameworks that support it, use a different table prefix per worktree:

```bash
# .env in the worktree
DB_TABLE_PREFIX=issue12_
# With this prefix: issue12_users, issue12_calculations, etc.
```

This is only practical if your framework supports table prefixes and you don't have many migrations.

---

### 4.3 Other Issues to Anticipate

**Node modules / vendor not present:**

```bash
# New worktree doesn't have node_modules
cd ../calculator-app-issue-12
npm install        # for Node.js
composer install   # for PHP
```

**Port conflicts when running dev servers:**

If the main repo and a worktree both run a dev server, they will compete for the same port.

```bash
# In the worktree's .env, override the port
PORT=3001          # main uses 3000, worktree uses 3001
```

Or tell Claude in the prompt:

```
> "If you need to run a server for testing, use port 3001
   instead of 3000 because port 3000 is already in use in the main repo."
```

**Interfering caches:**

Some tools store cache in shared locations (e.g., Jest cache):

```bash
# Run tests without cache in the worktree
npm test -- --no-cache
```

---

## 5. Daily Workflow

### Scenario: Bruce Gets a New Issue

Issue #12: "Calculator.divide() produces Infinity for input < 1e-15"

```bash
# 1. From the main repo, create a worktree + set up the environment
cd ~/projects/calculator-app
claude-task 12
# → Creates worktree, copies .env, creates DB, installs deps

# 2. Enter the worktree and open Claude Code
cd ../calculator-app-issue-12
claude

# 3. Inside Claude Code
# > "Read CLAUDE.md and src/Calculator.ts.
#    Fix the divide() bug per the issue: produces Infinity for input < 1e-15.
#    Add tests. Do not run any migrations."
```

### Scenario: Working on Two Issues Simultaneously

Bruce works on a new feature in main, Claude works on two different bug fixes:

```bash
# Terminal 1 — Bruce in the main repo
cd ~/projects/calculator-app
# Bruce codes here

# Terminal 2 — Claude working on issue #12
cd ~/projects/calculator-app-issue-12
claude
# > "Fix the division by zero bug..."

# Terminal 3 — Claude working on issue #15
cd ~/projects/calculator-app-issue-15
claude
# > "Fix the overflow bug in multiply()..."
```

All three terminals run concurrently without conflicts.

### Scenario: Review and Merge

```bash
# After Claude is done, go back to the main repo for review
cd ~/projects/calculator-app

# See all changes made by Claude
git diff main..fix/issue-12-division-by-zero

# Or more visual with difftool
git difftool main..fix/issue-12-division-by-zero

# Run tests from the worktree
cd ../calculator-app-issue-12
npm test

# If OK, push and create a PR from the main repo
cd ~/projects/calculator-app
git push origin fix/issue-12-division-by-zero
gh pr create --base main --head fix/issue-12-division-by-zero

# After PR is merged, clean up
git worktree remove ../calculator-app-issue-12
git branch -d fix/issue-12-division-by-zero

# Also drop the database
dropdb calculator_app_issue_12
```

---

## 6. Helper Scripts

Save all of these in `~/.bashrc` or `~/.zshrc`:

```bash
# ─────────────────────────────────────────────
# Git Worktree + Claude Code Helpers
# ─────────────────────────────────────────────

# Create a worktree for a Claude task (with full setup)
function claude-task() {
  if [ -z "$1" ]; then
    echo "Usage: claude-task <issue-number>"
    return 1
  fi

  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"
  local db_name="${repo_name//-/_}_issue_${issue}"

  # Make sure we're inside a git repo
  if ! git rev-parse --is-inside-work-tree > /dev/null 2>&1; then
    echo "❌ Not inside a git repository"
    return 1
  fi

  # Check if worktree already exists
  if [ -d "$worktree_dir" ]; then
    echo "⚠ Worktree already exists at $worktree_dir"
    echo "Run: cd $worktree_dir && claude"
    return 0
  fi

  echo "→ Creating worktree: $worktree_dir (branch: $branch)"
  git worktree add "$worktree_dir" -b "$branch" || return 1

  # Copy environment files
  local env_copied=0
  for env_file in .env .env.local .env.development .env.test .env.example; do
    if [ -f "$env_file" ]; then
      cp "$env_file" "${worktree_dir}/${env_file}"
      echo "✓ Copied ${env_file}"
      env_copied=$((env_copied + 1))
    fi
  done
  [ $env_copied -eq 0 ] && echo "⚠ No .env files found"

  # Override DATABASE_URL if present
  if [ -f "${worktree_dir}/.env" ] && grep -q "DATABASE_URL" "${worktree_dir}/.env" 2>/dev/null; then
    # Detect database type from URL
    local db_url=$(grep "DATABASE_URL" "${worktree_dir}/.env" | cut -d= -f2-)
    if echo "$db_url" | grep -q "postgresql\|postgres"; then
      local new_url=$(echo "$db_url" | sed "s|/[^/]*$|/${db_name}|")
      sed -i "s|DATABASE_URL=.*|DATABASE_URL=${new_url}|" "${worktree_dir}/.env"
      createdb "$db_name" 2>/dev/null \
        && echo "✓ PostgreSQL database '$db_name' created" \
        || echo "⚠ Database '$db_name' already exists"
    elif echo "$db_url" | grep -q "mysql"; then
      echo "⚠ MySQL detected — create database manually: CREATE DATABASE ${db_name};"
      echo "   Then update DATABASE_URL in ${worktree_dir}/.env"
    fi
  fi

  # Install dependencies
  if [ -f "package.json" ]; then
    echo "→ npm install..."
    (cd "$worktree_dir" && npm install --silent) && echo "✓ npm dependencies installed"
  fi
  if [ -f "composer.json" ]; then
    echo "→ composer install..."
    (cd "$worktree_dir" && composer install --quiet --no-interaction) \
      && echo "✓ composer dependencies installed"
  fi

  echo ""
  echo "════════════════════════════════"
  echo "✓ Worktree ready!"
  echo "  Path   : $worktree_dir"
  echo "  Branch : $branch"
  echo ""
  echo "  Run:"
  echo "  cd $worktree_dir && claude"
  echo "════════════════════════════════"
}

# Remove a worktree and its database
function claude-task-done() {
  if [ -z "$1" ]; then
    echo "Usage: claude-task-done <issue-number>"
    return 1
  fi

  local issue=$1
  local repo_name=$(basename $(pwd))
  local branch="task/issue-${issue}"
  local worktree_dir="../${repo_name}-issue-${issue}"
  local db_name="${repo_name//-/_}_issue_${issue}"

  # Remove worktree
  if [ -d "$worktree_dir" ]; then
    git worktree remove "$worktree_dir" --force
    echo "✓ Worktree removed: $worktree_dir"
  fi

  # Remove local branch (safe if already merged)
  git branch -d "$branch" 2>/dev/null \
    && echo "✓ Local branch deleted: $branch" \
    || echo "⚠ Branch not deleted (may not be merged yet): $branch"

  # Remove database
  if command -v dropdb > /dev/null 2>&1; then
    dropdb "$db_name" 2>/dev/null \
      && echo "✓ Database deleted: $db_name" \
      || echo "⚠ Database not found: $db_name"
  fi

  echo "✓ Cleanup complete for issue #${issue}"
}

# List all active worktrees with status
function claude-tasks() {
  echo "Active worktrees:"
  git worktree list
}

# Shorthand aliases
alias gwl='git worktree list'
alias gwp='git worktree prune'
```

---

## 7. Advanced Tips

### Use CLAUDE.md to Prevent Damage

Place important instructions in `CLAUDE.md` so Claude doesn't do anything dangerous without confirmation:

```markdown
# CLAUDE.md

## Worktree Context

You are working in an isolated worktree for a specific task.
The active branch is not `main` — changes here do not affect the main repo.

## Database

This worktree has its own database (see DATABASE_URL in .env).
You may run migrations on this database.
DO NOT change DATABASE_URL to point to a different database.

## Actions Requiring Confirmation Before Execution

- Deleting files unrelated to the task
- Changing deployment configuration
- Installing new dependencies not already in package.json
- Running scripts that affect data (seeders, fixture resets)

## Task Scope

Focus only on the changes needed for this issue.
If you find other bugs, just note them — don't fix them now.
```

### Frequent Commits from Claude

Ask Claude to commit incrementally, not all at once at the end:

```
> "Work in small commits:
   1. First commit: add failing tests
   2. Second commit: implement the fix
   3. Third commit: update documentation if any"
```

This makes the history easier to review and easier to roll back if something goes wrong.

### Check Differences Before PR

Before pushing, always review from the main repo:

```bash
# See all changed files
git diff --name-only main..fix/issue-12-division-by-zero

# See diff per file
git diff main..fix/issue-12-division-by-zero -- src/Calculator.ts

# See commit history in the worktree
git log main..fix/issue-12-division-by-zero --oneline
```

### Limit Claude's Scope via Flags

```bash
# Run Claude with read-only access first for review
claude --allowedTools "Read,Glob,Grep"

# After review, give full access for implementation
claude
```

### Worktrees for Free Experimentation

Sometimes you want Claude to experiment without fear of breaking anything — for example, trying a major refactor or proof of concept:

```bash
# Create an experimental worktree from a stale branch
git worktree add ../calculator-app-experiment -b experiment/refactor-engine

cd ../calculator-app-experiment
claude

# > "Try refactoring the Calculator class using the Strategy pattern.
#    This is an experiment, you're free to change anything.
#    Let's see the result before deciding whether to merge or not."
```

If the result is good, cherry-pick the relevant commits. If not, just delete the worktree with no side effects.

---

## 8. Worktrees: The Engine Behind Parallel Agents

Everything you've learned in this guide about worktrees is not just "good practice" — **it is the mechanism that both Claude Code and Cursor AI use to run parallel agents**.

When you run multiple agents at the same time, they are literally running in separate git worktrees behind the scene. This means every problem we covered in this guide (`.env` not copied, database conflicts, port collisions) applies to every parallel agent session.

### How Claude Code Uses Worktrees

Claude Code has **first-class worktree support** built directly into the tool:

**`-w` flag** — create an isolated session in a worktree:

```bash
# Claude creates a worktree + branch automatically
claude -w fix-division-bug

# Behind the scene, Claude runs:
# git worktree add .claude/worktrees/fix-division-bug -b worktree-fix-division-bug
# → Session starts inside the worktree directory
# → Agent works in full isolation
# → When done, merge like a normal PR
```

**Sub-agents with `isolation: worktree`** — in custom agent definitions (`.claude/agents/`):

```markdown
---
name: feature-builder
isolation: worktree
---

You are a feature builder. Work on the assigned task in your isolated worktree.
```

Every time this agent runs, Claude automatically creates a fresh worktree. Multiple agents running in parallel each get their own.

**`/worktree` command** — interactive worktree management from within Claude Code.

**`/batch` command** — spins up multiple worktree-isolated agents in parallel with a single prompt. This is the built-in way to say "run these 3 tasks in parallel."

**Agent Teams** — when you run Claude Code Agent Teams, each teammate session gets its own worktree for file isolation.

**`WorktreeCreate` / `WorktreeRemove` hooks** — Claude Code has dedicated hook events for worktree lifecycle. This is exactly how you solve the `.env` copy and database setup problem automatically:

```json
{
  "hooks": {
    "WorktreeCreate": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/worktree-setup.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# scripts/worktree-setup.sh
# Runs every time Claude creates a new worktree

WORKTREE_DIR=$1

# Copy .env files
for env_file in .env .env.local .env.test; do
  [ -f "$env_file" ] && cp "$env_file" "$WORKTREE_DIR/$env_file"
done

# Create isolated database
DB_NAME="app_$(basename $WORKTREE_DIR | sed 's/[^a-z0-9]/_/g')"
createdb "$DB_NAME" 2>/dev/null
sed -i "s|DB_DATABASE=.*|DB_DATABASE=$DB_NAME|" "$WORKTREE_DIR/.env"

# Install dependencies
(cd "$WORKTREE_DIR" && npm install --silent 2>/dev/null)

echo "$WORKTREE_DIR"
```

**Desktop app** — Claude Code Desktop automatically isolates every new session in its own worktree.

### How Cursor AI Uses Worktrees

Cursor's **Parallel Agents** (Agents Window in Cursor 3) use the exact same mechanism:

```
You assign tasks to multiple agents in the Agents Window:
  Agent A: "Fix bug #12"
  Agent B: "Add reporting feature"
  Agent C: "Refactor auth module"

Behind the scene, Cursor runs:
  git worktree add ../agent-a -b feature/agent-a
  git worktree add ../agent-b -b feature/agent-b
  git worktree add ../agent-c -b feature/agent-c
```

Each agent operates in its own worktree on its own branch. They can't touch each other's files. When agents finish, you review and merge their branches like normal PRs.

### Why This Matters

Understanding that worktrees power parallel agents changes how you think about both tools:

**1. The problems from Section 4 apply to every agent session.**

When Claude Code or Cursor spawns a parallel agent in a worktree, that worktree has the same problems we covered earlier:
- `.env` files are not copied (use `WorktreeCreate` hooks to fix this)
- Database is shared by default (configure per-worktree databases)
- Dependencies are not installed (use hooks or setup scripts)
- Dev server ports conflict (assign different ports per worktree)

**2. You can control agent behavior through worktree hooks.**

Claude Code's `WorktreeCreate` hook lets you run setup scripts every time an agent starts. This means you can:
- Auto-copy `.env` files
- Create isolated databases
- Install dependencies
- Set up the environment before the agent even starts coding

**3. Cleaning up after agents is the same as cleaning up worktrees.**

```bash
# List all active agent worktrees
git worktree list

# Clean up finished agent worktrees
git worktree remove .claude/worktrees/fix-division-bug
git branch -d worktree-fix-division-bug
```

**4. The `-w` flag makes manual worktree setup optional.**

If you're using Claude Code, you don't always need the helper scripts from Section 6. The `-w` flag and `WorktreeCreate` hooks handle most of the setup automatically. But understanding what happens behind the scene (this guide) helps you debug when things go wrong.

### Summary: Worktrees Everywhere

```
You manually create a worktree     Claude Code -w flag       Cursor Parallel Agents
          │                                │                              │
          ▼                                ▼                              ▼
  git worktree add <path>      git worktree add              git worktree add
  -b <branch>                  .claude/worktrees/<name>      ../agent-<name>
                               -b worktree-<name>           -b feature/agent-<name>
          │                                │                              │
          ▼                                ▼                              ▼
  All the same problems: .env not copied, DB shared, deps missing, port conflicts
          │                                │                              │
          ▼                                ▼                              ▼
  All the same solutions: setup scripts, hooks, isolated databases
```

**Same mechanism, different entry points.** Whether you create worktrees manually, let Claude Code do it with `-w`, or let Cursor handle it via Parallel Agents — it's all git worktrees under the hood. That's why this module comes first: everything else builds on this foundation.
