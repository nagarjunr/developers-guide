# Docker Compose Multi-Service Architecture

A complete guide to orchestrating multi-service applications with Docker Compose, covering database, backend, and frontend services with proper networking, health checks, and environment management.

## Table of Contents

1. [Introduction](#introduction)
2. [Basic Architecture](#basic-architecture)
3. [Service Definitions](#service-definitions)
4. [Networking](#networking)
5. [Volume Management](#volume-management)
6. [Health Checks](#health-checks)
7. [Environment Variables](#environment-variables)
8. [Dependencies](#dependencies)
9. [Complete Example](#complete-example)
10. [Best Practices](#best-practices)
11. [Troubleshooting](#troubleshooting)

---

## Introduction

**Docker Compose** orchestrates multi-container applications using a single YAML file. It's perfect for:

✅ Local development environments
✅ Integration testing
✅ Staging environments
✅ Small production deployments

**Why use Docker Compose?**
- Single command to start/stop entire stack (`docker-compose up/down`)
- Service isolation with virtual networks
- Easy environment configuration
- Consistent across development and production

---

## Basic Architecture

### Typical Three-Tier Application

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph TB
    A[User's Browser]:::purple -->|HTTP :8080| B[Frontend Container<br/>React + Nginx<br/>- Serves static files<br/>- Proxies /api]:::cyan
    B -->|HTTP :8001| C[Backend Container<br/>FastAPI/Express<br/>- REST API endpoints<br/>- Business logic]:::blue
    C -->|PostgreSQL :5432| D[Database Container<br/>PostgreSQL<br/>- Persistent storage<br/>- Volume mounted]:::green

    E[app-network]:::orange -.connects all services.- B
    E -.connects all services.- C
    E -.connects all services.- D

    classDef purple fill:#9B59B6,stroke:#6C3483,color:#fff
    classDef cyan fill:#3498DB,stroke:#2471A3,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
```

All services connected via: app-network

---

## Service Definitions

### 1. Database Service (PostgreSQL)

```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: my-app-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-myapp}
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

**Key points:**
- `postgres:15-alpine` - Lightweight Alpine Linux image
- `${VAR:-default}` - Environment variable with fallback default
- `volumes` - Persist data even after container stops
- `healthcheck` - Verify database is ready before dependent services start
- `restart: unless-stopped` - Auto-restart on crash

### 2. Backend Service (FastAPI/Express)

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        GIT_TOKEN: ${GIT_TOKEN}
    container_name: my-app-backend
    depends_on:
      postgres:
        condition: service_healthy  # Wait for DB to be healthy
    environment:
      # Database connection (use service name as host)
      DATABASE_URL: postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-postgres}@postgres:5432/${POSTGRES_DB:-myapp}

      # API keys
      API_KEY: ${API_KEY}

      # Application config
      PYTHONUNBUFFERED: 1
      PYTHONPATH: /app
    ports:
      - "${BACKEND_PORT:-8001}:8001"
    volumes:
      - ./src/data:/app/data  # Persistent data
      - ./logs:/app/logs      # Log files
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s  # Wait 40s before first check
    networks:
      - app-network
    restart: unless-stopped
```

**Key points:**
- `build` - Build from local Dockerfile
- `depends_on` with `condition` - Wait for database health check
- `DATABASE_URL` uses `postgres` as hostname (service name)
- `volumes` - Mount local directories for development
- `start_period` - Grace period for slow-starting services

### 3. Frontend Service (React + Nginx)

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VITE_API_URL: ${VITE_API_URL:-}  # Empty for nginx proxy
    container_name: my-app-frontend
    depends_on:
      backend:
        condition: service_healthy
    ports:
      - "${FRONTEND_PORT:-8080}:80"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
```

**Frontend Dockerfile with Nginx:**

```dockerfile
# Stage 1: Build React app
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=${VITE_API_URL}

RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf (API proxy):**

```nginx
server {
    listen 80;

    # Serve frontend
    location / {
        root /usr/share/nginx/html;
        try_files $uri /index.html;
    }

    # Proxy API requests to backend
    location /api/ {
        proxy_pass http://backend:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Networking

### Default Network (Bridge)

```yaml
networks:
  app-network:
    driver: bridge
```

**How it works:**
- All services in the same network can communicate
- Services use **service names** as hostnames (DNS resolution)
- Example: Backend connects to `postgres:5432` (not `localhost:5432`)

### Service Communication

```python
# Backend connecting to PostgreSQL
# ❌ Wrong: localhost (works only inside backend container)
DATABASE_URL = "postgresql://user:pass@localhost:5432/db"

# ✅ Correct: Use service name
DATABASE_URL = "postgresql://user:pass@postgres:5432/db"
```

```javascript
// Frontend calling backend API
// ✅ In Docker Compose: Use empty string (nginx proxy)
const API_BASE = import.meta.env.VITE_API_URL || '';
fetch(`${API_BASE}/api/users`);  // → /api/users → nginx proxies to backend:8001
```

### External Access

```yaml
services:
  backend:
    ports:
      - "8001:8001"  # Host:Container
```

- `localhost:8001` from host machine → `backend:8001` in container
- Port mapping allows external access for development/debugging

---

## Volume Management

### Types of Volumes

#### 1. Named Volumes (Persistent Data)

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    driver: local
```

**Use for:** Database data that should persist across container restarts

**Advantages:**
- Managed by Docker
- Persist even after `docker-compose down`
- Can be backed up

**Location:** `/var/lib/docker/volumes/` (Linux/Mac)

#### 2. Bind Mounts (Development)

```yaml
services:
  backend:
    volumes:
      - ./src:/app/src        # Hot reload in development
      - ./logs:/app/logs      # Access logs from host
```

**Use for:** Development files, logs, configuration

**Advantages:**
- Immediate sync with host filesystem
- Edit code on host, see changes in container
- Easy access to logs

#### 3. Read-Only Volumes

```yaml
services:
  backend:
    volumes:
      - ./config:/app/config:ro  # Read-only
```

**Use for:** Configuration files that shouldn't be modified by the container

### Volume Commands

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-project_postgres_data

# Remove unused volumes
docker volume prune

# Remove specific volume
docker volume rm my-project_postgres_data
```

---

## Health Checks

Health checks ensure services are ready before dependent services start.

### PostgreSQL Health Check

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 10s      # Check every 10 seconds
  timeout: 5s        # Fail if check takes > 5 seconds
  retries: 5         # Retry 5 times before marking unhealthy
  start_period: 10s  # Grace period before first check
```

**What it does:**
- Runs `pg_isready` command to check if database accepts connections
- Service is "healthy" only after check passes
- Dependent services wait for "healthy" status

### HTTP Health Check (Backend)

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**Backend health endpoint:**

```python
# FastAPI
@app.get("/health")
def health_check():
    """Health check endpoint for Docker."""
    return {"status": "healthy"}
```

```javascript
// Express
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});
```

### Custom Health Check (Python)

```yaml
healthcheck:
  test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8001/health')"]
  interval: 30s
  timeout: 10s
  retries: 3
```

**Useful when:** curl is not available in the container

---

## Environment Variables

### .env File

```bash
# .env
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_DB=myapp
POSTGRES_PORT=5432

BACKEND_PORT=8001
FRONTEND_PORT=8080

API_KEY=your_api_key_here
GIT_TOKEN=your_github_token

VITE_API_URL=
```

**Load in docker-compose.yml:**

```yaml
services:
  postgres:
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### Default Values

```yaml
environment:
  POSTGRES_USER: ${POSTGRES_USER:-postgres}  # Default: postgres
  BACKEND_PORT: ${BACKEND_PORT:-8001}        # Default: 8001
```

### Environment File Per Service

```yaml
services:
  backend:
    env_file:
      - .env              # Common variables
      - .env.backend      # Backend-specific
```

### Build-Time vs Runtime

```yaml
services:
  backend:
    build:
      args:
        GIT_TOKEN: ${GIT_TOKEN}  # Build-time variable
    environment:
      API_KEY: ${API_KEY}        # Runtime variable
```

**Build-time:** Used during `docker build`
**Runtime:** Used when container runs

---

## Dependencies

### Basic Dependency

```yaml
services:
  backend:
    depends_on:
      - postgres
```

**What it does:**
- Starts `postgres` before `backend`
- Does NOT wait for postgres to be ready

### Dependency with Health Check

```yaml
services:
  backend:
    depends_on:
      postgres:
        condition: service_healthy
```

**What it does:**
- Starts `postgres` first
- Waits for postgres health check to pass
- Then starts `backend`

### Multiple Dependencies

```yaml
services:
  frontend:
    depends_on:
      backend:
        condition: service_healthy
      postgres:
        condition: service_healthy
```

### Startup Order

```
1. postgres starts → health check passes
2. backend starts (waits for postgres healthy) → health check passes
3. frontend starts (waits for backend healthy)
```

---

## Complete Example

**File: docker-compose.yml**

```yaml
version: '3.8'

services:
  # Database
  postgres:
    image: postgres:15-alpine
    container_name: myapp-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-myapp}
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    restart: unless-stopped

  # Backend API
  backend:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        GIT_TOKEN: ${GIT_TOKEN}
    container_name: myapp-backend
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-postgres}@postgres:5432/${POSTGRES_DB:-myapp}
      API_KEY: ${API_KEY}
      PYTHONUNBUFFERED: 1
    ports:
      - "${BACKEND_PORT:-8001}:8001"
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app-network
    restart: unless-stopped

  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VITE_API_URL: ${VITE_API_URL:-}
    container_name: myapp-frontend
    depends_on:
      backend:
        condition: service_healthy
    ports:
      - "${FRONTEND_PORT:-8080}:80"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

**File: .env**

```bash
# Database
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mysecretpass
POSTGRES_DB=myapp
POSTGRES_PORT=5432

# Backend
BACKEND_PORT=8001
API_KEY=your_api_key

# Frontend
FRONTEND_PORT=8080
VITE_API_URL=

# Build args
GIT_TOKEN=your_github_token
```

**Commands:**

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f backend

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild and restart
docker-compose up --build -d

# Check service status
docker-compose ps

# Execute command in service
docker-compose exec backend bash
docker-compose exec postgres psql -U myuser -d myapp
```

---

## Best Practices

### 1. Use Health Checks

```yaml
# ✅ Good: Wait for services to be ready
depends_on:
  postgres:
    condition: service_healthy

# ❌ Bad: Start immediately (may fail if DB not ready)
depends_on:
  - postgres
```

### 2. Use Named Volumes for Data

```yaml
# ✅ Good: Persistent data survives container restart
volumes:
  - postgres_data:/var/lib/postgresql/data

# ❌ Bad: Data lost when container stops
# (no volume)
```

### 3. Set Restart Policies

```yaml
restart: unless-stopped  # ✅ Auto-restart on failure
restart: always          # ✅ Always restart
restart: on-failure      # ✅ Only on error
restart: no              # ❌ Never restart (default)
```

### 4. Use Environment Variable Defaults

```yaml
# ✅ Good: Works without .env file
POSTGRES_USER: ${POSTGRES_USER:-postgres}

# ❌ Bad: Fails if POSTGRES_USER not set
POSTGRES_USER: ${POSTGRES_USER}
```

### 5. Separate Dev and Prod Configs

```bash
# Development
docker-compose.yml              # Base config
docker-compose.override.yml     # Dev overrides (auto-loaded)

# Production
docker-compose.yml              # Base config
docker-compose.prod.yml         # Prod overrides
```

**Usage:**

```bash
# Development (auto-loads override)
docker-compose up

# Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

### 6. Use Multi-Stage Builds

Reduce image size and improve security:

```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### 7. Set Container Resource Limits

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '0.5'      # Max 50% of 1 CPU
          memory: 512M     # Max 512MB RAM
        reservations:
          memory: 256M     # Minimum 256MB RAM
```

---

## Troubleshooting

### Issue 1: Service Won't Start

**Check logs:**

```bash
docker-compose logs backend
```

**Common causes:**
- Missing environment variables
- Port already in use
- Failed health check

**Solution:**

```bash
# Check what's using port
lsof -i :8001

# Kill process
kill -9 <PID>

# Or change port in .env
BACKEND_PORT=8002
```

### Issue 2: Database Connection Failed

**Error:**

```
psycopg2.OperationalError: could not connect to server: Connection refused
```

**Solutions:**

1. **Use service name (not localhost):**
   ```python
   # ❌ Wrong
   DATABASE_URL = "postgresql://user:pass@localhost:5432/db"

   # ✅ Correct
   DATABASE_URL = "postgresql://user:pass@postgres:5432/db"
   ```

2. **Wait for health check:**
   ```yaml
   backend:
     depends_on:
       postgres:
         condition: service_healthy  # Wait for DB
   ```

3. **Check if services are on same network:**
   ```bash
   docker network inspect myapp_app-network
   ```

### Issue 3: Changes Not Reflected

**Cause:** Using old built image

**Solution:**

```bash
# Rebuild images
docker-compose up --build

# Or force rebuild
docker-compose build --no-cache
```

### Issue 4: Volume Permission Denied

**Error:**

```
PermissionError: [Errno 13] Permission denied: '/app/data/file.db'
```

**Solution:**

```dockerfile
# In Dockerfile, set proper permissions
RUN mkdir -p /app/data && \
    chmod -R 777 /app/data
```

### Issue 5: Container Keeps Restarting

**Check logs:**

```bash
docker-compose logs --tail=50 backend
```

**Common causes:**
- Application crashes immediately
- Failed health check
- Missing dependencies

**Debug:**

```bash
# Run container without restart policy
docker-compose run --rm backend bash

# Test command manually
python src/main.py
```

---

## Advanced Patterns

### Override for Local Development

**docker-compose.override.yml:**

```yaml
services:
  backend:
    volumes:
      - ./src:/app/src  # Hot reload
    command: uvicorn src.main:app --reload --host 0.0.0.0
```

### Production Optimization

**docker-compose.prod.yml:**

```yaml
services:
  backend:
    restart: always
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 1G
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## Summary

**Key Takeaways:**

1. **Service Isolation** - Each service runs in its own container
2. **Networking** - Use service names as hostnames (DNS resolution)
3. **Volumes** - Named volumes for data persistence, bind mounts for development
4. **Health Checks** - Ensure services are ready before dependencies start
5. **Environment Variables** - Use `.env` with defaults
6. **Dependencies** - Use `condition: service_healthy` for proper startup order
7. **Restart Policies** - `unless-stopped` for production reliability

## Further Reading

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Compose File Reference](https://docs.docker.com/compose/compose-file/)
- [Docker Networking](https://docs.docker.com/network/)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)

---

**Created:** 2026-02-06
**Tags:** #docker #docker-compose #devops #containers #orchestration #microservices
