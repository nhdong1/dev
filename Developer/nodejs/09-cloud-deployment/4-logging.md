# Logging — Pino/Winston, Structured Logging và Log Aggregation

> Logging (Ghi Nhật Ký) là pillar đầu tiên của observability (khả năng quan sát) — khi production incident xảy ra, logs thường là nguồn thông tin đầu tiên để debug. Structured logging (ghi log có cấu trúc) biến text thành data có thể search, filter, và alert.

## Mục Lục

1. [Tại Sao Structured Logging](#tại-sao-structured-logging)
2. [Pino — High-performance Logger](#pino--high-performance-logger)
3. [Winston — Flexible Logger](#winston--flexible-logger)
4. [Log Levels và Best Practices](#log-levels-và-best-practices)
5. [Request Correlation ID](#request-correlation-id)
6. [Tích Hợp với Express/Fastify](#tích-hợp-với-expressfastify)
7. [Log Aggregation — ELK và Loki](#log-aggregation--elk-và-loki)
8. [Sensitive Data và Security](#sensitive-data-và-security)
9. [Production Patterns](#production-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Structured Logging

```
Unstructured (khó parse):
  [2024-01-15 10:30:00] User john@example.com login failed from 192.168.1.1

Structured JSON (machine-parseable):
  {
    "level": "warn",
    "time": 1705312200000,
    "msg": "login failed",
    "userId": "john@example.com",
    "ip": "192.168.1.1",
    "requestId": "abc-123",
    "service": "auth-api"
  }
```

| Unstructured | Structured |
| ------------ | ---------- |
| `grep` text patterns | Query by field: `userId="john"` |
| Khó aggregate | Count, group, percentile |
| Không correlate requests | Trace request qua services |
| Manual parsing | Auto-indexed trong ELK/Loki |

**Nguyên tắc:** Log ra **stdout/stderr** dạng JSON — container orchestrator và log shipper thu thập.

---

## Pino — High-performance Logger

Pino là logger nhanh nhất cho Node.js — thiết kế cho production với minimal overhead.

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // Production: JSON output
  // Development: pretty print
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  base: {
    service: 'api',
    env: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
  // Redact sensitive fields
  redact: ['req.headers.authorization', 'password', 'token'],
  timestamp: pino.stdTimeFunctions.isoTime,
});

export default logger;
```

### Sử Dụng Pino

```typescript
import logger from './logger';

// Simple log
logger.info('Server started');

// Structured với context
logger.info({ port: 3000, pid: process.pid }, 'Server listening');

// Error với stack trace
try {
  await riskyOperation();
} catch (err) {
  logger.error({ err, userId: '123' }, 'Operation failed');
}

// Child logger — inherit context
const requestLogger = logger.child({ requestId: 'abc-123' });
requestLogger.info('Processing order');
requestLogger.info({ orderId: '456' }, 'Order created');
```

### Performance

Pino sử dụng worker thread cho pretty print — production JSON logging có overhead rất thấp (~30% faster than Winston).

---

## Winston — Flexible Logger

Winston phù hợp khi cần nhiều transport (file, HTTP, cloud) và format tùy chỉnh.

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
  ),
  defaultMeta: { service: 'api' },
  transports: [
    new winston.transports.Console(),
    // Production: chỉ console — log shipper thu thập
  ],
});

// Development pretty print
if (process.env.NODE_ENV === 'development') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple(),
    ),
  }));
}

export default logger;
```

### Pino vs Winston

| Tiêu Chí | Pino | Winston |
| -------- | ---- | ------- |
| Performance | ⭐⭐⭐ Rất nhanh | ⭐⭐ Chậm hơn |
| Ecosystem | Fastify native | Express phổ biến |
| Transports | Ít, qua pino transports | Nhiều built-in |
| Pretty print | pino-pretty (dev) | winston.format |
| Khuyến nghị | Production mới | Legacy projects |

---

## Log Levels và Best Practices

```
Severity (tăng dần):
trace → debug → info → warn → error → fatal

Production thường dùng: info (default), warn, error
Development: debug hoặc trace
```

| Level | Khi Dùng | Ví Dụ |
| ----- | -------- | ----- |
| **trace** | Chi tiết debug sâu | Function entry/exit |
| **debug** | Development debugging | Query parameters, cache hits |
| **info** | Business events bình thường | User login, order created |
| **warn** | Bất thường nhưng không fail | Retry attempt, deprecated API |
| **error** | Lỗi cần investigate | DB connection failed, unhandled exception |
| **fatal** | App không thể tiếp tục | Config missing, port in use |

### Quy Tắc Logging

```typescript
// ❌ Log quá nhiều trong hot path
for (const item of items) {
  logger.debug({ item }, 'Processing item'); // 10,000 logs/request!
}

// ✅ Log summary
logger.info({ count: items.length, durationMs: 45 }, 'Batch processed');

// ❌ Log sensitive data
logger.info({ password: req.body.password }, 'Login attempt');

// ✅ Redact hoặc omit
logger.info({ email: req.body.email }, 'Login attempt');

// ❌ String concatenation
logger.info('User ' + userId + ' created order ' + orderId);

// ✅ Structured fields
logger.info({ userId, orderId }, 'Order created');
```

---

## Request Correlation ID

Correlation ID (ID Tương Quan) — trace một request xuyên suốt services và log entries.

```typescript
import { randomUUID } from 'node:crypto';
import type { Request, Response, NextFunction } from 'express';

export function correlationMiddleware(req: Request, res: Response, next: NextFunction) {
  const requestId = (req.headers['x-request-id'] as string) || randomUUID();
  req.requestId = requestId;
  res.setHeader('x-request-id', requestId);
  next();
}
```

### AsyncLocalStorage — Context Propagation

```typescript
import { AsyncLocalStorage } from 'node:async_hooks';
import pino from 'pino';

const asyncLocalStorage = new AsyncLocalStorage<{ requestId: string }>();

const baseLogger = pino({ /* config */ });

// Logger tự động inject requestId từ context
export const logger = new Proxy(baseLogger, {
  get(target, prop) {
    const store = asyncLocalStorage.getStore();
    if (store?.requestId && typeof target[prop] === 'function') {
      return (...args: unknown[]) => {
        if (typeof args[0] === 'object' && args[0] !== null) {
          args[0] = { ...args[0] as object, requestId: store.requestId };
        } else {
          args.unshift({ requestId: store.requestId });
        }
        return (target[prop] as Function).apply(target, args);
      };
    }
    return target[prop as keyof typeof target];
  },
});

// Middleware
export function contextMiddleware(req: Request, res: Response, next: NextFunction) {
  const requestId = req.headers['x-request-id'] as string || randomUUID();
  asyncLocalStorage.run({ requestId }, () => next());
}
```

Mọi log trong request handler tự động có `requestId` — không cần pass manually.

---

## Tích Hợp với Express/Fastify

### Express + Pino

```typescript
import express from 'express';
import pinoHttp from 'pino-http';
import logger from './logger';

const app = express();

app.use(pinoHttp({
  logger,
  genReqId: (req) => req.headers['x-request-id'] as string || randomUUID(),
  customLogLevel: (_req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      requestId: req.id,
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
  },
}));

// Trong route handler
app.get('/users/:id', async (req, res) => {
  req.log.info({ userId: req.params.id }, 'Fetching user');
  const user = await userService.findById(req.params.id);
  res.json(user);
});
```

### Fastify — Pino Built-in

```typescript
import Fastify from 'fastify';

const app = Fastify({
  logger: {
    level: 'info',
    serializers: {
      req: (req) => ({ method: req.method, url: req.url }),
    },
  },
});

app.get('/users/:id', async (request, reply) => {
  request.log.info({ userId: request.params.id }, 'Fetching user');
  return userService.findById(request.params.id);
});
```

Fastify tích hợp Pino sẵn — không cần middleware thêm.

---

## Log Aggregation — ELK và Loki

```
Application Pods
      │ stdout (JSON logs)
      ▼
┌─────────────┐
│ Log Shipper │  Fluent Bit / Promtail / Filebeat
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌──────┐ ┌──────┐
│ ELK  │ │ Loki │
│Stack │ │Stack │
└──┬───┘ └──┬───┘
   ▼        ▼
┌──────────────────┐
│ Kibana/Grafana   │  ← Search, dashboards, alerts
└──────────────────┘
```

### ELK Stack (Elasticsearch + Logstash + Kibana)

Phù hợp full-text search mạnh, phức tạp hơn về vận hành.

### Grafana Loki (Khuyến Nghị cho K8s)

Nhẹ hơn ELK, tích hợp tốt với Grafana và Prometheus.

```yaml
# promtail-config.yaml (snippet)
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

### Query Logs trong Loki (LogQL)

```logql
# Tất cả error logs từ api service
{app="nodejs-api"} |= "error"

# Filter by requestId
{app="nodejs-api"} | json | requestId="abc-123"

# Rate of errors per minute
rate({app="nodejs-api"} |= "error" [1m])
```

---

## Sensitive Data và Security

```typescript
const logger = pino({
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie',
      'password',
      'token',
      'creditCard',
      'ssn',
      '*.password',
      'body.secret',
    ],
    censor: '[REDACTED]',
  },
});
```

| Không Bao Giờ Log | Thay Thế |
| ----------------- | -------- |
| Passwords, tokens | `[REDACTED]` hoặc omit |
| Full credit card | Last 4 digits only |
| PII không cần thiết | Hash hoặc omit |
| Full request body | Chỉ log fields cần thiết |

---

## Production Patterns

### 1. Log Sampling cho High-traffic

```typescript
function shouldLog(sampleRate = 0.1): boolean {
  return Math.random() < sampleRate;
}

// Chỉ log 10% debug requests
if (shouldLog(0.1)) {
  logger.debug({ query }, 'DB query executed');
}
```

### 2. Error Logging với Context

```typescript
app.use((err: Error, req: Request, res: Response, _next: NextFunction) => {
  req.log.error({
    err,
    method: req.method,
    url: req.url,
    statusCode: res.statusCode,
  }, 'Unhandled error');

  res.status(500).json({ error: 'Internal Server Error', requestId: req.id });
});
```

### 3. Không Log ra File trong Container

```typescript
// ❌ Ghi file trong container — mất khi pod restart
logger.add(new winston.transports.File({ filename: 'app.log' }));

// ✅ stdout — orchestrator thu thập
// Chỉ dùng Console transport
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Đáp Án Ngắn |
| ------- | ----------- |
| Structured logging là gì? | Log dạng JSON với key-value fields — machine-parseable |
| Pino vs Winston? | Pino nhanh hơn, phù hợp production; Winston linh hoạt hơn về transports |
| Correlation ID dùng để làm gì? | Trace một request qua nhiều services và log entries |
| Tại sao log ra stdout? | Container pattern — orchestrator thu thập, không cần file management |
| ELK vs Loki? | ELK mạnh về full-text search; Loki nhẹ hơn, tích hợp Grafana |
| AsyncLocalStorage trong logging? | Propagate context (requestId) qua async calls mà không pass manually |
