# Skill: When to Stop

Scope creep kills projects. This skill helps you recognize when to ship and when you're gold-plating.

## When to Use
- You've been working on something "almost done" for too long
- You keep finding "one more thing" to add
- You're not sure if a feature is necessary
- You're polishing instead of shipping

## The Core Question

> "If I shipped this right now, would it solve the user's core problem?"

If yes → ship it.
If no → what's the ONE thing blocking that?

## Scope Creep Warning Signs

### 🚨 Red Flags
- "While I'm in here, I might as well..."
- "It would be nice if..."
- "What if the user wants to..."
- "This won't take long..."
- "Let me just refactor this first..."
- "We should probably also..."

### 📊 Time Signals
- Task taking 2x+ the original estimate
- You've context-switched back to this 3+ times
- You're on the 4th approach to the same problem
- You're optimizing something that works

## The Stopping Framework

### 1. Define Done (Before Starting)
Write down what "done" means before you start:
- [ ] Feature X works
- [ ] Edge case Y is handled
- [ ] Tests pass
- [ ] Deployed to staging

If it's not on the list, it's not required.

### 2. The 80/20 Check
Ask: "Am I in the 80% that delivers value, or the 20% that's polish?"

| 80% (Do it) | 20% (Probably skip) |
|-------------|---------------------|
| Core functionality | Animations |
| Error handling | Perfect loading states |
| Basic styling | Pixel-perfect design |
| Happy path works | Every edge case |

### 3. The Reversal Test
Ask: "What's the cost of NOT doing this?"
- If low → skip it
- If "user can't complete task" → do it
- If "it looks slightly worse" → skip it

### 4. The Tomorrow Test
Ask: "If I shipped today and added this tomorrow, would anyone care?"
- Usually no
- Ship today

## Good Enough vs Not Done

### Good Enough ✅
- Works for 90% of users
- Handles common errors gracefully
- Looks acceptable (not beautiful)
- Has reasonable performance
- Can be improved later

### Not Done ❌
- Core feature is broken
- Common error crashes the app
- Looks broken/unusable
- Unusably slow
- Data loss possible

## Practical Tactics

### Timebox Everything
"I'll spend 2 hours max on this polish. Whatever state it's in, I ship."

### Make a "Later" List
Write down the "nice to haves" somewhere. They're not forgotten, just deferred.

### Ship and Watch
Real user behavior > your imagination. Ship, see if anyone complains, then fix what matters.

### Ask Someone Else
"Hey, is this done enough to ship?" External perspective breaks the loop.

## The Mantra

> "Shipping beats perfect. 
> Done is better than impressive.
> Users want solutions, not polish."

## Output

When you notice scope creep:
1. Stop what you're doing
2. Ask: "Does this solve the core problem?"
3. If yes → commit and ship
4. If no → identify the ONE blocker and fix only that
