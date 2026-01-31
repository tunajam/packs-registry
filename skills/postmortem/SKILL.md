# Skill: Postmortem

Analyze what went wrong without blame. Learn from failures to prevent recurrence.

## When to Use
- After an incident or outage
- After a project failure or major delay
- After a significant bug reaches production
- After any situation where you think "that shouldn't have happened"

## Core Principle: Blameless

**The goal is learning, not punishment.**

People make mistakes because systems allow them to. 
If someone could easily cause a disaster, the system is the problem.

```
❌ "John deployed bad code on Friday"

✅ "Our deployment process allowed untested code to reach production"
```

## The Postmortem Process

### 1. Gather Facts (Not Opinions)
Within 24-48 hours of the incident:

**What to collect:**
- Timeline of events (with timestamps)
- Who was involved and what they did
- System logs and metrics
- Customer reports
- What was tried during the incident

**How to collect:**
- Interview people involved (no blame, just facts)
- Pull logs and dashboards
- Check communication channels (Slack, etc.)

### 2. Build the Timeline

```markdown
## Timeline

**2026-01-30**
- 14:32 — Deploy to production (commit abc123)
- 14:35 — First error alerts fire
- 14:37 — On-call engineer paged
- 14:42 — Engineer begins investigation
- 14:55 — Root cause identified (database connection limit)
- 15:02 — Rollback initiated
- 15:08 — Service restored
- 15:15 — All-clear confirmed

**Total time to detection:** 3 minutes
**Total time to resolution:** 36 minutes
```

### 3. Identify Contributing Factors
Not "the cause"—usually multiple factors combine.

Ask "why" repeatedly (5 Whys technique):
1. Why did the service go down? → Database connections exhausted
2. Why were connections exhausted? → New feature opened connections without closing
3. Why weren't connections closed? → Missing cleanup in error handling path
4. Why wasn't this caught in testing? → Load tests don't run on every PR
5. Why don't load tests run on PRs? → They're slow and expensive

Now you have real factors to address.

### 4. Determine Impact
Be specific:
- How many users affected?
- For how long?
- What couldn't they do?
- Revenue impact?
- Reputation impact?

```markdown
## Impact
- **Duration:** 36 minutes
- **Users affected:** ~2,400 (15% of active users)
- **Revenue impact:** ~$1,200 in failed transactions
- **Customer tickets:** 47
```

### 5. Identify Action Items
For each contributing factor, what would prevent it?

**Good action items are:**
- Specific and concrete
- Have an owner
- Have a deadline
- Actually preventive (not just "be more careful")

```markdown
## Action Items

### Prevent (Stop this from happening)
- [ ] Add connection pool limits with automatic recycling — @Sarah, Feb 7
- [ ] Add integration test for connection cleanup — @Mike, Feb 5

### Detect (Catch it faster)
- [ ] Add alert for connection pool > 80% — @Sarah, Feb 3
- [ ] Add runbook for connection exhaustion — @DevOps, Feb 7

### Mitigate (Reduce impact when it happens)
- [ ] Implement circuit breaker for database calls — @Platform, Feb 14
- [ ] Add graceful degradation for non-critical features — @Product, Feb 21
```

### 6. Write the Postmortem

## Postmortem Template

```markdown
# [Incident Name] Postmortem

**Date:** YYYY-MM-DD
**Author:** [Name]
**Status:** Draft / Final

## Summary
[2-3 sentences: what happened, impact, resolution]

## Impact
- **Duration:** 
- **Users affected:** 
- **Severity:** Critical / Major / Minor

## Timeline
[Detailed timeline with timestamps]

## Root Cause
[What ultimately caused the incident—system-focused, not person-focused]

## Contributing Factors
1. [Factor 1]
2. [Factor 2]
3. [Factor 3]

## What Went Well
- [Thing that helped]
- [Good response]

## What Went Poorly
- [Thing that made it worse]
- [Delayed detection/response]

## Action Items

### Prevent
- [ ] [Action] — @Owner, Due Date

### Detect
- [ ] [Action] — @Owner, Due Date

### Mitigate
- [ ] [Action] — @Owner, Due Date

## Lessons Learned
[Key takeaways for the team]

## Appendix
[Logs, graphs, relevant data]
```

## Red Flags

### Blame Language
- ❌ "John should have known..."
- ❌ "If only Sarah had..."
- ❌ "The team failed to..."
- ✅ "The system allowed..."
- ✅ "The process didn't catch..."

### Vague Action Items
- ❌ "Be more careful with deployments"
- ❌ "Improve testing"
- ✅ "Add pre-deploy checklist to PR template"
- ✅ "Add integration test for X scenario"

### Missing Follow-Through
Postmortems are useless if action items aren't completed.
- Put action items in your ticketing system
- Assign owners and deadlines
- Review completion in 2-4 weeks

## Quick Postmortem (Minor Incidents)

For smaller issues, a lightweight version:

```markdown
## Quick Postmortem: [Issue]

**What happened:** [One paragraph]

**Why:** [Contributing factors]

**Action items:**
- [ ] [Item] — @Owner
```

## Output

Every postmortem should result in:
1. Shared understanding of what happened
2. At least one action item to prevent recurrence
3. No individual blame
4. Documentation for future reference
