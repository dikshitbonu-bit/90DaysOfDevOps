# Day 37 – Docker Revision

## Reference

[Docker cheatsheet](https://github.com/dikshitbonu-bit/Devops-notes/blob/main/Docker/docker-cheatsheet.md)


## Self Assessment

- Run container (interactive + detached) — **Can Do**
- List/stop/remove containers & images — **Can Do**
- Explain image layers & caching — **Can Do**
- Write Dockerfile from scratch — **Can Do**
- CMD vs ENTRYPOINT — **Can Do**
- Build & tag image — **Can Do**
- Named volumes — **Can Do**
- Bind mounts — **Can Do**
- Custom networks — **Can Do**
- `docker-compose.yml` multi-container — **Can Do**
- `.env` in Compose — **Can Do**
- Multi-stage Dockerfile — **Shaky**
- Push image to Docker Hub — **Can Do**
- Healthchecks & `depends_on` — **Shaky**

---

## Quick‑Fire Answers

1. **Image vs Container**
   - Image: blueprint (read‑only template)
   - Container: running instance of that image

2. **What happens to data when a container is removed?**
   Data inside the container is deleted unless it resides in a volume or bind mount.

3. **How do two containers communicate on the same custom network?**
   They use container names as DNS hostnames.

4. **`docker compose down` vs `docker compose down -v`**
   - `down`: removes containers & networks
   - `down -v`: also removes volumes (data loss)

5. **Why multi‑stage builds?**
   To reduce the final image size by copying only necessary artifacts to the last stage.

6. **COPY vs ADD**
   - `COPY`: simple file copy (recommended)
   - `ADD`: file copy + auto‑extract + URL support (less predictable)

7. **What does `-p 8080:80` mean?**
   Maps host port 8080 to container port 80.

8. **How to check Docker disk usage?**
   `docker system df`

---

## Weak‑Spot Revisit Plan

1. Rebuild a multi‑stage `Dockerfile` from scratch.
2. Create a Compose file with a `healthcheck` and `depends_on`, then test a failure scenario.

---

## Key Realizations

- Containers are temporary.
- Volumes store state.
- Images are layered.
- Compose is orchestration for local environments.