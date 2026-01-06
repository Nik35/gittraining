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

### 2. Essential Git Commands with Detailed Examples

This section provides detailed explanations and real-world examples of popular Git commands. Each command is shown in scenarios that developers often encounter.

| **Command**       | **Description**                                                                                                                                                                         | **Example**                                                 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| `git init`        | Initialize a new Git repository in your project folder. This adds a `.git` directory that tracks your changes.                                                                  | `git init` inside a project folder. Default branch: `main`. |
| `git clone`       | Clone (download) a remote repository to your local machine. It includes all history.                                                                                                 | `git clone https://github.com/user/repo.git`                |
| `git status`      | Show the current state of your repository, including staged, unstaged, and untracked files.                                                                                          | Run `git status` to check for files with pending changes.   |
| `git add`         | Stage files for the next commit. Staged files will be included when you run the `git commit` command.                                                                                | Stage individual files: `git add file1.js`.                |
| `git restore --staged` | Unstage a file previously added using `git add`.                                                                                                                                 | `git restore --staged file.js` (unstages the file).         |
| `git commit`      | Save staged changes, creating a snapshot (commit) in Git history, with a unique ID (SHA).                                                                                           | Commit staged files: `git commit -m "Add login feature"`. |
| `git log`         | View the history of commits. You can use options to customize logs.                                                                                                                 | `git log --oneline --graph` for a summary.                 |
| `git diff`        | Show differences between files. Useful to verify changes before staging.                                                                                                           | `git diff HEAD` to compare changes since the last commit.   |
| `git branch`      | View current branches or create a new one.                                                                                                                                          | List branches: `git branch`.                                |
| `git switch`      | Switch or create a new branch. This is simpler than older alternatives.                                                                                                              | Switch branches: `git switch feature-x`.                   |
| `git checkout`    | Switch branches or restore files. It works similarly to `git switch`, but has additional use cases in older Git versions.                                                           | Switch branches: `git checkout feature-x`.                 |
| `git checkout <hash>` | Check out a specific commit by its hash value (non-destructive).                                                                                                                | `git checkout abc1234`                                      |
| `git checkout -b` | Create and move to a new branch simultaneously.                                                                                                                                      | `git checkout -b test-branch`.                             |
| `git merge`       | Combine another branch's history into your current branch.                                                                                                                          | Merge feature branch: `git merge feature-x`.               |
| `git rebase`      | Replay commits from one branch onto another to create a linear history.                                                                                                             | `git rebase main`. Resolves conflicts interactively.        |
| `git stash`       | Temporarily save changes in the working directory for a clean slate.                                                                                                                | Use `git stash` before switching branches.                 |
| `git stash pop`   | Apply previously stashed changes.                                                                                                                                                    | `git stash pop`.                                            |
| `git cherry-pick` | Apply a specific commit from another branch to your current branch.                                                                                                                | `git cherry-pick ff612h83`.                                |
| `git revert`      | Undo specific commits by creating new ones.                                                                                                                                        | `git revert HEAD`.                                          |
| `git reset HEAD~1`| Undo your last commit without losing changes to files.                                                                                                                              | Discard last commit: `git reset --soft HEAD~1`.            |

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

### 4. Merge vs Rebase Explained with Examples

- **Merge**: Combines histories, creates a merge commit.
  Used when multiple developers work in parallel and it's essential to preserve everyone’s history.
  ```bash
  git merge feature-x
  ```

- **Rebase**: Rewrites history, applies commits on top of another branch.
  Suitable for keeping histories clean during development.
  ```bash
  git rebase main
  ```

#### Conflict Example and Resolution:
When both branches modify the same file:
```bash
# Resolve conflicts manually
<<<<<<< HEAD
Your changes...
=======
Incoming changes...
>>>>>>> feature-x
```
Use `git add <filename>` and continue the process.

---

### 5. Working with Specific Commits

- **Check out a specific commit**
  ```bash
  git checkout <commit-hash>
  ```
  This command switches your working directory to the state at that commit. Use this for debugging or temporarily reviewing distant history.

- **Restore a file from a specific commit**
  Restore file `index.html` from commit `abc1234` into the current branch:
  ```bash
  git checkout abc1234 -- index.html
  ```

---

### 6. Git Squash & Cherry-Pick

- **Squashing Commits**:
  Use squashing to make your history cleaner. Squashing is useful during pull request preparation or before merging a feature branch.
  ```bash
  git rebase -i HEAD~n
  # Mark commits as 'squash (s)' except the first one
  ```

- **Cherry-Picking a Commit**:
  Apply a beneficial commit from one branch into another without merging branch histories.
  ```bash
  git cherry-pick <commit-hash>
  ```

---

### 7. Git Configuration

- **Check Commit Author**:
  ```bash
  git log --pretty=format:'%h %an %ae %s'
  ```
  This shows the author name and email for each commit.

- **Updating Global or Local Usernames**: Save your information globally or per repository:
  ```bash
  git config --global user.name "John Doe"
  git config --global user.email "john@example.com"
  ```

---

### 8. Git Checkout vs Switch

The `git switch` command was introduced as a simpler alternative to `git checkout` for branch operations.

#### Key Differences:
| **Command**        | **Advantages**                  |
|---------------------|---------------------------------|
| `git switch`       | Cleaner, specific to switching branches.             |
| `git checkout`     | Supports file-level checkouts alongside branch switching. |

#### Examples of Use:
- **Switch to Branch**:
  ```bash
  git switch main
  git checkout main
  ```
- **Create a New Branch**:
  ```bash
  git switch -c new-feature
  git checkout -b new-feature
  ```

---

### 9. Summary

This guide extensively covers:
- Git commands with real-world examples.
- Selecting the right commands (`switch` vs `checkout`, etc.).
- Handling specific scenarios like cherry-picking, squashing commits.
- Checking out and restoring files from specific commits.
- Understanding Git workflows (merge/rebase).

*Perfect for onboarding developers.*