# Day 3 — tool_use Message Structure

**Domain**: Domain 2: Tool Design & MCP Integration (18% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Begin Building with the Claude API (Pod 1)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [The Tool Definition in the tools[] Array](#1-the-tool-definition-in-the-tools-array)
2. [The tool_use Content Block](#2-the-tool_use-content-block)
3. [The tool_result Content Block](#3-the-tool_result-content-block)
4. [Multi-Turn Conversation Flow with Tool Calls](#4-multi-turn-conversation-flow-with-tool-calls)
5. [How Claude Decides Which Tool to Call](#5-how-claude-decides-which-tool-to-call)
6. [Structuring Tool Descriptions for Clarity and Disambiguation](#6-structuring-tool-descriptions-for-clarity-and-disambiguation)
7. [Where Tools Run — Client vs Server](#7-where-tools-run--client-vs-server)
8. [Key Anti-Patterns for Day 3](#8-key-anti-patterns-for-day-3)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## 1. The Tool Definition in the tools[] Array

Before Claude can use a tool, you must define it. Tool definitions live in the `tools` parameter of the API request — a top-level array alongside `model`, `max_tokens`, `messages`, and `system`. Each element in the `tools[]` array describes one tool that Claude is allowed to call.

### Tool Definition Structure

Every tool definition has three fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the tool. Must match `^[a-zA-Z0-9_-]{1,64}$` |
| `description` | string | Yes | Explains what the tool does, when to use it, and what it returns. **This is the most important field.** |
| `input_schema` | object | Yes | A JSON Schema object defining the tool's expected parameters |

### Example Tool Definition

```json
{
  "name": "get_weather",
  "description": "Get the current weather conditions for a specific location. Use this tool when the user asks about weather, temperature, or atmospheric conditions for a city or region. Returns temperature in Fahrenheit, conditions (sunny, cloudy, rainy, etc.), and humidity percentage. Only supports US cities.",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "The city and state, e.g. 'San Francisco, CA'"
      }
    },
    "required": ["location"]
  }
}
```

### The `name` Field

The tool name is how Claude references the tool when it decides to call it. It must:

- Match the regex pattern `^[a-zA-Z0-9_-]{1,64}$`
- Contain only letters, numbers, underscores, and hyphens
- Be between 1 and 64 characters long
- Be unique within the `tools[]` array

**Good names**: `get_weather`, `search_database`, `github_list_prs`, `send-email`
**Bad names**: `get weather` (space), `tool.search` (dot), `a` (too vague, though technically valid)

**Exam tip**: Use meaningful, namespaced names when you have multiple tools from different domains. For example, `github_list_prs` and `slack_send_message` are immediately clear about their scope. This helps both Claude and human developers understand the tool's purpose.

### The `description` Field — The Most Important Field

The `description` is the **primary mechanism** Claude uses to decide whether and when to call a tool. A vague or incomplete description leads to misrouted tool calls, missed tool calls, or incorrect parameter values.

**Exam principle**: Tool descriptions are the single most important factor in tool selection. The exam tests whether you can write descriptions that are specific enough for Claude to choose the right tool in ambiguous situations.

#### What a Good Description Includes

A good tool description answers four questions in 3–4+ sentences:

1. **What does the tool do?** — The core functionality
2. **When should Claude use it?** — The trigger conditions
3. **What do the parameters mean?** — Clarification beyond the schema
4. **What are the limitations?** — Caveats, restrictions, edge cases

#### Good vs. Bad Tool Descriptions

**Bad description** (vague, incomplete):
```json
{
  "name": "search",
  "description": "Searches for things."
}
```

Problems: What does it search? When should Claude use it vs. other tools? What format should the query be in? What does it return?

**Good description** (specific, actionable):
```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal company knowledge base for articles, FAQs, and documentation. Use this tool when the user asks a question about company policies, product features, or internal processes. The query should be a natural language question or keyword phrase. Returns up to 5 matching articles ranked by relevance, each with a title, snippet, and URL. Does NOT search external websites or the public internet — use web_search for that."
}
```

This description tells Claude exactly what the tool does, when to use it (and when NOT to), what the query format should be, what it returns, and how it differs from similar tools.

**Another bad description** (too terse):
```json
{
  "name": "get_customer",
  "description": "Gets customer info."
}
```

**Good version**:
```json
{
  "name": "get_customer",
  "description": "Retrieve detailed customer profile information by email address. Use this tool when you need to look up a customer's account details, order history summary, or contact information. The email parameter must be a valid email address. Returns customer_id, name, email, account_status, and a summary of recent orders. Returns an error if no customer is found with the given email."
}
```

### The `input_schema` Field

The `input_schema` is a standard JSON Schema object that defines the parameters Claude must provide when calling the tool. Claude generates the `input` values based on this schema.

#### Basic Schema Structure

```json
{
  "type": "object",
  "properties": {
    "param_name": {
      "type": "string",
      "description": "What this parameter means and expected format"
    }
  },
  "required": ["param_name"]
}
```

#### Supported JSON Schema Types

| Type | Example | Use Case |
|------|---------|----------|
| `string` | `"San Francisco"` | Names, IDs, free text |
| `number` | `42.5` | Quantities, amounts |
| `integer` | `10` | Counts, indices |
| `boolean` | `true` | Flags, toggles |
| `array` | `["a", "b"]` | Lists of items |
| `object` | `{"key": "val"}` | Nested structures |
| `enum` | `"asc"` or `"desc"` | Constrained choices |

#### Schema with Multiple Parameters

```json
{
  "type": "object",
  "properties": {
    "customer_id": {
      "type": "string",
      "description": "The unique customer identifier (format: CUST-XXXXX)"
    },
    "order_id": {
      "type": "string",
      "description": "The order identifier to look up (format: ORD-XXXXX)"
    },
    "include_line_items": {
      "type": "boolean",
      "description": "Whether to include individual line items in the response. Defaults to false."
    }
  },
  "required": ["customer_id", "order_id"]
}
```

Note that `include_line_items` is not in the `required` array — it is optional. Claude may or may not include it in the tool call.

#### Schema with Enum Constraints

Use `enum` to restrict a parameter to a fixed set of values:

```json
{
  "type": "object",
  "properties": {
    "sort_order": {
      "type": "string",
      "enum": ["asc", "desc"],
      "description": "Sort order for results. 'asc' for ascending (oldest first), 'desc' for descending (newest first)."
    }
  },
  "required": ["sort_order"]
}
```

#### Schema with Nested Objects

```json
{
  "type": "object",
  "properties": {
    "filter": {
      "type": "object",
      "properties": {
        "status": {
          "type": "string",
          "enum": ["active", "inactive", "pending"]
        },
        "created_after": {
          "type": "string",
          "description": "ISO 8601 date string (e.g., '2024-01-15')"
        }
      },
      "required": ["status"]
    }
  },
  "required": ["filter"]
}
```

### Full API Request with Tool Definitions

Here is a complete API request that includes tool definitions:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get the current weather conditions for a specific location. Use this when the user asks about weather, temperature, or atmospheric conditions. Returns temperature (Fahrenheit), conditions, and humidity. Only supports US cities.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. 'San Francisco, CA'"
                    }
                },
                "required": ["location"]
            }
        }
    ],
    messages=[
        {"role": "user", "content": "What's the weather like in San Francisco?"}
    ]
)
```

---


## 2. The tool_use Content Block

When Claude decides to call a tool, it includes a `tool_use` content block in its response. This block tells your application which tool to execute and what arguments to pass. The response will have `stop_reason: "tool_use"` to signal that Claude is waiting for tool results.

### tool_use Block Structure

```json
{
  "type": "tool_use",
  "id": "toolu_01A09q90qw90lq917835lq9",
  "name": "get_weather",
  "input": {"location": "San Francisco, CA"}
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"tool_use"` |
| `id` | string | A unique identifier for this specific tool call. Format: `toolu_...`. You MUST use this ID when returning the tool result. |
| `name` | string | The name of the tool Claude wants to call. Matches one of the tools defined in your `tools[]` array. |
| `input` | object | The arguments for the tool call, as a JSON object. The keys and values match the tool's `input_schema`. |

### The `id` Field

The `id` is a unique identifier generated by Claude for each tool call. It follows the format `toolu_` followed by a random alphanumeric string (e.g., `toolu_01A09q90qw90lq917835lq9`).

**Why it matters**: When you return the tool result, you must include this exact `id` in the `tool_use_id` field of the `tool_result` block. This is how Claude matches results to their corresponding tool calls — especially important when Claude makes multiple tool calls in a single response.

**Exam tip**: Mismatching `tool_use_id` values is a tested anti-pattern. If Claude calls two tools in one response, each `tool_result` must reference the correct `id`.

### The `name` Field

The `name` field tells you which tool Claude wants to call. It will always match one of the tool names you defined in the `tools[]` array. If you defined a tool named `get_weather`, Claude will use exactly that string.

### The `input` Field

The `input` field contains the arguments Claude generated for the tool call. The structure matches the `input_schema` you defined for the tool:

- Required parameters from the schema will always be present
- Optional parameters may or may not be included
- Values conform to the types specified in the schema (string, number, boolean, etc.)

### Full Response with tool_use

When Claude decides to use a tool, the full API response looks like this:

```json
{
  "id": "msg_01234",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll look up the current weather in San Francisco for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "get_weather",
      "input": {"location": "San Francisco, CA"}
    }
  ],
  "model": "claude-sonnet-4-20250514",
  "stop_reason": "tool_use",
  "usage": {
    "input_tokens": 150,
    "output_tokens": 45
  }
}
```

**Key observations:**

1. **`stop_reason` is `"tool_use"`** — This tells your application that Claude is waiting for tool results, not that it has finished responding.
2. **The `content` array contains BOTH a text block AND a tool_use block** — Claude often explains what it is doing before making the tool call. Your code must iterate through the content array and handle each block type.
3. **The text block comes BEFORE the tool_use block** — Claude's natural language explanation precedes the structured tool call.

### Extracting tool_use Blocks from the Response

```python
# Extract all tool_use blocks from the response
tool_use_blocks = [
    block for block in response.content
    if block.type == "tool_use"
]

# Process each tool call
for tool_call in tool_use_blocks:
    tool_name = tool_call.name       # e.g., "get_weather"
    tool_input = tool_call.input     # e.g., {"location": "San Francisco, CA"}
    tool_id = tool_call.id           # e.g., "toolu_01A09q90qw90lq917835lq9"

    # Execute the tool and collect the result
    result = execute_tool(tool_name, tool_input)
```

### Multiple tool_use Blocks in One Response

Claude can request multiple tool calls in a single response. For example, if the user asks "What's the weather in San Francisco and New York?", Claude might respond with:

```json
{
  "content": [
    {
      "type": "text",
      "text": "I'll check the weather in both cities for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_01ABC",
      "name": "get_weather",
      "input": {"location": "San Francisco, CA"}
    },
    {
      "type": "tool_use",
      "id": "toolu_02DEF",
      "name": "get_weather",
      "input": {"location": "New York, NY"}
    }
  ],
  "stop_reason": "tool_use"
}
```

Each tool_use block has a **unique `id`**. When you return results, you must provide a separate `tool_result` for each one, matching the `tool_use_id` to the correct `id`.

---

## 3. The tool_result Content Block

After you execute a tool, you send the result back to Claude in a `tool_result` content block. This block goes inside a **`user` message** — not an assistant message. This is because from the API's perspective, tool execution happens on your side (the client), and you are reporting the result back to Claude.

### tool_result Block Structure

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
  "content": "The current weather in San Francisco is 65°F, partly cloudy, with 72% humidity."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"tool_result"` |
| `tool_use_id` | string | Yes | Must match the `id` from the corresponding `tool_use` block exactly |
| `content` | string or array | No | The result of the tool execution. Can be a plain string or an array of content blocks. |
| `is_error` | boolean | No | Set to `true` when the tool execution failed. Defaults to `false`. |

### The `tool_use_id` Field

This is the critical linking field. It must **exactly match** the `id` from the `tool_use` block that triggered this result. If Claude made a tool call with `id: "toolu_01A09q90qw90lq917835lq9"`, your `tool_result` must have `tool_use_id: "toolu_01A09q90qw90lq917835lq9"`.

**What happens if they don't match**: The API will return an error. Claude cannot process a tool result that doesn't correspond to a tool call it made.

### The `content` Field

The `content` field carries the tool's output. It supports two formats:

#### String Content (Simple)

For straightforward text results:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "content": "Temperature: 65°F, Conditions: Partly Cloudy, Humidity: 72%"
}
```

#### Array Content (Rich)

For results that include multiple content types (text, images, etc.):

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "content": [
    {
      "type": "text",
      "text": "Weather data retrieved successfully."
    },
    {
      "type": "text",
      "text": "Temperature: 65°F\nConditions: Partly Cloudy\nHumidity: 72%"
    }
  ]
}
```

You can also include image content blocks in the array if the tool returns visual data.

### The `is_error` Field

When a tool execution fails, set `is_error: true` and put the error message in `content`. This tells Claude that the tool call did not succeed, so it can adjust its approach — retry with different parameters, try a different tool, or inform the user.

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "content": "Error: Location 'Atlantis' not found. Please provide a valid US city and state.",
  "is_error": true
}
```

**Why `is_error` matters**: Without this flag, Claude treats the error message as a successful result and may try to interpret the error text as actual data. With `is_error: true`, Claude understands the tool failed and can respond appropriately.

### tool_result Goes in a user Message

This is a critical structural rule. The `tool_result` block is placed inside a `user` role message, not an `assistant` message:

```python
messages = [
    # Original user message
    {"role": "user", "content": "What's the weather in San Francisco?"},

    # Claude's response with tool_use (include the FULL response)
    {"role": "assistant", "content": [
        {"type": "text", "text": "I'll look up the weather for you."},
        {"type": "tool_use", "id": "toolu_01ABC", "name": "get_weather",
         "input": {"location": "San Francisco, CA"}}
    ]},

    # YOUR tool result — in a USER message
    {"role": "user", "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC",
            "content": "65°F, partly cloudy, 72% humidity"
        }
    ]}
]
```

**Why user role?** Because tool execution happens on your side. You are the "user" reporting back what happened when you ran the tool. Claude made the request (assistant), you executed it and returned the result (user).

### Returning Results for Multiple Tool Calls

When Claude makes multiple tool calls in one response, you return all results in a single `user` message:

```python
{
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC",
            "content": "San Francisco: 65°F, partly cloudy"
        },
        {
            "type": "tool_result",
            "tool_use_id": "toolu_02DEF",
            "content": "New York: 45°F, rainy"
        }
    ]
}
```

Each `tool_result` matches its corresponding `tool_use` block by `tool_use_id`.

---


## 4. Multi-Turn Conversation Flow with Tool Calls

The tool use flow follows a specific pattern that the exam calls the **agentic loop**. Understanding this loop is foundational for Domain 1 (Agentic Architecture & Orchestration) and Domain 2 (Tool Design & MCP Integration).

### The Complete Agentic Loop

```
┌─────────────────────────────────────────────────────┐
│  1. Send request with tools[] and user message       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  2. Claude responds with stop_reason: "tool_use"     │
│     and one or more tool_use blocks                  │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  3. Execute each tool, format outputs as             │
│     tool_result blocks                               │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  4. Send new request with:                           │
│     - Original messages                              │
│     - + Assistant response (with tool_use)           │
│     - + User message (with tool_result blocks)       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  5. Check stop_reason:                               │
│     - "tool_use" → go back to step 2                 │
│     - anything else → exit loop                      │
└─────────────────────────────────────────────────────┘
```

### Step-by-Step Walkthrough

#### Step 1: Send the Initial Request

You send a request with the `tools[]` array defining available tools and a `messages` array containing the user's question:

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[...],  # Tool definitions
    messages=[
        {"role": "user", "content": "What's the weather in San Francisco?"}
    ]
)
```

#### Step 2: Claude Responds with tool_use

Claude analyzes the user's request, reads the tool descriptions, and decides to call a tool. The response has `stop_reason: "tool_use"`:

```json
{
  "stop_reason": "tool_use",
  "content": [
    {"type": "text", "text": "Let me check the weather for you."},
    {"type": "tool_use", "id": "toolu_01ABC", "name": "get_weather",
     "input": {"location": "San Francisco, CA"}}
  ]
}
```

#### Step 3: Execute the Tool

Your application extracts the tool call, executes it against your backend, and formats the result:

```python
# Extract tool calls
for block in response.content:
    if block.type == "tool_use":
        # Execute the tool (your implementation)
        result = call_weather_api(block.input["location"])
        # result = "65°F, partly cloudy, 72% humidity"
```

#### Step 4: Send the Result Back

You construct a new request that includes the full conversation history — the original messages, Claude's response (with the tool_use block), and a new user message containing the tool_result:

```python
messages = [
    # Original user message
    {"role": "user", "content": "What's the weather in San Francisco?"},

    # Claude's full response (MUST include the tool_use block)
    {"role": "assistant", "content": response.content},

    # Tool result in a user message
    {"role": "user", "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC",
            "content": "65°F, partly cloudy, 72% humidity"
        }
    ]}
]

# Send the follow-up request
follow_up = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[...],  # Same tool definitions
    messages=messages
)
```

#### Step 5: Check stop_reason and Loop or Exit

If `stop_reason` is `"tool_use"` again, Claude wants to call another tool — repeat from Step 2. If `stop_reason` is anything else (`"end_turn"`, `"max_tokens"`, etc.), the loop exits and you have Claude's final response.

```python
if follow_up.stop_reason == "tool_use":
    # Claude wants another tool call — continue the loop
    pass
else:
    # Claude is done — extract the final response
    final_text = follow_up.content[0].text
    print(final_text)
    # "The current weather in San Francisco is 65°F with partly cloudy
    #  skies and 72% humidity."
```

### Full Python Implementation of the Agentic Loop

This is the complete, production-ready pattern for the agentic loop. The exam expects you to understand every part of this code:

```python
import anthropic

client = anthropic.Anthropic()

# Define available tools
tools = [
    {
        "name": "get_weather",
        "description": "Get the current weather conditions for a specific location. Use this when the user asks about weather, temperature, or atmospheric conditions. Returns temperature (Fahrenheit), conditions, and humidity. Only supports US cities.",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "The city and state, e.g. 'San Francisco, CA'"
                }
            },
            "required": ["location"]
        }
    }
]

# Your tool execution function
def execute_tool(name, input_data):
    if name == "get_weather":
        # In production, this calls your actual weather API
        location = input_data["location"]
        return f"65°F, partly cloudy, 72% humidity in {location}"
    else:
        return f"Error: Unknown tool '{name}'"

# The agentic loop
def run_agent(user_message, max_iterations=10):
    messages = [{"role": "user", "content": user_message}]

    for i in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        # Exit condition: stop_reason is NOT "tool_use"
        if response.stop_reason != "tool_use":
            # Extract and return the final text response
            return response.content[0].text

        # Claude wants to use tools — process each tool call
        # Append Claude's full response to the conversation
        messages.append({"role": "assistant", "content": response.content})

        # Build tool results
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })

        # Append tool results as a user message
        messages.append({"role": "user", "content": tool_results})

    # Safety: max iterations reached
    return "Error: Agent loop exceeded maximum iterations."

# Run it
answer = run_agent("What's the weather like in San Francisco?")
print(answer)
```

### Critical Details in the Agentic Loop

1. **Always include Claude's full response** — When appending the assistant message, include the entire `response.content` (text blocks AND tool_use blocks). Omitting the tool_use blocks breaks the conversation structure.

2. **The `max_iterations` guard** — Without this, a misbehaving tool or ambiguous request could cause an infinite loop. The exam tests this as an anti-pattern: loops without exit conditions.

3. **Same `tools[]` on every request** — You must include the tool definitions on every request in the loop, not just the first one. Claude needs to know what tools are available at each step.

4. **The loop exits on any non-tool_use stop_reason** — `end_turn`, `max_tokens`, `stop_sequence`, `refusal`, `pause_turn` — all of these exit the loop. Only `"tool_use"` continues it.

---

## 5. How Claude Decides Which Tool to Call

Claude's tool selection is not random or rule-based — it is driven by the tool descriptions you provide. Understanding this selection mechanism is essential for designing tools that Claude uses correctly.

### The Selection Process

When Claude receives a request with tools defined, it:

1. **Reads all tool descriptions** in the `tools[]` array
2. **Analyzes the user's request** to understand what information or action is needed
3. **Matches the request to the most relevant tool** based on the descriptions
4. **Generates the input parameters** based on the tool's `input_schema` and the user's request
5. **May include explanatory text** before the tool_use block, explaining its reasoning

### Tool Descriptions Are the Primary Selection Mechanism

Claude does not select tools based on the tool name alone. The `description` field is what Claude reads to understand:
- What the tool does
- When it should be used
- What it returns
- How it differs from other available tools

This is why the description is the most important field in a tool definition (see Section 1).

### Claude May Include Text Before tool_use

When Claude decides to call a tool, it often includes a natural language explanation before the tool_use block:

```json
{
  "content": [
    {
      "type": "text",
      "text": "I'll look up the current weather conditions in San Francisco for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_01ABC",
      "name": "get_weather",
      "input": {"location": "San Francisco, CA"}
    }
  ]
}
```

The `content` array can contain **both text blocks and tool_use blocks**. Your code must handle this — don't assume `content[0]` is always a text block or always a tool_use block. Iterate through the array and check each block's `type`.

### Claude May Choose NOT to Use a Tool

If Claude determines that it can answer the user's question without a tool (e.g., the question is about general knowledge), it will respond with a text-only response and `stop_reason: "end_turn"` — even if tools are available.

```python
# User asks a general question — Claude doesn't need a tool
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[weather_tool],
    messages=[{"role": "user", "content": "What is the capital of France?"}]
)
# response.stop_reason == "end_turn" (no tool call)
# response.content[0].text == "The capital of France is Paris."
```

This is expected behavior. Claude uses tools when they are relevant, not on every request.

### Claude May Call Multiple Tools

If the user's request requires information from multiple sources, Claude may include multiple tool_use blocks in a single response:

```json
{
  "content": [
    {"type": "text", "text": "Let me check both cities."},
    {"type": "tool_use", "id": "toolu_01", "name": "get_weather",
     "input": {"location": "San Francisco, CA"}},
    {"type": "tool_use", "id": "toolu_02", "name": "get_weather",
     "input": {"location": "New York, NY"}}
  ],
  "stop_reason": "tool_use"
}
```

Your agentic loop must handle all tool calls in the response, not just the first one.

---


## 6. Structuring Tool Descriptions for Clarity and Disambiguation

When your application exposes multiple tools, Claude must choose the right one for each request. Ambiguous or overlapping descriptions cause misrouting — Claude calls the wrong tool, or calls no tool when it should. This section covers the strategies for writing descriptions that eliminate ambiguity.

### The Disambiguation Problem

Consider two tools with vague descriptions:

```json
[
  {"name": "search", "description": "Searches for information."},
  {"name": "lookup", "description": "Looks up data."}
]
```

If the user asks "Find information about order #12345," which tool should Claude call? The descriptions are too similar to differentiate. Claude may pick either one, or alternate unpredictably.

### Strategy 1: Be Specific About What Each Tool Does

Rewrite the descriptions to clearly differentiate scope:

```json
[
  {
    "name": "search_knowledge_base",
    "description": "Search the internal knowledge base for articles, FAQs, and documentation about company policies and product features. Use this for general questions about how things work. Input is a natural language query. Returns matching articles with titles and snippets."
  },
  {
    "name": "lookup_order",
    "description": "Look up a specific customer order by order ID. Use this when the user provides an order number (format: ORD-XXXXX) and wants to know the order status, items, or shipping details. Returns order details including status, items, and tracking information."
  }
]
```

Now Claude can clearly distinguish: general questions go to `search_knowledge_base`, order-specific queries go to `lookup_order`.

### Strategy 2: Include When NOT to Use the Tool

Negative instructions are powerful disambiguators:

```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal knowledge base for articles and documentation. Use this for general questions about policies, features, and processes. Do NOT use this for looking up specific orders, customers, or transactions — use the appropriate lookup tool instead."
}
```

### Strategy 3: Consolidate Related Operations

Instead of creating many similar tools, consolidate related operations into a single tool with an `action` parameter:

**Before (too many similar tools):**
```json
[
  {"name": "create_ticket", "description": "Creates a support ticket."},
  {"name": "update_ticket", "description": "Updates a support ticket."},
  {"name": "close_ticket", "description": "Closes a support ticket."},
  {"name": "get_ticket", "description": "Gets a support ticket."}
]
```

**After (consolidated with action parameter):**
```json
{
  "name": "manage_ticket",
  "description": "Manage support tickets. Supports creating, updating, closing, and retrieving tickets. Use the 'action' parameter to specify the operation. For 'create': provide subject and description. For 'update': provide ticket_id and fields to change. For 'close': provide ticket_id and resolution. For 'get': provide ticket_id.",
  "input_schema": {
    "type": "object",
    "properties": {
      "action": {
        "type": "string",
        "enum": ["create", "update", "close", "get"],
        "description": "The operation to perform on the ticket."
      },
      "ticket_id": {
        "type": "string",
        "description": "The ticket ID (required for update, close, get)."
      },
      "subject": {
        "type": "string",
        "description": "Ticket subject (required for create)."
      },
      "description": {
        "type": "string",
        "description": "Ticket description or update content."
      },
      "resolution": {
        "type": "string",
        "description": "Resolution summary (required for close)."
      }
    },
    "required": ["action"]
  }
}
```

**Trade-off**: Consolidation reduces the number of tools Claude must choose between, but makes each tool's schema more complex. Use consolidation when the operations are closely related and share context.

### Strategy 4: Use Meaningful Namespacing

When tools come from different systems, prefix the tool name with the system name:

```json
[
  {"name": "github_list_prs", "description": "List open pull requests from a GitHub repository..."},
  {"name": "github_create_issue", "description": "Create a new issue in a GitHub repository..."},
  {"name": "slack_send_message", "description": "Send a message to a Slack channel..."},
  {"name": "slack_list_channels", "description": "List available Slack channels..."},
  {"name": "jira_create_ticket", "description": "Create a new Jira ticket..."}
]
```

Namespacing makes it immediately clear which system each tool belongs to, both for Claude and for human developers reading the code.

### Strategy 5: Document Parameter Meanings in the Description

Don't rely solely on the `input_schema` to explain parameters. Reinforce important parameter details in the description:

```json
{
  "name": "process_refund",
  "description": "Process a refund for a customer order. Use this after verifying the order details with lookup_order. The order_id must be a valid order (format: ORD-XXXXX). The amount is in USD and must not exceed the original order total. The reason should be a brief explanation (e.g., 'damaged item', 'wrong product'). Returns a confirmation with refund_id and estimated processing time.",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": {"type": "string", "description": "The order ID to refund"},
      "amount": {"type": "number", "description": "Refund amount in USD"},
      "reason": {"type": "string", "description": "Reason for the refund"}
    },
    "required": ["order_id", "amount", "reason"]
  }
}
```

---

## 7. Where Tools Run — Client vs Server

The Claude API supports two categories of tools based on where they execute. Understanding this distinction is important for the exam.

### Client Tools (User-Defined)

Client tools are tools **you define** in the `tools[]` array and **you execute** on your side. This is the pattern covered in Sections 1–6 of this document.

The flow for client tools:
1. You define the tool in `tools[]`
2. Claude returns a `tool_use` block
3. **You** execute the tool (call your API, query your database, etc.)
4. **You** return the result as a `tool_result` block in a user message
5. Claude processes the result and continues

Client tools are the foundation of the agentic loop. You have full control over what happens when the tool is called — you can add logging, validation, rate limiting, authentication, or any other logic.

### Server Tools (Anthropic-Hosted)

Server tools are tools that **Anthropic executes** on their infrastructure. You don't need to handle the execution or return results — Anthropic's servers do it automatically.

Current server tools include:

| Tool | Purpose |
|------|---------|
| `web_search` | Search the web for current information |
| `code_execution` | Execute code in a sandboxed environment |
| `web_fetch` | Fetch and read web page content |

Server tools are enabled differently from client tools — they use a specific configuration format rather than the `tools[]` array pattern.

### What the Exam Focuses On

The CCA-F exam primarily tests **client tools** and the agentic loop pattern. You need to know:

- How to define tools in the `tools[]` array
- How to process `tool_use` blocks
- How to construct `tool_result` blocks
- How to implement the agentic loop
- How to write effective tool descriptions

Server tools may appear in questions about built-in capabilities, but the core exam content is about the client tool pattern.

### Key Difference Summary

| Aspect | Client Tools | Server Tools |
|--------|-------------|--------------|
| **Who defines them** | You (in `tools[]`) | Anthropic |
| **Who executes them** | You (your code) | Anthropic's servers |
| **Who returns results** | You (via `tool_result`) | Automatic |
| **Agentic loop** | You implement the full loop | Handled server-side |
| **Customization** | Full control | Limited configuration |
| **Exam focus** | Primary | Secondary |

---

## 8. Key Anti-Patterns for Day 3

These are the mistakes the exam tests you on. Each anti-pattern represents a common error in tool use implementation.

### Anti-Pattern 1: Vague Tool Descriptions That Cause Misrouting

**Wrong:**
```json
{
  "name": "search",
  "description": "Searches for things."
}
```

**Right:**
```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal company knowledge base for articles, FAQs, and documentation. Use this when the user asks about company policies, product features, or internal processes. The query should be a natural language question or keyword phrase. Returns up to 5 matching articles ranked by relevance. Does NOT search external websites — use web_search for that."
}
```

**Why it matters**: Vague descriptions cause Claude to misroute tool calls — calling the wrong tool, or not calling any tool when it should. The description is the primary selection mechanism.

### Anti-Pattern 2: Not Matching tool_use_id in tool_result

**Wrong:**
```python
# Using a hardcoded or wrong ID
tool_result = {
    "type": "tool_result",
    "tool_use_id": "some_random_id",  # WRONG — doesn't match the tool_use block
    "content": "result data"
}
```

**Right:**
```python
# Use the exact ID from the tool_use block
for block in response.content:
    if block.type == "tool_use":
        tool_result = {
            "type": "tool_result",
            "tool_use_id": block.id,  # CORRECT — matches the tool_use block
            "content": execute_tool(block.name, block.input)
        }
```

**Why it matters**: The `tool_use_id` is how Claude matches results to requests. A mismatch causes an API error or, worse, Claude associating the wrong result with the wrong tool call.

### Anti-Pattern 3: Adding Text Blocks After tool_result

**Wrong:**
```python
# Adding a text block after tool_result in the same user message
{
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC",
            "content": "65°F, partly cloudy"
        },
        {
            "type": "text",
            "text": "Here are the results!"  # DON'T DO THIS
        }
    ]
}
```

**Right:**
```python
# Only tool_result blocks in the tool result message
{
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC",
            "content": "65°F, partly cloudy"
        }
    ]
}
```

**Why it matters**: Adding text blocks alongside `tool_result` blocks in the same user message can cause Claude to return empty responses. Claude interprets the text as a new user input that interrupts the tool flow, leading to confusion. If you need to add context, put it in a separate user message after Claude has processed the tool results.

### Anti-Pattern 4: Not Handling is_error in Tool Results

**Wrong:**
```python
# Tool execution failed, but you return it as a success
try:
    result = call_api(tool_input)
except Exception as e:
    result = str(e)  # Error message returned as if it were a successful result

tool_result = {
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": result  # Claude thinks this is valid data
}
```

**Right:**
```python
# Properly flag errors with is_error
try:
    result = call_api(tool_input)
    tool_result = {
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": result
    }
except Exception as e:
    tool_result = {
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": f"Error: {str(e)}",
        "is_error": True  # Claude knows the tool failed
    }
```

**Why it matters**: Without `is_error: true`, Claude treats error messages as successful results. It may try to parse "Error: Connection timeout" as weather data, leading to nonsensical responses. With the flag set, Claude understands the failure and can retry, try a different approach, or inform the user.

### Anti-Pattern 5: Loops Without Exit Conditions

**Wrong:**
```python
# No maximum iteration guard — infinite loop risk
def run_agent(user_message):
    messages = [{"role": "user", "content": user_message}]

    while True:  # DANGEROUS — no exit condition
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        if response.stop_reason != "tool_use":
            return response.content[0].text

        # Process tool calls...
        # What if the tool always fails and Claude keeps retrying?
```

**Right:**
```python
# With a maximum iteration guard
def run_agent(user_message, max_iterations=10):
    messages = [{"role": "user", "content": user_message}]

    for i in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        if response.stop_reason != "tool_use":
            return response.content[0].text

        # Process tool calls...

    # Safety exit
    return "Error: Maximum iterations reached. The agent could not complete the task."
```

**Why it matters**: A misbehaving tool, ambiguous request, or circular dependency can cause Claude to keep requesting tool calls indefinitely. The `max_iterations` guard is a circuit breaker that prevents runaway loops. This pattern is tested across Domain 1 (Agentic Architecture) and Domain 2 (Tool Design).

---


## 9. Quick Reference Card

### Tool Definition Structure

```json
{
  "name": "tool_name",
  "description": "3-4+ sentences: what it does, when to use it, parameter details, limitations.",
  "input_schema": {
    "type": "object",
    "properties": {
      "param": {"type": "string", "description": "What this param means"}
    },
    "required": ["param"]
  }
}
```

### tool_use Block (Claude → You)

```json
{
  "type": "tool_use",
  "id": "toolu_01ABC...",
  "name": "tool_name",
  "input": {"param": "value"}
}
```

### tool_result Block (You → Claude)

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC...",
  "content": "result string or content array",
  "is_error": false
}
```

### Agentic Loop Pattern

```python
for i in range(max_iterations):
    response = client.messages.create(
        model=MODEL, max_tokens=MAX, tools=TOOLS, messages=messages
    )
    if response.stop_reason != "tool_use":
        return response.content[0].text  # Done

    messages.append({"role": "assistant", "content": response.content})
    tool_results = []
    for block in response.content:
        if block.type == "tool_use":
            result = execute_tool(block.name, block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result
            })
    messages.append({"role": "user", "content": tool_results})
```

### Tool Name Rules

- Regex: `^[a-zA-Z0-9_-]{1,64}$`
- Letters, numbers, underscores, hyphens only
- 1–64 characters
- Must be unique within the `tools[]` array

### Message Flow Summary

| Step | Role | Content |
|------|------|---------|
| 1. User asks | `user` | Text message |
| 2. Claude calls tool | `assistant` | Text + `tool_use` block(s) |
| 3. You return result | `user` | `tool_result` block(s) |
| 4. Claude responds | `assistant` | Text (final answer) or more `tool_use` |

### Tool Description Checklist

| Include | Example |
|---------|---------|
| What it does | "Retrieve customer profile information" |
| When to use it | "Use when the user asks about a customer's account" |
| Parameter details | "Email must be a valid email address" |
| What it returns | "Returns customer_id, name, email, account_status" |
| Limitations | "Only supports US customers" |
| When NOT to use | "Do not use for order lookups — use lookup_order instead" |

### Key Fields to Remember

| Field | Where | Purpose |
|-------|-------|---------|
| `tools[]` | Request body | Define available tools |
| `tool_use` | Response `content[]` | Claude's tool call request |
| `tool_result` | User message `content[]` | Your tool execution result |
| `tool_use_id` | `tool_result` block | Links result to the correct tool call |
| `is_error` | `tool_result` block | Flags failed tool execution |
| `stop_reason: "tool_use"` | Response | Signals Claude is waiting for tool results |

### Anti-Pattern Quick Reference

| Anti-Pattern | Fix |
|-------------|-----|
| Vague tool descriptions | Write 3-4+ sentence descriptions with scope, triggers, and limitations |
| Mismatched `tool_use_id` | Always use `block.id` from the `tool_use` block |
| Text blocks after `tool_result` | Only include `tool_result` blocks in the tool result message |
| Not using `is_error` | Set `is_error: true` when tool execution fails |
| No `max_iterations` guard | Always use a `for` loop with a maximum iteration count |

---

## 10. Scenario Challenge

### Scenario Challenge: Domain 2 — Tool Design & MCP Integration

**Scenario**: You are building a customer support agent that has access to three tools: `get_customer(email)`, `lookup_order(customer_id, order_id)`, and `process_refund(order_id, amount, reason)`. During testing, you notice that Claude sometimes calls `process_refund` before calling `lookup_order`, skipping the order verification step. The tool descriptions are:

- `get_customer`: "Gets customer info."
- `lookup_order`: "Looks up an order."
- `process_refund`: "Processes a refund."

Additionally, your agentic loop uses `while True` with no iteration limit, and when `process_refund` fails (e.g., order not found), you return the error message as a normal `tool_result` without setting `is_error`.

**Question**: Which combination of changes would MOST improve the reliability of this agent?

A) Add `temperature: 0` to all API requests and switch to a more capable model

B) Rewrite tool descriptions to be specific (e.g., "Use process_refund only AFTER verifying the order with lookup_order"), add `is_error: true` to failed tool results, and add a `max_iterations` guard to the loop

C) Add a system prompt that says "Always call lookup_order before process_refund" and increase `max_tokens`

D) Remove the `process_refund` tool and handle refunds outside the agent

---

**Correct Answer**: **B**

**Explanation**: This question tests three Day 3 concepts simultaneously:

1. **Tool description quality** — The current descriptions are vague (Anti-Pattern 1). Rewriting them with specific guidance about when to use each tool — including the sequencing requirement ("use process_refund only AFTER verifying the order with lookup_order") — directly addresses the misrouting problem. Claude selects tools based on descriptions, so making the descriptions explicit about the required sequence is the correct fix.

2. **Error handling with `is_error`** — Returning error messages without `is_error: true` (Anti-Pattern 4) causes Claude to treat error text as valid data. Setting the flag lets Claude understand the failure and adjust its approach.

3. **Loop safety with `max_iterations`** — A `while True` loop (Anti-Pattern 5) risks infinite execution if the tool keeps failing or Claude keeps retrying. Adding a `max_iterations` guard is a standard circuit breaker.

- **Why A is wrong**: `temperature` controls randomness, not tool selection order. A more capable model doesn't fix vague descriptions — Claude still reads the same descriptions regardless of model.
- **Why C is wrong**: System prompt instructions about tool ordering are vague instructions (anti-pattern from Day 1). Claude may or may not follow them. The correct approach is to encode the sequencing guidance in the tool descriptions themselves, where Claude reads them at tool selection time. Also, `max_tokens` is unrelated to tool sequencing.
- **Why D is wrong**: Removing the tool eliminates the agent's ability to handle refunds autonomously, which defeats the purpose of the agent. The correct fix is to improve the tool descriptions and error handling, not remove capabilities.

**Domain Reference**: Domain 2: Tool Design & MCP Integration (18% weighting)
**Exam Principles Tested**: Tool description quality drives tool selection; use `is_error` for failed tools; always guard loops with `max_iterations`

---

## Sources

All technical content in this document is based on the official Anthropic documentation. Content was rephrased for compliance with licensing restrictions.

- [Tool use overview](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview) — Tool definition structure, tool_use and tool_result blocks, agentic loop pattern
- [Tool use best practices](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/best-practices-and-limitations) — Tool description guidelines, disambiguation strategies, anti-patterns
- [Messages API Reference](https://docs.anthropic.com/en/api/messages) — Request/response structure, tools parameter, stop_reason values
- [Client tools vs server tools](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview#client-tools-vs-server-tools) — Execution model differences between client and server tools
