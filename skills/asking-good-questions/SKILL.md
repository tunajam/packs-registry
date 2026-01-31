# Skill: Asking Good Questions

Clarify requirements before building. Half of all wasted work comes from building the wrong thing.

## When to Use
- Before starting any feature or task
- When requirements feel vague or ambiguous
- When you're making assumptions about behavior
- Before a design that could go multiple directions

## The Question Categories

### 1. Scope Questions
*What are we actually building?*

- What's the MVP vs nice-to-have?
- What's explicitly out of scope?
- Is this a quick fix or a foundation for more?
- What does "done" look like?

### 2. User Questions
*Who is this for and what do they need?*

- Who's the primary user?
- What's their current workflow/pain?
- What triggers them to use this?
- What does success look like for them?

### 3. Behavior Questions
*How should it actually work?*

- What happens when X? (edge cases)
- What's the error state?
- What's the empty state?
- What's the loading state?
- What if the user does Y unexpectedly?

### 4. Technical Questions
*What constraints exist?*

- What's the performance requirement?
- What's the data model?
- What existing systems does this touch?
- Are there security/privacy concerns?
- What's the rollback plan if it breaks?

### 5. Priority Questions
*What matters most?*

- If we can only ship one thing, what is it?
- What's the deadline/urgency?
- Is this blocking something else?
- Can we ship incrementally?

## The Process

### Step 1: List Your Assumptions
Before asking anything, write down what you're assuming:
- "I assume this needs to work on mobile"
- "I assume we're storing this in the database"
- "I assume the user is already logged in"

### Step 2: Convert to Questions
Turn each assumption into a question:
- "Does this need to work on mobile?"
- "Where should this data live?"
- "Can we assume the user is authenticated?"

### Step 3: Identify the Scary Ones
Which questions, if answered differently than you assumed, would change everything?
**Ask those first.**

### Step 4: Ask Efficiently
Don't ask 20 questions. Group them:
> "Before I start, a few quick clarifications:
> 1. Is mobile in scope for v1?
> 2. Should this persist or is session-only fine?
> 3. What happens if [edge case]?"

### Step 5: Document the Answers
Write down the answers. Put them in the ticket/doc/plan. Future you will thank present you.

## Red Flags (When to Dig Deeper)

- **"Just make it work"** → What does "work" mean specifically?
- **"Keep it simple"** → Simple for who? In what way?
- **"Like [other product]"** → Which specific aspects?
- **"ASAP"** → What's the actual deadline?
- **"Users will figure it out"** → Will they though?

## Anti-Patterns

❌ Asking questions you could answer yourself with 5 minutes of research
❌ Asking everything at once (overwhelming)
❌ Asking leading questions to confirm what you want to build
❌ Not asking anything and just assuming

## Output

Before starting work, you should have:
- [ ] Clear scope (what's in, what's out)
- [ ] Defined user and their goal
- [ ] Key behaviors documented
- [ ] Edge cases identified
- [ ] Priority clear

If you don't have these, keep asking.
