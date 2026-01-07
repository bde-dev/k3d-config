---
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git add:*), Bash(git commit:*), Bash(git log:*)
description: Stage changes and create a conventional commit
---

Review the current changes and create a git commit following the project's conventional commits format.

## Current State
- Git status: !`git status`
- Unstaged changes: !`git diff`
- Recent commits for style reference: !`git log --oneline -5`

## Instructions

1. Review the changes shown above
2. Stage all relevant changes with `git add`
3. Generate a commit message following Conventional Commits format:
   - Use appropriate type: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert
   - Include scope if changes are focused on a specific component
   - Write a concise description
4. Create the commit

Do NOT push to remote.
