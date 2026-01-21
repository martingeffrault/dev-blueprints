# Docker (2025)

> **Last updated**: January 2026
> **Versions covered**: Docker 24+, BuildKit 0.12+
> **Purpose**: Container platform for building, shipping, and running applications

---

## Philosophy (2025-2026)

Docker containers provide **consistent, isolated environments** that work the same everywhere. Modern Docker emphasizes multi-stage builds, security-first practices, and minimal images.

**Key philosophical shifts:**
- **Multi-stage builds default** — Separate build and runtime
- **Distroless/Alpine** — Minimal attack surface
- **Non-root by default** — Security-first containers
- **BuildKit everywhere** — Faster, more efficient builds
- **Compose v2** — Modern orchestration syntax
- **Containers ≠ VMs** — Ephemeral, immutable mindset

---

## TL;DR

- Use multi-stage builds always
- Never use `latest` tag in production
- Never run as root in production
- Order Dockerfile for cache optimization
- Use `.dockerignore` to exclude files
- Use health checks for all containers
- Use secrets management, not environment variables for credentials
- Clean up unused images/containers regularly
- Don't treat containers like VMs — rebuild, don't patch

---

## Best Practices

### Multi-Stage Build (Node.js)

```dockerfile
# Dockerfile
# Stage 1: Dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

# Stage 2: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable pnpm && pnpm run build

# Stage 3: Production
FROM node:22-alpine AS runner
WORKDIR /app

# Security: Non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

# Copy only production artifacts
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

### Multi-Stage Build (Python)

```dockerfile
# Dockerfile
# Stage 1: Build
FROM python:3.12-slim AS builder
WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /app/wheels -r requirements.txt

# Stage 2: Production
FROM python:3.12-slim AS runner
WORKDIR /app

# Security: Non-root user
RUN useradd --create-home --shell /bin/bash appuser

# Copy only wheels and install
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels

# Copy application
COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Multi-Stage Build (Go)

```dockerfile
# Dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Dependencies first (cache)
COPY go.mod go.sum ./
RUN go mod download

# Build binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /app/server ./cmd/server

# Stage 2: Minimal runtime
FROM scratch AS runner
WORKDIR /app

# Copy binary only
COPY --from=builder /app/server .

# Copy CA certificates for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

USER 1000:1000
EXPOSE 8080

ENTRYPOINT ["/app/server"]
```

### .dockerignore

```
# .dockerignore
# Dependencies
node_modules
.pnpm-store

# Build outputs
dist
build
.next
.nuxt

# Development
.git
.gitignore
*.md
LICENSE
docs/

# IDE
.vscode
.idea
*.swp

# Testing
coverage
.nyc_output
*.test.*
__tests__

# Environment
.env
.env.*
!.env.example

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Misc
.DS_Store
*.log
tmp/
```

### Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder  # Use builder stage for dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules  # Anonymous volume for node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

### Docker Compose (Production)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myregistry/myapp:${VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        max_attempts: 3
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    secrets:
      - db_password
      - api_key
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true

secrets:
  db_password:
    external: true
  api_key:
    external: true
```

### Cache Mount (BuildKit)

```dockerfile
# Dockerfile with cache mounts
# syntax=docker/dockerfile:1

FROM node:22-alpine AS builder
WORKDIR /app

# Cache npm packages
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    corepack enable pnpm && pnpm install --frozen-lockfile

COPY . .
RUN pnpm run build
```

### Security Scanning

```bash
# Scan with Docker Scout
docker scout cves myimage:latest

# Scan with Trivy
trivy image myimage:latest

# Scan with Grype
grype myimage:latest
```

### Layer Ordering for Cache

```dockerfile
# ✅ Optimal order - dependencies rarely change
FROM node:22-alpine
WORKDIR /app

# 1. Copy dependency files first (rarely change)
COPY package.json pnpm-lock.yaml ./

# 2. Install dependencies (cached if package files unchanged)
RUN corepack enable pnpm && pnpm install --frozen-lockfile

# 3. Copy source code last (changes frequently)
COPY . .

# 4. Build
RUN pnpm run build
```

### Health Checks

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# For minimal images without curl
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Python health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
```

---

## Anti-Patterns

### ❌ Using `latest` Tag in Production

**Why it's bad**: No version control, unpredictable behavior.

```dockerfile
# ❌ DON'T
FROM node:latest
# Production: FROM myapp:latest

# ✅ DO — Pin versions
FROM node:22.11-alpine
# Production: FROM myapp:1.2.3
```

### ❌ Running as Root

**Why it's bad**: Security vulnerability if container is compromised.

```dockerfile
# ❌ DON'T — Runs as root by default
FROM node:22-alpine
COPY . .
CMD ["node", "app.js"]

# ✅ DO — Non-root user
FROM node:22-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --chown=app:app . .
USER app
CMD ["node", "app.js"]
```

### ❌ Single-Stage Builds

**Why it's bad**: Bloated images with build tools.

```dockerfile
# ❌ DON'T — 1GB+ image with devDependencies
FROM node:22
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/index.js"]

# ✅ DO — Multi-stage, ~100MB image
# (see multi-stage examples above)
```

### ❌ Hardcoding Secrets

**Why it's bad**: Secrets exposed in image layers.

```dockerfile
# ❌ DON'T
ENV DATABASE_PASSWORD=supersecret123
ENV API_KEY=sk-abcd1234

# ✅ DO — Use secrets or runtime env
# docker-compose: secrets or env_file
# Kubernetes: Secrets
# Runtime: --env-file or -e
```

### ❌ Installing Unnecessary Packages

**Why it's bad**: Larger attack surface, bigger images.

```dockerfile
# ❌ DON'T
RUN apt-get update && apt-get install -y \
    vim nano curl wget git gcc make python3

# ✅ DO — Only what's needed
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

### ❌ Treating Containers Like VMs

**Why it's bad**: Containers are ephemeral, not persistent.

```bash
# ❌ DON'T — SSH into container, install updates
docker exec -it container bash
apt-get update && apt-get upgrade

# ✅ DO — Rebuild image with updates
docker build -t myapp:1.2.4 .
docker stop myapp && docker rm myapp
docker run -d --name myapp myapp:1.2.4
```

### ❌ Not Using .dockerignore

**Why it's bad**: Slower builds, larger context, leaked secrets.

```dockerfile
# ❌ DON'T — COPY without .dockerignore
COPY . .
# Copies node_modules, .git, .env, etc.

# ✅ DO — Use .dockerignore (see above)
```

---

## 2025-2026 Changelog

| Feature | Date | Description |
|---------|------|-------------|
| BuildKit default | 2023+ | Faster builds, better caching |
| Docker Scout | 2023+ | Vulnerability scanning |
| Compose v2 | 2023+ | `docker compose` (no hyphen) |
| Wasm support | 2024+ | WebAssembly containers |
| Docker Desktop 4.x | 2024-2025 | Improved performance |

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `docker build -t name .` | Build image |
| `docker run -d -p 3000:3000 name` | Run container |
| `docker compose up -d` | Start services |
| `docker compose down -v` | Stop and remove |
| `docker exec -it container sh` | Shell into container |
| `docker logs -f container` | Follow logs |
| `docker system prune -a` | Clean up everything |
| `docker scout cves image` | Scan for vulnerabilities |

| Dockerfile Instruction | Purpose |
|------------------------|---------|
| `FROM` | Base image |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files |
| `RUN` | Execute command |
| `ENV` | Set environment variable |
| `EXPOSE` | Document port |
| `USER` | Set user |
| `HEALTHCHECK` | Define health check |
| `CMD` | Default command |
| `ENTRYPOINT` | Main executable |

| Base Image | Use Case |
|------------|----------|
| `alpine` | Minimal (5MB), most apps |
| `slim` | Smaller than full, more compatible |
| `distroless` | Ultra-minimal, no shell |
| `scratch` | Go binaries, nothing else |

---

## Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Anti-Patterns](https://codefresh.io/blog/docker-anti-patterns/)
- [Modern Docker Best Practices 2025](https://talent500.com/blog/modern-docker-best-practices-2025/)
- [Docker Security Best Practices](https://betterstack.com/community/guides/scaling-docker/docker-build-best-practices/)
