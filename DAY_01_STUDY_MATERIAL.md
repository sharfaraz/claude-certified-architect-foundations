# Day 1 — Claude API Fundamentals & Message Structure

**Domain**: Domain 4: Prompt Engineering & Structured Output (20% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Begin Claude 101
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [What is the Claude API?](#1-what-is-the-claude-api)
2. [The Messages API — Core Endpoint](#2-the-messages-api--core-endpoint)
3. [Message Roles: user, assistant, system](#3-message-roles-user-assistant-system)
4. [System Prompt Design](#4-system-prompt-design)
5. [API Request Parameters](#5-api-request-parameters)
6. [Response Structure](#6-response-structure)
7. [stop_reason — Why Claude Stopped Responding](#7-stop_reason--why-claude-stopped-responding)
8. [Multi-Turn Conversations](#8-multi-turn-conversations)
9. [Key Anti-Patterns for Day 1](#9-key-anti-patterns-for-day-1)
10. [Exam-Relevant Principles](#10-exam-relevant-principles)
11. [Quick Reference Card](#11-quick-reference-card)
12. [Scenario Challenge](#12-scenario-challenge)

---

## 1. What is the Claude API?

The Claude API is a RESTful API hosted at `https://api.anthropic.com` that provides programmatic access to Claude models. Unlike the chat interface at claude.ai, the API gives you full control over every parameter of the interaction — the system prompt, message history, token limits, temperature, and tool definitions.

The API is **stateless**. Every request must include the full conversation history. There is no server-side session. This is a fundamental architectural fact that shapes how you build applications with Claude.

**Why this matters for the exam**: The exam tests whether you understand the request/response lifecycle and can design systems that use API parameters programmatically rather than relying on vague prompt instructions. This is the foundation of the "programmatic enforcement over vague instruction" principle that recurs across all five exam domains.

### Key Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/messages` | POST | Send messages and get a response |
| `/v1/messages/count_tokens` | POST | Count tokens in a message without generating a response |
| `/v1/messages/batches` | POST | Create a batch of message requests (covered on Day 8) |

---

## 2. The Messages API — Core Endpoint

The Messages API (`POST /v1/messages`) is the primary way to interact with Claude. You send a structured list of input messages, and Claude generates the next message in the conversation.

### Basic Request and Response

**Python SDK example:**

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude"}
    ]
)
print(message)
```

**Response:**

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Hello!"
    }
  ],
  "model": "claude-sonnet-4-20250514",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 12,
    "output_tokens": 6
  }
}
```

**Key observations from this response:**

- `id` — Unique identifier for this message
- `type` — Always `"message"` for a successful response
- `role` — Always `"assistant"` in the response
- `content` — An **array** of content blocks (not a plain string). Each block has a `type` field. For text responses, the type is `"text"`.
- `stop_reason` — Why Claude stopped generating (critical for control flow)
- `usage` — Token counts for billing and context window management

---

## 3. Message Roles: user, assistant, system

The Messages API uses three roles to structure conversations. Understanding these roles is essential for designing effective API interactions.

### The `user` Role

Represents the human (or your application acting on behalf of a human). User messages contain the input, questions, or instructions you want Claude to respond to.

```python
{"role": "user", "content": "What is the capital of France?"}
```

User messages can contain multiple content blocks — text, images, tool results, and documents:

```python
{
    "role": "user",
    "content": [
        {"type": "text", "text": "What is in this image?"},
        {
            "type": "image",
            "source": {
                "type": "base64",
                "media_type": "image/jpeg",
                "data": "<base64_encoded_image>"
            }
        }
    ]
}
```

### The `assistant` Role

Represents Claude's responses. In the `messages` array you send to the API, assistant messages serve two purposes:

1. **Conversation history**: Previous Claude responses included to maintain context
2. **Synthetic messages**: You can insert assistant messages that Claude didn't actually generate, to steer the conversation or provide context

```python
messages = [
    {"role": "user", "content": "Hello, Claude"},
    {"role": "assistant", "content": "Hello!"},           # Previous response
    {"role": "user", "content": "Can you describe LLMs?"} # New question
]
```

**Important**: Messages must alternate between `user` and `assistant` roles. The first message must always be a `user` message.

### The `system` Role

The system prompt is **not** part of the `messages` array. It is a separate top-level parameter in the API request:

```python
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a helpful coding assistant specializing in Python.",  # <-- separate parameter
    messages=[
        {"role": "user", "content": "How do I sort a list of dictionaries by key?"}
    ]
)
```

This is a common point of confusion. The system prompt is **not** a message with `role: "system"`. It is the `system` parameter on the request body.

---

## 4. System Prompt Design

The system prompt sets Claude's behavior, personality, constraints, and context for the entire conversation. It is processed before any user messages and shapes how Claude interprets and responds to everything that follows.

### Placement and Purpose

- The system prompt is set via the `system` parameter (not inside `messages`)
- It persists across all turns in the conversation
- It is the primary mechanism for defining Claude's role, output format, and behavioral constraints

### System Prompt Best Practices

These best practices are drawn from Anthropic's official prompting guidance and are directly relevant to the exam.

#### 1. Be Clear and Direct

Claude responds well to explicit instructions. Being specific about your desired output enhances results. Think of Claude as a brilliant but new employee who lacks context on your norms and workflows.

**Golden rule**: Show your prompt to a colleague with minimal context on the task. If they'd be confused, Claude will be too.

**Weak system prompt:**
```
You are a helpful assistant.
```

**Strong system prompt:**
```
You are a senior Python developer reviewing pull requests. For each code snippet the user provides:
1. Identify bugs or potential issues
2. Suggest improvements with code examples
3. Rate the code quality on a scale of 1-5
Format your response with clear headings for each section.
```

#### 2. Give Claude a Role

Setting a role focuses Claude's behavior and tone. Even a single sentence makes a difference:

```python
system="You are a helpful coding assistant specializing in Python."
```

Roles work because they activate relevant knowledge and set expectations for the style and depth of responses.

#### 3. Use XML Tags for Structure

XML tags help Claude parse complex prompts unambiguously, especially when mixing instructions, context, examples, and variable inputs:

```xml
<instructions>
Analyze the customer feedback and extract:
1. Overall sentiment (positive, negative, neutral)
2. Key topics mentioned
3. Action items for the product team
</instructions>

<output_format>
Return your analysis as a structured summary with clear headings.
</output_format>
```

Best practices for XML tags:
- Use consistent, descriptive tag names
- Nest tags when content has a natural hierarchy
- Wrap examples in `<example>` tags so Claude can distinguish them from instructions

#### 4. Provide Context and Motivation

Explaining *why* you want something helps Claude deliver more targeted responses:

```
Format all dates as YYYY-MM-DD. We use this format because our downstream 
data pipeline expects ISO 8601 dates and will reject other formats.
```

Claude is smart enough to generalize from the explanation.

#### 5. Use Examples (Few-Shot Prompting)

Examples are one of the most reliable ways to steer Claude's output. A few well-crafted examples dramatically improve accuracy and consistency. (Few-shot prompting is covered in depth on Day 7, but the principle starts here.)

```xml
<examples>
<example>
<input>The product arrived broken and customer service was unhelpful.</input>
<output>Sentiment: Negative | Topics: Product Quality, Customer Service | Priority: High</output>
</example>
</examples>
```

#### 6. Put Long-Form Data at the Top

When working with large documents (20k+ tokens), place the documents near the top of your prompt, above your query and instructions. Queries at the end can improve response quality by up to 30% in tests.

### What NOT to Do in System Prompts

| Anti-Pattern | Why It Fails |
|-------------|-------------|
| "Please always return JSON" | Vague instruction — no enforcement mechanism |
| Conflicting instructions | Claude may follow one and ignore the other unpredictably |
| Overly long, unfocused prompts | Dilutes the important instructions |
| Relying on system prompt alone for structured output | Use `tool_choice` + JSON Schema instead (Days 4–5) |

---

## 5. API Request Parameters

These are the key parameters you send in a `POST /v1/messages` request. The exam tests your understanding of what each parameter does and when to use it.

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | The model to use (e.g., `"claude-sonnet-4-20250514"`, `"claude-opus-4-7"`) |
| `max_tokens` | integer | Maximum number of tokens Claude can generate in its response |
| `messages` | array | The conversation history — an array of `user` and `assistant` message objects |

### Optional Parameters (Day 1 Focus)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `system` | string or array | none | System prompt — sets Claude's role and behavior |
| `temperature` | float | model-dependent | Controls randomness. Range: 0.0 to 1.0. Lower = more deterministic, higher = more creative |
| `stop_sequences` | array of strings | none | Custom strings that cause Claude to stop generating when encountered |
| `metadata` | object | none | Metadata about the request (e.g., `user_id` for abuse tracking) |

### Parameters Covered Later in Sprint 1

| Parameter | Day | Description |
|-----------|-----|-------------|
| `tools` | Day 3 | Tool definitions for tool_use |
| `tool_choice` | Day 4 | Controls how Claude selects tools |

### The `model` Parameter

Available model families (as of the exam preparation period):

| Model | Context Window | Best For |
|-------|---------------|----------|
| `claude-opus-4-7` | 200K tokens | Most capable; long-horizon agentic work, complex reasoning |
| `claude-sonnet-4-20250514` | 200K tokens | Balanced performance and cost |
| `claude-haiku-4-5-20251001` | 200K tokens | Fast, cost-efficient for simpler tasks |

### The `max_tokens` Parameter

This is a **hard ceiling** on Claude's output length. If Claude's response reaches this limit, it stops generating and returns `stop_reason: "max_tokens"`.

Key points:
- This does NOT control the quality or depth of the response — it only sets the maximum length
- Setting it too low truncates responses mid-sentence
- Setting it too high wastes nothing (you only pay for tokens actually generated)
- The exam tests whether you understand the relationship between `max_tokens` and `stop_reason`

### The `temperature` Parameter

Controls the randomness of Claude's output:

| Value | Behavior | Use Case |
|-------|----------|----------|
| 0.0 | Most deterministic (near-identical outputs for same input) | Data extraction, classification, factual Q&A |
| 0.5 | Balanced | General conversation |
| 1.0 | Most creative/random | Creative writing, brainstorming |

**Exam note**: Temperature affects randomness, NOT tool selection. Setting `temperature: 0` does not guarantee Claude will use a specific tool. That requires `tool_choice` (Day 4).

---

## 6. Response Structure

Every successful Messages API response has the same structure. Understanding each field is critical for building robust applications.

```json
{
  "id": "msg_01234",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Here's the answer to your question..."
    }
  ],
  "model": "claude-sonnet-4-20250514",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 100,
    "output_tokens": 50
  }
}
```

### Field-by-Field Breakdown

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique message identifier (format: `msg_...`) |
| `type` | string | Always `"message"` for successful responses |
| `role` | string | Always `"assistant"` |
| `content` | array | Array of content blocks. Can contain `text`, `tool_use`, `thinking`, and other block types |
| `model` | string | The model that generated the response |
| `stop_reason` | string | Why Claude stopped generating (see Section 7) |
| `stop_sequence` | string or null | The specific stop sequence that was matched, if `stop_reason` is `"stop_sequence"` |
| `usage` | object | Token usage: `input_tokens` and `output_tokens` |

### The `content` Array

The `content` field is an **array of content blocks**, not a simple string. This is important because Claude can return multiple types of content in a single response:

- **Text blocks**: `{"type": "text", "text": "..."}`
- **Tool use blocks**: `{"type": "tool_use", "id": "...", "name": "...", "input": {...}}` (Day 3)
- **Thinking blocks**: `{"type": "thinking", "thinking": "..."}` (when extended thinking is enabled)

For Day 1, you'll primarily work with text blocks. To extract the text from a response:

```python
# Access the text content
response_text = message.content[0].text
```

### The `usage` Object

Tracks token consumption for billing and context window management:

```json
"usage": {
    "input_tokens": 100,
    "output_tokens": 50
}
```

- `input_tokens` — Tokens in your request (system prompt + messages)
- `output_tokens` — Tokens Claude generated in its response

This becomes critical for Domain 5 (Context Management & Reliability) when you need to monitor token usage to avoid exceeding context windows.

---

## 7. stop_reason — Why Claude Stopped Responding

The `stop_reason` field is one of the most important fields in the API response. It tells you **why** Claude stopped generating, and your application's control flow should branch based on this value.

**Exam principle**: Ignoring `stop_reason` and assuming the response is always complete is a tested anti-pattern.

### stop_reason Values

The exam focuses on four core values. Newer models also support additional values listed below.

#### `"end_turn"` — Natural Completion

Claude finished its response naturally. This is the most common stop reason.

```python
if response.stop_reason == "end_turn":
    # Response is complete — process it normally
    print(response.content[0].text)
```

**What it means**: Claude decided it had fully answered the question or completed the task. The response is complete.

**Edge case — Empty responses with `end_turn`**: Sometimes Claude returns an empty response with `stop_reason: "end_turn"`. This typically happens when:
- You add text blocks immediately after tool results (teaches Claude to expect user input after every tool use)
- You send Claude's completed response back without adding anything new

**Fix**: Don't add text blocks after `tool_result` blocks. If you get an empty response, add a continuation prompt in a NEW user message rather than retrying blindly.

#### `"max_tokens"` — Truncated by Token Limit

Claude's response was cut off because it reached the `max_tokens` limit you specified.

```python
if response.stop_reason == "max_tokens":
    # Response was truncated — it is INCOMPLETE
    print("Warning: Response was cut off at token limit")
    # Option 1: Increase max_tokens and retry
    # Option 2: Ask Claude to continue from where it left off
```

**What it means**: The response is **incomplete**. Claude had more to say but ran out of budget. You must handle this — either by increasing `max_tokens`, or by continuing the conversation.

**Continuation pattern:**

```python
def get_complete_response(client, prompt, max_attempts=3):
    messages = [{"role": "user", "content": prompt}]
    full_response = ""

    for _ in range(max_attempts):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            messages=messages,
            max_tokens=4096
        )

        full_response += response.content[0].text

        if response.stop_reason != "max_tokens":
            break  # Response is complete

        # Continue from where Claude left off
        messages = [
            {"role": "user", "content": prompt},
            {"role": "assistant", "content": full_response},
            {"role": "user", "content": "Please continue from where you left off."}
        ]

    return full_response
```

#### `"stop_sequence"` — Hit a Custom Stop Sequence

Claude encountered one of the custom stop sequences you defined in the request.

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    stop_sequences=["END", "STOP"],
    messages=[{"role": "user", "content": "Generate text until you say END"}]
)

if response.stop_reason == "stop_sequence":
    print(f"Stopped at sequence: {response.stop_sequence}")
    # response.stop_sequence tells you WHICH sequence was matched
```

**What it means**: You defined boundaries, and Claude hit one. The `stop_sequence` field in the response tells you which specific sequence was matched.

#### `"tool_use"` — Claude Wants to Use a Tool

Claude is requesting to call a tool and expects you to execute it and return the result. This is covered in depth on Day 3, but understanding the stop_reason value is Day 1 material.

```python
if response.stop_reason == "tool_use":
    # Claude wants to call a tool
    # Extract the tool_use block from response.content
    # Execute the tool
    # Return the result as a tool_result message
    for block in response.content:
        if block.type == "tool_use":
            tool_name = block.name
            tool_input = block.input
            # Execute and return result...
```

**What it means**: Claude has decided it needs external information or capabilities. Your application must execute the tool and feed the result back. This is the foundation of the agentic loop pattern (Domain 1).

#### Additional stop_reason Values (Newer Models)

These are supported on newer models and may appear on the exam:

| Value | Meaning |
|-------|---------|
| `"pause_turn"` | Server-side sampling loop reached its iteration limit while executing server tools (e.g., web search). Continue the conversation by sending the response back. |
| `"refusal"` | Claude refused to generate a response due to safety concerns. |
| `"model_context_window_exceeded"` | Claude stopped because it reached the model's context window limit. The response is valid but was limited by context window size, not `max_tokens`. |

### Control Flow Based on stop_reason

This is the pattern the exam expects you to know:

```python
def handle_response(response):
    if response.stop_reason == "tool_use":
        return handle_tool_use(response)
    elif response.stop_reason == "max_tokens":
        return handle_truncation(response)
    elif response.stop_reason == "stop_sequence":
        return handle_stop_sequence(response)
    elif response.stop_reason == "refusal":
        return handle_refusal(response)
    elif response.stop_reason == "pause_turn":
        return handle_pause(response)
    elif response.stop_reason == "model_context_window_exceeded":
        return handle_context_limit(response)
    else:
        # end_turn — natural completion
        return response.content[0].text
```

### stop_reason vs. Errors

It is important to distinguish between `stop_reason` values and actual API errors:

| | stop_reason | Errors |
|---|-----------|--------|
| **Where** | Part of the response body | HTTP status codes 4xx or 5xx |
| **Meaning** | Why generation stopped *normally* | Request processing *failed* |
| **Content** | Response contains valid content | Response contains error details |

```python
import anthropic

try:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello!"}]
    )
    # Handle successful response — check stop_reason
    if response.stop_reason == "max_tokens":
        print("Response was truncated")

except anthropic.APIError as e:
    # Handle actual errors
    if e.status_code == 429:
        print("Rate limit exceeded")
    elif e.status_code == 500:
        print("Server error")
```

---

## 8. Multi-Turn Conversations

Because the Messages API is **stateless**, you must send the full conversation history with every request. There is no server-side session tracking.

### Building a Conversation

```python
messages = [
    {"role": "user", "content": "Hello, Claude"},
    {"role": "assistant", "content": "Hello!"},
    {"role": "user", "content": "Can you describe LLMs to me?"}
]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=messages
)
```

### Rules for Message Ordering

1. The first message must have `role: "user"`
2. Messages must alternate between `user` and `assistant`
3. The last message must have `role: "user"` (this is what Claude responds to)

### Synthetic Assistant Messages

You can insert assistant messages that Claude didn't actually generate. This is useful for:
- Setting up a conversation context
- Steering Claude's behavior by showing it "previous" responses in a desired style
- Providing conversation history from a different source

```python
messages = [
    {"role": "user", "content": "What's the weather like?"},
    {"role": "assistant", "content": "I don't have access to real-time weather data, but I can help you find weather information. What city are you interested in?"},
    {"role": "user", "content": "San Francisco"}
]
```

### Statelessness and Its Implications

| Implication | What It Means |
|------------|---------------|
| No server-side memory | Every request must include the full conversation |
| Token costs grow | Longer conversations use more input tokens per request |
| Context window limits | Eventually, the conversation history exceeds the model's context window |
| You control the history | You can edit, summarize, or truncate the history between requests |

This statelessness is a feature, not a limitation. It gives you full control over what context Claude sees, which is essential for context management strategies (Domain 5).

---

## 9. Key Anti-Patterns for Day 1

These are the mistakes the exam tests you on. Memorize them.

### Anti-Pattern 1: Ignoring stop_reason

**Wrong:**
```python
response = client.messages.create(...)
# Just grab the text and assume it's complete
result = response.content[0].text  # DANGEROUS — might be truncated!
```

**Right:**
```python
response = client.messages.create(...)
if response.stop_reason == "max_tokens":
    # Handle truncation
    pass
elif response.stop_reason == "tool_use":
    # Handle tool call
    pass
else:
    result = response.content[0].text
```

### Anti-Pattern 2: Relying on Vague Instructions Instead of Programmatic Enforcement

**Wrong:**
```python
system = "Please always return your answer as JSON."
# No guarantee Claude will comply
```

**Right (covered Days 4–5, but the principle starts here):**
```python
# Use tool_choice to FORCE structured output
# Use JSON Schema to DEFINE the output structure
# Use Pydantic to VALIDATE the output
```

### Anti-Pattern 3: Using temperature to Control Tool Selection

**Wrong:**
```python
# "Setting temperature to 0 will make Claude always use the tool"
response = client.messages.create(
    temperature=0,
    ...
)
```

**Right:**
```python
# Use tool_choice to control tool selection (Day 4)
response = client.messages.create(
    tool_choice={"type": "tool", "name": "extract_data"},
    ...
)
```

### Anti-Pattern 4: Not Sending Full Conversation History

**Wrong:**
```python
# Only sending the latest message, losing context
messages = [{"role": "user", "content": "What about the second point?"}]
```

**Right:**
```python
# Include the full conversation history
messages = [
    {"role": "user", "content": "List 3 benefits of exercise."},
    {"role": "assistant", "content": "1. Improved cardiovascular health..."},
    {"role": "user", "content": "What about the second point?"}
]
```

---

## 10. Exam-Relevant Principles

### Principle 1: Programmatic Enforcement Over Vague Instruction

This is THE recurring exam theme across all domains. Day 1 introduces it; Days 2–9 build on it.

- Don't tell Claude what to do with words alone — use API parameters to enforce behavior
- `tool_choice` beats "please use the tool" (Day 4)
- JSON Schema beats "please return JSON" (Day 5)
- `stop_sequences` beats "please stop when you see X"
- `max_tokens` is a hard limit, not a suggestion

### Principle 2: stop_reason Drives Control Flow

Your application should branch based on `stop_reason`, not based on parsing the text content. This is the foundation of reliable Claude API applications.

### Principle 3: The API is Stateless

You own the conversation history. This gives you power (you can edit, summarize, truncate) but also responsibility (you must manage context window limits).

### Principle 4: System Prompts Set Behavior, Not Guarantees

System prompts influence Claude's behavior but do not guarantee specific output formats or tool usage. For guarantees, use programmatic enforcement (tool_choice, JSON Schema, stop_sequences).

---

## 11. Quick Reference Card

### Request Structure

```python
client.messages.create(
    model="claude-sonnet-4-20250514",   # Required: which model
    max_tokens=1024,                     # Required: output token ceiling
    system="You are a...",               # Optional: system prompt
    temperature=0.5,                     # Optional: randomness (0.0–1.0)
    stop_sequences=["END"],              # Optional: custom stop strings
    messages=[                           # Required: conversation history
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."},
        {"role": "user", "content": "..."}
    ]
)
```

### Response Fields

| Field | What to Check |
|-------|--------------|
| `content[0].text` | The actual response text |
| `stop_reason` | Why Claude stopped — branch your logic here |
| `stop_sequence` | Which stop sequence matched (if applicable) |
| `usage.input_tokens` | How many tokens your request consumed |
| `usage.output_tokens` | How many tokens Claude generated |

### stop_reason Decision Table

| stop_reason | Is Response Complete? | What to Do |
|-------------|----------------------|------------|
| `end_turn` | Yes | Process normally |
| `max_tokens` | No — truncated | Increase max_tokens or continue the conversation |
| `stop_sequence` | Depends on use case | Check which sequence matched |
| `tool_use` | No — waiting for tool result | Execute the tool and return the result |
| `pause_turn` | No — server tool loop paused | Send response back to continue |
| `refusal` | N/A — refused | Handle gracefully, consider rephrasing |

---

## 12. Scenario Challenge

### Scenario Challenge: Domain 4 — Prompt Engineering & Structured Output

**Scenario**: You are building a customer feedback analysis API. Your application sends customer reviews to Claude and expects a structured analysis back. During testing, you notice that approximately 15% of responses are truncated mid-sentence, cutting off the analysis before the "Action Items" section is complete. Your current code simply reads `response.content[0].text` and passes it to the downstream service.

**Question**: What is the most robust way to fix this issue?

A) Increase `temperature` to 0.0 so Claude generates shorter, more focused responses that fit within the token limit

B) Add "Please keep your response under 500 words" to the system prompt

C) Check `response.stop_reason` for `"max_tokens"` and either increase `max_tokens` or implement a continuation loop to collect the complete response

D) Switch to a faster model like Claude Haiku, which generates shorter responses by default

---

**Correct Answer**: **C**

**Explanation**: The truncation is caused by Claude hitting the `max_tokens` ceiling, which results in `stop_reason: "max_tokens"`. The correct fix is to check `stop_reason` in your response handling logic and either increase the `max_tokens` parameter or implement a continuation pattern that asks Claude to continue from where it left off.

- **Why A is wrong**: `temperature` controls randomness, not response length. Setting it to 0.0 makes responses more deterministic but does not reliably shorten them.
- **Why B is wrong**: This is a vague instruction (anti-pattern). Claude may or may not comply, and even if it does, you lose the detailed analysis you need. Programmatic enforcement via `max_tokens` is the correct approach.
- **Why D is wrong**: Switching models doesn't address the root cause. Haiku may still exceed the token limit on complex reviews, and you'd lose the analytical depth of a more capable model.

**Domain Reference**: Domain 4: Prompt Engineering & Structured Output (20% weighting)
**Exam Principle Tested**: Programmatic enforcement over vague instruction; always check `stop_reason`

---

## Sources

All technical content in this document is based on the official Anthropic documentation. Content was rephrased for compliance with licensing restrictions.

- [Messages API Reference](https://docs.anthropic.com/en/api/messages) — API endpoint specifications and model types
- [Using the Messages API](https://docs.anthropic.com/en/api/prompt-validation) — Request/response patterns and multi-turn conversations
- [Handling stop reasons](https://docs.claude.com/en/api/handling-stop-reasons) — stop_reason values, best practices, and code examples
- [Prompting best practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts) — System prompts, XML tags, examples, and role assignment
