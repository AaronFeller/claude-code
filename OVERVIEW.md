# Claude Code — Big Picture Overview

> A deep-read summary of the `src/` directory from the leaked Anthropic Claude Code CLI (2026-03-31).

---

## What Is This?

This is the **full TypeScript source code of Claude Code** — Anthropic's official AI-powered developer CLI. It lets you talk to Claude directly in your terminal to perform real software engineering work: editing files, running shell commands, searching codebases, managing git, and much more.

The leaked source (~1,900 files, 512K+ lines of TypeScript) reveals a **mature, production-grade system** with far more sophistication than its "terminal chatbot" surface suggests.

---

## Directory at a Glance

```
claude-code/
├── README.md          # High-level overview of the leak and architecture
├── OVERVIEW.md        # This file — big picture takeaway
└── src/               # Entire TypeScript source
```

---

## Big Picture Takeaway

### It's an Agent Runtime, Not Just a Chatbot

The headline feature is the **Tool System** (`src/tools/`, ~43 tools). Every capability Claude Code has — reading files, writing code, running bash commands, searching the web, even spawning _other_ AI agents — is implemented as a discrete, permission-checked tool. The AI picks which tools to call, the runtime executes them, and the results flow back into the conversation. This is the full agentic loop.

Key tools that reveal the ambition of the system:

| Tool | What It Tells You |
|---|---|
| `AgentTool` | Claude can spin up sub-agents to work in parallel |
| `TeamCreateTool` | Groups of agents can be managed as a "team" |
| `EnterWorktreeTool` | Agents can isolate work in separate git worktrees |
| `CronCreateTool` | Agents can schedule themselves to run later |
| `RemoteTriggerTool` | Agents can be triggered remotely |
| `MCPTool` | Connects to external tool servers via MCP protocol |
| `LSPTool` | Speaks Language Server Protocol for real IDE-level code intelligence |
| `SkillTool` | Runs user-defined reusable workflows |

### The Query Engine Is the Heart

`QueryEngine.ts` (~46K lines) is where all the magic happens. It manages:
- The streaming request/response loop with the Anthropic API
- Tool-call dispatch and result injection
- Claude's "extended thinking" mode
- Cost and token tracking on every call
- Retry logic and error categorization
- Caching for repeated calls

This one file defines the personality of the entire system — it is the agent loop.

### A Serious Permission System

Before any tool executes, it passes through a multi-level permission gate (`src/hooks/toolPermission/`). Modes include `default`, `plan` (read-only preview), `bypassPermissions` (trust everything), and `auto`. Every destructive action — writing a file, running bash, calling a remote service — requires either explicit user approval or a configured trust level. This is a thoughtful production security model, not an afterthought.

### The Bridge: IDE Integration is a First-Class Concern

`src/bridge/` (~30+ files) is a full bidirectional communication layer for embedding Claude Code inside **VS Code and JetBrains IDEs**. It handles:
- IPC message protocols
- JWT-based authentication between the IDE extension and CLI process
- Permission callbacks routed through the IDE UI
- Session lifecycle management from a remote host

This is not a demo integration — it's a production-grade IPC system.

### Context Assembly Is Deeply Intentional

`src/context.ts` builds the system prompt injected into every Claude API call. It pulls in:
- Git repository state and diff
- `CLAUDE.md` memory files (hierarchical, from project root up to home directory)
- Skill context and custom instructions
- MDM (Mobile Device Management) policy overrides for enterprise deployments

The system has a persistent memory model where users and projects accumulate context over time through a structured memory file hierarchy. The AI isn't stateless between sessions — it remembers.

### Multi-Agent Orchestration Is Built In

`src/coordinator/` handles orchestrating multiple Claude agents working concurrently. `src/tasks/` manages a task graph. `AgentTool`, `TeamCreateTool`, `SendMessageTool`, and `TaskCreateTool` all compose into a system where a top-level agent can delegate, track, and synthesize parallel workstreams — an agent swarm.

### 100+ User-Facing Commands

`src/commands/` contains over a hundred slash-commands (`/commit`, `/review`, `/mcp`, `/skills`, `/tasks`, `/doctor`, etc.). This isn't a thin wrapper — it's a fully realized developer workflow tool with dedicated commands for git operations, PR reviews, bug hunting, security audits, session sharing, desktop/mobile app handoffs, and even Slack and GitHub App integrations.

### Performance Is Obsessed Over

The startup sequence in `main.tsx` fires parallel prefetches for MDM settings, keychain reads, and API preconnect _as import side-effects_, before any heavy module evaluation. OpenTelemetry (~400KB) and gRPC (~700KB) are lazy-loaded only when needed. Bun's `bun:bundle` feature flags enable dead-code elimination to strip entire subsystems at build time. Startup latency is clearly a product priority.

### Feature Flags Gate Entire Subsystems

Named flags (`PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `COORDINATOR_MODE`, `MONITOR_TOOL`) completely gate major subsystems. This reveals a product roadmap through its flag names — some features are staged or internal-only. `KAIROS` in particular appears to be an alternative advanced assistant mode not yet broadly released.

---

## Subsystem Map

| Subsystem | Location | What It Does |
|---|---|---|
| Agent Loop | `QueryEngine.ts` | Core LLM ↔ tool execution cycle |
| Tools | `src/tools/` | All 43 executable agent capabilities |
| Commands | `src/commands/` | 100+ user-facing slash commands |
| UI | `src/components/`, `src/screens/` | React + Ink terminal UI (~140 components) |
| Hooks | `src/hooks/` | 40+ React hooks (permissions, suggestions, state) |
| Services | `src/services/` | API, MCP, OAuth, LSP, analytics, compaction |
| State | `src/state/` | Centralized immutable app state store |
| Bridge | `src/bridge/` | IDE ↔ CLI IPC layer |
| Coordinator | `src/coordinator/` | Multi-agent orchestration |
| Context | `src/context.ts` | System prompt assembly (git, memory, settings) |
| Memory | `src/memdir/` | Persistent memory directory management |
| Skills | `src/skills/` | Reusable user-defined workflows |
| Tasks | `src/tasks/` | Task graph and management |
| Plugins | `src/plugins/` | Plugin loading system |
| Schemas | `src/schemas/` | Zod config validation schemas |
| Vim | `src/vim/` | Full vim motion/operator/text-object support |
| Voice | `src/voice/` | Voice input mode |
| Buddy | `src/buddy/` | Companion sprite Easter egg |
| Keybindings | `src/keybindings/` | Configurable keybinding system |
| Server | `src/server/` | Server mode for remote sessions |
| Remote | `src/remote/` | Remote session management |
| CLI | `src/cli/` | Structured I/O, transports, NDJSON serialization |
| Entrypoints | `src/entrypoints/` | Init, bootstrap, startup orchestration |
| Utils | `src/utils/` | Shared utility functions |
| Types | `src/types/` | Global TypeScript type definitions |

---

## Tech Stack Summary

| Layer | Technology |
|---|---|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript (strict) |
| Terminal UI | React + [Ink](https://github.com/vadimdemedes/ink) |
| CLI Parsing | [Commander.js](https://github.com/tj/commander.js) |
| Schema Validation | [Zod v4](https://zod.dev) |
| AI API | [Anthropic SDK](https://docs.anthropic.com) |
| Tool Protocol | [MCP SDK](https://modelcontextprotocol.io) |
| Code Intelligence | LSP (Language Server Protocol) |
| Code Search | ripgrep |
| Telemetry | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| Auth | OAuth 2.0, JWT, macOS Keychain |
| Linting/Formatting | Biome |

---

## Most Interesting Things Found

1. **Speculative execution**: The UI can preview tool outputs _while the user is still typing_ their next message — concurrent execution against user intent before confirmation.

2. **Git worktree isolation**: Agents can create isolated git worktrees to work in sandboxed branches, then merge results back.

3. **Cron scheduling**: `CronCreateTool` lets agents schedule themselves to run at a future time — proactive, autonomous agent execution without a human prompt.

4. **Buddy companion sprite**: `src/buddy/` contains a fully implemented interactive companion character with personality, sprites, and a notification system. An Easter egg living inside a production developer tool.

5. **KAIROS mode**: A feature-flagged advanced assistant mode that appears distinct from the standard REPL experience — likely an internal experiment or staged rollout.

6. **MDM support**: Enterprise Mobile Device Management policy enforcement is wired into the startup sequence and context assembly. Anthropic is clearly selling to enterprise.

7. **Team memory sync**: `src/services/teamMemorySync/` — shared memory state across a team of users. Collaborative AI context.

---

## One-Sentence Summary

Claude Code is a **permission-gated, multi-agent, context-aware developer runtime** built on React/Ink for terminal UIs, with a 46,000-line core loop, 43 tools, 100+ commands, full IDE bridge integration, persistent memory, and an architecture designed to run fleets of AI agents autonomously against real codebases.
