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
| `git remote`      | Manage remote repository connections. View, add, or remove remote repositories.                                                                                                     | List remotes: `git remote -v`.                             |
| `git push`        | Upload local commits to a remote repository.                                                                                                                                        | Push to remote: `git push origin main`.                    |
| `git pull`        | Fetch and merge changes from a remote repository to your current branch.                                                                                                            | Update local branch: `git pull origin main`.               |
| `git fetch`       | Download changes from remote without merging. Useful for reviewing before integrating.                                                                                              | Fetch all branches: `git fetch origin`.                    |
| `git tag`         | Create, list, or delete tags for marking specific commits (e.g., releases).                                                                                                         | Create tag: `git tag v1.0.0`.                              |
| `git show`        | Display detailed information about a commit, including changes made.                                                                                                                | Show commit details: `git show <commit-hash>`.             |
| `git blame`       | Show who last modified each line of a file and when.                                                                                                                                | Track changes: `git blame file.js`.                        |
| `git clean`       | Remove untracked files from the working directory.                                                                                                                                  | Remove untracked files: `git clean -fd`.                   |
| `git rm`          | Remove files from both the working directory and staging area.                                                                                                                      | Remove file: `git rm old-file.js`.                         |
| `git mv`          | Rename or move files while preserving Git history.                                                                                                                                  | Rename file: `git mv old.js new.js`.                       |
| `git reflog`      | View a log of all reference updates. Useful for recovering lost commits.                                                                                                            | View reflog: `git reflog`.                                 |

---

### 3. Working with Remote Repositories

Remote repositories are versions of your project hosted on the internet or network. They enable collaboration with other developers.

#### Common Remote Operations:

- **View Remote Repositories**:
  ```bash
  git remote -v
  ```
  This shows all configured remote repositories and their URLs.

- **Add a Remote Repository**:
  ```bash
  git remote add origin https://github.com/nikhil/my-project.git
  ```

- **Push Changes to Remote**:
  ```bash
  git push origin main
  ```
  Upload your local commits to the remote repository.

- **Push a New Branch**:
  ```bash
  git push -u origin feature-branch
  ```
  The `-u` flag sets up tracking between local and remote branches.

- **Pull Changes from Remote**:
  ```bash
  git pull origin main
  ```
  Fetch and merge changes from the remote repository.

- **Fetch Without Merging**:
  ```bash
  git fetch origin
  ```
  Download changes but don't merge them yet. Review with `git log origin/main`.

- **Remove a Remote**:
  ```bash
  git remote remove origin
  ```

- **Rename a Remote**:
  ```bash
  git remote rename old-name new-name
  ```

---

### 4. Understanding the Git Tree

The Git tree represents the history of commits. Each commit points to a snapshot of the project. Branches are pointers to commits, and `HEAD` points to the current branch.

**Diagram:**
```plaintext
main
  |
  o---o---o (HEAD)
         
          o feature-x
```

---

### 5. Git Stash Operations

The stash is a powerful feature for temporarily saving work without committing.

- **Stash Current Changes**:
  ```bash
  git stash
  ```
  Or with a message:
  ```bash
  git stash save "WIP: working on login feature"
  ```

- **List All Stashes**:
  ```bash
  git stash list
  ```
  Shows all saved stashes with their identifiers.

- **Apply Latest Stash**:
  ```bash
  git stash apply
  ```
  Applies stash but keeps it in the stash list.

- **Apply and Remove Latest Stash**:
  ```bash
  git stash pop
  ```

- **Apply Specific Stash**:
  ```bash
  git stash apply stash@{2}
  ```

- **Drop a Stash**:
  ```bash
  git stash drop stash@{0}
  ```

- **Clear All Stashes**:
  ```bash
  git stash clear
  ```

- **Show Stash Contents**:
  ```bash
  git stash show -p stash@{0}
  ```

---

### 6. Merge vs Rebase Explained with Examples

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

### 7. Git Reset Variations

The `git reset` command has different modes that affect your working directory, staging area, and commit history differently.

- **Soft Reset** (keeps changes staged):
  ```bash
  git reset --soft HEAD~1
  ```
  Moves HEAD back one commit but keeps changes in staging area. Useful for re-committing with a better message.

- **Mixed Reset** (default, keeps changes unstaged):
  ```bash
  git reset HEAD~1
  # or
  git reset --mixed HEAD~1
  ```
  Moves HEAD back and unstages changes, but keeps them in working directory.

- **Hard Reset** (discards all changes):
  ```bash
  git reset --hard HEAD~1
  ```
  ⚠️ **Warning**: This permanently deletes changes. Use with caution!

- **Reset to Specific Commit**:
  ```bash
  git reset --hard abc1234
  ```

---

### 8. Working with Specific Commits

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

- **View Commit Details**:
  ```bash
  git show abc1234
  ```
  Shows the commit message, author, date, and changes made.

- **Compare Two Commits**:
  ```bash
  git diff commit1 commit2
  ```

---

### 9. Git Tags for Version Management

Tags are used to mark specific points in history, typically for releases.

- **Create a Lightweight Tag**:
  ```bash
  git tag v1.0.0
  ```

- **Create an Annotated Tag** (recommended for releases):
  ```bash
  git tag -a v1.0.0 -m "Release version 1.0.0"
  ```

- **List All Tags**:
  ```bash
  git tag
  ```

- **Show Tag Details**:
  ```bash
  git show v1.0.0
  ```

- **Tag a Specific Commit**:
  ```bash
  git tag -a v0.9.0 abc1234 -m "Late tag for version 0.9.0"
  ```

- **Push Tags to Remote**:
  ```bash
  git push origin v1.0.0
  # or push all tags
  git push origin --tags
  ```

- **Delete a Tag Locally**:
  ```bash
  git tag -d v1.0.0
  ```

- **Delete a Tag from Remote**:
  ```bash
  git push origin --delete v1.0.0
  ```

- **Checkout a Tag**:
  ```bash
  git checkout v1.0.0
  ```

---

### 10. Git Squash & Cherry-Pick

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

- **Cherry-Pick Multiple Commits**:
  ```bash
  git cherry-pick abc1234 def5678
  ```

- **Cherry-Pick Without Committing**:
  ```bash
  git cherry-pick -n <commit-hash>
  ```
  Applies changes but doesn't create a commit, allowing you to modify before committing.

---

### 11. Undoing Changes

Different ways to undo changes depending on the state of your files:

- **Discard Changes in Working Directory**:
  ```bash
  git restore file.js
  # or for all files
  git restore .
  ```

- **Unstage a File**:
  ```bash
  git restore --staged file.js
  ```

- **Undo Last Commit (keep changes)**:
  ```bash
  git reset --soft HEAD~1
  ```

- **Undo Last Commit (discard changes)**:
  ```bash
  git reset --hard HEAD~1
  ```

- **Revert a Commit** (creates new commit):
  ```bash
  git revert HEAD
  # or specific commit
  git revert abc1234
  ```

- **Amend Last Commit**:
  ```bash
  git commit --amend -m "Updated commit message"
  # or add more changes to last commit
  git add forgotten-file.js
  git commit --amend --no-edit
  ```

---

### 12. Git Reflog - Recovering Lost Commits

The reflog tracks all changes to HEAD, making it possible to recover "lost" commits.

- **View Reflog**:
  ```bash
  git reflog
  ```

- **Recover a Lost Commit**:
  ```bash
  # Find the commit in reflog
  git reflog
  # Reset to that commit
  git reset --hard HEAD@{2}
  ```

- **Recover After Accidental Reset**:
  ```bash
  git reflog
  git checkout HEAD@{1}
  ```

---

### 13. Git Blame and History Tracking

- **See Who Modified Each Line**:
  ```bash
  git blame file.js
  ```

- **Blame with Line Range**:
  ```bash
  git blame -L 10,20 file.js
  ```

- **Show Author Email**:
  ```bash
  git blame -e file.js
  ```

- **Blame a Specific Commit**:
  ```bash
  git blame abc1234 file.js
  ```

---

### 14. Cleaning Up Untracked Files

- **Show What Would Be Removed** (dry run):
  ```bash
  git clean -n
  ```

- **Remove Untracked Files**:
  ```bash
  git clean -f
  ```

- **Remove Untracked Files and Directories**:
  ```bash
  git clean -fd
  ```

- **Remove Ignored Files Too**:
  ```bash
  git clean -fdx
  ```

---

### 15. Working with .gitignore

The `.gitignore` file tells Git which files or directories to ignore.

#### Creating a .gitignore File:

```bash
# Create .gitignore file
touch .gitignore
```

#### Common .gitignore Patterns:

```gitignore
# Dependencies
node_modules/
vendor/

# Build outputs
dist/
build/
*.exe
*.o

# Environment variables
.env
.env.local

# IDE files
.vscode/
.idea/
*.swp

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Temporary files
tmp/
temp/
*.tmp
```

#### Ignore Already Tracked Files:

If you've already committed a file and want to ignore it:
```bash
git rm --cached file.js
# Add file.js to .gitignore
echo "file.js" >> .gitignore
git commit -m "Stop tracking file.js"
```

---

### 16. Git Configuration

- **Check Commit Author**:
  ```bash
  git log --pretty=format:'%h %an %ae %s'
  ```
  This shows the author name and email for each commit.

- **Updating Global or Local Usernames**: Save your information globally or per repository:
  ```bash
  git config --global user.name "nikhil"
  git config --global user.email "nikhil@example.com"
  ```

- **Set Configuration for Current Repository Only**:
  ```bash
  git config user.name "nikhil"
  git config user.email "nikhil@example.com"
  ```

- **View All Configurations**:
  ```bash
  git config --list
  ```

- **View Specific Configuration**:
  ```bash
  git config user.name
  ```

- **Set Default Editor**:
  ```bash
  git config --global core.editor "vim"
  ```

- **Set Default Branch Name**:
  ```bash
  git config --global init.defaultBranch main
  ```

- **Enable Colorful Output**:
  ```bash
  git config --global color.ui auto
  ```

- **Set Up Aliases** (shortcuts for commands):
  ```bash
  git config --global alias.st status
  git config --global alias.co checkout
  git config --global alias.br branch
  git config --global alias.cm "commit -m"
  git config --global alias.lg "log --oneline --graph --all"
  ```

---

### 17. Collaboration Workflows

#### Feature Branch Workflow:

1. **Create a new feature branch**:
   ```bash
   git switch -c feature/login-page
   ```

2. **Make changes and commit**:
   ```bash
   git add .
   git commit -m "Add login page structure"
   ```

3. **Push branch to remote**:
   ```bash
   git push -u origin feature/login-page
   ```

4. **Keep branch updated with main**:
   ```bash
   git switch main
   git pull origin main
   git switch feature/login-page
   git rebase main
   ```

5. **Create Pull Request** on GitHub/GitLab

6. **Merge after review** (on GitHub/GitLab or locally):
   ```bash
   git switch main
   git merge feature/login-page
   git push origin main
   ```

7. **Delete merged branch**:
   ```bash
   git branch -d feature/login-page
   git push origin --delete feature/login-page
   ```

#### Forking Workflow:

1. **Fork repository** on GitHub

2. **Clone your fork**:
   ```bash
   git clone https://github.com/nikhil/forked-repo.git
   ```

3. **Add upstream remote**:
   ```bash
   git remote add upstream https://github.com/original/repo.git
   ```

4. **Keep fork synchronized**:
   ```bash
   git fetch upstream
   git switch main
   git merge upstream/main
   git push origin main
   ```

5. **Create feature branch and make changes**:
   ```bash
   git switch -c feature/new-feature
   # make changes
   git add .
   git commit -m "Add new feature"
   git push origin feature/new-feature
   ```

6. **Create Pull Request** from your fork to upstream

---

### 18. Git Hooks

Git hooks are scripts that run automatically at certain points in the Git workflow.

#### Common Hooks:

Located in `.git/hooks/` directory.

- **pre-commit**: Runs before a commit is created
  ```bash
  #!/bin/sh
  # .git/hooks/pre-commit
  npm run lint
  npm test
  ```

- **commit-msg**: Validates commit message format
  ```bash
  #!/bin/sh
  # .git/hooks/commit-msg
  commit_msg=$(cat $1)
  if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|test|chore):"; then
    echo "Error: Commit message must start with type (feat|fix|docs|...)"
    exit 1
  fi
  ```

- **pre-push**: Runs before pushing to remote
  ```bash
  #!/bin/sh
  # .git/hooks/pre-push
  npm run test
  ```

- **post-merge**: Runs after a merge
  ```bash
  #!/bin/sh
  # .git/hooks/post-merge
  npm install
  ```

#### Make Hook Executable:
```bash
chmod +x .git/hooks/pre-commit
```

---

### 19. Best Practices

1. **Commit Often, Push Regularly**:
   - Make small, logical commits
   - Push your work at least daily

2. **Write Meaningful Commit Messages**:
   ```bash
   # Good
   git commit -m "fix: resolve login authentication bug"
   
   # Bad
   git commit -m "fixed stuff"
   ```

3. **Use Branches for Features**:
   - Never commit directly to main/master
   - Create feature branches for new work

4. **Keep Commits Atomic**:
   - Each commit should represent one logical change
   - Makes it easier to review and revert

5. **Pull Before You Push**:
   ```bash
   git pull --rebase origin main
   git push origin main
   ```

6. **Review Changes Before Committing**:
   ```bash
   git diff
   git status
   ```

7. **Use .gitignore Properly**:
   - Don't commit dependencies, build files, or secrets
   - Keep repository clean

8. **Never Rebase Public Branches**:
   - Only rebase local/private branches
   - Rewriting public history causes problems for collaborators

9. **Use Tags for Releases**:
   ```bash
   git tag -a v1.0.0 -m "Release version 1.0.0"
   git push origin v1.0.0
   ```

10. **Regular Backups**:
    - Push to remote regularly
    - Consider multiple remotes for critical projects

---

### 20. Git Checkout vs Switch

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

### 21. Troubleshooting Common Issues

#### Merge Conflicts:

When Git can't automatically merge changes:
```bash
# After a merge conflict
git status  # See conflicted files
# Edit files to resolve conflicts
git add resolved-file.js
git commit -m "Resolve merge conflict"
```

#### Accidentally Committed to Wrong Branch:

```bash
# Move last commit to a new branch
git branch feature-branch
git reset --hard HEAD~1
git switch feature-branch
```

#### Undo a Push (before others pull):

```bash
git reset --hard HEAD~1
git push --force origin main
```
⚠️ **Warning**: Only use force push if no one else has pulled your changes!

#### Recover Deleted Branch:

```bash
git reflog
git checkout -b recovered-branch HEAD@{2}
```

#### Fix Wrong Commit Message:

```bash
# Last commit only
git commit --amend -m "Correct message"
git push --force origin branch-name
```

#### Stash Conflicts:

```bash
git stash pop
# If conflicts occur
git status  # See conflicted files
# Resolve conflicts manually
git add .
git stash drop  # Remove the stash
```

---

### 22. Advanced Git Commands

- **Interactive Staging**:
  ```bash
  git add -p file.js
  ```
  Allows you to stage parts of a file interactively.

- **Search Commit History**:
  ```bash
  git log --grep="login"
  ```

- **Find When a Bug Was Introduced** (bisect):
  ```bash
  git bisect start
  git bisect bad  # Current commit is bad
  git bisect good abc1234  # This commit was good
  # Git will checkout commits for you to test
  git bisect good  # or git bisect bad
  # Continue until bug is found
  git bisect reset
  ```

- **Show Files Changed in a Commit**:
  ```bash
  git diff-tree --no-commit-id --name-only -r abc1234
  ```

- **Count Commits by Author**:
  ```bash
  git shortlog -sn
  ```

- **Find Commits That Changed a Specific Line**:
  ```bash
  git log -L 10,20:file.js
  ```

- **Archive Repository**:
  ```bash
  git archive --format=zip --output=project.zip main
  ```

---

### 23. Summary

This comprehensive guide covers:
- **Basic Git commands** with real-world examples (init, clone, add, commit, status, diff)
- **Remote repository operations** (push, pull, fetch, remote management)
- **Branching and merging** strategies (merge vs rebase, conflict resolution)
- **Stash operations** for temporary work storage
- **Reset variations** (--soft, --mixed, --hard)
- **Working with commits** (cherry-pick, revert, amend, show)
- **Version management** with tags
- **Recovery operations** using reflog
- **History tracking** with blame
- **Cleaning operations** for untracked files
- **.gitignore** configuration and patterns
- **Git configuration** and aliases
- **Collaboration workflows** (feature branch, forking)
- **Git hooks** for automation
- **Best practices** for professional development
- **Troubleshooting** common issues
- **Advanced commands** for power users

This guide is designed to take you from beginner to advanced Git user, covering everything needed for modern software development workflows.

*Perfect for onboarding developers and as a comprehensive reference guide.*

---

### Quick Reference Cheat Sheet

```bash
# Configuration
git config --global user.name "nikhil"
git config --global user.email "nikhil@example.com"

# Starting a repository
git init
git clone <url>

# Basic workflow
git status
git add .
git commit -m "message"
git push origin main
git pull origin main

# Branching
git branch
git switch -c feature-branch
git merge feature-branch
git branch -d feature-branch

# Undoing
git restore file.js
git restore --staged file.js
git reset --soft HEAD~1
git revert HEAD

# Remote
git remote -v
git remote add origin <url>
git push -u origin main
git fetch origin

# Stashing
git stash
git stash pop
git stash list

# Viewing history
git log --oneline --graph
git show <commit-hash>
git reflog

# Tags
git tag v1.0.0
git push origin --tags
```
