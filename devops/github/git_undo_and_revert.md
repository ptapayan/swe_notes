# Git Undo and Revert — How to Undo Things Safely

**Related:** [Git Fundamentals](git_fundamentals.md) | [Merge and Rebase](git_merge_and_rebase.md) | [Merge Conflicts](git_merge_conflicts.md) | [Stash](git_stash.md)

---

## Overview — The Undo Landscape

Git has many ways to undo things. The critical distinction is whether the operation **rewrites history** (dangerous on shared branches) or **creates new history** (safe everywhere).

```
┌───────────────────────────────────────────────────────────────────┐
│                    Safe (adds new history)                         │
│  git revert     — creates a NEW commit that undoes a previous one │
│  git stash      — saves changes without committing                │
│  git checkout/restore — discard working directory changes          │
│                                                                   │
│                    Rewrites History (local only!)                  │
│  git reset      — moves HEAD backward, optionally discards work   │
│  git commit --amend — replaces the last commit                    │
│  git rebase -i  — rewrite/reorder/squash commits                  │
│  git cherry-pick — copy a commit (creates new hash)               │
└───────────────────────────────────────────────────────────────────┘
```

---

## git revert — Safely Undo a Commit

`git revert` creates a **NEW commit** that applies the inverse of a previous commit. It does NOT remove the original commit from history.

```bash
git revert abc1234
```

```
Before:

c1 ← c2 ← c3 ← c4    ← HEAD (main)
          ^^^^
          this commit introduced a bug

After git revert c3:

c1 ← c2 ← c3 ← c4 ← c5    ← HEAD (main)
                       ^^
                       c5 is the "anti-c3" — undoes c3's changes
```

**What's happening:** Git computes the diff that `c3` introduced, reverses it, and applies it as a new commit. The original `c3` still exists in history — `c5` is an additive fix.

```bash
# Revert the most recent commit
git revert HEAD

# Revert a specific commit
git revert abc1234

# Revert without auto-committing (stage the changes, let you edit)
git revert --no-commit abc1234
git status                        # see the staged reversal
git commit -m "Revert: remove broken cache logic from abc1234"

# Revert a range of commits
git revert abc1234..def5678       # reverts commits AFTER abc1234 through def5678
```

**Why revert is safe:** It only ADDS a new commit. It never changes existing commits. Everyone who has pulled the original commit will get the revert commit on their next pull. No history rewriting.

### Reverting a Merge Commit

Merge commits have TWO parents. You must tell Git which parent to revert against:

```
         c5 ← c6 ← M    ← HEAD (main)   M = merge commit
        /          /
c1 ← c2          /
        \        /
         c3 ← c4         ← feature

M has two parents:
  parent 1 (main line):    c6
  parent 2 (feature line): c4
```

```bash
# Revert the merge, keeping parent 1 (main line — undo the feature)
git revert -m 1 <merge-commit-hash>

# -m 1 = "keep parent 1's history, undo what parent 2 brought in"
# -m 2 = "keep parent 2's history, undo what parent 1 brought in" (rare)
```

**Important gotcha:** After reverting a merge, if you try to merge the feature branch again later, Git thinks those changes are already integrated (because the merge commit is still in history). To re-merge, you must revert the revert:

```bash
git revert <revert-commit-hash>    # revert the revert — re-introduces the changes
git merge feature                   # now this works properly
```

---

## git reset — Move HEAD Backward

`git reset` moves the HEAD (and current branch pointer) to a different commit. It has three modes that differ in what they do to the staging area and working directory.

### The Three Modes of git reset

```
                        Working    Staging    HEAD
                        Directory  Area       (Branch)
                        ─────────  ─────────  ─────────
git reset --soft  c2    Unchanged  Unchanged  Moved to c2
git reset --mixed c2    Unchanged  Reset      Moved to c2    ← DEFAULT
git reset --hard  c2    Reset      Reset      Moved to c2    ← DANGEROUS
```

### --soft (Move HEAD Only)

```bash
git reset --soft HEAD~1
```

```
Before:
  c1 ← c2 ← c3    ← HEAD (main)
  Working dir: clean
  Index: matches c3

After --soft HEAD~1:
  c1 ← c2    ← HEAD (main)    c3 still exists but unreferenced
  Working dir: unchanged (still has c3's files)
  Index: unchanged (c3's changes are STAGED, ready to commit)
```

**When to use:** You want to undo the commit but keep all changes staged. Perfect for:
- Rewording a commit message: `git reset --soft HEAD~1 && git commit -m "Better message"`
- Combining the last N commits: `git reset --soft HEAD~3 && git commit -m "Combined"`

### --mixed (Move HEAD + Unstage) — THE DEFAULT

```bash
git reset HEAD~1       # --mixed is the default
git reset --mixed HEAD~1
```

```
Before:
  c1 ← c2 ← c3    ← HEAD (main)
  Working dir: clean
  Index: matches c3

After --mixed HEAD~1:
  c1 ← c2    ← HEAD (main)
  Working dir: unchanged (still has c3's files)
  Index: reset to match c2 (c3's changes are UNSTAGED)
```

**When to use:** You want to undo the commit AND unstage the changes, but keep the files. Good for re-doing `git add` with different groupings.

### --hard (Move HEAD + Unstage + Discard Working Directory) — DANGEROUS

```bash
git reset --hard HEAD~1
```

```
Before:
  c1 ← c2 ← c3    ← HEAD (main)
  Working dir: clean
  Index: matches c3

After --hard HEAD~1:
  c1 ← c2    ← HEAD (main)
  Working dir: RESET to match c2 (c3's file changes are GONE)
  Index: RESET to match c2
```

**DANGER:** `--hard` permanently discards uncommitted changes in your working directory. If you had unstaged or staged modifications, they are lost (not recoverable from Git, only from your editor's undo or file recovery).

However, the COMMIT `c3` still exists in the object database and can be recovered via reflog — see below.

### Reset Cheat Sheet

```
┌───────────────────────┬──────────┬──────────┬──────────────────┐
│ Scenario              │ Command  │ Index    │ Working Dir      │
├───────────────────────┼──────────┼──────────┼──────────────────┤
│ Undo commit,          │ --soft   │ Kept     │ Kept             │
│ keep changes staged   │          │ (staged) │                  │
├───────────────────────┼──────────┼──────────┼──────────────────┤
│ Undo commit,          │ --mixed  │ Reset    │ Kept             │
│ keep changes unstaged │(default) │(unstaged)│                  │
├───────────────────────┼──────────┼──────────┼──────────────────┤
│ Undo commit,          │ --hard   │ Reset    │ Reset            │
│ DISCARD all changes   │          │          │ (changes GONE)   │
└───────────────────────┴──────────┴──────────┴──────────────────┘
```

---

## Discarding Working Directory Changes (Per-File)

### git restore — The Modern Way (Git 2.23+)

```bash
# Discard working directory changes for a specific file
git restore src/handler.go

# Discard ALL working directory changes
git restore .

# Unstage a file (move from staging area back to working directory)
git restore --staged src/handler.go

# Unstage ALL files
git restore --staged .

# Restore a file to a specific commit's version
git restore --source=abc1234 src/handler.go
```

### git checkout -- (The Old Way)

```bash
# Discard working directory changes (old syntax)
git checkout -- src/handler.go

# Discard all changes
git checkout -- .
```

**What's happening:** Both `git restore` and `git checkout --` replace the working directory version of a file with the version from the index (staging area) or from a specific commit.

```
git restore file.go:
  Index version ──→ overwrites ──→ Working directory version

git restore --staged file.go:
  HEAD version ──→ overwrites ──→ Index (staging area)
  Working directory is UNTOUCHED
```

---

## Reflog — Git's Safety Net

The reflog records every time HEAD moves — every commit, merge, rebase, reset, checkout. It's your undo history for Git itself.

```bash
git reflog
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: Add rate limiting
# 789abcd HEAD@{2}: commit: Add auth middleware
# bcd1234 HEAD@{3}: commit: Add user model
# cde5678 HEAD@{4}: checkout: moving from feature to main
# ...
```

**How to think about it:** Even after `git reset --hard`, the "lost" commits still exist in the object database for at least 30 days (until garbage collection). The reflog tells you their hashes.

### Recovering "Lost" Commits

```bash
# You accidentally reset --hard and lost 3 commits
git reset --hard HEAD~3     # oops!

# Find what you lost
git reflog
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: Add rate limiting      ← this was HEAD before reset

# Recover by resetting to the old HEAD
git reset --hard def5678    # go back to where you were

# Or create a new branch from the lost commit
git branch recovery def5678
```

### Recovering After a Bad Rebase

```bash
git rebase main
# ... rebase creates a mess ...

# Find the pre-rebase state
git reflog
# ... HEAD@{0}: rebase (finish): ...
# ... HEAD@{1}: rebase (start): checkout main
# abc1234 HEAD@{2}: commit: Your last commit before rebase

# Go back to pre-rebase state
git reset --hard abc1234
```

### Reflog Expiration

```bash
# Reflog entries expire after 90 days by default (30 days for unreachable commits)
git config gc.reflogExpire            # default: 90 days
git config gc.reflogExpireUnreachable # default: 30 days

# View reflog with timestamps
git reflog --date=relative
# abc1234 HEAD@{2 hours ago}: commit: Add rate limiting
```

---

## git cherry-pick — Apply a Specific Commit

Cherry-pick takes a commit from one branch and applies it to your current branch as a NEW commit (new hash).

```bash
git cherry-pick abc1234
```

```
Before:

main:     c1 ← c2 ← c5 ← c6    ← HEAD
                \
feature:         c3 ← c4 ← c7
                       ^^
                       want this commit on main

After git cherry-pick c4:

main:     c1 ← c2 ← c5 ← c6 ← c4'    ← HEAD  (c4' is a copy, new hash)
                \
feature:         c3 ← c4 ← c7                   (c4 unchanged)
```

```bash
# Cherry-pick a single commit
git cherry-pick abc1234

# Cherry-pick multiple commits
git cherry-pick abc1234 def5678

# Cherry-pick a range (exclusive of start, inclusive of end)
git cherry-pick abc1234..def5678

# Cherry-pick without auto-committing
git cherry-pick --no-commit abc1234
# Changes are staged, you can modify them before committing

# If there's a conflict during cherry-pick
git cherry-pick --continue    # after resolving
git cherry-pick --abort       # cancel the cherry-pick
```

**When to use:**
- Backporting a bugfix from main to a release branch
- Grabbing a specific commit from a feature branch without merging everything
- Recovering a specific commit from a deleted branch

---

## git commit --amend — Rewrite the Last Commit

```bash
# Change the last commit's message
git commit --amend -m "Better commit message"

# Add forgotten files to the last commit
git add forgotten-file.go
git commit --amend --no-edit    # keeps the same message
```

```
Before:

c1 ← c2 ← c3    ← HEAD (main)

After amend:

c1 ← c2 ← c3'   ← HEAD (main)   (c3' has a NEW hash)
       \
        c3  (old commit, now unreferenced — recoverable via reflog)
```

**Critical:** `--amend` creates a new commit with a new hash. It rewrites history. Only use on commits that haven't been pushed to shared branches.

---

## Decision Tree — "I Need to Undo X"

```
I need to undo something...
│
├─ "I want to undo a PUSHED commit (shared history)"
│   └─→ git revert <commit>
│       (creates new commit, safe for shared branches)
│
├─ "I want to undo my LAST commit (not pushed)"
│   ├─ Keep changes staged     → git reset --soft HEAD~1
│   ├─ Keep changes unstaged   → git reset --mixed HEAD~1
│   └─ Discard changes entirely→ git reset --hard HEAD~1
│
├─ "I want to undo changes in my working directory"
│   ├─ Specific file           → git restore <file>
│   └─ All files               → git restore .
│
├─ "I want to unstage a file (undo git add)"
│   └─→ git restore --staged <file>
│
├─ "I want to change my last commit message"
│   └─→ git commit --amend -m "new message"
│
├─ "I want to add a file to my last commit"
│   └─→ git add <file> && git commit --amend --no-edit
│
├─ "I want to undo a merge (not pushed)"
│   └─→ git reset --hard ORIG_HEAD
│       (ORIG_HEAD is set before merge/rebase/reset)
│
├─ "I want to undo a merge (already pushed)"
│   └─→ git revert -m 1 <merge-commit>
│
├─ "I want to undo a rebase"
│   └─→ git reflog → find pre-rebase commit → git reset --hard <hash>
│
├─ "I want to recover a deleted branch"
│   └─→ git reflog → find last commit → git branch <name> <hash>
│
├─ "I want to recover after git reset --hard"
│   └─→ git reflog → git reset --hard <lost-commit-hash>
│
└─ "I want a specific commit from another branch"
    └─→ git cherry-pick <commit>
```

---

## Safety Rules — What's Safe on Shared vs Local Branches

```
┌─────────────────────────────┬─────────────┬──────────────────────┐
│ Operation                   │ Local Only  │ Shared/Pushed Branch │
├─────────────────────────────┼─────────────┼──────────────────────┤
│ git revert                  │ ✅ Safe      │ ✅ Safe               │
│ git stash                   │ ✅ Safe      │ ✅ Safe               │
│ git restore / checkout --   │ ✅ Safe      │ ✅ Safe               │
│ git reset --soft/mixed      │ ✅ Safe      │ ❌ Rewrites history   │
│ git reset --hard            │ ⚠️  Careful  │ ❌ Rewrites history   │
│ git commit --amend          │ ✅ Safe      │ ❌ Rewrites history   │
│ git rebase                  │ ✅ Safe      │ ❌ Rewrites history   │
│ git rebase -i               │ ✅ Safe      │ ❌ Rewrites history   │
│ git cherry-pick             │ ✅ Safe      │ ✅ Safe               │
│ git push --force            │ N/A         │ ❌ Very dangerous     │
│ git push --force-with-lease │ N/A         │ ⚠️  Safer force push  │
└─────────────────────────────┴─────────────┴──────────────────────┘
```

**`--force-with-lease`:** Only force-pushes if the remote branch is where you think it is. Prevents accidentally overwriting someone else's commits:

```bash
# Instead of:
git push --force            # DANGEROUS: overwrites remote unconditionally

# Use:
git push --force-with-lease # SAFER: fails if someone else pushed since your last fetch
```

---

## Practical Examples / Real-World Scenarios

### Scenario 1: Undo the Last 3 Commits but Keep the Code

```bash
# Squash last 3 commits into one
git reset --soft HEAD~3
git commit -m "Add complete user management feature"

# All changes from the 3 commits are now in one commit
```

### Scenario 2: Remove a Secret That Was Accidentally Committed

```bash
# If NOT pushed yet — amend or reset
git reset --soft HEAD~1
# Remove the secret from the file
git add .
git commit -m "Add config (without secret)"

# If ALREADY pushed — you MUST rotate the secret
# Then remove from history (advanced — rewrite all history):
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/secret-file' \
  --prune-empty -- --all
# Or use the BFG Repo-Cleaner (faster):
# bfg --delete-files secret-file.env
# Then force push (coordinate with team!)
git push --force-with-lease --all
```

### Scenario 3: Backport a Fix to a Release Branch

```bash
# The fix is on main at commit abc1234
git checkout release/v2.1
git cherry-pick abc1234

# If it conflicts:
# resolve conflicts...
git add .
git cherry-pick --continue
```

### Scenario 4: Recover After Accidentally Resetting the Wrong Branch

```bash
# Oops, did git reset --hard on the wrong branch
git reset --hard HEAD~5    # accidentally nuked 5 commits!

# Don't panic. Use reflog.
git reflog
# abc1234 HEAD@{0}: reset: moving to HEAD~5
# def5678 HEAD@{1}: commit: Latest important work    ← there it is

git reset --hard def5678   # restored!
```

### Scenario 5: Undo a Merge That Broke Production

```bash
# The merge commit is abc1234 and it's already pushed and deployed
# DON'T reset — revert is safe on shared branches
git revert -m 1 abc1234
git push origin main
# Deploy the revert commit
```

### Scenario 6: Amend a Commit That's Not the Latest

```bash
# Can't use --amend for older commits. Use interactive rebase.
git rebase -i HEAD~5

# In the editor, change "pick" to "edit" for the commit you want to change:
# edit abc1234 The commit to modify
# pick def5678 ...

# Git stops at that commit. Make your changes:
git add .
git commit --amend
git rebase --continue
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| git revert | Creates new undo commit. Safe on shared branches. Use `-m 1` for merge commits |
| git reset --soft | Moves HEAD. Keeps staging + working directory. Good for re-committing |
| git reset --mixed | Moves HEAD + unstages. Keeps working directory. The default mode |
| git reset --hard | Moves HEAD + unstages + discards working dir. DANGEROUS but recoverable via reflog |
| git restore | Discard working dir changes (`restore file`) or unstage (`restore --staged file`) |
| Reflog | Records every HEAD movement. Your safety net. Commits survive for 30+ days |
| Cherry-pick | Copies a commit to current branch with a new hash. Great for backports |
| Amend | Rewrites last commit (new hash). Only use before pushing |
| Shared branch rule | Only use `revert` on pushed branches. Never `reset` or `amend` |
| force-with-lease | Safer alternative to `--force`. Fails if remote changed since last fetch |
