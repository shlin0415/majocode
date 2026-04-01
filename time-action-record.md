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

## Summary

**Deliverables:**
1. `time-action-record.md` — Timeline of all actions taken
2. `compare-code-agents.md` — Deep comparison of Claude Code, Codex, OpenCode, KiloCode, iFlow
3. `how-to-build-code-agent-majocode.md` — Architecture design for MajoCode
4. `10-questions-and-answers.md` — Key design decisions with recommendations

**Key Findings:**
- All code agents share the same core loop: Observe → Reason → Act → Verify
- OpenCode's architecture (TypeScript + Bun + Tool.define() + SQLite) is the best foundation
- Model-agnosticism is the winning strategy (avoid vendor lock-in)
- System design decisions matter more than code execution (sglang lesson)
- MVP should be 2-3 weeks: core loop + 6 tools + basic TUI

**Next Steps:**
1. Review documents with team
2. Make final design decisions based on Q&A
3. Start Phase 1 implementation (project scaffolding)
