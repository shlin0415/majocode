# 10 Key Questions & Answers: MajoCode Memory System Design

> Based on compare-memory-sys.md research.
> Date: 2026-04-13

---

## Q1: What memory architecture should MajoCode use?

**A. Simple 2-tier (like OpenCode)** — Hot tail (~40K) + cold storage that can be pruned.
**B. 3-tier Compact (like Claude Code)** — MicroCompact → Session Memory → Full Compact.
**C. 3-layer Memory (like Antigravity)** — Working, Episodic, Semantic layers.

**Recommendation: Hybrid B + C for team use**

For a "git memory hub" for teams:
- **Tier 1**: Working memory (current session, hot tail)
- **Tier 2**: Session memory (completed sessions, compressible)
- **Tier 3**: Shared memory hub (team knowledge, project context)

Start with B (3-tier compact) for single sessions. Add C (3-layer) for team memory hub later.

---

## Q2: Should MajoCode support long-term memory beyond session?

**A. SQLite only** — Keep simple, focus on session.
**B. Hybrid (SQLite + Vector)** — Sessions + semantic search.
**C. Full stack (PostgreSQL + Vector)** — Like Antigravity.

**Recommendation: A for MVP, B for Phase 2**

Per your input: "MVP simple" - start with SQLite for session persistence. Add vector DB later when:
- Team memory hub is needed
- Cross-project context becomes important

---

## Q3: How should compaction handle file restoration?

**A. Threshold-based** — LLM summarize when hitting limit.
**B. Full restore** — Re-read files after summarize (Claude Code pattern).
**C. Hybrid** — Both threshold trigger + file restore.

**Recommendation: C**

As you specified: "Hybrid" - 
1. Trigger: When tokens >= usable context (OpenCode isOverflow)
2. Action: LLM summarize to structured summary
3. Restore: Re-read top N recent files (5 files, 5K tokens each)
4. Continuation: Tell agent to resume work

---

## Q4: What memory vital levels for MajoCode?

**A. Single tier** — All messages equal priority.
**B. Two levels** — Hot (recent) + Cold (older).
**C. Multiple levels** — Fine-grained priority.

**Recommendation: B + team layer**

```
┌─────────────────────────────────────┐
│ Team Memory Hub (future)            │  ← Shared across projects
├─────────────────────────────────────┤
│ Session Memory (current)           │  ← Compressible
│   ├─ Hot: Last 40K tokens           │
│   └─ Cold: Older, pruned            │
├─────────────────────────────────────┤
│ Working Memory (in-turn)           │  ← Active context
└─────────────────────────────────────┘
```

---

## Q5: How should tool outputs be handled in memory?

**A. Keep all** — Full fidelity, but context grows.
**B. Prune old (like OpenCode)** — Clear outputs after threshold.
**C. Truncate all** — Every tool output capped.

**Recommendation: B with protection**

```typescript
const PRUNE_MINIMUM = 20_000   // Must exceed this to prune
const PRUNE_PROTECT = 40_000   // Always keep this much
const PROTECTED_TOOLS = ["skill", "mcp"]  // Never prune these
```

- Walk backwards through tool results
- Clear outputs until reaching PROTECT threshold
- Skip protected tools completely

---

## Q6: How to handle context overflow errors (413)?

**A. Stop immediately** — Surface error to user.
**B. Reactive compact (like Claude Code)** — Try to recover.
**C. Ask user** — Let them decide.

**Recommendation: B with fallback to A**

```
if (contextOverflow) {
  // Try 1: Prune old tool outputs
  await pruneToolOutputs()
  if (!overflow) return "continue"  // Recovered!
  
  // Try 2: LLM summarize
  await compactSession()
  if (!overflow) return "continue"  // Recovered!
  
  // Failed: Stop and surface error
  return "stop"
}
```

---

## Q7: Should session summaries be automatic or manual?

**A. Automatic only** — Compact when hitting limit.
**B. Manual only** — User triggers with /compact command.
**C. Both** — Auto on threshold + manual available.

**Recommendation: C**

- **Auto**: When `isOverflow()` returns true, trigger compaction
- **Manual**: `/compact` command for proactive compression
- This matches OpenCode behavior with Claude Code's manual option

---

## Q8: How to implement "git-like memory hub" for teams?

**A. Shared SQLite** — Team members share session DB.
**B. Memory snapshots** — Export/import session summaries.
**C. Central knowledge base** — Shared semantic memory.

**Recommendation: C (future), A (MVP)**

**MVP**: Single SQLite per project, sessions persist locally.

**Future Phase**:
- Export session summary as "memory snapshot"
- Store in central knowledge base (markdown files or DB)
- New sessions can reference "memory from project X"

This is like git but for knowledge instead of code.

---

## Q9: What should be preserved after compaction?

**A. Summary only** — Compressed conversation.
**B. Summary + file references** — What files were touched.
**C. Summary + files + todos** — Full restoration (Claude Code).

**Recommendation: C**

After LLM summarize, restore:
1. ✅ Summary message (structured, 9 sections)
2. ✅ Top 5 recent files (re-read from disk)
3. ✅ Active todos/plans (if tracked)
4. ✅ Skills that were invoked
5. ✅ Continuation instruction: "Resume work"

This ensures the agent can continue seamlessly.

---

## Q10: How to handle memory on crash/error?

**A. No special handling** — Rely on SQLite persistence.
**B. Checkpoint per turn** — Save state after each turn.
**C. Incremental save** — Save during turn, not just end.

**Recommendation: B**

```typescript
async function processTurn() {
  try {
    const result = await doTurn()
    await saveSession()  // Checkpoint after success
    return result
  } catch (error) {
    // On error, session already persisted
    // Resume will pick up from last checkpoint
    throw error
  }
}
```

This ensures:
- After each turn: session state saved to SQLite
- On crash: resume from last successful turn
- Tool outputs already pruned/compacted as needed

---

## Summary of Recommendations

| Priority | Feature | Implementation |
|----------|---------|----------------|
| P0 | Token counting | `Token.estimate()` from OpenCode |
| P0 | isOverflow detection | `overflow.ts` from OpenCode |
| P0 | Session persistence | SQLite (same as OpenCode) |
| P1 | 3-tier compaction | Claude Code tiers (Micro/Session/Full) |
| P1 | Prune tool outputs | `compaction.ts:prune()` logic |
| P1 | File rehydration | Re-read top 5 files after compact |
| P2 | Team memory hub | Future: shared knowledge base |
| P2 | Memory snapshots | Future: export/import summaries |

---

## Architecture Diagram

```
MajoCode Memory System
├── Working Memory (current turn)
│   └── In-context, no persistence needed
├── Session Memory (current session)
│   ├── Hot: Last 40K tokens (protected)
│   ├── Cold: Older (pruneable)
│   └── Persistence: SQLite
├── Team Memory Hub (future)
│   ├── Session snapshots (exported)
│   ├── Project knowledge (shared)
│   └── Persistence: Central DB
└── Compaction Layer
    ├── Trigger: isOverflow()
    ├── Method: LLM summarize
    └── Restore: Re-read files + continuation
```

---

## Files Referenced

- OpenCode: `third_party/opencode/packages/opencode/src/session/compaction.ts`
- OpenCode: `third_party/opencode/packages/opencode/src/session/overflow.ts`
- KiloCode: `third_party/kilocode/packages/opencode/src/session/compaction.ts`
- Claude Code: Web research - decodeclaude.com, compact system
- Codex: `third_party/codex/codex-rs/core/src/compact_remote.rs`
- Antigravity: Web research - antigravitylab.net memory patterns