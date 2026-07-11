# Day 20: Iterative Refinement Workflows & Sprint 1 Review

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Complete Claude Code in Action |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Iterative Refinement Patterns](#1-iterative-refinement-patterns)
2. [The --resume Flag](#2-the---resume-flag)
3. [fork_session: Branching Conversations](#3-fork_session-branching-conversations)
4. [Combining Plan Mode with Iterative Refinement](#4-combining-plan-mode-with-iterative-refinement)
5. [Bridging Sprint 1 API Concepts to MCP](#5-bridging-sprint-1-api-concepts-to-mcp)
6. [Sprint 1 Concepts Quick Review](#6-sprint-1-concepts-quick-review)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code Tutorials | https://docs.anthropic.com/en/docs/claude-code/tutorials |

---

## 1. Iterative Refinement Patterns

### Core Concept

Iterative refinement is a **feedback-loop workflow** where you progressively improve Claude's output through multiple rounds rather than expecting perfection in a single prompt. This mirrors real software development — code rarely ships perfectly on the first draft.

### The Fundamental Pattern

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Prompt  │ ──▶ │  Output  │ ──▶ │  Review  │ ──▶ │  Refine  │ ──╮
│  (User)  │     │ (Claude) │     │  (User)  │     │  (User)  │   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘   │
                                                                     │
     ┌───────────────────────────────────────────────────────────────╯
     │
     ▼
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Prompt  │ ──▶ │  Output  │ ──▶ │  Accept  │
│  (v2)    │     │  (v2)    │     │  (User)  │
└──────────┘     └──────────┘     └──────────┘
```

### Refinement Strategies

| Strategy | Description | Example |
|----------|-------------|---------|
| **Scope narrowing** | Focus on a specific aspect | "Just fix the error handling, leave the rest" |
| **Scope expanding** | Add requirements iteratively | "Good, now also add input validation" |
| **Style adjustment** | Change approach, keep functionality | "Use composition instead of inheritance" |
| **Error correction** | Fix specific issues | "The JWT expiry should be 1h, not 24h" |
| **Decomposition** | Break large tasks into passes | "First pass: interfaces only. Second: implementation" |

### Practical Workflow Example

```
Round 1 — Initial generation:
> Generate a rate limiter middleware for Express

Round 2 — Scope adjustment:
> Use a sliding window algorithm instead of fixed window.
  Also add per-route configuration.

Round 3 — Error correction:
> The Redis key should include the route path, not just the IP.
  Also handle Redis connection failures gracefully (fail-open).

Round 4 — Finalization:
> Add JSDoc documentation and export the types.
  This looks good — write it to src/middleware/rate-limiter.ts
```

### When Iterative Refinement Beats Single-Shot

| Scenario | Single-Shot | Iterative |
|----------|-------------|-----------|
| Simple utility function | ✅ Sufficient | Overkill |
| Complex middleware | Likely incomplete | ✅ Build up incrementally |
| Architecture decisions | Too many assumptions | ✅ Explore options first |
| Unfamiliar domain | High error rate | ✅ Course-correct early |
| Code review feedback | N/A | ✅ Address issues one by one |

---

## 2. The --resume Flag

### What is --resume?

The `--resume` flag allows you to **continue a previous Claude Code session** from where it left off. This preserves the full conversation context, including all refinements, decisions, and generated code.

### Syntax

```bash
# Resume the most recent session
claude --resume

# Resume a specific session by ID
claude --resume <session-id>
```

### How Sessions Work

```
Session 1 (Day 1):
  > Build the user model
  > Add email validation
  > [Session ends — context preserved]

Session 2 (Day 2, resumed):
  $ claude --resume
  > Now add password hashing to the user model
  > [Claude remembers all prior context from Session 1]
```

### Key Behaviors

| Aspect | Behavior |
|--------|----------|
| **Context** | Full conversation history is restored |
| **Files** | Claude remembers which files were created/modified |
| **Decisions** | Previous architectural choices are retained |
| **Rules** | CLAUDE.md and rules are re-loaded fresh |
| **Session ID** | Displayed at session start for future reference |

### Use Cases for --resume

| Use Case | Why Resume? |
|----------|-------------|
| Multi-day feature development | Keep context across work sessions |
| Interrupted work | Pick up exactly where you left off |
| Iterative refinement across sessions | Don't re-explain prior decisions |
| Debugging a previous implementation | Claude remembers what it built |
| Progressive enhancement | Add features to code from prior session |

### Practical Example

```bash
# Day 1: Start building the auth system
$ claude
> Build a JWT authentication system with refresh tokens
> [Generates code, discusses approach]
> [End session — note the session ID: abc123]

# Day 2: Continue where we left off
$ claude --resume
> The refresh token rotation has a race condition.
  Two concurrent requests with the same refresh token
  should not both succeed. Add locking.
# Claude remembers the entire auth system context
```

### --resume vs Starting Fresh

| Factor | --resume | New Session |
|--------|----------|-------------|
| Context | Full history available | Starts clean |
| Token usage | History counts toward context window | Fresh budget |
| Best for | Continuing related work | Unrelated new tasks |
| Risk | Context window fills up on long sessions | May re-explain things |
| Performance | May slow with very long histories | Consistent speed |

---

## 3. fork_session: Branching Conversations

### What is fork_session?

`fork_session` creates a **branch** of the current conversation, allowing you to explore an alternative approach without losing the original thread. Think of it as `git branch` for conversations.

### Mental Model

```
Original session:
  Turn 1 → Turn 2 → Turn 3 → Turn 4 → ...
                        │
                        └── fork_session ──▶ Fork Turn 3a → Fork Turn 4a → ...
                                             (explores alternative)
```

### Use Cases

| Use Case | Description |
|----------|-------------|
| **A/B approaches** | Try two architectures, compare results |
| **Risk-free exploration** | Test a risky change without losing stable state |
| **What-if analysis** | "What if we used MongoDB instead of Postgres?" |
| **Parallel refinement** | Refine UI and API independently from same base |
| **Rollback point** | Fork before a risky operation — revert by using original |

### How fork_session Works

```bash
# In a Claude Code session:
> We've built the base API. Now I want to explore two auth approaches.

# Fork 1: JWT-based auth
> fork_session
> Implement JWT-based authentication with access + refresh tokens

# Back in original (or fork again):
> fork_session  
> Implement session-based authentication with Redis store

# Compare both forks, pick the winner
```

### fork_session vs --resume

| Feature | fork_session | --resume |
|---------|-------------|----------|
| **Purpose** | Branch/explore alternatives | Continue linear session |
| **Context** | Copies current state to new branch | Loads previous session |
| **Relationship** | Parent-child (branching) | Sequential continuation |
| **When** | During an active session | Starting a new session |
| **Analogy** | `git branch` | `git stash pop` |

### Practical Workflow: Exploration Pattern

```
Step 1: Build base (shared context)
  > Set up the project structure with Express, Prisma, and TypeScript

Step 2: Fork to explore options
  > fork_session
  > Option A: Use tRPC for type-safe API layer
  
  > fork_session  
  > Option B: Use REST with OpenAPI code generation

Step 3: Evaluate each fork's output

Step 4: Continue with the preferred approach in that fork
  (or start fresh and tell Claude "use the tRPC approach from our exploration")
```

---

## 4. Combining Plan Mode with Iterative Refinement

### The Full Workflow

Plan mode and iterative refinement work together as a **safety-first iteration cycle**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    FULL REFINEMENT WORKFLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. PROMPT        "Add order processing endpoint"                │
│       │                                                           │
│       ▼                                                           │
│  2. PLAN          Claude presents approach (plan mode)           │
│       │                                                           │
│       ▼                                                           │
│  3. REFINE PLAN   "Use event sourcing, not direct writes"        │
│       │                                                           │
│       ▼                                                           │
│  4. REVISED PLAN  Claude updates approach                        │
│       │                                                           │
│       ▼                                                           │
│  5. APPROVE       "Looks good, proceed"                          │
│       │                                                           │
│       ▼                                                           │
│  6. EXECUTE       Claude implements the code                     │
│       │                                                           │
│       ▼                                                           │
│  7. REFINE CODE   "Fix the error handling in processOrder"       │
│       │                                                           │
│       ▼                                                           │
│  8. ACCEPT        Final output satisfactory                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Insight: Two Levels of Refinement

| Level | What's Refined | When |
|-------|---------------|------|
| **Plan refinement** | Architecture, approach, scope | Before execution |
| **Code refinement** | Implementation details, bugs, style | After execution |

### Example: Full Cycle

```
> /add-endpoint orders (invokes command with plan mode)

Claude [PLAN]:
  "I'll create 5 files following the skill:
   routes/orders.ts, controllers/orders.ts, services/orders.ts,
   repositories/orders.ts, schemas/orders.ts
   Using event-driven architecture with Kafka..."

User: "Don't use Kafka — we use RabbitMQ. Also add idempotency."

Claude [REVISED PLAN]:
  "Updated: Using RabbitMQ for events, adding idempotency key
   to the request schema and a deduplication check in the service..."

User: "Approved. Execute."

Claude [EXECUTES]: Creates all 5 files.

User: "The idempotency check should use Redis, not the database.
       Also add a TTL of 24 hours for the idempotency keys."

Claude [REFINES]: Updates the service to use Redis for idempotency.

User: "Perfect. Done."
```

---

## 5. Bridging Sprint 1 API Concepts to MCP

### Why This Bridge Matters

Sprint 1 covered the **Claude API** (messages, tools, streaming). Sprint 2 introduces **MCP** (Model Context Protocol). Understanding how they connect is essential for the exam.

### Concept Mapping: API → MCP

| Sprint 1 (API Concept) | Sprint 2 (MCP Equivalent) | Key Difference |
|------------------------|--------------------------|----------------|
| `tools[]` in API request | MCP tool definitions | MCP tools are protocol-registered, not per-request |
| `tool_use` content block | MCP tool call request | MCP routes through protocol layer |
| `tool_result` response | MCP tool call response | MCP adds transport + discovery |
| Function parameters (JSON Schema) | MCP input schema | Same schema format, different registration |
| System prompt context | MCP resources + context | MCP provides structured context injection |

### Architecture Comparison

```
SPRINT 1: Claude API Direct Tool Use
┌──────────┐     ┌───────────┐     ┌──────────┐
│  Client  │ ──▶ │ Claude API│ ──▶ │  Client  │ (handles tool_use)
│  App     │ ◀── │ (messages)│ ◀── │  App     │ (returns tool_result)
└──────────┘     └───────────┘     └──────────┘

SPRINT 2: MCP Architecture
┌──────────┐     ┌───────────┐     ┌───────────┐     ┌──────────┐
│  Claude  │ ──▶ │ MCP Client│ ──▶ │MCP Server │ ──▶ │  Tool    │
│  Code    │ ◀── │ (built-in)│ ◀── │(separate) │ ◀── │Execution │
└──────────┘     └───────────┘     └───────────┘     └──────────┘
```

### Key Differences Explained

| Aspect | API tools[] | MCP Tools |
|--------|-------------|-----------|
| **Registration** | Defined per API call | Registered on server startup |
| **Discovery** | Client knows tools upfront | Client discovers via `tools/list` |
| **Execution** | Client implements handlers | MCP server implements handlers |
| **Transport** | HTTP (API calls) | stdio or HTTP/SSE |
| **Lifecycle** | Stateless (per request) | Stateful (persistent connection) |
| **Scope** | Single application | Shared across MCP-compatible clients |

### How They Work Together

In Claude Code, both systems coexist:
- **API tools** power Claude's core capabilities (file editing, terminal, web search)
- **MCP tools** extend Claude Code with custom capabilities via external servers

```
Claude Code Session:
├── Built-in tools (API-level): file read, write, terminal, search
└── MCP tools (protocol-level): custom DB access, Jira, Slack, etc.
     └── Configured in .mcp.json (Day 21 topic)
```

### Exam Bridge Questions Pattern

The exam may ask you to compare or choose between API-level and MCP-level tool integration:

> "When should you define a tool in the API tools[] array vs implement it as an MCP server?"

**Answer framework:**
- API tools[] → When your app directly controls the tool execution loop
- MCP server → When the tool should be reusable across multiple AI clients, or when it wraps an external service

---

## 6. Sprint 1 Concepts Quick Review

### Key Concepts from Days 1-15

| Day | Topic | Key Takeaway |
|-----|-------|--------------|
| 1 | API Basics | Messages API with system/user/assistant roles |
| 2 | Prompt Engineering | Clear instructions, XML tags, examples |
| 3 | System Prompts | Sets behavior, persona, constraints |
| 4 | Multi-turn Conversations | Maintaining context across messages |
| 5 | Streaming | Server-Sent Events for real-time output |
| 6 | Tool Use (Basics) | tools[], tool_use, tool_result flow |
| 7 | Tool Use (Advanced) | Multi-tool, parallel, nested calls |
| 8 | Vision & Multimodal | Image input via base64 or URL |
| 9 | Extended Thinking | Chain-of-thought for complex reasoning |
| 10 | Token Management | Context windows, counting, optimization |
| 11 | Error Handling | Retry strategies, rate limits, fallbacks |
| 12 | Prompt Caching | Cache control for repeated context |
| 13 | Batches API | Async bulk processing at 50% cost |
| 14 | Content Moderation | Safety classifiers, content filtering |
| 15 | Sprint 1 Review | Integration patterns, architecture |

### Critical Sprint 1 Facts for Exam

```python
# The tool_use / tool_result cycle (Sprint 1, Day 6-7)
# This SAME pattern underlies MCP tool calls (Sprint 2)

import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[{
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"}
            },
            "required": ["location"]
        }
    }],
    messages=[{"role": "user", "content": "What's the weather in London?"}]
)

# Claude responds with tool_use → you execute → return tool_result
# MCP automates this same cycle through the protocol layer
```

---

## 7. Anti-Patterns

### ❌ Anti-Pattern 1: Never Iterating (Single-Shot Perfectionism)

```
User: Build a complete e-commerce backend with auth, payments, 
      orders, inventory, and shipping in one shot.
```

**Problem:** Complex systems need iterative development. One-shot attempts produce shallow implementations.

**Fix:** Break into stages. Build auth first, refine it, then move to orders.

### ❌ Anti-Pattern 2: Resuming When Context is Stale

```bash
# Session from 2 weeks ago — codebase has changed significantly
$ claude --resume
> Continue building the API from where we left off
```

**Problem:** Claude's memory of file contents is outdated. It may reference code that no longer exists.

**Fix:** Start fresh when the codebase has significantly changed. Reference specific current files.

### ❌ Anti-Pattern 3: Forking Without Purpose

```
> fork_session
> fork_session
> fork_session
# Three forks with no clear exploration goal
```

**Fix:** Fork with intent: "Fork A: explore Redis caching. Fork B: explore CDN approach."

### ❌ Anti-Pattern 4: Refining Infinitely Without Shipping

```
Round 1: Generate → Refine
Round 2: Refine → Refine
Round 3: Refine → Refine
Round 8: Still refining...
```

**Fix:** Set acceptance criteria upfront. "Done when: tests pass, types check, no linter errors."

### ❌ Anti-Pattern 5: Using --resume to Avoid Explaining Context

```bash
$ claude --resume
> Fix the bug
# Claude has 50+ turns of history but the "bug" was reported externally
```

**Fix:** Even with --resume, provide specific context: "The /orders endpoint returns 500 when quantity is 0. Fix the validation in orders.service.ts."

### ❌ Anti-Pattern 6: Mixing Concerns Across Refinement Rounds

```
Round 1: "Generate the API endpoint"
Round 2: "Also fix the unrelated CSS bug on the homepage"
Round 3: "And update the README"
```

**Fix:** Keep refinement rounds focused on the same topic. Separate concerns into separate sessions or commands.

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│       ITERATIVE REFINEMENT & SESSION MANAGEMENT                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  REFINEMENT PATTERN:                                              │
│    Prompt → Output → Review → Refine → ... → Accept              │
│                                                                    │
│  CLI FLAGS:                                                        │
│    --resume          Continue previous session (linear)           │
│    -p / --print      Output only, no side effects                │
│                                                                    │
│  SESSION TOOLS:                                                    │
│    fork_session      Branch conversation (exploration)            │
│    --resume          Continue conversation (continuation)         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  FULL WORKFLOW (PLAN + ITERATIVE):                                │
│                                                                    │
│  Prompt → Plan → Refine Plan → Approve →                         │
│           Execute → Refine Code → Accept                          │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  SPRINT 1 → SPRINT 2 BRIDGE:                                      │
│                                                                    │
│  API tools[]    ──maps-to──▶  MCP tool definitions               │
│  tool_use       ──maps-to──▶  MCP tool call                      │
│  tool_result    ──maps-to──▶  MCP tool response                  │
│  JSON Schema    ──shared──▶   Same in both layers                │
│                                                                    │
│  KEY DIFFERENCE:                                                   │
│    API = per-request, client-executed                             │
│    MCP = server-registered, protocol-routed                      │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  DECISION: --resume vs NEW SESSION vs fork_session               │
│                                                                    │
│  Same topic, next day?        → --resume                         │
│  Exploring alternatives?      → fork_session                     │
│  Unrelated new task?          → New session                      │
│  Codebase changed a lot?      → New session                      │
│  Context window getting full? → New session (summarize first)    │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Iterative refinement = multiple rounds toward quality         │
│  • --resume preserves FULL context (both good and bad)           │
│  • fork_session = branching (like git branch for conversations)  │
│  • Plan refinement happens BEFORE execution                      │
│  • Code refinement happens AFTER execution                       │
│  • API tool pattern is the FOUNDATION for MCP tool calls         │
│  • MCP adds: discovery, transport, server lifecycle              │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You are building a complex data pipeline feature over 3 days. The feature requires:
- Day 1: Design the pipeline architecture
- Day 2: Implement the core transformation logic
- Day 3: Add error handling and monitoring

During Day 2, you realize there are two viable approaches for the transformation:
- Approach A: Streaming with backpressure
- Approach B: Batch with parallel workers

You want to explore both before committing.

### Question

**What is the OPTIMAL sequence of Claude Code features to use across these 3 days?**

**A)**
```
Day 1: New session with plan mode → Design architecture
Day 2: New session → Implement (forget Day 1 context)
Day 3: New session → Add error handling (forget Day 1-2 context)
```

**B)**
```
Day 1: New session with plan mode → Design architecture
Day 2: --resume → fork_session (Approach A) + fork_session (Approach B) → Pick winner
Day 3: --resume (winning fork) → Add error handling with iterative refinement
```

**C)**
```
Day 1: Direct execution → Build everything at once
Day 2: --resume → Fix all the issues from Day 1
Day 3: --resume → Fix more issues
```

**D)**
```
Day 1: Plan mode → Architecture
Day 2: fork_session → Explore both approaches
Day 3: Start fresh session → Re-implement winning approach from scratch
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** uses the full toolkit optimally: plan mode for architecture (safety), --resume to maintain context across days, fork_session to explore alternatives without losing progress, and iterative refinement for error handling polish.
- **Option A** loses context between days — Claude won't remember architectural decisions from Day 1.
- **Option C** skips plan mode for a complex design (risky) and treats each day as a bug-fix pass rather than a structured workflow.
- **Option D** is mostly correct but wastes effort on Day 3 by starting fresh — --resume on the winning fork preserves all context and refinements.

**Key Exam Principle:** Use `--resume` for continuity, `fork_session` for exploration, plan mode for high-stakes design, and iterative refinement for progressive quality improvement. These features compose into a powerful development workflow.

---

## Summary: Key Exam Takeaways

1. Iterative refinement: **Prompt → Output → Review → Refine → Accept** (multiple rounds)
2. `--resume` continues a session preserving full context — use for multi-day work on same feature
3. `fork_session` branches conversations — use to explore alternatives safely
4. Plan + iterative refinement = two-level quality loop (refine plan, then refine code)
5. Sprint 1 API tools map directly to MCP tools: same JSON Schema, different registration/transport
6. API = per-request, client-executed; MCP = server-registered, protocol-routed
7. Choose --resume vs fresh session based on context relevance and codebase staleness
8. Set acceptance criteria early to avoid infinite refinement loops
9. Keep refinement rounds focused on one concern — don't mix unrelated changes

---

*End of Day 20 Study Material*
