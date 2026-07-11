# Day 22: MCP Tool Definitions

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 2: Tool Design & MCP Integration (18% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Continue Introduction to MCP |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [MCP Tool Definitions Overview](#1-mcp-tool-definitions-overview)
2. [Tool Definition Structure](#2-tool-definition-structure)
3. [Input Schema Design with JSON Schema](#3-input-schema-design-with-json-schema)
4. [Tool Descriptions: Best Practices](#4-tool-descriptions-best-practices)
5. [MCP Tools vs Claude API tools[]](#5-mcp-tools-vs-claude-api-tools)
6. [Tool Discovery and Listing](#6-tool-discovery-and-listing)
7. [Connecting MCP to tool_use/tool_result Flow](#7-connecting-mcp-to-tool_usetool_result-flow)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code MCP Documentation | https://docs.anthropic.com/en/docs/claude-code/mcp |
| MCP Specification | https://github.com/modelcontextprotocol/specification |

---

## 1. MCP Tool Definitions Overview

### What Are MCP Tool Definitions?

MCP tool definitions describe the **capabilities that an MCP server exposes** for an AI model to call. Each tool has a name, description, and input schema — the same conceptual model as Claude API `tools[]`, but registered at the protocol level rather than per-request.

### Core Concept

```
MCP Server registers tools at startup:
  ├── tool: "query_database"    → Runs SQL queries
  ├── tool: "create_ticket"     → Creates Jira tickets
  └── tool: "search_wiki"       → Searches internal docs

Claude Code discovers these tools via tools/list and can call them during conversation.
```

### The Three Parts of Every Tool

| Part | Purpose | Analogy |
|------|---------|---------|
| **Name** | Unique identifier for the tool | Function name |
| **Description** | Explains what the tool does (for the AI) | Docstring |
| **Input Schema** | Defines parameters and their types | Function signature |

### Where Tools Are Defined

Tools are defined **in the MCP server's code** (not in `.mcp.json`). The server registers them during initialization, and Claude Code discovers them via the `tools/list` protocol method.

```
.mcp.json         → "Start this server process"
Server code       → "I provide these tools: [...]"
Claude Code       → "I'll call these tools when appropriate"
```

---

## 2. Tool Definition Structure

### MCP Tool Schema

Each tool follows this structure when registered by the server:

```json
{
  "name": "query_database",
  "description": "Execute a read-only SQL query against the application database. Returns results as JSON rows. Use for data exploration and reporting queries only.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The SQL query to execute. Must be a SELECT statement."
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of rows to return. Defaults to 100.",
        "default": 100
      }
    },
    "required": ["query"]
  }
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique tool identifier (snake_case convention) |
| `description` | string | Yes | Human-readable explanation for the AI |
| `inputSchema` | object | Yes | JSON Schema defining the input parameters |

### Multiple Tool Definitions (Server Example in TypeScript)

```typescript
// tools/wiki-server.ts — MCP Server Implementation
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "wiki-tools",
  version: "1.0.0"
}, {
  capabilities: { tools: {} }
});

// Register tools
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "search_wiki",
        description: "Search the internal company wiki by keyword. Returns matching article titles and snippets.",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description: "Search keywords to find relevant wiki articles"
            },
            category: {
              type: "string",
              enum: ["engineering", "product", "design", "operations"],
              description: "Optional category filter to narrow search results"
            },
            max_results: {
              type: "integer",
              description: "Maximum number of results to return (1-50)",
              default: 10,
              minimum: 1,
              maximum: 50
            }
          },
          required: ["query"]
        }
      },
      {
        name: "get_article",
        description: "Retrieve the full content of a specific wiki article by its ID or slug.",
        inputSchema: {
          type: "object",
          properties: {
            article_id: {
              type: "string",
              description: "The unique ID or URL slug of the article to retrieve"
            }
          },
          required: ["article_id"]
        }
      },
      {
        name: "create_article",
        description: "Create a new wiki article. Requires title, content in Markdown format, and a category.",
        inputSchema: {
          type: "object",
          properties: {
            title: {
              type: "string",
              description: "Article title (max 200 characters)"
            },
            content: {
              type: "string",
              description: "Article body in Markdown format"
            },
            category: {
              type: "string",
              enum: ["engineering", "product", "design", "operations"],
              description: "Category for the new article"
            },
            tags: {
              type: "array",
              items: { "type": "string" },
              description: "Optional tags for discoverability"
            }
          },
          required: ["title", "content", "category"]
        }
      }
    ]
  };
});
```

---

## 3. Input Schema Design with JSON Schema

### JSON Schema Basics for MCP Tools

MCP tool input schemas use **standard JSON Schema** (same as Claude API tools). This ensures compatibility across both systems.

### Supported JSON Schema Types

| Type | Example | Use Case |
|------|---------|----------|
| `string` | `"type": "string"` | Names, queries, IDs |
| `integer` | `"type": "integer"` | Counts, limits, pages |
| `number` | `"type": "number"` | Prices, coordinates |
| `boolean` | `"type": "boolean"` | Flags, toggles |
| `array` | `"type": "array"` | Lists, collections |
| `object` | `"type": "object"` | Nested structures |

### Schema Features and Constraints

```json
{
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The resource name",
      "minLength": 1,
      "maxLength": 100
    },
    "priority": {
      "type": "string",
      "enum": ["low", "medium", "high", "critical"],
      "description": "Task priority level"
    },
    "count": {
      "type": "integer",
      "minimum": 1,
      "maximum": 1000,
      "default": 10,
      "description": "Number of items to process"
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "maxItems": 10,
      "description": "Categorization tags"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "author": { "type": "string" },
        "version": { "type": "integer" }
      },
      "required": ["author"]
    },
    "include_archived": {
      "type": "boolean",
      "default": false,
      "description": "Whether to include archived items in results"
    }
  },
  "required": ["name", "priority"]
}
```

### Key Schema Design Patterns

| Pattern | Schema Fragment | Purpose |
|---------|----------------|---------|
| **Enum constraint** | `"enum": ["a", "b", "c"]` | Restrict to valid values |
| **Default value** | `"default": 10` | Reduce required params |
| **Range validation** | `"minimum": 1, "maximum": 100` | Prevent invalid inputs |
| **Required fields** | `"required": ["field1"]` | Ensure essential params |
| **Nested objects** | `"type": "object", "properties": {...}` | Structured input |
| **Array with types** | `"items": {"type": "string"}` | Typed collections |

### Good vs Bad Schema Design

```json
// ❌ BAD: No descriptions, no constraints, overly broad
{
  "type": "object",
  "properties": {
    "q": { "type": "string" },
    "n": { "type": "integer" }
  }
}

// ✅ GOOD: Descriptive, constrained, clear purpose
{
  "type": "object",
  "properties": {
    "search_query": {
      "type": "string",
      "description": "Keywords to search for in the knowledge base. Use natural language.",
      "minLength": 2,
      "maxLength": 500
    },
    "max_results": {
      "type": "integer",
      "description": "Maximum number of search results to return.",
      "minimum": 1,
      "maximum": 50,
      "default": 10
    }
  },
  "required": ["search_query"]
}
```

---

## 4. Tool Descriptions: Best Practices

### Why Descriptions Matter

The tool description is the **primary signal** the AI model uses to decide:
1. **Whether** to use this tool (vs another tool or direct answer)
2. **When** to use it (which user intents map to this tool)
3. **How** to construct the input (what values to pass)

### Description Quality Spectrum

```
❌ Bad:   "Database tool"
⚠️  Weak:  "Query the database"
✅ Good:  "Execute a read-only SQL query against the PostgreSQL database.
           Returns results as JSON rows. Use for data exploration,
           reporting, and answering questions about stored data.
           Only SELECT statements are permitted."
```

### Best Practice Guidelines

| Guideline | Example | Rationale |
|-----------|---------|-----------|
| **State what it does** | "Execute a SQL query..." | Core purpose |
| **State constraints** | "Only SELECT statements" | Prevents misuse |
| **State return format** | "Returns JSON rows" | Sets expectations |
| **State when to use** | "Use for data exploration" | Guides selection |
| **State when NOT to use** | "Do not use for mutations" | Prevents errors |

### Description Template

```
[WHAT]: What the tool does in one sentence.
[WHEN]: When to use this tool (triggering scenarios).
[CONSTRAINTS]: Limitations or restrictions on usage.
[RETURNS]: What the output looks like.
[NOT FOR]: What this tool should NOT be used for (if ambiguous).
```

### Examples of Well-Written Descriptions

```json
{
  "name": "create_pull_request",
  "description": "Create a new pull request on GitHub. Use when the user has completed changes and wants to submit them for review. Requires a branch that has been pushed to the remote. Returns the PR URL and number. Do not use for draft PRs — use create_draft_pr instead."
}
```

```json
{
  "name": "run_test_suite",
  "description": "Execute the project's test suite and return results. Runs all tests matching the provided pattern, or all tests if no pattern is specified. Returns pass/fail counts and error details for failed tests. Use after code changes to verify correctness. Note: runs may take 30-60 seconds for the full suite."
}
```

```json
{
  "name": "search_codebase",
  "description": "Search the codebase using regex patterns. Returns matching file paths with line numbers and surrounding context (3 lines). Use for finding function definitions, usage patterns, or specific strings. Searches all tracked files by default — use the 'include_pattern' param to narrow scope. Limit: 100 results max."
}
```

### Property Descriptions

Property-level descriptions are equally important:

```json
{
  "properties": {
    "sql_query": {
      "type": "string",
      "description": "A valid PostgreSQL SELECT statement. Do not include semicolons. Subqueries and CTEs are supported. Example: SELECT name, email FROM users WHERE created_at > '2024-01-01'"
    }
  }
}
```

---

## 5. MCP Tools vs Claude API tools[]

### Fundamental Difference

| Aspect | Claude API `tools[]` | MCP Tool Definitions |
|--------|---------------------|---------------------|
| **Where defined** | In each API request body | In MCP server code |
| **Registration** | Per-request (ephemeral) | Per-server-session (persistent) |
| **Execution** | Client app implements handler | MCP server implements handler |
| **Discovery** | Client knows tools upfront | Client discovers via `tools/list` |
| **Transport** | HTTP (API calls) | stdio or HTTP+SSE (protocol) |
| **Reusability** | Bound to one application | Shared across MCP-compatible clients |
| **Schema format** | JSON Schema (inputSchema) | JSON Schema (inputSchema) — SAME |

### Side-by-Side: Same Tool, Different Systems

**Claude API style (Sprint 1):**
```python
# Defined in application code, sent with each API request
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=[{
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }],
    messages=[...]
)
# YOUR CODE handles the tool_use response and executes the function
```

**MCP style (Sprint 2):**
```typescript
// Defined in MCP server code, registered once at startup
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "get_weather",
    description: "Get current weather for a location",
    inputSchema: {
      type: "object",
      properties: {
        location: { type: "string", description: "City name" },
        units: { type: "string", enum: ["celsius", "fahrenheit"] }
      },
      required: ["location"]
    }
  }]
}));

// MCP SERVER handles execution when called
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "get_weather") {
    const result = await fetchWeather(request.params.arguments);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
});
```

### Key Insight: Same Schema, Different Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHARED: JSON Schema                            │
│   Both use identical inputSchema format for parameters           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   API tools[]:                                                    │
│     Request ──▶ Claude ──▶ tool_use ──▶ YOUR APP ──▶ tool_result │
│     (defined per request)   (you handle execution)               │
│                                                                   │
│   MCP tools:                                                      │
│     tools/list ──▶ Claude ──▶ tools/call ──▶ MCP SERVER ──▶ result│
│     (defined once)           (server handles execution)          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Which

| Scenario | Use API tools[] | Use MCP Tools |
|----------|-----------------|---------------|
| Building a custom chat app | ✅ | Overkill |
| Extending Claude Code capabilities | ❌ | ✅ |
| Tool needed by one application | ✅ | Unnecessary |
| Tool shared across multiple AI clients | Use MCP | ✅ |
| Wrapping an external service for AI access | Either works | ✅ (better reuse) |
| Rapid prototyping | ✅ (simpler) | More setup |
| Team/org standardization | Possible but manual | ✅ (protocol-standard) |

---

## 6. Tool Discovery and Listing

### The tools/list Protocol Method

When Claude Code connects to an MCP server, it calls `tools/list` to discover available tools:

```
Client → Server: { "method": "tools/list" }
Server → Client: { "tools": [ ...tool definitions... ] }
```

### Discovery Flow in Detail

```
┌──────────────────────────────────────────────────────────────┐
│                    TOOL DISCOVERY FLOW                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Claude Code spawns MCP server (from .mcp.json config)    │
│  2. Client sends: initialize (negotiate capabilities)         │
│  3. Server responds: { capabilities: { tools: {} } }         │
│  4. Client sends: tools/list                                  │
│  5. Server responds with array of tool definitions            │
│  6. Claude Code registers tools in its available tools set    │
│  7. Claude can now call these tools during conversation       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### What Claude Sees After Discovery

After `tools/list`, Claude Code's internal state looks like:

```
Available tools:
├── Built-in tools (always available):
│   ├── Read file
│   ├── Write file
│   ├── Execute command
│   └── Web search
│
└── MCP tools (from connected servers):
    ├── [postgres] query_database
    ├── [postgres] list_tables
    ├── [github] create_pull_request
    ├── [github] list_issues
    ├── [wiki] search_wiki
    └── [wiki] get_article
```

### Dynamic Tool Registration

MCP supports **notifications** for tool changes during a session:

```
Server → Client: { "method": "notifications/tools/list_changed" }
```

This tells the client to re-fetch `tools/list` because available tools have changed (e.g., a new tool was deployed to the server).

---

## 7. Connecting MCP to tool_use/tool_result Flow

### The Unified Mental Model

Whether using API tools or MCP tools, Claude follows the **same cognitive pattern**:

```
1. Claude determines a tool is needed
2. Claude produces a tool call (name + arguments)
3. Something executes the tool
4. Result is returned to Claude
5. Claude incorporates result into its response
```

### How MCP Routes Tool Calls

```
User: "How many orders were placed last month?"

Claude thinks: "I need to query the database"

Claude produces (internal):
  tool_use: {
    name: "query_database",
    input: {
      query: "SELECT COUNT(*) FROM orders WHERE created_at >= '2024-01-01'",
      limit: 1
    }
  }

MCP Client routes this to the "postgres" server:
  tools/call: {
    name: "query_database",
    arguments: { query: "...", limit: 1 }
  }

MCP Server executes and returns:
  { content: [{ type: "text", text: "{\"count\": 1847}" }] }

Claude receives the result and responds:
  "There were 1,847 orders placed last month."
```

### Protocol Message Mapping

| Claude API Concept | MCP Protocol Equivalent | Notes |
|---|---|---|
| `tool_use` content block | `tools/call` request | Same semantics, different transport |
| `tool_result` message | `tools/call` response | Result flows back to model |
| `tools[]` in request | `tools/list` response | Registration vs per-request |
| `tool_choice: "auto"` | N/A (always auto in MCP) | Claude decides autonomously |
| `tool_choice: { name }` | N/A (not supported in MCP) | MCP doesn't force specific tools |

### Code Comparison: Handling Tool Execution

**API approach (your code handles execution):**
```python
# Sprint 1 pattern — YOU execute the tool
response = client.messages.create(tools=[...], messages=[...])

for block in response.content:
    if block.type == "tool_use":
        # YOUR CODE decides how to handle this
        result = execute_tool(block.name, block.input)
        # YOU send tool_result back
        messages.append({"role": "user", "content": [{
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": result
        }]})
```

**MCP approach (server handles execution):**
```typescript
// Sprint 2 pattern — MCP SERVER executes the tool
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  
  switch (name) {
    case "query_database":
      const rows = await db.query(args.query, args.limit);
      return { content: [{ type: "text", text: JSON.stringify(rows) }] };
    
    case "list_tables":
      const tables = await db.listTables();
      return { content: [{ type: "text", text: JSON.stringify(tables) }] };
    
    default:
      throw new Error(`Unknown tool: ${name}`);
  }
});
```

### The Complete Flow (End-to-End)

```
┌────────┐     ┌───────────┐     ┌───────────┐     ┌──────────┐
│  User  │────▶│Claude Code│────▶│Claude Model│────▶│tool_use  │
│        │     │  (Host)   │     │  (AI)     │     │(decision)│
└────────┘     └───────────┘     └───────────┘     └────┬─────┘
                                                         │
                    ┌────────────────────────────────────╯
                    ▼
              ┌───────────┐     ┌──────────┐     ┌──────────────┐
              │MCP Client │────▶│tools/call │────▶│  MCP Server  │
              │(protocol) │     │ (request) │     │  (executes)  │
              └───────────┘     └──────────┘     └──────┬───────┘
                                                         │
                    ┌────────────────────────────────────╯
                    ▼
              ┌───────────┐     ┌───────────┐     ┌────────┐
              │  Result   │────▶│Claude Model│────▶│Response│────▶ User
              │(tool_result)    │(incorporates)    │(final) │
              └───────────┘     └───────────┘     └────────┘
```

---

## 8. Anti-Patterns

### ❌ Anti-Pattern 1: Vague Tool Descriptions

```json
{
  "name": "do_thing",
  "description": "Does a thing with the data",
  "inputSchema": { "type": "object", "properties": { "data": { "type": "string" } } }
}
```

**Problem:** Claude can't determine when to use this tool or how to construct input.

**Fix:** Specific, actionable description with when-to-use guidance.

### ❌ Anti-Pattern 2: Missing Property Descriptions

```json
{
  "inputSchema": {
    "type": "object",
    "properties": {
      "q": { "type": "string" },
      "n": { "type": "integer" },
      "f": { "type": "boolean" }
    }
  }
}
```

**Problem:** Single-letter property names with no descriptions. Claude must guess what they mean.

**Fix:**
```json
{
  "properties": {
    "search_query": { "type": "string", "description": "Keywords to search for" },
    "max_results": { "type": "integer", "description": "Maximum results to return" },
    "include_archived": { "type": "boolean", "description": "Whether to include archived items" }
  }
}
```

### ❌ Anti-Pattern 3: Overly Permissive Schemas

```json
{
  "name": "execute_sql",
  "description": "Execute any SQL statement",
  "inputSchema": {
    "type": "object",
    "properties": {
      "sql": { "type": "string" }
    },
    "required": ["sql"]
  }
}
```

**Problem:** No guardrails — Claude could generate DROP TABLE or DELETE statements.

**Fix:** Constrain in description and validate in server:
```json
{
  "name": "query_database",
  "description": "Execute a read-only SELECT query. INSERT, UPDATE, DELETE, and DDL statements are not permitted and will be rejected.",
  "inputSchema": { ... }
}
```

### ❌ Anti-Pattern 4: Too Many Tools (Cognitive Overload)

```
Server registers 50+ tools → Claude's context filled with tool definitions
→ Slower responses, higher token cost, more confusion about which tool to use
```

**Fix:** Keep tool count per server reasonable (5-15). Split into multiple focused servers if needed.

### ❌ Anti-Pattern 5: Duplicate Tools Across Servers

```json
// Server A
{ "name": "search", "description": "Search the codebase" }

// Server B  
{ "name": "search", "description": "Search the wiki" }
```

**Problem:** Name collision causes ambiguity — Claude may call the wrong one.

**Fix:** Use prefixed or descriptive names: `search_codebase`, `search_wiki`.

### ❌ Anti-Pattern 6: Not Using required Field

```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string" },
    "database": { "type": "string" },
    "timeout": { "type": "integer" }
  }
}
```

**Problem:** Claude may omit essential parameters since nothing is marked required.

**Fix:** Always specify `"required": ["query", "database"]` for essential parameters.

---

## 9. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│              MCP TOOL DEFINITIONS QUICK REFERENCE                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  TOOL STRUCTURE:                                                   │
│    name:        Unique identifier (snake_case)                    │
│    description: What, when, constraints, returns (for the AI)    │
│    inputSchema: JSON Schema defining parameters                   │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  INPUT SCHEMA TYPES:                                               │
│                                                                    │
│    string   → text, IDs, queries                                 │
│    integer  → counts, limits, pagination                         │
│    number   → decimals, coordinates                              │
│    boolean  → flags, toggles                                     │
│    array    → lists, collections                                 │
│    object   → nested structures                                  │
│                                                                    │
│  CONSTRAINTS: enum, minimum, maximum, minLength, maxLength,      │
│               minItems, maxItems, default, required               │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  MCP vs API tools[] COMPARISON:                                    │
│                                                                    │
│  SAME:    JSON Schema format, name/description/inputSchema       │
│  DIFFER:  Registration (protocol vs per-request)                 │
│  DIFFER:  Execution (server vs client app)                       │
│  DIFFER:  Discovery (tools/list vs pre-known)                    │
│  DIFFER:  Reusability (cross-client vs single-app)               │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  DESCRIPTION BEST PRACTICES:                                       │
│                                                                    │
│  ✅ State WHAT it does (one sentence)                            │
│  ✅ State WHEN to use it (triggering scenarios)                  │
│  ✅ State CONSTRAINTS (what's not allowed)                       │
│  ✅ State RETURN format (what comes back)                        │
│  ✅ State NOT-FOR cases (disambiguation)                         │
│                                                                    │
│  ❌ Single word descriptions ("Database")                        │
│  ❌ Missing property descriptions                                │
│  ❌ No required fields specified                                 │
│  ❌ Overly permissive (no guardrails)                            │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  PROTOCOL FLOW:                                                    │
│                                                                    │
│  initialize → tools/list → [conversation] → tools/call → result │
│                                                                    │
│  Maps to Sprint 1:                                                │
│    tools/list     ≈  tools[] in API request                      │
│    tools/call     ≈  tool_use content block                      │
│    call response  ≈  tool_result message                         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • inputSchema uses STANDARD JSON Schema (same as API tools)     │
│  • Tools are defined IN THE SERVER CODE (not in .mcp.json)       │
│  • Discovery happens via tools/list protocol method              │
│  • Good descriptions are CRITICAL for tool selection             │
│  • MCP tools are reusable across any MCP-compatible client       │
│  • API tools = per-request, you execute; MCP = per-server, it    │
│    executes                                                       │
│  • Keep tool count reasonable (5-15 per server)                  │
│  • Use unique, descriptive names to avoid collisions             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Scenario

You are building an MCP server for a project management tool. The server should provide three capabilities:

1. **Search tickets** — Find tickets by keyword, status, or assignee
2. **Create ticket** — Create a new ticket with title, description, priority, and assignee
3. **Update ticket status** — Change a ticket's status (open → in_progress → review → done)

### Question

**Which tool definition set BEST follows MCP best practices for descriptions and schemas?**

**A)**
```json
[
  {
    "name": "tickets",
    "description": "Manage tickets",
    "inputSchema": {
      "type": "object",
      "properties": {
        "action": { "type": "string", "enum": ["search", "create", "update"] },
        "data": { "type": "object" }
      }
    }
  }
]
```

**B)**
```json
[
  {
    "name": "search_tickets",
    "description": "Search project tickets by keyword, status, or assignee. Returns matching tickets with ID, title, status, and assignee. Use when the user asks about existing tickets or wants to find specific work items.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "query": { "type": "string", "description": "Keywords to search in ticket title and description" },
        "status": { "type": "string", "enum": ["open", "in_progress", "review", "done"], "description": "Filter by ticket status" },
        "assignee": { "type": "string", "description": "Filter by assignee username" },
        "max_results": { "type": "integer", "minimum": 1, "maximum": 50, "default": 20, "description": "Maximum number of tickets to return" }
      },
      "required": []
    }
  },
  {
    "name": "create_ticket",
    "description": "Create a new project ticket. Requires a title and assigns default priority of 'medium' if not specified. Returns the created ticket ID and URL. Use when the user wants to track new work items or bugs.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "title": { "type": "string", "description": "Ticket title (max 200 chars)", "maxLength": 200 },
        "description": { "type": "string", "description": "Detailed description in Markdown" },
        "priority": { "type": "string", "enum": ["low", "medium", "high", "critical"], "default": "medium", "description": "Ticket priority level" },
        "assignee": { "type": "string", "description": "Username to assign the ticket to" }
      },
      "required": ["title"]
    }
  },
  {
    "name": "update_ticket_status",
    "description": "Update the status of an existing ticket. Status transitions must follow the workflow: open → in_progress → review → done. Backward transitions are allowed. Returns the updated ticket. Use when a user reports progress on work items.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "ticket_id": { "type": "string", "description": "The unique ticket ID (e.g., 'PROJ-123')" },
        "new_status": { "type": "string", "enum": ["open", "in_progress", "review", "done"], "description": "The new status to set" }
      },
      "required": ["ticket_id", "new_status"]
    }
  }
]
```

**C)**
```json
[
  {
    "name": "search",
    "description": "Search tickets",
    "inputSchema": { "type": "object", "properties": { "q": { "type": "string" } } }
  },
  {
    "name": "create",
    "description": "Create ticket",
    "inputSchema": { "type": "object", "properties": { "title": { "type": "string" } } }
  },
  {
    "name": "update",
    "description": "Update ticket",
    "inputSchema": { "type": "object", "properties": { "id": { "type": "string" }, "status": { "type": "string" } } }
  }
]
```

**D)**
```json
[
  {
    "name": "search_tickets",
    "description": "Search project tickets by keyword, status, or assignee.",
    "inputSchema": { "type": "object", "properties": { "query": { "type": "string" } } }
  },
  {
    "name": "create_ticket",
    "description": "Create a new project ticket with title and optional fields.",
    "inputSchema": { "type": "object", "properties": { "title": { "type": "string" }, "description": { "type": "string" }, "priority": { "type": "string" }, "assignee": { "type": "string" } } }
  },
  {
    "name": "update_ticket_status",
    "description": "Update an existing ticket's status.",
    "inputSchema": { "type": "object", "properties": { "ticket_id": { "type": "string" }, "new_status": { "type": "string" } } }
  }
]
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** follows all MCP best practices:
  - Descriptive tool names (snake_case, verb_noun)
  - Rich descriptions with WHAT, WHEN, and RETURNS guidance
  - Full JSON Schema with constraints (enum, minimum, maximum, maxLength)
  - Property-level descriptions for every parameter
  - Appropriate `required` fields
  - Default values to reduce required input
  - Status workflow documented in description

- **Option A** uses a single "god tool" pattern with an action parameter — this makes it harder for Claude to select the right operation and provides no schema validation per action.

- **Option C** uses generic names ("search", "create", "update") that could collide with other servers, has cryptic property names ("q"), and lacks descriptions and constraints.

- **Option D** is decent but incomplete — missing enum constraints on status/priority, no property descriptions, no required fields, no min/max constraints. It's functional but not best-practice.

**Key Exam Principle:** MCP tool definitions should be **maximally informative for the AI model** — rich descriptions, typed schemas with constraints, clear required fields, and explicit when-to-use guidance. The better the definition, the more accurately Claude selects and calls the tool.

---

## Summary: Key Exam Takeaways

1. MCP tools have three parts: **name, description, inputSchema** — same concept as API tools[]
2. Tools are defined **in server code** and discovered via `tools/list` — not in .mcp.json
3. inputSchema uses **standard JSON Schema** — identical format to Claude API input_schema
4. Descriptions should include: WHAT, WHEN, CONSTRAINTS, RETURNS, NOT-FOR
5. API tools: per-request, client-executes; MCP tools: per-server, server-executes
6. Both systems share the same **tool_use → execute → tool_result** cognitive pattern
7. Good property descriptions and constraints are essential for accurate tool use
8. Keep tool count per server reasonable (5-15) — too many causes cognitive overload
9. Use unique, descriptive names — avoid generic names that could collide across servers

---

*End of Day 22 Study Material*
