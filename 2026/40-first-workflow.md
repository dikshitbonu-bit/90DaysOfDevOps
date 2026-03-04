# Day 40 –  First GitHub Actions Workflow

First real pipeline running in the cloud.

---


## Task 2: Hello Workflow

### .github/workflows/hello.yml
```yaml
name: Hello Workflow

on:
  push:
    branches: [main]

jobs:
  greet:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Print greeting
        run: echo "Hello from GitHub Actions!"
```

### Push and Watch

**Result:** Green checkmark = Success

---

## Task 3: Understand the Anatomy
```yaml
name: Hello Workflow
# Human-readable name shown in Actions tab

on:
  push:
# Trigger: when workflow runs (on every push)

jobs:
  greet:
# Job name (custom)

    runs-on: ubuntu-latest
    # Which OS/environment to run on
    
    steps:
    # Sequential actions within job
    
      - name: Checkout code
        # Description of step
        
        uses: actions/checkout@v4
        # Pre-built action from marketplace
      
      - name: Print greeting
        run: echo "Hello from GitHub Actions!"
        # Execute shell commands
```

### Key Definitions

**`on:`** - Event trigger (push, pull_request, schedule, manual)

**`jobs:`** - Collection of jobs (run in parallel by default)

**`runs-on:`** - Operating system (ubuntu-latest, windows-latest, macos-latest)

**`steps:`** - Sequential list of actions/commands

**`uses:`** - Run pre-built action (`owner/repo@version`)

**`run:`** - Execute shell commands

**`name:`** - Human-readable label for UI

---

## Task 4: Add More Steps

### Updated .github/workflows/hello.yml
```yaml
name: Hello Workflow

on:
  push:

jobs:
  greet:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Print greeting
        run: echo "Hello from GitHub Actions!"
      
      - name: Print current date and time
        run: date
      
      - name: Print branch name
        run: echo "Running on branch: ${{ github.ref_name }}"
      
      - name: List files in repository
        run: ls -la
      
      - name: Print runner OS
        run: echo "Running on: ${{ runner.os }}"
      
      - name: Print system info
        run: |
          echo "Runner name: ${{ runner.name }}"
          echo "Architecture: ${{ runner.arch }}"
          uname -a
```

### GitHub Context Variables
```yaml
${{ github.ref_name }}      # Branch name
${{ github.repository }}    # Repo name
${{ github.actor }}         # User who triggered
${{ github.sha }}           # Commit SHA
${{ runner.os }}            # OS (Linux, Windows, macOS)
${{ runner.name }}          # Runner name
${{ runner.arch }}          # Architecture
```

---

## Task 5: Break It On Purpose

### Add Failing Step
```yaml
name: Hello Workflow

on:
  push:

jobs:
  greet:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Print greeting
        run: echo "Hello from GitHub Actions!"
      
      - name: This will fail
        run: exit 1
      
      - name: This won't run
        run: echo "Skipped due to previous failure"
```

### What Happens

**Actions Tab:**
- Status: Failed (red X)
- Steps show:
  - Checkout code (green)
  - Print greeting (green)
  - This will fail (red)
  - This won't run (skipped/grey)

**Error message:**
```
Run exit 1
Error: Process completed with exit code 1.
```

### Fix It
```yaml
- name: This is now fixed
  run: echo "No more exit 1"

- name: This will run now
  run: echo "All steps complete"
```

---

## Reading Pipeline Errors

### Error Checklist

1. **Which step failed?** - Look for red X
2. **What command ran?** - Check for typos
3. **What's the error?** - Read output carefully
4. **Exit code?** - 0 = success, 1 = error, 127 = not found
5. **Check previous steps** - Dependencies installed?

### Common Errors

**Command not found:**
```
bash: python: command not found
```
Fix: Install tool or use correct command (`python3`)

**File not found:**
```
cat: README.md: No such file or directory
```
Fix: Run `actions/checkout` first

**Permission denied:**
```
bash: ./script.sh: Permission denied
```
Fix: `chmod +x script.sh`

---


## Workflow Structure
```yaml
name: Workflow Name

on:
  push:
  pull_request:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

jobs:
  job_name:
    runs-on: ubuntu-latest
    
    steps:
      - name: Step name
        uses: action@version
      
      - name: Another step
        run: command
        
      - name: Multi-line commands
        run: |
          command1
          command2
      
      - name: With environment
        env:
          VAR: value
        run: echo $VAR
```

---

## Popular Actions
```yaml
- uses: actions/checkout@v4              # Clone repo
- uses: actions/setup-node@v4            # Install Node.js
- uses: actions/setup-python@v5          # Install Python
- uses: actions/cache@v3                 # Cache dependencies
- uses: docker/build-push-action@v5      # Build Docker image
- uses: actions/upload-artifact@v3       # Save artifacts
- uses: actions/download-artifact@v3     # Retrieve artifacts
```


---

## Key Takeaways

**Workflow Basics:**
- Lives in `.github/workflows/*.yml`
- Triggers on events (push, PR, schedule)
- Runs on GitHub-hosted runners
- Jobs contain steps

**Execution:**
- Each push triggers new run
- Jobs run in parallel
- Steps run sequentially
- Failure stops the job

**Debugging:**
- Actions tab shows all runs
- Click step to see output
- Read error messages
- Check dependencies

**Best Practices:**
- Name workflows clearly
- Use `actions/checkout` first
- Pin action versions (`@v4`)
- Test on feature branches
- Keep workflows simple

---
