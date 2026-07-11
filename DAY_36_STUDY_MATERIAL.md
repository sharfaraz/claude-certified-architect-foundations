# Day 36: Session Management and Context Preservation

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27% — highest weighted!) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Complete Introduction to Subagents |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Session Management Overview](#1-session-management-overview)
2. [Context Preservation Techniques](#2-context-preservation-techniques)
3. [Resume and Fork: SDK-Level Session Management](#3-resume-and-fork-sdk-level-session-management)
4. [Passing Context Between Coordinator and Subagents](#4-passing-context-between-coordinator-and-subagents)
5. [Context Window Budgeting](#5-context-window-budgeting)
6. [Graceful Degradation Near Context Limits](#6-graceful-degradation-near-context-limits)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude SDK: Subagents | https://code.claude.com/docs/en/sdk/subagents |
| Anthropic Messages API | https://docs.anthropic.com/en/api/messages |

---

## 1. Session Management Overview

### The Session Problem

Claude is stateless — each API call is independent. **Sessions** are the application-layer pattern that creates continuity across multiple interactions.

```
Without Session Management:
  Call 1: "Analyze auth.py"        → Claude knows about auth.py
  Call 2: "Now fix the bug"        → Claude has NO idea what bug (new context!)

With Session Management:
  Call 1: "Analyze auth.py"        → Claude knows about auth.py
  Call 2: "Now fix the bug"        → Claude knows the bug from Call 1 (context preserved)
```

### Session State Components

| Component | What It Contains | Why It Matters |
|-----------|-----------------|----------------|
| **Conversation history** | All messages exchanged | Continuity of dialogue |
| **Task state** | What's been done, what remains | Progress tracking |
| **Key facts** | Important discoveries/decisions | Prevents re-discovery |
| **Tool results** | Outputs from tool executions | Reference without re-running |
| **Metadata** | Timestamps, costs, turn counts | Monitoring and limits |

### Session Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  CREATE  │───▶│  ACTIVE  │───▶│ SUSPEND  │───▶│  RESUME  │
│ Session  │    │ (working)│    │ (saved)  │    │ (reload) │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                     │                                │
                     │          ┌──────────┐          │
                     └─────────▶│   FORK   │◀─────────┘
                                │(branch)  │
                                └──────────┘
```

---

## 2. Context Preservation Techniques

### Technique 1: Full History Preservation

Keep the entire conversation in the messages array.

```python
# Full history — simplest but most expensive
class FullHistorySession:
    def __init__(self):
        self.messages = []
    
    def add_exchange(self, user_msg, assistant_msg):
        self.messages.append({"role": "user", "content": user_msg})
        self.messages.append({"role": "assistant", "content": assistant_msg})
    
    def get_context(self):
        return self.messages  # Everything, always
```

**Pros:** Complete context, no information loss  
**Cons:** Context window fills up quickly, expensive (more tokens = more cost)

### Technique 2: Summarization

Periodically summarize older context to free up space.

```python
class SummarizedSession:
    def __init__(self, max_recent_messages: int = 10):
        self.summary = ""  # Compressed history
        self.recent_messages = []  # Last N messages (full detail)
        self.max_recent = max_recent_messages
    
    def add_exchange(self, user_msg, assistant_msg):
        self.recent_messages.append({"role": "user", "content": user_msg})
        self.recent_messages.append({"role": "assistant", "content": assistant_msg})
        
        # When recent buffer is full, summarize oldest messages
        if len(self.recent_messages) > self.max_recent * 2:
            oldest = self.recent_messages[:4]  # Take oldest 2 exchanges
            self.summary = self._update_summary(self.summary, oldest)
            self.recent_messages = self.recent_messages[4:]  # Remove summarized
    
    def get_context(self):
        """Return summary + recent messages."""
        context = []
        if self.summary:
            context.append({
                "role": "user",
                "content": f"[Session summary: {self.summary}]"
            })
        context.extend(self.recent_messages)
        return context
    
    def _update_summary(self, current_summary, new_messages):
        """Use Claude to summarize older messages."""
        # This is itself an API call — but saves tokens long-term
        return call_claude_for_summary(current_summary, new_messages)
```

### Technique 3: Key-Fact Extraction

Extract and maintain a structured set of important facts.

```python
class KeyFactSession:
    def __init__(self):
        self.key_facts = {}   # Structured knowledge
        self.recent_messages = []
    
    def extract_facts(self, response: str):
        """Extract key facts from agent responses."""
        # Use structured extraction
        facts = call_claude(
            system="Extract key facts as JSON. Categories: "
                   "decisions, findings, blockers, next_steps",
            messages=[{"role": "user", "content": response}]
        )
        self.key_facts.update(facts)
    
    def get_context(self):
        """Inject key facts as system context."""
        facts_str = json.dumps(self.key_facts, indent=2)
        return [
            {"role": "user", "content": f"[Key facts from session:\n{facts_str}]"},
            *self.recent_messages
        ]

# Example key_facts structure:
key_facts = {
    "decisions": [
        "Using PostgreSQL for the database",
        "Auth will use JWT with 15-min expiry"
    ],
    "findings": [
        "SQL injection in user_query.py line 42",
        "No rate limiting on /api/login"
    ],
    "blockers": [
        "Need DBA approval for schema change"
    ],
    "next_steps": [
        "Implement parameterized queries",
        "Add rate limiter middleware"
    ]
}
```

### Technique 4: Structured State Objects

Maintain a typed state object that represents the current situation.

```python
from dataclasses import dataclass, field

@dataclass
class ProjectState:
    """Structured state for a development session."""
    project_name: str
    current_phase: str  # "analysis" | "design" | "implementation" | "testing"
    files_modified: list[str] = field(default_factory=list)
    files_analyzed: list[str] = field(default_factory=list)
    bugs_found: list[dict] = field(default_factory=list)
    decisions_made: list[str] = field(default_factory=list)
    remaining_tasks: list[str] = field(default_factory=list)
    
    def to_context_string(self) -> str:
        """Serialize state for injection into messages."""
        return f"""
## Current Project State
- Project: {self.project_name}
- Phase: {self.current_phase}
- Files modified: {', '.join(self.files_modified) or 'none'}
- Bugs found: {len(self.bugs_found)}
- Remaining tasks: {len(self.remaining_tasks)}

### Key Decisions:
{chr(10).join(f'- {d}' for d in self.decisions_made)}

### Remaining Work:
{chr(10).join(f'- {t}' for t in self.remaining_tasks)}
"""
```

---

## 3. Resume and Fork: SDK-Level Session Management

### --resume: Continuing a Previous Session

In Claude Code, `--resume` reloads a previous conversation and continues from where it left off.

```bash
# Start a session
claude "Analyze the auth module for vulnerabilities"
# ... session completes, session_id saved

# Resume later — picks up full context
claude --resume <session_id> "Now fix the SQL injection you found"
```

### How --resume Works Internally

```
┌─────────────────────────────────────────────────────────┐
│  --resume loads:                                         │
│    1. Full message history from session store            │
│    2. System prompt (unchanged)                          │
│    3. Tool definitions (unchanged)                       │
│    4. Any accumulated state                              │
│                                                         │
│  Then adds the new user message and continues            │
│  the agentic loop from where it left off.               │
└─────────────────────────────────────────────────────────┘
```

### fork_session: Branching a Session

Fork creates a **copy** of a session at a point in time, allowing exploration of different paths without losing the original.

```python
# Conceptual model of fork_session
class SessionManager:
    def fork_session(self, session_id: str, fork_name: str) -> str:
        """Create a branch of an existing session."""
        original = self.load_session(session_id)
        
        # Deep copy the session state
        forked = SessionState(
            messages=copy.deepcopy(original.messages),
            key_facts=copy.deepcopy(original.key_facts),
            metadata={
                "forked_from": session_id,
                "fork_name": fork_name,
                "fork_point": len(original.messages)
            }
        )
        
        new_id = self.save_session(forked)
        return new_id

# Usage pattern
# Original session: analyzed code, found 3 bugs
# Fork 1: try fixing with approach A
# Fork 2: try fixing with approach B
# Compare results, pick the best approach
```

### SDK-Level Session in Agent Definitions

```python
from claude_code_sdk import Agent, SessionConfig

agent = Agent(
    name="dev-assistant",
    model="claude-sonnet-4-20250514",
    session_config=SessionConfig(
        persistence="disk",           # Save sessions to disk
        auto_summarize=True,          # Summarize when context grows
        summarize_threshold=0.75,     # Summarize at 75% context usage
        max_history_tokens=50000,     # Hard limit on history size
    ),
    tools=["read_file", "write_file", "grep_search"],
    max_turns=25
)
```

---

## 4. Passing Context Between Coordinator and Subagents

### The Context Passing Problem

Coordinators know the big picture. Subagents need enough context to do their job, but NOT the entire conversation.

```
Coordinator Context (large):
├── Full user request history
├── Results from all prior subagents
├── Strategic decisions
└── Cross-cutting concerns

What to pass to subagent (focused):
├── This subagent's specific task
├── Relevant prior results only
├── Constraints and guidelines
└── NOT: unrelated subagent results
```

### Context Passing Strategies

```python
# Strategy 1: Minimal Context (most efficient)
Task(
    description="Write unit tests for UserService",
    instructions="...",
    context="UserService is in src/services/user.py, uses SQLAlchemy ORM",
    allowed_tools=["read_file", "write_file", "run_tests"]
)

# Strategy 2: Prior Step Summary (for sequential pipelines)
Task(
    description="Implement the design from phase 1",
    instructions="...",
    context=f"""
    Design decisions from Phase 1:
    - API uses REST with JSON responses
    - Auth uses JWT with 15-min expiry
    - Database schema has users, sessions, permissions tables
    
    Key constraints:
    - Must be backward-compatible with v1 API
    - Response time < 200ms for all endpoints
    """,
    allowed_tools=["read_file", "write_file"]
)

# Strategy 3: Selective Context (for parallel aggregation)
Task(
    description="Synthesize review findings into PR comment",
    instructions="...",
    context=f"""
    Security Review Results:
    {security_subagent_result}
    
    Performance Review Results:
    {performance_subagent_result}
    
    Style Review Results:
    {style_subagent_result}
    """,
    allowed_tools=["read_file"]
)
```

### Context Compression for Subagents

```python
def compress_for_subagent(full_context: str, task_domain: str) -> str:
    """Compress context to only what's relevant for this subagent's domain."""
    compressed = call_claude(
        system=f"Extract only information relevant to '{task_domain}' from this context. "
               f"Remove unrelated details. Keep it under 500 words.",
        messages=[{"role": "user", "content": full_context}]
    )
    return compressed
```

---

## 5. Context Window Budgeting

### The Context Window Budget

Every message in the conversation consumes tokens from the fixed context window. Budgeting ensures you don't overflow.

```
┌─────────────────────────────────────────────────────────┐
│           CONTEXT WINDOW BUDGET (200K tokens)            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────────────────┐ ~10%  System Prompt    │
│  ├─────────────────────────────┤                        │
│  │                             │                        │
│  │                             │ ~30%  Tool Definitions │
│  │                             │                        │
│  ├─────────────────────────────┤                        │
│  │                             │                        │
│  │                             │                        │
│  │                             │ ~40%  Conversation     │
│  │                             │       History          │
│  │                             │                        │
│  ├─────────────────────────────┤                        │
│  │                             │ ~15%  Current Turn     │
│  ├─────────────────────────────┤       (working space)  │
│  │                             │ ~5%   Safety Margin    │
│  └─────────────────────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

### Budget Allocation Strategy

| Category | Budget % | Tokens (200K window) | Notes |
|----------|----------|---------------------|-------|
| System prompt | 5-10% | 10K-20K | Fixed cost per request |
| Tool definitions | 10-30% | 20K-60K | Scales with tool count |
| Conversation history | 30-50% | 60K-100K | Grows each turn |
| Current turn working space | 10-20% | 20K-40K | Model's thinking room |
| Safety margin | 5-10% | 10K-20K | Buffer for unexpected growth |

### Token Monitoring Implementation

```python
import anthropic

class ContextBudgetMonitor:
    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic()
        self.max_context = 200000  # Model's context window
        self.safety_margin = 0.1   # 10% buffer
        self.usable_tokens = int(self.max_context * (1 - self.safety_margin))
    
    def estimate_usage(self, messages: list, tools: list, system: str) -> dict:
        """Estimate current context window usage."""
        # Use token counting API or heuristic
        system_tokens = self._count_tokens(system)
        tool_tokens = self._count_tokens(json.dumps(tools))
        history_tokens = sum(self._count_tokens(str(m)) for m in messages)
        
        total = system_tokens + tool_tokens + history_tokens
        remaining = self.usable_tokens - total
        
        return {
            "total_used": total,
            "remaining": remaining,
            "usage_percent": total / self.usable_tokens,
            "should_compress": remaining < (self.usable_tokens * 0.2)
        }
    
    def _count_tokens(self, text: str) -> int:
        """Approximate token count (4 chars ≈ 1 token)."""
        return len(text) // 4
```

---

## 6. Graceful Degradation Near Context Limits

### Degradation Strategy Ladder

```
Context Usage:    0% ─────────── 60% ──── 75% ──── 90% ──── 100%
                  │               │        │        │        │
Strategy:     Full Detail   Start      Aggressive  Emergency  STOP
                           Summarizing  Compression  Mode     
```

### Implementation

```python
class GracefulDegradation:
    """Progressive context compression as limits approach."""
    
    def apply_strategy(self, usage_percent: float, session) -> str:
        if usage_percent < 0.60:
            return "full_detail"  # No compression needed
        
        elif usage_percent < 0.75:
            # Start summarizing old messages
            session.summarize_oldest_messages(keep_recent=10)
            return "summarizing"
        
        elif usage_percent < 0.90:
            # Aggressive compression
            session.summarize_oldest_messages(keep_recent=4)
            session.drop_tool_results_older_than(turns=5)
            session.compress_key_facts()
            return "aggressive_compression"
        
        else:
            # Emergency mode
            session.replace_history_with_summary()
            session.keep_only_current_task()
            return "emergency_mode"
    
    def emergency_compress(self, session) -> list:
        """Last resort: compress everything to a brief summary."""
        summary = call_claude(
            system="Summarize this entire session in under 500 words. "
                   "Focus on: current task, key decisions, and next steps.",
            messages=session.messages[-4:]  # Only last 2 exchanges for reference
        )
        
        return [
            {"role": "user", "content": f"[Session compressed. Summary: {summary}]\n\n"
                                        f"Continue with the current task."}
        ]
```

### Tool Result Compression

```python
def compress_tool_results(messages: list, max_result_tokens: int = 1000) -> list:
    """Truncate old tool results to save context space."""
    compressed = []
    for msg in messages:
        if msg.get("role") == "user" and isinstance(msg.get("content"), list):
            new_content = []
            for block in msg["content"]:
                if block.get("type") == "tool_result":
                    result_text = block.get("content", "")
                    if len(result_text) > max_result_tokens * 4:  # Rough char-to-token
                        block = {
                            **block,
                            "content": result_text[:max_result_tokens * 4] 
                                       + "\n[... truncated for context space ...]"
                        }
                new_content.append(block)
            msg = {**msg, "content": new_content}
        compressed.append(msg)
    return compressed
```

---

## 7. Anti-Patterns

### ❌ ANTI-PATTERN: Unbounded Context Growth

```python
# ❌ BAD: Never manages context size
class NaiveSession:
    def __init__(self):
        self.messages = []
    
    def call_agent(self, user_input):
        self.messages.append({"role": "user", "content": user_input})
        # Messages grow forever — will eventually overflow
        response = client.messages.create(
            messages=self.messages  # All of history, always
        )
        self.messages.append({"role": "assistant", "content": response.content})
        # What happens at turn 100? 200? Context overflow!
```

### ✅ CORRECT: Managed Context with Compression

```python
# ✅ GOOD: Monitor and compress proactively
class ManagedSession:
    def __init__(self, max_history_tokens=80000):
        self.messages = []
        self.max_tokens = max_history_tokens
        self.monitor = ContextBudgetMonitor()
    
    def call_agent(self, user_input):
        self.messages.append({"role": "user", "content": user_input})
        
        # Check budget BEFORE calling API
        usage = self.monitor.estimate_usage(self.messages, self.tools, self.system)
        if usage["should_compress"]:
            self._compress_history()
        
        response = client.messages.create(messages=self.messages)
        self.messages.append({"role": "assistant", "content": response.content})
    
    def _compress_history(self):
        # Summarize oldest half, keep recent half
        midpoint = len(self.messages) // 2
        old_messages = self.messages[:midpoint]
        summary = summarize(old_messages)
        self.messages = [
            {"role": "user", "content": f"[Prior context: {summary}]"},
            *self.messages[midpoint:]
        ]
```

### Anti-Pattern Summary

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Unbounded history | Context overflow | Monitor and compress |
| Passing full history to subagents | Wastes subagent context | Pass relevant context only |
| No session persistence | Work lost on crash | Persist to disk/DB |
| Summarizing too aggressively | Loses important details | Preserve key facts separately |
| No degradation strategy | Hard failure at limit | Progressive compression |

---

## 8. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│     SESSION MANAGEMENT & CONTEXT — QUICK REFERENCE      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  PRESERVATION TECHNIQUES:                               │
│    1. Full History    → Complete but expensive           │
│    2. Summarization   → Compress old, keep recent       │
│    3. Key-Fact Extract→ Structured knowledge base       │
│    4. State Objects   → Typed, focused state            │
│                                                         │
│  SESSION OPERATIONS:                                    │
│    --resume         → Continue from saved state         │
│    fork_session     → Branch for exploration            │
│                                                         │
│  CONTEXT BUDGET (200K window):                          │
│    System prompt:     5-10%                             │
│    Tool definitions: 10-30%                             │
│    History:          30-50%                             │
│    Working space:    10-20%                             │
│    Safety margin:     5-10%                             │
│                                                         │
│  DEGRADATION LADDER:                                    │
│    <60%  → Full detail                                  │
│    60-75% → Start summarizing                           │
│    75-90% → Aggressive compression                     │
│    >90%  → Emergency mode                              │
│                                                         │
│  SUBAGENT CONTEXT: Pass ONLY what's relevant            │
│  Never pass full coordinator history to subagents       │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

Your agent has been running for 45 turns on a large refactoring task. The context window is at 82% capacity. The agent still has 3 more files to refactor.

**Question 1:** What context management strategy should activate at 82% usage?

- A) No action needed — 82% is fine
- B) Aggressive compression — summarize old messages, truncate tool results
- C) Emergency mode — replace everything with a brief summary
- D) Abort the task — context is too full

**Answer:** **B** — At 82%, you're in the "aggressive compression" zone (75-90%). Summarize older messages, truncate old tool results, and compress key facts. Emergency mode (C) is premature — save that for 90%+. No action (A) risks hitting the wall in 3 more turns.

---

**Question 2:** When passing context from a coordinator to a subagent, what should you include?

- A) The entire coordinator conversation history
- B) Only the subagent's specific task description
- C) Relevant prior results + task-specific context + constraints
- D) Just the user's original request

**Answer:** **C** — The subagent needs enough context to do its job: relevant prior results (what came before), task-specific context (what to focus on), and constraints (what's not allowed). Full history (A) wastes context. Too little (B, D) leaves the subagent without necessary information.

---

**Question 3:** What is the purpose of fork_session?

- A) To create a backup before dangerous operations
- B) To branch a session for exploring different approaches
- C) To split a session between two users
- D) To reduce context window usage

**Answer:** **B** — Fork creates a copy of the session at a point in time, allowing exploration of different solution paths (e.g., "try approach A in fork 1, approach B in fork 2") without losing the original state. It's not primarily for backups, sharing, or compression.

---

**Question 4:** At what context usage percentage should you start applying summarization?

- A) 30% — summarize early and often
- B) 60% — begin proactive summarization
- C) 90% — only summarize when absolutely necessary
- D) 100% — wait until you get an error

**Answer:** **B** — 60% is the recommended threshold to begin summarization. Starting at 30% (A) loses detail unnecessarily. Waiting until 90% (C) or 100% (D) risks running out of space during a critical operation with no room to compress.
