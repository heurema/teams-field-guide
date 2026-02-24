# Claude Code Agent Teams — Field Guide

This is an independently maintained reference for engineers building multi-agent systems with Claude Code. It covers the two parallel multi-agent systems (Subagents and Agent Teams), their full configuration schemas, known bugs with workarounds, orchestration patterns, cost optimization, and a catalog of ecosystem projects. It is intended for AI engineers who want to move beyond toy examples and ship reliable agentic pipelines in production.

For multi-model teams that mix Claude Code with Codex CLI, see [codex-partner](https://github.com/Real-AI-Engineering/codex-partner).

---

## Install as Claude Code Skill

```bash
git clone https://github.com/Real-AI-Engineering/teams-field-guide.git ~/.claude/skills/teams-field-guide-repo
ln -s ~/.claude/skills/teams-field-guide-repo/skills/teams-advisor ~/.claude/skills/teams-advisor
```

The `teams-advisor` skill activates automatically when you work with multi-agent setups, ask about orchestration patterns, or hit team-related issues.

---

## Table of Contents

1. [Requirements & Version Support](#1-requirements--version-support)
2. [TL;DR — Which System to Use](#2-tldr--which-system-to-use)
3. [Two Systems Compared](#3-two-systems-compared)
4. [Custom Agents Reference](#4-custom-agents-reference)
5. [Agent Teams Setup](#5-agent-teams-setup)
6. [Per-Agent Customization Matrix](#6-per-agent-customization-matrix)
7. [Orchestration Patterns](#7-orchestration-patterns)
8. [Known Limitations & Workarounds](#8-known-limitations--workarounds)
9. [Experimental Features](#9-experimental-features)
10. [Cost Optimization](#10-cost-optimization)
11. [Case Studies](#11-case-studies)
12. [Ecosystem](#12-ecosystem)
13. [Contributing](#13-contributing)
14. [Sources](#14-sources)
15. [Changelog](#15-changelog)

---

## 1. Requirements & Version Support

**Minimum version:** Claude Code v2.1.45 for full custom agent support (`memory:`, `TeammateIdle`/`TaskCompleted` hooks, `Task(agent_type)` syntax).

**Agent Teams (experimental):** Requires the environment variable `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

### Version Compatibility Matrix

| Version | Multi-Agent Changes |
|---------|---------------------|
| v2.1.33 | Fixed tmux teammate messaging; fixed false "not available on plan" warning |
| v2.1.41 | Guard against launching Claude inside Claude |
| v2.1.43 | Fixed wrong model identifier for Bedrock/Vertex/Foundry |
| v2.1.45 | `TeammateIdle`/`TaskCompleted` hooks; `Task(agent_type)` syntax; `memory` field in agent frontmatter |
| v2.1.47 | Fixed `model` in `.claude/agents/` being ignored for teammates; fixed agents not discovered from worktrees; `last_assistant_message` in Stop/SubagentStop hooks |
| v2.1.49 | `--worktree`/`-w` CLI flag; `isolation: worktree`; `background: true`; `claude agents` CLI command; `ConfigChange` hook |
| v2.1.50 | `WorktreeCreate`/`WorktreeRemove` hooks; memory leak fix for completed teammate tasks |

---

## 2. TL;DR — Which System to Use

```
Do agents need to talk directly to each other?
    |
    +--> YES:  Agent Teams (TeamCreate + SendMessage)
    |          Experimental. Has known bugs. Use only when
    |          peer-to-peer messaging is genuinely required.
    |
    +--> NO: Do you need per-agent tools, skills, or hooks?
                |
                +--> YES:  Custom Agents + parallel Task() calls
                |          Full configuration, stable, no Teams bugs.
                |          Recommended for most production use cases.
                |
                +--> NO:   Parallel Task() calls (general-purpose)
                           Simplest and cheapest option.
                           All agents use lead's configuration.
```

**Recommendation:** The hybrid approach — define agents in `.claude/agents/` and launch them via parallel `Task()` calls — gives you full per-agent configuration (tools, model, skills, hooks, memory) without any of the Agent Teams instability. Use Agent Teams only when agents genuinely must exchange messages with each other.

---

## 3. Two Systems Compared

| | **Subagents (Parallel Tasks)** | **Agent Teams** |
|---|---|---|
| **Status** | Stable, production-ready | Experimental (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) |
| **Communication** | Worker returns result to parent only | Mesh: any agent can message any other directly |
| **Coordination** | Parent manages all sequencing | Shared task list; agents self-organize |
| **Per-agent config** | Full: tools, model, hooks, memory, skills, permissions | Minimal: spawn prompt only (most config broken or ignored) |
| **Cost** | Lower — summary returned to parent | ~3-7x higher — each teammate holds full context window |
| **Best for** | Focused delegation, independent parallel work, isolation | Parallel research, competing hypotheses, peer coordination |

---

## 4. Custom Agents Reference

Custom agents are defined as Markdown files with YAML frontmatter. The file body is the agent's system prompt. Store them in `.claude/agents/` (project-scoped) or `~/.claude/agents/` (user-scoped).

### Frontmatter Schema

| Field | Required | Type | Default | Description |
|-------|----------|------|---------|-------------|
| `name` | Yes | string | — | Unique identifier. Use lowercase letters and hyphens only. |
| `description` | Yes | string | — | Tells Claude when to auto-delegate. Write "Use when..." with concrete examples. |
| `tools` | No | string or array | inherit all | Allowlist of permitted tools. Use `Task(agent1, agent2)` to restrict which subagents this agent can spawn. |
| `disallowedTools` | No | string or array | none | Denylist. Removed from the inherited tool set. |
| `model` | No | string | inherit | `sonnet`, `opus`, `haiku`, or `inherit`. |
| `permissionMode` | No | string | inherited | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, or `plan`. |
| `maxTurns` | No | integer | unlimited | Maximum agentic turns before the agent stops. |
| `skills` | No | list | none | Skills loaded at agent startup. Listed skills are fully injected (not lazy). Unlisted skills are invisible to the agent. |
| `mcpServers` | No | list | inherited | MCP servers available to this agent. String reference or inline `{name: config}` object. |
| `hooks` | No | object | none | Lifecycle hooks scoped to this agent. Note: only 6 of the 16 hook types work in agent frontmatter: `PreToolUse`, `PostToolUse`, `PermissionRequest`, `PostToolUseFailure`, `Stop`, `SubagentStop` (see [Experimental Features](#9-experimental-features)). |
| `memory` | No | string | none | `user`, `project`, or `local`. Enables persistent cross-session memory. First 200 lines of `MEMORY.md` are auto-injected. |
| `background` | No | boolean | false | Always run as a background task. Added v2.1.49. |
| `isolation` | No | string | none | `worktree` — run in an isolated git worktree with auto-cleanup. Added v2.1.49. |
| `color` | No | string | — | UI color in the terminal: `green`, `cyan`, `red`, `blue`, `yellow`, `magenta`. |

### Example Agent Definition

```yaml
---
name: security-reviewer
description: |
  Use this agent when code needs security review, vulnerability assessment,
  or OWASP compliance checking.
  <example>
  user: "Check the auth module for vulnerabilities"
  assistant: "I'll use security-reviewer agent to audit src/auth/"
  </example>
tools: Read, Glob, Grep, Bash
disallowedTools: Write, Edit
model: sonnet
permissionMode: plan
maxTurns: 30
skills:
  - api-conventions
memory: project
color: red
---

You are a senior security engineer. Focus on OWASP Top 10, injection
vulnerabilities, authentication flaws, and secrets in code. Report
findings with severity (critical/high/medium/low) and a specific fix.
```

### Context Rules

Agents are isolated by design. What an agent does and does not see:

- **Receives:** its own system prompt (file body) + basic environment info + `CLAUDE.md` from the project
- **Does NOT receive:** parent conversation history
- **Does NOT receive:** parent's Claude Code system prompt
- **Does NOT receive:** skills loaded by the parent (must be listed in the agent's own `skills:` field)
- **Loads fresh:** project's `CLAUDE.md` and git status

### Tools Field Syntax

```yaml
# Allowlist — agent can only use these tools
tools: Read, Glob, Grep, Bash

# Restrict which subagents this agent can spawn
tools: Task(worker, researcher), Read, Bash

# If Task is omitted entirely, the agent cannot spawn any subagents
```

Available tool names: `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, `Task`, `WebFetch`, `WebSearch`, `NotebookRead`, `NotebookEdit`, `LS`, `KillShell`, `BashOutput`, `TodoWrite`, plus any MCP tool names.

**Critical:** use `tools:` not `allowed-tools:`. The wrong field name silently grants ALL tools. See [Known Limitations](#8-known-limitations--workarounds).

### Memory Field

```yaml
memory: user     # ~/.claude/agent-memory/<name>/         (shared across all projects)
memory: project  # .claude/agent-memory/<name>/           (git-tracked, project-specific)
memory: local    # .claude/agent-memory-local/<name>/     (gitignored, local only)
```

When a memory location is configured, the first 200 lines of `MEMORY.md` in that directory are automatically injected into the agent's context. `Read`, `Write`, and `Edit` are automatically enabled for the memory path.

### Discovery Priority

| Location | Scope | Priority |
|----------|-------|----------|
| `--agents` CLI JSON flag | Current session only | 1 (highest) |
| `.claude/agents/` | Project (git-committable) | 2 |
| `~/.claude/agents/` | User (all projects) | 3 |
| Plugin `agents/` directory | Per installed plugin | 4 (lowest) |

### CLI Ephemeral Agents

Agents can be defined inline without a file, useful for one-off sessions:

```bash
claude --agents '{
  "reviewer": {
    "description": "Reviews code for security vulnerabilities",
    "prompt": "You are a security expert...",
    "tools": ["Read", "Grep", "Glob"],
    "model": "opus",
    "maxTurns": 30
  }
}'
```

### Managing Agents

```bash
claude agents    # List all discovered agents from CLI
/agents          # Interactive TUI: view, create, edit, delete agent files
```

---

## 5. Agent Teams Setup

Agent Teams enable mesh communication between agents — each teammate can send messages directly to any other. This comes at significant cost and stability tradeoffs.

### Activation

```json
// .claude/settings.json or ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "auto"
}
```

`teammateMode` options: `auto` (Claude picks the best available display), `tmux`, `in-process`.

### The 7 Core Tools

| Tool | Purpose |
|------|---------|
| `TeamCreate` | Create a named team; writes `~/.claude/teams/{name}/config.json` |
| `TaskCreate` | Add a task to the team's shared task list |
| `TaskUpdate` | Update task status, owner, or dependencies |
| `TaskList` | List all tasks (used by workers to claim available work) |
| `Task(team_name=)` | Spawn a teammate into the named team |
| `SendMessage` | Send a message to one or all teammates |
| `TeamDelete` | Tear down the team and clean up resources |

### SendMessage Types

| Type | Behavior |
|------|---------|
| `message` | Targeted direct message to one named teammate |
| `broadcast` | Sent to all active teammates simultaneously (costs N times as much) |
| `shutdown_request` / `shutdown_response` | Graceful coordinated teardown |
| `plan_approval_response` | Approve or reject a teammate's proposed plan |

### Display Modes

| Mode | Navigation Keys | Requirements |
|------|----------------|-------------|
| `in-process` | Shift+Up/Down (switch agents), Enter (select), Esc (back), Ctrl+T (team view) | Any terminal |
| `tmux` | Standard tmux pane navigation | tmux installed |
| `iterm2` | Split pane navigation | iTerm2 + `it2` CLI tool |

**Not supported:** VS Code terminal, Windows Terminal, Ghostty, Zellij, WezTerm.

### Team-Specific Hooks

| Hook Event | Fires When | Blocking? |
|-----------|-----------|-----------|
| `TeammateIdle` | A teammate signals it is going idle | Yes — exit code 2 keeps the teammate working |
| `TaskCompleted` | A task is marked as done in the shared list | Yes — exit code 2 keeps the task open |
| `SubagentStart` | A subagent spawns | No |
| `SubagentStop` | A subagent finishes | Yes |

### Environment Variables Set on Teammates

When a teammate is spawned, these environment variables are set in its process:

```
CLAUDE_CODE_TEAM_NAME           # Name of the team this agent belongs to
CLAUDE_CODE_AGENT_ID            # Unique agent identifier (name@team-name format)
CLAUDE_CODE_AGENT_NAME          # Human-readable agent name
CLAUDE_CODE_AGENT_TYPE          # Agent type (e.g., "general-purpose")
CLAUDE_CODE_PLAN_MODE_REQUIRED  # Whether plan mode is enforced
CLAUDE_CODE_PARENT_SESSION_ID   # Session ID of the lead agent
```

### File System Layout

```
~/.claude/
├── teams/{team-name}/
│   ├── config.json           # members[], leadAgentId, leadSessionId
│   └── inboxes/              # Per-agent message queues
├── tasks/{team-name}/
│   └── {id}.json             # Individual task files
└── agents/
    └── *.md                  # Custom agent definitions
```

---

## 6. Per-Agent Customization Matrix

| Capability | Subagents (Custom Agents + Tasks) | Agent Teams |
|-----------|----------------------------------|------------|
| Custom system prompt | Via `.claude/agents/*.md` body | Spawn prompt (natural language only) |
| Tools allowlist | `tools:` field | ⚠️ Broken — agents lose tools when spawned with `team_name` (#25608) |
| Tools denylist | `disallowedTools:` field | ⚠️ Broken |
| Model selection | `model:` field | `model=` parameter on Task (⚠️ unreliable for in-process teammates, fixed in v2.1.47 for `.claude/agents/`) |
| Skills | `skills:` field (explicit list) | Inherited from project (no per-agent control) |
| MCP servers | `mcpServers:` field | Inherited (no per-agent control) |
| Hooks | `hooks:` in frontmatter | ⚠️ Broken — session ID mismatch prevents firing (#27655) |
| Persistent memory | `memory: user/project/local` | Not supported |
| Permission mode | `permissionMode:` field | Post-spawn only (workaround required) |
| Max turns | `maxTurns:` field | Not supported |
| Git isolation | `isolation: worktree` | Not supported |
| Background mode | `background: true` | Not supported |

---

## 7. Orchestration Patterns

### Pattern 1: Parallel Specialists

Run multiple independent reviewers simultaneously, each with a different lens. Results are returned to the lead for synthesis.

**When to use:** Code review, multi-angle analysis, parallel audits.

```python
# Launch all reviewers simultaneously (run_in_background=True)
Task(subagent_type="security-reviewer", model="sonnet", run_in_background=True,
     prompt="Review src/auth/ for OWASP Top 10 vulnerabilities")

Task(subagent_type="perf-analyst", model="sonnet", run_in_background=True,
     prompt="Profile src/api/ for latency and memory issues")

Task(subagent_type="test-writer", model="sonnet", run_in_background=True,
     prompt="Write unit tests for src/auth/ targeting edge cases")

# Wait for all three, then synthesize findings
```

### Pattern 2: Dependency Wave

Execute tasks in parallel batches where each batch depends on the previous one completing. Avoids sequential bottlenecks while respecting true dependencies.

**When to use:** Build pipelines, staged migrations, multi-phase feature development.

```python
# Wave 1 — independent modules (parallel)
Task(model="sonnet", run_in_background=True, prompt="Build module A")
Task(model="sonnet", run_in_background=True, prompt="Build module B")
Task(model="sonnet", run_in_background=True, prompt="Build module C")
# Wait for all wave 1 tasks

# Wave 2 — depends on wave 1 output (parallel)
Task(model="sonnet", run_in_background=True, prompt="Integrate module A and B")
Task(model="opus",   run_in_background=True, prompt="Architecture review of all modules")
```

### Pattern 3: Self-Organizing Swarm

Workers poll a shared task list and atomically claim unclaimed work. No central dispatcher needed — the task list is the coordination mechanism.

**When to use:** Large backlogs of independent tasks, processing queues, workloads of unknown size.

```python
TeamCreate(team_name="file-processor")

# Add many tasks to the shared list
for file in file_list:
    TaskCreate(subject=f"Process {file}", description=f"Extract and transform {file}")

# Spawn workers that self-direct
for i in range(4):
    Task(team_name="file-processor", model="sonnet",
         prompt="Poll TaskList for unclaimed tasks. Claim one, complete it, repeat until no tasks remain.")
```

Note: Use [safe-task-claim](https://github.com/Osso/safe-task-claim) to prevent race conditions when multiple workers claim the same task.

### Pattern 4: Competing Hypotheses

Multiple agents simultaneously test different approaches to the same problem. The lead evaluates results and picks the winner (or synthesizes the best elements from each).

**When to use:** Uncertain implementation approach, algorithm selection, architectural trade-offs.

```python
Task(model="sonnet", run_in_background=True,
     prompt="Implement the caching layer using Redis. Write full implementation + benchmarks.")

Task(model="sonnet", run_in_background=True,
     prompt="Implement the caching layer using in-memory LRU. Write full implementation + benchmarks.")

Task(model="sonnet", run_in_background=True,
     prompt="Implement the caching layer using a CDN-first approach. Write full implementation + benchmarks.")

# Lead compares benchmark results and chooses
```

### Pattern 5: Council / Debate

Agents independently propose approaches, then the lead (or an opus agent) synthesizes them into a consensus design. Avoids anchoring bias from a single proposal.

**When to use:** Architecture decisions, API design, major refactors.

```python
# Proposal phase — parallel
Task(model="sonnet", run_in_background=True,
     prompt="Propose a database schema for the orders system. Justify trade-offs.")
Task(model="sonnet", run_in_background=True,
     prompt="Propose a database schema for the orders system from a read-performance perspective.")
Task(model="sonnet", run_in_background=True,
     prompt="Propose a database schema for the orders system prioritizing write throughput.")

# Synthesis phase — single opus agent reviews all proposals
Task(model="opus", prompt="Here are three schema proposals: [A] [B] [C]. Synthesize the best design.")
```

### Pattern 6: Plan-First

Generate a detailed plan before spawning any implementation agents. Planning costs ~10K tokens; a swarm costs ~800K tokens. Catching a bad plan early is far cheaper than discarding work.

**When to use:** Any non-trivial feature; whenever scope is ambiguous.

```python
# Phase 1 — plan (cheap, ~10K tokens)
plan = Task(model="opus",
            prompt="Create a detailed step-by-step implementation plan for: [feature description]. "
                   "Output JSON: {steps: [{id, title, files_affected, dependencies}]}")

# Phase 2 — implement in parallel (expensive, ~800K tokens)
for step in plan.steps:
    Task(model="sonnet", run_in_background=True,
         isolation="worktree",
         prompt=f"Implement step {step.id}: {step.title}. Files: {step.files_affected}")
```

### Pattern 7: Adversarial Review Loop

One agent implements, another critiques. The implementer revises based on critique. Cycle until the critic approves or the loop limit is reached.

**When to use:** High-stakes code, security-critical paths, complex algorithms.

```python
revision = 0
approved = False

while not approved and revision < 3:
    Task(model="sonnet", prompt=f"Implement the payment processor. Previous critique: {critique}")
    critique = Task(model="sonnet", subagent_type="security-reviewer",
                    prompt="Review this implementation. Output: APPROVED or CHANGES_NEEDED + list of issues.")
    approved = "APPROVED" in critique
    revision += 1
```

### Pattern 8: Codebase-Mediated Coordination

Agents coordinate through shared code and documentation rather than direct messages. File ownership boundaries are defined in `CLAUDE.md`. Each agent owns specific directories; they read shared contracts (interfaces, schemas) but never touch each other's files.

**When to use:** Large parallel builds, 5+ agent teams, any scenario where peer messaging is error-prone.

```
# In CLAUDE.md (the shared contract every agent reads):
# Module Ownership:
# - Agent "api-builder": owns src/api/ only
# - Agent "frontend-dev": owns src/ui/ only
# - Agent "db-layer": owns src/db/ only
# - Shared interfaces in src/contracts/ — read by all, written by none
```

No `SendMessage` calls needed. Each agent reads `CLAUDE.md`, knows its boundaries, and coordinates through the git repository.

### Pattern 9: Two-Layer Agent Definitions

Separate the persistent HOW (agent identity, skills, tools — defined in `.claude/agents/*.md`) from the ephemeral WHAT (task-specific instructions passed at spawn time). This avoids duplicating configuration while enabling reuse across many different tasks.

**When to use:** Reusable specialist agents deployed across many sessions or projects.

```python
# .claude/agents/backend-engineer.md defines:
#   model: sonnet, tools: [Read, Write, Edit, Bash, Grep],
#   skills: [api-conventions, db-patterns], memory: project
#   System prompt: "You are a backend engineer specializing in Go..."

# At spawn time, pass only WHAT (task-specific)
Task(subagent_type="backend-engineer", run_in_background=True,
     prompt="Implement the order processing endpoint per the spec in docs/orders-api.md")

Task(subagent_type="backend-engineer", run_in_background=True,
     prompt="Refactor the payment service to use the new retry library")
```

---

## 8. Known Limitations & Workarounds

Last verified: 2026-02-23. Check linked issues for current status.

| Limitation | Issue | Workaround |
|-----------|-------|-----------|
| `model=` silently ignored for in-process teammates (all run at lead's model) | [#27038](https://github.com/anthropics/claude-code/issues/27038) | Fixed in v2.1.47 for `.claude/agents/` — update to latest |
| Custom agents lose their `tools` list when spawned with `team_name` | [#25608](https://github.com/anthropics/claude-code/issues/25608) | Use custom agents + parallel `Task()` calls instead of Teams |
| Agent frontmatter hooks don't fire in teams (session ID mismatch) | [#27655](https://github.com/anthropics/claude-code/issues/27655) | Use custom agents + parallel `Task()` calls instead of Teams |
| `allowed-tools:` (wrong field name) silently grants ALL tools | [#27099](https://github.com/anthropics/claude-code/issues/27099) | Use `tools:` (no `allowed-` prefix) |
| `/resume` does not restore teammates | [#26323](https://github.com/anthropics/claude-code/issues/26323) | No workaround; avoid using `/resume` in team sessions |
| Zombie teammates remain after `TeamDelete` | [#26955](https://github.com/anthropics/claude-code/issues/26955) | Kill processes manually; claude-code-teams-mcp has `force_kill_teammate` |
| Plan mode auto-approves before lead reviews | [#27265](https://github.com/anthropics/claude-code/issues/27265) | Do not use `plan_approval_response` until fixed |
| Bedrock: 400 errors from normalized model ID | [#25193](https://github.com/anthropics/claude-code/issues/25193) | Fixed in v2.1.43 — update |
| Vertex AI: agent teams fail to launch | [#27729](https://github.com/anthropics/claude-code/issues/27729) | No workaround yet; use standard API |
| `PreToolUse` hooks not inherited by subagents (security gap) | [#21460](https://github.com/anthropics/claude-code/issues/21460) | Define hooks in agent frontmatter; validate in agent's own system prompt |
| Auto memory race condition with concurrent teams | [#24130](https://github.com/anthropics/claude-code/issues/24130) | Use `memory: local` per agent; avoid shared memory in concurrent teams |
| `CLAUDE_CONFIG_DIR` not passed to teammate processes | [#24989](https://github.com/anthropics/claude-code/issues/24989) | Set in global environment; avoid relying on custom config paths in teams |
| `CLAUDE_PROJECT_DIR` empty in subagent environments | [#26429](https://github.com/anthropics/claude-code/issues/26429) | Pass project path explicitly in agent prompt |
| Deferred MCP tools unavailable in custom agents | [#25200](https://github.com/anthropics/claude-code/issues/25200) | Use non-deferred (eager) MCP tool configuration |
| `SubagentStart`/`SubagentStop` hooks have unreliable delivery | [#27153](https://github.com/anthropics/claude-code/issues/27153) | Do not rely on these hooks for critical logic |
| `--agent` key in `settings.json` causes `Task` tool to disappear in Teams | [#13533](https://github.com/anthropics/claude-code/issues/13533) | Rename key to `_agent` temporarily to disable |
| ~37 agent limit: silently stops loading after ~37 agents, no error | [#18993](https://github.com/anthropics/claude-code/issues/18993) | Keep total agent definitions below 37; consolidate with broader descriptions |

### Additional Known Gotchas

These are behavioral quirks that are not bugs per se but will trip you up:

- **Delegate Mode first in Agent Teams.** Press Shift+Tab immediately after spawning teammates. If you do not, the lead agent will claim implementation tasks from the shared list before teammates can.
- **File ownership is the #1 rule.** Two agents editing the same file will silently overwrite each other's changes. Define explicit directory boundaries per agent in `CLAUDE.md`.
- **Plan mode is irreversible per teammate session.** Once a teammate is in plan mode, it cannot switch to execution mode. Spawn a new teammate for implementation.
- **Agents spawn sequentially.** Each agent takes 6-7 seconds to initialize. A team of 5 agents takes ~35 seconds before all are active.
- **Teammates do not see each other's text output.** The only way to share information between teammates is via `SendMessage` or through files written to the shared codebase.
- **`bypassPermissions` for teammates** requires setting it in `~/.claude/settings.json` (global), not project-level settings ([#26479](https://github.com/anthropics/claude-code/issues/26479)).
- **Git worktree path access** requires adding `additionalDirectories: ["../worktree-prefix-*"]` to permissions ([#12748](https://github.com/anthropics/claude-code/issues/12748)).
- **User-level** custom agents (`~/.claude/agents/`) cannot be used as `subagent_type` in Task calls. **Project-level** agents (`.claude/agents/`) can be referenced by name. Plugin agents (e.g., `superpowers:code-reviewer`) also work. If your user-level agent isn't found, use `subagent_type="general-purpose"` and inline the system prompt.

---

## 9. Experimental Features

The following features work in current versions but are not documented in official Anthropic docs. They may change or be removed without notice.

### `skills:` in Agent Frontmatter

⚠️ Undocumented but confirmed working.

```yaml
skills:
  - api-conventions
  - db-patterns
```

Restricts which skills the agent sees. Unlisted skills are invisible to the agent. Useful for reducing context bloat — agents with focused skill sets perform better on specialized tasks.

### `color:` in Agent Frontmatter

⚠️ Undocumented but confirmed working.

```yaml
color: cyan   # Options: blue, cyan, green, yellow, magenta, red
```

Sets the agent's color indicator in the terminal UI. Purely cosmetic; no behavioral effect.

### `background: true`

⚠️ Added in v2.1.49. Not in official docs.

```yaml
background: true
```

The agent always runs as a background task regardless of how it is invoked. Useful for monitoring and observer agents.

### `isolation: worktree`

⚠️ Added in v2.1.49. Not in official docs.

```yaml
isolation: worktree
```

Runs the agent in an isolated git worktree. The worktree is created at agent start and cleaned up automatically when the agent finishes. Prevents agents from conflicting on shared files. Custom branch name not yet supported ([#27749](https://github.com/anthropics/claude-code/issues/27749)).

### `use-when:` in Skill Frontmatter

⚠️ **Does not work.** Despite appearing in some community examples, the `use-when:` field in skill frontmatter is not functional ([#27569](https://github.com/anthropics/claude-code/issues/27569)). Use the `description:` field instead to control when a skill is triggered.

### Working Hooks in Agent Frontmatter

⚠️ Only 6 of the 16 hook event types fire correctly when defined in agent frontmatter `hooks:`. The 6 that work:

- `PreToolUse`
- `PostToolUse`
- `PermissionRequest`
- `PostToolUseFailure`
- `Stop`
- `SubagentStop`

The remaining 10 hook types (e.g., `TeammateIdle`, `TaskCompleted`, `SubagentStart`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`, and others) must be defined in `settings.json` at the project or user level and do not fire from agent frontmatter.

### ~37 Agent Limit

⚠️ Undocumented hard limit.

Claude Code silently stops loading agent definitions after approximately 37 agents. No error is raised. If you have a large agent collection and some agents seem to go missing, this is likely the cause ([#18993](https://github.com/anthropics/claude-code/issues/18993)).

### Skill-Agent Binding

⚠️ Undocumented combination.

```yaml
# In a skill's SKILL.md frontmatter:
---
context: fork
agent: my-specialist-agent
---
```

Binds a skill invocation to a specific agent. `agent:` without `context: fork` does not work. `context: fork` without `agent:` creates a fork without agent binding.

### Dynamic Context Injection via SubagentStart Hook

⚠️ Undocumented but functional in some versions.

A `SubagentStart` hook can read the `agent_type` field from the hook input, look up role-specific configuration, and return `hookSpecificOutput.additionalContext` to inject role-specific context before the agent begins. For teammates, the `name` parameter becomes `agent_type` when no matching agent definition exists.

### Hidden Environment Variables

Beyond the 6 documented team environment variables, these additional vars are available:

```
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1   # Disable all background tasks globally
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50       # Trigger context compaction earlier (default ~80%)
```

### 13 TeammateTool Operations

The internal `TeammateTool` API has 13 operations, more than the 7 documented tools expose:

```
spawnTeam, discoverTeams, requestJoin, approveJoin, rejectJoin,
write, broadcast, requestShutdown, approveShutdown, rejectShutdown,
approvePlan, rejectPlan, cleanup
```

These are accessible via [claude-code-teams-mcp](https://github.com/cs50victor/claude-code-teams-mcp), which reimplements the full protocol.

### Undocumented CLI Flags for Teammate Spawning

The exact command Claude Code uses internally to spawn a teammate (discovered via binary analysis):

```bash
CLAUDECODE=1 CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 \
claude \
  --agent-id "worker-1@my-team" \
  --agent-name "worker-1" \
  --team-name "my-team" \
  --agent-color "blue" \
  --parent-session-id "..." \
  --agent-type "general-purpose" \
  --model "sonnet"
  # Optional flags:
  # --plan-mode-required
  # --dangerously-skip-permissions
```

---

## 10. Cost Optimization

### Model Distribution

| Model | Cost | Use For | Never For |
|-------|------|---------|-----------|
| **opus** | $$$ | Architecture decisions, synthesis of competing results, critical trade-offs | Bulk research, code review, data gathering, formatting |
| **sonnet** | $$ | Research, implementation, writing, editing, **code review** | Final architecture decisions |
| **haiku** | $ | Trivial lookups, formatting, line counts, simple grep-and-report | Complex reasoning, security analysis, nuanced decisions |

**Decision rule:** Architecture / synthesis → opus. Volume / review → sonnet. Simple mechanical tasks → haiku.

**On code review specifically:** Sonnet is approximately 4-6x cheaper than Opus at list pricing. For structured review tasks like code review, sonnet with a focused skill produces comparable or better results — particularly for security issue detection where a narrower focus reduces false negatives. Use sonnet for all code review tasks.

### Practical Tips

- **`write()` over `broadcast()` in Teams.** `broadcast` delivers to N teammates and costs N times as much. Use targeted `SendMessage` with `type: message` unless you genuinely need all agents to receive the information.
- **Plan before spawning.** A planning phase costs ~10K tokens. A spawned swarm costs ~800K tokens. Catching a design flaw in planning saves the entire swarm cost.
- **Pre-approve tools.** Add expected tool patterns to `permissions.allow` in settings before starting a team. This prevents permission approval storms that stall every agent.
- **Trigger compaction early for subagents.** Set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` to compact context at 50% instead of the default ~80%. Prevents context overflow in long-running agents.
- **Cap team size.** Beyond 4-5 parallel agents, returns diminish and coordination overhead increases. Prefer 3-5 focused agents over 8+ generalist agents.
- **Use `isolation: worktree` selectively.** Worktree isolation prevents file conflicts but adds overhead. Use it only when agents edit overlapping files.

### Benchmark Numbers

Source: alexop.dev, 2026:
- Solo agent: ~200K tokens per complex task
- 3 subagents (parallel Tasks): ~440K tokens total
- 3-person Agent Team: ~800K tokens total

Source: Daytona benchmark, 2026:
- Agents in sandboxed environments use ~40% of the context window vs ~80-90% for a solo agent handling the same task (due to cleaner context, no accumulated history).

The implication: Agent Teams cost approximately 3-7x more than parallel Task calls for the same work. Use Teams only when peer communication genuinely provides value that offsets this cost.

---

## 11. Case Studies

### Anthropic C Compiler (Official)

Anthropic's engineering team built a C compiler using 16 agents running 2,000 sessions over two weeks at a total cost of approximately $20,000, producing ~100,000 lines of code.

The architecture used a bare git repository as the coordination point, with each agent cloning the repository into `/workspace`. A `current_tasks/` directory served as a file-lock mechanism, preventing agents from duplicating work. There was no central orchestrator — agents simply identified "the next obvious problem" from the test output and worked on it. The team discovered that the quality of the test harness mattered more than the agents themselves: "the task verifier must be nearly perfect." A `--fast` flag randomly sampled 1-10% of tests during development to reduce feedback latency.

**Lessons:**
- Test harness quality is the ceiling for agent quality — invest heavily in it
- File-based coordination (a directory as a lock) works well at scale without complex messaging
- No central orchestrator can work; agents can self-direct given clear criteria
- Random test sampling during development cuts wait time significantly

### 24-Agent Local Hardware (Community)

A community engineer ran 24 simultaneous Claude Code agents on local hardware, using a Rust/Tokio orchestrator and a local Mistral 7B model on an RTX 4070. Coordination happened entirely through the shared codebase — no direct agent messaging. `CLAUDE.md` defined module ownership boundaries. The result: 683 tests, zero failures, a 1.53:1 test-to-production-code ratio, enforced by pre-commit hooks that blocked any commit not meeting the ratio threshold.

**Lessons:**
- Codebase-mediated coordination scales to 24 agents without peer messaging
- Pre-commit hooks are an effective quality gate that agents cannot bypass
- Local models are viable for subagent workloads when latency is less critical
- File ownership boundaries defined in `CLAUDE.md` prevent silent overwrites at scale

### E-commerce Agent Teams Failure (Community)

A developer ran a 5-person Agent Team for two days on an e-commerce project. The lead agent "forgot to coordinate half the tasks" — a consequence of context filling up and the coordination state being lost. The result was 100,000+ tokens consumed with an incomplete, inconsistent codebase.

**Lessons:**
- Context independence is a strength for divergent work but a weakness for convergent work requiring continuous coordination
- Plan-First pattern (Pattern 6) would have caught the coordination gap early
- For convergent tasks (building one coherent system), codebase-mediated coordination (Pattern 8) is more reliable than relying on agent memory
- Agent Teams' lack of a persistent shared state makes them unsuitable for long-running projects without an external coordination mechanism

---

## 12. Ecosystem

Top projects by usefulness and community adoption:

| Project | Stars | Description |
|---------|-------|-------------|
| [steveyegge/gastown](https://github.com/steveyegge/gastown) | 10k | Multi-agent workspace manager with Beads issue tracking integration |
| [Yeachan-Heo/oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) | 7k | 32 agents, 40+ skills, smart model routing, Autopilot and Ultrapilot modes |
| [EveryInc/compound-engineering-plugin](https://github.com/EveryInc/compound-engineering-plugin) | 9.4k | Official plugin with `/work` command and pre-built team workflows |
| [danielmiessler/Personal_AI_Infrastructure](https://github.com/danielmiessler/Personal_AI_Infrastructure) | 9k | PAI v3.0 — Algorithm/Engineer subagent architecture |
| [cs50victor/claude-code-teams-mcp](https://github.com/cs50victor/claude-code-teams-mcp) | 181 | Standalone MCP reimplementation of the Agent Teams protocol; enables mixed Claude + OpenCode teams |
| [modu-ai/moai-adk](https://github.com/modu-ai/moai-adk) | 750 | Agent Development Kit with typed team roles (backend-dev, frontend-dev, tester) |
| [alinaqi/claude-bootstrap](https://github.com/alinaqi/claude-bootstrap) | 508 | Scaffolds agent teams by default in every new project |
| [jayminwest/overstory](https://github.com/jayminwest/overstory) | — | Coordinator → Supervisor → Workers hierarchy with SQLite-based message passing and 4-tier conflict resolution |
| [Ruya-AI/cozempic](https://github.com/Ruya-AI/cozempic) | 98 | Context pruning that protects team coordination state |
| [Osso/safe-task-claim](https://github.com/Osso/safe-task-claim) | — | Atomic task claiming MCP server — prevents race conditions in swarm patterns |

See [docs/ecosystem.md](docs/ecosystem.md) for the full catalog of 30+ projects, guides, and resources.

---

## 13. Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## 14. Sources

### Official

- [Anthropic — Agent Teams docs](https://code.claude.com/docs/en/agent-teams)
- [Anthropic — Sub-Agents docs](https://code.claude.com/docs/en/sub-agents)
- [Anthropic — Hooks reference](https://code.claude.com/docs/en/hooks)
- [Anthropic — CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Anthropic Engineering — Building a C Compiler with 16 agents](https://www.anthropic.com/engineering/building-c-compiler)
- [Anthropic Blog — Agent Skills announcement](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills)

### Community Guides

- [claudefa.st — Complete Agent Teams Guide](https://claudefa.st/blog/guide/agents/agent-teams)
- [claudefa.st — Agent Teams Controls](https://claudefa.st/blog/guide/agents/agent-teams-controls)
- [claudefa.st — Agent Teams Best Practices](https://claudefa.st/blog/guide/agents/agent-teams-best-practices)
- [alexop.dev — From Tasks to Swarms](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/)
- [paddo.dev — The Hidden Swarm](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [paddo.dev — The Switch Got Flipped](https://paddo.dev/blog/agent-teams-the-switch-got-flipped/)
- [claudecodecamp.com — Under the Hood](https://www.claudecodecamp.com/p/claude-code-agent-teams-how-they-work-under-the-hood)
- [addyosmani.com — Agent Teams Overview](https://addyosmani.com/blog/claude-code-agent-teams/)
- [scottspence.com — Enable Team Mode](https://scottspence.com/posts/enable-team-mode-in-claude-code)
- [scottspence.com — Claude Code Swarm with Daytona Sandboxes](https://scottspence.com/posts/claude-code-swarm-daytona-sandboxes)
- [shipyard.build — Multi-Agent Orchestration 2026](https://shipyard.build/blog/claude-code-multi-agent/)
- [incident.io — Shipping Faster with Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees)
- [zerofuturetech — Two Days with Agent Teams (failure report)](https://zerofuturetech.substack.com/p/i-spent-two-days-with-claude-agent)

### Key Gists

- [kieranklaassen — Swarm Orchestration (13 TeammateTool ops, 6 patterns)](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea)
- [kieranklaassen — Extended Multi-Agent System Reference](https://gist.github.com/kieranklaassen/d2b35569be2c7f1412c64861a219d51f)
- [cs50victor — Claude Code Internals (binary deep dive)](https://gist.github.com/cs50victor/0a7081e6824c135b4bdc28b566e1c719)
- [ruvnet — Claude Flow vs TeammateTool (92% overlap comparison)](https://gist.github.com/ruvnet/18dc8d060194017b989d1f8993919ee4)

### Hacker News Threads

- [Hidden Feature: Agent Swarms](https://news.ycombinator.com/item?id=46743908)
- [Orchestrating Claude Code Teams](https://news.ycombinator.com/item?id=46902368)
- [Shared Local Memory for Agent Teams](https://news.ycombinator.com/item?id=46913360)
- [24 Simultaneous Agents](https://news.ycombinator.com/item?id=47099597)
- [Amux tmux Multiplexer for Agents](https://news.ycombinator.com/item?id=47104424)

---

## 15. Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.
