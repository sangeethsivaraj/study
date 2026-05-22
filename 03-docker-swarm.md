# Docker Swarm - From Scratch to Advanced

---

## Chapter 1: What is Docker Swarm and Why Does It Exist?

### The Problem: Single Server Limitations

With Docker and Docker Compose, you run containers on ONE machine. But what happens when:

- Your application gets millions of users and one server can't handle the load?
- Your server crashes at 3 AM and your app goes down?
- You need to deploy a new version without any downtime?
- You need to distribute your containers across data centers?

**You need orchestration** — a system that manages containers across MULTIPLE servers.

### What Docker Swarm Does

Docker Swarm turns multiple Docker hosts (machines) into a single cluster. You tell Swarm "I want 10 copies of my web app running" and Swarm:

1. Decides which machines to put them on
2. Distributes them evenly
3. If a machine dies, moves containers to healthy machines
4. Handles rolling updates with zero downtime
5. Load-balances incoming traffic across all copies

### Swarm vs Kubernetes

| Aspect | Docker Swarm | Kubernetes |
|--------|-------------|------------|
| Setup | 1 command (`docker swarm init`) | Complex (kubeadm, cloud setup) |
| Learning curve | Easy (you already know Docker) | Steep (new concepts: pods, deployments, etc.) |
| Built-in | Part of Docker | Separate system |
| Scale | Good for small-medium (< 100 nodes) | Enterprise-scale (1000s of nodes) |
| Features | Essential orchestration | Advanced (auto-scaling, service mesh, etc.) |
| Community | Smaller | Massive ecosystem |

**Choose Swarm when:** You already know Docker, have a small team, and need simple orchestration quickly.

---

## Chapter 2: Swarm Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DOCKER SWARM CLUSTER                      │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              MANAGER NODES (Control Plane)                │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │  Manager 1  │  │  Manager 2  │  │  Manager 3  │     │   │
│  │  │  (Leader)   │  │  (Follower) │  │  (Follower) │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  │                                                           │   │
│  │  Responsibilities:                                        │   │
│  │  - Accept commands from users                             │   │
│  │  - Schedule tasks onto worker nodes                       │   │
│  │  - Maintain cluster state (Raft consensus)                │   │
│  │  - Handle service discovery                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              │ Assign tasks                       │
│                              ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              WORKER NODES (Data Plane)                     │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │  Worker 1   │  │  Worker 2   │  │  Worker 3   │     │   │
│  │  │ [app][app]  │  │ [app][db]   │  │ [app][cache]│     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  │                                                           │   │
│  │  Responsibilities:                                        │   │
│  │  - Run containers (tasks)                                 │   │
│  │  - Report status back to managers                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Terms

| Term | What it means | Analogy |
|------|--------------|---------|
| **Node** | A machine (physical or VM) in the swarm | A worker in a factory |
| **Manager Node** | Controls the cluster, assigns work | The factory supervisor |
| **Worker Node** | Runs containers | Factory floor workers |
| **Service** | Definition of what to run (image, replicas, ports) | A job order |
| **Task** | One container running as part of a service | One unit of work |
| **Stack** | A group of services (like docker-compose for Swarm) | A complete production line |
| **Overlay Network** | Network that spans multiple hosts | Phone system connecting all buildings |

---

## Chapter 3: Setting Up a Swarm

### 3.1 Initialize the First Manager

```bash
# On your first server (will become the manager)
docker swarm init --advertise-addr 192.168.1.10

# Output:
# Swarm initialized: current node is now a manager.
#
# To add a worker to this swarm, run the following command:
#
#     docker swarm join --token SWMTKN-1-abc123def456... 192.168.1.10:2377
#
# To add a manager to this swarm, run 'docker swarm join-token manager'
```

**What just happened:**
- This machine is now a Swarm Manager (and also the Leader)
- A join token was generated for workers
- Port 2377 is used for cluster management

### 3.2 Add Worker Nodes

```bash
# On each worker machine, run the join command from the output above:
docker swarm join --token SWMTKN-1-abc123def456... 192.168.1.10:2377

# Output: This node joined a swarm as a worker.
```

### 3.3 Add More Managers (for High Availability)

```bash
# On the existing manager, get the manager join token:
docker swarm join-token manager

# On new manager nodes:
docker swarm join --token SWMTKN-1-manager-token... 192.168.1.10:2377
```

**Important:** Always use an ODD number of managers (3, 5, or 7) for Raft consensus to work.

### 3.4 Verify the Cluster

```bash
# From any manager node:
docker node ls

# Output:
# ID            HOSTNAME    STATUS    AVAILABILITY   MANAGER STATUS
# abc123 *     manager1    Ready     Active         Leader
# def456       manager2    Ready     Active         Reachable
# ghi789       manager3    Ready     Active         Reachable
# jkl012       worker1     Ready     Active
# mno345       worker2     Ready     Active
```

### 3.5 Single-Machine Swarm (for Learning/Testing)

You can practice Swarm on a single machine:

```bash
docker swarm init
# That's it! You have a one-node swarm (both manager and worker)
```

---

## Chapter 4: Services — The Core of Swarm

### What is a Service?

In plain Docker, you run containers. In Swarm, you create **services**. A service is a declaration: "I want N copies of this container running, on these ports, with these settings."

Swarm ensures the desired state is maintained. If a container crashes, Swarm starts a new one.

### 4.1 Create Your First Service

```bash
# Create a service with 1 replica
docker service create --name web --publish 80:80 nginx:alpine

# What this does:
# 1. Pulls nginx:alpine on a node
# 2. Starts one container (task)
# 3. Maps port 80 on ALL nodes to port 80 in the container
```

### 4.2 Inspect the Service

```bash
# List all services
docker service ls
# ID       NAME   MODE         REPLICAS   IMAGE
# abc123   web    replicated   1/1        nginx:alpine

# See which nodes are running the tasks
docker service ps web
# ID       NAME    IMAGE          NODE       STATE
# xyz789   web.1   nginx:alpine   worker1    Running

# Detailed service info
docker service inspect --pretty web
```

### 4.3 Scale the Service

```bash
# Scale to 5 replicas
docker service scale web=5

# Verify
docker service ps web
# NAME    IMAGE          NODE       STATE
# web.1   nginx:alpine   worker1    Running
# web.2   nginx:alpine   worker2    Running
# web.3   nginx:alpine   manager1   Running
# web.4   nginx:alpine   worker1    Running
# web.5   nginx:alpine   worker2    Running
```

Swarm distributed 5 containers across the available nodes.

### 4.4 Service Modes

**Replicated (default):** Run N copies, Swarm decides where.
```bash
docker service create --name web --replicas 5 nginx
```

**Global:** Run exactly ONE on EVERY node. Used for monitoring agents, log collectors.
```bash
docker service create --name monitor --mode global datadog/agent
```

---

## Chapter 5: Routing Mesh (Built-in Load Balancing)

### How It Works

When you publish a port (`--publish 80:80`), Swarm creates a **routing mesh**:

```
User → http://ANY-NODE:80 → Swarm Routing Mesh → container (on any node)
```

You can hit port 80 on ANY node in the swarm, even if that node doesn't have the container running! The routing mesh forwards the request to a node that does.

```
┌────────────────────────────────────────────────────┐
│              Incoming Request: port 80              │
│                       │                            │
│     ┌─────────────────┼─────────────────┐         │
│     │                 │                 │         │
│     ▼                 ▼                 ▼         │
│ ┌────────┐      ┌────────┐      ┌────────┐      │
│ │ Node 1 │      │ Node 2 │      │ Node 3 │      │
│ │ [web]  │      │        │      │ [web]  │      │
│ └────────┘      └────────┘      └────────┘      │
│                                                   │
│ Even Node 2 (with no container) accepts traffic   │
│ and routes it to Node 1 or Node 3                 │
└────────────────────────────────────────────────────┘
```

**Benefit:** Put a single load balancer in front of your swarm, pointing to ALL nodes. Swarm handles the rest.

---

## Chapter 6: Rolling Updates (Zero-Downtime Deployments)

### Update a Service's Image

```bash
docker service update --image nginx:1.25 web
```

### Control the Update Strategy

```bash
docker service update \
  --image myapp:v2 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  --update-max-failure-ratio 0.25 \
  --update-order start-first \
  web
```

**Explanation:**
| Parameter | Meaning |
|-----------|---------|
| `--update-parallelism 2` | Update 2 containers at a time |
| `--update-delay 10s` | Wait 10 seconds between batches |
| `--update-failure-action rollback` | If update fails, roll back automatically |
| `--update-max-failure-ratio 0.25` | Tolerate up to 25% failures before acting |
| `--update-order start-first` | Start new container BEFORE stopping old one (no downtime) |

### How It Works Visually

```
Time →

Replicas: [v1] [v1] [v1] [v1] [v1]     (initial state)
           ↓    ↓
          [v2] [v2] [v1] [v1] [v1]     (batch 1: 2 updated)
                      ↓    ↓
          [v2] [v2] [v2] [v2] [v1]     (batch 2: 2 more updated)
                                ↓
          [v2] [v2] [v2] [v2] [v2]     (done!)
```

### Rollback

```bash
# Manual rollback to previous version
docker service rollback web

# Automatic rollback on failure is configured via:
# --update-failure-action rollback
```

---

## Chapter 7: Overlay Networks (Multi-Host Networking)

### The Problem

On a single machine, containers communicate via bridge networks. But in a swarm, containers are on DIFFERENT physical machines. How do they talk?

### The Solution: Overlay Networks

An overlay network creates a virtual network that spans all nodes. Containers on different machines can communicate as if they're on the same LAN.

```bash
# Create an overlay network
docker network create --driver overlay --attachable my-overlay

# Create services on the same overlay
docker service create --name api --network my-overlay --replicas 3 myapi:latest
docker service create --name db --network my-overlay postgres:16

# "api" containers can connect to "db" by name, regardless of which node they're on!
```

### Network Isolation

```bash
# Frontend network (public-facing)
docker network create --driver overlay frontend-net

# Backend network (internal only)
docker network create --driver overlay --internal backend-net

# Web service: on both networks
docker service create --name web \
  --network frontend-net \
  --network backend-net \
  --publish 80:3000 \
  my-web

# Database: only on backend (not directly reachable from outside)
docker service create --name db \
  --network backend-net \
  postgres:16
```

### Encrypted Overlay (Sensitive Data)

```bash
docker network create --driver overlay --opt encrypted secure-net
```

All traffic between nodes on this network is encrypted with IPsec.

---

## Chapter 8: Secrets Management

### Why Secrets?

You should NEVER put passwords in:
- Docker images (anyone can inspect them)
- Environment variables in compose files (checked into git)
- Plain text files in the container

Swarm Secrets stores sensitive data encrypted and only provides it to the containers that need it.

### Creating Secrets

```bash
# From a string (piped)
echo "my-database-password" | docker secret create db_password -

# From a file
docker secret create ssl_certificate ./server.crt
docker secret create ssl_key ./server.key

# List secrets
docker secret ls

# Inspect (shows metadata, NOT the value)
docker secret inspect db_password
```

### Using Secrets in Services

```bash
docker service create \
  --name api \
  --secret db_password \
  --secret ssl_certificate \
  --secret ssl_key \
  myapi:latest
```

Inside the container, secrets appear as files:
```
/run/secrets/db_password
/run/secrets/ssl_certificate
/run/secrets/ssl_key
```

### Reading Secrets in Your Application

**Node.js:**
```javascript
const fs = require('fs');

function getSecret(name) {
  try {
    return fs.readFileSync(`/run/secrets/${name}`, 'utf8').trim();
  } catch (err) {
    // Fallback to environment variable (for local development)
    return process.env[name.toUpperCase()];
  }
}

const dbPassword = getSecret('db_password');
```

**Python:**
```python
from pathlib import Path

def get_secret(name):
    secret_path = Path(f'/run/secrets/{name}')
    if secret_path.exists():
        return secret_path.read_text().strip()
    # Fallback for local dev
    import os
    return os.environ.get(name.upper())

db_password = get_secret('db_password')
```

### PostgreSQL with Secrets

PostgreSQL supports `_FILE` suffix for environment variables:

```bash
docker service create \
  --name db \
  --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:16
```

---

## Chapter 9: Configs (Non-Sensitive Configuration)

For configuration files (nginx.conf, app settings) that aren't sensitive:

```bash
# Create a config
docker config create nginx-config ./nginx.conf

# Use in a service
docker service create \
  --name web \
  --config source=nginx-config,target=/etc/nginx/nginx.conf \
  --publish 80:80 \
  nginx:alpine

# Update config (create new version, update service)
docker config create nginx-config-v2 ./nginx-v2.conf
docker service update \
  --config-rm nginx-config \
  --config-add source=nginx-config-v2,target=/etc/nginx/nginx.conf \
  web
```

**Secrets vs Configs:**
| | Secrets | Configs |
|--|---------|---------|
| Storage | Encrypted | Plain text |
| Location in container | `/run/secrets/` | Anywhere you specify |
| Use for | Passwords, keys, certificates | Config files, settings |

---

## Chapter 10: Stacks — Compose Files for Swarm

### What is a Stack?

A Stack is how you deploy a multi-service application to Swarm. It uses the same docker-compose.yml format with additional `deploy` options.

### Basic Stack File

```yaml
version: "3.8"

services:
  web:
    image: myregistry.com/web:latest
    ports:
      - "80:3000"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

  api:
    image: myregistry.com/api:latest
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    networks:
      - backend
    secrets:
      - db_password

  db:
    image: postgres:16-alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    secrets:
      - db_password

networks:
  backend:
    driver: overlay

volumes:
  db-data:

secrets:
  db_password:
    external: true    # Secret must already exist in the swarm
```

### Deploy the Stack

```bash
# Deploy (or update if it already exists)
docker stack deploy -c docker-stack.yml myapp

# List stacks
docker stack ls

# List services in a stack
docker stack services myapp

# List all tasks (containers) in a stack
docker stack ps myapp

# Remove a stack
docker stack rm myapp
```

### Key Differences from docker-compose

| Feature | docker compose up | docker stack deploy |
|---------|------------------|-------------------|
| `build:` | ✅ Supported | ❌ Not supported (must use pre-built images) |
| `deploy:` | ❌ Ignored | ✅ Used for replicas, resources, etc. |
| `depends_on:` | ✅ Supported | ❌ Ignored (use health checks instead) |
| Networks | Bridge | Overlay |
| Secrets | File-based only | Full Swarm secret management |

**Important:** Stacks require pre-built images. Push your images to a registry first:
```bash
docker build -t myregistry.com/myapp:v1 .
docker push myregistry.com/myapp:v1
```

---

## Chapter 11: Production Stack — Complete Example

### Full Production Stack with Traefik, API, DB, Monitoring

```yaml
version: "3.8"

services:
  # ========== REVERSE PROXY / LOAD BALANCER ==========
  traefik:
    image: traefik:v2.10
    command:
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.le.acme.email=devops@company.com"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
      - "--metrics.prometheus=true"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt
    networks:
      - public
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 30s

  # ========== WEB APPLICATION ==========
  web:
    image: myregistry.com/frontend:${VERSION:-latest}
    networks:
      - public
    deploy:
      replicas: 4
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
        order: start-first
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.web.rule=Host(`www.myapp.com`)"
        - "traefik.http.routers.web.tls.certresolver=le"
        - "traefik.http.services.web.loadbalancer.server.port=3000"

  # ========== API SERVICE ==========
  api:
    image: myregistry.com/api:${VERSION:-latest}
    networks:
      - public
      - backend
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://cache:6379
    secrets:
      - db_url
      - jwt_secret
    deploy:
      replicas: 6
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
        order: start-first
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.api.rule=Host(`api.myapp.com`)"
        - "traefik.http.routers.api.tls.certresolver=le"
        - "traefik.http.services.api.loadbalancer.server.port=4000"
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:4000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

  # ========== DATABASE ==========
  db:
    image: postgres:16-alpine
    networks:
      - backend
    environment:
      POSTGRES_DB: production
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ========== CACHE ==========
  cache:
    image: redis:7-alpine
    networks:
      - backend
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 512M

  # ========== BACKGROUND WORKER ==========
  worker:
    image: myregistry.com/worker:${VERSION:-latest}
    networks:
      - backend
    secrets:
      - db_url
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 256M

# ========== NETWORKS ==========
networks:
  public:
    driver: overlay
  backend:
    driver: overlay
    internal: true        # No external access

# ========== VOLUMES ==========
volumes:
  db-data:
  redis-data:
  letsencrypt:

# ========== SECRETS ==========
secrets:
  db_password:
    external: true
  db_url:
    external: true
  jwt_secret:
    external: true
```

### Deploy It

```bash
# Create secrets first
echo "postgres_password_here" | docker secret create db_password -
echo "postgresql://postgres:pass@db:5432/production" | docker secret create db_url -
echo "my-jwt-secret-key" | docker secret create jwt_secret -

# Label nodes
docker node update --label-add storage=ssd node-with-ssd

# Deploy the stack
VERSION=1.2.0 docker stack deploy -c production-stack.yml myapp

# Monitor deployment
watch docker stack ps myapp
```

---

## Chapter 12: Node Management

### Labeling Nodes

Labels let you control where services run:

```bash
# Add labels
docker node update --label-add region=us-east worker1
docker node update --label-add region=eu-west worker2
docker node update --label-add storage=ssd worker3
docker node update --label-add tier=gpu worker4

# Use in placement constraints
docker service create \
  --name ml-service \
  --constraint 'node.labels.tier==gpu' \
  ml-model:latest

docker service create \
  --name eu-api \
  --constraint 'node.labels.region==eu-west' \
  my-api:latest
```

### Draining Nodes (Maintenance)

```bash
# Before maintenance: drain the node (moves all tasks away)
docker node update --availability drain worker1

# The node is still in the swarm but won't receive new tasks
# Existing tasks are rescheduled to other nodes

# After maintenance: bring it back
docker node update --availability active worker1
```

### Promoting and Demoting

```bash
# Promote a worker to manager
docker node promote worker2

# Demote a manager to worker
docker node demote manager3
```

### Removing Nodes

```bash
# From the node itself:
docker swarm leave

# From a manager (if node is unresponsive):
docker node rm --force worker5
```

---

## Chapter 13: High Availability

### Manager Fault Tolerance

Swarm uses Raft consensus. You need a MAJORITY of managers alive:

| Managers | Can tolerate failures | Majority needed |
|----------|----------------------|-----------------|
| 1 | 0 | 1 |
| 3 | 1 | 2 |
| 5 | 2 | 3 |
| 7 | 3 | 4 |

**Best practice:** 3 or 5 managers. More than 7 slows down consensus.

### What Happens When Things Fail

**A worker node dies:**
- Swarm detects it's gone (within ~5 seconds)
- Reschedules all its tasks to other healthy workers
- Traffic is automatically re-routed

**A manager (non-leader) dies:**
- Cluster continues working normally
- One fewer manager for fault tolerance

**The leader dies:**
- Remaining managers elect a new leader (seconds)
- Cluster continues with brief pause

**Majority of managers die:**
- Cluster becomes read-only (can't make changes)
- Existing containers keep running
- Recovery: `docker swarm init --force-new-cluster` on a surviving manager

### Disaster Recovery

```bash
# If you lose majority of managers, on a surviving manager:
docker swarm init --force-new-cluster --advertise-addr 192.168.1.10

# This creates a new single-manager swarm with existing state
# Then add new managers
```

---

## Chapter 14: Health Checks and Self-Healing

### Service Health Checks

```yaml
services:
  api:
    image: myapi:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 15s        # Check every 15 seconds
      timeout: 5s          # Consider unhealthy if no response in 5s
      retries: 3           # Mark unhealthy after 3 consecutive failures
      start_period: 30s    # Grace period on startup
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

**What happens:**
1. Container starts → 30s grace period (start_period)
2. Every 15s, Docker runs the health check
3. If 3 consecutive checks fail → container marked "unhealthy"
4. Swarm kills the unhealthy container and schedules a new one

### Writing Good Health Checks

```javascript
// server.js — health endpoint
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    await db.query('SELECT 1');
    // Check Redis
    await redis.ping();
    
    res.status(200).json({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});
```

---

## Chapter 15: Logging and Monitoring

### Centralized Logging

In a swarm, containers are spread across machines. You need centralized logging.

```bash
# View logs for a service (aggregated from all replicas)
docker service logs -f myapp_api

# With timestamps
docker service logs --timestamps myapp_api

# Last 100 lines
docker service logs --tail 100 myapp_api
```

### Log Drivers

```yaml
services:
  api:
    image: myapi:latest
    logging:
      driver: json-file
      options:
        max-size: "10m"     # Max 10MB per log file
        max-file: "3"       # Keep 3 rotated files
```

### Monitoring with Prometheus + Grafana (Stack)

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    deploy:
      replicas: 1

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
    deploy:
      mode: global          # One on every node

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    deploy:
      mode: global          # One on every node

volumes:
  prometheus-data:
  grafana-data:
```

---

## Chapter 16: Advanced Deployment Patterns

### Blue-Green Deployment

Run two versions simultaneously, switch traffic:

```bash
# Blue is currently serving traffic
docker service create --name app-blue --network public \
  --publish 80:3000 --replicas 3 myapp:v1

# Deploy green alongside
docker service create --name app-green --network public \
  --replicas 3 myapp:v2

# Test green internally, then switch:
docker service update --publish-rm 80:3000 app-blue
docker service update --publish-add 80:3000 app-green

# Once confirmed, remove blue:
docker service rm app-blue
```

### Canary Deployment

Route a small percentage of traffic to the new version:

```bash
# Current: 10 replicas of v1
docker service create --name app --replicas 10 --publish 80:3000 myapp:v1

# Canary: update only 1 replica to v2
docker service update --replicas 11 app   # Add one more
# Then manually update one instance... 
# Or: use labels + Traefik weighted routing for true canary

# Simpler approach with separate services:
docker service create --name app-stable --replicas 9 myapp:v1
docker service create --name app-canary --replicas 1 myapp:v2
# Both behind same Traefik router = 10% canary traffic
```

---

## Chapter 17: Swarm Security Best Practices

1. **Encrypt overlay networks** for sensitive internal traffic
2. **Use secrets** — never put passwords in images or env vars
3. **Run as non-root** in your Dockerfiles
4. **Limit manager access** — only trusted operators should SSH into managers
5. **Rotate join tokens** periodically:
   ```bash
   docker swarm join-token --rotate worker
   docker swarm join-token --rotate manager
   ```
6. **Use TLS** — Swarm uses mutual TLS between nodes by default
7. **Scan images** for vulnerabilities before deploying
8. **Resource limits** — prevent one service from consuming all resources

---

## Chapter 18: Troubleshooting Swarm

### Common Issues and Solutions

```bash
# Service stuck at 0/N replicas
docker service ps --no-trunc myservice
# Look at the ERROR column for details

# Common causes:
# - Image not found (push to registry first!)
# - Port already in use
# - Resource constraints can't be satisfied
# - Placement constraints match no nodes

# Check if nodes are ready
docker node ls

# Check node details
docker node inspect worker1 --pretty

# Force service rebalance
docker service update --force myservice

# View all events in the swarm
docker events --filter type=service
```

---

## Docker Swarm Cheat Sheet

| Command | Purpose |
|---------|---------|
| `docker swarm init` | Create a swarm |
| `docker swarm join --token TOKEN IP:2377` | Join a swarm |
| `docker swarm leave` | Leave the swarm |
| `docker node ls` | List all nodes |
| `docker node update --label-add key=val node` | Label a node |
| `docker node update --availability drain node` | Drain node |
| `docker service create --name x --replicas N img` | Create service |
| `docker service ls` | List services |
| `docker service ps service_name` | List tasks |
| `docker service scale service_name=N` | Scale |
| `docker service update --image img:tag svc` | Update image |
| `docker service rollback svc` | Rollback |
| `docker service logs -f svc` | View logs |
| `docker service rm svc` | Remove service |
| `docker stack deploy -c file.yml name` | Deploy stack |
| `docker stack ls` | List stacks |
| `docker stack services name` | Stack services |
| `docker stack rm name` | Remove stack |
| `docker secret create name file` | Create secret |
| `docker config create name file` | Create config |
| `docker network create --driver overlay name` | Create overlay |
