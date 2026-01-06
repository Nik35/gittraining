# Comprehensive Git Training Guide for New Developers

### 1. Introduction to Git

#### What is Git?

Git is a distributed version control system used to track changes in source code. It allows multiple developers to work on the same project without overwriting each other’s work.

#### Why Use Git?
- **Collaboration**: Multiple developers can contribute simultaneously.
- **History**: Track every change made to the codebase.
- **Branching**: Work on new features without affecting the main code.
- **Backup**: Code is stored both locally and remotely.

---

### 2. Essential Git Commands with Examples

| **Command**       | **Description**                     | **Example**                    |
|--------------------|-------------------------------------|---------------------------------|
| `git init`        | Initialize a new Git repository     | `git init`                     |
| `git clone`       | Clone a remote repository           | `git clone https://gitlab.com/user/repo.git` |
| `git status`      | Show the working state of your directory and staged changes | `git status` |
| `git add`         | Stage changes to be committed       | `git add file.txt`             |
| `git unstage`     | Remove changes from staging area    | `git restore --staged file.txt`|
| `git commit`      | Commit staged changes               | `git commit -m "Initial commit"` |
| `git log`         | View commit history                 | `git log`                      |
| `git diff`        | Show file differences               | `git diff`                     |
| `git branch`      | List/create branches                | `git branch feature-x`         |
| `git switch`      | Switch branches (cleaner than checkout) | `git switch feature-x`    |
| `git checkout`    | Switch branches (legacy command)    | `git checkout feature-x`       |
| `git checkout -b` | Create and switch to a new branch   | `git checkout -b new-feature`  |
| `git merge`       | Merge branches                      | `git merge feature-x`          |
| `git rebase`      | Reapply commits on top of another base | `git rebase main`              |
| `git squash`      | Combine multiple commits into one   | Interactive rebase with squash: `git rebase -i HEAD~3` |
| `git stash`       | Temporarily save changes            | `git stash`                    |
| `git cherry-pick` | Apply a specific commit to a branch | `git cherry-pick abc1234`      |
| `git revert`      | Create a new commit to undo changes | `git revert <commit-hash>`     |
| `git reset`       | Undo commits (soft/mixed/hard)      | `git reset --soft HEAD~1`      |

---

### 3. Understanding the Git Tree

The Git tree represents the history of commits. Each commit points to a snapshot of the project. Branches are pointers to commits, and `HEAD` points to the current branch.

**Diagram:**
```plaintext
main
  |
  o---o---o (HEAD)
         
          o feature-x
```

---

### 4. Merge vs Rebase

- **Merge**: Combines histories, creates a merge commit.
  ```bash
  git merge feature-x
  ```
- **Rebase**: Rewrites history, applies commits on top of another branch.
  ```bash
  git rebase main
  ```

#### When to Use:
- Use **merge** for preserving history.
- Use **rebase** for a cleaner, linear history.

#### Conflict Example and Resolution:
When both branches modify the same line or conflicting changes exist, Git will pause the rebase or merge and mark the conflicted files.

1. To resolve conflicts during a rebase:
   - Open the conflicted file(s).
   - Manually edit the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) to resolve the differences.
   - Stage the resolved files: `git add <conflicted-file>`
   - Continue the rebase process: `git rebase --continue`
   - If you want to abort the rebase and return to the original state: `git rebase --abort`

2. For merge conflicts, the process is similar:
   - Resolve conflicts manually in the files.
   - Stage the resolved files with `git add`.
   - Complete the merge with: `git commit`

*Always review the changes carefully before continuing.*

---

### 5. Git Internals: Commit Unique ID Explained

Each commit has a unique ID known as a hash. This SHA-1 hash ensures the uniqueness of all commits and helps in identifying them. The hash is created using the commit's content, timestamp, and parent commit.

---

### 6. Undoing Changes

- **`git reset`**: Reset changes
  - `--soft`: Keeps changes staged
  - `--mixed`: Keeps changes in the working directory
  - `--hard`: Discards all changes

- **`git revert`**: Safely undo a commit by creating a new one
  ```bash
  # Undo last commit but keep changes
  git reset --soft HEAD~1

  # Revert a commit
  git revert abc1234
  ```

---

### 7. Git Squash & Cherry-Pick

- **Squashing Commits**:
  Combine multiple commits into one to clean up your branch history.
  ```bash
  git rebase -i HEAD~n
  # Mark commits as squash (s) except the first one
  # Save and close editor to apply
  ```

- **Cherry-Picking**:
  Introduce changes from a specific commit into another branch without merging entire branch histories.
  ```bash
  git cherry-pick <commit-hash>
  ```

---

### 8. Git Configuration and Author Details

- **Check Commit Author**:
  To verify the author of a commit:
  ```bash
  git log --pretty=format:'%h %an %ae %s'
  ```

- **Global Config**: Applies to all repos
  ```bash
  git config --global user.name "Nikhil"
  git config --global user.email "nikhil@example.com"
  ```
- **Repo-Specific Config**: Applies only to the current repo
  ```bash
  git config user.name "ProjectUser"
  ```

---

### 9. Git Checkout vs Switch

The `git switch` command was introduced as a simpler alternative to `git checkout`.

- **Switch Branches**:
  ```bash
  git switch feature-x  # Switch to an existing branch
  git checkout feature-x  # Equivalent legacy command
  ```
- **Create & Switch to a New Branch**:
  ```bash
  git switch -c new-feature  # New command
  git checkout -b new-feature  # Equivalent legacy command
  ```

| **Command**        | **Advantages**                  |
|---------------------|---------------------------------|
| `git switch`       | Cleaner, intuitive             |
| `git checkout`     | Supports other actions (e.g. checking out a file) |

---

### 10. Summary

This guide covers:
- Git basics and setup
- Core commands with examples, including new ones like `git switch`
- Git internals like commit hashing and unique IDs
- Merge vs Rebase with conflict resolution details
- Advanced use cases: squashing commits, cherry-picking
- Configuration and author checks
- The difference between `switch` and `checkout`

*Perfect for onboarding new developers using Git via CLI with GitLab.*