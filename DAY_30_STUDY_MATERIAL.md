# Day 30: Sprint 2 Milestone Checkpoint

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows + Domain 2 (deepened) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) — FINAL DAY |
| **Type** | Milestone Assessment & Sprint 3 Preparation |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Domain 3 Checkpoint: Claude Code Configuration](#1-domain-3-checkpoint-claude-code-configuration)
2. [Domain 2 Checkpoint: MCP Integration (Deepened)](#2-domain-2-checkpoint-mcp-integration-deepened)
3. [CLI Mastery Check](#3-cli-mastery-check)
4. [Scenario Readiness Assessment](#4-scenario-readiness-assessment)
5. [Sprint 1 Retention Check](#5-sprint-1-retention-check)
6. [Anti-Pattern Recognition Gallery](#6-anti-pattern-recognition-gallery)
7. [Confidence Tracker](#7-confidence-tracker)
8. [Sprint 3 Preview](#8-sprint-3-preview)
9. [Quick Reference Card: Sprint 2 Complete](#9-quick-reference-card-sprint-2-complete)
10. [Scenario Challenge: Comprehensive](#10-scenario-challenge-comprehensive)

---

## 1. Domain 3 Checkpoint: Claude Code Configuration

### Self-Assessment Questions

For each question, rate yourself: ✅ Confident | ⚠️ Needs Review | ❌ Needs Rework

#### CLAUDE.md Hierarchy

| Question | Status |
|----------|--------|
| Can you explain the three levels: global → project → subdirectory? | |
| Do you know which level overrides which? | |
| Can you decide WHERE to place a given rule? | |
| Can you identify when hierarchy creates conflicts? | |

#### Key Concept Review

```
CLAUDE.md Hierarchy:
  ~/.claude/CLAUDE.md         → Personal preferences (any project)
  project/CLAUDE.md           → Team standards (this project)
  project/src/api/CLAUDE.md   → Area-specific rules (this directory)

Scoping rule: More specific (deeper) wins for the path it covers.
```

#### .claude/ Directory Structure

| Question | Status |
|----------|--------|
| Can you explain .claude/commands/ purpose and invocation? | |
| Can you explain .claude/rules/ and path matching? | |
| Can you explain .claude/skills/ and their use case? | |
| Do you know commands vs rules vs skills differences? | |

#### Quick Reference: The Three .claude/ Mechanisms

| Mechanism | Location | Purpose | Invocation |
|-----------|----------|---------|------------|
| **Commands** | `.claude/commands/name.md` | Reusable workflows | User types `/name` |
| **Rules** | `.claude/rules/name.md` | Path-specific constraints | Auto-applied by file path |
| **Skills** | `.claude/skills/name.md` | Complex generation patterns | Referenced during tasks |

#### Plan Mode vs Direct Execution

| Question | Status |
|----------|--------|
| When should plan mode be used? | |
| What happens in plan mode in CI (-p mode)? | |
| How does plan mode relate to agentic planning (Sprint 3)? | |

```
Plan Mode:
  When: Multi-file changes, refactoring, unfamiliar code
  What: Claude proposes before executing
  CI:   Plan mode approval skipped (no interactive terminal)
  Exam: Know that -p cannot do interactive plan approval
```

---

## 2. Domain 2 Checkpoint: MCP Integration (Deepened)

### MCP Server Configuration (.mcp.json)

| Question | Status |
|----------|--------|
| Can you write a correct .mcp.json from scratch? | |
| Do you know the fields: command, args, env? | |
| Can you use ${VAR_NAME} for env var expansion? | |
| Do you know WHY env vars matter (security)? | |

#### .mcp.json Template (Know This Cold)

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@mcp/server-package", "${CONNECTION_STRING}"],
      "env": {
        "API_KEY": "${API_KEY_VAR}"
      }
    }
  }
}
```

### MCP Tools, Resources, and Prompts

| Question | Status |
|----------|--------|
| Can you define the three MCP primitives? | |
| Do you know when to use resource vs tool? | |
| Can you write a tool definition with proper schema? | |
| Do you understand resource templates (parameterized URIs)? | |
| Can you explain MCP prompts and their purpose? | |

#### The Three Primitives (Exam Essential)

| Primitive | Initiated By | Purpose | Side Effects? |
|-----------|-------------|---------|---------------|
| **Tools** | AI (tool_use) | Execute actions | Yes |
| **Resources** | Client (proactive) | Provide context | No (read-only) |
| **Prompts** | User (selection) | Structured templates | No |

### MCP Advanced Topics

| Question | Status |
|----------|--------|
| Can you describe the server lifecycle (init → operate → shutdown)? | |
| Do you know stdio vs HTTP+SSE transport differences? | |
| Can you explain protocol errors vs tool errors (isError)? | |
| Do you understand MCP sampling (server-initiated LLM requests)? | |
| Can you explain the security model (client as trust boundary)? | |

#### Transport Quick Reference

| Transport | Setup | Multi-client | Default for |
|-----------|-------|-------------|-------------|
| **stdio** | command + args in .mcp.json | No | Claude Code (local) |
| **HTTP+SSE** | URL + deployment | Yes | Remote/shared servers |

### Environment Variables and Built-in Tools

| Question | Status |
|----------|--------|
| Can you explain ${VAR_NAME} expansion? | |
| Can you list the built-in Claude Code tools? | |
| Do you know precedence rules (built-in > MCP)? | |
| Can you explain allowlists/denylists? | |

---

## 3. CLI Mastery Check

### Essential CLI Flags

Rate your knowledge of each flag:

| Flag | Purpose | Typical Use | Status |
|------|---------|-------------|--------|
| `-p` / `--print` | Non-interactive mode | CI/CD pipelines, scripts | |
| `--output-format json` | Structured JSON output | Programmatic consumption | |
| `--json-schema` | Enforce output schema | Guaranteed output structure | |
| `--resume session_id` | Continue previous session | Multi-day work | |
| `fork_session` | Branch conversation | Safe exploration | |
| `--allowedTools` | Restrict available tools | CI security, minimal perms | |
| `--denyTools` | Block specific tools | Prevent dangerous operations | |

### CLI Flag Combinations

| Scenario | Correct Flags |
|----------|--------------|
| CI documentation generation | `-p --output-format json` |
| CI with schema enforcement | `-p --json-schema '{...}'` |
| Continue yesterday's refactoring | `--resume session_id` |
| Explore alternative approach | `fork_session` (in session) |
| Restricted CI code review | `-p --allowedTools "read_file,search"` |

### Quick Test: Which Flag?

```
1. "Run Claude non-interactively" → -p / --print
2. "Get structured JSON output" → --output-format json
3. "Continue previous session" → --resume
4. "Explore without losing progress" → fork_session
5. "Restrict to read-only tools" → --allowedTools
6. "Enforce specific output shape" → --json-schema
7. "Block file deletion in CI" → --denyTools
```

---

## 4. Scenario Readiness Assessment

### Scenario: Code Generation with Claude Code

| Capability | Can You... | Status |
|-----------|-----------|--------|
| Configure CLAUDE.md for code gen standards | ✅/⚠️/❌ | |
| Create slash commands for workflows | | |
| Use plan mode for multi-file generation | | |
| Run non-interactively in CI (-p) | | |
| Configure .claude/rules/ for language patterns | | |
| Apply iterative refinement effectively | | |
| Place rules at correct hierarchy level | | |

### Scenario: Developer Productivity with Claude

| Capability | Can You... | Status |
|-----------|-----------|--------|
| Use iterative refinement (specific feedback) | | |
| Use --resume for session continuation | | |
| Use fork_session for exploration | | |
| Integrate output into toolchains (--output-format json) | | |
| Combine commands with iterative refinement | | |
| Connect patterns to agentic orchestration | | |

---

## 5. Sprint 1 Retention Check

### Quick Recall: Domain 4 (Prompt Engineering & Structured Output)

| Concept | Can You Still Explain It? | Status |
|---------|--------------------------|--------|
| System prompt best practices | | |
| stop_reason values (all 4) and their meaning | | |
| tool_choice: auto vs any vs specific tool | | |
| JSON Schema for tool input definitions | | |
| Pydantic model → JSON Schema → validation pipeline | | |
| Few-shot prompting (when and how) | | |
| Message Batches API (purpose and lifecycle) | | |
| Programmatic enforcement principle | | |

### Quick Recall: Domain 2 (Sprint 1 Portion)

| Concept | Still Clear? | Status |
|---------|-------------|--------|
| tool_use content block structure | | |
| tool_result content block structure | | |
| Multi-turn conversation flow with tools | | |
| Tool description best practices | | |
| stop_reason: "tool_use" triggering tool execution | | |

### Sprint 1 Key Formula (Should Be Automatic)

```
Structured Output Pipeline:
  Pydantic Model → .model_json_schema() → tool input_schema →
  tool_choice: {type: "tool", name: "X"} → stop_reason: "tool_use" →
  Parse tool_use block → Validate with Pydantic → Typed result

This is the GOLD STANDARD the exam tests repeatedly.
```

---

## 6. Anti-Pattern Recognition Gallery

### Can You Identify the Problem?

**Anti-Pattern 1:**
```json
// .mcp.json
{ "env": { "DB_PASSWORD": "supersecret123" } }
```
**Problem:** _Hardcoded secret in version-controlled file_
**Fix:** _Use `${DB_PASSWORD}` expansion_

---

**Anti-Pattern 2:**
```bash
# CI pipeline
claude "Generate the migration"
```
**Problem:** _Interactive mode in CI (will hang)_
**Fix:** _Use `claude -p "Generate the migration" --output-format json`_

---

**Anti-Pattern 3:**
```markdown
# Global ~/.claude/CLAUDE.md
Never modify packages/shared/ directory
```
**Problem:** _Path-specific restriction in global config (wrong scope)_
**Fix:** _Place in `packages/shared/CLAUDE.md` for proper scoping_

---

**Anti-Pattern 4:**
```json
{
  "name": "search",
  "description": "Search stuff"
}
```
**Problem:** _Generic name (collision risk) + vague description (poor tool selection)_
**Fix:** _`"name": "search_company_wiki", "description": "Search the company wiki for articles by keyword. Returns titles and snippets."`_

---

**Anti-Pattern 5:**
```
Developer refining code:
"Make it better"
"Try again"
"Still not right"
```
**Problem:** _Vague feedback — Claude can't determine what to improve_
**Fix:** _Specific, actionable feedback: "Extract the validation into a separate function and add error types"_

---

**Anti-Pattern 6:**
```
# Using tool_choice pattern thinking in MCP context
"I'll force Claude to use the MCP search tool with tool_choice"
```
**Problem:** _MCP doesn't support tool_choice — always auto selection_
**Fix:** _Write excellent descriptions so Claude naturally picks the right MCP tool_

---

**Anti-Pattern 7:**
```json
{
  "mcpServers": {
    "everything": {
      "command": "node",
      "args": ["super-server.js"]
    }
  }
}
```
**Problem:** _Single mega-server (single point of failure, all secrets in one place)_
**Fix:** _Separate servers per domain (postgres, github, jira)_

---

**Anti-Pattern 8:**
```
Using a tool for stable read-only data:
{ "name": "get_database_schema", "inputSchema": {} }
```
**Problem:** _Stable read-only data exposed as tool (requires AI to decide to call it)_
**Fix:** _Expose as MCP resource for proactive client loading_

---

## 7. Confidence Tracker

### Domain Confidence Self-Assessment

Rate each domain 1-5 (1 = need significant review, 5 = exam ready):

| Domain | Weighting | Sprint 2 Confidence | Target |
|--------|-----------|-------------------|--------|
| Domain 1: Agentic Architecture (27%) | Intro only | /5 | 2+ (full coverage Sprint 3) |
| Domain 2: Tool Design & MCP (18%) | Deep coverage | /5 | 4+ |
| Domain 3: Claude Code Config (20%) | Primary focus | /5 | 4+ |
| Domain 4: Prompt Eng & Structured Output (20%) | Sprint 1 retention | /5 | 4+ (maintain) |
| Domain 5: Context Management (15%) | Not yet | /5 | 1 (Sprint 3) |

### Action Items Based on Confidence

```
If Domain 2 < 4: Review Days 21-26 material, focus on:
  - .mcp.json structure
  - Resources vs Tools decision
  - env var expansion
  - Protocol lifecycle

If Domain 3 < 4: Review Days 16-20 + 27-29 material, focus on:
  - CLAUDE.md hierarchy and scoping
  - commands vs rules vs skills
  - Plan mode behavior
  - CLI flags

If Domain 4 < 4 (Sprint 1 retention): Review Days 1-9 material, focus on:
  - tool_choice options
  - stop_reason values
  - Pydantic → JSON Schema pipeline
  - Programmatic enforcement principle
```

---

## 8. Sprint 3 Preview

### What's Coming: Days 31-45

Sprint 3 covers the **highest-weighted exam domain** (Domain 1: 27%) plus Domain 5 (15%):

```
Sprint 3: Advanced Systems
├── Days 31-38: Agentic Architecture (Domain 1)
│   ├── Agentic loop design
│   ├── Agent SDK: hooks, lifecycle events
│   ├── Task decomposition (Task tool, allowedTools)
│   ├── Coordinator-subagent architecture
│   ├── Multi-agent orchestration patterns
│   ├── Session management at scale
│   └── Escalation patterns, error propagation
│
├── Days 39-42: Scenario Deep-Dives
│   ├── Multi-Agent Research System
│   └── Claude Code for Continuous Integration
│
└── Days 43-45: Context Management (Domain 5) + Final Assessment
    ├── Token monitoring, checkpointing
    ├── Context compression
    ├── Human review workflows
    └── Comprehensive readiness assessment
```

### Key Sprint 3 Concepts to Preview

| Concept | What It Is | Sprint 2 Connection |
|---------|-----------|---------------------|
| **Agentic loop** | Reason → act → observe → repeat | Iterative refinement |
| **max_turns** | Upper bound on loop iterations | "Stop after 3 attempts" |
| **Hooks** | Intercept agent behavior at lifecycle points | Plan mode, validation |
| **Task tool** | Spawn subtasks within an agent | Slash commands delegating work |
| **allowedTools** | Restrict subtask capabilities | Allowlists in Claude Code |
| **Coordinator-subagent** | Central agent delegates to specialists | MCP multi-server |
| **Circuit breaker** | Stop after N failures | "Don't retry forever" |

### How Sprint 2 Prepares You for Sprint 3

```
Sprint 2 taught:                    Sprint 3 will formalize:
  Plan mode (preview changes)    →    Agent planning phase
  Iterative refinement           →    Agentic loops
  CLI flags for control          →    Agent configuration
  MCP tool selection             →    Agent tool routing
  Multiple MCP servers           →    Multiple subagents
  fork_session (explore)         →    Subagent exploration
  --resume (session state)       →    Session persistence
  Error in tool result           →    Error-context feedback
  Allowlists in CI               →    allowedTools in Agent SDK
```

---

## 9. Quick Reference Card: Sprint 2 Complete

```
┌──────────────────────────────────────────────────────────────────┐
│             SPRINT 2 COMPLETE REFERENCE CARD                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  CLAUDE CODE CONFIGURATION (Domain 3):                            │
│                                                                    │
│  CLAUDE.md Hierarchy:                                             │
│    Global (~/.claude/) → Project (root) → Subdirectory           │
│    More specific wins for its path                               │
│                                                                    │
│  .claude/ Directory:                                              │
│    commands/  → Slash commands (/name)                            │
│    rules/    → Path-matched constraints (auto-applied)           │
│    skills/   → Complex generation patterns                       │
│                                                                    │
│  Plan Mode: Preview before execute (multi-file safety)           │
│  Iterative Refinement: Generate → review → feedback → improve   │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  MCP INTEGRATION (Domain 2 Deepened):                             │
│                                                                    │
│  .mcp.json: { mcpServers: { name: { command, args, env } } }    │
│  Env vars: ${VAR_NAME} (keep secrets out of version control)     │
│                                                                    │
│  Three Primitives:                                                │
│    Tools     → AI-initiated actions (tools/call)                 │
│    Resources → Client-loaded context (resources/read)            │
│    Prompts   → User-selected templates (prompts/get)             │
│                                                                    │
│  Lifecycle: Initialize → Operate → Shutdown                      │
│  Transport: stdio (local, default) | HTTP+SSE (remote, multi)    │
│  Errors: Protocol (JSON-RPC) vs Tool (isError: true)             │
│  Sampling: Server requests LLM call through client               │
│  Security: Client = trust boundary, input validation essential   │
│                                                                    │
│  Built-in tools: File ops, Terminal, Search, Web                 │
│  Precedence: Built-in > MCP (on name conflict)                  │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  CLI FLAGS:                                                        │
│                                                                    │
│    -p / --print           Non-interactive (CI mandatory)         │
│    --output-format json   Structured output for scripts          │
│    --json-schema          Enforce output shape                   │
│    --resume               Continue previous session              │
│    fork_session           Branch for exploration                  │
│    --allowedTools         Restrict to named tools                │
│    --denyTools            Block named tools                      │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  API vs MCP (Critical Comparison):                                │
│                                                                    │
│    API tools[]:  Per-request, you execute, tool_choice works     │
│    MCP tools:    Per-session, server executes, always auto       │
│    Both:         Same JSON Schema format for parameters          │
│    Key diff:     input_schema (API) vs inputSchema (MCP)         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  TOP ANTI-PATTERNS TO RECOGNIZE:                                   │
│                                                                    │
│    ❌ Hardcoded secrets in .mcp.json                              │
│    ❌ Interactive mode in CI                                      │
│    ❌ Path-specific rules in wrong scope                         │
│    ❌ Vague tool descriptions (MCP)                              │
│    ❌ Vague refinement feedback                                   │
│    ❌ tool_choice thinking applied to MCP                        │
│    ❌ Single mega-server for all capabilities                    │
│    ❌ Stable data exposed as tool instead of resource            │
│    ❌ No validation after code generation                        │
│    ❌ Duplicating tool definitions across layers                 │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge: Comprehensive

### Scenario

A development team is setting up Claude Code for a full-stack TypeScript project. They need:

1. **Code generation** that follows project conventions (ESLint, strict TS, Vitest)
2. **MCP integration** with a PostgreSQL database and GitHub API
3. **CI pipeline** that generates API documentation non-interactively
4. **Developer workflows** for iterative refactoring with session management
5. **Security**: No secrets in version control, restricted tool access in CI

### Question

**Which configuration set correctly addresses ALL five requirements?**

**A)**
```
CLAUDE.md: "Use TypeScript strict, ESLint, Vitest"
.mcp.json: { postgres: { env: { DB: "postgresql://user:pass@localhost" } },
             github: { env: { TOKEN: "ghp_abc123" } } }
CI: claude "Generate API docs"
Workflow: Start new session each time
Security: Use .gitignore for .mcp.json
```

**B)**
```
CLAUDE.md: "Use TypeScript strict, ESLint, Vitest. Run tsc + vitest after generation."
.mcp.json: { postgres: { env: { DB_URL: "${DATABASE_URL}" } },
             github: { env: { GITHUB_TOKEN: "${GH_TOKEN}" } } }
CI: claude -p "Generate API docs" --output-format json --allowedTools "read_file,search"
Workflow: --resume for continuation, fork_session for exploration
Security: ${VAR} expansion, allowlists in CI
```

**C)**
```
CLAUDE.md: "Generate code however you want"
.mcp.json: { everything: { command: "mega-server", env: { ALL_SECRETS: "${SECRETS}" } } }
CI: claude -p "Generate docs" --output-format json
Workflow: Always start fresh sessions
Security: ${VAR} expansion only
```

**D)**
```
CLAUDE.md: "Use TypeScript strict, ESLint, Vitest"
.mcp.json: { postgres: { env: { DB_URL: "${DATABASE_URL}" } },
             github: { env: { TOKEN: "${GH_TOKEN}" } } }
CI: claude --resume last-ci-session --output-format json
Workflow: --resume for everything (no forking needed)
Security: ${VAR} expansion, no tool restrictions
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

| Requirement | Option B Solution | Why Correct |
|-------------|-------------------|-------------|
| Code generation standards | CLAUDE.md with specific conventions + validation commands | Clear standards + automated validation |
| MCP integration | Separate postgres and github servers with ${VAR} expansion | Proper server separation + secure secrets |
| CI pipeline | `-p` + `--output-format json` + `--allowedTools` | Non-interactive + structured + restricted |
| Developer workflows | `--resume` + `fork_session` | Session continuation + safe exploration |
| Security | ${VAR} expansion + CI allowlists | Secrets safe + minimal CI permissions |

**Why not A?** Hardcoded secrets in .mcp.json, interactive CI mode, no session management features used.

**Why not C?** No coding standards in CLAUDE.md, single mega-server (poor separation), no session management, no CI tool restrictions.

**Why not D?** Using `--resume` in CI implies carrying state between CI runs (fragile, non-reproducible). Missing `-p` flag means interactive mode. No tool restrictions in CI.

**Key Exam Principle:** A complete Claude Code setup requires: clear CLAUDE.md standards, properly separated MCP servers with env var expansion, non-interactive CI with restricted tools, and session management for developer workflows.

---

## Sprint 2 Completion Checklist

- [ ] Domain 3 confidence ≥ 4/5
- [ ] Domain 2 confidence ≥ 4/5
- [ ] Domain 4 retention ≥ 4/5 (Sprint 1)
- [ ] All CLI flags memorized
- [ ] Can write .mcp.json from scratch
- [ ] Can explain CLAUDE.md hierarchy
- [ ] Can identify all anti-patterns above
- [ ] Can distinguish MCP vs API tools
- [ ] Ready for Sprint 3 agentic architecture
- [ ] Updated LEARNING_LOG.md with Sprint 2 results

---

*End of Day 30 Study Material — Sprint 2 Complete*

*Onward to Sprint 3: Advanced Systems (Days 31–45)*
