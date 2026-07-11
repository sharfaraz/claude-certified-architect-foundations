# Day 33: Task Decomposition Strategies

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Complete Introduction to Agent Skills |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Task Decomposition Overview](#1-task-decomposition-overview)
2. [The Task Tool in Claude Agent SDK](#2-the-task-tool-in-claude-agent-sdk)
3. [Decomposition Patterns](#3-decomposition-patterns)
4. [Deciding Granularity](#4-deciding-granularity)
5. [allowedTools: Restricting Subtask Access](#5-allowedtools-restricting-subtask-access)
6. [Mapping Strategies to Exam Scenarios](#6-mapping-strategies-to-exam-scenarios)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude SDK: Subagents | https://code.claude.com/docs/en/sdk/subagents |
| Agentic Tool Use | https://docs.anthropic.com/en/docs/build-with-claude/agentic-tool-use |

---

## 1. Task Decomposition Overview

### What Is Task Decomposition?

Task decomposition is the process of **breaking a complex goal into smaller, manageable subtasks** that can be executed independently or in sequence.

```
Complex Goal: "Refactor the authentication module"
    │
    ├── Subtask 1: Analyze current auth code structure
    ├── Subtask 2: Identify security vulnerabilities
    ├── Subtask 3: Design new interface
    ├── Subtask 4: Implement changes
    └── Subtask 5: Write tests for new code
```

### Why Decomposition Matters for Agents

| Benefit | Explanation |
|---------|-------------|
| **Focus** | Each subtask has a narrow, achievable objective |
| **Tool Scoping** | Subtasks only need specific tools (least privilege) |
| **Error Isolation** | Failure in one subtask doesn't corrupt the whole task |
| **Parallelization** | Independent subtasks can run concurrently |
| **Cost Control** | Smaller context windows per subtask = fewer tokens |

### The Decomposition Decision

```
┌─────────────────────────────────────────────────┐
│  Should I decompose this task?                   │
│                                                  │
│  YES if:                                         │
│    • Task requires multiple distinct skills      │
│    • Task has independent sub-problems           │
│    • Context window would overflow               │
│    • Different tools needed for different parts   │
│    • Failure in one part shouldn't block others   │
│                                                  │
│  NO if:                                          │
│    • Task is atomic (one tool call suffices)      │
│    • Subtasks are tightly coupled                 │
│    • Overhead of coordination > complexity saved  │
│    • Task requires holistic understanding         │
└─────────────────────────────────────────────────┘
```

---

## 2. The Task Tool in Claude Agent SDK

### Spawning Subtasks

The Claude Agent SDK provides a `Task` tool that spawns subtasks — each running as a mini-agent with its own instructions, tools, and context.

```python
from claude_code_sdk import Agent, Task

# Define a coordinator agent that can spawn subtasks
coordinator = Agent(
    name="project-coordinator",
    model="claude-sonnet-4-20250514",
    instructions="""You coordinate complex development tasks.
    Break work into subtasks using the Task tool.
    Synthesize results from completed subtasks.""",
    tools=[
        Task,  # The built-in subtask spawning tool
        "read_file",
        "list_directory"
    ],
    max_turns=20
)
```

### Task Tool Parameters

```python
# When the coordinator calls the Task tool, it specifies:
Task(
    description="Analyze auth.py for security vulnerabilities",
    instructions="Focus on SQL injection, XSS, and auth bypass patterns. "
                 "Report findings as a structured list.",
    allowed_tools=["read_file", "grep_search"],  # Restricted tool access
    context="The project uses Flask with SQLAlchemy ORM"  # Passed context
)
```

### Task Tool Schema

| Parameter | Type | Required | Purpose |
|-----------|------|----------|---------|
| `description` | string | Yes | Short summary of the subtask |
| `instructions` | string | Yes | Detailed instructions for the subtask |
| `allowed_tools` | list[str] | No | Tools the subtask can use (default: inherit all) |
| `context` | string | No | Additional context passed to the subtask |

### How It Works Internally

```
Coordinator Agent
    │
    │ calls Task(description="...", instructions="...", allowed_tools=[...])
    │
    ▼
┌─────────────────────────────────────┐
│  Subtask Agent (spawned)            │
│  • Gets its own conversation        │
│  • Has restricted tool access       │
│  • Runs its own agentic loop        │
│  • Returns result to coordinator    │
└─────────────────────────────────────┘
    │
    │ result string returned
    │
    ▼
Coordinator receives result, continues reasoning
```

---

## 3. Decomposition Patterns

### Pattern 1: Sequential Decomposition

Subtasks depend on each other — each uses the output of the previous one.

```
Task A → result → Task B → result → Task C → final result
```

```python
# Sequential: Each step builds on the previous
# The coordinator naturally does this by calling Task one at a time

# Step 1: Analyze
analysis = Task(
    description="Analyze the existing API structure",
    instructions="Map all endpoints, their methods, and response formats.",
    allowed_tools=["read_file", "grep_search", "list_directory"]
)

# Step 2: Design (depends on analysis)
design = Task(
    description="Design the new API version based on analysis",
    instructions=f"Given this analysis: {analysis.result}, design v2 endpoints.",
    allowed_tools=["read_file"]
)

# Step 3: Implement (depends on design)
implementation = Task(
    description="Implement the new API design",
    instructions=f"Implement this design: {design.result}",
    allowed_tools=["read_file", "write_file", "run_tests"]
)
```

### Pattern 2: Parallel Decomposition

Subtasks are independent — can run simultaneously.

```
         ┌── Task A ──┐
         │             │
Start ───┼── Task B ──┼──▶ Aggregate Results
         │             │
         └── Task C ──┘
```

```python
# Parallel: Independent subtasks that don't depend on each other
# The coordinator can spawn multiple Tasks and aggregate results

tasks = [
    Task(
        description="Review frontend code for accessibility",
        instructions="Check ARIA labels, keyboard nav, color contrast.",
        allowed_tools=["read_file", "grep_search"]
    ),
    Task(
        description="Review backend code for security",
        instructions="Check auth, input validation, SQL injection.",
        allowed_tools=["read_file", "grep_search"]
    ),
    Task(
        description="Review infrastructure for reliability",
        instructions="Check error handling, retries, timeouts.",
        allowed_tools=["read_file", "grep_search"]
    )
]
# Coordinator aggregates all results into a unified report
```

### Pattern 3: Hierarchical Decomposition

Subtasks themselves decompose into sub-subtasks.

```
Main Task
├── Subtask 1 (Coordinator)
│   ├── Sub-subtask 1.1
│   └── Sub-subtask 1.2
├── Subtask 2 (Simple)
└── Subtask 3 (Coordinator)
    ├── Sub-subtask 3.1
    └── Sub-subtask 3.2
```

```python
# Hierarchical: Subtask coordinators that further decompose
coordinator = Agent(
    name="project-lead",
    tools=[Task, "read_file"],
    instructions="You manage a large refactoring project. "
                 "Delegate to sub-coordinators for each module."
)

# The coordinator spawns a subtask that ITSELF can spawn subtasks
module_refactor = Task(
    description="Refactor the payments module",
    instructions="You are a sub-coordinator. Break this into: "
                 "1) Analyze current payment code, "
                 "2) Implement new payment processor, "
                 "3) Migrate existing integrations. "
                 "Use the Task tool for each sub-step.",
    allowed_tools=[Task, "read_file", "write_file", "run_tests"]
)
```

---

## 4. Deciding Granularity

### The Granularity Spectrum

```
Too Coarse                                          Too Fine
│                                                        │
│  "Refactor entire app"     "Write line 42 of auth.py" │
│  (overwhelming, no focus)   (overhead > value)         │
│                                                        │
│              ◄── SWEET SPOT ──►                        │
│     "Refactor the auth module"                         │
│     "Add input validation to /users endpoint"          │
│     "Write unit tests for PaymentService"              │
└────────────────────────────────────────────────────────┘
```

### Decision Framework

| Factor | Decompose Further | Keep as Single Task |
|--------|-------------------|---------------------|
| Tool diversity | Needs 5+ different tools | Needs 1-2 tools |
| Context size | Would exceed context window | Fits in one context |
| Failure impact | Partial failure is acceptable | All-or-nothing needed |
| Skill diversity | Requires different "expertise" | Single domain |
| Duration | Would take 20+ turns | Completable in 5-10 turns |

### Granularity Rules of Thumb

```python
# Rule 1: One subtask = one clear deliverable
# ✅ Good: "Generate unit tests for UserService"
# ❌ Bad: "Do testing stuff"

# Rule 2: Subtask should be completable in 5-15 turns
# ✅ Good: "Analyze auth.py for SQL injection vulnerabilities"
# ❌ Bad: "Make the app secure" (too broad)
# ❌ Bad: "Check if line 5 has a semicolon" (too narrow)

# Rule 3: If in doubt, start coarse and decompose only if needed
# Let the coordinator decide to decompose based on initial analysis
```

---

## 5. allowedTools: Restricting Subtask Access

### Principle of Least Privilege

Each subtask should only have access to the tools it actually needs.

```python
# ❌ BAD: Give every subtask access to everything
Task(
    description="Read the config file",
    allowed_tools=["read_file", "write_file", "delete_file", 
                   "run_sql", "deploy", "send_email"]  # Way too many!
)

# ✅ GOOD: Only the tools needed for THIS subtask
Task(
    description="Read the config file",
    allowed_tools=["read_file"]  # That's all it needs
)
```

### Tool Restriction Strategy

| Subtask Type | Recommended allowed_tools |
|-------------|--------------------------|
| Analysis/Read | `["read_file", "grep_search", "list_directory"]` |
| Code Writing | `["read_file", "write_file", "grep_search"]` |
| Testing | `["read_file", "run_tests"]` |
| Deployment | `["deploy", "check_status"]` (with approval hook!) |
| Research | `["web_search", "read_file"]` |

### Security Benefits

```
Full Agent (all tools):
  ├── write_file    ← Can modify code
  ├── delete_file   ← Can destroy data
  ├── run_sql       ← Can access database
  ├── deploy        ← Can push to production
  └── send_email    ← Can communicate externally

Analysis Subtask (restricted):
  ├── read_file     ← Can only READ
  └── grep_search   ← Can only SEARCH

If the analysis subtask is "jailbroken" or confused,
it literally CANNOT write, delete, or deploy.
```

---

## 6. Mapping Strategies to Exam Scenarios

### Scenario-to-Pattern Mapping

| Exam Scenario | Best Pattern | Why |
|---------------|-------------|-----|
| "Review code across 5 microservices" | Parallel | Each service is independent |
| "Migrate database schema then update code" | Sequential | Code depends on schema |
| "Build a full-stack feature" | Hierarchical | Frontend + backend + tests |
| "Fix a single bug" | No decomposition | Atomic task |
| "Audit security + performance + accessibility" | Parallel | Independent concerns |
| "Refactor then test then deploy" | Sequential | Each depends on previous |

### Exam Decision Tree

```
Is the task complex?
├── NO → Single agent, no decomposition
└── YES → Do subtasks depend on each other?
    ├── YES (all sequential) → Sequential pattern
    ├── NO (all independent) → Parallel pattern
    └── MIXED → Hierarchical or hybrid pattern
        └── Are there natural groupings?
            ├── YES → Sub-coordinators per group
            └── NO → Flat parallel with final aggregation
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Over-Decomposition

```python
# ❌ BAD: Decomposing trivial tasks
Task(description="Read line 1 of config.json", allowed_tools=["read_file"])
Task(description="Read line 2 of config.json", allowed_tools=["read_file"])
Task(description="Read line 3 of config.json", allowed_tools=["read_file"])
# Coordination overhead >> task complexity

# ✅ GOOD: One task for the whole file
Task(description="Analyze config.json structure", allowed_tools=["read_file"])
```

### ❌ ANTI-PATTERN: No Context Passing

```python
# ❌ BAD: Subtask has no context about what came before
Task(
    description="Implement the fix",
    instructions="Fix the bug.",  # What bug?? No context!
    allowed_tools=["read_file", "write_file"]
)

# ✅ GOOD: Pass relevant context from prior steps
Task(
    description="Implement the auth bypass fix",
    instructions=f"The analysis found a session validation bug in auth.py "
                 f"at line 42. The session token is not verified against the "
                 f"user's stored token. Fix by adding token comparison.",
    allowed_tools=["read_file", "write_file"]
)
```

### Anti-Pattern Summary

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Over-decomposition | Overhead exceeds benefit | Keep atomic tasks as single calls |
| Under-decomposition | Context overflow, unfocused | Break at natural skill boundaries |
| No context passing | Subtask can't do its job | Always include prior step results |
| All tools everywhere | Security risk | Use allowedTools restrictively |
| Deep nesting (>3 levels) | Coordination complexity explodes | Flatten to 2 levels max |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│       TASK DECOMPOSITION — QUICK REFERENCE              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  PATTERNS:                                              │
│    Sequential:   A → B → C (each depends on prior)     │
│    Parallel:     A | B | C (all independent)            │
│    Hierarchical: A spawns (A1, A2), B spawns (B1, B2)  │
│                                                         │
│  TASK TOOL PARAMS:                                      │
│    description    → What the subtask does               │
│    instructions   → Detailed how-to                     │
│    allowed_tools  → Principle of least privilege         │
│    context        → Info from prior steps               │
│                                                         │
│  GRANULARITY SWEET SPOT:                                │
│    • One clear deliverable per subtask                  │
│    • Completable in 5-15 turns                          │
│    • Needs 1-3 tools (not 10)                           │
│                                                         │
│  DECOMPOSE WHEN:                                        │
│    • Multiple distinct skills needed                    │
│    • Context would overflow                             │
│    • Different tool sets per part                       │
│    • Failure isolation needed                           │
│                                                         │
│  DON'T DECOMPOSE WHEN:                                  │
│    • Task is atomic                                     │
│    • Overhead > complexity saved                        │
│    • Tight coupling between parts                       │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You need to build an agent that reviews a pull request containing changes across 3 microservices (auth-service, payment-service, notification-service). The review should check code quality, security, and test coverage for each service.

**Question 1:** What decomposition pattern best fits this task?

- A) Sequential — review auth first, then payment, then notification
- B) Parallel — review all three services simultaneously
- C) Hierarchical — one coordinator per service, each decomposing into quality/security/tests
- D) No decomposition — single agent reviews everything

**Answer:** **C** — Hierarchical is best here. Each service is independent (parallel at the top level), but within each service, you need different expertise (code quality, security, test coverage). A sub-coordinator per service can decompose into these three aspects. This gives both parallelism AND focused expertise.

---

**Question 2:** A subtask for "analyzing code security" should have which allowed_tools?

- A) `["read_file", "write_file", "grep_search", "deploy"]`
- B) `["read_file", "grep_search", "list_directory"]`
- C) All tools — the subtask might need anything
- D) `["write_file", "run_tests"]`

**Answer:** **B** — Security analysis is a read-only operation. It needs to read files, search for patterns, and list directory structures. It should NOT have write_file (it's not making changes), deploy (irrelevant), or run_tests (that's a separate concern). Principle of least privilege.

---

**Question 3:** When is over-decomposition harmful?

- A) When you have many independent subtasks
- B) When coordination overhead exceeds the complexity saved
- C) When subtasks need different tools
- D) When the main task is time-sensitive

**Answer:** **B** — Over-decomposition becomes harmful when the cost of spawning, coordinating, and aggregating subtasks exceeds the benefit of breaking work apart. If a task can be done in 3 tool calls, spawning 3 separate subtasks (each with their own setup, context, and communication overhead) is wasteful.

---

**Question 4:** What must you always include when spawning a subtask that depends on prior work?

- A) The full conversation history
- B) All available tools
- C) Relevant context from prior steps
- D) A timeout value

**Answer:** **C** — Context from prior steps is essential. Without it, the subtask operates blind. You don't need the full conversation (too much noise) or all tools (violates least privilege). Timeouts are good practice but not the "must always include" answer when building on prior work.
