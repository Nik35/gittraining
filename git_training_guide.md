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
| `git status`      | Show current changes                | `git status`                   |
| `git add`         | Stage changes                       | `git add file.txt`             |
| `git commit`      | Commit staged changes               | `git commit -m "Initial commit"` |
| `git log`         | View commit history                 | `git log`                      |
| `git diff`        | Show file differences               | `git diff`                     |
| `git branch`      | List/create branches                | `git branch feature-x`         |
| `git checkout`    | Switch branches                     | `git checkout feature-x`       |
| `git merge`       | Merge branches                      | `git merge feature-x`          |
| `git rebase`      | Reapply commits on top of another base | `git rebase main`              |
| `git stash`       | Temporarily save changes            | `git stash`                    |
| `git stash pop`   | Reapply stashed changes             | `git stash pop`                |
| `git reset`       | Undo commits (soft/mixed/hard)      | `git reset --soft HEAD~1`      |
| `git revert`      | Create a new commit to undo changes | `git revert <commit-hash>`     |

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

### 5. Undoing Changes

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

### 6. Git Configuration

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

### 7. Local vs Remote Repositories

- **Local Repo**: Your working directory + `.git` folder
- **Remote Repo**: Hosted on GitLab (or similar)

**Diagram:**
```plaintext
[Working Directory] <--> [.git Directory] <--> [Remote Repository]
```

---

### 8. Additional Tips

- Use `.gitignore` to exclude files
- Use `git tag` to mark releases
- Use `git fetch` and `git pull` to sync with remote
- Use `git push` to upload changes

---

### 9. Summary

This guide covers:
- Git basics and setup
- Core commands with examples
- Visual understanding of the Git tree
- Merge vs Rebase with conflict resolution details
- Undoing changes safely
- Configuration and repo structure

*Perfect for onboarding new developers using Git via CLI with GitLab.*