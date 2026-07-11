# Day 16 — CLAUDE.md Hierarchy and Scoping

**Domain**: Domain 3: Claude Code Configuration & Workflows (20% of exam)
**Sprint**: Sprint 2 — The Stack (Days 16–30)
**Academy Course**: Begin Claude Code 101
**Estimated Study Time**: 2–3 hours

---

## Table of Contents

1. [What Is CLAUDE.md?](#1-what-is-claudemd)
2. [The Hierarchy: Global → Project → Subdirectory](#2-the-hierarchy-global--project--subdirectory)
3. [Scoping Rules](#3-scoping-rules)
4. [Content Patterns](#4-content-patterns)
5. [When to Use Each Level](#5-when-to-use-each-level)
6. [Anti-Patterns](#6-anti-patterns)
7. [Quick Reference Card](#7-quick-reference-card)
8. [Scenario Challenge](#8-scenario-challenge)

---

## Recommended Video Resources

- [Anthropic — Using CLAUDE.md Files](https://www.claude.com/blog/using-claude-md-files) — Official blog post with configuration patterns
- [Anthropic — Best Practices for Claude Code](https://www.anthropic.com/engineering/claude-code-best-practices) — Official engineering guidance
- [Anthropic Docs — How Claude Remembers Your Project](https://docs.claude.com/en/docs/claude-code/memory) — CLAUDE.md and auto memory docs

---

## 1. What Is CLAUDE.md?

CLAUDE.md is a special configuration file that provides Claude Code with persistent, project-specific instructions. It acts as Claude's "memory" of your project — coding standards, conventions, file paths, tool usage rules, and workflow preferences.

**Why this matters for the exam**: CLAUDE.md hierarchy and scoping are core Domain 3 topics. The exam tests whether you know where to place instructions (global vs. project vs. subdirectory) and how scoping rules work when multiple CLAUDE.md files exist.

### Key Properties

- **Automatically read** at the start of every Claude Code session
- **Markdown format** — plain text with standard markdown formatting
- **Hierarchical** — multiple files at different levels combine and override
- **Version-controlled** — project-level CLAUDE.md can be shared via git
- **Persistent** — survives across sessions (unlike conversation history)

---

## 2. The Hierarchy: Global → Project → Subdirectory

CLAUDE.md files exist at three levels, processed in order from broadest to narrowest scope:

```
~/.claude/CLAUDE.md                    ← Level 1: Global (user-level)
    │
    ▼
/project/CLAUDE.md                     ← Level 2: Project root
    │
    ▼
/project/src/CLAUDE.md                 ← Level 3: Subdirectory
/project/packages/shared/CLAUDE.md     ← Level 3: Another subdirectory
```

### Processing Order

1. **Global** (`~/.claude/CLAUDE.md`) — Read first; applies to ALL projects
2. **Project root** (`./CLAUDE.md`) — Read second; applies to this project
3. **Subdirectory** (`./src/CLAUDE.md`) — Read when working in that directory; scoped to that path

All applicable files are combined. Later files can extend or override earlier ones.

---

## 3. Scoping Rules

### Rule 1: Global Applies Everywhere

Instructions in `~/.claude/CLAUDE.md` apply to every project, every session. Use for personal preferences that span all work.

### Rule 2: Project Root Applies Project-Wide

Instructions in the project root `CLAUDE.md` apply anywhere within that project. This is where team-shared conventions live.

### Rule 3: Subdirectory Files Scope to That Path

A `CLAUDE.md` in `src/` only applies when Claude is working with files under `src/`. It extends (not replaces) the project-level instructions.

### Rule 4: More Specific Overrides More General

When instructions conflict, the more specific (deeper) file takes precedence:

```
Global: "Use tabs for indentation"
Project: "Use 2-space indentation"        ← Overrides global for this project
src/legacy/: "Use 4-space indentation"    ← Overrides project for this directory
```

### Rule 5: Extension, Not Replacement

A subdirectory CLAUDE.md doesn't replace the project-level one — it adds to it. Both apply simultaneously. Only conflicting instructions trigger the override behavior.

---

## 4. Content Patterns

### What Goes in CLAUDE.md

| Content Type | Example |
|-------------|---------|
| Coding standards | "Use TypeScript strict mode. No any types." |
| Project structure | "API routes are in src/routes/. Tests are in tests/." |
| Build/test commands | "Run tests with: npm run test" |
| Naming conventions | "Use camelCase for functions, PascalCase for classes" |
| Forbidden actions | "Never modify files in packages/shared/ without approval" |
| Tool usage | "Always run linter after code changes" |
| File path context | "The main entry point is src/index.ts" |
| Dependencies | "Use axios for HTTP requests, not fetch" |

### Example: Project Root CLAUDE.md

```markdown
# Project: ShopCo API

## Tech Stack
- TypeScript 5.x with strict mode
- Express.js for HTTP server
- PostgreSQL with Prisma ORM
- Jest for testing

## Coding Standards
- Use 2-space indentation
- No default exports — always named exports
- All functions must have explicit return types
- Use Zod for runtime validation at API boundaries

## Project Structure
- src/routes/ — Express route handlers
- src/services/ — Business logic
- src/models/ — Prisma schema and generated types
- src/utils/ — Shared utility functions
- tests/ — Jest test files (mirror src/ structure)

## Commands
- Build: `npm run build`
- Test: `npm run test`
- Lint: `npm run lint`
- Dev: `npm run dev`

## Rules
- Always run `npm run lint` after modifying TypeScript files
- Never commit code that fails type checking
- All new API endpoints need corresponding test files
```

---

## 5. When to Use Each Level

| Level | Use Case | Shared? |
|-------|----------|---------|
| Global (`~/.claude/CLAUDE.md`) | Personal coding preferences, editor habits, response style | No — personal only |
| Project root (`./CLAUDE.md`) | Team conventions, project structure, build commands | Yes — via git |
| Subdirectory | Path-specific rules, legacy code handling, different standards per module | Yes — via git |

### Decision Guide

```
Is this a personal preference that applies to all my projects?
  → Global (~/.claude/CLAUDE.md)

Is this a project convention the whole team should follow?
  → Project root (./CLAUDE.md) — commit to git

Is this specific to one part of the codebase?
  → Subdirectory (e.g., ./packages/legacy/CLAUDE.md)
```

---

## 6. Anti-Patterns

### Anti-Pattern 1: Conflicting Instructions Without Clear Precedence

```
# Project CLAUDE.md
Use semicolons at end of lines.

# src/scripts/ CLAUDE.md
No semicolons — this is a semicolons-free zone.
```

This works (subdirectory overrides project), but can confuse team members. Add a note explaining the exception.

### Anti-Pattern 2: Putting Path-Specific Rules in Global

```
# ~/.claude/CLAUDE.md
Never modify files in /projects/shopco/packages/shared/
```

This belongs in the project's CLAUDE.md or a subdirectory CLAUDE.md, not in global. Global should only contain universal preferences.

### Anti-Pattern 3: Overly Long CLAUDE.md

A CLAUDE.md with 500+ lines of instructions dilutes the important content. Claude actively filters content it deems irrelevant to the current task. Keep instructions focused and concise.

### Anti-Pattern 4: Duplicating Instructions Across Levels

```
# Global
Use TypeScript strict mode.

# Project
Use TypeScript strict mode.  ← Redundant! Already in global
```

---

## 7. Quick Reference Card

| Location | Scope | Shared via Git | Override Behavior |
|----------|-------|:-:|----------------|
| `~/.claude/CLAUDE.md` | All projects | No | Base layer (lowest priority) |
| `./CLAUDE.md` | Project-wide | Yes | Overrides global |
| `./subdir/CLAUDE.md` | That directory | Yes | Overrides project (for that path) |

### Exam Key Facts

- CLAUDE.md is read at session start, not per-message
- Multiple CLAUDE.md files combine (extend, not replace)
- Deeper files override conflicting instructions from broader files
- Content is markdown format
- Claude filters irrelevant instructions for the current task

---

## 8. Scenario Challenge

**Scenario**: You're configuring Claude Code for a TypeScript monorepo. The team wants:
1. Claude to always run `tsc --noEmit` after generating code
2. Follow the project's ESLint configuration
3. Never modify files in `packages/shared/` without explicit approval

Where should each rule be configured?

A) All three in the project root CLAUDE.md  
B) `tsc` in a slash command, ESLint in CLAUDE.md, `packages/shared/` in `.claude/rules/`  
C) All three in `.claude/rules/` with path-specific matching  
D) `tsc` and ESLint in CLAUDE.md, `packages/shared/` restriction in `packages/shared/CLAUDE.md`

**Answer**: D — Project-wide conventions (type checking, linting) belong in root CLAUDE.md. Directory-specific restriction belongs in a subdirectory CLAUDE.md scoped to that path. B misuses slash commands for automated checks. C puts global rules in path-specific files.
