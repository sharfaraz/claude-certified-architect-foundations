# Day 26: MCP Integration Patterns & Sprint 1 Domain 2 Review

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 2: Tool Design & MCP Integration (18% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Complete MCP: Advanced Topics |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [End-to-End MCP Integration Flow](#1-end-to-end-mcp-integration-flow)
2. [MCP vs Direct API tool_use Comparison](#2-mcp-vs-direct-api-tool_use-comparison)
3. [Sprint 1 Domain 2 Review](#3-sprint-1-domain-2-review)
4. [Bridging: How MCP Standardizes API-Level Tools](#4-bridging-how-mcp-standardizes-api-level-tools)
5. [Common Exam Patterns: Choosing MCP vs API Tools](#5-common-exam-patterns-choosing-mcp-vs-api-tools)
6. [Anti-Patterns](#6-anti-patterns)
7. [Quick Reference Card](#7-quick-reference-card)
8. [Scenario Challenge](#8-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| MCP Specification (Full) | https://spec.modelcontextprotocol.io/ |
| Claude API Tool Use | https://docs.anthropic.com/en/docs/build-with-claude/tool-use |
| Claude Code MCP Integration | https://docs.anthropic.com/en/docs/claude-code/mcp |

---

## 1. End-to-End MCP Integration Flow

### The Complete Journey: .mcp.json → Result

```
┌─────────────────────────────────────────────────────────────────────┐
│            END-TO-END MCP INTEGRATION FLOW                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. CONFIGURATION (.mcp.json)                                        │
│     Developer defines servers: command, args, env                    │
│                                                                       │
│  2. STARTUP (Claude Code launches)                                   │
│     Claude Code reads .mcp.json → spawns server processes           │
│                                                                       │
│  3. INITIALIZATION (per server)                                      │
│     Client ↔ Server: initialize → capabilities → initialized       │
│                                                                       │
│  4. DISCOVERY                                                        │
│     Client → Server: tools/list, resources/list, prompts/list       │
│     Client aggregates all capabilities into tool registry           │
│                                                                       │
│  5. OPERATION (during conversation)                                  │
│     User asks question → Claude decides tool is needed              │
│     Claude produces tool_use → Client routes to correct server      │
│     Server executes → returns result → Claude incorporates          │
│                                                                       │
│  6. SHUTDOWN (session ends)                                          │
│     Claude Code terminates server processes                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Walkthrough

**Step 1: Configuration**
```json
// .mcp.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "${DATABASE_URL}"]
    }
  }
}
```

**Step 2: Startup**
```bash
# Claude Code reads .mcp.json and spawns:
npx -y @mcp/server-postgres postgresql://localhost:5432/mydb
```

**Step 3: Initialization**
```
Client → Server: initialize { capabilities: { sampling: {} } }
Server → Client: { capabilities: { tools: { listChanged: true } } }
Client → Server: initialized
```

**Step 4: Discovery**
```
Client → Server: tools/list
Server → Client: { tools: [
  { name: "query_database", description: "...", inputSchema: {...} },
  { name: "list_tables", description: "...", inputSchema: {...} }
]}
```

**Step 5: Operation**
```
User: "How many active users do we have?"
Claude: tool_use { name: "query_database", input: { query: "SELECT COUNT(*) FROM users WHERE active = true" } }
Client → Server: tools/call { name: "query_database", arguments: { query: "..." } }
Server → Client: { content: [{ type: "text", text: "{ \"count\": 15847 }" }] }
Claude: "There are 15,847 active users in the database."
```

**Step 6: Shutdown**
```
Claude Code session ends → server process terminated
```

---

## 2. MCP vs Direct API tool_use Comparison

### Side-by-Side Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    DIRECT API TOOL_USE (Sprint 1)                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Your App → Claude API → tool_use block → Your App executes        │
│                                                                      │
│  You define tools[] per request                                     │
│  You handle tool_use responses                                     │
│  You execute the tool logic                                        │
│  You send tool_result back                                         │
│                                                                      │
├────────────────────────────────────────────────────────────────────┤
│                    MCP TOOL INTEGRATION (Sprint 2)                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  .mcp.json → Server process → tools/list → tools/call → result    │
│                                                                      │
│  Server defines tools at startup                                   │
│  Client discovers tools via protocol                               │
│  Server executes the tool logic                                    │
│  Server returns result via protocol                                │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Detailed Comparison Table

| Dimension | API tool_use | MCP Tools |
|-----------|-------------|-----------|
| **Tool definition location** | In each API request (`tools[]`) | In server code (registered at startup) |
| **Tool execution** | Your application code | MCP server code |
| **Discovery** | You know your tools upfront | Client discovers via `tools/list` |
| **Schema format** | JSON Schema (`input_schema`) | JSON Schema (`inputSchema`) — same! |
| **Transport** | HTTPS to Claude API | stdio or HTTP+SSE to MCP server |
| **Reusability** | Per-application | Cross-client (any MCP client) |
| **Tool selection control** | `tool_choice` parameter | Always "auto" (AI decides) |
| **Forced tool use** | `tool_choice: {type: "tool", name: "X"}` | Not supported in MCP |
| **Result format** | `tool_result` message block | Protocol response content |
| **Session scope** | Per-API-request | Per-server-session (persistent) |
| **Error handling** | You check `is_error` in tool_result | `isError` field in protocol response |

### The Shared Foundation: JSON Schema

Both systems use identical JSON Schema for defining tool parameters:

```json
// API tool input_schema (Sprint 1)
{
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "SQL query to run" }
    },
    "required": ["query"]
  }
}

// MCP tool inputSchema (Sprint 2)
{
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "SQL query to run" }
    },
    "required": ["query"]
  }
}
```

**Key difference:** `input_schema` (snake_case) in API vs `inputSchema` (camelCase) in MCP.

---

## 3. Sprint 1 Domain 2 Review

### Core Concepts from Days 3-6 (Quick Refresher)

#### tool_use Message Structure (Day 3)

```json
// Claude's tool_use response
{
  "type": "tool_use",
  "id": "toolu_01XYZ",
  "name": "get_weather",
  "input": { "location": "Seattle", "units": "celsius" }
}

// Your tool_result response
{
  "type": "tool_result",
  "tool_use_id": "toolu_01XYZ",
  "content": "Current temperature in Seattle: 18°C, partly cloudy"
}
```

#### tool_choice Options (Day 4)

| Value | Behavior | Use Case |
|-------|----------|----------|
| `"auto"` | Claude decides whether to use a tool | General conversation |
| `"any"` | Claude MUST use one of the available tools | Force tool usage |
| `{"type": "tool", "name": "X"}` | Claude MUST use the named tool | Guaranteed structured output |

#### Key Principle: Programmatic Enforcement

```python
# ✅ Programmatic enforcement (Sprint 1 principle)
response = client.messages.create(
    tools=[extraction_tool],
    tool_choice={"type": "tool", "name": "extract_data"},  # Forces tool use
    messages=[...]
)

# ❌ Vague instruction (anti-pattern)
response = client.messages.create(
    tools=[extraction_tool],
    system="Please always use the extract_data tool",  # Hope-based, not guaranteed
    messages=[...]
)
```

#### stop_reason Values (Day 2)

| Value | Meaning | Action |
|-------|---------|--------|
| `end_turn` | Claude finished responding | Done |
| `tool_use` | Claude wants to call a tool | Execute tool, continue |
| `max_tokens` | Hit token limit mid-response | Response may be truncated |
| `stop_sequence` | Hit a custom stop sequence | Custom logic |

---

## 4. Bridging: How MCP Standardizes API-Level Tools

### The Standardization Story

```
Sprint 1 (API tools):
  Each app defines its own tools per request
  Each app implements its own tool execution
  No sharing between applications

Sprint 2 (MCP tools):
  Tools defined ONCE in a server
  Execution handled by the server
  ANY MCP client can use the same server
  Standard protocol for discovery and invocation
```

### MCP as a "Tool Interface Standard"

```
Before MCP:
  App A: defines get_weather tool + implements fetch logic
  App B: defines get_weather tool + implements fetch logic (duplicate!)
  App C: defines get_weather tool + implements fetch logic (duplicate!)

After MCP:
  Weather MCP Server: defines + implements get_weather ONCE
  App A: connects to weather server via MCP
  App B: connects to weather server via MCP
  App C: connects to weather server via MCP
```

### How the Two Layers Connect

```
┌───────────────────────────────────────────────────┐
│  Claude Code (MCP Client)                          │
│  Sees unified tool set: MCP tools + built-in tools │
├───────────────────────────────────────────────────┤
│                    ↕ MCP Protocol                   │
├───────────────────────────────────────────────────┤
│  MCP Server                                        │
│  Internally may use Claude API tool_use!           │
│  (Server could call Claude API for subtasks)       │
└───────────────────────────────────────────────────┘
```

### Mental Model: Layers

| Layer | What | Sprint |
|-------|------|--------|
| **Application Layer** | Your custom chat app uses Claude API | Sprint 1 |
| **Protocol Layer** | MCP standardizes tool interfaces | Sprint 2 |
| **Orchestration Layer** | Agents coordinate multiple tools/servers | Sprint 3 |

---

## 5. Common Exam Patterns: Choosing MCP vs API Tools

### Decision Matrix

| Question Pattern | Answer Pattern |
|-----------------|----------------|
| "Building a custom chat app that needs weather data" | API tools (you control the app) |
| "Extending Claude Code with database access" | MCP server (Claude Code is MCP client) |
| "Tool needed by multiple AI applications" | MCP server (reusable across clients) |
| "Need to force Claude to use a specific tool" | API tool_choice (MCP doesn't support forced selection) |
| "CI pipeline needs structured output" | Claude Code -p + --output-format json or --json-schema |
| "Team wants shared integrations for Claude Code" | MCP servers in project .mcp.json |

### Exam Question Patterns

**Pattern 1: "Which approach is correct for...?"**
- If the scenario involves Claude Code → MCP
- If the scenario involves a custom API application → API tools
- If the scenario needs guaranteed tool use → API tool_choice

**Pattern 2: "What's wrong with this configuration?"**
- Hardcoded secrets in .mcp.json → Use ${VAR_NAME}
- Interactive mode in CI → Use -p/--print
- Duplicate definitions → Define once in the right layer

**Pattern 3: "How does Claude decide which tool to use?"**
- API: tool_choice parameter (auto/any/specific)
- MCP: Always auto — descriptions drive selection
- Both: Tool descriptions are the primary selection signal

---

## 6. Anti-Patterns

### ❌ Anti-Pattern 1: Duplicating Tool Definitions Across Layers

```python
# In your API application (Layer 1)
tools = [{
    "name": "search_docs",
    "description": "Search documentation",
    "input_schema": { "type": "object", "properties": { "query": { "type": "string" } } }
}]

# AND in your MCP server (Layer 2)
# Same tool re-defined — which is authoritative?
{
    "name": "search_docs",
    "description": "Search documentation",
    "inputSchema": { "type": "object", "properties": { "query": { "type": "string" } } }
}
```

**Problem:** Maintenance burden — updating one and forgetting the other leads to inconsistency. Confusion about which layer executes the tool.

**Fix:** Define tools in ONE layer. If using MCP, let the MCP server own the tool definition and execution. If building a custom app without MCP, use API tools.

### ❌ Anti-Pattern 2: Using MCP When API Tools Suffice

```json
// .mcp.json for a simple app that only YOUR code uses
{
  "mcpServers": {
    "my-simple-helper": {
      "command": "node",
      "args": ["simple-tool-server.js"]
    }
  }
}
```

**Problem:** MCP adds protocol overhead (server process, initialization, discovery) for a tool that only one application needs. Over-engineering.

**Fix:** For single-application tools, use API `tools[]` directly. Reserve MCP for shared/reusable tools.

### ❌ Anti-Pattern 3: Relying on tool_choice Patterns in MCP

```
Developer thinks: "I'll set tool_choice to force the MCP tool..."
Reality: MCP doesn't support tool_choice — Claude always decides autonomously
```

**Problem:** Confusing API-level control (`tool_choice`) with MCP-level behavior (always auto).

**Fix:** In MCP, write excellent tool descriptions so Claude selects the right tool. In API, use `tool_choice` for guaranteed selection.

### ❌ Anti-Pattern 4: Not Reviewing Tools After Discovery

```
Server registers 30 tools → Client discovers all 30 → Conversation context bloated
Claude confused by overlapping tools → Picks wrong one
```

**Problem:** No curation of discovered tools. The full tool set overwhelms the AI.

**Fix:** Keep servers focused (5-15 tools each). Use clear, disambiguating descriptions. Consider multiple specialized servers instead of one large one.

---

## 7. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│         MCP INTEGRATION PATTERNS QUICK REFERENCE                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  END-TO-END FLOW:                                                  │
│    .mcp.json → spawn server → initialize → discover tools →     │
│    conversation → tools/call → result → response → shutdown      │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  API vs MCP COMPARISON:                                            │
│                                                                    │
│           API tools[]           │       MCP Tools                 │
│  ─────────────────────────────│──────────────────────────────   │
│  Per-request definition        │ Per-server-session (persistent) │
│  You execute tools             │ Server executes tools           │
│  tool_choice controls          │ Always auto (AI decides)        │
│  input_schema (snake_case)     │ inputSchema (camelCase)         │
│  Single application            │ Cross-client reusable           │
│  HTTP to Claude API            │ stdio or HTTP+SSE to server     │
│                                                                    │
│  SHARED: JSON Schema format for parameters                       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  SPRINT 1 REVIEW (key concepts):                                   │
│                                                                    │
│  tool_use:    Claude's request to call a tool                    │
│  tool_result: Your response with tool output                     │
│  tool_choice: auto | any | {type:"tool", name:"X"}               │
│  stop_reason: end_turn | tool_use | max_tokens | stop_sequence   │
│  Principle:   Programmatic enforcement > vague instructions       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  CHOOSING MCP vs API:                                              │
│                                                                    │
│  Use MCP when:                                                    │
│    • Extending Claude Code specifically                          │
│    • Tool needed by multiple AI clients                          │
│    • Team wants shared integrations                              │
│    • Standard interface benefits outweigh setup cost             │
│                                                                    │
│  Use API tools when:                                              │
│    • Building a custom application                               │
│    • Need tool_choice for forced tool use                        │
│    • Simple, single-app tool (not shared)                        │
│    • Rapid prototyping                                           │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • MCP and API use the SAME JSON Schema format                   │
│  • MCP doesn't support tool_choice — always auto                 │
│  • Don't duplicate tools across both layers                      │
│  • MCP adds reusability but also complexity                      │
│  • Descriptions are even MORE critical in MCP (no tool_choice)   │
│  • API = per-request; MCP = per-session (persistent)             │
│  • Know when each is appropriate for exam scenarios              │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. Scenario Challenge

### Scenario

A company has two teams building AI-powered applications:
- **Team A** builds a custom customer support chatbot using the Claude API directly
- **Team B** uses Claude Code for internal developer workflows

Both teams need access to the same ticket management system (create, search, update tickets). Currently, Team A has defined the ticket tools in their API `tools[]` array, and Team B has a separate MCP server with the same tools.

The tools have drifted — Team A's `create_ticket` requires a `priority` field, but Team B's version doesn't.

### Question

**What is the BEST architectural approach to resolve this tool duplication and prevent future drift?**

**A)** Have Team A copy Team B's MCP server tool definitions into their API tools[] array whenever Team B updates them.

**B)** Create a single MCP ticket server that Team B uses via Claude Code's .mcp.json, and have Team A rewrite their application to be an MCP client instead of using the Claude API directly.

**C)** Create a single MCP ticket server as the authoritative definition. Team B uses it via .mcp.json. Team A's custom app continues using the Claude API but generates its tools[] from the same MCP server's schema (or shared schema source).

**D)** Abandon MCP and have both teams use the Claude API with a shared tools[] JSON file that both applications import.

---

### Answer

**✅ Correct Answer: C**

**Explanation:**

- **Option C** addresses the root cause (no single source of truth) without forcing either team to change their architecture:
  - Single MCP server = authoritative tool definition (one place to update)
  - Team B uses it natively via MCP (their existing approach, now pointing to the shared server)
  - Team A generates API tools[] from the shared schema — they keep their Claude API architecture but derive definitions from the authoritative source
  - No drift because there's one source of truth

- **Option A** is manual synchronization — error-prone and will drift again.

- **Option B** forces Team A to abandon their working Claude API application and rewrite as an MCP client. Over-engineering for a tool definition consistency problem.

- **Option D** abandons MCP entirely, losing the protocol benefits Team B already has, and requires both teams to maintain a shared JSON file (still can drift if not properly governed).

**Key Exam Principle:** Define tools in ONE authoritative location. Other layers should derive from that source, not duplicate it. MCP servers can serve as the canonical definition that both MCP clients and API-based apps reference.

---

## Summary: Key Exam Takeaways

1. End-to-end MCP flow: **.mcp.json → startup → initialize → discover → operate → shutdown**
2. API tools and MCP tools share **identical JSON Schema** format (only key casing differs)
3. Key difference: API supports `tool_choice` for forced selection; MCP is always auto
4. Sprint 1 recap: `tool_use`, `tool_result`, `tool_choice`, `stop_reason` — still critical
5. MCP **standardizes** what Sprint 1 covers at the API level — same concepts, protocol wrapper
6. Choose MCP for **reusable, shared** tools; API tools for **single-app, forced-selection** needs
7. Anti-pattern: duplicating tool definitions across layers leads to drift
8. Single source of truth for tool definitions prevents inconsistency
9. Descriptions are **more critical** in MCP since there's no tool_choice to force selection

---

*End of Day 26 Study Material*
