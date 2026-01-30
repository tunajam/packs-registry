---
name: overnight-build
description: |
  Autonomous overnight build challenge. Research a real problem on Reddit, validate it,
  and build a polished working app by morning. Use when given autonomy to find and solve
  a problem independently.
user-invocable: true
---

# Overnight Build Challenge

## Command Invocation

When invoked via `/overnight_build` command, **immediately spawn a sub-agent** to run autonomously:

```
sessions_spawn(
  task: "Load and execute skills/learned/overnight-build/SKILL.md fully. Research a real problem on Reddit that people are complaining about, validate it against our Linear backlog and existing market solutions, then build a polished working app. Follow ALL validation steps - do not skip checking for existing solutions. Report back with: 1) The problem you identified, 2) Why it's not already solved, 3) What you built, 4) Live URL.",
  label: "overnight-build",
  runTimeoutSeconds: 21600
)
```

Reply to the user: "🚀 Kicked off overnight build challenge. I'll research Reddit for a real problem, validate it's worth solving, and build a working app. Check back in the morning!"

---

Research a problem users are complaining about, validate it's worth solving, and build a full working solution.

## Trigger

When given a prompt like:
> "Research a problem that many users are complaining about on Reddit that can be solved with software and build a full working version of the app by the morning. It should be polished, make sense and provide a simple solution to the problem."

## Process

### Phase 1: Research (30-60 min)

1. **Load research skills:**
   - `skills/learned/reddit-business-research/SKILL.md`
   - `skills/learned/founder-idea-research/SKILL.md`

2. **Find pain points on Reddit:**
   - Search r/SaaS, r/Entrepreneur, r/smallbusiness for complaints
   - Look for "I wish there was" / "someone should build" / "I'd pay for"
   - Check niche subs (r/ADHD, r/personalfinance, r/cooking, etc.)
   - Measure frustration: longer posts = deeper pain

3. **Identify 3-5 candidate problems**

### Phase 2: Validation (CRITICAL — Don't Skip!)

**Before choosing a problem, run these checks:**

- [ ] **Search our Linear backlog:**
  ```bash
  linearis issues search "<problem keywords>" --team TUN
  linearis issues search "<problem keywords>" --team TEN
  ```

- [ ] **Check ideas.tunajam.com** — Is this already in our idea list?

- [ ] **Check our existing products** — Have we already built this?

- [ ] **Search for existing solutions:**
  - Google the problem + "app" / "tool" / "extension"
  - Check Product Hunt for similar products
  - If dominant solution exists (like "Just the Recipe" for recipe cleaning), **STOP**

- [ ] **Identify differentiation (if proceeding despite competition):**
  - What's our unique angle?
  - Why would someone use ours over existing solutions?
  - "Web app vs extension" is weak differentiation

**If the problem is already solved well, go back to Phase 1 and pick a different problem.**

### Phase 3: Plan (15 min)

1. **Create Linear issue FIRST:**
   ```bash
   linearis issues create "Project Name: Brief description" --team TUN \
     --description "Problem + Solution + Stack" \
     --assignee "1b156695-4fac-452e-b5bd-8c5e3c63b5a7"
   ```

2. **Write plan to file:**
   - Save to `memory/plans/<project-name>.md`
   - Include: problem, signal/research, solution sketch, tech stack, phases

3. **Define MVP scope:**
   - What's the simplest thing that delivers value?
   - Cut everything that's not essential for v1

### Phase 4: Build (3-5 hours)

**Standard Tunajam stack:**
- Next.js 15 (App Router)
- Tailwind CSS + shadcn/ui
- Vercel deployment
- OpenRouter API (Gemini Flash) if AI needed

**Build order:**
1. Project setup with shadcn/ui
2. Core UI (main page, input, display)
3. Backend/API logic
4. Polish (loading states, errors, mobile)
5. Deploy to Vercel

**Remember:**
- Get environment variables into Vercel
- Test the live deployment, not just localhost
- Commit and push to GitHub

### Phase 5: Document & Report

1. **Update Linear issue** with status and deployment URL

2. **Update daily notes** (`memory/YYYY-MM-DD.md`):
   - What was built
   - Research findings
   - Lessons learned

3. **Prepare summary for Hunter:**
   - Problem identified
   - Solution built
   - Live URL
   - Research signal (why this problem)
   - What makes it different (if applicable)

## Quality Bar

The finished product should:
- [ ] Solve a real problem people complained about
- [ ] Work end-to-end (not just a demo)
- [ ] Look polished and professional
- [ ] Be deployed and accessible via public URL
- [ ] Be something that **doesn't already exist** (or has clear differentiation)

## Lessons Learned

### 2026-01-28: WhatNow (SUCCESS)
- Built an ADHD-friendly task prioritizer
- **Signal:** r/ADHD has highest-signal feature requests; users frustrated with complex task apps
- **Differentiation:** Radical simplicity (ONE task at a time), privacy-focused (all client-side)
- **URL:** https://whatnow-one.vercel.app
- **Lesson:** Building for a specific niche with clear pain points beats generic tools
- **Lesson:** "Simplicity IS the feature" - less is more for overwhelmed users
- **Lesson:** Most problems have solutions, but differentiation through execution works

### 2026-01-28: RecipeZen (FAILED)
- Built a recipe cleaner (paste URL → get clean recipe)
- **Mistake:** Didn't check that "Just the Recipe" browser extension already exists
- **Lesson:** Always validate against existing solutions before building
- The research was good, the execution was fine, but the problem was already solved

## Anti-Patterns

❌ Building something that already exists with no differentiation
❌ Skipping the validation phase because you're excited to code
❌ "Web app vs extension" as your only differentiator
❌ Picking the first problem you find without checking alternatives
❌ Not searching our own Linear/ideas backlog first

## Good Patterns

✅ Check Linear and ideas.tunajam.com FIRST
✅ Search for existing solutions before committing
✅ Pick problems with high frustration + no dominant solution
✅ Define clear differentiation if competition exists
✅ Ship something that works, even if simple
