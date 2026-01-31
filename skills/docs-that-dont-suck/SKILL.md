# Skill: Docs That Don't Suck

Write documentation people actually read and find useful.

## When to Use
- Writing README files
- Creating API documentation
- Writing onboarding guides
- Documenting processes or decisions

## Why Most Docs Suck

- Too long (nobody reads them)
- Too abstract (no concrete examples)
- Out of date (written once, never updated)
- Wrong audience (written for experts or beginners, never the actual reader)
- Missing the "why" (all "what", no context)

## The Golden Rules

### 1. Start With a Working Example
Don't explain, then show. Show, then explain.

```markdown
❌ Bad:
"The API accepts a JSON payload with the following fields..."

✅ Good:
curl -X POST https://api.example.com/users \
  -d '{"email": "test@test.com", "name": "Test"}'

This creates a user. Here's what each field does...
```

### 2. Write for Scanning, Not Reading
People scan docs looking for their answer. Help them:

- **Headings** that are actual questions ("How do I authenticate?")
- **Bold** the key terms they're searching for
- **Code blocks** stand out visually
- **Short paragraphs** (3-4 lines max)
- **Lists** for multiple items

### 3. Answer "Why" Before "What"

```markdown
❌ Bad:
"Set ENABLE_CACHE=true to enable caching."

✅ Good:
"API responses are cached for 5 minutes to reduce load. 
Set ENABLE_CACHE=false during development to see changes immediately."
```

### 4. Include the Failure Modes
What happens when it goes wrong?

```markdown
✅ Good:
"If you see 'Connection refused', check that the database is running:
docker ps | grep postgres"
```

### 5. Keep It Short
Every sentence must earn its place. If it doesn't help someone do something, cut it.

## Doc Types and Templates

### README (Project Overview)

```markdown
# Project Name

One sentence: what is this and who is it for?

## Quick Start

[Minimal steps to see it working]

## Installation

[Copy-paste commands]

## Usage

[Most common use case with example]

## Configuration

[Table of env vars / options]

## Troubleshooting

[Common errors and fixes]
```

### API Documentation

```markdown
## Endpoint Name

Brief description of what this does.

**Request:**
[Full curl example that works]

**Response:**
[Actual JSON response]

**Errors:**
| Code | Meaning | Fix |
|------|---------|-----|
| 401  | Bad token | Refresh your API key |
| 429  | Rate limited | Wait 60 seconds |
```

### How-To Guide

```markdown
# How to [Accomplish Task]

**You'll need:**
- Prereq 1
- Prereq 2

**Steps:**

1. [Do this first]
   [Code or screenshot]

2. [Then this]
   [Code or screenshot]

**Verify it worked:**
[How to confirm success]

**Common issues:**
[Troubleshooting]
```

### Decision Document (ADR)

```markdown
# [Decision Title]

**Status:** Accepted
**Date:** YYYY-MM-DD

## Context
[What's the situation? What problem are we solving?]

## Decision
[What we chose to do]

## Consequences
[What happens as a result—good and bad]

## Alternatives Considered
[What else we looked at and why we didn't choose it]
```

## Writing Tips

### Verbs > Nouns
- ❌ "The configuration of the system..."
- ✅ "Configure the system by..."

### Active > Passive
- ❌ "The file should be saved..."
- ✅ "Save the file to..."

### Specific > Generic
- ❌ "It may take some time..."
- ✅ "This takes about 30 seconds..."

### Real Examples > Fake Ones
- ❌ `{"foo": "bar"}`
- ✅ `{"email": "jane@company.com", "role": "admin"}`

## The Maintenance Problem

Docs rot. Prevent it:

1. **Link to source of truth** — Don't duplicate, reference
2. **Date stamp** — "Last updated: Jan 2026"
3. **Keep it close to code** — README in repo, not wiki
4. **Review on change** — PR template: "Did you update the docs?"

## Output

Good docs should pass this test:
- [ ] Can a new person follow it without asking questions?
- [ ] Is there a working example in the first 30 seconds of reading?
- [ ] Would you actually read this?
