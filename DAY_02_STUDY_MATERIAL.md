# Day 2 — System Prompts & stop_reason Handling

**Domain**: Domain 4: Prompt Engineering & Structured Output (20% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Continue Claude 101
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [System Prompt Best Practices — Deep Dive](#1-system-prompt-best-practices--deep-dive)
2. [The system Parameter — Not a Message Role](#2-the-system-parameter--not-a-message-role)
3. [System Prompt vs User Message — What Goes Where](#3-system-prompt-vs-user-message--what-goes-where)
4. [XML Tags for Structured System Prompts](#4-xml-tags-for-structured-system-prompts)
5. [System Prompt Length Considerations](#5-system-prompt-length-considerations)
6. [System Prompts and Tool Definitions](#6-system-prompts-and-tool-definitions)
7. [stop_reason Deep Dive — All Values with Code Examples](#7-stop_reason-deep-dive--all-values-with-code-examples)
8. [stop_reason Control Flow — The Complete Pattern](#8-stop_reason-control-flow--the-complete-pattern)
9. [stop_reason Edge Cases and Pitfalls](#9-stop_reason-edge-cases-and-pitfalls)
10. [Streaming Considerations for stop_reason](#10-streaming-considerations-for-stop_reason)
11. [Key Anti-Patterns for Day 2](#11-key-anti-patterns-for-day-2)
12. [Exam-Relevant Principles](#12-exam-relevant-principles)
13. [Quick Reference Card](#13-quick-reference-card)
14. [Scenario Challenge](#14-scenario-challenge)

---

## 1. System Prompt Best Practices — Deep Dive

Day 1 introduced the system prompt as the `system` parameter that sets Claude's behavior. Day 2 goes deeper into the patterns that make system prompts effective — and the mistakes that make them unreliable.

**Anthropic's official guidance**: "Use the system parameter to set Claude's role. Put everything else, like task-specific instructions, in the user turn instead."

This single sentence is the foundation of system prompt design. The system prompt defines *who Claude is* for this conversation. The user message defines *what Claude should do right now*.

### Specificity and Role Assignment Patterns

The most effective system prompts are specific about Claude's role, expertise, and behavioral constraints. Vague system prompts produce vague behavior.

**Pattern 1: Role + Expertise + Constraints**

```python
import anthropic

client = anthropic.Anthropic()

# Strong: specific role, clear expertise, explicit constraints
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=(
        "You are a senior database administrator specializing in PostgreSQL. "
        "You have 15 years of experience optimizing queries for high-traffic applications. "
        "Always consider index usage, query plans, and connection pooling in your recommendations. "
        "Never suggest dropping tables or deleting data without explicit confirmation steps."
    ),
    messages=[
        {"role": "user", "content": "This query is taking 30 seconds. How can I speed it up?\n\nSELECT * FROM orders JOIN customers ON orders.customer_id = customers.id WHERE orders.created_at > '2024-01-01';"}
    ]
)
```

**Pattern 2: Role + Output Style**

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=(
        "You are a technical writer for a developer documentation team. "
        "Write in a clear, concise style. Use short paragraphs. "
        "Prefer active voice. Include code examples for every concept. "
        "Target audience: intermediate developers who know Python but are new to async programming."
    ),
    messages=[
        {"role": "user", "content": "Explain Python's asyncio event loop."}
    ]
)
```

**Pattern 3: Role + Behavioral Boundaries**

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=(
        "You are a customer support agent for Acme Corp. "
        "You can answer questions about products, process returns, and check order status. "
        "You cannot modify pricing, issue credits over $50, or access billing systems. "
        "If a request exceeds your capabilities, tell the customer you will escalate to a specialist."
    ),
    messages=[
        {"role": "user", "content": "I want a full refund of $200 for my damaged order."}
    ]
)
```

### Output Constraints via System Prompts

System prompts can influence output format, but they cannot *guarantee* it. This distinction is critical for the exam.

**Influencing output format (useful but not guaranteed):**

```python
system = (
    "You are a data analyst. When presenting findings, always structure your response as:\n"
    "1. Summary (2-3 sentences)\n"
    "2. Key Metrics (bullet points)\n"
    "3. Recommendations (numbered list)\n"
    "Keep responses under 500 words."
)
```

**Why this is influence, not enforcement**: Claude will *usually* follow this format, but there is no programmatic mechanism preventing it from deviating. For guaranteed structured output, use `tool_choice` + JSON Schema (Days 4–5).

**Exam principle**: System prompts set behavioral tendencies. Programmatic parameters (`tool_choice`, `stop_sequences`, `max_tokens`) enforce hard constraints. Know the difference.

---

## 2. The system Parameter — Not a Message Role

This is one of the most commonly tested misconceptions on the exam. The system prompt is **not** a message with `role: "system"`. It is a separate top-level parameter called `system` on the API request.

### Wrong — Treating system as a message role

```python
# WRONG — This will cause an API error
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},  # ERROR!
        {"role": "user", "content": "Hello!"}
    ]
)
# ValidationError: messages.0.role must be "user" or "assistant"
```

### Right — Using the system parameter

```python
# CORRECT — system is a separate parameter
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a helpful assistant.",  # Separate parameter
    messages=[
        {"role": "user", "content": "Hello!"}
    ]
)
```

### Why This Matters Architecturally

The `system` parameter is processed differently from messages:

| Aspect | `system` Parameter | Messages (`user`/`assistant`) |
|--------|-------------------|-------------------------------|
| **Position in context** | Processed first, before all messages | Processed in order |
| **Persistence** | Applies to the entire conversation | Each message is a single turn |
| **Role** | Defines Claude's identity and behavior | Provides conversation content |
| **Alternation rules** | Not subject to user/assistant alternation | Must alternate user ↔ assistant |
| **Format** | String or array of content blocks | Array of message objects |

### The system Parameter as an Array

The `system` parameter can also accept an array of content blocks, which is useful for combining text with cache control:

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a senior Python developer reviewing code for security vulnerabilities.",
        },
        {
            "type": "text",
            "text": "Focus on: SQL injection, XSS, CSRF, insecure deserialization, and hardcoded secrets.",
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[
        {"role": "user", "content": "Review this code:\n\nquery = f\"SELECT * FROM users WHERE id = {user_id}\""}
    ]
)
```

---

## 3. System Prompt vs User Message — What Goes Where

This is a design decision that the exam tests directly. The rule is simple but frequently violated:

- **System prompt**: Claude's role, identity, persistent behavioral rules, and global constraints
- **User message**: Task-specific instructions, data to process, questions to answer

### Examples of Correct Placement

| Content | Where It Goes | Why |
|---------|--------------|-----|
| "You are a legal document reviewer" | System prompt | Defines Claude's role |
| "Review this contract for liability clauses" | User message | Task-specific instruction |
| "Always respond in formal English" | System prompt | Persistent behavioral rule |
| "Summarize the following in 3 bullet points" | User message | Task-specific format request |
| "Never reveal internal pricing data" | System prompt | Global security constraint |
| "Here is the customer's email: ..." | User message | Data to process |
| "You have access to a database lookup tool" | System prompt | Persistent capability context |
| "Look up order #12345" | User message | Specific task |

### Anti-Pattern: Stuffing Everything into the System Prompt

```python
# WRONG — Task-specific instructions don't belong in the system prompt
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=(
        "You are a data analyst. "
        "The user will give you a CSV file. "  # Task-specific — should be in user message
        "Parse it and extract all email addresses. "  # Task-specific
        "Return them as a JSON array. "  # Task-specific
        "Sort them alphabetically. "  # Task-specific
    ),
    messages=[
        {"role": "user", "content": "name,email\nAlice,alice@example.com\nBob,bob@test.com"}
    ]
)
```

```python
# RIGHT — Role in system prompt, task in user message
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a data analyst specializing in data extraction and transformation.",
    messages=[
        {
            "role": "user",
            "content": (
                "Parse the following CSV and extract all email addresses. "
                "Return them as a JSON array, sorted alphabetically.\n\n"
                "name,email\nAlice,alice@example.com\nBob,bob@test.com"
            )
        }
    ]
)
```

### Why the Separation Matters

1. **Reusability**: The same system prompt can serve many different tasks. If you embed task instructions in the system prompt, you need a new system prompt for every task.
2. **Caching**: System prompts can be cached across requests. Task-specific content changes every request and defeats caching.
3. **Clarity**: Claude processes the system prompt as its identity. Mixing in task instructions dilutes the role definition.
4. **Multi-turn conversations**: The system prompt persists across turns. Task instructions should change per turn.

---

## 4. XML Tags for Structured System Prompts

When system prompts grow beyond a few sentences, XML tags provide structure that Claude can parse unambiguously. This is especially important when mixing different types of instructions.

### Basic XML Structure in System Prompts

```python
system_prompt = """
<role>
You are a senior security engineer conducting code reviews.
You specialize in identifying OWASP Top 10 vulnerabilities.
</role>

<rules>
- Always check for SQL injection, XSS, and CSRF vulnerabilities
- Rate severity as: Critical, High, Medium, Low, Informational
- Provide a fix for every vulnerability you identify
- If no vulnerabilities are found, explicitly state the code is clean
</rules>

<output_format>
Structure your review as:
1. Summary — one paragraph overview
2. Findings — each vulnerability with severity, description, and fix
3. Overall Risk Rating — aggregate assessment
</output_format>
"""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    system=system_prompt,
    messages=[
        {"role": "user", "content": "Review this Flask endpoint:\n\n@app.route('/search')\ndef search():\n    query = request.args.get('q')\n    return f'<h1>Results for {query}</h1>'"}
    ]
)
```

### Nesting XML Tags for Complex Prompts

```python
system_prompt = """
<role>
You are a multilingual customer support agent for a SaaS platform.
</role>

<capabilities>
  <can_do>
  - Answer product questions
  - Process subscription changes
  - Troubleshoot common technical issues
  - Provide billing information
  </can_do>
  <cannot_do>
  - Issue refunds over $100
  - Access raw database records
  - Modify user permissions
  - Share internal documentation
  </cannot_do>
</capabilities>

<escalation_rules>
If the customer's issue falls outside your capabilities:
1. Acknowledge the limitation
2. Summarize the issue clearly
3. Tell the customer you are escalating to a specialist
4. Do NOT attempt to resolve issues outside your scope
</escalation_rules>

<tone>
Professional but warm. Use the customer's name when available.
Avoid jargon. Explain technical concepts in plain language.
</tone>
"""
```

### Best Practices for XML Tags in System Prompts

| Practice | Example | Why |
|----------|---------|-----|
| Use descriptive tag names | `<escalation_rules>` not `<er>` | Clarity for both Claude and human readers |
| Nest when content has hierarchy | `<capabilities><can_do>...</can_do></capabilities>` | Reflects logical structure |
| Keep tags consistent | Always `<role>` not sometimes `<persona>` | Reduces ambiguity |
| Separate concerns | Different tags for role, rules, format, tone | Claude can reference each section independently |
| Wrap examples in `<example>` tags | `<example>Input: ... Output: ...</example>` | Distinguishes examples from instructions |

---

## 5. System Prompt Length Considerations

System prompts consume input tokens. Every token in the system prompt is sent with every request in the conversation. This has cost and performance implications.

### When System Prompts Get Too Long

| System Prompt Length | Typical Use Case | Concern |
|---------------------|-----------------|---------|
| < 200 tokens | Simple role + basic rules | No concern |
| 200–1,000 tokens | Detailed role + rules + format | Normal for production |
| 1,000–5,000 tokens | Complex rules + examples + context | Monitor token costs |
| 5,000+ tokens | Extensive documentation or reference material | Consider alternatives |

### Signs Your System Prompt Is Too Long

1. **It contains reference data**: Move large reference documents to the user message or use retrieval
2. **It contains task-specific examples**: Move examples to the user message where they are relevant
3. **It repeats instructions in different ways**: Consolidate — Claude understands the first time
4. **It includes content that changes per request**: That content belongs in the user message

### Strategies for Long System Prompts

```python
# Strategy 1: Use cache_control for stable portions
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "<role>You are a medical coding specialist...</role>",  # Stable — cache this
            "cache_control": {"type": "ephemeral"}
        },
        {
            "type": "text",
            "text": "<today_context>Current date: 2025-01-15. Active coding version: ICD-10-CM 2025.</today_context>"  # Changes — don't cache
        }
    ],
    messages=[{"role": "user", "content": "Code this diagnosis: acute bronchitis"}]
)
```

```python
# Strategy 2: Move reference material to user message
# Instead of putting a 10-page style guide in the system prompt:
system = "You are a technical editor. Follow the style guide provided in the user message."

messages = [
    {
        "role": "user",
        "content": (
            "<style_guide>\n"
            "... (large style guide content) ...\n"
            "</style_guide>\n\n"
            "Edit the following paragraph according to the style guide:\n\n"
            "The data was processed by the system and the results was sent to the user."
        )
    }
]
```

---

## 6. System Prompts and Tool Definitions

When you define tools in the API request, the system prompt and tool definitions work together but serve different purposes. Understanding their interaction is important for the exam.

### How They Interact

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=(
        "You are a customer support agent. "
        "Always look up the customer's account before answering questions about their orders. "
        "If a refund is needed, verify the order details first."
    ),
    tools=[
        {
            "name": "get_customer",
            "description": "Look up a customer by email address. Returns customer ID, name, and account status.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "email": {"type": "string", "description": "Customer's email address"}
                },
                "required": ["email"]
            }
        },
        {
            "name": "lookup_order",
            "description": "Look up an order by order ID. Returns order details including items, total, and status.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "The order ID to look up"}
                },
                "required": ["order_id"]
            }
        }
    ],
    messages=[
        {"role": "user", "content": "I need help with order #12345. My email is alice@example.com."}
    ]
)
```

### Division of Responsibility

| Aspect | System Prompt | Tool Definitions |
|--------|--------------|-----------------|
| **What it controls** | When and why to use tools | What tools are available and their schemas |
| **Behavioral guidance** | "Always look up the customer first" | Tool descriptions guide selection |
| **Schema enforcement** | Cannot enforce input/output schemas | JSON Schema enforces structure |
| **Tool selection** | Influences tool selection via instructions | `tool_choice` parameter enforces selection |

### Key Insight for the Exam

The system prompt can *guide* tool usage behavior ("always look up the customer first"), but it cannot *guarantee* it. For guaranteed tool usage, use `tool_choice`. For guaranteed tool call ordering, implement it in your application logic by checking `stop_reason: "tool_use"` and controlling which tools are available at each step.

```python
# System prompt guides behavior (influence)
system = "Always verify the customer's identity before processing any actions."

# tool_choice enforces behavior (guarantee)
# First call: force customer lookup
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=system,
    tools=tools,
    tool_choice={"type": "tool", "name": "get_customer"},  # FORCED
    messages=[{"role": "user", "content": "Process refund for order #12345"}]
)
```

---


## 7. stop_reason Deep Dive — All Values with Code Examples

Day 1 introduced `stop_reason` as the field that tells you why Claude stopped generating. Day 2 goes deep into every possible value, with detailed code examples and the edge cases the exam tests.

**Key exam fact**: The four core `stop_reason` values are `end_turn`, `max_tokens`, `stop_sequence`, and `tool_use`. Newer models also support `pause_turn`, `refusal`, and `model_context_window_exceeded`.

### 7.1 `"end_turn"` — Natural Completion

Claude finished its response naturally. It decided it had fully addressed the request and stopped on its own.

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "What is 2 + 2?"}
    ]
)

print(response.stop_reason)  # "end_turn"
print(response.content[0].text)  # "4" (or similar)
# The response is COMPLETE — process it normally
```

**What it means**: The response is complete. Claude said everything it wanted to say. This is the most common stop reason for simple question-answer interactions.

**When you see it**: Most text-only responses, completed analyses, finished explanations.

**Action**: Process the response normally. No special handling needed.

### 7.2 `"max_tokens"` — Truncated by Token Limit

Claude's response was cut off because it hit the `max_tokens` ceiling you set in the request. The response is **incomplete**.

```python
# Deliberately set a low max_tokens to demonstrate truncation
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=10,  # Very low — will truncate
    messages=[
        {"role": "user", "content": "Write a detailed essay about the history of computing."}
    ]
)

print(response.stop_reason)  # "max_tokens"
print(response.content[0].text)  # Truncated mid-sentence
# WARNING: This response is INCOMPLETE
```

**What it means**: Claude had more to say but ran out of token budget. The response may end mid-sentence, mid-paragraph, or mid-code-block. You **must** handle this.

**When you see it**: Long responses that exceed your `max_tokens` setting, complex analyses, code generation tasks.

**Action — Option 1: Increase max_tokens and retry**

```python
# If the response was truncated, retry with a higher limit
if response.stop_reason == "max_tokens":
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,  # Increased limit
        messages=[
            {"role": "user", "content": "Write a detailed essay about the history of computing."}
        ]
    )
```

**Action — Option 2: Continuation loop**

```python
def get_complete_response(client, system, messages, max_tokens=4096, max_continuations=5):
    """Collect a complete response, continuing if truncated."""
    full_text = ""

    for attempt in range(max_continuations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=max_tokens,
            system=system,
            messages=messages
        )

        # Collect text from this response
        for block in response.content:
            if block.type == "text":
                full_text += block.text

        # If response is complete, we're done
        if response.stop_reason != "max_tokens":
            break

        # Response was truncated — ask Claude to continue
        messages = messages + [
            {"role": "assistant", "content": full_text},
            {"role": "user", "content": "Please continue from where you left off."}
        ]

    return full_text, response.stop_reason
```

**Critical edge case — Incomplete tool_use blocks**: If Claude is generating a `tool_use` block and hits `max_tokens`, the tool call JSON may be incomplete and unparseable. The fix is to retry with a higher `max_tokens` value.

```python
if response.stop_reason == "max_tokens":
    # Check if there's an incomplete tool_use block
    has_tool_use = any(block.type == "tool_use" for block in response.content)
    if not has_tool_use:
        # Claude was probably trying to generate a tool call but got cut off
        # Retry with higher max_tokens
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=8192,  # Double the limit
            messages=messages
        )
```

### 7.3 `"stop_sequence"` — Hit a Custom Stop Sequence

Claude encountered one of the custom stop sequences you defined in the `stop_sequences` parameter.

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    stop_sequences=["```", "END_OUTPUT", "---"],
    messages=[
        {"role": "user", "content": "Write a Python function to sort a list, then explain it."}
    ]
)

if response.stop_reason == "stop_sequence":
    print(f"Stopped at: '{response.stop_sequence}'")  # Which sequence matched
    print(response.content[0].text)  # Content up to (but not including) the stop sequence
```

**What it means**: You defined boundaries, and Claude hit one. The `stop_sequence` field in the response tells you exactly which sequence was matched.

**When you see it**: When you use `stop_sequences` to control output boundaries — for example, stopping after the first code block, stopping at a delimiter, or implementing turn-based protocols.

**Action**: Check `response.stop_sequence` to determine which boundary was hit, then handle accordingly.

**Practical use case — Extracting just the code:**

```python
# Stop Claude after it writes the first code block
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    stop_sequences=["```"],  # Stop at the closing code fence
    messages=[
        {"role": "user", "content": "Write a Python function to calculate fibonacci numbers. Start with ```python"}
    ]
)

if response.stop_reason == "stop_sequence":
    code = response.content[0].text  # Just the code, no explanation after
```

### 7.4 `"tool_use"` — Claude Wants to Use a Tool

Claude has decided it needs to call a tool and is waiting for you to execute it and return the result. This is the foundation of the agentic loop pattern.

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get the current weather for a city.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"}
                },
                "required": ["city"]
            }
        }
    ],
    messages=[
        {"role": "user", "content": "What's the weather in Tokyo?"}
    ]
)

print(response.stop_reason)  # "tool_use"

# The content array may contain BOTH text and tool_use blocks
for block in response.content:
    if block.type == "text":
        print(f"Claude said: {block.text}")
    elif block.type == "tool_use":
        print(f"Tool call: {block.name}({block.input})")
        print(f"Tool use ID: {block.id}")
        # Execute the tool and return the result...
```

**What it means**: Claude is not done. It needs external information or capabilities. Your application must:
1. Extract the `tool_use` block(s) from `response.content`
2. Execute the tool(s)
3. Return the result(s) as `tool_result` message(s)
4. Send the updated conversation back to Claude

**When you see it**: Any time Claude decides to call a tool — weather lookups, database queries, calculations, API calls, etc.

**Action — The tool execution loop:**

```python
def run_tool_loop(client, system, tools, messages, max_iterations=10):
    """Execute the tool use loop until Claude is done."""
    for _ in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system=system,
            tools=tools,
            messages=messages
        )

        # If Claude is done (not requesting a tool), return
        if response.stop_reason != "tool_use":
            return response

        # Extract tool calls and execute them
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)  # Your tool execution logic
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result)
                })

        # Add Claude's response and tool results to the conversation
        messages = messages + [
            {"role": "assistant", "content": response.content},
            {"role": "user", "content": tool_results}
        ]

    raise RuntimeError("Tool loop exceeded max iterations — possible infinite loop")
```

**Detecting tool_use vs end_turn**: This is a key exam concept. When Claude has tools available, you must check `stop_reason` to determine whether Claude is requesting a tool call or has finished responding:

```python
if response.stop_reason == "tool_use":
    # Claude wants to call a tool — DO NOT treat this as a final response
    handle_tool_call(response)
elif response.stop_reason == "end_turn":
    # Claude is done — this is the final response
    process_final_response(response)
```

### 7.5 `"pause_turn"` — Server Tool Loop Paused

This stop reason appears when Claude is using server-side tools (like web search) and the server-side sampling loop reaches its iteration limit.

```python
if response.stop_reason == "pause_turn":
    # Claude was using server-side tools and the loop was paused
    # Continue by sending the response back as an assistant message
    messages = messages + [
        {"role": "assistant", "content": response.content},
        {"role": "user", "content": "Please continue."}
    ]
    # Make another API call to let Claude continue
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=messages
    )
```

**What it means**: Claude was in the middle of a multi-step server-side operation and hit an iteration limit. The response contains partial progress. You need to send it back to let Claude continue.

**When you see it**: When using features like web search or other server-side tools that involve multiple internal steps.

**Action**: Send the response back as an assistant message with a continuation prompt.

### 7.6 `"refusal"` — Safety Refusal

Claude refused to generate a response due to safety or policy concerns.

```python
if response.stop_reason == "refusal":
    # Claude refused to respond
    # The content may contain an explanation of why
    print("Claude declined to respond.")
    for block in response.content:
        if block.type == "text":
            print(f"Reason: {block.text}")
    # Handle gracefully — consider rephrasing the request
```

**What it means**: The request triggered Claude's safety filters. This is different from an API error — the request was processed successfully, but Claude chose not to generate the requested content.

**When you see it**: Requests that involve harmful content, policy violations, or safety-sensitive topics.

**Action**: Handle gracefully. Log the refusal. Consider whether the request can be rephrased appropriately.

### 7.7 `"model_context_window_exceeded"` — Context Window Limit

Claude stopped because the combined input and output exceeded the model's context window.

```python
if response.stop_reason == "model_context_window_exceeded":
    # The response is valid but was limited by context window size
    # This is different from max_tokens — it's the model's hard limit
    print("Response limited by context window.")
    # Consider: summarize conversation history, reduce input size,
    # or use a model with a larger context window
```

**What it means**: The total tokens (input + output) hit the model's context window ceiling (e.g., 200K tokens). The response is valid but may be incomplete. This is different from `max_tokens` — it's the model's architectural limit, not your configured limit.

**When you see it**: Very long conversations, large document processing, or when input is already close to the context window limit.

**Action**: Reduce input size (summarize history, trim documents) or use a model with a larger context window.

---

## 8. stop_reason Control Flow — The Complete Pattern

This is the pattern the exam expects you to implement. Every production application using the Claude API should have a response handler that branches on `stop_reason`.

### The Complete Handler

```python
import anthropic

client = anthropic.Anthropic()


def handle_claude_response(response):
    """
    Complete stop_reason handler — the pattern the exam tests.
    Every branch handles a different reason Claude stopped generating.
    """
    match response.stop_reason:
        case "end_turn":
            # Natural completion — response is complete
            return extract_text(response)

        case "max_tokens":
            # Truncated — response is INCOMPLETE
            print("Warning: Response truncated at max_tokens limit")
            return handle_truncation(response)

        case "stop_sequence":
            # Hit a custom stop sequence
            print(f"Stopped at sequence: {response.stop_sequence}")
            return extract_text(response)

        case "tool_use":
            # Claude wants to call a tool
            return handle_tool_use(response)

        case "pause_turn":
            # Server-side tool loop paused — continue the conversation
            return handle_pause(response)

        case "refusal":
            # Safety refusal
            print("Claude refused to respond")
            return handle_refusal(response)

        case "model_context_window_exceeded":
            # Context window limit reached
            print("Context window exceeded")
            return handle_context_limit(response)

        case _:
            # Unknown stop_reason — log and handle gracefully
            print(f"Unknown stop_reason: {response.stop_reason}")
            return extract_text(response)


def extract_text(response):
    """Extract all text blocks from a response."""
    texts = []
    for block in response.content:
        if block.type == "text":
            texts.append(block.text)
    return "\n".join(texts)


def handle_truncation(response):
    """Handle max_tokens truncation — return partial text with warning."""
    text = extract_text(response)
    return f"[TRUNCATED] {text}"


def handle_tool_use(response):
    """Extract tool calls from the response."""
    tool_calls = []
    for block in response.content:
        if block.type == "tool_use":
            tool_calls.append({
                "id": block.id,
                "name": block.name,
                "input": block.input
            })
    return {"tool_calls": tool_calls, "needs_continuation": True}


def handle_refusal(response):
    """Handle safety refusal gracefully."""
    text = extract_text(response)
    return f"[REFUSED] {text}" if text else "[REFUSED] No explanation provided."


def handle_pause(response):
    """Handle pause_turn — signal that conversation should continue."""
    return {"partial_response": extract_text(response), "needs_continuation": True}


def handle_context_limit(response):
    """Handle context window exceeded."""
    text = extract_text(response)
    return f"[CONTEXT_LIMIT] {text}"
```

### Why This Pattern Matters

The exam tests whether you understand that `stop_reason` is the **primary control flow mechanism** for Claude API applications. Without checking `stop_reason`:

- You might process a truncated response as if it were complete
- You might ignore a tool call and never execute it
- You might miss a safety refusal and pass empty content downstream
- You might not realize the conversation needs to continue

---

## 9. stop_reason Edge Cases and Pitfalls

These are the subtle issues that separate exam-ready knowledge from surface-level understanding.

### Edge Case 1: Empty Responses with `end_turn`

Sometimes Claude returns an empty `content` array (or a text block with empty string) with `stop_reason: "end_turn"`. This is confusing because Claude appears to have "finished" but said nothing.

**Common cause — Adding text blocks after tool_result:**

```python
# WRONG — This pattern causes empty responses
messages = [
    {"role": "user", "content": "What's the weather in Tokyo?"},
    {"role": "assistant", "content": [
        {"type": "text", "text": "Let me check the weather for you."},
        {"type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "Tokyo"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "72°F, sunny"},
        {"type": "text", "text": "Here's the tool result."}  # DON'T DO THIS
    ]}
]
# Claude may return an empty response with end_turn
```

**Fix — Don't add text blocks after tool_result:**

```python
# RIGHT — Only tool_result blocks in the user message after tool execution
messages = [
    {"role": "user", "content": "What's the weather in Tokyo?"},
    {"role": "assistant", "content": [
        {"type": "text", "text": "Let me check the weather for you."},
        {"type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "Tokyo"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "72°F, sunny"}
        # No extra text blocks here
    ]}
]
```

### Edge Case 2: Incomplete tool_use Blocks from max_tokens

When Claude is generating a `tool_use` block and hits `max_tokens`, the JSON in the tool call may be incomplete. The SDK may raise a parsing error, or you may get a malformed tool call.

```python
# This can happen when max_tokens is too low for the tool call
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=50,  # Too low for a complex tool call
    tools=tools,
    messages=[{"role": "user", "content": "Search for all orders from last month and summarize them."}]
)

if response.stop_reason == "max_tokens":
    # Check if we got a complete tool_use block
    has_complete_tool_call = any(
        block.type == "tool_use" for block in response.content
    )
    if not has_complete_tool_call:
        # Retry with higher max_tokens
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=[{"role": "user", "content": "Search for all orders from last month and summarize them."}]
        )
```

### Edge Case 3: stop_reason vs Errors — They Are Fundamentally Different

`stop_reason` is part of a **successful** API response. Errors are **failed** requests that never produce a response.

```python
import anthropic

try:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello!"}]
    )
    # SUCCESS — we have a response. Now check stop_reason.
    # stop_reason tells us WHY Claude stopped, not WHETHER the request succeeded.
    if response.stop_reason == "max_tokens":
        print("Response was truncated but the request succeeded")
    elif response.stop_reason == "end_turn":
        print("Response is complete")

except anthropic.BadRequestError as e:
    # 400 — Invalid request (bad parameters, malformed messages)
    print(f"Bad request: {e}")

except anthropic.AuthenticationError as e:
    # 401 — Invalid API key
    print(f"Auth error: {e}")

except anthropic.RateLimitError as e:
    # 429 — Too many requests
    print(f"Rate limited: {e}")

except anthropic.InternalServerError as e:
    # 500 — Anthropic server error
    print(f"Server error: {e}")

except anthropic.APIError as e:
    # Catch-all for other API errors
    print(f"API error ({e.status_code}): {e}")
```

**Key distinction for the exam:**

| Aspect | stop_reason | API Errors |
|--------|-----------|------------|
| HTTP status | 200 OK | 4xx or 5xx |
| Response body | Contains `content`, `stop_reason`, `usage` | Contains `error` object with `type` and `message` |
| Meaning | Claude generated a response (may be partial) | Request failed — no response generated |
| Retry strategy | Depends on the stop_reason value | Depends on the error code (429 → backoff, 500 → retry) |
| Content available | Yes — always has valid content blocks | No — no content generated |

---

## 10. Streaming Considerations for stop_reason

When using the streaming API, `stop_reason` is not available immediately. It arrives at the end of the stream.

### How stop_reason Appears in Streaming

```python
# Streaming response — stop_reason arrives in message_delta event
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Tell me a short story."}]
) as stream:
    for event in stream:
        if event.type == "message_start":
            # stop_reason is NULL here — not yet determined
            print(f"Message started. stop_reason: {event.message.stop_reason}")
            # Output: Message started. stop_reason: None

        elif event.type == "content_block_delta":
            # Text is streaming in
            if event.delta.type == "text_delta":
                print(event.delta.text, end="", flush=True)

        elif event.type == "message_delta":
            # stop_reason is provided HERE, at the end of the stream
            print(f"\nFinal stop_reason: {event.delta.stop_reason}")
            # Output: Final stop_reason: end_turn
```

### Key Streaming Facts for the Exam

| Event | stop_reason Available? | Notes |
|-------|----------------------|-------|
| `message_start` | No — always `null` | Message metadata only |
| `content_block_start` | No | Content block metadata |
| `content_block_delta` | No | Streaming content chunks |
| `content_block_stop` | No | Content block finished |
| `message_delta` | **Yes** | Final stop_reason provided here |
| `message_stop` | No | Stream complete signal |

### Practical Implication

You cannot make control flow decisions based on `stop_reason` until the stream is complete. If you need to branch on `stop_reason` (e.g., to handle tool calls), you must either:

1. **Collect the full stream first**, then check `stop_reason`
2. **Use the non-streaming API** when control flow decisions are needed immediately

```python
# Pattern: Collect stream, then branch on stop_reason
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    messages=messages
) as stream:
    response = stream.get_final_message()  # Collects the full stream

# Now we can branch on stop_reason
if response.stop_reason == "tool_use":
    handle_tool_call(response)
elif response.stop_reason == "end_turn":
    process_final_response(response)
```

---

## 11. Key Anti-Patterns for Day 2

These are the mistakes the exam tests you on. Each one maps to a specific exam principle.

### Anti-Pattern 1: Ignoring stop_reason and Assuming Response Is Always Complete

**Wrong:**
```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a detailed analysis..."}]
)
# Dangerous — response might be truncated, a tool call, or a refusal
result = response.content[0].text
send_to_downstream_service(result)
```

**Right:**
```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a detailed analysis..."}]
)

if response.stop_reason == "end_turn":
    result = response.content[0].text
    send_to_downstream_service(result)
elif response.stop_reason == "max_tokens":
    # Handle truncation — don't send incomplete data downstream
    result = collect_complete_response(response)
    send_to_downstream_service(result)
elif response.stop_reason == "tool_use":
    # Handle tool call — don't treat tool request as final output
    handle_tool_call(response)
```

### Anti-Pattern 2: Not Handling max_tokens Truncation

**Wrong:**
```python
# Setting max_tokens but never checking if it was hit
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=256,  # Low limit
    messages=[{"role": "user", "content": "Generate a comprehensive report..."}]
)
report = response.content[0].text  # Might be half a report!
save_report(report)
```

**Right:**
```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=256,
    messages=[{"role": "user", "content": "Generate a comprehensive report..."}]
)

if response.stop_reason == "max_tokens":
    raise IncompleteResponseError("Report was truncated. Increase max_tokens or use continuation.")

report = response.content[0].text
save_report(report)
```

### Anti-Pattern 3: Blind Retries Without Checking stop_reason

**Wrong:**
```python
# Retrying without understanding WHY the response was unsatisfactory
for attempt in range(3):
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=messages
    )
    if len(response.content[0].text) > 100:  # Arbitrary length check
        break
    # Retry blindly — doesn't address the root cause
```

**Right:**
```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=messages
)

if response.stop_reason == "max_tokens":
    # Root cause: token limit too low. Fix: increase it.
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,  # Address the actual problem
        messages=messages
    )
elif response.stop_reason == "refusal":
    # Root cause: safety filter. Fix: rephrase the request.
    log_refusal(response)
    messages = rephrase_request(messages)
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=messages
    )
```

### Anti-Pattern 4: Adding Text Blocks After tool_result Causing Empty Responses

**Wrong:**
```python
# Adding explanatory text after tool_result — causes empty responses
messages.append({
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "Result data here"},
        {"type": "text", "text": "I've provided the tool result above. Please continue."}
    ]
})
# Claude may return an empty response with end_turn
```

**Right:**
```python
# Only tool_result blocks — no extra text
messages.append({
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "Result data here"}
    ]
})
# Claude will process the tool result and respond normally
```

If you need to add instructions after a tool result, put them in a **separate** user message:

```python
# If you must add instructions, use a separate message turn
messages.append({
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "Result data here"}
    ]
})
# Let Claude respond to the tool result first, THEN add instructions in a new turn
```

---

## 12. Exam-Relevant Principles

### Principle 1: Programmatic Enforcement Over Vague Instruction

This principle, introduced on Day 1, deepens on Day 2:

- **System prompts influence** but do not guarantee behavior. They set tendencies, not hard constraints.
- **API parameters enforce** behavior. `max_tokens` is a hard ceiling. `stop_sequences` are hard boundaries. `tool_choice` forces tool usage.
- **stop_reason is the verification mechanism**. After Claude responds, `stop_reason` tells you what actually happened — not what you hoped would happen.

```python
# Influence (system prompt) — Claude USUALLY follows this
system = "Always respond in exactly 3 bullet points."

# Enforcement (API parameter) — Claude ALWAYS stops here
stop_sequences = ["4."]  # Stop before a 4th point

# Verification (stop_reason) — Check what actually happened
if response.stop_reason == "stop_sequence":
    print("Claude was about to write a 4th point — stopped by stop_sequence")
elif response.stop_reason == "end_turn":
    print("Claude finished naturally — verify it wrote exactly 3 points")
```

### Principle 2: stop_reason Is the Primary Control Flow Mechanism

Your application logic should branch on `stop_reason`, not on parsing text content. This is the single most important architectural pattern for Claude API applications.

- `end_turn` → Process the response
- `max_tokens` → Handle truncation
- `stop_sequence` → Handle the boundary
- `tool_use` → Execute the tool and continue
- `pause_turn` → Continue the conversation
- `refusal` → Handle gracefully
- `model_context_window_exceeded` → Reduce context

### Principle 3: Always Check stop_reason Before Processing Content

This is the defensive programming principle for Claude API applications:

```python
# ALWAYS check stop_reason FIRST
response = client.messages.create(...)

# Step 1: Check stop_reason
if response.stop_reason == "max_tokens":
    raise IncompleteResponseError("Response truncated")

if response.stop_reason == "tool_use":
    return handle_tool_calls(response)

if response.stop_reason == "refusal":
    return handle_refusal(response)

# Step 2: Only THEN process content
result = response.content[0].text
```

### Principle 4: System Prompts Define Identity, User Messages Define Tasks

Keep the separation clean:
- System prompt = who Claude is (role, rules, constraints, tone)
- User message = what Claude should do (task, data, question)

This separation enables reusability, caching, and clarity.

---


## 13. Quick Reference Card

### System Prompt Checklist

| Element | Guideline | Example |
|---------|-----------|---------|
| **Role** | Always define Claude's role | "You are a senior backend engineer" |
| **Expertise** | Specify domain knowledge | "specializing in distributed systems" |
| **Constraints** | Define what Claude should NOT do | "Never suggest deleting production data" |
| **Tone** | Set communication style | "Professional but approachable" |
| **Format** | Influence (not guarantee) output structure | "Structure responses with Summary, Details, and Next Steps" |

### System Prompt Placement Rules

| Content Type | Where It Goes |
|-------------|--------------|
| Role definition | `system` parameter |
| Persistent behavioral rules | `system` parameter |
| Global constraints and guardrails | `system` parameter |
| Task-specific instructions | User message |
| Data to process | User message |
| Examples for the current task | User message |
| Reference documents | User message (or retrieval) |

### Anthropic's Key Tip

> "Use the system parameter to set Claude's role. Put everything else, like task-specific instructions, in the user turn instead."

### stop_reason Decision Table (Complete)

| stop_reason | Response Complete? | Action Required | Common Cause |
|-------------|-------------------|-----------------|--------------|
| `end_turn` | Yes | Process normally | Claude finished naturally |
| `max_tokens` | **No — truncated** | Increase max_tokens or continue | Output exceeded token limit |
| `stop_sequence` | Depends on use case | Check which sequence matched | Hit a custom boundary |
| `tool_use` | **No — waiting** | Execute tool, return result | Claude needs external data |
| `pause_turn` | **No — paused** | Send response back to continue | Server tool loop limit |
| `refusal` | N/A — refused | Handle gracefully, rephrase | Safety filter triggered |
| `model_context_window_exceeded` | Partial | Reduce input, summarize history | Input + output > context window |

### stop_reason in Streaming

| Stream Event | stop_reason Available? |
|-------------|----------------------|
| `message_start` | No — always `null` |
| `content_block_start` | No |
| `content_block_delta` | No |
| `content_block_stop` | No |
| `message_delta` | **Yes — final value** |
| `message_stop` | No |

### The Complete Control Flow Pattern

```python
def process_response(response):
    """The exam-ready stop_reason handler."""
    match response.stop_reason:
        case "end_turn":
            return response.content[0].text
        case "max_tokens":
            raise IncompleteResponseError("Truncated")
        case "stop_sequence":
            return response.content[0].text  # Up to the stop sequence
        case "tool_use":
            return execute_tools_and_continue(response)
        case "pause_turn":
            return continue_conversation(response)
        case "refusal":
            return handle_refusal(response)
        case "model_context_window_exceeded":
            return reduce_context_and_retry(response)
```

### Key Facts for the Exam

1. The `system` parameter is **not** a message role — it is a separate top-level parameter
2. Messages must alternate `user` ↔ `assistant`. The `system` parameter is outside this alternation.
3. System prompts **influence** behavior. API parameters **enforce** behavior.
4. `stop_reason` is the **primary control flow mechanism** — always check it before processing content
5. `stop_reason` is part of a **successful** response (HTTP 200). API errors (4xx/5xx) are fundamentally different.
6. In streaming, `stop_reason` is `null` in `message_start` and provided in `message_delta`
7. Don't add text blocks after `tool_result` — it causes empty responses
8. If `max_tokens` truncates a `tool_use` block, the tool call JSON may be incomplete — retry with higher `max_tokens`
9. The four core stop_reason values: `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`
10. Additional stop_reason values on newer models: `pause_turn`, `refusal`, `model_context_window_exceeded`

---

## 14. Scenario Challenge

### Scenario Challenge: Domain 4 — Prompt Engineering & Structured Output

**Scenario**: You are building an automated code review system. The system prompt defines Claude as a senior code reviewer. When users submit code, Claude analyzes it and sometimes calls a `run_linter` tool to get static analysis results before providing its review. During testing, you notice two problems:

1. About 10% of the time, after Claude calls `run_linter` and you return the tool result, Claude responds with an empty message (`content: []`) and `stop_reason: "end_turn"`.
2. Occasionally, Claude's review is cut off mid-sentence, ending abruptly without completing the "Recommendations" section.

Your current code processes every response the same way — it extracts `response.content[0].text` and sends it to the review dashboard.

**Question**: Which combination of fixes addresses BOTH problems?

A) Increase `temperature` to encourage longer responses, and add "Always provide a complete review" to the system prompt

B) Check `stop_reason` for `"max_tokens"` and implement a continuation loop for truncated responses; also verify that you are not adding text blocks after `tool_result` messages in the conversation history

C) Switch to a larger model with a bigger context window, and add retry logic that resends the same request up to 3 times if the response is empty

D) Set `stop_sequences` to prevent Claude from stopping early, and add a system prompt instruction saying "Never return an empty response"

---

**Correct Answer**: **B**

**Explanation**: The two problems have distinct root causes, and option B addresses both:

1. **Empty responses after tool results**: This is the classic anti-pattern of adding text blocks after `tool_result` messages. When you include text content alongside `tool_result` blocks in the user message, Claude may interpret this as the user providing additional input and respond with an empty message. The fix is to ensure `tool_result` messages contain only `tool_result` blocks — no extra text.

2. **Truncated reviews**: Reviews cut off mid-sentence with an incomplete "Recommendations" section is the hallmark of `stop_reason: "max_tokens"`. The response hit the token ceiling. The fix is to check `stop_reason` for `"max_tokens"` and either increase the limit or implement a continuation loop.

- **Why A is wrong**: `temperature` controls randomness, not response length or completeness. "Always provide a complete review" is a vague instruction (anti-pattern) — it cannot prevent `max_tokens` truncation, which is a hard limit.
- **Why C is wrong**: A larger context window doesn't fix empty responses caused by malformed `tool_result` messages. Blind retries don't address the root cause of either problem — the same malformed message will produce the same empty response, and the same `max_tokens` limit will truncate the same way.
- **Why D is wrong**: `stop_sequences` prevent Claude from stopping at specific text patterns — they don't prevent `max_tokens` truncation or empty responses. "Never return an empty response" is a vague instruction that cannot override the structural issue with `tool_result` message formatting.

**Domain Reference**: Domain 4: Prompt Engineering & Structured Output (20% weighting)
**Exam Principles Tested**: Always check `stop_reason` before processing content; don't add text blocks after `tool_result`; programmatic enforcement over vague instruction

---

## Sources

All technical content in this document is based on the official Anthropic documentation. Content was rephrased for compliance with licensing restrictions.

- [Messages API Reference](https://docs.anthropic.com/en/api/messages) — API endpoint specifications, request/response structure, and the `system` parameter
- [Handling stop reasons](https://docs.anthropic.com/en/api/handling-stop-reasons) — stop_reason values, control flow patterns, and edge cases
- [System prompts](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts) — System prompt best practices, role assignment, and XML tag usage
- [Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — General prompting guidance including specificity, examples, and output formatting
- [Streaming Messages](https://docs.anthropic.com/en/api/streaming) — Streaming event types and when stop_reason becomes available
- [Tool use overview](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview) — Tool definitions, tool_result message structure, and interaction with system prompts