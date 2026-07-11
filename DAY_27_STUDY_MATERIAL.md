# Day 27: Scenario Deep-Dive: Code Generation with Claude Code

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Scenario** | Code Generation with Claude Code |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Scenario Overview: Code Generation](#1-scenario-overview-code-generation)
2. [CLAUDE.md for Code Generation](#2-claudemd-for-code-generation)
3. [Custom Slash Commands for Workflows](#3-custom-slash-commands-for-workflows)
4. [Plan Mode for Multi-File Generation](#4-plan-mode-for-multi-file-generation)
5. [-p/--print for Scripts and CI](#5--p--print-for-scripts-and-ci)
6. [.claude/rules/ for Language Patterns](#6-clauderules-for-language-patterns)
7. [Iterative Refinement for Code Generation](#7-iterative-refinement-for-code-generation)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code Configuration | https://docs.anthropic.com/en/docs/claude-code/configuration |
| Claude Code Commands | https://docs.anthropic.com/en/docs/claude-code/commands |
| Claude Code CLI Reference | https://docs.anthropic.com/en/docs/claude-code/cli |

---

## 1. Scenario Overview: Code Generation

### Exam Scenario Context

The "Code Generation with Claude Code" scenario tests your ability to **configure Claude Code for reliable, consistent code generation**. It's one of 6 exam scenarios (4 randomly selected per exam).

### What the Exam Tests

| Aspect | What You Must Know |
|--------|-------------------|
| **CLAUDE.md config** | How to define coding standards for generated code |
| **Slash commands** | How to create reusable generation workflows |
| **Plan mode** | When to use plan-then-execute for multi-file changes |
| **CLI flags** | How to run non-interactively (-p/--print) |
| **Rules** | How to enforce language patterns per directory |
| **Iterative refinement** | How to improve generated code through feedback loops |

### The Code Generation Workflow

```
Configure (CLAUDE.md + rules) → Invoke (commands + plan mode) → 
Generate (Claude writes code) → Review (plan mode / iterative) → 
Accept or Refine → Ship
```

---

## 2. CLAUDE.md for Code Generation

### Configuring Standards for Generated Code

CLAUDE.md files define **persistent instructions** that apply to all Claude Code interactions within their scope. For code generation, they establish the rules Claude follows.

### Project-Level CLAUDE.md for Code Gen

```markdown
# CLAUDE.md (project root)

## Code Generation Standards

### Language & Framework
- Use TypeScript with strict mode enabled
- Use React 18+ with functional components and hooks only
- Use Tailwind CSS for styling (no CSS modules, no styled-components)

### Code Style
- Use arrow functions for components: `const MyComponent = () => {}`
- Use explicit return types on all exported functions
- Prefer `interface` over `type` for object shapes
- Use absolute imports from `@/` (mapped to `src/`)

### Testing Requirements
- Every new component MUST have a corresponding test file
- Use Vitest for unit tests, Playwright for E2E
- Test file naming: `ComponentName.test.tsx`
- Minimum test coverage: happy path + one error case

### File Organization
- Components in `src/components/{Feature}/`
- Hooks in `src/hooks/`
- Types in `src/types/`
- Utils in `src/utils/`

### Generation Constraints
- Never modify files in `src/generated/` (auto-generated, do not touch)
- Always run `tsc --noEmit` after generating TypeScript files
- Follow existing patterns in the codebase — check similar files first
```

### Subdirectory CLAUDE.md for Specific Areas

```markdown
# src/api/CLAUDE.md

## API Layer Rules
- All API functions must return `Promise<Result<T, ApiError>>`
- Use the Result type from `src/types/result.ts`
- Include JSDoc with @throws documentation
- Rate limiting: all external API calls must use the `rateLimiter` from `src/utils/rate-limiter.ts`
- Never expose raw error messages — always wrap in ApiError
```

### Hierarchy in Action

```
~/.claude/CLAUDE.md           → "Use my preferred editor settings" (personal)
project/CLAUDE.md             → "Use TypeScript strict, Vitest, Tailwind" (team)
project/src/api/CLAUDE.md     → "All API functions return Result<T>" (specific)
project/src/tests/CLAUDE.md   → "Follow AAA pattern, use factories" (specific)
```

---

## 3. Custom Slash Commands for Workflows

### Creating Code Generation Commands

Slash commands in `.claude/commands/` define reusable workflows that Claude Code executes on demand.

### /generate Command

```markdown
<!-- .claude/commands/generate.md -->
# Generate Component

Create a new React component with the following structure:

1. Read the component specification from the user's input
2. Check `src/components/` for similar existing components to match patterns
3. Generate the component file at the appropriate location
4. Generate a corresponding test file
5. Generate a Storybook story file (if `src/stories/` directory exists)
6. Run `tsc --noEmit` to verify type safety
7. Run `vitest run --reporter=verbose` on the new test file

## Template Structure
```tsx
// Component: src/components/{Feature}/{Name}.tsx
// Test: src/components/{Feature}/{Name}.test.tsx
// Story: src/stories/{Feature}/{Name}.stories.tsx
```

Always follow the patterns defined in CLAUDE.md for imports, styling, and exports.
```

### /refactor Command

```markdown
<!-- .claude/commands/refactor.md -->
# Refactor Code

Refactor the specified code following these steps:

1. Analyze the current implementation for issues (complexity, duplication, naming)
2. Create a plan showing proposed changes BEFORE executing
3. Wait for user approval of the plan
4. Execute the refactoring
5. Ensure all existing tests still pass: `vitest run`
6. If tests break, fix the refactoring (not the tests) unless tests were wrong

## Constraints
- Do not change public API signatures unless explicitly requested
- Preserve all existing behavior (refactor, don't redesign)
- Keep changes minimal and focused
```

### /add-tests Command

```markdown
<!-- .claude/commands/add-tests.md -->
# Add Tests

Add comprehensive tests for the specified file or function:

1. Read the source file to understand behavior
2. Identify all paths: happy path, error cases, edge cases
3. Generate tests following the AAA pattern (Arrange, Act, Assert)
4. Use test factories from `src/tests/factories/` for test data
5. Run the new tests: `vitest run {test_file}`
6. Ensure all tests pass before finishing

## Test Quality Standards
- Each test should test ONE behavior
- Use descriptive test names: "should return error when input is empty"
- Mock external dependencies, don't mock the module under test
- Include at least: 1 happy path, 1 error case, 1 boundary case
```

### Invoking Commands

```
Claude Code > /generate UserProfile component with avatar, name, and bio fields
Claude Code > /refactor src/utils/validation.ts
Claude Code > /add-tests src/hooks/useAuth.ts
```

---

## 4. Plan Mode for Multi-File Generation

### What Is Plan Mode?

Plan mode causes Claude to **propose a plan** before executing changes. Claude shows what it intends to do, and the user reviews before proceeding.

### When Plan Mode Is Essential for Code Gen

| Scenario | Why Plan Mode? |
|----------|---------------|
| Multi-file generation | See all files that will be created/modified |
| Refactoring across modules | Understand the blast radius before changes |
| Architecture changes | Review the approach before committing |
| Unfamiliar codebase | Verify Claude's understanding before acting |

### Plan Mode Output Example

```
Plan: Generate UserProfile Feature

Files to create:
  ✨ src/components/UserProfile/UserProfile.tsx
  ✨ src/components/UserProfile/UserProfile.test.tsx
  ✨ src/components/UserProfile/index.ts (barrel export)
  ✨ src/types/user-profile.ts

Files to modify:
  📝 src/components/index.ts (add export)
  📝 src/types/index.ts (add export)

Approach:
  1. Define UserProfile interface in types
  2. Create component with avatar, name, bio props
  3. Add Tailwind styling matching existing components
  4. Write tests for rendering and prop handling
  5. Update barrel exports

Shall I proceed with this plan?
```

### Plan Mode in CLAUDE.md

You can configure when plan mode activates:

```markdown
# CLAUDE.md

## Plan Mode Rules
- Always use plan mode when creating more than 2 files
- Always use plan mode when modifying files outside the current directory
- Show the plan and wait for approval before executing multi-file changes
```

---

## 5. -p/--print for Scripts and CI

### Non-Interactive Code Generation

The `-p` / `--print` flag runs Claude Code **non-interactively** — essential for scripts, CI pipelines, and automation.

### Basic Usage

```bash
# Generate a component non-interactively
claude -p "Generate a UserProfile component following our CLAUDE.md conventions"

# Generate with structured output for parsing
claude -p "Generate API docs for src/api/" --output-format json

# Generate with schema enforcement
claude -p "Generate the migration" --json-schema '{"type":"object","properties":{"sql":{"type":"string"},"rollback":{"type":"string"}},"required":["sql","rollback"]}'
```

### CI Pipeline Example

```yaml
# .github/workflows/generate-docs.yml
name: Generate API Documentation
on:
  push:
    branches: [main]
    paths: ['src/api/**']

jobs:
  generate-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate API documentation
        run: |
          claude -p "Generate API documentation for all endpoints in src/api/" \
            --output-format json > docs/api-output.json
      - name: Commit generated docs
        run: |
          git add docs/
          git commit -m "docs: auto-generate API documentation" || exit 0
          git push
```

### Key Flags for Code Generation

| Flag | Purpose | Example |
|------|---------|---------|
| `-p` / `--print` | Non-interactive mode | `claude -p "Generate tests"` |
| `--output-format json` | Structured JSON output | Parse results in scripts |
| `--json-schema` | Enforce output schema | Guarantee output structure |
| `--allowedTools` | Restrict tool access | Limit to read-only in CI |

---

## 6. .claude/rules/ for Language Patterns

### Path-Specific Generation Rules

`.claude/rules/` contains rules that apply to specific paths in the codebase, enforcing different patterns for different areas.

### Rule File Examples

```markdown
<!-- .claude/rules/typescript.md -->
# TypeScript Rules (applies to **/*.ts, **/*.tsx)

- Use strict TypeScript (`strict: true` in tsconfig)
- Prefer `unknown` over `any` — cast explicitly when needed
- Use discriminated unions for state: `type State = { status: "loading" } | { status: "error", error: Error } | { status: "success", data: T }`
- Use `satisfies` operator for type narrowing at assignment
- Zod for runtime validation at boundaries (API inputs, env vars)
```

```markdown
<!-- .claude/rules/api-routes.md -->
# API Route Rules (applies to src/api/**)

- Every route handler must validate input with Zod schema
- Return standardized response: `{ success: boolean, data?: T, error?: string }`
- Use middleware pattern for auth/rate-limiting
- Log all requests with correlation ID
- Never return 500 with stack traces in production
```

```markdown
<!-- .claude/rules/tests.md -->
# Test Rules (applies to **/*.test.*, **/*.spec.*)

- Use AAA pattern: Arrange → Act → Assert
- One assertion concept per test (multiple expects OK if testing one behavior)
- Use factories from tests/factories/ — never inline large test objects
- Mock at boundaries: HTTP calls, database, file system
- Test behavior, not implementation — tests should survive refactoring
```

### How Rules Complement CLAUDE.md

| Mechanism | Scope | Use Case |
|-----------|-------|----------|
| CLAUDE.md (root) | Entire project | Global standards, conventions |
| CLAUDE.md (subdir) | Specific directory tree | Area-specific overrides |
| .claude/rules/ | Path-pattern matched | Language/file-type specific rules |

---

## 7. Iterative Refinement for Code Generation

### The Refinement Loop

```
┌────────┐     ┌──────────┐     ┌────────┐     ┌──────────┐
│Generate│────▶│  Review  │────▶│Feedback│────▶│Regenerate│
│        │     │ (human)  │     │        │     │(improved)│
└────────┘     └──────────┘     └────────┘     └──────────┘
                     │                               │
                     └──────── Accept ◀──────────────┘
```

### Effective Feedback Patterns

```
❌ Vague: "Make it better"
❌ Vague: "It's not right"

✅ Specific: "The UserProfile component needs error handling for missing avatar URLs — 
              add a fallback to a default avatar SVG"
✅ Specific: "The test for createUser is missing the case where email is already taken — 
              add a test that expects a DuplicateEmailError"
✅ Specific: "Refactor the switch statement in processOrder to use a strategy pattern — 
              each order type should have its own handler class"
```

### Multi-Round Generation Session

```
Round 1: "Generate a UserService class with CRUD operations"
  → Claude generates initial implementation

Round 2: "Add pagination to the list method — use cursor-based pagination, not offset"
  → Claude modifies the existing code

Round 3: "Add error handling — wrap database calls in try/catch and throw typed errors"
  → Claude adds error handling layer

Round 4: "Now generate tests for all public methods including error cases"
  → Claude generates comprehensive tests
```

### Using --resume for Continued Refinement

```bash
# First generation pass
claude -p "Generate UserService with CRUD operations"
# Session ID: session_abc123

# Continue refining in the same context
claude --resume session_abc123
# "Now add pagination to the list method"
```

---

## 8. Anti-Patterns

### ❌ Anti-Pattern 1: No CLAUDE.md Standards for Generation

```
# Project has NO CLAUDE.md
# Developer asks: "Generate a React component"
# Claude uses inconsistent patterns each time (sometimes class components, 
# sometimes functional, random styling approaches)
```

**Problem:** Without standards, generated code is inconsistent across sessions and developers.

**Fix:** Define clear standards in CLAUDE.md — language, framework, patterns, testing requirements.

### ❌ Anti-Pattern 2: Generating Without Plan Mode for Multi-File Changes

```
"Generate a full REST API with 5 endpoints, middleware, types, and tests"
# Claude starts writing files without showing a plan
# User discovers halfway through that the file structure is wrong
```

**Problem:** Large generation without review can create work that needs to be undone.

**Fix:** Use plan mode for multi-file generation. Review the plan before execution.

### ❌ Anti-Pattern 3: Overly Prescriptive Commands (No Flexibility)

```markdown
<!-- .claude/commands/generate-component.md -->
Generate EXACTLY this code:
```tsx
import React from 'react';
export const ${NAME} = () => <div>${NAME}</div>;
```
```

**Problem:** Treating Claude as a template engine. No intelligence applied — just string substitution.

**Fix:** Give Claude context and constraints, not literal templates. Let it apply intelligence:
```markdown
Generate a component following our CLAUDE.md conventions. 
Analyze existing components for patterns. Include tests.
```

### ❌ Anti-Pattern 4: Not Validating Generated Code

```bash
# Generate and immediately commit without validation
claude -p "Generate migration" >> migrations/new.sql
git add . && git commit -m "add migration"
```

**Problem:** Generated code may have syntax errors, type errors, or test failures.

**Fix:** Include validation in the workflow (tsc --noEmit, test run, lint check).

---

## 9. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│         CODE GENERATION WITH CLAUDE CODE QUICK REFERENCE          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  CONFIGURATION HIERARCHY:                                          │
│    ~/.claude/CLAUDE.md      → Personal preferences               │
│    project/CLAUDE.md        → Team coding standards              │
│    project/src/CLAUDE.md    → Directory-specific rules           │
│    .claude/rules/           → Path-pattern matched rules         │
│    .claude/commands/        → Reusable slash commands            │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  CODE GEN WORKFLOW:                                                │
│    1. CLAUDE.md defines standards and constraints                │
│    2. /command invokes specific generation workflow               │
│    3. Plan mode previews multi-file changes                      │
│    4. Claude generates code following all rules                  │
│    5. Validation (tsc, tests, lint) confirms correctness        │
│    6. Iterative refinement adjusts as needed                     │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  KEY CLI FLAGS:                                                     │
│    -p / --print         Non-interactive (CI/scripts)             │
│    --output-format json Structured output for parsing            │
│    --json-schema        Enforce output shape                     │
│    --resume             Continue previous session                │
│    --allowedTools       Restrict tools in CI                     │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  PLAN MODE:                                                        │
│    When: Multi-file generation, refactoring, architecture        │
│    What: Shows files to create/modify + approach before acting   │
│    Why: Review before execution prevents costly mistakes         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • CLAUDE.md = persistent standards (survive across sessions)    │
│  • Commands = reusable workflows (invoked with /command)         │
│  • Rules = path-specific patterns (matched by file path)         │
│  • Plan mode = preview before execution (multi-file safety)      │
│  • -p is REQUIRED in CI (non-interactive environments)           │
│  • Iterative refinement = generate → review → feedback → redo   │
│  • Always validate generated code (tsc, tests, lint)             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Scenario (From SPEC)

You are configuring Claude Code for a TypeScript monorepo. The team wants Claude to:
1. Always run `tsc --noEmit` after generating code
2. Follow the project's ESLint configuration
3. Never modify files in the `packages/shared/` directory without explicit approval

### Question

**Where should each of these rules be configured?**

**A)** All three rules in the project root CLAUDE.md

**B)** `tsc --noEmit` in a custom slash command, ESLint in CLAUDE.md, `packages/shared/` restriction in `.claude/rules/`

**C)** All three rules in `.claude/rules/` with path-specific matching

**D)** `tsc --noEmit` and ESLint in CLAUDE.md, `packages/shared/` restriction in a subdirectory `packages/shared/CLAUDE.md`

---

### Answer

**✅ Correct Answer: D**

**Explanation:**

- **`tsc --noEmit`** → **CLAUDE.md (root)** ✅ — This is a project-wide convention that applies globally after any TypeScript generation. It belongs in the root CLAUDE.md.

- **ESLint configuration** → **CLAUDE.md (root)** ✅ — Following ESLint is a project-wide standard that applies to all generated code. Root CLAUDE.md is the correct scope.

- **`packages/shared/` restriction** → **Subdirectory CLAUDE.md** ✅ — A directory-specific restriction belongs in a CLAUDE.md scoped to that directory (`packages/shared/CLAUDE.md`). This keeps the rule where it applies and makes the restriction visible to anyone working in that directory.

**Why not A?** Option A puts the path-specific restriction in the root CLAUDE.md where it applies globally but is really about one specific directory. This works functionally but isn't the best scoping practice.

**Why not B?** Slash commands are for user-invoked workflows, not automatic post-generation checks. `tsc --noEmit` should run automatically (defined in CLAUDE.md), not require the user to invoke a command.

**Why not C?** Rules files work for file-type patterns, but project-wide conventions (TypeScript compilation, ESLint) should live in CLAUDE.md where they're clearly visible as global standards.

**Key Exam Principle:** Use the **correct scope** for each rule — project-wide conventions in root CLAUDE.md, directory-specific restrictions in subdirectory CLAUDE.md files, and file-pattern rules in `.claude/rules/`.

---

## Summary: Key Exam Takeaways

1. CLAUDE.md defines **persistent coding standards** for code generation
2. Use **hierarchy** (root → subdirectory) to scope rules appropriately
3. Custom slash commands create **reusable generation workflows** (/generate, /refactor, /add-tests)
4. Plan mode is essential for **multi-file generation** — review before execution
5. `-p` / `--print` is mandatory for **non-interactive** CI/script usage
6. `.claude/rules/` enforces **language-specific patterns** by file path
7. Iterative refinement (generate → review → feedback → regenerate) produces better code
8. Always **validate** generated code (type check, lint, test)
9. Don't treat Claude as a template engine — give constraints, not literal code to copy

---

*End of Day 27 Study Material*
