# Day 42: Scenario Deep-Dive — Claude Code for CI (Advanced Patterns)

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20%) + Domain 1: Agentic Architecture & Orchestration (27%) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Advanced Hook Patterns](#1-advanced-hook-patterns)
2. [Error Handling in CI Hooks](#2-error-handling-in-ci-hooks)
3. [Multi-Stage CI Pipelines](#3-multi-stage-ci-pipelines)
4. [--json-schema for Structured CI Reports](#4---json-schema-for-structured-ci-reports)
5. [Combining Hooks with allowedTools](#5-combining-hooks-with-allowedtools)
6. [Security Considerations in Automated Environments](#6-security-considerations-in-automated-environments)
7. [CI as Specialized Agentic Loops](#7-ci-as-specialized-agentic-loops)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code Hooks Reference | https://docs.anthropic.com/en/docs/claude-code/hooks |
| Claude Code Security | https://docs.anthropic.com/en/docs/claude-code/security |
| Claude Agent SDK — allowedTools | https://code.claude.com/docs/en/agent-sdk/allowed-tools |
| CI/CD Best Practices | https://docs.anthropic.com/en/docs/claude-code/ci-cd |

---

## 1. Advanced Hook Patterns

### Conditional Hooks

Hooks that only trigger based on specific conditions — not every generation needs every check.

```json
{
  "hooks": {
    "post-response": [
      {
        "name": "sql-validation",
        "command": "sqlfluff lint ${output_file}",
        "condition": "file_extension == '.sql'",
        "on_failure": "halt"
      },
      {
        "name": "ts-typecheck",
        "command": "npx tsc --noEmit",
        "condition": "file_extension in ['.ts', '.tsx']",
        "on_failure": "halt"
      },
      {
        "name": "security-scan",
        "command": "scripts/security-check.sh ${output_file}",
        "condition": "path contains 'auth' OR path contains 'security'",
        "on_failure": "halt"
      }
    ]
  }
}
```

### Conditional Trigger Patterns

| Condition Type | Example | Use Case |
|---------------|---------|----------|
| File type | `*.sql`, `*.ts`, `*.py` | Language-specific validation |
| Path pattern | `src/auth/**`, `db/migrations/**` | Security-sensitive areas |
| Content pattern | Contains `DROP`, `DELETE`, `TRUNCATE` | Destructive operation detection |
| Change scope | Modified files > 5 | Large change review |
| Risk metadata | `risk_level == "high"` | Escalation triggers |

### Hook Composition Patterns

#### Sequential with Early Exit

```
Hook A (lint) ──PASS──→ Hook B (type) ──PASS──→ Hook C (test)
     │                       │                       │
     FAIL                    FAIL                    FAIL
     │                       │                       │
     ▼                       ▼                       ▼
  HALT pipeline          HALT pipeline          HALT pipeline
```

#### Parallel Independent Checks

```
                    ┌── Hook A (lint) ─────── result_a
                    │
Generation output ──┼── Hook B (security) ── result_b
                    │
                    └── Hook C (style) ────── result_c
                    
                    All must PASS for pipeline to continue
```

#### Tiered Hooks (Warn then Block)

```json
{
  "hooks": {
    "post-response": [
      {
        "name": "soft-check-complexity",
        "command": "complexity-check ${output_file}",
        "on_failure": "warn",
        "description": "Warn if cyclomatic complexity > 10"
      },
      {
        "name": "hard-check-security",
        "command": "security-scan ${output_file}",
        "on_failure": "halt",
        "description": "Block if security vulnerabilities found"
      }
    ]
  }
}
```

---

## 2. Error Handling in CI Hooks

### The Three Hook Failure Strategies

| Strategy | Behavior | When to Use |
|----------|----------|-------------|
| **Halt** | Stop the entire pipeline immediately | Critical failures: security issues, broken builds |
| **Log and Continue** | Record the failure, proceed anyway | Non-critical: style warnings, minor issues |
| **Escalate** | Notify humans, pause for approval | High-stakes: destructive operations, policy violations |

### Decision Matrix for Hook Failures

```
Hook fails →
  ├── Is it a security issue?
  │     YES → HALT immediately
  │
  ├── Is it a correctness issue (won't compile/run)?
  │     YES → HALT immediately
  │
  ├── Is it a quality issue (style, complexity)?
  │     YES → WARN and continue
  │
  ├── Is it a policy issue (destructive ops, scope creep)?
  │     YES → ESCALATE to human
  │
  └── Is it a transient issue (network timeout, service unavailable)?
        YES → RETRY with circuit breaker (max 3 attempts)
```

### Retry with Circuit Breaker for Transient Failures

```json
{
  "name": "notify-security-team",
  "command": "curl -X POST $SECURITY_WEBHOOK --data '${findings}'",
  "on_failure": "retry",
  "retry_config": {
    "max_attempts": 3,
    "backoff": "exponential",
    "initial_delay_ms": 1000,
    "on_exhaustion": "escalate"
  }
}
```

### Error Propagation in Hook Chains

```python
# Hook chain error propagation model
def execute_hook_chain(hooks, output):
    results = []
    for hook in hooks:
        try:
            result = execute_hook(hook, output)
            results.append(result)
        except HookFailure as e:
            if hook.on_failure == "halt":
                raise PipelineHalt(f"Hook '{hook.name}' failed: {e}")
            elif hook.on_failure == "warn":
                log_warning(f"Hook '{hook.name}': {e}")
                continue
            elif hook.on_failure == "escalate":
                submit_for_review(hook, e, output)
                raise PipelinePaused(f"Awaiting human review: {hook.name}")
            elif hook.on_failure == "retry":
                retry_with_backoff(hook, output, max_attempts=3)
    return results
```

---

## 3. Multi-Stage CI Pipelines

### Stage Definitions

| Stage | Claude Code Role | Hooks | Output |
|-------|-----------------|-------|--------|
| **Code Review** | Analyze PR for issues | Security scan, style check | Review JSON |
| **Code Generation** | Generate migrations, tests | Lint, type-check, test | Generated files |
| **Documentation** | Generate API docs | Link validation, format check | Markdown/HTML |
| **Testing** | Generate test cases | Compilation, execution | Test results JSON |

### Multi-Stage Pipeline Architecture

```yaml
# GitHub Actions multi-stage Claude Code pipeline
jobs:
  stage-review:
    runs-on: ubuntu-latest
    outputs:
      review-result: ${{ steps.review.outputs.result }}
    steps:
      - name: PR Review
        id: review
        run: |
          claude -p "Review this PR diff for security and correctness" \
            --json-schema '{"type":"object","properties":{"issues":{"type":"array"},"risk_level":{"type":"string","enum":["low","medium","high"]},"approve":{"type":"boolean"}},"required":["issues","risk_level","approve"]}' \
            --allowedTools "read_file,grep_search" \
            --max-turns 3 > review.json
            
  stage-generate:
    needs: stage-review
    if: ${{ fromJson(needs.stage-review.outputs.review-result).approve == true }}
    runs-on: ubuntu-latest
    steps:
      - name: Generate Tests
        run: |
          claude -p "Generate unit tests for changed files" \
            --json-schema '{"type":"object","properties":{"test_files":{"type":"array"},"coverage_estimate":{"type":"number"}},"required":["test_files"]}' \
            --allowedTools "read_file,write_file" \
            --max-turns 5

  stage-validate:
    needs: stage-generate
    runs-on: ubuntu-latest
    steps:
      - name: Run Generated Tests
        run: npm test -- --coverage
```

### Stage Dependencies and Gating

```
Review ──[approve=true]──→ Generate ──[hooks pass]──→ Validate ──[tests pass]──→ Deploy
   │                           │                          │
   │[approve=false]            │[hook fails]              │[tests fail]
   ▼                           ▼                          ▼
 Block PR                  Halt pipeline              Report failures
```

---

## 4. --json-schema for Structured CI Reports

### Migration Report Schema

```json
{
  "type": "object",
  "properties": {
    "migration_sql": {"type": "string", "description": "The forward migration SQL"},
    "rollback_sql": {"type": "string", "description": "The rollback SQL"},
    "risk_level": {"type": "string", "enum": ["low", "medium", "high"]},
    "affected_tables": {"type": "array", "items": {"type": "string"}},
    "breaking_changes": {"type": "boolean"},
    "estimated_duration": {"type": "string"},
    "requires_downtime": {"type": "boolean"}
  },
  "required": ["migration_sql", "rollback_sql", "risk_level", "affected_tables", "breaking_changes"]
}
```

### Test Result Schema

```json
{
  "type": "object",
  "properties": {
    "test_files_generated": {"type": "array", "items": {"type": "string"}},
    "total_tests": {"type": "integer"},
    "test_categories": {
      "type": "object",
      "properties": {
        "unit": {"type": "integer"},
        "integration": {"type": "integer"},
        "edge_cases": {"type": "integer"}
      }
    },
    "coverage_estimate": {"type": "number", "minimum": 0, "maximum": 100},
    "uncovered_paths": {"type": "array", "items": {"type": "string"}}
  },
  "required": ["test_files_generated", "total_tests", "coverage_estimate"]
}
```

### PR Review Schema

```json
{
  "type": "object",
  "properties": {
    "summary": {"type": "string"},
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": {"type": "string", "enum": ["critical", "warning", "info"]},
          "file": {"type": "string"},
          "line": {"type": "integer"},
          "description": {"type": "string"},
          "suggestion": {"type": "string"}
        }
      }
    },
    "risk_level": {"type": "string", "enum": ["low", "medium", "high"]},
    "approve": {"type": "boolean"},
    "blocking_issues_count": {"type": "integer"}
  },
  "required": ["summary", "issues", "risk_level", "approve"]
}
```

---

## 5. Combining Hooks with allowedTools

### The Principle: Defense in Depth

`allowedTools` restricts WHAT Claude can do. Hooks validate WHAT Claude produced. Together they provide complete CI safety.

```
allowedTools = ["read_file", "write_file", "grep_search"]
  → Claude CANNOT execute shell commands, delete files, or access network

Hooks validate the output:
  → Generated code MUST pass lint
  → Generated code MUST NOT contain secrets
  → Generated code MUST compile
```

### CI-Specific Tool Scoping

| CI Stage | allowedTools | Denied (Implicit) |
|----------|-------------|-------------------|
| Code Review | `read_file`, `grep_search` | `write_file`, `execute_command` |
| Code Generation | `read_file`, `write_file` | `execute_command`, `web_search` |
| Test Generation | `read_file`, `write_file` | `execute_command`, `delete_file` |
| Documentation | `read_file`, `write_file`, `web_search` | `execute_command`, `delete_file` |

### Implementation

```bash
# Code review: read-only access
claude -p "Review src/auth.ts for security issues" \
  --allowedTools "read_file,grep_search" \
  --json-schema '...' \
  --max-turns 3

# Code generation: read + write, no execution
claude -p "Generate migration for user_preferences table" \
  --allowedTools "read_file,write_file" \
  --json-schema '...' \
  --max-turns 5
```

---

## 6. Security Considerations in Automated Environments

### Threat Model for Claude in CI

| Threat | Mitigation |
|--------|-----------|
| Claude writes malicious code | Post-generation security scan hook |
| Claude accesses secrets | No secret-related tools in `allowedTools` |
| Claude runs destructive commands | Exclude `execute_command` from `allowedTools` |
| Claude infinite-loops | `--max-turns` circuit breaker |
| Output contains injections | `--json-schema` validates structure |
| Prompt injection via PR content | `allowedTools` prevents dangerous actions even if prompted |

### Security Layers for CI

```
┌─────────────────────────────────────────────────┐
│  SECURITY LAYERS (innermost = strongest)        │
├─────────────────────────────────────────────────┤
│                                                  │
│  1. allowedTools (prevent)                      │
│     └── Claude literally cannot call tools      │
│         not in the allowed list                 │
│                                                  │
│  2. --max-turns (limit)                         │
│     └── Bounds execution time and iterations    │
│                                                  │
│  3. Hooks (validate)                            │
│     └── Output checked against security rules   │
│                                                  │
│  4. --json-schema (structure)                   │
│     └── Output must conform to expected format  │
│                                                  │
│  5. Pipeline gating (decide)                    │
│     └── risk_level determines next action       │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Secrets Management

```bash
# NEVER expose secrets to Claude in CI
# BAD:
claude -p "Deploy using API key: $API_KEY"

# GOOD: Claude generates; pipeline applies with secrets
claude -p "Generate deployment configuration" \
  --json-schema '...' \
  --allowedTools "read_file" > config.json

# Pipeline (not Claude) handles secrets
deploy --config config.json --api-key $API_KEY
```

---

## 7. CI as Specialized Agentic Loops

### The Conceptual Connection

CI pipelines with Claude Code ARE agentic loops — just with extra guardrails.

| Agentic Loop Concept | CI Pipeline Equivalent |
|---------------------|----------------------|
| Agent prompt | CLI prompt via `-p` |
| Tool use | `allowedTools`-scoped operations |
| max_turns | `--max-turns` flag |
| Lifecycle hooks | Post-generation validation hooks |
| Structured output | `--json-schema` enforcement |
| Escalation | Pipeline gating (halt, notify, require approval) |
| Circuit breaker | Retry limits + halt on failure |

### Mental Model

```
Standard Agentic Loop:
  prompt → reason → act → observe → repeat/stop

CI Agentic Loop:
  -p prompt → generate → hooks validate → --json-schema output → pipeline consumes
       │                     │                                         │
       │                  (guardrails)                            (consumption)
       │                     │                                         │
  [allowedTools]      [halt/warn/retry]                    [gate on risk_level]
```

### Why This Matters for the Exam

The exam tests whether you recognize that CI integration is NOT a separate concept — it's an APPLICATION of agentic architecture with specific tooling. Questions may present CI scenarios and expect you to apply agentic loop principles (max_turns, hooks, circuit breakers, escalation).

---

## 8. Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Pattern |
|-------------|--------------|-----------------|
| Retrying hook indefinitely | Blocks pipeline forever | Circuit breaker: max 3 retries, then escalate |
| Ignoring hook failures for speed | Security/quality issues slip through | Halt on critical, warn on minor |
| All tools available in CI | Security risk | `allowedTools` scoped per stage |
| No structured output schema | Pipeline can't parse results | `--json-schema` for every CI stage |
| Secrets in prompts | Exposed to Claude unnecessarily | Generate config; pipeline applies secrets |
| Single monolithic CI stage | All-or-nothing failure mode | Multi-stage with clear gating |
| No max-turns in CI | Runaway agent, cost explosion | Always set `--max-turns` |
| Approval gate on every action | Pipeline too slow, defeats automation | Gate only high-risk operations |

---

## 9. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│        CLAUDE CODE CI — ADVANCED PATTERNS                    │
├─────────────────────────────────────────────────────────────┤
│ HOOK FAILURE STRATEGIES:                                     │
│   halt     → Critical failure, stop pipeline                │
│   warn     → Log and continue (non-critical)                │
│   escalate → Pause, notify human, await approval            │
│   retry    → Backoff retry with circuit breaker (max 3)     │
│                                                              │
│ CONDITIONAL HOOKS:                                           │
│   File type:     *.sql → SQL validation                     │
│   Path pattern:  auth/** → security scan                    │
│   Content match: DROP/TRUNCATE → destructive check          │
│                                                              │
│ MULTI-STAGE PIPELINE:                                        │
│   Review → [gate] → Generate → [hooks] → Validate → Deploy │
│                                                              │
│ SECURITY LAYERS (defense in depth):                          │
│   1. allowedTools (prevent)                                  │
│   2. --max-turns (limit)                                     │
│   3. Hooks (validate)                                        │
│   4. --json-schema (structure)                               │
│   5. Pipeline gating (decide)                                │
│                                                              │
│ EXAM KEY:                                                    │
│   CI pipelines ARE agentic loops with extra guardrails       │
│   Circuit breaker = retry limit + escalation                 │
│   Never retry indefinitely (anti-pattern)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Practice Question (From Exam SPEC)

> Your CI pipeline uses Claude Code to review pull requests. A post-review hook checks if Claude flagged any security issues. If security issues are found, the pipeline should block the PR and notify the security team. If no issues are found, the pipeline should auto-approve. The hook occasionally fails due to network timeouts when calling the notification service. What is the correct error handling strategy?
>
> **A)** Retry the hook indefinitely until the notification service responds
>
> **B)** Implement a circuit breaker: retry up to 3 times with exponential backoff, then escalate to a human reviewer if the hook still fails
>
> **C)** Ignore the hook failure and auto-approve the PR since the review itself completed
>
> **D)** Fail the entire pipeline and require a manual restart

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — The circuit breaker pattern (limited retries with exponential backoff, then escalation) is the standard approach for transient failures. It prevents infinite loops while ensuring security issues are not silently ignored. Escalation to a human reviewer is the safe fallback when automated notification fails.
- **A is wrong** — Retrying indefinitely is a loop without a circuit breaker (classic anti-pattern). It blocks the pipeline forever if the service is down.
- **C is wrong** — Ignoring a security-critical hook failure means potential security issues pass through unreviewed. This violates the principle that security checks should halt or escalate, never be silently skipped.
- **D is wrong** — Requiring manual restart is overly disruptive. A transient network issue shouldn't require human intervention to restart the entire pipeline — it should retry and then escalate gracefully.

**Key Concepts Tested:**
- Circuit breaker pattern (Domain 1)
- Hook failure handling strategies (Domain 3)
- Anti-pattern: infinite retry loops (Domain 1)
- Security-critical operations require escalation, not silent failure (Domain 3)

</details>

### Bonus Practice Question

> A multi-stage CI pipeline uses Claude Code for: (1) PR review, (2) test generation, and (3) documentation updates. Each stage needs different tool access. Which `allowedTools` configuration follows the principle of least privilege?
>
> **A)** All stages: `read_file, write_file, execute_command, web_search, delete_file`
>
> **B)** PR review: `read_file, grep_search` | Test gen: `read_file, write_file` | Docs: `read_file, write_file, web_search`
>
> **C)** All stages: `read_file, write_file` (uniform for simplicity)
>
> **D)** PR review: `read_file, write_file, execute_command` | Test gen: all tools | Docs: all tools

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — Each stage gets ONLY the tools it needs:
  - PR review only reads code (no writing needed)
  - Test generation reads existing code and writes test files
  - Documentation may need web search for API references, plus read/write
- **A is wrong** — Giving all tools to all stages violates least privilege. `execute_command` and `delete_file` are dangerous in CI.
- **C is wrong** — PR review doesn't need `write_file`, and docs may need `web_search`. Uniform scoping is either too permissive or too restrictive.
- **D is wrong** — PR review shouldn't write files or execute commands. "All tools" is never the right answer for CI.

**Key Concept:** `allowedTools` should be scoped per stage based on what each stage actually needs to accomplish.

</details>

---

## Key Takeaways for Exam Day

1. **Circuit breaker for transient failures** — retry (max 3) + exponential backoff + escalate on exhaustion
2. **Hook failure strategy depends on severity** — halt (critical), warn (minor), escalate (policy), retry (transient)
3. **Conditional hooks target specific file types/paths** — not every check applies to every file
4. **Multi-stage pipelines gate between stages** — review must pass before generation begins
5. **allowedTools scoped per CI stage** — principle of least privilege, defense in depth
6. **CI IS an agentic loop** — same patterns apply: max_turns, hooks, circuit breakers, escalation
7. **Never expose secrets to Claude** — Claude generates; the pipeline applies secrets
