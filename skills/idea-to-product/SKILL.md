# Idea to Product

Transform rough ideas into comprehensive product specifications and implementation plans. This skill bridges the gap between "what if we built..." and production-ready software.

## When to Use

- Starting a new project from scratch
- Taking a vague idea and making it buildable
- Creating PRDs that AI coding agents can implement
- Structuring work before writing any code

## The Process

### Phase 0: Idea Capture (5 minutes)

Before anything else, capture the raw idea:

```markdown
## Raw Idea
[One sentence: What is it?]

## The Problem
[What pain does this solve? Who has this pain?]

## Why Now
[Why hasn't this been built? What changed?]

## Initial Hypothesis
[If we build X, then Y will happen because Z]
```

Save to `memory/ideas/<idea-name>.md` immediately. Ideas evaporate.

### Phase 1: Validation (15-30 minutes)

Before building, validate the problem exists:

**Quick Checks:**
```bash
# Search for existing solutions
web_search "[problem] solution software"
web_search "[problem] app alternative"

# Check if people are complaining
web_search "reddit [problem] frustrating"
web_search "hacker news [problem]"

# Check Linear for existing ideas
linearis issues search "<keywords>" --team TUN
linearis issues search "<keywords>" --team TEN
```

**Validation Questions:**
- [ ] Is anyone actively complaining about this problem?
- [ ] Are people paying for inferior solutions?
- [ ] What's the existing workaround? (Excel, manual process, etc.)
- [ ] Why would someone switch to our solution?

**Red Flags:**
- No complaints found → Problem might not be real
- Dominant existing solutions → Need clear differentiator
- "No one's built this" → Investigate why carefully

### Phase 2: User Definition

Define exactly who you're building for:

```markdown
## Primary Persona

**Name:** [Give them a name]
**Role:** [Job title or life situation]
**Problem Statement:** As a [role], I struggle with [problem] because [reason].

**Context:**
- How often do they encounter this problem?
- What do they currently do about it?
- What would they pay? (time/money/attention)

**Jobs to Be Done:**
1. [Primary job they're trying to accomplish]
2. [Secondary job]
3. [Emotional job - how they want to feel]
```

### Phase 3: PRD Creation

Create a comprehensive PRD. This is the source of truth.

**PRD Structure:**

```markdown
# [Product Name] PRD

**Author:** [Your name]
**Status:** Draft | Review | Approved
**Last Updated:** [Date]
**Linear Project:** [TUN-XXX or TEN-XXX]

---

## 1. Overview

### Problem Statement
[One paragraph explaining the pain point]

### Proposed Solution
[One paragraph explaining our approach]

### Success Metrics
| Metric | Target | Measurement |
|--------|--------|-------------|
| [Primary metric] | [Value] | [How to measure] |
| [Secondary metric] | [Value] | [How to measure] |

### Non-Goals
- [Explicitly list what we're NOT building]
- [Be specific to prevent scope creep]
- [If not listed here, AI might build it]

---

## 2. User Stories

### Epic 1: [Feature Area]

#### US-1.1: [Action Name]
**As a** [persona]
**I want to** [action]
**So that** [outcome/value]

**Acceptance Criteria:**
- [ ] [Specific, testable requirement]
- [ ] [Another requirement]
- [ ] [Machine-verifiable where possible]

**UX Notes:**
- [Design considerations]
- [Edge cases]

---

## 3. Technical Requirements

### Functional Requirements
| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | System shall [action] when [trigger] | P0 |
| FR-2 | System shall [action] within [timeframe] | P1 |

### Non-Functional Requirements
| Category | Requirement |
|----------|-------------|
| Performance | Page load <2s, API response <500ms |
| Security | [Auth requirements, data handling] |
| Accessibility | WCAG 2.1 AA compliance |

---

## 4. Scope

### In Scope (MVP)
- [Feature 1]
- [Feature 2]
- [Feature 3]

### Out of Scope (v2+)
- [Deferred feature 1]
- [Deferred feature 2]

### Explicit Constraints
DO NOT IMPLEMENT:
- [Thing that might seem logical but isn't wanted]
- [Feature that could cause scope creep]

---

## 5. Open Questions
- [ ] [Unresolved design decision]
- [ ] [Technical question to research]
```

### Phase 4: Phased Implementation Plan

Break the PRD into sequential, AI-implementable phases:

```markdown
# Implementation Plan: [Product Name]

## Phasing Principles
- Each phase: 15-30 minutes of agent work
- Each phase ends with verifiable functionality
- No dead ends: codebase must be runnable after each phase
- Clear dependencies between phases

---

## Phase 1: Foundation
**Goal:** [What's complete when this phase ends]
**Dependencies:** None
**Verification:** [How to manually test]

### Tasks
- [ ] [Specific task with commit message]
- [ ] [Another task]

### DO NOT CHANGE (Protected)
- [Nothing yet - first phase]

---

## Phase 2: [Core Feature]
**Goal:** [What's complete]
**Dependencies:** Phase 1
**Verification:** [Manual test steps]

### Tasks
- [ ] [Task]

### DO NOT CHANGE (Protected)
- Database schema from Phase 1
- [Other stable components]

---

## Phase 3: [Next Feature]
...
```

**Phase Sizing Heuristic:**
- Small: 5-15 minutes agent work, 1-3 tasks
- Medium: 15-30 minutes, 3-5 tasks  
- Large: 30-60 minutes, 5-8 tasks (break down further if possible)

### Phase 5: Linear Integration

Create tickets for each phase:

```bash
# Create parent epic
linearis issues create "[Product] MVP" --team TUN \
  --description "Parent epic for [product] build"

# Create phase tickets
linearis issues create "Phase 1: Foundation" --team TUN \
  --description "$(cat << 'EOF'
## Goal
[From implementation plan]

## Tasks
- [ ] Task 1
- [ ] Task 2

## Verification
- [ ] [Test step 1]
- [ ] [Test step 2]

## Dependencies
None
EOF
)"

# Link phases to epic
linearis issues update TUN-XXX --parent TUN-YYY
```

## Output Files

After running this skill, you should have:

| File | Location | Purpose |
|------|----------|---------|
| Idea capture | `memory/ideas/<name>.md` | Raw idea + hypothesis |
| PRD | `<repo>/prd.md` | Product requirements |
| Implementation plan | `memory/plans/<name>.md` | Phased build plan |
| Linear tickets | Linear | Trackable work items |

## Quick Reference

### PRD Quality Checklist
- [ ] Problem statement is specific and validated
- [ ] Success metrics are quantified
- [ ] Non-goals are explicitly listed
- [ ] User stories have testable acceptance criteria
- [ ] Requirements use active verbs (display, calculate, send)
- [ ] No vague terms (appropriate, reasonable, etc.)
- [ ] Each requirement is atomic (one thing per statement)

### Phase Quality Checklist
- [ ] Each phase ends with working functionality
- [ ] Verification steps are manual and concrete
- [ ] DO NOT CHANGE sections protect stable code
- [ ] Dependencies are explicitly stated
- [ ] Tasks are small enough for one commit each

### Writing for AI Agents

**Good:** "System shall display error message 'Invalid email format' when email field fails regex validation"

**Bad:** "System should show appropriate feedback for invalid inputs"

**Good:** "API endpoint returns 200 with JSON body {success: true} within 500ms"

**Bad:** "API should respond quickly with success status"

## Integration with Tunajam Workflow

This skill feeds into:
1. **NEW_PROJECT.md** template — Use that for full project setup
2. **Linear** — All work gets tickets
3. **Planning files** — Saved to `memory/plans/`
4. **PR workflow** — Each phase becomes commits on a feature branch

## Resources

- [How to write PRDs for AI Coding Agents](https://medium.com/@haberlah/how-to-write-prds-for-ai-coding-agents-d60d72efb797)
- [GitHub Spec-Driven Development](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [ChatPRD Resources](https://www.chatprd.ai/resources)
