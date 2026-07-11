# Day 35: Multi-Agent Orchestration Patterns

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Continue Introduction to Subagents |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Orchestration Topologies Overview](#1-orchestration-topologies-overview)
2. [Sequential Pipeline (A→B→C)](#2-sequential-pipeline-abc)
3. [Parallel Fan-Out/Fan-In](#3-parallel-fan-outfan-in)
4. [Hierarchical Delegation](#4-hierarchical-delegation)
5. [Choosing the Right Topology](#5-choosing-the-right-topology)
6. [Error Handling When One Subagent Fails](#6-error-handling-when-one-subagent-fails)
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

## 1. Orchestration Topologies Overview

### The Three Core Topologies

```
┌─────────────────────────────────────────────────────────────────┐
│                  ORCHESTRATION TOPOLOGIES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. SEQUENTIAL PIPELINE         2. PARALLEL FAN-OUT/FAN-IN      │
│                                                                  │
│     A → B → C → D                    ┌── B ──┐                  │
│                                  A ──┼── C ──┼── E              │
│  Each step feeds next               └── D ──┘                  │
│                                                                  │
│  3. HIERARCHICAL DELEGATION                                      │
│                                                                  │
│          Coordinator                                             │
│         /     |     \                                            │
│     Sub-Coord  Agent  Sub-Coord                                  │
│      /    \             /    \                                    │
│   Agent  Agent       Agent  Agent                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Topology Comparison

| Topology | Latency | Complexity | Error Impact | Best For |
|----------|---------|-----------|--------------|----------|
| Sequential | High (sum of all steps) | Low | Blocks entire pipeline | Dependent transformations |
| Parallel | Low (max of parallel steps) | Medium | One failure, others continue | Independent analyses |
| Hierarchical | Medium | High | Isolated per branch | Complex multi-domain work |

---

## 2. Sequential Pipeline (A→B→C)

### When to Use Sequential

Use when **each step depends on the output of the previous step**.

```
Research → Analyze → Design → Implement → Test
   │          │         │         │         │
   │          │         │         │         └── Needs implementation
   │          │         │         └── Needs design spec
   │          │         └── Needs analysis results
   │          └── Needs research data
   └── Independent starting point
```

### Implementation

```python
from claude_code_sdk import Agent, Task

coordinator = Agent(
    name="pipeline-coordinator",
    model="claude-sonnet-4-20250514",
    instructions="""Execute tasks sequentially. Each step's output 
    feeds into the next step as context. Validate each step's output 
    before proceeding.""",
    tools=[Task, "read_file"],
    max_turns=40
)

# The coordinator executes this pipeline naturally:
# Turn 1: Spawn research task
# Turn 2: Validate research output, spawn analysis task with research as context
# Turn 3: Validate analysis output, spawn design task with analysis as context
# Turn 4: Validate design output, spawn implementation task
# Turn 5: Validate implementation, spawn testing task
```

### Sequential Pipeline Code Pattern

```python
async def sequential_pipeline(coordinator, steps: list[dict]):
    """Execute steps sequentially, passing results forward."""
    accumulated_context = ""
    
    for i, step in enumerate(steps):
        result = await coordinator.run_task(
            Task(
                description=step["description"],
                instructions=step["instructions"],
                context=accumulated_context,  # Prior results feed forward
                allowed_tools=step["tools"]
            )
        )
        
        # Validate step output
        if not validate_output(result, step.get("expected_format")):
            raise StepValidationError(f"Step {i} output invalid: {result[:100]}")
        
        # Accumulate context for next step
        accumulated_context += f"\n\n## Step {i+1} Result:\n{result}"
    
    return accumulated_context

# Usage
steps = [
    {
        "description": "Research current auth patterns",
        "instructions": "Survey existing auth implementations in src/",
        "tools": ["read_file", "grep_search", "list_directory"]
    },
    {
        "description": "Design new auth module",
        "instructions": "Based on research, design improved auth with OAuth2",
        "tools": ["read_file"]
    },
    {
        "description": "Implement the design",
        "instructions": "Implement the auth design from prior step",
        "tools": ["read_file", "write_file"]
    }
]
```

### Sequential Pipeline Characteristics

| Characteristic | Detail |
|---------------|--------|
| Data flow | Unidirectional: left to right |
| Failure mode | Pipeline halts at failed step |
| Latency | Sum of all step durations |
| Context growth | Accumulates with each step |
| Debugging | Easy — trace through steps in order |

---

## 3. Parallel Fan-Out/Fan-In

### When to Use Parallel

Use when **tasks are independent** and results can be aggregated after all complete.

```
                    ┌── Security Review ──┐
                    │                     │
User Request ──────┼── Performance Audit ─┼──── Aggregated Report
                    │                     │
                    └── Code Quality ─────┘
         
         FAN-OUT                    FAN-IN
    (dispatch tasks)          (aggregate results)
```

### Implementation

```python
coordinator = Agent(
    name="parallel-coordinator",
    model="claude-sonnet-4-20250514",
    instructions="""You run independent analysis tasks in parallel.
    After all complete, synthesize findings into a unified report.
    
    IMPORTANT: Dispatch all independent tasks in one turn, then
    aggregate results in the next turn.""",
    tools=[Task, "read_file"],
    max_turns=20
)

# Fan-out: Coordinator dispatches multiple tasks
# (Claude Agent SDK handles parallel execution internally)
parallel_tasks = [
    Task(
        description="Security audit of payment module",
        instructions="Check for injection, auth bypass, data exposure.",
        allowed_tools=["read_file", "grep_search"]
    ),
    Task(
        description="Performance analysis of payment module",
        instructions="Check for N+1 queries, unnecessary allocations, blocking I/O.",
        allowed_tools=["read_file", "grep_search"]
    ),
    Task(
        description="Code quality review of payment module",
        instructions="Check for code smells, complexity, test coverage gaps.",
        allowed_tools=["read_file", "grep_search"]
    )
]

# Fan-in: Coordinator receives all results and synthesizes
# The coordinator's next turn after dispatching will have all results
```

### Fan-In Aggregation Strategies

```python
# Strategy 1: Simple concatenation
def aggregate_concat(results: list[str]) -> str:
    return "\n\n---\n\n".join(results)

# Strategy 2: Structured synthesis (coordinator does this)
"""
## Unified Report

### Critical Issues (from all analyses):
1. [Security] SQL injection in payment_query.py:42
2. [Performance] N+1 query in get_user_payments()

### Medium Issues:
3. [Quality] Cyclomatic complexity > 15 in process_refund()
4. [Security] Missing rate limiting on /api/payments

### Recommendations (prioritized):
1. Fix SQL injection immediately (severity: CRITICAL)
2. Add query batching for payments (saves ~200ms per request)
3. Refactor process_refund into smaller functions
"""

# Strategy 3: Voting/consensus (for classification tasks)
def aggregate_vote(results: list[str]) -> str:
    classifications = [parse_classification(r) for r in results]
    # Majority vote
    from collections import Counter
    votes = Counter(classifications)
    return votes.most_common(1)[0][0]
```

### Parallel Characteristics

| Characteristic | Detail |
|---------------|--------|
| Data flow | Independent branches, merged at fan-in |
| Failure mode | One failure doesn't block others |
| Latency | Max of parallel branch durations |
| Context growth | Each branch starts fresh |
| Debugging | Check each branch independently |

---

## 4. Hierarchical Delegation

### When to Use Hierarchical

Use when **the problem has natural groupings** that themselves contain sub-problems.

```
                    Project Coordinator
                   /        |          \
            Frontend    Backend      DevOps
            Coord       Coord        Coord
           /    \       /    \       /    \
        UI     UX   API    DB    CI    Deploy
       Agent  Agent Agent Agent Agent  Agent
```

### Implementation

```python
# Top-level coordinator
project_coordinator = Agent(
    name="project-coordinator",
    model="claude-sonnet-4-20250514",
    instructions="""You coordinate a large project by delegating to 
    domain coordinators. Each domain coordinator manages its own 
    subagents. You focus on cross-domain integration and synthesis.""",
    tools=[Task, "read_file"],
    max_turns=30
)

# Domain coordinator (spawned as a Task that itself can spawn Tasks)
frontend_coord_task = Task(
    description="Coordinate frontend development for user dashboard",
    instructions="""You are the frontend coordinator. Break the dashboard into:
    1. UI component development (component agent)
    2. State management setup (state agent)  
    3. API integration layer (integration agent)
    
    Use the Task tool to delegate to each specialist.
    Synthesize their outputs into a working frontend.""",
    allowed_tools=[Task, "read_file", "write_file", "grep_search"],
    context="Dashboard requirements: user profile, activity feed, settings"
)
```

### Hierarchical Depth Limits

```
Recommended: 2-3 levels maximum

Level 0: Project Coordinator (plans, delegates)
Level 1: Domain Coordinators (plans within domain, delegates)
Level 2: Specialist Agents (executes, returns results)

Beyond 3 levels:
- Coordination overhead dominates
- Context passing becomes lossy
- Error propagation becomes complex
- Debugging becomes very difficult
```

### Hierarchical Characteristics

| Characteristic | Detail |
|---------------|--------|
| Data flow | Top-down delegation, bottom-up results |
| Failure mode | Isolated per branch — partial success possible |
| Latency | Depends on deepest/slowest branch |
| Context | Narrower at each level (more focused) |
| Complexity | Highest — needs clear boundary design |

---

## 5. Choosing the Right Topology

### Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│               TOPOLOGY SELECTION FLOWCHART                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Are all subtasks dependent on each other?                   │
│  ├── YES → SEQUENTIAL PIPELINE                               │
│  └── NO                                                      │
│      │                                                       │
│      Are all subtasks fully independent?                     │
│      ├── YES → PARALLEL FAN-OUT/FAN-IN                       │
│      └── NO (mix of dependent and independent)               │
│          │                                                   │
│          Can you group into independent domains?             │
│          ├── YES → HIERARCHICAL                              │
│          └── NO → HYBRID (sequential + parallel phases)      │
└─────────────────────────────────────────────────────────────┘
```

### Topology Selection Matrix

| Scenario | Topology | Reasoning |
|----------|----------|-----------|
| Parse → Transform → Validate → Store | Sequential | Each step needs prior output |
| Review security + performance + quality | Parallel | Three independent analyses |
| Build full-stack feature (frontend + backend + tests) | Hierarchical | Natural domain grouping |
| Research → then parallel analysis → then synthesize | Hybrid | Sequential with parallel middle |
| Simple bug fix | None (single agent) | Too simple for orchestration |

### Hybrid Topology Example

```
Sequential Shell with Parallel Interior:

Phase 1 (Sequential): Research & Analysis
    │
    ▼
Phase 2 (Parallel): Implementation
    ├── Frontend implementation
    ├── Backend implementation
    └── Database migration
    │
    ▼
Phase 3 (Sequential): Integration & Testing
    │
    ▼
Phase 4 (Sequential): Documentation
```

```python
# Hybrid implementation
async def hybrid_pipeline(coordinator):
    # Phase 1: Sequential research
    research = await coordinator.run_task(Task(
        description="Research requirements",
        instructions="...",
        allowed_tools=["read_file", "grep_search"]
    ))
    
    # Phase 2: Parallel implementation (all independent)
    implementations = await asyncio.gather(
        coordinator.run_task(Task(
            description="Implement frontend",
            context=research,
            allowed_tools=["read_file", "write_file"]
        )),
        coordinator.run_task(Task(
            description="Implement backend",
            context=research,
            allowed_tools=["read_file", "write_file"]
        )),
        coordinator.run_task(Task(
            description="Create DB migration",
            context=research,
            allowed_tools=["read_file", "write_file"]
        ))
    )
    
    # Phase 3: Sequential integration (depends on all implementations)
    integration = await coordinator.run_task(Task(
        description="Integration testing",
        context=str(implementations),
        allowed_tools=["read_file", "run_tests"]
    ))
    
    return integration
```

---

## 6. Error Handling When One Subagent Fails

### Failure Modes by Topology

| Topology | Failure Impact | Recovery Strategy |
|----------|---------------|-------------------|
| Sequential | Blocks entire pipeline | Retry step or skip with fallback |
| Parallel | Other branches unaffected | Complete what you can, report partial |
| Hierarchical | Branch isolated | Retry branch or escalate |

### Error Handling Strategies

```python
class OrchestrationErrorHandler:
    """Strategies for handling subagent failures."""
    
    def handle_sequential_failure(self, failed_step, pipeline):
        """Sequential: failure blocks downstream steps."""
        strategies = [
            # Strategy 1: Retry with adjusted instructions
            self.retry_with_feedback(failed_step),
            # Strategy 2: Skip and use fallback
            self.skip_with_fallback(failed_step, pipeline),
            # Strategy 3: Abort and report partial progress
            self.abort_with_partial(failed_step, pipeline)
        ]
        return self.try_strategies(strategies)
    
    def handle_parallel_failure(self, failed_branch, all_branches):
        """Parallel: one failure doesn't block others."""
        # Let successful branches complete
        successful = [b for b in all_branches if b.status == "success"]
        failed = [b for b in all_branches if b.status == "failed"]
        
        # Attempt retry on failed branches
        retried = [self.retry_with_feedback(b) for b in failed]
        
        # Aggregate whatever succeeded
        return self.aggregate_partial_results(successful + retried)
    
    def handle_hierarchical_failure(self, failed_branch, tree):
        """Hierarchical: isolate failure to branch."""
        # Only the failed branch is affected
        # Retry the branch coordinator
        return self.retry_branch(failed_branch)
```

### Error Context Feedback Pattern

```python
# When a subagent fails, feed the error back for self-correction
def retry_with_error_context(original_task, error_message):
    """Retry a failed task with error information."""
    return Task(
        description=f"RETRY: {original_task.description}",
        instructions=f"""
        Previous attempt failed with this error:
        ---
        {error_message}
        ---
        
        Please try again, addressing the error above.
        Original instructions: {original_task.instructions}
        """,
        allowed_tools=original_task.allowed_tools,
        context=original_task.context
    )
```

### Graceful Degradation Pattern

```python
# Return partial results when full completion is impossible
def aggregate_with_gaps(results: dict[str, str | None]) -> str:
    report = "## Results (Partial)\n\n"
    
    for task_name, result in results.items():
        if result:
            report += f"### ✅ {task_name}\n{result}\n\n"
        else:
            report += f"### ❌ {task_name}\nFailed — requires manual follow-up.\n\n"
    
    success_rate = sum(1 for r in results.values() if r) / len(results)
    report += f"\n**Completion Rate:** {success_rate:.0%}\n"
    
    return report
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Wrong Topology Choice

```python
# ❌ BAD: Using sequential for independent tasks
# These tasks don't depend on each other!
step1 = Task(description="Lint frontend code", ...)
# WAIT for step1...
step2 = Task(description="Lint backend code", ...)  # Why wait for frontend lint?
# WAIT for step2...
step3 = Task(description="Lint infrastructure code", ...)  # Why wait for backend lint?

# ✅ GOOD: Use parallel for independent tasks
parallel_tasks = [
    Task(description="Lint frontend code", ...),
    Task(description="Lint backend code", ...),
    Task(description="Lint infrastructure code", ...),
]
# All run simultaneously — 3x faster
```

### ❌ ANTI-PATTERN: No Error Isolation in Parallel

```python
# ❌ BAD: One failure kills all parallel branches
results = await asyncio.gather(*tasks)  # If one throws, ALL are lost

# ✅ GOOD: Return exceptions instead of raising
results = await asyncio.gather(*tasks, return_exceptions=True)
successes = [r for r in results if not isinstance(r, Exception)]
failures = [r for r in results if isinstance(r, Exception)]
# Process partial results
```

### Anti-Pattern Summary

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Sequential for independent tasks | Unnecessary latency | Use parallel |
| Parallel for dependent tasks | Missing data between steps | Use sequential |
| No error isolation | One failure kills everything | Catch per-branch |
| Too many hierarchy levels (>3) | Coordination overhead | Flatten to 2-3 |
| No aggregation strategy | Raw results dump on user | Define synthesis step |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│      MULTI-AGENT ORCHESTRATION — QUICK REFERENCE        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  TOPOLOGIES:                                            │
│    Sequential:   A → B → C (dependent steps)            │
│    Parallel:     A | B | C → Aggregate (independent)    │
│    Hierarchical: Coordinator → Sub-coords → Agents      │
│    Hybrid:       Sequential + Parallel phases           │
│                                                         │
│  SELECTION CRITERIA:                                    │
│    All dependent?      → Sequential                     │
│    All independent?    → Parallel                       │
│    Natural groupings?  → Hierarchical                   │
│    Mixed?              → Hybrid                         │
│                                                         │
│  ERROR HANDLING:                                        │
│    Sequential:   Retry step or abort with partial       │
│    Parallel:     Complete what you can, report gaps     │
│    Hierarchical: Isolate and retry the failed branch    │
│                                                         │
│  LATENCY:                                               │
│    Sequential:   Sum(all steps)                         │
│    Parallel:     Max(parallel branches)                 │
│    Hierarchical: Max(deepest branch)                    │
│                                                         │
│  DEPTH LIMIT: 2-3 levels max for hierarchical          │
│                                                         │
│  EXAM TIP: "Which topology?" = most common Q type      │
│  Look for dependency signals in the scenario            │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You're building an agent system for a code review pipeline. The process is: (1) fetch PR diff, (2) run three independent checks (security, performance, style), (3) synthesize findings into a review comment.

**Question 1:** What orchestration topology fits this pipeline?

- A) Pure sequential: fetch → security → performance → style → synthesize
- B) Pure parallel: all five tasks run simultaneously
- C) Hybrid: sequential fetch → parallel checks → sequential synthesis
- D) Hierarchical: coordinator → three sub-coordinators

**Answer:** **C** — This is a classic hybrid topology. Step 1 (fetch) must happen first (others need the diff). Steps 2a/2b/2c are independent (three different analyses on the same diff). Step 3 (synthesis) depends on all analyses completing. Sequential → Parallel → Sequential.

---

**Question 2:** During the parallel analysis phase, the security check subagent fails with a timeout. What's the correct handling?

- A) Abort the entire review — it's incomplete without security
- B) Let performance and style complete, retry security, then aggregate
- C) Let performance and style complete, report security as "pending manual review"
- D) Both B and C are acceptable; the choice depends on business requirements

**Answer:** **D** — Both B and C are valid strategies. If security review is critical (compliance requirement), you'd retry (B). If it's best-effort, you'd report partial results (C). The exam tests that you understand error isolation in parallel topologies — other branches continue regardless.

---

**Question 3:** Why is pure sequential wrong for the three independent checks?

- A) Sequential is always slower than parallel
- B) It adds unnecessary latency — the checks don't depend on each other
- C) Sequential can't handle errors
- D) The checks produce incompatible output formats

**Answer:** **B** — The three checks (security, performance, style) are independent — none needs the output of another. Running them sequentially adds unnecessary latency (total time = sum of all three) when parallel execution would give total time = max of the three. This is the classic "wrong topology" anti-pattern.

---

**Question 4:** In a hierarchical system with 4 levels deep, what's the primary concern?

- A) Token costs increase linearly with depth
- B) Coordination overhead and context loss at each level
- C) The top coordinator becomes a bottleneck
- D) Subagents at level 4 have no tools

**Answer:** **B** — Each level of hierarchy adds coordination overhead (delegation + synthesis) and risks context loss (summaries lose detail). Beyond 2-3 levels, the overhead of managing the hierarchy exceeds the benefit of decomposition. The recommended maximum is 2-3 levels.
