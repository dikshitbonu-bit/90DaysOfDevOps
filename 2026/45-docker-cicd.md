# Day 45 – Docker Build & Push in GitHub Actions

Complete CI/CD pipeline. Code pushed to GitHub automatically builds Docker image and ships to Docker Hub.

---



## Task 2: Build Docker Image in CI

### .github/workflows/docker-publish.yml
````yaml
name: Docker Build and Push

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: test-build:latest
````

Build successful ✓

---

## Task 3: Push to Docker Hub

### Updated .github/workflows/docker-publish.yml
````yaml
name: Docker Build and Push

on:
  push:
    branches: [main]

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
          images: ${{ secrets.DOCKER_USERNAME }}/github-actions-app
          tags: |
            type=raw,value=latest
            type=sha,prefix=sha-,format=short
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            GIT_COMMIT=${{ github.sha }}
````


---

## Task 4: Only Push on Main

### Final .github/workflows/docker-publish.yml
````yaml
name: Docker Build and Push

on:
  push:
    branches: [main] 

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKER_USERNAME }}/docker-cicd-demo
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,prefix=sha-,format=short
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
        
         
````

### Test on Feature Branch
````bash
git checkout -b feature/test
git push origin feature/test
````

**Result:**
- Build runs ✓
- Login skipped ⊘
- Push skipped ⊘
- Image NOT on Docker Hub ✓

**On main:**
- Build runs ✓
- Login runs ✓
- Push runs ✓
- Image on Docker Hub ✓

---

## Task 5: Add Status Badge

### Get Badge URL

Actions tab → Click workflow → Three dots (...) → Create status badge

**Badge URL:**
````
https://github.com/USERNAME/REPO/actions/workflows/docker-publish.yml/badge.svg
````

### Update README.md
````markdown
# Docker CI/CD Demo

![Docker Build](https://github.com/USERNAME/REPO/actions/workflows/docker-publish.yml/badge.svg)

Automated Docker build and push pipeline using GitHub Actions.

## Running the App
```bash
docker pull username/docker-cicd-demo:latest
docker run -p 3000:3000 username/docker-cicd-demo:latest
```

Visit: http://localhost:3000
````

**Push README.**

Badge shows green ✓

---

## Task 6: Pull and Run

### On Local Machine
````bash
# Pull image
docker pull username/docker-cicd-demo:latest

# Run container
docker run -d -p 3000:3000 --name cicd-demo username/docker-cicd-demo:latest

# Check logs
docker logs cicd-demo


# Or open in browser
open http://localhost:3000
````

**App displays:**
- Day 45 - Docker CI/CD Pipeline
- Glassy black design
- Badges for Node.js, Docker, GitHub Actions
- Commit SHA from build

**Cleanup:**
````bash
docker stop cicd-demo
docker rm cicd-demo
````

---
