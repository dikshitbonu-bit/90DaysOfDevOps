# Day 48 – GitHub Actions Capstone: Netflix Clone DevSecOps Pipeline

[![Main Pipeline](https://github.com/dikshitbonu-bit/netflix-devsecops/actions/workflows/main.yml/badge.svg)](https://github.com/dikshitbonu-bit/netflix-devsecops/actions/workflows/main.yml)
[![PR Pipeline](https://github.com/dikshitbonu-bit/netflix-devsecops/actions/workflows/pr-pipeline.yml/badge.svg)](https://github.com/dikshitbonu-bit/netflix-devsecops/actions/workflows/pr-pipeline.yml)
[![Health Check](https://github.com/dikshitbonu-bit/netflix-devsecops/actions/workflows/health-check.yml/badge.svg)](https://github.com/dikshitbonu-bit/netflix-devsecops/actions/workflows/health-check.yml)

---

## Pipeline Architecture

```
PR opened → main
│
├── build-test-frontend  (reusable-build-test.yml)
├── build-test-backend   (reusable-build-test.yml)
├── security-scans       (reusable-security-scan.yml)
│     ├── Gitleaks
│     ├── SonarQube
│     ├── ESLint
│     ├── npm audit
│     └── Hadolint
└── [No Docker build/push on PRs]

────────────────────────────────────────

Push to main
│
├── build-test-frontend + build-test-backend  (parallel)
├── security-scans
├── docker-build-push-frontend  (reusable-docker-build-push.yml)
│     ├── Build multi-stage image
│     ├── Trivy scan → CRITICAL/HIGH = fail
│     └── Push: latest + sha-<commit>
├── docker-build-push-backend   (same, parallel)
└── deploy  (environment: production — requires manual approval)
      ├── SSH to EC2
      ├── docker compose up
      ├── Health check (5 retries)
      └── Smoke test

────────────────────────────────────────

Every 12 hours (+ workflow_dispatch)
│
└── health-check.yml
      ├── Pull netflix-backend:latest
      ├── Run container → curl /api/health
      └── $GITHUB_STEP_SUMMARY report
```

---

## Task 1 – Project Repo Setup

**App:** Netflix clone — React frontend + Node.js/Express backend + MongoDB  
**Repo:** [netflix-devsecops](https://github.com/dikshitbonu-bit/netflix-devsecops)  
**Stack:** Node.js 22, Docker, Docker Compose, AWS EC2

```
netflix-devsecops/
├── .github/workflows/
├── netflix-clone/
│   ├── frontend/   (React 18 + Nginx)
│   ├── backend/    (Express + JWT + Mongoose)
│   └── docker-compose.yml
└── README.md
```

---

## Task 2 – Reusable Workflow: Build & Test

**File:** `.github/workflows/reusable-build-test.yml`

```yaml
name: Build and Test
on:
  workflow_call:
    inputs:
      working_directory:
        required: true
        type: string
      run_build:
        required: false
        type: boolean
        default: true

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working_directory }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
          cache-dependency-path: ${{ inputs.working_directory }}/package-lock.json
      
      - name: Install dependencies
        run: npm ci 
      
      - name: Build
        if: ${{ inputs.run_build }}
        run: npm run build
      
      - name: Run tests
        run: npm test
        env:
          NODE_ENV: test
          TMDB_API_KEY: ${{ secrets.TMDB_API_KEY }}
```

---

## Task 3 – Reusable Workflow: Docker Build & Push

**File:** `.github/workflows/reusable-docker-build-push.yml`

```yaml
name: Docker Build and Push

on:
  workflow_call:
    inputs:
      service_name:
        required: true
        type: string
        description: 'Service to build (frontend or backend)'
      dockerfile_path:
        required: true
        type: string
        description: 'Path to Dockerfile'
      context_path:
        required: true
        type: string
        description: 'Build context path'
    secrets:
      DOCKER_USERNAME:
        required: true
      DOCKER_TOKEN:
        required: true

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKER_USERNAME }}/netflix-${{ inputs.service_name }}
          tags: |
            type=raw,value=latest
            type=sha,prefix=,format=long
      
      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: ${{ inputs.context_path }}
          file: ${{ inputs.dockerfile_path }}
          push: false
          load: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.35.0
        with:
          image-ref: ${{ secrets.DOCKER_USERNAME }}/netflix-${{ inputs.service_name }}:latest
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      - name: Push image to Docker Hub
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/netflix-${{ inputs.service_name }}:latest
          docker push ${{ secrets.DOCKER_USERNAME }}/netflix-${{ inputs.service_name }}:${{ github.sha }}
```

---

## Task 4 – PR Pipeline

**File:** `.github/workflows/pr-pipeline.yml`

```yaml
name: PR Pipeline
on:
  pull_request:
    branches:
      - main
    types: [opened, synchronize]
    paths-ignore:
      - 'README.md'
      - '.github**'
      
  workflow_dispatch:

jobs:    
  build-test-frontend:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      working_directory: netflix-clone/frontend
      run_build: true

  build-test-backend:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      working_directory: netflix-clone/backend
      run_build: false

  security-scans:
    needs: [build-test-backend, build-test-frontend]
    uses: ./.github/workflows/reusable-security-scan.yml
    secrets: inherit
```

---

## Task 5 – Main Branch Pipeline

**File:** `.github/workflows/main.yml`

```yaml
name: Main Pipeline
on:
  push:
    branches:
      - main
    paths-ignore:
      - 'README.md'
      - '.github**'

jobs:
  build-test-frontend:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      working_directory: ./netflix-clone/frontend
      run_build: true

  build-test-backend:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      working_directory: ./netflix-clone/backend
      run_build: false

  security-scans:
    needs: [build-test-frontend, build-test-backend]
    uses: ./.github/workflows/reusable-security-scan.yml
    secrets: inherit

  docker-build-push-frontend:
    needs: build-test-frontend
    uses: ./.github/workflows/reusable-docker-build-push.yml
    with:
      service_name: frontend
      dockerfile_path: ./netflix-clone/frontend/Dockerfile
      context_path: ./netflix-clone/frontend
    secrets: inherit

  docker-build-push-backend:
    needs: build-test-backend
    uses: ./.github/workflows/reusable-docker-build-push.yml
    with:
      service_name: backend
      dockerfile_path: ./netflix-clone/backend/Dockerfile
      context_path: ./netflix-clone/backend
    secrets: inherit

  deploy:
    needs: [docker-build-push-frontend, docker-build-push-backend]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Deploy to EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            if [ ! -d "/home/ubuntu/netflix-devsecops" ]; then
              cd /home/ubuntu
              git clone https://github.com/dikshitbonu-bit/netflix-devsecops.git
            fi
            cd /home/ubuntu/netflix-devsecops
            git pull origin main
            
            cd ./netflix-clone/backend
            cat > .env << 'EOF'
            PORT=3000
            TMDB_API_KEY=${{ secrets.TMDB_API_KEY }}
            MONGODB_URI=mongodb://admin:${{ secrets.MONGO_PASSWORD }}@mongo:27017/netflix?authSource=admin
            JWT_SECRET=${{ secrets.JWT_SECRET }}
            JWT_EXPIRE=7d
            NODE_ENV=production
            EOF
            cd ..
            
            cat > .env << 'EOF'
            MONGO_INITDB_ROOT_USERNAME=admin
            MONGO_INITDB_ROOT_PASSWORD=${{ secrets.MONGO_PASSWORD }}
            MONGO_INITDB_DATABASE=netflix
            EOF
            docker compose up -d --pull always --force-recreate
            sleep 15
      
      - name: Health Check
        run: |
          for i in {1..5}; do
            curl -f http://${{ secrets.EC2_HOST }}/api/health && break || sleep 5
          done
      
      - name: Smoke Test
        run: |
          curl -f http://${{ secrets.EC2_HOST }}/api/health || exit 1
```

---

## Task 6 – Scheduled Health Check

**File:** `.github/workflows/health-check.yml`

```yaml
name: Health Check

on:
  schedule:
    - cron: '0 */12 * * *'
  workflow_dispatch:

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Pull latest backend image
        run: docker pull ${{ secrets.DOCKER_USERNAME }}/netflix-backend:latest

      - name: Run container
        run: |
          docker run -d --name health-check-app \
            -p 3000:3000 \
            -e NODE_ENV=production \
            -e PORT=3000 \
            -e TMDB_API_KEY=${{ secrets.TMDB_API_KEY }} \
            -e MONGODB_URI=mongodb://localhost:27017/netflix \
            -e JWT_SECRET=healthcheck-secret \
            ${{ secrets.DOCKER_USERNAME }}/netflix-backend:latest

      - name: Wait for startup
        run: sleep 5

      - name: Check health endpoint
        id: health
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/health || echo "000")
          echo "status=${STATUS}" >> $GITHUB_OUTPUT
          if [ "$STATUS" = "200" ]; then
            echo "result=PASSED" >> $GITHUB_OUTPUT
          else
            echo "result=FAILED (HTTP ${STATUS})" >> $GITHUB_OUTPUT
          fi

      - name: Stop and remove container
        if: always()
        run: docker stop health-check-app && docker rm health-check-app

      - name: Write step summary
        run: |
          echo "## Health Check Report" >> $GITHUB_STEP_SUMMARY
          echo "- Image: ${{ secrets.DOCKER_USERNAME }}/netflix-backend:latest" >> $GITHUB_STEP_SUMMARY
          echo "- Status: ${{ steps.health.outputs.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- Time: $(date)" >> $GITHUB_STEP_SUMMARY

      - name: Fail if unhealthy
        if: steps.health.outputs.result != 'PASSED'
        run: |
          echo "Health check failed: ${{ steps.health.outputs.result }}"
          exit 1
```

---

## Task 7 – Badges & Documentation

Badges are at the top of this file and in `README.md`.

### GitHub Secrets Required

| Secret | Used by |
|--------|---------|
| `TMDB_API_KEY` | build-test, deploy |
| `JWT_SECRET` | deploy |
| `MONGO_PASSWORD` | deploy |
| `DOCKER_USERNAME` | docker, health-check |
| `DOCKER_TOKEN` | docker, health-check |
| `EC2_HOST` | deploy |
| `EC2_USER` | deploy |
| `EC2_SSH_KEY` | deploy |
| `SONAR_TOKEN` | security-scans |
| `SONAR_HOST_URL` | security-scans |

---

## Brownie Points – DevSecOps Security Scans

**File:** `.github/workflows/reusable-security-scan.yml`  
Trivy container scanning is embedded inside `reusable-docker-build-push.yml` (Task 3) — blocks push on CRITICAL/HIGH CVEs.

```yaml
name: Security Scans

on:
  workflow_call:

jobs:
  eslint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
      
      - name: Install ESLint
        run: npm install -g eslint
      
      - name: Run ESLint on frontend
        run: npx eslint netflix-clone/frontend/src --ext .js,.jsx --max-warnings 0
        continue-on-error: false
      
      - name: Run ESLint on backend
        run: npx eslint netflix-clone/backend --max-warnings 0
        continue-on-error: false

  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: SonarQube Scan
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        with:
          args: >
           -Dsonar.projectKey=netflix-devsecops
           -Dsonar.organization=dikshitbonu-bit

  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Run GitLeaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  npm-audit:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        directory: [netflix-clone/frontend, netflix-clone/backend]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
      
      - name: Run npm audit
        working-directory: netflix-clone/backend
        run: npm audit --omit=dev --audit-level=high

  hadolint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run Hadolint on frontend Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: netflix-clone/frontend/Dockerfile
          failure-threshold: warning
      
      - name: Run Hadolint on backend Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: netflix-clone/backend/Dockerfile
          failure-threshold: warning
```

---

## What I'd Add Next

- **Slack notifications** on pipeline failure via `slackapi/slack-github-action`
- **Multi-environment** — staging → production with separate EC2s and environment-specific secrets
- **Rollback job** — on deploy failure, re-run `docker compose up` with previous SHA tag
- **DAST** — OWASP ZAP scan against deployed staging URL before promoting to production
- **Terraform** — replace manual EC2 setup with Infrastructure as Code
- **Kubernetes (EKS) + ArgoCD** — replace SSH + docker compose with GitOps deployment