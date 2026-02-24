# Day 30 – Docker Images & Container Lifecycle

## Task 1: Docker Images

### Pull Images from Docker Hub
```bash
# Pull nginx
docker pull nginx
# Output: Using default tag: latest

# Pull ubuntu
docker pull ubuntu

# Pull alpine
docker pull alpine
```

### List All Images
```bash
docker images
```


### Ubuntu vs Alpine - Size Comparison

**Ubuntu: 77.8MB**
- Full-featured Linux distribution
- Includes many utilities and libraries
- Based on Debian
- Good for general purpose

**Alpine: 7.05MB**
- Minimal Linux distribution
- Only essential tools
- Uses musl libc instead of glibc
- 10x smaller than Ubuntu
- Perfect for containers where size matters

**Why Alpine is smaller:**
- No unnecessary packages
- Minimal base system
- Optimized for containers
- Security-focused (smaller attack surface)

### Inspect an Image
```bash
docker inspect nginx
```

**Information you see:**
- Image ID
- Created date
- Architecture (amd64, arm64)
- OS (linux)
- Layers (each layer's digest)
- Environment variables
- Default command
- Exposed ports
- Labels and metadata

### Remove an Image
```bash
# Remove by name
docker rmi alpine

# Remove by ID
docker rmi abc123def456

# Force remove (if containers using it)
docker rmi -f alpine
```

---

## Task 2: Image Layers

### View Image History
```bash
docker image history nginx
```

**Output:**
```
IMAGE          CREATED       CREATED BY                                      SIZE
abc123def456   2 days ago    CMD ["nginx" "-g" "daemon off;"]                0B
def456ghi789   2 days ago    EXPOSE 80                                       0B
ghi789jkl012   2 days ago    COPY nginx.conf /etc/nginx/nginx.conf           5.2kB
jkl012mno345   2 days ago    RUN apt-get update && apt-get install...        54MB
mno345pqr678   2 days ago    ENV NGINX_VERSION=1.25.3                        0B
```

### Understanding Layers

**What are layers?**
- Each instruction in Dockerfile creates a layer
- Layers are stacked on top of each other
- Each layer contains only the changes from previous layer
- Layers are read-only once created

**Why Docker uses layers:**

1. **Caching:** If layer hasn't changed, Docker reuses it (faster builds)
2. **Sharing:** Multiple images can share same base layers (saves space)
3. **Efficiency:** Only changed layers need to be downloaded/uploaded


**Example:**
```
Image A: alpine + python + app1
Image B: alpine + python + app2

They share: alpine layer + python layer
Different: app1 vs app2 layer
```

**Why some layers show 0B:**
- Metadata-only changes (ENV, EXPOSE, CMD)
- Don't add files, just configuration
- Still counted as layers for caching

---

## Task 3: Container Lifecycle

### Complete Lifecycle Walkthrough
```bash
# 1. Create container (without starting)
docker create --name lifecycle-test nginx
docker ps -a
# STATUS: Created

# 2. Start the container
docker start lifecycle-test
docker ps
# STATUS: Up

# 3. Pause it
docker pause lifecycle-test
docker ps
# STATUS: Up (Paused)

# 4. Unpause it
docker unpause lifecycle-test
docker ps
# STATUS: Up

# 5. Stop it (graceful shutdown)
docker stop lifecycle-test
docker ps -a
# STATUS: Exited

# 6. Restart it
docker restart lifecycle-test
docker ps
# STATUS: Up

# 7. Kill it (force stop)
docker kill lifecycle-test
docker ps -a
# STATUS: Exited

# 8. Remove it
docker rm lifecycle-test
docker ps -a
# Container gone
```

### Lifecycle States
```
Created → Started → Running → Paused → Running → Stopped → Removed
   ↓         ↓         ↓         ↓         ↓         ↓
docker   docker   docker   docker   docker   docker
create   start    (running) pause   unpause  stop/kill  rm
```

**State Descriptions:**
- **Created:** Container exists but not started
- **Running:** Container actively executing
- **Paused:** Frozen (uses cgroups freezer)
- **Stopped/Exited:** Gracefully shut down
- **Dead:** Failed to stop properly (rare)

---

## Task 4: Working with Running Containers

### 1. Run Nginx in Detached Mode
```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

### 2. View Logs
```bash
# View all logs
docker logs my-nginx

# Last 20 lines
docker logs --tail 20 my-nginx
```

### 3. Real-time Logs (Follow Mode)
```bash
docker logs -f my-nginx

# Ctrl+C to exit
```

### 4. Exec into Container
```bash
docker exec -it my-nginx bash

# Inside container:
pwd                              # /
ls /usr/share/nginx/html        # index.html
cat /etc/nginx/nginx.conf       # nginx config
exit
```

### 5. Run Single Command Without Entering
```bash
# List files
docker exec my-nginx ls /usr/share/nginx/html

# Check nginx version
docker exec my-nginx nginx -v

# View running processes
docker exec my-nginx ps aux
```

### 6. Inspect Container
```bash
docker inspect my-nginx
```

**Find specific info:**
```bash
# IP Address
docker inspect my-nginx | grep IPAddress

# Or better:
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-nginx

# Port mappings
docker inspect -f '{{.NetworkSettings.Ports}}' my-nginx

# Mounts/Volumes
docker inspect -f '{{.Mounts}}' my-nginx

# State
docker inspect -f '{{.State.Status}}' my-nginx
```

**Key information in inspect:**
- IP address
- Port mappings
- Volumes/mounts
- Environment variables
- Network settings
- Resource limits
- Start time
- Exit code

---

## Task 5: Cleanup

### Stop All Running Containers
```bash
docker stop $(docker ps -q)

# -q = quiet (only IDs)
```

### Remove All Stopped Containers
```bash
docker rm $(docker ps -aq)

# -a = all
# -q = quiet
```

### Remove Unused Images
```bash
# Remove dangling images (untagged)
docker image prune

# Remove all unused images
docker image prune -a
```

### Check Disk Space Usage
```bash
docker system df
```

**Output:**
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         2         2.5GB     1.8GB (72%)
Containers      10        2         150MB     100MB (66%)
Local Volumes   3         1         500MB     300MB (60%)
Build Cache     20        0         1.2GB     1.2GB (100%)
```

### Clean Everything (Nuclear Option)
```bash
# Remove all stopped containers, unused networks, dangling images
docker system prune

# Remove everything including unused images
docker system prune -a

# Remove volumes too
docker system prune -a --volumes
```

**Warning:** `docker system prune -a` removes ALL unused images, not just dangling ones.

---

## Commands Quick Reference
```bash
# Images
docker images                           # List images
docker pull <image>                     # Download image
docker rmi <image>                      # Remove image
docker image history <image>            # View layers
docker inspect <image>                  # Detailed info

# Container Lifecycle
docker create <image>                   # Create (don't start)
docker start <container>                # Start
docker pause <container>                # Pause
docker unpause <container>              # Unpause
docker stop <container>                 # Stop (graceful)
docker kill <container>                 # Kill (force)
docker restart <container>              # Restart
docker rm <container>                   # Remove

# Working with Containers
docker logs <container>                 # View logs
docker logs -f <container>              # Follow logs
docker exec -it <container> bash        # Enter container
docker exec <container> <command>       # Run single command
docker inspect <container>              # Detailed info
docker stats                            # Resource usage

# Cleanup
docker ps -a                            # List all containers
docker stop $(docker ps -q)             # Stop all running
docker rm $(docker ps -aq)              # Remove all containers
docker image prune                      # Remove dangling images
docker system prune                     # Clean everything
docker system df                        # Disk usage
```

---

## Key Takeaways

**Images:**
- Images are templates made of layers
- Alpine is tiny (7MB) vs Ubuntu (77MB)
- Layers enable caching and sharing
- Each Dockerfile instruction = new layer

**Container Lifecycle:**
- Created → Running → Stopped → Removed
- Can pause/unpause without stopping
- Stop = graceful, Kill = force
- Containers don't disappear until removed

**Layers:**
- Read-only once created
- Stacked on top of each other
- Shared between images (saves space)
- Cached for faster builds

**Cleanup:**
- Stopped containers still use disk
- Images pile up quickly
- `docker system prune` is your friend
- Check `docker system df` regularly

