# Day 5 — JSON Schema Definition for Structured Output

**Domain**: Domain 4: Prompt Engineering & Structured Output (20% of exam)
**Sprint**: Sprint 1 — Foundations (Days 1–15)
**Academy Course**: (No specific course — supplementary study)
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [JSON Schema Fundamentals](#1-json-schema-fundamentals)
2. [Defining Tool Input Schemas](#2-defining-tool-input-schemas)
3. [Nested Object Schemas](#3-nested-object-schemas)
4. [Array Schemas](#4-array-schemas)
5. [Enum Constraints](#5-enum-constraints)
6. [Using description Fields to Guide Claude's Output](#6-using-description-fields-to-guide-claudes-output)
7. [Schema Design Patterns](#7-schema-design-patterns)
8. [Structured Outputs Feature (output_config.format)](#8-structured-outputs-feature-output_configformat)
9. [Strict Tool Use (strict: true)](#9-strict-tool-use-strict-true)
10. [How JSON Schema Enforces Output Structure](#10-how-json-schema-enforces-output-structure)
11. [Anti-Patterns](#11-anti-patterns)
12. [Quick Reference Card](#12-quick-reference-card)
13. [Scenario Challenge](#13-scenario-challenge)

---

## 1. JSON Schema Fundamentals

JSON Schema is a vocabulary for annotating and validating JSON documents. It provides a declarative way to describe the structure, types, and constraints of JSON data. In the context of the Claude API, JSON Schema is the mechanism that turns vague "please return JSON" instructions into programmatic enforcement of output structure.

**Why this matters for the exam**: JSON Schema is the bridge between your intent (what data you want) and Claude's output (what data you get). The exam tests whether you can design schemas that enforce structure without relying on prompt instructions — the core "programmatic enforcement over vague instruction" principle.

### Where JSON Schema Is Used in the Claude API

JSON Schema appears in **two** places in the Claude API:

| Location | Purpose | How It Works |
|----------|---------|-------------|
| `tools[].input_schema` | Defines the parameters a tool accepts | Claude's `tool_use` output must match this schema |
| `output_config.format` with `type: "json_schema"` | Defines the structure of Claude's direct JSON response | Claude's entire response is grammar-constrained to match the schema |

Both use standard JSON Schema syntax, but they serve different purposes. Tool input schemas define what Claude sends when calling a tool. Structured outputs define the shape of Claude's direct response.

### Core JSON Schema Keywords

These are the keywords you must know for the exam:

| Keyword | Purpose | Example |
|---------|---------|---------|
| `type` | Declares the data type | `"type": "string"` |
| `properties` | Defines fields of an object | `"properties": {"name": {"type": "string"}}` |
| `required` | Lists mandatory fields | `"required": ["name", "email"]` |
| `enum` | Restricts to a fixed set of values | `"enum": ["active", "inactive"]` |
| `description` | Documents the field (Claude reads this!) | `"description": "Customer's full name"` |
| `additionalProperties` | Controls whether extra fields are allowed | `"additionalProperties": false` |
| `items` | Defines the schema for array elements | `"items": {"type": "string"}` |

### The `type` Keyword

The `type` keyword declares what kind of value a field holds. JSON Schema supports seven primitive types:

| Type | JSON Value | Example |
|------|-----------|---------|
| `"string"` | Text | `"hello"` |
| `"number"` | Any numeric value (integer or float) | `3.14` |
| `"integer"` | Whole numbers only | `42` |
| `"boolean"` | True or false | `true` |
| `"array"` | Ordered list of values | `[1, 2, 3]` |
| `"object"` | Key-value pairs | `{"key": "value"}` |
| `"null"` | Null value | `null` |

```json
{
  "type": "object",
  "properties": {
    "name": {"type": "string"},
    "age": {"type": "integer"},
    "score": {"type": "number"},
    "is_active": {"type": "boolean"},
    "tags": {"type": "array"},
    "metadata": {"type": "object"}
  }
}
```

---

## 2. Defining Tool Input Schemas

Every tool in the `tools[]` array has an `input_schema` field that defines the parameters Claude must provide when calling that tool. The schema is a standard JSON Schema object with `type: "object"` at the top level.