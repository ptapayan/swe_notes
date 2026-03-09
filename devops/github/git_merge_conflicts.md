# Git Merge Conflicts — How to Read, Resolve, and Prevent Them

**Related:** [Git Fundamentals](git_fundamentals.md) | [Merge and Rebase](git_merge_and_rebase.md) | [Undo and Revert](git_undo_and_revert.md) | [Stash](git_stash.md)

---

## Why Conflicts Happen

A merge conflict occurs when Git cannot automatically combine changes from two branches because **both branches modified the same lines in the same file**, or one branch modified a file that the other branch deleted.

**How to think about it:** During a three-way merge, Git compares the common ancestor with both branch tips. If the ancestor's line X was changed differently by both branches, Git doesn't know which version to keep — so it asks you.

```
Common Ancestor (base):        Branch A:               Branch B:
─────────────────────          ─────────────            ─────────────
line 1: func getUser()         func getUser()           func getUser()
line 2:   return db.Get(id)      return cache.Get(id)     return db.Find(id)
line 3: }                      }                        }
                               ^^^^^^^^^^^^^^^^         ^^^^^^^^^^^^^^^^
                               Changed line 2           Changed line 2 differently

Git: "Both branches changed line 2. I don't know which version is correct."
     → CONFLICT
```

### Types of Conflicts

```
┌──────────────────────────────┬──────────────────────────────────────────────┐
│ Conflict Type                │ When It Happens                              │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Content conflict             │ Same lines modified differently              │
│ Delete/modify conflict       │ One branch deletes file, other modifies it  │
│ Rename/rename conflict       │ Both branches rename same file differently  │
│ Add/add conflict             │ Both branches add a file with same name     │
│ Rename/delete conflict       │ One branch renames file, other deletes it   │
└──────────────────────────────┴──────────────────────────────────────────────┘
```

---

## How to Read Conflict Markers

When a conflict occurs, Git writes both versions into the file with markers:

```
func getUser(id string) (*User, error) {
<<<<<<< HEAD
    // Check cache first for performance
    user, err := cache.Get(id)
    if err == nil {
        return user, nil
    }
    return db.Get(id)
=======
    // Use the new ORM query builder
    return db.Find(id).Where("active = ?", true).First()
>>>>>>> feature/orm-migration
}
```

**How to read this:**

```
<<<<<<< HEAD                          ← Start of YOUR changes (current branch)
    ... your version ...
=======                               ← Divider
    ... their version ...             ← The incoming branch's changes
>>>>>>> feature/orm-migration         ← End marker + branch name
```

- Everything between `<<<<<<< HEAD` and `=======` is what's on your current branch (the branch you're merging INTO)
- Everything between `=======` and `>>>>>>>` is what's on the incoming branch (the branch you're merging FROM)

### The Three-Way View — What Git Sees

```
                    Common Ancestor (base)
                    ──────────────────────
                    return db.Get(id)

                   /                     \
                  /                       \
        HEAD (ours)                   feature (theirs)
        ───────────                   ────────────────
        return cache.Get(id)          return db.Find(id)
        (cache-first strategy)        (ORM migration)

                          CONFLICT!
                          Both changed the same line from base.
```

To resolve, you choose one side, combine both, or write something entirely new.

---

## Step-by-Step Conflict Resolution Workflow

### 1. Start the Merge

```bash
git checkout main
git merge feature/orm-migration
# Auto-merging src/handler.go
# CONFLICT (content): Merge conflict in src/handler.go
# Auto-merging src/router.go
# Automatic merge failed; fix conflicts and then commit the result.
```

### 2. See Which Files Have Conflicts

```bash
git status
# On branch main
# You have unmerged paths.
#
# Unmerged paths:
#   (use "git add <file>..." to mark resolution)
#       both modified:   src/handler.go
#
# Changes to be committed:
#       modified:   src/router.go          ← auto-merged successfully
```

### 3. Open the Conflicted File and Resolve

Edit `src/handler.go` — remove the conflict markers and write the correct code:

```go
// BEFORE (with conflict markers):
<<<<<<< HEAD
    user, err := cache.Get(id)
    if err == nil {
        return user, nil
    }
    return db.Get(id)
=======
    return db.Find(id).Where("active = ?", true).First()
>>>>>>> feature/orm-migration

// AFTER (resolved — combined both approaches):
    user, err := cache.Get(id)
    if err == nil {
        return user, nil
    }
    return db.Find(id).Where("active = ?", true).First()
```

### 4. Mark as Resolved and Commit

```bash
git add src/handler.go              # marks the conflict as resolved
git commit                          # completes the merge (opens editor for merge message)
# Or: git commit -m "Merge feature/orm-migration: combine cache + ORM changes"
```

---

## Abort a Merge — Cancel Everything

```bash
# Undo the entire merge attempt — go back to before you ran git merge
git merge --abort
```

**What's happening:** Resets HEAD, index, and working directory to the state before the merge started. Uses `MERGE_HEAD` and `ORIG_HEAD` stored by Git to know what to revert to.

```bash
# If you've already committed the merge and want to undo it:
git revert -m 1 HEAD    # creates a new commit that undoes the merge
# See git_undo_and_revert.md for details
```

---

## Taking One Side Entirely

When you know one side is completely correct for a specific file:

```bash
# Take YOUR version (current branch / "ours") for a file
git checkout --ours src/handler.go
git add src/handler.go

# Take THEIR version (incoming branch / "theirs") for a file
git checkout --theirs src/handler.go
git add src/handler.go
```

**Careful with rebase:** During a rebase, ours/theirs are SWAPPED because rebase replays your commits onto the other branch:

```
During merge:                    During rebase:
  --ours   = current branch        --ours   = branch you're rebasing ONTO
  --theirs = incoming branch       --theirs = YOUR commits being replayed
```

For the entire merge (all files), you can set a strategy:

```bash
# Accept all ours on conflict (keep current branch version for conflicts)
git merge -X ours feature/orm-migration

# Accept all theirs on conflict (keep incoming branch version for conflicts)
git merge -X theirs feature/orm-migration
```

**Note:** `-X ours` / `-X theirs` only affects conflicting hunks. Non-conflicting changes from both sides are still merged normally.

---

## Rebase Conflicts — Resolving One Commit at a Time

Rebase conflicts work differently than merge conflicts. Since rebase replays commits one-by-one, you may need to resolve conflicts at each step.

```bash
git checkout feature
git rebase main
# Applying: Add user validation
# CONFLICT (content): Merge conflict in src/user.go
```

```
Rebase conflict resolution flow:

  Conflict on commit 1 of 4
  ┌──────────────────────────────────────────┐
  │ 1. Edit conflicted files                 │
  │ 2. git add <resolved-files>              │
  │ 3. git rebase --continue                 │
  │    → applies next commit                 │
  │    → may conflict again on commit 2      │
  │    → repeat until all commits replayed   │
  └──────────────────────────────────────────┘

  At any point:
  ├─ git rebase --abort     ← cancel entire rebase, go back to before
  ├─ git rebase --skip      ← skip this commit entirely (drops it)
  └─ git rebase --continue  ← proceed after resolving
```

```bash
# Resolve the conflict
vim src/user.go                 # fix the conflict markers
git add src/user.go             # mark as resolved

# Continue to the next commit
git rebase --continue
# Applying: Add user validation     ✓
# Applying: Add email verification
# CONFLICT (content): ...           ← another conflict!

# Resolve again...
vim src/user.go
git add src/user.go
git rebase --continue
# Applying: Add email verification  ✓
# Applying: Add rate limiting       ✓
# Applying: Add audit logging       ✓
# Successfully rebased and updated refs/heads/feature.
```

### Abort vs Skip

```bash
git rebase --abort       # Cancel everything. Go back to pre-rebase state.
                         # Use when things are too messy.

git rebase --skip        # Skip the current commit entirely — it will be DROPPED.
                         # Use when the commit's changes are already in the base.

git rebase --continue    # After resolving conflicts, continue with next commit.
```

---

## git rerere — Reuse Recorded Resolution

`rerere` (REuse REcorded REsolution) makes Git remember how you resolved a conflict so it can automatically apply the same resolution next time.

```bash
# Enable rerere globally
git config --global rerere.enabled true
```

**What's happening:** When you resolve a conflict, Git records the conflict (pre-image) and the resolution (post-image) in `.git/rr-cache/`. When it sees the same conflict again, it auto-applies the recorded resolution.

```
First time resolving:
  1. Git records:  conflict hash → your resolution
  2. Stored in .git/rr-cache/<hash>/

Second time encountering same conflict:
  1. Git recognizes the same conflict pattern
  2. Auto-applies the recorded resolution
  3. You just need to: git add <file> && git commit
```

**When this is invaluable:**

```bash
# Scenario: You rebase frequently and hit the same conflict every time
git rebase main
# CONFLICT in src/config.go
# Resolved by recorded resolution.    ← rerere applied it automatically!
git add src/config.go
git rebase --continue

# Scenario: You merge, realize it's wrong, undo, and merge again
git merge feature
# resolve conflicts...
git commit

# Undo and redo:
git reset --hard HEAD~1
git merge feature
# rerere auto-applies your previous resolution!
```

```bash
# Manage recorded resolutions
git rerere status          # show files with recorded resolutions
git rerere diff            # show what rerere would apply
git rerere forget <file>   # forget a recorded resolution for a file
```

---

## Using Merge Tools

### git mergetool — Visual Conflict Resolution

```bash
git mergetool
# Opens each conflicted file in your configured merge tool
```

Configure a merge tool:

```bash
# Use VS Code as merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait --merge $REMOTE $LOCAL $BASE $MERGED'

# Use vimdiff
git config --global merge.tool vimdiff

# Use IntelliJ / GoLand
git config --global merge.tool intellij
git config --global mergetool.intellij.cmd 'idea merge $LOCAL $REMOTE $BASE $MERGED'
```

**The four files a merge tool works with:**

```
┌─────────────────────────────────────────────────────────────┐
│                     Merge Tool Layout                        │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  LOCAL    │    │  BASE    │    │  REMOTE  │              │
│  │ (ours)   │    │(ancestor)│    │ (theirs) │              │
│  │          │    │          │    │          │              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                             │
│  ┌────────────────────────────────────────────┐             │
│  │              MERGED (output)               │             │
│  │    You edit this to resolve the conflict   │             │
│  └────────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Clean up .orig backup files that mergetool creates
git clean -f -- '*.orig'
# Or prevent them from being created:
git config --global mergetool.keepBackup false
```

---

## Preventing Conflicts — Best Practices

### Communication and Workflow

```
┌──────────────────────────────────────────────────────────────────────┐
│ Prevention Strategy              │ Why It Works                      │
├──────────────────────────────────┼───────────────────────────────────┤
│ Small, focused PRs               │ Less code changed = less overlap  │
│ Rebase onto main frequently      │ Stay close to latest code         │
│ Clear code ownership             │ Fewer people editing same files   │
│ Communicate about refactors      │ Warn team before renaming/moving  │
│ Avoid long-lived feature branches│ Divergence grows with time        │
│ Use feature flags                │ Merge to main early, activate     │
│                                  │   later via flag                  │
│ Modular architecture             │ Separate concerns = less overlap  │
│ Lock files (go.sum, yarn.lock)   │ Regenerate, don't manually merge │
└──────────────────────────────────┴───────────────────────────────────┘
```

### Automated Lock File Resolution

Lock files (go.sum, package-lock.json, yarn.lock) are the most common source of painful conflicts. Set up automatic resolution:

```bash
# .gitattributes — tell Git how to handle specific files during merge
# Regenerate lock files instead of merging them
*.lock merge=ours
go.sum merge=ours
```

Then after the merge:

```bash
# Regenerate the lock file properly
go mod tidy           # for Go
npm install           # for Node.js
```

---

## Complex Conflict Scenarios with Examples

### Scenario 1: Both Branches Added Code to the Same Function

```
<<<<<<< HEAD
func processOrder(order *Order) error {
    if err := validateOrder(order); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    if err := checkInventory(order); err != nil {
        return fmt.Errorf("inventory check failed: %w", err)
    }
=======
func processOrder(order *Order) error {
    if err := validateOrder(order); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    if err := applyDiscount(order); err != nil {
        return fmt.Errorf("discount failed: %w", err)
    }
>>>>>>> feature/discounts
```

**Resolution — keep both additions:**

```go
func processOrder(order *Order) error {
    if err := validateOrder(order); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    if err := checkInventory(order); err != nil {
        return fmt.Errorf("inventory check failed: %w", err)
    }
    if err := applyDiscount(order); err != nil {
        return fmt.Errorf("discount failed: %w", err)
    }
```

### Scenario 2: Delete/Modify Conflict

```bash
git merge feature/cleanup
# CONFLICT (modify/delete): src/legacy_handler.go deleted in feature/cleanup
#   and modified in HEAD. Version HEAD of src/legacy_handler.go left in tree.
```

**Resolution:**

```bash
# If the file should be deleted (the cleanup was correct):
git rm src/legacy_handler.go
git add .
git commit

# If the file should be kept (the modification is important):
git add src/legacy_handler.go
git commit
```

### Scenario 3: Conflict in a Rebase with Multiple Commits

```bash
git rebase main
# CONFLICT on commit 1/3: Add caching layer

# Check what the conflict is about
git diff                     # show current conflicts
git log --oneline --all -10  # see where you are in the graph

# Resolve commit 1
vim src/cache.go
git add src/cache.go
git rebase --continue

# CONFLICT on commit 2/3: Update cache config
vim src/config.go
git add src/config.go
git rebase --continue

# Commit 3 applies cleanly
# Successfully rebased.
```

### Scenario 4: Repeated Conflicts After Undo

```bash
# Enable rerere BEFORE you start
git config --global rerere.enabled true

# Merge and resolve conflicts
git merge feature/api
# ... resolve conflicts manually ...
git add .
git commit

# Realize you need to undo and redo
git reset --hard HEAD~1
git merge feature/api
# Resolved 'src/handler.go' using previous resolution.
# rerere auto-applied your previous resolution!
git add .
git commit
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Why conflicts | Both branches changed the same lines from the common ancestor |
| Conflict markers | `<<<<<<< HEAD` = yours, `=======` = divider, `>>>>>>> branch` = theirs |
| Resolution workflow | Edit file → remove markers → `git add` → `git commit` |
| `merge --abort` | Cancel the merge entirely, go back to pre-merge state |
| `--ours` / `--theirs` | Take one side entirely. SWAPPED during rebase |
| `-X ours` / `-X theirs` | Auto-resolve conflicting hunks only. Non-conflicts still merge |
| Rebase conflicts | Resolve per-commit. `--continue`, `--abort`, `--skip` |
| rerere | Records conflict resolutions for automatic reuse. Enable globally |
| Merge tools | Visual 3-way diff. Configure with `git config merge.tool` |
| Prevention | Small PRs, frequent rebasing, modular code, communication |
