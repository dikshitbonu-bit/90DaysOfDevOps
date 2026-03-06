# Day 41 – Triggers & Matrix Builds

Learning every way to trigger workflows and running jobs across multiple environments.

---

## Task 1: Trigger on Pull Request

### .github/workflows/pr-check.yml
```yaml
name: PR Check

on:
  pull_request:
    branches:
      - main

jobs:
  check:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Print PR info
        run: |
          echo "PR check running for branch: ${{ github.head_ref }}"
          echo "Target branch: ${{ github.base_ref }}"
          echo "PR number: ${{ github.event.pull_request.number }}"
          echo "PR title: ${{ github.event.pull_request.title }}"
```

### PR Triggers Explained
```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]  # Default
    branches: [main]                         # Only PRs to main
    
  # Or trigger on specific PR actions:
  pull_request:
    types:
      - opened        # PR created
      - synchronize   # New commits pushed
      - reopened      # Closed PR reopened
      - edited        # Title/description changed
      - closed        # PR closed/merged
```

**PR Context Variables:**
- `${{ github.head_ref }}` - Source branch
- `${{ github.base_ref }}` - Target branch
- `${{ github.event.pull_request.number }}` - PR number
- `${{ github.event.pull_request.title }}` - PR title

---

## Task 2: Scheduled Trigger

### .github/workflows/scheduled.yml
```yaml
name: Scheduled Workflow

on:
  schedule:
    - cron: '0 0 * * *'  # Every day at midnight UTC
  
  workflow_dispatch:  # Also allow manual trigger

jobs:
  nightly-build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run nightly tasks
        run: |
          echo "Nightly build running at: $(date)"
          echo "This runs automatically every day at midnight UTC"
      
      - name: Cleanup old artifacts
        run: echo "Simulating cleanup task"
```

### Cron Syntax
```
* * * * *
│ │ │ │ │
│ │ │ │ └─ Day of week (0-6, 0=Sunday)
│ │ │ └─── Month (1-12)
│ │ └───── Day of month (1-31)
│ └─────── Hour (0-23)
└───────── Minute (0-59)
```

### Common Cron Examples
```yaml
# Every day at midnight
- cron: '0 0 * * *'

# Every Monday at 9 AM
- cron: '0 9 * * 1'

# Every hour
- cron: '0 * * * *'

# Every 15 minutes
- cron: '*/15 * * * *'

# Every weekday at 6 PM
- cron: '0 18 * * 1-5'

# First day of every month at 2 AM
- cron: '0 2 1 * *'
```

**Answer: Every Monday at 9 AM:**
```yaml
- cron: '0 9 * * 1'
```

**Important Notes:**
- All times are UTC
- Scheduled workflows run on the default branch only
- GitHub may delay scheduled runs during high load
- Minimum interval is 5 minutes

---

## Task 3: Manual Trigger

### .github/workflows/manual.yml
```yaml
name: Manual Deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - staging
          - production
      
      version:
        description: 'Version to deploy'
        required: false
        default: 'latest'
      
      run_tests:
        description: 'Run tests before deploy'
        required: false
        type: boolean
        default: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Print inputs
        run: |
          echo "Deploying to: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Run tests: ${{ inputs.run_tests }}"
      
      - name: Run tests
        if: ${{ inputs.run_tests == true }}
        run: echo "Running tests..."
      
      - name: Deploy
        run: |
          echo "Deploying version ${{ inputs.version }} to ${{ inputs.environment }}"
          echo "Deployment would happen here"
```


### Input Types
```yaml
inputs:
  text_input:
    description: 'Text input'
    required: true
    type: string
  
  choice_input:
    description: 'Choice input'
    type: choice
    options:
      - option1
      - option2
  
  boolean_input:
    description: 'Boolean input'
    type: boolean
    default: false
  
  environment_input:
    description: 'Environment input'
    type: environment
    default: production
```

---

## Task 4: Matrix Builds

### .github/workflows/matrix.yml
```yaml
name: Matrix Build

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Print Python version
        run: |
          python --version
          echo "Testing with Python ${{ matrix.python-version }}"
      
```

**Result:** 3 jobs run in parallel (one for each Python version)

---

### Extended Matrix (OS + Python)
```yaml
name: Matrix Build Extended

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ['3.10', '3.11', '3.12']
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Print environment
        run: |
          python --version
          echo "OS: ${{ matrix.os }}"
          echo "Python: ${{ matrix.python-version }}"
      

```

**Total jobs:** 2 OS × 3 Python versions = **6 jobs** (all run in parallel)

---

### Matrix with Include/Exclude
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node-version: [16, 18, 20]
    
    # Add specific combinations
    include:
      - os: ubuntu-latest
        node-version: 14
        experimental: true
    
    # Exclude specific combinations
    exclude:
      - os: macos-latest
        node-version: 16
      - os: windows-latest
        node-version: 16
```

**Result:**
- Base: 3 OS × 3 Node = 9 jobs
- Exclude: -2 jobs (macOS/16, Windows/16)
- Include: +1 job (Ubuntu/14)
- **Total: 8 jobs**

---

## Task 5: Exclude & Fail-Fast

### .github/workflows/matrix-advanced.yml
```yaml
name: Matrix Advanced

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      fail-fast: false  # Don't stop other jobs if one fails
      
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.10', '3.11', '3.12']
        
        exclude:
          # Python 3.10 not supported on Windows in this project
          - os: windows-latest
            python-version: '3.10'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Print info
        run: |
          echo "OS: ${{ matrix.os }}"
          echo "Python: ${{ matrix.python-version }}"
      
      - name: Simulate failure on specific combo
        run: |
          if [ "${{ matrix.os }}" == "ubuntu-latest" ] && [ "${{ matrix.python-version }}" == "3.11" ]; then
            echo "Simulating failure"
            exit 1
          fi
        shell: bash
      
      - name: This runs if previous step succeeds
        run: echo "Success on ${{ matrix.os }} with Python ${{ matrix.python-version }}"
```

### Fail-Fast Behavior

**`fail-fast: true` (default):**
- One job fails → all other running jobs are cancelled immediately
- Saves CI/CD minutes
- Use when: Jobs are expensive, want fast feedback

**`fail-fast: false`:**
- One job fails → other jobs continue running
- See all failures at once
- Use when: Need to see all test results, different failures on different platforms

**Example scenario:**

With `fail-fast: true`:
```
Job 1 (Ubuntu/3.10): Running...
Job 2 (Ubuntu/3.11): FAILED → All jobs cancelled
Job 3 (Ubuntu/3.12): Cancelled
Job 4 (Windows/3.11): Cancelled
...
```

With `fail-fast: false`:
```
Job 1 (Ubuntu/3.10): Success
Job 2 (Ubuntu/3.11): FAILED
Job 3 (Ubuntu/3.12): Success
Job 4 (Windows/3.11): Success
...
All jobs complete, see all results
```

---

## All Triggers Reference

### Event Triggers
```yaml
on:
  # Code events
  push:
    branches: [main, develop]
    paths: ['src/**']
  
  pull_request:
    types: [opened, synchronize]
    branches: [main]
  
  pull_request_target:  # Safer for PRs from forks
  
  # Release events
  release:
    types: [published]
  
  create:  # Tag or branch created
  delete:  # Tag or branch deleted
  
  # Issue events
  issues:
    types: [opened, labeled]
  
  issue_comment:
  
  # Scheduled
  schedule:
    - cron: '0 0 * * *'
  
  # Manual
  workflow_dispatch:
    inputs:
      name:
        description: 'Input name'
        required: true
  
  # Webhook
  repository_dispatch:
    types: [custom-event]
  
  # Workflow call
  workflow_call:
    inputs:
      config:
        required: true
        type: string
```

### Combining Triggers
```yaml
on:
  push:
    branches: [main]
  
  pull_request:
    branches: [main]
  
  schedule:
    - cron: '0 0 * * *'
  
  workflow_dispatch:
```

---

## Matrix Strategy Examples

### Node.js Multi-Version
```yaml
strategy:
  matrix:
    node-version: [16, 18, 20]

steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}
```

### Multi-OS
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]

runs-on: ${{ matrix.os }}
```

### Database Testing
```yaml
strategy:
  matrix:
    db: [postgres, mysql, mongodb]
    version: ['14', '15', '16']

services:
  database:
    image: ${{ matrix.db }}:${{ matrix.version }}
```

### Custom Variables
```yaml
strategy:
  matrix:
    environment: [dev, staging, prod]
    region: [us-east-1, eu-west-1]

steps:
  - run: |
      echo "Deploying to ${{ matrix.environment }}"
      echo "Region: ${{ matrix.region }}"
```

---

## Best Practices

**Triggers:**
- Use `pull_request` for validation
- Use `push` on protected branches for deployment
- Use `schedule` for nightly builds and cleanup
- Use `workflow_dispatch` for manual deploys
- Combine triggers for flexibility

**Matrix Builds:**
- Test on target platforms (don't test everywhere)
- Use `exclude` to skip unsupported combinations
- Use `include` to add special cases
- Set `fail-fast: false` to see all failures
- Keep matrix small (costs add up)

**Performance:**
- More matrix combinations = more CI minutes
- Parallel jobs finish faster but cost more
- Use caching to speed up jobs
- Only run on changed files when possible


---

## Key Takeaways

**Triggers:**
- `push` - Code pushed to branch
- `pull_request` - PR opened/updated
- `schedule` - Cron-based timing
- `workflow_dispatch` - Manual with inputs
- Can combine multiple triggers

**Matrix Builds:**
- Test across multiple versions/platforms
- Jobs run in parallel
- Use `exclude` to skip combinations
- Use `include` to add special cases
- `fail-fast` controls failure behavior

**Cron Syntax:**
- `0 9 * * 1` = Every Monday 9 AM
- All times UTC
- Minimum 5-minute interval

**Best Practices:**
- Validate on PR, deploy on push
- Test on target platforms only
- Use `fail-fast: false` to see all results
- Cache dependencies for speed
- Keep matrices focused

---
