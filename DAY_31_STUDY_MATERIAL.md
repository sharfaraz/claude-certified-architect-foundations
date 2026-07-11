# Day 31: Agentic Loop Design Fundamentals

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Begin Introduction to Agent Skills |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [What Makes a System "Agentic"](#1-what-makes-a-system-agentic)
2. [The Agentic Loop](#2-the-agentic-loop)
3. [stop_reason in Agentic Contexts](#3-stop_reason-in-agentic-contexts)
4. [max_turns Guards](#4-max_turns-guards)
5. [Agent Definitions in Claude Agent SDK](#5-agent-definitions-in-claude-agent-sdk)
6. [Termination Conditions Design](#6-termination-conditions-design)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Agentic Tool Use Documentation | https://docs.anthropic.com/en/docs/build-with-claude/agentic-tool-use |
| Claude Agent SDK: Agent Loop | https://code.claude.com/docs/en/agent-sdk/agent-loop |

---

## 1. What Makes a System "Agentic"

### The Agentic Spectrum

Not every AI system is agentic. The exam tests whether you can distinguish between levels of autonomy:

| Level | Description | Example |
|-------|-------------|---------|
| **Static** | Single prompt → single response | Chatbot Q&A |
| **Tool-Augmented** | Single prompt → tool call → response | Calculator lookup |
| **Agentic** | Iterative reasoning with autonomous decisions | Code refactoring agent |
| **Multi-Agent** | Multiple agents collaborating | Coordinator + specialist agents |

### Three Pillars of Agentic Behavior

```
┌─────────────────────────────────────────────────┐
│              AGENTIC SYSTEM                      │
├─────────────────────────────────────────────────┤
│  1. AUTONOMOUS TOOL USE                         │
│     → Agent decides WHICH tools to call         │
│     → Agent decides WHEN to call them           │
│     → No human approval per-step                │
│                                                 │
│  2. ITERATIVE REASONING                         │
│     → Observe results, reason about next step   │
│     → Adapt strategy based on intermediate data │
│     → Self-correct on errors                    │
│                                                 │
│  3. GOAL-DIRECTED BEHAVIOR                      │
│     → Work toward a defined objective           │
│     → Decide when objective is met              │
│     → Know when to stop                         │
└─────────────────────────────────────────────────┘
```

### Key Exam Distinction

> **Tool-augmented ≠ Agentic.** A single tool call triggered by `tool_use` stop_reason is tool-augmented. An **agentic** system loops: it calls tools, observes results, reasons about what to do next, and repeats until the goal is met.

---

## 2. The Agentic Loop

### The Core Loop: Prompt → Reason → Act → Observe → Repeat/Stop

```
┌──────────────────────────────────────────────────────┐
│                  THE AGENTIC LOOP                     │
│                                                      │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│   │ PROMPT  │───▶│ REASON  │───▶│   ACT   │        │
│   └─────────┘    └─────────┘    └─────────┘        │
│        ▲                              │              │
│        │                              ▼              │
│   ┌─────────┐                   ┌─────────┐        │
│   │  STOP   │◀── decision ◀─────│ OBSERVE │        │
│   └─────────┘                   └─────────┘        │
│        │                              │              │
│        ▼                              │              │
│   [Final Response]          [Loop back to REASON]    │
└──────────────────────────────────────────────────────┘
```

### Phase Breakdown

| Phase | What Happens | API-Level Signal |
|-------|-------------|------------------|
| **Prompt** | User goal + system instructions enter the loop | Initial `messages` array |
| **Reason** | Model analyzes context, plans next action | Internal chain-of-thought |
| **Act** | Model calls a tool | `stop_reason: "tool_use"` |
| **Observe** | Tool result is appended to conversation | `tool_result` message added |
| **Repeat/Stop** | Model decides: more work needed or goal met | `stop_reason: "end_turn"` to exit |

### Implementation Pattern

```python
import anthropic

client = anthropic.Anthropic()

def agentic_loop(system_prompt: str, user_goal: str, tools: list, max_turns: int = 10):
    """Core agentic loop implementation."""
    messages = [{"role": "user", "content": user_goal}]
    
    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )
        
        # Check termination condition
        if response.stop_reason == "end_turn":
            return extract_text(response)  # Goal met — exit loop
        
        if response.stop_reason == "tool_use":
            # ACT: Extract tool calls
            tool_calls = [b for b in response.content if b.type == "tool_use"]
            
            # Append assistant response
            messages.append({"role": "assistant", "content": response.content})
            
            # OBSERVE: Execute tools and collect results
            tool_results = []
            for call in tool_calls:
                result = execute_tool(call.name, call.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": call.id,
                    "content": result
                })
            
            messages.append({"role": "user", "content": tool_results})
        else:
            # Unexpected stop_reason — exit with what we have
            return extract_text(response)
    
    # max_turns exceeded — circuit breaker triggered
    raise AgentLoopExhausted(f"Agent did not complete within {max_turns} turns")
```

---

## 3. stop_reason in Agentic Contexts

### stop_reason Values and Their Loop Semantics

| stop_reason | Meaning in Agentic Context | Loop Action |
|-------------|---------------------------|-------------|
| `"end_turn"` | Agent believes goal is complete | **Exit loop** — return response |
| `"tool_use"` | Agent needs to call a tool | **Continue loop** — execute tool, feed result back |
| `"max_tokens"` | Response truncated (ran out of tokens) | **Handle gracefully** — may need continuation |
| `"stop_sequence"` | Custom stop sequence hit | **Exit loop** — application-defined stopping |

### stop_reason as the Loop Driver

```python
# The stop_reason IS the loop control mechanism
while True:
    response = call_claude(messages)
    
    match response.stop_reason:
        case "tool_use":
            # Agent wants to act — continue loop
            process_tool_calls(response, messages)
        case "end_turn":
            # Agent is done — break loop
            break
        case "max_tokens":
            # Truncated — decide: retry or fail
            handle_truncation(response, messages)
        case _:
            # Unexpected — break with warning
            log_warning(f"Unexpected stop_reason: {response.stop_reason}")
            break
```

### Exam Tip

> The exam will test: "What drives the iteration in an agentic loop?" The answer is `stop_reason`. When it's `"tool_use"`, keep looping. When it's `"end_turn"`, the agent has decided the goal is met.

---

## 4. max_turns Guards

### Why max_turns Is Essential

Without an upper bound, an agentic loop can run indefinitely — consuming tokens, hitting rate limits, and potentially taking destructive actions in a loop.

### Guard Implementation Patterns

```python
# Pattern 1: Simple counter (most common)
MAX_TURNS = 25

for turn in range(MAX_TURNS):
    response = call_claude(messages)
    if response.stop_reason == "end_turn":
        return response
    # ... process tool calls

raise LoopExhausted("Max turns reached")

# Pattern 2: Cost-based guard
MAX_COST_USD = 5.00
total_cost = 0.0

while total_cost < MAX_COST_USD:
    response = call_claude(messages)
    total_cost += calculate_cost(response)
    if response.stop_reason == "end_turn":
        return response
    # ... process tool calls

raise CostLimitExceeded(f"Spent ${total_cost:.2f}")

# Pattern 3: Time-based guard
import time
MAX_DURATION_SECONDS = 300  # 5 minutes
start_time = time.time()

while (time.time() - start_time) < MAX_DURATION_SECONDS:
    response = call_claude(messages)
    if response.stop_reason == "end_turn":
        return response
    # ...

raise TimeoutExceeded("Agent loop timed out")
```

### Choosing max_turns Values

| Task Type | Recommended max_turns | Rationale |
|-----------|----------------------|-----------|
| Simple lookup | 3–5 | One search + one refinement |
| Code generation | 10–15 | Write + test + fix cycle |
| Complex research | 20–30 | Multiple searches + synthesis |
| Open-ended exploration | 30–50 | Many branches, but still bounded |

---

## 5. Agent Definitions in Claude Agent SDK

### Defining an Agent

```python
from claude_code_sdk import Agent, AgentConfig

# Basic agent definition
agent = Agent(
    name="code-reviewer",
    model="claude-sonnet-4-20250514",
    instructions="You are a code review agent. Analyze code for bugs, "
                 "style issues, and security vulnerabilities.",
    tools=["read_file", "grep_search", "list_directory"],
    max_turns=15
)
```

### AgentConfig Properties

| Property | Purpose | Exam Relevance |
|----------|---------|----------------|
| `name` | Identifier for the agent | Used in multi-agent routing |
| `model` | Which Claude model to use | Cost/capability tradeoff |
| `instructions` | System prompt for the agent | Defines behavior boundaries |
| `tools` | Allowed tool list | Security: principle of least privilege |
| `max_turns` | Loop iteration limit | Safety: circuit breaker |

### Running an Agent

```python
# Execute the agent with a task
result = await agent.run(
    task="Review the file src/auth.py for security vulnerabilities",
    context={"repo_path": "/home/user/project"}
)

# The SDK manages the agentic loop internally:
# 1. Sends task as user message
# 2. Loops on tool_use stop_reason
# 3. Exits on end_turn or max_turns
# 4. Returns final response
```

---

## 6. Termination Conditions Design

### Termination Taxonomy

```
Termination Conditions:
├── Natural Completion
│   └── stop_reason == "end_turn" (agent decides goal is met)
├── Safety Limits
│   ├── max_turns exceeded
│   ├── max_cost exceeded
│   └── max_time exceeded
├── Error Conditions
│   ├── Consecutive tool failures > threshold
│   ├── Unrecoverable error detected
│   └── Infinite loop detected (same tool call repeated)
└── External Signals
    ├── User cancellation
    ├── System shutdown
    └── Upstream timeout
```

### Designing Robust Termination

```python
class TerminationConfig:
    max_turns: int = 25
    max_consecutive_errors: int = 3
    max_cost_usd: float = 2.00
    max_duration_seconds: int = 300
    repeated_action_threshold: int = 3  # Same tool+args N times = stuck

class AgentLoop:
    def __init__(self, config: TerminationConfig):
        self.config = config
        self.turn_count = 0
        self.consecutive_errors = 0
        self.total_cost = 0.0
        self.action_history = []
    
    def should_terminate(self) -> tuple[bool, str]:
        """Check all termination conditions."""
        if self.turn_count >= self.config.max_turns:
            return True, "max_turns_exceeded"
        if self.consecutive_errors >= self.config.max_consecutive_errors:
            return True, "consecutive_errors"
        if self.total_cost >= self.config.max_cost_usd:
            return True, "cost_limit"
        if self._detect_stuck_loop():
            return True, "stuck_loop_detected"
        return False, ""
    
    def _detect_stuck_loop(self) -> bool:
        """Detect if agent is repeating the same action."""
        if len(self.action_history) < self.config.repeated_action_threshold:
            return False
        recent = self.action_history[-self.config.repeated_action_threshold:]
        return len(set(str(a) for a in recent)) == 1
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Loops Without Circuit Breakers

```python
# ❌ DANGEROUS: No termination guard
def naive_agent_loop(messages, tools):
    while True:  # <-- NO EXIT CONDITION besides end_turn
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            tools=tools,
            messages=messages
        )
        if response.stop_reason == "end_turn":
            return response
        # What if the model NEVER returns end_turn?
        # What if a tool keeps failing and the model keeps retrying?
        process_tool_calls(response, messages)
```

**Why it's dangerous:**
- Model may never decide the goal is met
- Tool errors can create infinite retry loops
- Unbounded token consumption
- No cost or time controls

### ✅ CORRECT: Always Include Circuit Breakers

```python
# ✅ SAFE: Multiple termination conditions
def safe_agent_loop(messages, tools, max_turns=25, max_errors=3):
    errors = 0
    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            tools=tools,
            messages=messages
        )
        if response.stop_reason == "end_turn":
            return response
        
        try:
            process_tool_calls(response, messages)
            errors = 0  # Reset on success
        except ToolExecutionError as e:
            errors += 1
            if errors >= max_errors:
                raise CircuitBreakerTripped(f"Failed {errors} times consecutively")
            # Feed error back to model for self-correction
            messages.append(error_as_tool_result(e, response))
    
    raise LoopExhausted(f"Did not complete in {max_turns} turns")
```

### Additional Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| No max_turns | Infinite loops possible | Always set upper bound |
| Ignoring max_tokens stop_reason | Truncated responses treated as complete | Handle truncation explicitly |
| No error counting | Infinite retry on broken tools | Track consecutive failures |
| Silent failure on loop exhaustion | User gets no feedback | Log + escalate on termination |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│          AGENTIC LOOP DESIGN — QUICK REFERENCE          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  AGENTIC = Autonomous Tool Use                          │
│          + Iterative Reasoning                          │
│          + Goal-Directed Behavior                       │
│                                                         │
│  THE LOOP: Prompt → Reason → Act → Observe → Repeat    │
│                                                         │
│  LOOP DRIVER: stop_reason                               │
│    "tool_use"   → continue looping                      │
│    "end_turn"   → exit (goal met)                       │
│    "max_tokens" → handle truncation                     │
│                                                         │
│  CIRCUIT BREAKERS (always include!):                    │
│    • max_turns (iteration cap)                          │
│    • max_cost (spend limit)                             │
│    • max_errors (consecutive failure cap)               │
│    • stuck detection (repeated identical actions)       │
│                                                         │
│  SDK AGENT DEFINITION:                                  │
│    Agent(name, model, instructions, tools, max_turns)   │
│                                                         │
│  EXAM TIP: 27% of exam is Domain 1 — know this cold!   │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You're designing an agent that researches competitor pricing by searching the web, extracting data from pages, and compiling a comparison table. The task typically requires 5-8 tool calls but sometimes requires up to 20 if sources are sparse.

**Question 1:** What is the minimum set of termination conditions you should implement?

- A) Only `max_turns = 20`
- B) `max_turns = 25`, consecutive error limit of 3, and cost cap
- C) Only `stop_reason == "end_turn"` check
- D) Time-based timeout of 60 seconds only

**Answer:** **B** — A robust agent needs multiple circuit breakers. `max_turns` prevents infinite loops, consecutive error limits catch broken tools, and cost caps prevent runaway spending. Option A lacks error handling, C has no safety limits at all, D ignores iteration and cost.

---

**Question 2:** Your agent keeps calling the same `web_search` tool with identical parameters. What termination condition should catch this?

- A) max_turns — it will eventually exhaust turns
- B) Stuck loop detection — same action repeated N times
- C) Cost cap — it will eventually exceed the budget
- D) Time-based timeout

**Answer:** **B** — Stuck loop detection specifically identifies when the agent is repeating identical actions, providing an early exit with a descriptive reason. While A and C would eventually trigger, they waste resources. B catches the problem immediately.

---

**Question 3:** In the agentic loop, what API response field drives the decision to continue iterating or stop?

- A) `response.content[0].text`
- B) `response.stop_reason`
- C) `response.usage.output_tokens`
- D) `response.model`

**Answer:** **B** — `stop_reason` is the loop control signal. `"tool_use"` means continue, `"end_turn"` means stop. The other fields provide information but don't drive loop control.

---

**Question 4:** What distinguishes a "tool-augmented" system from an "agentic" system?

- A) Tool-augmented systems use more expensive models
- B) Agentic systems never use tools
- C) Agentic systems iterate autonomously; tool-augmented systems do single tool calls
- D) Tool-augmented systems require MCP; agentic systems don't

**Answer:** **C** — The key distinction is iteration and autonomy. A tool-augmented system makes one tool call and returns. An agentic system loops: calling tools, observing results, reasoning about next steps, and deciding when the goal is met.
