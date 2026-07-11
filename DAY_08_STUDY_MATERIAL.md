# Day 8 — Message Batches API

**Domain**: Domain 4: Prompt Engineering & Structured Output (20% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Continue Building with the Claude API (Pod 3)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [What Is the Message Batches API?](#1-what-is-the-message-batches-api)
2. [When to Use Batches vs Single Requests](#2-when-to-use-batches-vs-single-requests)
3. [Batch Request Structure](#3-batch-request-structure)
4. [Batch Processing Lifecycle](#4-batch-processing-lifecycle)
5. [Retrieving Batch Results](#5-retrieving-batch-results)
6. [Combining Batches with Structured Output](#6-combining-batches-with-structured-output)
7. [Error Handling in Batches](#7-error-handling-in-batches)
8. [Cost and Rate Limit Advantages](#8-cost-and-rate-limit-advantages)
9. [Anti-Patterns](#9-anti-patterns)
10. [Quick Reference Card](#10-quick-reference-card)
11. [Scenario Challenge](#11-scenario-challenge)

---

## Recommended Resources

- [Anthropic Message Batches API Documentation](https://docs.anthropic.com/en/docs/build-with-claude/message-batches) — Official API reference
- [Anthropic Cookbook — Batch Processing](https://platform.claude.com/cookbook/misc-batch-processing) — Practical examples
- [Anthropic Blog — Introducing the Message Batches API](https://claude.com/blog/message-batches-api) — Feature announcement and use cases

---

## 1. What Is the Message Batches API?

The Message Batches API allows you to submit up to 10,000 Messages API requests as a single batch for asynchronous processing. Instead of making thousands of individual API calls (each requiring rate limit management, retry logic, and connection handling), you submit them all at once and retrieve the results later.

**Why this matters for the exam**: The Message Batches API is explicitly listed in Domain 4. The exam tests whether you know when to use batch processing, how the lifecycle works, and how to combine batches with structured output patterns (tool_use + JSON Schema).

### Key Facts

| Property | Value |
|----------|-------|
| Max requests per batch | 10,000 |
| Processing time | Most complete in < 1 hour (guaranteed within 24 hours) |
| Cost savings | 50% discount compared to individual requests |
| Rate limits | Separate from synchronous API limits |
| Availability | All Claude models |

---

## 2. When to Use Batches vs Single Requests

| Use Case | Approach | Why |
|----------|----------|-----|
| Real-time chat | Single request | User expects immediate response |
| Processing 500 documents | Batch | No immediacy required; cost savings |
| Interactive tool use | Single request | Multi-turn requires synchronous flow |
| Nightly classification job | Batch | Offline processing; 50% cost reduction |
| Bulk data extraction | Batch | Large volume; no real-time constraint |
| User-facing API endpoint | Single request | Latency requirements |

**Decision rule**: If the results don't need to be available within seconds, and you have more than ~10 requests to process, use batches.

---

## 3. Batch Request Structure

A batch is created by sending an array of request objects, each containing a `custom_id` and the parameters you'd normally send to the Messages API:

```python
import anthropic

client = anthropic.Anthropic()

# Create a batch of requests
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": "doc-001",
            "params": {
                "model": "claude-sonnet-4-20250514",
                "max_tokens": 1024,
                "messages": [
                    {"role": "user", "content": "Extract the invoice number from: Invoice #INV-2024-0042 dated March 15, 2024"}
                ],
                "tools": [{
                    "name": "extract_invoice",
                    "description": "Extract invoice details",
                    "input_schema": {
                        "type": "object",
                        "properties": {
                            "invoice_number": {"type": "string"},
                            "date": {"type": "string"},
                            "amount": {"type": "number"}
                        },
                        "required": ["invoice_number"]
                    }
                }],
                "tool_choice": {"type": "tool", "name": "extract_invoice"}
            }
        },
        {
            "custom_id": "doc-002",
            "params": {
                "model": "claude-sonnet-4-20250514",
                "max_tokens": 1024,
                "messages": [
                    {"role": "user", "content": "Extract the invoice number from: Bill #B-9981 for $1,250.00"}
                ],
                "tools": [{
                    "name": "extract_invoice",
                    "description": "Extract invoice details",
                    "input_schema": {
                        "type": "object",
                        "properties": {
                            "invoice_number": {"type": "string"},
                            "date": {"type": "string"},
                            "amount": {"type": "number"}
                        },
                        "required": ["invoice_number"]
                    }
                }],
                "tool_choice": {"type": "tool", "name": "extract_invoice"}
            }
        }
    ]
)

print(f"Batch ID: {batch.id}")
print(f"Status: {batch.processing_status}")
```

### The `custom_id` Field

Each request in a batch must have a unique `custom_id`. This is YOUR identifier — it lets you match results back to the original input:

- Use meaningful IDs: `"doc-001"`, `"customer-feedback-42"`, `"invoice-2024-03-15"`
- IDs must be unique within a batch
- IDs are returned with results for correlation

---

## 4. Batch Processing Lifecycle

Batches move through a defined set of states:

```
created  →  in_progress  →  ended
                              ├── succeeded (all requests processed)
                              ├── errored (batch-level failure)
                              └── expired (exceeded 24-hour window)
```

### Polling for Status

```python
import time

# Poll until the batch completes
while True:
    batch_status = client.messages.batches.retrieve(batch.id)
    
    if batch_status.processing_status == "ended":
        print(f"Batch completed!")
        print(f"  Succeeded: {batch_status.request_counts.succeeded}")
        print(f"  Errored: {batch_status.request_counts.errored}")
        print(f"  Expired: {batch_status.request_counts.expired}")
        print(f"  Canceled: {batch_status.request_counts.canceled}")
        break
    
    print(f"Status: {batch_status.processing_status} — waiting...")
    time.sleep(30)  # Poll every 30 seconds
```

### Request Counts Object

When a batch ends, the `request_counts` field tells you what happened to each request:

| Field | Meaning |
|-------|---------|
| `succeeded` | Requests that completed successfully |
| `errored` | Requests that failed (model errors, invalid params) |
| `expired` | Requests that weren't processed within the 24-hour window |
| `canceled` | Requests canceled before processing |

---

## 5. Retrieving Batch Results

Once a batch ends, retrieve results by iterating over the results:

```python
# Retrieve all results
results = client.messages.batches.results(batch.id)

for result in results:
    print(f"Request: {result.custom_id}")
    
    if result.result.type == "succeeded":
        message = result.result.message
        # Process the successful response
        for block in message.content:
            if block.type == "tool_use":
                print(f"  Extracted: {block.input}")
    
    elif result.result.type == "errored":
        error = result.result.error
        print(f"  Error: {error.type} — {error.message}")
    
    elif result.result.type == "expired":
        print(f"  Expired — request was not processed in time")
    
    elif result.result.type == "canceled":
        print(f"  Canceled")
```

### Result Types

| Type | Contains | Action |
|------|----------|--------|
| `succeeded` | Full message response (same as single request) | Process normally |
| `errored` | Error type and message | Log and retry individually |
| `expired` | Nothing — request wasn't processed | Resubmit in a new batch |
| `canceled` | Nothing — you canceled it | Resubmit if still needed |

---

## 6. Combining Batches with Structured Output

The most powerful pattern for the exam: batch processing + tool_use + JSON Schema for reliable bulk extraction:

```python
from pydantic import BaseModel, Field
from typing import Optional, List

class ProductReview(BaseModel):
    sentiment: str = Field(description="positive, negative, or neutral")
    rating: Optional[int] = Field(default=None, description="Star rating 1-5 if mentioned")
    key_points: List[str] = Field(description="Main points from the review")
    purchase_verified: bool = Field(description="Whether the review mentions a verified purchase")

# Build batch requests from a list of reviews
reviews = [
    "Amazing product! 5 stars. Verified purchase. Fast shipping.",
    "Terrible quality. Broke after one day. Would not recommend.",
    "It's okay, nothing special. Does what it says on the box.",
    # ... potentially thousands more
]

batch_requests = []
for i, review in enumerate(reviews):
    batch_requests.append({
        "custom_id": f"review-{i:04d}",
        "params": {
            "model": "claude-sonnet-4-20250514",
            "max_tokens": 512,
            "tools": [{
                "name": "analyze_review",
                "description": "Analyze a product review",
                "input_schema": ProductReview.model_json_schema()
            }],
            "tool_choice": {"type": "tool", "name": "analyze_review"},
            "messages": [
                {"role": "user", "content": f"Analyze this product review:\n\n{review}"}
            ]
        }
    })

# Submit the batch
batch = client.messages.batches.create(requests=batch_requests)
```

---

## 7. Error Handling in Batches

### Individual Request Errors

Individual requests within a batch can fail even if the batch itself succeeds. Common causes:

| Error | Cause | Fix |
|-------|-------|-----|
| `invalid_request_error` | Malformed request parameters | Fix the params and resubmit |
| `overloaded_error` | Model temporarily unavailable | Resubmit in a new batch |
| Request expired | Not processed within 24 hours | Resubmit in a new batch |

### Retry Strategy

```python
# Collect failed requests for retry
failed_requests = []
for result in client.messages.batches.results(batch.id):
    if result.result.type in ("errored", "expired"):
        failed_requests.append(result.custom_id)

if failed_requests:
    # Rebuild and resubmit only the failed requests
    retry_batch = build_batch_for_ids(failed_requests)
    client.messages.batches.create(requests=retry_batch)
```

---

## 8. Cost and Rate Limit Advantages

| Factor | Single Requests | Batch API |
|--------|----------------|-----------|
| Cost per token | Standard pricing | **50% discount** |
| Rate limits | Shared with all API calls | Separate, higher limits |
| Connection overhead | Per-request TCP/TLS handshake | Single submission |
| Retry complexity | You manage retries | Anthropic handles processing |
| Throughput | Limited by rate limits | Up to 10,000 per batch |

**50% cost reduction** means that a pipeline processing 10,000 documents at $0.003/1K input tokens saves $15 per batch run compared to individual requests.

---

## 9. Anti-Patterns

### Anti-Pattern 1: Using Batches for Real-Time Interactions

```python
# ❌ WRONG: Batch for a user-facing feature
user_message = get_user_input()
batch = client.messages.batches.create(requests=[{
    "custom_id": "user-query",
    "params": {"messages": [{"role": "user", "content": user_message}], ...}
}])
# User now waits potentially hours for a response
```

### Anti-Pattern 2: Not Using custom_id for Correlation

```python
# ❌ WRONG: Generic IDs that can't be traced back to input
requests = [{"custom_id": f"req-{i}", "params": {...}} for i in range(1000)]
# When results come back, you can't match them to the original documents
```

### Anti-Pattern 3: Not Handling Partial Failures

```python
# ❌ WRONG: Assuming all requests in a batch succeeded
results = client.messages.batches.results(batch.id)
for result in results:
    data = result.result.message.content[0]  # Crashes if result.type is "errored"
```

---

## 10. Quick Reference Card

### Batch API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Create batch | POST | `/v1/messages/batches` |
| Get batch status | GET | `/v1/messages/batches/{batch_id}` |
| Get results | GET | `/v1/messages/batches/{batch_id}/results` |
| List batches | GET | `/v1/messages/batches` |
| Cancel batch | POST | `/v1/messages/batches/{batch_id}/cancel` |

### Decision Framework

```
Need results immediately?  →  Yes  →  Use single Messages API
                           →  No   →  Processing > 10 items?  →  Yes  →  Use Batch API
                                                                →  No   →  Either works (batch for cost savings)
```

### Batch + Structured Output Recipe

```
1. Define Pydantic model
2. Generate JSON Schema (.model_json_schema())
3. Build tool definition with schema
4. Create batch requests with tool_choice enforcement
5. Submit batch
6. Poll for completion
7. Iterate results, validate each with Pydantic
8. Retry failed requests
```

---

## 11. Scenario Challenge

### Scenario: Domain 4 — Structured Data Extraction at Scale

**Scenario**: Your company processes 5,000 customer support emails every night to extract key information: customer ID, issue category, urgency level, and a brief summary. Currently, this runs as 5,000 individual API calls over 3 hours, and costs approximately $300/night. The team wants to reduce cost and processing time. Results are needed by 6 AM for the morning report (processing starts at midnight).

**Question**: What is the optimal architecture for this pipeline?

A) Use the Message Batches API with `tool_choice` enforcement and a Pydantic-validated extraction schema, submitting all 5,000 requests in a single batch

B) Increase parallelism by running 50 concurrent threads, each making individual API calls with retry logic

C) Switch to Claude Haiku for lower cost per token and run the same sequential pipeline

D) Reduce the volume by sampling 1,000 random emails and extrapolating trends from the sample

---

**Correct Answer**: **A**

**Explanation**: The Message Batches API is designed exactly for this use case — high-volume, non-real-time processing. It provides a 50% cost reduction ($300 → $150/night), handles rate limiting automatically, processes within the 6-hour window easily (most batches complete in < 1 hour), and `tool_choice` + schema ensures reliable structured extraction. The `custom_id` field links results back to source emails.

- **Why B is wrong**: More parallelism helps throughput but doesn't reduce cost. You still fight rate limits and need complex retry logic. The Batch API handles all of this for you at half the price.
- **Why C is wrong**: Switching to Haiku reduces cost but may reduce extraction quality for complex emails. The Batch API's 50% discount on Sonnet may match or beat Haiku pricing while maintaining quality.
- **Why D is wrong**: Sampling loses data. The business requirement is to process ALL 5,000 emails. This changes the business requirement rather than solving the engineering problem.
