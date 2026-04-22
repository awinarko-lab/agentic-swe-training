# Agentic SWE Training

> Practical training material for software engineers learning to work effectively with agentic AI systems like Claude Code. Built from real hands-on experience.

---

## 🗺️ Learning Map

Follow this path from top to bottom. Each module builds on the previous one.

```
Module 1          Module 2              Module 3
Fundamentals      Environment           Advanced
──────────        ──────────            ────────
Git Worktree     Claude Code           Plugins &
Setup            Basics                Extensions
     │                │                    │
     ▼                ▼                    ▼
```

### Module 1: Git Worktree Setup
**[→ Read the guide](guides/id/01-git-worktree-and-claude-code.md)**

Learn how to create isolated workspaces so you and Claude Code can work on different tasks simultaneously without conflicts.

- What git worktrees are and why they matter for agentic workflows
- Setting up worktrees for Claude Code tasks
- Handling `.env`, database sharing, and dependency issues
- Helper scripts for daily workflow

### Module 2: Claude Code Basics *(coming soon)*

Core concepts for working with Claude Code effectively.

- How to write good prompts and task descriptions
- CLAUDE.md as project constitution
- Reading and reviewing Claude's output
- When to trust, when to verify

### Module 3: Plugins & Extensions
**[→ Read the guide](guides/id/02-claude-code-plugins-guide.md)**

Level up with Claude Code's extension system — sub-agents, skills, hooks, MCP, and plugins.

- Sub-Agents for parallel isolated tasks
- Agent Teams for multi-agent coordination
- Skills (auto-invocable & manual slash commands)
- Hooks for deterministic enforcement
- MCP for external tool integration

---

## ✅ Prerequisites

Before starting, make sure you have these foundations:

### Required

| Skill | Why it matters |
|-------|---------------|
| **Git basics** — commit, branch, merge, rebase, diff | Every guide assumes git fluency. Claude Code works within git workflows. |
| **Command line** — navigate directories, run scripts, edit files | Claude Code is a CLI tool. You need to be comfortable in the terminal. |
| **Your primary language** — read/write code confidently | Claude assists you, not replaces you. You must understand the code it generates. |

### Recommended

| Skill | Why it matters |
|-------|---------------|
| **Code review** — reading diffs, spotting issues | You'll be reviewing Claude's output constantly. |
| **Testing** — writing and running tests | "Trust but verify" — tests are your safety net. |
| **Project you work on daily** | Apply these techniques on a real codebase, not just tutorials. |

---

## 🛠️ Tools You'll Use

### Core Tool

| Tool | What it does | Install |
|------|-------------|---------|
| [Claude Code](https://claude.ai/claude-code) | Agentic coding assistant (CLI) | `npm install -g @anthropic-ai/claude-code` |

### Supporting Tools (referenced in guides)

| Tool | Used in | What for |
|------|---------|----------|
| **Git** | All modules | Version control, worktrees, branching |
| **GitHub CLI (`gh`)** | Module 3 | PR creation, issue management from terminal |
| **Node.js / npm** | Module 3 | MCP servers, some Claude Code plugins |
| **jq** | Module 3 (Hooks) | Parsing JSON in shell scripts |
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

---

## 📁 Repository Structure

```
.
├── guides/
│   └── id/                              # Bahasa Indonesia guides
│       ├── 01-git-worktree-and-claude-code.md
│       └── 02-claude-code-plugins-guide.md
├── .claude/
│   └── agents/
│       └── commit-message-writer.md     # Example sub-agent
└── README.md                            # You are here
```

---

## 📝 How to Use This Repo

1. **Clone it** and read through the learning map above
2. **Check prerequisites** — don't skip fundamentals
3. **Install tools** — at minimum Claude Code and Git
4. **Follow modules in order** — each builds on the last
5. **Practice on your real project** — this is a training repo, not a sandbox

---

## 🤝 Contributing

This is a living training resource. If you discover a useful workflow, hit a pitfall worth documenting, or want to add a guide in another language — contributions are welcome.

---

*Built from real experience. Updated as we learn.*
