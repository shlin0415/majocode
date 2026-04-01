# 10 Key Questions & Answers for Building MajoCode

> Design decisions with A/B/C options and recommendations.
> Date: 2026-04-01

---

## Q1: What programming language should MajoCode be built in?

**A. TypeScript + Bun** — Like OpenCode/KiloCode. Fast iteration, large ecosystem, good enough performance.
**B. Rust** — Like Codex CLI. Maximum performance, type safety, but slower development.
**C. Python** — Like Claude Code port. Simple, but slower runtime and weaker CLI tooling.

**Recommendation: A (TypeScript + Bun)**

Rationale: OpenCode proves TypeScript+Bun works well for code agents. Development speed matters more than raw performance for a CLI tool. Bun provides native TypeScript support and good SQLite integration. Rust can be added later for performance-critical paths.

---

## Q2: Should MajoCode be model-agnostic or locked to one provider?

**A. Model-agnostic (like OpenCode)** — Support 75+ providers, no vendor lock-in.
**B. Single provider (like Claude Code)** — Optimize for one model family, best performance.
**C. Primary + fallback** — Default to one provider, allow fallback to others.

**Recommendation: A (Model-agnostic)**

Rationale: The code agent market is moving toward model agnosticism. Models improve quarterly, and locking to one provider limits flexibility. OpenCode's approach of supporting 75+ providers via AI SDK is proven. KiloCode extends this to 500+ models via gateway.

---

## Q3: What TUI framework should be used?

**A. ink (React for terminal)** — Familiar React paradigm, good component model.
**B. Bubble Tea (Go)** — Like OpenCode's TUI. Excellent terminal UI, but requires Go.
**C. ratatui (Rust)** — Like Codex. Excellent, but requires Rust.

**Recommendation: A (ink)**

Rationale: Since we chose TypeScript, ink provides the best DX. React's component model maps well to terminal UIs. OpenCode's Go TUI is excellent but would require a polyglot codebase. ink keeps everything in one language.

---

## Q4: How should tools be defined?

**A. Tool.define() + Zod (like OpenCode)** — Type-safe, auto-generated schemas, clean API.
**B. JSON schema (like Claude Code/iFlow)** — Simple, but error-prone and verbose.
**C. Rust traits (like Codex)** — Type-safe, but requires Rust.

**Recommendation: A (Tool.define() + Zod)**

Rationale: OpenCode's pattern is the most elegant:
```typescript
const MyTool = Tool.define("my_tool", {
  description: "Does something",
  parameters: z.object({ path: z.string() }),
  execute: async (args, ctx) => ({ output: "...", metadata: {} })
})
```
This provides runtime validation, TypeScript types, and clean API in one package.

---

## Q5: How should session state be persisted?

**A. SQLite via Drizzle (like OpenCode)** — Structured, queryable, ACID, no external deps.
**B. File-based JSON** — Simple, but hard to query and lacks transactions.
**C. PostgreSQL** — Powerful, but requires external service.

**Recommendation: A (SQLite via Drizzle)**

Rationale: OpenCode proves SQLite works well for session management. It provides structured queries for history, ACID transactions for consistency, and zero external dependencies. Drizzle ORM provides type-safe queries and migration support.

---

## Q6: Should MajoCode have sandbox support from the start?

**A. Yes, sandbox-first (like Codex)** — Safety by default, approve dangerous ops.
**B. No, trust-based (like Claude Code)** — Simpler, rely on git rollback.
**C. Optional, configurable** — Allow users to choose.

**Recommendation: C (Optional, configurable)**

Rationale: Sandboxing is valuable but adds complexity. Start with trust-based + git snapshots (like OpenCode), then add optional sandbox support. Codex's Landlock/Seatbelt approach is excellent but requires significant Rust infrastructure. Git snapshots provide a good safety net.

---

## Q7: How should context be managed across sessions?

**A. CLAUDE.md files + SQLite persistence** — Project context files + session history.
**B. CLAUDE.md files only** — Simple, but no cross-session memory.
**C. RAG-based retrieval** — Advanced, but complex and may hallucinate.

**Recommendation: A (CLAUDE.md + SQLite)**

Rationale: All successful code agents use project context files (CLAUDE.md, AGENTS.md, IFLOW.md). Adding SQLite persistence enables cross-session memory, session forking, and history search. RAG is overkill for most use cases.

---

## Q8: Should MajoCode support MCP (Model Context Protocol)?

**A. Yes, full MCP support** — Extensibility via MCP servers.
**B. No, plugin-only** — Custom plugin system, no MCP.
**C. Yes, but as optional bridge** — MCP as one of several extension mechanisms.

**Recommendation: A (Full MCP support)**

Rationale: MCP has become the de facto standard for tool extensibility. Claude Code, Codex, OpenCode, and iFlow all support it. Building a custom plugin system would fragment the ecosystem. MCP provides a standard interface for tool servers.

---

## Q9: What should the agent permission model look like?

**A. Per-agent ruleset with merge (like OpenCode)** — Flexible, composable permissions.
**B. Approval modes (like Codex)** — Simple, but less granular.
**C. Trust-based (like Claude Code)** — Simplest, but least safe.

**Recommendation: A (Per-agent ruleset with merge)**

Rationale: OpenCode's permission system is the most flexible:
```typescript
permission: Permission.merge(
  defaults,                    // base rules
  Permission.fromConfig({      // agent-specific overrides
    question: "allow",
    plan_enter: "allow",
  }),
  user,                        // user overrides
)
```
This allows fine-grained control per agent, per tool, and per file pattern.

---

## Q10: What is the minimum viable product (MVP)?

**A. Full-featured CLI with all tools** — Complete but takes 10+ weeks.
**B. Core loop with basic tools** — Functional in 2-3 weeks, iterate from there.
**C. Library/SDK first** — Build the engine, TUI later.

**Recommendation: B (Core loop with basic tools)**

Rationale: Start with the core agent loop + 6 essential tools (bash, read, write, edit, glob, grep) + basic TUI. This is functional in 2-3 weeks and provides a working prototype. Iterate based on real usage feedback. This follows the "shortest path" principle from AGENTS.md.

### MVP Scope (Week 1-3):
- [ ] Agent loop (LLM → Tools → Execute → Repeat)
- [ ] Tool system with Tool.define() + Zod
- [ ] 6 core tools: bash, read, write, edit, glob, grep
- [ ] Provider abstraction (AI SDK, Claude + OpenAI)
- [ ] Basic TUI (ink)
- [ ] Configuration (opencode.json style)
- [ ] Git integration (snapshots, undo)

### After MVP (Week 4+):
- [ ] SQLite persistence
- [ ] Session forking/sharing
- [ ] Permission system
- [ ] More tools (webfetch, websearch, task, etc.)
- [ ] MCP integration
- [ ] LSP client
- [ ] Agent registry (plan, explore, general)

---

## Summary of Recommendations

| Question | Recommendation | Key Rationale |
|----------|---------------|---------------|
| Language | TypeScript + Bun | Fast iteration, proven by OpenCode |
| Model support | Model-agnostic | Flexibility, no vendor lock-in |
| TUI | ink (React) | Familiar paradigm, single language |
| Tool definition | Tool.define() + Zod | Type-safe, clean API |
| Persistence | SQLite via Drizzle | Structured, ACID, zero deps |
| Sandbox | Optional | Start simple, add later |
| Context | CLAUDE.md + SQLite | Project context + session history |
| MCP | Full support | De facto standard |
| Permissions | Per-agent ruleset | Flexible, composable |
| MVP | Core loop + basic tools | 2-3 weeks to functional prototype |
