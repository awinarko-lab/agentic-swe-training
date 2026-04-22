# Setup Project & Guardrails — Before Working with AI

> A guide to setting up a project before using an AI coding assistant for the first time. Covers privacy, security, code conventions, testing, and permission layers. Required reading before proceeding to other modules.

---

## Table of Contents

1. [Why Does This Matter?](#1-why-does-this-matter)
2. [Privacy & Security](#2-privacy--security)
   - [2.1 AI Can Read All Your Files](#21-ai-can-read-all-your-files)
   - [2.2 Ignore Files per Tool](#22-ignore-files-per-tool)
   - [2.3 Protect Secrets](#23-protect-secrets)
   - [2.4 Audit What Gets Sent to AI](#24-audit-what-gets-sent-to-ai)
3. [Style Guide & Conventions](#3-style-guide--conventions)
   - [3.1 CLAUDE.md — Project Constitution for Claude Code](#31-claudemd--project-constitution-for-claude-code)
   - [3.2 .cursorrules — Project Constitution for Cursor](#32-cursorrules--project-constitution-for-cursor)
   - [3.3 Conventions That Must Be Documented](#33-conventions-that-must-be-documented)
   - [3.4 Single Source of Truth](#34-single-source-of-truth)
4. [Testing Guardrails](#4-testing-guardrails)
   - [4.1 Pre-commit Hooks](#41-pre-commit-hooks)
   - [4.2 Testing Instructions in AI Config](#42-testing-instructions-in-ai-config)
   - [4.3 CI Pipeline Remains Mandatory](#43-ci-pipeline-remains-mandatory)
   - [4.4 Plan Mode — Read First, Then Modify](#44-plan-mode--read-first-then-modify)
5. [Permission Layers (Defense in Depth)](#5-permission-layers-defense-in-depth)
   - [5.1 Layer 1: OS-Level](#51-layer-1-os-level)
   - [5.2 Layer 2: Config-Level](#52-layer-2-config-level)
   - [5.3 Layer 3: Instruction-Level](#53-layer-3-instruction-level)
   - [5.4 Summary of the 3 Layers](#54-summary-of-the-3-layers)
6. [Checklist — Before You Start](#6-checklist--before-you-start)
7. [References](#7-references)

---

## 1. Why Does This Matter?

Imagine you hire a new junior developer. Before they start coding, you would definitely:

- **Brief them** on project conventions
- **Grant access** to the repo, but not to production secrets
- **Explain the workflow** — how to test, how to deploy
- **Set expectations** — what's allowed, what's not

An AI coding assistant is just like a new junior developer — but more dangerous because:

1. **It reads every file in the project directory**, including ones that should be private
2. **It can execute terminal commands** with your permissions
3. **It has no common sense** — it will follow instructions literally
4. **It's very fast** — it can break many things before you get a chance to review

Good setup takes time upfront, but saves you from major problems later.

---

## 2. Privacy & Security

### 2.1 AI Can Read All Your Files

This is a fact that's often overlooked: **`.gitignore` does NOT protect you from AI**.

`.gitignore` prevents files from being added to the git repository. But AI coding assistants read the **local file system** — not the git index. This means:

```
Files that ARE in .gitignore:
├── .env                    ← AI can read this
├── .env.production         ← AI can read this
├── id_rsa                  ← AI can read this
├── deploy/credentials.json ← AI can read this
└── secrets/                ← AI can read everything in this folder
```

All of those files could be sent to AI servers (Anthropic, OpenAI, etc.) as part of the context.

**What does this mean?**
- API keys could leak
- Database credentials could be exposed
- Internal infrastructure configuration could be read
- Compliance-sensitive data could be sent out

### 2.2 Ignore Files per Tool

Each AI tool has its own ignore mechanism. You need to set up **all** the tools used by your team:

#### Claude Code — `.claudeignore`

```bash
# Create the file at the project root
touch .claudeignore
```

```gitignore
# .claudeignore — files and folders Claude Code must NOT read

# Secrets
.env
.env.*
!.env.example

# Infrastructure
deploy/
terraform/
ansible/
helm/

# Private keys
*.pem
*.key
id_rsa*

# Database dumps
*.sql
*.dump

# Logs (may contain sensitive data)
logs/
*.log

# Internal vendor that must not be shared
internal-tools/
proprietary/
```

#### Cursor — `.cursorignore`

```bash
touch .cursorignore
```

The contents are the same as `.claudeignore`. Cursor respects this file to prevent the agent from reading sensitive files.

#### Claude Code — `.claude/settings.json`

You can also restrict via the settings file:

```json
{
  "permissions": {
    "deny": [
      "Bash(git push --force*)",
      "Bash(rm -rf *)",
      "Bash(DROP TABLE*)",
      "Bash(curl*production*)"
    ]
  }
}
```

> **Important:** Only a small fraction of projects using AI assistants configure ignore rules. Don't be one of those that don't.

### 2.3 Protect Secrets

**Mandatory steps before using AI in a project:**

```bash
# 1. Check if there are secrets in git history
git log --all --diff-filter=A -- '*.env' '*.key' '*.pem' 'credentials*'

# 2. Make sure .env files are in .gitignore
cat .gitignore | grep -E '\.env'

# 3. Make sure .env files are also in .claudeignore and .cursorignore
cat .claudeignore | grep -E '\.env'
cat .cursorignore | grep -E '\.env'

# 4. If secrets were ever committed, ROTATE them now
# API keys, database passwords, JWT secrets — assume they've been leaked
```

**Best practices:**

- Use `.env.example` as a template (this is OK for AI to read)
- Store secrets in **environment variables** or a **secret manager**, not in files
- If the project uses Docker, use Docker secrets or a vault
- Rotate all secrets that have ever been in git history

### 2.4 Audit What Gets Sent to AI

For Claude Code, you can check what gets sent:

```bash
# Run Claude Code with verbose logging
claude --debug

# Or check the transcript
claude sessions list
```

For Cursor, check:
- **Settings → General → Privacy** — you can view and delete data that was sent
- **Activity log** — to see what the agent did

---

## 3. Style Guide & Conventions

### 3.1 CLAUDE.md — Project Constitution for Claude Code

`CLAUDE.md` is a file that Claude reads at **every session**. This is the most important place to define how AI should work in your project.

```bash
# Generate an initial template (optional)
claude
> /init
```

**Example of a good CLAUDE.md:**

```markdown
# Project: payment-gateway

## Build & Test
- Build: `make build`
- Test: `make test`
- Lint: `make lint`
- Run all before commit: `make ci`

## Tech Stack
- Language: Go 1.22
- Database: PostgreSQL + golang-migrate for migrations
- HTTP Framework: chi router
- Queue: Redis + asynq

## Conventions
- All API responses use envelope format: `{data, error, meta}`
- Error handling: return error to caller, DO NOT use panic()
- Naming: camelCase for variables, PascalCase for exports, snake_case for DB columns
- Commit messages: conventional commits (`feat:`, `fix:`, `docs:`, etc.)

## Architecture
- Double-entry accounting: see `docs/accounting.md`
- QRIS payment flow: see `docs/qris-flow.md`
- Service layer pattern: handler → service → repository

## Testing
- Always write tests for new code
- Test files in `*_test.go` next to the file being tested
- Mock external services, DO NOT mock the database (integration tests)
- Target coverage: at least 80% for the service layer

## Things NOT allowed without confirmation
- Running migrations on the production database
- Modifying deployment config files
- Deleting files unrelated to the current task
- Installing dependencies not already in go.mod
```

**CLAUDE.md tips:**

- **Don't make it too long** — Claude reads this on every request. 100-200 lines is ideal.
- **Front-load what's important** — build/test commands at the top, conventions in the middle, restrictions at the bottom.
- **Be specific, not generic** — "use conventional commits" is better than "write good commits."
- **Update when the project changes** — add new tech stack, change conventions, etc.

### 3.2 .cursorrules — Project Constitution for Cursor

The `.cursorrules` file at the project root works the same as CLAUDE.md but for Cursor:

```bash
touch .cursorrules
```

```markdown
You are an expert developer working on the payment-gateway project.

## Rules
- Always run `make lint` and `make test` before suggesting changes
- Follow Go naming conventions and project patterns
- Use chi router for new HTTP endpoints
- All responses use envelope format: {data, error, meta}
- Write tests for new code (target 80% coverage on service layer)
- Use conventional commits: feat:, fix:, docs:, refactor:

## Restrictions
- Never modify deployment configs without asking
- Never run database migrations on production
- Never install packages not in go.mod
- Always ask before deleting files

## Architecture
- Double-entry accounting pattern (see docs/accounting.md)
- Handler → Service → Repository layering
- Integration tests with real database, not mocks
```

> **Note:** For more complex rules in Cursor, use `.cursor/rules/*.mdc` — it supports multiple rule files per concern.

### 3.3 Conventions That Must Be Documented

Not all conventions need to be written down. Focus on the ones **AI frequently gets wrong**:

| Must document | Optional |
|---|---|
| Build, test, lint commands | Comment style |
| Commit message format | Variable naming (there's already a linter) |
| Framework & version used | File organization |
| Architecture patterns (layering, etc.) | Import order |
| Error handling approach | Spacing / formatting |
| Database approach (ORM? raw queries?) | Git branching strategy |
| Testing approach (unit? integration? what to mock?) | Deployment process |
| **Restrictions** — what must not be done | |

**Restrictions are more important than conventions.** AI is more likely to do something it shouldn't than to use the wrong format.

### 3.4 Single Source of Truth

If your team uses both Claude Code AND Cursor, you have a duplication problem:

```
CLAUDE.md        ← conventions for Claude Code
.cursorrules     ← conventions for Cursor
AGENTS.md        ← conventions for Codex/OpenCode (if used later)
```

**Strategies to avoid drift:**

1. **Pick one as the source of truth** — for example, CLAUDE.md
2. **Generate the others automatically** — a script that converts CLAUDE.md → .cursorrules
3. **Or: separate concerns** — shared parts in all files, tool-specific parts in each file

Example structure:

```
docs/
└── ai-conventions.md      ← SOURCE OF TRUTH (human-maintained)

CLAUDE.md                   ← generated from ai-conventions.md + claude-specific
.cursorrules                ← generated from ai-conventions.md + cursor-specific
```

Generator script (optional):

```bash
#!/bin/bash
# scripts/generate-ai-config.sh

CONVENTIONS="docs/ai-conventions.md"

# Claude Code: conventions + claude-specific rules
cat "$CONVENTIONS" > CLAUDE.md
cat docs/claude-specific.md >> CLAUDE.md

# Cursor: conventions + cursor-specific rules
cat "$CONVENTIONS" > .cursorrules
cat docs/cursor-specific.md >> .cursorrules

echo "✓ Generated CLAUDE.md and .cursorrules from $CONVENTIONS"
```

---

## 4. Testing Guardrails

### 4.1 Pre-commit Hooks

Pre-commit hooks ensure every commit (from humans or AI) passes through the same quality gates.

```bash
# Install pre-commit framework
pip install pre-commit

# Create .pre-commit-config.yaml
```

```yaml
# .pre-commit-config.yaml
repos:
  # Linting
  - repo: https://github.com/golangci/golangci-lint
    rev: v1.59.0
    hooks:
      - id: golangci-lint

  # Formatting
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black

  # Secret detection — IMPORTANT!
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

  # Conventional commits
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.1.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

```bash
# Install hooks
pre-commit install
pre-commit install --hook-type commit-msg

# Test that hooks are running
pre-commit run --all-files
```

**Why does this matter for AI?**

AI doesn't always follow formatting rules. Pre-commit hooks enforce consistency regardless of who wrote the code — human or AI.

### 4.2 Testing Instructions in AI Config

Add testing instructions to CLAUDE.md and .cursorrules:

```markdown
## Testing Rules

1. MUST write tests for every new code or logic change
2. Run tests after every change — make sure they all pass
3. If any test is failing, DO NOT proceed to the next task
4. Target coverage: 80% for the service layer
5. Mock external services (API calls, email), DO NOT mock the database
6. Test file naming: `*_test.go` (Go), `*.test.ts` (TypeScript), `*_test.py` (Python)
```

### 4.3 CI Pipeline Remains Mandatory

**AI-generated code goes through the same CI as human-written code.** No exceptions.

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make lint
      - run: make test
      - run: make build
```

> **Never skip CI** for PRs from AI — even if you're "sure" the code is correct.

### 4.4 Plan Mode — Read First, Then Modify

Before letting AI change anything, ask it to **plan first**:

**Claude Code:**

```bash
# Run in plan mode (read-only, no file changes)
claude --plan

# Or restricted tools
claude --allowedTools "Read,Glob,Grep"
```

**Cursor:**

```
# In chat, ask for a plan first:
> "Analyze this bug and create a plan. DO NOT edit any files — just explain
   the steps you'll take and which files will change."
```

**Why is this useful?**

- You can review the plan before AI changes anything
- AI often discovers insights while planning that change its approach
- It reduces wasted work if AI is heading in the wrong direction from the start

---

## 5. Permission Layers (Defense in Depth)

AI security isn't one thing — it's multiple layers. The more layers, the more secure.

### 5.1 Layer 1: OS-Level (Strongest)

Enforcement at the operating system level. Even if AI "misbehaves," it can't bypass this.

| Method | Example | When to use |
|--------|--------|-------------|
| **Docker container** | Run AI in an isolated container | High-risk projects |
| **Separate user** | `ai-user` without sudo access | Simple setups |
| **Read-only filesystem** | Mount project as read-only for plan mode | Review phase |
| **Network restrictions** | Block outbound to production endpoints | During testing |

Example Docker setup for isolation:

```bash
# Run Claude Code in an isolated container
docker run -it --rm \
  -v $(pwd):/workspace \
  --network none \          # no network access
  --read-only \             # read-only filesystem (except workspace)
  claude-code:latest
```

> **Note:** For most projects, Layer 1 is overkill. But it's important to know this option exists.

### 5.2 Layer 2: Config-Level (Practical)

Enforcement through tool configuration. This is the **most balanced** layer between security and convenience.

#### Claude Code — `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(make test*)",
      "Bash(make lint*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git add*)",
      "Bash(npm test*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git push origin main*)",
      "Bash(DROP TABLE*)",
      "Bash(DROP DATABASE*)",
      "Bash(curl*production*)",
      "Bash(curl*staging*)"
    ]
  }
}
```

#### Claude Code — Hooks

Hooks are **shell scripts that run deterministically** on certain events. They are more powerful than instructions in CLAUDE.md.

```bash
# Create a guard script
mkdir -p .claude/hooks
```

```bash
#!/bin/bash
# .claude/hooks/guard.sh — Block dangerous commands

INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# Block destructive commands
if echo "$CMD" | grep -qiE "DROP TABLE|DROP DATABASE|rm -rf|git push --force"; then
  echo "🚫 BLOCKED: This command is not allowed." >&2
  echo "If truly necessary, run it manually in the terminal." >&2
  exit 2  # exit 2 = block + send feedback to AI
fi

exit 0
```

```json
// .claude/settings.json — register the hook
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/hooks/guard.sh"
          }
        ]
      }
    ]
  }
}
```

> **Key difference:** Rules in CLAUDE.md are followed ~70% of the time. Hooks are blocked **100%** of the time. For safety-critical rules, always use hooks.

#### Cursor — Permission Settings

In Cursor, configure through:
- **Settings → Features → Agent** — toggle what the agent is allowed to do
- **Approval mode** — choose "Auto-accept reads, ask for writes" or stricter

### 5.3 Layer 3: Instruction-Level (Weakest)

These are instructions in CLAUDE.md, .cursorrules, or prompts. AI treats them as *context*, not *law*.

```markdown
# In CLAUDE.md or .cursorrules

## ⚠️ Safety Rules
- NEVER run migrations without confirmation
- NEVER push to the main branch
- NEVER delete files unrelated to the current task
- NEVER install new dependencies without confirmation
- Always ask before running commands that modify the database
```

**Why is this still needed?** Because:
- It helps AI make better decisions by default
- It reduces the frequency of dangerous actions (from often → rarely)
- It serves as "human-readable documentation" for other developers on the team

But **don't rely on this as your only line of defense**.

### 5.4 Summary of the 3 Layers

```
┌─────────────────────────────────────────────────┐
│  Layer 3: Instruction (CLAUDE.md, .cursorrules) │  ← ~70% compliance
│  "Don't push to main"                           │
├─────────────────────────────────────────────────┤
│  Layer 2: Config (settings.json, hooks, deny)   │  ← ~100% enforcement
│  Block git push --force via hook                │
├─────────────────────────────────────────────────┤
│  Layer 1: OS (Docker, user isolation, network)  │  ← 100% enforcement
│  Container without network access                │
└─────────────────────────────────────────────────┘
```

| | Layer 3 | Layer 2 | Layer 1 |
|---|---|---|---|
| **Security** | Low | High | Very high |
| **Ease of setup** | Easy | Medium | Hard |
| **Recommended for** | All projects | All projects | High-risk / enterprise |

**Practical recommendations:**
- **All projects:** Layer 2 + Layer 3 (config + instructions)
- **Production / enterprise:** Add Layer 1 (OS isolation)
- **Never:** Only Layer 3 (instructions alone are not enough)

---

## 6. Checklist — Before You Start

Use this checklist for every new project before using AI:

### 🔒 Security

- [ ] Create `.claudeignore` with a list of sensitive files
- [ ] Create `.cursorignore` with a list of sensitive files
- [ ] Ensure `.env` and secrets **cannot** be read by AI
- [ ] Set up `.claude/settings.json` with deny rules
- [ ] (Optional) Set up hooks to block dangerous commands
- [ ] Rotate secrets that were ever committed to git

### 📏 Conventions

- [ ] Create `CLAUDE.md` with build/test commands, conventions, and restrictions
- [ ] Create `.cursorrules` with the same rules
- [ ] Document architecture patterns that must be followed
- [ ] Document what the AI is **not allowed** to do

### 🧪 Testing

- [ ] Set up pre-commit hooks (lint + secret detection + commit format)
- [ ] Add testing instructions to the AI config
- [ ] Ensure the CI pipeline is active and cannot be skipped
- [ ] Test plan mode before letting AI edit files

### ✅ Verification

- [ ] Run AI on the project in **plan mode** — make sure it understands the conventions
- [ ] Ask AI to write simple code — check if it follows the style guide
- [ ] Ask AI to write tests — check if the testing approach is appropriate
- [ ] Try to trigger dangerous commands — make sure they're blocked by hooks/settings

---

## 7. References

- [AI Ignore Rules: Protect Your Secrets](https://appga.pl/2025/11/22/ai-ignore-rules-protect-your-secrets-when-using-code-assistants/)
- [gitignore Won't Save Your Secrets from AI Coding Agents](https://medium.com/ai-mindset/gitignore-wont-save-your-secrets-from-ai-coding-agents-35dddf892061)
- [Claude Code Permissions Docs](https://code.claude.com/docs/en/agent-sdk/permissions)
- [AI Coding Agent Security: Practical Guardrails](https://dev.to/maxkrivich/ai-coding-agent-security-practical-guardrails-for-claude-code-copilot-and-codex-och)
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
- [CLAUDE.md Best Practices](https://uxplanet.org/claude-md-best-practices-1ef4f861ce7c)
- [Claude Code Best Practices from Real Projects](https://ranthebuilder.cloud/blog/claude-code-best-practices-lessons-from-real-projects/)
- [Context Engineering Best Practices for AI Dev Teams](https://packmind.com/context-engineering-ai-coding/context-engineering-best-practices/)
- [Cursor AI IDE Setup Guide](https://petronellatech.com/blog/cursor-ai-ide-setup-guide/)

---

*Good setup upfront saves hours of time later. Don't skip this module.*
