# Docker Expert — Implementation Playbook

Complete, copy-paste ready configurations for production Docker setups.

---

## 1. Complete Project Setup: Python + Next.js + Postgres + Redis

### Directory structure
```
project/
├── backend/
│   ├── .dockerignore
│   ├── Dockerfile            # production (multi-stage)
│   ├── Dockerfile.dev        # development
│   └── requirements.txt
├── frontend/
│   ├── .dockerignore
│   ├── Dockerfile            # production (multi-stage standalone)
│   ├── Dockerfile.dev        # development
│   ├── next.config.ts        # requires output: "standalone"
│   └── package.json
├── docker-compose.dev.yml
├── docker-compose.prod.yml
└── .env.example              # template — .env is gitignored
```

### `backend/.dockerignore`
```
__pycache__/
*.py[cod]
*.pyo
venv/
.venv/
env/
*.egg-info/
dist/
build/
tests/
.pytest_cache/
htmlcov/
.coverage
*.log
logs/
seed.py
*.csv
*.sql
*.xlsx
backup_*.sql
.env
.env.*
.git/
.idea/
.vscode/
.DS_Store
README.md
MANUAL_TESTS.md
```

### `frontend/.dockerignore`
```
node_modules/
npm-debug.log*
.next/
out/
dist/
build/
*.tsbuildinfo
tests/
coverage/
.env
.env.*
.git/
.idea/
.vscode/
.DS_Store
*.md
```

### `backend/Dockerfile` (production — multi-stage)
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip wheel --wheel-dir /wheels -r requirements.txt

FROM python:3.11-slim AS runtime
WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /wheels /wheels
RUN pip install /wheels/* && rm -rf /wheels

COPY . .

RUN adduser --disabled-password --gecos "" appuser \
    && chown -R appuser /app
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### `backend/Dockerfile.dev`
```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### `frontend/Dockerfile` (production — standalone)
```dockerfile
FROM node:20-alpine AS base
WORKDIR /app

FROM base AS deps
RUN apk add --no-cache libc6-compat
COPY package.json package-lock.json ./
RUN npm ci

FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ARG NEXT_PUBLIC_ENVIRONMENT=production
ENV NEXT_PUBLIC_ENVIRONMENT=${NEXT_PUBLIC_ENVIRONMENT}
RUN npm run build

FROM base AS runner
ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs \
    && adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000 HOSTNAME="0.0.0.0"
CMD ["node", "server.js"]
```

Requires in `next.config.ts`:
```typescript
const nextConfig: NextConfig = {
  output: "standalone",
};
```

### `frontend/Dockerfile.dev`
```dockerfile
FROM node:20-alpine

RUN apk add --no-cache libc6-compat

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --legacy-peer-deps

COPY . .

EXPOSE 3000
CMD ["npm", "run", "dev"]
```

### `docker-compose.dev.yml`
```yaml
# Docker Compose — Development

services:
  postgres:
    image: postgres:16-alpine
    container_name: app-postgres-dev
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: app-redis-dev
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    container_name: app-backend-dev
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/app_dev
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET: dev-secret-NEVER-use-in-production
      ENVIRONMENT: development
      DEBUG: "true"
      ALLOWED_ORIGINS: http://localhost:3000
    volumes:
      - ./backend:/app
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    networks:
      - app-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    container_name: app-frontend-dev
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000/api/v1
      NODE_ENV: development
    volumes:
      - ./frontend:/app
      - /app/node_modules   # preserve image's node_modules from bind mount override
      - /app/.next          # preserve .next build cache
    depends_on:
      - backend
    command: npm run dev
    networks:
      - app-network

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### `docker-compose.prod.yml`
```yaml
# Docker Compose — Production

services:
  backend:
    image: ${REGISTRY_URL}/${PROJECT_NAME}/backend:latest
    container_name: app-backend
    restart: always
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      JWT_SECRET: ${JWT_SECRET}
      ENVIRONMENT: production
      ALLOWED_ORIGINS: ${ALLOWED_ORIGINS}
      FRONTEND_URL: ${FRONTEND_URL}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
    depends_on:
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    networks:
      - app-prod-network

  frontend:
    image: ${REGISTRY_URL}/${PROJECT_NAME}/frontend:latest
    container_name: app-frontend
    restart: always
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: ${FRONTEND_URL}/api/v1
      NODE_ENV: production
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    depends_on:
      backend:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 128M
    networks:
      - app-prod-network

  redis:
    image: redis:7-alpine
    container_name: app-redis
    restart: always
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-prod-network

volumes:
  redis_data:
    driver: local

networks:
  app-prod-network:
    driver: bridge
```

---

## 2. GitHub Actions — Build, Push & Deploy

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
  PROJECT_NAME: my-app

jobs:
  build-and-push:
    name: Build & Push Images
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Login to Registry
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | \
          docker login ${{ env.REGISTRY_URL }} \
          -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin

      - name: Build & push backend
        run: |
          docker build \
            -f backend/Dockerfile \
            -t $REGISTRY_URL/$PROJECT_NAME/backend:latest \
            -t $REGISTRY_URL/$PROJECT_NAME/backend:${{ github.sha }} \
            backend/
          docker push $REGISTRY_URL/$PROJECT_NAME/backend:latest
          docker push $REGISTRY_URL/$PROJECT_NAME/backend:${{ github.sha }}

      - name: Build & push frontend
        run: |
          docker build \
            -f frontend/Dockerfile \
            --build-arg NEXT_PUBLIC_API_URL=${{ secrets.NEXT_PUBLIC_API_URL }} \
            -t $REGISTRY_URL/$PROJECT_NAME/frontend:latest \
            -t $REGISTRY_URL/$PROJECT_NAME/frontend:${{ github.sha }} \
            frontend/
          docker push $REGISTRY_URL/$PROJECT_NAME/frontend:latest
          docker push $REGISTRY_URL/$PROJECT_NAME/frontend:${{ github.sha }}

  deploy:
    name: Deploy to VPS
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/app
            # Pull latest compose file
            git pull origin main
            # Login to registry
            echo "${{ secrets.REGISTRY_PASSWORD }}" | \
              docker login ${{ env.REGISTRY_URL }} \
              -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
            # Pull new images and recreate containers
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d
            # Cleanup old images
            docker image prune -f
```

**Required GitHub Secrets:**
```
REGISTRY_URL          # e.g., registry.magalu.cloud
REGISTRY_USERNAME
REGISTRY_PASSWORD
NEXT_PUBLIC_API_URL   # e.g., https://api.yourdomain.com
VPS_HOST              # VPS IP or hostname
VPS_USER              # e.g., ubuntu or deploy
VPS_SSH_KEY           # private key (public key added to VPS authorized_keys)
```

---

## 3. Environment Variable Management

### `.env.example` (committed to git)
```bash
# ============================
# Database
# ============================
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/app_dev

# ============================
# Redis
# ============================
REDIS_URL=redis://localhost:6379/0

# ============================
# JWT
# ============================
JWT_SECRET=change-me-in-production
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7

# ============================
# App
# ============================
ENVIRONMENT=development
ALLOWED_ORIGINS=http://localhost:3000
FRONTEND_URL=http://localhost:3000

# ============================
# Email (optional)
# ============================
RESEND_API_KEY=
EMAIL_FROM=noreply@example.com

# ============================
# Registry (CI/CD)
# ============================
REGISTRY_URL=
PROJECT_NAME=my-app
```

### `.gitignore` entries (mandatory)
```
.env
.env.local
.env.*.local
!.env.example    # keep the template
```

---

## 4. VPS Deployment Checklist

### Initial VPS setup
```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Docker Compose plugin
sudo apt-get install docker-compose-plugin

# Create app directory
mkdir -p /opt/app
cd /opt/app

# Clone repo
git clone https://github.com/org/repo .

# Create .env from example
cp .env.example .env
nano .env   # fill in production secrets
```

### First deploy
```bash
# Login to registry
docker login <registry-url>

# Pull images
docker compose -f docker-compose.prod.yml pull

# Start services
docker compose -f docker-compose.prod.yml up -d

# Check status
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs -f backend
```

### Monitoring
```bash
# Container health
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Resource usage
docker stats --no-stream

# Logs
docker logs app-backend --tail 100 -f
docker logs app-frontend --tail 50

# Disk usage
docker system df
```

### Safe cleanup (monthly)
```bash
# Safe: removes only unused/dangling
docker builder prune -a -f
docker image prune -f

# Check volumes before cleaning
docker volume ls
# NEVER: docker system prune --volumes (risk of data loss)
```

### Database backup (before any migration)
```bash
# Backup (from VPS or local)
docker exec <postgres-container> pg_dump -U postgres -d <db_name> \
  --no-password > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore
psql -U postgres -d <db_name> -f backup_YYYYMMDD_HHMMSS.sql
```

---

## 5. Registry Integration Patterns

### Magalu Container Registry (Brazil)
```bash
# Login
docker login registry.magalu.cloud -u <username> -p <token>

# Tag and push
docker tag myapp:latest registry.magalu.cloud/<namespace>/myapp:latest
docker push registry.magalu.cloud/<namespace>/myapp:latest
```

### GitHub Container Registry (GHCR)
```bash
# Login with PAT
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin

# Tag and push
docker tag myapp:latest ghcr.io/<org>/myapp:latest
docker push ghcr.io/<org>/myapp:latest
```

### Docker Hub
```bash
docker login -u <username>
docker tag myapp:latest <username>/myapp:latest
docker push <username>/myapp:latest
```

---

## 6. Quick Diagnostic Commands

```bash
# How big is my image, really?
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# What's eating space inside the image?
docker run --rm <image> sh -c 'du -hs /* 2>/dev/null | sort -rh | head -20'

# Layer-by-layer breakdown
docker history <image> --format "table {{.Size}}\t{{.CreatedBy}}" | head -20

# Is my .dockerignore working?
# Build with --progress=plain and look for "sending build context" line
docker build --progress=plain -t test . 2>&1 | grep "build context"

# Validate compose files
docker compose -f docker-compose.dev.yml config --quiet && echo "OK"
docker compose -f docker-compose.prod.yml config --quiet && echo "OK"

# Force rebuild without cache (get accurate timing)
docker compose build --no-cache backend frontend

# Check which ports are in use
docker ps --format "table {{.Names}}\t{{.Ports}}"
```
