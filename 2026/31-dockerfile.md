# Day 31 – Dockerfile: Build Your Own Images

## Task 1: Your First Dockerfile

### Create Project Structure
```bash
mkdir my-first-image
cd my-first-image
touch Dockerfile
```

### Dockerfile
```dockerfile
FROM ubuntu

RUN apt-get update && apt-get install -y curl

CMD ["echo", "Hello from my custom image!"]
```

### Build the Image
```bash
docker build -t my-ubuntu:v1 .

# -t = tag (name:version)
# . = build context (current directory)
```

### Run Container
```bash
docker run my-ubuntu:v1

# Output: Hello from my custom image!
```

**What happened:**
1. Docker read the Dockerfile
2. Pulled ubuntu base image
3. Ran apt-get commands in a temporary container
4. Created a new layer with curl installed
5. Set default command to echo message
6. Tagged final image as my-ubuntu:v1

---

## Task 2: Dockerfile Instructions

### Create Project
```bash
mkdir docker-instructions
cd docker-instructions
echo "Hello World" > hello.txt
```

### Dockerfile
```dockerfile
# Base image
FROM ubuntu:22.04

# Execute commands during build (creates layer)
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean

# Set working directory (creates if doesn't exist)
WORKDIR /app

# Copy files from host to container
COPY hello.txt /app/

# Document which port the app uses (doesn't actually expose)
EXPOSE 8080

# Default command when container starts
CMD ["cat", "/app/hello.txt"]
```

### Build and Run
```bash
docker build -t instruction-demo:v1 .
docker run instruction-demo:v1

# Output: Hello World
```

### Instruction Breakdown

| Instruction | Purpose | Creates Layer? |
|-------------|---------|----------------|
| `FROM` | Sets base image | No (uses existing) |
| `RUN` | Executes commands during build | Yes |
| `COPY` | Copies files from host to image | Yes |
| `WORKDIR` | Sets working directory | No (metadata) |
| `EXPOSE` | Documents port (informational) | No (metadata) |
| `CMD` | Default command to run | No (metadata) |

---

## Task 3: CMD vs ENTRYPOINT

### 1. Image with CMD

**Dockerfile:**
```dockerfile
FROM alpine
CMD ["echo", "hello"]
```

**Build and test:**
```bash
docker build -t cmd-test .

# Run without arguments
docker run cmd-test
# Output: hello

# Run with custom command (overrides CMD)
docker run cmd-test echo "goodbye"
# Output: goodbye
```

**Result:** CMD is completely replaced by custom command.

---

### 2. Image with ENTRYPOINT

**Dockerfile:**
```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
```

**Build and test:**
```bash
docker build -t entrypoint-test .

# Run without arguments
docker run entrypoint-test
# Output: (nothing - echo with no args)

# Run with arguments (appended to ENTRYPOINT)
docker run entrypoint-test "hello world"
# Output: hello world

docker run entrypoint-test "foo" "bar"
# Output: foo bar
```

**Result:** ENTRYPOINT cannot be overridden. Arguments are appended to it.

---

### 3. Combining ENTRYPOINT and CMD

**Dockerfile:**
```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["default message"]
```

**Build and test:**
```bash
docker build -t combo-test .

# Run without arguments (uses CMD as default)
docker run combo-test
# Output: default message

# Run with arguments (replaces CMD)
docker run combo-test "custom message"
# Output: custom message
```

---

### When to Use What?

**Use CMD when:**
- You want the entire command to be overridable
- Container might run different things
- Flexibility is important
- Example: `CMD ["python", "app.py"]` - can override with `bash`

**Use ENTRYPOINT when:**
- Container should always run specific executable
- Arguments should be appended, not replaced
- Container is a tool/utility
- Example: `ENTRYPOINT ["curl"]` - container becomes curl wrapper

**Use both when:**
- ENTRYPOINT = fixed command
- CMD = default arguments (can be overridden)
- Example: `ENTRYPOINT ["nginx"]` + `CMD ["-g", "daemon off;"]`

---

## Task 4: Build a Simple Web App Image

### Create Project
```bash
mkdir my-website
cd my-website
```

### Create index.html
```html
<!DOCTYPE html>
<html>
<head>
    <title>My Docker Website</title>
</head>
<body>
    <h1>Hello from Docker!</h1>
    <p>This website is running in a container built from my custom image.</p>
    <p>Nginx + Alpine + Custom HTML = Lightweight web server</p>
</body>
</html>
```

### Dockerfile
```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/

EXPOSE 80
```

### Build and Run
```bash
# Build
docker build -t my-website:v1 .

# Run with port mapping
docker run -d -p 8080:80 --name my-site my-website:v1

# Access in browser
# http://localhost:8080
```


**Why it works:**
- Nginx serves files from `/usr/share/nginx/html/`
- We copied our HTML to that location
- Port 80 (nginx) mapped to port 8080 (host)

---

## Task 5: .dockerignore

### Create Project with Files
```bash
mkdir dockerignore-demo
cd dockerignore-demo

# Create various files
mkdir node_modules
touch node_modules/package.js
echo "# README" > README.md
echo "API_KEY=secret" > .env
git init
echo "code" > app.js
```

### Create .dockerignore
```
# Dependencies
node_modules
npm-debug.log

# Git
.git
.gitignore

# Documentation
*.md
README*

# Secrets
.env
*.key
*.pem

# OS files
.DS_Store
Thumbs.db
```

### Dockerfile
```dockerfile
FROM node:alpine
WORKDIR /app
COPY . .
CMD ["ls", "-la"]
```

### Build and Verify
```bash
docker build -t ignore-test .
docker run ignore-test

# You'll see app.js but NOT:
# - node_modules/
# - .git/
# - .env
# - README.md
```

**Why .dockerignore matters:**
- Reduces build context size
- Faster builds
- Smaller images
- Prevents secrets from being baked into image
- Security best practice

---

## Task 6: Build Optimization

### Bad Dockerfile (Cache breaks easily)
```dockerfile
FROM node:alpine
WORKDIR /app

# Code changes frequently - this breaks cache every time
COPY . .

# Dependencies rarely change - but have to reinstall every time
RUN npm install

CMD ["node", "app.js"]
```

**Problem:** Code changes → cache breaks → reinstall dependencies (slow)

---

### Optimized Dockerfile (Cache-friendly)
```dockerfile
FROM node:alpine
WORKDIR /app

# Copy dependency files first (change rarely)
COPY package.json package-lock.json ./

# Install dependencies (cached unless package files change)
RUN npm install

# Copy code last (changes frequently)
COPY . .

CMD ["node", "app.js"]
```

**Benefit:** Code changes → only last COPY rebuilds → dependencies cached (fast)

---

### Build Comparison

**First build (both):**
```bash
docker build -t app:v1 .
# All layers built from scratch
```

**Second build after changing app.js:**

**Bad Dockerfile:**
```
Step 1: FROM node:alpine              CACHED
Step 2: WORKDIR /app                  CACHED
Step 3: COPY . .                      REBUILT (code changed)
Step 4: RUN npm install               REBUILT (previous layer changed)
         ⬆ SLOW - reinstalling everything
```

**Optimized Dockerfile:**
```
Step 1: FROM node:alpine              CACHED
Step 2: WORKDIR /app                  CACHED
Step 3: COPY package*.json            CACHED (didn't change)
Step 4: RUN npm install               CACHED (previous layer didn't change)
Step 5: COPY . .                      REBUILT (code changed)
         ⬆ FAST - only copying new code
```

---

### Why Layer Order Matters

**Principle:** Put frequently-changing layers last.
```dockerfile
# Order of stability (most stable → least stable):

FROM ubuntu                    # Never changes
RUN apt-get update            # Changes monthly
COPY package.json             # Changes occasionally  
RUN npm install               # Changes when package.json changes
COPY . .                      # Changes constantly (your code)
```

**Rules:**
1. Base image first
2. System dependencies next
3. App dependencies next
4. Application code last
5. If a layer changes, all layers after it rebuild

---

## Dockerfile Best Practices

### 1. Use Specific Tags
```dockerfile
# Bad
FROM node

# Good
FROM node:18-alpine
```

### 2. Minimize Layers
```dockerfile
# Bad (3 layers)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good (1 layer)
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean
```

### 3. Use .dockerignore
```
.git
node_modules
*.md
.env
```

### 4. Don't Run as Root
```dockerfile
RUN useradd -m appuser
USER appuser
```

### 5. Multi-stage Builds (Preview)
```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/app.js"]
```

---

## Common Dockerfile Instructions
```dockerfile
FROM ubuntu:22.04              # Base image
LABEL maintainer="you@example.com"  # Metadata
ENV NODE_ENV=production        # Environment variable
WORKDIR /app                   # Set working directory
COPY src/ /app/                # Copy files/directories
ADD https://url/file /app/     # Copy + can extract archives/fetch URLs
RUN npm install                # Execute command (creates layer)
EXPOSE 3000                    # Document port (informational)
VOLUME ["/data"]               # Create mount point
USER appuser                   # Switch user
CMD ["node", "app.js"]         # Default command (overridable)
ENTRYPOINT ["docker-entrypoint.sh"]  # Fixed command
HEALTHCHECK CMD curl localhost # Health check command
```

---

## Build Commands
```bash
# Basic build
docker build -t myapp:v1 .

# Build with different Dockerfile
docker build -t myapp:v1 -f Dockerfile.prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:v1 .

# Build without cache
docker build --no-cache -t myapp:v1 .

# View build history
docker image history myapp:v1

# See layer sizes
docker image inspect myapp:v1
```

---

## Key Takeaways

**Dockerfile:**
- Recipe for building images
- Each instruction = one layer
- Order matters for caching

**CMD vs ENTRYPOINT:**
- CMD = overridable default command
- ENTRYPOINT = fixed command, args appended
- Combine for flexibility

**Build Optimization:**
- Put stable layers first
- Put changing code last
- Use .dockerignore
- Minimize layers with && chains

**Best Practices:**
- Use specific base image tags
- Don't run as root
- Use .dockerignore
- Keep images small (use alpine)
- One process per container

