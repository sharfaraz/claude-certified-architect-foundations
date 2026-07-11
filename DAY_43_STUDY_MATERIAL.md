# Day 43: Context Management Deep-Dive — Token Monitoring and Checkpointing

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 5: Context Management & Reliability (15%) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Token Monitoring Fundamentals](#1-token-monitoring-fundamentals)
2. [Monitoring Strategies](#2-monitoring-strategies)
3. [Context Window Checkpointing](#3-context-window-checkpointing)
4. [Checkpoint Design Decisions](#4-checkpoint-design-decisions)
5. [Connecting to Session Management](#5-connecting-to-session-management)
6. [Practical Patterns](#6-practical-patterns)
7. [Degradation Strategies](#7-degradation-strategies)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude API — Token Counting | https://docs.anthropic.com/en/api/messages-count-tokens |
| Claude API — Messages | https://docs.anthropic.com/en/api/messages |
| Claude Docs — Long Context | https://docs.anthropic.com/en/docs/build-with-claude/long-context |
| Claude Agent SDK — Context | https://code.claude.com/docs/en/agent-sdk/context |

---

## 1. Token Monitoring Fundamentals

### Why Monitor Tokens?

Every Claude API call operates within a **context window** — the total tokens (input + output) the model can process. Exceeding it causes failures. Approaching it degrades quality.

### Token Types

| Token Type | What It Counts | Where It Comes From |
|-----------|----------------|---------------------|
| **Input tokens** | Everything you send TO Claude | System prompt + messages + tool definitions + tool results |
| **Output tokens** | Everything Claude returns | Response text + tool_use blocks |
| **Cumulative** | Running total across a session | All turns summed |

### The API Response Tells You

```json
{
  "usage": {
    "input_tokens": 2847,
    "output_tokens": 512
  }
}
```

### Context Window Sizes (Key Knowledge)

| Model | Context Window | Effective Budget (after system prompt) |
|-------|---------------|---------------------------------------|
| Claude 3.5 Sonnet | 200K tokens | ~180K usable |
| Claude 3 Opus | 200K tokens | ~180K usable |
| Claude 3 Haiku | 200K tokens | ~180K usable |

### The Critical Insight

Context window is SHARED between input and output. If you send 195K tokens as input, Claude has very little room to respond meaningfully.

```
|←────────── Context Window (200K) ──────────→|
|  Input Tokens  |  Output Tokens  |  Wasted  |
|   (you send)   |  (Claude returns)|          |
```

---

## 2. Monitoring Strategies

### Strategy 1: Per-Request Tracking

```python
class TokenMonitor:
    def __init__(self, context_limit=200000, warning_threshold=0.8):
        self.context_limit = context_limit
        self.warning_threshold = warning_threshold
        self.cumulative_input = 0
        self.cumulative_output = 0
        self.turn_count = 0
    
    def record_usage(self, response):
        """Track token usage from API response."""
        self.cumulative_input += response.usage.input_tokens
        self.cumulative_output += response.usage.output_tokens
        self.turn_count += 1
        
        # Check thresholds
        current_input = response.usage.input_tokens
        utilization = current_input / self.context_limit
        
        if utilization > self.warning_threshold:
            return "WARNING: approaching context limit"
        if utilization > 0.95:
            return "CRITICAL: must compress before next turn"
        return "OK"
```

### Strategy 2: Predictive Monitoring

Estimate how much the NEXT request will use before sending it.

```python
def estimate_next_request_tokens(messages, tools, system_prompt):
    """Predict token count for next API call."""
    # Approximate: 1 token ≈ 4 characters for English text
    estimated_input = (
        count_tokens(system_prompt) +
        sum(count_tokens(msg) for msg in messages) +
        sum(count_tokens(tool) for tool in tools)
    )
    return estimated_input

def should_compress_before_next_turn(monitor, estimated_next_input):
    """Decide if compression is needed."""
    remaining_budget = monitor.context_limit - estimated_next_input
    min_output_space = 4000  # Minimum tokens for a useful response
    return remaining_budget < min_output_space
```

### Strategy 3: Rolling Window Monitoring

Track the growth rate to predict when you'll hit limits.

```python
class RollingTokenMonitor:
    def __init__(self, context_limit=200000):
        self.context_limit = context_limit
        self.history = []  # (turn_number, input_tokens)
    
    def record_turn(self, turn_number, input_tokens):
        self.history.append((turn_number, input_tokens))
    
    def predict_turns_remaining(self):
        """Estimate how many more turns before hitting limit."""
        if len(self.history) < 2:
            return float('inf')
        
        # Calculate average growth rate
        growth_rates = [
            self.history[i][1] - self.history[i-1][1]
            for i in range(1, len(self.history))
        ]
        avg_growth = sum(growth_rates) / len(growth_rates)
        
        if avg_growth <= 0:
            return float('inf')
        
        current = self.history[-1][1]
        remaining_budget = self.context_limit - current
        return int(remaining_budget / avg_growth)
```

### Warning Thresholds

| Utilization | Status | Action |
|-------------|--------|--------|
| < 60% | Green | Continue normally |
| 60-80% | Yellow | Start planning compression |
| 80-90% | Orange | Compress older messages, summarize tool results |
| 90-95% | Red | Aggressive compression, checkpoint state |
| > 95% | Critical | Must compress or start new session |

---

## 3. Context Window Checkpointing

### What Is Checkpointing?

Saving the current conversation state at a specific point so you can:
1. **Recover** from failures without losing progress
2. **Branch** to explore alternatives (like `fork_session`)
3. **Resume** later (like `--resume`)
4. **Compress** old context while preserving a recovery point

### Checkpoint Anatomy

```python
@dataclass
class Checkpoint:
    # IDENTITY
    checkpoint_id: str
    timestamp: datetime
    turn_number: int
    
    # STATE
    messages: list[Message]          # Full conversation history
    system_prompt: str               # Current system prompt
    tool_definitions: list[Tool]     # Available tools
    
    # METADATA
    token_usage: TokenUsage          # Input + output at checkpoint time
    agent_state: dict                # Any custom state (task progress, etc.)
    
    # COMPRESSED SUMMARY
    summary_to_this_point: str       # Human-readable summary of progress
    key_decisions: list[str]         # Important decisions made so far
    pending_tasks: list[str]         # What still needs to be done
```

### When to Checkpoint

| Trigger | Rationale |
|---------|-----------|
| After completing a milestone | Natural save point |
| Before risky operations | Recovery point if it fails |
| At phase transitions | Between search → analysis → synthesis |
| Approaching token limits (80%) | Save before compression |
| Before human review gates | Preserve state while waiting |
| After expensive operations | Don't repeat costly work |

---

## 4. Checkpoint Design Decisions

### What to Save

| Component | Include? | Rationale |
|-----------|----------|-----------|
| Full message history | Yes (at checkpoint) | Complete state for recovery |
| System prompt | Yes | May change between phases |
| Tool results | Summarized form | Full results too large; summaries preserve meaning |
| Agent custom state | Yes | Task-specific progress |
| Token usage metrics | Yes | Know budget state at checkpoint |
| Pending tasks | Yes | Know what's left to do |

### What NOT to Save

| Component | Why Exclude |
|-----------|-------------|
| Raw API responses (full) | Too large; extract just usage and content |
| Intermediate reasoning | Captured in message history |
| Expired tool results | Data may be stale on resume |

### Checkpoint Storage Patterns

```python
# Pattern 1: In-memory (single session)
checkpoints = {}

def save_checkpoint(checkpoint_id, state):
    checkpoints[checkpoint_id] = {
        "messages": state.messages.copy(),
        "token_usage": state.token_usage,
        "summary": generate_summary(state),
        "timestamp": datetime.now()
    }

def restore_checkpoint(checkpoint_id):
    return checkpoints[checkpoint_id]

# Pattern 2: Persistent (cross-session)
def save_to_disk(checkpoint_id, state):
    with open(f"checkpoints/{checkpoint_id}.json", "w") as f:
        json.dump(serialize(state), f)

# Pattern 3: Compressed checkpoint (for long sessions)
def save_compressed_checkpoint(checkpoint_id, state):
    """Save summary instead of full history."""
    return {
        "summary": summarize_conversation(state.messages),
        "key_facts": extract_key_facts(state.messages),
        "pending_tasks": state.pending_tasks,
        "token_usage": state.token_usage
    }
```

---

## 5. Connecting to Session Management

### Relationship to Day 36 Concepts

| Day 36 Concept | Day 43 Extension |
|----------------|------------------|
| `--resume` | Resuming from a checkpoint |
| `fork_session` | Branching from a checkpoint |
| Session state | Checkpoint = snapshot of session state |
| Context preservation | Checkpoint + compression = long-running reliability |

### Session Management with Checkpointing

```python
class ManagedSession:
    def __init__(self, session_id, context_limit=200000):
        self.session_id = session_id
        self.messages = []
        self.checkpoints = []
        self.monitor = TokenMonitor(context_limit)
    
    def process_turn(self, user_message):
        self.messages.append(user_message)
        
        # Pre-turn check
        if self.monitor.should_compress():
            self.checkpoint_and_compress()
        
        # Make API call
        response = call_claude(self.messages)
        self.monitor.record_usage(response)
        self.messages.append(response.message)
        
        # Post-turn check
        if self.monitor.utilization > 0.8:
            self.save_checkpoint("auto_" + str(len(self.checkpoints)))
        
        return response
    
    def checkpoint_and_compress(self):
        """Save full state, then compress messages."""
        checkpoint_id = f"cp_{len(self.checkpoints)}"
        self.save_checkpoint(checkpoint_id)
        
        # Compress old messages into summary
        summary = summarize_messages(self.messages[:-5])  # Keep last 5 turns
        self.messages = [
            {"role": "user", "content": f"[Previous context summary: {summary}]"},
            *self.messages[-5:]  # Recent messages
        ]
```

### Fork from Checkpoint

```python
def fork_from_checkpoint(checkpoint_id, new_direction):
    """Create a new session branch from a saved checkpoint."""
    checkpoint = load_checkpoint(checkpoint_id)
    
    new_session = ManagedSession(
        session_id=f"fork_{checkpoint_id}_{uuid4()}",
        context_limit=checkpoint.context_limit
    )
    new_session.messages = checkpoint.messages.copy()
    new_session.process_turn(new_direction)
    
    return new_session
```

---

## 6. Practical Patterns

### Pattern 1: Token Counter with Alerts

```python
class TokenCounter:
    """Simple token counter for exam-relevant pattern."""
    
    def __init__(self, limit=200000):
        self.limit = limit
        self.current_input = 0
        self.alerts = []
    
    def update(self, api_response):
        self.current_input = api_response.usage.input_tokens
        utilization = self.current_input / self.limit
        
        if utilization > 0.9:
            self.alerts.append("CRITICAL: Compress immediately")
        elif utilization > 0.8:
            self.alerts.append("WARNING: Plan compression")
        
        return {
            "utilization": utilization,
            "tokens_remaining": self.limit - self.current_input,
            "alert": self.alerts[-1] if self.alerts else None
        }
```

### Pattern 2: Warning Threshold Trigger

```python
def check_and_act(monitor, messages):
    """Threshold-based action trigger."""
    status = monitor.get_status()
    
    if status == "CRITICAL":
        # Aggressive compression
        compressed = compress_all_but_last_3(messages)
        return compressed, "compressed_critical"
    
    elif status == "WARNING":
        # Moderate compression
        compressed = summarize_tool_results(messages)
        return compressed, "compressed_tools"
    
    elif status == "YELLOW":
        # Just checkpoint, no compression yet
        save_checkpoint("auto", messages)
        return messages, "checkpointed"
    
    return messages, "ok"
```

### Pattern 3: Compression Trigger in Agentic Loop

```python
async def agentic_loop(agent, task, max_turns=10):
    """Agentic loop with integrated token monitoring."""
    monitor = TokenMonitor(context_limit=200000, warning_threshold=0.8)
    messages = [{"role": "user", "content": task}]
    
    for turn in range(max_turns):
        # Check if we need to compress before this turn
        estimated = estimate_tokens(messages)
        if estimated > monitor.context_limit * 0.85:
            # Checkpoint before compressing
            save_checkpoint(f"turn_{turn}", messages)
            messages = compress_conversation(messages)
        
        # Make API call
        response = await call_claude(messages, agent.tools)
        monitor.record_usage(response)
        
        # Check stop condition
        if response.stop_reason == "end_turn":
            return response
        
        # Process tool use and continue
        messages = process_tool_results(messages, response)
    
    # Max turns reached — circuit breaker
    return {"status": "max_turns_reached", "last_checkpoint": f"turn_{turn}"}
```

---

## 7. Degradation Strategies

### Graceful Degradation When Approaching Limits

| Stage | Action | Quality Impact |
|-------|--------|---------------|
| Normal (< 60%) | Full context, no compression | Best quality |
| Caution (60-80%) | Summarize old tool results | Minimal quality loss |
| Warning (80-90%) | Compress all but last 5 turns | Some detail loss |
| Critical (90-95%) | Keep only summary + last 3 turns | Significant detail loss |
| Emergency (> 95%) | New session with checkpoint summary | Context break |

### Degradation Implementation

```python
def degrade_context(messages, utilization):
    """Progressive context degradation based on utilization."""
    
    if utilization < 0.6:
        return messages  # No degradation
    
    elif utilization < 0.8:
        # Level 1: Compress tool results to summaries
        return [
            compress_tool_results(msg) if msg.get("role") == "tool" else msg
            for msg in messages
        ]
    
    elif utilization < 0.9:
        # Level 2: Keep summary + recent turns only
        summary = summarize_conversation(messages[:-5])
        return [
            {"role": "system", "content": f"Context summary: {summary}"},
            *messages[-5:]
        ]
    
    else:
        # Level 3: Emergency — minimal context
        summary = summarize_conversation(messages[:-3])
        key_facts = extract_key_facts(messages)
        return [
            {"role": "system", "content": f"Summary: {summary}\nKey facts: {key_facts}"},
            *messages[-3:]
        ]
```

---

## 8. Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Pattern |
|-------------|--------------|-----------------|
| No token monitoring | Hits context limit unexpectedly, request fails | Track usage every turn |
| Confusing max_tokens with context limit | max_tokens is OUTPUT only; context is INPUT | Monitor input_tokens from usage |
| Compressing too early | Loses information unnecessarily | Compress at 80%+ utilization |
| Compressing too late | Request fails, no recovery | Set thresholds and automate |
| No checkpointing before compression | Can't recover full context if needed | Always checkpoint before compress |
| Storing full raw API responses | Checkpoints too large | Store messages + metadata + summary |
| Single checkpoint only | Can't branch or recover intermediate states | Checkpoint at milestones |
| Resuming without checking staleness | Data may have changed since checkpoint | Validate checkpoint relevance |

---

## 9. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│      TOKEN MONITORING & CHECKPOINTING                        │
├─────────────────────────────────────────────────────────────┤
│ TOKEN TYPES:                                                 │
│   input_tokens  → What you send (system + messages + tools)  │
│   output_tokens → What Claude returns                        │
│   Context window = max(input + output) per request           │
│                                                              │
│ MONITORING THRESHOLDS:                                       │
│   < 60%  → Green (normal)                                    │
│   60-80% → Yellow (plan compression)                         │
│   80-90% → Orange (compress now)                             │
│   > 90%  → Red (critical, emergency compress)                │
│                                                              │
│ CHECKPOINT CONTAINS:                                         │
│   messages + system_prompt + token_usage +                   │
│   agent_state + summary + pending_tasks                      │
│                                                              │
│ CHECKPOINT TRIGGERS:                                         │
│   • After milestones                                         │
│   • Before risky operations                                  │
│   • At phase transitions                                     │
│   • Approaching 80% utilization                              │
│                                                              │
│ DEGRADATION ORDER:                                           │
│   1. Compress tool results                                   │
│   2. Summarize old turns                                     │
│   3. Keep only summary + last 3-5 turns                     │
│   4. New session with checkpoint summary                     │
│                                                              │
│ EXAM KEY:                                                    │
│   max_tokens ≠ context window                                │
│   max_tokens controls OUTPUT length only                     │
│   Context window = total capacity for INPUT + OUTPUT         │
│   Monitor input_tokens from response.usage                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Practice Question

> An agentic system is analyzing a large codebase over multiple turns. After 12 turns, the token monitor shows 82% context utilization. The agent has completed analysis of 3 modules and has 2 remaining. What is the correct action?
>
> **A)** Continue without changes — 18% remaining should be enough for 2 more modules
>
> **B)** Checkpoint the current state, then compress the analyses of the completed 3 modules into structured summaries before continuing
>
> **C)** Start a completely new session and re-analyze everything from scratch
>
> **D)** Increase max_tokens to give the agent more space

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — At 82% utilization (orange zone), the correct action is to: (1) checkpoint for recovery safety, then (2) compress completed work into summaries. This frees space for the remaining 2 modules while preserving the completed analysis through compression.
- **A is wrong** — 18% remaining is ~36K tokens. Each module analysis adds context (code read + reasoning + results). Two more modules will likely push past 95%, causing quality degradation or failure.
- **C is wrong** — Starting over wastes 12 turns of completed work. Compression preserves the results without re-doing the analysis.
- **D is wrong** — `max_tokens` controls OUTPUT length, not context window capacity. This is the classic exam distractor.

**Key Concepts Tested:**
- Token monitoring thresholds (Domain 5)
- Checkpoint before compress (Domain 5)
- max_tokens misconception (Domain 4/5)
- Graceful degradation strategy (Domain 5)

</details>

### Bonus Practice Question

> A long-running agent session has hit 94% context utilization. The agent has a checkpoint saved at 75% utilization (turn 8). It's currently on turn 15. What is the best recovery strategy?
>
> **A)** Restore the checkpoint at turn 8 and redo turns 9-15
>
> **B)** Summarize turns 9-15 progress, then start a new session with the summary as initial context
>
> **C)** Compress everything before turn 12 into a summary, keeping turns 12-15 intact
>
> **D)** Increase the model's context window size

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: C**

**Explanation:**
- **C is correct** — Compressing older turns (before turn 12) while keeping recent turns (12-15) intact preserves the most relevant context. Recent turns contain the current working state and are most important for continuity. This is the standard "keep recent, summarize old" pattern.
- **A is wrong** — Restoring to turn 8 and redoing 7 turns wastes significant computation and API costs. The work from turns 9-15 should be preserved through compression, not discarded.
- **B is wrong** — Starting a completely new session loses the message continuity. Compression within the current session is less disruptive.
- **D is wrong** — Context window is a model property, not a parameter you can change per request.

</details>

---

## Key Takeaways for Exam Day

1. **max_tokens ≠ context window** — max_tokens limits output; context window is total capacity
2. **Monitor input_tokens from response.usage** — this tells you how much of the window you've used
3. **Checkpoint BEFORE compressing** — always save full state before you throw anything away
4. **Threshold-based actions** — 80% = compress, 90% = aggressive compress, 95% = new session
5. **Compression preserves: key findings + decisions + pending tasks** — discard verbose reasoning
6. **Graceful degradation** — reduce detail progressively, don't wait for catastrophic failure
