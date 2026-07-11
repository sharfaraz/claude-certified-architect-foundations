# Day 4 — tool_choice Parameter Options

**Domain**: Domain 2: Tool Design & MCP Integration (18% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Continue Building with the Claude API (Pod 1)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [The tool_choice Parameter](#1-the-tool_choice-parameter)
2. [The Four tool_choice Values](#2-the-four-tool_choice-values)
3. [When to Use Each Option](#3-when-to-use-each-option)
4. [Programmatic Enforcement Pattern](#4-programmatic-enforcement-pattern)
5. [Combining tool_choice with stop_reason: "tool_use"](#5-combining-tool_choice-with-stop_reason-tool_use)
6. [The "Tool as Structured Output" Pattern](#6-the-tool-as-structured-output-pattern)
7. [Behavior Differences with tool_choice](#7-behavior-differences-with-tool_choice)
8. [Key Anti-Patterns for Day 4](#8-key-anti-patterns-for-day-4)
9. [Exam-Relevant Principles](#9-exam-relevant-principles)
10. [Quick Reference Card](#10-quick-reference-card)
11. [Scenario Challenge](#11-scenario-challenge)

---

## 1. The tool_choice Parameter

The `tool_choice` parameter controls **how** Claude uses the tools you provide. On Day 3, you learned how to define tools in the `tools[]` array and how the `tool_use` / `tool_result` flow works. But Day 3 left a critical question unanswered: what if Claude decides NOT to use a tool when you need it to? What if Claude responds with plain text instead of calling your extraction tool?

This is where `tool_choice` comes in. It is a **top-level parameter** in the API request, sitting alongside `tools`, `model`, `max_tokens`, and `messages`:

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[...],                          # What tools are available
    tool_choice={"type": "auto"},         # HOW Claude should use them
    messages=[...]
)
```

### Where tool_choice Fits in the Request

| Parameter | Purpose |
|-----------|---------|
| `model` | Which model to use |
| `max_tokens` | Maximum output length |
| `system` | System prompt — behavioral instructions |
| `messages` | Conversation history |
| `tools` | **What** tools are available |
| `tool_choice` | **How** Claude should use the tools |

Think of it this way: `tools` defines the menu, and `tool_choice` tells Claude whether it must order from the menu, can choose freely, or is required to order a specific dish.

**Why this matters for the exam**: `tool_choice` is the primary mechanism for the "programmatic enforcement over vague instruction" principle. Instead of telling Claude "please use the extract_data tool" in the system prompt (a vague instruction that Claude may or may not follow), you SET `tool_choice` to force it. This is the difference between hoping Claude does what you want and guaranteeing it.

---

## 2. The Four tool_choice Values

The `tool_choice` parameter accepts four possible values. Each one changes Claude's behavior in a distinct way.

### 2.1 `{"type": "auto"}` — Claude Decides

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get current weather for a US city. Returns temperature, conditions, and humidity.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City and state, e.g. 'San Francisco, CA'"
                    }
                },
                "required": ["location"]
            }
        }
    ],
    tool_choice={"type": "auto"},
    messages=[
        {"role": "user", "content": "What's the weather in San Francisco?"}
    ]
)
```

**Behavior**: Claude reads the user's message and the tool descriptions, then decides whether to call a tool or respond with text. It may call a tool, call multiple tools, or respond without any tool call at all.

**This is the DEFAULT**. When you provide `tools` but do not specify `tool_choice`, the API behaves as if you set `tool_choice: {"type": "auto"}`.

**When Claude calls a tool with `auto`**:
- `stop_reason` will be `"tool_use"`
- The `content` array may contain BOTH text blocks and tool_use blocks
- Claude often includes explanatory text before the tool call

**When Claude does NOT call a tool with `auto`**:
- `stop_reason` will be `"end_turn"` (or another non-tool reason)
- The `content` array contains only text blocks
- Claude answered the question using its own knowledge

### 2.2 `{"type": "any"}` — Must Use a Tool (Claude Chooses Which)

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get current weather for a US city.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "City and state"}
                },
                "required": ["location"]
            }
        },
        {
            "name": "get_forecast",
            "description": "Get a 5-day weather forecast for a US city.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "City and state"},
                    "days": {"type": "integer", "description": "Number of days (1-5)"}
                },
                "required": ["location"]
            }
        }
    ],
    tool_choice={"type": "any"},
    messages=[
        {"role": "user", "content": "What's the weather like in San Francisco?"}
    ]
)
```

**Behavior**: Claude MUST call one of the provided tools. It cannot respond with text only. However, Claude gets to choose WHICH tool to call based on the user's request and the tool descriptions.

**Key characteristics**:
- `stop_reason` will ALWAYS be `"tool_use"`
- Claude will NOT emit natural language text before the tool_use block (the API prefills the response to force a tool call)
- Claude selects the most appropriate tool from the available options

In the example above, Claude would likely choose `get_weather` over `get_forecast` because the user asked about current weather, not a forecast.

### 2.3 `{"type": "tool", "name": "specific_tool_name"}` — Must Use a Specific Tool

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "extract_invoice",
            "description": "Extract structured invoice data from text. Returns vendor name, invoice number, date, total amount, and line items.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "vendor_name": {
                        "type": "string",
                        "description": "The name of the vendor or supplier"
                    },
                    "invoice_number": {
                        "type": "string",
                        "description": "The invoice identifier"
                    },
                    "date": {
                        "type": "string",
                        "description": "Invoice date in YYYY-MM-DD format"
                    },
                    "total_amount": {
                        "type": "number",
                        "description": "Total invoice amount in USD"
                    },
                    "line_items": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "description": {"type": "string"},
                                "quantity": {"type": "integer"},
                                "unit_price": {"type": "number"}
                            },
                            "required": ["description", "quantity", "unit_price"]
                        },
                        "description": "Individual line items on the invoice"
                    }
                },
                "required": ["vendor_name", "invoice_number", "date", "total_amount", "line_items"]
            }
        }
    ],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[
        {"role": "user", "content": "Invoice #INV-2024-001 from Acme Corp, dated 2024-03-15. Total: $1,250.00. Items: 5x Widget A at $150.00 each, 2x Widget B at $250.00 each."}
    ]
)
```

**Behavior**: Claude MUST call the specified tool. There is no choice — Claude will call `extract_invoice` and populate its `input` field according to the tool's `input_schema`. This is **deterministic enforcement**.

**Key characteristics**:
- `stop_reason` will ALWAYS be `"tool_use"`
- Claude will NOT emit natural language text before the tool_use block
- The `name` in `tool_choice` must exactly match a tool name in the `tools[]` array
- Claude focuses entirely on generating the correct input for the specified tool

**This is the key to programmatic enforcement.** When you set `tool_choice: {"type": "tool", "name": "extract_invoice"}`, you are not asking Claude to use the tool — you are TELLING the API to force it. Combined with the JSON Schema in `input_schema`, this guarantees both that the tool is called AND that the output matches your schema.

### 2.4 `{"type": "none"}` — No Tools Allowed

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get current weather for a US city.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "City and state"}
                },
                "required": ["location"]
            }
        }
    ],
    tool_choice={"type": "none"},
    messages=[
        {"role": "user", "content": "What's the weather in San Francisco?"}
    ]
)
```

**Behavior**: Claude cannot use any tools, even though they are defined in the `tools[]` array. Claude will respond with text only.

**Key characteristics**:
- `stop_reason` will NEVER be `"tool_use"` (Claude cannot call tools)
- Claude responds as if no tools were provided
- The tools are still visible to Claude (it knows they exist) but it cannot call them
- This is the DEFAULT behavior when no `tools` parameter is provided at all

**Why would you include tools but disable them?** This is useful when you want to temporarily disable tool access without restructuring your request. For example, in a multi-turn conversation where tools were used earlier, you might want Claude to summarize the results without making additional tool calls.

---

## 3. When to Use Each Option

Choosing the right `tool_choice` value depends on your use case. Here is a decision framework:

### `auto` — Flexible Conversations

**Use when**: You want Claude to decide whether a tool is needed based on the conversation context.

**Good for**:
- General-purpose chatbots with tool access
- Conversational agents where not every message requires a tool call
- Scenarios where Claude should answer from its own knowledge when appropriate
- Multi-turn conversations where tool use is intermittent

**Example scenario**: A customer support agent that has access to `get_customer`, `lookup_order`, and `process_refund` tools. When the customer says "Hello, I need help with my order," Claude should respond conversationally. When the customer provides an order number, Claude should call `lookup_order`. The `auto` setting lets Claude make this judgment.

```python
# General-purpose agent — Claude decides when to use tools
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[get_customer_tool, lookup_order_tool, process_refund_tool],
    tool_choice={"type": "auto"},  # Or simply omit tool_choice (auto is default)
    messages=conversation_history
)
```

### `any` — Guaranteed Tool Call, Flexible Selection

**Use when**: You want to guarantee a tool call but don't care which tool Claude selects.

**Good for**:
- **Routing scenarios**: You have multiple tools representing different actions, and you want Claude to pick the right one based on user input
- **Classification pipelines**: Each tool represents a category, and Claude must classify the input into one
- **First-step forcing**: You want to ensure the agent starts by calling a tool rather than responding with text

**Example scenario**: A request router with tools for `handle_billing_inquiry`, `handle_technical_support`, and `handle_general_question`. Every incoming message must be routed to one of these handlers — Claude should never respond directly.

```python
# Routing scenario — Claude must pick a tool, but chooses which one
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        billing_tool,
        technical_support_tool,
        general_question_tool
    ],
    tool_choice={"type": "any"},
    messages=[
        {"role": "user", "content": "My credit card was charged twice for the same order."}
    ]
)
# Claude will call handle_billing_inquiry (forced to pick a tool, chooses the right one)
```

### `tool` — Deterministic, Guaranteed Structured Output

**Use when**: You need to guarantee that a specific tool is called. This is THE key pattern for the exam.

**Good for**:
- **Extraction pipelines**: Force Claude to extract structured data using a specific schema
- **Classification with a fixed schema**: Force Claude to classify input into predefined categories
- **Structured output generation**: Use a tool as a schema enforcement mechanism (the tool doesn't actually DO anything)
- **Deterministic workflows**: When your pipeline depends on a specific tool being called every time

**Example scenario**: An invoice extraction pipeline where every input document must produce structured output with vendor, amount, date, and line items. You cannot afford for Claude to sometimes respond with free text.

```python
# Extraction pipeline — Claude MUST call extract_invoice
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[
        {"role": "user", "content": invoice_text}
    ]
)
# Guaranteed: stop_reason == "tool_use", tool called == "extract_invoice"
```

### `none` — Temporarily Disable Tools

**Use when**: You want Claude to respond without using tools, even though tools are defined.

**Good for**:
- **Summarization steps**: After a series of tool calls, ask Claude to summarize without making more calls
- **Explanation requests**: Ask Claude to explain its reasoning without triggering additional tool use
- **Testing and debugging**: Temporarily disable tools to see how Claude responds without them
- **Conversation phases**: Switch between tool-enabled and tool-disabled phases in a workflow

**Example scenario**: After an agent has gathered data through several tool calls, you want Claude to synthesize the information into a final answer without making any more tool calls.

```python
# Summarization phase — no more tool calls
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    tools=[get_customer_tool, lookup_order_tool],  # Still defined but disabled
    tool_choice={"type": "none"},
    messages=[
        # ... previous conversation with tool calls ...
        {"role": "user", "content": "Now please summarize everything you found about this customer's issue."}
    ]
)
# Claude responds with text only — no tool calls possible
```

---


## 4. Programmatic Enforcement Pattern

This section covers THE most important exam concept for Day 4: using `tool_choice` to programmatically enforce behavior instead of relying on vague prompt instructions.

### The Problem: Vague Instructions Are Unreliable

Consider this approach to getting structured output from Claude:

```python
# ❌ ANTI-PATTERN: Vague instruction
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a data extraction assistant. Always use the extract_invoice tool to extract invoice data. Never respond with plain text.",
    tools=[extract_invoice_tool],
    # tool_choice not set — defaults to "auto"
    messages=[
        {"role": "user", "content": invoice_text}
    ]
)
```

**What can go wrong**: Even with explicit instructions in the system prompt, Claude may:
- Respond with plain text explaining the invoice instead of calling the tool
- Call the tool sometimes but not always (inconsistent behavior)
- Include explanatory text alongside the tool call (extra content you don't need)
- Decide the input doesn't look like an invoice and skip the tool entirely

The system prompt is a **suggestion**. Claude will usually follow it, but there is no guarantee. For a production extraction pipeline processing thousands of documents, "usually" is not good enough.

### The Solution: Programmatic Enforcement with tool_choice

```python
# ✅ CORRECT: Programmatic enforcement
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},  # FORCED
    messages=[
        {"role": "user", "content": invoice_text}
    ]
)
```

**What this guarantees**:
1. Claude WILL call `extract_invoice` — no exceptions
2. Claude will NOT respond with plain text before the tool call
3. `stop_reason` will be `"tool_use"` — always
4. The `input` field will conform to the tool's `input_schema`

This is the "programmatic enforcement over vague instruction" principle in its purest form. You are not asking Claude to use the tool — you are configuring the API to force it.

### The Two Layers of Enforcement

When you combine `tool_choice` with a well-defined `input_schema`, you get two layers of enforcement:

| Layer | Mechanism | What It Enforces |
|-------|-----------|-----------------|
| **Layer 1** | `tool_choice: {"type": "tool", "name": "..."}` | Claude MUST call the specified tool |
| **Layer 2** | `input_schema` (JSON Schema) | The tool's input MUST conform to the defined schema |

Together, these two layers guarantee:
- The tool is called (Layer 1)
- The output has the correct structure (Layer 2)
- Required fields are present (Layer 2 — `required` array)
- Field types are correct (Layer 2 — `type` constraints)
- Enum values are constrained (Layer 2 — `enum` constraints)

### Side-by-Side Comparison

| Approach | Reliability | Mechanism |
|----------|------------|-----------|
| "Please use the extract_data tool" in system prompt | ~80-90% | Vague instruction — Claude may or may not comply |
| `tool_choice: {"type": "auto"}` with good tool descriptions | ~90-95% | Claude usually picks the right tool, but not guaranteed |
| `tool_choice: {"type": "tool", "name": "extract_data"}` | **100%** | Programmatic enforcement — the API forces the tool call |

**Exam principle**: When the exam asks about the "most reliable" way to ensure structured output, the answer is always `tool_choice: {"type": "tool", "name": "..."}` combined with a JSON Schema `input_schema`. Never choose the option that relies on prompt instructions alone.

---

## 5. Combining tool_choice with stop_reason: "tool_use"

When `tool_choice` forces a tool call (using `any` or `tool`), the response will always have `stop_reason: "tool_use"`. This creates a **reliable extraction pipeline** — a predictable, deterministic flow from input to structured output.

### The Extraction Pipeline Pattern

```
┌─────────────────────────────────────────────────────────┐
│  1. Set tool_choice: {"type": "tool", "name": "..."}    │
│     → Guarantees Claude will call the specified tool     │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  2. Claude calls the tool                                │
│     → stop_reason is ALWAYS "tool_use"                   │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  3. Extract the tool_use block from response.content     │
│     → The tool's INPUT is your structured output         │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  4. Use the input field directly as structured data      │
│     → No parsing, no regex, no text extraction needed    │
└─────────────────────────────────────────────────────────┘
```

### Full Extraction Pipeline Example

This is the complete, production-ready pattern. The exam expects you to understand every part of this code:

```python
import anthropic
import json

client = anthropic.Anthropic()

# Step 1: Define the extraction tool with a strict schema
extract_invoice_tool = {
    "name": "extract_invoice",
    "description": "Extract structured invoice data from raw text. Extracts vendor name, invoice number, date, total amount, and individual line items.",
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor_name": {
                "type": "string",
                "description": "The name of the vendor or supplier"
            },
            "invoice_number": {
                "type": "string",
                "description": "The invoice identifier (e.g., INV-2024-001)"
            },
            "date": {
                "type": "string",
                "description": "Invoice date in YYYY-MM-DD format"
            },
            "total_amount": {
                "type": "number",
                "description": "Total invoice amount in USD"
            },
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "description": {"type": "string"},
                        "quantity": {"type": "integer"},
                        "unit_price": {"type": "number"}
                    },
                    "required": ["description", "quantity", "unit_price"]
                },
                "description": "Individual line items on the invoice"
            }
        },
        "required": ["vendor_name", "invoice_number", "date", "total_amount", "line_items"]
    }
}

# Step 2: Send the request with tool_choice forcing the extraction tool
invoice_text = """
Invoice #INV-2024-001
From: Acme Corporation
Date: March 15, 2024

Items:
- 5x Widget A @ $150.00 each = $750.00
- 2x Widget B @ $250.00 each = $500.00

Total: $1,250.00
"""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},  # FORCE the tool
    messages=[
        {"role": "user", "content": invoice_text}
    ]
)

# Step 3: Verify stop_reason (will always be "tool_use" when tool_choice forces it)
assert response.stop_reason == "tool_use", f"Unexpected stop_reason: {response.stop_reason}"

# Step 4: Extract the structured data from the tool_use block
tool_use_block = next(
    block for block in response.content
    if block.type == "tool_use"
)

# The tool's INPUT is your structured output
extracted_data = tool_use_block.input

# Step 5: Use the extracted data directly
print(f"Vendor: {extracted_data['vendor_name']}")
print(f"Invoice #: {extracted_data['invoice_number']}")
print(f"Date: {extracted_data['date']}")
print(f"Total: ${extracted_data['total_amount']}")
print(f"Line items: {len(extracted_data['line_items'])}")

# Output:
# Vendor: Acme Corporation
# Invoice #: INV-2024-001
# Date: 2024-03-15
# Total: $1250.0
# Line items: 2
```

### The Critical Insight: The Tool's Input IS Your Output

This is the conceptual leap that the exam tests. In a normal tool use flow:
- Claude calls a tool → you execute the tool → you return the result
- The tool's `input` is just parameters for execution

In the extraction pipeline pattern:
- Claude calls a tool → **you read the input and you're done**
- The tool's `input` IS the structured data you wanted
- You don't need to "execute" the tool at all — the input field contains the extracted data

The `input_schema` defines the **output format** of your extraction. The tool is not a function to execute — it is a **schema enforcement mechanism**.

---

## 6. The "Tool as Structured Output" Pattern

This pattern takes the extraction pipeline concept to its logical conclusion: you define a tool whose sole purpose is to structure Claude's output. The tool doesn't correspond to any real function — it exists only to enforce a schema.

### Defining a Schema-Only Tool

```python
# This tool doesn't DO anything — it's a schema enforcement mechanism
sentiment_analysis_tool = {
    "name": "record_sentiment",
    "description": "Record the sentiment analysis results for a piece of text. Analyze the text and record the overall sentiment, confidence score, key phrases, and any detected emotions.",
    "input_schema": {
        "type": "object",
        "properties": {
            "sentiment": {
                "type": "string",
                "enum": ["positive", "negative", "neutral", "mixed"],
                "description": "Overall sentiment of the text"
            },
            "confidence": {
                "type": "number",
                "description": "Confidence score between 0.0 and 1.0"
            },
            "key_phrases": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Key phrases that influenced the sentiment determination"
            },
            "emotions": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["joy", "anger", "sadness", "fear", "surprise", "disgust", "trust", "anticipation"]
                },
                "description": "Detected emotions in the text"
            },
            "summary": {
                "type": "string",
                "description": "One-sentence summary of the sentiment analysis"
            }
        },
        "required": ["sentiment", "confidence", "key_phrases", "emotions", "summary"]
    }
}
```

### Using the Schema-Only Tool

```python
def analyze_sentiment(text):
    """Analyze sentiment using Claude with guaranteed structured output."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=[sentiment_analysis_tool],
        tool_choice={"type": "tool", "name": "record_sentiment"},
        messages=[
            {"role": "user", "content": f"Analyze the sentiment of this text:\n\n{text}"}
        ]
    )

    # Extract the structured result — no tool execution needed
    tool_use_block = next(
        block for block in response.content
        if block.type == "tool_use"
    )

    return tool_use_block.input  # This IS the structured output

# Use it
result = analyze_sentiment("The product is amazing! Best purchase I've made all year. Shipping was a bit slow though.")

print(result)
# {
#   "sentiment": "positive",
#   "confidence": 0.85,
#   "key_phrases": ["amazing", "best purchase", "shipping was a bit slow"],
#   "emotions": ["joy", "anticipation"],
#   "summary": "Overwhelmingly positive sentiment about the product quality, with minor dissatisfaction about shipping speed."
# }
```

### Why This Pattern Works

1. **`tool_choice` forces the tool call** — Claude must call `record_sentiment`
2. **`input_schema` defines the output structure** — Claude must populate all required fields with the correct types
3. **`enum` constraints limit values** — Sentiment must be one of four values; emotions must be from the predefined list
4. **No tool execution needed** — You read `tool_use_block.input` and you're done
5. **No text parsing needed** — The output is already structured JSON

### Another Example: Entity Extraction

```python
entity_extraction_tool = {
    "name": "extract_entities",
    "description": "Extract named entities from text. Identify all people, organizations, locations, dates, and monetary amounts mentioned in the text.",
    "input_schema": {
        "type": "object",
        "properties": {
            "people": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Names of people mentioned"
            },
            "organizations": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Names of organizations, companies, or institutions"
            },
            "locations": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Geographic locations (cities, countries, addresses)"
            },
            "dates": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Dates mentioned, normalized to YYYY-MM-DD format"
            },
            "monetary_amounts": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "amount": {"type": "number"},
                        "currency": {"type": "string"}
                    },
                    "required": ["amount", "currency"]
                },
                "description": "Monetary amounts with currency"
            }
        },
        "required": ["people", "organizations", "locations", "dates", "monetary_amounts"]
    }
}

def extract_entities(text):
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=[entity_extraction_tool],
        tool_choice={"type": "tool", "name": "extract_entities"},
        messages=[
            {"role": "user", "content": text}
        ]
    )

    tool_use_block = next(
        block for block in response.content
        if block.type == "tool_use"
    )

    return tool_use_block.input

# Use it
entities = extract_entities(
    "On January 15, 2024, CEO Jane Smith announced that Acme Corp "
    "will invest $50 million in their new London headquarters."
)

# entities = {
#   "people": ["Jane Smith"],
#   "organizations": ["Acme Corp"],
#   "locations": ["London"],
#   "dates": ["2024-01-15"],
#   "monetary_amounts": [{"amount": 50000000, "currency": "USD"}]
# }
```

### The Pattern Summary

| Component | Role |
|-----------|------|
| Tool definition | Defines the output schema (not a real function) |
| `tool_choice: {"type": "tool", "name": "..."}` | Forces Claude to call the tool |
| `input_schema` | Defines the exact structure of the output |
| `tool_use_block.input` | Contains the structured output — read it directly |
| Tool execution | **Not needed** — the input IS the output |

---


## 7. Behavior Differences with tool_choice

Different `tool_choice` values change Claude's response behavior in ways that go beyond just "which tool gets called." Understanding these behavioral differences is critical for building reliable applications and for the exam.

### Natural Language Output Behavior

| tool_choice Value | Natural Language Before Tool Call? | Explanation |
|-------------------|-----------------------------------|-------------|
| `auto` | **Yes** — Claude may include text before the tool_use block | Claude explains what it is doing before calling the tool |
| `any` | **No** — Claude goes straight to the tool_use block | The API prefills the response to force an immediate tool call |
| `tool` | **No** — Claude goes straight to the tool_use block | The API prefills the response to force an immediate tool call |
| `none` | N/A — no tool call possible | Claude responds with text only |

### What "Prefilling" Means

When `tool_choice` is set to `any` or `tool`, the API internally prefills the beginning of Claude's response to force a tool call. This is similar to how you can insert synthetic assistant messages to steer Claude's behavior (covered on Day 1). The prefill ensures Claude starts generating a tool_use block immediately, without any preceding text.

**Practical implication**: If you set `tool_choice: {"type": "tool", "name": "extract_data"}`, the `content` array in the response will contain ONLY a `tool_use` block — no text blocks. This simplifies your parsing code:

```python
# With tool_choice: "tool" — content has only the tool_use block
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_data_tool],
    tool_choice={"type": "tool", "name": "extract_data"},
    messages=[{"role": "user", "content": document_text}]
)

# Safe to access directly — only one block in content
tool_use_block = response.content[0]
assert tool_use_block.type == "tool_use"
extracted = tool_use_block.input
```

Compare this with `auto`, where you must iterate through the content array:

```python
# With tool_choice: "auto" — content may have text AND tool_use blocks
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_data_tool],
    # tool_choice defaults to auto
    messages=[{"role": "user", "content": document_text}]
)

# Must iterate — content may have multiple blocks
for block in response.content:
    if block.type == "tool_use":
        extracted = block.input
    elif block.type == "text":
        explanation = block.text  # Claude's explanation before the tool call
```

### Extended Thinking Compatibility

Extended thinking (where Claude shows its reasoning process in `thinking` blocks) has specific compatibility constraints with `tool_choice`:

| tool_choice Value | Extended Thinking Compatible? |
|-------------------|------------------------------|
| `auto` | **Yes** — Claude can think before deciding whether to use a tool |
| `none` | **Yes** — Claude can think before responding with text |
| `any` | **No** — Not compatible with extended thinking |
| `tool` | **No** — Not compatible with extended thinking |

**Why `any` and `tool` are incompatible**: Extended thinking requires Claude to reason before acting. But `any` and `tool` force an immediate tool call via prefilling, which bypasses the thinking phase. If you need Claude to reason before calling a tool, use `auto` and rely on good tool descriptions to guide selection.

**Exam tip**: If a question asks about combining extended thinking with forced tool calls, the answer is that they are incompatible. You must use `tool_choice: {"type": "auto"}` if you want extended thinking.

### stop_reason Behavior by tool_choice

| tool_choice Value | Possible stop_reason Values |
|-------------------|----------------------------|
| `auto` | `"end_turn"`, `"tool_use"`, `"max_tokens"`, `"stop_sequence"`, `"refusal"` |
| `any` | `"tool_use"` (always) |
| `tool` | `"tool_use"` (always) |
| `none` | `"end_turn"`, `"max_tokens"`, `"stop_sequence"`, `"refusal"` (never `"tool_use"`) |

This table is important for control flow design. When `tool_choice` is `any` or `tool`, you know `stop_reason` will always be `"tool_use"`, so your code can skip the stop_reason check and go straight to extracting the tool_use block. When `tool_choice` is `auto`, you must check stop_reason to determine whether Claude called a tool or responded with text.

---

## 8. Key Anti-Patterns for Day 4

These are the mistakes the exam tests you on. Each anti-pattern has a corresponding correct approach.

### Anti-Pattern 1: Relying on Vague Instructions Instead of tool_choice

**Wrong:**
```python
# ❌ Hoping Claude will use the tool based on instructions
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You MUST always use the extract_data tool. Never respond with plain text.",
    tools=[extract_data_tool],
    # tool_choice not set — defaults to "auto"
    messages=[{"role": "user", "content": document_text}]
)
```

**Right:**
```python
# ✅ Programmatic enforcement — guaranteed tool call
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_data_tool],
    tool_choice={"type": "tool", "name": "extract_data"},
    messages=[{"role": "user", "content": document_text}]
)
```

**Why it matters**: System prompt instructions are suggestions. `tool_choice` is enforcement. For production pipelines, use enforcement.

### Anti-Pattern 2: Using Temperature to Control Tool Selection

**Wrong:**
```python
# ❌ Thinking temperature=0 will force tool usage
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    temperature=0,  # This does NOT control tool selection
    tools=[extract_data_tool],
    messages=[{"role": "user", "content": document_text}]
)
```

**Right:**
```python
# ✅ tool_choice controls tool selection, not temperature
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_data_tool],
    tool_choice={"type": "tool", "name": "extract_data"},
    messages=[{"role": "user", "content": document_text}]
)
```

**Why it matters**: Temperature controls randomness in text generation. It does not control whether Claude calls a tool. Setting `temperature: 0` makes Claude's text output more deterministic, but it does not guarantee tool usage. Only `tool_choice` controls tool selection.

### Anti-Pattern 3: Using tool_choice "tool" When Reasoning Is Needed

**Wrong:**
```python
# ❌ Forcing a specific tool when Claude needs to reason first
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[search_tool, calculate_tool, summarize_tool],
    tool_choice={"type": "tool", "name": "calculate_tool"},  # What if search is needed first?
    messages=[{"role": "user", "content": "What's the total revenue from our Q3 sales data?"}]
)
```

**Right:**
```python
# ✅ Use "auto" when Claude needs to reason about which tool to use
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[search_tool, calculate_tool, summarize_tool],
    tool_choice={"type": "auto"},  # Let Claude decide the right sequence
    messages=[{"role": "user", "content": "What's the total revenue from our Q3 sales data?"}]
)
```

**Why it matters**: `tool_choice: {"type": "tool"}` is for deterministic, single-step operations like extraction. When Claude needs to reason about which tool to use, or needs to call multiple tools in sequence, use `auto` so Claude can make intelligent decisions.

### Anti-Pattern 4: Not Understanding That Forced Tool Calls Suppress Text

**Wrong:**
```python
# ❌ Expecting explanatory text with forced tool calls
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_data_tool],
    tool_choice={"type": "tool", "name": "extract_data"},
    messages=[{"role": "user", "content": document_text}]
)

# This will fail — there's no text block when tool_choice forces a tool
explanation = response.content[0].text  # ERROR: content[0] is a tool_use block, not text
```

**Right:**
```python
# ✅ When tool_choice forces a tool, expect only tool_use blocks
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extract_data_tool],
    tool_choice={"type": "tool", "name": "extract_data"},
    messages=[{"role": "user", "content": document_text}]
)

# Access the tool_use block directly
tool_use_block = response.content[0]
assert tool_use_block.type == "tool_use"
extracted = tool_use_block.input
```

**Why it matters**: When `tool_choice` is `any` or `tool`, Claude does not emit natural language before the tool call. The `content` array contains only the `tool_use` block. Code that assumes a text block at `content[0]` will break.

### Anti-Pattern 5: Forcing a Tool That Doesn't Exist in tools[]

**Wrong:**
```python
# ❌ tool_choice references a tool not in the tools[] array
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[get_weather_tool],  # Only get_weather is defined
    tool_choice={"type": "tool", "name": "get_forecast"},  # get_forecast doesn't exist!
    messages=[{"role": "user", "content": "What's the weather?"}]
)
# API ERROR: tool_choice references undefined tool
```

**Right:**
```python
# ✅ tool_choice name must match a tool in the tools[] array
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[get_weather_tool],
    tool_choice={"type": "tool", "name": "get_weather"},  # Matches tools[] definition
    messages=[{"role": "user", "content": "What's the weather?"}]
)
```

**Why it matters**: The `name` in `tool_choice: {"type": "tool", "name": "..."}` must exactly match a tool name defined in the `tools[]` array. A mismatch causes an API error.

---

## 9. Exam-Relevant Principles

### Principle 1: Programmatic Enforcement Over Vague Instruction

This is THE recurring exam theme, and `tool_choice` is its primary mechanism.

| Vague Instruction (Anti-Pattern) | Programmatic Enforcement (Correct) |
|----------------------------------|-----------------------------------|
| "Please use the extract_data tool" | `tool_choice: {"type": "tool", "name": "extract_data"}` |
| "Always return JSON" | `input_schema` with JSON Schema |
| "Please classify into one of these categories" | `enum` constraint in the schema |
| "Make sure to include all required fields" | `required` array in the schema |

The exam will present scenarios where one option uses vague instructions and another uses `tool_choice`. Always choose `tool_choice`.

### Principle 2: tool_choice + JSON Schema = Guaranteed Structured Output

This is the gold standard for reliable extraction pipelines:

```
tool_choice: {"type": "tool", "name": "extract_data"}
    → Guarantees the tool is called

input_schema: { JSON Schema definition }
    → Guarantees the output structure

Together:
    → Guaranteed structured output every time
```

No other combination provides this level of reliability. Not system prompts. Not few-shot examples. Not temperature settings. Only `tool_choice` + JSON Schema.

### Principle 3: tool_choice "tool" + stop_reason "tool_use" = Reliable Pipeline

When you set `tool_choice: {"type": "tool", "name": "..."}`:
- `stop_reason` is always `"tool_use"`
- The `content` array contains only the `tool_use` block
- The `input` field contains your structured data

This creates a pipeline with zero ambiguity:
1. Send the request
2. Read `response.content[0].input`
3. Done

### Principle 4: The Tool's Input IS Your Output

In the "tool as structured output" pattern:
- The tool definition is a schema, not a function
- The `input_schema` defines your output format
- The `tool_use_block.input` contains your structured data
- You don't execute the tool — you read the input

This conceptual inversion is a key exam concept. The exam may describe a scenario where a developer defines a tool, forces it with `tool_choice`, and then asks "what do you do with the tool call?" The answer is: read the `input` field. You don't need to execute anything.

---

## 10. Quick Reference Card

### tool_choice Values at a Glance

| Value | Claude Must Call a Tool? | Claude Chooses Which Tool? | Natural Language Before Tool? | Extended Thinking? | Default? |
|-------|-------------------------|---------------------------|------------------------------|-------------------|----------|
| `{"type": "auto"}` | No — Claude decides | Yes (if it calls one) | Yes | Yes | Yes (when `tools` provided) |
| `{"type": "any"}` | **Yes** — must call one | Yes — picks the best fit | **No** | **No** | No |
| `{"type": "tool", "name": "X"}` | **Yes** — must call tool X | No — forced to use X | **No** | **No** | No |
| `{"type": "none"}` | **No** — cannot call any | N/A | N/A | Yes | Yes (when no `tools` provided) |

### Request Structure with tool_choice

```python
client.messages.create(
    model="claude-sonnet-4-20250514",       # Required: which model
    max_tokens=1024,                         # Required: output token ceiling
    system="You are a...",                   # Optional: system prompt
    tools=[                                  # Optional: available tools
        {
            "name": "tool_name",
            "description": "What the tool does...",
            "input_schema": { ... }
        }
    ],
    tool_choice={"type": "auto"},            # Optional: how to use tools
    messages=[                               # Required: conversation history
        {"role": "user", "content": "..."}
    ]
)
```

### Extraction Pipeline Cheat Sheet

```python
# 1. Define the tool (schema only — no real function needed)
tool = {
    "name": "extract_X",
    "description": "Extract X from text...",
    "input_schema": { "type": "object", "properties": {...}, "required": [...] }
}

# 2. Force the tool call
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[tool],
    tool_choice={"type": "tool", "name": "extract_X"},
    messages=[{"role": "user", "content": raw_text}]
)

# 3. Read the structured output
assert response.stop_reason == "tool_use"
structured_data = response.content[0].input
```

### stop_reason by tool_choice

| tool_choice | stop_reason Will Be |
|-------------|-------------------|
| `auto` | `"end_turn"` OR `"tool_use"` (Claude decides) |
| `any` | Always `"tool_use"` |
| `tool` | Always `"tool_use"` |
| `none` | Never `"tool_use"` |

### Decision Framework

```
Do you need Claude to ALWAYS call a tool?
├── No → Use "auto" (default)
└── Yes
    ├── Do you care WHICH tool?
    │   ├── No → Use "any"
    │   └── Yes → Use {"type": "tool", "name": "specific_tool"}
    └── Do you need to DISABLE tools temporarily?
        └── Yes → Use "none"
```

---

## 11. Scenario Challenge

### Scenario Challenge: Domain 2 — Tool Design & MCP Integration

**Scenario**: You are building a document processing pipeline that extracts structured data from customer feedback forms. Each form contains a customer name, email, satisfaction rating (1-5), feedback category (product, service, shipping, other), and free-text comments. Your pipeline must process thousands of forms per day with 100% structural consistency — every output must have the same fields in the same format.

During testing, you notice that approximately 10% of the time, Claude responds with a natural language summary of the feedback instead of calling the extraction tool. Your current implementation defines the extraction tool in `tools[]` but does not set `tool_choice`.

**Question 1**: What is the most reliable way to ensure Claude always calls the extraction tool?

A) Add "You must always use the extract_feedback tool. Never respond with plain text." to the system prompt

B) Set `tool_choice: {"type": "any"}` to force Claude to use one of the available tools

C) Set `tool_choice: {"type": "tool", "name": "extract_feedback"}` to force Claude to call the specific extraction tool

D) Set `temperature: 0` to make Claude's behavior deterministic and always choose the tool

---

**Correct Answer**: **C**

**Explanation**: Setting `tool_choice: {"type": "tool", "name": "extract_feedback"}` programmatically forces Claude to call the `extract_feedback` tool on every request. This is the "programmatic enforcement over vague instruction" principle — you are not asking Claude to use the tool, you are configuring the API to force it. Combined with the JSON Schema in the tool's `input_schema`, this guarantees both that the tool is called AND that the output matches your schema.

- **Why A is wrong**: System prompt instructions are vague instructions (anti-pattern). Claude will usually follow them, but "usually" is not 100%. For a pipeline processing thousands of forms, even a 1% failure rate means dozens of malformed outputs per day.
- **Why B is wrong**: `tool_choice: {"type": "any"}` forces a tool call, but if you have multiple tools defined, Claude might call the wrong one. Option C is more precise because it forces the specific tool you need.
- **Why D is wrong**: Temperature controls randomness in text generation, not tool selection. Setting `temperature: 0` makes text output more deterministic but does not guarantee Claude will call a tool. Only `tool_choice` controls tool selection.

---

**Question 2**: After implementing the fix from Question 1, a colleague asks: "If we're forcing Claude to call the tool, do we still need to check `stop_reason` in our code?" What is the correct answer?

A) Yes — `stop_reason` could still be `"end_turn"` if Claude decides the input doesn't look like a feedback form

B) No — when `tool_choice` forces a tool call, `stop_reason` is always `"tool_use"`, so the check is redundant but harmless as defensive coding

C) Yes — `stop_reason` could be `"max_tokens"` if the extraction is too complex

D) No — `stop_reason` is deprecated when `tool_choice` is set

---

**Correct Answer**: **B**

**Explanation**: When `tool_choice` is set to `{"type": "tool", "name": "..."}`, the API forces Claude to call the specified tool. The `stop_reason` will always be `"tool_use"`. Checking it is technically redundant, but it is good defensive coding practice — it costs nothing and protects against unexpected API behavior changes.

- **Why A is wrong**: With `tool_choice: {"type": "tool"}`, Claude cannot choose to respond with text instead. The API forces the tool call regardless of the input content.
- **Why C is wrong**: When `tool_choice` forces a tool call, Claude generates the tool_use block. If `max_tokens` is reached during generation, the response may be truncated, but the `stop_reason` mechanism with forced tool choice ensures the tool call structure is maintained. In practice, extraction tool inputs are much smaller than typical `max_tokens` values.
- **Why D is wrong**: `stop_reason` is not deprecated. It is always present in the response and always meaningful.

---

**Question 3**: Your pipeline needs to handle a new requirement: after extracting the feedback data, Claude should generate a one-paragraph summary of the customer's experience in natural language. The summary should NOT trigger any tool calls. How should you structure this two-step workflow?

A) Define a second tool called `generate_summary` and set `tool_choice: {"type": "any"}` so Claude can pick between extraction and summarization

B) Make two separate API calls: first with `tool_choice: {"type": "tool", "name": "extract_feedback"}` for extraction, then with `tool_choice: {"type": "none"}` for the summary

C) Add "After extracting the data, please also write a summary paragraph" to the system prompt and keep `tool_choice: {"type": "tool", "name": "extract_feedback"}`

D) Set `tool_choice: {"type": "auto"}` and let Claude decide when to extract and when to summarize

---

**Correct Answer**: **B**

**Explanation**: The two-step workflow requires different `tool_choice` settings for each step. The first call uses `tool_choice: {"type": "tool", "name": "extract_feedback"}` to force structured extraction. The second call uses `tool_choice: {"type": "none"}` to disable tools and get a natural language summary. This is the correct pattern for workflows that mix structured output with free-text generation.

- **Why A is wrong**: `tool_choice: {"type": "any"}` forces a tool call on every request. You cannot get a natural language summary when a tool call is forced. Also, having Claude choose between extraction and summarization introduces the unreliability you're trying to avoid.
- **Why C is wrong**: When `tool_choice: {"type": "tool", "name": "extract_feedback"}` is set, Claude is forced to call the extraction tool. It cannot also generate a text summary in the same response — the forced tool call suppresses natural language output.
- **Why D is wrong**: `auto` reintroduces the original problem — Claude might not call the extraction tool 100% of the time. For a pipeline requiring structural consistency, `auto` is insufficient.

**Domain Reference**: Domain 2: Tool Design & MCP Integration (18% weighting)
**Exam Principles Tested**: Programmatic enforcement over vague instruction; tool_choice + JSON Schema = guaranteed structured output; tool_choice "tool" + stop_reason "tool_use" = reliable extraction pipeline

---

## Sources

All technical content in this document is based on the official Anthropic documentation. Content was rephrased for compliance with licensing restrictions.

- [Tool use with Claude](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview) — Tool definitions, tool_choice parameter, and structured output patterns
- [Forcing tool use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview#forcing-tool-use) — tool_choice values (auto, any, tool), behavior differences, and extraction pipeline patterns
- [Messages API Reference](https://docs.anthropic.com/en/api/messages) — API request parameters including tool_choice specification
- [Extended thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking) — Compatibility constraints with tool_choice values