# @circlesac/skills

Meta-skills for building CLI tools and packaging them as AI skills.

Started with [mcp-docs-server](https://github.com/circlesac/mcp-docs-server) — bundled docs as MCP prompts so agents could use them as slash commands. Two problems: Claude Code didn't reliably surface MCP prompts as slash commands, and prompts only run when you explicitly invoke them. ACL was on the roadmap but auth is always painful to build.

Skills fix all of this. A `SKILL.md` file works as both a slash command and ambient context the agent picks up on its own. Deployed via npm or git, so access control is just repo visibility — no auth layer needed. Compare this to the manual symlink setup in [pi-skills](https://github.com/badlogic/pi-skills) — `/cross-agent-skills` handles all the packaging boilerplate so you just write a `SKILL.md` and ship it.

## Available Skills

| Skill | Description |
|-------|-------------|
| `/cross-agent-skills` | Author a skill package that works on Claude Code, Pi, and OpenClaw |
| `/cross-platform-cli` | Scaffold and distribute a compiled CLI across npm, Homebrew, and GitHub Releases |

## CLI tools + skills

CLI tools that ship with skills — describe the interface once in a `SKILL.md`, the agent knows it forever.

| CLI | Skill | Does | Install |
|-----|-------|------|---------|
| [`holla`](https://github.com/circlesac/holla-cli) | `/slack` | Slack — messages, threads, search, canvases | `circlesac/holla-cli` |
| [`notas`](https://github.com/circlesac/notas-cli) | `/notion` | Notion — pages, databases, blocks, comments | `circlesac/notas-cli` |
| [`oneup`](https://github.com/circlesac/oneup) | `/oneup` | CalVer version management | `circlesac/oneup` |
| [`sandbox`](https://github.com/circlesac/sandbox) | `/sandbox-guide` | E2B-compatible local sandbox with Docker | `circlesac/sandbox` |

Each repo ships both a binary and a skill. The agent calls the CLI directly — no separate API needed.

## Installation

### Claude Code

```bash
# Add marketplace
/plugin marketplace add circlesac/skills

# Install plugins
/plugin install cross-agent-skills
/plugin install cross-platform-cli
```

### Pi

```bash
pi install git:circlesac/skills
# or: npx @mariozechner/pi-coding-agent install git:circlesac/skills
```

## License

MIT
