---
description: Merge a zap fix branch to main and clean up worktrees and branches
argument-hint: [branch-name]
allowed-tools: Bash(git:*), Read
model: haiku
---

# Zap Cleanup

Merge a zap fix branch to main, then remove worktrees and local branches created during the zap.

**Argument**: $ARGUMENTS (branch name, or empty to auto-detect)

## Step 1: Identify the fix branch

If a branch name is given, use it directly.

If no argument given, detect from current branch:
```bash
BRANCH_NAME=$(git branch --show-current)
```

If the branch is `main`, list recent branches matching `fix/*` or `worktree-agent-*` and ask the user which to merge.

## Step 2: Merge to main

```bash
git checkout main
git merge $BRANCH_NAME --no-ff
```

If merge fails (e.g., conflicts), report the error and stop.

## Step 3: Clean up worktrees

List all worktrees and remove any created by zap agents:

```bash
git worktree list
```

Remove worktrees on `worktree-agent-*` branches:

```bash
# For each worktree-agent-* worktree found:
git worktree remove $WORKTREE_PATH --force
```

Prune stale references:
```bash
git worktree prune
```

## Step 4: Clean up branches

Delete the fix branch and any `worktree-agent-*` branches:

```bash
# Delete the fix branch
git branch -d $BRANCH_NAME 2>/dev/null || true

# Delete all worktree-agent-* branches
git branch | grep 'worktree-agent-' | xargs -r git branch -D
```

## Step 5: Verify

```bash
git worktree list
git branch | grep -E '(worktree-agent-|fix/)' || echo "All zap branches cleaned up"
git log --oneline -3
```

## Output

Report:
- Merge status (merged / failed)
- Number of worktrees removed
- Number of branches deleted

If any step fails, report what succeeded and what didn't so the user can finish manually.
