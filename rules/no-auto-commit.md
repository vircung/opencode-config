# Git Commit Policy

## Critical Rule: No Automatic Commits

You must NEVER create git commits automatically. This is a strict policy that overrides all other instructions.

## When the user asks to commit changes:

1. **Stage files** if needed using `git add`
2. **Show status** using `git status` so the user can review
3. **Inform the user** that changes are ready to commit
4. **Ask the user** to manually run the commit command themselves

## Forbidden Commands:

- `git commit` (in ANY form - with -m, -am, --amend, etc.)
- `git commit -m "message"`
- `git commit --amend`
- `git commit -a`
- Any other variant of git commit

## Allowed Git Commands:

- `git status` - to check repository state
- `git add` - to stage files
- `git diff` - to show changes
- `git log` - to view history
- `git branch` - to manage branches
- `git checkout` / `git switch` - to change branches
- `git pull` / `git fetch` - to sync with remote
- Any other git command EXCEPT commit

## Example Workflow:

When user says "commit these changes":

```bash
# ✅ Allowed:
git add file1.py file2.js
git status

# ❌ FORBIDDEN:
git commit -m "Update files"  # DO NOT DO THIS
```

Then respond: "Files are staged and ready. Please run `git commit -m "your message"` to commit the changes."

## Summary

- NEVER execute `git commit` in any form
- Always stage files and show status
- Always inform the user to commit manually
- This rule applies to ALL projects and sessions
