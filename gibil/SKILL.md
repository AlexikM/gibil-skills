---
name: gibil
description: >
  Run builds, tests, and commands on disposable remote servers instead of locally.
  Use when the user wants to offload compute, test on a clean environment, run Docker
  services, or give an AI agent its own machine. Supports CLI mode (gibil create/run/destroy)
  and MCP mode (vm_bash, vm_read, vm_write, vm_ls, vm_grep).
compatibility: Requires Node.js 20+ and a Hetzner Cloud API token
metadata:
  author: AlexikM
  version: "0.1.3"
  homepage: https://gibil.dev
  repository: https://github.com/AlexikM/gibil-skills
---

# Gibil — Ephemeral Remote Servers

Gibil creates disposable Linux servers with root access, Docker, and SSH. Forge a server in 90 seconds, run your workload, burn it when done.

## Install and setup

```bash
npm install -g gibil
gibil init  # one-time: saves your Hetzner API token
```

## CLI mode

The primary interface. Every command supports `--json` for structured output.

### Forge a server

```bash
gibil create --name my-task --repo https://github.com/user/project --ttl 30 --json
```

The server boots with Ubuntu 24.04, your repo cloned to `/root/project`, and your chosen runtime installed (Node.js 20 by default). The `--ttl 30` auto-destroys it after 30 minutes.

Output:

```json
{
  "name": "my-task",
  "ip": "65.21.x.x",
  "status": "running",
  "ttl_minutes": 30
}
```

### Run commands

```bash
gibil run my-task "cd /root/project && pnpm install && pnpm test" --json
```

Output:

```json
{
  "stdout": "42 tests passed",
  "stderr": "",
  "exit_code": 0
}
```

Always use `--json` when automating. Parse `exit_code` to determine success or failure.

### Burn the server

```bash
gibil destroy my-task --json
```

### SSH in for interactive debugging

```bash
gibil ssh my-task
```

### Parallel servers with fleet mode

```bash
gibil create --name shard --fleet 5 --repo https://github.com/user/project --ttl 30 --json
gibil run shard-1-abc "cd /root/project && pnpm test -- --shard=1/5" --json
gibil run shard-2-abc "cd /root/project && pnpm test -- --shard=2/5" --json
gibil destroy --all
```

### All flags

| Flag | Purpose | Example |
|---|---|---|
| `--name` | Server name | `--name my-task` |
| `--repo` | Clone a repo on boot | `--repo github.com/user/project` |
| `--ttl` | Auto-destroy after N minutes (default: 60) | `--ttl 30` |
| `--json` | Machine-readable output | Always use for automation |
| `--fleet` | Create N servers in parallel (max 20) | `--fleet 5` |
| `--server-type` | Server size | `--server-type cpx41` (8 vCPU) |
| `--location` | Datacenter | `--location fsn1` (default) |

## MCP mode

For Claude Code and other MCP-compatible agents. Instead of shelling out to the CLI, the agent gets typed tools over a persistent connection.

### Setup

1. Create a server first: `gibil create --name my-task --repo ... --ttl 60`
2. Add to MCP config:

```json
{
  "mcpServers": {
    "gibil": {
      "command": "gibil",
      "args": ["mcp", "my-task"]
    }
  }
}
```

### Available tools

| Tool | What it does |
|---|---|
| `vm_bash` | Run any shell command. Returns `{ stdout, stderr, exit_code }` |
| `vm_read` | Read a file with optional line offset and limit |
| `vm_write` | Create or overwrite a file |
| `vm_ls` | List directory contents with optional glob filter |
| `vm_grep` | Search file contents with regex |

### Edit-test loop pattern

1. `vm_read({ path: "/root/project/src/auth.ts" })` — read the file
2. `vm_write({ path: "/root/project/src/auth.ts", content: "..." })` — write the fix
3. `vm_bash({ command: "cd /root/project && pnpm test" })` — run tests
4. If tests fail, read output, adjust, repeat from step 2
5. `vm_bash({ command: "cd /root/project && git add -A && git commit -m 'fix' && git push" })` — commit

### Important details

- The repo is cloned to `/root/project`. Always reference that path.
- `vm_bash` defaults to 30 second timeout. Pass `timeout_ms` for longer operations.
- `vm_write` replaces the entire file. Read first, modify, write back.
- The server is disposable. If something breaks, the user can destroy and recreate.
- Create and destroy are still CLI commands. MCP is only for working on an existing server.

## When to use gibil

**Good fit:**
- Running the full test suite on a clean environment
- Heavy builds that would slow down the local machine
- Tasks that need Docker services (Postgres, Redis) without local Docker
- Giving an AI agent root access safely on a disposable server
- Parallel test sharding across multiple servers
- Reproducing bugs on a fresh environment

**Not the right fit:**
- Quick local file edits or reads (do it locally)
- Tasks under 30 seconds (server boot takes ~90s)
- Modifying the user's local git working tree

## More information

- Docs: https://gibil.dev/docs
- CLI reference: https://gibil.dev/docs/cli/create
- Recipes: https://gibil.dev/docs/recipes/code-test-loop
