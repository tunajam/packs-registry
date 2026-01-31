# GitHub Templates Context

Comprehensive guide to GitHub repository templates for issues, PRs, and community guidelines.

## Directory Structure

```
.github/
├── ISSUE_TEMPLATE/
│   ├── bug_report.yml
│   ├── feature_request.yml
│   └── config.yml
├── PULL_REQUEST_TEMPLATE.md
├── CODEOWNERS
├── FUNDING.yml
├── dependabot.yml
├── workflows/
│   └── ci.yml
└── SECURITY.md

# Root level
CONTRIBUTING.md
CODE_OF_CONDUCT.md
LICENSE
README.md
```

## Issue Templates (YAML Forms)

### Bug Report
```yaml
# .github/ISSUE_TEMPLATE/bug_report.yml
name: 🐛 Bug Report
description: Report a bug or unexpected behavior
title: "[Bug]: "
labels: ["bug", "triage"]
assignees: []

body:
  - type: markdown
    attributes:
      value: |
        Thanks for taking the time to report a bug!
        Please fill out the form below.

  - type: textarea
    id: description
    attributes:
      label: Bug Description
      description: A clear description of what the bug is.
      placeholder: What happened?
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Steps to Reproduce
      description: Steps to reproduce the behavior.
      placeholder: |
        1. Go to '...'
        2. Click on '....'
        3. Scroll down to '....'
        4. See error
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected Behavior
      description: What did you expect to happen?
    validations:
      required: true

  - type: textarea
    id: screenshots
    attributes:
      label: Screenshots
      description: If applicable, add screenshots to help explain your problem.

  - type: dropdown
    id: os
    attributes:
      label: Operating System
      options:
        - macOS
        - Windows
        - Linux
        - iOS
        - Android
    validations:
      required: true

  - type: dropdown
    id: browser
    attributes:
      label: Browser
      options:
        - Chrome
        - Firefox
        - Safari
        - Edge
        - Other

  - type: input
    id: version
    attributes:
      label: Version
      description: What version of the software are you running?
      placeholder: "e.g., 1.0.0"

  - type: textarea
    id: additional
    attributes:
      label: Additional Context
      description: Any other context about the problem.

  - type: checkboxes
    id: terms
    attributes:
      label: Checklist
      options:
        - label: I have searched for existing issues
          required: true
        - label: I have read the documentation
          required: true
```

### Feature Request
```yaml
# .github/ISSUE_TEMPLATE/feature_request.yml
name: ✨ Feature Request
description: Suggest a new feature or enhancement
title: "[Feature]: "
labels: ["enhancement"]

body:
  - type: textarea
    id: problem
    attributes:
      label: Problem Statement
      description: What problem does this feature solve?
      placeholder: I'm frustrated when...
    validations:
      required: true

  - type: textarea
    id: solution
    attributes:
      label: Proposed Solution
      description: Describe the solution you'd like.
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives Considered
      description: Have you considered any alternative solutions?

  - type: dropdown
    id: impact
    attributes:
      label: Impact
      description: How important is this feature to you?
      options:
        - Nice to have
        - Would be helpful
        - Important
        - Critical
    validations:
      required: true

  - type: textarea
    id: context
    attributes:
      label: Additional Context
      description: Any other context, mockups, or screenshots.
```

### Issue Template Config
```yaml
# .github/ISSUE_TEMPLATE/config.yml
blank_issues_enabled: false
contact_links:
  - name: 💬 Discussions
    url: https://github.com/org/repo/discussions
    about: Use discussions for questions and general help
  - name: 📖 Documentation
    url: https://docs.example.com
    about: Check our documentation first
  - name: 💻 Discord
    url: https://discord.gg/example
    about: Join our Discord community
```

## Pull Request Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->
## Description

<!-- Describe your changes in detail -->

## Type of Change

<!-- Mark relevant options with [x] -->

- [ ] 🐛 Bug fix (non-breaking change that fixes an issue)
- [ ] ✨ New feature (non-breaking change that adds functionality)
- [ ] 💥 Breaking change (fix or feature that would cause existing functionality to change)
- [ ] 📝 Documentation update
- [ ] 🎨 Style/formatting (no functional changes)
- [ ] ♻️ Refactoring (no functional changes)
- [ ] 🧪 Test update
- [ ] 🔧 Configuration change

## Related Issues

<!-- Link to related issues: Fixes #123, Closes #456 -->

## Screenshots (if applicable)

<!-- Add screenshots to help explain your changes -->

## Testing

<!-- Describe how you tested your changes -->

- [ ] Unit tests pass locally
- [ ] Integration tests pass locally
- [ ] Manual testing completed

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or feature works
- [ ] New and existing unit tests pass locally with my changes
```

## CODEOWNERS

```
# .github/CODEOWNERS

# Default owners for everything
* @org/maintainers

# Specific paths
/docs/ @org/docs-team
/packages/api/ @org/backend-team
/packages/web/ @org/frontend-team

# Specific files
package.json @org/maintainers
*.yml @org/devops

# Specific users for critical files
/security/ @security-lead @org/maintainers
```

## Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2

updates:
  # npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "America/New_York"
    open-pull-requests-limit: 10
    groups:
      development-dependencies:
        dependency-type: "development"
        patterns:
          - "*"
      production-dependencies:
        dependency-type: "production"
    ignore:
      - dependency-name: "typescript"
        update-types: ["version-update:semver-major"]
    labels:
      - "dependencies"
      - "automated"
    commit-message:
      prefix: "deps"
      include: "scope"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "ci"
      - "automated"
    commit-message:
      prefix: "ci"
```

## Funding Configuration

```yaml
# .github/FUNDING.yml
github: [username]
patreon: username
open_collective: project-name
ko_fi: username
custom: ["https://example.com/donate"]
```

## Security Policy

```markdown
<!-- SECURITY.md -->
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 2.x.x   | :white_check_mark: |
| 1.x.x   | :x:                |

## Reporting a Vulnerability

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, please report them via email to security@example.com.

You should receive a response within 48 hours. If for some reason you do not, please follow up via email to ensure we received your original message.

Please include:

- Type of issue (e.g., buffer overflow, SQL injection, cross-site scripting)
- Full paths of source file(s) related to the issue
- Location of the affected source code (tag/branch/commit or direct URL)
- Any special configuration required to reproduce the issue
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact of the issue, including how an attacker might exploit it

## Preferred Languages

We prefer all communications to be in English.
```

## Contributing Guide

```markdown
<!-- CONTRIBUTING.md -->
# Contributing Guide

Thank you for your interest in contributing! 🎉

## Getting Started

1. Fork the repository
2. Clone your fork: `git clone https://github.com/YOUR_USERNAME/REPO_NAME.git`
3. Create a branch: `git checkout -b feat/your-feature`
4. Install dependencies: `pnpm install`
5. Make your changes
6. Run tests: `pnpm test`
7. Commit your changes (see commit guidelines below)
8. Push to your fork: `git push origin feat/your-feature`
9. Open a Pull Request

## Commit Guidelines

We use [Conventional Commits](https://www.conventionalcommits.org/):

- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `style:` - Formatting, no code change
- `refactor:` - Code change that neither fixes nor adds
- `test:` - Adding or updating tests
- `chore:` - Maintenance

Examples:
```
feat: add user authentication
fix: resolve login timeout issue
docs: update README installation steps
```

## Code Style

- Run `pnpm lint` before committing
- Run `pnpm format` to format code
- Follow existing code patterns

## Pull Request Process

1. Update documentation if needed
2. Add tests for new features
3. Ensure all tests pass
4. Request review from maintainers

## Questions?

Open a [Discussion](https://github.com/org/repo/discussions) or join our [Discord](https://discord.gg/example).
```

## Repository Labels

Recommended labels to create:

```
bug           - #d73a4a - Something isn't working
enhancement   - #a2eeef - New feature or request
documentation - #0075ca - Improvements to docs
good first issue - #7057ff - Good for newcomers
help wanted   - #008672 - Extra attention needed
duplicate     - #cfd3d7 - This issue already exists
wontfix       - #ffffff - This will not be worked on
triage        - #fbca04 - Needs investigation
breaking      - #b60205 - Breaking change
dependencies  - #0366d6 - Dependency updates
automated     - #ededed - Automated PRs
```

## Best Practices

1. **Use YAML forms** - Better UX than markdown templates
2. **Require essential fields** - Don't let incomplete issues through
3. **Link to resources** - Docs, Discord, discussions
4. **Set up CODEOWNERS** - Automatic review assignment
5. **Configure Dependabot** - Keep dependencies updated
6. **Add security policy** - Clear vulnerability reporting process
7. **Write contributing guide** - Lower barrier to entry
