# Day 35 – Multi-Stage Builds & Docker Hub

## Overview

This document covers how to build optimized Docker images using multi-stage builds and how to push them to Docker Hub. We built a simple Java HTTP server and progressively reduced the image size from 390MB down to 50MB.

---

## The Java App

A simple HTTP server built using Java's built-in `com.sun.net.httpserver` — no external dependencies required. The file is named `App.java` because Java requires the filename to match the class name exactly. When running, visit `http://localhost:8080` to see: `Hello from Java in Docker!`

The app starts an HTTP server on port 8080, listens for any incoming request, and responds with plain text. It uses only two Java modules — `java.base` for core Java and `jdk.httpserver` for the built-in HTTP server.

---

## Task 1: Single Stage Dockerfile (The Problem)

**Dockerfile.single** uses `eclipse-temurin:21-jdk-alpine` as both the build and runtime image. It copies `App.java`, compiles it with `javac`, exposes port 8080, and runs the app with `java App`.

**Image size: ~390MB**

This is large because the full JDK is included — it contains `javac`, `jshell`, `jdeps`, and hundreds of tools you never need at runtime. The entire JDK stays in the final image even though it was only needed during compilation.

---

## Task 2: Multi-Stage Build

### Distroless Approach

The first improvement is switching to a multi-stage build with `gcr.io/distroless/java21-debian12` as the final base image. Stage 1 uses the full JDK to compile the app. Stage 2 copies only the `.class` file into the distroless image which contains just enough to run Java — no shell, no package manager, no OS utilities.

**Image size: ~192MB** — nearly 50% smaller than the single stage build.

### Why is the multi-stage image smaller?

In a single-stage build, the final image contains everything — the full JDK, build tools, source files, and compiled output. In a multi-stage build, Stage 1 (the builder) does all the compilation work but is then completely discarded. Only the compiled `.class` files are copied into Stage 2 which uses a minimal base image with no JDK tools. The result is a lean image that contains only what is needed to run the app.

---

## Task 3: Push to Docker Hub

**Step 1 — Create a free account** at `https://hub.docker.com` if you don't have one.

**Step 2 — Login from terminal** using `docker login` and enter your username and password.

**Step 3 — Tag your image** in the format `yourusername/image-name:tag`. For example: `docker tag java-distroless yourusername/java-hello:1.0`

**Step 4 — Push** using `docker push yourusername/java-hello:1.0`

**Step 5 — Verify** by removing the image locally with `docker rmi yourusername/java-hello:1.0` then pulling it back with `docker pull yourusername/java-hello:1.0`

---

## Task 4: Docker Hub Repository

After pushing, visit `https://hub.docker.com/r/yourusername/java-hello` to see your repository.

Click **Edit Repository** to add a description — this is the description we wrote for our image: a lightweight Java HTTP server using jlink and Alpine that responds with plain text on port 8080, reduced from 390MB to 50MB using multi-stage builds.

The **Tags** tab shows all versions you have pushed. Each tag is an independent image you can pull individually. Pulling `yourusername/java-hello:1.0` gives you exactly that version. Pulling `yourusername/java-hello:latest` gives you whatever image was tagged as `latest` — if no latest tag exists, the pull will fail. This is why you should always tag images intentionally rather than relying on `latest` in production.

---

## Task 5: Best Practices — Final Optimized Dockerfile

**Dockerfile.optimized** applies all best practices and brings the image down to ~50MB. Here is what changed and why:

**1. Minimal base image** — The final stage uses `alpine:3.19` (~5MB) instead of distroless. Alpine is a stripped down Linux distribution built specifically for containers. It is smaller than Ubuntu (~80MB) and gives us just enough OS to run our custom JRE.

**2. jlink custom JRE** — Instead of using a pre-built JRE, we use `jlink` to build a custom JRE containing only the two modules our app needs: `java.base` and `jdk.httpserver`. Everything else — SQL, XML, desktop, security, and hundreds of other modules — is excluded. To find which modules your app needs, run `jdeps --print-module-deps App.class` inside the build stage and it will print the exact module list to use.

**3. Non-root user** — A new group `appgroup` and user `appuser` are created before the app runs. The app files are owned by this user via `chown` and the container switches to this user via `USER appuser`. Running as root inside a container is a security risk — if an attacker exploits the app they would have root access to the host. A non-root user limits the blast radius significantly.

**4. Combined RUN commands** — The `javac` compile and `jlink` commands are combined into a single `RUN` instruction using `&&`. Each `RUN` instruction creates a new image layer. Combining them reduces the layer count and total image size.

**5. Specific base image tags** — We use `alpine:3.19` instead of `alpine:latest`. Pinning to a specific version ensures reproducible builds. The `latest` tag can change at any time and silently break your image in production.

**Image size: ~50MB** — 87% smaller than the original single stage build.

---

## Dockerfile.single — Contents Explained

The single stage Dockerfile starts with `eclipse-temurin:21-jdk-alpine` as the only image. It sets `/app` as the working directory, copies `App.java` into it, compiles it with `javac`, exposes port 8080 and starts the app. Simple — but the problem is the full JDK never leaves the image.

---

## Dockerfile.optimized — Contents Explained

**Stage 1 (build)** starts from `eclipse-temurin:21-jdk-alpine`. It compiles `App.java` and then runs `jlink` with `--add-modules java.base,jdk.httpserver` to build a custom JRE at `/javaruntime`. The flags `--strip-debug`, `--no-man-pages`, `--no-header-files`, and `--compress=2` strip everything unnecessary and compress the output to maximum level.

**Stage 2 (final)** starts fresh from `alpine:3.19`. It creates a non-root group and user, sets `/app` as the working directory, copies the custom JRE and the compiled `.class` file from Stage 1, assigns ownership to `appuser`, switches to that user, sets `JAVA_HOME` and `PATH` so the container knows where to find Java, exposes port 8080, and starts the app with `java App`.

---

## Size Comparison Summary

| Approach | Base Image | Size |
|---|---|---|
| Single stage | eclipse-temurin:21-jdk-alpine | ~390MB |
| Multi-stage distroless | gcr.io/distroless/java21 | ~192MB |
| Multi-stage jlink + alpine | alpine:3.19 | ~50MB |

**Total reduction: 390MB → 50MB (~87% smaller)**

---

## Key Commands Reference

| Action | Command |
|---|---|
| Build image | `docker build -t image-name .` |
| Build with custom Dockerfile | `docker build -f Dockerfile.custom -t image-name .` |
| Check image size | `docker images image-name` |
| Tag for Docker Hub | `docker tag image-name username/repo:tag` |
| Login to Docker Hub | `docker login` |
| Push image | `docker push username/repo:tag` |
| Pull image | `docker pull username/repo:tag` |
| Inspect layers | `docker history image-name` |
| Find Java modules needed | `jdeps --print-module-deps App.class` |
| List all Java modules | `java --list-modules` |

---

## Key Takeaways

- Multi-stage builds separate the build environment from the runtime environment — the builder stage is thrown away after compilation
- The final image only contains what is needed to run the app, nothing used to build it
- jlink creates a custom JRE with only the Java modules your app actually uses — use jdeps to find them
- Always run containers as a non-root user in production to limit security exposure
- Pin base image versions with specific tags, never use `latest` in production
- Combine `RUN` commands with `&&` to reduce image layers and size
- Docker Hub tags are how you version and distribute images — always tag intentionally
- Distroless images have no shell which makes them more secure but harder to debug