# Day 49 – DevSecOps: Security Added to CI/CD Pipeline

---


## Full Secure Pipeline Diagram

```
PR opened → main
│
├── build-test-frontend
├── build-test-backend
├── security-scans
│     ├── Gitleaks          (secret detection in code)
│     ├── SonarQube         (SAST)
│     ├── ESLint            (code quality)
│     ├── npm audit         (dependency CVEs)
│     └── Hadolint          (Dockerfile lint)
├── dependency-review       ← NEW (Task 3)
└── [No Docker push on PRs]

────────────────────────────────────────

Push to main
│
├── build-test-frontend + build-test-backend  (parallel)
├── security-scans
├── docker-build-push-frontend
│     ├── Build image
│     ├── Trivy scan → CRITICAL/HIGH = fail  ← NEW (Task 1)
│     └── Push only if scan passes
├── docker-build-push-backend  (same, parallel)
└── deploy (environment: production)

────────────────────────────────────────

Always active (no workflow needed)
│
├── GitHub Secret Scanning   ← NEW (Task 2)
└── Push Protection          ← NEW (Task 2)
```

---

## Task 1 – Scan Docker Image for Vulnerabilities (Trivy)

Trivy is already integrated inside `reusable-docker-build-push.yml`. It runs **after** the image is built and **before** it is pushed — so a vulnerable image never reaches Docker Hub.

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
    permissions:
      contents: read
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
      
      - name: Scan Docker image for vulnerabilities
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

**Base images used:**
- Frontend: `nginx:alpine`
- Backend: `node:22-alpine`

Alpine-based images have a significantly smaller attack surface than full Debian/Ubuntu images — fewer packages = fewer CVEs.

---

## Task 2 – GitHub Secret Scanning

**Setup (no YAML needed):**  
Repo → Settings → Code security and analysis → Enable **Secret scanning** + **Push protection**

**Secret scanning vs Push protection:**

| | Secret Scanning | Push Protection |
|---|---|---|
| When it runs | After the push, scans existing commits | Before the push is accepted |
| What it does | Alerts you that a secret was found in history | Blocks the push entirely |
| Fixes needed | Rotate the secret + purge git history | Fix the commit before pushing |

**If GitHub detects a leaked AWS key:**  
GitHub automatically notifies AWS, which revokes the key. GitHub also alerts the repo owner via email and creates a security alert in the repo's Security tab. The key is considered compromised the moment it was pushed — rotation is mandatory even if caught immediately.

---

## Task 3 – Dependency Review on PRs

Added to `pr-pipeline.yml` — runs only on `pull_request` events, checks any newly added packages against the GitHub Advisory Database. Fails the PR if a dependency with a CRITICAL CVE is introduced.

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

permissions:
  contents: read
  pull-requests: write

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

  dependency-review:
    runs-on: ubuntu-latest
    needs: [build-test-frontend, build-test-backend]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Check dependencies for vulnerabilities
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: critical
```

---

## Task 4 – Workflow Permissions

Added `permissions` blocks to `main.yml` and `pr-pipeline.yml`.

**Why limit permissions?**  
By default GitHub gives workflows `GITHUB_TOKEN` with broad write access. If a third-party action in your pipeline is compromised (supply chain attack), it could use that token to push malicious code, delete branches, or exfiltrate secrets. `contents: read` means even a fully compromised action can only read the repo — it cannot write or modify anything.

**File:** `.github/workflows/main.yml` — with `permissions` added

```yaml
name: Main Pipeline
on:
  push:
    branches:
      - main
    paths-ignore:
      - 'README.md'
      - '.github**'

permissions:
  contents: read

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

## Task 5 – Full Secure Pipeline (Summary)

| Stage | Tool | What it catches | Blocks pipeline? |
|---|---|---|---|
| PR | Gitleaks | Secrets in code | Yes |
| PR | SonarQube | Code-level vulns (SAST) | Yes |
| PR | ESLint | Insecure code patterns | Yes |
| PR | npm audit | Known CVEs in packages | Yes (HIGH+) |
| PR | Hadolint | Dockerfile misconfigs | Yes |
| PR | Dependency review | New CVEs added by PR | Yes (CRITICAL) |
| Main | Trivy | CVEs in Docker image | Yes (CRITICAL/HIGH) |
| Always | Secret scanning | Leaked tokens/keys | Alert + notify |
| Always | Push protection | Secrets in commits | Blocks push |