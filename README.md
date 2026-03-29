<p align="center">
  <img src="https://i.imgur.com/l5uHTnh.png" width="120" alt="Gibil" />
</p>

<h1 align="center">Gibil Agent Skills</h1>

<p align="center">
  Teach your AI agent how to use <a href="https://gibil.dev">gibil</a> — ephemeral remote servers for builds, tests, and deployments.
</p>

---

## Available Skills

### `gibil`

Teaches Claude Code, Cursor, Copilot, and other [Agent Skills](https://agentskills.io)-compatible agents how to forge disposable servers, run commands remotely, and use MCP tools for iterative development.

**What the skill covers:**
- When to offload work to a remote server vs. running locally
- The forge → run → burn lifecycle with `--json` for structured output
- MCP tool patterns (`vm_bash`, `vm_read`, `vm_write`, `vm_ls`, `vm_grep`)
- Fleet mode for parallel test sharding
- How to parse exit codes and handle failures

## Install

### Claude Code

```bash
npx @anthropic-ai/claude-code skills add github:AlexikM/gibil-skills/gibil
```

Or add to your `.claude/settings.json`:

```json
{
  "skills": ["github:AlexikM/gibil-skills/gibil"]
}
```

### Other agents

Any agent that supports the [Agent Skills](https://agentskills.io) standard can use this skill. Point it at `github:AlexikM/gibil-skills/gibil` or copy the `gibil/SKILL.md` file into your agent's skill directory.

## Prerequisites

The skill teaches your agent how to use gibil, but you still need gibil installed:

```bash
npm install -g gibil
gibil init
```

## Links

- [gibil.dev](https://gibil.dev) — docs, recipes, blog
- [npm package](https://www.npmjs.com/package/gibil)
- [Claude Code guide](https://gibil.dev/docs/guides/claude-code)

---

<p align="center"><sub>Named after the Sumerian god of fire. 𒀭𒉈𒄀</sub></p>
