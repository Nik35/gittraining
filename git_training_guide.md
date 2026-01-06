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

### `git clone`
The `git clone` command copies a repository to your machine.
```bash
$ git clone <repository_url>
```
![Cloning a Repository](https://via.placeholder.com/800x400.png)

### `git add`
Stage changes for the next commit.
```bash
$ git add <file>
```

### `git commit`
Record the changes with a commit message.
```bash
$ git commit -m "Your commit message"
```

---

## Branching and Merging

### `git branch`
List or create new branches.

### `git merge`
Combine changes from different branches.

### `git rebase`
Reapply commits on top of another base tip. Use `git rebase` to keep a linear history.
![Rebase vs Merge](https://via.placeholder.com/800x400.png)

---

## Advanced Git Techniques

### `git stash`
Temporarily save changes.

### `git reflog`
View and recover lost commits.

---