# Skill: Commit Message Generator

Generate a conventional commit message for staged or unstaged changes.

## When to Use
- Before any commit
- When you need a well-formatted conventional commit message
- After making code changes that need to be committed

## Instructions

1. **Gather context:**
   - Check staged changes: `git diff --cached`
   - If nothing staged, check unstaged changes: `git diff`
   - List changed files: `git status --short`

2. **Analyze what the code actually does** - read the diff carefully.

3. **Generate a conventional commit message:**

```
<type>(<scope>): <subject>

<body>
```

## Types
- `feat`: New feature
- `fix`: Bug fix  
- `refactor`: Code change that neither fixes nor adds
- `docs`: Documentation only
- `chore`: Build, config, tooling
- `test`: Adding tests
- `perf`: Performance improvement

## Guidelines
- **Subject:** Imperative mood, lowercase, no period, max 50 chars
- **Body:** Explain the "why" not the "what"

## Output
Write the message to `.git/COMMIT_MSG` and show for review.
