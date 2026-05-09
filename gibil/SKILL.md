---
name: gibil
description: >
  Forge, use, and burn disposable remote servers. Use when the user wants to
  run builds, tests, or commands on a clean environment instead of locally.
  Full lifecycle via MCP: create_server, vm_bash, vm_read, vm_write, destroy_server.
  Multi-provider — Hetzner (EU/US, cheapest) and Vultr (APAC: Tokyo, Seoul, Singapore, Sydney, Mumbai).
compatibility: Requires Node.js 20+ and an API token from Hetzner or Vultr
metadata:
  author: AlexikM
  version: "0.5.0"
  homepage: https://gibil.dev
  repository: https://github.com/AlexikM/gibil-skills
---

# Gibil — Your Own Machine, On Demand

Gibil forges your own machine on demand — own kernel, full root, Docker-in-Docker, SSH, real systemd, on the cloud you choose. Forge a box, do your work, burn it when done. Set the TTL anywhere from 15 minutes to 30 days; extend if the work runs long. Ubuntu Linux on either Hetzner Cloud (EU/US baseline) or Vultr (best APAC coverage); the architecture lets additional clouds plug in without changing the surface.

## Setup

```bash
npm install -g gibil
gibil init                    # interactive: pick hetzner or vultr
# or non-interactive:
gibil init --provider vultr --token <key>
gibil init --add hetzner      # add a second provider later
```

`gibil init` configures your provider token(s), installs the agent skill, and registers the MCP server. The picker shows hetzner (default — EU/US, cheapest) and vultr (APAC density). Run `gibil providers` to see configured providers, regions, and the size table. For Claude Code, init merges the gibil entry into `~/.claude.json`, preserving any other `mcpServers` you have. For other MCP-compatible agents, run `gibil mcp --print-config`. After init, restart your agent — the MCP tools are immediately available.

## MCP Tools

### Lifecycle

| Tool | What it does | Params |
|---|---|---|
| `create_server` | Forge a new server. Returns name and IP when ready. | `name?`, `repo?`, `ttl?` (e.g. `"2h"`, `"7d"`, `"1mo"`), `provider?` (`hetzner`/`vultr`), `size?` (`small`/`medium`/`large`), `server_type?`, `location?`, `env?` |
| `destroy_server` | Burn a server by name. | `name` |
| `list_servers` | List all active servers with IPs and remaining TTL. | — |
| `extend_server` | Extend a server's auto-destroy timer. | `name`, `ttl` (e.g. `"7d"`, `"1mo"`, `"1y"`) |

### Working on a server

| Tool | What it does | Params |
|---|---|---|
| `vm_bash` | Run any shell command. Set `background: true` for long ops. | `command`, `working_dir?`, `timeout_ms?`, `background?`, `server?` |
| `vm_read` | Read a file with optional line offset and limit. | `path`, `offset?`, `limit?`, `server?` |
| `vm_write` | Create or overwrite a file. | `path`, `content`, `server?` |
| `vm_ls` | List directory contents with optional glob filter. | `path?`, `glob?`, `server?` |
| `vm_grep` | Search file contents with regex. | `pattern`, `path?`, `include?`, `server?` |
| `vm_stats` | Get server resource usage (CPU cores, load, memory, disk, uptime). | `server?` |

### Background jobs

| Tool | What it does | Params |
|---|---|---|
| `vm_job_status` | Poll a background job for completion. | `job_id` |
| `vm_job_list` | List all background jobs across servers. | — |
| `vm_sweep_orphans` | Mark running jobs as orphaned if their server no longer exists. Use after `destroy_server` to clean up lingering job records. | — |

All vm_* tools accept an optional `server` parameter. If only one server is running, it auto-selects.

### Parameter details

- `working_dir` — defaults to `/root/project`. Set to `/root` if no repo was cloned.
- `timeout_ms` — defaults to 120000 (120s). Increase for long builds: `timeout_ms: 300000`. Or use `background: true` instead.
- `background` — set to `true` on `vm_bash` to run in background. Returns a `job_id` you poll with `vm_job_status`.
- `glob` — shell glob for `vm_ls`, e.g. `"**/*.ts"` to find all TypeScript files.
- `include` — file glob for `vm_grep`, e.g. `"*.ts"` to search only TypeScript files.
- `provider` — `hetzner` or `vultr`. Defaults to the configured default. Pick `vultr` for APAC users.
- `size` — `small` (2 vCPU/4 GB), `medium` (4 vCPU/8 GB), `large` (8 vCPU/16 GB). Resolves to the right SKU per provider. Run `gibil providers` to see actual specs.
- `server_type` — Provider-native SKU. Overrides `size`. Hetzner: `cax11`/`cax21`/`cax31` (ARM, fsn1/nbg1 only) or `cpx21`/`cpx31` (x86, all locations). Vultr: `vc2-2c-4gb`/`vc2-4c-8gb`.
- `location` — Provider region. Hetzner: `fsn1`/`nbg1`/`ash`. Vultr: `nrt` (Tokyo)/`sgp` (Singapore)/`syd` (Sydney)/`icn` (Seoul)/`bom` (Mumbai).
- `env` — key-value pairs exported on the server and persisted to `/etc/environment`.

## Workflow

### Forge a server and run tests

```
create_server({ name: "test", repo: "https://github.com/user/project", ttl: "2h" })
→ { name: "test", ip: "65.21.x.x", status: "running" }

vm_bash({ command: "pnpm install && pnpm test" })
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

vm_bash({ command: "pnpm test src/auth.test.ts" })
→ test output
```

Repeat until tests pass, then commit:

```
vm_bash({ command: "git add -A && git commit -m 'fix' && git push" })
```

### Search the codebase

```
vm_grep({ pattern: "TODO", include: "*.ts" })
→ matching lines with file paths and line numbers

vm_ls({ glob: "**/*.test.ts" })
→ list of test files
```

### Pick a provider and region (e.g. APAC)

```
create_server({
  name: "tokyo-test",
  repo: "https://github.com/user/project",
  provider: "vultr",
  location: "nrt",
  size: "medium",
  ttl: "2h",
})
→ { name: "tokyo-test", ip: "...", status: "running" }
```

### Pass secrets

```
create_server({ name: "task", repo: "https://github.com/org/private-repo", env: { "GITHUB_TOKEN": "ghp_xxx" } })
```

### Long-running builds (background)

```
vm_bash({ command: "pnpm build && pnpm test", background: true })
→ { job_id: "j-a3f1b2c8", status: "running" }

// ... do other work, then poll ...

vm_job_status({ job_id: "j-a3f1b2c8" })
→ { status: "done", exit_code: 0, stdout: "...", duration_s: 142 }
```

### Parallel test sharding across fleet

```
create_server({ name: "shard-1", repo: "...", ttl: "1h" })
create_server({ name: "shard-2", repo: "...", ttl: "1h" })

vm_bash({ command: "pnpm test -- --shard=1/2", background: true, server: "shard-1" })
→ { job_id: "j-aaa" }
vm_bash({ command: "pnpm test -- --shard=2/2", background: true, server: "shard-2" })
→ { job_id: "j-bbb" }

// Poll both
vm_job_status({ job_id: "j-aaa" })
vm_job_status({ job_id: "j-bbb" })
```

## Key details

- The repo is cloned to `/root/project`. `vm_bash` defaults to that directory.
- `vm_bash` defaults to 30 second timeout. Use `background: true` for long operations, or pass `timeout_ms` to increase the limit.
- `vm_write` replaces the entire file. Read first, modify, write back.
- Boxes are disposable by default. If something breaks, destroy and forge a new one. If you need it longer, `extend_server`.
- `create_server` waits until the server is SSH-ready before returning.

## When to use gibil

**Good fit:**
- Running the full test suite on a clean machine
- Heavy builds that would slow down the local machine
- Tasks needing Docker services without local Docker
- Giving an agent root access safely on a disposable box
- Reproducing bugs on a fresh environment
- Long-lived dev sessions or staging boxes (set TTL to days, extend as needed)

**Not the right fit:**
- Quick local file reads or edits (do it locally)
- Tasks under 30 seconds (box boot takes ~90s)
- Modifying the user's local git working tree

## More information

- Docs: https://gibil.dev/docs
- CLI reference: https://gibil.dev/docs/cli/create
- Recipes: https://gibil.dev/docs/recipes/code-test-loop
