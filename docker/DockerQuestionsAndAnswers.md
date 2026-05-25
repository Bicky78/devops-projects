# Docker Interview Questions and Answers

A focused collection of Docker interview questions for DevOps engineers — covering containers, images, Dockerfile, networking, volumes, Docker Compose, security, and real-world scenarios.

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Images & Dockerfile](#images--dockerfile)
- [Containers & Commands](#containers--commands)
- [Networking & Volumes](#networking--volumes)
- [Docker Compose](#docker-compose)
- [Security & Best Practices](#security--best-practices)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Core Concepts

### 1. What is Docker?

**Answer:** Docker is a containerization platform that packages applications with all their dependencies into lightweight, portable containers. Containers share the host OS kernel, making them faster and more efficient than virtual machines.

**Key components:**

| Component | Description |
|-----------|-------------|
| **Docker Engine** | Runtime that builds and runs containers |
| **Docker Daemon** | Background service managing containers (`dockerd`) |
| **Docker Client** | CLI that sends commands to the daemon (`docker`) |
| **Docker Images** | Read-only templates used to create containers |
| **Docker Containers** | Running instances of images |
| **Docker Registry** | Storage for images (Docker Hub, ECR, ACR) |

---

### 2. What is the difference between Docker and Virtual Machines?

**Answer:**

| Feature | Docker (Containers) | Virtual Machines |
|---------|-------------------|-----------------|
| **OS** | Shares host kernel | Full guest OS per VM |
| **Size** | MBs (lightweight) | GBs (heavy) |
| **Startup** | Seconds | Minutes |
| **Isolation** | Process-level (namespaces, cgroups) | Hardware-level (hypervisor) |
| **Performance** | Near-native | Overhead from hypervisor |
| **Portability** | Highly portable | Less portable |
| **Resource usage** | Efficient (shared kernel) | More resources needed |

**Use VMs when:** Full OS isolation needed, running different OS kernels.
**Use Docker when:** Lightweight, fast, consistent deployments across environments.

---

### 3. What is a Docker image vs a Docker container?

**Answer:**

| Feature | Image | Container |
|---------|-------|-----------|
| **State** | Read-only template | Running instance of an image |
| **Analogy** | Class (blueprint) | Object (instance) |
| **Layers** | Made of stacked layers | Adds a writable layer on top |
| **Storage** | Stored in registry/local | Exists in memory + writable layer |
| **Created by** | `docker build` | `docker run` |

```bash
# Build image
docker build -t myapp:1.0 .

# Create container from image
docker run -d --name myapp-container myapp:1.0

# List images
docker images

# List containers
docker ps -a
```

---

### 4. What is Docker Hub?

**Answer:** Docker Hub is a cloud-based registry for Docker images — the default public registry.

```bash
# Pull image from Docker Hub
docker pull nginx:latest

# Push image to Docker Hub
docker login
docker tag myapp:1.0 username/myapp:1.0
docker push username/myapp:1.0

# Search images
docker search nginx
```

**Alternatives:** Amazon ECR, Azure ACR, Google GCR, GitHub Container Registry, Harbor (self-hosted).

---

### 5. Explain Docker architecture.

**Answer:**

```
Docker Client (CLI)  ──REST API──►  Docker Daemon (dockerd)
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                  ▼
               Containers         Images            Networks/Volumes
                    │                 │
                    ▼                 ▼
              containerd          Registry
                    │           (Docker Hub)
                    ▼
                  runc
            (creates containers)
```

- **Docker Client:** Sends commands via REST API
- **Docker Daemon:** Manages images, containers, networks, volumes
- **containerd:** Container runtime that manages container lifecycle
- **runc:** Low-level runtime that creates and runs containers using Linux kernel features (namespaces, cgroups)

---

### 6. What are Docker namespaces and cgroups?

**Answer:** These are Linux kernel features Docker uses for container isolation.

**Namespaces (isolation):**

| Namespace | What it isolates |
|-----------|-----------------|
| **PID** | Process IDs — each container sees its own PID 1 |
| **NET** | Network interfaces, routing tables, ports |
| **MNT** | Filesystem mount points |
| **UTS** | Hostname and domain name |
| **IPC** | Inter-process communication |
| **USER** | User and group IDs |

**Cgroups (resource limits):**
- Limit CPU, memory, disk I/O, network bandwidth per container
- Prevent one container from consuming all host resources

```bash
# Limit container resources
docker run -d --name myapp \
  --cpus="1.5" \
  --memory="512m" \
  --memory-swap="1g" \
  myapp:1.0
```

---

## Images & Dockerfile

### 7. What is a Dockerfile? Explain key instructions.

**Answer:** A Dockerfile is a text file with instructions to build a Docker image.

```dockerfile
# Base image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy dependency files first (cache optimization)
COPY package*.json ./

# Install dependencies
RUN npm ci --production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Set environment variable
ENV NODE_ENV=production

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1

# Default command
CMD ["node", "server.js"]
```

**Key instructions:**

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from host to image |
| `ADD` | Like COPY but supports URLs and auto-extracts archives |
| `RUN` | Execute commands during build |
| `CMD` | Default command when container starts (overridable) |
| `ENTRYPOINT` | Fixed command when container starts (not easily overridden) |
| `EXPOSE` | Document which port the app uses |
| `ENV` | Set environment variables |
| `ARG` | Build-time variables |
| `VOLUME` | Declare mount points |
| `HEALTHCHECK` | Define health check for container |
| `USER` | Set non-root user |
| `LABEL` | Add metadata |

---

### 8. What is the difference between CMD and ENTRYPOINT?

**Answer:**

| Feature | CMD | ENTRYPOINT |
|---------|-----|------------|
| **Purpose** | Default command/args | Fixed executable |
| **Override** | Easily overridden at `docker run` | Requires `--entrypoint` flag |
| **Use case** | Provide defaults that can change | Container should always run same command |

```dockerfile
# CMD — can be overridden
FROM nginx
CMD ["nginx", "-g", "daemon off;"]
# docker run myimage        → runs nginx
# docker run myimage ls     → runs ls (overrides CMD)

# ENTRYPOINT — fixed command
FROM python:3.11
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myimage            → runs python app.py
# docker run myimage test.py    → runs python test.py (CMD overridden, not ENTRYPOINT)
```

**Best practice:** Use `ENTRYPOINT` for the executable, `CMD` for default arguments.

---

### 9. What is the difference between COPY and ADD?

**Answer:**

| Feature | COPY | ADD |
|---------|------|-----|
| **Local files** | Yes | Yes |
| **URLs** | No | Yes (downloads) |
| **Auto-extract** | No | Yes (`.tar`, `.gz`) |
| **Recommended** | Yes (explicit) | Only when auto-extract needed |

```dockerfile
# Prefer COPY (explicit, predictable)
COPY app.py /app/

# Use ADD only for auto-extraction
ADD archive.tar.gz /app/
```

---

### 10. What is a multi-stage build and why use it?

**Answer:** Multi-stage builds use multiple `FROM` statements to create smaller, production-ready images by discarding build tools and intermediate files.

```dockerfile
# Stage 1: Build
FROM maven:3.8-jdk-11 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:resolve
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime (much smaller)
FROM eclipse-temurin:11-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/myapp.jar .
EXPOSE 8080
CMD ["java", "-jar", "myapp.jar"]
```

**Benefits:**
- **Smaller images:** Final image has only runtime, no build tools
- **Security:** Fewer packages = smaller attack surface
- **Cache-friendly:** Build dependencies cached separately

**Size comparison:**
- Without multi-stage: ~800MB (includes Maven, JDK, source code)
- With multi-stage: ~150MB (only JRE + JAR)

---

### 11. What are Docker image layers?

**Answer:** Each instruction in a Dockerfile creates a new read-only layer. Layers are cached and shared between images.

```dockerfile
FROM ubuntu:22.04          # Layer 1 (base)
RUN apt-get update         # Layer 2
RUN apt-get install -y curl # Layer 3
COPY app.py /app/          # Layer 4
CMD ["python", "/app/app.py"] # Layer 5 (metadata, no layer)
```

```bash
# View image layers
docker history myimage

# Inspect image
docker inspect myimage
```

**Optimization tips:**
- Combine `RUN` commands to reduce layers
- Order instructions from least to most frequently changed
- Copy dependency files before source code (better caching)

---

## Containers & Commands

### 12. What are the essential Docker commands?

**Answer:**

| Command | Purpose |
|---------|---------|
| `docker build -t name:tag .` | Build image from Dockerfile |
| `docker run -d --name c1 image` | Run container in background |
| `docker ps` / `docker ps -a` | List running / all containers |
| `docker stop c1` | Gracefully stop container |
| `docker start c1` | Start stopped container |
| `docker restart c1` | Restart container |
| `docker rm c1` | Remove container |
| `docker rmi image` | Remove image |
| `docker exec -it c1 /bin/sh` | Execute command in running container |
| `docker logs c1` / `docker logs -f c1` | View / follow container logs |
| `docker inspect c1` | Detailed container info (JSON) |
| `docker cp file c1:/path` | Copy file to/from container |
| `docker stats` | Live resource usage |
| `docker system prune -a` | Remove all unused data |
| `docker tag img:v1 repo/img:v1` | Tag image for pushing |
| `docker push` / `docker pull` | Push/pull images from registry |

---

### 13. Describe the lifecycle of a Docker container.

**Answer:**

```
docker create → docker start → (running) → docker pause → docker unpause
                                    │
                              docker stop → (stopped) → docker start
                                    │                       │
                              docker kill              docker restart
                                    │
                              docker rm (removed)
```

**States:**

| State | Description |
|-------|-------------|
| **Created** | Container exists but not started |
| **Running** | Actively executing |
| **Paused** | Processes suspended (frozen in memory) |
| **Stopped** | Processes terminated gracefully |
| **Killed** | Processes terminated forcefully |
| **Removed** | Container deleted |

---

### 14. What are Docker restart policies?

**Answer:**

| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `on-failure[:max]` | Restart only on non-zero exit code (optional retry limit) |
| `always` | Always restart, including on daemon startup |
| `unless-stopped` | Like `always`, but not if manually stopped before daemon restart |

```bash
# Always restart
docker run -d --restart always nginx

# Restart on failure, max 5 retries
docker run -d --restart on-failure:5 myapp

# Unless manually stopped
docker run -d --restart unless-stopped myapp
```

---

### 15. How do you limit CPU and memory for a container?

**Answer:**

```bash
# Limit memory
docker run -d --memory="512m" --memory-swap="1g" myapp

# Limit CPU
docker run -d --cpus="1.5" myapp          # 1.5 CPU cores
docker run -d --cpu-shares=512 myapp       # Relative weight

# Both
docker run -d \
  --memory="256m" \
  --cpus="0.5" \
  --name limited-app \
  myapp:1.0

# Check resource usage
docker stats
```

---

## Networking & Volumes

### 16. Explain Docker networking types.

**Answer:**

| Network Driver | Description | Use Case |
|---------------|-------------|----------|
| **bridge** | Default. Containers on same host communicate | Single-host apps |
| **host** | Container shares host network stack | Performance-critical apps |
| **none** | No networking | Isolated batch jobs |
| **overlay** | Multi-host networking (Swarm/K8s) | Distributed apps |
| **macvlan** | Assigns MAC address, appears as physical device | Legacy apps needing direct network |

```bash
# Create custom bridge network
docker network create mynet

# Run containers on same network
docker run -d --name app --network mynet myapp
docker run -d --name db --network mynet postgres

# Containers can reach each other by name
# app can connect to db using hostname "db"

# Inspect network
docker network inspect mynet

# List networks
docker network ls
```

---

### 17. What are Docker volumes and why use them?

**Answer:** Volumes provide persistent storage that survives container restarts and removal.

**Three types of storage:**

| Type | Description | Managed by |
|------|-------------|-----------|
| **Volume** | Stored in `/var/lib/docker/volumes/` | Docker |
| **Bind mount** | Maps host directory to container | User |
| **tmpfs** | Stored in memory only | Kernel |

```bash
# Named volume (recommended)
docker volume create mydata
docker run -d -v mydata:/app/data myapp

# Bind mount (host directory)
docker run -d -v /host/path:/container/path myapp

# Read-only bind mount
docker run -d -v /host/config:/app/config:ro myapp

# tmpfs (in-memory, lost on restart)
docker run -d --tmpfs /app/temp myapp

# List volumes
docker volume ls

# Remove unused volumes
docker volume prune
```

**When you lose data:**
- Container deleted without volumes
- Using ephemeral storage (no volume/bind mount)
- `docker system prune` with `--volumes` flag

---

### 18. How do you share data between containers?

**Answer:**

```bash
# Method 1: Shared named volume
docker volume create shared-data
docker run -d --name writer -v shared-data:/data myapp-writer
docker run -d --name reader -v shared-data:/data myapp-reader

# Method 2: --volumes-from (inherit volumes from another container)
docker run -d --name data-container -v /data busybox
docker run -d --volumes-from data-container myapp

# Method 3: Shared bind mount
docker run -d --name c1 -v /host/shared:/data app1
docker run -d --name c2 -v /host/shared:/data app2
```

---

## Docker Compose

### 19. What is Docker Compose?

**Answer:** Docker Compose is a tool for defining and running multi-container applications using a YAML file.

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-net
    restart: unless-stopped

  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-net

  redis:
    image: redis:7-alpine
    networks:
      - app-net

volumes:
  db-data:

networks:
  app-net:
```

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f app

# Scale a service
docker compose up -d --scale app=3

# Rebuild and restart
docker compose up -d --build
```

---

### 20. How does `depends_on` work in Docker Compose?

**Answer:** `depends_on` controls startup **order**, but does NOT wait for a service to be "ready."

```yaml
services:
  app:
    depends_on:
      - db        # Simple: starts db before app
      
  # Better: wait for health check
  app:
    depends_on:
      db:
        condition: service_healthy

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5
```

**Without `condition: service_healthy`**, the app container starts as soon as the db container is *created*, not when the database is actually accepting connections.

---

## Security & Best Practices

### 21. What are Docker security best practices?

**Answer:**

```dockerfile
# 1. Use minimal base images
FROM node:18-alpine    # Not node:18 (full Debian)

# 2. Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 3. Don't store secrets in images
# BAD:  ENV API_KEY=mysecret
# GOOD: Pass at runtime via --env-file or secrets

# 4. Use .dockerignore
# .dockerignore
node_modules
.git
.env
*.md

# 5. Pin image versions
FROM node:18.17.1-alpine3.18    # Not node:latest

# 6. Scan images for vulnerabilities
# docker scout cves myimage:1.0
```

**Additional practices:**
- Use read-only filesystem: `docker run --read-only`
- Drop unnecessary capabilities: `--cap-drop ALL --cap-add NET_BIND_SERVICE`
- Use Docker Content Trust: `export DOCKER_CONTENT_TRUST=1`
- Limit resources (CPU/memory)
- Keep Docker and host OS updated
- Use multi-stage builds to reduce attack surface

---

### 22. What are Docker secrets and how do you manage sensitive data?

**Answer:**

**Docker Swarm Secrets:**
```bash
# Create secret
echo "mypassword" | docker secret create db_password -

# Use in service
docker service create \
  --name myapp \
  --secret db_password \
  myapp:1.0

# Secret available at /run/secrets/db_password inside container
```

**Docker Compose Secrets:**
```yaml
services:
  app:
    secrets:
      - db_password
    environment:
      DB_PASS_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Other approaches:**
- Environment variables via `--env-file` (not in image)
- HashiCorp Vault integration
- AWS Secrets Manager / Azure Key Vault
- Never bake secrets into images or Dockerfiles

---

### 23. How do you optimize Docker image size?

**Answer:**

```dockerfile
# 1. Use multi-stage builds (see Q10)

# 2. Use alpine/slim base images
FROM python:3.11-slim    # ~150MB vs ~900MB for python:3.11

# 3. Combine RUN commands
# BAD (3 layers):
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD (1 layer):
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# 4. Use .dockerignore
node_modules
.git
*.md
tests/

# 5. Order layers for cache efficiency
COPY package*.json ./     # Changes rarely
RUN npm ci --production   # Cached if package.json unchanged
COPY . .                  # Changes often (last)
```

---

### 24. What is Docker system prune?

**Answer:** Removes unused Docker data to free disk space.

```bash
# Remove stopped containers, unused networks, dangling images
docker system prune

# Also remove unused images (not just dangling)
docker system prune -a

# Also remove volumes
docker system prune -a --volumes

# Individual cleanup
docker container prune    # Stopped containers
docker image prune -a     # Unused images
docker volume prune       # Unused volumes
docker network prune      # Unused networks

# Check disk usage
docker system df
```

---

## Scenario-Based Questions

### 25. How do you debug a failing Docker container?

**Answer:**

```bash
# 1. Check container status
docker ps -a

# 2. View logs
docker logs mycontainer
docker logs --tail 50 -f mycontainer

# 3. Inspect container config
docker inspect mycontainer

# 4. Check events
docker events --since 30m

# 5. Exec into running container
docker exec -it mycontainer /bin/sh

# 6. If container exits immediately, run interactively
docker run -it --entrypoint /bin/sh myimage

# 7. Check resource usage
docker stats mycontainer

# 8. Check health status
docker inspect --format='{{.State.Health.Status}}' mycontainer

# 9. Check container processes
docker top mycontainer
```

---

### 26. How do you monitor Docker in production?

**Answer:**

```bash
# Built-in monitoring
docker stats                          # Live resource usage
docker events                         # Real-time Docker events
docker logs -f --timestamps container # Container logs
```

**Production monitoring stack:**

| Tool | Purpose |
|------|---------|
| **Prometheus + cAdvisor** | Container metrics collection |
| **Grafana** | Visualization and dashboards |
| **ELK Stack / Loki** | Centralized log aggregation |
| **Docker Scout** | Image vulnerability scanning |
| **Datadog / New Relic** | Commercial APM solutions |

---

### 27. How do you implement CI/CD with Docker?

**Answer:**

```yaml
# Example: GitHub Actions
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Run tests
        run: docker run --rm myapp:${{ github.sha }} npm test
      
      - name: Push to registry
        run: |
          docker tag myapp:${{ github.sha }} registry.example.com/myapp:${{ github.sha }}
          docker push registry.example.com/myapp:${{ github.sha }}
      
      - name: Deploy
        run: |
          helm upgrade --install myapp ./chart \
            --set image.tag=${{ github.sha }}
```

**Best practices:**
- Tag images with Git SHA (not `latest`)
- Scan images in CI pipeline
- Use multi-stage builds for smaller images
- Cache Docker layers in CI
- Use `--no-cache` for release builds

---

### 28. A container keeps restarting. How do you troubleshoot?

**Answer:**

```bash
# 1. Check restart count and last exit code
docker inspect mycontainer --format='{{.RestartCount}} - {{.State.ExitCode}}'

# 2. View logs from last run
docker logs mycontainer --tail 100

# 3. Common exit codes:
# 0   — Normal exit
# 1   — Application error
# 137 — OOM killed (out of memory) or SIGKILL
# 139 — Segmentation fault
# 143 — SIGTERM (graceful stop)

# 4. Check if OOM killed
docker inspect mycontainer | grep -i oom
dmesg | grep -i "oom\|killed"

# 5. Check resource limits
docker stats mycontainer

# 6. Override entrypoint to debug
docker run -it --entrypoint /bin/sh myimage

# 7. Fix: increase memory, fix app error, fix health check
docker run -d --memory="1g" --restart on-failure:3 myapp
```

---

### 29. How do you update a running application without downtime?

**Answer:**

```bash
# Method 1: Blue-green with Docker Compose
# 1. Build new image
docker build -t myapp:v2 .

# 2. Start new container
docker run -d --name myapp-new -p 3001:3000 myapp:v2

# 3. Test new container
curl http://localhost:3001/health

# 4. Switch traffic (update load balancer/proxy)
# 5. Stop old container
docker stop myapp-old && docker rm myapp-old

# Method 2: Docker Compose rolling update
docker compose up -d --no-deps --build app

# Method 3: Docker Swarm rolling update
docker service update \
  --image myapp:v2 \
  --update-parallelism 1 \
  --update-delay 30s \
  myapp-service
```

---

### 30. How do you handle logging in Docker?

**Answer:**

**Logging drivers:**

| Driver | Description |
|--------|-------------|
| `json-file` | Default, logs to JSON files on host |
| `syslog` | Send to syslog |
| `fluentd` | Send to Fluentd collector |
| `awslogs` | Send to AWS CloudWatch |
| `gcplogs` | Send to Google Cloud Logging |
| `splunk` | Send to Splunk |

```bash
# View logs
docker logs mycontainer
docker logs -f --tail 100 mycontainer    # Follow last 100 lines
docker logs --since 2h mycontainer       # Last 2 hours

# Set logging driver
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Set default in daemon.json
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Best practice:** Always set `max-size` and `max-file` to prevent disk fill.

---

### 31. What is the difference between `ENV` and `ARG` in a Dockerfile?

**Answer:**

| Feature | `ARG` | `ENV` |
|---------|-------|-------|
| **Available during** | Build only | Build + runtime |
| **Override at** | `docker build --build-arg` | `docker run -e` or `--env-file` |
| **Persisted in image** | No | Yes |
| **Visible in running container** | No | Yes (`printenv`) |

```dockerfile
# ARG — build-time only (e.g., selecting base image version)
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

# ENV — available at runtime
ENV APP_PORT=3000
EXPOSE ${APP_PORT}

# ARG value passed to ENV
ARG BUILD_VERSION
ENV APP_VERSION=${BUILD_VERSION}
```

```bash
# Override at build time
docker build --build-arg NODE_VERSION=20 --build-arg BUILD_VERSION=1.5.0 -t myapp .

# Override ENV at runtime
docker run -e APP_PORT=8080 myapp
```

---

### 32. What is `.dockerignore` and why is it important?

**Answer:** `.dockerignore` excludes files from the Docker build context, reducing build time and image size.

```
# .dockerignore
.git
.gitignore
node_modules
npm-debug.log
Dockerfile
docker-compose*.yml
.env
*.md
tests/
coverage/
.vscode/
.idea/
```

**Why it matters:**
- **Faster builds:** Smaller build context sent to daemon
- **Smaller images:** Prevents unnecessary files from being `COPY`ed
- **Security:** Keeps `.env`, secrets, `.git` out of images
- **Cache efficiency:** Irrelevant file changes don't bust the cache

---

### 33. What is the difference between `docker export/import` and `docker save/load`?

**Answer:**

| Command | Works on | Preserves layers | Use case |
|---------|----------|-----------------|----------|
| `docker save` | **Image** | Yes | Transfer images between hosts |
| `docker load` | **Image** | Yes | Import saved image archive |
| `docker export` | **Container** | No (flat filesystem) | Backup container filesystem |
| `docker import` | **Container** | No (single layer) | Create image from exported container |

```bash
# Save/Load images (preserves layers and tags)
docker save -o myapp.tar myapp:1.0
docker load -i myapp.tar

# Export/Import containers (flat filesystem)
docker export mycontainer > container-backup.tar
docker import container-backup.tar myapp-restored:1.0
```

---

### 34. What is Docker Swarm?

**Answer:** Docker Swarm is Docker's native container orchestration tool for clustering multiple Docker hosts and deploying services across them.

```bash
# Initialize a swarm (manager node)
docker swarm init --advertise-addr 192.168.1.10

# Join worker node
docker swarm join --token <token> 192.168.1.10:2377

# Deploy a service
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx

# Scale service
docker service scale web=5

# List services and nodes
docker service ls
docker node ls

# Rolling update
docker service update --image nginx:1.26 web
```

| Feature | Docker Swarm | Kubernetes |
|---------|-------------|-----------|
| **Complexity** | Simple, built into Docker | Complex, steeper learning curve |
| **Setup** | Minutes | Hours (or use managed K8s) |
| **Scaling** | Good for small/medium | Production-grade, auto-scaling |
| **Ecosystem** | Limited | Vast (Helm, Istio, ArgoCD, etc.) |
| **Use case** | Small teams, simple workloads | Large-scale, enterprise production |

---

### 35. What is the difference between `docker stop` and `docker kill`?

**Answer:**

| Command | Signal | Behavior |
|---------|--------|----------|
| `docker stop` | SIGTERM → (grace period) → SIGKILL | Graceful shutdown; app can clean up |
| `docker kill` | SIGKILL (immediate) | Forceful termination; no cleanup |

```bash
# Graceful stop (default 10s grace period)
docker stop mycontainer

# Graceful stop with custom timeout
docker stop -t 30 mycontainer

# Force kill immediately
docker kill mycontainer

# Send specific signal
docker kill --signal=SIGHUP mycontainer
```

**Best practice:** Always prefer `docker stop`. Use `docker kill` only when a container is unresponsive.
