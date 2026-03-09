# Git Stash — Saving Work in Progress

**Related:** [Git Fundamentals](git_fundamentals.md) | [Merge and Rebase](git_merge_and_rebase.md) | [Merge Conflicts](git_merge_conflicts.md) | [Undo and Revert](git_undo_and_revert.md)

---

## What Is Stash?

`git stash` saves your uncommitted changes (working directory + staging area) into a stack of WIP snapshots, then reverts your working directory to the last commit. It lets you context-switch without committing half-done work.

**How to think about it:** Stash is a stack (LIFO). Each entry is a snapshot of your dirty state. Push saves, pop restores. The stash entries are stored as commit objects on a special ref — they're real Git objects, not temporary files.

---

## Internal Structure — How Stash Actually Works

A stash entry is stored as a **commit object** attached to the special ref `refs/stash`. Each stash commit has a unique structure: it has **2 or 3 parents**.

```
                    Stash Commit (stash@{0})
                   /          |          \
                  /           |           \
           Parent 1      Parent 2      Parent 3 (optional)
           HEAD commit   Index state   Untracked files
           at time of    (staging      (only if -u or -a
            stash        area snapshot) was used)
```

**What's happening internally:**

```bash
git stash push
```

1. Git creates a **tree** from the current index → becomes parent 2 (index state)
2. Git creates a **tree** from the working directory → stored in the stash commit
3. If `-u` was used, creates another tree for untracked files → parent 3
4. Git creates a commit object with these parents, attached to `refs/stash`
5. Git resets the working directory and index to HEAD

```bash
# You can verify this:
git cat-file -p stash
# tree abc1234...
# parent def5678... (HEAD at time of stash)
# parent 789abcd... (index state)
# author ...
# WIP on main: abc1234 Fix cache bug

git cat-file -p stash^1   # the HEAD commit
git cat-file -p stash^2   # the index state
git cat-file -p stash^3   # untracked files (if stashed with -u)
```

The stash "stack" is implemented using the **reflog** of `refs/stash`. Each new stash updates the ref and the previous value becomes `stash@{1}`, then `stash@{2}`, etc.

---

## Basic Stash Operations

### git stash / git stash push — Save Changes

```bash
# Stash tracked modified + staged changes
git stash

# Equivalent (explicit form)
git stash push

# Stash with a descriptive message
git stash push -m "WIP: halfway through refactoring auth"
```

```
Before:                               After git stash:

Working Directory: modified files     Working Directory: clean (matches HEAD)
Staging Area: staged changes          Staging Area: clean
                                      Stash Stack: [stash@{0}] = saved state
```

**What gets stashed by default:**
- Tracked files with modifications (working directory changes)
- Staged changes (index)

**What does NOT get stashed by default:**
- Untracked files (new files never added)
- Ignored files

### git stash pop — Restore and Remove

```bash
git stash pop
```

Applies the most recent stash (`stash@{0}`) and removes it from the stash stack.

```
Before:                               After git stash pop:

Working Directory: clean              Working Directory: restored modifications
Stash: [stash@{0}] = saved state     Stash: (empty — entry removed)
```

**Important:** If applying the stash causes a conflict, `pop` behaves like `apply` — the stash is NOT removed. You must resolve the conflict and then manually `git stash drop`.

### git stash apply — Restore but Keep

```bash
git stash apply             # apply most recent stash
git stash apply stash@{2}   # apply a specific stash entry
```

Same as `pop` but the stash entry stays in the stack. Use this when you want to apply the same stash to multiple branches.

### pop vs apply — When to Use Which

```
pop   = apply + drop  (removes from stack on success)
apply = apply only    (keeps the entry in the stack)

Use pop when:   You're done with the stash, want it cleaned up
Use apply when: You might need the stash again (e.g., applying to multiple branches)
```

---

## Viewing the Stash Stack

### git stash list — Show All Entries

```bash
git stash list
# stash@{0}: WIP on feature/auth: abc1234 Add login endpoint
# stash@{1}: On main: def5678 Fix cache bug
# stash@{2}: WIP on feature/auth: 789abcd Refactor middleware
```

The format is: `stash@{N}: <type> on <branch>: <commit> <message>`

### git stash show — View Changes in a Stash

```bash
# Summary (stat view)
git stash show
#  src/auth.go      | 42 +++++++++++++++++++---
#  src/middleware.go | 15 ++++++
#  2 files changed, 53 insertions(+), 4 deletions(-)

# Full diff
git stash show -p
# diff --git a/src/auth.go b/src/auth.go
# --- a/src/auth.go
# +++ b/src/auth.go
# ...

# Show a specific stash entry
git stash show -p stash@{2}
```

---

## Dropping and Clearing Stashes

### git stash drop — Remove One Entry

```bash
git stash drop             # drop the most recent (stash@{0})
git stash drop stash@{2}   # drop a specific entry
```

After dropping `stash@{2}`, what was `stash@{3}` becomes `stash@{2}`, etc. The indices shift.

### git stash clear — Remove ALL Entries

```bash
git stash clear
```

**Caution:** This removes ALL stash entries. They can still be recovered from the reflog for a limited time:

```bash
# Find lost stash commits
git fsck --unreachable | grep commit
git show <commit-hash>   # inspect to find your stash
git stash apply <commit-hash>
```

---

## Advanced Stash Operations

### Stash Specific Files (Partial Stash)

```bash
# Stash only specific files
git stash push -- src/auth.go src/middleware.go

# Stash with a message and specific files
git stash push -m "WIP auth changes" -- src/auth.go src/middleware.go
```

**What's happening:** Only the specified files are stashed and reverted. Other modified files remain in your working directory.

### Include Untracked Files

```bash
# Include untracked files (new files that haven't been git added)
git stash push -u
git stash push --include-untracked

# Include untracked AND ignored files (rarely needed)
git stash push -a
git stash push --all
```

```
Without -u:                          With -u:

Stashed:                             Stashed:
  ✓ Tracked modified files             ✓ Tracked modified files
  ✓ Staged changes                     ✓ Staged changes
  ✗ Untracked files                    ✓ Untracked files
  ✗ Ignored files                      ✗ Ignored files
```

### Keep the Staging Area Intact

```bash
# Stash only working directory changes, keep the index (staging area)
git stash push --keep-index
```

**When to use:** You've carefully staged specific changes and want to test JUST the staged changes. Stash the rest, run tests on what's staged, then pop the stash back.

```bash
# Workflow: test only staged changes
git add -p                    # carefully stage specific hunks
git stash push --keep-index   # stash unstaged changes
npm test                      # test only what's staged
git stash pop                 # restore the rest
```

### Create a Branch from a Stash

```bash
git stash branch new-feature stash@{0}
```

**What's happening:**

1. Creates a new branch `new-feature` starting from the commit where the stash was created
2. Applies the stash on the new branch
3. Drops the stash entry (if applied cleanly)

**When to use:** When the branch has moved forward since you stashed, and applying the stash causes conflicts. Creating a branch from the stash checkpoints you at the exact state where the stash was created — no conflicts possible.

```
Before:

main:  c1 ← c2 ← c3 ← c4 ← c5    (branch moved forward)
stash@{0}: created at c2             (stash is "old")

After git stash branch recovery stash@{0}:

main:     c1 ← c2 ← c3 ← c4 ← c5
                 \
recovery:         c2 + stash changes applied cleanly
```

---

## Stash vs Temporary Commit — When to Use Each

```
┌──────────────────────┬───────────────────────────────────────────┐
│                      │ git stash          │ Temp commit           │
├──────────────────────┼────────────────────┼───────────────────────┤
│ How                  │ git stash push     │ git commit -m "WIP"  │
│ Restore              │ git stash pop      │ git reset HEAD~1     │
│ Survives branch ops  │ Yes (global stack) │ Yes (in branch)      │
│ Visible in log       │ No                 │ Yes (until amended)  │
│ Shareable via push   │ No                 │ Yes                  │
│ Named/discoverable   │ Stash list + msg   │ Commit message       │
│ Risk of forgetting   │ Higher (hidden)    │ Lower (in git log)   │
│ Best for             │ Quick switch, <1hr │ Longer pauses, days  │
└──────────────────────┴────────────────────┴───────────────────────┘
```

**Rule of thumb:**
- **Stash** for quick context switches (minutes to an hour)
- **Temporary commit** for longer pauses (overnight, switching to another task for days)
- **Feature branch + commit** for anything you might want to share or come back to later

---

## Practical Examples / Real-World Scenarios

### Scenario 1: Switch Branches with Dirty Working Directory

```bash
# You're working on feature/auth with unsaved changes
git status
# modified: src/auth.go
# modified: src/handler.go

# Need to fix a bug on main — stash your work
git stash push -m "WIP: auth token validation"

# Switch to main and fix the bug
git checkout main
git checkout -b hotfix/null-check
# ... fix the bug, commit, push ...

# Go back to your feature branch
git checkout feature/auth
git stash pop
# Your changes are restored
```

### Scenario 2: Pull with a Dirty Working Directory

```bash
# You have uncommitted changes and need to pull
git pull
# error: Your local changes to the following files would be overwritten by merge

# Stash, pull, then pop
git stash push -m "WIP before pull"
git pull
git stash pop
# If the pull changed the same files you were editing, you may get conflicts
```

### Scenario 3: Test Only Staged Changes

```bash
# You've staged specific changes with git add -p
git stash push --keep-index -m "unstaged changes"
# Working directory now only has staged changes
make test
# Tests pass? Great. Restore the rest.
git stash pop
```

### Scenario 4: Accidentally Stashed and Need to Recover

```bash
# You stashed something a week ago and forgot about it
git stash list
# stash@{0}: WIP on main: ... (recent)
# stash@{1}: WIP on main: ... (recent)
# stash@{2}: WIP on feature/billing: abc1234 Add invoice model
#                                     ^^^^^^^^ there it is!

# Check what's in it
git stash show -p stash@{2}

# Apply it to a new branch
git stash branch feature/billing-recovery stash@{2}
```

### Scenario 5: Stash Only Specific Files

```bash
# You changed 5 files but only want to stash 2 of them
git stash push -m "WIP: database layer" -- src/db/connection.go src/db/queries.go

# The other 3 files remain modified in your working directory
git status
# modified: src/handler.go       (still here)
# modified: src/router.go        (still here)
# modified: src/middleware.go     (still here)
```

### Scenario 6: Recover a Dropped Stash

```bash
# Oops, dropped a stash by mistake
git stash drop stash@{0}

# Stash commits become unreachable but aren't garbage collected immediately
# Find them:
git fsck --unreachable --no-reflogs | grep commit
# unreachable commit abc1234...
# unreachable commit def5678...

# Inspect each to find yours
git show abc1234
# Shows the commit — check if it's your stash

# Recover it
git stash apply abc1234
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| What stash is | Stack of WIP snapshots stored as commit objects on `refs/stash` |
| Internal structure | Stash commit has 2-3 parents: HEAD, index state, untracked (optional) |
| `git stash push` | Saves working directory + index. Add `-m` for messages, `-u` for untracked |
| pop vs apply | pop = apply + drop. apply keeps the entry. Use apply for multi-branch |
| `git stash show -p` | View the full diff of a stash entry |
| `--keep-index` | Stash only working directory changes, keep staging area intact |
| `-- file1 file2` | Stash only specific files |
| `git stash branch` | Create branch from stash at the original commit — avoids conflicts |
| Stash vs temp commit | Stash for quick switches (<1hr). Temp commit for longer pauses |
| Recovery | Dropped stashes recoverable via `git fsck --unreachable` until GC |
