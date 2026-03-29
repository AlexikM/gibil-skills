---
name: gibil
description: >
  Forge, use, and burn disposable remote servers. Use when the user wants to
  run builds, tests, or commands on a clean environment instead of locally.
  Full lifecycle via MCP: create_server, vm_bash, vm_read, vm_write, destroy_server.
compatibility: Requires Node.js 20+ and a Hetzner Cloud API token
metadata:
  author: AlexikM
  version: "0.2.0"
  homepage: https://gibil.dev
  repository: https://github.com/AlexikM/gibil-skills
---

# Gibil — Ephemeral Remote Servers

Gibil gives you disposable Linux servers with root access, Docker, and SSH. Forge a server, do your work, burn it when done.

## Setup

```bash
npm install -g gibil
gibil init
```

MCP config (for Claude Code, Cursor, and other agents):

```json
{
  "mcpServers": {
    "gibil": {
      "command": "gibil",
      "args": ["mcp"]
    }
  }
}
```

## MCP Tools

### Lifecycle

| Tool | What it does |
|---|---|
| `create_server` | Forge a new server. Returns name and IP when ready. |
| `destroy_server` | Burn a server by name. |
| `list_servers` | List all active servers with IPs and remaining TTL. |
| `extend_server` | Extend a server's auto-destroy timer. |

### Working on a server

| Tool | What it does |
|---|---|
| `vm_bash` | Run any shell command. Returns stdout, stderr, exit code. |
| `vm_read` | Read a file with optional line offset and limit. |
| `vm_write` | Create or overwrite a file. |
| `vm_ls` | List directory contents with optional glob filter. |
| `vm_grep` | Search file contents with regex. |

All vm_* tools accept an optional `server` parameter. If only one server is running, it auto-selects.

## Workflow

### Forge a server and run tests

```
create_server({ name: "test", repo: "https://github.com/user/project", ttl: 30 })
→ { name: "test", ip: "65.21.x.x", status: "running" }

vm_bash({ command: "cd /root/project && pnpm install && pnpm test" })
→ stdout with test results, exit code 0 or 1

destroy_server({ name: "test" })
→ Server "test" destroyed.
```

### Edit-test loop

```
vm_read({ path: "/root/project/src/auth.ts" })
→ file contents with line numbers

vm_write({ path: "/root/project/src/auth.ts", content: "..." })
→ Wrote /root/project/src/auth.ts

vm_bash({ command: "cd /root/project && pnpm test src/auth.test.ts" })
→ test output
```

Repeat until tests pass, then commit:

```
vm_bash({ command: "cd /root/project && git add -A && git commit -m 'fix' && git push" })
```

### Pass secrets

```
create_server({ name: "task", repo: "https://github.com/org/private-repo", env: { "GITHUB_TOKEN": "ghp_xxx" } })
```

Environment variables are exported on the server and persisted to `.bashrc`.

## Key details

- The repo is cloned to `/root/project`. Always reference that path.
- `vm_bash` defaults to 30 second timeout. Pass `timeout_ms` for longer operations.
- `vm_write` replaces the entire file. Read first, modify, write back.
- Servers are disposable. If something breaks, destroy and create a new one.
- `create_server` waits until the server is SSH-ready before returning.

## When to use gibil

**Good fit:**
- Running the full test suite on a clean environment
- Heavy builds that would slow down the local machine
- Tasks needing Docker services without local Docker
- Giving an agent root access safely on a disposable server
- Reproducing bugs on a fresh environment

**Not the right fit:**
- Quick local file reads or edits (do it locally)
- Tasks under 30 seconds (server boot takes ~90s)
- Modifying the user's local git working tree

## More information

- Docs: https://gibil.dev/docs
- CLI reference: https://gibil.dev/docs/cli/create
- Recipes: https://gibil.dev/docs/recipes/code-test-loop
