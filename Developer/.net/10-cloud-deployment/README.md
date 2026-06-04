# Cloud & Triển Khai — Cloud & Deployment

> Hướng dẫn toàn diện về đóng gói, triển khai và vận hành ứng dụng .NET trên môi trường cloud — đám mây: từ Docker container đến Kubernetes cluster, Azure services, CI/CD pipeline và observability.

---

## 📚 Mục Lục

| File | Chủ Đề | Tóm Tắt |
|------|---------|---------|
| [1-docker-dotnet.md](1-docker-dotnet.md) | Docker cho .NET | Dockerfile, multi-stage build, image optimization |
| [2-kubernetes-deployment.md](2-kubernetes-deployment.md) | Kubernetes Deployment | Deployment, Service, ConfigMap, Secret, HPA |
| [3-azure-services.md](3-azure-services.md) | Azure Services | App Service, AKS, Azure Functions, Azure SQL |
| [4-cicd-pipeline.md](4-cicd-pipeline.md) | CI/CD Pipeline | GitHub Actions, Azure DevOps cho .NET |
| [5-configuration-management.md](5-configuration-management.md) | Quản Lý Cấu Hình | Secrets, biến môi trường, Azure Key Vault |
| [6-observability.md](6-observability.md) | Observability | OpenTelemetry, Serilog, metrics, distributed tracing |

---

## 🗺️ Bức Tranh Tổng Thể

```
┌─────────────────────────────────────────────────────────────────┐
│                  VÒNG ĐỜI TRIỂN KHAI .NET APP                   │
│                                                                  │
│  Code  ──►  Build  ──►  Test  ──►  Package  ──►  Deploy  ──►  Run│
│                                                                  │
│  ┌──────┐  ┌──────────────────┐  ┌────────┐  ┌───────────────┐  │
│  │ Dev  │  │  CI/CD Pipeline  │  │ Docker │  │  Kubernetes   │  │
│  │      │  │  GitHub Actions  │  │ Image  │  │  Azure AKS    │  │
│  │ git  │  │  Azure DevOps    │  │        │  │  App Service  │  │
│  │ push │  │  Test / Lint     │  │ Docker │  │               │  │
│  └──────┘  │  Docker Build    │  │  Hub / │  └───────────────┘  │
│            │  Push to Registry│  │  ACR   │                      │
│            └──────────────────┘  └────────┘                      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   OBSERVABILITY                            │  │
│  │  Logs (Serilog)  │  Metrics (Prometheus)  │  Traces (OTEL) │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Khi Nào Dùng Gì

### Docker — Container đơn giản

**Dùng khi:**
- Cần đóng gói ứng dụng và chạy nhất quán mọi môi trường
- Development — môi trường phát triển local cần thống nhất với production
- Bước đầu container hóa — containerize

**Không đủ khi:**
- Cần scale — mở rộng tự động theo tải
- Cần high availability — tính sẵn sàng cao
- Quản lý nhiều services cùng lúc

---

### Kubernetes — Điều Phối Container

**Dùng khi:**
- Cần tự động scale — tự động mở rộng
- Cần self-healing — tự phục hồi khi container crash
- Chạy nhiều microservices phụ thuộc nhau
- Zero-downtime deployment — triển khai không gián đoạn

**Cân nhắc:**
- Phức tạp hơn Docker Compose nhiều
- Cần đội ngũ có kinh nghiệm K8s
- Chi phí vận hành cao hơn

---

### Azure App Service — PaaS đơn giản

**Dùng khi:**
- Không muốn quản lý infrastructure — hạ tầng
- Ứng dụng web đơn giản đến trung bình
- Team nhỏ, ít DevOps experience

**Hạn chế:**
- Ít kiểm soát hơn K8s
- Chi phí có thể cao hơn ở scale lớn

---

### Azure Kubernetes Service — AKS

**Dùng khi:**
- Cần full Kubernetes nhưng không muốn tự quản lý control plane — mặt phẳng điều khiển
- Enterprise workload — khối lượng công việc doanh nghiệp
- Cần tích hợp sâu với Azure ecosystem

---

## 📋 Kiến Trúc Triển Khai Tham Khảo

### Môi Trường Tiêu Chuẩn

```
┌────────────┐    ┌────────────┐    ┌────────────┐
│    Dev     │    │  Staging   │    │ Production │
│            │    │            │    │            │
│ Docker     │    │ Kubernetes │    │ Kubernetes │
│ Compose    │    │ (giống     │    │ (full HA)  │
│            │    │  prod)     │    │            │
│ Local DB   │    │ Test DB    │    │ Azure SQL  │
│ Local Redis│    │ Redis Cache│    │ Redis Cache│
└────────────┘    └────────────┘    └────────────┘
       │                │                  │
       └────────────────┴──────────────────┘
                        │
              CI/CD Pipeline (GitHub Actions / Azure DevOps)
```

---

### Stack Công Nghệ Điển Hình

| Tầng | Công Nghệ | Mục Đích |
|------|-----------|----------|
| **Container** | Docker + multi-stage build | Đóng gói ứng dụng |
| **Registry** | Azure Container Registry — ACR | Lưu trữ Docker images |
| **Orchestration** | Azure Kubernetes Service — AKS | Điều phối container |
| **Ingress** | NGINX Ingress Controller | Routing HTTP traffic |
| **Config/Secrets** | Azure Key Vault + Kubernetes Secrets | Quản lý bí mật |
| **Database** | Azure SQL / PostgreSQL Flexible Server | Lưu trữ dữ liệu |
| **Cache** | Azure Cache for Redis | Cache phân tán |
| **Messaging** | Azure Service Bus | Nhắn tin bất đồng bộ |
| **CI/CD** | GitHub Actions / Azure DevOps | Tự động hóa build và deploy |
| **Monitoring** | Azure Monitor + Application Insights | Theo dõi hệ thống |
| **Logging** | Serilog → Azure Log Analytics | Ghi và phân tích logs |
| **Tracing** | OpenTelemetry + Jaeger/Zipkin | Theo dõi distributed traces |

---

## 🔑 Khái Niệm Cốt Lõi

### Container vs Virtual Machine

```
┌─────────────────────────┐     ┌─────────────────────────┐
│     VIRTUAL MACHINE     │     │       CONTAINER         │
│                         │     │                         │
│  ┌─────────┐ ┌────────┐ │     │  ┌────────┐ ┌────────┐  │
│  │  App A  │ │ App B  │ │     │  │ App A  │ │ App B  │  │
│  ├─────────┤ ├────────┤ │     │  ├────────┤ ├────────┤  │
│  │Guest OS │ │Guest OS│ │     │  │ Libs   │ │ Libs   │  │
│  ├─────────┴─┴────────┤ │     │  ├────────┴─┴────────┤  │
│  │    Hypervisor      │ │     │  │  Container Runtime │  │
│  ├────────────────────┤ │     │  ├────────────────────┤  │
│  │      Host OS       │ │     │  │      Host OS       │  │
│  ├────────────────────┤ │     │  ├────────────────────┤  │
│  │     Hardware       │ │     │  │     Hardware       │  │
│  └────────────────────┘ │     │  └────────────────────┘  │
└─────────────────────────┘     └─────────────────────────┘
  Kích thước: GB                  Kích thước: MB
  Khởi động: phút                 Khởi động: giây
  Cô lập: cao                     Cô lập: vừa
  Overhead: cao                   Overhead: thấp
```

### Immutable Infrastructure — Hạ Tầng Bất Biến

Nguyên tắc: **không sửa** server đang chạy — thay vào đó, tạo mới container/instance với code mới và thay thế cái cũ.

```
Cách cũ (Mutable):
  Server → SSH → apt install → config thay đổi → "works on my machine" nightmare

Cách mới (Immutable):
  Code → Build Image → Test → Deploy new pod → Delete old pod
  Mọi thứ nhất quán, reproducible — tái tạo được
```

---

## 🚀 Lộ Trình Học Topic Này

```
Bước 1: Docker cơ bản
  → Hiểu Dockerfile, multi-stage build
  → Biết chạy .NET app trong container

Bước 2: Kubernetes cơ bản
  → Hiểu Pod, Deployment, Service
  → Biết deploy .NET app lên K8s local (minikube/kind)

Bước 3: Azure Cloud
  → Azure App Service cho ứng dụng đơn giản
  → Azure Container Registry + AKS cho production

Bước 4: CI/CD
  → GitHub Actions: build, test, push, deploy tự động
  → Hiểu pipeline as code — pipeline dưới dạng code

Bước 5: Configuration & Secrets
  → Environment variables theo từng môi trường
  → Azure Key Vault cho production secrets

Bước 6: Observability
  → Structured logging với Serilog
  → Metrics với OpenTelemetry + Prometheus
  → Distributed tracing — theo dõi phân tán
```

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp

1. **Docker:** Giải thích multi-stage build — tại sao nên dùng?
2. **Kubernetes:** Sự khác nhau giữa Deployment và StatefulSet?
3. **K8s:** Liveness probe và Readiness probe — khác nhau chỗ nào?
4. **K8s:** HPA — Horizontal Pod Autoscaler — hoạt động như thế nào?
5. **CI/CD:** Blue-Green deployment vs Rolling update vs Canary — khi nào dùng gì?
6. **Secrets:** Tại sao không nên đưa secrets vào environment variables thuần?
7. **Observability:** Sự khác nhau giữa Logging, Metrics và Tracing?
8. **Azure:** Khi nào dùng App Service, khi nào dùng AKS?

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
