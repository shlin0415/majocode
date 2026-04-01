# Compare Code Agents: Claude Code, Codex, OpenCode, KiloCode, iFlow

> Deep analysis based on third_party source code inspection and web research.
> Date: 2026-04-01

---

## 1. Executive Summary

| Agent | Creator | Language | License | Model Lock-in | Architecture | Key Differentiator |
|-------|---------|----------|---------|---------------|-------------|-------------------|
| Claude Code | Anthropic | TypeScript/Python | Proprietary | Claude only | Single-threaded agent loop | Best reasoning, minimal surface |
| Codex CLI | OpenAI | Rust | Apache-2.0 | GPT-Codex only | Rust CLI + cloud sandbox | GitHub/ChatGPT ecosystem |
| OpenCode | anomalyco | TypeScript (Bun) + Go TUI | MIT | 75+ providers | Client-server | Model-agnostic, LSP integration |
| KiloCode | Kilo-Org | TypeScript (Bun) + Go TUI | MIT | 500+ models via gateway | Client-server (OpenCode fork) | VS Code extension, Agent Manager |
| iFlow CLI | iFlow AI | Node.js | - | OpenAI-compatible | Node.js CLI | Free models, Open Market |

---

## 2. Architecture Deep Dive

### 2.1 Claude Code (from exposed source port)

**Core Architecture:**
- Single main thread with flat message history
- Classic while-loop agent: model generates tool calls → execute → append results → repeat
- ~15 built-in tools (file ops, search, execution, web, code intelligence)
- CLAUDE.md files for persistent project context
- Subagent support for parallel isolated tasks (Agent Teams: 2-16 agents)

**Key Design Decisions:**
- Debuggability over complexity — no multi-agent orchestration by default
- Exact-match edits (not line numbers) for reliability
- Conservative permission model — read-only until user approves
- Auto-memory for cross-session learnings

**Source Code Structure (Python port):**
```
src/
├── main.py          # CLI entrypoint
├── models.py        # Dataclasses for subsystems
├── commands.py      # Command metadata
├── tools.py         # Tool metadata
├── query_engine.py  # Summary rendering
└── task.py          # Task representation
```

### 2.2 Codex CLI (Rust)

**Core Architecture:**
- Rust-based CLI with 80+ crates in `codex-rs/`
- Core crate: `codex-core` (largest, contains agent loop, tool execution, sandboxing)
- TUI crate: `codex-tui` (ratatui-based terminal UI)
- App-server protocol for client-server communication
- Sandbox support: Linux (Landlock), macOS (Seatbelt), Windows

**Key Design Decisions:**
- Safety-first: sandbox by default, approval policies
- AGENTS.md files with cascading discovery
- `apply_patch` tool for atomic file edits
- Hierarchical agent system (built-in + external agents)
- MCP server integration for extensibility
- OpenAI Responses API (not Chat Completions)

**Source Code Structure:**
```
codex-rs/
├── core/            # Agent loop, tools, sandboxing, config
│   └── src/
│       ├── codex.rs         # Main orchestrator (7602 lines)
│       ├── tools/           # Tool registry, handlers, routing
│       ├── agent/           # Agent registry, control, mailbox
│       ├── config/          # Config loading
│       └── sandboxing/      # Security sandbox
├── tui/             # Terminal UI (ratatui)
├── cli/             # CLI entrypoint
├── app-server/      # Client-server protocol
└── ...
```

### 2.3 OpenCode

**Core Architecture:**
- Client-server architecture: Go TUI (Bubble Tea) + Bun/JS HTTP server (Hono)
- Effect.ts for service composition and dependency injection
- Drizzle ORM with SQLite for session/message persistence
- Tool system with `Tool.define()` pattern and Zod schemas
- Agent namespace with configurable permissions

**Key Design Decisions:**
- Model-agnostic: 75+ providers via Models.dev
- LSP integration: spawns language servers, feeds diagnostics to LLM
- Git-based snapshots before changes, `/undo` and `/redo`
- Four built-in agents: build (full access), plan (read-only), general (subagent), explore (subagent)
- Plugin system for extensibility
- AI SDK (Vercel) for provider abstraction

**Source Code Structure:**
```
packages/opencode/src/
├── agent/           # Agent definitions, prompts
│   ├── agent.ts     # Agent namespace with 7 built-in agents
│   └── prompt/      # System prompts (compaction, explore, summary, title)
├── tool/            # Tool implementations
│   ├── tool.ts      # Tool.define() interface
│   ├── registry.ts  # Tool registration and filtering
│   ├── bash.ts      # Shell execution
│   ├── edit.ts      # File editing
│   ├── read.ts      # File reading
│   ├── write.ts     # File writing
│   └── ...          # 20+ tool implementations
├── session/         # Session management
│   ├── index.ts     # Session CRUD, forking, sharing
│   ├── llm.ts       # LLM streaming with AI SDK
│   ├── message-v2.ts # Message schema
│   ├── compaction.ts # Context compaction
│   └── prompt.ts    # Prompt construction
├── provider/        # Provider abstraction
│   ├── provider.ts  # Provider interface
│   ├── models.ts    # Model definitions
│   └── transform.ts # Provider-specific transforms
├── server/          # HTTP server (Hono)
├── config/          # Configuration (opencode.json)
├── permission/      # Permission system
├── mcp/             # MCP server integration
├── lsp/             # LSP integration
└── storage/         # SQLite database
```

### 2.4 KiloCode (OpenCode Fork)

**Core Architecture:**
- Fork of OpenCode with Kilo-specific additions
- Same client-server architecture as OpenCode
- Added `kilo-gateway` for auth, provider routing, API integration
- VS Code extension with Agent Manager (multi-session orchestration)
- `kilocode_change` markers for merge conflict management

**Key Additions over OpenCode:**
- `packages/kilo-gateway/` — Auth, provider routing
- `packages/kilo-vscode/` — VS Code extension with Agent Manager
- `packages/kilo-telemetry/` — PostHog analytics
- `packages/kilo-i18n/` — Internationalization
- `AgentManagerApp.tsx` — Multi-session orchestration UI

### 2.5 iFlow CLI

**Core Architecture:**
- Node.js application installed via npm
- OpenAI-compatible API protocol
- Built-in Open Market for MCP servers, SubAgents, custom instructions
- Supports 4 running modes: yolo, accepting edits, plan mode, default
- SubAgent system for specialized tasks
- Context auto-compression at 70%

**Key Features:**
- Free AI models (Kimi K2, Qwen3 Coder, DeepSeek v3)
- Multimodal capability (paste images with Ctrl+V)
- Conversation history saving and rollback
- Workflow automation support
- VSCode and JetBrains plugins

---

## 3. Comparison Matrix

### 3.1 Tool System

| Feature | Claude Code | Codex | OpenCode | KiloCode | iFlow |
|---------|------------|-------|----------|----------|-------|
| Tool count | ~15 | ~10 core + MCP | 20+ built-in | 20+ built-in | Variable (MCP) |
| Tool definition | JSON schema | Rust traits | Tool.define() + Zod | Same as OpenCode | JSON schema |
| Custom tools | MCP | MCP | Plugin + MCP | Plugin + MCP | MCP + Open Market |
| Permission system | Conservative | Sandbox + approval | Per-agent ruleset | Same as OpenCode | 4 modes |
| Tool routing | Flat | Registry + router | Registry + filter | Same as OpenCode | Flat |
| Error recovery | Retry + context | Retry + sandbox | Retry + truncation | Same as OpenCode | Retry |
| Batch operations | No | No | BatchTool (experimental) | Same as OpenCode | No |

### 3.2 Agent System

| Feature | Claude Code | Codex | OpenCode | KiloCode | iFlow |
|---------|------------|-------|----------|----------|-------|
| Built-in agents | 1 + subagents | 1 + hierarchical | 4 (build, plan, general, explore) | 4 + Kilo modes | 4 modes |
| Custom agents | Agent Teams | External agents | Config-driven | Same as OpenCode | SubAgents |
| Agent coordination | Teams (2-16) | Multi-task | Subagent spawning | Same as OpenCode | SubAgent |
| Plan mode | No | update_plan tool | Dedicated plan agent | Same as OpenCode | Plan mode |
| Context management | CLAUDE.md | AGENTS.md | CLAUDE.md + .opencode/ | Same as OpenCode | IFLOW.md |
| Memory | Auto-memory | Session | Session + DB | Same as OpenCode | Conversation history |

### 3.3 Session & State Management

| Feature | Claude Code | Codex | OpenCode | KiloCode | iFlow |
|---------|------------|-------|----------|----------|-------|
| Persistence | File-based | Rollout files | SQLite (Drizzle) | Same as OpenCode | File-based |
| Session forking | No | No | Yes | Yes | No |
| Session sharing | No | No | Yes (URL sharing) | Yes | No |
| Undo/Redo | Git-based | Git-based | Git snapshots + /undo | Same as OpenCode | Rollback |
| Compaction | Auto | Auto | Dedicated agent | Same as OpenCode | Auto at 70% |
| Message format | Flat history | Turn items | MessageV2 with parts | Same as OpenCode | Flat history |

### 3.4 Provider & Model Support

| Feature | Claude Code | Codex | OpenCode | KiloCode | iFlow |
|---------|------------|-------|----------|----------|-------|
| Provider count | 1 (Anthropic) | 1 (OpenAI) | 75+ | 500+ (via gateway) | OpenAI-compatible |
| Local models | No | No | Ollama, LM Studio | Same as OpenCode | No |
| Model selection | CLI flag | Config | Config + TUI | Same + Gateway | Settings.json |
| Cost tracking | Limited | Limited | Full (per-token) | Same + Gateway | Limited |
| Streaming | Yes | Yes | Yes (AI SDK) | Same as OpenCode | Yes |

### 3.5 Safety & Sandboxing

| Feature | Claude Code | Codex | OpenCode | KiloCode | iFlow |
|---------|------------|-------|----------|----------|-------|
| Sandboxing | No (trust-based) | Landlock/Seatbelt | Git snapshots | Same as OpenCode | No |
| Approval policy | Interactive | Configurable modes | Per-agent ruleset | Same as OpenCode | 4 modes |
| File system safety | Read-only default | Sandbox default | Permission rules | Same as OpenCode | Mode-based |
| Network safety | No | Network policy | No | No | No |
| Rollback | Git | Git | Git snapshots + /undo | Same as OpenCode | Conversation rollback |

### 3.6 Extensibility

| Feature | Claude Code | Codex | OpenCode | KiloCode | iFlow |
|---------|------------|-------|----------|----------|-------|
| Plugin system | MCP | MCP | Plugin.ts | Same + Kilo plugins | MCP |
| MCP support | Yes | Yes | Yes | Yes | Yes |
| LSP integration | No | No | Yes | Yes | No |
| SDK | Claude Agent SDK | Codex SDK | TypeScript SDK | Same + Kilo SDK | iFlow SDK |
| Hooks | Limited | Hooks system | Plugin hooks | Same as OpenCode | Hook support |
| IDE integration | Terminal | CLI + IDE | TUI + Desktop + Web | VS Code + CLI | VS Code + JetBrains |

---

## 4. Benchmark Comparison (Web Research)

### SWE-bench Pro (Private Repos, More Credible)

| Agent | Score |
|-------|-------|
| Opus 4.6 + Claude Code | ~57.5% |
| GPT-5.3-Codex | ~57.0% |
| OpenCode (with Claude) | Depends on model |
| iFlow | Not benchmarked |

### Key Insight
The gap between Claude Code and Codex is closing. The agent harness matters less than the model quality. OpenCode's performance depends entirely on which model it runs.

---

## 5. Strengths & Weaknesses Summary

### Claude Code
- **Strengths:** Best reasoning, fastest execution, minimal configuration, subagent support
- **Weaknesses:** Vendor lock-in, rate limits, no LSP, no git snapshots

### Codex CLI
- **Strengths:** Safety-first sandboxing, GitHub ecosystem, Rust performance, ChatGPT bundle
- **Weaknesses:** Vendor lock-in, fragmented surface, less mature TUI

### OpenCode
- **Strengths:** Model-agnostic, open source, LSP integration, best TUI, session management
- **Weaknesses:** Slower execution, more configuration, local model tool calling unreliable

### KiloCode
- **Strengths:** OpenCode flexibility + Kilo gateway, VS Code Agent Manager, 500+ models
- **Weaknesses:** Fork maintenance overhead, upstream merge conflicts

### iFlow CLI
- **Strengths:** Free models, Open Market, multimodal, workflow automation
- **Weaknesses:** Shutting down April 17, 2026, less mature architecture

---

## 6. Key Architectural Patterns Observed

### 6.1 Agent Loop (Universal)
All agents implement the same core loop:
```
User Input → LLM → Tool Calls → Execute → Results → LLM → ... → Final Response
```

### 6.2 Tool Definition Pattern
- **OpenCode/KiloCode:** `Tool.define(id, { description, parameters: z.ZodSchema, execute })` — most elegant
- **Codex:** Rust traits with `ToolSpec` — type-safe but verbose
- **Claude Code/iFlow:** JSON schema — simple but error-prone

### 6.3 Permission Model
- **OpenCode:** Per-agent ruleset with merge semantics — most flexible
- **Codex:** Sandbox + approval modes — most secure
- **Claude Code:** Conservative default — simplest

### 6.4 Context Management
- **All:** Project context files (CLAUDE.md, AGENTS.md, IFLOW.md)
- **OpenCode:** + SQLite persistence + session forking + compaction agents
- **Codex:** + Rollout recording for replay

### 6.5 Provider Abstraction
- **OpenCode:** AI SDK (Vercel) — most abstract
- **Claude Code:** Direct Anthropic API
- **Codex:** Direct OpenAI Responses API
- **iFlow:** OpenAI-compatible protocol

---

## 7. What to Learn from Each

| From | Learn | Why |
|------|-------|-----|
| Claude Code | Minimal design, exact-match edits, CLAUDE.md pattern | Simplicity = debuggability |
| Codex | Rust safety, sandbox-first, AGENTS.md cascading, rollout recording | Security + performance |
| OpenCode | Tool.define() pattern, Effect.ts services, LSP integration, session management | Flexibility + type safety |
| KiloCode | Fork management, Agent Manager, gateway pattern | Multi-session orchestration |
| iFlow | Open Market, SubAgent system, free model access | Ecosystem + accessibility |
| sglang blog | System design > code execution, task×model orthogonal separation | Architecture decisions matter most |
