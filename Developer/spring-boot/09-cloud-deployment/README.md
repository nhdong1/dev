# 09 — Triển Khai Đám Mây (Cloud Deployment)

> Hướng dẫn toàn diện về Docker hóa, triển khai Kubernetes, giám sát, cấu hình tập trung và CI/CD pipeline cho ứng dụng Spring Boot trong môi trường sản xuất.

---

## 📋 Mục Lục

| File | Chủ Đề | Mức Độ |
|------|--------|--------|
| [1-docker-spring.md](1-docker-spring.md) | Docker hóa Spring Boot — Dockerfile, Jib, Layer caching | ⭐⭐ |
| [2-kubernetes-deployment.md](2-kubernetes-deployment.md) | Kubernetes — Deployment, Service, ConfigMap, HPA, Probes | ⭐⭐⭐ |
| [3-spring-boot-actuator.md](3-spring-boot-actuator.md) | Spring Boot Actuator — Health, Metrics, Custom Endpoints | ⭐⭐ |
| [4-observability.md](4-observability.md) | Observability — Micrometer, Prometheus, Grafana, Zipkin, Jaeger | ⭐⭐⭐ |
| [5-spring-cloud-config.md](5-spring-cloud-config.md) | Spring Cloud Config — Config Server, Git backend, Refresh Scope | ⭐⭐⭐ |
| [6-cicd-pipeline.md](6-cicd-pipeline.md) | CI/CD Pipeline — GitHub Actions, Jenkins, Maven/Gradle CI | ⭐⭐ |

---

## 🎯 Tổng Quan

Triển khai ứng dụng Spring Boot lên môi trường đám mây đòi hỏi hiểu biết về toàn bộ vòng đời production:

```
Code → Build → Containerize → Deploy → Monitor → Scale → Repeat
```

### Các Thành Phần Cốt Lõi

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline                           │
│  GitHub Actions / Jenkins / GitLab CI                       │
└───────────────────────┬─────────────────────────────────────┘
                        │ Build & Push
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                 Container Registry                          │
│         Docker Hub / ECR / GCR / ACR                        │
└───────────────────────┬─────────────────────────────────────┘
                        │ Deploy
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐    │
│  │  Deployment │  │   Service   │  │ HPA (Autoscaler)│    │
│  │  (Pod x N)  │  │  (L4/L7)    │  │ Min/Max Replicas│    │
│  └─────────────┘  └─────────────┘  └─────────────────┘    │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │  ConfigMap  │  │   Secret    │                          │
│  │  (env cfg)  │  │  (password) │                          │
│  └─────────────┘  └─────────────┘                          │
└───────────────────────┬─────────────────────────────────────┘
                        │ Metrics/Traces/Logs
                        ▼
┌─────────────────────────────────────────────────────────────┐
│             Observability Stack                             │
│  Prometheus → Grafana (Metrics)                             │
│  Zipkin / Jaeger (Distributed Tracing — Theo Dõi Phân Tán) │
│  ELK Stack / Loki (Logs)                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 Luồng Triển Khai Điển Hình

### 1. Phát Triển Cục Bộ (Local Development)

```bash
# Chạy Spring Boot locally với profile dev
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Hoặc dùng Docker Compose cho toàn bộ stack
docker-compose up -d
```

### 2. Build Docker Image

```bash
# Cách 1: Dockerfile truyền thống
docker build -t my-app:1.0.0 .

# Cách 2: Jib (không cần Dockerfile, không cần Docker daemon)
./mvnw jib:build -Dimage=registry.example.com/my-app:1.0.0

# Cách 3: Spring Boot Buildpacks (Paketo Buildpacks)
./mvnw spring-boot:build-image
```

### 3. Deploy Lên Kubernetes

```bash
# Apply manifests
kubectl apply -f k8s/

# Kiểm tra trạng thái
kubectl rollout status deployment/my-app -n production

# Rollback nếu có vấn đề
kubectl rollout undo deployment/my-app -n production
```

### 4. Xác Nhận Health (Sức Khỏe)

```bash
# Kiểm tra Actuator health endpoint
curl https://api.example.com/actuator/health

# Kiểm tra metrics
curl https://api.example.com/actuator/prometheus
```

---

## 📦 Yêu Cầu Môi Trường

### Công Cụ Cần Cài Đặt

| Công Cụ | Phiên Bản Khuyến Nghị | Mục Đích |
|---------|----------------------|----------|
| Docker | 24+ | Container runtime (Môi Trường Chạy Container) |
| kubectl | 1.28+ | Kubernetes CLI |
| Helm | 3.x | Package manager cho K8s |
| Maven / Gradle | 3.9+ / 8.x | Build tool (Công Cụ Build) |
| Java | 17 / 21 LTS | Runtime |

### Docker Compose cho Lab Environment

```yaml
# docker-compose.yml — môi trường phát triển cục bộ
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

---

## 🔍 Concepts Cốt Lõi Cần Nắm

### Docker

| Khái Niệm | Giải Thích |
|-----------|------------|
| **Image** | Snapshot bất biến của ứng dụng và dependencies |
| **Container** | Instance đang chạy của một image |
| **Layer** | Mỗi lệnh Dockerfile tạo một layer — được cache riêng biệt |
| **Multi-stage Build** | Build trong nhiều giai đoạn để tối giảm kích thước image cuối |
| **Buildpack** | Công cụ tự động tạo image mà không cần viết Dockerfile |

### Kubernetes

| Khái Niệm | Giải Thích |
|-----------|------------|
| **Pod** | Đơn vị triển khai nhỏ nhất — chứa 1+ container |
| **Deployment** | Quản lý ReplicaSet, rolling update, rollback |
| **Service** | Trừu tượng hóa mạng, load balancing nội bộ |
| **ConfigMap** | Lưu trữ cấu hình không nhạy cảm |
| **Secret** | Lưu trữ dữ liệu nhạy cảm (mã hóa base64) |
| **HPA** | HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang |
| **Liveness Probe** | K8s kiểm tra Pod còn sống không — restart nếu fail |
| **Readiness Probe** | K8s kiểm tra Pod sẵn sàng nhận traffic không |
| **Ingress** | HTTP/HTTPS routing từ ngoài vào cluster |

### Observability (Khả Năng Quan Sát)

| Trụ Cột | Công Cụ | Mục Đích |
|---------|---------|----------|
| **Metrics (Chỉ Số)** | Micrometer → Prometheus → Grafana | Giám sát số liệu |
| **Traces (Dấu Vết)** | Micrometer Tracing → Zipkin / Jaeger | Theo dõi request xuyên services |
| **Logs (Nhật Ký)** | Logback → ELK Stack / Loki | Phân tích sự kiện |

---

## 🏗️ Kiến Trúc Tham Chiếu — Spring Boot Production

```
Internet
    │
    ▼
[Load Balancer / Ingress Controller]
    │
    ▼
[API Gateway Pod] ←→ [Service Discovery / Consul]
    │
    ├── [Auth Service Pod]
    │       └── Spring Security + JWT
    │
    ├── [User Service Pod x3]
    │       ├── Spring Boot App
    │       ├── HikariCP → PostgreSQL
    │       └── Redis Cache
    │
    └── [Order Service Pod x2]
            ├── Spring Boot App
            └── Kafka Producer/Consumer

Monitoring:
    ├── Actuator /health → Kubernetes Probes
    ├── Actuator /prometheus → Prometheus → Grafana
    └── Micrometer Tracing → Jaeger
```

---

## ✅ Checklist Production Readiness (Sẵn Sàng Production)

### Docker

- [ ] Dùng multi-stage build để tối giảm image size
- [ ] Chạy container với non-root user (người dùng không phải root)
- [ ] Không hardcode secrets trong Dockerfile
- [ ] Image được scan lỗ hổng bảo mật (Trivy / Snyk)
- [ ] Tag image theo semantic versioning (không dùng `latest` ở production)

### Kubernetes

- [ ] Khai báo `resources.requests` và `resources.limits` cho mỗi Pod
- [ ] Cấu hình `livenessProbe` và `readinessProbe`
- [ ] Dùng ConfigMap / Secret cho cấu hình
- [ ] Cấu hình HPA với CPU/memory metrics
- [ ] Khai báo PodDisruptionBudget (PDB — Ngân Sách Gián Đoạn Pod)

### Observability

- [ ] Spring Boot Actuator `/health` trả về thông tin chi tiết
- [ ] Metrics xuất ra Prometheus format
- [ ] Distributed tracing với correlation ID
- [ ] Structured logging (JSON) với trace ID/span ID
- [ ] Alerting (Cảnh Báo) được cấu hình trong Grafana

### CI/CD

- [ ] Pipeline chạy test trước khi build image
- [ ] Image được push kèm SHA commit làm tag
- [ ] Deployment có approval gate (cổng phê duyệt) cho production
- [ ] Rollback tự động khi health check thất bại
- [ ] Secrets được quản lý qua Vault / Kubernetes Secrets

---

## 🔗 Liên Kết Giữa Các Chủ Đề

```
Docker (1) ──────────────────────→ Kubernetes (2)
                                        │
                              ┌─────────┴──────────┐
                              ▼                    ▼
                       Actuator (3)          Config (5)
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
             Observability (4)      CI/CD (6)
```

- **Docker → K8s:** Image build từ Docker được deploy lên Kubernetes
- **Actuator → K8s Probes:** `/actuator/health` dùng cho liveness/readiness probe
- **Actuator → Observability:** `/actuator/prometheus` cấp metrics cho Prometheus
- **Config Server → K8s:** Kết hợp Spring Cloud Config với ConfigMap/Secret
- **CI/CD:** Orchestrate toàn bộ pipeline từ code đến production

---

## 📚 Tài Liệu Tham Khảo

- [Spring Boot Docker Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/container-images.html)
- [Spring Boot Actuator Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
