# Production Checklist — Pre-deployment Checklist Toàn Diện

> Checklist (Danh Sách Kiểm Tra) trước khi go-live — đảm bảo ứng dụng Node.js sẵn sàng cho production traffic. In checklist này và đánh dấu từng mục trước mỗi lần deploy quan trọng.

## Mục Lục

1. [Application Readiness](#application-readiness)
2. [Security Checklist](#security-checklist)
3. [Performance & Reliability](#performance--reliability)
4. [Observability](#observability)
5. [Infrastructure & Deployment](#infrastructure--deployment)
6. [Database & Data](#database--data)
7. [Operations & Runbooks](#operations--runbooks)
8. [Compliance & Documentation](#compliance--documentation)
9. [Go-Live Day Checklist](#go-live-day-checklist)
10. [Post-Deploy Verification](#post-deploy-verification)

---

## Application Readiness

### Code Quality

- [ ] Tất cả tests pass (unit, integration, e2e)
- [ ] Code coverage đạt threshold đã định (ví dụ: > 80% cho critical paths)
- [ ] Không có `console.log` debug statements trong production code
- [ ] ESLint/Prettier pass không có warnings
- [ ] TypeScript strict mode enabled, không có `any` không cần thiết
- [ ] Dependencies updated, `npm audit` không có critical/high vulnerabilities

### Configuration

- [ ] `NODE_ENV=production` được set
- [ ] Tất cả environment variables documented
- [ ] Không có hardcoded secrets trong source code
- [ ] Config validation khi app startup (fail fast nếu thiếu config)
- [ ] Feature flags configured cho features mới

```typescript
// Config validation example
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  REDIS_URL: z.string().url(),
});

const env = envSchema.parse(process.env); // Throws nếu invalid
```

### Error Handling

- [ ] Global error handler không leak stack traces ra client
- [ ] Unhandled promise rejection handler configured
- [ ] Uncaught exception handler configured
- [ ] Custom error classes với proper HTTP status codes
- [ ] Graceful shutdown handler (`SIGTERM`, `SIGINT`)

```typescript
process.on('unhandledRejection', (reason) => {
  logger.error({ err: reason }, 'Unhandled promise rejection');
  // Không exit ngay — log và monitor
});

process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception — shutting down');
  process.exit(1); // Uncaught exception → phải restart
});
```

---

## Security Checklist

### Authentication & Authorization

- [ ] JWT secrets đủ mạnh (≥ 256 bits), stored trong Secret manager
- [ ] Token expiration configured (access: 15min, refresh: 7 days)
- [ ] Rate limiting trên auth endpoints (login, register, password reset)
- [ ] RBAC (Role-Based Access Control — Phân Quyền Theo Vai Trò) implemented và tested
- [ ] CORS configured với whitelist origins (không dùng `*` trong production)

### HTTP Security

- [ ] Helmet.js enabled — security headers configured
- [ ] HTTPS enforced (HSTS header)
- [ ] Input validation trên tất cả endpoints (Zod/Joi)
- [ ] SQL injection prevention — parameterized queries only
- [ ] XSS prevention — output encoding, CSP headers
- [ ] Request size limits configured (`express.json({ limit: '1mb' })`)

### Secrets & Access

- [ ] Secrets trong K8s Secret / AWS Secrets Manager / Vault
- [ ] Không có secrets trong Docker image hoặc git history
- [ ] `.env` files trong `.gitignore` và `.dockerignore`
- [ ] Service accounts với least privilege
- [ ] API keys rotated và có expiration policy

### Dependency Security

- [ ] `npm audit` pass (no critical/high)
- [ ] Docker image scanned (Trivy/Snyk)
- [ ] Dependabot hoặc Renovate configured cho auto-updates
- [ ] Lock file (`package-lock.json`) committed

---

## Performance & Reliability

### Node.js Specific

- [ ] Event loop không bị block bởi sync operations
- [ ] `--max-old-space-size` set phù hợp với container memory limit
- [ ] Connection pooling configured cho database
- [ ] Redis connection pool configured
- [ ] Không có memory leaks (heap snapshot verified)
- [ ] CPU-intensive tasks offloaded (Worker Threads hoặc separate service)

### Caching

- [ ] Cache strategy defined (cache-aside, TTL)
- [ ] Cache invalidation logic tested
- [ ] Redis failover/reconnection handled

### Resilience

- [ ] Timeouts configured cho external API calls
- [ ] Retry logic với exponential backoff
- [ ] Circuit breaker cho downstream services
- [ ] Bulkhead pattern — resource isolation
- [ ] Idempotency keys cho critical operations (payments, orders)

```typescript
// Timeout example
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch(url, { signal: controller.signal });
} finally {
  clearTimeout(timeout);
}
```

### Load Testing

- [ ] Load test completed với k6/Artillery
- [ ] Breaking point documented (max RPS trước khi degrade)
- [ ] p95 latency < SLO target dưới expected load
- [ ] Error rate < 0.1% dưới normal load

---

## Observability

### Logging

- [ ] Structured logging (JSON) với Pino/Winston
- [ ] Log levels configured (`info` cho production)
- [ ] Request correlation ID (x-request-id) implemented
- [ ] Sensitive data redacted (passwords, tokens, PII)
- [ ] Log aggregation configured (Loki/ELK)
- [ ] Log retention policy defined

### Metrics

- [ ] `/metrics` endpoint với prom-client
- [ ] RED metrics: Rate, Errors, Duration
- [ ] Custom business metrics (orders, payments, etc.)
- [ ] Node.js default metrics (heap, event loop, GC)
- [ ] Prometheus scraping configured
- [ ] Grafana dashboards created

### Tracing

- [ ] OpenTelemetry instrumentation enabled
- [ ] Trace context propagated qua service calls
- [ ] Jaeger/Zipkin backend configured
- [ ] traceId included trong log fields

### Alerting

- [ ] Alert rules configured:
  - [ ] Error rate > threshold
  - [ ] p95 latency > threshold
  - [ ] Event loop lag > threshold
  - [ ] Memory usage > threshold
  - [ ] Pod restart count
- [ ] Alert routing configured (PagerDuty/Slack)
- [ ] On-call rotation defined
- [ ] Runbooks linked từ alerts

---

## Infrastructure & Deployment

### Docker

- [ ] Multi-stage Dockerfile
- [ ] Image size optimized (< 200MB)
- [ ] Non-root user trong container
- [ ] `.dockerignore` configured
- [ ] Health check trong Dockerfile
- [ ] Image tagged với git SHA (immutable)

### Kubernetes (nếu áp dụng)

- [ ] Deployment manifest với resource requests/limits
- [ ] Liveness, readiness, startup probes configured
- [ ] HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) configured
- [ ] PodDisruptionBudget defined
- [ ] ConfigMap và Secret separated
- [ ] `terminationGracePeriodSeconds: 30`
- [ ] Ingress với TLS configured
- [ ] Network policies (nếu cần isolation)

### CI/CD

- [ ] Pipeline: lint → test → build → scan → deploy
- [ ] Automated tests trong pipeline
- [ ] Docker image security scan
- [ ] Staging deployment trước production
- [ ] Smoke tests sau deploy
- [ ] Rollback procedure documented và tested
- [ ] Manual approval cho production deploy

### DNS & Networking

- [ ] DNS records configured
- [ ] SSL/TLS certificates valid và auto-renewal
- [ ] CDN configured (nếu cần)
- [ ] Firewall rules reviewed
- [ ] Load balancer health checks aligned với app health endpoints

---

## Database & Data

### Migrations

- [ ] Database migrations tested trên staging
- [ ] Migrations backward compatible (có thể rollback)
- [ ] Migration script trong CI/CD pipeline
- [ ] Backup trước khi chạy migration trên production

### Backup & Recovery

- [ ] Automated database backups configured
- [ ] Backup restoration tested (không chỉ backup!)
- [ ] RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) defined
- [ ] RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) defined
- [ ] Point-in-time recovery available

### Data Integrity

- [ ] Foreign key constraints enabled
- [ ] Database indexes cho frequent queries
- [ ] Connection pool size tuned
- [ ] Query timeout configured
- [ ] Read replicas configured (nếu read-heavy)

---

## Operations & Runbooks

### Documentation

- [ ] Architecture diagram updated
- [ ] API documentation (OpenAPI/Swagger) published
- [ ] Environment variables documented
- [ ] Deployment procedure documented
- [ ] Rollback procedure documented

### Runbooks

- [ ] **High error rate** — investigation steps
- [ ] **High latency** — profiling và scaling steps
- [ ] **Database connection issues** — pool exhaustion recovery
- [ ] **Memory leak** — heap snapshot analysis
- [ ] **Pod crash loop** — log analysis, resource check
- [ ] **Certificate expiry** — renewal procedure

### Incident Response

- [ ] On-call schedule defined
- [ ] Escalation path documented
- [ ] Incident communication template (status page, Slack)
- [ ] Post-mortem template prepared
- [ ] Blameless post-mortem culture established

---

## Compliance & Documentation

- [ ] Privacy policy updated (GDPR, data retention)
- [ ] Data encryption at rest và in transit
- [ ] Audit logging cho sensitive operations
- [ ] Access logs retained theo policy
- [ ] Third-party dependency licenses reviewed

---

## Go-Live Day Checklist

### Trước Deploy (T-1 giờ)

- [ ] Team notified về deploy window
- [ ] On-call engineer identified
- [ ] Rollback plan reviewed
- [ ] Database backup verified (recent và restorable)
- [ ] Monitoring dashboards open
- [ ] Communication channels ready (Slack war room)

### Trong Deploy

- [ ] Deploy lên staging — smoke test pass
- [ ] Database migration (nếu có) — verified
- [ ] Deploy lên production — rolling update
- [ ] Monitor error rate và latency real-time
- [ ] Verify health endpoints
- [ ] Test critical user flows manually

### Sau Deploy (T+30 phút)

- [ ] Error rate bình thường
- [ ] Latency trong SLO
- [ ] Không có alert mới
- [ ] Critical flows verified
- [ ] Team notified deploy thành công

---

## Post-Deploy Verification

```bash
# Quick verification script
#!/bin/bash
API_URL="${1:-https://api.example.com}"

echo "=== Health Check ==="
curl -sf "$API_URL/health" | jq .

echo "=== Readiness Check ==="
curl -sf "$API_URL/ready" | jq .

echo "=== Metrics Available ==="
curl -sf "$API_URL/metrics" | head -5

echo "=== Response Time ==="
curl -sf -w "Time: %{time_total}s\n" -o /dev/null "$API_URL/health"

echo "=== SSL Certificate ==="
echo | openssl s_client -connect api.example.com:443 2>/dev/null | openssl x509 -noout -dates
```

### Monitoring First 24 Hours

| Metric | Normal Range | Action nếu Abnormal |
| ------ | ------------ | ------------------- |
| Error rate | < 0.1% | Check logs, consider rollback |
| p95 latency | < SLO target | Profile, check DB/cache |
| CPU usage | < 70% | Review HPA settings |
| Memory usage | Stable, không tăng liên tục | Heap snapshot nếu tăng |
| Pod restarts | 0 | Check liveness probe, OOM |

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│              PRODUCTION DEPLOY QUICK REFERENCE           │
│                                                         │
│  Health:    GET /health  → 200                          │
│  Ready:     GET /ready   → 200                          │
│  Metrics:   GET /metrics → Prometheus format            │
│                                                         │
│  Rollback:  kubectl rollout undo deployment/nodejs-api  │
│  Logs:      kubectl logs -f deployment/nodejs-api       │
│  Scale:     kubectl scale deployment/nodejs-api --replicas=5 │
│                                                         │
│  On-call:   [PagerDuty/Slack channel]                   │
│  Runbooks:  [Wiki/Notion link]                          │
│  Dashboard: [Grafana URL]                               │
└─────────────────────────────────────────────────────────┘
```

---

**Lưu ý:** Checklist này là baseline — customize theo requirements cụ thể của dự án. Mỗi mục unchecked là một rủi ro tiềm ẩn trong production.

**Liên quan:**
- [1-docker-nodejs.md](./1-docker-nodejs.md) — Container setup
- [3-kubernetes-deployment.md](./3-kubernetes-deployment.md) — K8s manifests
- [4-logging.md](./4-logging.md) — Logging setup
- [5-metrics-tracing.md](./5-metrics-tracing.md) — Observability stack
- [6-cicd-pipeline.md](./6-cicd-pipeline.md) — Automated deployment
