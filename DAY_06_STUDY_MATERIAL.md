# Day 6 — Pydantic Model Integration

**Domain**: Domain 4: Prompt Engineering & Structured Output (20% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: Continue Building with the Claude API (Pod 2)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Pydantic Fundamentals](#1-pydantic-fundamentals)
2. [BaseModel for Structured Output Types](#2-basemodel-for-structured-output-types)
3. [Generating JSON Schema from Pydantic Models](#3-generating-json-schema-from-pydantic-models)
4. [Validating Claude's tool_use Output](#4-validating-claudes-tool_use-output)
5. [Field Validators and Constraints](#5-field-validators-and-constraints)
6. [The End-to-End Pattern](#6-the-end-to-end-pattern)
7. [Why Pydantic + tool_choice Is the Gold Standard](#7-why-pydantic--tool_choice-is-the-gold-standard)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video Resources

- [Anthropic's Prompt Engineering Tutorial — Structured Output](https://github.com/anthropics/prompt-eng-interactive-tutorial) — Official Anthropic interactive notebook covering structured output patterns
- [Pydantic Official Docs — Model JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/) — How `.model_json_schema()` works

---

## 1. Pydantic Fundamentals

Pydantic is a Python data validation library that uses Python type annotations to define data schemas. In the context of the Claude API, Pydantic serves as the bridge between your Python application's type system and the JSON Schema that Claude uses for structured output.

**Why this matters for the exam**: Pydantic is the programmatic enforcement mechanism for structured output. It transforms the abstract concept of "validate Claude's output" into concrete, testable code. The exam tests whether you understand the full pipeline: Pydantic model → JSON Schema → tool definition → Claude's tool_use response → Pydantic validation.

### Core Concepts

| Concept | Purpose | Example |
|---------|---------|---------|
| `BaseModel` | Define a data structure with typed fields | `class User(BaseModel): name: str` |
| `.model_json_schema()` | Generate JSON Schema from a Pydantic model | `User.model_json_schema()` |
| `.model_validate()` | Validate a dictionary against the model | `User.model_validate({"name": "Alice"})` |
| `Field()` | Add constraints and descriptions to fields | `Field(description="User's full name")` |
| `field_validator` | Custom validation logic for individual fields | `@field_validator('email')` |

---

## 2. BaseModel for Structured Output Types

Every Pydantic model inherits from `BaseModel`. Fields are defined using Python type annotations:

```python
from pydantic import BaseModel, Field
from typing import Optional, List
from enum import Enum

class Sentiment(str, Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

class FeedbackAnalysis(BaseModel):
    """Analysis of a customer feedback message."""
    sentiment: Sentiment = Field(description="Overall sentiment of the feedback")
    confidence: float = Field(description="Confidence score between 0 and 1", ge=0, le=1)
    key_topics: List[str] = Field(description="Main topics mentioned in the feedback")
    action_required: bool = Field(description="Whether this feedback requires follow-up action")
    summary: str = Field(description="One-sentence summary of the feedback")
    product_id: Optional[str] = Field(default=None, description="Product ID if mentioned")
```

### Type Mapping: Python → JSON Schema

| Python Type | JSON Schema Type | Notes |
|-------------|-----------------|-------|
| `str` | `"string"` | Basic text |
| `int` | `"integer"` | Whole numbers |
| `float` | `"number"` | Decimal numbers |
| `bool` | `"boolean"` | True/false |
| `List[str]` | `{"type": "array", "items": {"type": "string"}}` | Typed arrays |
| `Optional[str]` | `{"anyOf": [{"type": "string"}, {"type": "null"}]}` | Nullable fields |
| `Enum` | `{"enum": [...]}` | Fixed set of values |
| Nested `BaseModel` | Nested `"object"` with `$defs` | Composition |

---

## 3. Generating JSON Schema from Pydantic Models

The `.model_json_schema()` method generates a complete JSON Schema from your Pydantic model. This schema can be used directly as the `input_schema` for a Claude API tool definition:

```python
import json

schema = FeedbackAnalysis.model_json_schema()
print(json.dumps(schema, indent=2))
```

**Output:**

```json
{
  "title": "FeedbackAnalysis",
  "description": "Analysis of a customer feedback message.",
  "type": "object",
  "properties": {
    "sentiment": {
      "description": "Overall sentiment of the feedback",
      "enum": ["positive", "negative", "neutral"],
      "title": "Sentiment",
      "type": "string"
    },
    "confidence": {
      "description": "Confidence score between 0 and 1",
      "maximum": 1,
      "minimum": 0,
      "title": "Confidence",
      "type": "number"
    },
    "key_topics": {
      "description": "Main topics mentioned in the feedback",
      "items": {"type": "string"},
      "title": "Key Topics",
      "type": "array"
    },
    "action_required": {
      "description": "Whether this feedback requires follow-up action",
      "title": "Action Required",
      "type": "boolean"
    },
    "summary": {
      "description": "One-sentence summary of the feedback",
      "title": "Summary",
      "type": "string"
    },
    "product_id": {
      "anyOf": [{"type": "string"}, {"type": "null"}],
      "default": null,
      "description": "Product ID if mentioned",
      "title": "Product Id"
    }
  },
  "required": ["sentiment", "confidence", "key_topics", "action_required", "summary"]
}
```

**Key observation**: `product_id` is NOT in `required` because it has a default value (`None`). Pydantic automatically determines which fields are required based on whether they have defaults.

---

## 4. Validating Claude's tool_use Output

After Claude returns a `tool_use` content block, you validate the `input` field against your Pydantic model:

```python
import anthropic
from pydantic import BaseModel, Field, ValidationError

client = anthropic.Anthropic()

# Define the tool using the Pydantic-generated schema
tools = [
    {
        "name": "analyze_feedback",
        "description": "Analyze customer feedback and extract structured insights",
        "input_schema": FeedbackAnalysis.model_json_schema()
    }
]

# Force Claude to use the tool
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "analyze_feedback"},
    messages=[
        {"role": "user", "content": "Analyze this feedback: 'The new dashboard is amazing! Loading times are much faster and the UI is intuitive. Would love to see dark mode added though.'"}
    ]
)

# Extract and validate the tool_use output
for block in response.content:
    if block.type == "tool_use":
        try:
            result = FeedbackAnalysis.model_validate(block.input)
            print(f"Sentiment: {result.sentiment}")
            print(f"Confidence: {result.confidence}")
            print(f"Topics: {result.key_topics}")
        except ValidationError as e:
            print(f"Validation failed: {e}")
            # Handle the error — retry, log, or escalate
```

### Why Validation Matters

Even with `tool_choice` forcing Claude to call the tool and a JSON Schema defining the structure, you should still validate the output with Pydantic because:

1. **Schema enforcement is not 100% guaranteed** — edge cases exist, especially with complex schemas
2. **Business rule validation** — Pydantic validators can enforce rules beyond what JSON Schema expresses (e.g., "if sentiment is negative, action_required must be true")
3. **Type coercion** — Pydantic can handle minor type mismatches (e.g., `"42"` → `42`)
4. **Defense in depth** — multiple layers of validation catch more errors

---

## 5. Field Validators and Constraints

Pydantic supports both declarative constraints (via `Field()`) and custom validation logic (via `@field_validator`):

### Declarative Constraints

```python
from pydantic import BaseModel, Field

class InvoiceExtraction(BaseModel):
    vendor_name: str = Field(min_length=1, description="Name of the vendor")
    amount: float = Field(gt=0, description="Invoice amount in USD")
    date: str = Field(pattern=r"^\d{4}-\d{2}-\d{2}$", description="Invoice date in YYYY-MM-DD format")
    line_items: List[str] = Field(min_length=1, description="At least one line item required")
```

### Custom Validators

```python
from pydantic import BaseModel, Field, field_validator

class RefundRequest(BaseModel):
    order_id: str = Field(description="The order ID to refund")
    amount: float = Field(gt=0, description="Refund amount in USD")
    reason: str = Field(description="Reason for the refund")

    @field_validator('order_id')
    @classmethod
    def validate_order_id_format(cls, v):
        if not v.startswith('ORD-'):
            raise ValueError('Order ID must start with ORD-')
        return v

    @field_validator('amount')
    @classmethod
    def validate_refund_limit(cls, v):
        if v > 10000:
            raise ValueError('Refund amount cannot exceed $10,000 without manager approval')
        return v
```

---

## 6. The End-to-End Pattern

This is the complete pipeline the exam expects you to know:

```
Pydantic Model  →  JSON Schema  →  Tool Definition  →  API Call  →  tool_use Response  →  Pydantic Validation
```

```python
import anthropic
from pydantic import BaseModel, Field, ValidationError
from typing import List, Optional

# Step 1: Define the Pydantic model
class ContactInfo(BaseModel):
    full_name: str = Field(description="Person's full name")
    email: Optional[str] = Field(default=None, description="Email address if found")
    phone: Optional[str] = Field(default=None, description="Phone number if found")
    company: Optional[str] = Field(default=None, description="Company name if mentioned")

# Step 2: Generate JSON Schema
schema = ContactInfo.model_json_schema()

# Step 3: Create tool definition
tools = [{
    "name": "extract_contact",
    "description": "Extract contact information from text",
    "input_schema": schema
}]

# Step 4: Call Claude with tool_choice enforcement
client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_contact"},
    messages=[{"role": "user", "content": "Extract contact info from: John Smith from Acme Corp, reach him at john@acme.com or 555-0123"}]
)

# Step 5: Validate with Pydantic
for block in response.content:
    if block.type == "tool_use":
        try:
            contact = ContactInfo.model_validate(block.input)
            # contact is now a fully validated, typed Python object
            print(f"Name: {contact.full_name}")
            print(f"Email: {contact.email}")
        except ValidationError as e:
            print(f"Extraction failed validation: {e}")
```

---

## 7. Why Pydantic + tool_choice Is the Gold Standard

This combination provides **three layers of enforcement**:

| Layer | Mechanism | What It Guarantees |
|-------|-----------|-------------------|
| 1. Force tool use | `tool_choice: {"type": "tool", "name": "..."}` | Claude WILL call the tool (not respond with text) |
| 2. Schema structure | `input_schema` (from `.model_json_schema()`) | Output matches the defined JSON structure |
| 3. Business validation | Pydantic's `model_validate()` | Output passes custom business rules |

**Contrast with the vague instruction approach:**

```python
# ❌ WRONG: Vague instruction — no enforcement
system = "Please return the contact info as JSON with fields: name, email, phone, company"
# Claude might: return markdown, miss fields, use different field names, add extra text

# ✅ RIGHT: Programmatic enforcement — guaranteed structure
tool_choice = {"type": "tool", "name": "extract_contact"}
# Claude MUST call the tool with data matching the schema
```

---

## 8. Anti-Patterns

### Anti-Pattern 1: Using Pydantic Without tool_choice

```python
# ❌ Schema defined, but Claude might respond with text instead of using the tool
tools = [{"name": "extract", "description": "...", "input_schema": schema}]
# No tool_choice — Claude decides whether to use the tool
response = client.messages.create(tools=tools, messages=[...])
# Claude might return: "Here's the contact info: John Smith, john@acme.com"
```

### Anti-Pattern 2: Skipping Validation After tool_use

```python
# ❌ Trusting Claude's output without validation
for block in response.content:
    if block.type == "tool_use":
        data = block.input  # Raw dict — no validation!
        process(data["email"])  # Might crash if email is missing or malformed
```

### Anti-Pattern 3: Over-Complex Schemas

```python
# ❌ Too deeply nested — increases the chance of extraction errors
class Address(BaseModel):
    class GeoCoordinates(BaseModel):
        class Precision(BaseModel):
            level: str
            confidence: float
        lat: float
        lng: float
        precision: Precision
    street: str
    city: str
    geo: GeoCoordinates
```

Keep schemas as flat as possible. One level of nesting is fine; three or more increases failure rates.

---

## 9. Quick Reference Card

### The Pipeline

```
Pydantic BaseModel  →  .model_json_schema()  →  tools[].input_schema  →  tool_choice  →  .model_validate(block.input)
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `MyModel.model_json_schema()` | Generate JSON Schema for API tool definition |
| `MyModel.model_validate(dict)` | Validate a dict and return a typed model instance |
| `MyModel.model_validate_json(json_str)` | Validate a JSON string directly |
| `Field(description="...")` | Add description (Claude reads this!) |
| `Field(ge=0, le=1)` | Numeric constraints |
| `@field_validator('field')` | Custom validation logic |

### Exam Decision Table

| Situation | Approach |
|-----------|----------|
| Need structured extraction | Pydantic + tool_choice + validation |
| Field might be missing | Use `Optional[type]` with `default=None` |
| Fixed set of values | Use `Enum` (becomes `enum` in schema) |
| Business rules beyond types | Use `@field_validator` |
| Multiple extraction targets | Define multiple tools with different Pydantic models |

---

## 10. Scenario Challenge

### Scenario: Structured Data Extraction

**Scenario**: You're building a pipeline that extracts product information from customer reviews. Some reviews mention specific product IDs, others don't. Some include star ratings as numbers, others as text ("five stars"). Your current implementation uses a simple system prompt asking Claude to return JSON, but approximately 20% of responses are either plain text or have missing/malformed fields.

**Question**: What is the most reliable approach to fix this?

A) Add more examples of correct JSON output to the system prompt and set `temperature: 0`

B) Define a Pydantic model with appropriate types and optional fields, generate the JSON Schema, create a tool definition, use `tool_choice` to force the tool, and validate the output with `model_validate()`

C) Parse Claude's text response with regex to extract the JSON, then validate it manually

D) Switch to a larger model (Claude Opus) which follows JSON formatting instructions more reliably

---

**Correct Answer**: **B**

**Explanation**: This is the gold standard pattern — programmatic enforcement at every layer. The Pydantic model defines the structure (with `Optional` for fields that may be absent), `tool_choice` guarantees Claude uses the extraction tool, and `model_validate()` catches any remaining issues.

- **Why A is wrong**: Few-shot examples help but don't guarantee structure. Temperature doesn't control format compliance. This is still a vague instruction approach.
- **Why C is wrong**: Regex parsing of LLM output is fragile and breaks when Claude varies its formatting. This is a brittle workaround, not a solution.
- **Why D is wrong**: Larger models follow instructions better on average, but don't provide the programmatic guarantee that tool_choice + schema enforcement provides. This is a "throw money at it" approach rather than an engineering solution.
