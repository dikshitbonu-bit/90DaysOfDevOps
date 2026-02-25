# Day 32 – Docker Volumes & Networking

## Task 1: The Problem - Data Loss

### Run Database Container
```bash
# Run MySQL container
docker run -d --name test-db -e MYSQL_ROOT_PASSWORD=password mysql:8

# Wait for it to start
docker logs test-db

# Enter the container
docker exec -it test-db mysql -uroot -ppassword
```

### Create Some Data
```sql
CREATE DATABASE testdb;

EXIT;
```

### Stop and Remove Container
```bash
docker stop test-db
docker rm test-db
```

### Run New Container
```bash
docker run -d --name test-db -e MYSQL_ROOT_PASSWORD=password mysql:8
docker exec -it test-db mysql -uroot -ppassword
```
```sql
USE testdb;
-- ERROR: Database doesn't exist

SHOW DATABASES;
-- Only default databases, no testdb
```

### What Happened?

**Data is GONE.**

**Why:**
- Containers are ephemeral
- Data stored inside container filesystem
- When container deleted, all data deleted
- Each new container starts fresh from image

**This is a problem for databases, logs, uploads, anything that needs to persist.**

---

## Task 2: Named Volumes - The Solution

### Create Named Volume
```bash
# Create volume
docker volume create mysql-data

# List volumes
docker volume ls
# Output:
# DRIVER    VOLUME NAME
# local     mysql-data

# Inspect volume
docker volume inspect mysql-data
```

**Output:**
```json
[
    {
        "CreatedAt": "2024-01-15T10:30:00Z",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/mysql-data/_data",
        "Name": "mysql-data"
    }
]
```

### Run Container with Volume
```bash
# Run MySQL with volume attached
docker run -d \
  --name mysql-persistent \
  -e MYSQL_ROOT_PASSWORD=password \
  -v mysql-data:/var/lib/mysql \
  mysql:8

# -v volume_name:/container/path
```

### Add Data
```bash
docker exec -it mysql-persistent mysql -uroot -ppassword
```
```sql
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE users (id INT, name VARCHAR(50));
INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Charlie');
SELECT * FROM users;
EXIT;
```

### Remove Container
```bash
docker stop mysql-persistent
docker rm mysql-persistent

# Volume still exists
docker volume ls
# mysql-data is still there
```

### Run New Container with Same Volume
```bash
# New container, same volume
docker run -d \
  --name mysql-new \
  -e MYSQL_ROOT_PASSWORD=password \
  -v mysql-data:/var/lib/mysql \
  mysql:8

# Check data
docker exec -it mysql-new mysql -uroot -ppassword
```
```sql
SHOW DATABASES;
-- testdb is there!

USE testdb;
SELECT * FROM users;
-- All data intact!
-- +------+---------+
-- | id   | name    |
-- +------+---------+
-- |    1 | Alice   |
-- |    2 | Bob     |
-- |    3 | Charlie |
-- +------+---------+
```

**Data survived! Volume persists independently of containers.**

---

## Task 3: Bind Mounts

### Create HTML File on Host
```bash
mkdir ~/my-website
cd ~/my-website
```

**index.html:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Bind Mount Demo</title>
</head>
<body>
    <h1>Hello from Host Machine!</h1>
    <p>This file is on my host, served by container.</p>
</body>
</html>
```

### Run Nginx with Bind Mount
```bash
# Bind mount host directory to container
docker run -d \
  --name nginx-bind \
  -p 8080:80 \
  -v ~/my-website:/usr/share/nginx/html \
  nginx:alpine

# -v /host/path:/container/path
```

### Access in Browser
```
http://localhost:8080
```

You should see "Hello from Host Machine!"

### Edit File on Host
```bash
# Edit on host
echo '<h1>Updated from host!</h1>' > ~/my-website/index.html

# Refresh browser immediately - changes appear instantly
```

**No container restart needed. Live sync.**

---

### Named Volume vs Bind Mount

| Named Volume | Bind Mount |
|--------------|------------|
| Managed by Docker | You manage the path |
| Stored in Docker's directory | Stored anywhere on host |
| `/var/lib/docker/volumes/` | `/home/user/mydata` |
| Better for production | Better for development |
| Can't easily browse files | Easy to browse/edit files |
| Portable across systems | Tied to host filesystem |
| `docker volume create` | Just use any host path |

**When to use:**
- **Named Volume:** Databases, persistent app data, production
- **Bind Mount:** Development (live code editing), config files, logs you want to see

---

## Task 4: Docker Networking Basics

### List Networks
```bash
docker network ls
```

**Output:**
```
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   bridge    bridge    local
def456ghi789   host      host      local
ghi789jkl012   none      null      local
```

**Default networks:**
- **bridge:** Default network for containers
- **host:** Container uses host's network directly
- **none:** No networking

### Inspect Bridge Network
```bash
docker network inspect bridge
```

**Key info:**
- Subnet: `172.17.0.0/16`
- Gateway: `172.17.0.1`
- Connected containers and their IPs

### Test Communication - Default Bridge
```bash
# Run container 1
docker run -d --name container1 alpine sleep 3600

# Run container 2
docker run -d --name container2 alpine sleep 3600

# Check IPs
docker inspect container1 | grep IPAddress
# "IPAddress": "172.17.0.2"

docker inspect container2 | grep IPAddress
# "IPAddress": "172.17.0.3"
```

**Try ping by name:**
```bash
docker exec container1 ping container2
# ping: bad address 'container2'
# FAILS - name resolution doesn't work on default bridge
```

**Try ping by IP:**
```bash
docker exec container1 ping 172.17.0.3
# PING 172.17.0.3 (172.17.0.3): 56 data bytes
# 64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.123 ms
# SUCCESS - IP works
```

**Conclusion:** Default bridge allows communication by IP only, not by name.

---

## Task 5: Custom Networks

### Create Custom Network
```bash
# Create custom bridge network
docker network create my-app-net

# Verify
docker network ls
# my-app-net appears

# Inspect
docker network inspect my-app-net
```

### Run Containers on Custom Network
```bash
# Run container 1 on custom network
docker run -d --name app1 --network my-app-net alpine sleep 3600

# Run container 2 on custom network
docker run -d --name app2 --network my-app-net alpine sleep 3600
```

### Test Name-Based Communication
```bash
# Ping by name
docker exec app1 ping app2
# PING app2 (172.18.0.3): 56 data bytes
# 64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.089 ms
# SUCCESS!

# Ping reverse direction
docker exec app2 ping app1
# SUCCESS!
```

**It works! Containers can reach each other by name.**

---

### Why Custom Networks Allow Name Resolution?

**Default bridge:**
- Legacy behavior
- No built-in DNS
- Manual linking required (deprecated)

**Custom bridge:**
- Modern approach
- Built-in DNS server
- Automatic service discovery
- Container names = hostnames

**Docker's embedded DNS:**
- Runs at `127.0.0.11`
- Resolves container names to IPs
- Only works on user-defined networks
- Automatically updates when containers start/stop

---

## Task 6: Put It Together - Full Stack

### 1. Create Custom Network
```bash
docker network create app-network
```

### 2. Run Database with Volume
```bash
# Create volume for database
docker volume create postgres-data

# Run PostgreSQL on custom network with volume
docker run -d \
  --name database \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15-alpine
```

### 3. Run App Container
```bash
# Run alpine container as "app"
docker run -d \
  --name app \
  --network app-network \
  alpine sleep 3600
```

### 4. Verify App Can Reach Database
```bash
# Install postgres client in app container
docker exec app apk add --no-cache postgresql-client

# Connect to database by container name
docker exec -it app psql -h database -U postgres
# Password: secret

# Inside postgres:
\l
# Lists databases

CREATE DATABASE testdb;
\c testdb
CREATE TABLE products (id INT, name VARCHAR(50));
INSERT INTO products VALUES (1, 'Docker Book');
SELECT * FROM products;

\q
```

**Success! App connected to database using container name "database".**

---

### Architecture Visualization
```
┌─────────────────────────────────────────────────┐
│           app-network (Custom Bridge)            │
│                                                 │
│  ┌─────────────────┐      ┌─────────────────┐  │
│  │   app           │      │   database      │  │
│  │   (alpine)      │─────▶│   (postgres)    │  │
│  │                 │ ping │                 │  │
│  └─────────────────┘      └────────┬────────┘  │
│                                     │           │
│                                     │ volume    │
└─────────────────────────────────────┼───────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │ postgres-data   │
                             │ (named volume)  │
                             └─────────────────┘
```

---

## Volume Commands Reference
```bash
# Create volume
docker volume create volume_name

# List volumes
docker volume ls

# Inspect volume
docker volume inspect volume_name

# Remove volume
docker volume rm volume_name

# Remove unused volumes
docker volume prune

# Use volume with container
docker run -v volume_name:/path/in/container image

# Bind mount
docker run -v /host/path:/container/path image

# Read-only volume
docker run -v volume_name:/path:ro image
```

---

## Network Commands Reference
```bash
# List networks
docker network ls

# Create network
docker network create network_name

# Create network with subnet
docker network create --subnet=172.20.0.0/16 network_name

# Inspect network
docker network inspect network_name

# Remove network
docker network rm network_name

# Remove unused networks
docker network prune

# Run container on network
docker run --network network_name image

# Connect running container to network
docker network connect network_name container_name

# Disconnect container from network
docker network disconnect network_name container_name
```

---

## Key Takeaways

**Volumes:**
- Solve data persistence problem
- Exist independently of containers
- Named volumes managed by Docker
- Bind mounts use host filesystem
- Always use volumes for databases

**Named Volume vs Bind Mount:**
- Named volume = production, managed by Docker
- Bind mount = development, live editing

**Networking:**
- Default bridge = IP-only communication
- Custom bridge = name-based DNS
- Isolated networks for security
- Container name = hostname on custom networks

**Best Practices:**
- One volume per data concern (db data, uploads, logs)
- Use custom networks for multi-container apps
- Never store data inside container filesystem
- Name your volumes meaningfully
- Use network isolation for security

**Real-World Pattern:**
```bash
# Create infrastructure
docker network create myapp
docker volume create myapp-db

# Run services
docker run -d --name db --network myapp -v myapp-db:/data postgres
docker run -d --name api --network myapp -p 3000:3000 myapi:v1
docker run -d --name web --network myapp -p 80:80 nginx
```
