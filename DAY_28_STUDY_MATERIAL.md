# Day 28: Code Generation with Claude Code (Advanced Patterns)

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows + Domain 1 (intro) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Scenario** | Code Generation with Claude Code (Advanced) |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Multi-File Code Generation with Plan Mode](#1-multi-file-code-generation-with-plan-mode)
2. [--print for Batch CI Generation](#2---print-for-batch-ci-generation)
3. [CLAUDE.md TDD Patterns](#3-claudemd-tdd-patterns)
4. [Introduction to Agentic Patterns](#4-introduction-to-agentic-patterns)
5. [.claude/skills/ for Generation Patterns](#5-claudeskills-for-generation-patterns)
6. [Advanced Generation Strategies](#6-advanced-generation-strategies)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code Skills | https://docs.anthropic.com/en/docs/claude-code/skills |
| Claude Code CLI Reference | https://docs.anthropic.com/en/docs/claude-code/cli |
| Claude Code Configuration | https://docs.anthropic.com/en/docs/claude-code/configuration |

---

## 1. Multi-File Code Generation with Plan Mode

### Why Plan Mode for Multi-File Generation?

When generating features that span multiple files, plan mode provides:
- **Visibility** — See all files that will be created or modified
- **Coherence** — Verify the approach is internally consistent
- **Safety** — Catch structural issues before code is written
- **Coordination** — Ensure imports, exports, and types align

### Multi-File Generation Flow

```
┌─────────────────────────────────────────────────────────────┐
│            MULTI-FILE GENERATION WITH PLAN MODE              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. User request: "Generate a complete auth module"          │
│  2. Claude analyzes existing codebase patterns               │
│  3. Claude produces PLAN (not code yet):                     │
│     ├── src/auth/auth.service.ts (new)                      │
│     ├── src/auth/auth.controller.ts (new)                   │
│     ├── src/auth/auth.middleware.ts (new)                   │
│     ├── src/auth/types.ts (new)                             │
│     ├── src/auth/auth.test.ts (new)                         │
│     └── src/routes/index.ts (modify: add auth routes)       │
│  4. User reviews plan → approves or adjusts                 │
│  5. Claude executes plan — generates all files              │
│  6. Validation: tsc + tests + lint                          │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Plan Output Structure

```
📋 Plan: Generate Authentication Module

📁 Files to CREATE (5):
  ✨ src/auth/auth.service.ts
     - AuthService class with login, register, validateToken methods
     - Uses bcrypt for password hashing, jose for JWT
     
  ✨ src/auth/auth.controller.ts  
     - Express route handlers: POST /login, POST /register, GET /me
     - Input validation with Zod schemas
     
  ✨ src/auth/auth.middleware.ts
     - requireAuth middleware: validates JWT from Authorization header
     - extractUser middleware: optional user extraction
     
  ✨ src/auth/types.ts
     - AuthUser, LoginRequest, RegisterRequest, TokenPayload interfaces
     
  ✨ src/auth/auth.test.ts
     - Unit tests for AuthService
     - Integration tests for auth endpoints

📝 Files to MODIFY (1):
  ✏️ src/routes/index.ts
     - Import and mount authController at /api/auth

🔗 Dependencies:
  - bcrypt, jose (must be installed)
  - Existing: express, zod (already available)

Shall I proceed?
```

### Coordinating Multi-File Changes

Key challenge: files must be **mutually consistent**:

```typescript
// types.ts exports → used by service.ts → used by controller.ts → imported in routes/index.ts
// Plan mode ensures this chain is correct BEFORE writing code
```

---

## 2. --print for Batch CI Generation

### Batch Code Generation in CI

Using `--print` mode in CI enables automated code generation as part of build pipelines:

```bash
# Generate multiple artifacts in a single CI step
claude -p "Generate OpenAPI spec from our route handlers in src/routes/" --output-format json

# Generate typed client from API schema
claude -p "Generate a TypeScript API client from docs/openapi.yaml following src/api-client/ patterns" --output-format json

# Generate database migration from model changes
claude -p "Compare src/models/ with current schema and generate migration SQL" --output-format json
```

### CI Pipeline Pattern: Generate → Validate → Commit

```yaml
# .github/workflows/codegen.yml
name: Auto-generate Code Artifacts
on:
  push:
    branches: [main]
    paths: ['src/models/**', 'src/routes/**']

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate API types
        run: |
          RESULT=$(claude -p "Generate TypeScript types from src/models/ into src/generated/types.ts" --output-format json)
          echo "$RESULT" | jq -r '.result' > /tmp/gen-output.txt
      
      - name: Validate generated code
        run: npx tsc --noEmit
      
      - name: Run tests
        run: npx vitest run
      
      - name: Commit if changed
        run: |
          git diff --quiet || (
            git add src/generated/
            git commit -m "chore: auto-generate types from models"
            git push
          )
```

### Batch Processing Multiple Files

```bash
#!/bin/bash
# generate-all-tests.sh — Generate tests for files without them

for file in src/services/*.ts; do
  test_file="${file%.ts}.test.ts"
  if [ ! -f "$test_file" ]; then
    echo "Generating tests for $file..."
    claude -p "Generate comprehensive unit tests for $file following our test conventions" \
      --output-format json | jq -r '.result' > "$test_file"
  fi
done

# Validate all generated tests
npx vitest run
```

### Key Constraints in CI/--print Mode

| Constraint | Why | Solution |
|-----------|-----|----------|
| No interactive input | CI has no terminal | Use -p flag |
| No plan mode approval | Can't click "approve" | Plan mode skipped in -p mode |
| Must be deterministic | CI should be reproducible | Use --json-schema for structure |
| Limited tool access | Security in automation | Use --allowedTools |
| Must handle errors | Pipeline needs to proceed | Check exit codes and JSON output |

---

## 3. CLAUDE.md TDD Patterns

### Test-Driven Generation

Configure CLAUDE.md to enforce TDD (Test-Driven Development) patterns in code generation:

```markdown
# CLAUDE.md

## Test-Driven Development Rules

### Generation Order
1. ALWAYS write tests BEFORE implementation
2. Tests define the expected behavior contract
3. Implementation should make tests pass
4. Never modify tests to match implementation (modify implementation instead)

### TDD Workflow
When generating a new feature:
1. Generate test file FIRST with all expected behaviors
2. Run tests — they should all FAIL (red phase)
3. Generate implementation to make tests pass (green phase)
4. Refactor if needed while keeping tests green

### Test Quality Gates
- All tests must pass after generation: `vitest run`
- Type checking must pass: `tsc --noEmit`
- No test should be `it.skip()` or `xit()`
- Coverage check: new code must have >80% coverage
```

### TDD Generation Example

```
User: "Generate a EmailValidator utility"

Claude (following TDD CLAUDE.md):

Step 1 — Write tests first:
  ✨ src/utils/email-validator.test.ts
  
Step 2 — Run tests (they fail):
  ❌ 6 tests FAIL (expected — no implementation yet)

Step 3 — Write implementation:
  ✨ src/utils/email-validator.ts

Step 4 — Run tests again:
  ✅ 6 tests PASS

Step 5 — Type check:
  ✅ tsc --noEmit passes
```

### TDD-Focused Slash Command

```markdown
<!-- .claude/commands/tdd-generate.md -->
# TDD Generate

Generate code using strict Test-Driven Development:

1. Generate the test file first with comprehensive test cases
2. Show me the tests and wait for approval
3. After approval, run tests to confirm they fail (RED phase)
4. Generate the implementation (GREEN phase)
5. Run tests to confirm they pass
6. Ask if refactoring is desired

Never skip the red-green cycle. If tests pass immediately (before implementation),
the tests are likely not testing the right things — regenerate them.
```

---

## 4. Introduction to Agentic Patterns

### How Code Generation Mirrors Agentic Loops

Claude Code's plan-then-execute pattern is a **precursor** to the agentic architecture patterns covered in Sprint 3. Understanding this connection is key for the exam.

### The Parallel

```
CODE GENERATION (Sprint 2):                AGENTIC LOOP (Sprint 3):
  Plan (propose changes)                     Reason (analyze task)
  Review (human approves)                    Decide (pick next action)
  Execute (write code)                       Act (call tool)
  Validate (run tests)                       Observe (check result)
  Iterate (refine if needed)                 Loop or stop
```

### Key Agentic Concepts Preview

| Agentic Concept | Code Gen Equivalent |
|----------------|---------------------|
| **Agentic loop** | Generate → validate → refine cycle |
| **max_turns guard** | Maximum refinement iterations before stopping |
| **Tool selection** | Claude choosing which file to read/write |
| **Plan-then-execute** | Plan mode before multi-file generation |
| **Termination condition** | All tests pass, types check — done |
| **Circuit breaker** | Stop refining after 3 failed attempts |

### Why This Matters for the Exam

The exam tests whether you can **connect concepts across domains**:

```
Day 27-28 (Domain 3):  Claude Code's plan mode and iterative refinement
Day 31+ (Domain 1):    Formal agentic loop design with Agent SDK

The CONCEPTS are the same — the IMPLEMENTATION moves from
Claude Code configuration to Agent SDK architecture.
```

### From Configuration to Architecture

```
Sprint 2: CLAUDE.md says "always write tests first" (configuration)
Sprint 3: Agent SDK hook says "run tests after code generation" (architecture)

Sprint 2: Plan mode previews multi-file changes (feature)
Sprint 3: Coordinator-subagent proposes plans before delegating (pattern)

Sprint 2: Iterative refinement through conversation (workflow)
Sprint 3: Agentic loop with max_turns and circuit breakers (system design)
```

---

## 5. .claude/skills/ for Generation Patterns

### What Are Skills?

Skills in `.claude/skills/` package **reusable generation patterns** — they're more structured than commands and can include context, examples, and multi-step workflows.

### Skill Structure

```markdown
<!-- .claude/skills/create-api-endpoint.md -->
# Skill: Create API Endpoint

## Description
Generate a complete REST API endpoint following our conventions.

## Required Context
- Read src/routes/ for existing route patterns
- Read src/middleware/ for available middleware
- Check src/types/ for existing type definitions

## Steps
1. Define request/response types in src/types/
2. Create route handler in src/routes/{resource}/
3. Add input validation middleware (Zod schema)
4. Add authentication middleware if endpoint requires auth
5. Generate unit tests
6. Generate integration tests
7. Update src/routes/index.ts with new route

## Conventions
- Route files: src/routes/{resource}/{resource}.routes.ts
- Types: src/types/{resource}.types.ts
- Tests: src/routes/{resource}/{resource}.test.ts
- Use express.Router() for route grouping
- All responses use standard envelope: { success, data?, error? }

## Example Output Structure
```
src/
  routes/
    users/
      users.routes.ts    (handlers)
      users.test.ts      (tests)
  types/
    users.types.ts       (interfaces)
```
```

### Skills vs Commands vs Rules

| Mechanism | Purpose | When Invoked |
|-----------|---------|--------------|
| **Commands** | Quick, user-invoked workflows | User types /command-name |
| **Skills** | Complex, reusable patterns with context | Referenced during generation |
| **Rules** | Constraints applied by file path | Automatically when editing matching files |

### Skills for Common Generation Patterns

```markdown
<!-- .claude/skills/create-react-feature.md -->
# Skill: Create React Feature

## Context Gathering
- Read src/components/ for component patterns
- Read src/hooks/ for custom hook patterns
- Read src/styles/ for styling conventions
- Read package.json for available dependencies

## Generation Steps
1. Feature directory: src/features/{name}/
2. Main component: {Name}.tsx
3. Sub-components: components/*.tsx
4. Custom hooks: hooks/use{Name}.ts
5. Types: types.ts
6. Tests: __tests__/{Name}.test.tsx
7. Index barrel export: index.ts

## Quality Checks
- Run tsc --noEmit
- Run vitest run src/features/{name}/
- Verify no eslint errors
```

---

## 6. Advanced Generation Strategies

### Strategy 1: Scaffold Then Flesh Out

```
Phase 1: Generate file structure (stubs/interfaces only)
Phase 2: Review and approve structure
Phase 3: Implement each file (with types already defined)
Phase 4: Connect components (imports, integration)
Phase 5: Generate tests
Phase 6: Validate everything together
```

### Strategy 2: Example-Driven Generation

```markdown
# In CLAUDE.md or skills:
When generating new code, ALWAYS:
1. Find the most similar existing implementation in the codebase
2. Follow its patterns for structure, naming, and style
3. Only deviate from existing patterns when explicitly requested
```

### Strategy 3: Constraint-Based Generation

Rather than telling Claude what to generate, define what generated code must satisfy:

```markdown
# CLAUDE.md constraints for generated code:
- Must compile without errors (tsc --noEmit)
- Must pass all existing tests (vitest run)
- Must not increase bundle size by more than 5KB
- Must not add new dependencies without explicit approval
- Must pass ESLint without warnings
- Must maintain 80%+ coverage for new code
```

---

## 7. Anti-Patterns

### ❌ Anti-Pattern 1: Skipping Plan Mode for Feature Generation

```
"Generate a complete authentication system with OAuth, JWT, sessions, 
 password reset, and 2FA"
# Claude immediately starts writing files without a plan
# Files conflict with each other, wrong architecture
```

**Problem:** Complex generation without a plan leads to inconsistent architecture.

**Fix:** Always use plan mode for features with 3+ files. Review the approach before execution.

### ❌ Anti-Pattern 2: TDD Without Actually Running Tests

```markdown
# CLAUDE.md says:
"Write tests first"

# But no validation step:
# Claude writes tests, then implementation, never runs tests
# Tests may be wrong or implementation may not match
```

**Problem:** TDD without the red-green cycle is just "writing tests and implementation" — no verification of correctness.

**Fix:** Include actual test execution in the workflow: `vitest run` after writing tests (should fail), then after implementation (should pass).

### ❌ Anti-Pattern 3: Batch Generation Without Error Handling

```bash
# CI script generates code but doesn't check for failures
claude -p "Generate types" --output-format json
claude -p "Generate tests" --output-format json
# Never checks if generation failed or produced invalid code
```

**Problem:** Silent failures in CI. Invalid code gets committed.

**Fix:** Check exit codes and validate output:
```bash
RESULT=$(claude -p "Generate types" --output-format json)
if echo "$RESULT" | jq -e '.is_error' | grep -q true; then
  echo "Generation failed!" && exit 1
fi
npx tsc --noEmit || exit 1
```

### ❌ Anti-Pattern 4: Skills That Are Just Templates

```markdown
<!-- .claude/skills/create-component.md -->
Create this exact file:
```tsx
import React from 'react';
export const NAME = () => <div>NAME</div>;
export default NAME;
```
```

**Problem:** No intelligence — just string substitution. Doesn't leverage Claude's ability to analyze context and adapt.

**Fix:** Skills should define patterns and constraints, letting Claude apply intelligence to the specific context.

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│      ADVANCED CODE GENERATION PATTERNS QUICK REFERENCE            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  MULTI-FILE GENERATION:                                            │
│    Plan mode → preview all files → approve → execute → validate  │
│    Key: coherence across files (types, imports, exports align)    │
│                                                                    │
│  CI GENERATION (--print mode):                                     │
│    claude -p "..." --output-format json                           │
│    Always: check is_error, validate output, run tests            │
│    Never: interactive mode, plan mode approval, unchecked output  │
│                                                                    │
│  TDD PATTERN:                                                      │
│    1. Write tests (define expected behavior)                     │
│    2. Run tests → RED (fail — no implementation)                 │
│    3. Write implementation                                       │
│    4. Run tests → GREEN (pass)                                   │
│    5. Refactor if needed                                         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  AGENTIC CONNECTION (Preview Sprint 3):                            │
│                                                                    │
│    Plan-then-execute  ≈  Agentic reasoning loop                  │
│    Plan mode          ≈  Agent proposes actions                   │
│    Iterative refine   ≈  Agentic loop iterations                 │
│    Validation step    ≈  Observation phase in loop               │
│    Max iterations     ≈  max_turns guard                         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  .claude/skills/:                                                  │
│    Purpose: Reusable generation patterns with context            │
│    Content: Steps, conventions, quality checks, examples         │
│    vs Commands: More structured, include context gathering       │
│    vs Rules: Proactive patterns, not reactive constraints        │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Plan mode is about REVIEW before execution (not just planning)│
│  • -p in CI cannot do interactive plan approval                  │
│  • TDD requires actually RUNNING tests (red → green cycle)       │
│  • Skills package complex patterns; commands are quick workflows │
│  • Claude Code generation patterns mirror agentic loops          │
│  • Always validate: tsc + tests + lint after generation          │
│  • Check is_error and exit codes in CI scripts                   │
│  • Skills should define PATTERNS, not templates                  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario (From SPEC)

Your CI pipeline needs to generate API documentation from source code using Claude Code. The pipeline runs in a headless environment with no interactive terminal.

### Question

**Which approach correctly configures this?**

**A)** Run `claude` with the `--resume` flag to continue a previous documentation session

**B)** Run `claude -p "Generate API docs from src/" --output-format json` to get structured output non-interactively

**C)** Create a `.claude/commands/generate-docs.md` and run it interactively in the CI container

**D)** Set `plan_mode: true` in CLAUDE.md so Claude always plans before generating docs

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** correctly addresses all requirements:
  - `-p` / `--print` runs Claude Code **non-interactively** (essential for headless CI)
  - `--output-format json` provides **structured output** that the pipeline can parse
  - The prompt clearly specifies the generation task
  - This is the standard pattern for CI integration with Claude Code

- **Option A** uses `--resume` which continues a previous interactive session. This doesn't make sense for fresh CI documentation generation, and resume still expects interactive capabilities.

- **Option C** creates a slash command but then tries to run it **interactively** in CI — this will fail because CI environments have no interactive terminal.

- **Option D** configures plan mode, but plan mode requires interactive approval ("Shall I proceed?") which isn't possible in a headless environment. Also, it doesn't address the non-interactive output requirement.

**Key Exam Principle:** CI pipelines require `-p` for non-interactive execution. `--output-format json` enables programmatic consumption of results. Plan mode and interactive features are incompatible with headless environments.

---

## Summary: Key Exam Takeaways

1. Plan mode for multi-file generation provides **visibility and coherence** before execution
2. In CI (`-p` mode), plan mode approval is **skipped** — no interactive approval possible
3. TDD patterns in CLAUDE.md enforce **write tests first, then implementation** order
4. The red-green cycle requires **actually running tests** — not just writing them
5. Claude Code's plan-then-execute mirrors Sprint 3's **agentic loop** design
6. `.claude/skills/` packages **complex, reusable generation patterns** with context
7. Skills define patterns and constraints — not literal templates
8. Always **validate** CI-generated code: type check, lint, test
9. Check `is_error` and exit codes in automated scripts
10. Connection to Sprint 3: iterative refinement = agentic loop; plan mode = agent reasoning

---

*End of Day 28 Study Material*
