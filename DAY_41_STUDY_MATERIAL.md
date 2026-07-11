# Day 41: Scenario Deep-Dive — Claude Code for Continuous Integration

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20%) + Domain 1: Agentic Architecture & Orchestration (27%) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [CI Pipeline Integration Overview](#1-ci-pipeline-integration-overview)
2. [Hooks and Lifecycle Events for CI](#2-hooks-and-lifecycle-events-for-ci)
3. [Automated Checks via Hooks](#3-automated-checks-via-hooks)
4. [The --json-schema Flag for Structured CI Output](#4-the---json-schema-flag-for-structured-ci-output)
5. [Non-Interactive Mode: -p/--print](#5-non-interactive-mode--p--print)
6. [CI Pipeline Design Pattern](#6-ci-pipeline-design-pattern)
7. [Lifecycle Event Sequencing](#7-lifecycle-event-sequencing)
8. [Hooks as Guardrails](#8-hooks-as-guardrails)
9. [Anti-Patterns](#9-anti-patterns)
10. [Quick Reference Card](#10-quick-reference-card)
11. [Scenario Challenge](#11-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code CLI Reference | https://docs.anthropic.com/en/docs/claude-code/cli |
| Claude Code Hooks | https://docs.anthropic.com/en/docs/claude-code/hooks |
| Claude Code in CI/CD | https://docs.anthropic.com/en/docs/claude-code/ci-cd |
| Claude Agent SDK — Hooks | https://code.claude.com/docs/en/agent-sdk/hooks |

---

## 1. CI Pipeline Integration Overview

### Why Claude Code in CI?

Claude Code can be integrated into CI/CD pipelines to automate:
- Code generation (migrations, boilerplate, tests)
- Code review (PR analysis, security scanning)
- Documentation generation (API docs, changelogs)
- Structured reporting (test results, coverage analysis)

### The CI Execution Model

```
Traditional CI:        code → build → test → deploy
                            
Claude Code CI:   code → [Claude generates/reviews] → hook validation → structured output → pipeline consumes
```

### Key CI Requirements

| Requirement | Solution |
|-------------|----------|
| Non-interactive execution | `-p` / `--print` flag |
| Structured output for pipeline parsing | `--json-schema` flag |
| Automated validation after generation | Hooks (post-generation lifecycle events) |
| Restricted capabilities | `allowedTools` scoping |
| Deterministic behavior | Structured prompts + schema enforcement |

---

## 2. Hooks and Lifecycle Events for CI

### What Are Hooks?

Hooks are functions that execute at specific lifecycle points during Claude Code's operation. In CI, they act as automated quality gates.

### Lifecycle Event Types

| Event | When It Fires | CI Use Case |
|-------|---------------|-------------|
| **pre-tool-call** | Before Claude executes a tool | Validate the operation is safe |
| **post-tool-call** | After a tool completes | Check output quality |
| **pre-response** | Before Claude returns its response | Validate response format |
| **post-response** | After Claude's response is finalized | Run validation suite |
| **on-error** | When an error occurs | Log, alert, or halt pipeline |

### Hook Definition Structure

```json
{
  "hooks": {
    "post-response": [
      {
        "name": "validate-sql",
        "command": "sqlfluff lint --dialect postgres ${output_file}",
        "on_failure": "halt"
      },
      {
        "name": "check-destructive",
        "command": "grep -E 'DROP|TRUNCATE|DELETE FROM' ${output_file}",
        "on_failure": "warn"
      }
    ],
    "pre-tool-call": [
      {
        "name": "block-file-delete",
        "command": "check-tool-safety ${tool_name} ${tool_args}",
        "on_failure": "block"
      }
    ]
  }
}
```

### Hook Execution Flow in CI

```
Claude generates code
       │
       ▼
[post-response hook: lint check]
       │
       ├── PASS → continue
       └── FAIL → on_failure action (halt/warn/retry)
       │
       ▼
[post-response hook: security scan]
       │
       ├── PASS → continue  
       └── FAIL → on_failure action
       │
       ▼
Pipeline receives validated output
```

---

## 3. Automated Checks via Hooks

### Common CI Hook Patterns

#### Pattern 1: Lint After Generation

```json
{
  "name": "eslint-check",
  "event": "post-response",
  "command": "npx eslint --fix ${generated_files}",
  "on_failure": "halt",
  "description": "Run ESLint on all generated files"
}
```

#### Pattern 2: Type Check After Generation

```json
{
  "name": "typecheck",
  "event": "post-response", 
  "command": "npx tsc --noEmit",
  "on_failure": "halt",
  "description": "Verify TypeScript types are correct"
}
```

#### Pattern 3: Test Execution After Code Gen

```json
{
  "name": "run-tests",
  "event": "post-response",
  "command": "npm test -- --coverage",
  "on_failure": "halt",
  "description": "Run test suite and verify coverage"
}
```

#### Pattern 4: Security Scan (Destructive Operation Detection)

```json
{
  "name": "destructive-check",
  "event": "post-response",
  "command": "scripts/check-destructive-ops.sh ${output_file}",
  "on_failure": "halt",
  "description": "Block destructive SQL operations without explicit approval"
}
```

### Hook Chaining

Hooks execute in order. If any hook with `on_failure: "halt"` fails, subsequent hooks don't run.

```
Hook 1 (lint) → PASS → Hook 2 (typecheck) → PASS → Hook 3 (test) → PASS → Output
                                                                       │
                                                              FAIL → Pipeline stops
```

---

## 4. The --json-schema Flag for Structured CI Output

### Purpose

`--json-schema` forces Claude Code to output data conforming to a specific JSON Schema. This is essential for CI pipelines that need to parse Claude's output programmatically.

### Usage

```bash
claude -p "Generate a database migration for adding user_preferences table" \
  --json-schema '{"type":"object","properties":{"migration_sql":{"type":"string"},"rollback_sql":{"type":"string"},"risk_level":{"type":"string","enum":["low","medium","high"]},"affected_tables":{"type":"array","items":{"type":"string"}}},"required":["migration_sql","rollback_sql","risk_level","affected_tables"]}'
```

### Example Output (Guaranteed Structure)

```json
{
  "migration_sql": "CREATE TABLE user_preferences (...);",
  "rollback_sql": "DROP TABLE IF EXISTS user_preferences;",
  "risk_level": "low",
  "affected_tables": ["user_preferences"]
}
```

### Why --json-schema in CI?

| Without Schema | With Schema |
|---------------|-------------|
| Free-text output, unpredictable format | Guaranteed JSON structure |
| Pipeline must parse natural language | Pipeline directly consumes JSON fields |
| May fail downstream processing | Reliable machine-to-machine handoff |
| Non-deterministic | Schema-validated output |

### Connecting to Sprint 1 Concepts

`--json-schema` is the Claude Code CLI equivalent of `tool_choice: {type: "tool"}` + JSON Schema in the API. Same principle: **programmatic enforcement over vague instruction**.

---

## 5. Non-Interactive Mode: -p/--print

### The Problem

Claude Code's default mode is interactive (terminal-based). CI environments have no interactive terminal.

### The Solution

```bash
# -p / --print: Run Claude non-interactively
claude -p "Generate the migration script for ticket JIRA-1234"

# Combine with --json-schema for structured CI output
claude -p "Review this PR for security issues" --json-schema '...'

# Pipe input
cat src/api/routes.ts | claude -p "Identify security vulnerabilities"
```

### CLI Flags for CI (Complete Reference)

| Flag | Purpose | CI Use Case |
|------|---------|-------------|
| `-p` / `--print` | Non-interactive mode | Run in headless CI |
| `--json-schema` | Enforce output structure | Machine-parseable output |
| `--output-format json` | Output as JSON (less strict than schema) | General structured output |
| `--allowedTools` | Restrict available tools | Security in automated environments |
| `--max-turns` | Limit agent iterations | Prevent runaway in CI |

### Exam Key Distinction

| Flag | What It Does |
|------|-------------|
| `--output-format json` | Outputs response as JSON (format only, no schema validation) |
| `--json-schema` | Enforces a specific schema on the output (structure + validation) |

`--json-schema` is STRONGER than `--output-format json`. The exam tests this distinction.

---

## 6. CI Pipeline Design Pattern

### The Standard CI Pipeline with Claude Code

```
┌────────────────────────────────────────────────────────────────┐
│  CI PIPELINE: Claude Code Integration                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Stage 1: Generation                                          │
│  ┌──────────────────────────────────────────────────────┐     │
│  │ claude -p "Generate migration" --json-schema '{...}' │     │
│  └──────────────────────┬───────────────────────────────┘     │
│                         │                                      │
│  Stage 2: Hook-Triggered Validation                           │
│  ┌──────────────────────▼───────────────────────────────┐     │
│  │ post-response hooks:                                  │     │
│  │   1. SQL syntax validation (sqlfluff)                │     │
│  │   2. Destructive operation check                     │     │
│  │   3. Schema compatibility check                      │     │
│  └──────────────────────┬───────────────────────────────┘     │
│                         │                                      │
│  Stage 3: Structured Output                                   │
│  ┌──────────────────────▼───────────────────────────────┐     │
│  │ JSON output: {migration_sql, rollback_sql,            │     │
│  │              risk_level, affected_tables}             │     │
│  └──────────────────────┬───────────────────────────────┘     │
│                         │                                      │
│  Stage 4: Pipeline Consumption                                │
│  ┌──────────────────────▼───────────────────────────────┐     │
│  │ if risk_level == "high": require manual approval      │     │
│  │ else: auto-apply migration                            │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Pipeline as YAML (GitHub Actions Example)

```yaml
jobs:
  generate-migration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate Migration
        run: |
          claude -p "Generate migration for ${{ github.event.inputs.description }}" \
            --json-schema "${{ env.MIGRATION_SCHEMA }}" \
            --allowedTools "read_file,write_file" \
            --max-turns 5 \
            > migration_output.json
      
      - name: Check Risk Level
        run: |
          RISK=$(jq -r '.risk_level' migration_output.json)
          if [ "$RISK" == "high" ]; then
            echo "::warning::High-risk migration requires manual approval"
            exit 1
          fi
      
      - name: Apply Migration
        if: success()
        run: |
          SQL=$(jq -r '.migration_sql' migration_output.json)
          psql -c "$SQL"
```

---

## 7. Lifecycle Event Sequencing

### Correct Ordering for CI

```
PRE-GENERATION (setup)
  │  • Validate inputs
  │  • Check prerequisites
  │  • Load context (CLAUDE.md, relevant files)
  ▼
GENERATION (Claude executes)
  │  • Claude processes the prompt
  │  • Tools are called (if allowed)
  │  • Output is produced
  ▼
POST-GENERATION (validation via hooks)
  │  • Hook 1: Syntax validation
  │  • Hook 2: Security scan
  │  • Hook 3: Test execution
  │  • Hook 4: Schema conformance
  ▼
COMPLETION (reporting)
  │  • Structured output emitted
  │  • Pipeline proceeds or halts based on hook results
  ▼
PIPELINE CONSUMPTION (downstream)
     • Parse JSON output
     • Make deployment decisions
     • Update tracking systems
```

### Exam Key: Event Ordering Matters

The exam may ask about the correct sequencing of hooks. Remember:
- Pre-hooks run BEFORE Claude acts (validation of inputs, safety checks)
- Post-hooks run AFTER Claude acts (validation of outputs, quality checks)
- Hooks chain — failure in an earlier hook can prevent later hooks from running

---

## 8. Hooks as Guardrails

### The Guardrail Mental Model

In CI, hooks serve as **guardrails** — they don't guide Claude's behavior (that's what prompts and CLAUDE.md do), they VALIDATE the output and BLOCK unsafe results.

```
┌─────────────────────────────────────────┐
│  GUARDRAIL LAYERS IN CI                 │
├─────────────────────────────────────────┤
│  Layer 1: allowedTools (prevent)        │
│    → Claude CAN'T use restricted tools  │
│                                         │
│  Layer 2: CLAUDE.md (guide)             │
│    → Claude SHOULD follow conventions   │
│                                         │
│  Layer 3: Hooks (validate + block)      │
│    → Output MUST pass checks to proceed │
│                                         │
│  Layer 4: --json-schema (enforce)       │
│    → Output MUST conform to schema      │
└─────────────────────────────────────────┘
```

### Layered Defense for CI

| Layer | Mechanism | Enforcement Level |
|-------|-----------|-------------------|
| Prevention | `allowedTools` | Hard — tools not in list are unavailable |
| Guidance | CLAUDE.md + prompt | Soft — Claude follows but isn't forced |
| Validation | Hooks | Hard — output rejected if hooks fail |
| Structure | `--json-schema` | Hard — output must match schema |

---

## 9. Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Pattern |
|-------------|--------------|-----------------|
| Running Claude interactively in CI | CI has no terminal; hangs or fails | Use `-p` / `--print` for non-interactive |
| No hooks — trusting Claude's output blindly | Generated code may have bugs/security issues | Post-generation hooks validate output |
| Using `--output-format json` when you need schema | No structural guarantee | Use `--json-schema` for strict enforcement |
| All validation in external pipeline steps | Misses hook-based lifecycle integration | Use hooks for Claude-aware validation |
| No `--max-turns` in CI | Agent may loop indefinitely, blocking pipeline | Set explicit `max-turns` limit |
| Allowing all tools in CI | Security risk in automated environments | Scope with `--allowedTools` |
| Interactive slash commands in CI | Requires terminal input | Script with `-p` + prompt text |

---

## 10. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│         CLAUDE CODE FOR CI — QUICK REFERENCE                 │
├─────────────────────────────────────────────────────────────┤
│ ESSENTIAL CI FLAGS:                                          │
│   -p / --print         → Non-interactive execution          │
│   --json-schema '{}'   → Enforced structured output         │
│   --output-format json → JSON output (no schema validation) │
│   --allowedTools       → Restrict tool access               │
│   --max-turns N        → Circuit breaker for iterations     │
│                                                              │
│ HOOK EVENTS:                                                 │
│   pre-tool-call  → Validate before action                   │
│   post-tool-call → Check after action                       │
│   post-response  → Validate final output (most common CI)   │
│   on-error       → Handle failures                          │
│                                                              │
│ HOOK FAILURE ACTIONS:                                        │
│   halt  → Stop pipeline immediately                         │
│   warn  → Log warning, continue                             │
│   block → Prevent the specific action                       │
│   retry → Attempt again (with limit)                        │
│                                                              │
│ CI PIPELINE PATTERN:                                         │
│   Generation → Hook Validation → Structured Output →        │
│   Pipeline Consumption                                       │
│                                                              │
│ EXAM KEY:                                                    │
│   --json-schema > --output-format json (stricter)           │
│   Hooks = automated guardrails, not guidance                │
│   -p is REQUIRED for any CI usage of Claude Code            │
└─────────────────────────────────────────────────────────────┘
```

---

## 11. Scenario Challenge

### Practice Question (From Exam SPEC)

> You are integrating Claude Code into a CI pipeline that generates database migration scripts. The pipeline must: (1) generate the migration, (2) validate the SQL syntax, (3) check for destructive operations (DROP TABLE, TRUNCATE), and (4) output a structured JSON report. Which configuration correctly implements this?
>
> **A)** Write all four steps as instructions in CLAUDE.md and run `claude` interactively
>
> **B)** Use a post-generation hook to run SQL validation and destructive operation checks, then use `--json-schema` to enforce the report format, all invoked via `claude -p`
>
> **C)** Create four separate slash commands and run them sequentially in the CI script
>
> **D)** Use `--output-format json` to capture the migration, then run validation in a separate non-Claude pipeline step

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — This uses all the right CI patterns: post-generation hooks automate SQL validation and destructive operation checks as part of Claude Code's lifecycle. `--json-schema` enforces the structured report format. `-p` enables headless CI execution. All in one integrated flow.
- **A is wrong** — Interactive mode fails in CI (no terminal). CLAUDE.md provides guidance but doesn't enforce validation.
- **C is wrong** — Slash commands require interactive invocation. You can't run them in a headless CI environment.
- **D is wrong** — While technically workable, it separates validation from Claude Code's lifecycle (no hooks). `--output-format json` doesn't enforce a schema. The exam prefers hook-based automation.

**Key Concepts Tested:**
- Non-interactive execution (`-p`) (Domain 3)
- Hooks for automated validation (Domain 1/3)
- `--json-schema` for structured output (Domain 3)
- CI pipeline design patterns (Domain 3 + Domain 1)

</details>

### Bonus Practice Question

> A CI pipeline uses Claude Code to generate unit tests. After generation, the pipeline needs to verify that: (a) the tests compile, (b) the tests pass, and (c) code coverage exceeds 80%. In what order should lifecycle hooks execute these checks?
>
> **A)** Coverage check → test execution → compilation
>
> **B)** Compilation → test execution → coverage check
>
> **C)** Test execution → compilation → coverage check
>
> **D)** All three in parallel for faster CI feedback

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — Hooks must chain in dependency order: compilation must succeed before tests can run, and tests must run before coverage can be measured. Each step depends on the previous one succeeding.
- **A is wrong** — Can't check coverage before running tests, can't run tests before compiling.
- **C is wrong** — Can't execute tests that don't compile.
- **D is wrong** — These operations have dependencies; they can't run in parallel. Compilation must precede execution which must precede coverage analysis.

**Key Concept:** Lifecycle event sequencing — hooks execute in order, and dependency-chain ordering is critical in CI pipelines.

</details>

---

## Key Takeaways for Exam Day

1. **`-p` is non-negotiable for CI** — without it, Claude Code hangs waiting for terminal input
2. **`--json-schema` > `--output-format json`** — schema enforces structure, format just wraps in JSON
3. **Hooks are post-generation guardrails** — they validate, they don't guide
4. **Hook ordering follows dependency chains** — compile before test, test before coverage
5. **`allowedTools` restricts what Claude can do in CI** — security through least privilege
6. **CI pipelines are specialized agentic loops** — generation → validation → output → consumption
