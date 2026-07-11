# Day 37: Escalation Patterns and Error Propagation

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Begin Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Escalation Patterns Overview](#1-escalation-patterns-overview)
2. [Designing Escalation Triggers](#2-designing-escalation-triggers)
3. [Error Propagation in Multi-Agent Systems](#3-error-propagation-in-multi-agent-systems)
4. [Error-Context Feedback](#4-error-context-feedback)
5. [Circuit Breaker Pattern](#5-circuit-breaker-pattern)
6. [Connecting to Sprint 1: escalate_to_human](#6-connecting-to-sprint-1-escalate_to_human)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Agentic Tool Use | https://docs.anthropic.com/en/docs/build-with-claude/agentic-tool-use |
| Claude SDK: Subagents | https://code.claude.com/docs/en/sdk/subagents |

---

## 1. Escalation Patterns Overview

### What Is Escalation?

Escalation is the process of **handing off a problem to a higher authority** when an agent cannot resolve it autonomously. The "higher authority" can be:

- A human operator
- A higher-level coordinator agent
- A fallback system or service

```
┌─────────────────────────────────────────────────────────┐
│                ESCALATION HIERARCHY                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Level 0: Subagent (specialist execution)               │
│     │ ── can't solve → escalate up                      │
│     ▼                                                   │
│  Level 1: Coordinator (orchestration & retry)           │
│     │ ── can't solve → escalate up                      │
│     ▼                                                   │
│  Level 2: Human Operator (judgment & override)          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Why Escalation Matters

| Without Escalation | With Escalation |
|-------------------|-----------------|
| Agent retries forever | Agent knows when to stop |
| Errors compound silently | Errors surface to appropriate handler |
| No human safety net | Human can intervene on complex issues |
| Costs accumulate blindly | Cost limits trigger escalation |
| Agent makes bad decisions in ambiguity | Ambiguity triggers human judgment |

---

## 2. Designing Escalation Triggers

### Trigger Categories

```
┌─────────────────────────────────────────────────────────┐
│              ESCALATION TRIGGER TAXONOMY                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. CONFIDENCE THRESHOLDS                               │
│     "I'm not sure enough to proceed"                    │
│     • Model expresses uncertainty                       │
│     • Multiple conflicting interpretations              │
│     • Ambiguous user intent                             │
│                                                         │
│  2. POLICY BOUNDARIES                                   │
│     "This exceeds my authorized scope"                  │
│     • Action would modify production                    │
│     • Request involves sensitive data                   │
│     • Cost would exceed budget                          │
│                                                         │
│  3. ERROR COUNTS                                        │
│     "I've failed too many times"                        │
│     • N consecutive tool failures                       │
│     • Same error repeated despite different approaches  │
│     • Stuck in a loop                                   │
│                                                         │
│  4. AMBIGUOUS INPUTS                                    │
│     "I don't understand what's being asked"             │
│     • Contradictory requirements                        │
│     • Insufficient context to proceed                   │
│     • Multiple valid interpretations                    │
└─────────────────────────────────────────────────────────┘
```

### Trigger Implementation

```python
from enum import Enum
from dataclasses import dataclass

class EscalationReason(Enum):
    CONFIDENCE_LOW = "confidence_below_threshold"
    POLICY_VIOLATION = "action_exceeds_policy"
    ERROR_THRESHOLD = "consecutive_errors_exceeded"
    AMBIGUOUS_INPUT = "cannot_determine_intent"
    COST_LIMIT = "cost_budget_approaching"
    TIMEOUT = "execution_time_exceeded"
    STUCK_LOOP = "repeated_identical_actions"

@dataclass
class EscalationTrigger:
    reason: EscalationReason
    context: str
    severity: str  # "low", "medium", "high", "critical"
    suggested_action: str

class EscalationManager:
    def __init__(self, config: dict):
        self.max_consecutive_errors = config.get("max_errors", 3)
        self.cost_threshold = config.get("max_cost", 5.00)
        self.confidence_threshold = config.get("min_confidence", 0.7)
        self.max_identical_actions = config.get("max_repeats", 3)
    
    def check_triggers(self, agent_state: dict) -> EscalationTrigger | None:
        """Check all escalation conditions."""
        
        # Check consecutive errors
        if agent_state["consecutive_errors"] >= self.max_consecutive_errors:
            return EscalationTrigger(
                reason=EscalationReason.ERROR_THRESHOLD,
                context=f"Failed {agent_state['consecutive_errors']} times. "
                        f"Last error: {agent_state['last_error']}",
                severity="high",
                suggested_action="Human review of task feasibility"
            )
        
        # Check cost limits
        if agent_state["total_cost"] >= self.cost_threshold * 0.9:
            return EscalationTrigger(
                reason=EscalationReason.COST_LIMIT,
                context=f"Spent ${agent_state['total_cost']:.2f} "
                        f"of ${self.cost_threshold:.2f} budget",
                severity="medium",
                suggested_action="Approve additional budget or terminate"
            )
        
        # Check stuck loop
        if self._is_stuck(agent_state["action_history"]):
            return EscalationTrigger(
                reason=EscalationReason.STUCK_LOOP,
                context="Agent repeating same action with same parameters",
                severity="high",
                suggested_action="Review task instructions or provide guidance"
            )
        
        return None  # No escalation needed
    
    def _is_stuck(self, history: list) -> bool:
        if len(history) < self.max_identical_actions:
            return False
        recent = history[-self.max_identical_actions:]
        return len(set(str(a) for a in recent)) == 1
```

---

## 3. Error Propagation in Multi-Agent Systems

### The Error Propagation Chain

```
Subagent Error → Coordinator Receives → Coordinator Decides → User Notified
                                             │
                                    ┌────────┼────────┐
                                    │        │        │
                                  Retry   Reassign  Escalate
                                  (same)  (different (to human)
                                           approach)
```

### Error Propagation Implementation

```python
class ErrorPropagation:
    """Handles error flow from subagent to coordinator to user."""
    
    def handle_subagent_error(
        self, 
        error: Exception, 
        subagent_name: str,
        task_description: str,
        retry_count: int
    ) -> dict:
        """Determine how to handle a subagent error."""
        
        error_info = {
            "source": subagent_name,
            "task": task_description,
            "error_type": type(error).__name__,
            "error_message": str(error),
            "retry_count": retry_count
        }
        
        # Decision logic
        if retry_count < 2 and self._is_transient(error):
            return {
                "action": "retry",
                "strategy": "same_approach",
                "error_info": error_info,
                "feedback": f"Transient error, retrying: {error}"
            }
        
        elif retry_count < 3 and self._is_fixable(error):
            return {
                "action": "retry",
                "strategy": "different_approach",
                "error_info": error_info,
                "feedback": f"Adjusting approach due to: {error}"
            }
        
        else:
            return {
                "action": "escalate",
                "target": "human",
                "error_info": error_info,
                "feedback": f"Cannot resolve after {retry_count} attempts"
            }
    
    def _is_transient(self, error: Exception) -> bool:
        """Is this a temporary error that might succeed on retry?"""
        transient_types = ["TimeoutError", "RateLimitError", "ConnectionError"]
        return type(error).__name__ in transient_types
    
    def _is_fixable(self, error: Exception) -> bool:
        """Can this error be fixed with a different approach?"""
        fixable_patterns = ["not found", "permission denied", "invalid format"]
        return any(p in str(error).lower() for p in fixable_patterns)
```

### Error Context Enrichment

```python
def enrich_error_for_coordinator(
    error: Exception,
    subagent_context: dict
) -> str:
    """Add context to error before propagating to coordinator."""
    return f"""
## Subagent Error Report

**Agent:** {subagent_context['agent_name']}
**Task:** {subagent_context['task_description']}
**Turn:** {subagent_context['turn_number']} of {subagent_context['max_turns']}
**Error Type:** {type(error).__name__}
**Error Message:** {str(error)}

**Last Action Attempted:**
  Tool: {subagent_context.get('last_tool_called', 'unknown')}
  Input: {subagent_context.get('last_tool_input', 'unknown')}

**Partial Results (if any):**
{subagent_context.get('partial_results', 'None')}

**Suggested Next Steps:**
1. Retry with modified parameters
2. Try alternative approach
3. Escalate to human for guidance
"""
```

---

## 4. Error-Context Feedback

### The Feedback Loop Pattern

When an agent encounters an error, **feeding the error details back** into the next attempt helps the agent self-correct.

```python
def retry_with_error_feedback(
    client, 
    messages: list, 
    failed_response, 
    error: str
) -> dict:
    """Feed error information back to the model for self-correction."""
    
    # Append the failed assistant response
    messages.append({
        "role": "assistant",
        "content": failed_response.content
    })
    
    # Append error as a tool_result (so model sees what went wrong)
    tool_call = next(b for b in failed_response.content if b.type == "tool_use")
    messages.append({
        "role": "user",
        "content": [{
            "type": "tool_result",
            "tool_use_id": tool_call.id,
            "is_error": True,  # Signal that this is an error
            "content": f"Error: {error}\n\n"
                       f"Please try a different approach. "
                       f"Consider what went wrong and adjust your strategy."
        }]
    })
    
    # Next model call will see the error and can self-correct
    return client.messages.create(
        model="claude-sonnet-4-20250514",
        messages=messages,
        tools=tools
    )
```

### Error Feedback Quality

| Poor Feedback | Good Feedback |
|--------------|---------------|
| `"Error occurred"` | `"FileNotFoundError: /src/auth.py does not exist. Available files in /src/: main.py, utils.py, config.py"` |
| `"Try again"` | `"The SQL query failed because table 'users' doesn't exist. The correct table name is 'app_users'. Try using that."` |
| `"Permission denied"` | `"Permission denied for /etc/prod.conf. You only have access to files in /home/agent/workspace/. Try an alternative path."` |

### Self-Correction Pattern

```python
class SelfCorrectingAgent:
    """Agent that uses error feedback to improve its approach."""
    
    def __init__(self, max_retries: int = 3):
        self.max_retries = max_retries
        self.error_history = []
    
    async def execute_with_correction(self, task, tools, messages):
        for attempt in range(self.max_retries):
            response = await self.call_model(messages, tools)
            
            if response.stop_reason == "end_turn":
                return response  # Success!
            
            if response.stop_reason == "tool_use":
                try:
                    result = await self.execute_tool(response)
                    messages = self.append_success(messages, response, result)
                    self.error_history = []  # Reset on success
                except ToolError as e:
                    self.error_history.append(str(e))
                    
                    # Feed error back WITH context about prior errors
                    feedback = self._build_feedback(e)
                    messages = self.append_error(messages, response, feedback)
        
        # All retries exhausted — escalate
        raise EscalationRequired(
            f"Failed after {self.max_retries} attempts. "
            f"Errors: {self.error_history}"
        )
    
    def _build_feedback(self, error: ToolError) -> str:
        """Build rich feedback including error history."""
        feedback = f"Error: {error}\n"
        
        if len(self.error_history) > 1:
            feedback += f"\nPrevious errors in this task:\n"
            for i, prev_error in enumerate(self.error_history[:-1]):
                feedback += f"  Attempt {i+1}: {prev_error}\n"
            feedback += f"\nYou've tried {len(self.error_history)} approaches. "
            feedback += "Try something fundamentally different."
        
        return feedback
```

---

## 5. Circuit Breaker Pattern

### What Is a Circuit Breaker?

A circuit breaker **stops execution after N consecutive failures**, preventing resource waste and cascading errors.

```
┌─────────────────────────────────────────────────────────┐
│              CIRCUIT BREAKER STATES                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  CLOSED (normal operation)                              │
│    │  Errors counted but execution continues            │
│    │                                                    │
│    │  error_count >= threshold                          │
│    ▼                                                    │
│  OPEN (execution blocked)                               │
│    │  All calls immediately fail                        │
│    │  Wait for cooldown period                          │
│    │                                                    │
│    │  cooldown elapsed                                  │
│    ▼                                                    │
│  HALF-OPEN (testing)                                    │
│    │  Allow one call through                            │
│    │                                                    │
│    ├── Success → return to CLOSED                       │
│    └── Failure → return to OPEN                         │
└─────────────────────────────────────────────────────────┘
```

### Implementation

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Blocking all calls
    HALF_OPEN = "half_open" # Testing with one call

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 3,
        cooldown_seconds: int = 60,
        name: str = "default"
    ):
        self.failure_threshold = failure_threshold
        self.cooldown_seconds = cooldown_seconds
        self.name = name
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0
    
    def can_execute(self) -> bool:
        """Check if execution is allowed."""
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            # Check if cooldown has elapsed
            if time.time() - self.last_failure_time > self.cooldown_seconds:
                self.state = CircuitState.HALF_OPEN
                return True  # Allow one test call
            return False  # Still cooling down
        
        if self.state == CircuitState.HALF_OPEN:
            return True  # Allow the test call
        
        return False
    
    def record_success(self):
        """Record a successful execution."""
        self.failure_count = 0
        self.state = CircuitState.CLOSED
    
    def record_failure(self):
        """Record a failed execution."""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            raise CircuitBreakerOpen(
                f"Circuit breaker '{self.name}' opened after "
                f"{self.failure_count} failures. "
                f"Cooldown: {self.cooldown_seconds}s"
            )

# Usage in agent loop
breaker = CircuitBreaker(failure_threshold=3, cooldown_seconds=30)

for turn in range(max_turns):
    if not breaker.can_execute():
        # Escalate — circuit is open
        return escalate_to_human("Tool repeatedly failing, circuit breaker tripped")
    
    try:
        result = execute_tool(tool_name, tool_input)
        breaker.record_success()
    except ToolError as e:
        breaker.record_failure()  # May raise CircuitBreakerOpen
```

### Per-Tool Circuit Breakers

```python
class MultiToolCircuitBreaker:
    """Separate circuit breakers for each tool."""
    
    def __init__(self, default_threshold: int = 3):
        self.breakers = {}
        self.default_threshold = default_threshold
    
    def get_breaker(self, tool_name: str) -> CircuitBreaker:
        if tool_name not in self.breakers:
            self.breakers[tool_name] = CircuitBreaker(
                failure_threshold=self.default_threshold,
                name=tool_name
            )
        return self.breakers[tool_name]
    
    def can_call_tool(self, tool_name: str) -> bool:
        return self.get_breaker(tool_name).can_execute()
    
    def report_tool_success(self, tool_name: str):
        self.get_breaker(tool_name).record_success()
    
    def report_tool_failure(self, tool_name: str):
        self.get_breaker(tool_name).record_failure()
```

---

## 6. Connecting to Sprint 1: escalate_to_human

### From Sprint 1: The escalate_to_human Tool

In Sprint 1 (Day 8-10), you learned about designing an `escalate_to_human` tool. Now in Sprint 3, this tool becomes part of the escalation architecture:

```python
# Sprint 1 concept: A tool the agent can call
escalate_to_human_tool = {
    "name": "escalate_to_human",
    "description": "Escalate a task to a human operator when the agent "
                   "cannot resolve it autonomously. Use when: confidence is low, "
                   "task exceeds policy, or repeated failures occur.",
    "input_schema": {
        "type": "object",
        "properties": {
            "reason": {
                "type": "string",
                "description": "Why escalation is needed"
            },
            "context": {
                "type": "string", 
                "description": "Relevant context for the human"
            },
            "severity": {
                "type": "string",
                "enum": ["low", "medium", "high", "critical"]
            },
            "partial_results": {
                "type": "string",
                "description": "Any work completed before escalation"
            }
        },
        "required": ["reason", "severity"]
    }
}
```

### Sprint 3 Integration: Escalation in Multi-Agent Context

```python
# In Sprint 3, escalation is part of the architecture
class AgentWithEscalation:
    def __init__(self):
        self.tools = [
            escalate_to_human_tool,  # From Sprint 1
            # ... other tools
        ]
        self.circuit_breaker = CircuitBreaker(failure_threshold=3)
        self.escalation_manager = EscalationManager()
    
    async def run_loop(self, messages):
        for turn in range(self.max_turns):
            # Check escalation triggers BEFORE each turn
            trigger = self.escalation_manager.check_triggers(self.state)
            if trigger:
                # Model can now call escalate_to_human based on trigger
                messages.append({
                    "role": "user",
                    "content": f"[System: Escalation trigger fired: {trigger.reason}. "
                               f"Consider using escalate_to_human tool.]"
                })
            
            response = await self.call_model(messages)
            # ... normal loop processing
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Blind Retries

```python
# ❌ BAD: Retry without any feedback or limit
def retry_forever(tool_name, tool_input):
    while True:
        try:
            return execute_tool(tool_name, tool_input)
        except Exception:
            pass  # Just try again... and again... and again...
            # Same input, same error, forever
```

### ✅ CORRECT: Informed Retries with Limits

```python
# ✅ GOOD: Limited retries with error feedback and strategy changes
def smart_retry(tool_name, tool_input, max_retries=3):
    errors = []
    for attempt in range(max_retries):
        try:
            result = execute_tool(tool_name, tool_input)
            return result
        except Exception as e:
            errors.append(str(e))
            
            # Adjust strategy based on error
            if "not found" in str(e):
                tool_input = adjust_path(tool_input, e)
            elif "timeout" in str(e):
                tool_input = reduce_scope(tool_input)
            elif "permission" in str(e):
                # Can't fix — escalate immediately
                raise EscalationRequired(f"Permission issue: {e}")
    
    # All retries exhausted
    raise EscalationRequired(
        f"Failed after {max_retries} attempts. Errors: {errors}"
    )
```

### Anti-Pattern Summary

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Blind retries | Same error repeated forever | Change strategy between retries |
| No error context | Agent can't self-correct | Feed error details back |
| Swallowing errors | Problems hidden until catastrophic | Propagate with enriched context |
| Escalating too early | Human overloaded with trivial issues | Try 2-3 approaches first |
| Escalating too late | Damage done before human sees it | Set clear trigger thresholds |
| No circuit breaker | Broken tools called indefinitely | Stop after N consecutive failures |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│     ESCALATION & ERROR PROPAGATION — QUICK REFERENCE    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ESCALATION TRIGGERS:                                   │
│    • Confidence below threshold                         │
│    • Policy boundary exceeded                           │
│    • N consecutive errors (circuit breaker)             │
│    • Ambiguous or contradictory input                   │
│    • Cost/time limits approaching                       │
│                                                         │
│  ERROR PROPAGATION CHAIN:                               │
│    Subagent → Coordinator → (Retry|Reassign|Escalate)  │
│                                                         │
│  ERROR-CONTEXT FEEDBACK:                                │
│    • Feed error details back via tool_result            │
│    • Set is_error: true                                 │
│    • Include prior error history                        │
│    • Suggest trying different approach                  │
│                                                         │
│  CIRCUIT BREAKER:                                       │
│    CLOSED → (N failures) → OPEN → (cooldown) → HALF    │
│    Stops execution after N consecutive failures         │
│                                                         │
│  SPRINT CONNECTION:                                     │
│    Sprint 1: escalate_to_human tool design              │
│    Sprint 3: escalation architecture + triggers         │
│                                                         │
│  KEY ANTI-PATTERN: Blind retries                        │
│  (same input, same error, no strategy change)           │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

Your multi-agent system has a coordinator delegating to 3 subagents. Subagent B fails 3 times in a row with "Connection timeout" errors. The other two subagents completed successfully.

**Question 1:** What should the circuit breaker do after the 3rd failure?

- A) Retry one more time with the same parameters
- B) Open the circuit — block further calls and escalate
- C) Silently skip the task and pretend it succeeded
- D) Kill the entire multi-agent system

**Answer:** **B** — After 3 consecutive failures (the configured threshold), the circuit breaker transitions to OPEN state. This blocks further calls to the failing tool/service and triggers escalation. It prevents resource waste while isolating the failure. The other subagents are unaffected.

---

**Question 2:** When feeding error context back to the model, which approach is correct?

- A) Simply say "try again" without any error details
- B) Include the error message, what was attempted, and suggest trying differently
- C) Include the full stack trace and internal system details
- D) Tell the model exactly what to do next (remove all autonomy)

**Answer:** **B** — Good error feedback includes: what went wrong (error message), what was attempted (context), and a nudge to try differently (guidance without micromanaging). Option A provides no useful information. Option C exposes internals the model doesn't need. Option D removes the agent's ability to reason about alternatives.

---

**Question 3:** The coordinator receives an error from a subagent. In what order should it consider responses?

- A) Escalate → Retry → Reassign
- B) Retry with feedback → Try different approach → Escalate to human
- C) Kill the session → Restart from scratch → Alert human
- D) Ignore the error → Continue with other tasks → Report at the end

**Answer:** **B** — The correct escalation ladder is: (1) retry with error feedback (cheapest, might work for transient errors), (2) try a fundamentally different approach (if the same approach failed twice), (3) escalate to human (last resort when automated recovery fails). This minimizes human interruption while ensuring recovery.

---

**Question 4:** How does the `escalate_to_human` tool from Sprint 1 connect to Sprint 3's escalation architecture?

- A) They're completely separate concepts
- B) Sprint 1 defined the tool interface; Sprint 3 defines when and why to invoke it
- C) Sprint 3 replaces the Sprint 1 approach entirely
- D) escalate_to_human is only used in single-agent systems

**Answer:** **B** — Sprint 1 established the tool design (name, description, input schema). Sprint 3 adds the architectural layer: escalation triggers (when to call it), error propagation chains (how errors flow), and circuit breakers (automated escalation on repeated failures). The tool is the same; the surrounding architecture is what Sprint 3 adds.
