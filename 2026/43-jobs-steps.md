# Day 43 – Jobs, Steps, Env Vars & Conditionals

---

## Task 1: Multi-Job Workflow

### .github/workflows/multi-job.yml
```yaml
name: Multi-Job Pipeline

on:
  push:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build app
        run: echo "Building the app"
  
  test:
    runs-on: ubuntu-latest
    needs: build  # Waits for build to complete
    steps:
      - name: Run tests
        run: echo "Running tests"
  
  deploy:
    runs-on: ubuntu-latest
    needs: test  # Waits for test to complete
    steps:
      - name: Deploy
        run: echo "Deploying"
```

**Dependency chain:** build → test → deploy (sequential)


---

## Task 2: Environment Variables

### .github/workflows/env-vars.yml
```yaml
name: Environment Variables

on:
  workflow_dispatch:

env:
  APP_NAME: myapp  # Workflow level

jobs:
  test-env:
    runs-on: ubuntu-latest
    env:
      ENVIRONMENT: staging  # Job level
    
    steps:
      - name: Print all env vars
        env:
          VERSION: 1.0.0  # Step level
        run: |
          echo "App: $APP_NAME"
          echo "Environment: $ENVIRONMENT"
          echo "Version: $VERSION"
      
```

**Scope:**
- Workflow env: accessible in all jobs/steps
- Job env: accessible in all steps of that job
- Step env: accessible only in that step

---

## Task 3: Job Outputs

### .github/workflows/job-outputs.yml
```yaml
name: Job Outputs

on:
  workflow_dispatch:

jobs:
  generate-data:
    runs-on: ubuntu-latest
    outputs:
      build-time: ${{ steps.set-time.outputs.time }}  # Expose output
    
    steps:
      - name: Set build time
        id: set-time  # Step ID required
        run: echo "time=$(date +'%Y-%m-%d %H:%M:%S')" >> $GITHUB_OUTPUT
  
  use-data:
    runs-on: ubuntu-latest
    needs: generate-data  # Wait for first job
    
    steps:
      - name: Print build time
        run: |
          echo "Build completed at: ${{ needs.generate-data.outputs.build-time }}"
```

**Why pass outputs?**
- Share data between jobs (version numbers, artifact paths, test results)
- Jobs run on different runners (can't share filesystem)
- Coordinate deployment decisions

---

## Task 4: Conditionals

### .github/workflows/conditionals.yml
```yaml
name: Conditionals

on:
  push:
  pull_request:

jobs:
  conditional-steps:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Only on main branch
        if: github.ref == 'refs/heads/main'
        run: echo "This runs only on main"
      
      - name: Simulate failure
        id: test-step
        run: exit 1
        continue-on-error: true  # Don't fail job, continue
      
      - name: Run if previous failed
        if: steps.test-step.outcome == 'failure'
        run: echo "Previous step failed but we continued"
    
  
  deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'  # Only on push, not PR
    
    steps:
      - name: Deploy
        run: echo "Deploying (push only)"
```

**`continue-on-error: true`:**
- Step fails but job continues
- Job marked as success even with failed step
- Useful for optional steps (linting, optional tests)

---

## Task 5: Smart Pipeline

### .github/workflows/smart-pipeline.yml
```yaml
name: Smart Pipeline

on:
  push:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run linter
        run: echo "Linting code"
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: echo "Running tests"
  
  summary:
    runs-on: ubuntu-latest
    needs: [lint, test]  # Wait for both
    
    steps:
      - name: Determine branch type
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "Branch: main (production)"
          else
            echo "Branch: ${{ github.ref_name }} (feature)"
          fi
      
      - name: Print commit info
        run: |
          echo "Commit: ${{ github.sha }}"
          echo "Message: ${{ github.event.head_commit.message }}"
          echo "Author: ${{ github.event.head_commit.author.name }}"
```

**Parallel execution:** lint and test run simultaneously

**Sequential:** summary waits for both to finish

---

## Key Concepts

### Job Dependencies
```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
      - run: echo "First"
  
  job2:
    runs-on: ubuntu-latest
    needs: job1  # Runs after job1 succeeds
    steps:
      - run: echo "Second"
  
  job3:
    runs-on: ubuntu-latest
    needs: [job1, job2]  # Runs after both succeed
    steps:
      - run: echo "Third"
```

**Without `needs`:** Jobs run in parallel

**With `needs`:** Jobs run sequentially or after dependencies complete

---

### Setting and Using Outputs
```yaml
jobs:
  job1:
    outputs:
      my-output: ${{ steps.step-id.outputs.value }}
    steps:
      - id: step-id
        run: echo "value=hello" >> $GITHUB_OUTPUT
  
  job2:
    needs: job1
    steps:
      - run: echo "${{ needs.job1.outputs.my-output }}"
```

**Flow:**
1. Step sets output: `echo "key=value" >> $GITHUB_OUTPUT`
2. Job exposes it: `outputs: { my-output: ${{ steps.id.outputs.key }} }`
3. Other job reads it: `${{ needs.job-name.outputs.my-output }}`

---

### Conditionals Reference
```yaml
# Branch conditions
if: github.ref == 'refs/heads/main'
if: github.ref_name == 'develop'

# Event type
if: github.event_name == 'push'
if: github.event_name == 'pull_request'

# Step outcome
if: steps.step-id.outcome == 'success'
if: steps.step-id.outcome == 'failure'

# Job status
if: needs.job-name.result == 'success'
if: needs.job-name.result == 'failure'

# Always run (even if previous failed)
if: always()

# Only if workflow successful so far
if: success()

# Only if previous step/job failed
if: failure()
```

---

### Environment Variable Levels
```yaml
# Workflow level
env:
  GLOBAL_VAR: value

jobs:
  my-job:
    # Job level
    env:
      JOB_VAR: value
    
    steps:
      # Step level
      - env:
          STEP_VAR: value
        run: echo "$GLOBAL_VAR $JOB_VAR $STEP_VAR"
```

**Priority:** Step > Job > Workflow (inner scope overrides outer)

---

### GitHub Context Variables
```yaml
${{ github.sha }}                    # Commit SHA
${{ github.ref }}                    # refs/heads/main
${{ github.ref_name }}               # main
${{ github.actor }}                  # Username who triggered
${{ github.repository }}             # owner/repo
${{ github.event_name }}             # push, pull_request, etc.
${{ github.event.head_commit.message }}  # Commit message
```

---


## Key Takeaways

**Jobs:**
- Run in parallel by default
- Use `needs:` for dependencies
- Each job runs on fresh runner

**Outputs:**
- Pass data between jobs
- Set: `echo "key=value" >> $GITHUB_OUTPUT`
- Expose: job `outputs:`
- Read: `${{ needs.job.outputs.key }}`

**Environment Variables:**
- Three levels: workflow, job, step
- Access with `$VAR_NAME`
- GitHub context: `${{ github.* }}`

**Conditionals:**
- `if:` on jobs or steps
- Branch: `github.ref == 'refs/heads/main'`
- Event: `github.event_name == 'push'`
- Status: `success()`, `failure()`, `always()`
- `continue-on-error: true` - step fails but job continues

---
