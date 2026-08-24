# Skill Manager

Centralized skill management for AI coding tools. All skills live in one place — the **skill store** at `~/.skills/` — and projects link to them via symlinks. No duplication, always in sync.

Works with Claude Code, OpenAI Codex, and any tool that reads skills from a directory.

## What's Included

| Skill | Description |
|-------|-------------|
| **skill-store** | Manage the central repository (`~/.skills/`). Install from GitHub, migrate local skills, uninstall, list, browse. |
| **link-manager** | Connect skills from the store to your projects. Link, unlink, and view project-level bindings. |

## How It Works

```
~/.skills/                              ← central store (git repo)
├── management/
│   ├── skill-store/                    ← this skill
│   └── link-manager/                   ← companion skill
├── <category>/
│   └── <skill-name>/
│       └── SKILL.md
└── .skill-meta.json

~/.claude/skills/                       ← project reads from here
├── skill-store  → ~/.skills/management/skill-store   (symlink)
└── link-manager → ~/.skills/management/link-manager  (symlink)
```

## Installation

### Quick Start (3 commands)

```bash
# 1. Clone this repo to the central store
git clone <your-repo-url> ~/.skills

# 2. Link management skills to your AI tool
ln -s ~/.skills/management/skill-store ~/.claude/skills/skill-store
ln -s ~/.skills/management/link-manager ~/.claude/skills/link-manager

# 3. Done. Use skill-store to install more skills.
```

### For Other AI Tools

Replace `.claude` with the tool's skill directory:

```bash
# OpenAI Codex / .agent platform
ln -s ~/.skills/management/skill-store ~/.agent/skills/skill-store
ln -s ~/.skills/management/link-manager ~/.agent/skills/link-manager
```

### Windows Notes

On Windows (Git Bash), `ln -s` silently copies instead of creating symlinks. You must set the MSYS environment variable first:

```bash
export MSYS="winsymlinks:native"
ln -s ~/.skills/management/skill-store ~/.claude/skills/skill-store
ln -s ~/.skills/management/link-manager ~/.claude/skills/link-manager
```

**Prerequisites:** Developer Mode must be enabled (Settings → Developer Options → Developer Mode).

## Usage

### Install a skill from GitHub

```
User: install mattpocock/skills/grill-me from GitHub

skill-store:
1. Clones the repo
2. Reads SKILL.md, recommends category: "productivity"
3. Copies to ~/.skills/productivity/grill-me/
4. Git commits
5. Suggests: run `link-manager link grill-me` to use it
```

### Migrate a local skill to the store

```
User: migrate .claude/skills/build-your-own-x to the store

skill-store:
1. Reads SKILL.md, recommends category: "learning"
2. Moves to ~/.skills/learning/build-your-own-x/
3. Creates symlink at original location
4. Git commits
```

### Link a skill to a project

```
User: link grill-me to this project

link-manager:
1. Found in store: ~/.skills/productivity/grill-me/
2. Detected platform: .claude
3. Creates symlink .claude/skills/grill-me → ~/.skills/productivity/grill-me/
```

### Browse and unlink

```
User: list all skills in the store
User: what skills are linked in this project?
User: unlink grill-me from this project
```

## Cross-Platform Support

| Platform | Symlink Method | Notes |
|----------|---------------|-------|
| Linux | `ln -s` | Works natively |
| macOS | `ln -s` | Works natively. `readlink -f` not available, use `greadlink` if needed. |
| Windows (Git Bash) | `MSYS="winsymlinks:native" ln -s` | Requires Developer Mode. Without MSYS setting, `ln -s` silently copies. |
| Windows (WSL) | `ln -s` | Works natively within WSL |

## Directory Structure

```
skill-manager/
├── README.md
├── .gitignore
├── skill-store/
│   └── SKILL.md          ← central store management
└── link-manager/
    └── SKILL.md          ← project binding management
```
