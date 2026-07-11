# Day 7 — Few-Shot Prompting Techniques

**Domain**: Domain 4: Prompt Engineering & Structured Output (20% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Continue Building with the Claude API (Pod 2)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [What Is Few-Shot Prompting?](#1-what-is-few-shot-prompting)
2. [Structuring Examples as Message Pairs](#2-structuring-examples-as-message-pairs)
3. [Few-Shot with tool_use](#3-few-shot-with-tool_use)
4. [When Few-Shot Helps Most](#4-when-few-shot-helps-most)
5. [How Many Examples Are Enough?](#5-how-many-examples-are-enough)
6. [Few-Shot in System Prompts vs Messages](#6-few-shot-in-system-prompts-vs-messages)
7. [Combining Few-Shot with Programmatic Enforcement](#7-combining-few-shot-with-programmatic-enforcement)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video Resources

- [Anthropic Prompt Engineering Interactive Tutorial](https://github.com/anthropics/prompt-eng-interactive-tutorial) — Official Anthropic notebook (Chapter 7: Few-Shot Prompting)
- [Anthropic Docs — Multishot Prompting](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting) — Official documentation on examples

---

## 1. What Is Few-Shot Prompting?

Few-shot prompting is the technique of providing example input/output pairs in your prompt so Claude can learn the desired pattern from examples rather than (or in addition to) explicit instructions.

**Terminology:**
- **Zero-shot**: No examples — just instructions
- **One-shot**: One example
- **Few-shot**: 2–5 examples (the sweet spot for most tasks)
- **Many-shot**: 6+ examples (useful for complex or ambiguous tasks)

**Why this matters for the exam**: Few-shot prompting is tested as both a standalone technique and in combination with tool_use. The exam distinguishes between using few-shot as a complement to programmatic enforcement (good) vs. using it as a substitute (anti-pattern).

### The Core Principle

Few-shot prompting works because Claude is an excellent pattern matcher. When you show it examples of the desired behavior, it infers the underlying pattern and applies it to new inputs — often more reliably than following abstract instructions alone.

```
Show Claude what you want → Claude infers the pattern → Claude applies it to new input
```

---

## 2. Structuring Examples as Message Pairs

The most effective way to provide few-shot examples in the Claude API is as `user`/`assistant` message pairs in the conversation history:

```python
messages = [
    # Example 1
    {"role": "user", "content": "Classify: 'The product broke after one week'"},
    {"role": "assistant", "content": "Category: Product Quality\nSentiment: Negative\nPriority: High"},
    
    # Example 2
    {"role": "user", "content": "Classify: 'Love the new color options!'"},
    {"role": "assistant", "content": "Category: Product Design\nSentiment: Positive\nPriority: Low"},
    
    # Example 3
    {"role": "user", "content": "Classify: 'Shipping took 3 days as expected'"},
    {"role": "assistant", "content": "Category: Shipping\nSentiment: Neutral\nPriority: Low"},
    
    # Actual input
    {"role": "user", "content": "Classify: 'Customer support never responded to my email'"}
]
```

### Why Message Pairs Work Better Than System Prompt Examples

| Approach | Pros | Cons |
|----------|------|------|
| Message pairs | Claude treats these as "real" conversation history; strongest pattern enforcement | Uses more tokens; counts against context window |
| System prompt examples | Saves context window space; grouped with instructions | Slightly weaker pattern adherence for complex formats |

**Exam tip**: For the exam, message pairs are the canonical approach. System prompt examples (using XML tags) are the alternative when token budget is tight.

---

## 3. Few-Shot with tool_use

You can show Claude examples of correct tool calls by including `tool_use` and `tool_result` blocks in the conversation history:

```python
messages = [
    # Example: Show Claude a correct tool call
    {"role": "user", "content": "Extract contact info from: Jane Doe, jane@example.com, 555-1234"},
    {"role": "assistant", "content": [
        {
            "type": "tool_use",
            "id": "example_1",
            "name": "extract_contact",
            "input": {
                "full_name": "Jane Doe",
                "email": "jane@example.com",
                "phone": "555-1234",
                "company": None
            }
        }
    ]},
    {"role": "user", "content": [
        {
            "type": "tool_result",
            "tool_use_id": "example_1",
            "content": "Contact extracted successfully"
        }
    ]},
    
    # Actual request
    {"role": "user", "content": "Extract contact info from: Bob Smith at Acme Inc, bob@acme.com"}
]
```

**Key point**: When showing tool_use examples, you must include the matching `tool_result` block to maintain valid message structure (tool_use must always be followed by tool_result).

---

## 4. When Few-Shot Helps Most

| Scenario | Why Few-Shot Helps | Example |
|----------|-------------------|---------|
| Ambiguous output format | Shows the exact format rather than describing it | Custom report layouts |
| Domain-specific terminology | Demonstrates correct usage of jargon | Medical coding, legal citations |
| Edge case handling | Shows how to handle tricky inputs | Partial data, multi-language text |
| Consistent tone/style | Demonstrates the desired voice | Brand-specific copywriting |
| Classification tasks | Shows category boundaries | Sentiment analysis, ticket routing |

### When Few-Shot Is Less Critical

- **Simple, unambiguous tasks** — "Translate this to French" doesn't need examples
- **Schema-enforced output** — When `tool_choice` + JSON Schema already guarantees the structure
- **Well-defined instructions** — Clear, specific system prompts may suffice alone

---

## 5. How Many Examples Are Enough?

Research and practice show diminishing returns after 3–5 examples for most tasks:

| Number | When to Use |
|--------|-------------|
| 1–2 | Simple format demonstration; when token budget is limited |
| 3–5 | Most classification and extraction tasks; covers common patterns |
| 6–10 | Complex tasks with many edge cases; ambiguous category boundaries |
| 10+ | Rarely needed; consider fine-tuning or better schema design instead |

**Diminishing returns principle**: The marginal improvement from the 4th example is much smaller than the improvement from the 1st example. Each additional example costs input tokens on every API call.

### Example Selection Strategy

Choose examples that:
1. **Cover the range** — Include positive, negative, and edge cases
2. **Are diverse** — Don't repeat the same pattern multiple times
3. **Are representative** — Match the distribution of actual inputs
4. **Demonstrate boundaries** — Show what's "in" and "out" for each category

---

## 6. Few-Shot in System Prompts vs Messages

### System Prompt Approach (XML Tags)

```python
system = """You are a customer feedback classifier.

<examples>
<example>
<input>The product broke after one week</input>
<output>Category: Product Quality | Sentiment: Negative | Priority: High</output>
</example>
<example>
<input>Love the new color options!</input>
<output>Category: Product Design | Sentiment: Positive | Priority: Low</output>
</example>
</examples>

Classify the user's feedback using the same format as the examples above."""
```

### Message Pairs Approach

```python
messages = [
    {"role": "user", "content": "Classify: The product broke after one week"},
    {"role": "assistant", "content": "Category: Product Quality | Sentiment: Negative | Priority: High"},
    {"role": "user", "content": "Classify: Love the new color options!"},
    {"role": "assistant", "content": "Category: Product Design | Sentiment: Positive | Priority: Low"},
    {"role": "user", "content": "Classify: Customer support never responded to my email"}
]
```

### Decision Guide

| Factor | System Prompt | Message Pairs |
|--------|--------------|---------------|
| Token efficiency in multi-turn | Better (examples sent once) | Worse (examples in every message history) |
| Pattern adherence | Good | Slightly better |
| Combining with tool_use examples | Awkward | Natural |
| When to choose | Long conversations, simple formats | Single-shot tasks, tool_use demos |

---

## 7. Combining Few-Shot with Programmatic Enforcement

The exam tests this combination: few-shot examples complement (not replace) programmatic enforcement.

```python
# Best practice: Few-shot PLUS tool_choice + schema
tools = [{
    "name": "classify_feedback",
    "description": "Classify customer feedback",
    "input_schema": FeedbackClassification.model_json_schema()
}]

messages = [
    # Few-shot example showing the TOOL CALL pattern
    {"role": "user", "content": "Classify: Great product, fast shipping!"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "ex1", "name": "classify_feedback",
         "input": {"category": "general", "sentiment": "positive", "priority": "low"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "ex1", "content": "Classified successfully"}
    ]},
    # Actual request
    {"role": "user", "content": "Classify: The delivery was two weeks late and the box was damaged"}
]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "classify_feedback"},  # Enforcement
    messages=messages
)
```

**The examples guide Claude on HOW to fill the schema fields (especially for ambiguous cases), while `tool_choice` guarantees Claude WILL use the tool.**

---

## 8. Anti-Patterns

### Anti-Pattern 1: Using Few-Shot as a Substitute for Schema Enforcement

```python
# ❌ WRONG: Examples alone don't guarantee the format
messages = [
    {"role": "user", "content": "Extract: John, john@example.com"},
    {"role": "assistant", "content": '{"name": "John", "email": "john@example.com"}'},
    {"role": "user", "content": "Extract: Jane at Acme Corp"}
]
# Claude MIGHT return plain text, different JSON structure, or markdown
```

### Anti-Pattern 2: Contradictory Examples

```python
# ❌ WRONG: Examples show conflicting patterns
messages = [
    {"role": "user", "content": "Classify: Late delivery"},
    {"role": "assistant", "content": "Shipping | Negative | High"},  # Format A
    {"role": "user", "content": "Classify: Great product"},
    {"role": "assistant", "content": "Category: Product | Sentiment: Positive | Priority: Low"},  # Format B (different!)
]
```

### Anti-Pattern 3: Too Many Examples of the Same Pattern

```python
# ❌ WRONG: All examples are positive sentiment — Claude learns a bias
messages = [
    {"role": "user", "content": "Classify: Love it!"},
    {"role": "assistant", "content": "Positive"},
    {"role": "user", "content": "Classify: Amazing product!"},
    {"role": "assistant", "content": "Positive"},
    {"role": "user", "content": "Classify: Best purchase ever!"},
    {"role": "assistant", "content": "Positive"},
    # Claude may now be biased toward "Positive" for all inputs
]
```

---

## 9. Quick Reference Card

### Few-Shot Implementation Checklist

1. ✅ Select 3–5 diverse, representative examples
2. ✅ Use consistent format across all examples
3. ✅ Cover edge cases and boundary conditions
4. ✅ Structure as user/assistant message pairs (preferred) or XML in system prompt
5. ✅ Combine with `tool_choice` + schema for guaranteed structure
6. ✅ Include `tool_result` blocks when demonstrating tool_use patterns

### When to Add Examples vs. When to Improve the Schema

| Symptom | Fix |
|---------|-----|
| Claude uses wrong field names | Improve schema `description` fields |
| Claude misclassifies edge cases | Add few-shot examples of those edge cases |
| Claude returns text instead of using tool | Add `tool_choice` enforcement (not more examples) |
| Claude fills fields with wrong granularity | Add examples showing desired granularity |

---

## 10. Scenario Challenge

### Scenario: Domain 4 — Prompt Engineering & Structured Output

**Scenario**: You're building a ticket routing system that classifies support tickets into categories: billing, technical, feature_request, and account. During testing, you notice that Claude correctly classifies obvious cases but struggles with ambiguous tickets (e.g., "I can't log in to pay my bill" could be technical OR billing OR account). Adding more instructions to the system prompt hasn't helped.

**Question**: What is the most effective approach to improve classification of ambiguous tickets?

A) Increase the number of few-shot examples to 20+, covering every possible ambiguity

B) Add 3–5 few-shot examples specifically showing how ambiguous tickets should be classified, combined with `tool_choice` to enforce the classification tool, and add a `reasoning` field to the schema so Claude explains its decision

C) Lower the temperature to 0.0 to make Claude's classifications more deterministic

D) Create separate tools for each category and let Claude choose which one to call

---

**Correct Answer**: **B**

**Explanation**: Targeted few-shot examples that address the specific ambiguity problem teach Claude where the category boundaries are. The `tool_choice` enforcement guarantees structured output, and the `reasoning` field (a common pattern) forces Claude to articulate its logic, improving accuracy on edge cases.

- **Why A is wrong**: 20+ examples have diminishing returns and waste tokens. Targeted examples are more effective than volume.
- **Why C is wrong**: Temperature affects randomness, not classification accuracy. A deterministic wrong answer is still wrong.
- **Why D is wrong**: Separate tools per category doesn't help with ambiguity — Claude still must choose, but now without a reasoning mechanism and with a more complex tool setup.
