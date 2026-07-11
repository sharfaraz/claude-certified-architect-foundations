# Day 29: Scenario Deep-Dive: Developer Productivity with Claude

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows + Domain 1 (intro) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Scenario** | Developer Productivity with Claude |
| **Academy Course** | Building with Claude API Pod 4 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Scenario Overview: Developer Productivity](#1-scenario-overview-developer-productivity)
2. [Iterative Refinement as a Productivity Pattern](#2-iterative-refinement-as-a-productivity-pattern)
3. [Session Management: --resume and fork_session](#3-session-management---resume-and-fork_session)
4. [--output-format json for Toolchain Integration](#4---output-format-json-for-toolchain-integration)
5. [Slash Commands + Iterative Refinement](#5-slash-commands--iterative-refinement)
6. [Connecting to Agentic Orchestration (Sprint 3 Preview)](#6-connecting-to-agentic-orchestration-sprint-3-preview)
7. [Advanced Tool Use Patterns (API Pod 4)](#7-advanced-tool-use-patterns-api-pod-4)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code CLI Reference | https://docs.anthropic.com/en/docs/claude-code/cli |
| Claude Code Sessions | https://docs.anthropic.com/en/docs/claude-code/sessions |
| Claude API Advanced Tool Use | https://docs.anthropic.com/en/docs/build-with-claude/tool-use |

---

## 1. Scenario Overview: Developer Productivity

### Exam Scenario Context

The "Developer Productivity with Claude" scenario tests your understanding of **workflows that maximize developer efficiency** using Claude Code features. It focuses on session management, iterative refinement, and toolchain integration.

### What the Exam Tests

| Aspect | What You Must Know |
|--------|-------------------|
| **Iterative refinement** | The prompt → review → refine → accept workflow |
| **--resume** | Continuing previous sessions with preserved context |
| **fork_session** | Branching conversations for exploration |
| **--output-format json** | Integrating Claude output into toolchains |
| **Slash commands** | Combining commands with refinement workflows |
| **Agentic connection** | How these patterns relate to orchestration |

### The Productivity Stack

```
┌────────────────────────────────────────────────────┐
│          DEVELOPER PRODUCTIVITY FEATURES           │
├────────────────────────────────────────────────────┤
│                                                      │
│  Session Management:                                │
│    --resume     → Continue where you left off       │
│    fork_session → Explore without losing progress   │
│                                                      │
│  Output Integration:                                │
│    --output-format json → Feed output to tools      │
│    --json-schema        → Enforce output structure   │
│                                                      │
│  Workflow Automation:                                │
│    Slash commands → Repeatable workflows             │
│    Iterative refinement → Progressive improvement   │
│    Plan mode → Review before execution              │
│                                                      │
└────────────────────────────────────────────────────┘
```

---

## 2. Iterative Refinement as a Productivity Pattern

### The Core Loop

```
┌─────────┐     ┌────────┐     ┌──────────┐     ┌──────────┐
│ Initial │────▶│ Review │────▶│ Feedback │────▶│ Improved │
│ Prompt  │     │ Output │     │ (specific)│    │ Output   │
└─────────┘     └────────┘     └──────────┘     └──────────┘
                     │                               │
                     ▼                               ▼
                  Accept                     Continue refining?
```

### Why Iterative Refinement Is a Productivity Multiplier

| Without Refinement | With Refinement |
|-------------------|----------------|
| Must write perfect prompt upfront | Start with approximate intent |
| Discard and retry from scratch | Build on what's already good |
| Lose context between attempts | Preserve context across rounds |
| One-shot quality ceiling | Progressively higher quality |

### Effective Refinement Strategies

**Strategy 1: Incremental Specification**
```
Round 1: "Refactor the UserService class for better testability"
Round 2: "Extract the database queries into a separate Repository class"
Round 3: "Add dependency injection so the Repository can be mocked in tests"
Round 4: "Generate tests using the new injectable Repository"
```

**Strategy 2: Constraint Tightening**
```
Round 1: "Generate an API endpoint for user search"
Round 2: "Add pagination — cursor-based, not offset"
Round 3: "Add rate limiting — max 100 requests per minute per user"
Round 4: "Add caching — 30s TTL, invalidate on user update"
```

**Strategy 3: Quality Escalation**
```
Round 1: "Make it work" (correct functionality)
Round 2: "Make it right" (clean code, proper patterns)
Round 3: "Make it fast" (optimize performance)
Round 4: "Make it safe" (add error handling, validation)
```

### Context Preservation Across Rounds

Each refinement round **preserves full context** from previous rounds:
- Claude remembers the original code
- Claude remembers your previous feedback
- Claude remembers what it already changed
- Claude can build on accumulated understanding

---

## 3. Session Management: --resume and fork_session

### --resume: Continue a Previous Session

The `--resume` flag reloads a previous session's context, allowing you to continue work across terminal sessions:

```bash
# Start working on a refactoring task
claude "Refactor the payment processing module"
# ... work happens, session ends ...
# Session ID: session_xyz789

# Next day, continue where you left off
claude --resume session_xyz789
# Full context is restored — Claude remembers everything
```

### When to Use --resume

| Scenario | Why --resume? |
|----------|--------------|
| Multi-day refactoring | Continue with full context next session |
| Interrupted work | Pick up after a meeting/break |
| Progressive development | Build features across multiple sittings |
| Context-heavy tasks | Avoid re-explaining complex codebases |

### fork_session: Branch for Exploration

`fork_session` creates a **copy** of the current conversation, allowing you to explore an alternative approach without affecting the original:

```
Original session: Refactoring UserService
  └─── fork_session ─── Alternative approach: Event-driven UserService
                         (explore without risk)

Original session still intact → --resume to return
```

### The fork_session Workflow

```
┌─────────────────────────────────────────────────────────┐
│  1. Working on main refactoring (session A)             │
│  2. "What if I used an event-driven approach instead?"  │
│  3. fork_session → creates session B (copy of A)        │
│  4. Explore event-driven approach in session B          │
│  5. Decide: keep fork or discard?                       │
│     Option A: Like the fork → continue in B            │
│     Option B: Prefer original → --resume A             │
└─────────────────────────────────────────────────────────┘
```

### Key Differences

| Feature | --resume | fork_session |
|---------|----------|-------------|
| **Purpose** | Continue previous session | Branch for exploration |
| **Original session** | Exactly as left | Preserved, unmodified |
| **New session** | Same session, continued | Copy of current session |
| **Use case** | Multi-session work | Alternative exploration |
| **Risk** | None (continues existing) | None (original preserved) |

---

## 4. --output-format json for Toolchain Integration

### Integrating Claude Output Into Developer Tools

`--output-format json` produces structured output that other tools can parse:

```bash
# Generate and pipe to downstream tools
claude -p "Analyze src/auth/ for security issues" --output-format json | jq '.result'

# Use in scripts
ANALYSIS=$(claude -p "List all TODO comments with file and line" --output-format json)
echo "$ANALYSIS" | jq -r '.result' >> reports/todo-report.txt

# Feed to other tools
claude -p "Generate test data for user table" --output-format json | jq -r '.result' | psql -d testdb
```

### JSON Output Fields for Scripting

```json
{
  "type": "result",
  "subtype": "success",
  "result": "The analysis found 3 security issues...",
  "cost_usd": 0.0045,
  "duration_ms": 3200,
  "duration_api_ms": 2800,
  "num_turns": 2,
  "is_error": false,
  "session_id": "session_abc123"
}
```

### Toolchain Integration Patterns

| Pattern | Command | Downstream |
|---------|---------|-----------|
| Code review → Slack | `claude -p "Review PR #42" --output-format json` | `jq '.result' \| slack-notify` |
| Type analysis → Dashboard | `claude -p "Find type errors" --output-format json` | `jq \| curl dashboard-api` |
| Migration gen → DB | `claude -p "Generate migration" --output-format json` | `jq '.result' \| psql` |
| Doc gen → Static site | `claude -p "Generate docs" --output-format json` | `jq '.result' > docs/api.md` |

### Combining Flags for Full Automation

```bash
# Complete automation: non-interactive + structured output + schema enforcement
claude -p "Extract all API endpoints and their methods" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"endpoints":{"type":"array","items":{"type":"object","properties":{"path":{"type":"string"},"method":{"type":"string"},"description":{"type":"string"}},"required":["path","method"]}}},"required":["endpoints"]}'
```

---

## 5. Slash Commands + Iterative Refinement

### Combining Commands with Refinement

Slash commands provide the **starting point**, and iterative refinement provides the **quality improvement**:

```
/refactor src/services/payment.ts    ← Command starts the workflow
"Extract the validation logic"       ← Refinement round 1
"Add error types for each failure"   ← Refinement round 2
"Now generate tests"                 ← Refinement round 3 (cross-command)
```

### Workflow Commands Designed for Iteration

```markdown
<!-- .claude/commands/improve.md -->
# Improve Code

Analyze the specified code and suggest improvements in categories:
1. **Readability**: Naming, structure, comments
2. **Performance**: Unnecessary computations, N+1 patterns
3. **Safety**: Error handling, null checks, validation
4. **Testing**: Missing test coverage, untested edge cases

Present improvements as a numbered list. Wait for the user to select
which improvements to apply. Apply selected improvements one at a time,
running tests after each change.

This enables iterative selection rather than all-or-nothing changes.
```

```markdown
<!-- .claude/commands/explore.md -->
# Explore Approaches

For the given problem, present 3 different implementation approaches:
1. **Simple**: Minimal code, easy to understand
2. **Robust**: Full error handling, edge cases covered
3. **Performant**: Optimized for speed/memory

For each approach, show:
- Key code snippet (pseudocode or real)
- Trade-offs (pros and cons)
- Best suited for (when to choose this)

Wait for the user to select an approach before implementing.
This supports exploration before commitment.
```

### Session Flow with Commands and Refinement

```
Session start
  ├── /explore "how to implement caching for user profiles"
  │   → Claude presents 3 approaches
  │
  ├── "I prefer approach 2 (Redis-based). Implement it."
  │   → Claude generates implementation
  │
  ├── "Add a fallback to in-memory cache if Redis is down"
  │   → Claude refines the implementation
  │
  ├── /add-tests src/cache/user-cache.ts
  │   → Claude generates tests
  │
  └── "Add a test for the Redis-down fallback scenario"
      → Claude adds specific test
```

---

## 6. Connecting to Agentic Orchestration (Sprint 3 Preview)

### How Developer Productivity Patterns Map to Agents

| Productivity Feature | Agentic Equivalent (Sprint 3) |
|---------------------|------------------------------|
| **Iterative refinement** | Agentic loop (reason → act → observe → repeat) |
| **--resume** | Session persistence in multi-turn agents |
| **fork_session** | Subagent exploration (try approach without committing) |
| **Plan mode** | Agent planning phase before execution |
| **Slash commands** | Agent tools (pre-defined capabilities) |
| **--output-format json** | Structured inter-agent communication |

### The Orchestration Preview

```
Sprint 2 (Manual):                    Sprint 3 (Automated):
  Developer decides next step           Coordinator agent decides
  Developer provides feedback           Error-context feedback loop
  Developer forks to explore            Subagent explores alternatives
  Developer resumes original            Coordinator maintains state
  Developer validates output            Hooks validate automatically
```

### Key Insight: From Manual to Automated

The productivity patterns you learn in Sprint 2 become **automated agent behaviors** in Sprint 3:

```
Manual: Developer types feedback → Claude refines
Automated: Agent observes test failure → retries with error context

Manual: Developer uses fork_session to explore
Automated: Coordinator spawns subagent to explore alternatives

Manual: Developer uses --resume to continue
Automated: Agent maintains persistent session state
```

---

## 7. Advanced Tool Use Patterns (API Pod 4)

### Connecting API-Level Patterns to MCP Integration

Academy Pod 4 covers advanced tool use patterns that bridge Sprint 1 (API) and Sprint 2 (MCP):

### Multi-Tool Workflows

```python
# Claude calling multiple tools in sequence (API level)
# This pattern mirrors MCP multi-server tool selection

tools = [
    {"name": "search_codebase", "description": "Find code patterns", "input_schema": {...}},
    {"name": "read_file", "description": "Read file content", "input_schema": {...}},
    {"name": "write_file", "description": "Write/modify files", "input_schema": {...}},
    {"name": "run_tests", "description": "Execute test suite", "input_schema": {...}}
]

# Claude orchestrates: search → read → modify → test (multi-step workflow)
```

### Conditional Tool Selection

Claude selects tools based on context — same pattern in both API and MCP:

```
User: "Fix the bug in authentication"

Claude's reasoning:
  1. Need to find auth code → search_codebase
  2. Need to read the file → read_file
  3. Need to understand the issue → (reasoning, no tool)
  4. Need to fix the code → write_file
  5. Need to verify → run_tests
```

### Advanced Pattern: Tool Chaining

```python
# Tool results from one call inform the next call
# This is the foundation of agentic behavior

# Round 1: Claude searches
tool_use: search_codebase(pattern="authenticate")
# Returns: src/auth/service.ts:42

# Round 2: Claude reads (informed by search result)
tool_use: read_file(path="src/auth/service.ts")
# Returns: file content

# Round 3: Claude fixes (informed by reading)
tool_use: write_file(path="src/auth/service.ts", content="...")
# Returns: success

# Round 4: Claude verifies (informed by fix)
tool_use: run_tests(pattern="auth")
# Returns: 5/5 tests pass
```

### The Unified Mental Model

```
API tools[] (Sprint 1) → MCP tools (Sprint 2) → Agent tools (Sprint 3)

All share:
  - JSON Schema for parameters
  - AI-driven selection based on descriptions
  - Sequential/conditional execution based on results
  - The same cognitive pattern: observe → decide → act → observe
```

---

## 8. Anti-Patterns

### ❌ Anti-Pattern 1: Starting Fresh Instead of Resuming

```bash
# Developer worked on a complex refactoring yesterday
# Today, instead of resuming:
claude "Continue the refactoring of the payment module"
# ❌ New session — Claude has NO context of yesterday's work
```

**Problem:** Lost context forces re-explanation and inconsistency with prior decisions.

**Fix:** Use `--resume` to continue with full context:
```bash
claude --resume session_from_yesterday
```

### ❌ Anti-Pattern 2: Modifying Original Work While Exploring

```
# Developer exploring alternative approaches
"Actually, let me try a completely different approach"
# Claude modifies the existing code with the experimental approach
# If the experiment fails, original work is lost
```

**Problem:** Exploration overwrites working code without a safety net.

**Fix:** Use `fork_session` before exploring alternatives:
```
"Let me fork this session to explore an alternative approach"
# fork_session → explore freely without risking original work
```

### ❌ Anti-Pattern 3: Vague Refinement Feedback

```
Round 1: "Generate a user service"
Round 2: "Make it better"
Round 3: "It's still not great"
Round 4: "Try again"
```

**Problem:** Vague feedback gives Claude no direction for improvement. Changes are random.

**Fix:** Specific, actionable feedback:
```
Round 2: "Add error handling — wrap DB calls in try/catch, throw typed ServiceError"
Round 3: "The pagination is offset-based — change to cursor-based using createdAt"
Round 4: "Add JSDoc comments to all public methods"
```

### ❌ Anti-Pattern 4: JSON Output Without Error Checking

```bash
# CI script uses output without checking for errors
RESULT=$(claude -p "Generate migration" --output-format json)
echo "$RESULT" | jq -r '.result' | psql -d production  # ❌ Dangerous!
```

**Problem:** If Claude errors or generates invalid SQL, it goes directly to production.

**Fix:** Always validate before acting on output:
```bash
RESULT=$(claude -p "Generate migration" --output-format json)
IS_ERROR=$(echo "$RESULT" | jq -r '.is_error')
if [ "$IS_ERROR" = "true" ]; then
  echo "Generation failed!" && exit 1
fi
# Validate SQL syntax before applying
echo "$RESULT" | jq -r '.result' | pg_format --check || exit 1
```

---

## 9. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│       DEVELOPER PRODUCTIVITY WITH CLAUDE QUICK REFERENCE          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ITERATIVE REFINEMENT:                                             │
│    Pattern:   Generate → Review → Feedback (specific) → Improve │
│    Key:       Each round builds on previous context              │
│    Strategy:  Incremental spec, constraint tightening, quality ↑ │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  SESSION MANAGEMENT:                                               │
│                                                                    │
│    --resume session_id:                                           │
│      Continue previous session with full context                 │
│      Use: multi-day work, interrupted tasks                      │
│                                                                    │
│    fork_session:                                                   │
│      Branch conversation for safe exploration                    │
│      Original session preserved, return via --resume             │
│      Use: trying alternatives without risk                       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  TOOLCHAIN INTEGRATION:                                            │
│                                                                    │
│    --output-format json → Structured output for scripts          │
│    Key fields: result, is_error, cost_usd, session_id            │
│    Always check is_error before using result                     │
│    Combine with -p for full automation                            │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  AGENTIC CONNECTION (Sprint 3 Preview):                            │
│                                                                    │
│    Iterative refinement → Agentic loop                           │
│    --resume → Session persistence                                │
│    fork_session → Subagent exploration                            │
│    Plan mode → Agent planning phase                              │
│    Commands → Agent tools                                        │
│    Manual → Automated (Sprint 2 → Sprint 3 evolution)            │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • --resume CONTINUES a session (same session, more context)     │
│  • fork_session COPIES a session (explore without risk)          │
│  • Use fork for alternatives, resume for continuation            │
│  • Refinement feedback must be SPECIFIC and ACTIONABLE           │
│  • --output-format json requires is_error checking               │
│  • These patterns become automated in Sprint 3 agentic loops    │
│  • API Pod 4: multi-tool workflows = same pattern as MCP tools   │
│  • Tool chaining = foundation of agentic behavior               │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Scenario (From SPEC)

A developer is using Claude Code to refactor a large module. After the initial refactoring pass, they want to explore an alternative approach without losing the current refactored state. Later, they want to return to the original refactoring and continue from where they left off.

### Question

**Which combination of features supports this workflow?**

**A)** Use `--resume` to go back to the original session, then start a new session for the alternative

**B)** Use `fork_session` to branch the conversation for the alternative approach, then `--resume` the original session to continue

**C)** Save the refactored files manually, start a new session for the alternative, then restore the files

**D)** Use `--output-format json` to export both approaches and compare them externally

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** correctly uses both features for their intended purpose:
  - `fork_session` creates a **branch** of the current conversation for exploring the alternative approach. The original session is preserved intact.
  - `--resume` on the original session returns to the initial refactoring, with all its context, to continue where they left off.
  - This workflow: refactor → fork (explore alternative) → resume original → continue refactoring

- **Option A** gets the order wrong. `--resume` goes back to a previous session, but starting a "new session" for the alternative loses the context of what was already done. The fork is the correct way to preserve context for exploration.

- **Option C** is manual file management — error-prone, loses conversation context (Claude's understanding of the code), and doesn't leverage Claude Code's session features.

- **Option D** exports output but doesn't manage **session state** (Claude's memory of the conversation, decisions made, and accumulated understanding).

**Key Exam Principle:** `fork_session` is for **safe exploration** (branch without risk), `--resume` is for **continuation** (pick up where you left off). Together they support explore-and-return workflows.

---

## Summary: Key Exam Takeaways

1. Iterative refinement: **generate → review → specific feedback → improve** (builds on context)
2. `--resume` continues a session — **same context**, more conversation
3. `fork_session` branches a session — **safe exploration**, original preserved
4. Use fork for alternatives, resume for continuation — don't mix them up
5. `--output-format json` enables **toolchain integration** — always check `is_error`
6. Refinement feedback must be **specific and actionable** (not "make it better")
7. These patterns become **automated** in Sprint 3's agentic loops
8. API Pod 4 patterns (multi-tool, chaining) are the **same cognitive pattern** as MCP
9. The unified mental model: observe → decide → act → observe (human or automated)
10. Sprint 2 teaches the **manual** version of what Sprint 3 automates

---

*End of Day 29 Study Material*
