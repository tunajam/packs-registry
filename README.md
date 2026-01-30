# 🎒 packs-registry

Official registry for [packs](https://packs.sh) — skills, contexts, and prompts for AI agents.

## Install the CLI

```bash
brew install packs
```

## Browse Packs

```bash
# Interactive TUI
packs

# Search
packs find "commit message"

# Install
packs get commit-message
```

## Available Packs

### Skills

| Name | Description | Author |
|------|-------------|--------|
| [commit-message](./skills/commit-message) | Generate conventional commit messages | tunajam |
| [pr-description](./skills/pr-description) | Write PR descriptions from branch diff | tunajam |
| [humanizer](./skills/humanizer) | Remove AI patterns from writing | blader |
| [claudeception](./skills/claudeception) | Extract learnings into reusable skills | blader |
| [test-driven-development](./skills/test-driven-development) | TDD workflow for features and bugfixes | obra |
| [brainstorming](./skills/brainstorming) | Transform ideas into designs | obra |
| [git-worktrees](./skills/git-worktrees) | Work with isolated git worktrees | obra |
| [mcp-builder](./skills/mcp-builder) | Build MCP servers for LLM integrations | composio |
| [youtube-transcript](./skills/youtube-transcript) | Extract transcripts from YouTube videos | tapestry |
| [skill-creator](./skills/skill-creator) | Guide for creating effective skills | anthropic |
| [webapp-testing](./skills/webapp-testing) | Test web apps with Playwright | anthropic |
| [web-artifacts-builder](./skills/web-artifacts-builder) | Build complex React/Tailwind artifacts | anthropic |

## Contributing

Want to add a pack to the registry?

1. Fork this repo
2. Create a folder in `skills/`, `contexts/`, or `prompts/`
3. Add `pack.yaml` and your content file (`SKILL.md`, `CONTEXT.md`, or `PROMPT.md`)
4. Submit a PR

Or submit directly from GitHub:

```bash
packs submit @yourname/yourrepo/yourpack
```

## Pack Structure

```
skills/
└── my-skill/
    ├── pack.yaml    # metadata
    └── SKILL.md     # instructions
```

**pack.yaml:**
```yaml
name: my-skill
version: 1.0.0
type: skill
description: What this skill does
author: your-name
tags:
  - tag1
  - tag2
license: MIT
```

## License

Packs in this registry are licensed individually by their authors. See each pack's `pack.yaml` for license info.

The registry infrastructure is MIT licensed.
