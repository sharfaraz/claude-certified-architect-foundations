# Day 45: Sprint 3 Milestone Checkpoint — Comprehensive Readiness Assessment

---

| Field | Detail |
|-------|--------|
| **Domain** | ALL FIVE DOMAINS — Comprehensive Final Assessment |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) — FINAL DAY |
| **Academy Course** | Complete Building with Claude API Pod 5 |
| **Estimated Study Time** | 3–4 hours |

---

## Table of Contents

1. [Large Codebase Exploration Strategies](#1-large-codebase-exploration-strategies)
2. [Domain 1 Comprehensive Review (27%)](#2-domain-1-comprehensive-review-27)
3. [Domain 2 Comprehensive Review (18%)](#3-domain-2-comprehensive-review-18)
4. [Domain 3 Comprehensive Review (20%)](#4-domain-3-comprehensive-review-20)
5. [Domain 4 Comprehensive Review (20%)](#5-domain-4-comprehensive-review-20)
6. [Domain 5 Comprehensive Review (15%)](#6-domain-5-comprehensive-review-15)
7. [Comprehensive Practice Exam (10 Questions)](#7-comprehensive-practice-exam-10-questions)
8. [Domain Confidence Tracker](#8-domain-confidence-tracker)
9. [Anti-Pattern Master List](#9-anti-pattern-master-list)
10. [Final Exam Preparation Tips](#10-final-exam-preparation-tips)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Full Exam Guide | Claude Certified Architect Foundations Certification Exam Guide (PDF) |
| Claude API Reference | https://docs.anthropic.com/en/api/messages |
| Claude Code Docs | https://docs.anthropic.com/en/docs/claude-code |
| MCP Specification | https://docs.anthropic.com/en/docs/claude-code/mcp |
| Agent SDK Reference | https://code.claude.com/docs/en/agent-sdk |

---

## 1. Large Codebase Exploration Strategies

### Why This Matters (Domain 5)

Agents working with large codebases need efficient strategies to understand code without consuming the entire context window.

### Exploration Techniques

| Technique | What It Does | When to Use |
|-----------|-------------|-------------|
| **File tree analysis** | Map directory structure, identify modules | First step — understand project layout |
| **Dependency mapping** | Trace imports/requires between files | Understand how components connect |
| **Symbol search** | Find function/class definitions by name | When you know WHAT to find |
| **Targeted reading** | Read specific files/sections | When you know WHERE to look |
| **Broad scanning** | Skim many files for patterns | When you don't know where to start |

### When to Use Broad vs. Targeted

| Scenario | Strategy | Rationale |
|----------|----------|-----------|
| "Fix bug in login" | Targeted: find auth files, read login logic | Known scope |
| "Understand the architecture" | Broad: file tree → dependency map → key interfaces | Unknown scope |
| "Add feature to existing module" | Semi-targeted: read module, check interfaces | Known area, unknown details |
| "Find all usages of function X" | Symbol search + grep | Specific reference lookup |

### Efficient Exploration Pattern

```python
# Step 1: File tree analysis (low token cost)
tree = list_directory("/project", depth=2)  # ~200 tokens

# Step 2: Identify key files from tree
key_files = [
    "package.json",      # Dependencies
    "tsconfig.json",     # Config
    "src/index.ts",      # Entry point
    "src/types/",        # Type definitions
]

# Step 3: Targeted reading (focused token spend)
entry = read_file("src/index.ts")  # Understand routing
types = read_file("src/types/index.ts")  # Understand data model

# Step 4: Dependency trace (follow imports)
# From index.ts imports → read those specific files
```

### Context Budget for Exploration

```
Total budget: 200K tokens
├── File tree scan: ~500 tokens (cheap)
├── Key file reads: ~5K tokens (moderate)
├── Targeted deep reads: ~15K tokens (focused)
└── Working memory for analysis: ~30K tokens
    
Total exploration cost: ~50K (25% of budget)
Remaining for actual work: ~150K (75%)
```

---

## 2. Domain 1 Comprehensive Review (27%)

### Core Concepts Checklist

| Concept | Key Exam Points |
|---------|-----------------|
| **Agentic loops** | prompt → reason → act → observe → repeat/stop. stop_reason drives loop control. |
| **max_turns** | Circuit breaker. Prevents infinite loops. Set per agent/subagent (3-10 typical). |
| **Coordinator-subagent** | Coordinator orchestrates (dispatch/route/pass/trigger/escalate). Subagents execute. |
| **allowedTools** | Principle of least privilege. Each subagent gets ONLY needed tools. |
| **Hooks** | Lifecycle events: pre-tool-call, post-tool-call, pre-response, post-response. |
| **Task decomposition** | Sequential, parallel, hierarchical. Task tool spawns subtasks. |
| **Escalation** | Confidence thresholds, policy boundaries, error counts → hand off to human. |
| **Error-context feedback** | Pass failure details to retry (NOT blind retry with same input). |
| **Session management** | State preservation across interactions. --resume, fork_session. |
| **Orchestration topologies** | Sequential pipeline, parallel fan-out/fan-in, hierarchical delegation. |

### Domain 1 Formula

```
Agentic Architecture = 
  Agentic Loop (max_turns guard) +
  Coordinator-Subagent (allowedTools scoping) +
  Hooks (lifecycle guardrails) +
  Escalation (circuit breakers) +
  Error-context feedback (NOT blind retry)
```

---

## 3. Domain 2 Comprehensive Review (18%)

### Core Concepts Checklist

| Concept | Key Exam Points |
|---------|-----------------|
| **MCP architecture** | Client ↔ Server protocol. Servers expose tools, resources, prompts. |
| **.mcp.json** | Config file. Defines servers: name, command, args, environment. |
| **MCP tools** | Actions the model can take. Input schema (JSON Schema). Clear descriptions. |
| **MCP resources** | Data the model can read. URI-based. Parameterized templates. |
| **MCP prompts** | Server-defined prompt templates. Reusable workflow starters. |
| **Env var expansion** | `${VAR_NAME}` in .mcp.json. Keeps secrets out of version control. |
| **Built-in tools** | File ops, terminal, web search. Interact with MCP tools by precedence. |
| **Tool descriptions** | Clear, unambiguous, specific. Guide Claude's tool selection. |
| **tool_use / tool_result** | API-level tool calling. stop_reason: "tool_use" triggers execution. |
| **tool_choice** | auto / any / tool (specific). Programmatic enforcement. |

### Domain 2 Formula

```
Tool Design & MCP = 
  .mcp.json (server config + env vars) +
  Tool definitions (schema + description) +
  Resources (data exposure, URI-based) +
  tool_use → tool_result cycle +
  tool_choice (auto/any/tool for enforcement)
```

---

## 4. Domain 3 Comprehensive Review (20%)

### Core Concepts Checklist

| Concept | Key Exam Points |
|---------|-----------------|
| **CLAUDE.md hierarchy** | Global (~/.claude/CLAUDE.md) → Project (./CLAUDE.md) → Subdirectory (./src/CLAUDE.md) |
| **.claude/commands/** | Custom slash commands. Markdown files. Parameterized. Team-sharable. |
| **.claude/rules/** | Path-specific rules. Different rules for different code areas. |
| **.claude/skills/** | Reusable capabilities. Packaged knowledge for Claude Code. |
| **Plan mode** | Claude proposes plan before executing. Safety + visibility. Higher latency. |
| **-p / --print** | Non-interactive mode. REQUIRED for CI. Headless execution. |
| **--output-format json** | JSON output wrapper (no schema validation). |
| **--json-schema** | Enforced structured output. STRONGER than --output-format json. |
| **--resume** | Continue previous session. Preserves context. |
| **fork_session** | Branch conversation for exploration. Non-destructive. |
| **Hooks in CI** | Post-generation validation. Guardrails for automated pipelines. |

### Domain 3 Formula

```
Claude Code Config = 
  CLAUDE.md hierarchy (global → project → subdirectory) +
  .claude/ directory (commands + rules + skills) +
  CLI flags (-p, --json-schema, --resume, fork_session) +
  Hooks (lifecycle guardrails in CI) +
  Plan mode (propose before execute)
```

---

## 5. Domain 4 Comprehensive Review (20%)

### Core Concepts Checklist

| Concept | Key Exam Points |
|---------|-----------------|
| **tool_use** | Content block type. Contains: type, id, name, input. |
| **tool_result** | Response to tool_use. Contains: tool_use_id, content, is_error. |
| **tool_choice** | auto (flexible), any (force tool), tool (specific tool). |
| **JSON Schema** | type, properties, required, enum, description. In tools[].input_schema. |
| **Pydantic** | BaseModel → .model_json_schema() → tool definition → validate response. |
| **Few-shot prompting** | Example user/assistant pairs. Helps with ambiguous tasks. Not a substitute for schema. |
| **Message Batches API** | Bulk processing. Async: created → processing → completed. Cost advantages. |
| **stop_reason** | end_turn, max_tokens, stop_sequence, tool_use. Drives control flow. |
| **System prompts** | Role assignment, output constraints, behavioral instructions. |
| **Programmatic enforcement** | tool_choice + JSON Schema + Pydantic > vague instructions. THE KEY PRINCIPLE. |

### Domain 4 Formula

```
Prompt Engineering & Structured Output = 
  Programmatic enforcement (tool_choice + schema) +
  JSON Schema (type, properties, required) +
  Pydantic (model → schema → validate) +
  stop_reason handling (control flow) +
  Few-shot (supplement, not replace schema) +
  Batches (bulk async processing)
```

---

## 6. Domain 5 Comprehensive Review (15%)

### Core Concepts Checklist

| Concept | Key Exam Points |
|---------|-----------------|
| **Token monitoring** | Track input_tokens from response.usage. Threshold-based alerts. |
| **Context window** | Total capacity (input + output). NOT the same as max_tokens. |
| **max_tokens** | Controls OUTPUT length ONLY. Common exam distractor. |
| **Checkpointing** | Save state at milestones. Enables recovery, branching, compression. |
| **Context compression** | Summarization, key-fact extraction, dropping low-relevance. |
| **Compression timing** | >80% utilization, between phases, before subagent handoff, fan-in. |
| **Human review workflows** | Approval gates at critical decisions. Trigger: confidence, conflicts, domain. |
| **Review triggers** | Low confidence (<0.6), conflicts (>2), sensitive domain, destructive actions. |
| **Information provenance** | Source tracking through pipeline. Enables audit, debug, trust. |
| **Large codebase exploration** | File tree → dependency map → symbol search → targeted reading. |

### Domain 5 Formula

```
Context Management & Reliability = 
  Token monitoring (thresholds: 80/90/95%) +
  Checkpointing (save before compress) +
  Compression (summarize/extract/drop) +
  Human review (risk-proportional triggers) +
  Provenance (source → extraction → transformation → output) +
  Codebase exploration (broad → targeted)
```

---

## 7. Comprehensive Practice Exam (10 Questions)

### Question 1 — Domain 1 (Agentic Architecture)

> A coordinator-subagent system has a document processing subagent that times out on 5% of documents. After 3 retries with error-context feedback, the document still fails. What should happen next?
>
> **A)** Continue retrying until it succeeds
> **B)** Skip the document entirely and continue with available results
> **C)** Escalate to the coordinator, which logs the failure, marks the document as unprocessed, and continues the pipeline with a note about incomplete coverage
> **D)** Restart the entire pipeline from the beginning

<details><summary>Answer</summary>

**C** — After max retries (circuit breaker), escalate gracefully. The coordinator logs the failure (auditability), continues the pipeline (availability), and notes the gap (transparency). A is an infinite loop. B silently drops data. D is wasteful.

</details>

---

### Question 2 — Domain 1 (Agentic Architecture)

> In a multi-agent system, which orchestration topology is correct for: "Analyze 5 independent code files, then merge findings into a single report"?
>
> **A)** Sequential: analyze file 1 → analyze file 2 → ... → analyze file 5 → merge
> **B)** Parallel fan-out for analysis (5 concurrent), then fan-in to merge
> **C)** Hierarchical: create 5 sub-coordinators, each handling one file
> **D)** Single agent analyzes all 5 files in one context window

<details><summary>Answer</summary>

**B** — The analyses are independent (parallel fan-out). The merge requires all results (fan-in). A adds unnecessary latency. C over-engineers with unnecessary sub-coordinators. D may exceed context limits and violates single-responsibility.

</details>

---

### Question 3 — Domain 2 (Tool Design & MCP)

> An MCP server provides a `query_database` tool. The .mcp.json config includes `"env": {"DB_PASSWORD": "${DB_PASSWORD}"}`. A developer commits the .mcp.json to git. Is this a security issue?
>
> **A)** Yes — the password is exposed in the committed file
> **B)** No — `${DB_PASSWORD}` is a reference to an environment variable, not the actual password. The actual password stays in the environment.
> **C)** Yes — MCP servers should never access databases
> **D)** No — .mcp.json files are automatically excluded from git

<details><summary>Answer</summary>

**B** — Environment variable expansion (`${VAR_NAME}`) means the .mcp.json file contains only the REFERENCE, not the value. The actual secret is read from the environment at runtime. This is the correct pattern for secrets in MCP config. A would be true if the actual password were hardcoded (which is the anti-pattern).

</details>

---

### Question 4 — Domain 2 (Tool Design & MCP)

> A tool definition has this description: "Does stuff with data." Claude frequently calls the wrong tool. What's the fix?
>
> **A)** Add more few-shot examples of correct tool usage
> **B)** Rewrite the description to be specific and unambiguous: "Retrieves customer order history by customer_id. Returns array of orders with dates, items, and amounts."
> **C)** Set tool_choice to force this specific tool
> **D)** Remove all other tools so only this one is available

<details><summary>Answer</summary>

**B** — Tool descriptions guide Claude's tool selection. Vague descriptions cause misrouting. Specific, clear descriptions that state: what the tool does, what input it expects, and what it returns enable correct tool selection. A supplements but doesn't fix the root cause. C removes flexibility. D is impractical.

</details>

---

### Question 5 — Domain 3 (Claude Code Configuration)

> A project has rules in both `./CLAUDE.md` (root) and `./src/api/CLAUDE.md` (subdirectory). The root says "Use camelCase for all variables." The subdirectory says "Use snake_case for database-related variables." A developer asks Claude Code to generate code in `src/api/models.ts`. Which naming convention applies?
>
> **A)** camelCase only (root CLAUDE.md takes precedence)
> **B)** snake_case only (subdirectory CLAUDE.md overrides root completely)
> **C)** snake_case for database variables in src/api/ (subdirectory extends/overrides root for files in its scope)
> **D)** Error — conflicting CLAUDE.md files are not allowed

<details><summary>Answer</summary>

**C** — The CLAUDE.md hierarchy allows subdirectory files to extend or override project-level instructions for files within their scope. The subdirectory CLAUDE.md provides a more specific rule for its domain (database variables) while the root's general rule applies elsewhere. This is scoped configuration.

</details>

---

### Question 6 — Domain 3 (Claude Code Configuration)

> A CI pipeline needs Claude Code to generate a changelog from git commits and output it as structured JSON with fields: version, date, changes[], breaking_changes[]. Which command is correct?
>
> **A)** `claude "Generate changelog" --output-format json`
> **B)** `claude -p "Generate changelog from recent commits" --json-schema '{"type":"object","properties":{"version":{"type":"string"},"date":{"type":"string"},"changes":{"type":"array","items":{"type":"string"}},"breaking_changes":{"type":"array","items":{"type":"string"}}},"required":["version","date","changes","breaking_changes"]}'`
> **C)** `claude --resume "changelog-session"`
> **D)** `claude -p "Generate changelog" > changelog.json`

<details><summary>Answer</summary>

**B** — Three requirements met: (1) `-p` for non-interactive CI execution, (2) `--json-schema` for enforced structure with specific fields, (3) explicit schema defining the exact output format. A lacks `-p` (hangs in CI) and has no schema enforcement. C resumes a session rather than generating. D redirects stdout but has no schema guarantee.

</details>

---

### Question 7 — Domain 4 (Prompt Engineering & Structured Output)

> You want Claude to ALWAYS use the `extract_invoice` tool when processing documents, and the extracted data must include: vendor (string), amount (number), date (string), line_items (array). Which combination guarantees this?
>
> **A)** System prompt: "Always use extract_invoice tool" + JSON Schema on the tool
> **B)** `tool_choice: {type: "tool", name: "extract_invoice"}` + JSON Schema defining required fields in tools[].input_schema
> **C)** Few-shot examples showing correct tool usage + Pydantic validation after the fact
> **D)** `tool_choice: "auto"` + detailed tool description

<details><summary>Answer</summary>

**B** — `tool_choice: {type: "tool", name: "..."}` GUARANTEES Claude calls that tool (programmatic enforcement). JSON Schema in input_schema GUARANTEES the output structure. Together they provide complete enforcement. A uses vague instruction (anti-pattern). C validates after-the-fact but doesn't prevent incorrect behavior. D is auto (no guarantee).

</details>

---

### Question 8 — Domain 4 (Prompt Engineering & Structured Output)

> Claude returns stop_reason: "max_tokens" for an extraction task. The extracted JSON appears truncated. What happened and how do you fix it?
>
> **A)** The context window was full — compress the input
> **B)** Claude's response was cut short because max_tokens was too low. Increase max_tokens to allow a longer response.
> **C)** The tool_choice was set incorrectly
> **D)** The JSON Schema was too complex

<details><summary>Answer</summary>

**B** — `stop_reason: "max_tokens"` means Claude's OUTPUT was truncated because it reached the `max_tokens` limit before finishing. The fix is to increase `max_tokens` to allow a longer response. A confuses input context with output limit. C and D are unrelated to truncation.

</details>

---

### Question 9 — Domain 5 (Context Management)

> An agent session shows response.usage.input_tokens = 170,000 for a model with 200K context window. The agent still has work to do. What is the correct immediate action?
>
> **A)** Increase the model's context window
> **B)** Set max_tokens higher to compensate
> **C)** Checkpoint the current state, then compress older messages (keeping last 3-5 turns), reducing input tokens below 60% utilization
> **D)** Start a new session without preserving any context

<details><summary>Answer</summary>

**C** — At 85% utilization (170K/200K), the agent is in the "orange/red" zone. Correct action: (1) checkpoint for safety, (2) compress older context. This preserves progress while freeing space. A is not a configurable parameter. B confuses output with input. D loses all progress.

</details>

---

### Question 10 — Domain 5 (Context Management)

> In a multi-agent research system, the coordinator compresses 12 document analyses before passing to the synthesis subagent. What information MUST be preserved during compression?
>
> **A)** The full text of each document that was analyzed
> **B)** Key findings, confidence scores, and source attribution (provenance) for each analysis
> **C)** The step-by-step reasoning the document analysis subagent used
> **D)** The timestamps of when each API call was made

<details><summary>Answer</summary>

**B** — Compression must preserve: key findings (the actual conclusions), confidence (reliability indicator), and source attribution/provenance (traceability). A is the raw input (too large). C is reasoning detail (safe to drop). D is metadata not needed for synthesis.

</details>

---

## 8. Domain Confidence Tracker

### Self-Assessment Template

Rate your confidence 1-5 for each concept after reviewing:

| Domain | Concept | Confidence (1-5) | Notes |
|--------|---------|:-----------------:|-------|
| **D1 (27%)** | Agentic loop design | __ | |
| **D1** | Coordinator-subagent | __ | |
| **D1** | Hooks/lifecycle events | __ | |
| **D1** | Escalation patterns | __ | |
| **D1** | Task decomposition | __ | |
| **D1** | max_turns / circuit breakers | __ | |
| **D1** | Error-context feedback | __ | |
| **D2 (18%)** | .mcp.json configuration | __ | |
| **D2** | MCP tool definitions | __ | |
| **D2** | MCP resources vs. tools | __ | |
| **D2** | Env var expansion | __ | |
| **D2** | Built-in tools | __ | |
| **D2** | tool_use / tool_result | __ | |
| **D3 (20%)** | CLAUDE.md hierarchy | __ | |
| **D3** | .claude/commands/ | __ | |
| **D3** | .claude/rules/ | __ | |
| **D3** | .claude/skills/ | __ | |
| **D3** | Plan mode | __ | |
| **D3** | CLI flags (-p, --json-schema) | __ | |
| **D3** | Hooks in CI | __ | |
| **D4 (20%)** | tool_choice enforcement | __ | |
| **D4** | JSON Schema | __ | |
| **D4** | Pydantic validation | __ | |
| **D4** | stop_reason handling | __ | |
| **D4** | Few-shot prompting | __ | |
| **D4** | Message Batches API | __ | |
| **D5 (15%)** | Token monitoring | __ | |
| **D5** | Checkpointing | __ | |
| **D5** | Context compression | __ | |
| **D5** | Human review workflows | __ | |
| **D5** | Information provenance | __ | |
| **D5** | Large codebase exploration | __ | |

### Scoring Guide

- **5** — Can explain, implement, and identify anti-patterns confidently
- **4** — Solid understanding, might miss edge cases
- **3** — Know the concept but uncertain about exam-level details
- **2** — Vague understanding, needs review
- **1** — Cannot explain this concept

### Readiness Thresholds

| Average Score | Readiness | Action |
|:---:|:---:|---|
| 4.5+ | Exam ready | Take the exam with confidence |
| 4.0-4.4 | Nearly ready | Focus on concepts rated 3 or below |
| 3.5-3.9 | Needs work | Review weak domains, redo scenario challenges |
| < 3.5 | Not ready | Full domain review needed before exam |

---

## 9. Anti-Pattern Master List

### Domain 1 Anti-Patterns

| Anti-Pattern | Correct Pattern |
|-------------|-----------------|
| Loops without circuit breakers | max_turns guard on every agent |
| Blind retry (same input) | Error-context feedback with failure details |
| Coordinator performs work directly | Coordinator orchestrates, subagents execute |
| Subagent has all tools | allowedTools: principle of least privilege |
| No escalation path | Define escalation triggers and actions |
| Micromanaging subagents | Delegate and synthesize results |

### Domain 2 Anti-Patterns

| Anti-Pattern | Correct Pattern |
|-------------|-----------------|
| Hardcoded secrets in .mcp.json | `${ENV_VAR}` expansion |
| Vague tool descriptions | Specific: what it does, inputs, outputs |
| Duplicating tools across MCP and API | Clear separation of concerns |
| No input validation in schemas | Required fields, enum constraints, types |
| Overly broad tool access | Scope with allowedTools |

### Domain 3 Anti-Patterns

| Anti-Pattern | Correct Pattern |
|-------------|-----------------|
| Interactive mode in CI | `-p` / `--print` for non-interactive |
| Conflicting CLAUDE.md hierarchy | Clear scope: global → project → subdirectory |
| --output-format json when schema needed | `--json-schema` for enforcement |
| No max-turns in CI | Always set iteration limits |
| Slash commands in CI | Script with `-p` + prompt |

### Domain 4 Anti-Patterns

| Anti-Pattern | Correct Pattern |
|-------------|-----------------|
| Vague instructions ("please use JSON") | tool_choice + JSON Schema (programmatic enforcement) |
| Ignoring stop_reason | Handle all 4 values: end_turn, max_tokens, stop_sequence, tool_use |
| Few-shot as substitute for schema | Few-shot supplements, schema enforces |
| Not validating with Pydantic | tool_use response → Pydantic model validation |
| Treating max_tokens as context control | max_tokens = output limit only |

### Domain 5 Anti-Patterns

| Anti-Pattern | Correct Pattern |
|-------------|-----------------|
| No token monitoring | Track response.usage.input_tokens every turn |
| Compressing without checkpointing | Always checkpoint BEFORE compressing |
| Dropping provenance during compression | Source attribution survives all compression |
| Human review on everything | Risk-proportional triggers |
| Human review on nothing | At minimum: confidence, conflicts, domain sensitivity |
| Confusing max_tokens with context window | max_tokens = output; context = input + output capacity |

---

## 10. Final Exam Preparation Tips

### Exam Format Reminders

| Aspect | Detail |
|--------|--------|
| Questions | 60 multiple-choice |
| Time | 90 minutes (1.5 min per question avg) |
| Passing | 720/1000 |
| Scenarios | 4 of 6 randomly selected |
| Format | 1 correct answer, 3 distractors |

### The 6 Scenarios (4 randomly selected)

| # | Scenario | Primary Domains |
|---|----------|-----------------|
| 1 | Customer Support Resolution Agent | D2 + D1 |
| 2 | Code Generation with Claude Code | D3 |
| 3 | Multi-Agent Research System | D1 + D5 |
| 4 | Developer Productivity with Claude | D3 + D1 |
| 5 | Claude Code for Continuous Integration | D3 + D1 |
| 6 | Structured Data Extraction | D4 + D2 |

### Top 10 Exam Day Principles

1. **Programmatic enforcement > vague instruction** — tool_choice + schema beats "please do X"
2. **Error-context feedback > blind retry** — pass failure details, don't repeat same input
3. **max_turns = circuit breaker** — ALWAYS set on agents, prevents infinite loops
4. **max_tokens ≠ context window** — max_tokens limits OUTPUT only
5. **allowedTools = least privilege** — scope per subagent/CI stage
6. **Coordinator orchestrates, never performs** — dispatch/route/pass/trigger/escalate
7. **Checkpoint before compress** — save full state before losing detail
8. **Provenance survives compression** — source attribution always maintained
9. **-p is required for CI** — no interactive terminal in CI environments
10. **--json-schema > --output-format json** — schema enforces structure, format just wraps

### Distractor Recognition

The exam uses predictable distractor patterns. Watch for:

| Distractor Pattern | Why It's Wrong |
|-------------------|----------------|
| "Add a system prompt instruction to..." | Vague instruction, not programmatic enforcement |
| "Retry with the same input" | Blind retry anti-pattern |
| "Increase max_tokens" when context is the issue | Confuses output limit with input capacity |
| "Use --output-format json" when structure matters | No schema validation |
| "Let the agent decide" for high-stakes choices | Needs human review or explicit policy |
| "All tools available" in any restricted context | Violates least privilege |
| "Skip/ignore the failure" for critical checks | Silent data loss or security bypass |

### Time Management Strategy

```
60 questions in 90 minutes:
  First pass (60 min): Answer all questions you're confident about
  Second pass (20 min): Return to flagged/uncertain questions  
  Final review (10 min): Check for silly mistakes

Per question target:
  Easy (recognize pattern): 30-60 seconds
  Medium (apply concept): 60-90 seconds
  Hard (multi-concept scenario): 90-120 seconds
```

### Day-of Checklist

- [ ] Review Quick Reference Cards from Days 39-44
- [ ] Re-read anti-pattern master list (Section 9 above)
- [ ] Recall the 10 principles
- [ ] Know all 4 stop_reason values and their meanings
- [ ] Know the CLAUDE.md hierarchy (global → project → subdirectory)
- [ ] Know the difference between max_tokens and context window
- [ ] Know when to escalate vs. retry vs. compress
- [ ] Remember: programmatic enforcement is ALWAYS the answer over vague instruction

---

## Completion

**Congratulations!** You have completed the 45-day Claude Certified Architect exam preparation plan.

### What You've Covered

| Sprint | Days | Focus | Domains |
|--------|------|-------|---------|
| Sprint 1 | 1-15 | API Foundations | D4 (20%) + D2 (18%) |
| Sprint 2 | 16-30 | The Stack | D3 (20%) + D2 (18%) |
| Sprint 3 | 31-45 | Advanced Systems | D1 (27%) + D5 (15%) |

### Final Action Items

1. Record Sprint 3 checkpoint results in LEARNING_LOG.md
2. Update Domain Mastery Tracker confidence levels for all 5 domains
3. Address any concepts rated below 4 with targeted review
4. Schedule the exam when average confidence ≥ 4.0 across all domains
5. Night before: review Quick Reference Cards and anti-pattern list only (no new material)

**Target: 720/1000. You've prepared for this. Trust the preparation.**
