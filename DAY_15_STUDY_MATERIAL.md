# Day 15 — Sprint 1 Milestone Checkpoint

**Domains**: Domain 4: Prompt Engineering & Structured Output + Domain 2: Tool Design & MCP Integration
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Checkpoint Assessment](#1-checkpoint-assessment)
2. [Domain 4 Readiness Check](#2-domain-4-readiness-check)
3. [Domain 2 Readiness Check](#3-domain-2-readiness-check)
4. [Practice Exam Questions](#4-practice-exam-questions)
5. [Confidence Tracker](#5-confidence-tracker)
6. [Sprint 2 Preview](#6-sprint-2-preview)

---

## 1. Checkpoint Assessment

This is a self-assessment day. Work through the questions below WITHOUT referring to notes. Track your confidence and identify gaps to address during Sprint 2.

### Sprint 1 Key Concepts to Verify

| Concept | Can I Explain It? | Can I Apply It? | Can I Spot Anti-Patterns? |
|---------|:-:|:-:|:-:|
| stop_reason values | ☐ | ☐ | ☐ |
| tool_choice enforcement | ☐ | ☐ | ☐ |
| JSON Schema keywords | ☐ | ☐ | ☐ |
| Pydantic pipeline | ☐ | ☐ | ☐ |
| Few-shot message pairs | ☐ | ☐ | ☐ |
| Message Batches API | ☐ | ☐ | ☐ |
| Tool description design | ☐ | ☐ | ☐ |
| Tool sequencing | ☐ | ☐ | ☐ |
| Escalation patterns | ☐ | ☐ | ☐ |
| Programmatic enforcement | ☐ | ☐ | ☐ |

---

## 2. Domain 4 Readiness Check

Answer without notes:

1. **What are the three layers of enforcement in the Pydantic + tool_choice pattern?**
2. **What does `stop_reason: "max_tokens"` mean and what should your app do?**
3. **When should you use Message Batches API vs. single requests?**
4. **How does `.model_json_schema()` differ from manually writing JSON Schema?**
5. **Why is "Please always return JSON" an anti-pattern?**
6. **How many few-shot examples should you typically provide?**
7. **What is the difference between `tool_choice: "any"` and `tool_choice: {"type": "tool", "name": "..."}`?**

---

## 3. Domain 2 Readiness Check

Answer without notes:

1. **What makes a good tool description? List 4 elements.**
2. **What is the correct tool sequence for a customer support interaction?**
3. **When should an agent escalate vs. retry?**
4. **What is a "circuit breaker" in the context of agent tool use?**
5. **What is the purpose of `is_error: true` in a `tool_result` block?**
6. **Why should tool input schemas be minimal?**

---

## 4. Practice Exam Questions

### Question 1

You're designing a structured extraction tool for processing legal contracts. The tool must extract: parties involved, effective date, termination clause, and governing law. Some contracts include arbitration clauses; others don't. Which schema design is correct?

A) All fields required, including arbitration_clause
B) All fields required; add `arbitration_clause: Optional[str] = Field(default=None)`
C) All fields optional with no required fields
D) Use a discriminator field to switch between "with_arbitration" and "without_arbitration" schemas

**Answer**: B — Core fields required, optional fields use Optional with default. C is too permissive. A rejects valid contracts. D is over-engineered.

### Question 2

Your Claude API application receives `stop_reason: "tool_use"` but your code only handles `"end_turn"`. What happens?

A) The application crashes with an exception
B) The application processes the text content normally, missing the tool call
C) Claude retries automatically
D) The API returns an error on the next request

**Answer**: B — If you don't handle `tool_use`, you'll process whatever text content exists (possibly empty) and miss that Claude wanted to call a tool. The conversation breaks because Claude never gets the tool result.

### Question 3

A customer support agent has been retrying `process_refund` for 5 iterations. Each time, it returns "insufficient_funds_in_merchant_account." What's the correct behavior?

A) Continue retrying — the merchant account may be replenished
B) Escalate to human after the 2nd failure with error context
C) Inform the customer that refunds are temporarily unavailable
D) Call a different tool to check merchant account balance first

**Answer**: B — A persistent error should trigger escalation after 2 failures (circuit breaker pattern). The human can investigate the system issue. A is the loops-without-circuit-breakers anti-pattern. C gives an incomplete resolution. D may not exist as a tool.

---

## 5. Confidence Tracker

Rate your confidence (1-5) for each area. Record in LEARNING_LOG.md:

| Topic | Confidence (1-5) | Notes |
|-------|:-:|-------|
| API request/response structure | | |
| stop_reason handling | | |
| tool_use / tool_result flow | | |
| tool_choice parameter | | |
| JSON Schema design | | |
| Pydantic integration | | |
| Few-shot prompting | | |
| Message Batches API | | |
| Tool description design | | |
| Escalation patterns | | |
| Circuit breaker pattern | | |
| Extraction pipeline design | | |

**Target**: All areas should be 4+ before entering Sprint 2. Areas at 3 or below should be flagged for review.

---

## 6. Sprint 2 Preview

Sprint 2 (Days 16–30) shifts focus to:

- **Domain 3: Claude Code Configuration & Workflows** (20%)
  - CLAUDE.md hierarchy (global → project → subdirectory)
  - .claude/rules/, .claude/commands/, .claude/skills/
  - Plan mode vs. direct execution
  - CLI flags: `-p`, `--output-format json`, `--resume`, `fork_session`

- **Domain 2: Tool Design & MCP Integration** (deeper coverage)
  - MCP architecture (clients, servers, protocol)
  - .mcp.json configuration
  - MCP tools vs. resources vs. prompts
  - Environment variable expansion
  - Built-in tools

**What carries forward**: Everything from Sprint 1. The tool_use patterns you learned (tool descriptions, schemas, sequencing) apply directly to MCP tool definitions. The JSON Schema skills apply to MCP tool input schemas.
