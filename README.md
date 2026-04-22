# Agentic SWE Training

> Practical training material for software engineers learning to work effectively with agentic AI coding assistants. Covers **Claude Code** and **Cursor AI**. Built from real hands-on experience.

---

## 🗺️ Learning Map

Follow this path from top to bottom. Each module builds on the previous one.

```
Module 0          Module 1          Module 2               Module 3               Module 4
Setup &           Git Worktree      Claude Code            Cursor AI              Cross-Tool
Guardrails        Setup              Agent in               Agent Orchestrator     Patterns &
──────────        ──────────        ──────────             ─────────               ────────
Privacy,          Isolated          Terminal               (from Editor)          Workflow
Security,         Workspaces                               & Cloud
Conventions                         for AI Tasks
     │                │                      │                      │                      │
     ▼                ▼                      ▼                      ▼                      ▼
```

### Module 0: Project Setup & Guardrails
**[🇬🇧 English](guides/en/00-project-setup-and-guardrails.md)** · **[🇮🇩 Bahasa Indonesia](guides/id/00-project-setup-and-guardrails.md)**

**⚠️ Start here.** Before using any AI assistant on your project, set up guardrails for privacy, security, code conventions, and testing.

- Why AI can read your secrets (`.gitignore` won't save you)
- Ignore files: `.claudeignore`, `.cursorignore`
- Writing `CLAUDE.md` and `.cursorrules` — your project's constitution
- Testing guardrails: pre-commit hooks, CI, plan mode
- 3-layer permission model: instructions → config → OS-level

### Module 1: Git Worktree Setup
**[🇬🇧 English](guides/en/01-git-worktree-and-claude-code.md)** · **[🇮🇩 Bahasa Indonesia](guides/id/01-git-worktree-and-claude-code.md)**

Learn how to create isolated workspaces so you and AI assistants can work on different tasks simultaneously without conflicts.

- What git worktrees are and why they matter for agentic workflows
- Setting up worktrees for AI-assisted tasks
- Handling `.env`, database sharing, and dependency issues
- Helper scripts for daily workflow

> Applies to both Claude Code and Cursor AI.

### Module 2: Claude Code — Agent in Terminal
**[🇬🇧 English](guides/en/02-claude-code-plugins-guide.md)** · **[🇮🇩 Bahasa Indonesia](guides/id/02-claude-code-plugins-guide.md)**

Master Claude Code — Anthropic's agentic coding assistant that lives in your terminal.

- Core concepts: prompts, CLAUDE.md, context management
- Sub-Agents for parallel isolated tasks
- Agent Teams for multi-agent coordination
- Skills (auto-invocable & manual slash commands)
- Hooks for deterministic enforcement
- MCP for external tool integration

### Module 3: Cursor AI — Agent Orchestrator *(coming soon)*

Master Cursor AI — evolved from AI-powered editor into a full agent orchestrator.

> **Note:** As of Cursor 3 (April 2026), Cursor's primary interface is the **Agents Window** — a parallel agent orchestration view. Developers spend more time managing agents than editing code directly. This module focuses on the agent-first workflow.

- Agents Window: run and monitor multiple agents in parallel
- Cloud Agents: offload long tasks to the cloud, pick up from web or mobile
- Cursor CLI: agent modes in the terminal with cloud handoff
- `.cursorrules` and `.cursor/rules/` for project configuration
- MCP integration in Cursor
- When to use the editor view vs the agents view

### Module 4: Cross-Tool Patterns *(coming soon)*

Patterns that apply regardless of which tool you use.

- Writing effective prompts and task descriptions
- Reviewing and verifying AI-generated code
- Choosing the right tool: terminal (Claude Code) vs orchestrator (Cursor) vs both
- Building a personal AI-assisted workflow
- Safety: what to automate, what to verify

---

## 💡 Two Tools, One Philosophy

Both Claude Code and Cursor AI share the same goal: **you orchestrate, agents execute**.

| | Claude Code | Cursor AI |
|---|---|---|
| **Interface** | Terminal / CLI | Agents Window + Editor |
| **Strength** | Lightweight, scriptable, CI/CD friendly | Visual, parallel agents, cloud offload |
| **Best for** | Quick tasks, automation, headless workflows | Multi-file features, long-running tasks, visual monitoring |
| **Agent model** | Sub-agents, Agent Teams | Parallel agents, Cloud Agents |
| **Config** | `CLAUDE.md`, Skills, Hooks | `.cursorrules`, `.cursor/rules/` |
| **Extensibility** | MCP, Plugins | MCP, Marketplace |

You don't have to pick one — most teams use both depending on the task.

---

## ✅ Prerequisites

Before starting, make sure you have these foundations:

### Required

| Skill | Why it matters |
|-------|---------------|
| **Git basics** — commit, branch, merge, rebase, diff | Every guide assumes git fluency. AI assistants work within git workflows. |
| **Command line** — navigate directories, run scripts, edit files | Claude Code is a CLI tool. Cursor also has a CLI mode. |
| **Your primary language** — read/write code confidently | AI assists you, not replaces you. You must understand the code it generates. |

### Recommended

| Skill | Why it matters |
|-------|---------------|
| **Code review** — reading diffs, spotting issues | You'll be reviewing AI output constantly. |
| **Testing** — writing and running tests | "Trust but verify" — tests are your safety net. |
| **VS Code basics** | Cursor is built on VS Code. Familiarity helps. |
| **Project you work on daily** | Apply these techniques on a real codebase, not just tutorials. |

---

## 🛠️ Tools You'll Use

### Core Tools

| Tool | What it does | Install |
|------|-------------|---------|
| [Claude Code](https://claude.ai/claude-code) | Agentic coding assistant (CLI) | `npm install -g @anthropic-ai/claude-code` |
| [Cursor](https://cursor.com) | Agent orchestrator + AI code editor | https://cursor.com |

### Supporting Tools (referenced in guides)

| Tool | Used in | What for |
|------|---------|----------|
| **Git** | All modules | Version control, worktrees, branching |
| **GitHub CLI (`gh`)** | Module 2 & 3 | PR creation, issue management from terminal |
| **Node.js / npm** | Module 2 & 3 | MCP servers, plugins |
| **jq** | Module 2 (Hooks) | Parsing JSON in shell scripts |
| **Docker** | Module 1 (optional) | Isolated databases per worktree |
| **PostgreSQL client** (`psql`, `createdb`) | Module 1 (optional) | Database isolation for worktrees |

### Quick install (Ubuntu/Debian)

```bash
# Core
npm install -g @anthropic-ai/claude-code

# Supporting
sudo apt install -y git jq
npm install -g gh

# Optional
sudo apt install -y postgresql-client docker.io
```

> Cursor is installed via its website — it bundles everything you need for the IDE/agent experience.

---

## 📝 How to Use This Repo

1. **Clone it** and read through the learning map above
2. **Start with Module 0** — set up guardrails before using AI on any project
3. **Check prerequisites** — don't skip fundamentals
4. **Install tools** — start with one (Claude Code or Cursor), add the other later
5. **Follow modules in order** — each builds on the last
6. **Practice on your real project** — this is a training repo, not a sandbox

---

## 🤝 Contributing

This is a living training resource. If you discover a useful workflow, hit a pitfall worth documenting, or want to add a guide in another language — contributions are welcome.

---

*Built from real experience. Updated as we learn.*
