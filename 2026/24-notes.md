# Day 24 – Advanced Git: Merge, Rebase, Stash & Cherry-Pick

## Table of Contents

- [Task 1: Git Merge](#task-1-git-merge)
- [Task 2: Git Rebase](#task-2-git-rebase)
- [Task 3: Squash Merge](#task-3-squash-merge)
- [Task 4: Git Stash](#task-4-git-stash)
- [Task 5: Cherry-Pick](#task-5-cherry-pick)

---

## Task 1: Git Merge

### What is a fast-forward merge?

When `main` has no new commits since the feature branch was created, Git moves the `main` pointer forward to the feature branch tip. No merge commit is created and history stays linear.
```
Before:               After:
A (main)              A─B─C (main, feature)
└─B─C (feature)
```

### When does Git create a merge commit?

When both branches have diverged. Git creates a new commit to join the histories. Force with `git merge --no-ff`.
```
Before:              After:
A─D (main)           A─D────M (main)
└─B─C (feature)         └─B─C┘
```

### What is a merge conflict?

Occurs when two branches edit the same lines of the same file. Git halts the merge, inserts conflict markers (`<<<<<<<` / `=======` / `>>>>>>>`), and waits for manual resolution.

**Resolution:**
```bash
# Edit file to resolve conflicts
git add filename
git commit
```

---

## Task 2: Git Rebase

### What does rebase do?

Rebase replays commits one-by-one on top of a new base. Each replayed commit gets a new SHA hash because its parent changed.
```
Before:              After:
A─D (main)           A─D─B'─C' (feature)
└─B─C (feature)
```

### Rebase vs Merge

| Merge | Rebase |
|-------|--------|
| Preserves branching history | Creates linear history |
| Adds merge commit | No extra commit |
| Shows divergence | Hides divergence |
| Safe on shared branches | Dangerous on shared branches |

### Why never rebase pushed commits?

Rebase creates new commits with new SHAs. If teammates already pulled your old commits, their history diverges from the rewritten remote. Requires `git push --force` which can overwrite their work.

**Golden Rule: Only rebase local commits.**

### When to use rebase vs merge?

- **Rebase:** Tidying local feature branch before merging (solo, local-only)
- **Merge:** Integrating into shared/team branch
- **Best practice:** Rebase locally, merge via Pull Request

---

## Task 3: Squash Merge

### What does squash merge do?

`git merge --squash` collapses all feature branch commits into a single staged diff. Does NOT auto-commit. Individual commit history never appears in `main`.
```bash
git merge --squash feature-branch
git commit -m "feat: add complete feature"
```

### Squash vs Regular Merge

| Squash Merge | Regular Merge |
|--------------|---------------|
| Messy WIP commits | Clean, meaningful commits |
| Clean main history | Full audit trail |
| Bug fixes, small chores | Large features, collaboration |

### Trade-offs

**Pros:** Clean main history, one commit per feature  
**Cons:** Lost granular history, harder to debug with `git bisect`, less useful `git blame`

---

## Task 4: Git Stash

### Pop vs Apply

| Command | Behavior |
|---------|----------|
| `git stash pop` | Apply and delete from stash list |
| `git stash apply` | Apply only, keep in stash list |

### Common Commands
```bash
git stash push -m "description"
git stash list
git stash pop
git stash apply stash@{0}
git stash drop stash@{0}
git stash clear
```

### When to use stash

Use when unfinished work blocks branch switching:
- Emergency hotfix
- Wrong branch commits
- Pulling updates with dirty working tree
- Helping teammate
- Clean-state testing

---

## Task 5: Cherry-Pick

### What does cherry-pick do?

`git cherry-pick <commit-hash>` takes a single specific commit from anywhere in history and re-applies it as a new commit on your current branch.

**Key points:**
- Creates new commit with new SHA
- Only applies that commit's changes
- Does not bring other commits
- Supports ranges: `git cherry-pick A^..C`
```bash
git log --oneline
git cherry-pick abc1234
```

### When to use cherry-pick

1. **Emergency production hotfix** - Apply critical fix to main without merging entire develop branch
2. **Backporting** - Apply security patch from v2.0 to v1.x maintenance branch
3. **Salvaging work** - Rescue valuable commit from abandoned branch
4. **Wrong-branch commits** - Move commit from wrong branch to correct one
5. **Selective deployment** - Deploy specific improvement to staging for testing

### What can go wrong

| Problem | Description |
|---------|-------------|
| **Duplicate commits** | Same change appears twice with different SHAs if source branch is later merged |
| **Context dependency** | Commit may depend on other commits not on target branch, causing silent bugs |
| **Merge conflicts** | Diverged target branch may conflict with replayed commit |
| **Lost context** | Commit separated from related commits and PR discussions |
| **History pollution** | Overuse creates fragmented, difficult-to-read history |

### Best Practices

- Use for surgical, one-off situations (hotfixes, backports)
- Document why you cherry-picked in commit message
- Prefer proper merges for regular workflow
- Don't use as substitute for good branching strategy
- Avoid if planning to merge source branch later

**Cherry-pick is a precision tool for emergencies, not daily workflow.**

---

## Quick Command Reference
```bash
# Merge
git merge feature-branch
git merge --no-ff feature-branch
git merge --squash feature-branch

# Rebase
git rebase main
git rebase --continue
git rebase --abort

# Stash
git stash push -m "description"
git stash list
git stash pop
git stash apply stash@{0}
git stash drop stash@{0}

# Cherry-Pick
git cherry-pick abc1234
git cherry-pick A^..C
git cherry-pick --continue
git cherry-pick --abort

# Visualization
git log --oneline --graph --all
```

---