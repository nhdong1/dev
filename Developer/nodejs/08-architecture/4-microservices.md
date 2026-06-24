# Microservices — Service Decomposition, API Gateway và Service Mesh

> Microservices (Kiến Trúc Vi Dịch Vụ) chia ứng dụng thành các services độc lập, deploy và scale riêng. Chủ đề này cover **khi nào nên/không nên**, **decomposition strategies (chiến lược phân tách)**, và patterns vận hành với Node.js.

## Mục Lục

1. [Microservices Là Gì](#microservices-là-gì)
2. [Monolith vs Microservices](#monolith-vs-microservices)
3. [Decomposition Strategies](#decomposition-strategies)
4. [Communication Patterns](#communication-patterns)
5. [API Gateway](#api-gateway)
6. [Service Discovery và Load Balancing](#service-discovery-và-load-balancing)
7. [Data Management trong Microservices](#data-management-trong-microservices)
8. [Node.js Considerations](#nodejs-considerations)
9. [Migration Path — Monolith to Microservices](#migration-path--monolith-to-microservices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Microservices Là Gì

```
┌─────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                              │
│              Auth, Rate Limit, Routing, Aggregation              │
└───────┬─────────────┬─────────────┬─────────────┬───────────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ User Service│ │Order Service│ │Payment Svc  │ │Notification │
│  (Node.js)  │ │  (Node.js)  │ │  (Node.js)  │ │  (Node.js)  │
│  PostgreSQL │ │  PostgreSQL │ │   Stripe    │ │  SendGrid   │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
        │             │             │
        └─────────────┴─────────────┴── Message Broker (Kafka/RabbitMQ)
```

**Đặc điểm cốt lõi:**

| Đặc Điểm | Ý Nghĩa |
| -------- | ------- |
| **Independent deployability** | Deploy một service không cần deploy toàn bộ |
| **Decentralized data** | Mỗi service sở hữu database riêng |
| **Technology diversity** | Service A dùng Node.js, Service B dùng Go — nếu cần |
| **Organized around business capabilities** | Team ownership theo domain |

---

## Monolith vs Microservices

| Tiêu Chí | Modular Monolith | Microservices |
| -------- | ---------------- | ------------- |
| **Complexity** | Thấp — một codebase, một deploy | Cao — distributed systems problems |
| **Team size** | 1–10 devs | 10+ devs, multiple teams |
| **Deploy** | All-or-nothing | Independent per service |
| **Scale** | Scale toàn bộ app | Scale service cần thiết |
| **Debugging** | Stack trace trong một process | Distributed tracing cần thiết |
| **Data consistency** | ACID transactions dễ | Distributed transactions khó |
| **Latency** | In-process calls | Network overhead |

### Khi NÀO Dùng Microservices

- Team lớn cần **autonomous deployment**
- Phần hệ thống cần **scale khác nhau** (ví dụ: search vs billing)
- **Different SLAs** cho subsystems
- Đã có **operational maturity** (CI/CD, monitoring, on-call)

### Khi KHÔNG Nên

- Startup/MVP — "premature microservices"
- Team nhỏ không có DevOps capacity
- Chưa hiểu domain boundaries
- "Because Netflix/Google does it"

> **Rule of thumb:** Bắt đầu modular monolith. Extract microservice khi có **pain point cụ thể** chứng minh cần tách.

---

## Decomposition Strategies

### 1. By Business Capability (Theo Năng Lực Nghiệp Vụ)

```
E-commerce:
├── Catalog Service    — products, categories, search
├── Cart Service       — shopping cart, wishlist
├── Order Service      — order lifecycle
├── Payment Service    — billing, refunds
├── User Service       — auth, profiles
└── Notification Service — email, SMS, push
```

### 2. By Subdomain — DDD Bounded Context

Áp dụng **Domain-Driven Design (Thiết Kế Hướng Miền)**:

| Bounded Context | Ubiquitous Language | Service |
| --------------- | ------------------- | ------- |
| Sales | Order, LineItem, Checkout | Order Service |
| Inventory | Stock, Warehouse, Reservation | Inventory Service |
| Shipping | Shipment, Tracking, Carrier | Fulfillment Service |

### 3. Strangler Fig Pattern — Migration Từ Từ

Thay thế monolith incrementally:

```
Phase 1: Monolith + API Gateway
         All traffic → Monolith

Phase 2: Extract Notification Service
         Gateway routes /notifications/* → New Service
         Everything else → Monolith

Phase 3: Extract Order Service
         ...

Phase N: Monolith retired
```

```typescript
// API Gateway routing — strangler fig
app.use('/api/v1/notifications', proxy('http://notification-service:3001'));
app.use('/api/v1/orders', proxy('http://order-service:3002'));
app.use('/api/v1', proxy('http://legacy-monolith:3000')); // fallback
```

---

## Communication Patterns

### Synchronous — HTTP/REST hoặc gRPC

```
Order Service ──HTTP POST──► Payment Service
                ◄── 200 OK ──
```

**Pros:** Simple, request-response natural
**Cons:** Tight coupling, cascading failures, latency chain

```typescript
// Order service gọi Payment service
async function placeOrder(order: Order) {
  const response = await fetch('http://payment-service/charges', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'X-Request-Id': requestId },
    body: JSON.stringify({ amount: order.total, customerId: order.customerId }),
    signal: AbortSignal.timeout(5000),
  });

  if (!response.ok) throw new PaymentFailedError(await response.text());
  return response.json();
}
```

### Resilience Patterns

| Pattern | Mục Đích |
| ------- | -------- |
| **Timeout** | Không chờ vô hạn |
| **Retry with backoff** | Transient failures |
| **Circuit Breaker (Cầu Dao)** | Stop calling failing service |
| **Bulkhead** | Isolate failures — không drain toàn bộ thread pool |

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(callPaymentService, {
  timeout: 5000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
});

breaker.fallback(() => ({ status: 'PENDING', message: 'Payment queued for retry' }));
```

### Asynchronous — Message Broker

```
Order Service ──publish──► Kafka ──subscribe──► Notification Service
                           │
                           └──subscribe──► Inventory Service
```

**Pros:** Loose coupling, buffering, eventual consistency
**Cons:** Complexity, debugging harder, idempotency required

---

## API Gateway

**API Gateway (Cổng API)** là single entry point cho clients — handle cross-cutting concerns:

```
Client ──► API Gateway ──► Internal Services
              │
              ├── Authentication / JWT validation
              ├── Rate limiting
              ├── Request routing
              ├── Response aggregation (BFF pattern)
              └── SSL termination
```

### BFF — Backend for Frontend

Tách gateway theo client type:

| BFF | Client | Aggregation |
| --- | ------ | ----------- |
| Mobile BFF | iOS/Android app | Lightweight payloads |
| Web BFF | SPA browser | Full data + SEO metadata |
| Partner BFF | Third-party API | API key auth, limited fields |

```typescript
// Web BFF — aggregate multiple services
app.get('/api/web/dashboard', async (req, res) => {
  const [user, orders, recommendations] = await Promise.all([
    fetch(`${USER_SERVICE}/users/${req.userId}`),
    fetch(`${ORDER_SERVICE}/orders?userId=${req.userId}&limit=5`),
    fetch(`${CATALOG_SERVICE}/recommendations/${req.userId}`),
  ]);

  res.json({
    user: await user.json(),
    recentOrders: await orders.json(),
    recommendations: await recommendations.json(),
  });
});
```

### Tools Phổ Biến

| Tool | Đặc Điểm |
| ---- | -------- |
| **Kong** | Plugin ecosystem, K8s native |
| **AWS API Gateway** | Serverless, AWS integration |
| **Express/Fastify custom** | Full control, Node.js native |
| **Traefik** | Dynamic routing, Let's Encrypt |

---

## Service Discovery và Load Balancing

Trong Kubernetes, **Service Discovery (Khám Phá Dịch Vụ)** built-in:

```yaml
# K8s Service — DNS: order-service.default.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 3000
```

Node.js gọi service qua DNS name — K8s load balance across pods.

**Service Mesh (Lưới Dịch Vụ)** — Istio, Linkerd: sidecar proxy handle retries, mTLS, observability transparently.

---

## Data Management trong Microservices

### Database per Service

Mỗi service **sở hữu data** — không share database trực tiếp:

```
❌ Order Service ──JOIN──► User table (shared DB)
✅ Order Service ──API───► User Service (get user info)
✅ Order Service stores userId + cached userName snapshot
```

### Challenges

| Challenge | Giải Pháp |
| --------- | --------- |
| Cross-service queries | API composition, CQRS read models |
| Distributed transactions | Saga pattern (xem [5-event-driven.md](./5-event-driven.md)) |
| Data duplication | Event-driven sync, accept eventual consistency |
| Schema changes | Version events, backward compatible APIs |

---

## Node.js Considerations

| Constraint | Impact trên Microservices |
| ---------- | ------------------------- |
| **Single-threaded** | Mỗi instance handle I/O well — scale horizontally |
| **CPU-bound tasks** | Offload to Worker Threads hoặc separate service |
| **Memory per instance** | Smaller services = smaller memory footprint per pod |
| **Cold start (serverless)** | Keep services warm hoặc dùng containers |

### Service Template

```
order-service/
├── src/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── presentation/
├── Dockerfile
├── package.json
├── docker-compose.yml    # local dev với dependencies
└── k8s/
    ├── deployment.yaml
    └── service.yaml
```

---

## Migration Path — Monolith to Microservices

```
Step 1: Modular Monolith
        └── Clear module boundaries, no cross-module DB access

Step 2: Extract least coupled module
        └── Notification (email) — async, no critical path

Step 3: API Gateway + routing
        └── Strangler fig — route new paths to new service

Step 4: Event-driven integration
        └── Replace sync calls với events where possible

Step 5: Extract core domains incrementally
        └── Order, Payment — higher risk, need sagas

Step 6: Retire monolith modules
```

**Anti-pattern:** "Big bang" rewrite sang microservices — thường fail.

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Microservices vs monolith trade-offs? | Microservices: scale/deploy independence vs operational complexity |
| Khi nào tách service? | Team autonomy, scale mismatch, clear bounded context — not prematurely |
| API Gateway làm gì? | Single entry, auth, routing, rate limit, aggregation |
| Database per service? | Mỗi service owns data — no shared tables, sync via API/events |
| Distributed transaction? | Avoid 2PC — use saga pattern with compensating actions |
| Circuit breaker? | Stop calling failing service, fail fast, auto recovery |
| Strangler fig? | Incrementally replace monolith — route traffic gradually |
| Node.js fit microservices? | Yes for I/O-bound services — horizontal scale, small focused services |

---

**Tiếp theo:** [5-event-driven.md](./5-event-driven.md) — Event sourcing, CQRS, và saga pattern
