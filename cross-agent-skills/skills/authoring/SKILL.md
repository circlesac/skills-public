---
name: cross-agent-skills
description: Create and manage skill packages that work across Pi, Claude Code, and OpenClaw
user-invocable: true
argument-hint: "<create|structure|publish>"
---

# Cross-Platform Skill Authoring

How to structure a skill package so it works across Pi, Claude Code, and OpenClaw with zero duplication.

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

One repo, multiple independently toggleable plugins via marketplace. Each plugin lives at the repo root. Claude Code requires skills inside a `skills/` subdirectory within each plugin:

```
my-plugins-repo/
├── package.json              # Pi / OpenClaw manifest (npm)
├── .claude-plugin/
│   └── marketplace.json      # Claude Code marketplace (indexes all plugins)
├── code-review/
│   ├── .claude-plugin/
│   │   └── plugin.json       # independent Claude Code plugin
│   └── skills/
│       └── review/
│           └── SKILL.md      # → /code-review:review
├── deploy/
│   ├── .claude-plugin/
│   │   └── plugin.json       # independent Claude Code plugin
│   └── skills/
│       └── run/
│           ├── SKILL.md      # → /deploy:run
│           └── checklist.md
├── LICENSE
└── README.md
```

Plugin directories must be at the repo root — Claude Code's marketplace does not support nested subdirectories (e.g., `plugins/code-review/` won't resolve).

### `package.json` (Pi / OpenClaw)

```json
{
  "name": "@org/skills-devops",
  "version": "1.0.0",
  "description": "Code review and deployment skills",
  "files": ["code-review", "deploy"],
  "pi": {
    "skills": ["./code-review", "./deploy"]
  },
  "license": "MIT"
}
```

**Security best practice**: Explicitly list each plugin directory rather than a parent directory that contains multiple plugins. Pi recursively scans each listed directory for `SKILL.md` files. This approach gives you control over which plugin directories are included without exposing unintended plugins or accidentally scanning parent directories.

### `.claude-plugin/marketplace.json` (Claude Code)

```json
{
  "name": "skills-devops",
  "owner": { "name": "Your Org" },
  "plugins": [
    {
      "name": "code-review",
      "source": "./code-review",
      "description": "Review code for best practices and security"
    },
    {
      "name": "deploy",
      "source": "./deploy",
      "description": "Deployment automation and checklists"
    }
  ]
}
```

Each plugin folder has its own `plugin.json`:

### `code-review/.claude-plugin/plugin.json`

```json
{
  "name": "code-review",
  "version": "1.0.0",
  "description": "Review code for best practices and security",
  "author": { "name": "Your Org" }
}
```

The marketplace is just an index — each plugin is independently installable and toggleable. The `source` must be a relative path starting with `./` pointing to a directory at the repo root.

Claude Code discovers skills from the `skills/` subdirectory within each plugin. The folder name becomes the skill name, prefixed with the plugin namespace (e.g., `review/` in a plugin named `code-review` creates `/code-review:review`).

## How Each Platform Finds Skills

| Platform | Manifest | Install | Discovery |
|----------|----------|---------|-----------|
| **Pi** | `package.json` → `pi.skills` | `pi install npm:@org/skills-devops` | Explicitly list each plugin directory; Pi recursively scans each for `SKILL.md` files |
| **OpenClaw** | — | `clawhub install skills-devops` | Requires publishing to ClawHub registry; or manually place skill folder in `skills/` |
| **Claude Code** | `.claude-plugin/marketplace.json` | `/plugin install code-review@skills-devops` | Each skill is a separate, toggleable plugin; skills in `skills/` subdir |

## Install Commands

```bash
# Pi
pi install npm:@org/skills-devops

# OpenClaw
clawhub install skills-devops

# Claude Code (add marketplace once, then install individual plugins)
/plugin marketplace add org/my-plugins-repo
/plugin install code-review
/plugin install deploy
```

## SKILL.md Frontmatter Compatibility

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

Skills double as slash commands on all platforms — no need for a separate `commands/` directory:

| Platform | Invocation | Notes |
|----------|-----------|-------|
| **Pi** | `/skill:code-review [args]` | All skills are always invocable as slash commands |
| **OpenClaw** | `/skill:code-review [args]` | Same as Pi |
| **Claude Code** | `/code-review:review [args]` | Requires `user-invocable: true` in frontmatter |

Pi and OpenClaw make all skills user-invocable by default. Claude Code requires the explicit opt-in. The `disable-model-invocation: true` frontmatter (Pi/OpenClaw) does the inverse — hides a skill from the model's auto-discovery so only the user can trigger it.

## Single Skill vs Multi-Skill Repo

**Single skill** — one SKILL.md, one purpose, one plugin:

```
code-review/
├── package.json
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── review/
        └── SKILL.md
```

**Multi-skill repo** — several skills in one repo, each as an independent plugin at the repo root:

```
skills-devops/
├── package.json
├── .claude-plugin/
│   └── marketplace.json          # indexes all plugins
├── code-review/
│   ├── .claude-plugin/
│   │   └── plugin.json           # independent plugin
│   └── skills/
│       └── review/
│           └── SKILL.md
├── deploy/
│   ├── .claude-plugin/
│   │   └── plugin.json           # independent plugin
│   └── skills/
│       └── run/
│           ├── SKILL.md
│           └── checklist.md
└── incident-response/
    ├── .claude-plugin/
    │   └── plugin.json           # independent plugin
    └── skills/
        └── respond/
            └── SKILL.md
```

In Claude Code, skills within a single plugin can't be individually toggled. Using a marketplace with separate plugins per skill gives users granular control over which skills they enable.

## Development Workflow

```bash
# 1. Create skill
mkdir -p code-review/.claude-plugin
mkdir -p code-review/skills/review
cat > code-review/skills/review/SKILL.md << 'EOF'
---
name: code-review
description: Review code for best practices and security
user-invocable: true
---

When reviewing code...
EOF

# 2. Test locally with each platform
pi --skill ./code-review/skills/review    # Pi
claude --plugin-dir ./code-review          # Claude Code

# 3. Publish
npm publish                               # → Pi (npm)
clawhub publish                           # → OpenClaw (ClawHub registry)
git push                                  # → Claude Code (git)
```

## Supporting Files

Skills can include supporting docs alongside SKILL.md:

```
deploy/skills/run/
├── SKILL.md              # main skill (always loaded)
├── checklist.md          # reference doc (loaded on-demand by agent)
└── scripts/
    └── validate.sh       # executable (Claude Code only — shell expansion)
```

- **Pi**: agent reads supporting files via `read` tool when needed
- **Claude Code**: supports shell expansion syntax in SKILL.md to inline script output
- **OpenClaw**: same as Pi

## Summary

1. **Write SKILL.md once** — follows Agent Skills spec, works everywhere
2. **Add both manifests** — `package.json` (Pi/OpenClaw) + `.claude-plugin/marketplace.json` (Claude Code)
3. **Plugins at repo root** — Claude Code marketplace resolves plugin directories from the repo root only; Pi recursively scans and finds SKILL.md at any depth
4. **One plugin per skill** — use marketplace to keep each skill independently toggleable in Claude Code
5. **Publish to npm + ClawHub + git** — npm (Pi), ClawHub (OpenClaw), git (Claude Code)
