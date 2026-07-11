# Day 19: .claude/skills/ Configuration and Plan Mode

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 3: Claude Code Configuration & Workflows (20% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Continue Claude Code in Action |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [What is .claude/skills/ Directory](#1-what-is-claudeskills-directory)
2. [Skill Definitions and Structure](#2-skill-definitions-and-structure)
3. [Packaging Reusable Capabilities](#3-packaging-reusable-capabilities)
4. [Plan Mode vs Direct Execution](#4-plan-mode-vs-direct-execution)
5. [How Plan Mode Interacts with CLAUDE.md and Commands](#5-how-plan-mode-interacts-with-claudemd-and-commands)
6. [Iterative Refinement in Plan Mode](#6-iterative-refinement-in-plan-mode)
7. [Trade-offs: Safety vs Latency](#7-trade-offs-safety-vs-latency)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Best practices for Claude Code | https://www.anthropic.com/engineering/claude-code-best-practices |

---

## 1. What is .claude/skills/ Directory

The `.claude/skills/` directory is a **capability packaging system** that allows teams to define complex, multi-step skills that Claude Code can activate on demand. Skills go beyond simple slash commands by packaging domain knowledge, step-by-step procedures, and contextual awareness into reusable units.

### Key Characteristics

- Skills live in `.claude/skills/` at the project root
- Each skill is a Markdown file that defines a **multi-step capability**
- Skills can be referenced in prompts and slash commands
- They encapsulate domain expertise (e.g., "how to add a new API endpoint in our stack")
- They guide Claude through complex procedures rather than simple one-shot tasks

### Mental Model: Commands vs Skills

```
Commands   → "What to do" (one-shot task prompt)
Skills     → "How to do it" (multi-step procedure with domain knowledge)
Rules      → "Constraints while doing it" (guardrails on specific file paths)
CLAUDE.md  → "What this project is" (global context)
```

### Directory Structure

```
my-project/
├── .claude/
│   ├── skills/
│   │   ├── add-api-endpoint.md
│   │   ├── create-react-component.md
│   │   ├── database-migration.md
│   │   └── deploy-process.md
│   ├── commands/
│   ├── rules/
│   └── settings.local.json
├── CLAUDE.md
└── src/
```

---

## 2. Skill Definitions and Structure

### Anatomy of a Skill File

A skill file contains structured instructions that guide Claude through a multi-step procedure:

```markdown
<!-- File: .claude/skills/add-api-endpoint.md -->

# Skill: Add API Endpoint

## Context
This project uses Express.js with TypeScript, Zod validation, and Prisma ORM.
All endpoints follow the controller-service-repository pattern.

## Steps

1. **Create the route file** in `src/routes/<resource>.routes.ts`
   - Define the route using Express Router
   - Apply authentication middleware for protected routes
   - Apply rate limiting for public routes

2. **Create the controller** in `src/controllers/<resource>.controller.ts`
   - Parse and validate request using Zod schema
   - Delegate to service layer
   - Return standardized response format

3. **Create the service** in `src/services/<resource>.service.ts`
   - Implement business logic
   - Call repository for data access
   - Handle domain-specific errors

4. **Create the repository** in `src/repositories/<resource>.repository.ts`
   - Use Prisma client for database operations
   - Return typed results

5. **Add Zod schemas** in `src/schemas/<resource>.schema.ts`
   - Define request body, params, and query schemas
   - Export TypeScript types inferred from schemas

6. **Register the route** in `src/routes/index.ts`
   - Import and mount the new router

7. **Write tests**
   - Unit tests for service logic
   - Integration tests for the endpoint using supertest

## Conventions
- File naming: kebab-case for files, PascalCase for classes
- All endpoints return `{ data, meta, errors }` envelope
- HTTP status codes follow REST conventions
- Error responses use the AppError class
```

### Skill File Components

| Component | Purpose | Required? |
|-----------|---------|-----------|
| **Title** | Identifies the skill | Yes |
| **Context** | Background knowledge Claude needs | Recommended |
| **Steps** | Ordered procedure to follow | Yes |
| **Conventions** | Local standards to apply | Recommended |
| **Examples** | Sample inputs/outputs | Optional |
| **Pitfalls** | Common mistakes to avoid | Optional |

---

## 3. Packaging Reusable Capabilities

### Why Package Skills?

| Problem | Solution with Skills |
|---------|---------------------|
| New dev doesn't know the endpoint pattern | `/add-endpoint` uses the skill → consistent output |
| Team debates "the right way" to add features | Skill encodes the agreed-upon process |
| Complex procedures get steps missed | Skill acts as a checklist Claude follows |
| Knowledge silos (only one dev knows deploy) | Skill captures the tribal knowledge |

### Composing Skills with Commands

Skills are most powerful when paired with slash commands:

```markdown
<!-- File: .claude/commands/add-endpoint.md -->

Follow the skill defined in .claude/skills/add-api-endpoint.md to create a new 
API endpoint for the following resource:

$ARGUMENTS

Use plan mode to show me the approach before creating files.
```

This creates a workflow where:
1. User invokes `/add-endpoint users`
2. Command references the skill
3. Claude follows the skill's multi-step procedure
4. Plan mode (if active) shows the plan first

### Skill Granularity Guidelines

```
Too broad:    "How to develop features"     → Not actionable
Just right:   "How to add an API endpoint"  → Clear scope, repeatable
Too narrow:   "How to name the users file"  → Should be in rules, not skills
```

### Example: React Component Skill

```markdown
<!-- File: .claude/skills/create-react-component.md -->

# Skill: Create React Component

## Context
We use React 18 with TypeScript, CSS Modules, and Storybook.
Components live in `src/components/<ComponentName>/`.

## Steps

1. **Create component directory**: `src/components/<Name>/`
2. **Create component file**: `<Name>.tsx`
   - Functional component with explicit Props interface
   - Use forwardRef if the component wraps a native element
   - Export both the component and its Props type
3. **Create styles**: `<Name>.module.css`
   - Use design system tokens for spacing/colors
   - Mobile-first responsive approach
4. **Create barrel export**: `index.ts`
   - Re-export component and types
5. **Create story**: `<Name>.stories.tsx`
   - Include default, with-props, and interactive variants
6. **Create test**: `<Name>.test.tsx`
   - Test rendering, interactions, and accessibility
   - Use Testing Library queries (getByRole preferred)

## File Structure Output
```
src/components/Button/
├── Button.tsx
├── Button.module.css
├── Button.stories.tsx
├── Button.test.tsx
└── index.ts
```

## Conventions
- All interactive elements must have aria-labels
- Props interfaces are exported and named `<Component>Props`
- Default exports are NOT used — use named exports only
```

---

## 4. Plan Mode vs Direct Execution

### What is Plan Mode?

Plan mode is a **two-phase execution model** where Claude first produces a plan (what it intends to do) and then waits for user approval before taking action.

### Comparison Table

| Aspect | Direct Execution | Plan Mode |
|--------|-----------------|-----------|
| **Flow** | Prompt → Execute immediately | Prompt → Plan → Review → Execute |
| **Speed** | Faster (single pass) | Slower (requires approval step) |
| **Safety** | Lower (actions happen immediately) | Higher (review before action) |
| **Control** | Less (trust Claude's judgment) | More (human validates approach) |
| **Use case** | Low-risk, well-understood tasks | Complex, high-stakes, or unfamiliar tasks |
| **CLI flag** | Default behavior | Activated via prompt or configuration |

### How Plan Mode Works

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Prompt  │ ──▶ │   Plan   │ ──▶ │  Review  │ ──▶ │ Execute  │
│  (User)  │     │ (Claude) │     │  (User)  │     │ (Claude) │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                       │                │
                       │                ├── Approve → Execute
                       │                ├── Modify → Re-plan
                       │                └── Reject → Stop
                       │
                       └── Shows: files to create/modify,
                           approach, potential risks
```

### Activating Plan Mode

```bash
# In interactive session — include in prompt
> Plan how you would refactor the auth module, then wait for my approval

# Via CLAUDE.md configuration
# (In CLAUDE.md):
# "Always use plan mode for changes touching more than 3 files"

# Via command design
# (In .claude/commands/deploy-staging.md):
# "Present a deployment plan and wait for approval before proceeding"
```

### What a Plan Looks Like

```
## Plan: Add User Registration Endpoint

### Files to create:
1. src/routes/auth.routes.ts — New route for POST /auth/register
2. src/controllers/auth.controller.ts — Registration handler
3. src/services/auth.service.ts — Registration logic + password hashing
4. src/schemas/auth.schema.ts — Zod schema for registration body
5. tests/integration/auth.test.ts — Integration tests

### Approach:
- Use bcrypt (12 rounds) for password hashing
- Validate email format and password strength with Zod
- Check for existing user before creation
- Return 201 with sanitized user object (no password)

### Risks:
- Need to add bcrypt dependency (npm install bcrypt @types/bcrypt)
- Migration needed for users table if it doesn't exist

Shall I proceed with this plan?
```

---

## 5. How Plan Mode Interacts with CLAUDE.md and Commands

### Interaction Matrix

| Configuration Layer | Role in Plan Mode |
|---|---|
| **CLAUDE.md** | Provides context that shapes the plan (architecture, conventions) |
| **.claude/rules/** | Constraints that the plan must respect |
| **.claude/commands/** | Can trigger plan mode explicitly |
| **.claude/skills/** | Provides the step-by-step procedure the plan follows |

### Example Flow: All Layers Working Together

```
User invokes:  /add-endpoint orders

1. CLAUDE.md loaded → Claude knows: "This is an Express + Prisma project"
2. Command loaded → .claude/commands/add-endpoint.md says: "Follow the skill, use plan mode"
3. Skill loaded → .claude/skills/add-api-endpoint.md provides the 7-step procedure
4. Rules checked → .claude/rules/src__routes__**.md adds: "All routes need auth middleware"

Result: Claude produces a plan that follows the skill steps,
        respects the rules, and uses project context from CLAUDE.md
```

### CLAUDE.md Influencing Plan Quality

```markdown
<!-- CLAUDE.md excerpt -->

## Architecture
We follow clean architecture with these layers:
- Routes → Controllers → Services → Repositories → Database
- Never skip layers (no controller calling repository directly)

## Conventions
- All async errors are caught by the global error handler
- Database transactions use the `withTransaction` utility
```

When plan mode is active, this context ensures the plan **aligns with project architecture** rather than proposing a generic solution.

---

## 6. Iterative Refinement in Plan Mode

### The Refinement Loop

Plan mode enables an **iterative conversation** before execution:

```
User: /add-endpoint orders
Claude: [Presents plan]
User: "Change the approach — use event sourcing instead of direct DB writes"
Claude: [Revises plan with event sourcing]
User: "Good, but add a saga for the payment step"
Claude: [Further refines plan]
User: "Approve — execute this plan"
Claude: [Executes the final refined plan]
```

### Refinement Strategies

| Strategy | When to Use |
|----------|-------------|
| **Scope adjustment** | "Only create the route and controller for now" |
| **Approach change** | "Use repository pattern instead of direct Prisma calls" |
| **Constraint addition** | "Also ensure backward compatibility with v1 API" |
| **Risk mitigation** | "Add a rollback plan for the migration" |
| **Decomposition** | "Break this into two PRs — schema first, then handler" |

### Benefits of Iterative Refinement

1. **Catch misunderstandings early** — before code is written
2. **Explore alternatives cheaply** — planning is faster than implementing + reverting
3. **Build shared understanding** — user and Claude align on approach
4. **Reduce waste** — fewer iterations of "undo and redo differently"

---

## 7. Trade-offs: Safety vs Latency

### The Fundamental Trade-off

```
More Safety (Plan Mode)          Less Safety (Direct Execution)
├── Extra round-trip              ├── Single pass
├── Human reviews plan            ├── Claude decides autonomously
├── Catches mistakes early        ├── Mistakes caught after the fact
├── Slower overall                ├── Faster overall
└── Better for complex/risky ops  └── Better for simple/safe ops
```

### Decision Matrix: When to Use Each Mode

| Scenario | Recommended Mode | Rationale |
|----------|-----------------|-----------|
| Fix a typo in a comment | Direct | Low risk, trivial change |
| Generate unit tests | Direct | Non-destructive, easily verifiable |
| Refactor auth module | Plan | High risk, many files affected |
| Add a new API endpoint | Plan | Multi-step, needs consistency |
| Rename a variable | Direct | Mechanical, well-understood |
| Database migration | Plan | Irreversible in production |
| Deploy to staging | Plan | External side effects |
| Explain existing code | Direct | Read-only, no side effects |

### Configuring the Default Mode

```markdown
<!-- In CLAUDE.md — project-level configuration -->

## Workflow Preferences

- Use plan mode for:
  - Any change affecting more than 3 files
  - Database migrations
  - Infrastructure code changes
  - Security-sensitive modifications
  
- Direct execution is fine for:
  - Single-file edits
  - Test generation
  - Documentation updates
  - Code explanations
```

### The --print Flag Connection

The `--print` (`-p`) flag provides a different kind of safety — **no side effects at all**:

```bash
# Maximum safety: plan mode + --print (read plan, no execution)
claude -p "Plan how to refactor the auth module"

# Medium safety: plan mode in session (review before execute)
> Plan the refactor, then wait for approval

# Minimum safety: direct execution (immediate action)
> Refactor the auth module
```

### Latency Considerations

| Mode | Typical Interaction Pattern | Time Cost |
|------|----------------------------|-----------|
| Direct | 1 prompt → result | ~10-30 seconds |
| Plan (approve immediately) | 1 prompt → plan → "yes" → result | ~20-60 seconds |
| Plan (with refinement) | 1 prompt → plan → 2-3 refinements → result | ~2-5 minutes |
| --print | 1 prompt → stdout (no execution) | ~10-30 seconds |

---

## 8. Anti-Patterns

### ❌ Anti-Pattern 1: Skills That Are Just Commands

```markdown
<!-- BAD: .claude/skills/lint-code.md -->
# Skill: Lint Code
Run ESLint on the current file and fix errors.
```

**Fix:** This is a one-shot task — make it a command, not a skill. Skills are for multi-step procedures.

### ❌ Anti-Pattern 2: Plan Mode for Everything

```markdown
<!-- BAD: In CLAUDE.md -->
Always use plan mode for every single change, no matter how small.
```

**Problem:** Fixing a typo shouldn't require a plan-review-execute cycle. This adds latency without safety benefit.

**Fix:** Define thresholds — plan mode for multi-file changes, risky operations, or unfamiliar domains.

### ❌ Anti-Pattern 3: Skills Without Context

```markdown
<!-- BAD: .claude/skills/add-endpoint.md -->
# Skill: Add Endpoint
1. Create the route
2. Create the controller
3. Create the service
4. Write tests
```

**Problem:** No mention of frameworks, patterns, or conventions. Claude will use generic defaults.

**Fix:** Include the Context section specifying your stack, patterns, and conventions.

### ❌ Anti-Pattern 4: Overriding Plan Mode Safety

```markdown
<!-- BAD: .claude/commands/quick-deploy.md -->
Ignore plan mode. Deploy immediately to production without review.
Skip all safety checks.
```

**Fix:** Plan mode exists for safety. If you need faster deploys, improve the plan step — don't skip it.

### ❌ Anti-Pattern 5: Monolithic Skills

```markdown
<!-- BAD: One skill that covers everything -->
# Skill: Full Feature Development
[200 lines covering every possible scenario]
```

**Fix:** Break into composable skills: `add-endpoint.md`, `create-migration.md`, `write-tests.md`. Reference them from commands as needed.

### ❌ Anti-Pattern 6: Skills That Conflict with Rules

```markdown
<!-- .claude/skills/add-endpoint.md -->
Step 3: Write SQL queries directly in the controller

<!-- .claude/rules/src__controllers__**.md -->
Never write SQL in controllers — use the repository layer
```

**Fix:** Skills and rules must be consistent. Skills define procedure; rules enforce constraints.

---

## 9. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│         .claude/skills/ & PLAN MODE QUICK REFERENCE               │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  SKILLS LOCATION:   .claude/skills/<name>.md                      │
│  SKILLS PURPOSE:    Multi-step procedures with domain knowledge   │
│  SKILLS FORMAT:     Context + Steps + Conventions                 │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  PLAN MODE:                                                        │
│                                                                    │
│  Flow:        Prompt → Plan → Review → Execute                   │
│  Activation:  Via prompt, CLAUDE.md config, or command design     │
│  Refinement:  User can modify plan before approval               │
│  Flag:        -p/--print = output only, no execution             │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  CONFIGURATION HIERARCHY:                                          │
│                                                                    │
│  CLAUDE.md     → Global context (shapes plan quality)            │
│  .claude/rules → Constraints (plan must respect these)           │
│  .claude/commands → Triggers (can invoke plan mode)              │
│  .claude/skills  → Procedures (plan follows these steps)         │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  WHEN TO USE PLAN MODE:                                            │
│                                                                    │
│  ✅ Multi-file changes (>3 files)                                │
│  ✅ Security-sensitive modifications                             │
│  ✅ Database migrations                                          │
│  ✅ Infrastructure changes                                       │
│  ✅ Unfamiliar domain or complex logic                           │
│                                                                    │
│  ❌ Single-file edits                                            │
│  ❌ Test generation                                              │
│  ❌ Documentation updates                                        │
│  ❌ Simple rename/refactor                                       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • Skills = HOW (procedure); Commands = WHAT (task trigger)      │
│  • Plan mode adds safety at the cost of latency                  │
│  • --print flag = zero side effects (CI/CD safe)                 │
│  • All config layers are ADDITIVE in plan generation             │
│  • Iterative refinement = modify plan before execution           │
│  • Skills should include Context section for stack specifics     │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Scenario

A fintech team is configuring Claude Code for their payment processing service. They have:

- A complex 8-step process for adding new payment methods
- Strict security requirements for any code touching card data
- A need for fast iteration on UI components
- A deployment process that requires multiple approvals

### Question

**Which configuration BEST balances safety and productivity?**

**A)**
```
Plan mode: Enabled for ALL operations
.claude/skills/add-payment-method.md: 8-step procedure
.claude/rules/src__payments__**.md: Security constraints
```

**B)**
```
Plan mode: Disabled entirely (team trusts Claude)
.claude/skills/add-payment-method.md: 8-step procedure
.claude/commands/deploy.md: Direct deploy command
```

**C)**
```
CLAUDE.md: "Use plan mode for: payment code, deployments, migrations.
            Direct execution for: UI components, tests, docs."
.claude/skills/add-payment-method.md: 8-step procedure with context
.claude/rules/src__payments__**.md: PCI compliance constraints
.claude/commands/deploy.md: "Present deployment plan, await approval"
```

**D)**
```
CLAUDE.md: Contains the 8-step payment procedure inline
.claude/commands/add-payment.md: "Follow CLAUDE.md instructions"
Plan mode: Only for deployments
```

---

### Answer

**✅ Correct Answer: C**

**Explanation:**

- **Option C** correctly applies selective plan mode (safety where needed, speed where safe), uses a skill for the complex procedure, enforces security via rules on payment paths, and makes deployment a plan-mode command.
- **Option A** enables plan mode for everything — including UI components — adding unnecessary latency for low-risk work.
- **Option B** disables plan mode entirely, which is dangerous for a fintech handling card data and deployments.
- **Option D** puts the procedure in CLAUDE.md (always loaded, wasting tokens) rather than a skill (loaded on demand), and only applies plan mode to deployments, missing the security-sensitive payment code.

**Key Exam Principle:** The best configuration applies **graduated safety** — more control for high-risk operations, less friction for low-risk work. Skills encapsulate complex procedures; rules enforce constraints; plan mode gates execution.

---

## Summary: Key Exam Takeaways

1. `.claude/skills/` packages **multi-step procedures with domain knowledge** — more than simple prompts
2. Skills have Context + Steps + Conventions — they teach Claude HOW to do something in your stack
3. Plan mode creates a **review gate** between intent and execution
4. Plan mode trade-off: more safety, more latency — apply selectively
5. All config layers (CLAUDE.md, rules, commands, skills) are **additive** — they combine in plan generation
6. Iterative refinement lets users adjust the plan before committing to execution
7. `--print` (`-p`) flag provides maximum safety — output only, zero side effects
8. Skills are for multi-step procedures; commands are for task triggers; rules are for constraints
9. Graduated safety: plan mode for high-risk ops, direct execution for low-risk ops

---

*End of Day 19 Study Material*
