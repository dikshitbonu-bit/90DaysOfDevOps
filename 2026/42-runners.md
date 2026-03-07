# Day 42 – Runners: GitHub-Hosted & Self-Hosted.

---

## Task 1: GitHub-Hosted Runners

### .github/workflows/multi-os.yml
```yaml
name: Multi-OS Test

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test-ubuntu:
    runs-on: ubuntu-latest
    
    steps:
      - name: Print Ubuntu info
        run: |
          echo "OS: Ubuntu Linux"
          echo "Hostname: $(hostname)"
          echo "Current user: $(whoami)"
          
  
  test-windows:
    runs-on: windows-latest
    
    steps:
      - name: Print Windows info
        run: |
          echo "OS: Windows"
          echo "Hostname: $env:COMPUTERNAME"
          echo "Current user: $env:USERNAME"
          echo "Home directory: $env:USERPROFILE"
        shell: pwsh
  
  test-macos:
    runs-on: macos-latest
    
    steps:
      - name: Print macOS info
        run: |
          echo "OS: macOS"
          echo "Hostname: $(hostname)"
          echo "Current user: $(whoami)"
          echo "Home directory: $HOME"
          sw_vers
```

### What is a GitHub-Hosted Runner?

**Definition:**
A virtual machine provided and managed by GitHub to run your workflows.

**Who manages it:**
- GitHub manages everything
- Pre-configured with tools
- Fresh VM for each job
- Automatically cleaned up after job completes

**Characteristics:**
- Ephemeral (temporary)
- Isolated (each job gets clean VM)
- No persistent state between runs
- Managed by GitHub (OS updates, security patches)

---

## Task 2: Explore Pre-installed Tools

### .github/workflows/check-tools.yml
```yaml
name: Check Pre-installed Tools

on:
  workflow_dispatch:

jobs:
  check-tools:
    runs-on: ubuntu-latest
    
    steps:
      - name: Check installed versions
        run: |
          echo "=== Docker ==="
          docker --version
          docker compose version
          
          echo "=== Python ==="
          python --version
          pip3 --version
          
          echo "=== Node.js ==="
          node --version
          npm --version
          
          echo "=== Git ==="
          git --version
          
          echo "=== Other Tools ==="
          java -version
          go version
          ruby --version
          php --version
          
```

### Pre-installed Software on ubuntu-latest

**Languages & Runtimes:**
- Python 3.x (multiple versions)
- Node.js (multiple versions)
- Ruby
- PHP
- Java (multiple versions)
- Go
- .NET Core

**Tools:**
- Docker + Docker Compose
- Git, Git LFS
- kubectl, Helm
- AWS CLI, Azure CLI, gcloud
- Terraform
- Ansible
- Maven, Gradle
- npm, yarn, pip

**Databases:**
- PostgreSQL
- MySQL
- MongoDB (client)


### Why Pre-installed Tools Matter

**Benefits:**
- No setup time (ready to use)
- Faster workflows (skip installation steps)
- Consistent versions across runs
- Less workflow code to maintain


---

### Step 5: Start the Runner

**Option 1: Run interactively (testing)**
```bash
./run.sh

# Output:
# Connected to GitHub
# Listening for Jobs
```

**Option 2: Run as service (production)**
```bash
# Install service
sudo ./svc.sh install

# Start service
sudo ./svc.sh start

# Check status
sudo ./svc.sh status

# Stop service
sudo ./svc.sh stop
```

---

## Task 4: Use Self-Hosted Runner

### .github/workflows/self-hosted.yml
```yaml
name: Self-Hosted Runner Test

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  run-on-my-machine:
    runs-on: self-hosted
    
    steps:
      
      - name: Print hostname
        run: |
          echo "Running on: $(hostname)"
          echo "This is MY machine, not GitHub's!"
      
      - name: Print working directory
        run: |
          echo "Current directory: $(pwd)"
          echo "Contents:"
          ls -la
      
      - name: Print system info
        run: |
          echo "User: $(whoami)"
          echo "OS: $(uname -a)"
          echo "IP: $(hostname -I)"
      
      - name: Create test file
        run: |
          echo "Hello from GitHub Actions on $(date)" > test-output.txt
          echo "File created at: $(pwd)/test-output.txt"
          cat test-output.txt
      
    
```
---



### Why Labels Are Useful

**Scenario: Multiple self-hosted runners**
```
Runners:
- runner-1: [self-hosted, linux, production, us-east]
- runner-2: [self-hosted, linux, staging, us-west]
- runner-3: [self-hosted, windows, production]
- runner-4: [self-hosted, linux, gpu, ml-training]
```

**Target specific runner:**
```yaml
jobs:
  prod-deploy:
    runs-on: [self-hosted, linux, production, us-east]
  
  staging-deploy:
    runs-on: [self-hosted, linux, staging]
  
  ml-training:
    runs-on: [self-hosted, gpu, ml-training]
  
  windows-build:
    runs-on: [self-hosted, windows]
```

**Benefits:**
- Route jobs to appropriate hardware
- Separate production from staging
- Use GPU runners for ML workloads
- Geographic distribution (us-east vs us-west)
- Cost optimization (use expensive runners only when needed)

---

## Task 6: GitHub-Hosted vs Self-Hosted

| Aspect | GitHub-Hosted | Self-Hosted |
|--------|---------------|-------------|
| **Who manages it** | GitHub (automatic updates, maintenance) | You (OS updates, security, monitoring) |
| **Cost** | Free tier: 2,000 minutes/month (Linux)<br>Paid: $0.008/minute (Linux)<br>$0.016/minute (Windows)<br>$0.08/minute (macOS) | Infrastructure cost only<br>No per-minute charges<br>Pay for VM/hardware |
| **Pre-installed tools** | Extensive list (Docker, Node, Python, etc.)<br>Auto-updated | Install yourself<br>Maintain yourself |
| **Setup time** | Instant (just specify OS) | Manual setup required<br>Configuration needed |
| **Performance** | Standard VM specs<br>2-core, 7GB RAM (Linux)<br>4-core, 16GB RAM (macOS) | Your hardware<br>Can be more powerful<br>Or cheaper/weaker |
| **Persistence** | Ephemeral (deleted after job)<br>No state between runs | Persistent<br>Files remain between runs<br>Can cache globally |
| **Good for** | Standard CI/CD<br>Multi-OS testing<br>Open source projects<br>No infrastructure management | Private data/code<br>Custom hardware (GPU)<br>Cost savings (heavy usage)<br>Network restrictions |
| **Security concern** | Code runs on shared infrastructure<br>Logs visible to GitHub | Full control over environment<br>Secrets stay in your network<br>Must secure yourself |
| **Availability** | High availability (GitHub manages) | You ensure uptime<br>Your network/power issues = downtime |
| **Scaling** | Automatic (GitHub provides runners) | Manual (provision more VMs/machines) |
| **Network access** | Public internet only | Can access private networks<br>Internal databases, APIs |

---


## Key Takeaways

**GitHub-Hosted:**
- Managed by GitHub
- Pay per minute
- Pre-configured tools
- Ephemeral (no persistence)
- Good for standard CI/CD

**Self-Hosted:**
- You manage infrastructure
- Pay for VM only (unlimited minutes)
- Custom hardware possible
- Persistent state
- Good for high usage, private networks, special hardware

**Labels:**
- Route jobs to specific runners
- Essential with multiple runners
- Combine multiple labels for precision


---
