# Day 9 — Sprint 1 Review & Consolidation (API & Structured Output)

**Domain**: Domain 4: Prompt Engineering & Structured Output (review)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Complete Building with the Claude API (Pod 3)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Sprint 1 Concept Map](#1-sprint-1-concept-map)
2. [End-to-End Pipeline Walkthrough](#2-end-to-end-pipeline-walkthrough)
3. [Key Exam Concept: Programmatic Enforcement](#3-key-exam-concept-programmatic-enforcement)
4. [stop_reason Comprehensive Review](#4-stop_reason-comprehensive-review)
5. [Common Anti-Patterns Catalog](#5-common-anti-patterns-catalog)
6. [Self-Assessment Questions](#6-self-assessment-questions)
7. [Quick Recall Flashcards](#7-quick-recall-flashcards)

---

## 1. Sprint 1 Concept Map

```
Day 1: API Fundamentals
  └── Messages API, roles, system prompts, parameters, response structure
      │
Day 2: System Prompts & stop_reason
  └── stop_reason values (end_turn, max_tokens, stop_sequence, tool_use)
      │
Day 3: tool_use Message Structure
  └── tool_use blocks, tool_result blocks, multi-turn tool flow
      │
Day 4: tool_choice Parameter
  └── auto, any, tool — programmatic enforcement of tool selection
      │
Day 5: JSON Schema
  └── type, properties, required, enum, description — schema design
      │
Day 6: Pydantic Integration
  └── BaseModel → .model_json_schema() → validation pipeline
      │
Day 7: Few-Shot Prompting
  └── Message pairs, tool_use examples, combining with enforcement
      │
Day 8: Message Batches API
  └── Async bulk processing, 50% cost savings, lifecycle management
```

**The Throughline**: Each day builds toward the same principle — **programmatic enforcement over vague instruction**. Day 1 introduces the API. Days 2–4 show how `stop_reason` and `tool_choice` give you control. Days 5–6 add structural enforcement via schema + validation. Day 7 adds guidance through examples. Day 8 scales the whole pattern.

---

## 2. End-to-End Pipeline Walkthrough

Here's the complete structured extraction pipeline — the culmination of Days 1–8:

```python
import anthropic
from pydantic import BaseModel, Field, field_validator, ValidationError
from typing import List, Optional
from enum import Enum

# --- STEP 1: Define the output structure (Day 6) ---
class Urgency(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class SupportTicketExtraction(BaseModel):
    customer_name: str = Field(description="Customer's full name")
    issue_category: str = Field(description="Category: billing, technical, account, or feature_request")
    urgency: Urgency = Field(description="Urgency level based on the tone and content")
    summary: str = Field(max_length=200, description="Brief one-sentence summary")
    action_items: List[str] = Field(description="Specific actions needed to resolve")
    
    @field_validator('issue_category')
    @classmethod
    def validate_category(cls, v):
        valid = {"billing", "technical", "account", "feature_request"}
        if v not in valid:
            raise ValueError(f"Category must be one of {valid}")
        return v

# --- STEP 2: Generate JSON Schema (Day 5) ---
schema = SupportTicketExtraction.model_json_schema()

# --- STEP 3: Create tool definition (Day 3) ---
tools = [{
    "name": "extract_ticket_info",
    "description": "Extract structured information from a customer support ticket",
    "input_schema": schema
}]

# --- STEP 4: Call Claude with enforcement (Day 4) ---
client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a support ticket analysis system. Extract structured data from tickets accurately.",  # Day 1
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_ticket_info"},  # Day 4 — FORCE the tool
    messages=[
        {"role": "user", "content": "Analyze this ticket: 'Hi, I'm John Smith. I've been charged twice for my subscription this month. This is urgent as my bank account is overdrawn. Please fix this ASAP and refund the extra charge.'"}
    ]
)

# --- STEP 5: Handle stop_reason (Day 2) ---
assert response.stop_reason == "tool_use"  # Guaranteed by tool_choice

# --- STEP 6: Validate with Pydantic (Day 6) ---
for block in response.content:
    if block.type == "tool_use":
        try:
            ticket = SupportTicketExtraction.model_validate(block.input)
            print(f"Customer: {ticket.customer_name}")
            print(f"Category: {ticket.issue_category}")
            print(f"Urgency: {ticket.urgency.value}")
            print(f"Summary: {ticket.summary}")
        except ValidationError as e:
            print(f"Validation failed: {e}")
```

---

## 3. Key Exam Concept: Programmatic Enforcement

The single most tested principle across Domain 4:

| Vague Instruction (❌) | Programmatic Enforcement (✅) |
|------------------------|-------------------------------|
| "Please return JSON" | `tool_choice` + JSON Schema |
| "Always use the extract tool" | `tool_choice: {"type": "tool", "name": "..."}` |
| "Keep responses short" | `max_tokens` parameter |
| "Stop when you see END" | `stop_sequences: ["END"]` |
| "Return data in this format" | Pydantic model → schema → validation |
| "Make sure all fields are present" | `required` in JSON Schema |

---

## 4. stop_reason Comprehensive Review

| Value | Complete? | What Happened | Your Action |
|-------|-----------|---------------|-------------|
| `end_turn` | Yes | Claude finished naturally | Process the response |
| `max_tokens` | No | Hit token ceiling | Increase limit or continue |
| `stop_sequence` | Maybe | Hit your custom stop string | Check which sequence matched |
| `tool_use` | No | Claude wants to call a tool | Execute tool, return result |
| `pause_turn` | No | Server tool loop paused | Send response back |
| `refusal` | N/A | Safety refusal | Handle gracefully |

---

## 5. Common Anti-Patterns Catalog

| # | Anti-Pattern | Why It Fails | Fix |
|---|--------------|-------------|-----|
| 1 | Ignoring stop_reason | Truncated responses processed as complete | Always check stop_reason |
| 2 | Vague JSON instructions | No guarantee of compliance | tool_choice + schema |
| 3 | Temperature for tool selection | Temperature ≠ tool selection | Use tool_choice |
| 4 | Not sending conversation history | Stateless API loses context | Include full history |
| 5 | Few-shot as only enforcement | Examples don't guarantee format | Combine with tool_choice |
| 6 | Skipping Pydantic validation | Unvalidated data in production | Always validate |
| 7 | Over-complex schemas | More nesting = more errors | Keep flat where possible |
| 8 | Batches for real-time | Users wait hours | Use single requests |

---

## 6. Self-Assessment Questions

Answer these without looking at notes. If you can't answer confidently, review that day's material.

1. What are the four core `stop_reason` values and what does each mean?
2. What is the difference between `tool_choice: "auto"`, `"any"`, and `{"type": "tool", "name": "..."}`?
3. How do you generate a JSON Schema from a Pydantic model?
4. Why is `tool_choice` + schema called "programmatic enforcement"?
5. When should you use the Message Batches API vs. single requests?
6. What is the correct message structure for few-shot examples with tool_use?
7. What happens when a batch request expires?
8. How do `Optional` fields in Pydantic map to JSON Schema?

---

## 7. Quick Recall Flashcards

| Question | Answer |
|----------|--------|
| Force Claude to call a specific tool | `tool_choice: {"type": "tool", "name": "tool_name"}` |
| Generate schema from model | `MyModel.model_json_schema()` |
| Validate dict against model | `MyModel.model_validate(data)` |
| Batch API cost savings | 50% discount |
| Max requests per batch | 10,000 |
| Response truncated indicator | `stop_reason: "max_tokens"` |
| Claude wants to use a tool | `stop_reason: "tool_use"` |
| System prompt location in API | `system` parameter (NOT in messages) |
| API is stateless means | Send full conversation history every time |
| `required` in JSON Schema | Lists mandatory fields that must be present |
