# Day 38: Sprint 1 & 2 Review: Bridging to Advanced Architecture

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (review and synthesis) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Continue Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Sprint 1 Review: API Foundations](#1-sprint-1-review-api-foundations)
2. [Sprint 2 Review: Configuration & MCP](#2-sprint-2-review-configuration--mcp)
3. [Synthesis: Building Blocks for Agentic Architecture](#3-synthesis-building-blocks-for-agentic-architecture)
4. [Full Stack Mapping](#4-full-stack-mapping)
5. [Connecting the Dots: Sprint 3 Builds on 1 & 2](#5-connecting-the-dots-sprint-3-builds-on-1--2)
6. [Self-Assessment: Identify Weak Areas](#6-self-assessment-identify-weak-areas)
7. [Anti-Patterns Across All Sprints](#7-anti-patterns-across-all-sprints)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Messages API Reference | https://docs.anthropic.com/en/api/messages |
| Claude Code MCP | https://docs.anthropic.com/en/docs/claude-code/mcp |
| Claude Agent SDK | https://code.claude.com/docs/en/agent-sdk/agent-loop |

---

## 1. Sprint 1 Review: API Foundations

### Sprint 1 Concept Map (Days 1–15)

```
Sprint 1: The Foundation
├── Messages API Structure
│   ├── system, messages[], model, max_tokens
│   └── Content blocks: text, image, tool_use, tool_result
│
├── Tool Use (tool_use → tool_result cycle)
│   ├── tools[] array with name + description + input_schema
│   ├── stop_reason: "tool_use" triggers tool execution
│   └── tool_result feeds back into messages
│
├── tool_choice (controlling tool selection)
│   ├── "auto" → model decides
│   ├── "any" → model MUST use a tool
│   └── {"type": "tool", "name": "X"} → force specific tool
│
├── JSON Schema & Pydantic (input validation)
│   ├── type, properties, required, enum, descriptions
│   └── Pydantic for Python-side validation
│
├── stop_reason (response termination signals)
│   ├── "end_turn" → natural completion
│   ├── "tool_use" → wants to call a tool
│   ├── "max_tokens" → truncated
│   └── "stop_sequence" → custom stop hit
│
└── Error Handling & Retries
    ├── 4xx errors (client) vs 5xx (server)
    ├── Exponential backoff for rate limits
    └── escalate_to_human tool design
```

### Sprint 1 Key Code Pattern

```python
import anthropic

client = anthropic.Anthropic()

# The fundamental API call pattern from Sprint 1
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system="You are a helpful assistant with access to tools.",
    tools=[
        {
            "name": "get_weather",
            "description": "Get current weather for a city",
            "input_schema": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"},
                    "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                },
                "required": ["city"]
            }
        }
    ],
    tool_choice={"type": "auto"},
    messages=[
        {"role": "user", "content": "What's the weather in Paris?"}
    ]
)

# Sprint 1 knowledge: check stop_reason to know what happened
if response.stop_reason == "tool_use":
    # Extract tool call, execute it, feed result back
    tool_block = next(b for b in response.content if b.type == "tool_use")
    # ... execute and loop
elif response.stop_reason == "end_turn":
    # Final response ready
    text = next(b for b in response.content if b.type == "text")
```

### Sprint 1 Exam-Critical Facts

| Concept | Key Fact | Exam Relevance |
|---------|----------|----------------|
| `stop_reason` | Drives the agentic loop (Sprint 3!) | Highest exam signal |
| `tool_choice: "any"` | Forces tool use on next turn | Common trick question |
| `input_schema` | Must be valid JSON Schema | Schema errors = exam traps |
| `tool_result` | Must reference correct `tool_use_id` | ID mismatch = silent failure |
| Content blocks | Array of typed objects, not strings | Format questions |

---

## 2. Sprint 2 Review: Configuration & MCP

### Sprint 2 Concept Map (Days 16–30)

```
Sprint 2: The Stack
├── CLAUDE.md Configuration
│   ├── Hierarchy: global → project → subdirectory
│   ├── Purpose: rules, conventions, constraints
│   ├── Override behavior: more specific wins
│   └── Content: coding standards, forbidden patterns, tool preferences
│
├── .claude/ Directory Structure
│   ├── .claude/settings.json → project-level settings
│   ├── .claude/commands/ → custom slash commands
│   └── .claude/settings.local.json → personal overrides
│
├── MCP (Model Context Protocol)
│   ├── .mcp.json → server registration
│   ├── Server types: stdio, sse
│   ├── tools/list → discovery, tools/call → execution
│   ├── Tool definitions: name + description + inputSchema
│   └── Resources & prompts (beyond tools)
│
├── CLI Flags & Environment
│   ├── --model, --max-turns, --system-prompt
│   ├── --allowedTools (restrict available tools)
│   ├── --resume (session continuation)
│   └── Environment variables for API keys
│
└── CI/CD Integration
    ├── Claude Code in pipelines
    ├── Automated code review via CLI
    └── Headless mode for automation
```

### Sprint 2 Key Code Pattern

```json
// .mcp.json — MCP server registration (Sprint 2 Day 21+)
{
  "mcpServers": {
    "postgres-tools": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    },
    "jira-tools": {
      "command": "node",
      "args": ["./mcp-servers/jira-server.js"],
      "env": {
        "JIRA_API_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}
```

```markdown
<!-- CLAUDE.md — Sprint 2 Day 16-18 -->
# Project Rules

## Coding Standards
- Use TypeScript strict mode
- All functions must have JSDoc comments
- No console.log in production code (use logger)

## Forbidden Patterns
- Never use `any` type
- Never write to files outside /workspace
- Never modify .env files directly

## Testing
- All PRs must include tests
- Minimum 80% coverage for new code
```

### Sprint 2 Exam-Critical Facts

| Concept | Key Fact | Exam Relevance |
|---------|----------|----------------|
| CLAUDE.md hierarchy | Subdirectory > Project > Global | Override questions |
| .mcp.json | Registers servers, NOT tool definitions | Common confusion |
| Tool definitions | Live in server code, discovered via tools/list | Architecture questions |
| --allowedTools | Restricts which tools agent can use | Security questions |
| stdio vs sse | Transport types for MCP servers | Protocol questions |

---

## 3. Synthesis: Building Blocks for Agentic Architecture

### How Sprint 1 + Sprint 2 = Sprint 3

```
┌─────────────────────────────────────────────────────────────────┐
│              SPRINT SYNTHESIS MAP                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Sprint 1 (API)          Sprint 3 (Architecture)                 │
│  ─────────────           ────────────────────────                │
│  stop_reason ──────────▶ Agentic loop driver (Day 31)            │
│  tool_use/result ──────▶ Agent-tool interaction pattern          │
│  tool_choice ──────────▶ Controlling agent tool access           │
│  input_schema ─────────▶ Subagent parameter contracts            │
│  escalate_to_human ───▶ Escalation architecture (Day 37)         │
│                                                                  │
│  Sprint 2 (Config)       Sprint 3 (Architecture)                 │
│  ──────────────          ────────────────────────                │
│  CLAUDE.md rules ──────▶ Hook-based enforcement (Day 32)         │
│  .mcp.json servers ────▶ Tool registration for subagents         │
│  --allowedTools ───────▶ Subagent tool restriction (Day 33)      │
│  --resume ─────────────▶ Session management (Day 36)             │
│  CI/CD integration ───▶ Automated agent pipelines                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Progression: Concepts Layer on Each Other

```python
# Sprint 1: You learned to call the API with tools
response = client.messages.create(tools=tools, messages=messages)

# Sprint 2: You learned to configure WHICH tools and HOW
# (.mcp.json, CLAUDE.md, --allowedTools)

# Sprint 3: You learned to ORCHESTRATE multiple calls
# Agentic loop = Sprint 1's tool_use cycle + Sprint 2's tool config + loop control
for turn in range(max_turns):  # Day 31
    response = client.messages.create(  # Sprint 1 API
        tools=allowed_tools,  # Sprint 2 allowedTools concept
        messages=messages
    )
    if response.stop_reason == "end_turn":  # Sprint 1 stop_reason
        break
    # ... hooks fire (Day 32), check policy (Sprint 2 CLAUDE.md rules)
    # ... execute tools via MCP (Sprint 2) or direct
    # ... check circuit breakers (Day 37)
```

---

## 4. Full Stack Mapping

### End-to-End Request Flow

```
┌──────────────────────────────────────────────────────────────────┐
│             FULL STACK: FROM USER TO EXECUTION                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  USER REQUEST                                                     │
│       │                                                           │
│       ▼                                                           │
│  ┌────────────────┐                                               │
│  │ API Request    │ ◀── Sprint 1: messages, model, tools,         │
│  │ (Messages API) │     max_tokens, system prompt                 │
│  └───────┬────────┘                                               │
│          │                                                        │
│          ▼                                                        │
│  ┌────────────────┐                                               │
│  │ Tool Invocation│ ◀── Sprint 1: stop_reason="tool_use"          │
│  │ Decision       │     Sprint 2: tool_choice, allowedTools       │
│  └───────┬────────┘                                               │
│          │                                                        │
│          ▼                                                        │
│  ┌────────────────┐                                               │
│  │ MCP Server     │ ◀── Sprint 2: .mcp.json, tools/call          │
│  │ (Tool Provider)│     Tool executes on the server               │
│  └───────┬────────┘                                               │
│          │                                                        │
│          ▼                                                        │
│  ┌────────────────┐                                               │
│  │ Agent Loop     │ ◀── Sprint 3 Day 31: iterate on tool_use      │
│  │ (Iterate)      │     Hooks fire (Day 32), guards check         │
│  └───────┬────────┘                                               │
│          │                                                        │
│          ▼                                                        │
│  ┌────────────────┐                                               │
│  │ Coordinator    │ ◀── Sprint 3 Day 34: delegate to subagents    │
│  │ (Orchestrate)  │     Synthesize results                        │
│  └───────┬────────┘                                               │
│          │                                                        │
│          ▼                                                        │
│  ┌────────────────┐                                               │
│  │ Subagent       │ ◀── Sprint 3 Day 33-35: focused execution     │
│  │ (Execute)      │     allowedTools, own loop, return result     │
│  └───────┬────────┘                                               │
│          │                                                        │
│          ▼                                                        │
│  FINAL RESPONSE TO USER                                           │
└──────────────────────────────────────────────────────────────────┘
```

### Layer-to-Day Mapping

| Layer | Sprint | Key Days | Concepts |
|-------|--------|----------|----------|
| API Request | Sprint 1 | Days 1-5 | messages, model, system, tools |
| Tool Invocation | Sprint 1 | Days 6-10 | tool_use, tool_result, stop_reason |
| Tool Configuration | Sprint 2 | Days 16-22 | CLAUDE.md, .mcp.json, MCP tools |
| Agent Loop | Sprint 3 | Day 31 | Agentic loop, max_turns, termination |
| Hooks & Guards | Sprint 3 | Day 32 | Lifecycle hooks, enforcement |
| Task Decomposition | Sprint 3 | Day 33 | Task tool, allowedTools |
| Coordination | Sprint 3 | Days 34-35 | Coordinator-subagent, topologies |
| Session/Context | Sprint 3 | Day 36 | Persistence, compression, budgets |
| Escalation | Sprint 3 | Day 37 | Error propagation, circuit breakers |

---

## 5. Connecting the Dots: Sprint 3 Builds on 1 & 2

### Connection 1: stop_reason → Loop Control

```python
# Sprint 1 taught: stop_reason tells you WHY Claude stopped
# Sprint 3 uses it as: THE LOOP DRIVER

# Sprint 1 (single call):
if response.stop_reason == "tool_use":
    execute_tool()

# Sprint 3 (agentic loop):
while True:
    response = call_claude(messages)
    match response.stop_reason:
        case "tool_use": continue_loop()   # Act + Observe
        case "end_turn": break              # Goal met
        case "max_tokens": handle_overflow()
```

### Connection 2: tool_choice → Subagent Restrictions

```python
# Sprint 1 taught: tool_choice controls which tools Claude picks
# Sprint 3 uses it as: SUBAGENT TOOL BOUNDARIES

# Sprint 1 (force specific tool):
tool_choice = {"type": "tool", "name": "search"}

# Sprint 3 (restrict subagent to read-only tools):
Task(
    description="Analyze code",
    allowed_tools=["read_file", "grep_search"],  # Like tool_choice but broader
)
```

### Connection 3: CLAUDE.md Rules → Runtime Hooks

```python
# Sprint 2 taught: CLAUDE.md defines rules (static, advisory)
# Sprint 3 enforces: Hooks block violations at runtime

# Sprint 2 (CLAUDE.md):
# "Never write to production database"

# Sprint 3 (Hook):
class ProdDBGuard(Hook):
    event = "pre-tool-call"
    def execute(self, ctx):
        if ctx.tool_name == "run_sql" and "production" in ctx.tool_input.get("db", ""):
            return HookAction.BLOCK(reason="CLAUDE.md rule: no prod DB writes")
```

### Connection 4: MCP Servers → Agent Tool Providers

```python
# Sprint 2 taught: MCP servers register tools
# Sprint 3 uses them as: Tool backends for agents and subagents

# Sprint 2 (.mcp.json):
# Registers "postgres-tools" server with query_database tool

# Sprint 3 (agent uses MCP tool):
agent = Agent(
    tools=["query_database"],  # Provided by MCP server
    # Agent calls this tool during its agentic loop
)
```

### Connection 5: escalate_to_human → Escalation Architecture

```python
# Sprint 1 taught: Design escalate_to_human as a tool
# Sprint 3 builds: Full escalation architecture around it

# Sprint 1 (tool design):
escalate_tool = {"name": "escalate_to_human", "input_schema": {...}}

# Sprint 3 (architecture):
# Escalation triggers → circuit breakers → error propagation → escalate_to_human
```

---

## 6. Self-Assessment: Identify Weak Areas

### Sprint 1 Confidence Check

| Topic | Can You? | Status |
|-------|----------|--------|
| Write a complete Messages API call | ✅ / ⚠️ / ❌ | |
| Explain all stop_reason values | ✅ / ⚠️ / ❌ | |
| Design a tool with proper JSON Schema | ✅ / ⚠️ / ❌ | |
| Implement the tool_use → tool_result cycle | ✅ / ⚠️ / ❌ | |
| Explain tool_choice options and when to use each | ✅ / ⚠️ / ❌ | |
| Handle errors with exponential backoff | ✅ / ⚠️ / ❌ | |

### Sprint 2 Confidence Check

| Topic | Can You? | Status |
|-------|----------|--------|
| Explain CLAUDE.md hierarchy and overrides | ✅ / ⚠️ / ❌ | |
| Write a valid .mcp.json configuration | ✅ / ⚠️ / ❌ | |
| Explain MCP tool discovery (tools/list) | ✅ / ⚠️ / ❌ | |
| Use CLI flags (--model, --allowedTools, --resume) | ✅ / ⚠️ / ❌ | |
| Distinguish server-registered tools from .mcp.json | ✅ / ⚠️ / ❌ | |
| Set up Claude Code for CI/CD pipelines | ✅ / ⚠️ / ❌ | |

### Sprint 3 (So Far) Confidence Check

| Topic | Can You? | Status |
|-------|----------|--------|
| Implement an agentic loop with circuit breakers | ✅ / ⚠️ / ❌ | |
| Design hooks for the four lifecycle events | ✅ / ⚠️ / ❌ | |
| Choose correct decomposition pattern | ✅ / ⚠️ / ❌ | |
| Design coordinator-subagent boundaries | ✅ / ⚠️ / ❌ | |
| Select the right orchestration topology | ✅ / ⚠️ / ❌ | |
| Implement context preservation strategies | ✅ / ⚠️ / ❌ | |
| Design escalation triggers and circuit breakers | ✅ / ⚠️ / ❌ | |

### Priority Review Areas

```
If you marked ❌ or ⚠️ on any Sprint 1 item:
  → CRITICAL: These are foundations. Review Days 1-15 material.
  → Sprint 3 CANNOT be understood without Sprint 1 mastery.

If you marked ❌ or ⚠️ on any Sprint 2 item:
  → IMPORTANT: These are configuration patterns used daily.
  → Sprint 3 hooks and agent config build directly on these.

If you marked ❌ or ⚠️ on Sprint 3 items:
  → EXPECTED: You just learned these. Review Days 31-37.
  → Focus on the connections to Sprint 1 & 2, not just new concepts.
```

---

## 7. Anti-Patterns Across All Sprints

### Cross-Sprint Anti-Pattern Gallery

| Sprint | Anti-Pattern | Why It's Wrong | Correct Approach |
|--------|-------------|----------------|------------------|
| 1 | Ignoring stop_reason | Miss tool calls or truncation | Always check and branch |
| 1 | Hardcoded tool_use_id | IDs are dynamic per response | Extract from response |
| 1 | No input_schema validation | Invalid params pass silently | Always define full schema |
| 2 | Rules in wrong CLAUDE.md level | Rules don't apply where needed | Match rule to scope |
| 2 | Tool definitions in .mcp.json | .mcp.json only registers servers | Define tools in server code |
| 2 | No --allowedTools in CI | CI agent has too much power | Restrict tools for automation |
| 3 | Loops without circuit breakers | Infinite loops possible | Always set max_turns |
| 3 | Silent hook modifications | Debugging impossible | Log all modifications |
| 3 | Over-decomposition | Overhead exceeds benefit | Match granularity to complexity |
| 3 | Coordinator micromanages | Defeats delegation purpose | Set goals, not implementations |
| 3 | Unbounded context growth | Context overflow crashes | Monitor and compress |
| 3 | Blind retries | Same error repeated forever | Feed error context back |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│      SPRINT 1-2-3 SYNTHESIS — QUICK REFERENCE           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  SPRINT 1 → SPRINT 3 CONNECTIONS:                       │
│    stop_reason      → Agentic loop driver               │
│    tool_use/result  → Agent-tool interaction             │
│    tool_choice      → Subagent tool restrictions         │
│    input_schema     → Subagent parameter contracts       │
│    escalate_to_human→ Escalation architecture            │
│                                                         │
│  SPRINT 2 → SPRINT 3 CONNECTIONS:                       │
│    CLAUDE.md rules  → Hook-based enforcement             │
│    .mcp.json        → Tool registration for agents       │
│    --allowedTools   → Subagent tool boundaries           │
│    --resume         → Session management                 │
│    CI/CD            → Automated agent pipelines          │
│                                                         │
│  FULL STACK (bottom to top):                            │
│    API Call → Tool Invocation → MCP Server               │
│    → Agent Loop → Coordinator → Subagent → Result       │
│                                                         │
│  EXAM WEIGHT REMINDER:                                  │
│    Domain 1 (Agentic): 27%  ← Sprint 3 focus           │
│    Domain 2 (Tools):   18%  ← Sprint 1+2               │
│    Domain 3 (Config):  16%  ← Sprint 2                  │
│                                                         │
│  NEXT: Days 39-43 → Deep scenario practice              │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You're asked to design a complete system: a Claude-powered agent that reviews PRs, runs security scans, and posts comments. It must use MCP for GitHub integration, enforce coding standards from CLAUDE.md, and escalate to a human reviewer for critical findings.

**Question 1:** Which Sprint 1 concept is the FOUNDATION for the agent's iterative behavior?

- A) tool_choice: "any"
- B) stop_reason driving the loop (tool_use → continue, end_turn → stop)
- C) max_tokens parameter
- D) The system prompt content

**Answer:** **B** — The agentic loop is fundamentally driven by stop_reason. When Claude returns `"tool_use"`, the loop continues (execute tool, feed result back). When it returns `"end_turn"`, the agent is done. This Sprint 1 concept IS the Sprint 3 agentic loop.

---

**Question 2:** Where do the coding standards come from, and how are they enforced at runtime?

- A) Hardcoded in the agent's source code
- B) CLAUDE.md (Sprint 2) defines them; hooks (Sprint 3) enforce them at runtime
- C) The MCP server enforces coding standards
- D) The user specifies them in each request

**Answer:** **B** — This is the Sprint 2 → Sprint 3 bridge. CLAUDE.md defines static rules ("no console.log in production"). Hooks provide runtime enforcement (pre-tool-call hook blocks write_file if content contains console.log). Static policy → dynamic enforcement.

---

**Question 3:** The system needs to post GitHub comments via MCP. What Sprint 2 configuration enables this?

- A) Adding the tool definition to CLAUDE.md
- B) .mcp.json registering a GitHub MCP server that provides comment tools
- C) CLI flag --tools="github_comment"
- D) Defining the tool in the API request's tools[] array

**Answer:** **B** — .mcp.json registers the MCP server (Sprint 2, Day 21). The server's code defines tools like `post_pr_comment`. Claude Code discovers these tools via `tools/list`. This is the MCP architecture: .mcp.json → server starts → tools registered → agent discovers → agent calls.

---

**Question 4:** When should this system escalate to a human reviewer?

- A) Always — every PR should have human review
- B) When the security scan finds critical/high severity issues (confidence threshold)
- C) Never — the agent should handle everything autonomously
- D) Only when the agent crashes

**Answer:** **B** — This connects Sprint 1's `escalate_to_human` tool with Sprint 3's escalation trigger design. The trigger is: severity ≥ critical/high. The agent handles low/medium findings autonomously but escalates critical findings because the consequences of a wrong decision are too high (policy boundary trigger).

---

**Question 5:** Map the full request flow for this system from user to final comment.

- A) User → API → Tool → Response
- B) User → Coordinator → (Security Subagent | Style Subagent) → Aggregation → GitHub MCP → Comment
- C) User → CLAUDE.md → MCP → Response
- D) User → CLI → Single Agent → All work done in one call

**Answer:** **B** — This is the full stack from Sprint 3 Day 35. The coordinator (topology: parallel fan-out/fan-in) dispatches independent analyses (security, style) to subagents. Each subagent uses restricted tools. The coordinator aggregates results, then calls the GitHub MCP tool to post the synthesized comment. Every Sprint contributes: API (Sprint 1), MCP/config (Sprint 2), orchestration (Sprint 3).
