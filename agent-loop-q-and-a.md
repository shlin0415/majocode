# 10 Key Questions & Answers: MajoCode Agent Loop Design

> Based on compare-agent-loop.md research.
> Date: 2026-04-13

---

## Q1: What loop pattern should MajoCode use?

**A. while(true) + Effect (like OpenCode/KiloCode)** — Simple, readable, easy to debug.
**B. Async generator (like Claude Code)** — Better backpressure handling, yields control more often.
**C. Rust async/await (like Codex)** — Maximum performance, but requires Rust.

**Recommendation: A (Hybrid: B as base)**

Based on research:
- OpenCode uses plain while(true) with try/catch for error handling
- Claude Code's async generator pattern provides better control flow
- For MVP, use while(true) pattern (like OpenCode) but add generator-like yielding for better cancellation

---

## Q2: How should MajoCode handle the "continue" decision after each turn?

**A. Check needsFollowUp flag** — If model produced tool calls, continue; else stop.
**B. Token budget enforcement** — Check remaining budget before each turn.
**C. Both A + B** — Check both tool calls and budget.

**Recommendation: C**

```
if (!result.needsFollowUp) return "complete"
if (!checkTokenBudget(result)) return "budget_exceeded"
return "continue"
```

---

## Q3: How should doom loops be detected?

**A. Count identical tool calls (3x like OpenCode)** — Same tool + same args 3 times.
**B. Circuit breaker (3 failures like Claude Code)** — Any tool fails 3 times consecutively.
**C. Progress tracking** — Check if output changed from previous round.

**Recommendation: A + C hybrid**

- Primary: Count identical tool calls (3 consecutive with same tool + args)
- Secondary: Track output delta - if identical to previous turn for N rounds, force-stop

Source: OpenCode `DOOM_LOOP_THRESHOLD = 3` in `session/processor.ts:22`

---

## Q4: What retry strategy for API errors?

**A. Exponential backoff only** — Simple, but can retry forever (OpenCode issue #17648).
**B. Fixed max retries + backoff** — Cap at N attempts, then surface error.
**C. Circuit breaker pattern** — After failures, stop attempting for a period.

**Recommendation: B**

```
const MAX_RETRIES = 3
const MAX_BACKOFF_MS = 30000

for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
  try {
    return await processTurn()
  } catch (error) {
    if (!isRetryable(error)) throw error
    const delay = Math.min(INITIAL_DELAY * Math.pow(2, attempt), MAX_BACKOFF_MS)
    await sleep(delay)
  }
}
throw new Error("Max retries exceeded")
```

---

## Q5: Which errors should be retryable?

**A. All 5xx errors** — Server errors typically transient.
**B. SDK-provided isRetryable** — Trust the upstream SDK.
**C. Custom classification** — Override SDK for specific cases.

**Recommendation: C**

Based on OpenCode issue #19394: SDK may mark 5xx as non-retryable incorrectly.

```
function isRetryable(error) {
  if (error.statusCode >= 500 && error.statusCode < 600) return true  // 5xx always retry
  return error.isRetryable ?? false
}
```

---

## Q6: How to handle context overflow (413 errors)?

**A. Stop immediately** — Hard limit, surface error to user.
**B. Reactive compact (like Claude Code)** — Attempt compaction, retry.
**C. Both A + B** — Try compact first, then stop if fails.

**Recommendation: C**

Claude Code's reactive compact:
1. Trigger context collapse on 413
2. If succeeds, retry with compacted context
3. If fails again, stop

Source: query.ts reactive_compact_retry transition

---

## Q7: Should tool denials continue the loop?

**A. No (like Claude Code)** — Denied tool = stop.
**B. Yes (configurable)** — Allow via experimental flag.
**C. Ask user** — Prompt to decide.

**Recommendation: B**

```
const shouldBreak = (await Config.get()).experimental?.continue_loop_on_deny !== true
if (blocked && shouldBreak) return "stop"
continue
```

Source: OpenCode `session/processor.ts:51`

---

## Q8: How to persist session for crash recovery?

**A. Save after each turn** — Full state persisted.
**B. Save on checkpoint** — Periodic saves.
**C. Save on error only** — Recover failed states only.

**Recommendation: A**

```
interface SessionState {
  id: string
  messages: Message[]
  toolcalls: Record<string, ToolCall>
  turnCount: number
  lastCheckpoint: Date
}

async function saveSession(state: SessionState) {
  await Storage.write(["session", state.id], state)
}
```

---

## Q9: Should subagents have their own loop?

**A. Yes (fully isolated)** — Each subagent runs independently.
**B. No (shared loop)** — Subagent is a task, not a process.
**C. Hybrid** — Light subagents share, heavy ones isolate.

**Recommendation: B**

Subagent pattern (like OpenCode):
- Parent session manages context
- Subagent is a logical task, not separate process
- Results flow back to parent session

---

## Q10: What termination conditions for the loop?

**A. Single condition** — No more tool calls.
**B. Multiple conditions** — completed, budget_exceeded, error, stop_hook, etc.
**C. Customizable** — Configurable exit conditions.

**Recommendation: B**

| Reason | Trigger |
|-------|---------|
| completed | Normal completion (no tool calls) |
| budget_exceeded | Token budget hit |
| max_turns | Turn limit hit |
| error | Non-retryable error |
| stop_hook | User prevented continuation |
| stopped | Tool permission denied |

---

## Summary of Recommendations

| Priority | Feature | Source | Implementation |
|----------|---------|--------|----------------|
| P0 | while(true) loop | OpenCode | Core |
| P0 | Tool execution | OpenCode/Claude | Core |
| P0 | Retry with backoff | OpenCode | Core + MAX_RETRIES |
| P0 | Session persistence | OpenCode | Save after turn |
| P1 | Doom loop detection | OpenCode | 3x same tool |
| P1 | Context overflow | Claude Code | Reactive compact |
| P1 | Circuit breaker | Claude Code | After 3 failures |
| P1 | Token budget | Claude Code | Per-turn check |
| P1 | Termination reasons | OpenCode | Enum of states |
| P2 | Subagent support | All | Task-based not process |

---

## Files Referenced

- OpenCode: `third_party/opencode/packages/opencode/src/session/processor.ts`
- KiloCode: `third_party/kilocode/packages/opencode/src/session/processor.ts`
- Codex: `third_party/codex/codex-rs/core/src/codex.rs`
- GitHub Issues:
  - #30150: Agent loop non-termination
  - #17648: Session processor infinite retry
  - #19394: 5xx errors non-retryable