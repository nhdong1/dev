# Docker cho Node.js — Multi-stage Dockerfile, Layer Caching và Security

> Docker (Đóng Gói Container) là bước đầu tiên để đảm bảo ứng dụng Node.js chạy nhất quán từ development đến production — cùng runtime, cùng dependencies, cùng environment.

## Mục Lục

1. [Tại Sao Dockerize Node.js App](#tại-sao-dockerize-nodejs-app)
2. [Cấu Trúc Dockerfile Cơ Bản](#cấu-trúc-dockerfile-cơ-bản)
3. [Multi-stage Build](#multi-stage-build)
4. [Layer Caching — Tối Ưu Build Time](#layer-caching--tối-ưu-build-time)
5. [.dockerignore](#dockerignore)
6. [Security Best Practices](#security-best-practices)
7. [Health Check và Graceful Shutdown](#health-check-và-graceful-shutdown)
8. [Docker Compose cho Local Development](#docker-compose-cho-local-development)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Dockerize Node.js App

| Vấn Đề Không Docker | Giải Pháp Docker |
| ------------------- | ---------------- |
| "Works on my machine" | Reproducible environment |
| Dependency conflicts | Isolated container filesystem |
| Manual deploy steps | `docker run` hoặc orchestrator pull image |
| Khó scale horizontal | Container là unit of deployment |

```
Development Machine          CI/CD Server              Production
      │                           │                         │
      ▼                           ▼                         ▼
┌──────────┐              ┌──────────┐              ┌──────────┐
│  docker  │   ──push──►  │ Registry │  ──pull──►   │  K8s/VM  │
│  build   │              │ (GHCR/ECR)│              │  runtime │
└──────────┘              └──────────┘              └──────────┘
       Same image SHA everywhere ✅
```

---

## Cấu Trúc Dockerfile Cơ Bản

### Single-stage (Không Khuyến Nghị cho Production)

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**Vấn đề:** Image chứa cả devDependencies, source TypeScript, build tools → image lớn, attack surface rộng.

---

## Multi-stage Build

Tách **build stage** (biên dịch, cài devDependencies) và **production stage** (chỉ runtime artifacts).

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────
FROM node:22-alpine AS builder

WORKDIR /app

# Copy dependency manifests trước — tận dụng layer cache
COPY package.json package-lock.json ./

RUN npm ci

COPY . .

RUN npm run build

# Prune devDependencies sau khi build
RUN npm prune --production

# ── Stage 2: Production ─────────────────────────────────────
FROM node:22-alpine AS production

# Security: chạy với non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Chỉ copy artifacts cần thiết từ builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

USER nodejs

ENV NODE_ENV=production

EXPOSE 3000

# HEALTHCHECK — Docker native health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/main.js"]
```

### So Sánh Image Size

| Approach | Kích Thước Thường Gặp |
| -------- | --------------------- |
| Single-stage (full node) | 800MB – 1.2GB |
| Multi-stage alpine | 80MB – 200MB |
| Multi-stage + distroless | 50MB – 120MB |

### Distroless Image (Nâng Cao)

```dockerfile
FROM gcr.io/distroless/nodejs22-debian12

COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist

CMD ["dist/main.js"]
```

**Distroless** — image không có shell, package manager → giảm attack surface tối đa.

---

## Layer Caching — Tối Ưu Build Time

Docker cache layers theo thứ tự. **Thay đổi layer = invalidate tất cả layers phía sau.**

```
Thứ tự tối ưu:
1. COPY package.json package-lock.json    ← ít thay đổi → cache hit cao
2. RUN npm ci                             ← chỉ rebuild khi deps thay đổi
3. COPY source code                       ← thay đổi thường xuyên
4. RUN npm run build
```

### BuildKit Cache Mount (Docker BuildKit)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./

# Cache npm directory giữa các lần build
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .
RUN npm run build
```

```bash
# Enable BuildKit
DOCKER_BUILDKIT=1 docker build -t my-api:latest .
```

---

## .dockerignore

Loại trừ files không cần thiết khỏi build context — giảm context size, tăng tốc build.

```dockerignore
node_modules
npm-debug.log
dist
.git
.gitignore
.env
.env.*
*.md
coverage
.nyc_output
.vscode
.idea
Dockerfile*
docker-compose*
**/*.test.ts
**/*.spec.ts
```

**Quan trọng:** Không bao giờ copy `.env` vào image — dùng environment variables hoặc secrets manager tại runtime.

---

## Security Best Practices

| Practice | Lý Do |
| -------- | ----- |
| **Dùng alpine hoặc distroless** | Image nhỏ, ít CVE hơn |
| **Non-root user** | Giảm blast radius nếu container bị compromise |
| **Pin base image version** | `node:22.4.0-alpine` thay vì `node:latest` |
| **npm ci thay vì npm install** | Reproducible installs từ lock file |
| **Scan image** | `docker scout cves` hoặc Trivy trong CI |
| **Không chứa secrets** | Dùng K8s Secret, AWS Secrets Manager |
| **Set NODE_ENV=production** | Tắt debug features, optimize performance |

### Scan Image trong CI

```bash
# Trivy vulnerability scanner
trivy image --severity HIGH,CRITICAL my-api:latest
```

---

## Health Check và Graceful Shutdown

### Health Endpoint trong App

```typescript
import express from 'express';

const app = express();

// Liveness — process còn sống
app.get('/health', (_req, res) => {
  res.status(200).json({ status: 'ok', uptime: process.uptime() });
});

// Readiness — sẵn sàng nhận traffic (DB connected, cache warm)
app.get('/ready', async (_req, res) => {
  try {
    await db.ping(); // kiểm tra dependency
    res.status(200).json({ status: 'ready' });
  } catch {
    res.status(503).json({ status: 'not ready' });
  }
});
```

### Graceful Shutdown

```typescript
const server = app.listen(3000);

function shutdown(signal: string) {
  console.log(`${signal} received — shutting down gracefully`);
  server.close(() => {
    console.log('HTTP server closed');
    db.disconnect().then(() => process.exit(0));
  });

  // Force exit sau 30 giây
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30_000);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

**SIGTERM** — signal Kubernetes gửi trước khi terminate pod. App phải handle để drain connections.

---

## Docker Compose cho Local Development

```yaml
# docker-compose.yml
services:
  api:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
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
  pgdata:
```

```bash
docker compose up -d --build
docker compose logs -f api
```

---

## Best Practices

1. **Pin Node.js LTS version** — `node:22-alpine`, theo dõi [Node.js release schedule](https://nodejs.org/en/about/previous-releases)
2. **Một process per container** — không dùng PM2 trong Docker (K8s/ECS handle scaling)
3. **Immutable tags** — dùng git SHA làm image tag, không chỉ `latest`
4. **Resource limits** — set memory/CPU limits trong orchestrator
5. **Log to stdout/stderr** — container orchestrator thu thập logs
6. **Không dùng `npm start` trong CMD** — gọi `node` trực tiếp để nhận signals đúng

```dockerfile
# ❌ npm start có thể không forward SIGTERM
CMD ["npm", "start"]

# ✅ node nhận signals trực tiếp
CMD ["node", "dist/main.js"]
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Đáp Án Ngắn |
| ------- | ----------- |
| Multi-stage build lợi ích gì? | Image nhỏ hơn, không chứa build tools và devDependencies |
| Tại sao COPY package.json trước source code? | Tận dụng Docker layer cache — deps ít thay đổi hơn source |
| PM2 trong Docker container có nên dùng không? | Không — một process per container, scaling do orchestrator |
| NODE_ENV=production ảnh hưởng gì? | Express/Fastify tắt verbose errors, npm prune devDeps |
| Graceful shutdown quan trọng thế nào? | Tránh drop in-flight requests khi rolling update |
| .dockerignore khác .gitignore? | Tương tự mục đích nhưng cho Docker build context |
