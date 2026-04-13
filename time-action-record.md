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
| 2026-04-13 01:10 | 2026-04-13 01:20 | Git add and commit | Ensure safe rollback | compare-agent-loop.md committed | Review docs |

## Summary

**Deliverables:**
1. `time-action-record.md` — Timeline of all actions taken
2. `compare-code-agents.md` — Deep comparison of Claude Code, Codex, OpenCode, KiloCode, iFlow
3. `how-to-build-code-agent-majocode.md` — Architecture design for MajoCode
4. `10-questions-and-answers.md` — Key design decisions with recommendations
5. `compare-agent-loop.md` — Agent loop comparison (2026-04-13 update)

**Key Findings (Agent Loop):**
- All agents use while(true) or async generator loop pattern
- OpenCode/KiloCode: Doom loop detection (3 consecutive same tool calls), retry with exponential backoff
- Claude Code: Circuit breaker (3 failures), reactive compact for 413 errors
- Codex: Rust async loop with error withholding
- Error handling: Retryable errors get exponential backoff, non-retryable surface to user
- Restart: Manual via --resume, no automatic crash recovery

**Critical Issues Found:**
- OpenCode #17648: Infinite retry with unbounded exponential backoff (no max retries)
- Claude Code #30150: Model retries identical failing tool calls indefinitely
- OpenCode #19394: 5xx errors incorrectly marked non-retryable

**Recommended MajoCode Features:**
1. Circuit breaker: Stop after N consecutive failures
2. Doom loop detection: Detect repeated tool calls
3. Max retries: Cap exponential backoff
4. Token budget: Per-turn enforcement
5. Session persistence: Save state for crash recovery

**Next Steps:**
1. Review compare-agent-loop.md with team
2. Apply recommendations to MajoCode design
3. Discuss specific Q&A on error handling strategy
