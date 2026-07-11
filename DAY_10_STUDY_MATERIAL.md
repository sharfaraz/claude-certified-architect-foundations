# Day 10 — Scenario Deep-Dive: Structured Data Extraction

**Domain**: Domain 4: Prompt Engineering & Structured Output + Domain 2: Tool Design & MCP Integration
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [Scenario Overview](#1-scenario-overview)
2. [The Extraction Pipeline Architecture](#2-the-extraction-pipeline-architecture)
3. [Forcing Extraction with tool_choice](#3-forcing-extraction-with-tool_choice)
4. [Schema Design for Extraction](#4-schema-design-for-extraction)
5. [Handling Partial and Ambiguous Extractions](#5-handling-partial-and-ambiguous-extractions)
6. [Validation and Error Recovery](#6-validation-and-error-recovery)
7. [Practice Challenges](#7-practice-challenges)

---

## Recommended Video Resources

- [Anthropic Courses — Tool Use / Structured Outputs Notebook](https://github.com/anthropics/courses/blob/master/tool_use/03_structured_outputs.ipynb) — Official hands-on notebook

---

## 1. Scenario Overview

The **Structured Data Extraction** scenario is one of 6 exam scenarios (4 randomly selected per exam). It tests your ability to design reliable extraction pipelines using:

- `tool_use` to extract structured fields from unstructured text
- `tool_choice` to force the extraction tool
- JSON Schema to define the extraction target
- Pydantic to validate extracted data
- Handling of edge cases (missing fields, ambiguous data)

**Real-world context**: Invoice processing, resume parsing, form extraction, log analysis, email triage.

---

## 2. The Extraction Pipeline Architecture

```
Input Document → Claude (with tool_choice) → tool_use Response → Pydantic Validation → Structured Output
                         ↑                                                    ↓
                   JSON Schema                                          Business Logic
                 (defines target)                                    (uses validated data)
```

### Complete Example: Invoice Extraction

```python
from pydantic import BaseModel, Field
from typing import List, Optional
import anthropic

class LineItem(BaseModel):
    description: str = Field(description="Item description")
    quantity: int = Field(ge=1, description="Quantity ordered")
    unit_price: float = Field(gt=0, description="Price per unit in USD")
    total: float = Field(gt=0, description="Line item total")

class InvoiceData(BaseModel):
    vendor_name: str = Field(description="Name of the vendor/supplier")
    invoice_number: str = Field(description="Invoice or bill number")
    date: str = Field(description="Invoice date in YYYY-MM-DD format")
    total_amount: float = Field(gt=0, description="Total invoice amount in USD")
    currency: str = Field(default="USD", description="Currency code")
    line_items: List[LineItem] = Field(description="Individual line items")
    payment_terms: Optional[str] = Field(default=None, description="Payment terms if specified")
    due_date: Optional[str] = Field(default=None, description="Due date if specified")

# Tool definition using generated schema
tools = [{
    "name": "extract_invoice",
    "description": "Extract structured invoice data from document text. Use this tool to parse invoices, bills, and payment documents into structured fields.",
    "input_schema": InvoiceData.model_json_schema()
}]

client = anthropic.Anthropic()

def extract_invoice(document_text: str) -> InvoiceData:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        tools=tools,
        tool_choice={"type": "tool", "name": "extract_invoice"},
        messages=[{
            "role": "user",
            "content": f"Extract all invoice data from the following document:\n\n{document_text}"
        }]
    )
    
    for block in response.content:
        if block.type == "tool_use":
            return InvoiceData.model_validate(block.input)
    
    raise ValueError("No tool_use block in response")
```

---

## 3. Forcing Extraction with tool_choice

**The critical pattern**: `tool_choice: {"type": "tool", "name": "extract_invoice"}` guarantees Claude will produce structured output rather than responding conversationally.

Without `tool_choice`, Claude might respond:
> "This appears to be an invoice from Acme Corp for $1,250. The invoice number is INV-2024-042..."

With `tool_choice`, Claude MUST call the extraction tool with data matching the schema.

### When to Use Each tool_choice Mode for Extraction

| Mode | Use Case |
|------|----------|
| `{"type": "tool", "name": "..."}` | Single extraction target — always use this |
| `{"type": "any"}` | Multiple possible extraction types (invoice OR receipt OR PO) |
| `{"type": "auto"}` | When extraction is optional (Claude decides if data is present) |

---

## 4. Schema Design for Extraction

### Principles for Extraction Schemas

1. **Use `Optional` for fields that may not exist in every document**
2. **Use `description` to tell Claude where to look** — these are instructions to Claude
3. **Use `enum` for fields with a known set of values**
4. **Keep nesting shallow** — one level of nested objects is fine, avoid deeper
5. **Include a `confidence` field** when extraction reliability matters

```python
class ExtractionWithConfidence(BaseModel):
    extracted_value: str = Field(description="The extracted value")
    confidence: float = Field(
        ge=0, le=1, 
        description="Confidence in the extraction: 1.0 = clearly stated, 0.5 = inferred, 0.0 = not found"
    )
    source_text: Optional[str] = Field(
        default=None,
        description="The exact text from which this value was extracted"
    )
```

---

## 5. Handling Partial and Ambiguous Extractions

### Strategy: Optional Fields with Defaults

```python
class FlexibleExtraction(BaseModel):
    # Required — must be present in every document
    document_type: str = Field(description="Type of document: invoice, receipt, PO, or unknown")
    
    # Optional — may or may not be present
    vendor_name: Optional[str] = Field(default=None, description="Vendor name if identifiable")
    amount: Optional[float] = Field(default=None, description="Total amount if stated")
    date: Optional[str] = Field(default=None, description="Document date if found")
    
    # With default for ambiguous cases
    currency: str = Field(default="USD", description="Currency code; default USD if not specified")
```

### Strategy: Enum for Classification with "Unknown"

```python
class DocumentCategory(str, Enum):
    INVOICE = "invoice"
    RECEIPT = "receipt"
    PURCHASE_ORDER = "purchase_order"
    UNKNOWN = "unknown"  # Always include an escape hatch
```

---

## 6. Validation and Error Recovery

```python
from pydantic import ValidationError

def extract_with_retry(document_text: str, max_retries: int = 2) -> Optional[InvoiceData]:
    for attempt in range(max_retries + 1):
        try:
            response = client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=2048,
                tools=tools,
                tool_choice={"type": "tool", "name": "extract_invoice"},
                messages=[{
                    "role": "user",
                    "content": f"Extract invoice data from:\n\n{document_text}"
                }]
            )
            
            for block in response.content:
                if block.type == "tool_use":
                    return InvoiceData.model_validate(block.input)
        
        except ValidationError as e:
            if attempt < max_retries:
                # Retry with error feedback
                continue
            else:
                # Log and return None after max retries
                print(f"Extraction failed after {max_retries} retries: {e}")
                return None
    
    return None
```

---

## 7. Practice Challenges

### Challenge 1 (from SPEC)

> You are building a document processing pipeline that extracts invoice data (vendor name, amount, date, line items) from PDF text. Claude sometimes returns the data as free text instead of using the extraction tool. Which approach most reliably ensures structured output?
>
> A) Add "Please always use the extract_invoice tool" to the system prompt
> B) Set `tool_choice: {type: "tool", name: "extract_invoice"}` and define a strict JSON Schema
> C) Provide 10 few-shot examples of correct tool usage
> D) Set `temperature: 0` to make Claude's behavior deterministic

**Answer**: B — Programmatic enforcement via `tool_choice` guarantees tool usage. Schema enforces structure.

### Challenge 2

> Your extraction pipeline processes customer feedback forms. Some include a "product_id" field, others don't. Your Pydantic model requires product_id. 30% of extractions fail validation. Best fix?
>
> A) Remove product_id entirely
> B) Make product_id `Optional[str]` with default `None`
> C) Add system prompt saying "always generate a product_id"
> D) Post-process to insert placeholder product_id

**Answer**: B — Optional fields correctly model real-world data variability. A loses data, C forces hallucination, D is fragile.

### Challenge 3

> You need to extract both contact information AND meeting details from emails. Some emails contain only contacts, some only meetings, some both. What's the best tool design?
>
> A) One tool with all fields optional
> B) Two separate tools (`extract_contact`, `extract_meeting`) with `tool_choice: "any"`
> C) Two tools with `tool_choice: {"type": "tool", "name": "..."}` — run the pipeline twice
> D) One tool with a discriminator field that changes the required fields

**Answer**: B — Two focused tools with `tool_choice: "any"` lets Claude choose the appropriate extraction(s) while still guaranteeing it uses a tool. Running twice (C) wastes tokens on emails with only one type.
