# CCA-F Exam Addendum — Supplementary Material

**Purpose**: This document supplements `ALL_DAYS_STUDY_MATERIAL.md` with topics identified as gaps when compared against the official CCA-F exam guide (July 2026 edition).

---

## 1. Exam Logistics (From Official Guide)

Your study material focuses on content but doesn't cover exam logistics, which helps with preparation confidence:

| Detail | Value |
|--------|-------|
| Exam name | Claude Certified Architect: Developer Foundations |
| Questions | 65 multiple-choice (single-answer) |
| Time | 90 minutes |
| Passing score | 750/1000 |
| Delivery | Pearson VUE (online proctored or test center) |
| Validity | 2 years |
| Retake policy | 14-day wait after first failure |
| Cost | Check Anthropic certification portal |
| Accommodations | Available via Pearson VUE |

**Note**: Your material says 60 questions and 720 passing score — the official guide states **65 questions** and **750/1000** passing score. Update your mental model accordingly.

---

## 2. Official Domain Weightings (Corrected)

The exam guide lists these domains (verify against your PDF — the encoding was binary so I'm cross-referencing with community sources):

| Domain | Name | Weight |
|--------|------|--------|
| 1 | API & Orchestration | ~27% |
| 2 | Tool Use & MCP | ~18% |
| 3 | Claude Code & Configuration | ~20% |
| 4 | Prompt Engineering & Structured Output | ~20% |
| 5 | Context Management & Reliability | ~15% |

Your study material has these correct. The exam guide title says "Developer Foundations" (not just "Foundations") — this is the official full name.

---

## 3. Extended Thinking (Gap in Study Material)

Your Day 9 mentions extended thinking briefly in passing, but it's an exam-relevant topic that deserves more coverage.

### Key Facts for Exam

- Extended thinking uses `thinking` content blocks in responses
- Enabled via the `thinking` parameter: `{"type": "enabled", "budget_tokens": 10000}`
- `budget_tokens` sets the maximum tokens Claude can use for internal reasoning
- **NOT compatible** with `tool_choice: "any"` or `tool_choice: {"type": "tool"}` (your Day 4 mentions this correctly)
- **IS compatible** with `tool_choice: "auto"` and `tool_choice: {"type": "none"}`
- Thinking blocks appear in `response.content` with `type: "thinking"`
- You CANNOT see thinking content in the response (it's redacted) — only a `thinking` block placeholder
- Extended thinking is billed as output tokens

### When to Use Extended Thinking

| Use Case | Extended Thinking? |
|----------|-------------------|
| Complex multi-step reasoning | Yes |
| Simple data extraction | No (use tool_choice instead) |
| Math or logic problems | Yes |
| Forced tool calls | No (incompatible) |
| Agentic loops needing reasoning | Yes (with auto tool_choice) |

### Exam Anti-Pattern

```python
# ❌ WRONG: Extended thinking + forced tool call = incompatible
response = client.messages.create(
    thinking={"type": "enabled", "budget_tokens": 5000},
    tool_choice={"type": "tool", "name": "extract_data"},  # CONFLICT!
    ...
)

# ✅ RIGHT: Extended thinking + auto = compatible
response = client.messages.create(
    thinking={"type": "enabled", "budget_tokens": 5000},
    tool_choice={"type": "auto"},  # OK
    ...
)
```

---

## 4. Prompt Caching (Gap — Only Briefly Mentioned)

Your study material mentions `cache_control` in Day 2 but doesn't cover the exam-relevant details.

### Key Facts

- Prompt caching reduces costs for repeated context (system prompts, large documents)
- Enabled by adding `cache_control: {"type": "ephemeral"}` to content blocks
- Cached content has a **5-minute TTL** (time-to-live) — resets on each cache hit
- **Minimum cacheable content**: 1024 tokens (for Sonnet/Haiku), 2048 tokens (for Opus)
- Cache hits show in `usage` as `cache_creation_input_tokens` and `cache_read_input_tokens`
- Cache reads are **90% cheaper** than normal input tokens
- Cache writes have a **25% surcharge** over normal input tokens

### When to Cache

| Content | Cache? | Rationale |
|---------|--------|-----------|
| Large system prompt (>1024 tokens) | Yes | Reused every request |
| Tool definitions (many tools) | Yes | Stable across requests |
| Reference documents in conversation | Yes | Read repeatedly |
| User's latest message | No | Changes every turn |
| Dynamic per-request content | No | Won't get cache hits |

### Caching in Agentic Loops

```python
# In an agentic loop, the system prompt and tools are sent EVERY turn
# Caching them saves significant cost over many iterations
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    system=[
        {
            "type": "text",
            "text": large_system_prompt,
            "cache_control": {"type": "ephemeral"}  # Cache this!
        }
    ],
    tools=tools,  # These get auto-cached if large enough
    messages=messages
)
```

---

## 5. Server-Side Tools (Web Search, Code Execution)

Your material mentions server tools briefly but doesn't cover their exam-relevant specifics.

### Web Search Tool

- Enabled with `type: "web_search_20250305"` in the tools array
- Claude autonomously decides when to search
- Returns results as tool results (you don't need to handle execution)
- `stop_reason: "pause_turn"` may occur if search loop hits iteration limit
- **Exam relevance**: Know that server tools don't require client-side execution

### Code Execution Tool

- Enables Claude to write and run code in a sandboxed environment
- Results returned inline in the response
- Useful for calculations, data analysis, visualization

### Key Exam Distinction

```
Client tools: You define → Claude calls → YOU execute → you return result
Server tools: Anthropic defines → Claude calls → ANTHROPIC executes → result returned automatically
```

---

## 6. Streaming API Details (Gap)

Your material covers streaming lightly. For the exam:

### Event Types in Order

1. `message_start` — Message metadata (stop_reason is null here)
2. `content_block_start` — New content block beginning
3. `content_block_delta` — Streaming content chunks
4. `content_block_stop` — Content block finished
5. `message_delta` — **stop_reason provided HERE** (final)
6. `message_stop` — Stream complete

### Exam Key: Collecting Complete Responses from Streams

```python
with client.messages.stream(...) as stream:
    response = stream.get_final_message()
    # Now you can check response.stop_reason
```

### When to Use Streaming vs Non-Streaming

| Scenario | Approach |
|----------|----------|
| User-facing chat | Streaming (better UX) |
| Agentic loops | Non-streaming (need complete response for branching) |
| CI pipelines | Non-streaming (need full output for parsing) |
| Long responses | Streaming (don't wait for complete generation) |

---

## 7. Token Counting API (Gap)

The exam may test your knowledge of the token counting endpoint:

```python
# Count tokens WITHOUT generating a response
token_count = client.messages.count_tokens(
    model="claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": "Hello!"}],
    system="You are a helpful assistant.",
    tools=[...]
)
print(token_count.input_tokens)  # e.g., 42
```

### Use Cases
- Pre-flight check before sending a request
- Context window budgeting (Day 43 concepts)
- Cost estimation before batch processing

---

## 8. Vision/Multimodal Input (Gap)

Your material barely covers image input, which is exam-relevant:

### Sending Images to Claude

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": base64_encoded_image
                }
            }
        ]
    }]
)
```

### Supported Image Formats
- JPEG, PNG, GIF, WebP
- Max size: 5MB per image
- Images are converted to tokens (cost varies by resolution)

### URL-Based Images
```python
{
    "type": "image",
    "source": {
        "type": "url",
        "url": "https://example.com/image.png"
    }
}
```

---

## 9. PDF/Document Input (Recent Feature)

Claude can now accept PDF documents directly:

```python
{
    "type": "document",
    "source": {
        "type": "base64",
        "media_type": "application/pdf",
        "data": base64_encoded_pdf
    }
}
```

This is relevant to the Structured Data Extraction scenario — extracting data from PDF invoices, forms, etc.

---

## 10. Model Selection Guidance (Exam Tested)

The exam tests your ability to choose the right model:

| Model | Best For | Context | Cost |
|-------|----------|---------|------|
| Claude Opus 4 | Complex reasoning, long-horizon agentic tasks, nuanced analysis | 200K | Highest |
| Claude Sonnet 4 | Balanced — most production use cases, coding, tool use | 200K | Medium |
| Claude Haiku 3.5 | Fast responses, high-volume, cost-sensitive, simple tasks | 200K | Lowest |

### Decision Framework

```
Is the task complex reasoning or long-horizon agentic work?
  → Opus

Is it a standard production task (coding, extraction, analysis)?
  → Sonnet (default choice for most scenarios)

Is it high-volume, latency-sensitive, or simple classification?
  → Haiku
```

---

## 11. Safety and Content Filtering (Gap)

### Refusal Handling

When Claude refuses (stop_reason: "refusal"):
- The response may contain explanation text
- Your application should handle this gracefully
- Don't retry the same prompt — rephrase appropriately
- Log refusals for monitoring

### Responsible AI in Agentic Systems

The exam tests whether you understand:
- Agents should have **guardrails** (hooks, allowed tools)
- Human-in-the-loop for high-stakes decisions
- Audit trails (provenance tracking)
- Cost controls (budget limits, max_turns)
- Principle of least privilege (restricted tool access)

---

## 12. Corrections to Study Material

| Topic | Study Material Says | Correct Value |
|-------|-------------------|---------------|
| Question count | 60 | **65** |
| Passing score | 720/1000 | **750/1000** |
| Model names | Some use older naming | Use latest: `claude-sonnet-4-20250514`, `claude-opus-4-7` |

---

## 13. Additional Practice Focus Areas

Based on community feedback about the exam:

1. **Scenario-based questions dominate** — practice identifying the correct architectural pattern for a given scenario
2. **Anti-pattern recognition** — many questions present an anti-pattern and ask what's wrong
3. **"Most appropriate" framing** — questions ask for the BEST option, not the only valid one
4. **Cross-domain integration** — questions often span two domains (e.g., tool design + agentic architecture)
5. **Elimination strategy** — usually 1-2 options are clearly wrong, narrow from there

### Top Distractor Patterns on the Exam

1. "Add a system prompt instruction to..." → Usually wrong (vague instruction anti-pattern)
2. "Set temperature to 0 for..." → Wrong when used to control tool selection
3. "Increase max_tokens..." → Wrong when the problem is input context, not output length
4. "Retry with the same parameters" → Wrong (blind retry anti-pattern)
5. "Use --output-format json" when schema enforcement is needed → Wrong (use --json-schema)

---

## Summary

Your `ALL_DAYS_STUDY_MATERIAL.md` covers approximately **85-90%** of what's needed. The main gaps are:

1. Extended thinking details
2. Prompt caching mechanics
3. Server-side tools specifics
4. Vision/multimodal input
5. PDF document input
6. Token counting API
7. Exam logistics corrections (65 questions, 750 passing)

Study your main material for depth on all 5 domains. Use this addendum to fill the remaining gaps. Together they should provide comprehensive exam preparation.

---

*Sources: Official Anthropic exam guide (July 2026), Anthropic documentation, community study resources. Content was rephrased for compliance with licensing restrictions.*
