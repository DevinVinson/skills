---
name: use-git-worktrees
description: Use Git worktrees confidently for agent coding tasks. Use when you need to inspect whether the current checkout is a main or linked worktree, create or remove a worktree, diagnose branch switch errors such as "already checked out at", explain why one branch cannot be checked out by two worktrees at once, or help a user recover from worktree-related branch confusion.
---

# Use Git Worktrees

## Overview

Use worktrees as separate working directories that share one repository database. Each worktree has its own working tree, index, and `HEAD`, but local branch names are shared across the repository.

Remember the core rule: a local branch can be checked out in only one worktree at a time. If branch `feature-x` is active in `/repo-feature-x`, Git refuses to switch another worktree to `feature-x` because two independent indexes would otherwise try to move the same branch ref.

## Quick Diagnosis

Run these before explaining or changing anything:

```bash
pwd
git rev-parse --show-toplevel
git status --short --branch
git branch --show-current
git worktree list --porcelain
```

Interpret the output this way:

- `git worktree list --porcelain` is the authoritative map. Each stanza starts with `worktree <path>` and may include `branch refs/heads/<name>`.
- If the current branch command prints nothing, the worktree is probably detached. Confirm with `git rev-parse --short HEAD`.
- A linked worktree often has a `.git` file that points into the main repo's `.git/worktrees/...`; the main worktree usually has a `.git` directory. Treat `git worktree list --porcelain` as more reliable than file-shape guesses.
- Use `git -C <path> ...` to inspect another worktree without changing directories.

## Explain Branch Lockups

When Git says a branch is already checked out somewhere else, explain it plainly:

```text
You are in <current-path> on <current-branch>. Git will not switch this worktree to <target-branch> because <target-branch> is already checked out at <other-path>. That path currently owns the branch. Use that worktree, switch it away from the branch, or create a new branch from the same commit.
```

Do not describe this as a remote, merge, or permissions problem. It is local Git worktree branch ownership.

Common fixes:

- Work in the existing worktree: `git -C <other-path> status --short --branch`
- Switch the other worktree to a different branch if it is clean and the user wants to free the branch: `git -C <other-path> switch <other-branch>`
- Create a new branch from the target branch for parallel work: `git switch -c <new-branch> <target-branch>`
- Add a worktree on a new branch from a base: `git worktree add -b <new-branch> <new-path> <base-branch>`
- If a deleted worktree left stale metadata, inspect first with `git worktree list --porcelain`, then use `git worktree prune` only when the path is truly gone.

## Create Worktrees

Choose a path and branch deliberately:

```bash
git worktree add -b <new-branch> <new-path> <base-branch>
git worktree add <new-path> <existing-branch>
git worktree add --detach <new-path> <commit-or-tag>
```

Prefer `-b <new-branch> <base-branch>` for agent tasks so the user can keep their main checkout untouched. Use an existing branch only if `git worktree list --porcelain` shows that branch is not already checked out.

Before creating the worktree, check:

- The target path does not already contain unrelated user files.
- The branch name is not already checked out by another worktree.
- The base branch or commit is the intended starting point.
- The user has not asked you to reuse a branch that is already active elsewhere.

## Remove Or Release Worktrees

Before removing or freeing a worktree, inspect it:

```bash
git -C <path> status --short --branch
git worktree list --porcelain
```

Only remove a worktree when its changes are committed, stashed, or intentionally disposable:

```bash
git worktree remove <path>
git worktree prune
```

Use `git worktree remove --force`, `git branch -D`, or deletion of a worktree directory only after explicit user approval and only after explaining what uncommitted work may be lost.

## Agent Operating Habits

- Treat the user's main checkout as shared space. Prefer creating or using a task worktree instead of switching the user's active branch.
- Always report the current path and branch when worktree confusion is part of the task.
- Keep branch and worktree concepts separate: a worktree is a directory; a branch is a movable ref; a worktree can own a branch or be detached at a commit.
- If the user says they "cannot switch to my branch," look for another worktree holding that branch before trying merges, fetches, or resets.
- If you created a worktree for the task, tell the user its path and branch, and leave cleanup instructions when relevant.
