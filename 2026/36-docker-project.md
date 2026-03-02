# Day 36 – Docker Project: Dockerize a Full Application

## What I Built

Today was not just about Dockerizing an app. I treated it like a real production deployment — app running, database persisting, and a full monitoring stack watching over everything in real time.

I built two things:

**1. Flask Todo App with MySQL** — a full CRUD web application containerized with Docker, connected to a MySQL database, with volumes for persistence, healthchecks, environment variables, and a custom network.

**2. Prometheus + Grafana + Node Exporter Monitoring Stack** — a production-grade observability platform that monitors host system metrics in real time. Instead of just running the app and calling it done, I asked myself — "how do I know the system is healthy?" and then answered that question by setting up a full monitoring stack alongside the app.

Together these two projects cover every requirement of Day 36 and go beyond it.

---

## Project Repositories

- **Flask Todo App** — [github.com/dikshitbonu-bit/Docker-projects/tree/main/todo-python-app](https://github.com/dikshitbonu-bit/Docker-projects/tree/main/todo-python-app)
- **Monitoring Stack** — [github.com/dikshitbonu-bit/Docker-projects/tree/main/Monitoring-stack](https://github.com/dikshitbonu-bit/Docker-projects/tree/main/Monitoring-stack)

- **Dockerhub image of Todo app** - [https://hub.docker.com/repository/docker/dikshithbonu/todo-app/general]

---

## Task 1: The App — Flask Todo with MySQL

A simple CRUD todo application built with Python Flask and MySQL. Users can create, read, update and delete tasks. The app runs in a Docker container, talks to a MySQL container over a custom Docker network, and persists data through named volumes so nothing is lost on container restarts.

Flask is lightweight and straightforward — perfect for demonstrating containerization concepts without the app code getting in the way. MySQL is a real production database that shows how to handle stateful services in Docker properly with volumes and healthchecks.

---

## Task 2: The Dockerfile

The Dockerfile for the Flask app uses a non-root user for security, a slim Python base image to keep the size down, and separates dependency installation from app code copying so Docker can cache layers efficiently. A `.dockerignore` file excludes `__pycache__`, `.env`, `.git`, and other unnecessary files from being copied into the image.

---

## Task 3: Docker Compose

The docker-compose file wires together the Flask app service and MySQL database service on a custom bridge network. MySQL uses a named volume so data persists across restarts. Environment variables for database credentials are loaded from a `.env` file. The database has a healthcheck so the app container only starts after MySQL is confirmed healthy using `depends_on` with `condition: service_healthy`.

---

## Task 4: The Monitoring Stack — Production Thinking

After the todo app was running I set up a three service monitoring stack alongside it to observe real system metrics in real time.

**Node Exporter** — exposes system metrics at port 9100. It reads CPU and memory data from `/proc` — a virtual filesystem the Linux kernel generates live in memory, hardware and network data from `/sys`, and real disk usage by mounting the host root filesystem. It uses `pid: host` to see all host processes.

**Prometheus** — scrapes Node Exporter every 15 seconds and stores all metrics in its own built-in time series database. It uses a `prometheus.yml` config file that tells it where to find Node Exporter. A named volume persists the database across restarts. It has a healthcheck on its `/-/healthy` endpoint.

**Grafana** — connects to Prometheus as a data source and visualizes all metrics through dashboards. Admin credentials are loaded from a `.env` file via environment variables. A named volume persists dashboards so they survive container restarts. It has a healthcheck on its `/api/health` endpoint and only starts after Prometheus is confirmed healthy.

All three services run on the same custom `monitoring` bridge network so they can reach each other by container name.

### Key Design Decisions

**No custom Dockerfile** — all three monitoring tools have official images on Docker Hub. No custom building needed. This is how real teams deploy monitoring — pull official images and configure them.

**Specific image versions pinned** — never using `latest` in any service because `latest` can change silently and break your stack at any time.

**Node Exporter on bridge network not host network** — for simplicity and to keep all services on the same network. The tradeoff is network metrics reflect the container interface rather than the true host interface. In production Node Exporter would run as a systemd service directly on the host, or use `extra_hosts: host-gateway` to bridge the gap.

**Volumes on both Prometheus and Grafana** — Prometheus needs a volume because its time series database lives on disk. Grafana needs a volume to save dashboards so you do not have to recreate them every restart.

**The filesystem exclude flag** — Node Exporter mounts `/proc`, `/sys`, and `/` from the host into the container. Without the exclude flag it would try to measure `/proc` and `/sys` as real disks — but they are virtual filesystems with no actual disk behind them. The flag tells only the filesystem collector to skip them. CPU and memory collectors are unaffected.

---

## Challenges Faced

**Network mismatch between Node Exporter and Prometheus** — initially Node Exporter used `network_mode: host` which meant it was not on any Docker network. Prometheus could not reach it by container name. Solved by switching Node Exporter to a regular bridge network with port mapping.

**Environment variable syntax in compose** — used `$(VAR)` shell syntax instead of `${VAR}` Docker Compose syntax for Grafana credentials. Docker Compose uses `${}` to read from the `.env` file, not shell substitution.

**`depends_on` not waiting for readiness** — by default `depends_on` only waits for a container to start, not for it to be actually ready. Solved by adding healthchecks and using `condition: service_healthy`.

---

## Final Image Sizes

| Service | Image | Size |
|---|---|---|
| Flask Todo App | Custom built — python:3.11-slim | ~150MB |
| Prometheus | prom/prometheus:v3.10.0 | ~250MB |
| Node Exporter | prom/node-exporter:v1.9.1 | ~25MB |
| Grafana | grafana/grafana:v12.4.0 | ~400MB |

---

## Key Takeaways

- Dockerizing an app is only half the job — knowing it is healthy after deployment is the other half
- Monitoring is not optional in production — every real system has observability built in
- Virtual filesystems like `/proc` and `/sys` exist in memory only — the kernel generates them live so fast changing data like CPU usage never needs to touch disk
- `depends_on` with `condition: service_healthy` is the correct way to handle startup order — not just `depends_on` alone
- Named volumes are essential for any stateful service — databases and monitoring tools both need them
- Never hardcode credentials — always use `.env` files and reference them with `${}` syntax in compose
- Pinning image versions is not optional — `latest` is a liability in any real deployment