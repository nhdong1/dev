# Triển Khai & Observability — Tổng Quan

> Hướng dẫn đưa ứng dụng Node.js từ development lên production — containerization (đóng gói container), orchestration (điều phối), observability (khả năng quan sát hệ thống), và CI/CD (Continuous Integration/Continuous Deployment — Tích Hợp/Triển Khai Liên Tục).

## Mục Lục

1. [Tại Sao Triển Khai & Observability Quan Trọng](#tại-sao-triển-khai--observability-quan-trọng)
2. [Production Stack Cho Node.js](#production-stack-cho-nodejs)
3. [Deployment Workflow — Quy Trình Triển Khai](#deployment-workflow--quy-trình-triển-khai)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Pre-Deploy Checklist Tóm Tắt](#pre-deploy-checklist-tóm-tắt)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Triển Khai & Observability Quan Trọng

Code chạy tốt trên máy local ≠ chạy ổn định trong production. Các incident phổ biến khi thiếu deployment discipline:

| Triệu Chứng | Nguyên Nhân Thường Gặp |
| ----------- | ---------------------- |
| App crash sau deploy | Thiếu health checks, env vars sai, memory limit thấp |
| Không biết lỗi ở đâu | Logging không structured, không có correlation ID |
| Downtime khi scale | Không có rolling update, thiếu readiness probe |
| Secret leak | Hardcode credentials trong image hoặc repo |
| Rollback chậm | Không có automated rollback, không tag image versions |

**Nguyên tắc cốt lõi:** Production-ready = **reproducible builds (build tái tạo được)** + **observable systems (hệ thống quan sát được)** + **safe deployments (triển khai an toàn)**.

---

## Production Stack Cho Node.js

```
┌─────────────────────────────────────────────────────────────────┐
│                    NODE.JS PRODUCTION STACK                      │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────────┐  │
│  │  Docker  │───►│ PM2/K8s  │───►│  Load Balancer / Ingress │  │
│  │  Image   │    │ Runtime  │    │  (ALB, Nginx, Traefik)   │  │
│  └──────────┘    └──────────┘    └──────────────────────────┘  │
│       │               │                      │                  │
│       ▼               ▼                      ▼                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              OBSERVABILITY LAYER                         │   │
│  │  Logs (Pino/Loki) │ Metrics (Prometheus) │ Traces (OTel) │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              CI/CD PIPELINE                              │   │
│  │  Test → Build → Scan → Deploy → Smoke Test → Rollback   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

| Layer | Công Cụ Phổ Biến | Vai Trò |
| ----- | ---------------- | ------- |
| **Container** | Docker, BuildKit | Đóng gói app + dependencies, đảm bảo consistency |
| **Process Manager** | PM2 | Single-server deployment, cluster mode |
| **Orchestration** | Kubernetes (K8s), ECS, Fly.io | Multi-node scaling, self-healing |
| **Logging** | Pino, Winston, Loki, ELK | Debug incidents, audit trail |
| **Metrics** | Prometheus, Grafana | SLI/SLO monitoring, alerting |
| **Tracing** | OpenTelemetry, Jaeger | Distributed request flow |
| **CI/CD** | GitHub Actions, GitLab CI | Automated test & deploy |

---

## Deployment Workflow — Quy Trình Triển Khai

```
1. CONTAINERIZE           → Multi-stage Dockerfile, .dockerignore
        ↓
2. CONFIG MANAGEMENT      → env vars, ConfigMap, Secret (không hardcode)
        ↓
3. HEALTH CHECKS          → /health, /ready — liveness & readiness probes
        ↓
4. OBSERVABILITY          → Structured logs, /metrics, tracing middleware
        ↓
5. CI/CD PIPELINE         → Test → Build → Deploy → Smoke test
        ↓
6. PRODUCTION CHECKLIST   → Security, performance, rollback plan
        ↓
7. MONITOR & ITERATE      → Alerts, dashboards, post-deploy review
```

---

## Lộ Trình Học Trong Chủ Đề

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-docker-nodejs.md](./1-docker-nodejs.md) | Multi-stage build, layer caching, security | 1–1.5 giờ |
| 2 | [2-pm2-process-manager.md](./2-pm2-process-manager.md) | Cluster mode, zero-downtime reload | 45 phút |
| 3 | [3-kubernetes-deployment.md](./3-kubernetes-deployment.md) | Deployment, Service, HPA, probes | 1.5–2 giờ |
| 4 | [4-logging.md](./4-logging.md) | Pino/Winston, structured logging, ELK/Loki | 1 giờ |
| 5 | [5-metrics-tracing.md](./5-metrics-tracing.md) | Prometheus, Grafana, OpenTelemetry | 1–1.5 giờ |
| 6 | [6-cicd-pipeline.md](./6-cicd-pipeline.md) | GitHub Actions, automated rollback | 1 giờ |
| 7 | [7-production-checklist.md](./7-production-checklist.md) | Pre-deployment checklist đầy đủ | 30 phút |

**Tổng thời gian ước tính:** 6–8 giờ

---

## Các Tài Liệu Chi Tiết

### [1. Docker cho Node.js](./1-docker-nodejs.md)

Multi-stage Dockerfile, `.dockerignore`, layer caching, non-root user, image size optimization.

### [2. PM2 Process Manager](./2-pm2-process-manager.md)

Cluster mode, ecosystem file, graceful shutdown, zero-downtime reload, log rotation.

### [3. Kubernetes Deployment](./3-kubernetes-deployment.md)

Deployment, Service, ConfigMap, Secret, HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang), liveness/readiness/startup probes.

### [4. Logging](./4-logging.md)

Structured logging với Pino/Winston, log levels, correlation ID, log aggregation (ELK, Loki).

### [5. Metrics & Tracing](./5-metrics-tracing.md)

Prometheus metrics, Grafana dashboards, OpenTelemetry instrumentation, distributed tracing.

### [6. CI/CD Pipeline](./6-cicd-pipeline.md)

GitHub Actions workflow, build & deploy stages, automated rollback strategies.

### [7. Production Checklist](./7-production-checklist.md)

Checklist toàn diện trước khi go-live — security, performance, observability, operations.

---

## Bài Tập Thực Hành

### Bài 1: Dockerize REST API

```
1. Viết multi-stage Dockerfile cho Express/Fastify app
2. Thêm health check endpoint /health
3. Chạy container với docker compose (app + PostgreSQL + Redis)
4. Verify: curl http://localhost:3000/health
```

### Bài 2: PM2 Cluster Mode

```
1. Deploy app với PM2 ecosystem.config.js
2. Test cluster mode với 4 instances
3. Thực hiện pm2 reload và verify zero downtime
4. Cấu hình log rotation
```

### Bài 3: Observability Stack

```
1. Tích hợp Pino structured logging
2. Expose /metrics endpoint với prom-client
3. Thêm OpenTelemetry tracing cho HTTP requests
4. Chạy local: Prometheus + Grafana với docker compose
```

### Bài 4: CI/CD Pipeline

```
1. Tạo GitHub Actions workflow: lint → test → build → push image
2. Deploy lên staging environment
3. Thêm smoke test sau deploy
4. Cấu hình rollback khi smoke test fail
```

---

## Pre-Deploy Checklist Tóm Tắt

- [ ] Multi-stage Dockerfile, image < 200MB
- [ ] Không có secrets trong image hoặc source code
- [ ] Health check endpoints (`/health`, `/ready`)
- [ ] Graceful shutdown (`SIGTERM` handler)
- [ ] Structured logging với request ID
- [ ] Metrics endpoint (`/metrics`) hoặc APM agent
- [ ] CI/CD pipeline với automated tests
- [ ] Rollback plan đã document
- [ ] Resource limits (CPU, memory) đã set
- [ ] Environment variables qua ConfigMap/Secret

Chi tiết đầy đủ: [7-production-checklist.md](./7-production-checklist.md)

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Multi-stage Dockerfile khác gì single-stage? | Tách build stage và runtime stage — image nhỏ hơn, không chứa devDependencies |
| PM2 cluster mode hoạt động thế nào? | Fork nhiều worker processes, load balance qua OS, tận dụng multi-core |
| Liveness vs Readiness probe? | Liveness: restart pod nếu dead; Readiness: ngừng route traffic khi chưa sẵn sàng |
| Tại sao dùng structured logging? | Machine-parseable JSON → dễ search, filter, alert trong log aggregation system |
| Three pillars of observability? | Logs, Metrics, Traces — bổ sung cho nhau để debug production issues |
| Zero-downtime deployment strategy? | Rolling update, blue-green, hoặc canary — kết hợp readiness probe |
| Graceful shutdown trong Node.js? | Listen SIGTERM → stop accepting requests → drain connections → exit |

---

**Tiếp theo:** Bắt đầu với [1-docker-nodejs.md](./1-docker-nodejs.md) — nền tảng cho mọi deployment strategy.
