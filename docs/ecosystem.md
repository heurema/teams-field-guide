# Ecosystem Catalog

Full catalog of projects, tools, guides, and resources for Claude Code multi-agent development. Last updated: 2026-02-24.

---

## Protocol & Infrastructure

Projects that implement or extend the core agent coordination protocol.

| Project | Stars | Description |
|---------|-------|-------------|
| [cs50victor/claude-code-teams-mcp](https://github.com/cs50victor/claude-code-teams-mcp) | 181 | Standalone MCP server reimplementing Claude Code's internal Agent Teams protocol. Enables mixed Claude + OpenCode teams, adds 12 tools (vs 7 built-in), and provides workarounds for zombie teammates and `model=` bugs. |
| [jayminwest/overstory](https://github.com/jayminwest/overstory) | — | Multi-tier orchestration: Coordinator → Supervisor → Workers hierarchy with SQLite-based message passing and 4-tier conflict resolution. |
| [Osso/safe-task-claim](https://github.com/Osso/safe-task-claim) | — | Atomic task claiming MCP server. Prevents race conditions when multiple swarm workers attempt to claim the same task simultaneously. |
| [MaorBril/clauder](https://github.com/MaorBril/clauder) | — | Cross-agent bridge using MCP + SQLite. Enables coordination across Claude, Cursor, Windsurf, Codex, and Gemini agents. |
| [SukinShetty/Nemp-memory](https://github.com/SukinShetty/Nemp-memory) | — | Shared local JSON memory store for agent teams. Provides persistent key-value state accessible by all agents in a session. |
| [yuvalsuede/claude-teams-language-protocol](https://github.com/yuvalsuede/claude-teams-language-protocol) | 11 | AgentSpeak v2 protocol for inter-agent messaging — claims 60-70% token reduction vs raw natural language messages. |

---

## Frameworks & Meta-tools

Full orchestration frameworks and developer experience tools.

| Project | Stars | Description |
|---------|-------|-------------|
| [steveyegge/gastown](https://github.com/steveyegge/gastown) | 10k | Multi-agent workspace manager. Handles agent lifecycle, context switching, and integrates with Beads for issue tracking. The most widely adopted community framework. |
| [Yeachan-Heo/oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) | 7k | Comprehensive agent collection: 32 pre-built agents, 40+ skills, smart model routing, and Autopilot/Ultrapilot modes for hands-off operation. |
| [EveryInc/compound-engineering-plugin](https://github.com/EveryInc/compound-engineering-plugin) | 9.4k | Official community plugin. `/work` command with pre-defined team workflows. Good starting point for understanding plugin-based team patterns. |
| [danielmiessler/Personal_AI_Infrastructure](https://github.com/danielmiessler/Personal_AI_Infrastructure) | 9k | PAI v3.0 — Personal AI Infrastructure. Algorithm/Engineer subagent architecture with detailed role definitions. |
| [ruvnet/claude-flow](https://github.com/ruvnet/claude-flow) | — | Enterprise orchestration framework with RAG integration, MCP support, and 28 integration points. |
| [modu-ai/moai-adk](https://github.com/modu-ai/moai-adk) | 750 | Agent Development Kit with typed team roles: `backend-dev`, `frontend-dev`, `tester`. Enforces structure through typed interfaces. |
| [alinaqi/claude-bootstrap](https://github.com/alinaqi/claude-bootstrap) | 508 | Project scaffolder that initializes agent teams by default in every new project. Opinionated setup for teams-first development. |
| [shintaro-sprech/agent-orchestrator-template](https://github.com/shintaro-sprech/agent-orchestrator-template) | 117 | Self-evolving subagent system. Orchestrator spawns specialists based on task analysis; specialists can spawn further subagents. |
| [serejaris/ris-claude-code](https://github.com/serejaris/ris-claude-code) | — | Russian-language collection of Claude Code patterns including an agent teams skill. |

---

## Monitoring & Visualization

Tools for observing and managing running agent teams.

| Project | Stars | Description |
|---------|-------|-------------|
| [Glsme/agent-monitor](https://github.com/Glsme/agent-monitor) | 45 | Tauri + React desktop application. Displays agents as pixel art characters in an office view. Shows agent status, current task, and output in real time. |
| [mixpeek/amux](https://github.com/mixpeek/amux) | — | Single Python file. Provides a PWA dashboard, SSE-based real-time monitoring, and a REST API for agent management. Works with any terminal multiplexer. |
| [atani/teamwatch](https://github.com/atani/teamwatch) | — | Terminal-native monitor for Kitty and Zellij. Displays agent status panels within the multiplexer layout. |

---

## Multi-Model / Cross-Agent

Projects enabling coordination across different AI models and providers.

| Project | Stars | Description |
|---------|-------|-------------|
| [Pickle-Pixel/HydraTeams](https://github.com/Pickle-Pixel/HydraTeams) | 29 | Proxy layer translating Claude Teams protocol to GPT, Gemini, and Ollama. Enables mixed-model teams without changing orchestration code. |
| [7836246/claude-team-mcp](https://github.com/7836246/claude-team-mcp) | 44 | Claude + GPT + Gemini as a unified development team via MCP. Each model plays a defined role (architect, implementer, reviewer). |
| [johannesjo/parallel-code](https://github.com/johannesjo/parallel-code) | — | Runs Claude, Codex, and Gemini side by side in isolated git worktrees. Useful for comparing model outputs on identical tasks. |
| [Real-AI-Engineering/codex-partner](https://github.com/Real-AI-Engineering/codex-partner) | — | Patterns for multi-model teams combining Claude Code with Codex CLI. |

---

## Agent Collections

Pre-built agents for common engineering tasks.

| Project | Stars | Description |
|---------|-------|-------------|
| [wshobson/agents](https://github.com/wshobson/agents) | — | 112 agents, 72 plugins, 146 skills, 7 pre-configured team presets covering common development workflows. |
| [VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) | — | 127+ subagent definitions organized into 10 categories. Community-curated with usage examples. |
| [hesreallyhim/a-list-of-claude-code-agents](https://github.com/hesreallyhim/a-list-of-claude-code-agents) | 1.2k | Community-maintained list of agent definitions. Regularly updated with new contributions. |

---

## Context & Cost Management

Tools for managing context size and token costs in agent workflows.

| Project | Stars | Description |
|---------|-------|-------------|
| [Ruya-AI/cozempic](https://github.com/Ruya-AI/cozempic) | 98 | Context pruning that protects team coordination state. Compresses conversation history while preserving agent role definitions and task assignments. |

---

## Key Gists

Reference gists that document Claude Code internals and patterns.

| Gist | Author | Description |
|------|--------|-------------|
| [Swarm Orchestration Patterns](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea) | kieranklaassen | 13 TeammateTool operations documented with 6 coordination patterns. |
| [Extended Multi-Agent Reference](https://gist.github.com/kieranklaassen/d2b35569be2c7f1412c64861a219d51f) | kieranklaassen | Comprehensive multi-agent system reference with advanced patterns. |
| [Claude Code Internals (binary deep dive)](https://gist.github.com/cs50victor/0a7081e6824c135b4bdc28b566e1c719) | cs50victor | Reverse-engineered internal protocol. Source for claude-code-teams-mcp. Documents undocumented CLI flags and data models. |
| [Claude Flow vs TeammateTool](https://gist.github.com/ruvnet/18dc8d060194017b989d1f8993919ee4) | ruvnet | Analysis showing 92% overlap between Claude Flow and the built-in TeammateTool API. |

---

## Official Documentation

| Resource | Description |
|----------|-------------|
| [Agent Teams](https://code.claude.com/docs/en/agent-teams) | Official reference for the experimental Agent Teams system |
| [Sub-Agents](https://code.claude.com/docs/en/sub-agents) | Official reference for custom agents and Task-based subagents |
| [Hooks](https://code.claude.com/docs/en/hooks) | Lifecycle hook documentation including team-specific hooks |
| [CLI Reference](https://code.claude.com/docs/en/cli-reference) | Full CLI flag documentation |
| [Building a C Compiler — Engineering Blog](https://www.anthropic.com/engineering/building-c-compiler) | Anthropic's case study: 16 agents, $20K, 100K lines |
| [Agent Skills Announcement](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills) | Blog post introducing the 3-tier progressive disclosure skills model |

---

## Community Guides & Blogs

| Resource | Description |
|----------|-------------|
| [claudefa.st — Complete Agent Teams Guide](https://claudefa.st/blog/guide/agents/agent-teams) | Comprehensive tutorial covering setup, configuration, and usage |
| [claudefa.st — Agent Teams Controls](https://claudefa.st/blog/guide/agents/agent-teams-controls) | Keyboard shortcuts, display modes, and team management UI |
| [claudefa.st — Agent Teams Best Practices](https://claudefa.st/blog/guide/agents/agent-teams-best-practices) | Distilled operational best practices from community experience |
| [alexop.dev — From Tasks to Swarms](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/) | Progressive guide: solo → tasks → swarms. Token benchmarks included. |
| [paddo.dev — The Hidden Swarm](https://paddo.dev/blog/claude-code-hidden-swarm/) | Early discovery of the experimental teams feature |
| [paddo.dev — The Switch Got Flipped](https://paddo.dev/blog/agent-teams-the-switch-got-flipped/) | Follow-up when teams became more accessible |
| [claudecodecamp.com — Under the Hood](https://www.claudecodecamp.com/p/claude-code-agent-teams-how-they-work-under-the-hood) | Technical breakdown of internal architecture |
| [addyosmani.com — Agent Teams Overview](https://addyosmani.com/blog/claude-code-agent-teams/) | Overview from a web engineering perspective |
| [scottspence.com — Enable Team Mode](https://scottspence.com/posts/enable-team-mode-in-claude-code) | Quick-start guide for enabling and configuring teams |
| [scottspence.com — Swarm with Daytona Sandboxes](https://scottspence.com/posts/claude-code-swarm-daytona-sandboxes) | Running agent swarms in Daytona cloud sandboxes with benchmarks |
| [shipyard.build — Multi-Agent Orchestration 2026](https://shipyard.build/blog/claude-code-multi-agent/) | Current-state review of Claude Code multi-agent capabilities |
| [incident.io — Shipping Faster with Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees) | Production experience using `isolation: worktree` |
| [zerofuturetech — Two Days with Agent Teams (failure report)](https://zerofuturetech.substack.com/p/i-spent-two-days-with-claude-agent) | Honest post-mortem on a failed 5-agent e-commerce project |

---

## Hacker News Threads

| Thread | Description |
|--------|-------------|
| [Hidden Feature: Agent Swarms](https://news.ycombinator.com/item?id=46743908) | Original discovery thread with early experiments |
| [Orchestrating Claude Code Teams](https://news.ycombinator.com/item?id=46902368) | Discussion of orchestration patterns and limitations |
| [Shared Local Memory for Agent Teams](https://news.ycombinator.com/item?id=46913360) | Community solutions for shared agent state |
| [24 Simultaneous Agents](https://news.ycombinator.com/item?id=47099597) | The 24-agent local hardware experiment |
| [Amux tmux Multiplexer for Agents](https://news.ycombinator.com/item?id=47104424) | Discussion of agent monitoring tools |
