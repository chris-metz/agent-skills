---
name: uncommit
description: Undo the last commit, keeping changes as unstaged in the working directory
disable-model-invocation: true
---

# Uncommit

Undo the last commit on the current branch, leaving the changed files as unstaged modifications in the working directory.

## Instructions

1. Run `git log --oneline -1` to show the user which commit will be undone.
2. Run `git reset HEAD~1` to undo the last commit (mixed reset — changes become unstaged).
3. Run `git status` to show the resulting working directory state.
4. Report the result to the user.
