# Day 32: Agent SDK: Hooks and Lifecycle Events

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Continue Introduction to Agent Skills |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Hooks in Claude Agent SDK](#1-hooks-in-claude-agent-sdk)
2. [Lifecycle Event Types](#2-lifecycle-event-types)
3. [Implementing Hooks for Business Rules](#3-implementing-hooks-for-business-rules)
4. [Use Cases: Logging, Validation, and Guardrails](#4-use-cases-logging-validation-and-guardrails)
5. [Hook Ordering and Composition](#5-hook-ordering-and-composition)
6. [Connecting Hooks to CI/CD Patterns](#6-connecting-hooks-to-cicd-patterns)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code Features (Hooks) | https://code.claude.com/docs/en/agent-sdk/claude-code-features |
| Agentic Tool Use | https://docs.anthropic.com/en/docs/build-with-claude/agentic-tool-use |

---

## 1. Hooks in Claude Agent SDK

### What Are Hooks?

Hooks are **interceptor functions** that execute at specific points in an agent's lifecycle. They let you inject custom logic without modifying the core agent loop.

```
┌─────────────────────────────────────────────────────────┐
│                    AGENT LOOP                            │
│                                                         │
│  ┌──────────┐   HOOK    ┌──────────┐   HOOK            │
│  │  REASON  │──────────▶│   ACT    │──────────▶        │
│  └──────────┘  pre-tool └──────────┘  post-tool         │
│                                                         │
│       HOOK                      HOOK                    │
│    pre-response              post-response              │
│       │                          │                      │
│       ▼                          ▼                      │
│  ┌──────────┐              ┌──────────┐                │
│  │ VALIDATE │              │   LOG    │                │
│  └──────────┘              └──────────┘                │
└─────────────────────────────────────────────────────────┘
```

### Why Hooks Matter for the Exam

- Domain 1 (27%) covers **how to control agent behavior**
- Hooks are the mechanism for **guardrails, auditing, and policy enforcement**
- They connect to Sprint 2 concepts (CLAUDE.md rules → runtime enforcement)

---

## 2. Lifecycle Event Types

### The Four Core Hook Points

| Hook Point | When It Fires | Common Use Cases |
|------------|---------------|-----------------|
| **pre-tool-call** | Before a tool is executed | Validation, approval gates, parameter sanitization |
| **post-tool-call** | After a tool returns a result | Logging, result filtering, error transformation |
| **pre-response** | Before the final response is sent to user | Content filtering, compliance checks |
| **post-response** | After the response is delivered | Analytics, cost tracking, audit logging |

### Lifecycle Flow Diagram

```
User Request
    │
    ▼
┌─────────────────┐
│   Agent Loop     │
│                  │
│   Model reasons  │
│        │         │
│        ▼         │
│   ┌──────────┐   │
│   │pre-tool  │◀── Intercept before execution
│   │  hook    │    Can BLOCK or MODIFY the call
│   └────┬─────┘   │
│        │         │
│        ▼         │
│   [Tool Executes] │
│        │         │
│        ▼         │
│   ┌──────────┐   │
│   │post-tool │◀── Intercept after execution
│   │  hook    │    Can MODIFY or LOG the result
│   └────┬─────┘   │
│        │         │
│        ▼         │
│   Model reasons  │
│   again...       │
│        │         │
│        ▼         │
│   ┌──────────┐   │
│   │pre-resp  │◀── Intercept before delivery
│   │  hook    │    Can FILTER or BLOCK response
│   └────┬─────┘   │
│        │         │
│        ▼         │
│   [Response Sent] │
│        │         │
│        ▼         │
│   ┌──────────┐   │
│   │post-resp │◀── Intercept after delivery
│   │  hook    │    Logging and analytics only
│   └──────────┘   │
└─────────────────┘
```

### Hook Context Object

Each hook receives a context object with relevant information:

```python
class HookContext:
    agent_name: str          # Which agent is running
    turn_number: int         # Current iteration in the loop
    tool_name: str | None    # Tool being called (tool hooks only)
    tool_input: dict | None  # Tool parameters (pre-tool only)
    tool_result: str | None  # Tool output (post-tool only)
    messages: list           # Conversation history
    total_cost: float        # Running cost for this session
```

---

## 3. Implementing Hooks for Business Rules

### Basic Hook Structure

```python
from claude_code_sdk import Hook, HookAction

# Pre-tool hook: Block dangerous operations
class FileWriteGuard(Hook):
    """Prevent writing to production config files."""
    
    event = "pre-tool-call"
    
    def execute(self, context: HookContext) -> HookAction:
        if context.tool_name == "write_file":
            filepath = context.tool_input.get("path", "")
            
            # Block writes to production configs
            if "/prod/" in filepath or filepath.endswith(".env"):
                return HookAction.BLOCK(
                    reason="Cannot write to production files. "
                           "Use staging environment instead."
                )
            
            # Allow other writes
            return HookAction.ALLOW()
        
        return HookAction.ALLOW()
```

### Human Approval Gate

```python
class HumanApprovalGate(Hook):
    """Require human approval for high-risk operations."""
    
    event = "pre-tool-call"
    HIGH_RISK_TOOLS = ["delete_file", "run_sql", "deploy"]
    
    def execute(self, context: HookContext) -> HookAction:
        if context.tool_name in self.HIGH_RISK_TOOLS:
            # Pause and request human approval
            approved = request_human_approval(
                action=context.tool_name,
                params=context.tool_input,
                agent=context.agent_name
            )
            
            if approved:
                return HookAction.ALLOW()
            else:
                return HookAction.BLOCK(
                    reason="Human operator denied this action."
                )
        
        return HookAction.ALLOW()
```

### Cost Tracking Hook

```python
class CostTracker(Hook):
    """Track and enforce cost budgets."""
    
    event = "post-tool-call"
    
    def __init__(self, max_budget: float = 10.00):
        self.max_budget = max_budget
        self.session_costs = {}
    
    def execute(self, context: HookContext) -> HookAction:
        # Log cost
        cost = estimate_tool_cost(context.tool_name, context.tool_result)
        self.session_costs[context.agent_name] = (
            self.session_costs.get(context.agent_name, 0) + cost
        )
        
        total = self.session_costs[context.agent_name]
        
        if total > self.max_budget * 0.8:
            log_warning(f"Agent {context.agent_name} at 80% budget: ${total:.2f}")
        
        if total > self.max_budget:
            return HookAction.TERMINATE(
                reason=f"Cost budget exceeded: ${total:.2f} > ${self.max_budget:.2f}"
            )
        
        return HookAction.ALLOW()
```

---

## 4. Use Cases: Logging, Validation, and Guardrails

### Use Case Matrix

| Use Case | Hook Point | Action | Example |
|----------|-----------|--------|---------|
| **Audit logging** | post-tool-call | Log tool name + result | Compliance trail |
| **Input validation** | pre-tool-call | Validate parameters | SQL injection prevention |
| **Content filtering** | pre-response | Filter response content | PII redaction |
| **Cost tracking** | post-tool-call | Accumulate costs | Budget enforcement |
| **Human approval** | pre-tool-call | Pause for approval | High-risk operations |
| **Rate limiting** | pre-tool-call | Throttle calls | API protection |
| **Error transformation** | post-tool-call | Rewrite errors | User-friendly messages |
| **Analytics** | post-response | Record metrics | Performance monitoring |

### Validation Hook Example

```python
class SQLInjectionGuard(Hook):
    """Validate SQL parameters before execution."""
    
    event = "pre-tool-call"
    DANGEROUS_PATTERNS = ["DROP", "DELETE FROM", "--", ";--", "UNION SELECT"]
    
    def execute(self, context: HookContext) -> HookAction:
        if context.tool_name == "run_sql":
            query = context.tool_input.get("query", "")
            
            for pattern in self.DANGEROUS_PATTERNS:
                if pattern.upper() in query.upper():
                    return HookAction.BLOCK(
                        reason=f"Potentially dangerous SQL detected: {pattern}"
                    )
        
        return HookAction.ALLOW()
```

### PII Redaction Hook

```python
class PIIRedactor(Hook):
    """Redact PII from agent responses before delivery."""
    
    event = "pre-response"
    
    def execute(self, context: HookContext) -> HookAction:
        response_text = context.response_content
        
        # Redact email addresses
        redacted = re.sub(
            r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
            '[EMAIL REDACTED]',
            response_text
        )
        
        # Redact phone numbers
        redacted = re.sub(
            r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b',
            '[PHONE REDACTED]',
            redacted
        )
        
        return HookAction.MODIFY(content=redacted)
```

---

## 5. Hook Ordering and Composition

### Multiple Hooks on the Same Event

When multiple hooks are registered for the same event, they execute in registration order:

```python
agent = Agent(
    name="secure-agent",
    hooks=[
        # Order matters! First registered = first executed
        RateLimiter(),          # 1st: Check rate limits
        SQLInjectionGuard(),    # 2nd: Validate input safety
        HumanApprovalGate(),    # 3rd: Require approval if needed
        AuditLogger(),          # 4th: Log the action
    ]
)
```

### Hook Chain Behavior

```
Hook 1 (RateLimiter)    → ALLOW  → continues to next hook
Hook 2 (SQLGuard)       → ALLOW  → continues to next hook
Hook 3 (HumanApproval)  → BLOCK  → STOPS chain, tool not executed
Hook 4 (AuditLogger)    → never reached (chain stopped)
```

### Composition Rules

| Rule | Behavior |
|------|----------|
| First BLOCK wins | If any hook returns BLOCK, execution stops |
| All must ALLOW | Tool only executes if all pre-hooks allow |
| Post-hooks always run | Post-tool hooks run regardless of result |
| Order is deterministic | Hooks run in registration order |

---

## 6. Connecting Hooks to CI/CD Patterns

### From Sprint 2: CLAUDE.md Rules → Runtime Hooks

Sprint 2 taught that CLAUDE.md defines static rules. Hooks enforce them at runtime:

```
Sprint 2 (Static):          Sprint 3 (Dynamic):
┌──────────────────┐        ┌──────────────────┐
│ CLAUDE.md        │        │ Runtime Hook     │
│ "Never write to  │───────▶│ FileWriteGuard   │
│  production DB"  │        │ blocks prod      │
└──────────────────┘        │ write attempts   │
                            └──────────────────┘
```

### CI/CD Integration Pattern

```python
class CIGateHook(Hook):
    """Enforce CI/CD pipeline rules at agent runtime."""
    
    event = "pre-tool-call"
    
    def execute(self, context: HookContext) -> HookAction:
        if context.tool_name == "deploy":
            target = context.tool_input.get("environment", "")
            
            # Check if tests passed (from CI system)
            if target == "production":
                ci_status = check_ci_pipeline_status()
                if ci_status != "green":
                    return HookAction.BLOCK(
                        reason="Cannot deploy to production: CI pipeline not green"
                    )
            
            # Check if deployment window is open
            if not is_deployment_window_open():
                return HookAction.BLOCK(
                    reason="Outside deployment window. Next window: Mon-Fri 9am-4pm"
                )
        
        return HookAction.ALLOW()
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Hooks That Modify Silently

```python
# ❌ BAD: Silently changes tool input without logging
class SilentModifier(Hook):
    event = "pre-tool-call"
    
    def execute(self, context: HookContext) -> HookAction:
        # Silently rewrites the query — agent doesn't know!
        if context.tool_name == "search":
            context.tool_input["query"] += " site:internal.com"
        return HookAction.ALLOW()
```

### ✅ CORRECT: Transparent Modification

```python
# ✅ GOOD: Logs modifications, agent sees the change
class TransparentModifier(Hook):
    event = "pre-tool-call"
    
    def execute(self, context: HookContext) -> HookAction:
        if context.tool_name == "search":
            original = context.tool_input["query"]
            modified = original + " site:internal.com"
            log_info(f"Hook modified search: '{original}' → '{modified}'")
            return HookAction.MODIFY(
                tool_input={"query": modified},
                reason="Restricted to internal sources per policy"
            )
        return HookAction.ALLOW()
```

### Common Anti-Patterns Table

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Silent modification | Debugging becomes impossible | Always log modifications |
| Blocking without reason | Agent can't adapt or explain | Always provide reason string |
| Heavy computation in hooks | Slows down every turn | Keep hooks lightweight, async if needed |
| State in hooks without cleanup | Memory leaks across sessions | Reset state per session |
| Hooks that call the agent | Recursive loops | Hooks should be one-directional |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│        HOOKS & LIFECYCLE EVENTS — QUICK REFERENCE       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  HOOK POINTS:                                           │
│    pre-tool-call   → Before tool executes               │
│    post-tool-call  → After tool returns                 │
│    pre-response    → Before response to user            │
│    post-response   → After response delivered           │
│                                                         │
│  HOOK ACTIONS:                                          │
│    ALLOW()         → Continue execution                 │
│    BLOCK(reason)   → Stop execution, provide reason     │
│    MODIFY(...)     → Change input/output                │
│    TERMINATE(...)  → End the entire agent loop          │
│                                                         │
│  ORDERING:                                              │
│    • Hooks run in registration order                    │
│    • First BLOCK stops the chain                        │
│    • All pre-hooks must ALLOW for execution             │
│                                                         │
│  KEY USE CASES:                                         │
│    • Human approval gates (pre-tool)                    │
│    • Cost tracking (post-tool)                          │
│    • PII redaction (pre-response)                       │
│    • Audit logging (post-tool, post-response)           │
│                                                         │
│  CONNECTS TO:                                           │
│    Sprint 2: CLAUDE.md rules → runtime enforcement      │
│    Sprint 2: CI/CD patterns → deployment guardrails     │
│    Day 31: Circuit breakers → termination hooks         │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

Your company's compliance team requires that any agent interacting with customer data must: (1) log every tool call, (2) never expose raw email addresses in responses, (3) require human approval before deleting any record.

**Question 1:** Which combination of hooks satisfies all three requirements?

- A) Three pre-tool-call hooks
- B) post-tool-call logger + pre-response PII filter + pre-tool-call approval gate
- C) One post-response hook that handles all three
- D) pre-tool-call logger + post-tool-call PII filter + pre-response approval gate

**Answer:** **B** — Logging belongs at post-tool-call (you log after execution to capture results). PII filtering belongs at pre-response (filter before delivery to user). Human approval belongs at pre-tool-call (block before destructive action). This maps each requirement to its natural hook point.

---

**Question 2:** A pre-tool-call hook returns BLOCK. What happens to subsequent hooks in the chain?

- A) They all still execute for logging purposes
- B) They are skipped — first BLOCK stops the chain
- C) They execute but their return values are ignored
- D) Only post-tool hooks are skipped

**Answer:** **B** — When a hook returns BLOCK, the chain stops immediately. The tool is not executed, and no subsequent pre-tool hooks run. This is the "first BLOCK wins" rule.

---

**Question 3:** Where should cost tracking hooks be placed?

- A) pre-tool-call — to estimate costs before execution
- B) post-tool-call — to calculate actual costs after execution
- C) pre-response — costs are only meaningful at response time
- D) post-response — track costs after everything is done

**Answer:** **B** — Post-tool-call is the correct placement because you can calculate actual costs (based on tokens consumed, API calls made, etc.) after the tool executes. Pre-tool can only estimate. Post-response is too late to enforce budget limits.

---

**Question 4:** How do hooks relate to the CLAUDE.md rules from Sprint 2?

- A) Hooks replace CLAUDE.md entirely
- B) CLAUDE.md defines static rules; hooks enforce them at runtime
- C) Hooks only work without CLAUDE.md
- D) CLAUDE.md and hooks are completely independent systems

**Answer:** **B** — CLAUDE.md defines policy (e.g., "never write to production"). Hooks are the runtime enforcement mechanism that actually blocks the action if the agent attempts to violate the policy. They're complementary: static policy definition + dynamic policy enforcement.
