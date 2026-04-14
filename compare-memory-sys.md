# Compare Memory Systems: Claude Code, Codex, OpenCode, KiloCode, iFlow, Antigravity

> Analysis based on third_party source code inspection and web research.
> Date: 2026-04-13

---

## 1. Executive Summary

| Agent | Memory Layers | Compaction Strategy | Overflow Handling | Persistence |
|-------|---------------|---------------------|-------------------|--------------|
| Claude Code | 3-tier (Micro, Session, Full) | Summarize + restore files | Reactive compact | Session-based |
| Codex | Context Manager | Trim + compact remote | Token budget | SQLite |
| OpenCode | 2-tier (working + prune) | LLM summarize + prune | isOverflow check | SQLite |
| KiloCode | 2-tier (working + prune) | Same as OpenCode | Same as OpenCode | SQLite + JSON |
| iFlow | Unknown | Unknown | Unknown | Unknown |
| Antigravity | 3-layer (Episodic, Semantic, Procedural) | Vector DB + skill-based | Session resume | Vector + PostgreSQL |

---

## 2. Memory Vital Levels

### 2.1 OpenCode/KiloCode - Working + Prune

**Two-tier architecture:**

```typescript
// session/compaction.ts
export const PRUNE_MINIMUM = 20_000  // Minimum tokens to prune
export const PRUNE_PROTECT = 40_000  // Protected token window
const PRUNE_PROTECTED_TOOLS = ["skill"]  // Tools to never prune
```

**How it splits memory:**
1. **Hot tail**: Last ~40K tokens of recent tool results (protected)
2. **Cold storage**: Older tool results can be compacted
3. **Pruning**: Walks backwards through messages, clears old tool outputs

**Process:**
- Skip first 2 user turns (keep recent context)
- Skip if message already summarized
- Skip protected tools (skill)
- Stop when PRUNE_PROTECT tokens reached
- Mark with `part.state.time.compacted = Date.now()`

### 2.2 Claude Code - 3-Tier System

**Three-tier architecture:**

| Tier | Name | Trigger | Compression | Cache Impact |
|-------|------|---------|--------------|---------------|
| 1 | MicroCompact | Every turn | ~10-50K tokens | Preserves |
| 2 | Session Memory | Auto-compact threshold | ~60-80% | Invalidates but optimized |
| 3 | Full Compact | Auto-compact or /compact | ~80-95% | Invalidates |

**MicroCompact (Tier 1):**
- Clear old tool results early
- Keep hot tail (recent N tool results)
- Save bulky outputs to disk, keep reference in context

**Session Memory Compact (Tier 2):**
- Uses pre-built session memory as summary
- Avoids LLM summarization cost
- 40K token hard cap

**Full Compact (Tier 3):**
- LLM-powered summarization
- 9-section structured summary: intent, concepts, files, errors, pending tasks
- Chain-of-thought reasoning (stripped afterward)

### 2.3 Codex - Context Manager

**Trim-based approach:**
- `trim_function_call_history_to_fit_context_window()`
- Uses `ContextManager` to estimate tokens
- Remote compact via `/v1/compact` API

### 2.4 Antigravity - 3-Layer Memory

**Cognitive-inspired layers:**

| Layer | Type | Storage | Duration |
|-------|------|---------|----------|
| 1 | Working Memory | Context window | Session |
| 2 | Episodic Memory | Vector DB | Medium-term |
| 3 | Semantic Memory | PostgreSQL | Long-term |

```typescript
interface Memory {
  id: string;
  type: 'episodic' | 'semantic' | 'procedural';
  content: string;
  embedding: number[];
  importance: number;  // 0.0 to 1.0 retention priority
  timestamp: Date;
  tags: string[];
  source: string;
}
```

---

## 3. What Happens When Memory Exceeds Max Token Length

### 3.1 OpenCode/KiloCode - isOverflow Detection

**In `session/overflow.ts`:**

```typescript
const COMPACTION_BUFFER = 20_000

export function isOverflow(input) {
  const context = input.model.limit.context
  const count = input.tokens.total || input.tokens.input + input.tokens.output
  const reserved = input.cfg.compaction?.reserved ?? Math.min(COMPACTION_BUFFER, maxOutputTokens)
  const usable = input.model.limit.input ? input.model.limit.input - reserved : context - maxOutputTokens
  return count >= usable
}
```

**Flow:**
1. Check if tokens >= usable context (input - reserved for output)
2. If overflow: trigger `needsCompaction = true`
3. Return "compact" from processor loop
4. Run `SessionCompaction.create()`

### 3.2 Claude Code - Reactive Compact

**Trigger on 413 error (context_overflow):**

```typescript
// In query.ts
if (error.type === "context_overflow") {
  // Stage 1: Context collapse drain
  if (transition === "collapse_drain_retry") {
    // Stage 2: Reactive compact
    yield* reactiveCompact()
  }
}
```

**Process:**
1. First 413 → context collapse drain (try to free space)
2. Second 413 → reactive compact (LLM summarize + retry)
3. If fails → stop with error

### 3.3 Codex - Token Budget Enforcement

**In context_manager:**
- Pre-check if response will fit
- Trim oldest messages if needed
- Remote compact API for server-side

---

## 4. How They Compress Memory

### 4.1 OpenCode - Prune + LLM Summarize

**Step 1: Prune (automatic)**
```typescript
// Walk backwards, clear old tool outputs
for (let msgIndex = msgs.length - 1; msgIndex >= 0; msgIndex--) {
  // Skip first 2 user turns
  if (turns < 2) continue
  
  // Clear completed tool results until PRUNE_PROTECT reached
  if (total > PRUNE_PROTECT) {
    part.state.time.compacted = Date.now()
  }
}
```

**Step 2: LLM Summarize (manual or auto)**
```typescript
// Default prompt template:
`Provide a detailed prompt for continuing our conversation above.
Focus on information that would be helpful for continuing the conversation,
including what we did, what we're doing, which files we're working on,
and what we're going to do next.

## Goal
[What goal(s) is the user trying to accomplish?]

## Progress
[What has been accomplished so far?]

## Current Work
[What is currently being worked on?]

## Files
[List of files involved in this conversation]

## Errors
[Any errors encountered and how they were resolved]

## Plan
[What are the next steps?]
`
```

### 4.2 Claude Code - Structured Summary + Restore

**Post-compaction restoration:**
1. Boundary marker with pre-compaction metadata
2. Formatted 9-section summary
3. Top 5 recently-read files (50K token budget, 5K per file)
4. Re-injected skills (25K budget)
5. Tool definitions re-announced
6. Session hooks re-run
7. CLAUDE.md restored
8. Continuation message: "you were already working, don't recap"

### 4.3 Codex - Trim Function Call History

```rust
// In compact_remote.rs
let deleted_items = trim_function_call_history_to_fit_context_window(
    &mut history,
    turn_context.as_ref(),
    turn_context.model_context_window(),
);
```

---

## 5. How They Save Memory on Error

### 5.1 OpenCode - Persistent Storage

**SQLite-based session storage:**
```typescript
// session/index.ts - stores in SQLite
await session.create({
  id: sessionID,
  messages: [...],
  compacting: time_compacting
})
```

**On error:**
1. Session state persisted to SQLite
2. Messages with compact flag preserved
3. Tool results with `compacted` timestamp kept
4. Resume via `--resume` or `/chat`

### 5.2 Claude Code - Checkpoint System

**On error:**
1. Save last N turns as checkpoint
2. Tool results already truncated to disk
3. Session can resume from checkpoint
4. Uses `session_id` for resume

### 5.3 Antigravity - Vector + PostgreSQL

**Three-layer persistence:**
1. **Working memory**: In context window (lost on close)
2. **Episodic memory**: Vector DB (semantic search)
3. **Semantic memory**: PostgreSQL (structured facts)

**Error recovery:**
- Episodic memories searchable by embedding
- Semantic memories queryable by tags
- Session resume loads relevant memories

---

## 6. Comparison Summary

### 6.1 Memory Vital Levels

| Agent | Hot (protected) | Cold (compressible) | Long-term |
|-------|-----------------|-------------------|-----------|
| OpenCode | Last 40K tokens | Older tool outputs | SQLite |
| Claude Code | Hot tail (~5 recent) | All old content | Session memory |
| Codex | Recent messages | Trimmed history | SQLite |
| Antigravity | Context window | Vector DB | PostgreSQL |

### 6.2 Compaction Trigger Points

| Agent | Trigger | Method |
|-------|---------|--------|
| OpenCode | tokens >= input - reserved | isOverflow check |
| Claude Code | Free space < reserved headroom | Auto-compact |
| Claude Code | 413 error | Reactive compact |
| Codex | Token budget check | Trim + remote |

### 6.3 Compression Effectiveness

| Agent | Compression Ratio | Latency | Preserves |
|-------|-------------------|---------|-----------|
| OpenCode | ~60-80% | Medium | Tool references |
| Claude Code | ~80-95% | High | Files, todos, skills |
| Codex | ~50-70% | Low | Original structure |
| Antigravity | Variable | Depends on tier | Context + semantics |

---

## 7. Best Practices for MajoCode

Based on this analysis, recommend implementing:

### 7.1 Recommended Memory Architecture

```
┌─────────────────────────────────────────────┐
│           Working Memory                    │
│  (Context window - last N turns)           │
├─────────────────────────────────────────────┤
│         Hot Tail (protected)               │
│  Last ~40K tokens, recent tool outputs     │
├─────────────────────────────────────────────┤
│         Cold Storage                       │
│  Older content, pruneable tool results      │
├─────────────────────────────────────────────┤
│         Compaction Layer                   │
│  LLM summarize → structured summary         │
├─────────────────────────────────────────────┤
│         Persistence Layer                  │
│  SQLite (sessions) + disk (truncated)       │
└─────────────────────────────────────────────┘
```

### 7.2 Implementation Priority

| Priority | Feature | Source |
|----------|---------|--------|
| P0 | Token counting | OpenCode `Token.estimate()` |
| P0 | isOverflow detection | OpenCode `overflow.ts` |
| P0 | Prune old tool outputs | OpenCode `compaction.ts:prune()` |
| P1 | LLM summarization | OpenCode `compaction.ts:create()` |
| P1 | Session persistence | SQLite |
| P1 | File rehydration | Claude Code pattern |
| P2 | Vector memory | Antigravity pattern |
| P2 | Session memory compact | Claude Code Tier 2 |

### 7.3 Key Configuration

```typescript
export const MEMORY_CONFIG = {
  PRUNE_MINIMUM: 20_000,      // Minimum tokens to prune
  PRUNE_PROTECT: 40_000,      // Protected window
  COMPACTION_BUFFER: 20_000,  // Reserved for output
  MAX_FILE_RESTORE: 5,        // Files to re-read
  MAX_FILE_TOKENS: 5_000,     // Per-file budget
}
```

---

## 8. References

- OpenCode: `third_party/opencode/packages/opencode/src/session/compaction.ts`
- OpenCode: `third_party/opencode/packages/opencode/src/session/overflow.ts`
- KiloCode: `third_party/kilocode/packages/opencode/src/session/compaction.ts`
- Claude Code: Web research - decodeclaude.com compaction system
- Codex: `third_party/codex/codex-rs/core/src/compact_remote.rs`
- Antigravity: Web research - antigravitylab.net memory patterns
- GitHub: `architecture/11-compact-system.md` (Claude Code)