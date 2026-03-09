# Git Merge and Rebase — Deep Dive

**Related:** [Git Fundamentals](git_fundamentals.md) | [Merge Conflicts](git_merge_conflicts.md) | [Undo and Revert](git_undo_and_revert.md) | [Stash](git_stash.md) | [GitHub Best Practices](github_best_practices.md)

---

## git fetch — What Actually Happens

Before understanding merge and rebase, you need to understand `git fetch` because `git pull` depends on it.

```bash
git fetch origin
```

**What's happening step by step:**

1. Git contacts the remote (`origin`) and asks: "What commits do you have that I don't?"
2. Downloads any new **objects** (blobs, trees, commits) from the remote
3. Updates **remote tracking branches** (`refs/remotes/origin/*`) to match the remote
4. Does NOT touch your working directory, staging area, or local branches

```
Before fetch:                          After fetch:

Local:                                 Local:
  main ──→ c3                            main ──→ c3  (unchanged!)
  origin/main ──→ c3                     origin/main ──→ c5  (updated!)

Remote:                                  (new objects c4, c5 downloaded
  main ──→ c5                             and stored in .git/objects/)
```

**How to think about it:** `git fetch` is always safe. It downloads data and updates bookmarks. It never changes your code.

```bash
# Fetch and see what changed
git fetch origin
git log main..origin/main --oneline   # commits on remote that you don't have locally
git log origin/main..main --oneline   # commits you have locally that remote doesn't
```

---

## git pull = git fetch + git merge

```bash
git pull origin main
# is equivalent to:
git fetch origin
git merge origin/main
```

**What's happening:**

1. `fetch` downloads new objects and updates `origin/main`
2. `merge` integrates `origin/main` into your current branch

The merge step is where it gets interesting — Git chooses between a **fast-forward** or a **three-way merge** depending on the commit graph.

---

## Fast-Forward Merge

A fast-forward merge happens when the current branch has NO new commits since the branches diverged — the branch pointer just moves forward.

```
Before: (main has no new commits since feature branched off)

    main
      │
      ▼
c1 ← c2 ← c3 ← c4
                  ▲
                  │
               feature


After: git checkout main && git merge feature

                        main
                          │
                          ▼
c1 ← c2 ← c3 ← c4
                  ▲
                  │
               feature
```

**What's happening:** Git just moves the `main` pointer from `c2` to `c4`. No new commit is created. The history is perfectly linear.

```bash
git checkout main
git merge feature
# Updating a1b2c3d..d4e5f6a
# Fast-forward
#  src/api.go | 42 +++++++
#  2 files changed, 42 insertions(+)
```

**Force a merge commit even when fast-forward is possible:**

```bash
git merge --no-ff feature
# Creates a merge commit even though fast-forward was possible
# Useful when you want to preserve the fact that a feature branch existed
```

---

## Three-Way Merge

A three-way merge is needed when BOTH branches have new commits since they diverged.

```
Before: (both main and feature have new commits)

         c5 ← c6        ← main
        /
c1 ← c2
        \
         c3 ← c4        ← feature


After: git checkout main && git merge feature

         c5 ← c6 ← c7   ← main (c7 is the merge commit)
        /          /
c1 ← c2          /
        \        /
         c3 ← c4         ← feature
```

**What's happening:**

1. Git finds the **common ancestor** (also called merge base): `c2`
2. Git computes three sets of changes:
   - Changes from `c2` → `c6` (what main did)
   - Changes from `c2` → `c4` (what feature did)
3. Git combines both sets of changes into a new **merge commit** (`c7`)
4. The merge commit has TWO parents: `c6` and `c4`

```bash
git checkout main
git merge feature
# Merge made by the 'ort' strategy.   (ort = Ostensibly Recursive's Twin)
#  src/handler.go | 15 ++++++
#  src/model.go   |  8 ++++
#  2 files changed, 23 insertions(+)
```

**Finding the merge base manually:**

```bash
git merge-base main feature
# c2a3b4c5...   (the common ancestor)
```

If both branches modified the SAME lines in the SAME file, you get a **merge conflict**. See [Merge Conflicts](git_merge_conflicts.md).

---

## git rebase — Replay Commits on a New Base

Rebase takes your commits and replays them one-by-one on top of another branch. The commits get **new SHA hashes** because their parent changed.

```
Before: git checkout feature && git rebase main

         c5 ← c6              ← main
        /
c1 ← c2
        \
         c3 ← c4              ← feature


After rebase:

                   c3' ← c4'   ← feature  (NEW hashes! c3' != c3)
                  /
c1 ← c2 ← c5 ← c6             ← main
```

**What's happening step by step:**

1. Git finds the common ancestor of `feature` and `main` (`c2`)
2. Git computes the diffs introduced by each feature commit (`c3`, `c4`)
3. Git resets `feature` to point to the tip of `main` (`c6`)
4. Git replays each diff on top, creating NEW commits (`c3'`, `c4'`)
5. The old commits (`c3`, `c4`) are now unreferenced (still exist until garbage collected)

```bash
git checkout feature
git rebase main
# Successfully rebased and updated refs/heads/feature.
```

**Critical:** Commits `c3'` and `c4'` have DIFFERENT SHA hashes than `c3` and `c4` because their parent commit changed. The content of the changes is the same, but the commit identity is different.

---

## git pull --rebase = git fetch + git rebase

```bash
git pull --rebase origin main
# is equivalent to:
git fetch origin
git rebase origin/main
```

**When to use:** When you want to put your local commits on top of the latest remote changes, keeping a linear history.

```
Before pull --rebase:

Local:   c1 ← c2 ← c5 ← c6    ← main  (your local commits)
Remote:  c1 ← c2 ← c3 ← c4    ← origin/main

After pull --rebase:

c1 ← c2 ← c3 ← c4 ← c5' ← c6'   ← main
                 ▲
                 origin/main

Your commits (c5, c6) replayed on top of remote commits (c3, c4)
```

---

## Interactive Rebase — Rewriting History

Interactive rebase lets you modify, reorder, squash, or drop commits before they're replayed.

```bash
git rebase -i HEAD~4    # interactively rebase last 4 commits
git rebase -i main      # interactively rebase all commits since diverging from main
```

This opens an editor with a "todo" list:

```
pick a1b2c3d Add user model
pick b2c3d4e Add user repository
pick c3d4e5f Fix typo in user model
pick d4e5f6a Add user service

# Commands:
# p, pick   = use commit as-is
# r, reword = use commit, but edit the commit message
# e, edit   = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit (keep both messages)
# f, fixup  = like squash, but discard this commit's message
# d, drop   = remove commit entirely
```

### Common Interactive Rebase Operations

**Squash a fixup commit into the commit it fixes:**

```
pick a1b2c3d Add user model
fixup c3d4e5f Fix typo in user model      ← moved up and changed to fixup
pick b2c3d4e Add user repository
pick d4e5f6a Add user service
```

Result: The typo fix is absorbed into "Add user model." Clean history.

**Reorder commits:**

```
pick b2c3d4e Add user repository
pick a1b2c3d Add user model               ← moved below repository
pick d4e5f6a Add user service
```

**Reword a commit message:**

```
reword a1b2c3d Add user model             ← will open editor for new message
pick b2c3d4e Add user repository
pick d4e5f6a Add user service
```

**Drop a commit entirely:**

```
pick a1b2c3d Add user model
drop b2c3d4e Add user repository           ← this commit will be removed
pick d4e5f6a Add user service
```

---

## git merge --squash — Collapse Into One Commit

Squash merge takes ALL commits from a branch and collapses them into a SINGLE set of changes, staged but NOT committed. No merge commit is created.

```bash
git checkout main
git merge --squash feature
git commit -m "Add user management feature"
```

```
Before:

         c5 ← c6                  ← main
        /
c1 ← c2
        \
         c3 ← c4                  ← feature (4 commits of work)


After merge --squash + commit:

c1 ← c2 ← c5 ← c6 ← c7          ← main  (c7 has ALL changes from c3+c4)
        \
         c3 ← c4                  ← feature (still exists, unchanged)
```

**What's happening:** `c7` is a regular commit (ONE parent: `c6`). It contains the combined changes of `c3` and `c4`. The branch history of `feature` is lost — the commit graph doesn't show the merge relationship.

**When to use:** When you want to keep `main` history very clean and don't care about preserving individual feature branch commits. Common in repositories that prefer a linear main branch history.

---

## Merge vs Rebase — When to Use Which

### The Golden Rule

> **Never rebase commits that have been pushed to a shared/public branch.**

Rebasing rewrites commit hashes. If someone else has based work on the original commits, rebasing will cause diverging histories and force everyone to reconcile.

```
SAFE to rebase:
  - Your local feature branch that hasn't been pushed
  - Your feature branch that only YOU work on (even if pushed for backup)

DANGEROUS to rebase:
  - main / develop / release branches
  - Any branch that other people have checked out or based work on
  - Commits that are part of an open PR others are reviewing
```

### Comparison

| Aspect | Merge | Rebase |
|---|---|---|
| History | Non-linear (preserves branch topology) | Linear (as if work happened sequentially) |
| Merge commits | Creates a merge commit with 2 parents | No merge commit — replays commits |
| Commit hashes | Original commits preserved | New hashes for replayed commits |
| Conflicts | Resolve once for the entire merge | Resolve per-commit during replay |
| Traceability | Can see when/where branches merged | Branch history is flattened |
| Safety | Always safe — never rewrites history | Dangerous on shared branches |
| `git bisect` | Merge commits can complicate bisect | Clean linear history is easier to bisect |
| PR workflow | `Merge commit` button on GitHub | `Rebase and merge` button on GitHub |

### Recommended Workflows

**Feature branch workflow (most common):**

```bash
# While developing on your feature branch — rebase to stay current
git checkout feature/user-api
git fetch origin
git rebase origin/main

# When feature is done — merge to main (or squash merge via PR)
git checkout main
git merge --no-ff feature/user-api   # preserves branch history in the graph
```

**Trunk-based development:**

```bash
# Short-lived branches, always rebase before merging
git checkout -b fix/cache-bug
# ... make changes ...
git fetch origin
git rebase origin/main
git checkout main
git merge --ff-only fix/cache-bug    # fast-forward only — ensures linear history
```

---

## Practical Examples / Real-World Scenarios

### Scenario 1: Update Feature Branch with Latest main

```bash
# Option A: Rebase (clean linear history)
git checkout feature/payments
git fetch origin
git rebase origin/main
# If conflicts occur, resolve them, then:
git add .
git rebase --continue

# Option B: Merge (preserves history, adds merge commit)
git checkout feature/payments
git fetch origin
git merge origin/main
```

### Scenario 2: Clean Up Commits Before Opening a PR

```bash
# You made 7 messy commits. Squash into 2 logical commits.
git rebase -i origin/main

# In the editor:
pick a1b2c3d Add payment model and migration
squash b2c3d4e Fix payment model field types
squash c3d4e5f Add missing index
pick d4e5f6a Add payment processing service
fixup e5f6a7b Fix lint errors
fixup f6a7b8c Fix test
fixup a7b8c9d Remove debug logging

# Result: 2 clean commits instead of 7
```

### Scenario 3: Rebase Went Wrong — Abort or Undo

```bash
# In the middle of a rebase with conflicts everywhere
git rebase --abort    # cancel — go back to before the rebase started

# Already finished the rebase but want to undo it
git reflog
# abc1234 HEAD@{0}: rebase (finish): ...
# def5678 HEAD@{1}: rebase (start): ...
# 789abcd HEAD@{2}: commit: My last commit before rebase

git reset --hard 789abcd    # go back to before the rebase
```

### Scenario 4: Squash Merge a Feature Branch via CLI

```bash
git checkout main
git pull origin main
git merge --squash feature/notifications
git commit -m "Add push notification system

- Implement FCM integration for Android/iOS
- Add notification preferences per user
- Add rate limiting for notification sends
- Closes #142"
git push origin main
git branch -d feature/notifications
```

### Scenario 5: Resolving Diverged Local and Remote

```bash
# You committed locally AND someone else pushed to the same branch
git pull
# hint: You have divergent branches and need to specify how to reconcile them.

# Option A: Merge (creates merge commit)
git pull --no-rebase

# Option B: Rebase (puts your commits on top)
git pull --rebase

# Set default behavior globally
git config --global pull.rebase true     # always rebase on pull
git config --global pull.rebase false    # always merge on pull (default)
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| git fetch | Downloads objects, updates remote tracking branches. Never touches your code. Always safe |
| git pull | fetch + merge (by default). Use `--rebase` for linear history |
| Fast-forward | Branch pointer moves forward. No merge commit. Only works when no divergence |
| Three-way merge | Finds common ancestor, combines changes, creates merge commit with 2 parents |
| Rebase | Replays commits on new base. Creates NEW hashes. Never rebase shared branches |
| Interactive rebase | squash, fixup, reword, edit, drop. Clean up history before PRs |
| merge --squash | Collapses all branch commits into one. No merge commit. Loses branch history |
| Golden rule | Never rebase commits that exist on shared/public branches |
| merge --no-ff | Forces merge commit even when fast-forward is possible. Preserves branch topology |
| pull.rebase | Set globally to control default `git pull` behavior |
