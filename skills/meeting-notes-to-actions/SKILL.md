# Skill: Meeting Notes to Actions

Extract actionable items from messy meeting transcripts or notes.

## When to Use
- Processing meeting recordings/transcripts
- Reviewing raw meeting notes
- Creating follow-up tasks from discussions
- Summarizing decisions for stakeholders

## The Extraction Framework

### Step 1: Identify Decisions
Look for:
- "We decided to..."
- "Let's go with..."
- "The plan is..."
- "We agreed that..."
- Someone proposing + others not objecting
- Explicit votes or approvals

**Format:**
```markdown
## Decisions
- **[Topic]:** [What was decided] — [Who has final say if relevant]
```

### Step 2: Extract Action Items
Look for:
- "I'll do X"
- "[Person] will handle..."
- "Can you..."
- "We need to..."
- "Next step is..."
- "By [date]..."

**For each action, capture:**
- **What:** Specific, concrete task
- **Who:** Owner (single person, not "the team")
- **When:** Deadline if mentioned

**Format:**
```markdown
## Action Items
- [ ] [Task] — @[Owner] by [Date]
- [ ] [Task] — @[Owner] (no date given)
```

### Step 3: Note Open Questions
Look for:
- Questions without answers
- "We need to figure out..."
- "I'm not sure about..."
- "Let's think about..."
- Unresolved debates

**Format:**
```markdown
## Open Questions
- [Question] — [Context if helpful]
```

### Step 4: Capture Key Information
Look for:
- New information shared
- Updates on projects
- Blockers mentioned
- Context others should know

**Format:**
```markdown
## Key Updates
- [Update] — [Source/Person]
```

## Output Template

```markdown
# [Meeting Name] — [Date]

**Attendees:** [Names]
**Duration:** [Time]

## TL;DR
[2-3 sentence summary of the meeting]

## Decisions
- **[Topic]:** [Decision]

## Action Items
- [ ] [Task] — @[Owner] by [Date]
- [ ] [Task] — @[Owner]

## Key Updates
- [Update or context shared]

## Open Questions
- [Unresolved question]

## Raw Notes
[Optional: cleaned up version of original notes]
```

## Processing Tips

### Messy Transcript? Clean First
1. Remove filler words ("um", "uh", "like")
2. Fix obvious transcription errors
3. Add paragraph breaks at topic changes
4. Mark speaker changes if not clear

### Multiple Owners? Pick One
"The team will do X" → Pick the most likely owner or flag for clarification.
Every action needs ONE accountable person.

### Vague Actions? Make Specific
- "Look into X" → "Research X and share findings in #channel by Friday"
- "Think about Y" → "Draft proposal for Y"
- "Follow up" → "Email [Person] about [Topic]"

### No Date? Add One
If no deadline mentioned, either:
- Ask for one
- Suggest one ("Assuming by EOW?")
- Mark as "TBD" and follow up

## Red Flags

### Decisions That Aren't Decisions
- "We should probably..." (not committed)
- "Maybe we could..." (not decided)
- "Let's think about..." (still open)

Mark these as Open Questions, not Decisions.

### Actions Without Owners
"We need to update the docs"
→ Who specifically? If unclear, add to Open Questions.

### Too Many Actions
If you have 20+ action items, meeting might have been unfocused.
Prioritize: What are the top 5 that actually matter?

## Distribution

### Who Gets What

**Full notes:** Attendees + relevant stakeholders

**Summary only:** Leadership, people who need context but not details

**Just actions:** People with assigned tasks

### Follow-Up
- Send within 24 hours (while memory is fresh)
- @ mention people on their action items
- Put actions in your task system (Linear, etc.)
- Schedule follow-up for open questions

## Example

**Input (messy transcript):**
> "So yeah I think we should go with React for the frontend, everyone cool with that? Yeah okay cool. Um, Sarah can you look into the auth options? We need to figure out if we're using Clerk or Auth0. By like, end of week? Also Mark mentioned the API is almost done but blocked on the database schema, we should probably help with that..."

**Output:**
```markdown
# Frontend Kickoff — Jan 30, 2026

## Decisions
- **Frontend framework:** Going with React

## Action Items
- [ ] Research auth options (Clerk vs Auth0) — @Sarah by Feb 7
- [ ] Help Mark with database schema — @[TBD - needs owner]

## Key Updates
- API almost done but blocked on database schema (Mark)

## Open Questions
- Who will help Mark with the database schema?
- Auth provider decision needed after Sarah's research
```
