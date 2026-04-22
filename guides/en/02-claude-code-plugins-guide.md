# Claude Code Plugins — Team Guide

> This document explains Claude Code extension features: Sub-Agents, Agent Teams, Skills (+ CMDs), Hooks, and MCP.

---

## What are Claude Code Plugins?

Claude Code Plugins are a **bundle** of various extension components:

- Sub-Agents & Agent Teams
- MCP (Model Context Protocol)
- Skills (including CMDs/Slash Commands)
- Hooks
- LSP

With plugins, you can package and share all these configurations across your entire team or project at once.

Reference: https://medium.com/@alexanderekb/claude-code-plugins-are-confusing-heres-a-quick-start-overview-of-what-s-actually-inside-bb0c2ad1e199

---

## Sub-Agent

Sub-agents are Claude Code's ability to **spawn** one or more agents running in their own **isolated context**. Each sub-agent has its own system prompt, tool permissions, and even a different model from the parent session.

There is no shared state between sub-agents — each agent works independently and only reports results back to the parent.

### When to use

- Parallel tasks that can be worked on independently
- Keeping the main session clean from accumulated context
- Task specialization (code reviewer, security auditor, explorer, etc.)

### How to spawn

**Via natural language:**

```
use parallel sub-agents:
- sub-agent 1: review code quality on this feature
- sub-agent 2: review security issue on this feature
```

**Via @mention:**

```
@security-auditor please audit the authentication module
```

**Via `/agents` command** for interactive management in the UI.

### How to create a custom sub-agent

Create a markdown file in `.claude/agents/` (project-level) or `~/.claude/agents/` (global):

```markdown
---
name: security-auditor
description: Reviews code for security vulnerabilities. Use when auditing authentication, input validation, or data handling.
tools: Read, Grep, Glob
model: claude-opus-4-6
---

You are a security expert. When reviewing code, focus on:
1. Authentication and authorization flaws
2. Input validation and injection risks
3. Sensitive data exposure
4. Dependency vulnerabilities

Report findings grouped by severity: Critical / High / Medium / Low.
```

### Important: Sub-agent limitations

- Sub-agents **cannot spawn other sub-agents** (no infinite recursion)
- Each sub-agent has a **separate context window** — there is no shared state
- Sub-agents can only **report to the parent**, they cannot communicate with other sub-agents
- For coordination between agents, use **Agent Teams** (see the following section)

### References

- https://timdietrich.me/blog/claude-code-parallel-subagents/
- https://code.claude.com/docs/en/sub-agents
- https://github.com/VoltAgent/awesome-claude-code-subagents

---

## Agent Teams *(new feature, Feb 2026)*

Unlike regular sub-agents, **Agent Teams** allow multiple Claude sessions to coordinate with each other — not just reporting to the parent, but communicating directly with one another.

### Sub-Agent vs Agent Teams differences

|| | Sub-Agent | Agent Teams |
|---|---|---|
|| Communication | One-way to parent | Can communicate with each other |
|| Task assignment | Delegated by parent | Self-assigned from shared list |
|| Best for | Simple parallel tasks | Complex multi-step workflows |
|| Release | Jul 2025 | Feb 2026 |

### Example use case

One agent researches, one agent codes, one agent reviews — and they can coordinate on their own without going through the parent session. Ideal for automated end-to-end feature development.

Reference: https://code.claude.com/docs/en/agent-teams

---

## Skills (+ CMDs / Slash Commands)

> ⚠️ **Important update**: Slash Commands (formerly called CMDs) have been **merged into Skills**. The `.claude/commands/` path is still supported as a legacy path, but the canonical path is now `.claude/skills/`. The file format is the same, and Skills gain additional capabilities.

Skills are predefined instruction sets in markdown files that can be invoked either **manually by the user** or **by Claude itself** based on context.

### Two invocation modes

|| Mode | Description |
||------|-----------|
|| **User-invocable** | You type `/skill-name` manually in the terminal |
|| **Auto-invocable** | Claude reads the `description` field and decides on its own when this skill is relevant to load |

This is what differentiates Skills from old CMDs — they can now be **passive** (auto-triggered by Claude) as well as **active** (manually by the user).

### Directory structure

```
~/.claude/skills/
└── code-review/
    ├── SKILL.md          ← required, entry point
    ├── checklist.md      ← supporting file (optional)
    └── scripts/
        └── lint.sh       ← script that Claude can execute
```

### Example SKILL.md

```markdown
---
name: code-review
description: Reviews code for quality, security, and maintainability. Use when user asks to review, audit, or check their code.
disable-model-invocation: false
allowed-tools: Read, Grep, Glob
---

When reviewing code, always check:

1. **Security**: input validation, auth checks, data exposure
2. **Performance**: N+1 queries, unnecessary loops, memory leaks
3. **Maintainability**: naming, complexity, test coverage
4. **Standards**: project conventions (see checklist.md)

Report findings grouped by severity.
```

### Important frontmatter fields

|| Field | Function |
||-------|---------|
|| `name` | Slash command name (`/name`). Lowercase, letters, numbers, and hyphens (max 64 characters) |
|| `description` | Used by Claude for auto-invoke. **Front-load important keywords** at the beginning |
|| `disable-model-invocation: true` | Only the user can invoke — Claude will not auto-trigger. **Required for skills with side effects** such as `/deploy` or `/commit` |
|| `user-invocable: false` | Skill does not appear in the `/` menu, only Claude can invoke it (suitable for background knowledge) |
|| `allowed-tools` | Tools that are pre-approved when the skill is active — without additional permission prompts |
|| `context: fork` | Skill runs in an isolated sub-agent context |
|| `effort` | Override effort level when the skill is active (`low`, `medium`, `high`, `max`) |

### Invocation control

|| Frontmatter | User can invoke | Claude can invoke |
||---|---|---|
|| (default) | ✅ | ✅ |
|| `disable-model-invocation: true` | ✅ | ❌ |
|| `user-invocable: false` | ❌ | ✅ |

### Inject dynamic context (shell execution)

Skills support shell execution **before** the prompt is sent to Claude using the `` !`command` `` syntax:

```markdown
---
name: pr-summary
description: Summarize a pull request for standup
allowed-tools: Bash(gh *)
---

## PR Context
- Diff: !`gh pr diff`
- Comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

Summarize the above pull request in 3 bullet points for a standup update.
```

The `` !`command` `` is executed first, and its output replaces the placeholder. Claude only receives the final result — not the command itself.

### Passing arguments to a skill

Use `$ARGUMENTS` in the skill content:

```markdown
---
name: fix-issue
description: Fix a GitHub issue by number
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue description
2. Implement the fix
3. Write tests
4. Create a commit
```

Type `/fix-issue 123` → Claude receives "Fix GitHub issue 123 following our coding standards..."

For by-index arguments: `$ARGUMENTS[0]`, `$ARGUMENTS[1]`, or shorthand `$0`, `$1`.

### Skill locations

|| Location | Path | Scope |
||--------|------|-------|
|| Enterprise (managed) | via admin | All users in the org |
|| Personal | `~/.claude/skills/` | All your projects |
|| Project | `.claude/skills/` | This project only |
|| Plugin | `<plugin>/skills/` | When the plugin is active |

If the same skill name exists at multiple levels, priority order: **enterprise > personal > project**.

### References

- https://code.claude.com/docs/en/skills
- https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf
- https://claude.com/blog/improving-frontend-design-through-skills

---

## CMDs (Slash Commands) — Current Status

As mentioned above, CMDs have been **merged into Skills**. But some things remain the same:

- CMDs still cannot be used with `claude -p` (non-interactive/headless mode)
- You can still type `/command-name` as usual
- Files in `.claude/commands/` still work (legacy support)

### Example built-in commands that still exist

```
/init       Initialize a new CLAUDE.md for the project
/agents     Manage agent configurations
/hooks      Browse and manage active hooks
/skills     List available skills
/compact    Manual context compaction
/debug      Enable debug logging
```

The main difference between skills and commands now is more about **packaging and UX**: commands are single-file entries with nice autocomplete in the terminal, while skills are directories with supporting files and auto-invocation capabilities.

---

## Hooks

Hooks are user-defined shell commands (or scripts) that are executed **deterministically** at certain lifecycle events in Claude Code.

> **Crucial analogy**: If CLAUDE.md is "ask Claude to always do X", hooks are "force the system to always do X".
>
> Proven test: the rule "never run rm -rf" in CLAUDE.md is followed ~70% of the time. With a hook? **100%** blocked.

This means for safety rules, security enforcement, or automation that must not be missed — use Hooks, not CLAUDE.md.

### Supported lifecycle events

|| Event | When it occurs |
||-------|--------------|
|| `SessionStart` | When a session starts or is resumed |
|| `UserPromptSubmit` | Before the prompt is processed by Claude |
|| `PreToolUse` | Before a tool call — can **block** execution |
|| `PostToolUse` | After a successful tool call |
|| `PostToolUseFailure` | After a failed tool call |
|| `PermissionRequest` | When the permission dialog appears — can auto-approve |
|| `PermissionDenied` | When a tool call is denied by the classifier |
|| `Stop` | When Claude finishes responding |
|| `StopFailure` | When a turn ends due to an API error |
|| `Notification` | When Claude needs input from the user |
|| `SubagentStart` / `SubagentStop` | When a sub-agent spawns/completes |
|| `TaskCreated` / `TaskCompleted` | When a task is created/completed |
|| `PreCompact` / `PostCompact` | Before/after context compaction |
|| `ConfigChange` | When a settings file changes (for audit trail) |
|| `FileChanged` | When a watched file changes on disk |
|| `CwdChanged` | When the working directory changes |
|| `SessionEnd` | When the session ends |

### Example use cases

#### 1. Desktop notification when Claude needs input

```json
{
  "hooks": {
    "Notification": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "notify-send 'Claude Code' 'Needs your attention'"
      }]
    }]
  }
}
```

#### 2. Auto-format files after Claude edits

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
      }]
    }]
  }
}
```

#### 3. Block dangerous commands (PreToolUse)

Create a `.claude/hooks/guard.sh` script:

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# Block destructive commands
if echo "$CMD" | grep -qiE "DROP TABLE|rm -rf|git push --force"; then
  echo "Blocked: this command is not allowed in this project" >&2
  exit 2  # exit code 2 = block + send feedback to Claude
fi

exit 0  # continue
```

Register it in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "./.claude/hooks/guard.sh"
      }]
    }]
  }
}
```

#### 4. Re-inject context after context compaction

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "compact",
      "hooks": [{
        "type": "command",
        "command": "echo 'Reminder: use Bun, not npm. Run bun test before commit.'"
      }]
    }]
  }
}
```

#### 5. Audit log every config change

```json
{
  "hooks": {
    "ConfigChange": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "jq -c '{timestamp: now | todate, file: .file_path}' >> ~/claude-audit.log"
      }]
    }]
  }
}
```

### Exit codes

|| Exit code | Effect |
||-----------|-------|
|| `0` | Continue. stdout output is injected into Claude's context (for `SessionStart` and `UserPromptSubmit`) |
|| `2` | **Block execution**. stderr is sent to Claude as feedback, Claude can adjust |
|| Others | Continue, but log as an error in the transcript and debug log |

### Hook types other than `command`

|| Type | Description |
||------|-----------|
|| `command` | Run a shell script (most common) |
|| `http` | POST event data to an HTTP endpoint — suitable for centralized audit logging or team-wide enforcement |
|| `prompt` | Evaluate conditions using an LLM call (default: Haiku) |
|| `agent` | Verify using a sub-agent with tool access (can read files, run commands) |

### Hook configuration locations

|| File | Scope | Shareable? |
||------|-------|-----------|
|| `~/.claude/settings.json` | Global, all projects | No (local on your machine) |
|| `.claude/settings.json` | Per-project | Yes, can be committed to the repo |
|| `.claude/settings.local.json` | Per-project | No (gitignored) |
|| Managed policy | Organization-wide | Yes, via admin |

### References

- https://code.claude.com/docs/en/hooks-guide
- https://code.claude.com/docs/en/hooks (full reference with all event schemas)

---

## MCP (Model Context Protocol)

MCP connects Claude Code to external tools and data sources — GitHub, databases, browser automation, internal APIs, etc.

```bash
# Install MCP server
claude mcp add playwright npx @playwright/mcp@latest

# Use via slash command
/mcp__playwright__create-test [args]
```

> **Performance note**: Claude Code no longer loads all MCP tool schemas at startup. Only tool names are loaded, and full schemas are fetched on-demand (deferred tool loading). This significantly reduces context overhead if you have 50+ MCP tools.

---

## CLAUDE.md — The Foundation Before All Features

Before using the features above, make sure your `CLAUDE.md` is solid. This file is read by Claude at **every session** and serves as the project's "constitution."

```markdown
# Project: payment-gateway

## Build & Test
- Build: `make build`
- Test: `make test`
- Lint: `make lint`

## Conventions
- Language: Go
- DB migrations via golang-migrate
- All API responses use envelope format: `{data, error, meta}`
- No direct panic() — return errors to the caller

## Architecture
- Double-entry accounting: see `docs/accounting.md`
- QRIS flow: see `docs/qris-flow.md`
```

**Important note**: CLAUDE.md is not a rule that is always followed 100% — Claude treats it as *context*, not *law*. For truly deterministic enforcement that must not be missed, use **Hooks**.

---

## Summary: When to Use What?

|| Need | The right tool |
||-----------|----------------|
|| Project context/conventions that are always active | CLAUDE.md |
|| Reusable workflows that you trigger manually | Skills (`disable-model-invocation: true`) |
|| Knowledge/patterns that Claude auto-applies | Skills (auto-invocable, without flag) |
|| Parallel tasks / isolated execution | Sub-Agents |
|| Coordination between multiple agents | Agent Teams |
|| External tool integration (GitHub, DB, etc.) | MCP |
|| Deterministic enforcement (formatting, blocking, notifications) | Hooks |
|| Bundle all of the above for team sharing | Plugins |

---

## ⚠️ Common Misconceptions

1. **CMDs are no longer a separate component** — they have been merged into Skills. Old files in `.claude/commands/` still work but you should migrate to `.claude/skills/`.

2. **Skills are not just passive** — they can be auto-triggered by Claude. For skills with side effects (deploy, commit, etc.), you **must** add `disable-model-invocation: true`.

3. **CLAUDE.md is not deterministic** — it is followed ~70-80% of the time. For safety rules that must not be missed, use Hooks.

4. **Sub-agents ≠ Agent Teams** — Sub-agents work in isolation and report to the parent. Agent Teams can communicate with each other. Choose based on your coordination needs.

5. **`-p` / headless mode** — Slash commands and interactive features (CMDs) are not available in this mode. Use skills via SDK if you need non-interactive automation.

---

*This document was created based on official Claude Code documentation as of April 2026.*
*Official docs: https://code.claude.com/docs/en/overview*
