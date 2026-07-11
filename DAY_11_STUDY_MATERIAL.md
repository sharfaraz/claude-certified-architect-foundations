# Day 11 — Scenario Deep-Dive: Structured Data Extraction (Advanced Patterns)

**Domain**: Domain 4: Prompt Engineering & Structured Output + Domain 2: Tool Design & MCP Integration
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Multi-Step Extraction](#1-multi-step-extraction)
2. [Schema Evolution for Varying Documents](#2-schema-evolution-for-varying-documents)
3. [Validation Pipelines with Error Feedback](#3-validation-pipelines-with-error-feedback)
4. [Batch Extraction with Message Batches API](#4-batch-extraction-with-message-batches-api)
5. [Edge Cases and Strategies](#5-edge-cases-and-strategies)
6. [Practice Challenges](#6-practice-challenges)

---

## 1. Multi-Step Extraction

Some documents require multiple extraction passes — first identifying what kind of document it is, then extracting type-specific fields:

```python
# Step 1: Classify the document type
class DocumentClassification(BaseModel):
    document_type: str = Field(description="One of: invoice, receipt, contract, letter")
    confidence: float = Field(ge=0, le=1)
    language: str = Field(description="ISO language code of the document")

# Step 2: Extract based on classification
class InvoiceFields(BaseModel):
    vendor: str
    amount: float
    line_items: List[LineItem]

class ContractFields(BaseModel):
    parties: List[str]
    effective_date: str
    term_length: Optional[str] = None

# Route to the correct extraction tool based on Step 1
def extract_document(text: str):
    # First pass: classify
    classification = classify_document(text)
    
    # Second pass: type-specific extraction
    if classification.document_type == "invoice":
        return extract_with_tool(text, "extract_invoice", InvoiceFields)
    elif classification.document_type == "contract":
        return extract_with_tool(text, "extract_contract", ContractFields)
```

### Chaining Tool Calls

In multi-turn extraction, each tool result feeds the next step:

```python
messages = [
    {"role": "user", "content": f"Classify this document:\n\n{text}"},
]

# First call: classification
response1 = client.messages.create(
    tools=classification_tools,
    tool_choice={"type": "tool", "name": "classify_document"},
    messages=messages,
    ...
)

# Get classification result
doc_type = response1.content[0].input["document_type"]

# Second call: extraction with type-specific tool
response2 = client.messages.create(
    tools=extraction_tools[doc_type],
    tool_choice={"type": "tool", "name": f"extract_{doc_type}"},
    messages=[{"role": "user", "content": f"Extract {doc_type} fields from:\n\n{text}"}],
    ...
)
```

---

## 2. Schema Evolution for Varying Documents

Real-world documents change over time. Your schemas must handle version differences:

### Strategy: Union Types with Discriminator

```python
from typing import Union, Literal

class InvoiceV1(BaseModel):
    version: Literal["v1"] = "v1"
    vendor: str
    amount: float
    date: str

class InvoiceV2(BaseModel):
    version: Literal["v2"] = "v2"
    vendor: str
    subtotal: float
    tax: float
    total: float
    date: str
    purchase_order: Optional[str] = None

# Use the more comprehensive schema for extraction
# Post-process to normalize v1/v2 differences
```

### Strategy: Generous Optional Fields

```python
class RobustInvoiceExtraction(BaseModel):
    """Handles both old and new invoice formats."""
    vendor_name: str = Field(description="Vendor/supplier name — always present")
    
    # Always present in some form
    total_amount: float = Field(description="Total amount due")
    
    # May or may not be present depending on format
    subtotal: Optional[float] = Field(default=None, description="Subtotal before tax")
    tax_amount: Optional[float] = Field(default=None, description="Tax amount if itemized")
    discount: Optional[float] = Field(default=None, description="Discount if applied")
    
    # Newer fields that older documents won't have
    purchase_order: Optional[str] = Field(default=None, description="PO number if referenced")
    payment_method: Optional[str] = Field(default=None, description="Payment method if stated")
```

---

## 3. Validation Pipelines with Error Feedback

When extraction fails validation, feed the error back to Claude for correction:

```python
def extract_with_feedback(text: str, max_attempts: int = 3) -> Optional[InvoiceData]:
    messages = [{"role": "user", "content": f"Extract invoice data:\n\n{text}"}]
    
    for attempt in range(max_attempts):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            tools=tools,
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages
        )
        
        for block in response.content:
            if block.type == "tool_use":
                try:
                    return InvoiceData.model_validate(block.input)
                except ValidationError as e:
                    # Feed error back for correction
                    messages = [
                        {"role": "user", "content": f"Extract invoice data:\n\n{text}"},
                        {"role": "assistant", "content": [block.to_dict()]},
                        {"role": "user", "content": [
                            {"type": "tool_result", "tool_use_id": block.id, 
                             "content": f"Validation failed: {e}. Please fix and try again.",
                             "is_error": True}
                        ]}
                    ]
    
    return None  # Failed after max attempts
```

---

## 4. Batch Extraction with Message Batches API

For processing document collections at scale:

```python
def build_extraction_batch(documents: List[dict]) -> list:
    """Build batch requests for a collection of documents."""
    requests = []
    for doc in documents:
        requests.append({
            "custom_id": doc["id"],
            "params": {
                "model": "claude-sonnet-4-20250514",
                "max_tokens": 2048,
                "tools": tools,
                "tool_choice": {"type": "tool", "name": "extract_invoice"},
                "messages": [{"role": "user", "content": f"Extract invoice data:\n\n{doc['text']}"}]
            }
        })
    return requests

# Submit batch
batch = client.messages.batches.create(requests=build_extraction_batch(all_documents))

# Process results with validation
results = {}
for result in client.messages.batches.results(batch.id):
    if result.result.type == "succeeded":
        for block in result.result.message.content:
            if block.type == "tool_use":
                try:
                    results[result.custom_id] = InvoiceData.model_validate(block.input)
                except ValidationError:
                    results[result.custom_id] = None  # Mark for manual review
```

---

## 5. Edge Cases and Strategies

| Edge Case | Strategy |
|-----------|----------|
| Missing fields | Use `Optional` with `default=None` |
| Conflicting data | Add `confidence` field; extract both values |
| Multi-language docs | Add `language` field; use system prompt for language handling |
| Scanned/OCR text | Add `extraction_quality` field; lower confidence for garbled text |
| Multiple entities | Use `List[Entity]` in schema for repeated structures |
| Dates in various formats | Use `str` type with description specifying target format |

---

## 6. Practice Challenges

### Challenge (from SPEC)

> Your extraction pipeline processes customer feedback forms. Some forms include a "product_id" field and others don't. Your Pydantic model currently requires product_id. Extracted data fails validation for ~30% of forms. What is the best fix?
>
> A) Remove product_id from the Pydantic model entirely
> B) Make product_id an `Optional[str]` field with a default of `None` and update the JSON Schema accordingly
> C) Add a system prompt instruction telling Claude to always generate a product_id
> D) Post-process the output to insert a placeholder product_id when missing

**Answer**: B — Making the field optional correctly models real-world variability. A loses data, C forces hallucination, D is fragile post-processing.

### Challenge 2

> You're extracting data from medical forms that contain dates in multiple formats: "03/15/2024", "March 15, 2024", "2024-03-15", and "15 Mar 24". What schema design handles this best?
>
> A) `date: str = Field(pattern=r"^\d{4}-\d{2}-\d{2}$")`
> B) `date: str = Field(description="Date in YYYY-MM-DD format, normalized from any input format")`
> C) `date: str = Field(description="Date exactly as it appears in the document")`
> D) `date: int = Field(description="Unix timestamp of the date")`

**Answer**: B — Asking Claude to normalize to a standard format via the description gives you consistent output while handling any input format. A rejects valid extractions. C gives inconsistent output. D is an unreasonable ask for an LLM.
