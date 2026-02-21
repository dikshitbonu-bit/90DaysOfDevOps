# Day 28 – Revision Day: Everything from Day 1 to Day 27

## Task 1: Self-Assessment Checklist

### Linux

- [✓] Navigate file system, create/move/delete files and directories
- [✓] Manage processes — list, kill, background/foreground
- [✓] Work with systemd — start, stop, enable, check status
- [✓] Read and edit text files using vi/vim or nano
- [✓] Troubleshoot CPU, memory, disk using top, free, df, du
- [✓] Explain Linux file system hierarchy (/, /etc, /var, /home, /tmp)
- [✓] Create users and groups, manage passwords
- [✓] Set file permissions using chmod (numeric and symbolic)
- [✓] Change file ownership with chown and chgrp
- [✓] Create and manage LVM volumes
- [ ] Check network connectivity — ping, curl, netstat, ss, dig, nslookup
- [✓] Explain DNS resolution, IP addressing, subnets, common ports

### Shell Scripting

- [✓] Write script with variables, arguments, user input
- [✓] Use if/elif/else and case statements
- [✓] Write for, while, until loops
- [✓] Define and call functions with arguments and return values
- [✓] Use grep, awk, sed, sort, uniq for text processing
- [ ] Handle errors with set -e, set -u, set -o pipefail, trap
- [✓] Schedule scripts with crontab

### Git & GitHub

- [✓] Initialize repo, stage, commit, view history
- [✓] Create and switch branches
- [✓] Push to and pull from GitHub
- [✓] Explain clone vs fork
- [✓] Merge branches — understand fast-forward vs merge commit
- [✓] Rebase a branch and explain when to use it vs merge
- [✓] Use git stash and git stash pop
- [✓] Cherry-pick a commit from another branch
- [✓] Explain squash merge vs regular merge
- [✓] Use git reset (soft, mixed, hard) and git revert
- [ ] Explain GitFlow, GitHub Flow, Trunk-Based Development
- [✓] Use GitHub CLI to create repos, PRs, issues

**Confidence Level:**
- Can do confidently: 27/30
- Need to revisit: 3/30

---

## Task 2: Weak Spots Revisited

### Topics I Needed to Revisit

**1. Error Handling in Shell Scripts (set -e, set -u, set -o pipefail, trap)**

Forgot these completely. Revisited today.

**What I re-learned:**
- `set -e` stops script immediately when any command fails
- `set -u` crashes script if you use undefined variables
- `set -o pipefail` makes pipes fail properly (normally hidden)
- `trap` runs cleanup code when script exits or crashes
- Always use `set -euo pipefail` at the top of production scripts
- Use trap for cleanup: `trap cleanup EXIT`

**2. Network Connectivity Commands (ping, curl, netstat, ss, dig, nslookup)**

Forgot the differences between these tools. Revisited today.

**What I re-learned:**
- `ping` - checks if host is reachable (ICMP)
- `curl` - makes HTTP requests, tests web services
- `netstat` - old tool for network connections (deprecated)
- `ss` - modern replacement for netstat (faster)
- `dig` - DNS lookup tool, shows full DNS query details
- `nslookup` - simpler DNS lookup (less detailed than dig)
- Use `ss -tulnp` to find which process uses which port

**3. Branching Strategies (GitFlow, GitHub Flow, Trunk-Based Development)**

Forgot which strategy to use when. Revisited today.

**What I re-learned:**
- **GitFlow:** Multiple long-lived branches (main, develop, feature, release, hotfix). Use for large teams with scheduled releases.
- **GitHub Flow:** One main branch + feature branches via PRs. Simple, fast. Use for continuous deployment and small teams.
- **Trunk-Based Development:** Everyone commits to main with very short-lived branches. Use for high-performing teams with strong CI/CD and feature flags.
- Recommendation: Startups → GitHub Flow, Enterprise → GitFlow, Google/Facebook → Trunk-Based

---

## Task 3: Quick-Fire Questions

**1. What does chmod 755 script.sh do?**

Owner gets read(4) + write(2) + execute(1) = 7
Group gets read(4) + execute(1) = 5
Others get read(4) + execute(1) = 5

Makes script executable by everyone, writable only by owner.

**2. What is the difference between a process and a service?**

Process = any running program with a PID (can be temporary)
Service = process managed by systemd, designed to run continuously in background (like nginx, ssh)

**3. How do you find which process is using port 8080?**
```bash
ss -tulnp | grep 8080
# or
netstat -tulnp | grep 8080
# or
lsof -i :8080
```

**4. What does set -euo pipefail do in a shell script?**

- `set -e` = exit on any error
- `set -u` = exit on undefined variable
- `set -o pipefail` = detect failures in pipes

Together = strict mode that catches errors immediately. Production scripts should always use this.

**5. What is the difference between git reset --hard and git revert?**

`git reset --hard` = rewrites history, deletes commits (dangerous on shared branches)
`git revert` = creates new commit that undoes changes (safe for shared branches)

**6. What branching strategy would you recommend for a team of 5 developers shipping weekly?**

GitHub Flow - simple, fast, perfect for weekly releases. Main stays deployable, feature branches via PRs, no complex release management needed.

**7. What does git stash do and when would you use it?**

Temporarily saves uncommitted work so you can switch branches. Use when you need to context-switch mid-work without committing half-done code.

**8. How do you schedule a script to run every day at 3 AM?**
```bash
crontab -e
# Add line:
0 3 * * * /path/to/script.sh
```

**9. What is the difference between git fetch and git pull?**

`git fetch` = downloads changes but doesn't merge
`git pull` = fetch + merge in one command

**10. What is LVM and why would you use it instead of regular partitions?**

LVM = Logical Volume Manager. Lets you resize volumes dynamically without repartitioning. Can add/remove disks, create snapshots, much more flexible than fixed partitions.

---

## Task 4: Work Organization Check

- [✓] All daily submissions (day-1 to day-27) committed and pushed
- [✓] git-commands.md up to date
- [✓] Shell scripting cheat sheet complete
- [✓] GitHub profile clean and organized
- [✓] All repos have descriptions and READMEs
- [✓] Profile README created

**Status:** Repository organized and ready for next phase

---

## Task 5: Teach It Back

### Topic: What is Crontab and Why Sysadmins Use It

Crontab is Linux's built-in task scheduler. It's a file where you write rules that tell the system "run this script at this time."

Sysadmins use it because servers need stuff done automatically - backups at 2 AM, log cleanup every Sunday, certificate renewals monthly, health checks every 5 minutes.

Without cron, you'd need someone awake 24/7 running commands manually. With cron, you write it once and it runs forever.

Each line in crontab has 5 time fields (minute, hour, day, month, weekday) followed by the command. `0 3 * * *` means "3 AM every day." `*/5 * * * *` means "every 5 minutes."

It's why your servers can take care of themselves while you sleep. Set it, forget it, check logs if something breaks.

Every sysadmin lives in crontab because automation is the only way to manage hundreds of servers without losing your mind.

---

