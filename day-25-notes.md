# Day 25 – Git Reset vs Revert & Branching Strategies

## Task 1: Git Reset

### What is the difference between --soft, --mixed, and --hard?

| Flag | Commit History | Staging Area | Working Directory |
|------|---------------|--------------|-------------------|
| --soft | Reset | Unchanged | Unchanged |
| --mixed | Reset | Reset | Unchanged |
| --hard | Reset | Reset | Reset |

### Which one is destructive and why?

--hard is destructive because it discards all changes in both staging area and working directory. Once executed, uncommitted work is permanently lost unless recovered via git reflog.

### When would you use each one?

**--soft:** Undo commits but keep changes staged. Useful for:
- Combining multiple commits into one (re-commit after reset)
- Fixing commit message of recent commits

**--mixed:** Undo commits and unstage changes. Useful for:
- Redoing staging area (reorganize what goes in next commit)
- Accidentally committed too many files

**--hard:** Completely discard commits and changes. Useful for:
- Abandoning experimental work
- Reverting to a clean state
- Starting over from a specific commit

### Should you ever use git reset on pushed commits?

NO. Never use git reset on commits already pushed to a shared branch.

Why:
- Rewrites history
- Causes divergence with teammates' local copies
- Requires force push
- Can lose others' work
- Breaks collaborative workflow

Safe alternative: Use git revert for pushed commits.

---

## Task 2: Git Revert

### How is git revert different from git reset?

| Aspect | git reset | git revert |
|--------|-----------|------------|
| History | Rewrites history (removes commits) | Preserves history (adds new commit) |
| Method | Moves branch pointer backward | Creates inverse commit |
| Collaboration | Dangerous on shared branches | Safe on shared branches |
| Trace | Original commit disappears | Original commit remains visible |

### Why is revert considered safer than reset for shared branches?

1. Preserves history - Original commit stays in log, provides audit trail
2. No rewriting - Doesn't change existing commits
3. No force push needed - Works with normal git push
4. No divergence - Teammates don't get conflicts from history changes
5. Reversible - Can revert the revert if needed

### When would you use revert vs reset?

**Use git revert:**
- Commits already pushed to shared branch
- Need to undo changes while preserving history
- Working in team environment
- Production hotfix rollback
- Public repositories

**Use git reset:**
- Commits only on local branch (not pushed)
- Want to clean up messy local history
- Made mistake in recent commits
- Reorganizing local work before pushing
- Solo projects or experimental branches

---

## Task 3: Reset vs Revert Summary

| Aspect | git reset | git revert |
|--------|-----------|------------|
| What it does | Moves branch pointer backward, removing commits from history | Creates new commit that undoes changes from target commit |
| Removes commit from history? | Yes - commit disappears from log | No - original commit remains in history |
| Safe for shared/pushed branches? | No - rewrites history, causes conflicts | Yes - adds new commit, preserves history |
| When to use | Local unpushed commits, cleaning local history, experimental work | Pushed commits, shared branches, production rollbacks, team collaboration |

### Quick Decision Guide
```
Is the commit pushed to a shared branch?
│
├─ YES → Use git revert
│
└─ NO → Can use git reset
          │
          ├─ Keep changes staged? → git reset --soft
          ├─ Keep changes unstaged? → git reset --mixed
          └─ Discard all changes? → git reset --hard
```

---

## Task 4: Branching Strategies

### 1. GitFlow

**How it works:**
Multiple long-lived branches with strict rules:
- main - production-ready code
- develop - integration branch for features
- feature/* - new features (branch from develop)
- release/* - release preparation (branch from develop)
- hotfix/* - urgent production fixes (branch from main)

**When used:**
- Large teams with scheduled releases
- Multiple versions in production
- Clear release cycles
- Need for hotfix process

**Pros:**
- Clear structure and rules
- Supports parallel development
- Isolated feature development
- Handles releases and hotfixes well

**Cons:**
- Complex for small teams
- Slower deployment cycle
- Merge overhead
- Can be overkill for simple projects

---

### 2. GitHub Flow

**How it works:**
Simple workflow with one main branch:
- main - always deployable
- feature/* - short-lived feature branches
- Pull Request for review
- Deploy from main

**Workflow:**
1. Create feature branch from main
2. Commit changes
3. Open Pull Request
4. Review and discuss
5. Deploy to test/staging
6. Merge to main
7. Deploy main to production

**When used:**
- Continuous deployment environments
- Startups moving fast
- Web applications
- Small to medium teams

**Pros:**
- Simple and easy to understand
- Fast deployment cycle
- Encourages code review
- Minimal overhead

**Cons:**
- No formal release process
- Harder with multiple versions
- Requires good CI/CD
- Less control over releases

---

### 3. Trunk-Based Development

**How it works:**
Everyone commits to main (trunk) with very short-lived branches:
- main - trunk, always stable
- Feature branches live hours/days max
- Feature flags for incomplete features
- Continuous integration required

**Key practices:**
- Small, frequent commits
- Feature flags/toggles for incomplete work
- Strong automated testing
- Continuous integration
- Fast feedback loops

**When used:**
- High-performing engineering teams
- Continuous deployment culture
- Companies like Google, Facebook
- Microservices architectures

**Pros:**
- Fast integration
- Reduces merge conflicts
- Encourages small changes
- Aligns with CI/CD
- Simplified workflow

**Cons:**
- Requires discipline
- Needs strong testing
- Feature flags add complexity
- Not suitable for scheduled releases
- Requires mature DevOps practices

---

### Strategy Recommendations

**Which strategy for a startup shipping fast?**

GitHub Flow - Best choice because:
- Simple to implement and learn
- Fast deployment cycle
- Minimal process overhead
- Focuses on shipping features quickly
- Easy to adapt as team grows

**Which strategy for a large team with scheduled releases?**

GitFlow - Best choice because:
- Handles multiple releases in parallel
- Clear release process
- Supports hotfixes cleanly
- Structured workflow for coordination
- Works well with traditional release cycles

**Which one do popular open-source projects use?**

Examples:
- Linux Kernel: Custom variant of trunk-based
- Kubernetes: GitHub Flow with release branches
- React: GitHub Flow (main branch + feature branches via PRs)
- Node.js: GitFlow-like (main, release branches, backports)
- VS Code: GitHub Flow with continuous deployment

Most popular: GitHub Flow - simple, flexible, works well with PR-based contributions.