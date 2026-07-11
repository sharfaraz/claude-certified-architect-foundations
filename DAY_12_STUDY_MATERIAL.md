# Day 12 — Scenario Deep-Dive: Customer Support Resolution Agent

**Domain**: Domain 2: Tool Design & MCP Integration + Domain 1: Agentic Architecture & Orchestration
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Scenario Overview](#1-scenario-overview)
2. [Tool Design for Customer Service](#2-tool-design-for-customer-service)
3. [Tool Description Best Practices](#3-tool-description-best-practices)
4. [Tool Sequencing Patterns](#4-tool-sequencing-patterns)
5. [Input Schema Design](#5-input-schema-design)
6. [The Escalation Circuit Breaker](#6-the-escalation-circuit-breaker)
7. [Practice Challenges](#7-practice-challenges)

---

## Recommended Video Resources

- [Anthropic — How we built our multi-agent research system](https://www.anthropic.com/engineering/claude-code-best-practices) — Architectural patterns for agent tool design
- [Anthropic Docs — Tool Use Implementation](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use) — Official tool_use guide

---

## 1. Scenario Overview

The **Customer Support Resolution Agent** scenario tests your ability to:

- Design tools with clear, unambiguous descriptions
- Sequence tool calls correctly (lookup → verify → act)
- Implement escalation patterns (when the agent should hand off to a human)
- Handle errors gracefully (failed lookups, policy violations)

**Core tools in this scenario**:
- `get_customer(email)` — Look up customer by email
- `lookup_order(customer_id, order_id)` — Retrieve order details
- `process_refund(order_id, amount, reason)` — Process a refund
- `escalate_to_human(customer_id, reason)` — Hand off to human agent

---

## 2. Tool Design for Customer Service

Each tool needs a **clear, unambiguous description** that tells Claude:
- WHAT the tool does
- WHEN to use it
- WHAT it returns

```python
tools = [
    {
        "name": "get_customer",
        "description": "Look up a customer's account by their email address. Returns customer_id, name, account status, and membership tier. Use this FIRST when a customer contacts support to identify their account.",
        "input_schema": {
            "type": "object",
            "properties": {
                "email": {
                    "type": "string",
                    "description": "Customer's email address"
                }
            },
            "required": ["email"]
        }
    },
    {
        "name": "lookup_order",
        "description": "Retrieve details of a specific order. Returns order status, items, amounts, shipping info, and delivery date. Use AFTER get_customer to verify order details before taking action.",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {
                    "type": "string",
                    "description": "Customer ID from get_customer result"
                },
                "order_id": {
                    "type": "string",
                    "description": "Order ID provided by the customer"
                }
            },
            "required": ["customer_id", "order_id"]
        }
    },
    {
        "name": "process_refund",
        "description": "Process a refund for a specific order. Only use AFTER verifying the order with lookup_order. Refund amount must not exceed order total. Returns success/failure status.",
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "description": "Order ID to refund"},
                "amount": {"type": "number", "description": "Refund amount in USD (must not exceed order total)"},
                "reason": {"type": "string", "description": "Reason for the refund"}
            },
            "required": ["order_id", "amount", "reason"]
        }
    },
    {
        "name": "escalate_to_human",
        "description": "Escalate to a human agent. Use when: (1) refund exceeds policy limits, (2) customer requests a manager, (3) issue cannot be resolved with available tools, (4) safety or legal concerns arise.",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {"type": "string", "description": "Customer ID"},
                "reason": {"type": "string", "description": "Why this needs human review"},
                "conversation_summary": {"type": "string", "description": "Brief summary of the interaction so far"}
            },
            "required": ["customer_id", "reason", "conversation_summary"]
        }
    }
]
```

---

## 3. Tool Description Best Practices

| Principle | Example | Why |
|-----------|---------|-----|
| State the purpose | "Look up a customer's account by email" | Claude knows WHAT it does |
| State the trigger | "Use this FIRST when a customer contacts support" | Claude knows WHEN to use it |
| State the output | "Returns customer_id, name, account status" | Claude knows what to expect |
| State constraints | "Refund amount must not exceed order total" | Prevents misuse |
| State prerequisites | "Use AFTER get_customer" | Enforces correct sequencing |

### Anti-Pattern: Vague Descriptions

```python
# ❌ WRONG: Too vague — Claude can't determine sequencing
{"name": "get_customer", "description": "Gets customer info"}
{"name": "process_refund", "description": "Processes a refund"}

# ✅ RIGHT: Clear, specific, with usage guidance
{"name": "get_customer", "description": "Look up a customer's account by email. Use FIRST to identify the customer before any other action."}
{"name": "process_refund", "description": "Process a refund for a verified order. Only use AFTER lookup_order confirms the order exists and the amount is within policy limits."}
```

---

## 4. Tool Sequencing Patterns

The correct flow for customer support resolution:

```
Customer contacts support
    │
    ▼
get_customer(email)  ←── Identify who they are
    │
    ▼
lookup_order(customer_id, order_id)  ←── Verify their claim
    │
    ├── Order valid + within policy  →  process_refund(order_id, amount, reason)
    │
    ├── Order valid + exceeds policy  →  escalate_to_human(customer_id, reason, summary)
    │
    └── Order not found  →  Ask customer for correct order details (or escalate)
```

### Why Sequencing Matters

The exam tests whether you understand that:
1. **You must identify before acting** — Never process a refund without verifying the customer and order
2. **Each tool provides context for the next** — `get_customer` returns `customer_id` needed by `lookup_order`
3. **Escalation is a valid outcome** — Not every interaction can be resolved autonomously

---

## 5. Input Schema Design

### Principle: Each Tool Gets Only What It Needs

```python
# ❌ WRONG: Tool takes everything, even irrelevant fields
{
    "name": "process_refund",
    "input_schema": {
        "properties": {
            "customer_name": {"type": "string"},  # Not needed for refund processing
            "email": {"type": "string"},           # Not needed
            "order_id": {"type": "string"},        # Needed
            "amount": {"type": "number"},          # Needed
            "reason": {"type": "string"}           # Needed
        }
    }
}

# ✅ RIGHT: Minimal, focused schema
{
    "name": "process_refund",
    "input_schema": {
        "properties": {
            "order_id": {"type": "string"},
            "amount": {"type": "number"},
            "reason": {"type": "string"}
        },
        "required": ["order_id", "amount", "reason"]
    }
}
```

---

## 6. The Escalation Circuit Breaker

`escalate_to_human` is the circuit breaker in this system — it prevents the agent from looping on problems it can't solve.

### When to Escalate (Exam-Critical)

| Trigger | Example |
|---------|---------|
| Policy boundary exceeded | Refund > $500 limit |
| Tool returns an error | `process_refund` fails |
| Customer requests human | "I want to speak to a manager" |
| Ambiguous situation | Customer's story doesn't match order data |
| Repeated failures | Same tool fails 2+ times |
| Safety/legal concern | Fraud indicators, legal threats |

### Anti-Pattern: Loops Without Escalation

```python
# ❌ WRONG: Agent retries indefinitely when refund fails
while True:
    result = process_refund(order_id, amount, reason)
    if result.success:
        break
    # Never escalates — loops forever on policy violation

# ✅ RIGHT: Escalate after understanding the error
result = process_refund(order_id, amount, reason)
if not result.success:
    if "exceeds policy" in result.error:
        escalate_to_human(customer_id, "Refund exceeds policy limit", summary)
    elif attempt_count >= 2:
        escalate_to_human(customer_id, "Repeated failure", summary)
```

---

## 7. Practice Challenges

### Challenge 1 (from SPEC)

> Customer says order arrived damaged. Agent has: `get_customer(email)`, `lookup_order(customer_id, order_id)`, `process_refund(order_id, amount, reason)`, `escalate_to_human(customer_id, reason)`. Customer provides email and order number. Correct sequence?
>
> A) `process_refund` → `get_customer` → `lookup_order`
> B) `get_customer` → `lookup_order` → `process_refund`
> C) `get_customer` → `process_refund` → `escalate_to_human`
> D) `escalate_to_human` immediately

**Answer**: B — Identify → verify → act. Must confirm customer identity and order details before processing any refund.

### Challenge 2

> The `process_refund` tool returns: "Error: Refund amount exceeds policy limit." The agent has no tool to override limits. What next?
>
> A) Retry with same parameters
> B) Reduce amount and retry
> C) `escalate_to_human` with context
> D) Inform customer and end conversation

**Answer**: C — When encountering a policy boundary that can't be overridden, escalate with full context. A is a blind retry, B changes the refund without authorization, D abandons the customer.

### Challenge 3

> You're designing the tool descriptions for this agent. The `process_refund` tool currently has no mention of prerequisites or limits in its description. During testing, Claude sometimes calls `process_refund` before `lookup_order`. What's the fix?
>
> A) Add "IMPORTANT: Always call lookup_order first" to the system prompt
> B) Add a prerequisite statement to the `process_refund` tool description: "Only use AFTER verifying the order with lookup_order"
> C) Remove `process_refund` from the tools array and only add it after `lookup_order` succeeds
> D) Set `tool_choice` to force `lookup_order` first

**Answer**: B — Tool descriptions are the primary mechanism for guiding Claude's tool selection behavior. Clear sequencing guidance in the description solves the problem at the right layer. A is indirect, C is over-engineered, D only works for one turn.
