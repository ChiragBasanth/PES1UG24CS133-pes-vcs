# Building PES-VCS — A Version Control System from Scratch

**Objective:** Build a local version control system that tracks file changes, stores snapshots efficiently, and supports commit history. Every component maps directly to operating system and filesystem concepts.

**Platform:** Ubuntu 22.04

---

## Getting Started

### Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Using This Repository

This is a **template repository**. Do **not** fork it.

1. Click **"Use this template"** → **"Create a new repository"** on GitHub
2. Name your repository (e.g., `SRN-pes-vcs`) and set it to **public**. Replace `SRN` with your actual SRN, e.g., `PESXUG24CSYYY-pes-vcs`
3. Clone this repository to your local machine and do all your lab work inside this directory.
4.  **Important:** Remember to commit frequently as you progress. You are required to have a minimum of 5 detailed commits per phase. Refer to [Submission Requirements](#submission-requirements) for more details.
5. Clone your new repository and start working

The repository contains skeleton source files with `// TODO` markers where you need to write code. Functions marked `// PROVIDED` are complete — do not modify them.

### Building

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove all build artifacts
```

### Author Configuration

PES-VCS reads the author name from the `PES_AUTHOR` environment variable:

```bash
export PES_AUTHOR="Your Name <PESXUG24CS042>"
```

If unset, it defaults to `"PES User <pes@localhost>"`.

### File Inventory

| File               | Role                                 | Your Task                                          |
| ------------------ | ------------------------------------ | -------------------------------------------------- |
| `pes.h`            | Core data structures and constants   | Do not modify                                      |
| `object.c`         | Content-addressable object store     | Implement `object_write`, `object_read`            |
| `tree.h`           | Tree object interface                | Do not modify                                      |
| `tree.c`           | Tree serialization and construction  | Implement `tree_from_index`                        |
| `index.h`          | Staging area interface               | Do not modify                                      |
| `index.c`          | Staging area (text-based index file) | Implement `index_load`, `index_save`, `index_add`  |
| `commit.h`         | Commit object interface              | Do not modify                                      |
| `commit.c`         | Commit creation and history          | Implement `commit_create`                          |
| `pes.c`            | CLI entry point and command dispatch | Do not modify                                      |
| `test_objects.c`   | Phase 1 test program                 | Do not modify                                      |
| `test_tree.c`      | Phase 2 test program                 | Do not modify                                      |
| `test_sequence.sh` | End-to-end integration test          | Do not modify                                      |
| `Makefile`         | Build system                         | Do not modify                                      |

---

## Understanding Git: What You're Building

Before writing code, understand how Git works under the hood. Git is a content-addressable filesystem with a few clever data structures on top. Everything in this lab is based on Git's real design.

### The Big Picture

When you run `git commit`, Git doesn't store "changes" or "diffs." It stores **complete snapshots** of your entire project. Git uses two tricks to make this efficient:

1. **Content-addressable storage:** Every file is stored by the SHA hash of its contents. Same content = same hash = stored only once.
2. **Tree structures:** Directories are stored as "tree" objects that point to file contents, so unchanged files are just pointers to existing data.

```
Your project at commit A:          Your project at commit B:
                                   (only README changed)

    root/                              root/
    ├── README.md  ─────┐              ├── README.md  ─────┐
    ├── src/            │              ├── src/            │
    │   └── main.c ─────┼─┐            │   └── main.c ─────┼─┐
    └── Makefile ───────┼─┼─┐          └── Makefile ───────┼─┼─┐
                        │ │ │                              │ │ │
                        ▼ ▼ ▼                              ▼ ▼ ▼
    Object Store:       ┌─────────────────────────────────────────────┐
                        │  a1b2c3 (README v1)    ← only this is new   │
                        │  d4e5f6 (README v2)                         │
                        │  789abc (main.c)       ← shared by both!    │
                        │  fedcba (Makefile)     ← shared by both!    │
                        └─────────────────────────────────────────────┘
```

### The Three Object Types

#### 1. Blob (Binary Large Object)

A blob is just file contents. No filename, no permissions — just the raw bytes.

```
blob 16\0Hello, World!\n
     ↑    ↑
     │    └── The actual file content
     └─────── Size in bytes
```

The blob is stored at a path determined by its SHA-256 hash. If two files have identical contents, they share one blob.

#### 2. Tree

A tree represents a directory. It's a list of entries, each pointing to a blob (file) or another tree (subdirectory).

```
100644 blob a1b2c3d4... README.md
100755 blob e5f6a7b8... build.sh        ← executable file
040000 tree 9c0d1e2f... src             ← subdirectory
       ↑    ↑           ↑
       │    │           └── name
       │    └── hash of the object
       └─────── mode (permissions + type)
```

Mode values:
- `100644` — regular file, not executable
- `100755` — regular file, executable
- `040000` — directory (tree)

#### 3. Commit

A commit ties everything together. It points to a tree (the project snapshot) and contains metadata.

```
tree 9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Alice <alice@example.com> 1699900000
committer Alice <alice@example.com> 1699900000

Add new feature
```

The parent pointer creates a linked list of history:

```
    C3 ──────► C2 ──────► C1 ──────► (no parent)
    │          │          │
    ▼          ▼          ▼
  Tree3      Tree2      Tree1
```

### How Objects Connect

```
                    ┌─────────────────────────────────┐
                    │           COMMIT                │
                    │  tree: 7a3f...                  │
                    │  parent: 4b2e...                │
                    │  author: Alice                  │
                    │  message: "Add feature"         │
                    └─────────────┬───────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────────┐
                    │         TREE (root)             │
                    │  100644 blob f1a2... README.md  │
                    │  040000 tree 8b3c... src        │
                    │  100644 blob 9d4e... Makefile   │
                    └──────┬──────────┬───────────────┘
                           │          │
              ┌────────────┘          └────────────┐
              ▼                                    ▼
┌─────────────────────────┐          ┌─────────────────────────┐
│      TREE (src)         │          │     BLOB (README.md)    │
│ 100644 blob a5f6 main.c │          │  # My Project           │
└───────────┬─────────────┘          └─────────────────────────┘
            ▼
       ┌────────┐
       │ BLOB   │
       │main.c  │
       └────────┘
```

### References and HEAD

References are files that map human-readable names to commit hashes:

```
.pes/
├── HEAD                    # "ref: refs/heads/main"
└── refs/
    └── heads/
        └── main            # Contains: a1b2c3d4e5f6...
```

**HEAD** points to a branch name. The branch file contains the latest commit hash. When you commit:

1. Git creates the new commit object (pointing to parent)
2. Updates the branch file to contain the new commit's hash
3. HEAD still points to the branch, so it "follows" automatically

```
Before commit:                    After commit:

HEAD ─► main ─► C2 ─► C1         HEAD ─► main ─► C3 ─► C2 ─► C1
```

### The Index (Staging Area)

The index is the "preparation area" for the next commit. It tracks which files are staged.

```
Working Directory          Index               Repository (HEAD)
─────────────────         ─────────           ─────────────────
README.md (modified) ──── pes add ──► README.md (staged)
src/main.c                            src/main.c          ──► Last commit's
Makefile                               Makefile                snapshot
```

The workflow:

1. `pes add file.txt` → computes blob hash, stores blob, updates index
2. `pes commit -m "msg"` → builds tree from index, creates commit, updates branch ref

### Content-Addressable Storage

Objects are named by their content's hash:

```python
# Pseudocode
def store_object(content):
    hash = sha256(content)
    path = f".pes/objects/{hash[0:2]}/{hash[2:]}"
    write_file(path, content)
    return hash
```

This gives us:
- **Deduplication:** Identical files stored once
- **Integrity:** Hash verifies data isn't corrupted
- **Immutability:** Changing content = different hash = different object

Objects are sharded by the first two hex characters to avoid huge directories:

```
.pes/objects/
├── 2f/
│   └── 8a3b5c7d9e...
├── a1/
│   ├── 9c4e6f8a0b...
│   └── b2d4f6a8c0...
└── ff/
    └── 1234567890...
```

### Exploring a Real Git Repository

You can inspect Git's internals yourself:

```bash
mkdir test-repo && cd test-repo && git init
echo "Hello" > hello.txt
git add hello.txt && git commit -m "First commit"

find .git/objects -type f          # See stored objects
git cat-file -t <hash>            # Show type: blob, tree, or commit
git cat-file -p <hash>            # Show contents
cat .git/HEAD                     # See what HEAD points to
cat .git/refs/heads/main          # See branch pointer
```

---

## What You'll Build

PES-VCS implements five commands across four phases:

```
pes init              Create .pes/ repository structure
pes add <file>...     Stage files (hash + update index)
pes status            Show modified/staged/untracked files
pes commit -m <msg>   Create commit from staged files
pes log               Walk and display commit history
```

The `.pes/` directory structure:

```
my_project/
├── .pes/
│   ├── objects/          # Content-addressable blob/tree/commit storage
│   │   ├── 2f/
│   │   │   └── 8a3b...   # Sharded by first 2 hex chars of hash
│   │   └── a1/
│   │       └── 9c4e...
│   ├── refs/
│   │   └── heads/
│   │       └── main      # Branch pointer (file containing commit hash)
│   ├── index             # Staging area (text file)
│   └── HEAD              # Current branch reference
└── (working directory files)
```

### Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                      WORKING DIRECTORY                        │
│                  (actual files you edit)                       │
└───────────────────────────────────────────────────────────────┘
                              │
                        pes add <file>
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           INDEX                               │
│                (staged changes, ready to commit)              │
│                100644 a1b2c3... src/main.c                    │
└───────────────────────────────────────────────────────────────┘
                              │
                       pes commit -m "msg"
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                       OBJECT STORE                            │
│  ┌───────┐    ┌───────┐    ┌────────┐                         │
│  │ BLOB  │◄───│ TREE  │◄───│ COMMIT │                         │
│  │(file) │    │(dir)  │    │(snap)  │                         │
│  └───────┘    └───────┘    └────────┘                         │
│  Stored at: .pes/objects/XX/YYY...                            │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           REFS                                │
│       .pes/refs/heads/main  →  commit hash                    │
│       .pes/HEAD             →  "ref: refs/heads/main"         │
└───────────────────────────────────────────────────────────────┘
```

---
What I Implemented
object_write — builds the full object (header + data), computes SHA-256, deduplicates, shards into .pes/objects/XX/, and writes atomically using temp-file + rename pattern.
object_read — reads the object file, parses the header, recomputes the hash to verify integrity, and returns the data portion



## Phase 1: Object Storage Foundation

**Filesystem Concepts:** Content-addressable storage, directory sharding, atomic writes, hashing for integrity

**Files:** `pes.h` (read), `object.c` (implement `object_write` and `object_read`)

### What to Implement

Open `object.c`. Two functions are marked `// TODO`:

1. **`object_write`** — Stores data in the object store.
   - Prepends a type header (`"blob <size>\0"`, `"tree <size>\0"`, or `"commit <size>\0"`)
   - Computes SHA-256 of the full object (header + data)
   - Writes atomically using the temp-file-then-rename pattern
   - Shards into subdirectories by first 2 hex chars of hash

2. **`object_read`** — Retrieves and verifies data from the object store.
   - Reads the file, parses the header to extract type and size
   - **Verifies integrity** by recomputing the hash and comparing to the filename
   - Returns the data portion (after the `\0`)

Read the detailed step-by-step comments in `object.c` before starting.

### Testing

```bash
make test_objects
./test_objects
```

The test program verifies:
- Blob storage and retrieval (write, read back, compare)
- Deduplication (same content → same hash → stored once)
- Integrity checking (detects corrupted objects)

**📸 Screenshot 1A:** Output of `./test_objects` showing all tests passing:
<img width="944" height="452" alt="image" src="https://github.com/user-attachments/assets/d76f2382-fd6e-4a4f-bba6-47399104b9f7" />


**📸 Screenshot 1B:** `find .pes/objects -type f` showing the sharded directory structure:
<img width="1285" height="162" alt="image" src="https://github.com/user-attachments/assets/0fab31e7-d5d0-4f47-aac2-d82538bc77e2" />


---

## Phase 2: Tree Objects

**Filesystem Concepts:** Directory representation, recursive structures, file modes and permissions

**Files:** `tree.h` (read), `tree.c` (implement all TODO functions)

### What to Implement

Open `tree.c`. Implement the function marked `// TODO`:

1. **`tree_from_index`** — Builds a tree hierarchy from the index.
   - Handles nested paths: `"src/main.c"` must create a `src` subtree
   - This is what `pes commit` uses to create the snapshot
   - Writes all tree objects to the object store and returns the root hash

### Testing

```bash
make test_tree
./test_tree
```

The test program verifies:
- Serialize → parse roundtrip preserves entries, modes, and hashes
- Deterministic serialization (same entries in any order → identical output)

**📸 Screenshot 2A:** Output of `./test_tree` showing all tests passing:
<img width="1039" height="223" alt="image" src="https://github.com/user-attachments/assets/04af0ae6-1252-41ea-b74f-327c4ebbf60d" />

**📸 Screenshot 2B:** Pick a tree object from `find .pes/objects -type f` and run `xxd .pes/objects/XX/YYY... | head -20` to show the raw binary format:
<img width="1547" height="1017" alt="image" src="https://github.com/user-attachments/assets/ed070c2e-be5d-4640-897c-774758416f57" />


---

## Phase 3: The Index (Staging Area)

**Filesystem Concepts:** File format design, atomic writes, change detection using metadata

**Files:** `index.h` (read), `index.c` (implement all TODO functions)

### What to Implement

Open `index.c`. Three functions are marked `// TODO`:

1. **`index_load`** — Reads the text-based `.pes/index` file into an `Index` struct.
   - If the file doesn't exist, initializes an empty index (this is not an error)
   - Parses each line: `<mode> <hash-hex> <mtime> <size> <path>`

2. **`index_save`** — Writes the index atomically (temp file + rename).
   - Sorts entries by path before writing
   - Uses `fsync()` on the temp file before renaming

3. **`index_add`** — Stages a file: reads it, writes blob to object store, updates index entry.
   - Use the provided `index_find` to check for an existing entry

`index_find` , `index_status` and `index_remove` are already implemented for you — read them to understand the index data structure before starting.

#### Expected Output of `pes status`

```
Staged changes:
  staged:     hello.txt
  staged:     src/main.c

Unstaged changes:
  modified:   README.md
  deleted:    old_file.txt

Untracked files:
  untracked:  notes.txt
```

If a section has no entries, print the header followed by `(nothing to show)`.

### Testing

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index    # Human-readable text format
```

**📸 Screenshot 3A:** Run `./pes init`, `./pes add file1.txt file2.txt`, `./pes status` — show the output:
<img width="1366" height="607" alt="image" src="https://github.com/user-attachments/assets/a185d82a-5d30-4545-802b-c48da7e62c7d" />


**📸 Screenshot 3B:** `cat .pes/index` showing the text-format index with your entries:
<img width="986" height="53" alt="image" src="https://github.com/user-attachments/assets/dfe2a0a1-10d6-4276-9bc8-b3b19f6e76ce" />


---

## Phase 4: Commits and History

**Filesystem Concepts:** Linked structures on disk, reference files, atomic pointer updates

**Files:** `commit.h` (read), `commit.c` (implement all TODO functions)

### What to Implement

Open `commit.c`. One function is marked `// TODO`:

1. **`commit_create`** — The main commit function:
   - Builds a tree from the index using `tree_from_index()` (**not** from the working directory — commits snapshot the staged state)
   - Reads current HEAD as the parent (may not exist for first commit)
   - Gets the author string from `pes_author()` (defined in `pes.h`)
   - Writes the commit object, then updates HEAD

`commit_parse`, `commit_serialize`, `commit_walk`, `head_read`, and `head_update` are already implemented — read them to understand the commit format before writing `commit_create`.

The commit text format is specified in the comment at the top of `commit.c`.

### Testing

```bash
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

You can also run the full integration test:

```bash
make test-integration
```

**📸 Screenshot 4A:** Output of `./pes log` showing three commits with hashes, authors, timestamps, and messages:
<img width="818" height="255" alt="image" src="https://github.com/user-attachments/assets/249fc7e3-58d7-4606-8807-d7d00a5f6637" />


**📸 Screenshot 4B:** `find .pes -type f | sort` showing object store growth after three commits:
<img width="1504" height="1046" alt="image" src="https://github.com/user-attachments/assets/b56cb93f-1c40-41dd-a390-0febeae4fdba" />



**📸 Screenshot 4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain:

<img width="1600" height="738" alt="image" src="https://github.com/user-attachments/assets/3e15427b-3f82-4db9-a155-e9f04ff95e47" />
<img width="1022" height="1538" alt="image" src="https://github.com/user-attachments/assets/f476efb8-beb3-43f6-a171-a8e4a776f5d4" />
<img width="894" height="790" alt="image" src="https://github.com/user-attachments/assets/f1deae2b-a8ca-4d8a-83e7-a318094c84da" />




## Phase 5 & 6: Analysis-Only Questions:

The following questions cover filesystem concepts beyond the implementation scope of this lab. Answer them in writing — no code required.

### Branching and Checkout

Q5.1 — How would you implement pes checkout <branch>?
To implement pes checkout <branch>, the following steps are needed:

Files that change in .pes/:

.pes/HEAD must be updated to ref: refs/heads/<branch> to point to the new branch.
No other .pes/ files change — the branch ref file itself already exists (or must be created for a new branch).
What must happen to the working directory:

Read the current HEAD commit and walk its tree to get the current file snapshot.
Read the target branch's commit and walk its tree to get the target file snapshot.
Diff the two trees — for each file that exists in the target but not the current, write it to disk. For each file that exists in the current but not the target, delete it. For files that exist in both but with different blob hashes, overwrite with the target version.
What makes this complex:

Trees are recursive — you must walk subtrees depth-first to enumerate all files.
You must restore file permissions (mode) correctly from the tree entries.
You must handle the case where a path is a file in one branch and a directory in another.
The entire working directory sync must be atomic enough that a crash mid-checkout doesn't leave the repo in a broken state.
Untracked files must be preserved (don't delete files that aren't tracked in either branch).
Q5.2 — Detecting dirty working directory conflict
The algorithm uses only the index and the object store:

For each file in the current index, compare its mtime and size against the actual file on disk using stat(). If either differs, the file is locally modified (dirty).

For each dirty file, check whether it differs between branches: read the blob hash for that file from the current HEAD's tree, and the blob hash from the target branch's tree. If the two hashes differ, then the file has changed between branches AND has local modifications — this is a conflict.

If any such conflict exists, refuse the checkout and print an error.

This is efficient because it uses the mtime/size fast-path (same as git status) to avoid re-hashing every file. Only dirty files need their blob hashes compared across trees.

Q5.3 — Detached HEAD
When HEAD contains a commit hash directly instead of ref: refs/heads/<branch>, new commits are created and chained correctly, but no branch pointer is updated. The commits exist in the object store and are reachable from HEAD, but as soon as you switch to another branch, HEAD changes and the old commits become unreachable — no branch points to them.

Recovery: If you remember the commit hash (from terminal history or pes log output), you can create a new branch pointing to it:

# Create a branch at the detached commit before switching away
echo "<commit-hash>" > .pes/refs/heads/recovery-branch
If you've already switched away and lost the hash, the commits are still in the object store until garbage collection runs. You'd need to scan all objects in .pes/objects/ and find commit objects with no incoming references — essentially a manual GC traversal in reverse.

Phase 6: Analysis — Garbage Collection
Q6.1 — Algorithm to find and delete unreachable objects
Algorithm (Mark and Sweep):

Mark phase — start from all branch refs in .pes/refs/heads/. For each ref, read the commit hash and do a BFS/DFS traversal:

Add the commit hash to a reachable set.
Read the commit object, add its tree hash to reachable.
Recursively walk the tree: add all blob hashes and subtree hashes to reachable.
Follow the parent pointer and repeat until no parent exists.
Also include HEAD itself as a starting point.
Sweep phase — enumerate all files under .pes/objects/. For each file, reconstruct the hash from its path (XX + YYYY...). If the hash is not in the reachable set, delete the file.

Data structure: A hash set (e.g. a hash table or sorted array of 32-byte hashes) for O(1) or O(log n) membership checks.

Estimate for 100,000 commits and 50 branches:

Starting from 50 branch tips, each commit has 1 tree + ~N blobs. Assuming average 20 files per commit and 10% change rate between commits, roughly 10,000 unique trees and 50,000 unique blobs are reachable.
Total objects to visit: ~160,000 (100,000 commits + 10,000 trees + 50,000 blobs).
In the sweep phase, you'd enumerate all objects in the store — potentially more if there are unreachable ones.
Q6.2 — GC race condition with concurrent commit
The race condition:

GC starts its mark phase and determines that blob X is unreachable (no branch points to it yet).
Concurrently, a pes add writes blob X to the object store and a pes commit begins building a tree that references X.
GC's sweep phase runs and deletes blob X before the commit finishes writing the commit object.
The commit object is written pointing to a tree that points to blob X — but blob X no longer exists. The repository is now corrupt.
How Git avoids this:

Git uses a grace period (default 2 weeks) — objects newer than the grace period are never deleted by GC, even if they appear unreachable. This means a concurrent commit always has time to complete and create references before GC can touch the new objects. Git also writes a gc.log lock file to prevent two GC processes from running simultaneously. Additionally, Git's pack-refs and ref locking mechanisms ensure that refs are read atomically, so the mark phase sees a consistent snapshot of all reachable objects.


Summary
Phase	Files Modified	Status
1	object.c	✅ Complete
2	tree.c	✅ Complete
3	index.c	✅ Complete
4	commit.c	✅ Complete
5 & 6	Analysis	✅ Complete
