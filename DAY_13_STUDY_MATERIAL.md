# Day 13 — Scenario Deep-Dive: Customer Support Resolution Agent (Escalation Patterns)

**Domain**: Domain 2: Tool Design & MCP Integration + Domain 1: Agentic Architecture & Orchestration
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Escalation Pattern Design](#1-escalation-pattern-design)
2. [Guardrails: Refund Limits and Policy Compliance](#2-guardrails-refund-limits-and-policy-compliance)
3. [Error Handling in Tool Chains](#3-error-handling-in-tool-chains)
4. [Tool Description Refinement](#4-tool-description-refinement)
5. [Anti-Pattern: Loops Without Circuit Breakers](#5-anti-pattern-loops-without-circuit-breakers)
6. [Complete Agent Flow Example](#6-complete-agent-flow-example)
7. [Practice Challenges](#7-practice-challenges)

---

## 1. Escalation Pattern Design

Escalation is not failure — it's a **design feature**. A well-designed agent knows its boundaries and hands off gracefully when it reaches them.

### Escalation Trigger Categories

| Category | Examples | Agent Action |
|----------|----------|--------------|
| Policy boundaries | Refund > limit, restricted account actions | Escalate with policy context |
| Customer demand | "I want a manager", "This is unacceptable" | Escalate with conversation history |
| Repeated failures | Same tool fails 2+ times | Escalate with error details |
| Ambiguity | Conflicting information, unclear intent | Escalate with what's known |
| Safety/Legal | Fraud indicators, threats, legal requests | Immediate escalation |
| Capability limits | No tool available for the needed action | Escalate with action needed |

### The `escalate_to_human` Tool Design

A good escalation tool captures everything a human agent needs to continue:

```python
{
    "name": "escalate_to_human",
    "description": "Hand off to a human agent when the issue cannot be resolved autonomously. Include all relevant context so the human can continue without asking the customer to repeat themselves.",
    "input_schema": {
        "type": "object",
        "properties": {
            "customer_id": {
                "type": "string",
                "description": "Customer ID (from get_customer)"
            },
            "reason": {
                "type": "string",
                "enum": ["policy_limit", "customer_request", "repeated_failure", "ambiguous_situation", "safety_concern", "capability_limit"],
                "description": "Categorized reason for escalation"
            },
            "conversation_summary": {
                "type": "string",
                "description": "Complete summary: what the customer wants, what was attempted, and why escalation is needed"
            },
            "attempted_actions": {
                "type": "array",
                "items": {"type": "string"},
                "description": "List of actions already attempted"
            },
            "recommended_resolution": {
                "type": "string",
                "description": "What the AI agent thinks should happen next (optional guidance for the human)"
            }
        },
        "required": ["customer_id", "reason", "conversation_summary"]
    }
}
```

---

## 2. Guardrails: Refund Limits and Policy Compliance

Guardrails prevent the agent from taking unauthorized actions:

### Guardrail Types

| Guardrail | Implementation | Example |
|-----------|---------------|---------|
| Amount limits | Schema constraint or server-side check | Refund ≤ $500 |
| Action restrictions | Only certain tools available | Can't delete accounts |
| Rate limits | Track actions per session | Max 1 refund per interaction |
| Scope limits | Customer can only modify their own data | customer_id must match |

### Implementing Guardrails in Tool Descriptions

```python
{
    "name": "process_refund",
    "description": "Process a refund for a verified order. CONSTRAINTS: (1) Must call lookup_order first to verify the order exists, (2) Refund amount cannot exceed the order total, (3) Maximum single refund is $500 — amounts above this require human approval via escalate_to_human, (4) Only one refund per order is allowed.",
    "input_schema": {
        "type": "object",
        "properties": {
            "order_id": {"type": "string"},
            "amount": {"type": "number", "maximum": 500},
            "reason": {
                "type": "string",
                "enum": ["damaged_item", "wrong_item", "not_received", "quality_issue", "other"]
            }
        },
        "required": ["order_id", "amount", "reason"]
    }
}
```

---

## 3. Error Handling in Tool Chains

When a tool returns an error, the agent must interpret the error and decide what to do:

### Error Response Patterns

```python
# Simulated tool results returned to Claude
tool_results = {
    "success": {"status": "success", "message": "Refund of $45.99 processed for order ORD-1234"},
    "not_found": {"status": "error", "error_code": "NOT_FOUND", "message": "Order ORD-9999 not found for this customer"},
    "policy_limit": {"status": "error", "error_code": "POLICY_LIMIT", "message": "Refund amount $750 exceeds policy limit of $500"},
    "already_refunded": {"status": "error", "error_code": "DUPLICATE", "message": "Order ORD-1234 has already been refunded"}
}
```

### Agent Decision Tree on Errors

```
Tool returns error
    │
    ├── NOT_FOUND → Ask customer to verify order details
    │                (limit to 2 attempts, then escalate)
    │
    ├── POLICY_LIMIT → Escalate to human with amount context
    │
    ├── DUPLICATE → Inform customer refund already processed
    │
    ├── SYSTEM_ERROR → Retry once, then escalate if still failing
    │
    └── UNKNOWN → Escalate immediately with full error context
```

---

## 4. Tool Description Refinement

### Before Refinement (Vague)

```python
tools = [
    {"name": "get_customer", "description": "Gets a customer"},
    {"name": "lookup_order", "description": "Looks up an order"},
    {"name": "process_refund", "description": "Does a refund"},
    {"name": "escalate_to_human", "description": "Escalates to human"}
]
```

### After Refinement (Specific)

```python
tools = [
    {
        "name": "get_customer",
        "description": "Look up customer account by email. Returns: customer_id, name, membership_tier, account_status. Use FIRST in every interaction to identify the customer."
    },
    {
        "name": "lookup_order", 
        "description": "Get order details by customer_id and order_id. Returns: status, items, total, shipping_address, delivery_date. Use AFTER get_customer. Required before any refund."
    },
    {
        "name": "process_refund",
        "description": "Refund a verified order. PREREQUISITES: get_customer AND lookup_order must be called first. LIMITS: max $500, one per order. If over $500, use escalate_to_human instead."
    },
    {
        "name": "escalate_to_human",
        "description": "Hand off to human agent. Use when: refund > $500, customer requests manager, tool errors persist (2+ failures), or safety concern. Always include conversation_summary."
    }
]
```

---

## 5. Anti-Pattern: Loops Without Circuit Breakers

**This is a key exam anti-pattern**: An agent that retries a failed operation indefinitely without analyzing the error or setting a retry limit.

```python
# ❌ ANTI-PATTERN: Blind retry loop
attempt = 0
while True:
    result = call_tool("process_refund", {"order_id": "ORD-123", "amount": 750, "reason": "damaged"})
    if result["status"] == "success":
        break
    attempt += 1
    # Never stops. Amount exceeds limit — will NEVER succeed.

# ✅ CORRECT: Error-context feedback with circuit breaker
MAX_RETRIES = 2
for attempt in range(MAX_RETRIES):
    result = call_tool("process_refund", params)
    if result["status"] == "success":
        break
    elif result["error_code"] == "POLICY_LIMIT":
        # Don't retry — this error won't resolve itself
        call_tool("escalate_to_human", {
            "customer_id": customer_id,
            "reason": "policy_limit",
            "conversation_summary": f"Refund of ${params['amount']} exceeds $500 policy limit"
        })
        break
    elif attempt == MAX_RETRIES - 1:
        call_tool("escalate_to_human", {
            "reason": "repeated_failure",
            "conversation_summary": f"Tool failed {MAX_RETRIES} times: {result['message']}"
        })
```

---

## 6. Complete Agent Flow Example

```python
# The system prompt sets the agent's overall behavior
system_prompt = """You are a customer support agent for ShopCo. 
Follow this workflow:
1. Always identify the customer first (get_customer)
2. Verify order details before any action (lookup_order)
3. Process refunds only for verified orders within policy limits
4. Escalate to a human when you cannot resolve the issue

Policy limits:
- Maximum refund: $500
- One refund per order
- Refunds only for orders delivered in the last 30 days

Be empathetic and professional. If you need to escalate, reassure the customer that a specialist will help them."""
```

### Multi-Turn Conversation Flow

```
Turn 1: Customer → "My order ORD-5678 arrived damaged. Email: john@example.com"
         Agent → calls get_customer("john@example.com") → gets customer_id: "CUST-42"
         Agent → calls lookup_order("CUST-42", "ORD-5678") → gets order details

Turn 2: Agent → Sees order is valid, amount is $89.99 (within limit)
         Agent → calls process_refund("ORD-5678", 89.99, "damaged_item")
         Agent → "I've processed a full refund of $89.99. You'll see it in 3-5 business days."
```

---

## 7. Practice Challenges

### Challenge (from SPEC)

> The customer support agent attempts to process a refund, but `process_refund` returns: "Refund amount exceeds policy limit." The agent has no tool to override policy limits. What should it do next?
>
> A) Retry `process_refund` with the same parameters
> B) Reduce the refund amount and retry
> C) Call `escalate_to_human` with customer_id, reason, and conversation summary
> D) Inform the customer the refund cannot be processed and end the conversation

**Answer**: C — Escalate with full context when hitting a policy boundary. A is blind retry (anti-pattern). B modifies the refund without authorization. D abandons the customer without resolution.

### Challenge 2

> During testing, your agent occasionally calls `process_refund` before calling `lookup_order`. The tool descriptions mention sequencing but Claude still sometimes skips steps. What's the most effective fix?
>
> A) Add "IMPORTANT: NEVER skip lookup_order" to the system prompt in uppercase
> B) Make `lookup_order_result` a required field in `process_refund`'s input_schema, forcing Claude to have the data
> C) Remove `process_refund` from the tools array and only provide it after `lookup_order` succeeds
> D) Set `tool_choice` to always call `lookup_order` first

**Answer**: B — Making the refund tool require data that can only come from `lookup_order` creates a structural dependency. Claude must call `lookup_order` first to get the required input. C is valid but over-engineered for this use case. D only works for one turn. A is a vague instruction.
