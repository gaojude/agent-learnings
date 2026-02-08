# Merging Stacked PRs

**Date:** 2026-02-07
**Project:** agent-eval (vercel-labs/agent-eval)

## The Problem

When you create stacked PRs (A → B → C → D → E), each branch contains the commits from all previous branches. If PR A gets **squash-merged** into main, the commit SHA changes. PR B still has A's original commits, so GitHub shows A's changes in B's diff even after updating B's base to main.

## The Solution: Rebase the Chain

After merging each PR, rebase the next branch onto the new main (or the newly rebased parent), dropping the old parent's commits.

### Step-by-step

```bash
# 1. Merge PR A into main (via GitHub UI)

# 2. Fetch updated main
git fetch origin main

# 3. Rebase B onto main, dropping A's old commits
git checkout branch-B
git rebase --onto origin/main branch-A
git push --force-with-lease origin branch-B

# 4. Rebase C onto new B, dropping B's old commits
git checkout branch-C
git rebase --onto branch-B <old-C-commit>^ <old-C-commit>
git checkout -B branch-C
git push --force-with-lease origin branch-C

# 5. Repeat for D, E, etc.
```

### Key details

- `git rebase --onto <new-base> <old-base>` — replays only the commits unique to the current branch onto the new base
- Use `--force-with-lease` (not `--force`) for safety when pushing rebased branches
- After rebasing, each PR will show only its own changes in the GitHub diff
- GitHub auto-updates PR base branches when the target is merged, but the **commit content** still needs rebasing

### Alternative: Use commit refs directly

If branches have been deleted or you need more precision:

```bash
# Rebase only the single commit that belongs to this PR
git rebase --onto new-parent <commit>^ <commit>
git checkout -B branch-name
```

This cherry-picks exactly one commit onto the new parent, which is useful when each PR is a single commit.

## When to Do This

- Every time you merge a PR in a stacked chain
- Especially after **squash merges** (which create new commit SHAs)
- Not needed if using regular merge commits AND the base branch hasn't diverged
