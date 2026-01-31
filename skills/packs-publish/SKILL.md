# Packs Publish

Push skills to the packs registry so other agents can use them.

## When to Use

- After creating a new skill locally
- When you want to share a learned skill with the community
- After improving an existing skill

## Prerequisites

- GitHub CLI (`gh`) authenticated
- Write access to `tunajam/packs-registry`

## The Process

### Step 1: Create the Skill Locally

Every skill needs at minimum:
```
skills/<skill-name>/
├── SKILL.md      # Main skill file (required)
├── pack.yaml     # Metadata (required)
└── references/   # Optional supporting files
```

### Step 2: Write SKILL.md

Structure:
```markdown
# Skill Name

One-line description of what this skill does.

## When to Use

- Trigger condition 1
- Trigger condition 2

## Instructions

Step-by-step guide for the agent to follow.

## Examples

Concrete examples of inputs/outputs.

## Resources

Links to external docs, references.
```

**Quality checklist:**
- [ ] Clear trigger conditions (when should an agent load this?)
- [ ] Actionable instructions (not just concepts)
- [ ] Examples where helpful
- [ ] No vague language

### Step 3: Create pack.yaml

```yaml
name: skill-name           # kebab-case, matches folder name
version: 1.0.0             # semver
type: skill                # skill | context | prompt
description: Short description for search results
author: your-name
tags:
  - relevant-tag
  - another-tag
license: MIT
source_url: https://github.com/tunajam/packs-registry
```

**Tag guidelines:**
- Use existing tags when possible (check other skills)
- 3-5 tags is ideal
- Include the primary domain (git, react, planning, etc.)

### Step 4: Clone and Add to Registry

```bash
# Clone the registry
cd /tmp
rm -rf packs-registry
gh repo clone tunajam/packs-registry
cd packs-registry

# Copy your skill
cp -r ~/path/to/your/skill skills/

# Verify structure
ls -la skills/your-skill-name/
```

### Step 5: Commit and Push

```bash
cd /tmp/packs-registry

# Stage
git add skills/your-skill-name

# Commit with conventional message
git commit -m "feat: add your-skill-name skill

Brief description of what it does.

Co-authored-by: Fred <fred@tunajam.com>"

# Push
git push
```

### Step 6: Verify

```bash
# Direct fetch (works immediately)
packs get gh:tunajam/packs-registry/skills/your-skill-name

# Search (may take a few minutes to index)
packs find "keywords" --no-cache
```

## Updating Existing Skills

```bash
cd /tmp
rm -rf packs-registry
gh repo clone tunajam/packs-registry
cd packs-registry

# Edit the skill
vim skills/existing-skill/SKILL.md

# Bump version in pack.yaml
vim skills/existing-skill/pack.yaml

# Commit
git add skills/existing-skill
git commit -m "fix: improve existing-skill instructions"
git push
```

## Adding Reference Files

For skills that need examples, templates, or checklists:

```
skills/your-skill/
├── SKILL.md
├── pack.yaml
└── references/
    ├── examples.md
    ├── checklist.md
    └── template.md
```

Reference these in SKILL.md:
```markdown
See `references/examples.md` for detailed examples.
```

## Quick Reference

| Field | Required | Notes |
|-------|----------|-------|
| SKILL.md | Yes | Main content |
| pack.yaml | Yes | Metadata for registry |
| references/ | No | Supporting files |
| assets/ | No | Images, templates |

## Common Mistakes

- **Missing pack.yaml** — Skill won't be indexed
- **Name mismatch** — Folder name must match `name` in pack.yaml
- **Vague descriptions** — Makes skills hard to find
- **No trigger conditions** — Agents don't know when to use it
