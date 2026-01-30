# PR Description Generator

Generate a PR description by analyzing git changes between the current branch and a target base branch.

## When to Use
- Before creating a pull request
- When you need to summarize branch changes
- To generate a well-formatted PR body

## Instructions

1. **Determine the base branch:**
   - If specified, use it
   - Otherwise, use `master` if it exists, else `main`

2. **Gather context:**
   - Get current branch: `git rev-parse --abbrev-ref HEAD`
   - Get commits: `git log --oneline <base>..HEAD`
   - Get changed files: `git diff --name-only <base>...HEAD`
   - Get the diff: `git diff <base>...HEAD`
   - If no commits exist, check for uncommitted changes with `git status --short` and `git diff`

3. **Read the PR template** at `.github/pull_request_template.md` if it exists

4. **Analyze the changes:**
   - Extract Linear tickets from commit messages (patterns like `TUN-123`, `TEN-456`)
   - Understand what the code actually does, not just what files changed
   - Identify breaking changes or migration requirements

5. **Generate the PR description:**

```markdown
# Title
<imperative title with conventional commit prefix: feat:, fix:, refactor:, docs:, chore:>

# Body

## Summary
<2-4 bullet points explaining what changed and why>

Closes TUN-XXX (only if found)

## Testing Done
<specific testing performed or suggested>
```

6. **Guidelines:**
   - **Title:** Concise, imperative mood, 50-72 chars
   - **Summary:** Focus on the "why" and impact, not just listing files
   - **Testing Done:** Be specific about what was tested
   - **Omit empty sections:** No "Details", "Screenshots", or "Additional information" sections unless there's actual content
   - **No filler:** Don't include HTML comments, placeholder text, or unchecked checkbox lists

7. **Output** the PR description in a markdown code block for easy copying.
