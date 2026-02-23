# Day 29 – Introduction to Docker

## Task 1: What is Docker?

### What is a container and why do we need them?

A container is a lightweight, isolated environment that packages an application and all its dependencies together. Think of it as a shipping container for code.

**Why we need them:**
- "Works on my machine" problem solved - runs the same everywhere
- Lightweight - shares host OS kernel, starts in seconds
- Isolated - apps don't interfere with each other
- Portable - build once, run anywhere
- Efficient - one server can run hundreds of containers

### Containers vs Virtual Machines

| Containers | Virtual Machines |
|------------|------------------|
| Share host OS kernel | Each VM has full OS |
| Lightweight (MBs) | Heavy (GBs) |
| Start in seconds | Start in minutes |
| More containers per host | Fewer VMs per host |
| Process-level isolation | Hardware-level isolation |
| Docker, Podman | VMware, VirtualBox |

**Simple analogy:**
- VM = entire house with own plumbing, electricity
- Container = apartment in a building sharing utilities

### Docker Architecture
```
┌─────────────────────────────────────────────────────┐
│                   Docker Client                      │
│           (docker commands you run)                  │
└──────────────────┬──────────────────────────────────┘
                   │
                   │ REST API
                   ▼
┌─────────────────────────────────────────────────────┐
│                Docker Daemon (dockerd)               │
│         (background service managing everything)     │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │Container │  │Container │  │Container │         │
│  │          │  │          │  │          │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│       │             │             │                │
│       └─────────────┴─────────────┘                │
│                     │                              │
│              ┌──────▼──────┐                       │
│              │   Images    │                       │
│              │  (templates) │                       │
│              └─────────────┘                       │
└─────────────────────────────────────────────────────┘
                   │
                   │ pulls/pushes
                   ▼
┌─────────────────────────────────────────────────────┐
│            Docker Registry (Docker Hub)              │
│         (public/private image storage)               │
└─────────────────────────────────────────────────────┘
```

**Components:**

- **Docker Client:** CLI tool (docker commands)
- **Docker Daemon:** Background service that manages containers
- **Images:** Read-only templates (blueprints for containers)
- **Containers:** Running instances of images
- **Registry:** Storage for images (Docker Hub, private registries)

**Flow:**
1. You run `docker run nginx`
2. Client sends request to daemon
3. Daemon checks if image exists locally
4. If not, pulls from Docker Hub
5. Creates container from image
6. Runs it

---

## Task 2: Install Docker

### Installation

**Linux (Ubuntu):**
```bash
# Update packages
sudo apt update

# Install Docker
sudo apt install docker.io -y

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (avoid sudo)
sudo usermod -aG docker $USER
newgrp docker
```

**Verify Installation:**
```bash
docker --version
# Output: Docker version 24.0.x

docker info
# Shows Docker system info
```

**Run Hello World:**
```bash
docker run hello-world
```

**Output explanation:**
```
1. Client contacted daemon
2. Daemon pulled "hello-world" image from Docker Hub
3. Daemon created container from image
4. Daemon ran container (printed message)
5. Container exited
```

This proves Docker is installed and working.

---

## Task 3: Run Real Containers

### 1. Run Nginx Container
```bash
# Run Nginx
docker run -d -p 8080:80 --name my-nginx nginx

# -d = detached mode (background)
# -p 8080:80 = map host port 8080 to container port 80
# --name = custom name
# nginx = image name
```

**Access in browser:**
```
http://localhost:8080
```

You should see "Welcome to nginx!" page.

### 2. Run Ubuntu Container (Interactive)
```bash
# Run Ubuntu interactively
docker run -it ubuntu bash

# -i = interactive (keep STDIN open)
# -t = allocate pseudo-TTY (terminal)
# bash = command to run inside container

# You're now inside Ubuntu container
cat /etc/os-release
ls
pwd
# Exit with: exit
```

### 3. List Running Containers
```bash
docker ps

# Output shows:
# CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

### 4. List All Containers (including stopped)
```bash
docker ps -a

# Shows all containers (running + stopped)
```

### 5. Stop and Remove Container
```bash
# Stop container
docker stop my-nginx

# Remove container
docker rm my-nginx

# Or force remove running container
docker rm -f my-nginx
```

---

## Task 4: Explore

### 1. Detached Mode vs Foreground
```bash
# Foreground (blocks terminal)
docker run nginx
# Ctrl+C to stop

# Detached (runs in background)
docker run -d nginx
# Returns container ID, terminal is free
```

### 2. Custom Container Name
```bash
docker run -d --name web-server nginx

# Without --name, Docker assigns random name like "quirky_tesla"
```

### 3. Port Mapping
```bash
# Map host port 3000 to container port 80
docker run -d -p 3000:80 --name nginx-3000 nginx

# Access: http://localhost:3000
```

**Port mapping syntax:** `-p HOST_PORT:CONTAINER_PORT`

### 4. Check Container Logs
```bash
# View logs
docker logs my-nginx

# Follow logs (live)
docker logs -f my-nginx

# Last 50 lines
docker logs --tail 50 my-nginx
```

### 5. Run Command Inside Running Container
```bash
# Execute command in running container
docker exec -it my-nginx bash

# Now you're inside the running container
ls /usr/share/nginx/html
cat /etc/nginx/nginx.conf
exit

# Run single command without entering
docker exec my-nginx ls /usr/share/nginx/html
```

---

## Commands Summary
```bash
# Basic
docker run <image>              # Run container
docker ps                       # List running containers
docker ps -a                    # List all containers
docker stop <container>         # Stop container
docker rm <container>           # Remove container
docker images                   # List images
docker rmi <image>              # Remove image

# Run options
-d                              # Detached mode
-it                             # Interactive with terminal
-p <host>:<container>           # Port mapping
--name <name>                   # Custom name
-v <host>:<container>           # Volume mount
-e KEY=VALUE                    # Environment variable

# Management
docker logs <container>         # View logs
docker logs -f <container>      # Follow logs
docker exec -it <container> bash # Enter container
docker inspect <container>      # Detailed info
docker stats                    # Resource usage

# Cleanup
docker stop $(docker ps -aq)    # Stop all containers
docker rm $(docker ps -aq)      # Remove all containers
docker system prune             # Remove unused data
```

---


## Key Takeaways

**What I Learned:**
- Containers are lightweight, isolated environments
- Docker uses client-daemon architecture
- Images are templates, containers are running instances
- Port mapping connects host to container
- Detached mode runs containers in background
- Can execute commands inside running containers

**Containers vs VMs:**
- Containers share OS kernel = faster, lighter
- VMs have full OS = heavier, slower
- Containers for apps, VMs for full isolation

**Docker Workflow:**
1. Pull image from registry
2. Run container from image
3. Container runs isolated app
4. Stop/remove when done

