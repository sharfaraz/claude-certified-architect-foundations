# Day 24: Environment Variable Expansion and Built-in Tools

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 2: Tool Design & MCP Integration (18% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Begin MCP: Advanced Topics |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Environment Variable Expansion in .mcp.json](#1-environment-variable-expansion-in-mcpjson)
2. [Security Implications of Env Var References](#2-security-implications-of-env-var-references)
3. [Built-in Claude Code Tools](#3-built-in-claude-code-tools)
4. [Built-in Tools vs MCP Tools: Interaction and Precedence](#4-built-in-tools-vs-mcp-tools-interaction-and-precedence)
5. [Allowlists and Denylists](#5-allowlists-and-denylists)
6. [The --output-format json Flag](#6-the---output-format-json-flag)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code MCP Configuration | https://docs.anthropic.com/en/docs/claude-code/mcp |
| Claude Code CLI Reference | https://docs.anthropic.com/en/docs/claude-code/cli |
| MCP Security Best Practices | https://spec.modelcontextprotocol.io/specification/security/ |

---

## 1. Environment Variable Expansion in .mcp.json

### The `${VAR_NAME}` Syntax

MCP server configurations in `.mcp.json` support environment variable expansion using the `${VAR_NAME}` syntax. This allows secrets and environment-specific values to be referenced without hardcoding them.

### Basic Syntax

```json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"],
      "env": {
        "PGPASSWORD": "${DB_PASSWORD}",
        "PGHOST": "${DB_HOST}",
        "PGPORT": "${DB_PORT}"
      }
    }
  }
}
```

### How Expansion Works

```
.mcp.json contains:        "${DATABASE_URL}"
Shell environment has:      DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
At server startup:          Resolves to "postgresql://user:pass@localhost:5432/mydb"
```

### Supported Locations for `${VAR_NAME}`

| Location | Example | Use Case |
|----------|---------|----------|
| `args[]` array | `"${API_KEY}"` | Passing secrets as CLI arguments |
| `env` object values | `"PGPASSWORD": "${DB_PASSWORD}"` | Setting server environment |
| `command` field | `"${MCP_BINARY_PATH}"` | Dynamic binary path (rare) |

### Multiple Variables in One String

```json
{
  "args": ["--connection", "postgresql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}"]
}
```

### Complete .mcp.json Example with Env Vars

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${SLACK_TEAM_ID}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"],
      "env": {}
    }
  }
}
```

---

## 2. Security Implications of Env Var References

### Why Env Vars Are Critical for Security

The `.mcp.json` file is typically **committed to version control** (it defines the project's MCP server setup). Without env var expansion, secrets would be exposed in the repository.

### The Security Model

```
┌─────────────────────────────────────────────────────────┐
│  VERSION CONTROL (.mcp.json committed to git)           │
│                                                          │
│  ✅ Safe: "${GITHUB_TOKEN}" — placeholder, no secret    │
│  ❌ Dangerous: "ghp_abc123xyz..." — actual token        │
│                                                          │
├─────────────────────────────────────────────────────────┤
│  LOCAL ENVIRONMENT (not committed)                       │
│                                                          │
│  ~/.bashrc, .env, CI secrets:                           │
│  GITHUB_TOKEN=ghp_abc123xyz...                          │
│  DATABASE_URL=postgresql://user:pass@host/db            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Security Best Practices

| Practice | Rationale |
|----------|-----------|
| Always use `${VAR}` for secrets | Keeps credentials out of version control |
| Use `.env` files locally (gitignored) | Convenient local development |
| Use CI/CD secret stores in pipelines | Secure secret injection at runtime |
| Scope tokens with minimum permissions | Limit blast radius if compromised |
| Document required env vars in README | Team members know what to configure |
| Never log resolved env var values | Prevent accidental secret exposure |

### What Happens When a Variable Is Missing?

If `${VAR_NAME}` references a variable that doesn't exist in the environment:
- The server startup will typically **fail** with an error
- The variable resolves to an empty string in some implementations
- This is a common debugging issue — always check env var availability

---

## 3. Built-in Claude Code Tools

### What Are Built-in Tools?

Claude Code comes with a set of **always-available tools** that don't require MCP server configuration. These are the foundational capabilities Claude Code uses to interact with the development environment.

### Core Built-in Tools

| Tool Category | Capabilities | Examples |
|--------------|--------------|----------|
| **File Operations** | Read, write, edit files | Read file, Write file, Edit file |
| **Terminal/Shell** | Execute commands | Run bash commands, scripts |
| **Search** | Find content in codebase | Grep, file search, symbol search |
| **Web Search** | Look up information online | Search web for documentation |
| **Directory** | Navigate filesystem | List directory contents |

### Built-in Tool Characteristics

```
Built-in Tools:
  ├── Always available (no configuration needed)
  ├── Provided by Claude Code itself (not MCP servers)
  ├── Cover fundamental dev operations
  ├── Cannot be removed (but can be restricted)
  └── Take precedence in naming conflicts
```

### How Built-in Tools Appear to Claude

Claude sees built-in tools as part of its capability set alongside MCP tools:

```
Available capabilities:
├── Built-in:
│   ├── Read file (path) → file content
│   ├── Write file (path, content) → confirmation
│   ├── Execute command (command) → stdout/stderr
│   ├── Search files (pattern) → matches
│   └── Web search (query) → results
│
└── MCP (from configured servers):
    ├── [postgres] query_database
    ├── [github] create_pull_request
    └── [jira] search_issues
```

---

## 4. Built-in Tools vs MCP Tools: Interaction and Precedence

### Precedence Rules

When built-in tools and MCP tools have overlapping capabilities, Claude Code follows a precedence model:

| Scenario | Behavior |
|----------|----------|
| Same name conflict | Built-in tools take precedence |
| Overlapping capability | Claude chooses based on context and description |
| MCP tool adds new capability | Added to available tool set alongside built-ins |
| MCP tool specializes a built-in | Both available, AI picks the more specific one |

### Naming Considerations

```json
// ❌ BAD: MCP tool name conflicts with built-in
{
  "name": "read_file",  // Conflicts with built-in file read
  "description": "Read a file from the repo"
}

// ✅ GOOD: Specific name that doesn't conflict
{
  "name": "read_config_from_vault",
  "description": "Read a configuration file from the encrypted vault"
}
```

### How Claude Selects Between Tools

```
User: "What's in the database schema?"

Claude evaluates:
  1. Built-in "Read file" → Could read schema.sql from disk
  2. MCP "postgres://get_schema" → Gets live schema from database

Decision factors:
  - Tool descriptions (which is more specific to the intent?)
  - Context (is the user asking about a file or the live database?)
  - Specificity (MCP tool is purpose-built for this exact task)

Result: Claude picks the MCP postgres tool (more specific, more appropriate)
```

### Integration Model

```
┌────────────────────────────────────────────────────┐
│              CLAUDE CODE TOOL REGISTRY              │
├────────────────────────────────────────────────────┤
│                                                      │
│  Priority 1: Built-in tools (always available)      │
│              - File operations                       │
│              - Terminal commands                     │
│              - Search (grep, files)                  │
│              - Web search                           │
│                                                      │
│  Priority 2: MCP tools (from .mcp.json servers)    │
│              - Custom integrations                  │
│              - External service access              │
│              - Specialized capabilities             │
│                                                      │
│  Selection: AI picks based on description match,   │
│  specificity, and user intent                      │
│                                                      │
└────────────────────────────────────────────────────┘
```

---

## 5. Allowlists and Denylists

### Controlling Tool Access

Claude Code supports **allowlists** and **denylists** to control which tools Claude can use during a session. This is critical for security and for scoping capabilities in automated environments.

### Configuration Approaches

```
Allowlist (whitelist):
  "Only these tools are permitted"
  → Restrictive by default, explicit permission

Denylist (blacklist):
  "Everything EXCEPT these tools is permitted"
  → Permissive by default, explicit restriction
```

### Use Cases

| Approach | Use Case | Example |
|----------|----------|---------|
| **Allowlist** | CI pipelines (restrict capabilities) | Only allow read, search, and output tools |
| **Allowlist** | Security-sensitive environments | Only allow specific MCP tools |
| **Denylist** | General development (block dangerous ops) | Block file deletion, network access |
| **Denylist** | Shared environments | Block specific MCP servers |

### How This Relates to the Exam

The exam tests understanding of:
- **When to restrict** tool access (CI pipelines, automated runs)
- **How allowlists/denylists interact** with built-in and MCP tools
- **Security implications** of overly broad access

### Example: CI Pipeline Restriction

```bash
# In CI, restrict Claude to only safe, read-only operations
claude -p "Review this PR" --allowedTools "read_file,search,web_search"
```

### Example: Blocking Dangerous Tools

```bash
# Block file deletion and terminal execution
claude --denyTools "delete_file,execute_command"
```

---

## 6. The --output-format json Flag

### Purpose

The `--output-format json` flag makes Claude Code output its responses as **structured JSON** instead of human-readable text. This is essential for programmatic consumption in scripts, CI pipelines, and toolchain integration.

### Basic Usage

```bash
# Human-readable output (default)
claude -p "List all TODO comments in src/"

# Structured JSON output (for scripts)
claude -p "List all TODO comments in src/" --output-format json
```

### JSON Output Structure

```json
{
  "type": "result",
  "subtype": "success",
  "cost_usd": 0.003,
  "is_error": false,
  "duration_ms": 2150,
  "duration_api_ms": 1850,
  "num_turns": 1,
  "result": "Found 5 TODO comments:\n1. src/auth.ts:42 - TODO: Add rate limiting\n2. ...",
  "session_id": "abc123"
}
```

### Use Cases for --output-format json

| Use Case | Why JSON? |
|----------|-----------|
| CI/CD pipelines | Parse results programmatically |
| Script automation | Extract specific fields from output |
| Toolchain integration | Feed Claude output to other tools |
| Monitoring/logging | Structured data for observability |
| Cost tracking | Extract `cost_usd` field |

### Combining with -p (Print Mode)

```bash
# Non-interactive + structured output = CI-ready
claude -p "Generate migration script for adding email column" --output-format json | jq '.result'
```

### Key Fields in JSON Output

| Field | Type | Description |
|-------|------|-------------|
| `result` | string | The actual response content |
| `is_error` | boolean | Whether an error occurred |
| `cost_usd` | number | API cost for this request |
| `duration_ms` | number | Total execution time |
| `num_turns` | number | Number of conversation turns |
| `session_id` | string | Session identifier for --resume |

---

## 7. Anti-Patterns

### ❌ Anti-Pattern 1: Hardcoding Secrets in .mcp.json

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

**Problem:** Secret committed to version control. Anyone with repo access gets the token.

**Fix:** Use environment variable expansion:
```json
{
  "env": {
    "GITHUB_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
  }
}
```

### ❌ Anti-Pattern 2: Overly Broad Tool Access

```bash
# Running Claude in CI with ALL tools available (including write, delete, execute)
claude -p "Review this code and suggest improvements"
# Claude has access to: write files, delete files, run commands, etc.
```

**Problem:** In a CI review context, Claude should only read and analyze — not modify files or execute commands.

**Fix:** Restrict to read-only tools:
```bash
claude -p "Review this code" --allowedTools "read_file,search,grep"
```

### ❌ Anti-Pattern 3: Using Interactive Mode in CI

```bash
# CI pipeline step
claude "Generate API docs"  # Hangs waiting for interactive input
```

**Problem:** CI environments have no terminal for interactive input. The process hangs indefinitely.

**Fix:** Always use `-p` / `--print` in CI:
```bash
claude -p "Generate API docs" --output-format json
```

### ❌ Anti-Pattern 4: MCP Tool Names Shadowing Built-ins

```json
{
  "name": "search",
  "description": "Search our internal wiki"
}
```

**Problem:** Generic name "search" may conflict with Claude Code's built-in search. Claude might call the wrong one.

**Fix:** Use specific, namespaced names:
```json
{
  "name": "search_internal_wiki",
  "description": "Search the company internal wiki for articles and documentation"
}
```

### ❌ Anti-Pattern 5: Not Documenting Required Env Vars

```json
{
  "mcpServers": {
    "custom-server": {
      "command": "node",
      "args": ["server.js"],
      "env": {
        "API_KEY": "${CUSTOM_API_KEY}",
        "ENDPOINT": "${SERVICE_ENDPOINT}",
        "SECRET": "${AUTH_SECRET}"
      }
    }
  }
}
```

**Problem:** Team members clone the repo and Claude Code fails silently because they don't know which env vars to set.

**Fix:** Document required variables in README or a `.env.example` file:
```bash
# .env.example (committed to repo)
CUSTOM_API_KEY=your-api-key-here
SERVICE_ENDPOINT=https://api.example.com
AUTH_SECRET=your-auth-secret-here
```

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│         ENV VARS & BUILT-IN TOOLS QUICK REFERENCE                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ENV VAR EXPANSION:                                                │
│    Syntax:    ${VAR_NAME}                                        │
│    Location:  .mcp.json (args[], env{}, command)                 │
│    Purpose:   Keep secrets out of version control                 │
│    Behavior:  Resolved at server startup from shell environment  │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  BUILT-IN TOOLS:                                                   │
│                                                                    │
│    File ops:    Read, Write, Edit files                           │
│    Terminal:    Execute shell commands                            │
│    Search:     Grep, file search, symbol search                  │
│    Web:        Web search for online info                        │
│    Directory:  List and navigate filesystem                      │
│                                                                    │
│  PRECEDENCE: Built-in > MCP tools (on name conflict)             │
│  SELECTION:  AI picks by description specificity + context       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  TOOL ACCESS CONTROL:                                              │
│                                                                    │
│    Allowlist:  Only named tools permitted (restrictive)           │
│    Denylist:   Named tools blocked, rest permitted (permissive)  │
│    Use case:   CI pipelines, security-sensitive environments     │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  CLI FLAGS:                                                        │
│                                                                    │
│    --output-format json  → Structured JSON output for scripts    │
│    -p / --print          → Non-interactive mode (CI-ready)       │
│    --allowedTools        → Restrict available tools              │
│    --denyTools           → Block specific tools                  │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • ${VAR_NAME} in .mcp.json = safe secret handling               │
│  • NEVER hardcode secrets in .mcp.json (version controlled!)     │
│  • Built-in tools are ALWAYS available (can't be removed)        │
│  • Built-in tools take precedence on name conflicts              │
│  • Use allowlists in CI for minimal-permission principle         │
│  • Always use -p in CI (non-interactive is mandatory)            │
│  • --output-format json enables programmatic consumption         │
│  • Document required env vars for team collaboration             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

Your team is configuring Claude Code for a monorepo. The `.mcp.json` needs to connect to three services:
- A PostgreSQL database (connection string in `DATABASE_URL`)
- A GitHub API (token in `GH_TOKEN`)
- An internal documentation server (API key in `DOCS_API_KEY`)

Additionally, the CI pipeline needs to run Claude Code to generate API documentation non-interactively and parse the output programmatically.

### Question

**Which configuration and invocation combination is CORRECT and follows security best practices?**

**A)**
```json
// .mcp.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "postgresql://admin:password123@prod-db:5432/app"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@mcp/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_realtoken123" }
    }
  }
}
```
```bash
# CI invocation
claude "Generate API docs"
```

**B)**
```json
// .mcp.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "${DATABASE_URL}"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@mcp/server-github"],
      "env": { "GITHUB_TOKEN": "${GH_TOKEN}" }
    },
    "docs": {
      "command": "npx",
      "args": ["-y", "@internal/docs-server"],
      "env": { "API_KEY": "${DOCS_API_KEY}" }
    }
  }
}
```
```bash
# CI invocation
claude -p "Generate API documentation from src/" --output-format json
```

**C)**
```json
// .mcp.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "${DATABASE_URL}"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@mcp/server-github"],
      "env": { "GITHUB_TOKEN": "${GH_TOKEN}" }
    }
  }
}
```
```bash
# CI invocation
claude -p "Generate API docs" --resume last-session
```

**D)**
```json
// .mcp.json
{
  "mcpServers": {
    "all-services": {
      "command": "npx",
      "args": ["-y", "@mcp/super-server", "--pg=${DATABASE_URL}", "--gh=${GH_TOKEN}", "--docs=${DOCS_API_KEY}"]
    }
  }
}
```
```bash
# CI invocation
claude -p "Generate API docs" --output-format json
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** follows all best practices:
  - All secrets use `${VAR_NAME}` expansion (never hardcoded)
  - Each service has its own dedicated MCP server (separation of concerns)
  - CI invocation uses `-p` for non-interactive mode
  - `--output-format json` enables programmatic output parsing
  - All three required services are configured

- **Option A** hardcodes secrets directly in `.mcp.json` (critical security anti-pattern) and uses interactive mode in CI (will hang).

- **Option C** uses env vars correctly and `-p` for non-interactive, but is missing the docs server configuration and uses `--resume` which isn't appropriate for generating fresh documentation in CI.

- **Option D** bundles all services into one "super server" (poor separation of concerns) and passes secrets as command args (could appear in process listings).

**Key Exam Principle:** Always use `${VAR_NAME}` for secrets in `.mcp.json`, separate MCP servers by concern, and use `-p` + `--output-format json` for CI pipelines.

---

## Summary: Key Exam Takeaways

1. `${VAR_NAME}` syntax in `.mcp.json` resolves environment variables at server startup
2. **Never hardcode secrets** in `.mcp.json` — it's committed to version control
3. Built-in tools (file ops, terminal, search, web) are always available in Claude Code
4. Built-in tools take **precedence** on name conflicts with MCP tools
5. Use **allowlists** in CI to enforce minimal permissions (principle of least privilege)
6. `--output-format json` produces structured output for programmatic consumption
7. `-p` / `--print` is **mandatory** for CI/CD (non-interactive environments)
8. Document required env vars so team members can set up their environment
9. Use specific, descriptive MCP tool names to avoid shadowing built-in tools

---

*End of Day 24 Study Material*
