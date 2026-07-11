# Day 23: MCP Resource Definitions

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 2: Tool Design & MCP Integration (18% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Complete Introduction to MCP |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [MCP Resources Overview](#1-mcp-resources-overview)
2. [Resource URI Schemes and Content Types](#2-resource-uri-schemes-and-content-types)
3. [Resources vs Tools: When to Use Which](#3-resources-vs-tools-when-to-use-which)
4. [Resource Templates (Parameterized URIs)](#4-resource-templates-parameterized-uris)
5. [Resource Listing and Reading](#5-resource-listing-and-reading)
6. [MCP Prompts: Server-Defined Templates](#6-mcp-prompts-server-defined-templates)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| MCP Specification — Resources | https://spec.modelcontextprotocol.io/specification/server/resources/ |
| MCP Specification — Prompts | https://spec.modelcontextprotocol.io/specification/server/prompts/ |
| Anthropic MCP Documentation | https://docs.anthropic.com/en/docs/claude-code/mcp |

---

## 1. MCP Resources Overview

### What Are MCP Resources?

MCP resources represent **data that can be read by the client** — they expose information without performing actions. Think of them as "read-only endpoints" that provide context to the AI model.

### Core Concept: Resources vs Tools

```
MCP Primitives:
  ├── Tools      → Actions (execute something, produce side effects)
  ├── Resources  → Data (read-only, provide context)
  └── Prompts    → Templates (pre-defined prompt structures)
```

### Resource Structure

Every resource has these key fields:

```json
{
  "uri": "file:///project/src/schema.sql",
  "name": "Database Schema",
  "description": "The current database schema definition",
  "mimeType": "text/plain"
}
```

| Field | Purpose | Example |
|-------|---------|---------|
| `uri` | Unique identifier for the resource | `file:///config.json` |
| `name` | Human-readable display name | "Project Configuration" |
| `description` | What this resource contains (for AI) | "Current app config settings" |
| `mimeType` | Content format indicator | `application/json`, `text/plain` |

### Why Resources Exist

Resources solve the problem of **providing context without requiring a tool call**. Instead of building a "read_config" tool that the AI must decide to call, you expose the config as a resource that the client can proactively load.

```
Without Resources (tool-only approach):
  User asks about config → AI decides to call read_config tool → Gets data → Answers

With Resources:
  Client loads config resource into context → AI already has the data → Answers directly
```

---

## 2. Resource URI Schemes and Content Types

### URI Schemes

MCP resources use URIs to identify content. The URI scheme indicates the source type:

| URI Scheme | Example | Use Case |
|------------|---------|----------|
| `file://` | `file:///project/README.md` | Local filesystem files |
| `http://` / `https://` | `https://api.example.com/schema` | Remote web resources |
| Custom schemes | `postgres://tables/users` | Application-specific resources |
| Custom schemes | `git://main/src/index.ts` | Version-controlled content |
| Custom schemes | `jira://PROJECT-123` | Integration-specific content |

### Custom URI Schemes

Servers can define **any URI scheme** meaningful to their domain:

```typescript
// A database server might use:
"postgres://tables/users"          // Table schema
"postgres://views/active_orders"   // View definition
"postgres://functions/calculate"   // Function signature

// A documentation server might use:
"docs://api/authentication"        // API docs section
"docs://guides/getting-started"    // Guide content
```

### Content Types (mimeType)

The `mimeType` field tells the client how to interpret the resource content:

| mimeType | Content | Common Use |
|----------|---------|------------|
| `text/plain` | Plain text | Logs, configs, documentation |
| `application/json` | JSON data | API schemas, structured configs |
| `text/markdown` | Markdown | Documentation, READMEs |
| `text/html` | HTML content | Web pages, rendered docs |
| `application/sql` | SQL statements | Database schemas, migrations |
| `image/png` | Binary image | Diagrams, screenshots |

### Resource Content Structure

When a client reads a resource, the server returns content blocks:

```json
{
  "contents": [
    {
      "uri": "file:///project/schema.sql",
      "mimeType": "text/plain",
      "text": "CREATE TABLE users (\n  id SERIAL PRIMARY KEY,\n  email VARCHAR(255) NOT NULL\n);"
    }
  ]
}
```

For binary content (images), use `blob` instead of `text`:

```json
{
  "contents": [
    {
      "uri": "diagram://architecture",
      "mimeType": "image/png",
      "blob": "base64-encoded-content-here"
    }
  ]
}
```

---

## 3. Resources vs Tools: When to Use Which

### Decision Framework

| Factor | Use a Resource | Use a Tool |
|--------|---------------|------------|
| **Side effects?** | No side effects (read-only) | Has side effects (writes, mutations) |
| **Static vs Dynamic?** | Relatively stable data | Dynamic computation needed |
| **Context loading?** | Client can load proactively | AI decides when to invoke |
| **User intent?** | Background context | Explicit action request |
| **Caching?** | Cacheable (changes infrequently) | Results vary per invocation |

### Examples: Resource vs Tool

```
✅ RESOURCE: Database schema (stable context, read-only, useful background)
❌ TOOL:     Execute a SQL query (dynamic, has side effects, needs user intent)

✅ RESOURCE: API documentation (static reference material)
❌ TOOL:     Call an API endpoint (dynamic, produces results)

✅ RESOURCE: Project configuration file (background context)
❌ TOOL:     Update a configuration value (mutation, side effect)

✅ RESOURCE: Git log / recent commits (read-only context)
❌ TOOL:     Create a git commit (action, side effect)
```

### Key Insight for the Exam

```
Resources = "Here is some context the AI might need"
Tools     = "Here is something the AI can DO"

Resources are PULLED by the client (proactive context loading)
Tools are CALLED by the AI (reactive action execution)
```

---

## 4. Resource Templates (Parameterized URIs)

### What Are Resource Templates?

Resource templates allow servers to expose **parameterized resources** — resources whose content depends on a variable in the URI. They use URI Template syntax (RFC 6570).

### Template Syntax

```json
{
  "uriTemplate": "postgres://tables/{table_name}/schema",
  "name": "Table Schema",
  "description": "Get the schema for a specific database table",
  "mimeType": "application/json"
}
```

### Template vs Static Resource

| Aspect | Static Resource | Resource Template |
|--------|----------------|-------------------|
| URI | `postgres://tables/users` | `postgres://tables/{table_name}/schema` |
| Content | Fixed, one resource | Variable, generates many resources |
| Discovery | Listed in `resources/list` | Listed in `resources/templates/list` |
| Reading | Direct read by URI | Client resolves template, then reads |

### Template Examples

```typescript
// Server exposes templates for dynamic content
const templates = [
  {
    uriTemplate: "git://branches/{branch}/file/{path}",
    name: "Git File Content",
    description: "Read a file from a specific git branch",
    mimeType: "text/plain"
  },
  {
    uriTemplate: "jira://projects/{project_key}/issues/{issue_id}",
    name: "Jira Issue",
    description: "Get details of a specific Jira issue",
    mimeType: "application/json"
  },
  {
    uriTemplate: "docs://api/{endpoint}/examples",
    name: "API Endpoint Examples",
    description: "Get usage examples for a specific API endpoint",
    mimeType: "text/markdown"
  }
];
```

### How Templates Work in Practice

```
1. Client discovers templates via resources/templates/list
2. Client (or AI) determines which template to use
3. Client resolves the template by filling in parameters
4. Client reads the resolved URI via resources/read

Example:
  Template:   "postgres://tables/{table_name}/schema"
  Resolved:   "postgres://tables/users/schema"
  Read result: { columns: [...], indexes: [...] }
```

---

## 5. Resource Listing and Reading

### Protocol Methods for Resources

| Method | Purpose | Returns |
|--------|---------|---------|
| `resources/list` | List all available static resources | Array of resource descriptors |
| `resources/read` | Read content of a specific resource | Content blocks (text or blob) |
| `resources/templates/list` | List available resource templates | Array of template descriptors |
| `resources/subscribe` | Subscribe to resource change notifications | Confirmation |

### Listing Resources (Server Implementation)

```typescript
server.setRequestHandler("resources/list", async () => {
  return {
    resources: [
      {
        uri: "file:///project/schema.sql",
        name: "Database Schema",
        description: "Current database schema with all tables and indexes",
        mimeType: "text/plain"
      },
      {
        uri: "config://app/settings",
        name: "Application Settings",
        description: "Current application configuration values",
        mimeType: "application/json"
      },
      {
        uri: "docs://api/overview",
        name: "API Overview",
        description: "High-level API documentation and endpoints list",
        mimeType: "text/markdown"
      }
    ]
  };
});
```

### Reading Resources (Server Implementation)

```typescript
server.setRequestHandler("resources/read", async (request) => {
  const { uri } = request.params;
  
  switch (uri) {
    case "file:///project/schema.sql":
      const schema = await fs.readFile("/project/schema.sql", "utf-8");
      return {
        contents: [{ uri, mimeType: "text/plain", text: schema }]
      };
    
    case "config://app/settings":
      const config = await loadAppConfig();
      return {
        contents: [{ uri, mimeType: "application/json", text: JSON.stringify(config, null, 2) }]
      };
    
    default:
      throw new Error(`Resource not found: ${uri}`);
  }
});
```

### Resource Change Notifications

Resources can notify clients when their content changes:

```typescript
// Server notifies client that a resource has been updated
server.sendNotification("notifications/resources/updated", {
  uri: "config://app/settings"
});
```

---

## 6. MCP Prompts: Server-Defined Templates

### What Are MCP Prompts?

MCP prompts are **server-defined prompt templates** — reusable, parameterized prompts that servers expose for common workflows. They provide structured starting points for conversations.

### Prompt Structure

```json
{
  "name": "code_review",
  "description": "Review code for bugs, security issues, and style violations",
  "arguments": [
    {
      "name": "code",
      "description": "The code to review",
      "required": true
    },
    {
      "name": "language",
      "description": "Programming language of the code",
      "required": true
    },
    {
      "name": "focus_areas",
      "description": "Specific areas to focus on (security, performance, style)",
      "required": false
    }
  ]
}
```

### Prompts vs Tools vs Resources

| Primitive | Initiated By | Purpose | Returns |
|-----------|-------------|---------|---------|
| **Tools** | AI model (tool_use) | Execute actions | Action results |
| **Resources** | Client (proactive load) | Provide context | Data content |
| **Prompts** | User (explicit selection) | Structure conversations | Message templates |

### Prompt Discovery and Usage

```typescript
// Server registers prompts
server.setRequestHandler("prompts/list", async () => {
  return {
    prompts: [
      {
        name: "explain_code",
        description: "Explain what a piece of code does in plain language",
        arguments: [
          { name: "code", description: "The code to explain", required: true },
          { name: "audience", description: "Target audience (beginner, intermediate, expert)", required: false }
        ]
      },
      {
        name: "generate_tests",
        description: "Generate unit tests for a function or class",
        arguments: [
          { name: "code", description: "The code to test", required: true },
          { name: "framework", description: "Test framework (jest, pytest, mocha)", required: true }
        ]
      }
    ]
  };
});

// Server resolves a prompt into messages
server.setRequestHandler("prompts/get", async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "explain_code") {
    return {
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text: `Please explain the following ${args.audience || "intermediate"}-level code:\n\n${args.code}`
          }
        }
      ]
    };
  }
});
```

### Use Cases for MCP Prompts

| Use Case | Prompt Name | Arguments |
|----------|-------------|-----------|
| Code review | `code_review` | code, language, focus_areas |
| Bug analysis | `analyze_bug` | error_message, stack_trace, context |
| Documentation | `generate_docs` | code, format, audience |
| Refactoring | `suggest_refactor` | code, goals, constraints |
| SQL generation | `write_query` | description, tables, constraints |

---

## 7. Anti-Patterns

### ❌ Anti-Pattern 1: Using Tools for Read-Only Data

```json
{
  "name": "get_database_schema",
  "description": "Get the database schema",
  "inputSchema": { "type": "object", "properties": {} }
}
```

**Problem:** The database schema is stable read-only data. Exposing it as a tool means the AI must decide to call it, adding latency and using a tool invocation unnecessarily.

**Fix:** Expose as a resource that the client can proactively load into context:
```json
{
  "uri": "postgres://schema/all",
  "name": "Database Schema",
  "description": "Complete database schema with all tables, columns, and indexes",
  "mimeType": "text/plain"
}
```

### ❌ Anti-Pattern 2: Resources Without Descriptions

```json
{
  "uri": "file:///data.json",
  "name": "data",
  "mimeType": "application/json"
}
```

**Problem:** No description. The AI (and user) can't determine what this resource contains or when it's useful.

**Fix:** Always include a meaningful description:
```json
{
  "uri": "file:///data.json",
  "name": "User Analytics Data",
  "description": "Aggregated user behavior metrics for the current month including page views, session duration, and conversion rates",
  "mimeType": "application/json"
}
```

### ❌ Anti-Pattern 3: Overly Broad Resource Templates

```json
{
  "uriTemplate": "file:///{path}",
  "name": "Any File",
  "description": "Read any file from the filesystem"
}
```

**Problem:** No scoping or restriction — exposes the entire filesystem.

**Fix:** Scope templates to specific, safe paths:
```json
{
  "uriTemplate": "project://docs/{doc_name}",
  "name": "Project Documentation",
  "description": "Read project documentation files from the docs/ directory"
}
```

### ❌ Anti-Pattern 4: Dynamic Data as Static Resources

```json
{
  "uri": "metrics://current/cpu-usage",
  "name": "CPU Usage",
  "description": "Current CPU usage percentage"
}
```

**Problem:** CPU usage changes constantly. Caching a resource that's immediately stale is misleading.

**Fix:** Use a tool for highly dynamic data that requires a fresh read each time:
```json
{
  "name": "get_system_metrics",
  "description": "Get current system metrics (CPU, memory, disk). Returns real-time values.",
  "inputSchema": { "type": "object", "properties": { "metric": { "type": "string", "enum": ["cpu", "memory", "disk", "all"] } } }
}
```

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│              MCP RESOURCES & PROMPTS QUICK REFERENCE               │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  THREE MCP PRIMITIVES:                                            │
│    Tools     → AI-initiated actions (side effects OK)            │
│    Resources → Client-loaded context (read-only data)            │
│    Prompts   → User-selected templates (structured starts)       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  RESOURCE STRUCTURE:                                               │
│    uri:         Unique identifier (any scheme)                   │
│    name:        Human-readable label                             │
│    description: What it contains (for AI understanding)          │
│    mimeType:    Content format (text/plain, application/json)    │
│                                                                    │
│  RESOURCE TEMPLATES:                                               │
│    uriTemplate: Parameterized URI (RFC 6570 syntax)              │
│    Example: "postgres://tables/{table_name}/schema"              │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  PROTOCOL METHODS:                                                 │
│    resources/list            → Static resource descriptors       │
│    resources/read            → Content of a specific resource    │
│    resources/templates/list  → Available templates               │
│    resources/subscribe       → Change notifications              │
│    prompts/list              → Available prompt templates        │
│    prompts/get               → Resolved prompt messages          │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  RESOURCE vs TOOL DECISION:                                        │
│                                                                    │
│  Use RESOURCE when:                                               │
│    • Data is read-only (no side effects)                         │
│    • Content is relatively stable (cacheable)                    │
│    • Client should proactively provide context                   │
│    • No computation needed (just data retrieval)                 │
│                                                                    │
│  Use TOOL when:                                                    │
│    • Action has side effects (writes, mutations)                 │
│    • Dynamic computation required                                │
│    • AI needs to decide when to invoke                           │
│    • Results depend on parameters at call time                   │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Resources are READ-ONLY — never use for mutations             │
│  • Custom URI schemes are valid (postgres://, docs://, jira://)  │
│  • Templates use RFC 6570 syntax: {param_name}                   │
│  • Prompts are USER-initiated, not AI-initiated                  │
│  • Resources are CLIENT-loaded proactively                       │
│  • Tools are AI-called reactively                                │
│  • Resources + tools + prompts = complete MCP server             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You're building an MCP server for a development team. The server integrates with the team's PostgreSQL database and Jira project tracker. You need to decide how to expose the following capabilities:

1. **Database table schema** (structure of tables — stable, useful background)
2. **Execute a SQL query** (run a SELECT statement — dynamic, user-requested)
3. **Current sprint Jira issues** (list of active issues — changes daily)
4. **Jira issue details by ID** (specific issue content — parameterized)
5. **Create a Jira issue** (new issue — mutation, side effect)

### Question

**Which classification BEST assigns each capability to the correct MCP primitive?**

**A)**
| Capability | Primitive |
|-----------|-----------|
| Database schema | Resource |
| Execute SQL query | Resource |
| Current sprint issues | Resource |
| Jira issue by ID | Resource Template |
| Create Jira issue | Tool |

**B)**
| Capability | Primitive |
|-----------|-----------|
| Database schema | Tool |
| Execute SQL query | Tool |
| Current sprint issues | Tool |
| Jira issue by ID | Tool |
| Create Jira issue | Tool |

**C)**
| Capability | Primitive |
|-----------|-----------|
| Database schema | Resource |
| Execute SQL query | Tool |
| Current sprint issues | Resource |
| Jira issue by ID | Resource Template |
| Create Jira issue | Tool |

**D)**
| Capability | Primitive |
|-----------|-----------|
| Database schema | Resource |
| Execute SQL query | Tool |
| Current sprint issues | Tool |
| Jira issue by ID | Resource Template |
| Create Jira issue | Tool |

---

### Answer

**✅ Correct Answer: C**

**Explanation:**

- **Database schema** → **Resource** ✅ — Stable, read-only data useful as background context. Changes infrequently.
- **Execute SQL query** → **Tool** ✅ — Dynamic computation, user-requested action, parameterized at call time.
- **Current sprint issues** → **Resource** ✅ — Read-only list that can be proactively loaded. While it changes daily, it's stable within a session and useful as background context.
- **Jira issue by ID** → **Resource Template** ✅ — Read-only data, parameterized by issue ID (`jira://issues/{issue_id}`).
- **Create Jira issue** → **Tool** ✅ — Mutation with side effects (creates data in Jira).

**Why not D?** Option D marks "current sprint issues" as a tool. While it changes daily, it's still read-only context that the client can load proactively — it doesn't need AI decision-making to invoke. The resource + subscription pattern handles updates if needed.

**Why not A?** Option A marks "Execute SQL query" as a resource. SQL execution is dynamic, parameterized, and could have side effects — it's clearly a tool.

**Why not B?** Option B makes everything a tool, losing the advantages of proactive context loading and parameterized templates.

---

## Summary: Key Exam Takeaways

1. MCP has three primitives: **Tools** (actions), **Resources** (context), **Prompts** (templates)
2. Resources are **read-only** and loaded **proactively by the client** — not called by the AI
3. Resource templates use **RFC 6570 URI syntax** with `{parameter}` placeholders
4. Custom URI schemes are valid — servers define whatever scheme fits their domain
5. The decision between resource and tool depends on: **side effects, stability, and who initiates**
6. Prompts are **user-initiated** structured templates — different from both tools and resources
7. Protocol methods: `resources/list`, `resources/read`, `resources/templates/list`, `prompts/list`, `prompts/get`
8. Resources support **change notifications** via `notifications/resources/updated`
9. Always include descriptions on resources — they help the AI understand what context is available

---

*End of Day 23 Study Material*
