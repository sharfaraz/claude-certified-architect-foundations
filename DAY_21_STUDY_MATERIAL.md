# Day 21: MCP Server Configuration (.mcp.json)

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 2: Tool Design & MCP Integration (18% of exam) |
| **Sprint** | Sprint 2 — The Stack (Days 16–30) |
| **Academy Course** | Begin Introduction to MCP |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [MCP Architecture Overview](#1-mcp-architecture-overview)
2. [The .mcp.json Configuration File](#2-the-mcpjson-configuration-file)
3. [Defining Servers: Name, Command, Args, Env](#3-defining-servers-name-command-args-env)
4. [Local vs Remote Servers](#4-local-vs-remote-servers)
5. [How Claude Code Discovers and Connects to MCP Servers](#5-how-claude-code-discovers-and-connects-to-mcp-servers)
6. [Configuration Examples](#6-configuration-examples)
7. [Anti-Patterns](#7-anti-patterns)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Scenario Challenge](#9-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Code MCP Documentation | https://docs.anthropic.com/en/docs/claude-code/mcp |
| MCP Specification | https://github.com/modelcontextprotocol/specification |

---

## 1. MCP Architecture Overview

### What is MCP?

The **Model Context Protocol (MCP)** is an open standard that defines how AI applications (clients) communicate with external tools and data sources (servers). It provides a structured, discoverable way to extend AI capabilities beyond built-in tools.

### Core Architecture: Three Layers

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   ┌──────────┐        ┌──────────────┐       ┌──────────┐  │
│   │   Host   │◀──────▶│  MCP Client  │◀─────▶│MCP Server│  │
│   │(Claude   │        │  (Protocol   │       │(External │  │
│   │ Code)    │        │   Layer)     │       │ Service) │  │
│   └──────────┘        └──────────────┘       └──────────┘  │
│                                                               │
│   The HOST is the AI application (Claude Code)               │
│   The CLIENT handles protocol communication                  │
│   The SERVER exposes tools, resources, and prompts           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Example |
|-----------|------|---------|
| **Host** | The AI application that embeds the MCP client | Claude Code, Claude Desktop |
| **Client** | Manages protocol communication with servers | Built into Claude Code |
| **Server** | Exposes capabilities (tools, resources, prompts) | A Node.js process, a Python script |
| **Transport** | Communication channel between client and server | stdio, HTTP+SSE |

### What MCP Servers Provide

MCP servers can expose three types of capabilities:

| Capability | Description | Example |
|------------|-------------|---------|
| **Tools** | Functions Claude can call | `query_database`, `create_jira_ticket` |
| **Resources** | Data/context Claude can read | Database schemas, API docs, config files |
| **Prompts** | Template prompts for common tasks | "Summarize this PR", "Generate SQL" |

### Protocol Flow

```
1. Claude Code starts → reads .mcp.json
2. For each server: spawn process (stdio) or connect (HTTP)
3. Client sends initialize request → server responds with capabilities
4. Client calls tools/list → discovers available tools
5. During conversation: Claude decides to use a tool → client routes to server
6. Server executes tool → returns result → client passes to Claude
```

---

## 2. The .mcp.json Configuration File

### Location and Purpose

The `.mcp.json` file tells Claude Code **which MCP servers to connect to and how to start them**. It lives at the project root.

### File Locations (Resolution Order)

| Location | Scope | Shared? |
|----------|-------|---------|
| `./.mcp.json` (project root) | Project-level | ✅ Via Git |
| `~/.claude/.mcp.json` | User-level (global) | ❌ Personal |

### Basic Structure

```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "<executable>",
      "args": ["<arg1>", "<arg2>"],
      "env": {
        "<KEY>": "<VALUE>"
      }
    }
  }
}
```

### Schema Breakdown

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mcpServers` | object | Yes | Container for all server definitions |
| `<server-name>` | string (key) | Yes | Unique identifier for this server |
| `command` | string | Yes | The executable to run |
| `args` | string[] | No | Command-line arguments |
| `env` | object | No | Environment variables for the process |

### Minimal Example

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"]
    }
  }
}
```

### Full Example with Multiple Servers

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "custom-tools": {
      "command": "node",
      "args": ["./tools/mcp-server.js"],
      "env": {
        "API_BASE_URL": "https://api.internal.company.com"
      }
    }
  }
}
```

---

## 3. Defining Servers: Name, Command, Args, Env

### Server Name

The server name is the **key** in the `mcpServers` object. It serves as:
- A unique identifier for the server within the project
- The label shown in Claude Code's UI when listing connected servers
- A reference in logs and error messages

**Naming conventions:**
```json
{
  "mcpServers": {
    "postgres": {},          // ✅ Short, descriptive
    "github": {},            // ✅ Service name
    "internal-api": {},      // ✅ Kebab-case for multi-word
    "my_first_server": {},   // ⚠️ Works but prefer hyphens
    "": {}                   // ❌ Empty names not allowed
  }
}
```

### Command

The `command` field specifies **what executable to run** to start the MCP server:

| Command Type | Example | Use Case |
|---|---|---|
| `npx` | `"command": "npx"` | Run npm-published MCP servers |
| `node` | `"command": "node"` | Run local JavaScript servers |
| `python` | `"command": "python"` | Run local Python servers |
| `uvx` | `"command": "uvx"` | Run Python packages (like npx for pip) |
| `docker` | `"command": "docker"` | Run containerized servers |
| Direct path | `"command": "./bin/my-server"` | Run compiled binaries |

### Args

The `args` array passes **command-line arguments** to the server process:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",                                    // Auto-confirm npx install
        "@modelcontextprotocol/server-filesystem", // Package name
        "/Users/dev/projects",                    // Allowed directory
        "/Users/dev/documents"                    // Another allowed directory
      ]
    }
  }
}
```

### Env (Environment Variables)

The `env` object sets **environment variables** available to the server process:

```json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/dev",
        "MAX_CONNECTIONS": "10",
        "LOG_LEVEL": "info"
      }
    }
  }
}
```

### Environment Variable Patterns

| Pattern | Example | Description |
|---------|---------|-------------|
| Literal value | `"API_KEY": "sk-abc123"` | ⚠️ Avoid for secrets |
| Shell expansion | `"TOKEN": "${GITHUB_TOKEN}"` | References system env var |
| Computed path | `"DB_PATH": "${HOME}/.local/data.db"` | Uses system variables |

### Important: The `${VAR}` Pattern

```json
{
  "env": {
    "GITHUB_TOKEN": "${GITHUB_TOKEN}"
  }
}
```

This tells Claude Code to read `GITHUB_TOKEN` from the **host system's environment** at runtime, rather than hardcoding the value. This is the **recommended pattern for secrets**.

---

## 4. Local vs Remote Servers

### Local Servers (stdio Transport)

Local servers run as **child processes** on the same machine. Communication happens via standard I/O (stdin/stdout).

```
┌──────────────┐    stdio    ┌──────────────┐
│  Claude Code │◀──────────▶│  MCP Server  │ (child process)
│  (parent)    │ stdin/stdout│  (local)     │
└──────────────┘             └──────────────┘
```

**Characteristics:**
- Started automatically by Claude Code
- Lifecycle managed by the client (start on connect, stop on disconnect)
- Communication via stdin/stdout (fast, no network)
- Server process inherits security context of the user

### Remote Servers (HTTP+SSE Transport)

Remote servers run on **separate machines** and communicate over HTTP with Server-Sent Events.

```
┌──────────────┐   HTTP/SSE   ┌──────────────┐
│  Claude Code │◀────────────▶│  MCP Server  │ (remote host)
│  (client)    │   network    │  (external)  │
└──────────────┘              └──────────────┘
```

**Characteristics:**
- Must be started/managed separately (not by Claude Code)
- Communication over network (HTTP + SSE for streaming)
- Can be shared across multiple clients/users
- Requires authentication and secure transport

### Comparison Table

| Aspect | Local (stdio) | Remote (HTTP+SSE) |
|--------|--------------|-------------------|
| **Startup** | Auto-started by Claude Code | Pre-running, externally managed |
| **Latency** | Very low (IPC) | Network-dependent |
| **Config in .mcp.json** | `command` + `args` | `url` field (or proxy command) |
| **Security** | Inherits user permissions | Needs auth tokens |
| **Sharing** | Single user | Multi-user capable |
| **Lifecycle** | Tied to Claude Code session | Independent |
| **Use case** | Dev tools, local DBs | Shared services, cloud APIs |
| **Setup complexity** | Low (just config) | Higher (deployment + auth) |

### Configuration for Remote Servers

```json
{
  "mcpServers": {
    "remote-analytics": {
      "command": "npx",
      "args": ["-y", "mcp-remote-proxy", "https://mcp.company.com/analytics"],
      "env": {
        "MCP_AUTH_TOKEN": "${MCP_ANALYTICS_TOKEN}"
      }
    }
  }
}
```

---

## 5. How Claude Code Discovers and Connects to MCP Servers

### Discovery Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP SERVER DISCOVERY FLOW                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Claude Code starts in a project directory                    │
│       │                                                           │
│       ▼                                                           │
│  2. Reads .mcp.json from project root                            │
│       │                                                           │
│       ▼                                                           │
│  3. Also reads ~/.claude/.mcp.json (user-level)                  │
│       │                                                           │
│       ▼                                                           │
│  4. Merges configurations (project takes precedence)             │
│       │                                                           │
│       ▼                                                           │
│  5. For each server: spawns process with command + args + env    │
│       │                                                           │
│       ▼                                                           │
│  6. Sends `initialize` request to each server                    │
│       │                                                           │
│       ▼                                                           │
│  7. Server responds with capabilities (tools, resources, etc.)   │
│       │                                                           │
│       ▼                                                           │
│  8. Client calls `tools/list` to enumerate available tools       │
│       │                                                           │
│       ▼                                                           │
│  9. Tools are now available for Claude to use in conversation    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Connection Lifecycle

| Phase | What Happens | Protocol Message |
|-------|-------------|-----------------|
| **Startup** | Process spawned, transport established | (OS-level) |
| **Initialize** | Client and server negotiate capabilities | `initialize` / `initialized` |
| **Discovery** | Client learns what tools are available | `tools/list` |
| **Active** | Tools are called as needed during conversation | `tools/call` |
| **Shutdown** | Session ends, server process terminated | `shutdown` (or SIGTERM) |

### What Claude Code Does Automatically

1. **Spawns processes** — you don't run the MCP server manually
2. **Manages lifecycle** — servers start/stop with your session
3. **Routes tool calls** — Claude's tool requests are routed to the right server
4. **Handles errors** — connection failures are reported gracefully
5. **Merges configs** — project + user configs are combined

### Precedence Rules

When the same server name exists in both project and user config:

```
./.mcp.json (project):     "postgres": { "command": "..." }
~/.claude/.mcp.json (user): "postgres": { "command": "..." }

Result: Project-level definition WINS (takes precedence)
```

---

## 6. Configuration Examples

### Example 1: Full-Stack Development Setup

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./src", "./docs"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### Example 2: Custom MCP Server (Local Development)

```json
{
  "mcpServers": {
    "internal-api": {
      "command": "node",
      "args": ["./tools/internal-api-server.mjs"],
      "env": {
        "API_BASE": "http://localhost:8080",
        "LOG_LEVEL": "debug"
      }
    }
  }
}
```

### Example 3: Python-Based MCP Server

```json
{
  "mcpServers": {
    "data-pipeline": {
      "command": "python",
      "args": ["-m", "my_mcp_server"],
      "env": {
        "WAREHOUSE_URL": "${SNOWFLAKE_URL}",
        "WAREHOUSE_TOKEN": "${SNOWFLAKE_TOKEN}"
      }
    }
  }
}
```

### Example 4: Docker-Based Server

```json
{
  "mcpServers": {
    "isolated-tools": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--network=host",
        "-e", "DB_URL=${DATABASE_URL}",
        "my-org/mcp-tools:latest"
      ]
    }
  }
}
```

### Example 5: Multiple Environments

```json
{
  "mcpServers": {
    "db-dev": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/myapp_dev"
      }
    },
    "db-staging": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${STAGING_DATABASE_URL}"
      }
    }
  }
}
```

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
        "GITHUB_TOKEN": "ghp_abc123realtoken456xyz"
      }
    }
  }
}
```

**Problem:** This file is version-controlled. Secrets will be committed to Git.

**Fix:** Use environment variable references:
```json
{
  "env": {
    "GITHUB_TOKEN": "${GITHUB_TOKEN}"
  }
}
```

### ❌ Anti-Pattern 2: Too Many Servers (Context Overload)

```json
{
  "mcpServers": {
    "server1": {},
    "server2": {},
    "server3": {},
    "server4": {},
    "server5": {},
    "server6": {},
    "server7": {},
    "server8": {}
  }
}
```

**Problem:** Each server registers tools that consume Claude's context window. 8 servers with 5 tools each = 40 tool descriptions always in context.

**Fix:** Only configure servers you actively need. Remove unused ones.

### ❌ Anti-Pattern 3: Missing Error Handling Configuration

```json
{
  "mcpServers": {
    "critical-db": {
      "command": "node",
      "args": ["./db-server.js"]
    }
  }
}
```

**Problem:** If the server crashes, Claude Code may hang or produce confusing errors.

**Fix:** Ensure your server implementation handles errors gracefully. Add health checks and timeouts.

### ❌ Anti-Pattern 4: Pointing to Production Databases

```json
{
  "mcpServers": {
    "production-db": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://admin:pass@prod-host:5432/production"
      }
    }
  }
}
```

**Problem:** Claude could inadvertently run destructive queries on production data.

**Fix:** Always connect to development or staging databases. Use read-only credentials if connecting to shared environments.

### ❌ Anti-Pattern 5: Duplicate Server Names

```json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgresql://localhost/dev" }
    },
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgresql://localhost/test" }
    }
  }
}
```

**Problem:** JSON keys must be unique. The second definition silently overwrites the first.

**Fix:** Use distinct names: `"db-dev"` and `"db-test"`.

### ❌ Anti-Pattern 6: Not Using .gitignore for Local Overrides

```
# If .mcp.json contains developer-specific paths:
{
  "mcpServers": {
    "filesystem": {
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/john/specific/path"]
    }
  }
}
```

**Fix:** Keep shared config in project `.mcp.json` with generic paths. Use `~/.claude/.mcp.json` for personal overrides, or use `${HOME}` variable references.

---

## 8. Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│              .mcp.json QUICK REFERENCE                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  LOCATION:    ./.mcp.json (project) or ~/.claude/.mcp.json (user)│
│  PURPOSE:     Define which MCP servers Claude Code connects to    │
│  FORMAT:      JSON with mcpServers object                        │
│  PRECEDENCE:  Project-level overrides user-level (same name)     │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  STRUCTURE:                                                        │
│                                                                    │
│  {                                                                │
│    "mcpServers": {                                                │
│      "<name>": {                                                  │
│        "command": "<executable>",                                 │
│        "args": ["<arg1>", "<arg2>"],                             │
│        "env": { "KEY": "${SYSTEM_VAR}" }                         │
│      }                                                            │
│    }                                                              │
│  }                                                                │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  TRANSPORT TYPES:                                                  │
│                                                                    │
│  stdio (local)     → command spawns child process                │
│  HTTP+SSE (remote) → connects to running server via network      │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  MCP ARCHITECTURE:                                                 │
│                                                                    │
│  Host (Claude Code) → Client (protocol) → Server (tools)        │
│  Server exposes: Tools, Resources, Prompts                       │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  SECRETS HANDLING:                                                 │
│                                                                    │
│  ✅ "TOKEN": "${ENV_VAR}"     (reference system env)             │
│  ❌ "TOKEN": "ghp_abc123..."  (hardcoded — will leak in Git)    │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  LIFECYCLE:                                                        │
│                                                                    │
│  Session start → Read config → Spawn servers → Initialize →     │
│  Discover tools → Active use → Session end → Shutdown servers    │
│                                                                    │
├──────────────────────────────────────────────────────────────────┤
│  EXAM TIPS:                                                        │
│                                                                    │
│  • .mcp.json is the ENTRY POINT for MCP in Claude Code          │
│  • Server names must be unique within a config file              │
│  • ${VAR} syntax reads from host system environment              │
│  • Project .mcp.json takes precedence over user-level            │
│  • stdio = local child process; HTTP+SSE = remote server         │
│  • Claude Code AUTOMATICALLY manages server lifecycle            │
│  • NEVER hardcode secrets — always use ${ENV_VAR} pattern        │
│  • MCP servers expose Tools + Resources + Prompts                │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Scenario Challenge

### Scenario

Your team is setting up Claude Code for a project that needs:
- Access to a PostgreSQL development database
- GitHub integration for PR management
- A custom internal API tool for querying the company wiki
- The GitHub token and database URL must not be exposed in version control

### Question

**Which .mcp.json configuration correctly satisfies ALL requirements?**

**A)**
```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://dev:password@localhost:5432/myapp"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_realtoken123"
      }
    },
    "wiki": {
      "command": "node",
      "args": ["./tools/wiki-server.js"]
    }
  }
}
```

**B)**
```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "wiki": {
      "command": "node",
      "args": ["./tools/wiki-server.js"],
      "env": {
        "WIKI_API_BASE": "${INTERNAL_WIKI_URL}"
      }
    }
  }
}
```

**C)**
```json
{
  "mcpServers": {
    "all-tools": {
      "command": "node",
      "args": ["./tools/all-in-one-server.js"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}",
        "GITHUB_TOKEN": "${GITHUB_TOKEN}",
        "WIKI_URL": "${INTERNAL_WIKI_URL}"
      }
    }
  }
}
```

**D)**
```json
{
  "servers": {
    "postgres": {
      "cmd": "npx @modelcontextprotocol/server-postgres",
      "env": { "DATABASE_URL": "${DATABASE_URL}" }
    },
    "github": {
      "cmd": "npx @modelcontextprotocol/server-github",
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

---

### Answer

**✅ Correct Answer: B**

**Explanation:**

- **Option B** correctly uses `${VAR}` references for all secrets (no hardcoding), defines three separate servers for separation of concerns, uses the correct `.mcp.json` schema (`mcpServers`, `command`, `args`, `env`), and includes the custom wiki server.
- **Option A** hardcodes the database password and GitHub token directly — these will be committed to Git (security vulnerability).
- **Option C** is architecturally problematic — combining all tools into one monolithic server violates separation of concerns and makes it harder to manage, debug, and evolve individual integrations.
- **Option D** uses incorrect schema (`servers` instead of `mcpServers`, `cmd` instead of `command`). The schema must match the MCP specification.

**Key Exam Principle:** Always use `${ENV_VAR}` for secrets in `.mcp.json`. Use separate servers for separate concerns. Use the correct schema: `mcpServers` → `command` + `args` + `env`.

---

## Summary: Key Exam Takeaways

1. `.mcp.json` is the **configuration entry point** for MCP servers in Claude Code
2. MCP architecture: Host (Claude Code) → Client (protocol) → Server (tools)
3. Servers expose three capabilities: **Tools, Resources, Prompts**
4. Server config requires: `command` (executable), optional `args` and `env`
5. **NEVER hardcode secrets** — use `${ENV_VAR}` to reference system environment
6. Local servers use **stdio** transport; remote servers use **HTTP+SSE**
7. Claude Code **automatically** manages server lifecycle (spawn, initialize, shutdown)
8. Project `.mcp.json` takes precedence over `~/.claude/.mcp.json` for same-named servers
9. Discovery flow: read config → spawn → initialize → tools/list → active use

---

*End of Day 21 Study Material*
