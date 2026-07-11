# Day 34: Coordinator-Subagent Architecture

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Begin Introduction to Subagents |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [The Coordinator-Subagent Pattern](#1-the-coordinator-subagent-pattern)
2. [Coordinator Responsibilities](#2-coordinator-responsibilities)
3. [Subagent Responsibilities](#3-subagent-responsibilities)
4. [Communication Patterns](#4-communication-patterns)
5. [When to Use Coordinator-Subagent vs Single Agent](#5-when-to-use-coordinator-subagent-vs-single-agent)
6. [Designing Subagent Boundaries](#6-designing-subagent-boundaries)
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

## 1. The Coordinator-Subagent Pattern

### Core Concept

The coordinator-subagent pattern is a **hierarchical architecture** where a central coordinator agent delegates work to specialized subagents, then synthesizes their results.

```
┌─────────────────────────────────────────────────────────┐
│                     USER REQUEST                         │
│          "Build a user registration feature"            │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    COORDINATOR                           │
│   • Analyzes the request                                │
│   • Plans decomposition                                 │
│   • Delegates to specialists                            │
│   • Aggregates results                                  │
│   • Synthesizes final answer                            │
└──────┬──────────────┬──────────────────┬────────────────┘
       │              │                  │
       ▼              ▼                  ▼
┌────────────┐ ┌────────────┐ ┌─────────────────┐
│ SUBAGENT 1 │ │ SUBAGENT 2 │ │   SUBAGENT 3    │
│ Backend    │ │ Frontend   │ │   Testing       │
│ Developer  │ │ Developer  │ │   Specialist    │
├────────────┤ ├────────────┤ ├─────────────────┤
│ Tools:     │ │ Tools:     │ │ Tools:          │
│ write_file │ │ write_file │ │ run_tests       │
│ read_file  │ │ read_file  │ │ read_file       │
│ run_sql    │ │ grep_search│ │ grep_search     │
└────────────┘ └────────────┘ └─────────────────┘
```

### Why This Pattern Exists

| Problem with Single Agent | Coordinator-Subagent Solution |
|--------------------------|-------------------------------|
| Context window overflows on large tasks | Each subagent gets focused context |
| One agent needs too many tools | Each subagent gets only its tools |
| No separation of concerns | Clear responsibility boundaries |
| Single point of failure | Isolated failures per subagent |
| Hard to add new capabilities | Add new subagent types modularly |

---

## 2. Coordinator Responsibilities

### The Coordinator's Five Jobs

```python
class CoordinatorAgent:
    """The coordinator has exactly five responsibilities."""
    
    def analyze_request(self, user_request: str) -> TaskPlan:
        """1. TASK ROUTING: Understand what needs to be done."""
        # Parse the request, identify required skills
        pass
    
    def plan_decomposition(self, task_plan: TaskPlan) -> list[Subtask]:
        """2. PLANNING: Break work into delegatable units."""
        # Decide what to delegate, in what order
        pass
    
    def delegate_to_subagent(self, subtask: Subtask) -> str:
        """3. DELEGATION: Spawn subagent with correct instructions."""
        # Use Task tool with appropriate instructions + allowed_tools
        pass
    
    def handle_errors(self, error: SubagentError) -> Recovery:
        """4. ERROR HANDLING: Recover from subagent failures."""
        # Retry, reassign, or escalate
        pass
    
    def synthesize_results(self, results: list[str]) -> str:
        """5. SYNTHESIS: Combine subagent outputs into coherent response."""
        # Aggregate, resolve conflicts, format final answer
        pass
```

### Coordinator Implementation

```python
from claude_code_sdk import Agent, Task

coordinator = Agent(
    name="project-coordinator",
    model="claude-sonnet-4-20250514",
    instructions="""You are a project coordinator. Your job is to:
    1. Analyze user requests and identify required work streams
    2. Delegate to specialized subagents using the Task tool
    3. Synthesize results into a coherent deliverable
    
    RULES:
    - Never do implementation work yourself
    - Always explain your delegation plan before acting
    - If a subagent fails, try a different approach before escalating
    - Resolve conflicts between subagent outputs
    """,
    tools=[Task, "read_file", "list_directory"],  # Coordinator reads, delegates
    max_turns=30  # Higher limit — coordination takes more turns
)
```

### What the Coordinator Should NOT Do

| Should Do | Should NOT Do |
|-----------|---------------|
| Plan and delegate | Write code directly |
| Synthesize results | Execute low-level tools |
| Handle cross-cutting concerns | Micromanage subagent steps |
| Make architectural decisions | Duplicate subagent work |
| Track progress | Do work a subagent should do |

---

## 3. Subagent Responsibilities

### The Subagent Contract

Each subagent has a clear, narrow responsibility:

```python
# Subagent: focused execution with restricted tools
backend_subagent = Task(
    description="Implement the user registration API endpoint",
    instructions="""
    Create a POST /api/users/register endpoint that:
    - Accepts email, password, and name
    - Validates input (email format, password strength)
    - Hashes password with bcrypt
    - Stores user in database
    - Returns 201 with user ID
    
    Use the existing patterns in src/api/ for consistency.
    """,
    allowed_tools=["read_file", "write_file", "grep_search"],
    context="Project uses Express.js + PostgreSQL + bcrypt"
)
```

### Subagent Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Single Responsibility** | One domain, one deliverable |
| **Least Privilege** | Only the tools it needs |
| **Self-Contained** | Doesn't need to call other subagents |
| **Clear Output** | Returns structured, usable result |
| **Failure Transparency** | Reports errors clearly to coordinator |

### Subagent Types by Specialty

```
┌─────────────────────────────────────────────────────────┐
│              COMMON SUBAGENT SPECIALIZATIONS             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  🔍 ANALYST         → read_file, grep_search            │
│     Analyzes code, identifies patterns, reports         │
│                                                         │
│  ✍️  IMPLEMENTER    → read_file, write_file             │
│     Writes new code based on specifications             │
│                                                         │
│  🧪 TESTER          → read_file, run_tests              │
│     Writes and runs tests, reports coverage             │
│                                                         │
│  🔒 SECURITY        → read_file, grep_search            │
│     Audits code for vulnerabilities                     │
│                                                         │
│  📝 DOCUMENTER      → read_file, write_file             │
│     Generates documentation from code                   │
│                                                         │
│  🚀 DEPLOYER        → deploy, check_status              │
│     Handles deployment operations (with approval)       │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Communication Patterns

### Coordinator → Subagent Communication

```python
# The coordinator communicates via Task parameters:
Task(
    description="...",       # WHAT to do (brief)
    instructions="...",      # HOW to do it (detailed)
    context="...",           # WHAT CAME BEFORE (prior results)
    allowed_tools=[...]      # WHAT YOU CAN USE (constraints)
)
```

### Subagent → Coordinator Communication

The subagent communicates back through its **return value** — the final text response when the subtask's agentic loop completes (stop_reason: "end_turn").

```python
# Subagent's final response becomes the tool_result for the coordinator
# Good subagent output: structured, actionable
"""
## Analysis Complete

### Findings:
1. SQL injection in user_query() at line 42 — uses string concatenation
2. Missing rate limiting on /api/login endpoint
3. Session tokens not rotated after password change

### Severity: HIGH (finding 1), MEDIUM (findings 2-3)

### Recommended Fixes:
1. Use parameterized queries: cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
2. Add rate limiter middleware to /api/login
3. Invalidate all sessions on password change
"""
```

### Communication Quality Rules

| Good Communication | Bad Communication |
|-------------------|-------------------|
| Structured output with clear sections | Wall of unformatted text |
| Actionable recommendations | Vague observations |
| Specific file paths and line numbers | "Somewhere in the code..." |
| Explicit severity/priority | No prioritization |
| Clear success/failure status | Ambiguous completion state |

---

## 5. When to Use Coordinator-Subagent vs Single Agent

### Decision Matrix

| Factor | Single Agent | Coordinator-Subagent |
|--------|-------------|---------------------|
| Task complexity | Simple, focused | Multi-faceted, large |
| Tools needed | 1-5 tools | 5+ tools across domains |
| Context size | Fits in one window | Would overflow |
| Skill diversity | One domain | Multiple domains |
| Error tolerance | All-or-nothing fine | Need partial success |
| Team mental model | One person's job | Team of specialists |

### Decision Flowchart

```
┌─────────────────────────────────────────┐
│  Can one agent with ~5 tools handle it? │
├──────────────┬──────────────────────────┘
│              │
│  YES         │  NO
│              │
▼              ▼
┌──────┐    ┌──────────────────────────────┐
│Single│    │ Does it need multiple domains │
│Agent │    │ of expertise?                 │
└──────┘    ├──────────────┬───────────────┘
            │              │
            │  YES         │  NO
            │              │
            ▼              ▼
  ┌────────────────┐   ┌───────────────────┐
  │ Coordinator +  │   │ Single agent with  │
  │ Specialized    │   │ Task decomposition │
  │ Subagents      │   │ (same-type tasks)  │
  └────────────────┘   └───────────────────┘
```

### Real-World Examples

```python
# ✅ Single Agent: Fix a specific bug
# One domain, few tools, focused context
agent = Agent(
    name="bug-fixer",
    tools=["read_file", "write_file", "run_tests"],
    instructions="Fix the null pointer exception in user_service.py"
)

# ✅ Coordinator-Subagent: Full feature development
# Multiple domains, many tools, large context
coordinator = Agent(
    name="feature-coordinator",
    tools=[Task, "read_file", "list_directory"],
    instructions="Coordinate development of the new payment feature"
)
# Delegates to: backend dev, frontend dev, tester, security reviewer
```

---

## 6. Designing Subagent Boundaries

### Boundary Design Principles

```
Good boundaries follow the same rules as good microservices:
  • High cohesion WITHIN a subagent (related tasks grouped)
  • Low coupling BETWEEN subagents (minimal dependencies)
  • Clear interfaces (well-defined inputs and outputs)
  • Single responsibility (one reason to change)
```

### Boundary Design Process

```python
# Step 1: Identify distinct skill domains
skills_needed = [
    "backend_development",
    "frontend_development", 
    "database_design",
    "security_review",
    "testing"
]

# Step 2: Group related skills into subagent boundaries
subagent_boundaries = {
    "backend_subagent": ["backend_development", "database_design"],
    "frontend_subagent": ["frontend_development"],
    "quality_subagent": ["security_review", "testing"]
}

# Step 3: Assign tools per boundary
subagent_tools = {
    "backend_subagent": ["read_file", "write_file", "run_sql"],
    "frontend_subagent": ["read_file", "write_file"],
    "quality_subagent": ["read_file", "grep_search", "run_tests"]
}
```

### Boundary Anti-Pattern: Overlapping Responsibilities

```
❌ BAD: Two subagents that both modify the same files

Subagent A: "Write the API endpoint"     → writes src/api/users.ts
Subagent B: "Add validation to the API"  → ALSO writes src/api/users.ts

CONFLICT! Who wins? Merge conflicts, lost work.

✅ GOOD: One subagent owns the file

Subagent A: "Write the API endpoint WITH validation"  → owns src/api/users.ts
Subagent B: "Write tests for the API endpoint"        → owns tests/api/users.test.ts
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Coordinator That Micromanages

```python
# ❌ BAD: Coordinator dictates every line of code
Task(
    description="Write the login function",
    instructions="""
    Write exactly this code:
    ```
    def login(email, password):
        user = db.query(f"SELECT * FROM users WHERE email = '{email}'")
        if user.password == password:
            return create_session(user)
    ```
    Do not deviate from this implementation.
    """,
    allowed_tools=["write_file"]
)
# The coordinator is writing the code itself and just using the subagent
# as a typing tool. This defeats the purpose of delegation.
```

### ✅ CORRECT: Coordinator Sets Goals, Subagent Decides How

```python
# ✅ GOOD: Coordinator specifies WHAT, subagent decides HOW
Task(
    description="Implement secure login function",
    instructions="""
    Implement a login function that:
    - Accepts email and password
    - Looks up user in the database securely
    - Verifies password using constant-time comparison
    - Creates and returns a session token
    - Handles invalid credentials gracefully
    
    Follow the existing patterns in src/auth/ for consistency.
    Use parameterized queries for database access.
    """,
    allowed_tools=["read_file", "write_file", "grep_search"]
)
# Subagent has autonomy to implement the best solution
```

### Additional Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Coordinator does implementation | Defeats purpose of delegation | Coordinator plans + delegates only |
| No error handling for failed subagents | Coordinator hangs or crashes | Catch failures, retry or escalate |
| Subagent with ALL tools | No security boundary | Restrict to needed tools only |
| Subagent that calls other subagents (uncontrolled) | Runaway recursion | Limit nesting depth |
| No result validation by coordinator | Bad output propagates | Coordinator validates before synthesis |
| Coordinator passes full conversation | Context overflow in subagent | Pass only relevant summary |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│     COORDINATOR-SUBAGENT ARCHITECTURE — QUICK REF       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  COORDINATOR RESPONSIBILITIES:                          │
│    1. Task routing (understand the request)             │
│    2. Planning (decompose into delegatable units)       │
│    3. Delegation (spawn subagents with Task tool)       │
│    4. Error handling (retry/reassign/escalate)          │
│    5. Synthesis (combine results into final answer)     │
│                                                         │
│  SUBAGENT RESPONSIBILITIES:                             │
│    • Execute focused task with restricted tools         │
│    • Return structured, actionable results              │
│    • Report errors transparently                        │
│                                                         │
│  USE COORDINATOR-SUBAGENT WHEN:                         │
│    • Multiple skill domains needed                      │
│    • Context would overflow single agent                │
│    • Need error isolation                               │
│    • 5+ tools across different domains                  │
│                                                         │
│  USE SINGLE AGENT WHEN:                                 │
│    • Task is focused and simple                         │
│    • 1-5 tools in same domain                           │
│    • Fits in one context window                         │
│                                                         │
│  BOUNDARY DESIGN:                                       │
│    • High cohesion within subagent                      │
│    • Low coupling between subagents                     │
│    • No overlapping file ownership                      │
│    • Clear input/output contracts                       │
│                                                         │
│  KEY ANTI-PATTERN: Coordinator that micromanages        │
│  (specifies HOW instead of WHAT)                        │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You're architecting a system to handle customer support tickets. Tickets require: (1) classification by category, (2) sentiment analysis, (3) response drafting, and (4) routing to the appropriate department.

**Question 1:** Which architecture is most appropriate?

- A) Single agent with all four capabilities
- B) Coordinator with four specialized subagents
- C) Four independent agents with no coordination
- D) Coordinator with two subagents (analysis + action)

**Answer:** **D** — While B seems logical, the four tasks naturally group into two domains: analysis (classification + sentiment) and action (response + routing). Sentiment and classification are related analysis tasks that share context. Response drafting and routing are related action tasks. Over-decomposing into four subagents adds unnecessary coordination overhead.

---

**Question 2:** What is the coordinator's role when a subagent fails?

- A) Immediately return an error to the user
- B) Retry the same subagent with the same instructions
- C) Assess the failure, then retry with adjusted instructions or escalate
- D) Skip the failed task and continue with remaining subagents

**Answer:** **C** — The coordinator should intelligently handle failures: assess what went wrong, decide whether to retry with better instructions, try a different approach, or escalate to the user. Immediate failure (A) wastes the other work. Blind retry (B) may repeat the same error. Skipping (D) may produce incomplete results.

---

**Question 3:** What distinguishes a micromanaging coordinator from a properly delegating one?

- A) Micromanaging coordinators use more API calls
- B) Proper coordinators specify WHAT to achieve; micromanagers specify HOW to implement
- C) Micromanaging coordinators use fewer subagents
- D) Proper coordinators never provide context to subagents

**Answer:** **B** — The key distinction is WHAT vs HOW. A proper coordinator says "implement secure login" (goal). A micromanager says "write exactly this code" (implementation). The subagent should have autonomy to decide the best implementation approach.

---

**Question 4:** A subagent needs to read files from the project and also write new test files. What's the correct allowedTools?

- A) `["read_file", "write_file", "grep_search", "delete_file", "deploy"]`
- B) `["read_file", "write_file"]`
- C) `["read_file", "write_file", "grep_search", "run_tests"]`
- D) All tools — let the subagent decide what it needs

**Answer:** **C** — The subagent needs to read existing code (read_file), find patterns (grep_search), write test files (write_file), and verify tests pass (run_tests). It doesn't need delete_file or deploy. This follows the principle of least privilege while giving the subagent everything it needs for its job.
