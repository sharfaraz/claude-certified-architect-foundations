# Day 25: Advanced MCP Topics

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 2: Tool Design & MCP Integration (18% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Continue MCP: Advanced Topics |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [MCP Server Lifecycle](#1-mcp-server-lifecycle)
2. [Transport Mechanisms: stdio and HTTP+SSE](#2-transport-mechanisms-stdio-and-httpsse)
3. [Error Handling in MCP](#3-error-handling-in-mcp)
4. [Multi-Server Configurations](#4-multi-server-configurations)
5. [MCP Sampling: Server-Initiated LLM Requests](#5-mcp-sampling-server-initiated-llm-requests)
6. [Security Considerations](#6-security-considerations)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| MCP Specification — Lifecycle | https://spec.modelcontextprotocol.io/specification/lifecycle/ |
| MCP Specification — Transports | https://spec.modelcontextprotocol.io/specification/transports/ |
| MCP Specification — Sampling | https://spec.modelcontextprotocol.io/specification/client/sampling/ |

---

## 1. MCP Server Lifecycle

### The Three Phases

Every MCP server connection follows a strict lifecycle:

```
┌────────────────────────────────────────────────────────────┐
│              MCP SERVER LIFECYCLE                            │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  Phase 1: INITIALIZATION                                    │
│    Client → Server: initialize (protocol version, caps)    │
│    Server → Client: initialize response (server caps)      │
│    Client → Server: initialized (notification)             │
│                                                              │
│  Phase 2: OPERATION                                         │
│    Client ↔ Server: requests, responses, notifications     │
│    (tools/list, tools/call, resources/read, etc.)          │
│                                                              │
│  Phase 3: SHUTDOWN                                          │
│    Client → Server: close connection                        │
│    Server: cleanup and exit                                 │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### Initialization (Capability Negotiation)

During initialization, client and server **negotiate capabilities** — each declares what it supports:

```json
// Client sends initialize request
{
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": { "listChanged": true },
      "sampling": {}
    },
    "clientInfo": {
      "name": "claude-code",
      "version": "1.0.0"
    }
  }
}

// Server responds with its capabilities
{
  "protocolVersion": "2024-11-05",
  "capabilities": {
    "tools": { "listChanged": true },
    "resources": { "subscribe": true, "listChanged": true },
    "prompts": { "listChanged": true }
  },
  "serverInfo": {
    "name": "postgres-server",
    "version": "2.1.0"
  }
}
```

### Server Capabilities Matrix

| Capability | Meaning | Example |
|-----------|---------|---------|
| `tools` | Server exposes tools | Query database, create ticket |
| `resources` | Server exposes resources | DB schema, config files |
| `prompts` | Server exposes prompts | Code review template |
| `tools.listChanged` | Tools can change mid-session | Dynamic tool registration |
| `resources.subscribe` | Client can subscribe to resource updates | Config file changes |

### Client Capabilities

| Capability | Meaning |
|-----------|---------|
| `roots` | Client can provide root URIs (project directories) |
| `sampling` | Client supports server-initiated LLM requests |
| `roots.listChanged` | Client will notify when roots change |

### The initialized Notification

After receiving the server's capabilities, the client sends an `initialized` notification. Only **after this** can normal operations begin:

```
initialize (request) → response → initialized (notification) → READY
```

---

## 2. Transport Mechanisms: stdio and HTTP+SSE

### Two Transport Options

MCP supports two transport mechanisms for client-server communication:

| Transport | How It Works | Use Case |
|-----------|-------------|----------|
| **stdio** | Communication over stdin/stdout | Local servers, Claude Code default |
| **HTTP+SSE** | HTTP requests + Server-Sent Events | Remote servers, web-based clients |

### stdio Transport

```
┌───────────┐     stdin (JSON-RPC)      ┌───────────┐
│   Client  │ ─────────────────────────▶│   Server  │
│           │◀───────────────────────── │  (process)│
│(Claude    │     stdout (JSON-RPC)      │           │
│ Code)     │                            │           │
└───────────┘                            └───────────┘
```

**Characteristics:**
- Server runs as a **child process** of the client
- Communication via process stdin/stdout
- Messages are **JSON-RPC 2.0** format, newline-delimited
- Simplest setup — just specify `command` and `args` in `.mcp.json`
- Server lifetime matches the client session

**Example .mcp.json (stdio):**
```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    }
  }
}
```

### HTTP+SSE Transport

```
┌───────────┐     HTTP POST (requests)   ┌───────────┐
│   Client  │ ─────────────────────────▶│   Server  │
│           │◀───────────────────────── │  (remote) │
│           │     SSE (responses/notif)  │           │
└───────────┘                            └───────────┘
```

**Characteristics:**
- Server runs as a **separate HTTP service** (potentially remote)
- Client sends requests via HTTP POST
- Server pushes responses and notifications via Server-Sent Events (SSE)
- Supports **multiple simultaneous clients**
- Server lifetime is independent of any single client
- Better for shared/team servers

**When to Use Each:**

| Factor | stdio | HTTP+SSE |
|--------|-------|----------|
| **Setup complexity** | Simple (command in .mcp.json) | More complex (server deployment) |
| **Server lifetime** | Tied to client session | Independent |
| **Multi-client** | Single client only | Multiple clients supported |
| **Network** | Local only | Local or remote |
| **Claude Code default** | ✅ | Requires explicit URL config |
| **Typical use** | Dev tools, local integrations | Shared team servers, remote APIs |

---

## 3. Error Handling in MCP

### Error Response Format

MCP uses JSON-RPC 2.0 error responses:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "Invalid request: missing required field 'name'",
    "data": { "field": "name" }
  }
}
```

### Standard Error Codes

| Code | Name | Meaning |
|------|------|---------|
| `-32700` | Parse error | Invalid JSON received |
| `-32600` | Invalid request | Request structure is invalid |
| `-32601` | Method not found | Method doesn't exist on server |
| `-32602` | Invalid params | Method parameters are invalid |
| `-32603` | Internal error | Server internal error |

### Tool Call Errors

When a tool call fails, the server returns an error in the tool result:

```json
// Successful tool call
{
  "content": [
    { "type": "text", "text": "Query returned 42 rows" }
  ],
  "isError": false
}

// Failed tool call
{
  "content": [
    { "type": "text", "text": "Connection timeout: database unreachable after 30s" }
  ],
  "isError": true
}
```

### Key Distinction: Protocol Errors vs Tool Errors

```
Protocol Error (JSON-RPC error):
  → The request itself was malformed or the method doesn't exist
  → Client should NOT retry with same request
  → Example: calling a tool that doesn't exist

Tool Error (isError: true in result):
  → The request was valid but the tool execution failed
  → Client/AI MAY retry or adjust approach
  → Example: database timeout, invalid SQL syntax
```

### Timeout Behavior

- MCP doesn't define a standard timeout — clients implement their own
- Claude Code applies reasonable timeouts for tool calls
- Long-running tools should provide progress notifications
- If a tool times out, the client treats it as an error result

### Partial Results

Some tools may return partial results before completing:

```json
{
  "content": [
    { "type": "text", "text": "Found 50 of estimated 200 results (timeout reached)" }
  ],
  "isError": false
}
```

The `isError: false` indicates the tool ran successfully but may have incomplete data. The description in `text` communicates the limitation to the AI.

---

## 4. Multi-Server Configurations

### Running Multiple MCP Servers

Claude Code can connect to **multiple MCP servers simultaneously**, each providing different capabilities:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "${DATABASE_URL}"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@mcp/server-github"],
      "env": { "GITHUB_TOKEN": "${GH_TOKEN}" }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "@mcp/server-jira"],
      "env": { "JIRA_API_TOKEN": "${JIRA_TOKEN}" }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@mcp/server-filesystem", "/project/data"]
    }
  }
}
```

### How Multi-Server Discovery Works

```
Claude Code starts → spawns ALL configured servers in parallel
Each server: initialize → capabilities → tools/list

Aggregated tool set:
  [postgres] query_database, list_tables, describe_table
  [github]   create_pr, list_issues, get_commit
  [jira]     search_issues, create_issue, update_status
  [filesystem] read_file, write_file, list_directory
```

### Tool Namespace (Server Prefix)

Claude Code internally tracks which server owns which tool. When Claude calls a tool, the client routes the `tools/call` request to the correct server:

```
Claude: "I'll call query_database"
Client: "That's from the postgres server" → routes to postgres process
```

### Multi-Server Best Practices

| Practice | Rationale |
|----------|-----------|
| One server per service/domain | Clean separation of concerns |
| Unique tool names across servers | Avoid ambiguity in routing |
| Independent failure handling | One server crashing doesn't affect others |
| Minimal tool overlap | Reduce confusion about which tool to pick |

---

## 5. MCP Sampling: Server-Initiated LLM Requests

### What Is Sampling?

Sampling allows an MCP **server** to request the **client** to make an LLM call. This inverts the normal flow — instead of the AI calling tools, a tool requests AI assistance.

### Normal Flow vs Sampling Flow

```
Normal:   User → AI → tools/call → Server executes → result
Sampling: User → AI → tools/call → Server needs AI help → 
          Server → sampling/createMessage → Client calls LLM → 
          LLM result → Server continues → final result
```

### Why Sampling Exists

| Use Case | How Sampling Helps |
|----------|-------------------|
| Code analysis tools | Server processes code, asks LLM to summarize |
| Data enrichment | Server fetches raw data, asks LLM to classify |
| Multi-step reasoning | Server orchestrates, delegates reasoning to LLM |
| Content generation | Server provides context, LLM generates content |

### Sampling Request Structure

```json
// Server → Client: sampling/createMessage
{
  "method": "sampling/createMessage",
  "params": {
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Classify this customer feedback as positive, negative, or neutral: 'The product works great but shipping was slow'"
        }
      }
    ],
    "modelPreferences": {
      "hints": [{ "name": "claude-sonnet-4-20250514" }]
    },
    "maxTokens": 100
  }
}
```

### Security: Human-in-the-Loop

Sampling requires the **client's approval** — the client controls whether to honor the server's LLM request. This prevents servers from making unauthorized AI calls:

```
Server requests sampling → Client evaluates → 
  If approved: Client calls LLM, returns result to server
  If denied:   Client returns error to server
```

### Key Exam Point

Sampling is about **capability negotiation** — the client must declare `sampling` support during initialization, and the client has final say on whether to execute the request. This maintains the security boundary.

---

## 6. Security Considerations

### MCP Security Model

```
┌────────────────────────────────────────────────────────────┐
│              MCP SECURITY BOUNDARIES                         │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  USER (human)                                               │
│    ↓ grants permissions                                    │
│  CLIENT (Claude Code)                                       │
│    ↓ spawns and controls                                   │
│  SERVER (MCP process)                                       │
│    ↓ accesses                                              │
│  EXTERNAL SYSTEMS (databases, APIs, filesystems)           │
│                                                              │
│  Key principle: Client is the TRUST BOUNDARY               │
│  Server cannot access anything the client doesn't allow    │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### Security Layers

| Layer | Control | Example |
|-------|---------|---------|
| **Tool Access Control** | Which tools are available | Allowlists, denylists |
| **Input Validation** | Validate tool inputs before execution | SQL injection prevention |
| **Sandboxing** | Limit server's system access | Filesystem restrictions |
| **Env Var Scoping** | Secrets only where needed | Minimal env var exposure |
| **Capability Negotiation** | What features are enabled | Sampling opt-in |
| **Transport Security** | Protect communication | TLS for HTTP+SSE |

### Input Validation (Server-Side)

```typescript
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "query_database") {
    // ✅ Validate: only SELECT statements allowed
    const sql = args.query.trim().toUpperCase();
    if (!sql.startsWith("SELECT")) {
      return {
        content: [{ type: "text", text: "Error: Only SELECT queries are permitted" }],
        isError: true
      };
    }
    
    // ✅ Validate: no dangerous patterns
    if (sql.includes("DROP") || sql.includes("DELETE") || sql.includes("TRUNCATE")) {
      return {
        content: [{ type: "text", text: "Error: Destructive operations are not allowed" }],
        isError: true
      };
    }
    
    // Execute validated query
    const result = await db.query(args.query);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
});
```

### Filesystem Sandboxing

```typescript
// ✅ Restrict file access to a specific directory
const ALLOWED_ROOT = "/project/data";

function validatePath(requestedPath: string): boolean {
  const resolved = path.resolve(requestedPath);
  return resolved.startsWith(ALLOWED_ROOT);
}
```

### The --json-schema CLI Flag

The `--json-schema` flag enforces a **specific output schema** from Claude Code, adding a programmatic validation layer:

```bash
claude -p "Extract project metadata" --json-schema '{"type":"object","properties":{"name":{"type":"string"},"version":{"type":"string"}},"required":["name","version"]}'
```

This ensures output conforms to a defined structure — relevant for security-sensitive pipelines where unstructured output could introduce injection risks.

---

## 7. Anti-Patterns

### ❌ Anti-Pattern 1: Skipping Capability Negotiation

```typescript
// Server immediately starts accepting tool calls without initialization
server.setRequestHandler("tools/call", async (request) => {
  // Processing before initialize was completed...
});
```

**Problem:** Server may offer tools the client doesn't support, or miss important client capabilities.

**Fix:** Always complete the full initialization handshake before handling operational requests.

### ❌ Anti-Pattern 2: No Input Validation in Tool Handlers

```typescript
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "execute_query") {
    // ❌ Directly executing user-provided SQL without validation
    const result = await db.query(request.params.arguments.sql);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
});
```

**Problem:** SQL injection, destructive queries, resource exhaustion.

**Fix:** Validate inputs, restrict operations, set limits.

### ❌ Anti-Pattern 3: Single Mega-Server for All Capabilities

```json
{
  "mcpServers": {
    "everything": {
      "command": "node",
      "args": ["mega-server.js"],
      "env": {
        "DB_URL": "${DB_URL}",
        "GITHUB_TOKEN": "${GH_TOKEN}",
        "JIRA_TOKEN": "${JIRA_TOKEN}",
        "SLACK_TOKEN": "${SLACK_TOKEN}"
      }
    }
  }
}
```

**Problem:** Single point of failure. If this server crashes, all capabilities are lost. Also violates principle of least privilege (one server has all secrets).

**Fix:** Separate servers per domain with independent lifecycle and scoped secrets.

### ❌ Anti-Pattern 4: Ignoring Tool Errors

```
AI calls query_database → Server returns isError: true
AI ignores the error → Proceeds with empty/invalid data
```

**Problem:** The AI may generate incorrect responses based on missing data.

**Fix:** Tool errors should inform the AI's next action — retry with different params, escalate, or inform the user.

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│              ADVANCED MCP TOPICS QUICK REFERENCE                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  SERVER LIFECYCLE:                                                 │
│    1. Initialize (capability negotiation)                        │
│    2. Operate (requests, responses, notifications)               │
│    3. Shutdown (cleanup and exit)                                │
│                                                                    │
│  CAPABILITY NEGOTIATION:                                           │
│    Client declares: roots, sampling                              │
│    Server declares: tools, resources, prompts + listChanged      │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  TRANSPORTS:                                                       │
│                                                                    │
│    stdio:     Local process, stdin/stdout, Claude Code default   │
│    HTTP+SSE:  Remote service, HTTP POST + SSE, multi-client      │
│                                                                    │
│    stdio = simple, local, single-client                          │
│    HTTP+SSE = complex, remote-capable, multi-client              │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  ERROR HANDLING:                                                   │
│                                                                    │
│    Protocol errors: JSON-RPC error codes (-32600, -32601, etc.)  │
│    Tool errors: isError: true in tools/call response             │
│                                                                    │
│    Protocol error = malformed request (don't retry as-is)        │
│    Tool error = execution failed (may retry or adjust)           │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  MULTI-SERVER:                                                     │
│                                                                    │
│    • Multiple servers in one .mcp.json                           │
│    • Each spawned independently in parallel                      │
│    • Tools aggregated into single set for Claude                 │
│    • Client routes tools/call to correct server                  │
│    • One server crash doesn't affect others                      │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  SAMPLING:                                                         │
│                                                                    │
│    Server → Client: "Please make an LLM call for me"            │
│    Client controls: approve/deny the request                     │
│    Use case: Server needs AI reasoning during tool execution     │
│    Requires: Client declares sampling capability at init         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  SECURITY:                                                         │
│                                                                    │
│    Client = trust boundary (controls all access)                 │
│    Input validation = prevent injection/abuse                    │
│    Sandboxing = limit filesystem/network access                  │
│    Env vars = secrets management via ${VAR_NAME}                 │
│    --json-schema = enforce output structure                       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Initialization must complete BEFORE operations begin          │
│  • stdio is the default transport for Claude Code                │
│  • HTTP+SSE supports remote and multi-client scenarios           │
│  • Protocol errors vs tool errors = different handling           │
│  • isError: true means tool failed, not protocol failure         │
│  • Sampling inverts the flow: server asks client for LLM call   │
│  • Client has final authority over sampling requests             │
│  • Separate servers per domain = better isolation + security     │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You're designing an MCP architecture for a data analytics platform. The platform needs:
- A PostgreSQL server for queries (local, single-developer use)
- A shared team analytics server (multiple developers need access simultaneously)
- The analytics server performs complex computations and sometimes needs LLM help to generate natural language summaries of results

### Question

**Which architecture correctly addresses the transport, multi-client, and sampling requirements?**

**A)**
- PostgreSQL server: HTTP+SSE transport (remote)
- Analytics server: stdio transport (local process)
- Sampling: Not needed, just return raw numbers

**B)**
- PostgreSQL server: stdio transport (local process per developer)
- Analytics server: HTTP+SSE transport (shared remote service)
- Sampling: Analytics server requests LLM calls via `sampling/createMessage`
- Client declares `sampling` capability during initialization

**C)**
- PostgreSQL server: stdio transport (local)
- Analytics server: stdio transport (each developer spawns their own)
- Sampling: Analytics server calls the Claude API directly with its own API key

**D)**
- Both servers: HTTP+SSE transport (shared remote)
- Sampling: Client makes all LLM calls proactively before sending results to server

**A)** Incorrect transport choices (PostgreSQL is local, analytics needs multi-client)
**B)** Correct architecture
**C)** Server calling Claude API directly bypasses the MCP sampling protocol and trust boundary
**D)** PostgreSQL doesn't need remote transport, and proactive LLM calls aren't the same as sampling

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **PostgreSQL server: stdio** ✅ — Local, single-developer tool. Simple stdio transport is appropriate since each developer runs their own instance.

- **Analytics server: HTTP+SSE** ✅ — Shared team resource that multiple developers need to access simultaneously. HTTP+SSE supports multiple concurrent clients.

- **Sampling: server requests LLM via sampling/createMessage** ✅ — The analytics server needs AI help to generate summaries. Sampling is the correct MCP mechanism for server-initiated LLM requests, maintaining the trust boundary (client approves the call).

- **Client declares sampling capability** ✅ — Required during initialization for the server to know it can make sampling requests.

**Why not C?** A server calling the Claude API directly bypasses the MCP client's trust boundary. The client loses visibility and control over LLM usage. Sampling exists specifically to keep the client in the loop.

**Key Exam Principle:** Transport choice depends on locality and multi-client needs. Sampling is the correct mechanism for server-initiated LLM calls — it maintains security by keeping the client as the trust boundary.

---

## Summary: Key Exam Takeaways

1. MCP lifecycle: **Initialize → Operate → Shutdown** (initialization must complete first)
2. Capability negotiation declares what both client and server support
3. **stdio** = local, simple, single-client (Claude Code default)
4. **HTTP+SSE** = remote-capable, multi-client, independent server lifecycle
5. Protocol errors (JSON-RPC) ≠ Tool errors (`isError: true`) — different handling
6. Multi-server configs spawn servers independently with aggregated tool sets
7. **Sampling** = server asks client to make an LLM call (inverted flow)
8. Client is the **trust boundary** — controls sampling approval and tool access
9. Input validation and sandboxing are essential server-side security measures
10. `--json-schema` enforces structured output from Claude Code CLI

---

*End of Day 25 Study Material*
