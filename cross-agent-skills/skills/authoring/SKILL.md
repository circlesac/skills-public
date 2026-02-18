---
name: cross-agent-skills
description: Create and manage skill packages that work across Pi, Claude Code, and OpenClaw
user-invocable: true
argument-hint: "<create|structure|publish>"
---

# Cross-Platform Skill Authoring

How to structure a skill package so it works across Pi, Claude Code, and OpenClaw with zero duplication.

## Before You Start

Check the current repo:

1. Look for `.claude-plugin/marketplace.json` — lists available plugins
2. Look for `.claude-plugin/plugin.json` — plugin manifest
3. Look for `package.json` with `pi.skills` — Pi/OpenClaw is configured
4. If none exist, this is a new setup

## The Universal Artifact

All three platforms read the same file: `SKILL.md` following the [Agent Skills](https://agentskills.io/specification) spec.

```markdown
---
name: code-review
description: Review code for best practices, security, and performance
---

When reviewing code, check for:
1. Security vulnerabilities (OWASP top 10)
2. Error handling completeness
3. Performance bottlenecks
4. Code style consistency
```

The only difference is **packaging** — how each platform discovers and installs that file.

## Package Structure

Every repo needs three things for Claude Code: `marketplace.json`, `plugin.json`, and a `skills/` directory.

A plugin can live at the repo root or in a subdirectory. Multiple plugins each get their own subdirectory.

### Plugin at repo root

When the repo IS the plugin (one plugin per repo):

```
my-tool/
├── .claude-plugin/
│   ├── marketplace.json      # indexes plugins — source: "./"
│   └── plugin.json           # plugin manifest
├── skills/
│   └── guide/
│       └── SKILL.md
├── package.json
└── src/                      # other repo files (code, etc.)
```

### Multiple plugins in one repo

When the repo hosts several independently toggleable plugins:

```
my-skills/
├── .claude-plugin/
│   └── marketplace.json      # indexes plugins — source: "./code-review", "./deploy"
├── code-review/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── review/
│           └── SKILL.md      # → /code-review:review
├── deploy/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── run/
│           ├── SKILL.md      # → /deploy:run
│           └── checklist.md
└── package.json
```

Plugin directories must be at the repo root — nested subdirectories (e.g., `plugins/code-review/`) won't resolve.

## Required Files

### `marketplace.json`

```json
{
  "name": "my-tool",
  "owner": { "name": "Your Org" },
  "plugins": [
    {
      "name": "my-tool",
      "source": "./",
      "description": "Description of the plugin"
    }
  ]
}
```

Use `"source": "./"` when the plugin lives at the repo root. Use `"source": "./plugin-name"` for subdirectories. Each entry in `plugins` is independently installable and toggleable.

### `plugin.json`

```json
{
  "name": "my-tool",
  "description": "Description of the plugin",
  "author": { "name": "Your Org" }
}
```

One per plugin. Lives in `.claude-plugin/plugin.json` within each plugin directory.

### `package.json` (Pi / OpenClaw)

```json
{
  "name": "@org/my-tool",
  "files": ["skills"],
  "pi": { "skills": ["./skills"] }
}
```

Pi recursively scans each directory listed in `pi.skills` for `SKILL.md` files. The `files` field controls what gets published to npm — include `skills` so Pi can discover them.

For repos that also ship code (CLI tools, libraries), include both:

```json
{
  "name": "@org/my-tool",
  "bin": { "my-tool": "bin/my-tool" },
  "files": ["bin", "skills"],
  "pi": { "skills": ["./skills"] }
}
```

For multiple plugins, list each directory:

```json
{
  "files": ["code-review", "deploy"],
  "pi": { "skills": ["./code-review", "./deploy"] }
}
```

## Install

```bash
# Claude Code (two-step: add marketplace, then install plugin)
/plugin marketplace add org/my-tool
/plugin install my-tool

# Pi
pi install npm:@org/my-tool

# OpenClaw
clawhub install my-tool
```

## How Each Platform Finds Skills

| Platform | Manifest | Discovery |
|----------|----------|-----------|
| **Claude Code** | `marketplace.json` → `plugin.json` | Skills in `skills/` subdir within each plugin |
| **Pi** | `package.json` → `pi.skills` | Recursively scans listed directories for `SKILL.md` |
| **OpenClaw** | — | ClawHub registry or manual placement |

## SKILL.md Frontmatter

The Agent Skills spec defines shared frontmatter. Some fields are platform-specific:

```markdown
---
# Universal (all platforms)
name: code-review
description: Review code for best practices and security

# Pi / OpenClaw only
license: MIT
compatibility: ">=1.0.0"

# Claude Code only
user-invocable: true
model: claude-opus-4-6
context: fork
argument-hint: "<file-or-PR-url>"
---
```

Platforms ignore frontmatter keys they don't recognize, so including all of them is safe.

## Slash Command Invocation

Skills double as slash commands on all platforms:

| Platform | Invocation | Notes |
|----------|-----------|-------|
| **Pi** | `/skill:code-review [args]` | All skills invocable by default |
| **OpenClaw** | `/skill:code-review [args]` | Same as Pi |
| **Claude Code** | `/code-review:review [args]` | Requires `user-invocable: true` in frontmatter |

Claude Code requires `user-invocable: true` opt-in. The `disable-model-invocation: true` frontmatter (Pi/OpenClaw) does the inverse — hides from auto-discovery.

## Development Workflow

```bash
# 1. Create skill
mkdir -p skills/review
cat > skills/review/SKILL.md << 'EOF'
---
name: code-review
description: Review code for best practices and security
user-invocable: true
---

When reviewing code...
EOF

# 2. Test locally
pi --skill ./skills/review              # Pi
claude --plugin-dir .                    # Claude Code

# 3. Publish
npm publish                             # → Pi (npm)
clawhub publish                         # → OpenClaw
git push                                # → Claude Code (git)
```

## Supporting Files

Skills can include supporting docs alongside SKILL.md:

```
skills/run/
├── SKILL.md              # main skill (always loaded)
├── checklist.md          # reference doc (loaded on-demand by agent)
└── scripts/
    └── validate.sh       # executable (Claude Code only — shell expansion)
```

## README Install Instructions

Add a **Skills** section to your repo's README so users know how to install:

```markdown
## Skills

| Skill | Description |
|-------|-------------|
| **skill-name** | Short description |

### Claude Code

\```bash
# Add marketplace
/plugin marketplace add org/repo-name

# Install plugin
/plugin install plugin-name
\```

### Pi

\```bash
pi install npm:@org/package-name
\```
```

Use the **skill name** from SKILL.md frontmatter in the table, the **plugin name** from `plugin.json` in the install command, and the **npm package name** from `package.json` for Pi.

## Summary

1. **Write SKILL.md once** — follows Agent Skills spec, works everywhere
2. **Always provide `marketplace.json` + `plugin.json`** — Claude Code needs both. Use `"source": "./"` for repo-root plugins.
3. **Add `package.json` with `pi.skills`** for Pi/OpenClaw
4. **Add install instructions to README** — skill table + Claude Code / Pi install commands
5. **Publish to npm + ClawHub + git** — npm (Pi), ClawHub (OpenClaw), git (Claude Code)
