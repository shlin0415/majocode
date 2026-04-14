# Time Action Record - MajoCode Research & Design

## Timeline

| Start | End | Action | Why | Brief Result | Next |
|-------|-----|--------|-----|-------------|------|
| 2026-04-01 00:00 | 2026-04-01 00:15 | Initial workspace exploration | Understand project structure | Found AGENTS.md, CLAUDE.md, third_party/ with 5 code agents + 2 system design examples | Deep dive into source codes |
| 2026-04-01 00:15 | 2026-04-01 00:45 | Deep exploration of third_party source codes | Understand architecture of each agent | Mapped Claude Code (Python port), Codex (Rust), OpenCode (TS/Bun), KiloCode (TS/Bun fork), iFlow (Node.js) | Read core implementation files |
| 2026-04-01 00:45 | 2026-04-01 01:15 | Read core implementation files | Understand agent loop, tool system, session management | OpenCode: Effect.ts services, Tool.define(), Agent namespace, Session management with Drizzle DB | Web research |
| 2026-04-01 01:15 | 2026-04-01 01:30 | Web research on code agent architecture | Get industry benchmarks and best practices | Found SWE-bench Pro scores, architecture patterns, comparison articles | Write comparison doc |
| 2026-04-01 01:30 | 2026-04-01 02:00 | Write compare-code-agents.md | Document findings from source code analysis | Created detailed comparison of all 5 agents + Antigravity reference | Write design doc |
| 2026-04-01 02:00 | 2026-04-01 02:30 | Write how-to-build-code-agent-majocode.md | Design MajoCode based on best practices | Created architecture design with tool system, agent loop, session management, provider abstraction | Q&A section |
| 2026-04-01 02:30 | 2026-04-01 02:45 | Write 10 Q&A with recommendations | Clarify design decisions | Created 10 key questions with A/B/C options and recommendations | Final commit |
| 2026-04-01 02:45 | 2026-04-01 03:00 | Final git commits and cleanup | Ensure all changes are committed | 4 commits: time-action-record, compare-code-agents, how-to-build, 10-Q&A | Review with user |

## 2026-04-13 Research: Agent Loop Deep Dive

| Start | End | Action | Why | Brief Result | Next |
|-------|-----|--------|-----|-------------|------|
| 2026-04-13 00:00 | 2026-04-13 00:10 | Explore third_party/ structure | Find code agent source codes | Found 5 agents: claude-code, codex, opencode, kilocode, iflow | Read source files |
| 2026-04-13 00:10 | 2026-04-13 00:30 | Read session processor files | Understand agent loop implementation | OpenCode/KiloCode use while(true)+Effect, Claude has circuit breaker | Web research |
| 2026-04-13 00:30 | 2026-04-13 00:50 | Web search on agent loop architecture | Get detailed patterns for continue/error/restart | Found Claude Code #30150 (doom loop), OpenCode #17648 (infinite retry) | Write comparison |
| 2026-04-13 00:50 | 2026-04-13 01:10 | Write compare-agent-loop.md | Document findings with source citations | Created detailed comparison with 7 sections and recommended patterns | Commit changes |
| 2026-04-13 01:10 | 2026-04-13 01:20 | Git commit previous work | Ensure safe rollback | Committed compare-agent-loop.md and agent-loop-q-and-a.md | Start memory research |
| 2026-04-13 01:20 | 2026-04-13 01:40 | Research memory/compaction in OpenCode | Find source code for memory system | Found SessionCompaction, overflow.ts, prune mechanism | Research other agents |
| 2026-04-13 01:40 | 2026-04-13 02:00 | Web search on Claude Code compaction | Get detailed 3-tier system info | Found MicroCompact, Session Memory, Full Compact tiers | Web research Codex |
| 2026-04-13 02:00 | 2026-04-13 02:20 | Research KiloCode memory system | Compare fork differences | Same as OpenCode with minor additions | Research Antigravity |
| 2026-04-13 02:20 | 2026-04-13 02:40 | Web search Antigravity memory | Get 3-layer memory pattern | Found Episodic/Semantic/Procedural layers, Vector DB | Write comparison doc |
| 2026-04-13 02:40 | 2026-04-13 03:00 | Write compare-memory-sys.md | Document memory system findings | Created 8-section comparison with recommendations | Commit changes |
| 2026-04-13 03:00 | 2026-04-13 03:10 | Git commit | Ensure safe rollback | compare-memory-sys.md committed | Q&A phase |

## Summary

**Deliverables (2026-04-13):**
1. `time-action-record.md` — Timeline of all actions taken
2. `compare-code-agents.md` — Deep comparison of Claude Code, Codex, OpenCode, KiloCode, iFlow
3. `how-to-build-code-agent-majocode.md` — Architecture design for MajoCode
4. `10-questions-and-answers.md` — Key design decisions with recommendations
5. `compare-agent-loop.md` — Agent loop comparison (2026-04-13 update)
6. `compare-memory-sys.md` — Memory system comparison (new)

**Key Findings (Memory System):**
- OpenCode/KiloCode: 2-tier (hot tail ~40K protected, cold storage pruneable), LLM summarize
- Claude Code: 3-tier (MicroCompact → Session Memory → Full Compact), file rehydration
- Codex: Context manager with trim + remote compact
- Antigravity: 3-layer (Working, Episodic/Semantic, Procedural) with Vector DB

**Critical Memory Issues:**
- OpenCode: PRUNE_MINIMUM=20K, PRUNE_PROTECT=40K, prune clears tool outputs
- Claude Code: Structured 9-section summary, continuation message after compact
- Overflow: isOverflow check triggers compaction before hitting limit

**Recommended MajoCode Features:**
1. Token counting with Token.estimate()
2. isOverflow detection before limit
3. Prune old tool outputs (keep hot tail)
4. LLM summarization with structured prompt
5. Session persistence to SQLite
6. File rehydration after compaction

**Next Steps:**
1. Review compare-memory-sys.md with team
2. Discuss memory architecture for MajoCode
3. Start implementation planning
