# Day 33 – Docker Compose: Multi-Container Basics

## Task 1: Install & Verify

### Check Docker Compose
```bash
# Check if installed
docker compose version

# Output:
# Docker Compose version v2.24.0
```

**Note:** Modern Docker includes Compose V2 (`docker compose`). Old version was `docker-compose` (with hyphen).

---

## Task 2: Your First Compose File

### Create Project
```bash
mkdir compose-basics
cd compose-basics
touch docker-compose.yml
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
```

### Start the Service
```bash
# Start in foreground
docker compose up

# You'll see:
# [+] Running 1/1
# ✔ Container compose-basics-web-1 Started

# Access: http://localhost:8080
```

### Stop the Service
```bash
# Ctrl+C to stop (if running in foreground)

# Or from another terminal:
docker compose down

# Output:
# [+] Running 1/1
# ✔ Container compose-basics-web-1 Removed
# ✔ Network compose-basics_default Removed
```

**What happened:**
- Compose created a network automatically
- Started nginx container
- Mapped port 8080 to 80
- On `down`, removed container and network

---

## Task 3: Two-Container Setup - WordPress

### Create Project
```bash
mkdir wordpress-app
cd wordpress-app
touch docker-compose.yml
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  db:
    image: mysql:8
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    restart: always

  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db
    restart: always

volumes:
  db_data:
```

### Start the Stack
```bash
docker compose up -d

# Output:
# [+] Running 3/3
# ✔ Network wordpress-app_default Created
# ✔ Container wordpress-app-db-1 Started
# ✔ Container wordpress-app-wordpress-1 Started
```

### Access WordPress
```
http://localhost:8080
```

Complete the WordPress setup:
- Choose language
- Create admin account
- Install WordPress

### Test Data Persistence
```bash
# Stop everything
docker compose down

# Output:
# [+] Running 2/2
# ✔ Container wordpress-app-wordpress-1 Removed
# ✔ Container wordpress-app-db-1 Removed
# ✔ Network wordpress-app_default Removed

# Volume still exists!
docker volume ls | grep db_data
# wordpress-app_db_data

# Start again
docker compose up -d

# Access http://localhost:8080
# WordPress is configured! Data persisted!
```

**Key points:**
- WordPress connects to MySQL using service name `db`
- Compose creates network automatically
- Both services on same network
- Volume `db_data` persists MySQL data
- `depends_on` ensures db starts before wordpress

---

## Task 4: Compose Commands

### Start Services
```bash
# Foreground (see logs)
docker compose up

# Detached mode (background)
docker compose up -d

# Start specific service
docker compose up -d db
```

### View Running Services
```bash
docker compose ps

# Output:
# NAME                        IMAGE               STATUS
# wordpress-app-db-1          mysql:8            Up 2 minutes
# wordpress-app-wordpress-1   wordpress:latest   Up 2 minutes
```

### View Logs
```bash
# All services
docker compose logs

# Follow logs (live)
docker compose logs -f

# Specific service
docker compose logs wordpress
docker compose logs -f db

# Last 50 lines
docker compose logs --tail=50
```

### Stop Services (Keep Containers)
```bash
# Stop without removing
docker compose stop

# Restart stopped services
docker compose start
```

### Stop and Remove Everything
```bash
# Remove containers and networks
docker compose down

# Remove containers, networks, AND volumes
docker compose down -v

# Remove containers, networks, volumes, AND images
docker compose down -v --rmi all
```

### Rebuild Images
```bash
# If you change Dockerfile, rebuild
docker compose build

# Build and start
docker compose up --build

# Force rebuild (no cache)
docker compose build --no-cache
```

### Other Useful Commands
```bash
# Execute command in running container
docker compose exec wordpress bash

# Run one-off command
docker compose run wordpress ls /var/www/html

# View resource usage
docker compose stats

# Pause services
docker compose pause

# Unpause services
docker compose unpause
```

---

## Task 5: Environment Variables

### Method 1: Direct in Compose File

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    image: nginx:alpine
    ports:
      - "8080:80"
    environment:
      - NGINX_HOST=localhost
      - NGINX_PORT=80
      - ENV=production
```

### Method 2: Using .env File

**Create .env file:**
```bash
# Database config
DB_ROOT_PASSWORD=supersecret
DB_NAME=myapp
DB_USER=appuser
DB_PASSWORD=apppass123

# App config
APP_PORT=8080
APP_ENV=production
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  app:
    image: myapp:latest
    ports:
      - "${APP_PORT}:3000"
    environment:
      - NODE_ENV=${APP_ENV}
      - DATABASE_URL=mysql://${DB_USER}:${DB_PASSWORD}@db:3306/${DB_NAME}
    depends_on:
      - db

volumes:
  db_data:
```

### Verify Variables
```bash
# Start services
docker compose up -d

# Check environment variables in container
docker compose exec app env

# Or
docker compose exec app printenv NODE_ENV
```

**Important:**
- `.env` file must be in same directory as `docker-compose.yml`
- Variables loaded automatically
- Add `.env` to `.gitignore` (contains secrets!)

---

## Compose File Structure Explained
```yaml
version: '3.8'              # Compose file version

services:                   # Define containers
  service_name:             # Your custom name (becomes DNS name)
    image: image:tag        # Pull from registry
    # OR
    build: ./path           # Build from Dockerfile
    
    ports:                  # Port mapping
      - "host:container"
    
    volumes:                # Mount volumes
      - volume_name:/path
      - /host/path:/container/path
    
    environment:            # Environment variables
      - KEY=value
      - KEY=${ENV_VAR}
    
    env_file:               # Load from file
      - .env
    
    depends_on:             # Start order
      - other_service
    
    networks:               # Custom networks
      - my_network
    
    restart: always         # Restart policy
    
    command: ["echo", "hi"] # Override default command

volumes:                    # Named volumes
  volume_name:

networks:                   # Custom networks
  my_network:
```

---

## Commands Quick Reference
```bash
# Start
docker compose up                    # Foreground
docker compose up -d                 # Detached
docker compose up --build            # Rebuild images

# Stop
docker compose stop                  # Stop (keep containers)
docker compose down                  # Stop and remove
docker compose down -v               # Remove volumes too

# View
docker compose ps                    # List services
docker compose logs                  # View logs
docker compose logs -f               # Follow logs
docker compose logs service_name     # Specific service

# Execute
docker compose exec service_name bash    # Enter container
docker compose exec service_name cmd     # Run command

# Manage
docker compose build                 # Build images
docker compose pull                  # Pull images
docker compose restart               # Restart services
docker compose pause                 # Pause services
docker compose unpause               # Unpause services

# Cleanup
docker compose down --rmi all        # Remove everything
docker compose down -v --rmi all     # Nuclear option
```

---

## Key Takeaways

**What Compose Solves:**
- Multi-container apps in one file
- No manual network creation
- No manual volume creation
- Service discovery automatic
- One command to start/stop everything

**Service Names = DNS Names:**
- Service `db` is reachable at hostname `db`
- Service `api` is reachable at hostname `api`
- No IP addresses needed

**Compose vs Docker CLI:**

| Task | Docker CLI | Docker Compose |
|------|------------|----------------|
| Run 2 containers | 2 commands | 1 command |
| Create network | Manual | Automatic |
| Link containers | Manual | Automatic |
| Environment vars | Long flags | YAML config |
| Recreate setup | Remember commands | `docker compose up` |

**Best Practices:**
- One compose file per project
- Use named volumes for data
- Use `.env` for secrets
- Add `.env` to `.gitignore`
- Use `depends_on` for start order
- Use specific image tags, not `latest`

**Workflow:**
```bash
# Development
docker compose up        # See logs

# Production
docker compose up -d     # Detached
docker compose logs -f   # View logs separately
```

