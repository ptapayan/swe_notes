# Git Fundamentals — What Git Actually Is

**Related:** [Merge and Rebase](git_merge_and_rebase.md) | [Stash](git_stash.md) | [Merge Conflicts](git_merge_conflicts.md) | [Undo and Revert](git_undo_and_revert.md) | [GitHub Actions Fundamentals](github_actions_fundamentals.md)

---

## What Is Git?

Git is a **distributed version control system** — but more precisely, it is a **content-addressable filesystem** with a version control user interface built on top of it.

**How to think about it:** Git is a key-value store where every piece of content (file, directory, commit) is hashed with SHA-1 and stored as an object. The version history is a **directed acyclic graph (DAG)** of commit objects, where each commit points to its parent(s).

```
                        Content-Addressable Filesystem
                        ──────────────────────────────
  Content (bytes) ──→ SHA-1 hash ──→ stored as object in .git/objects/

  "hello world\n"  ──→  95d09f2b10159347eece71399a7e2e907ea3df4f
                         ^^^^^^^^
                         first 2 chars = directory name
                         remaining 38 chars = filename
```

Git stores **snapshots**, NOT diffs. Every commit records the complete state of the project at that point in time. This is the single most important thing to understand about Git's design.

---

## Git Objects — The Four Building Blocks

Everything in Git is stored as one of four object types. Each is identified by its SHA-1 hash.

### 1. Blob (Binary Large Object)

A blob stores the **contents of a single file** — raw bytes, nothing else. No filename, no permissions, no metadata.

```
┌──────────────────────────────────┐
│  Blob Object (SHA: a1b2c3...)    │
│                                  │
│  "package main\n\nfunc main()   │
│   {\n\tfmt.Println(\"hello\")   │
│   \n}\n"                         │
│                                  │
│  (just the raw file content)     │
└──────────────────────────────────┘
```

**What's happening:** When you `git add main.go`, Git computes the SHA-1 of the file contents, compresses it with zlib, and stores it in `.git/objects/`. If two files have identical content, they share the same blob — even across branches.

```bash
# See blob hash of a file
git hash-object main.go
# a1b2c3d4e5f6...

# Read a blob's content
git cat-file -p a1b2c3d
```

### 2. Tree

A tree stores a **directory listing** — mapping filenames to blob hashes (files) or other tree hashes (subdirectories). It also stores file permissions (100644 for regular files, 100755 for executables, 040000 for directories).

```
┌──────────────────────────────────────────────────────────┐
│  Tree Object (SHA: d4e5f6...)                            │
│                                                          │
│  100644 blob a1b2c3...  main.go                          │
│  100644 blob b2c3d4...  go.mod                           │
│  040000 tree c3d4e5...  pkg/                             │
│  100755 blob d4e5f6...  run.sh                           │
└──────────────────────────────────────────────────────────┘
```

**How to think about it:** A tree is like a directory — it maps names to content. Trees can contain other trees (subdirectories), forming a recursive structure.

### 3. Commit

A commit object stores: a pointer to a **tree** (the snapshot), pointer(s) to **parent commit(s)**, the **author**, the **committer**, a **timestamp**, and a **message**.

```
┌──────────────────────────────────────────────────────────┐
│  Commit Object (SHA: f6a7b8...)                          │
│                                                          │
│  tree      d4e5f6...          (root tree of snapshot)    │
│  parent    e5f6a7...          (previous commit)          │
│  author    Alice <a@b.com> 1700000000 -0700              │
│  committer Alice <a@b.com> 1700000000 -0700              │
│                                                          │
│  Fix null pointer in user service                        │
└──────────────────────────────────────────────────────────┘
```

**Key detail:** A merge commit has TWO parent pointers. The initial commit has ZERO parents. Regular commits have ONE parent.

```bash
# Inspect a commit object
git cat-file -p HEAD
# tree d4e5f6a7b8c9...
# parent e5f6a7b8c9d0...
# author Alice <alice@example.com> 1700000000 -0700
# committer Alice <alice@example.com> 1700000000 -0700
#
# Fix null pointer in user service
```

### 4. Tag (Annotated)

An annotated tag is an object that points to a commit with additional metadata (tagger, date, message). Lightweight tags are just refs (no object).

```
┌──────────────────────────────────────────────────────────┐
│  Tag Object (SHA: 1a2b3c...)                             │
│                                                          │
│  object    f6a7b8...          (points to a commit)       │
│  type      commit                                        │
│  tag       v1.2.0                                        │
│  tagger    Alice <a@b.com> 1700000000 -0700              │
│                                                          │
│  Release version 1.2.0                                   │
└──────────────────────────────────────────────────────────┘
```

### How Objects Form the DAG

```
                    Commit DAG
                    ──────────

    commit c3    commit c2    commit c1
    ┌────────┐   ┌────────┐   ┌────────┐
    │tree: t3│──→│tree: t2│──→│tree: t1│──→ (no parent)
    │parent: │   │parent: │   │        │
    │  c2    │   │  c1    │   │        │
    └────────┘   └────────┘   └────────┘
        │            │            │
        ▼            ▼            ▼
    tree t3      tree t2      tree t1
    ┌────────┐   ┌────────┐   ┌────────┐
    │main.go │   │main.go │   │main.go │
    │  →blob │   │  →blob │   │  →blob │
    │go.mod  │   │go.mod  │   │go.mod  │
    │  →blob │   │  →blob │   │  →blob │
    └────────┘   └────────┘   └────────┘
```

**What's happening:** Each commit points to a tree (complete snapshot). If a file didn't change between commits, both trees point to the SAME blob — Git deduplicates automatically. This is why Git is efficient despite storing snapshots: unchanged files are shared, not duplicated.

---

## Refs — Branches, HEAD, and Tags

A **ref** is just a file containing a SHA-1 hash that points to a commit. That's it. Branches are not complex structures — they are 41-byte files.

### Branches Are Just Pointers

```
.git/refs/heads/main     → contains: f6a7b8c9d0e1...  (a commit hash)
.git/refs/heads/feature  → contains: a1b2c3d4e5f6...  (a commit hash)
```

```
    a1b2c3 ← feature (refs/heads/feature)
      │
    f6a7b8 ← main (refs/heads/main)
      │
    e5f6a7
      │
    d4e5f6
```

**How to think about it:** Creating a branch (`git branch feature`) literally writes a 41-byte file. Deleting a branch deletes that file. The commits themselves are untouched.

### HEAD — Where You Are Now

HEAD is a **symbolic reference** — it usually points to a branch, not directly to a commit.

```bash
cat .git/HEAD
# ref: refs/heads/main       ← "I'm on the main branch"
```

```
  HEAD
   │
   ▼
  main ──→ f6a7b8  (commit)
```

When you `git checkout feature`, Git just updates `.git/HEAD` to say `ref: refs/heads/feature`.

**Detached HEAD:** When HEAD points directly to a commit (not a branch), you're in "detached HEAD" state. This happens when you check out a specific commit hash or tag.

```bash
git checkout f6a7b8
cat .git/HEAD
# f6a7b8c9d0e1...    ← raw hash, not a ref — detached HEAD
```

### Remote Tracking Branches

Remote tracking branches are refs that record the last known state of branches on a remote.

```
.git/refs/remotes/origin/main     → a commit hash
.git/refs/remotes/origin/feature  → a commit hash
```

```
    c3 ← feature (local)
    │
    c2 ← origin/feature (remote tracking — last known state)
    │
    c1 ← main, origin/main
```

**What's happening:** `origin/main` is NOT the remote. It's a local ref that records where `main` was on the remote the last time you did `git fetch` or `git pull`. Running `git fetch` updates these remote tracking refs.

---

## The Three Areas — Working Directory, Staging Area, Repository

```
┌──────────────────┐    git add     ┌──────────────────┐   git commit   ┌──────────────────┐
│                  │  ──────────→   │                  │  ──────────→   │                  │
│ Working Directory│                │  Staging Area    │                │   Repository     │
│ (Working Tree)   │  ←──────────   │  (Index)         │                │   (.git/)        │
│                  │  git checkout  │                  │                │                  │
│ The actual files │                │ .git/index       │                │ .git/objects/    │
│ on your disk     │                │ binary file that │                │ The DAG of all   │
│                  │                │ lists files +    │                │ commits, trees,  │
│                  │                │ their blob hashes│                │ blobs            │
└──────────────────┘                └──────────────────┘                └──────────────────┘
```

### Working Directory

The actual files on disk — what you see in your editor. This is the checked-out version of one commit, plus any modifications you've made.

### Staging Area (Index)

A **binary file** at `.git/index` that stores the list of tracked files, their blob hashes, timestamps, and permissions. It represents the NEXT commit you're about to make.

```bash
# See what's in the index
git ls-files --stage
# 100644 a1b2c3d4... 0   main.go
# 100644 b2c3d4e5... 0   go.mod
# 100644 c3d4e5f6... 0   pkg/handler.go
```

**How to think about it:** The staging area is a draft of your next commit. `git add` copies the working directory version of a file into the index. `git commit` packages the index into a tree object and creates a commit pointing to it.

### Repository (.git/)

The object database — all blobs, trees, commits, and tags stored under `.git/objects/`, plus all refs under `.git/refs/`.

---

## How git init, git add, and git commit Work at the Object Level

### git init

```bash
git init
```

**What's happening:** Creates the `.git/` directory with the following structure:

```
.git/
├── HEAD              # ref: refs/heads/main (symbolic ref to current branch)
├── config            # repository-level configuration
├── description       # used by GitWeb (mostly unused)
├── hooks/            # sample hook scripts
├── info/             # exclude patterns (like .gitignore but not committed)
│   └── exclude
├── objects/          # the object database (empty at init)
│   ├── info/
│   └── pack/         # packfiles (compressed objects for efficiency)
└── refs/             # branch and tag pointers
    ├── heads/        # local branches (empty — no commits yet)
    └── tags/         # tags (empty)
```

### git add

```bash
echo "hello" > hello.txt
git add hello.txt
```

**What's happening step by step:**

1. Git reads the contents of `hello.txt`
2. Computes SHA-1: `ce013625030ba8dba906f756967f9e9ca394464a`
3. Compresses the content with zlib
4. Stores it in `.git/objects/ce/013625030ba8dba906f756967f9e9ca394464a`
5. Updates `.git/index` to record: `100644 ce0136... hello.txt`

The blob now exists in the object database. The index (staging area) knows about it. But no commit exists yet.

### git commit

```bash
git commit -m "Add hello.txt"
```

**What's happening step by step:**

1. Git reads the index (`.git/index`)
2. Creates a **tree object** from the index entries (files → blobs, directories → sub-trees)
3. Creates a **commit object** pointing to that tree, with the current HEAD as parent
4. Updates the current branch ref to point to the new commit

```bash
# After the commit, you can trace the object chain:
git cat-file -p HEAD           # shows commit → tree hash, parent, message
git cat-file -p <tree-hash>    # shows tree → blob hashes, filenames
git cat-file -p <blob-hash>    # shows the raw file content
```

---

## Reading git log — Understanding the Commit Graph

```bash
git log --oneline --graph --all
```

```
* e4f5a6b (HEAD -> feature/api) Add rate limiting
* d3e4f5a Add authentication middleware
| * c2d3e4f (main) Fix memory leak in cache
| * b1c2d3e Update README
|/
* a0b1c2d Initial project setup
```

**How to read this:**

- `*` = a commit
- `|` = the line of history continues
- `/` and `\` = branches diverging or merging
- `(HEAD -> feature/api)` = HEAD is on branch feature/api, which points to this commit
- `(main)` = the main branch points to this commit
- The graph shows that `feature/api` and `main` diverged from `a0b1c2d`

### A Merge Commit in the Graph

```bash
git log --oneline --graph
```

```
*   f5a6b7c (HEAD -> main) Merge branch 'feature/api'
|\
| * e4f5a6b Add rate limiting
| * d3e4f5a Add authentication middleware
|/
* c2d3e4f Fix memory leak in cache
* b1c2d3e Update README
* a0b1c2d Initial project setup
```

The merge commit `f5a6b7c` has TWO parents: `c2d3e4f` (main) and `e4f5a6b` (feature/api).

---

## The .git Directory — Full Structure

```
.git/
├── HEAD                  # symbolic ref → current branch
├── ORIG_HEAD             # previous HEAD (set by merge, rebase, reset)
├── FETCH_HEAD            # result of last git fetch
├── MERGE_HEAD            # commit being merged (during merge)
├── config                # local repo config (remotes, branch tracking)
├── index                 # staging area (binary file)
├── objects/
│   ├── 0a/               # first 2 hex chars of SHA
│   │   └── 1b2c3d...     # remaining 38 hex chars (loose object)
│   ├── info/
│   └── pack/             # packfiles — compressed bundles of objects
│       ├── pack-abc.idx  # index for fast lookup
│       └── pack-abc.pack # packed objects (deltified)
├── refs/
│   ├── heads/            # local branches
│   │   ├── main          # file containing commit hash
│   │   └── feature/api   # can use / for namespacing
│   ├── remotes/
│   │   └── origin/       # remote tracking branches
│   │       ├── main
│   │       └── feature/api
│   ├── tags/             # tags
│   │   └── v1.0.0
│   └── stash             # stash ref (latest stash)
├── hooks/                # client-side and server-side hooks
│   ├── pre-commit.sample
│   ├── pre-push.sample
│   └── commit-msg.sample
├── info/
│   └── exclude           # per-repo gitignore (not committed)
└── logs/                 # reflog — history of ref movements
    ├── HEAD              # every time HEAD changes
    └── refs/
        └── heads/
            └── main      # every time main branch moves
```

**Packfiles:** For efficiency, Git periodically packs loose objects into a single **packfile** using delta compression (storing diffs between similar objects). This happens during `git gc`, `git push`, or `git repack`. The `.idx` file provides O(1) lookup by SHA.

---

## Remote Tracking Branches — origin/main vs main

```
                      Remote (GitHub)              Local Machine
                    ┌─────────────────┐         ┌─────────────────────────────┐
                    │  refs/heads/     │         │  refs/heads/                │
                    │    main → c5     │         │    main → c5                │
                    │    feature → c3  │         │    feature → c7             │
                    │                  │         │                             │
                    │                  │         │  refs/remotes/origin/       │
                    │                  │         │    main → c5                │
                    │                  │         │    feature → c3             │
                    └─────────────────┘         └─────────────────────────────┘
                                                         ▲
                                                         │
                                           git fetch updates these
```

**What's happening:**
- `main` = your local branch. You move it by committing.
- `origin/main` = your local copy of where main is on the remote. Moved by `git fetch`.
- `origin/feature` is at `c3` (the remote hasn't seen your new commits `c4-c7` yet).
- Your local `feature` is at `c7` (you've made new commits locally).

```bash
# See all branches including remote tracking
git branch -a
# * main
#   feature
#   remotes/origin/main
#   remotes/origin/feature

# See tracking relationships
git branch -vv
# * main     c5d6e7f [origin/main] Fix cache bug
#   feature  c7a8b9c [origin/feature: ahead 4] Add new endpoint
```

---

## Practical Examples / Real-World Scenarios

### Scenario 1: Inspect What a Commit Changed at the Object Level

```bash
# Show the commit object
git cat-file -p abc1234
# tree 9f8e7d6c...
# parent 8e7d6c5b...
# author ...

# Compare trees to see what files changed
git diff-tree --no-commit-id -r abc1234
# :100644 100644 old-blob new-blob M  src/handler.go
# :000000 100644 0000000 new-blob A  src/middleware.go
```

### Scenario 2: Recover a Deleted Branch

```bash
# Oops, deleted a branch
git branch -D feature/experiment

# The commits still exist — find them via reflog
git reflog
# abc1234 HEAD@{3}: checkout: moving from feature/experiment to main

# Recreate the branch
git branch feature/experiment abc1234
```

### Scenario 3: Understand Why Two Files Share a Blob

```bash
# Two files with identical content share the same blob
echo "shared content" > file1.txt
cp file1.txt file2.txt
git add file1.txt file2.txt
git ls-files --stage
# 100644 d95f3ad... 0  file1.txt
# 100644 d95f3ad... 0  file2.txt
#         ^^^^^^^^ same hash — same blob object
```

### Scenario 4: Verify Repo Integrity

```bash
# Git can verify all objects are valid and reachable
git fsck --full
# Checking object directories: 100% done.
# Checking objects: 100% done.
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Content-addressable | Every object is identified by SHA-1 of its content. Same content = same hash |
| Four objects | Blob (file content), Tree (directory), Commit (snapshot + metadata), Tag (named pointer) |
| Snapshots not diffs | Each commit stores a complete tree. Unchanged files are deduplicated via shared blobs |
| Branch | A 41-byte file containing a commit hash. Creating/deleting branches is instant |
| HEAD | Symbolic ref pointing to current branch. Detached HEAD = points directly to a commit |
| Staging area | `.git/index` — the draft of your next commit. `git add` writes blobs and updates index |
| Remote tracking | `origin/main` is a local ref updated by `git fetch`. It is NOT the remote itself |
| Three areas | Working directory → staging area (index) → repository (object database) |
| Packfiles | Git compresses loose objects into packfiles with delta compression for efficiency |
| DAG | Commit history is a directed acyclic graph. Merge commits have 2+ parents |
