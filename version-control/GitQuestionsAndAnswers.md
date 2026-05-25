# Git Interview Questions and Answers

A comprehensive collection of Git interview questions and answers — from beginner to advanced level. Covers Git fundamentals, commands, branching, merging, conflict resolution, and best practices.

---

## Table of Contents

- [Basic Git Questions](#basic-git-questions)
- [Git Commands & Operations](#git-commands--operations)
- [Branching & Merging](#branching--merging)
- [Advanced Git Concepts](#advanced-git-concepts)

---

## Basic Git Questions

### 1. What is Git?

**Answer:** Git is a distributed version control system (DVCS) that tracks changes in source code during software development. It allows multiple developers to work on a project simultaneously without interfering with each other's changes. Git is known for its speed, data integrity, and support for non-linear workflows.

**Key characteristics:**
- **Distributed:** Every developer has a complete copy of the repository
- **Fast:** Most operations are local and very fast
- **Reliable:** Uses SHA-1 hashing for data integrity
- **Flexible:** Supports both linear and non-linear development workflows

---

### 2. What is the difference between Git and GitHub?

**Answer:**

| Feature | Git | GitHub |
|---------|------|--------|
| **Type** | Version control system | Platform for hosting Git repositories |
| **Location** | Runs locally on your computer | Cloud-based web service |
| **Offline access** | Works offline | Requires internet connection |
| **Functionality** | Track changes, branch, merge | Collaborate, review, discuss, CI/CD |
| **Cost** | Free (open-source) | Free for public repos, paid for private |

**Analogy:** Git is like a word processor that tracks document versions. GitHub is like Google Drive where you store and share those documents with others.

---

### 3. What is a Git repository?

**Answer:** A Git repository (repo) is a directory containing all the files and project history that Git manages. It consists of:

- **Working directory:** Where you edit files
- **Staging area (index):** Where you prepare changes for commit
- **.git directory:** Contains all Git's metadata and object database

```bash
# Initialize a new repository
git init my-project
cd my-project

# Clone an existing repository
git clone https://github.com/user/repo.git
```

---

### 4. What does "origin" mean in Git?

**Answer:** "origin" is the default name for the remote repository from which you cloned. It's a convention, not a special keyword. You can have multiple remotes with different names.

```bash
# Check remotes
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)

# Add another remote
git remote add upstream https://github.com/original/repo.git
```

---

### 5. What is the purpose of the .gitignore file?

**Answer:** The `.gitignore` file tells Git which files and directories to ignore. This prevents:

- Temporary files from being committed
- Build artifacts (`.jar`, `.class`, `node_modules/`)
- IDE configuration files
- Sensitive data (credentials, API keys)
- OS-generated files (`.DS_Store`, `Thumbs.db`)

```gitignore
# Node.js
node_modules/
*.log

# Python
__pycache__/
*.pyc
venv/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Secrets
.env
*.key
```

---

### 6. What is the difference between `git init` and `git clone`?

**Answer:**

| Command | Purpose | Result |
|---------|---------|--------|
| `git init` | Create a new repository in current directory | Empty repo with no remote |
| `git clone` | Copy an existing remote repository | Full repo with remote configured |

```bash
# Start a new project
git init
git add .
git commit -m "Initial commit"

# Work on existing project
git clone https://github.com/user/repo.git
cd repo
```

---

### 7. What are the advantages of Git over SVN?

**Answer:**

| Feature | Git (Distributed) | SVN (Centralized) |
|---------|------------------|-------------------|
| **Offline work** | Yes - full local copy | No - requires server |
| **Branching** | Lightweight, fast | Heavyweight, slow |
| **Performance** | Fast (local operations) | Slower (network-dependent) |
| **History** | Complete local history | Only remote history |
| **Merging** | Advanced conflict resolution | Basic conflict resolution |
| **Storage** | Efficient (delta compression) | Less efficient |

---

## Git Commands & Operations

### 8. Explain the basic Git workflow.

**Answer:** Git has three main areas in its workflow:

```
Working Directory ──(git add)──► Staging Area ──(git commit)──► Repository
       ▲                                                     │
       │                                                     ▼
   (git checkout)                                    (git push/pull)
```

**Commands:**
- `git add <file>`: Move changes from working directory to staging area
- `git commit`: Save staged changes to repository
- `git push`: Send commits to remote repository
- `git pull`: Fetch and merge changes from remote

---

### 9. What is the difference between `git fetch` and `git pull`?

**Answer:**

| Command | Action | When to Use |
|---------|--------|------------|
| `git fetch` | Downloads changes from remote but doesn't merge | Review changes before merging |
| `git pull` | Downloads and automatically merges with current branch | Quick update, confident in changes |

```bash
# Safe approach - review first
git fetch origin
git diff main origin/main
git merge origin/main

# Quick approach
git pull origin main
```

---

### 10. What is `git stash` and when would you use it?

**Answer:** `git stash` temporarily saves uncommitted changes so you can work on something else.

**Common use cases:**
- Switch branches with uncommitted changes
- Pull latest changes before committing
- Save work-in-progress temporarily

```bash
# Save current changes
git stash

# List stashes
git stash list

# Apply most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{2}

# Create stash with message
git stash save "Work on feature X"
```

---

### 11. How do you view Git history?

**Answer:**

```bash
# Basic log
git log

# Compact view
git log --oneline

# Graph view with branches
git log --oneline --graph --decorate --all

# Show changes in each commit
git log --stat

# Search commits by message
git log --grep="fix"

# Show commits by specific author
git log --author="John Doe"

# Limit to last 5 commits
git log -5
```

---

### 12. What is the difference between `git reset`, `git revert`, and `git checkout`?

**Answer:**

| Command | Purpose | Effect |
|---------|---------|--------|
| `git reset` | Move branch pointer to previous commit | Changes history (destructive) |
| `git revert` | Create new commit that undoes previous commit | Preserves history |
| `git checkout` | Switch branches or restore files | Doesn't change history |

```bash
# Undo last commit (destructive)
git reset --hard HEAD~1

# Undo commit (safe)
git revert HEAD

# Switch to branch
git checkout feature-branch

# Restore file from last commit
git checkout HEAD -- file.txt
```

---

### 13. What is `git cherry-pick`?

**Answer:** `git cherry-pick` applies specific commits from one branch to another.

```bash
# Apply a specific commit
git cherry-pick abc1234

# Apply multiple commits
git cherry-pick abc1234..def5678

# Apply without committing
git cherry-pick --no-commit abc1234
```

**Use cases:**
- Apply hotfix to multiple branches
- Move specific commits between branches
- Backport features to older versions

---

### 14. What is a Git commit hash?

**Answer:** A commit hash is a 40-character SHA-1 hash that uniquely identifies a commit. It's calculated based on:
- Commit message
- Author and timestamp
- Parent commit(s)
- Tree object (file contents)

```bash
# Show commit hash
git rev-parse HEAD

# Show short hash (first 7 characters)
git rev-parse --short HEAD

# Find commit that introduced a bug
git bisect start
git bisect bad
git bisect good v1.0.0
```

---

## Branching & Merging

### 15. What is branching in Git?

**Answer:** Branching creates independent lines of development. Each branch has its own commit history but shares the same files.

**Benefits:**
- Parallel development
- Feature isolation
- Bug fixes without affecting main code
- Experimental work

```bash
# Create and switch to new branch
git checkout -b feature-x

# List branches
git branch -a

# Delete branch
git branch -d feature-x  # Safe (merged)
git branch -D feature-x  # Force (unmerged)
```

---

### 16. What is the difference between `git merge` and `git rebase`?

**Answer:**

| Feature | Merge | Rebase |
|---------|-------|--------|
| **History** | Creates merge commit | Linear history |
| **Complexity** | Simple to understand | More complex |
| **Conflict resolution** | Once at merge time | For each commit |
| **Best for** | Preserving exact history | Clean linear history |

```bash
# Merge - creates merge commit
git checkout main
git merge feature-branch

# Rebase - linear history
git checkout feature-branch
git rebase main
```

**Golden rule:** Never rebase commits that exist outside your local repository (pushed to shared branches).

---

### 17. How do you resolve merge conflicts?

**Answer:**

```bash
# 1. Git marks conflicts in files
<<<<<<< HEAD
Current branch content
=======
Incoming branch content
>>>>>>> feature-branch

# 2. Edit files to resolve conflicts
# Remove markers and keep desired content

# 3. Stage resolved files
git add resolved-file.txt

# 4. Continue merge
git commit  # Creates merge commit
```

**Tools for conflict resolution:**
- `git mergetool` - Launches visual merge tool
- `git diff --base` - Shows common ancestor
- `git diff --ours` - Shows your version
- `git diff --theirs` - Shows their version

---

### 18. What is a detached HEAD state?

**Answer:** A detached HEAD occurs when you checkout a specific commit instead of a branch. HEAD points directly to a commit, not to a branch.

```bash
# Create detached HEAD
git checkout abc1234

# Check status
git status
# HEAD detached at abc1234

# Fix by creating a branch
git checkout -b new-branch abc1234
```

**Use cases:**
- Inspect historical state
- Create branch from specific commit
- Bisect for bug hunting

---

### 19. What is the difference between `git merge` and `git merge --no-ff`?

**Answer:**

| Option | Description |
|--------|-------------|
| `git merge` | Fast-forward if possible (no merge commit) |
| `git merge --no-ff` | Always creates merge commit, even if fast-forward possible |

```bash
# Fast-forward (no merge commit)
git merge feature-branch

# Always create merge commit
git merge --no-ff feature-branch -m "Merge feature-branch"
```

**Use `--no-ff` to:**
- Preserve feature branch history
- Clearly indicate when features were merged
- Make it easier to revert features

---

## Advanced Git Concepts

### 20. What is Git's object model?

**Answer:** Git stores everything as objects in its `.git` directory:

| Object Type | Description |
|------------|-------------|
| **Blob** | File content (no filename) |
| **Tree** | Directory structure (maps filenames to blobs) |
| **Commit** | Points to a tree, parent commits, and metadata |
| **Tag** | Permanent reference to a commit |

```bash
# Explore Git objects
git cat-file -t abc1234  # Show object type
git cat-file -p abc1234  # Show object content
```

---

### 21. What are Git hooks?

**Answer:** Git hooks are scripts that run automatically at certain points in Git's lifecycle.

**Common hooks:**
- `pre-commit` - Runs before commit, can block commit
- `commit-msg` - Validates commit message format
- `pre-push` - Runs before pushing to remote
- `post-receive` - Runs on server after receiving push

```bash
# Example pre-commit hook
#!/bin/sh
# Check for TODO comments
if git diff --cached --name-only | xargs grep -l "TODO"; then
    echo "Remove TODO comments before committing"
    exit 1
fi
```

---

### 22. What is Git LFS (Large File Storage)?

**Answer:** Git LFS handles large files (images, videos, datasets) by storing them in a separate storage system while Git only stores pointers.

```bash
# Install Git LFS
git lfs install

# Track large files
git lfs track "*.pdf"
git lfs track "*.zip"
git add .gitattributes
git commit -m "Add LFS tracking"

# Push large files
git push origin main
```

**Benefits:**
- Smaller repository size
- Faster clone/pull operations
- Better handling of binary files

---

### 23. What is the Git reflog?

**Answer:** The reflog (reference log) records all movements of HEAD and branch references. It's a safety net for recovering lost commits.

```bash
# Show reflog
git reflog

# Show reflog for specific branch
git reflog main

# Recover lost commit
git checkout -b recovered-branch HEAD@{5}
```

**Use cases:**
- Restore commits after `git reset --hard`
- Find commits after branch deletion
- Track branch movements

---

### 24. What is a bare repository?

**Answer:** A bare repository contains only the `.git` directory without a working directory. It's used for central repositories.

```bash
# Create bare repository
git init --bare my-project.git

# Clone from bare repository
git clone my-project.git my-project
```

**Characteristics:**
- No working directory
- Cannot edit files directly
- Used for shared repositories
- Typically named with `.git` suffix

---

### 25. What are Git submodules?

**Answer:** Submodules allow you to include a Git repository as a subdirectory of another Git repository.

```bash
# Add submodule
git submodule add https://github.com/user/library.git libs/library

# Clone repository with submodules
git clone --recurse-submodules https://github.com/user/project.git

# Update submodules
git submodule update --init --recursive
```

**Use cases:**
- Include third-party libraries
- Share common code across projects
- Manage dependencies

---

### 26. How do you squash multiple commits into one?

**Answer:**

```bash
# Interactive rebase to squash last 3 commits
git rebase -i HEAD~3

# In editor, change 'pick' to 'squash' for commits to merge:
pick abc1234 First commit
squash def5678 Second commit
squash ghi9012 Third commit

# Save and exit to combine commits
```

**Alternative:**
```bash
# Soft reset and recommit
git reset --soft HEAD~3
git commit -m "Combined commit message"
```

---

### 27. What is the difference between `git reset --soft`, `--mixed`, and `--hard`?

**Answer:**

| Option | Staging Area | Working Directory |
|--------|-------------|-------------------|
| `--soft` | Keeps changes | Keeps changes |
| `--mixed` (default) | Clears changes | Keeps changes |
| `--hard` | Clears changes | Clears changes |

```bash
# Remove last commit but keep changes staged
git reset --soft HEAD~1

# Remove last commit and unstage changes
git reset HEAD~1

# Remove last commit and discard changes
git reset --hard HEAD~1
```

---

### 28. How do you find which commit introduced a bug?

**Answer:** Use `git bisect` for binary search through commit history:

```bash
# Start bisect
git bisect start

# Mark current commit as bad
git bisect bad

# Mark known good commit
git bisect good v1.0.0

# Git will checkout middle commit
# Test the code, then mark as good or bad
git bisect bad  # or git bisect good

# Continue until bug is found
# Git will show the exact commit that introduced the bug

# End bisect session
git bisect reset
```

---

### 29. What is the difference between HEAD, working tree, and index?

**Answer:**

| Concept | Description |
|---------|-------------|
| **HEAD** | Pointer to current commit (last commit on current branch) |
| **Working tree** | Files you see and edit in your filesystem |
| **Index (staging area)** | Area where you prepare changes for next commit |

```
Working Directory ──(git add)──► Index ──(git commit)──► HEAD
       ▲                                          │
       │                                          ▼
   (git checkout)                           (git reset)
```

---

### 30. How do you configure Git for a specific project?

**Answer:**

```bash
# Set local configuration (project-specific)
git config user.name "Your Name"
git config user.email "your.email@example.com"
git config core.autocrlf input  # Windows line endings
git config core.editor "vim"     # Default editor

# Set global configuration (all projects)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# View configuration
git config --list
git config --list --local
git config --list --global
```

**Local vs Global:**
- Local: Stored in `.git/config` (project-specific)
- Global: Stored in `~/.gitconfig` (user-specific)
- System: Stored in `/etc/gitconfig` (all users)
