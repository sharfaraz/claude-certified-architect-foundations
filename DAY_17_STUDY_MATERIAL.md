# Day 17: .claude/rules/ — Path-Specific Rules

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Continue Claude Code 101 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [What is .claude/rules/ Directory](#1-what-is-clauderules-directory)
2. [Rule File Naming and Structure](#2-rule-file-naming-and-structure)
3. [Path-Specific Matching Behavior](#3-path-specific-matching-behavior)
4. [Use Cases](#4-use-cases)
5. [Relationship Between .claude/rules/ and CLAUDE.md](#5-relationship-between-clauderules-and-claudemd)
6. [When to Use Rules vs CLAUDE.md](#6-when-to-use-rules-vs-claudemd)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Best practices for Claude Code | https://www.anthropic.com/engineering/claude-code-best-practices |
| How Claude remembers your project | https://docs.claude.com/en/docs/claude-code/memory |

---

## 1. What is .claude/rules/ Directory

The `.claude/rules/` directory is a **path-specific rule system** that allows teams to define targeted instructions that Claude Code applies **only when working with files matching specific glob patterns**. Unlike the general-purpose `CLAUDE.md` file, rules in this directory activate conditionally based on the file paths being edited or referenced in a conversation.

### Key Characteristics

- Rules live in `.claude/rules/` at the project root
- Each rule file targets specific file paths using glob patterns encoded in the filename
- Rules are **automatically injected** into Claude's context when relevant files are touched
- They enable **fine-grained, automated enforcement** of coding standards
- Rules are version-controlled and shared across the team

### Mental Model

Think of `.claude/rules/` as a **path-triggered instruction system**:

```
CLAUDE.md        → "Always do X" (global, always active)
.claude/rules/   → "When touching files in src/api/**, do Y" (conditional, auto-triggered)
```

### Directory Structure Example

```
my-project/
├── .claude/
│   ├── rules/
│   │   ├── src__api__**.md
│   │   ├── src__components__**.md
│   │   ├── tests__**.md
│   │   ├── *.py.md
│   │   └── infra__**.md
│   └── settings.local.json
├── CLAUDE.md
├── src/
│   ├── api/
│   └── components/
├── tests/
└── infra/
```

---

## 2. Rule File Naming and Structure

### Filename Conventions

The filename of each rule file **encodes the glob pattern** that determines when the rule activates. The mapping uses specific character substitutions:

| Character in Path | Character in Filename | Example |
|---|---|---|
| `/` (directory separator) | `__` (double underscore) | `src/api/` → `src__api__` |
| `*` (wildcard) | `*` (preserved) | `*.py` → `*.py` |
| `**` (recursive glob) | `**` (preserved) | `src/**` → `src__**` |

### Filename Pattern Examples

| Target Path Pattern | Rule Filename |
|---|---|
| `src/api/**` | `src__api__**.md` |
| `src/components/**/*.tsx` | `src__components__**__*.tsx.md` |
| `*.py` | `*.py.md` |
| `tests/unit/**` | `tests__unit__**.md` |
| `infra/terraform/**` | `infra__terraform__**.md` |
| `docs/**/*.md` | `docs__**__*.md.md` |

### Rule File Content Structure

Each `.md` file contains plain-text instructions in Markdown format. There is no required schema — you write natural language instructions:

```markdown
<!-- File: .claude/rules/src__api__**.md -->

## API Layer Rules

- All API handlers must validate input using Zod schemas before processing
- Return standardized error responses using the ApiError class from `src/utils/errors.ts`
- Every endpoint must have corresponding OpenAPI documentation in the docstring
- Use the repository pattern — never access the database directly from handlers
- Rate limiting middleware must be applied to all public endpoints
- Log all incoming requests using the structured logger (not console.log)
```

### Another Example — Python Rules

```markdown
<!-- File: .claude/rules/*.py.md -->

## Python Conventions

- Use type hints for all function parameters and return values
- Follow Google-style docstrings for all public functions and classes
- Use `pathlib.Path` instead of `os.path` for file operations
- Prefer dataclasses or Pydantic models over raw dictionaries
- All exceptions must be caught specifically — never use bare `except:`
- Use `logging` module, never `print()` for any output in production code
```

---

## 3. Path-Specific Matching Behavior

### How Matching Works

When Claude Code is working in a session, it tracks which files are being:
- **Edited** (created, modified, or deleted)
- **Read** (referenced or analyzed)
- **Discussed** (mentioned by the user)

If any active file matches a rule's glob pattern, that rule's content is **automatically injected** into Claude's system context for the duration of that interaction.

### Glob Pattern Semantics

| Pattern | Matches | Does NOT Match |
|---|---|---|
| `src__api__**.md` | `src/api/routes.ts`, `src/api/v2/users.ts` | `src/utils/api.ts` |
| `*.py.md` | `app.py`, `utils.py`, `src/main.py` | `app.pyc`, `pytest.ini` |
| `src__components__*.tsx.md` | `src/components/Button.tsx` | `src/components/forms/Input.tsx` (no recursive) |
| `src__components__**__*.tsx.md` | `src/components/Button.tsx`, `src/components/forms/Input.tsx` | `src/pages/Home.tsx` |

### Important Matching Rules

1. **Multiple rules can activate simultaneously** — if a file matches multiple patterns, all matching rules are injected
2. **Rules are additive** — they don't override each other; all matched rules contribute instructions
3. **Rules don't conflict-resolve** — if two rules give contradictory instructions, Claude will attempt to reconcile them (which may produce unpredictable results)
4. **Pattern specificity doesn't create priority** — a more specific pattern doesn't override a broader one

### Example: Multiple Rules Activating

When editing `src/api/auth/login.ts`:

```
✅ src__api__**.md          (matches — file is under src/api/)
✅ src__**.md               (matches — file is under src/)
✅ *.ts.md                  (matches — file is a .ts file)
❌ tests__**.md             (no match)
❌ src__components__**.md   (no match)
```

All three matching rules will be injected into context simultaneously.

---

## 4. Use Cases

### 4.1 Language-Specific Conventions

```markdown
<!-- File: .claude/rules/*.ts.md -->

## TypeScript Conventions

- Use `interface` for object shapes that may be extended; use `type` for unions/intersections
- Prefer `unknown` over `any` — narrow with type guards
- Use `readonly` arrays and properties where mutation is not needed
- Enum alternatives: use `as const` objects for string enums
- All async functions must handle errors with try/catch or .catch()
- Use barrel exports (index.ts) only at module boundaries
```

```markdown
<!-- File: .claude/rules/*.rs.md -->

## Rust Conventions

- Use `thiserror` for library errors, `anyhow` for application errors
- Prefer `impl Trait` in argument position over generics when only one impl is expected
- All public items must have doc comments with examples
- Use `clippy::pedantic` lint level — address all warnings
- Prefer `&str` over `String` in function parameters when ownership isn't needed
```

### 4.2 Restricting Operations in Sensitive Directories

```markdown
<!-- File: .claude/rules/infra__**.md -->

## Infrastructure Safety Rules

⚠️ CRITICAL: Files in this directory affect production infrastructure.

- NEVER modify terraform state files directly
- Always use `terraform plan` output to verify changes before suggesting `apply`
- Do not remove or rename existing resources — use `moved` blocks for refactoring
- All security group changes must include a comment explaining the business need
- Database changes must be backwards-compatible (no column drops without migration plan)
- Tag all resources with: environment, team, cost-center
```

```markdown
<!-- File: .claude/rules/src__auth__**.md -->

## Authentication Module Rules

🔒 SECURITY-SENSITIVE CODE

- Never log tokens, passwords, or session IDs — even in debug mode
- All crypto operations must use the `crypto` module from `src/utils/crypto.ts`
- Password hashing must use bcrypt with minimum 12 rounds
- JWT tokens must include `exp`, `iat`, and `sub` claims
- Never store secrets in code — always reference environment variables
- Changes to this directory require security review — add TODO comment for reviewer
```

### 4.3 Team Guidelines for Specific Areas

```markdown
<!-- File: .claude/rules/src__components__**.md -->

## Frontend Component Guidelines

- All components must be functional (no class components)
- Use the project's design system tokens — never hardcode colors or spacing
- Every component must export a Props interface
- Include `data-testid` attributes on interactive elements
- Accessibility: all interactive elements need aria-labels; images need alt text
- Co-locate component tests: `Component.test.tsx` next to `Component.tsx`
- Use CSS Modules for styling — no inline styles except for dynamic values
```

### 4.4 Test Directory Rules

```markdown
<!-- File: .claude/rules/tests__**.md -->

## Testing Standards

- Use descriptive test names: `it('should return 404 when user not found')`
- Follow Arrange-Act-Assert pattern
- No test should depend on another test's state
- Mock external services at the boundary — never mock internal modules
- Use factory functions for test data, not raw object literals
- Integration tests must clean up their database state in afterEach
- Snapshot tests are banned — use explicit assertions
```

---

## 5. Relationship Between .claude/rules/ and CLAUDE.md

### Hierarchy Overview

```
┌─────────────────────────────────────────────┐
│          ~/.claude/CLAUDE.md                 │  ← User-level (personal preferences)
├─────────────────────────────────────────────┤
│          ./CLAUDE.md (project root)          │  ← Project-level (always active)
├─────────────────────────────────────────────┤
│     ./src/CLAUDE.md (subdirectory)           │  ← Subdirectory-level (active in scope)
├─────────────────────────────────────────────┤
│     .claude/rules/src__api__**.md            │  ← Path-specific (conditional activation)
└─────────────────────────────────────────────┘
```

### Comparison Table

| Feature | CLAUDE.md | .claude/rules/ |
|---------|-----------|----------------|
| **Activation** | Always loaded (or scoped to directory) | Conditionally loaded based on file pattern |
| **Scope** | Broad project context | Narrow path-specific instructions |
| **Content type** | Project overview, architecture, conventions | Targeted enforcement rules |
| **When read** | Every session start | Only when matching files are active |
| **Location** | Project root or subdirectories | Always in `.claude/rules/` directory |
| **Naming** | Must be `CLAUDE.md` | Filename encodes the glob pattern |
| **Best for** | "What is this project?" | "How should I handle these specific files?" |
| **Granularity** | Per-directory (coarse) | Per-glob-pattern (fine) |
| **Overhead** | Always in context (uses tokens) | Only injected when relevant (efficient) |
| **Team sharing** | Via version control | Via version control |

### How They Work Together

```
Session starts:
  1. Load ~/.claude/CLAUDE.md (user preferences)
  2. Load ./CLAUDE.md (project context — always)
  3. Load subdirectory CLAUDE.md files (if working in those dirs)
  4. Check active files against .claude/rules/ patterns
  5. Inject all matching rule files into context
```

All sources are **additive** — nothing overrides. They combine to form the full instruction set for the session.

---

## 6. When to Use Rules vs CLAUDE.md

### Decision Framework

```
Is the instruction relevant to ALL files in the project?
  → YES: Put it in CLAUDE.md
  → NO: Continue...

Is the instruction specific to a particular set of file paths?
  → YES: Put it in .claude/rules/ with the appropriate glob pattern
  → NO: Continue...

Is the instruction specific to a subdirectory's "identity"?
  → YES: Put a CLAUDE.md in that subdirectory
  → NO: Consider if it's even needed
```

### Use CLAUDE.md When...

- Describing project architecture and tech stack
- Listing global coding conventions (naming, formatting)
- Documenting how to build, test, and deploy
- Explaining key abstractions and design patterns
- Providing context about the team and development workflow
- Specifying the preferred language/framework versions

### Use .claude/rules/ When...

- Enforcing language-specific linting beyond what tools catch
- Applying security constraints to sensitive directories
- Defining module-specific architectural boundaries
- Setting testing standards that vary by test type
- Restricting operations in infrastructure code
- Applying different documentation standards to public vs internal APIs

### Practical Example: Choosing the Right Location

| Instruction | Where to Put It | Why |
|---|---|---|
| "We use TypeScript 5.3 with strict mode" | `CLAUDE.md` | Applies to entire project |
| "API handlers must validate with Zod" | `.claude/rules/src__api__**.md` | Only relevant to API files |
| "Use pytest fixtures, not setUp/tearDown" | `.claude/rules/tests__**.md` | Only relevant to test files |
| "We follow trunk-based development" | `CLAUDE.md` | General workflow context |
| "Never modify migration files after merge" | `.claude/rules/migrations__**.md` | Only relevant to migrations |
| "The frontend uses React 18 with RSC" | `src/frontend/CLAUDE.md` | Broad context for a subproject |

---

## 7. Anti-Patterns

### ❌ Anti-Pattern 1: Putting Everything in Rules

```
# BAD: Overly fragmented rules for general information
.claude/rules/src__**.md         → "We use TypeScript"
.claude/rules/tests__**.md       → "We use TypeScript"
.claude/rules/scripts__**.md     → "We use TypeScript"
```

**Fix:** Put universal facts in `CLAUDE.md`.

### ❌ Anti-Pattern 2: Contradictory Rules

```markdown
<!-- .claude/rules/src__**.md -->
Always use console.log for debugging

<!-- .claude/rules/src__api__**.md -->
Never use console.log — use structured logger only
```

**Problem:** A file in `src/api/` matches both rules, creating a contradiction.

**Fix:** Ensure broader rules don't conflict with narrower ones, or phrase them to coexist:

```markdown
<!-- .claude/rules/src__**.md -->
Use structured logger for all production logging. 
(Exception: console.log is acceptable in development scripts only.)
```

### ❌ Anti-Pattern 3: Overly Broad Glob Patterns

```
.claude/rules/**.md    → Matches literally everything (same as CLAUDE.md)
```

**Fix:** If it matches everything, put it in `CLAUDE.md` instead.

### ❌ Anti-Pattern 4: Too Many Overlapping Rules

```
.claude/rules/src__**.md
.claude/rules/src__api__**.md
.claude/rules/src__api__v2__**.md
.claude/rules/src__api__v2__users__**.md
```

**Problem:** Working on `src/api/v2/users/handler.ts` loads ALL FOUR rules, consuming context tokens and potentially creating confusion.

**Fix:** Keep nesting to maximum 2 levels. Use the most specific pattern that covers your need.

### ❌ Anti-Pattern 5: Duplicating Linter/Formatter Rules

```markdown
<!-- .claude/rules/*.ts.md -->
- Use 2 spaces for indentation
- Add semicolons at end of statements
- Use single quotes for strings
```

**Fix:** Let ESLint/Prettier handle formatting. Use rules for **semantic** guidance that tooling can't enforce.

### ❌ Anti-Pattern 6: Rules with No Actionable Instructions

```markdown
<!-- .claude/rules/src__auth__**.md -->
This is the authentication module. It handles user login and registration.
It was written in 2022 by the platform team.
```

**Fix:** Rules should contain **instructions**, not descriptions. Descriptions belong in `CLAUDE.md` or code comments.

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│              .claude/rules/ QUICK REFERENCE                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  LOCATION:    .claude/rules/<pattern>.md                          │
│  ENCODING:    / → __    (slashes become double underscores)       │
│  WILDCARDS:   *  = single level    ** = recursive                 │
│  FORMAT:      Plain Markdown with instructions                    │
│  ACTIVATION:  Automatic when matching file is in context          │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  COMMON PATTERNS:                                                  │
│                                                                    │
│  *.py.md              → All Python files anywhere                 │
│  src__api__**.md      → Everything under src/api/                 │
│  tests__unit__**.md   → Everything under tests/unit/              │
│  *.test.ts.md         → All TypeScript test files                 │
│  infra__**.md         → All infrastructure code                   │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  BEST PRACTICES:                                                   │
│                                                                    │
│  ✅ Actionable instructions (not descriptions)                    │
│  ✅ Path-specific guidance that varies by file type               │
│  ✅ Security constraints on sensitive directories                 │
│  ✅ Maximum 2 levels of pattern nesting                           │
│  ✅ Complement (don't duplicate) CLAUDE.md                        │
│                                                                    │
│  ❌ Universal rules (put in CLAUDE.md)                            │
│  ❌ Formatting rules (use linters/formatters)                     │
│  ❌ Contradictory overlapping rules                               │
│  ❌ Descriptive content without instructions                      │
│  ❌ Catch-all patterns like **.md                                 │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Rules are ADDITIVE — multiple can activate at once             │
│  • No priority system — specificity doesn't override              │
│  • Rules are for ENFORCEMENT, CLAUDE.md is for CONTEXT            │
│  • Filename IS the glob pattern (with __ for /)                   │
│  • Rules only load when matching files are active (token-saving)  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

You are configuring Claude Code for a monorepo with the following structure:

```
my-app/
├── .claude/
│   └── rules/
├── CLAUDE.md
├── packages/
│   ├── api/          (Express.js REST API)
│   ├── web/          (Next.js frontend)
│   └── shared/       (Shared utilities)
├── infra/            (Terraform IaC)
└── scripts/          (Build/deploy scripts)
```

Requirements:
- The API package must use Zod for request validation
- The web package must use Server Components by default
- The shared package must not import from api or web
- Infrastructure code must never delete resources without a migration plan
- All packages use TypeScript strict mode (universal)

### Question

**Which of the following configurations BEST follows best practices for `.claude/rules/` and `CLAUDE.md`?**

**A)**
```
CLAUDE.md: Contains all five requirements listed above
.claude/rules/: Empty
```

**B)**
```
CLAUDE.md: "All packages use TypeScript strict mode"
.claude/rules/packages__api__**.md: "Use Zod for request validation"
.claude/rules/packages__web__**.md: "Use Server Components by default"
.claude/rules/packages__shared__**.md: "Must not import from api or web"
.claude/rules/infra__**.md: "Never delete resources without migration plan"
```

**C)**
```
CLAUDE.md: Contains all five requirements
.claude/rules/**.md: Contains all five requirements (duplicated for enforcement)
```

**D)**
```
.claude/rules/packages__**.md: All five requirements
CLAUDE.md: Empty
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** correctly separates universal context (`CLAUDE.md` → TypeScript strict mode applies everywhere) from path-specific enforcement (`.claude/rules/` → each rule targets only the relevant package/directory).
- **Option A** is suboptimal because path-specific rules in `CLAUDE.md` waste tokens on every session regardless of what files are being edited.
- **Option C** is an anti-pattern (duplication) and uses `**.md` which matches everything, defeating the purpose of rules.
- **Option D** puts a universal rule (TypeScript strict mode) in a path-scoped file and leaves `CLAUDE.md` empty, losing the project context benefit.

**Key Exam Principle:** Use `CLAUDE.md` for universal project context that's always relevant. Use `.claude/rules/` for conditional, path-specific enforcement that should only activate when those files are being worked on.

---

## Summary: Key Exam Takeaways

1. `.claude/rules/` provides **path-specific, auto-triggered instructions** — the filename IS the glob pattern
2. Slashes in paths become `__` in filenames; wildcards (`*`, `**`) are preserved
3. Rules are **additive and concurrent** — no priority or override system
4. Use rules for **enforcement**, `CLAUDE.md` for **context**
5. Rules save tokens by only loading when relevant files are active
6. Avoid contradictions between overlapping patterns
7. Don't duplicate what linters/formatters already handle
8. Maximum 2 levels of nesting to avoid context bloat
9. Rules are version-controlled and team-shared via the `.claude/` directory

---

*End of Day 17 Study Material*
