---
name: packs
version: 1.0.0
description: |
  Use the packs CLI to discover, install, and share AI agent skills.
  The package manager for agent skills - like npm but for AI.
allowed-tools:
  - exec
  - Read
  - Write
---

# packs — Agent Skill Package Manager

packs is a CLI for discovering, installing, and sharing AI agent skills.

## Installation

```bash
brew install tunajam/tap/packs
```

## Core Commands

### Browse skills interactively
```bash
packs
```
Launches the TUI browser to explore available skills.

### Search for skills
```bash
packs find <query>
```
Search the registry for skills matching your query.

### Install a skill
```bash
packs get <skill-name>
```
Downloads and outputs the skill content. Pipe to a file or use `--install` to save to disk.

**Examples:**
```bash
# Output to stdout (for piping to agent context)
packs get humanizer

# Install to ./skills/ directory
packs get humanizer --install

# Install to specific directory
packs get humanizer --install --output ~/my-agent/skills/
```

### Direct GitHub fetch
```bash
packs get gh:user/repo/path/to/skill
# or shorthand
packs get @user/repo/path/to/skill
```
Fetch any skill directly from GitHub without it being in the registry.

**Examples:**
```bash
packs get gh:tunajam/packs-registry/skills/humanizer
packs get @anthropics/claude-code-skills/debugging
```

### Get skill info
```bash
packs info <skill-name>
```
View metadata, version, description, and dependencies.

### Submit a skill
```bash
packs submit @user/repo/path/to/skill
```
Submit a skill from your GitHub repo to the registry.

## Version Pinning

Install a specific version:
```bash
packs get humanizer@2.1.0
```

## Flags

| Flag | Description |
|------|-------------|
| `--install`, `-i` | Save skill to disk (default: ./skills/) |
| `--output`, `-o` | Specify install directory |
| `--force`, `-f` | Overwrite existing skill |
| `--json`, `-j` | Output as JSON |
| `--no-cache` | Bypass local cache |

## Skill Structure

Skills are markdown files with YAML frontmatter:

```markdown
---
name: my-skill
version: 1.0.0
description: What this skill does
allowed-tools:
  - Read
  - Write
  - exec
---

# My Skill

Instructions for the AI agent...
```

## Examples

### Find and install a skill
```bash
packs find email
# Shows: email-formatting, email-templates, etc.

packs get email-formatting --install
```

### Use a skill immediately
```bash
# Pipe directly into your agent's context
packs get humanizer >> AGENTS.md
```

### Share your skill
```bash
# From your repo with skills/my-skill/SKILL.md
packs submit @myuser/myrepo/skills/my-skill
```

## Tips

1. **Discovery first**: Use `packs find` or the TUI (`packs`) to explore before installing
2. **Direct fetch for speed**: Use `gh:` or `@` prefix to grab skills without waiting for registry
3. **Version pin for stability**: Use `skill@version` in production setups
4. **Local override**: Skills in your workspace take precedence over registry versions
