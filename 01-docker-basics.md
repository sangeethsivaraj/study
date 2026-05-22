# Docker - From Scratch to Advanced (Complete Guide)

---

## Chapter 1: Understanding the Problem Docker Solves

### The Problem Before Docker

Imagine you build a Node.js application on your laptop. It works perfectly. You send it to your friend, and it breaks. Why?

- Your laptop has Node.js v18, your friend has v16
- You have certain libraries installed globally, your friend doesn't
- Your OS is Windows, the server is Linux
- The database version differs

This is the classic **"It works on my machine"** problem.

### How Docker Solves It

Docker packages your application along with EVERYTHING it needs:
- The operating system (a lightweight Linux)
- The runtime (Node.js, Python, Java, etc.)
- The libraries and dependencies
- Your application code
- Configuration files

All of this goes into a single unit called a **Container**.

**Analogy:** Think of a shipping container. Before standardized shipping containers, loading cargo onto ships was chaotic — different shapes, sizes, handling requirements. Shipping containers standardized this. Docker does the same for software.

---

## Chapter 2: Containers vs Virtual Machines

```
┌────────────────────────────────────────────────────────────┐
│              Virtual Machines                                │
│                                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │  App A   │  │  App B   │  │  App C   │                │
│  ├──────────┤  ├──────────┤  ├──────────┤                │
│  │ Bins/Libs│  │ Bins/Libs│  │ Bins/Libs│                │
│  ├──────────┤  ├──────────┤  ├──────────┤                │
│  │ Guest OS │  │ Guest OS │  │ Guest OS │  ← Full OS each│
│  └──────────┘  └──────────┘  └──────────┘                │
│  ┌────────────────────────────────────────┐               │
│  │           Hypervisor (VMware, etc)      │               │
│  └────────────────────────────────────────┘               │
│  ┌────────────────────────────────────────┐               │
│  │              Host OS                    │               │
│  └────────────────────────────────────────┘               │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│              Docker Containers                              │
│                                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │  App A   │  │  App B   │  │  App C   │                │
│  ├──────────┤  ├──────────┤  ├──────────┤                │
│  │ Bins/Libs│  │ Bins/Libs│  │ Bins/Libs│  ← No Guest OS │
│  └──────────┘  └──────────┘  └──────────┘                │
│  ┌────────────────────────────────────────┐               │
│  │           Docker Engine                 │               │
│  └────────────────────────────────────────┘               │
│  ┌────────────────────────────────────────┐               │
│  │              Host OS                    │               │
│  └────────────────────────────────────────┘               │
└────────────────────────────────────────────────────────────┘
```

| Feature | Virtual Machine | Docker Container |
|---------|----------------|-----------------|
| Size | GBs (includes full OS) | MBs (shares host kernel) |
| Startup time | Minutes | Seconds |
| Resource usage | Heavy | Lightweight |
| Isolation | Complete (separate kernel) | Process-level (shared kernel) |
| Portability | Limited | Very high |
| Running instances | ~10-20 per host | ~100s per host |

**Key insight:** Containers share the host's OS kernel. They don't need their own full operating system. This makes them tiny and fast.

---

## Chapter 3: Docker Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Docker Client                       │
│              (docker CLI commands)                    │
│    docker build, docker run, docker pull, etc.       │
└──────────────────────┬──────────────────────────────┘
                       │ REST API
┌──────────────────────▼──────────────────────────────┐
│                   Docker Daemon                       │
│                   (dockerd)                           │
│                                                      │
│  ┌─────────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │   Images    │ │Containers│ │    Networks       │ │
│  └─────────────┘ └──────────┘ └──────────────────┘ │
│  ┌─────────────┐ ┌──────────────────────────────┐  │
│  │   Volumes   │ │    Container Runtime          │  │
│  └─────────────┘ └──────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                Docker Registry                        │
│         (Docker Hub, AWS ECR, etc.)                   │
└─────────────────────────────────────────────────────┘
```

**Components:**
1. **Docker Client** — The command-line tool you type commands into
2. **Docker Daemon (dockerd)** — The background service that manages everything
3. **Docker Registry** — Storage for Docker images (Docker Hub is the default public registry)

---

## Chapter 4: Core Concepts Explained

### 4.1 Docker Image

An image is a **read-only blueprint** for creating containers. Think of it as a recipe — you can create many dishes (containers) from one recipe (image).

Images are built in **layers**. Each instruction in a Dockerfile creates a new layer:

```
┌─────────────────────────────┐
│  Layer 5: CMD node app.js   │  ← Your start command
├─────────────────────────────┤
│  Layer 4: COPY . /app       │  ← Your application code
├─────────────────────────────┤
│  Layer 3: RUN npm install   │  ← Your dependencies
├─────────────────────────────┤
│  Layer 2: WORKDIR /app      │  ← Working directory set
├─────────────────────────────┤
│  Layer 1: FROM node:18      │  ← Base OS + Node.js
└─────────────────────────────┘
```

**Why layers?** Caching. If you change your app code (Layer 4), Docker only rebuilds Layer 4 and above. Layers 1-3 are cached, making builds fast.

### 4.2 Docker Container

A container is a **running instance** of an image. It's a lightweight, isolated process on your machine.

- An image can create many containers (like a class creates objects in OOP)
- Containers are isolated from each other and from the host
- Containers are ephemeral — when deleted, their internal changes are lost (unless you use volumes)

### 4.3 Dockerfile

A text file with instructions on how to build an image. Each line is a step.

### 4.4 Docker Registry

A storage system for Docker images. Like GitHub is for code, Docker Hub is for images.

- **Public**: Docker Hub (hub.docker.com)
- **Private**: AWS ECR, Google GCR, Azure ACR, self-hosted

---

## Chapter 5: Installation

### Windows
1. Download Docker Desktop from https://www.docker.com/products/docker-desktop
2. Run the installer
3. Restart your computer
4. Open PowerShell and verify:
```powershell
docker --version
docker run hello-world
```

### Linux (Ubuntu)
```bash
# Remove old versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install prerequisites
sudo apt update
sudo apt install ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add the repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Run without sudo
sudo usermod -aG docker $USER
# Log out and back in

# Verify
docker --version
docker run hello-world
```

### What "docker run hello-world" Does

```
1. Docker client contacts Docker daemon
2. Daemon looks for "hello-world" image locally → not found
3. Daemon pulls "hello-world" image from Docker Hub
4. Daemon creates a container from that image
5. Container runs, prints a message, and exits
```

---

## Chapter 6: Working with Images

### 6.1 Pulling Images

```bash
# Pull the latest nginx image
docker pull nginx
# This actually means: docker pull docker.io/library/nginx:latest

# Pull a specific version
docker pull nginx:1.25

# Pull from a different registry
docker pull mycompany.azurecr.io/myapp:v2
```

**Image naming format:** `registry/repository:tag`
- `nginx` = `docker.io/library/nginx:latest`
- `myuser/myapp:v2` = `docker.io/myuser/myapp:v2`

### 6.2 Listing and Managing Images

```bash
# List all local images
docker images

# Output:
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        latest    a8758716bb6a   2 weeks ago    187MB
# node         18        b5912e3b3e3c   3 weeks ago    992MB

# Remove an image
docker rmi nginx

# Remove all unused images
docker image prune -a

# See image history (layers)
docker history nginx

# Inspect image details (full JSON)
docker inspect nginx
```

### 6.3 Understanding Tags

Tags are labels for different versions of an image.

```bash
docker pull node:18          # Node.js version 18
docker pull node:18-alpine   # Node.js 18 on Alpine Linux (small!)
docker pull node:18-slim     # Node.js 18 on Debian slim
docker pull node:latest      # Whatever the latest is (AVOID in production)
```

**Image variants:**
| Tag suffix | Meaning | Size |
|-----------|---------|------|
| (none) | Full Debian-based | Large (~900MB for Node) |
| `-slim` | Minimal Debian | Medium (~200MB) |
| `-alpine` | Alpine Linux based | Small (~50-150MB) |
| `-bullseye` | Specific Debian version | Large |

**Rule:** Always use specific tags in production. `node:18-alpine` not `node:latest`.

---

## Chapter 7: Running Containers

### 7.1 Basic Run Commands

```bash
# Simplest run (runs in foreground, shows output)
docker run nginx

# Problem: terminal is blocked! Press Ctrl+C to stop.

# Run in detached mode (background)
docker run -d nginx

# Run with a name (easier to reference later)
docker run -d --name my-web nginx

# Run and auto-remove when stopped
docker run -d --rm --name temp-web nginx
```

### 7.2 Port Mapping

Containers are isolated. To access a web server inside a container, you must map ports.

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 --name web nginx

# Now visit http://localhost:8080 in your browser!

# Map multiple ports
docker run -d -p 8080:80 -p 8443:443 nginx

# Map to a random available host port
docker run -d -p 80 nginx
# Check which port: docker port <container_id>
```

**Format:** `-p HOST_PORT:CONTAINER_PORT`

Think of it as: "Anything coming to my machine on port 8080, send it to port 80 inside the container."

### 7.3 Environment Variables

```bash
# Pass environment variables
docker run -d \
  --name my-db \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:16

# Use an env file
docker run -d --env-file .env --name my-app myimage
```

### 7.4 Interactive Mode

```bash
# Run an interactive shell inside a NEW container
docker run -it ubuntu bash

# You're now INSIDE the container!
# root@abc123:/# ls
# root@abc123:/# exit    ← this stops the container

# Run a command inside a RUNNING container
docker exec -it my-web bash
# or for Alpine-based images:
docker exec -it my-web sh
```

**Flags explained:**
- `-i` = interactive (keep STDIN open)
- `-t` = allocate a TTY (terminal)
- `-it` = interactive terminal

### 7.5 Container Lifecycle

```
     docker create        docker start        docker stop
Image ──────────► Created ──────────► Running ──────────► Stopped
                                         │                    │
                                         │ docker kill        │ docker start
                                         ▼                    │
                                      Stopped ◄───────────────┘
                                         │
                                         │ docker rm
                                         ▼
                                      Deleted
```

```bash
# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# Stop a container (graceful - sends SIGTERM, then SIGKILL after 10s)
docker stop my-web

# Kill immediately (sends SIGKILL)
docker kill my-web

# Remove a stopped container
docker rm my-web

# Force remove a running container
docker rm -f my-web

# Remove ALL stopped containers
docker container prune
```

### 7.6 Viewing Container Information

```bash
# View logs
docker logs my-web
docker logs -f my-web              # Follow (like tail -f)
docker logs --tail 50 my-web       # Last 50 lines
docker logs --since 1h my-web      # Last 1 hour

# View resource usage (live)
docker stats

# Detailed container info
docker inspect my-web

# See which process is running
docker top my-web

# See port mappings
docker port my-web
```

---

## Chapter 8: Writing Dockerfiles (From Zero)

### 8.1 Your First Dockerfile

A Dockerfile is just a text file named `Dockerfile` (no extension) that tells Docker how to build your image.

**Simplest possible example:**

```dockerfile
FROM ubuntu:22.04
RUN echo "Hello" > /hello.txt
CMD ["cat", "/hello.txt"]
```

**What this does:**
1. Start from Ubuntu 22.04 image
2. Create a file with "Hello" in it
3. When a container runs, cat that file

**Build and run:**
```bash
docker build -t my-first-image .
docker run my-first-image
# Output: Hello
```

### 8.2 Dockerfile Instructions — Explained One by One

#### FROM — The base image

Every Dockerfile starts with FROM. It's the foundation layer.

```dockerfile
FROM node:18-alpine
```

**Think of it as:** "Start with a mini Linux that already has Node.js 18 installed."

You can also start from scratch (an empty image):
```dockerfile
FROM scratch
```

#### WORKDIR — Set the working directory

```dockerfile
WORKDIR /app
```

All subsequent commands run from this directory. If it doesn't exist, it's created.

**Without WORKDIR:** Files end up in `/` (root), which is messy.
**With WORKDIR:** Everything is organized in `/app`.

#### COPY — Copy files from your machine into the image

```dockerfile
COPY package.json /app/package.json
COPY . /app/
COPY . .    # If WORKDIR is /app, copies to /app
```

**First path:** source on your machine (relative to build context)
**Second path:** destination inside the image

#### RUN — Execute a command during BUILD time

```dockerfile
RUN npm install
RUN apt-get update && apt-get install -y curl
```

Each RUN creates a new layer. Combine related commands with `&&` to reduce layers:

```dockerfile
# Bad: 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good: 1 layer
RUN apt-get update && apt-get install -y curl && apt-get clean
```

#### CMD — Default command when the container STARTS

```dockerfile
CMD ["node", "server.js"]
```

- Only ONE CMD per Dockerfile (last one wins)
- Can be overridden at runtime: `docker run myimage python other.py`
- Use exec form (with brackets): `CMD ["executable", "arg1"]`

#### ENTRYPOINT — Fixed command (CMD becomes its arguments)

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

Now:
- `docker run myimage` → runs `python app.py`
- `docker run myimage test.py` → runs `python test.py`

**ENTRYPOINT vs CMD:**
- CMD: "This is the default, but you can override it entirely"
- ENTRYPOINT: "This always runs. CMD just provides default arguments"

#### EXPOSE — Document which port the app uses

```dockerfile
EXPOSE 3000
```

**Important:** This does NOT publish the port! It's documentation. You still need `-p 3000:3000` when running.

#### ENV — Set environment variables

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

Available both during build AND at runtime.

#### ARG — Build-time only variables

```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}
```

ARG values are not available at runtime (unlike ENV).

```bash
docker build --build-arg NODE_VERSION=20 -t myapp .
```

#### USER — Run as non-root

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**Why?** Security. If an attacker exploits your app, they can't do root-level damage.

#### HEALTHCHECK — Define how to check if the container is healthy

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

Docker periodically runs this command. If it fails, the container is marked "unhealthy."

---

### 8.3 Complete Dockerfile — Node.js App (Explained Line by Line)

```dockerfile
# Start from Node.js 18 on Alpine Linux (small, ~50MB)
FROM node:18-alpine

# Create a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Set the working directory
WORKDIR /app

# Copy ONLY package files first
# Why? If your code changes but packages don't, Docker uses cached node_modules
COPY package.json package-lock.json ./

# Install dependencies (production only, skip devDependencies)
RUN npm ci --production

# Now copy the rest of your source code
COPY . .

# Change ownership to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Document the port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start the application
CMD ["node", "server.js"]
```

### 8.4 The Build Context and .dockerignore

When you run `docker build .`, the `.` is the **build context** — Docker sends this entire directory to the daemon.

**Problem:** If your folder has `node_modules` (500MB), Docker uploads all of it, even though the Dockerfile runs `npm install`.

**Solution:** `.dockerignore`

```
node_modules
npm-debug.log
.git
.gitignore
.env
.env.local
Dockerfile
docker-compose.yml
.DS_Store
coverage
dist
*.md
```

---

## Chapter 9: Layer Caching — Why Order Matters

```dockerfile
# ❌ BAD: Any code change invalidates npm install cache
COPY . .
RUN npm install

# ✅ GOOD: npm install is cached unless package.json changes
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

**How caching works:**
1. Docker checks each instruction from top to bottom
2. If the instruction AND its inputs haven't changed, use the cached layer
3. Once ONE layer is invalidated, ALL layers below it are rebuilt

**Rule:** Put things that change RARELY at the top, things that change OFTEN at the bottom.

```
Rarely changes:  FROM, system packages install
Sometimes:       package.json (dependencies)
Often:           Your source code (COPY . .)
```

---

## Chapter 10: Docker Volumes — Persistent Data

### Why Volumes?

Containers are **ephemeral**. When you remove a container, everything inside is gone. But databases need to persist data!

### Three Types of Mounts

```
┌─────────────────────────────────────────────────────────┐
│                      Host Machine                         │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Named Volume │  │ Bind Mount   │  │    tmpfs      │  │
│  │              │  │              │  │   (RAM only)  │  │
│  │ Docker       │  │ Host path    │  │   Not on disk │  │
│  │ manages it   │  │ you specify  │  │               │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                  │                  │          │
│         ▼                  ▼                  ▼          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Container File System               │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

| Type | Use Case | Example |
|------|----------|---------|
| Named Volume | Database storage, persistent data | `db-data:/var/lib/postgresql/data` |
| Bind Mount | Development (hot-reload source code) | `./src:/app/src` |
| tmpfs | Sensitive data that shouldn't hit disk | Secrets, temp files |

### Named Volumes

```bash
# Create a volume
docker volume create my-data

# Use it with a container
docker run -d \
  --name postgres \
  -v my-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# Even if you remove the container, data persists in the volume:
docker rm -f postgres

# Start a new container with the same volume — data is still there!
docker run -d \
  --name postgres-new \
  -v my-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect my-data

# Remove a volume (WARNING: deletes data!)
docker volume rm my-data

# Remove all unused volumes
docker volume prune
```

### Bind Mounts (Development)

```bash
# Mount your local source code into the container
docker run -d \
  --name dev-app \
  -v C:\Users\myuser\project\src:/app/src \
  -p 3000:3000 \
  my-node-app

# Now when you edit files on your machine, the container sees changes instantly!
```

**Use case:** During development, you want to edit code in your IDE and see changes immediately without rebuilding the image.

---

## Chapter 11: Docker Networking

### Default Behavior

By default, containers are isolated. They can't talk to each other unless you connect them.

### Network Types

```bash
# List networks
docker network ls

# Default networks:
# bridge    — default for standalone containers
# host      — container shares host's network
# none      — no networking
```

### Creating Custom Networks

```bash
# Create a network
docker network create my-app-net

# Run containers on the same network
docker run -d --name api --network my-app-net my-api
docker run -d --name db --network my-app-net postgres:16

# Now "api" container can reach "db" by name!
# Inside api container: connect to "db:5432"
```

**Key insight:** On a custom network, containers can find each other by name (DNS resolution). This is how a web app connects to its database.

### Practical Example

```bash
# Create network
docker network create webapp-net

# Start database
docker run -d \
  --name database \
  --network webapp-net \
  -e POSTGRES_PASSWORD=secret \
  -v db-data:/var/lib/postgresql/data \
  postgres:16

# Start application (connects to "database:5432")
docker run -d \
  --name webapp \
  --network webapp-net \
  -e DATABASE_URL=postgresql://postgres:secret@database:5432/myapp \
  -p 3000:3000 \
  my-webapp
```

---

## Chapter 12: Multi-Stage Builds

### The Problem

A typical Node.js build image might be 1GB. But to run the app, you only need the built output (~50MB).

### The Solution

Use one stage to BUILD, another to RUN:

```dockerfile
# ============ STAGE 1: Build ============
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install           # Install ALL deps (including devDependencies)
COPY . .
RUN npm run build         # Compile TypeScript, bundle, etc.

# ============ STAGE 2: Production ============
FROM node:18-alpine AS production
WORKDIR /app

# Copy ONLY what we need from the builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Result:**
- Build stage: ~1GB (has all build tools)
- Final image: ~150MB (only runtime + built code)

### Real Example: Go Application

Go can compile to a single binary, so the final image can be almost empty:

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# Production stage — FROM SCRATCH (no OS at all!)
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

Final image size: ~10MB (just the binary).

---

## Chapter 13: Docker Best Practices Summary

1. **Use specific image tags** — `node:18.19-alpine3.19` over `node:latest`
2. **Use .dockerignore** — don't send unnecessary files to build context
3. **Layer caching** — copy package files before source code
4. **Multi-stage builds** — separate build and runtime
5. **Run as non-root** — create and use a dedicated user
6. **One process per container** — easier scaling and debugging
7. **Use HEALTHCHECK** — let Docker know if your app is actually working
8. **Minimize layers** — combine RUN commands with `&&`
9. **Don't store secrets in images** — use env vars or secrets management
10. **Scan for vulnerabilities** — `docker scout cves myimage`

---

## Chapter 14: Practical Project — Full Node.js API

### Project Structure
```
my-api/
├── src/
│   ├── index.js
│   ├── routes/
│   │   └── users.js
│   └── db.js
├── package.json
├── Dockerfile
├── .dockerignore
└── .env
```

### src/index.js
```javascript
const express = require('express');
const userRoutes = require('./routes/users');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

app.use('/api/users', userRoutes);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Dockerfile
```dockerfile
FROM node:18-alpine

RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

COPY package*.json ./
RUN npm ci --production

COPY --chown=app:app . .

USER app

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --spider --quiet http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

### Build, Run, Test
```bash
# Build the image
docker build -t my-api:1.0 .

# Run it
docker run -d \
  --name api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  my-api:1.0

# Test it
curl http://localhost:3000/health

# View logs
docker logs -f api

# Check health status
docker inspect --format='{{.State.Health.Status}}' api
```
