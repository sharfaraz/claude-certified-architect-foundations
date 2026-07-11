# Day 14 — Sprint 1 Comprehensive Review

**Domains**: Domain 4: Prompt Engineering & Structured Output + Domain 2: Tool Design & MCP Integration
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Full API Lifecycle Review](#1-full-api-lifecycle-review)
2. [Structured Output Pipeline Review](#2-structured-output-pipeline-review)
3. [Tool Design Principles Review](#3-tool-design-principles-review)
4. [Anti-Pattern Catalog (Complete)](#4-anti-pattern-catalog-complete)
5. [Scenario Pattern Comparisons](#5-scenario-pattern-comparisons)
6. [Weak Area Identification Checklist](#6-weak-area-identification-checklist)

---

## 1. Full API Lifecycle Review

```
Request:
┌────────────────────────────────────────────────────────┐
│ model: "claude-sonnet-4-20250514"                      │
│ max_tokens: 1024                                       │
│ system: "You are..."                  (behavior)       │
│ tools: [{name, description, input_schema}]  (capabilities) │
│ tool_choice: {type: "tool", name: "..."} (enforcement) │
│ messages: [{role, content}]            (conversation)  │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
Response:
┌────────────────────────────────────────────────────────┐
│ content: [{type: "tool_use", id, name, input}]         │
│ stop_reason: "tool_use"                                │
│ usage: {input_tokens, output_tokens}                   │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
Your Application:
┌────────────────────────────────────────────────────────┐
│ 1. Check stop_reason → branch control flow             │
│ 2. Extract tool_use block → get input data             │
│ 3. Validate with Pydantic → catch errors               │
│ 4. Execute business logic → use the structured data    │
│ 5. (Optional) Return tool_result → continue loop       │
└────────────────────────────────────────────────────────┘
```

---

## 2. Structured Output Pipeline Review

### The Complete Stack

| Layer | Mechanism | Guarantees |
|-------|-----------|-----------|
| Tool definition | `tools[].input_schema` (JSON Schema) | Structure of Claude's output |
| Tool enforcement | `tool_choice: {"type": "tool"}` | Claude WILL call the tool |
| Schema generation | `Pydantic.model_json_schema()` | Schema matches your code |
| Validation | `Pydantic.model_validate()` | Data passes business rules |
| Batch processing | Message Batches API | Scale to 10,000 requests |
| Guidance | Few-shot examples | Better handling of edge cases |

### Key Relationships

- **JSON Schema** defines the contract between Claude and your code
- **tool_choice** enforces that Claude respects the contract
- **Pydantic** generates the schema AND validates the output (both ends)
- **Few-shot** teaches Claude HOW to fill the schema correctly for ambiguous cases

---

## 3. Tool Design Principles Review

| Principle | Description | Example |
|-----------|-------------|---------|
| Clear purpose | Tool description says exactly what it does | "Look up customer by email" |
| Usage guidance | Description says WHEN to use it | "Use FIRST to identify customer" |
| Prerequisites | States what must happen before | "Only after lookup_order" |
| Return value docs | Says what the tool returns | "Returns customer_id, name, tier" |
| Constraints | States limits and guardrails | "Max refund $500" |
| Minimal input | Only required fields in schema | Just what the tool needs |
| Error guidance | Describes possible error responses | "May return NOT_FOUND or POLICY_LIMIT" |

---

## 4. Anti-Pattern Catalog (Complete)

| # | Anti-Pattern | Category | Fix |
|---|-------------|----------|-----|
| 1 | Ignoring stop_reason | API handling | Always check and branch |
| 2 | Vague JSON instructions | Structured output | tool_choice + schema |
| 3 | Temperature for tool selection | Misunderstanding | Use tool_choice |
| 4 | Missing conversation history | Stateless API | Send full history |
| 5 | Few-shot as sole enforcement | Structured output | Combine with tool_choice |
| 6 | Skipping validation | Data integrity | Always use Pydantic |
| 7 | Over-complex schemas | Schema design | Keep flat/shallow |
| 8 | Batches for real-time | Batches API | Use single requests |
| 9 | Loops without circuit breakers | Agentic patterns | Max retries + escalation |
| 10 | Blind retries | Error handling | Error-context feedback |
| 11 | Vague tool descriptions | Tool design | Specific, with guidance |
| 12 | Skipping verification steps | Tool sequencing | Enforce prerequisites |

---

## 5. Scenario Pattern Comparisons

### Structured Data Extraction Scenario

| Question Type | Key Concept |
|---------------|------------|
| "Claude returns text instead of using tool" | → tool_choice enforcement |
| "30% of extractions fail validation" | → Optional fields in schema |
| "Processing 5000 documents" | → Message Batches API |
| "Ambiguous classifications" | → Few-shot + reasoning field |
| "Documents in varying formats" | → Robust schema with Optional |

### Customer Support Scenario

| Question Type | Key Concept |
|---------------|------------|
| "Correct tool sequence" | → Identify → verify → act |
| "Tool error: exceeds policy" | → Escalate (don't retry) |
| "Claude skips verification" | → Better tool descriptions or structural deps |
| "Agent retries indefinitely" | → Circuit breaker pattern |
| "When to escalate" | → Policy boundary, repeated failure, customer request |

---

## 6. Weak Area Identification Checklist

Rate yourself 1–5 on each topic. Any score below 4 needs review before Day 15:

- [ ] Can explain all stop_reason values and control flow implications
- [ ] Can design a tool_use pipeline from Pydantic → Schema → tool_choice → validation
- [ ] Can write tool descriptions that guide sequencing
- [ ] Can identify when to use Batches vs single requests
- [ ] Can design schemas with appropriate Optional/required fields
- [ ] Can explain why programmatic enforcement beats vague instructions
- [ ] Can identify loops-without-circuit-breakers anti-pattern
- [ ] Can design escalation triggers for an agent
- [ ] Can combine few-shot with tool_choice correctly
- [ ] Understand the full request/response lifecycle
