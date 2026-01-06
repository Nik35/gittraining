# Comprehensive Guide to Git

Welcome to this Git Training Resource! This guide will cover everything from the basics to advanced Git techniques. Whether you're a beginner or looking to deepen your Git knowledge, this guide will help you.

---

## Table of Contents

1. [Getting Started with Git](#getting-started-with-git)
    - What is Git?
    - Installing Git
    - Set up Git Configuration
2. [Git Basics](#git-basics)
    - git init
    - git clone
    - git add
    - git commit
    - git status
    - git log
3. [Branching and Merging](#branching-and-merging)
    - git branch
    - git checkout
    - git switch
    - git merge
    - git rebase
4. [Working with Remotes](#working-with-remotes)
    - git remote
    - git fetch
    - git pull
    - git push
5. [Advanced Git Techniques](#advanced-git-techniques)
    - Stashing Changes (git stash)
    - Annotated Tags (git tag)
    - Reset, Revert, and Restore
    - The Reflog (git reflog)
    - Cherry-picking (git cherry-pick)
    - Using git read-tree
6. [Git Workflows](#git-workflows)
    - Gitflow Workflow
    - Forking Workflow
    - Feature Branch Workflow
7. [Git Troubleshooting](#git-troubleshooting)
    - Resolving Merge Conflicts
    - Recovering Lost Commits with git reflog
8. [Illustrations and Examples](#illustrations-and-examples)

---

## Getting Started with Git

### What is Git?
Git is a distributed version control system designed to handle everything from small to very large projects with speed and efficiency.

### Installing Git
To install Git, follow the instructions on the [official Git website](https://git-scm.com/).

### Set up Git Configuration
Run these commands to configure Git:
```bash
# Set your username
$ git config --global user.name "Your Name"

# Set your email
$ git config --global user.email "youremail@example.com"
```

---

## Git Basics

### `git init`
The `git init` command creates a new Git repository.
```bash
$ git init
```
*Example:*
Run this in an empty directory to initialize a new Git repository.

### `git clone`
The `git clone` command copies a repository to your machine.
```bash
$ git clone <repository_url>
```
*Example:*
If you want to clone the public GitHub repository located at `https://github.com/Nik35/gittraining`, use:
```bash
$ git clone https://github.com/Nik35/gittraining.git
```

![Cloning a Repository](https://via.placeholder.com/800x400.png)

### `git add`
Stage changes for the next commit.
```bash
$ git add <file>
```
*Example:*
```bash
$ git add README.md
```
This tells Git to stage `README.md` so it can be committed later.

### `git commit`
Record the changes with a commit message.
```bash
$ git commit -m "Your commit message"
```
*Example:*
```bash
$ git commit -m "Add initial version of README file"
```

### `git status`
Check the status of your repository.
```bash
$ git status
```
*Example:*
```bash
$ git status
On branch main
Your branch is up to date with 'origin/main'.
Changes to be committed:
    modified: README.md
```

### `git log`
View the commit history.
```bash
$ git log
```
*Example:*
```bash
$ git log --oneline
1f94060 Add README file
9bf6ea6 Create a new comprehensive Git training guide
```

---

## Branching and Merging

### `git branch`
List or create branches.
```bash
$ git branch
```
*Example:*
```bash
$ git branch feature-new
```
This creates a branch named `feature-new`.

### `git switch`
Move between branches.
```bash
$ git switch <branch_name>
```
*Example:*
```bash
$ git switch feature-new
```

### `git merge`
Combine changes from one branch with another.
```bash
$ git merge <branch_name>
```
*Example:*
```bash
$ git merge feature-new
```
This merges `feature-new` into the current branch.

### `git rebase`
Reapply commits on top of another branch.
```bash
$ git rebase <branch_name>
```
*Example:*
```bash
$ git rebase main
```
This reapplies the current branch's commits on top of `main`.

![Rebase vs Merge](https://via.placeholder.com/800x400.png)

### `git read-tree`
Read tree structure from another branch or commit.
```bash
$ git read-tree <ref>
```
*Example:*
```bash
$ git read-tree -m -u HEAD
```
This updates the working tree with the index contents for branch `HEAD`.

---

## Advanced Git Techniques

### `git stash`
Temporarily save changes.
```bash
$ git stash
```

### `git reflog`
View and recover lost commits.
```bash
$ git reflog
```

---

*Content expanded to include examples where relevant...*