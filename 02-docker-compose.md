# Docker Compose - From Scratch to Advanced

---

## Chapter 1: What is Docker Compose and Why Do You Need It?

### The Problem

Your application isn't just one container. A typical web app needs:
- A web server (Nginx)
- An application server (Node.js/Python/Java)
- A database (PostgreSQL/MySQL)
- A cache (Redis)
- A message queue (RabbitMQ)

Without Compose, you'd do this:

```bash
# Create a network
docker network create myapp-net

# Start database
docker run -d --name db --network myapp-net \
  -e POSTGRES_PASSWORD=secret \
  -v db-data:/var/lib/postgresql/data \
  postgres:16

# Start Redis
docker run -d --name cache --network myapp-net redis:7-alpine

# Start the app
docker run -d --name app --network myapp-net \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  -e REDIS_URL=redis://cache:6379 \
  -p 3000:3000 \
  my-app:latest

# Start Nginx
docker run -d --name nginx --network myapp-net \
  -v ./nginx.conf:/etc/nginx/nginx.conf \
  -p 80:80 \
  nginx:alpine
```

**Problems:**
- You need to remember all these commands
- Order matters (database must start before the app)
- Hard to share with team members
- Impossible to version control

### The Solution: Docker Compose

Write ONE YAML file (`docker-compose.yml`), run ONE command (`docker compose up`), and everything starts correctly.

---

## Chapter 2: Installation

Docker Compose v2 is included with Docker Desktop (Windows/Mac). For Linux:

```bash
# Install as a Docker plugin
sudo apt install docker-compose-plugin

# Verify
docker compose version
# Docker Compose version v2.24.0
```

**Note:** The old `docker-compose` (with hyphen) is v1 and deprecated. Use `docker compose` (space, v2).

---

## Chapter 3: Your First docker-compose.yml

### 3.1 The Simplest Possible Compose File

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
```

That's it! Run:
```bash
docker compose up
# Visit http://localhost:8080
# Press Ctrl+C to stop
```

### 3.2 Understanding the YAML Structure

```yaml
services:         # Define your containers here
  service-name:   # Each service = one container type
    image:        # Which image to use
    build:        # OR how to build the image
    ports:        # Port mappings
    volumes:      # Data persistence
    environment:  # Environment variables
    depends_on:   # Start order
    networks:     # Which networks to join
    restart:      # Restart policy
    deploy:       # Resource limits, replicas

volumes:          # Define named volumes
networks:         # Define custom networks
secrets:          # Define secrets
```

### 3.3 Realistic First Example: App + Database

```yaml
services:
  app:
    build: .                           # Build from Dockerfile in current dir
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:secret@db:5432/myapp
      - NODE_ENV=development
    depends_on:
      - db
    volumes:
      - ./src:/app/src                 # Hot-reload for development

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"                    # Expose for local DB tools
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:                              # Named volume (persists data)
```

**What happens when you run `docker compose up`:**
1. Creates a default network (e.g., `myproject_default`)
2. Builds the `app` image from your Dockerfile
3. Creates a named volume `pgdata`
4. Starts `db` first (because `app` depends on it)
5. Starts `app` after `db`
6. Both containers are on the same network — `app` can reach `db` by hostname "db"

---

## Chapter 4: Essential Commands (Explained)

```bash
# Start everything in background
docker compose up -d

# Start and force rebuild images
docker compose up -d --build

# Stop everything (containers stop, network removed)
docker compose down

# Stop everything AND delete volumes (WARNING: deletes database data!)
docker compose down -v

# View running services
docker compose ps

# View logs for all services
docker compose logs

# Follow logs for a specific service
docker compose logs -f app

# Run a one-off command in a service
docker compose exec app sh           # Shell into running container
docker compose run app npm test       # Start NEW container and run command

# Restart a specific service
docker compose restart app

# Scale a service (run multiple instances)
docker compose up -d --scale worker=5

# Validate your compose file (check for errors)
docker compose config

# Pull latest images
docker compose pull
```

**`exec` vs `run`:**
- `exec` — runs command in an EXISTING running container
- `run` — creates a NEW container to run the command (useful for one-off tasks)

---

## Chapter 5: Building Images in Compose

### 5.1 Simple Build

```yaml
services:
  app:
    build: .    # Uses ./Dockerfile
```

### 5.2 Custom Dockerfile Location

```yaml
services:
  app:
    build:
      context: ./backend           # Build context directory
      dockerfile: Dockerfile.dev   # Custom Dockerfile name
```

### 5.3 Build Arguments

```yaml
services:
  app:
    build:
      context: .
      args:
        - NODE_VERSION=20
        - APP_ENV=staging
```

Corresponds to `ARG` in Dockerfile:
```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine
```

### 5.4 Multi-Stage Build Target

```yaml
services:
  app:
    build:
      context: .
      target: development    # Build only up to this stage
```

```dockerfile
FROM node:18-alpine AS development
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "run", "dev"]

FROM node:18-alpine AS production
WORKDIR /app
COPY . .
RUN npm ci --production
CMD ["node", "dist/index.js"]
```

---

## Chapter 6: Environment Variables (3 Methods)

### Method 1: Inline in compose file

```yaml
services:
  app:
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
```

### Method 2: Using an .env file

**.env** (in the same directory as docker-compose.yml):
```
POSTGRES_USER=admin
POSTGRES_PASSWORD=supersecret
APP_PORT=3000
NODE_ENV=development
```

**docker-compose.yml:**
```yaml
services:
  app:
    ports:
      - "${APP_PORT}:3000"
    environment:
      - NODE_ENV=${NODE_ENV}

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### Method 3: env_file directive

```yaml
services:
  app:
    env_file:
      - .env              # Default env vars
      - .env.local        # Local overrides (gitignored)
```

**Priority (highest to lowest):**
1. `environment:` in compose file
2. Shell environment variables
3. `env_file:`
4. `.env` file (Compose auto-loads this)

---

## Chapter 7: Volumes — Every Pattern Explained

### 7.1 Named Volumes (for persistence)

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data    # Named volume

volumes:
  pgdata:    # Declare the volume
```

Docker manages these internally. Data persists even if the container is deleted.

### 7.2 Bind Mounts (for development)

```yaml
services:
  app:
    volumes:
      - ./src:/app/src              # Your source code → container
      - ./config:/app/config:ro     # Read-only mount
```

Changes on your machine instantly appear in the container. Perfect for development.

### 7.3 Anonymous Volumes (prevent overwriting)

```yaml
services:
  app:
    volumes:
      - ./src:/app/src          # Mount source code
      - /app/node_modules       # But DON'T override node_modules!
```

**Why?** When you bind-mount `./src:/app/src`, it could override `node_modules` inside the container. The anonymous volume preserves the container's `node_modules`.

### 7.4 tmpfs (in-memory, not persisted)

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /app/cache
```

Data lives in RAM only. Fast, but gone when container stops. Good for temporary files and caches.

---

## Chapter 8: Networking in Compose

### Default Behavior

Compose automatically creates a network. All services in the same compose file can reach each other by service name.

```yaml
services:
  frontend:
    # Can reach backend at http://backend:4000
    build: ./frontend

  backend:
    # Can reach db at postgresql://db:5432
    build: ./backend

  db:
    image: postgres:16
```

### Custom Networks (Isolation)

```yaml
services:
  frontend:
    networks:
      - public          # Only on public network

  backend:
    networks:
      - public          # Can talk to frontend
      - private         # Can talk to database

  db:
    networks:
      - private         # ONLY backend can reach it

networks:
  public:
  private:
```

**Why?** Security. The database is only accessible from the backend. The frontend cannot directly access it, even if compromised.

### Internal Networks (No outside access)

```yaml
networks:
  internal:
    internal: true    # No external/internet access
```

---

## Chapter 9: depends_on — Service Start Order

### Basic (just ordering)

```yaml
services:
  app:
    depends_on:
      - db
      - cache
```

This starts `db` and `cache` before `app`. BUT it doesn't wait for them to be READY, just started.

### With Health Check (wait until ready)

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy      # Wait until DB is actually ready
      cache:
        condition: service_started      # Just wait until started

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

**This is important!** Without health checks, your app might start before the database is ready to accept connections, causing crashes.

---

## Chapter 10: Development vs Production Setup

### Base: docker-compose.yml

```yaml
services:
  app:
    build: .
    environment:
      - NODE_ENV=production

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Development Override: docker-compose.override.yml

Docker Compose automatically merges this with the base file.

```yaml
services:
  app:
    build:
      context: .
      target: development              # Use dev stage of multi-stage build
    ports:
      - "3000:3000"
      - "9229:9229"                    # Node.js debugger port
    volumes:
      - ./src:/app/src                 # Hot-reload
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
    command: npm run dev               # Override CMD for development

  db:
    ports:
      - "5432:5432"                    # Expose DB for local tools (pgAdmin)

  # Extra service only for development
  adminer:
    image: adminer
    ports:
      - "8080:8080"                    # Database GUI
```

### Production: docker-compose.prod.yml

```yaml
services:
  app:
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    restart: always
    # No exposed ports in production!
```

### Running Different Configs

```bash
# Development (auto-loads docker-compose.yml + docker-compose.override.yml)
docker compose up

# Production (explicitly specify files)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Test environment
docker compose -f docker-compose.yml -f docker-compose.test.yml up
```

---

## Chapter 11: Real-World Use Cases

### Use Case 1: Full-Stack Web App (React + Express + PostgreSQL + Redis)

```yaml
services:
  # React Frontend
  frontend:
    build:
      context: ./frontend
      target: development
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src
    environment:
      - REACT_APP_API_URL=http://localhost:4000

  # Express API
  backend:
    build:
      context: ./backend
      target: development
    ports:
      - "4000:4000"
    volumes:
      - ./backend/src:/app/src
    environment:
      - DATABASE_URL=postgresql://postgres:secret@db:5432/myapp
      - REDIS_URL=redis://cache:6379
      - JWT_SECRET=development-secret
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started

  # PostgreSQL Database
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/01-init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Redis Cache
  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

  # Database Admin UI
  pgadmin:
    image: dpage/pgadmin4
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin

volumes:
  pgdata:
  redis-data:
```

### Use Case 2: Microservices Architecture

```yaml
services:
  # API Gateway / Reverse Proxy
  gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - user-service
      - order-service
      - product-service

  # User Service
  user-service:
    build: ./services/user
    environment:
      - DB_HOST=user-db
      - DB_NAME=users
    depends_on:
      user-db:
        condition: service_healthy

  # Order Service
  order-service:
    build: ./services/order
    environment:
      - DB_HOST=order-db
      - DB_NAME=orders
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672
    depends_on:
      order-db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  # Product Service
  product-service:
    build: ./services/product
    environment:
      - DB_HOST=product-db
      - DB_NAME=products

  # Databases (separate DB per service)
  user-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: secret
    volumes:
      - user-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5

  order-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: secret
    volumes:
      - order-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5

  product-db:
    image: mongo:7
    volumes:
      - product-db-data:/data/db

  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"     # Management UI
    healthcheck:
      test: ["CMD", "rabbitmqctl", "status"]
      interval: 10s
      retries: 5

volumes:
  user-db-data:
  order-db-data:
  product-db-data:
```

**nginx.conf for API Gateway:**
```nginx
events { worker_connections 1024; }

http {
  upstream user_service {
    server user-service:3000;
  }
  upstream order_service {
    server order-service:3000;
  }
  upstream product_service {
    server product-service:3000;
  }

  server {
    listen 80;

    location /api/users {
      proxy_pass http://user_service;
    }
    location /api/orders {
      proxy_pass http://order_service;
    }
    location /api/products {
      proxy_pass http://product_service;
    }
  }
}
```

### Use Case 3: CI/CD Testing Environment

```yaml
# docker-compose.test.yml
services:
  app:
    build:
      context: .
      target: test
    command: npm run test:ci
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://postgres:test@db:5432/test_db
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: test_db
      POSTGRES_PASSWORD: test
    # No volume — we want a fresh DB every time
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 3s
      retries: 10
```

Run tests:
```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from app
```
- `--abort-on-container-exit`: Stop everything when any container exits
- `--exit-code-from app`: Use the app container's exit code (0=pass, 1=fail)

### Use Case 4: Monitoring Stack (Prometheus + Grafana)

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/dashboards:/etc/grafana/provisioning/dashboards
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'

volumes:
  prometheus-data:
  grafana-data:
```

---

## Chapter 12: Resource Limits and Restart Policies

```yaml
services:
  app:
    # Restart policy
    restart: unless-stopped
    # Options: no | always | on-failure | unless-stopped

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: "2.0"        # Max 2 CPU cores
          memory: 1G          # Max 1GB RAM
        reservations:
          cpus: "0.5"        # Guaranteed 0.5 CPU
          memory: 256M        # Guaranteed 256MB RAM

    # Logging
    logging:
      driver: json-file
      options:
        max-size: "10m"      # Max 10MB per log file
        max-file: "5"        # Keep 5 rotated files
```

---

## Chapter 13: Secrets (Sensitive Data)

### File-based secrets

```yaml
services:
  app:
    secrets:
      - db_password
      - api_key

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
```

Inside the container, secrets are at `/run/secrets/<secret_name>`.

**Reading in Node.js:**
```javascript
const fs = require('fs');
const dbPassword = fs.readFileSync('/run/secrets/db_password', 'utf8').trim();
```

---

## Chapter 14: Advanced Patterns

### 14.1 Wait-for-it Script (Custom readiness check)

Sometimes health checks aren't enough. Use a wait script:

```yaml
services:
  app:
    command: ["./wait-for-it.sh", "db:5432", "--", "node", "server.js"]
    depends_on:
      - db
```

### 14.2 Shared Build Cache

```yaml
services:
  app:
    build:
      context: .
      cache_from:
        - myregistry.com/myapp:latest
```

### 14.3 Extending Services

```yaml
# Common config for multiple services
x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

services:
  service-a:
    <<: *common
    image: my-service-a

  service-b:
    <<: *common
    image: my-service-b
```

The `x-` prefix defines reusable fragments. `&common` names it, `<<: *common` injects it.

### 14.4 Profiles (Optional Services)

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"

  db:
    image: postgres:16

  # Only starts with --profile debug
  pgadmin:
    image: dpage/pgadmin4
    profiles: ["debug"]
    ports:
      - "5050:80"

  # Only starts with --profile monitoring
  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]
    ports:
      - "9090:9090"
```

```bash
docker compose up                              # Only app + db
docker compose --profile debug up              # app + db + pgadmin
docker compose --profile debug --profile monitoring up  # Everything
```

---

## Chapter 15: Docker Compose Cheat Sheet

| Command | Purpose |
|---------|---------|
| `docker compose up -d` | Start all services (detached) |
| `docker compose up -d --build` | Rebuild and start |
| `docker compose down` | Stop and remove |
| `docker compose down -v` | Stop, remove, delete volumes |
| `docker compose ps` | Show running services |
| `docker compose logs -f service` | Follow logs |
| `docker compose exec service sh` | Shell into container |
| `docker compose run service cmd` | Run one-off command |
| `docker compose restart service` | Restart one service |
| `docker compose pull` | Pull latest images |
| `docker compose config` | Validate compose file |
| `docker compose top` | Show running processes |
| `docker compose cp file service:/path` | Copy file into container |
