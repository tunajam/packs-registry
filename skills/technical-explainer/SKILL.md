# Skill: Technical Explainer

Explain complex technical concepts to people without technical backgrounds.

## When to Use
- Explaining a feature to stakeholders
- Writing docs for end users
- Presenting to executives
- Teaching a concept to beginners
- Writing marketing copy for technical products

## Core Principles

### 1. Start With Why They Care
Don't explain what it is. Explain why it matters to them.

```
❌ "We implemented a caching layer using Redis..."

✅ "The app will load 3x faster now. Here's how..."
```

### 2. Use Analogies
Map the unfamiliar to the familiar.

| Technical Concept | Analogy |
|-------------------|---------|
| API | A waiter taking your order to the kitchen |
| Database | A filing cabinet that never forgets |
| Cache | Keeping frequently-used items on your desk |
| Encryption | A lockbox only you have the key to |
| Load balancer | A traffic cop directing cars |
| Version control | Track changes in Word, but for code |
| Bug | A typo that breaks things |
| Server | A computer that's always on, waiting to help |
| Cloud | Someone else's computer you rent |
| Algorithm | A recipe for solving a problem |

### 3. Concrete Before Abstract
Show an example, then explain the pattern.

```
❌ "Authentication verifies user identity using credentials."

✅ "When you log in with your email and password, we check 
   that you're really you—like showing ID at a bar. 
   That's authentication."
```

### 4. One Concept at a Time
Don't explain APIs, databases, AND caching in one breath.
Build up piece by piece.

### 5. Check Understanding
"Does that make sense so far?"
"Should I explain [X] more?"

## The Explanation Framework

### Step 1: Identify What They Actually Need to Know
Ask: What decision or understanding does this enable?
- They don't need to know HOW Redis works
- They need to know the app will be faster

### Step 2: Find Their Frame of Reference
What do they already understand that you can build on?
- Business person? Use money/efficiency analogies
- Designer? Use visual/creative analogies
- Manager? Use process/workflow analogies

### Step 3: Build the Bridge

```
[Thing they know] → [Connection] → [New concept]

"You know how Google Docs saves every change automatically? 
 Git is like that for code, but with more control over 
 which versions to keep."
```

### Step 4: Strip Jargon
Every field has words that seem normal but aren't:
- Deploy → Put live / Launch
- Refactor → Clean up the code
- Latency → Delay / Wait time
- Scalable → Can handle more users
- Robust → Reliable / Won't break easily
- Leverage → Use
- Synergy → (just don't)

### Step 5: Test With the "Mom Test"
Could your non-technical parent understand this?
If not, simplify further.

## Common Mistakes

### Too Much Detail
They asked how the clock works, not how to build one.

```
❌ "The JWT token contains a header with the algorithm, 
   a payload with claims, and a signature using HMAC-SHA256..."

✅ "You get a digital pass that proves you logged in. 
   It expires after a while for security."
```

### Condescension
Simplify without talking down.

```
❌ "So basically, computers are really good at math..."

✅ "The short version is..." (then give them credit to follow)
```

### False Simplification
Some things can't be made simple without being wrong.
It's okay to say: "The details are complex, but the key point is..."

### Jargon Creep
You explained one term with three other terms.

```
❌ "The API endpoint validates the schema against the contract..."

✅ "We check that the data looks right before accepting it."
```

## Templates

### Feature Explanation (to Stakeholders)
```
**What it does:** [One sentence, user perspective]

**Why it matters:** [Business value—time saved, revenue, user happiness]

**How it works:** [Simple analogy, no jargon]

**What you'll see:** [Observable change]
```

### Technical Decision (to Leadership)
```
**The situation:** [Problem in plain terms]

**What we're doing:** [Solution, one sentence]

**Why this approach:** [1-2 reasons they'd care about—cost, speed, risk]

**What it means for you:** [Impact on timeline, budget, product]
```

### Bug Explanation (to Non-Technical Reporter)
```
**What happened:** [What they experienced]

**Why it happened:** [Simple cause—no blame, no jargon]

**What we're doing:** [Fix in progress/done]

**Will it happen again:** [What we're doing to prevent it]
```

## Practice Phrases

Start explanations with:
- "Think of it like..."
- "Imagine if..."
- "You know how [familiar thing]? It's like that, but..."
- "The simple version is..."
- "What this means for you is..."

End explanations with:
- "Does that make sense?"
- "Want me to go deeper on any part?"
- "The bottom line is..."

## Output

Good technical explanations:
- Lead with why it matters
- Use familiar analogies
- Avoid jargon (or define it immediately)
- Are shorter than you think they need to be
- End with the so-what
