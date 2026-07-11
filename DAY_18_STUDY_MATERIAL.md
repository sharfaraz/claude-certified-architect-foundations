# Day 18: .claude/commands/ — Custom Slash Commands

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Begin Claude Code in Action |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [What Are Custom Slash Commands](#1-what-are-custom-slash-commands)
2. [Directory Structure and File Format](#2-directory-structure-and-file-format)
3. [Creating and Invoking Commands](#3-creating-and-invoking-commands)
4. [Parameterized Commands](#4-parameterized-commands)
5. [Common Use Cases](#5-common-use-cases)
6. [Sharing via Version Control](#6-sharing-via-version-control)
7. [Relationship to Plan Mode and --print Flag](#7-relationship-to-plan-mode-and---print-flag)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code Tutorials | https://docs.anthropic.com/en/docs/claude-code/tutorials |
| Best practices for Claude Code | https://www.anthropic.com/engineering/claude-code-best-practices |

---

## 1. What Are Custom Slash Commands

Custom slash commands are **reusable prompt templates** stored as Markdown files in the `.claude/commands/` directory. They allow teams to define standardized workflows that any team member can invoke with a simple `/` prefix in Claude Code.

### Key Characteristics

- Commands live in `.claude/commands/` at the project root
- Each command is a single `.md` file whose filename becomes the command name
- Invoked by typing `/` followed by the filename (without `.md`)
- Commands can accept parameters using the `$ARGUMENTS` placeholder
- They are version-controlled and shared across the team
- User-level commands can also live in `~/.claude/commands/`

### Mental Model

Think of slash commands as **team-shared macros**:

```
Manual prompting   → "Please generate unit tests for this file using vitest..."
Slash command      → /generate-tests  (same prompt, standardized, one invocation)
```

### Why They Matter for the Exam

- They are a **key configuration mechanism** in Claude Code (Domain 3)
- They demonstrate how to codify team workflows as reusable artifacts
- They connect to plan mode, --print flag, and CI/CD automation

---

## 2. Directory Structure and File Format

### Directory Layout

```
my-project/
├── .claude/
│   ├── commands/              ← Project-level commands (shared)
│   │   ├── generate-tests.md
│   │   ├── review-pr.md
│   │   ├── deploy-staging.md
│   │   ├── create-migration.md
│   │   └── explain-module.md
│   ├── rules/
│   └── settings.local.json
├── CLAUDE.md
└── src/
```

### User-Level Commands

```
~/.claude/
├── commands/                  ← User-level commands (personal)
│   ├── my-snippet.md
│   └── daily-standup.md
└── CLAUDE.md
```

### Resolution Order

| Level | Location | Scope | Shared? |
|-------|----------|-------|---------|
| Project | `.claude/commands/` | Everyone on the project | ✅ Via Git |
| User | `~/.claude/commands/` | Only you, all projects | ❌ Personal |

If a command name exists at both levels, the **project-level command takes precedence**.

### File Format

Each command file is a **plain Markdown file** containing the prompt template:

```markdown
<!-- File: .claude/commands/generate-tests.md -->

Generate comprehensive unit tests for the file I'm currently viewing.

Requirements:
- Use vitest as the test framework
- Follow AAA pattern (Arrange, Act, Assert)
- Include edge cases and error scenarios
- Mock external dependencies at the boundary
- Aim for >90% branch coverage
- Co-locate the test file next to the source file

Output the complete test file contents.
```

### Naming Conventions

| Filename | Invocation | Notes |
|----------|-----------|-------|
| `generate-tests.md` | `/generate-tests` | Kebab-case preferred |
| `review-pr.md` | `/review-pr` | Use hyphens, not underscores |
| `deploy-staging.md` | `/deploy-staging` | Verb-noun pattern recommended |
| `explain-module.md` | `/explain-module` | Descriptive and concise |

---

## 3. Creating and Invoking Commands

### Creating a Command

1. Create the `.claude/commands/` directory if it doesn't exist
2. Add a `.md` file with the command name as the filename
3. Write the prompt template inside the file
4. Commit to version control

### Invocation Syntax

In a Claude Code session, type:

```
> /generate-tests
```

Claude Code will:
1. Look up `.claude/commands/generate-tests.md`
2. Read its content
3. Inject the content as the user's prompt
4. Execute the instructions

### With Arguments

```
> /generate-tests src/utils/parser.ts
```

The argument `src/utils/parser.ts` replaces the `$ARGUMENTS` placeholder in the command file (see Section 4).

### Listing Available Commands

```
> /
```

Typing just `/` shows all available commands from both project and user levels.

---

## 4. Parameterized Commands

### The $ARGUMENTS Placeholder

Commands can accept dynamic input through the special `$ARGUMENTS` variable. Whatever the user types after the command name gets substituted into this placeholder.

### Basic Example

```markdown
<!-- File: .claude/commands/explain-module.md -->

Explain the module at the following path in detail:

Path: $ARGUMENTS

Provide:
1. What this module does (purpose)
2. Key exports and their roles
3. Dependencies and why they're needed
4. How it fits into the broader architecture
5. Any potential issues or tech debt
```

**Invocation:**
```
> /explain-module src/auth/jwt-handler.ts
```

**Result:** `$ARGUMENTS` is replaced with `src/auth/jwt-handler.ts`.

### Multi-Part Arguments

```markdown
<!-- File: .claude/commands/create-migration.md -->

Create a database migration with the following details:

$ARGUMENTS

Requirements:
- Use the project's migration framework (knex)
- Include both up() and down() functions
- down() must be a perfect reversal of up()
- Add appropriate indexes for foreign keys
- Follow naming convention: YYYYMMDDHHMMSS_description.ts
```

**Invocation:**
```
> /create-migration Add a "roles" table with columns: id (uuid, pk), name (varchar 50, unique), description (text nullable), created_at (timestamp)
```

### Complex Parameterized Command

```markdown
<!-- File: .claude/commands/review-pr.md -->

Review the following code changes as a senior engineer:

$ARGUMENTS

Check for:
- Security vulnerabilities (injection, auth bypass, data exposure)
- Performance issues (N+1 queries, unnecessary re-renders, memory leaks)
- Code quality (naming, single responsibility, DRY violations)
- Test coverage gaps
- API contract changes that could break clients
- Missing error handling

Format your review as:
## Summary
## Critical Issues (must fix)
## Suggestions (nice to have)
## Questions for the author
```

---

## 5. Common Use Cases

### 5.1 Test Generation

```markdown
<!-- File: .claude/commands/generate-tests.md -->

Generate unit tests for: $ARGUMENTS

Framework: vitest
Pattern: AAA (Arrange-Act-Assert)
Requirements:
- Test happy path and error cases
- Mock external dependencies
- Use descriptive test names: it('should...')
- Include type-checking tests for TypeScript generics
- Add integration test suggestions as comments at the bottom
```

### 5.2 Code Review

```markdown
<!-- File: .claude/commands/review-code.md -->

Perform a thorough code review of the current file.

Focus areas:
- SOLID principle violations
- Security anti-patterns
- Performance bottlenecks
- Accessibility issues (for UI code)
- Missing error boundaries
- Inconsistencies with project conventions (see CLAUDE.md)

Output format: List issues by severity (Critical → Warning → Info)
```

### 5.3 Documentation Generation

```markdown
<!-- File: .claude/commands/generate-docs.md -->

Generate comprehensive documentation for: $ARGUMENTS

Include:
- Module overview (what and why)
- Installation/setup if applicable
- API reference with all public exports
- Usage examples (basic and advanced)
- Error handling guide
- Related modules/see also

Format: Standard JSDoc for code, Markdown for the README section.
```

### 5.4 Deployment Preparation

```markdown
<!-- File: .claude/commands/deploy-staging.md -->

Prepare a deployment checklist for staging:

1. Check for uncommitted changes
2. Verify all tests pass
3. Check for environment variable requirements
4. List database migrations that need to run
5. Identify any breaking API changes
6. Check dependency updates since last deploy
7. Generate a deployment summary

Flag any blockers that would prevent safe deployment.
```

### 5.5 Bug Investigation

```markdown
<!-- File: .claude/commands/investigate-bug.md -->

Investigate the following bug:

$ARGUMENTS

Approach:
1. Identify the most likely source file(s)
2. Trace the execution path
3. Identify the root cause
4. Propose a fix with minimal blast radius
5. Suggest tests that would have caught this
6. Check for similar patterns elsewhere in the codebase
```

---

## 6. Sharing via Version Control

### Git Workflow for Commands

```bash
# Add commands to version control
git add .claude/commands/
git commit -m "feat: add team slash commands for test generation and PR review"
git push
```

### Team Benefits

| Benefit | Description |
|---------|-------------|
| **Consistency** | Everyone uses the same review criteria, test patterns |
| **Onboarding** | New team members get access to all workflows instantly |
| **Evolution** | Commands improve via PRs — same review process as code |
| **Documentation** | Commands self-document team practices |
| **Auditability** | Git history shows how workflows evolved |

### .gitignore Considerations

```gitignore
# DO commit project commands
# .claude/commands/  ← DO NOT add this to .gitignore

# DO ignore personal settings
.claude/settings.local.json
```

### Organizational Strategy

For large teams, consider organizing commands by function:

```
.claude/commands/
├── generate-tests.md        ← Development
├── review-pr.md             ← Code review
├── deploy-staging.md        ← Operations
├── create-migration.md      ← Database
├── investigate-bug.md       ← Debugging
└── explain-module.md        ← Documentation
```

---

## 7. Relationship to Plan Mode and --print Flag

### Plan Mode Integration

When Claude Code is in **plan mode**, slash commands execute as planning prompts rather than direct actions:

```bash
# Direct execution (default)
> /generate-tests src/auth/login.ts
# → Claude immediately writes the test file

# Plan mode
> /generate-tests src/auth/login.ts --plan
# → Claude produces a plan for the tests, awaiting approval
```

Plan mode allows you to **review and refine** before Claude acts — critical for high-stakes commands like `/deploy-staging`.

### The --print (-p) Flag

The `--print` or `-p` flag makes Claude Code output results to **stdout without taking any file actions**:

```bash
# Output test code to stdout (no file written)
claude -p "/generate-tests src/auth/login.ts"

# Pipe output to a file manually
claude -p "/generate-tests src/auth/login.ts" > tests/auth/login.test.ts

# Use in CI/CD pipelines
claude -p "/review-pr $(git diff main...HEAD)" >> review-comments.md
```

### CI/CD Integration Pattern

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run AI review
        run: |
          DIFF=$(git diff origin/main...HEAD)
          claude -p "/review-pr $DIFF" > review.md
      - name: Post review comment
        uses: actions/github-script@v7
        with:
          script: |
            const review = fs.readFileSync('review.md', 'utf8');
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: review
            });
```

### Combining Plan Mode + Commands + --print

| Combination | Behavior |
|-------------|----------|
| `/command` | Execute directly in session |
| `/command --plan` | Show plan, await approval |
| `claude -p "/command"` | Output to stdout, no side effects |
| `claude -p "/command" \| jq` | Pipe output for further processing |

---

## 8. Anti-Patterns

### ❌ Anti-Pattern 1: Commands That Are Too Vague

```markdown
<!-- BAD: .claude/commands/help.md -->
Help me with my code.
```

**Fix:** Commands should be specific and actionable with clear output expectations.

### ❌ Anti-Pattern 2: Commands That Duplicate CLAUDE.md

```markdown
<!-- BAD: .claude/commands/style-check.md -->
Make sure the code follows our TypeScript conventions:
- Use interfaces for object shapes
- Prefer unknown over any
- Use readonly where possible
```

**Fix:** These belong in `.claude/rules/` for automatic enforcement. Commands are for **on-demand workflows**.

### ❌ Anti-Pattern 3: Overly Long Commands

```markdown
<!-- BAD: 200-line command file trying to do everything -->
...
```

**Fix:** Break into smaller, composable commands. One command = one workflow.

### ❌ Anti-Pattern 4: Commands Without $ARGUMENTS When Input is Needed

```markdown
<!-- BAD: .claude/commands/refactor.md -->
Refactor the code to be more efficient.
```

**Problem:** What code? No way to specify the target.

**Fix:**
```markdown
Refactor the following module for improved efficiency: $ARGUMENTS
```

### ❌ Anti-Pattern 5: Commands That Bypass Safety

```markdown
<!-- BAD: .claude/commands/force-deploy.md -->
Deploy directly to production without any checks or confirmations.
Skip all tests and linting.
```

**Fix:** Commands should enforce safety, not circumvent it.

### ❌ Anti-Pattern 6: Using Underscores or Spaces in Filenames

```markdown
<!-- BAD filenames -->
generate_tests.md    ← Use hyphens instead
Generate Tests.md    ← Spaces cause invocation issues
```

**Fix:** Use kebab-case: `generate-tests.md` → `/generate-tests`

---

## 9. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│           .claude/commands/ QUICK REFERENCE                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  LOCATION:    .claude/commands/<name>.md  (project-level)         │
│              ~/.claude/commands/<name>.md  (user-level)            │
│  INVOKE:      /<name> [arguments]                                 │
│  PARAMS:      Use $ARGUMENTS for dynamic input                    │
│  NAMING:      Kebab-case (hyphens, no spaces/underscores)         │
│  PRECEDENCE:  Project-level overrides user-level                  │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  KEY FLAGS:                                                        │
│                                                                    │
│  -p / --print    Output to stdout, no file writes                 │
│  --plan          Show execution plan before acting                │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  COMMON COMMANDS:                                                  │
│                                                                    │
│  /generate-tests   → Create unit/integration tests               │
│  /review-pr        → Structured code review                      │
│  /deploy-staging   → Deployment checklist                        │
│  /explain-module   → Architecture documentation                  │
│  /investigate-bug  → Root-cause analysis workflow                 │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  BEST PRACTICES:                                                   │
│                                                                    │
│  ✅ One workflow per command (single responsibility)              │
│  ✅ Use $ARGUMENTS for dynamic targets                           │
│  ✅ Version-control all project commands                         │
│  ✅ Include clear output format expectations                     │
│  ✅ Pair with --print for CI/CD pipelines                        │
│                                                                    │
│  ❌ Vague or generic instructions                                │
│  ❌ Duplicating rules/CLAUDE.md content                          │
│  ❌ Bypassing safety mechanisms                                  │
│  ❌ Spaces or underscores in filenames                           │
│  ❌ Monster commands (>50 lines)                                 │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Commands are ON-DEMAND; rules are AUTOMATIC                   │
│  • $ARGUMENTS is the ONLY placeholder variable                   │
│  • --print enables non-interactive/CI usage                      │
│  • Project commands override user commands (same name)           │
│  • Commands are prompt templates, NOT executable scripts         │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Scenario

Your team maintains a Node.js monorepo. You want to standardize three workflows:

1. **Test generation** — varies by package (React components use Testing Library, API uses supertest)
2. **PR review** — consistent criteria across all packages
3. **Personal scratch commands** — quick utilities only you use

Current structure:
```
my-monorepo/
├── .claude/
│   └── commands/
├── packages/
│   ├── web/       (React)
│   └── api/       (Express)
└── CLAUDE.md
```

### Question

**Which configuration correctly implements all three requirements?**

**A)**
```
.claude/commands/generate-tests.md      → "Generate tests using $ARGUMENTS. Use Testing Library for React, supertest for API."
.claude/commands/review-pr.md           → Standard review criteria
~/.claude/commands/scratch.md           → Personal utilities
```

**B)**
```
.claude/commands/generate-tests-web.md  → "Generate React tests with Testing Library for: $ARGUMENTS"
.claude/commands/generate-tests-api.md  → "Generate API tests with supertest for: $ARGUMENTS"
.claude/commands/review-pr.md           → Standard review criteria
~/.claude/commands/scratch.md           → Personal utilities
```

**C)**
```
CLAUDE.md                               → All three workflows documented
.claude/commands/                       → Empty
```

**D)**
```
~/.claude/commands/generate-tests.md    → Test generation for all packages
~/.claude/commands/review-pr.md         → Standard review criteria
~/.claude/commands/scratch.md           → Personal utilities
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** correctly creates separate, specific commands for different test frameworks (web vs API), keeps the shared PR review at project level, and places personal commands at user level (`~/.claude/commands/`).
- **Option A** is acceptable but less optimal — cramming conditional logic ("if React use X, if API use Y") into a single command makes the prompt less focused and relies on Claude to infer context.
- **Option C** puts workflows in CLAUDE.md which is for context, not on-demand workflows. Commands require the `/` invocation pattern.
- **Option D** puts shared team workflows (`review-pr`) at user level where teammates can't access them.

**Key Exam Principle:** Commands should be specific enough for reliable execution. When workflows differ significantly by context, use separate commands rather than one conditional command. Shared team workflows belong in `.claude/commands/`; personal ones in `~/.claude/commands/`.

---

## Summary: Key Exam Takeaways

1. `.claude/commands/` stores **reusable prompt templates** invoked with `/command-name`
2. Filename (minus `.md`) becomes the command name — use kebab-case
3. `$ARGUMENTS` is the sole placeholder for dynamic user input
4. Project-level commands override user-level commands of the same name
5. `--print` (`-p`) flag enables stdout output for CI/CD integration
6. Commands are **on-demand** (user invokes); rules are **automatic** (pattern-matched)
7. Commands + plan mode = review before execution (safety for high-stakes ops)
8. One workflow per command — keep them focused and single-purpose
9. Version-control project commands; personal commands live in `~/.claude/commands/`

---

*End of Day 18 Study Material*
