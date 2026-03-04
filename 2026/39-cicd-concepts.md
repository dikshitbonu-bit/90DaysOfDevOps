# Day 39 – What is CI/CD?

Understanding why CI/CD exists before writing pipelines. Getting the concepts right first.

---

## Task 1: The Problem

### Scenario: 5 Developers, Manual Deployment

**What can go wrong:**

1. **Integration Hell**
   - Developer A's code breaks Developer B's code
   - Nobody knows until manual merge
   - Conflicts discovered days/weeks later
   - "But it worked yesterday!"

2. **Environment Drift**
   - Works on Dev's laptop, not on production
   - Different versions of dependencies
   - Missing environment variables
   - OS differences (Mac dev, Linux prod)

3. **Human Error**
   - Forgot to run tests before deploying
   - Deployed wrong branch
   - Skipped a build step
   - Deployed to production instead of staging
   - No rollback plan

4. **Inconsistent Process**
   - Each developer deploys differently
   - No documentation of deployment steps
   - Knowledge locked in one person's head
   - Bus factor = 1

5. **Slow Feedback**
   - Bug found in production days after commit
   - Don't know which commit broke it
   - Wasted time debugging

6. **Fear of Deployment**
   - Deployments only on Fridays (then break weekend)
   - Or never on Fridays (too risky)
   - Weeks between deployments
   - Large, risky releases

---

### "It Works on My Machine"

**What it means:**
Code runs perfectly on developer's laptop but fails everywhere else.

**Why it's a real problem:**

| Developer's Machine | Production Server |
|---------------------|-------------------|
| macOS | Linux |
| Node 16.x | Node 14.x |
| SQLite | PostgreSQL |
| Local files | S3 buckets |
| 16GB RAM | 2GB RAM |
| Fast SSD | Slow disk |
| No load | 1000 users |

**The disconnect:**
- Different OS, dependencies, configuration
- Missing environment variables
- Different database
- No load testing
- Development shortcuts enabled

**Solution:** Run code in **identical environments** from dev to prod (enter Docker and CI/CD).

---

### Manual Deployment Frequency

**How many times a day can a team safely deploy manually?**

**Realistically: 1-2 times per day, maximum.**

**Why so few:**
- Each deployment takes 30-60 minutes
- Manual steps prone to errors
- Requires coordination (all devs stop work)
- Someone needs to stay and monitor
- Rollback is manual and slow
- Fear accumulates, releases get bigger
- Risk increases with release size

**With CI/CD:**
- Amazon: Deploys every 11.7 seconds
- Netflix: Thousands of deploys per day
- Facebook: Twice per day to 1+ billion users
- Small startups: 10-50 times per day

**The difference:** Automation removes fear.

---

## Task 2: CI vs CD vs CD

### Continuous Integration (CI)

**Definition:**
Developers merge code into a shared repository multiple times per day. Each merge triggers an automated build and test process to catch bugs early.

**What happens:**
1. Developer pushes code to Git
2. Automated tests run immediately
3. Code is built/compiled
4. Integration tests verify it works with existing code
5. Team notified if anything fails

**How often:**
Every commit. Multiple times per day, per developer.

**What it catches:**
- Syntax errors
- Failing tests
- Integration conflicts
- Code quality issues
- Security vulnerabilities

**Real-world example:**
Google's internal CI runs **150+ million tests per day**. Every code commit triggers tests. If tests fail, commit is blocked. No broken code reaches main branch.

---

### Continuous Delivery (CD)

**Definition:**
Code is always in a deployable state. After passing CI, code is automatically prepared for release, but **deployment to production requires manual approval**.

**What "delivery" means:**
Code is packaged, tested, and ready to deploy at any moment. One button click away from production.

**Difference from CI:**
- CI = merge and test
- CD = also build artifacts, deploy to staging, ready for production
- Human decides when to deploy to production

**Real-world example:**
Etsy's CD pipeline: Every commit that passes tests is deployed to staging automatically. Developers review staging, then click "Deploy to Production" when ready. They deploy 50+ times per day.

---

### Continuous Deployment (CD)

**Definition:**
Every commit that passes all automated tests is **automatically deployed to production** with no human intervention.

**Difference from Continuous Delivery:**
- **Delivery:** Human approves production deploy
- **Deployment:** Fully automated to production

**When teams use it:**
- High confidence in automated tests
- Mature monitoring and alerting
- Fast rollback capability
- Feature flags to disable broken features
- SaaS products (not embedded systems)

**Real-world example:**
Facebook uses Continuous Deployment. Code that passes all tests goes straight to production. If monitoring detects issues, automated rollback happens in seconds. Ship fast, fix fast.

---

### CI vs CD vs CD Summary

| Aspect | Continuous Integration | Continuous Delivery | Continuous Deployment |
|--------|------------------------|---------------------|-----------------------|
| **Automates** | Build + Test | Build + Test + Staging Deploy | Build + Test + Production Deploy |
| **Production Deploy** | Manual | Manual (but ready) | Automatic |
| **Human Approval** | No | Yes (for production) | No |
| **Frequency** | Every commit | Every commit ready | Every commit live |
| **Risk** | Low | Medium | High (requires maturity) |
| **Speed** | Fast feedback | Fast + always ready | Fastest to users |

---

## Task 3: Pipeline Anatomy

### Components of a CI/CD Pipeline

**Trigger**
- What starts the pipeline
- Examples:
  - Push to `main` branch
  - Pull request opened
  - Tag created (`v1.0.0`)
  - Scheduled (cron: daily at 2am)
  - Manual button click
  - Webhook from external service

**Stage**
- Logical phase in the pipeline
- Runs sequentially (one after another)
- Examples:
  - Build stage
  - Test stage
  - Deploy to staging stage
  - Deploy to production stage
- If one stage fails, next stages don't run

**Job**
- Unit of work inside a stage
- Multiple jobs in a stage can run in parallel
- Examples:
  - "Run unit tests" job
  - "Run integration tests" job
  - "Build Docker image" job
- Each job runs independently

**Step**
- Single command or action inside a job
- Runs sequentially within the job
- Examples:
  - `npm install` (install dependencies)
  - `npm test` (run tests)
  - `docker build -t myapp .` (build image)
  - `docker push myapp` (push to registry)

**Runner**
- The machine that executes the job
- Can be:
  - GitHub-hosted runner (GitHub provides)
  - Self-hosted runner (your own server)
  - Cloud VM (AWS, GCP, Azure)
  - Docker container
- Has specific OS and tools installed

**Artifact**
- Output produced by a job
- Stored and passed to later stages
- Examples:
  - Compiled binary
  - Docker image
  - Test results/reports
  - Build logs
  - Deployment package (ZIP, JAR, etc.)
- Lives beyond the job that created it

---

### Hierarchy
```
Pipeline
├── Trigger (what starts it)
├── Stage 1: Build
│   ├── Job 1: Compile code
│   │   ├── Step 1: Install dependencies
│   │   ├── Step 2: Run build
│   │   └── Step 3: Create artifact
│   └── Job 2: Run linter (parallel)
│       └── Step 1: Check code style
├── Stage 2: Test
│   ├── Job 1: Unit tests
│   └── Job 2: Integration tests (parallel)
└── Stage 3: Deploy
    └── Job 1: Deploy to production
        ├── Step 1: Pull artifact
        ├── Step 2: Deploy to server
        └── Step 3: Run health check

Runs on: Runner (Ubuntu VM, Docker container, etc.)
Produces: Artifacts (binaries, images, reports)
```

---

## Task 4: Draw a Pipeline

### Scenario

Developer pushes code to GitHub → App is tested, built into Docker image, deployed to staging.

### Pipeline Diagram
```
┌─────────────────────────────────────────────────────────────────────┐
│                           CI/CD Pipeline                             │
└─────────────────────────────────────────────────────────────────────┘

Trigger: Push to main branch
   │
   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 1: Build                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Job: Install & Build                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Step 1: Checkout code                                        │   │
│  │ Step 2: Install dependencies (npm install)                   │   │
│  │ Step 3: Run linter (npm run lint)                            │   │
│  │ Step 4: Build application (npm run build)                    │   │
│  │ Artifact: Build output saved                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 2: Test                                                       │
├─────────────────────────────────────────────────────────────────────┤
│  Job 1: Unit Tests (parallel)    │  Job 2: Integration Tests       │
│  ┌─────────────────────────────┐ │ ┌──────────────────────────────┐│
│  │ Step 1: Run unit tests      │ │ │ Step 1: Start test DB        ││
│  │ Step 2: Generate coverage   │ │ │ Step 2: Run integration tests││
│  │ Artifact: Test report       │ │ │ Artifact: Integration report ││
│  └─────────────────────────────┘ │ └──────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 3: Build & Push Docker Image                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Job: Docker Build                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Step 1: Login to Docker Hub                                  │   │
│  │ Step 2: Build Docker image (docker build -t app:v1 .)       │   │
│  │ Step 3: Tag image (docker tag app:v1 user/app:latest)       │   │
│  │ Step 4: Push to registry (docker push user/app:latest)      │   │
│  │ Artifact: Docker image in registry                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 4: Deploy to Staging                                          │
├─────────────────────────────────────────────────────────────────────┤
│  Job: Deploy                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Step 1: SSH to staging server                                │   │
│  │ Step 2: Pull latest image (docker pull user/app:latest)     │   │
│  │ Step 3: Stop old container                                   │   │
│  │ Step 4: Start new container (docker run -d -p 80:3000 ...)  │   │
│  │ Step 5: Run health check (curl http://staging/health)       │   │
│  │ Step 6: Send Slack notification ✅ Deployed!                │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
   │
   ▼
Result: App running on staging server
        URL: https://staging.myapp.com
```

### What Happens at Each Stage

**Stage 1 - Build:**
- Get code from GitHub
- Install dependencies
- Check code quality (linting)
- Build production assets
- Save build output as artifact

**Stage 2 - Test:**
- Run unit tests (fast, no dependencies)
- Run integration tests (with database)
- Both jobs run in parallel (faster)
- Generate test coverage reports

**Stage 3 - Build & Push Image:**
- Package app into Docker image
- Push to Docker registry
- Now deployable anywhere

**Stage 4 - Deploy to Staging:**
- Connect to staging server
- Pull latest Docker image
- Replace old container with new one
- Verify it's working
- Notify team

---

## Task 5: Explore in the Wild

### Analyzed Repository: React

**Repository:** https://github.com/facebook/react

**Workflow File:** `.github/workflows/runtime_commit.yml`

### Analysis

**What triggers it?**
```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```
- Triggers on every push to `main` branch
- Triggers on every pull request to `main`

**How many jobs does it have?**

5 jobs:
1. `test_devtools` - Test React DevTools
2. `test_dom_fixtures` - Test DOM fixtures
3. `test_fizz` - Test server rendering
4. `test_flight` - Test React Flight
5. `test_main` - Main test suite

**What does it do?**

Best guess from reading the YAML:

1. **Checkout code** from GitHub
2. **Setup Node.js** environment
3. **Install dependencies** with Yarn
4. **Run multiple test suites** in parallel:
   - DevTools tests
   - DOM fixture tests
   - Server-side rendering tests
   - React Flight tests (experimental)
   - Main React test suite
5. **Upload test results** as artifacts
6. **Report status** back to PR

**Key observations:**
- Tests run on every PR (catches bugs before merge)
- Multiple jobs run in parallel (faster feedback)
- Tests different parts of React independently
- Matrix strategy tests multiple Node versions
- Artifacts saved for debugging if tests fail

**Why this matters:**
- React has millions of users
- Breaking changes affect entire ecosystem
- CI catches issues before they reach users
- Every PR must pass all tests before merge
- Automated testing enables fast development

---

## Key Takeaways

**CI/CD solves real problems:**
- Integration hell → Test every commit
- "Works on my machine" → Identical environments
- Human error → Automated, consistent process
- Slow feedback → Know immediately if code breaks
- Fear of deployment → Deploy confidently, frequently

**CI/CD is not optional:**
- Every modern development team uses it
- Industry standard, expected in interviews
- Enables fast, safe software delivery
- The bigger the team, the more critical it is

**Pipelines are code:**
- Defined in YAML files
- Version controlled
- Reviewed like application code
- Reusable and shareable

---

