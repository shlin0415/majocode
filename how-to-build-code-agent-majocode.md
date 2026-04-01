# How to Build Code Agent MajoCode

> Architecture design based on deep analysis of Claude Code, Codex, OpenCode, KiloCode, and iFlow.
> Date: 2026-04-01

---

## 1. First Principles: What is a Code Agent?

A code agent is an AI system that autonomously reads, edits, tests, and debugs code in a loop:

```
User Task → Plan → Read Files → Reason → Apply Edits → Run Tests → Verify → Report
                              ↑                                    |
                              └────────── Fail ←───────────────────┘
```

The core loop is: **Observe → Reason → Act → Verify**

### What Makes a Great Code Agent?

1. **Correctness**: Edits must be precise, not hallucinated
2. **Safety**: Never break the system without rollback capability
3. **Speed**: Minimize latency between user input and action
4. **Thoroughness**: Understand the full context before acting
5. **Extensibility**: Support new tools, models, and workflows
6. **Debuggability**: When things go wrong, understand why

---

## 2. MajoCode Design Philosophy

Based on analysis of existing agents and the sglang system design example:

> "Architecture decisions matter more than code execution. The right architecture makes 10,000 lines of code an asset; the wrong architecture makes them a debt." — from sglang-omni blog

### Design Principles

1. **Minimal surface area** (from Claude Code): Simple design = debuggable design
2. **Safety-first** (from Codex): Sandbox by default, approve dangerous operations
3. **Model-agnostic** (from OpenCode): Don't lock to one provider
4. **Type-safe tool system** (from OpenCode): Tool.define() with Zod schemas
5. **Session persistence** (from OpenCode): SQLite for history, forking, sharing
6. **System design over code** (from sglang): task×model orthogonal separation

### What MajoCode Should NOT Be

- NOT a VS Code extension (that's a separate product)
- NOT a cloud service (run locally first)
- NOT locked to one model family
- NOT over-engineered (no premature abstraction)

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                   MajoCode CLI                   │
├─────────────────────────────────────────────────┤
│  TUI Layer (Terminal UI)                         │
│  ├── Input handling                              │
│  ├── Message rendering                           │
│  └── Tool call visualization                     │
├─────────────────────────────────────────────────┤
│  Agent Layer                                     │
│  ├── Agent Loop (core orchestrator)              │
│  ├── Agent Registry (build, plan, explore, ...)  │
│  ├── Permission System (per-agent ruleset)       │
│  └── SubAgent Spawning                           │
├─────────────────────────────────────────────────┤
│  Tool Layer                                      │
│  ├── Tool Registry                               │
│  ├── Tool Definitions (Zod schemas)              │
│  ├── Built-in Tools (bash, read, write, edit...) │
│  ├── MCP Tool Bridge                             │
│  └── Plugin Tool Loader                          │
├─────────────────────────────────────────────────┤
│  Session Layer                                   │
│  ├── Session Manager (create, fork, share)       │
│  ├── Message Store (SQLite/Drizzle)              │
│  ├── Context Compaction                          │
│  └── Git Snapshot Integration                    │
├─────────────────────────────────────────────────┤
│  Provider Layer                                  │
│  ├── Provider Abstraction (AI SDK)               │
│  ├── Model Registry (Claude, GPT, Gemini, local) │
│  ├── Streaming Handler                           │
│  └── Cost Tracker                                │
├─────────────────────────────────────────────────┤
│  Infrastructure Layer                            │
│  ├── Configuration (opencode.json style)         │
│  ├── Project Detection (.git, package.json, etc) │
│  ├── LSP Client                                  │
│  └── File System Watcher                         │
└─────────────────────────────────────────────────┘
```

---

## 4. Core Components Design

### 4.1 Agent Loop

The heart of MajoCode. This is the simplest but most critical component.

**Design (inspired by Claude Code's minimal approach):**

```typescript
interface AgentLoop {
  // Core loop: LLM → Tools → Execute → Repeat
  run(session: Session, input: UserMessage): AsyncGenerator<AgentEvent>
  
  // Event types
  // - text: streaming text from LLM
  // - tool_call: tool invocation request
  // - tool_result: tool execution result
  // - error: error occurred
  // - done: agent finished
}
```

**Key decisions:**
- Single main thread (like Claude Code) — no multi-agent orchestration by default
- Subagents for parallel isolated tasks (like OpenCode's explore/general agents)
- Preamble messages before tool calls (like Codex)
- Plan tool for multi-step tasks (like Codex's update_plan)

### 4.2 Tool System

**Design (inspired by OpenCode's Tool.define() pattern):**

```typescript
interface ToolDef<Params extends z.ZodType, M extends Metadata> {
  id: string
  description: string
  parameters: Params
  execute(args: z.infer<Params>, ctx: ToolContext): Promise<{
    title: string
    metadata: M
    output: string
  }>
}
```

**Built-in tools (prioritized by analysis):**

| Tool | Priority | Inspired By | Notes |
|------|----------|-------------|-------|
| bash | P0 | All agents | Shell execution |
| read | P0 | All agents | File reading |
| write | P0 | All agents | File writing |
| edit | P0 | All agents | Exact-match editing |
| glob | P0 | All agents | File pattern matching |
| grep | P0 | All agents | Content search |
| todowrite | P1 | OpenCode | Task tracking |
| webfetch | P1 | OpenCode, Claude Code | URL fetching |
| websearch | P2 | OpenCode | Web search (Exa) |
| task | P2 | OpenCode | Subagent spawning |
| question | P2 | OpenCode | User clarification |
| skill | P2 | OpenCode | Skill invocation |
| apply_patch | P2 | Codex | Atomic patch application |
| codesearch | P3 | OpenCode | Code search |
| lsp | P3 | OpenCode | LSP diagnostics |
| batch | P3 | OpenCode | Batch operations |

### 4.3 Agent Registry

**Design (inspired by OpenCode's Agent namespace):**

```typescript
interface AgentInfo {
  name: string
  description?: string
  mode: "subagent" | "primary" | "all"
  permission: PermissionRuleset
  model?: { providerID: string; modelID: string }
  prompt?: string
  temperature?: number
}
```

**Built-in agents:**

| Agent | Mode | Description |
|-------|------|-------------|
| build | primary | Default, full-access agent for development |
| plan | primary | Read-only agent for analysis and planning |
| explore | subagent | Fast codebase exploration |
| general | subagent | Complex multi-step tasks |

### 4.4 Permission System

**Design (inspired by OpenCode's Permission.merge() pattern):**

```typescript
type PermissionAction = "allow" | "deny" | "ask"

interface PermissionRule {
  permission: string  // tool ID or special permission
  action: PermissionAction
  pattern?: string    // glob pattern for file-specific rules
}

type PermissionRuleset = PermissionRule[]
```

**Default rules:**
- `*`: allow (all tools allowed by default)
- `edit.*.env`: ask (environment files require approval)
- `question`: deny (for subagents)
- `plan_enter`/`plan_exit`: deny (for subagents)

### 4.5 Session Management

**Design (inspired by OpenCode's Session namespace with SQLite):**

```typescript
interface Session {
  id: string
  slug: string
  projectID: string
  directory: string
  title: string
  parentID?: string  // for forked sessions
  time: { created: number; updated: number }
}
```

**Features:**
- SQLite persistence (Drizzle ORM)
- Session forking (copy messages up to a point)
- Session sharing (URL-based)
- Git-based undo/redo with snapshots
- Context compaction (dedicated agent)

### 4.6 Provider Abstraction

**Design (inspired by OpenCode's AI SDK integration):**

```typescript
interface Provider {
  id: string
  name: string
  models: ModelInfo[]
  getModel(modelID: string): LanguageModel
}
```

**Supported providers (priority order):**
1. Anthropic (Claude)
2. OpenAI (GPT)
3. Google (Gemini)
4. DeepSeek
5. Ollama (local)
6. LM Studio (local)
7. Any OpenAI-compatible API

### 4.7 Configuration

**Design (inspired by OpenCode's opencode.json):**

```jsonc
{
  "$schema": "https://majocode.dev/config.schema.json",
  "provider": {
    "anthropic": {
      "apiKey": "sk-..."
    }
  },
  "model": "anthropic/claude-sonnet-4-6",
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4-6"
    }
  },
  "permission": {
    "edit": {
      "*.env": "ask"
    }
  }
}
```

---

## 5. Implementation Roadmap

### Phase 1: Core (Week 1-2)
- [ ] Project scaffolding (TypeScript + Bun)
- [ ] Agent loop implementation
- [ ] Tool system with Tool.define()
- [ ] Basic tools: bash, read, write, edit, glob, grep
- [ ] Provider abstraction (AI SDK)
- [ ] Simple TUI (ink or bubbletea)

### Phase 2: Session (Week 3-4)
- [ ] SQLite persistence (Drizzle)
- [ ] Session CRUD operations
- [ ] Message history management
- [ ] Context compaction
- [ ] Git snapshot integration

### Phase 3: Agents (Week 5-6)
- [ ] Agent registry
- [ ] Permission system
- [ ] Built-in agents (build, plan, explore, general)
- [ ] Subagent spawning
- [ ] Plan tool

### Phase 4: Extensibility (Week 7-8)
- [ ] MCP server integration
- [ ] Plugin system
- [ ] LSP client
- [ ] Configuration system
- [ ] Multi-provider support

### Phase 5: Polish (Week 9-10)
- [ ] TUI improvements
- [ ] Error recovery
- [ ] Cost tracking
- [ ] Documentation
- [ ] Testing

---

## 6. Technology Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Language | TypeScript | Type safety, ecosystem, team familiarity |
| Runtime | Bun | Fast, native TypeScript, good for CLI |
| TUI | ink (React for terminal) | Familiar paradigm, good DX |
| Database | SQLite via Drizzle | Lightweight, embedded, proven (OpenCode uses it) |
| LLM SDK | Vercel AI SDK | Provider-agnostic, streaming, tool support |
| Schema | Zod | Runtime validation, TypeScript integration |
| Config | JSON with JSON Schema | Simple, validated, IDE support |
| HTTP | Hono (if needed) | Lightweight, fast, Bun-compatible |
| Testing | Bun test | Native, fast |
| Build | Bun bundler | Native, fast |

---

## 7. Key Design Decisions & Rationale

### 7.1 Why TypeScript (not Rust)?

- **Codex chose Rust** for performance and safety, but development is slower
- **OpenCode chose TypeScript** for development speed and ecosystem
- **MajoCode chooses TypeScript** for: faster iteration, larger contributor pool, good enough performance for CLI
- Rust can be added later for performance-critical paths if needed

### 7.2 Why Bun (not Node.js)?

- OpenCode and KiloCode both use Bun
- Bun is faster for CLI tools
- Native TypeScript support (no compilation step)
- Good SQLite support
- Cross-platform (Windows, macOS, Linux)

### 7.3 Why AI SDK (not direct API calls)?

- OpenCode uses it successfully
- Provider-agnostic: switch models without code changes
- Built-in streaming, tool calling, error handling
- Active development and good documentation

### 7.4 Why SQLite (not file-based)?

- OpenCode proves this works well at scale
- Structured queries for session history
- ACID transactions for consistency
- No external dependencies
- Good performance for local use

### 7.5 Why Single-Threaded Agent Loop?

- Claude Code proves this is sufficient
- Simpler to debug and reason about
- Subagents handle parallelism when needed
- Avoids concurrency bugs

---

## 8. Lessons from System Design Examples

### From MiniCPM-SALA (DDIA principles)
- **Reliability**: System must continue working when things go wrong
- **Scalability**: Design for growth, but don't over-engineer
- **Maintainability**: Make life easier for future developers

### From sglang-omni benchmark refactoring
- **task×model orthogonal separation**: Separate "what to test" from "how to test"
- **System design > code**: Architecture decisions have more value than code execution
- **Threshold design**: Performance margins are system engineering problems, not math problems
- **Human judgment > AI execution**: AI amplifies good and bad decisions equally

---

## 9. What MajoCode Inherits from Each Agent

| Feature | Source | Adaptation |
|---------|--------|-----------|
| Single-threaded agent loop | Claude Code | Core design |
| CLAUDE.md context files | Claude Code | MAJO.md support |
| Sandbox-first safety | Codex | Optional sandbox mode |
| AGENTS.md cascading | Codex | Cascading config discovery |
| Tool.define() pattern | OpenCode | Zod-based tool definitions |
| Effect.ts services | OpenCode | Simplified version (no Effect.ts initially) |
| SQLite persistence | OpenCode | Drizzle ORM |
| Session forking/sharing | OpenCode | Core feature |
| LSP integration | OpenCode | Phase 4 |
| Agent Manager pattern | KiloCode | Multi-session orchestration |
| Gateway pattern | KiloCode | Provider routing |
| Open Market concept | iFlow | MCP marketplace integration |
| SubAgent system | iFlow | Agent registry |
| System design approach | sglang | task×model separation |

---

## 10. Success Criteria

MajoCode is successful when:

1. **It works**: Can complete real coding tasks end-to-end
2. **It's safe**: Never breaks the system without rollback
3. **It's fast**: Response latency comparable to Claude Code
4. **It's extensible**: New tools and providers can be added easily
5. **It's debuggable**: When things go wrong, understand why quickly
6. **It's maintainable**: Code is clean, well-structured, easy to modify
