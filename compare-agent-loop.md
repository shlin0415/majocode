# Compare Agent Loop: How Code Agents Continue, Handle Errors, and Restart

> Analysis based on third_party source code inspection and web research.
> Date: 2026-04-13

---

## 1. Executive Summary

| Agent | Loop Type | Continue Mechanism | Error Handling | Restart Capability |
|-------|-----------|---------------------|---------------|---------------------|
| Claude Code | async generator | Step/budget limits | Circuit breaker, reactive compact | No automatic restart |
| Codex | Rust async loop | Turn limits, token budgets | Error withholding, fallback | Depends on sandbox |
| OpenCode | while(true) + Effect | Doom loop detection, compaction | Retry with exponential backoff | Via session recovery |
| KiloCode | while(true) + Effect | Doom loop detection | Same as OpenCode | Via session recovery |
| iFlow | Node.js CLI | Mode-based (yolo/plan) | Limited documentation | Unknown |

---

## 2. Agent Loop Patterns

### 2.1 OpenCode / KiloCode (TypeScript + Bun)

**Core Loop in `session/processor.ts`:**

```typescript
async process(streamInput: LLM.StreamInput) {
  while (true) {
    try {
      const stream = await LLM.stream(streamInput)
      for await (const value of stream.fullStream) {
        // Handle: start, reasoning-*, tool-*, error, etc.
        switch (value.type) {
          case "tool-call": {
            // Doom loop detection
            const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)
            if (lastThree.length === DOOM_LOOP_THRESHOLD && 
                lastThree.every(p => p.tool === value.toolName && ...)) {
              await Permission.ask({ permission: "doom_loop", ... })
            }
            break
          }
          case "error":
            throw value.error
        }
      }
    } catch (e) {
      const retry = SessionRetry.retryable(error)
      if (retry !== undefined) {
        attempt++
        const delay = SessionRetry.delay(attempt, error)
        await SessionRetry.sleep(delay, input.abort)
        continue  // Continue the loop
      }
      input.assistantMessage.error = error
      return "stop"
    }
  }
}
```

**Key Features:**
- `DOOM_LOOP_THRESHOLD = 3`: Detect when same tool called 3 times consecutively
- Retry with exponential backoff via `SessionRetry`
- Context compaction when hitting token limits
- Continue loop config: `experimental.continue_loop_on_deny`

### 2.2 Claude Code (TypeScript/Python)

**Core Loop in `query.ts` (async generator):**

```typescript
async function* queryLoop() {
  const state = {
    messages: [],
    context: {},
    turnCount: 0,
    transition: undefined,
  }
  
  while (true) {
    // Context preparation (5 stages)
    const input = await prepareContext(state)
    
    // API call with fallback loop
    const result = await withRetry(input, config)
    
    // Tool execution
    for (const toolCall of result.toolCalls) {
      const toolResult = await executeTool(toolCall)
      
      // Check for doom loop (circuit breaker)
      if (isDoomLoop(toolCall, history)) {
        await Permission.ask({ permission: "doom_loop", ... })
      }
      
      state.messages.push(toolResult)
    }
    
    // Continue or break
    if (!result.needsFollowUp) {
      return checkTokenBudget(result)
    }
  }
}
```

**Key Features:**
- Circuit breaker: After 3 consecutive failures, stops trying
- Reactive compact: Triggered on 413 errors
- MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
- Token budget enforcement per turn

### 2.3 Codex (Rust)

**Core Loop in `codex.rs` (handle_responses):**

```rust
let outcome: CodexResult<SamplingRequestResult> = loop {
    let event = match stream.next().or_cancel(&cancellation_token).await {
        Ok(event) => event,
        Err(CancelErr::Cancelled) => break Err(CodexErr::TurnAborted),
    };
    
    match event {
        ResponseEvent::Created => {}
        ResponseEvent::OutputItemDone(item) => {
            let output_result = handle_output_item_done(&mut ctx, item, previously_active_item).await?;
            if let Some(tool_future) = output_result.tool_future {
                in_flight.push_back(tool_future);
            }
            needs_follow_up |= output_result.needs_follow_up;
        }
        ResponseEvent::Error(e) => {
            // Error handling with possible retry
            if is_retryable(&e) {
                continue; // Retry the turn
            }
            break Err(e);
        }
    }
    
    // Check termination conditions
    if !needs_follow_up {
        break Ok(final_result);
    }
};
```

**Key Features:**
- Rust async/await with tokio
- Tool execution with in_flight tracking
- Error withholding: Certain errors pushed to assistantMessages only
- Model fallback via inner loop

---

## 3. Error Handling Deep Dive

### 3.1 Retry Mechanisms

**OpenCode (`session/retry.ts`):**

```typescript
export const retry = (fn, options) => {
  const { attempts = 3, delay = 500, factor = 2, maxDelay = 10000 } = options
  for (let attempt = 0; attempt < attempts; attempt++) {
    try {
      return await fn()
    } catch (error) {
      if (attempt === attempts - 1 || !retryIf(error)) throw error
      await sleep(delay * Math.pow(factor, attempt))
    }
  }
}
```

**Exponential Backoff:**
- Initial delay: 500ms
- Backoff factor: 2
- Max delay: 10s (or 30s without retry-after header)
- Issue: No max attempts in some versions → infinite retry

**Claude Code (`withRetry.ts`):**

| Category | Status Codes | Behavior |
|----------|-------------|-----------|
| Rate limit | 429 | Retry with retry-after header |
| Server error | 500-599 | Retry with backoff (varies by SDK) |
| Overloaded | 529 | Special retry path |
| Timeout | 408 | Retry up to 3 times |

### 3.2 Error Classification

**Claude Code Error Categories:**

| Error | Code | Retryable | Recovery |
|------|------|-----------|-----------|
| APIError | 400+ | Depends on type | Context collapse |
| RateLimitError | 429 | Yes | Wait + retry |
| TimeoutError | 408 | Yes | Retry with backoff |
| ContextOverflowError | 413 | Yes | Reactive compact |
| PermissionDeniedError | - | No | Surface to user |

**OpenCode Error Types:**

```typescript
export const ErrorTypes = {
  APIError: { isRetryable: true },
  ContextOverflowError: { isRetryable: false, recoverable: true },
  PermissionError: { isRetryable: false },
  TimeoutError: { isRetryable: true },
  Unknown: { isRetryable: false },
}
```

### 3.3 Doom Loop Detection

**OpenCode Implementation:**

```typescript
const DOOM_LOOP_THRESHOLD = 3

// In tool-call handler:
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)
if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(p => 
    p.type === "tool" &&
    p.tool === value.toolName &&
    p.state.status !== "pending" &&
    JSON.stringify(p.state.input) === JSON.stringify(value.input)
  )
) {
  // Prompt for doom loop permission
  await Permission.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    metadata: { tool: value.toolName, input: value.input }
  })
}
```

**Claude Code Circuit Breaker:**
- After 3 identical tool calls in a row → inject system message
- Force stop if K > threshold with diagnostic
- Progress metric: Track delta between rounds

---

## 4. Continue / Restart Mechanisms

### 4.1 Session Recovery

**OpenCode Session Structure:**

```typescript
interface Session {
  id: string
  messages: Message[]
  toolcalls: Record<string, ToolCall>
  snapshot?: string
  status: "active" | "completed" | "failed"
}
```

**Recovery Points:**
1. Save checkpoint after each turn completion
2. If crash, resume from last checkpoint
3. Session stored in SQLite (opencode) or JSON (kilocode)

### 4.2 Subagent Restart

**OpenCode Subagent Pattern:**

```typescript
async function createSubagent(input) {
  const session = await Session.create({
    title: `Subagent: ${input.agentName}`,
    parentID: input.parentSession,
  })
  
  const processor = await SessionProcessor.create({
    sessionID: session.id,
    model: input.model,
    abort: input.abort,
  })
  
  // Process with own loop
  while (true) {
    const result = await processor.process(streamInput)
    if (result === "stop") break
    if (result === "compact") {
      // Compact context and continue
    }
  }
}
```

### 4.3 Crash Recovery

**Stateless Restart:**
- Session state persisted to disk
- On restart, reload last N messages
- Re-execute in-progress tools from state

**Limitations:**
- No automatic restart on crash (depends on launcher)
- In-progress tool calls may be lost
- Manual resume via `--resume` or `/chat` commands

---

## 5. Comparison Summary

### 5.1 Continue Mechanism

| Agent | Continue on Deny | Max Turns | Token Budget | Doom Loop |
|-------|------------------|----------|-------------|------------|
| Claude Code | No | Configurable | Yes | Detection + circuit |
| Codex | Unknown | Yes | Yes | Unknown |
| OpenCode | Configurable | No limit | Per-turn check | Permission prompt |
| KiloCode | Configurable | No limit | Per-turn check | Permission prompt |
| iFlow | Unknown | Unknown | Unknown | Unknown |

### 5.2 Error Recovery

| Agent | Retry | Exponential Backoff | Circuit Breaker | Max Retries |
|-------|-------|---------------------|-----------------|------------|
| Claude Code | Yes | Yes (3x) | Yes (3 failures) | 3 |
| Codex | Yes | Yes | Yes | Unknown |
| OpenCode | Yes | Yes | No (issue #17648) | Unlimited |
| KiloCode | Yes | Yes | No | Unlimited |
| iFlow | Unknown | Unknown | Unknown | Unknown |

### 5.3 Restart Capability

| Agent | Crash Restart | Subagent | Session Resume |
|-------|---------------|----------|----------------|
| Claude Code | Manual | Yes | Yes (session_id) |
| Codex | Unknown | Yes | Unknown |
| OpenCode | Manual | Yes | Yes |
| KiloCode | Manual | Yes | Yes |
| iFlow | Unknown | Yes | Yes |

---

## 6. Best Practices for MajoCode

Based on this analysis:

### 6.1 Recommended Loop Pattern

```typescript
async function agentLoop(input) {
  const MAX_TURNS = 100
  const MAX_RETRIES = 3
  const DOOM_THRESHOLD = 3
  
  for (let turn = 0; turn < MAX_TURNS; turn++) {
    try {
      const result = await processTurn(input)
      
      if (result.needsFollowUp) {
        checkDoomLoop(result)
        checkTokenBudget(result)
        continue
      }
      
      return result // Completed
    } catch (error) {
      if (isRetryable(error) && turn < MAX_RETRIES) {
        const delay = calculateBackoff(turn)
        await sleep(delay)
        continue
      }
      
      // Non-retryable or max retries exceeded
      await saveSessionState()
      throw error
    }
  }
}
```

### 6.2 Required Safety Features

1. **Circuit Breaker:** Stop after N consecutive failures
2. **Doom Loop Detection:** Detect repeated tool calls
3. **Max Retries:** Cap exponential backoff
4. **Token Budget:** Per-turn budget enforcement
5. **Session Persistence:** Save state for crash recovery

### 6.3 Implementation Priority

| Priority | Feature | Source |
|----------|---------|--------|
| P0 | Basic while(true) loop | OpenCode |
| P0 | Tool execution | OpenCode/Claude |
| P0 | Error classification | Claude Code |
| P1 | Retry with backoff | OpenCode |
| P1 | Doom loop detection | OpenCode |
| P1 | Session persistence | OpenCode |
| P2 | Reactive compact | Claude Code |
| P2 | Circuit breaker | Claude Code |
| P3 | Subagent support | All |

---

## 7. References

- OpenCode: `third_party/opencode/packages/opencode/src/session/processor.ts`
- KiloCode: `third_party/kilocode/packages/opencode/src/session/processor.ts`
- Claude Code: Web research - query.ts loop architecture
- Codex: `third_party/codex/codex-rs/core/src/codex.rs` (handle_responses)
- GitHub Issues:
  - #30150: Agent loop non-termination (Claude Code)
  - #17648: Session processor retries indefinitely (OpenCode)
  - #19394: 5xx errors incorrectly marked non-retryable (OpenCode)