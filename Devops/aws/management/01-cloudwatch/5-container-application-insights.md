# CloudWatch Container Insights & Application Insights

> **Container Insights** (Thông Tin Chi Tiết Container) thu thập metrics và logs chi tiết từ ECS, EKS và Kubernetes. **Application Insights** (Thông Tin Chi Tiết Ứng Dụng) tự động phát hiện và chẩn đoán sự cố ứng dụng mà không cần cấu hình thủ công.

---

## 📚 Mục Lục

1. [Container Insights — Tổng Quan](#container-insights--tổng-quan)
2. [Container Insights Cho ECS](#container-insights-cho-ecs)
3. [Container Insights Cho EKS & Kubernetes](#container-insights-cho-eks--kubernetes)
4. [Container Insights Metrics Quan Trọng](#container-insights-metrics-quan-trọng)
5. [Application Insights — Tổng Quan](#application-insights--tổng-quan)
6. [Application Insights — Cách Hoạt Động](#application-insights--cách-hoạt-động)
7. [So Sánh Container Insights vs Application Insights](#so-sánh-container-insights-vs-application-insights)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Container Insights — Tổng Quan

**CloudWatch Container Insights** là tính năng thu thập, tổng hợp và tóm tắt metrics và logs từ ứng dụng containerized (chứa trong container) chạy trên AWS.

### Tại Sao Cần Container Insights?

Trong môi trường container, hàng trăm pods (đơn vị container Kubernetes) có thể chạy đồng thời, liên tục scale in/out. Không có công cụ chuyên dụng, bạn không thể:

- Biết pod nào đang dùng quá nhiều CPU
- Phát hiện OOMKill (Out Of Memory Kill — Bị Tắt Do Hết RAM) trên container cụ thể
- Tìm container nào restart liên tục (crash loop)
- Xem node-level vs pod-level resource usage

**Container Insights giải quyết** bằng cách thu thập performance data ở nhiều tầng:
```
Cluster Level → Service Level → Task/Pod Level → Container Level
```

### Kiến Trúc Tổng Quát

```
┌────────────────────────────────────────────────────────────────┐
│          ECS Cluster / EKS Cluster / Kubernetes                │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Service A   │  │  Service B   │  │  Service C   │         │
│  │  Task 1      │  │  Pod 1       │  │  Pod 1       │         │
│  │  Task 2      │  │  Pod 2       │  │  Pod 2       │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                 │                  │                  │
│  ┌──────▼─────────────────▼──────────────────▼────────────┐   │
│  │          CloudWatch Agent / Fluent Bit / ADOT           │   │
│  │          (Thu thập metrics & logs từ mỗi node)          │   │
│  └──────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────┼──────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │         CloudWatch             │
              │  Namespace: ContainerInsights  │
              │  Log Groups:                   │
              │    /aws/ecs/containerinsights/ │
              │    /aws/eks/cluster/...        │
              └───────────────────────────────┘
```

---

## Container Insights Cho ECS

### Cách Bật Container Insights Trên ECS

```bash
# Cách 1: Bật khi tạo cluster
aws ecs create-cluster \
  --cluster-name my-production-cluster \
  --settings name=containerInsights,value=enabled

# Cách 2: Bật trên cluster hiện có
aws ecs update-cluster-settings \
  --cluster my-production-cluster \
  --settings name=containerInsights,value=enabled

# Cách 3: Account-level default
aws ecs put-account-setting-default \
  --name containerInsights \
  --value enabled
```

### ECS Metrics Thu Thập Được

**Cluster Level (Cấp Cluster):**
- `CpuReserved` / `CpuUtilized` — CPU được đặt trước và thực sự dùng
- `MemoryReserved` / `MemoryUtilized` — Memory được đặt trước và thực sự dùng
- `RunningTaskCount` — Số task đang chạy

**Service Level (Cấp Service):**
- `CpuUtilized` / `MemoryUtilized` — Tính theo service
- `RunningTaskCount` — Số task trong service

**Task Level (Cấp Task):**
- `CpuUtilized` / `MemoryUtilized` — Của từng task
- `StorageReadBytes` / `StorageWriteBytes`
- `NetworkRxBytes` / `NetworkTxBytes`

**Container Level (Cấp Container):**
- CPU và Memory của từng container bên trong task

### Log Insights Cho ECS

Với Container Insights, logs được thu thập vào:
```
/aws/ecs/containerinsights/{cluster-name}/performance
```

Ví dụ query phân tích:
```sql
# Top tasks dùng nhiều CPU nhất
fields TaskId, ContainerName, CpuUtilized
| filter Type = "Container"
| stats max(CpuUtilized) as MaxCPU by TaskId, ContainerName
| sort MaxCPU desc
| limit 10
```

---

## Container Insights Cho EKS & Kubernetes

### Cách Bật Container Insights Trên EKS

Container Insights trên EKS dùng **CloudWatch Agent** (deployed như DaemonSet — một pod trên mỗi node) và **Fluent Bit** (cho log collection):

```bash
# Cách đơn giản nhất: dùng addon EKS
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name amazon-cloudwatch-observability \
  --service-account-role-arn arn:aws:iam::ACCOUNT:role/CloudWatchAgentRole
```

Addon tự động:
1. Deploy CloudWatch Agent DaemonSet — thu thập metrics
2. Deploy Fluent Bit DaemonSet — thu thập logs từ containers
3. Cấu hình permissions cần thiết

### Kubernetes Metrics Thu Thập Được

**Node Level (Cấp Node):**
- CPU, Memory, Disk, Network của mỗi EC2 node trong cluster

**Pod Level (Cấp Pod):**
- `pod_cpu_utilization` — CPU của pod theo %
- `pod_memory_utilization` — Memory của pod theo %
- `pod_cpu_request_total` / `pod_cpu_limit_total` — Request vs Limit
- `pod_memory_request_total` / `pod_memory_limit_total`
- `pod_number_of_containers` — Số containers trong pod
- `pod_number_of_running_containers` — Số đang running
- `pod_status` — Pod health

**Container Level (Cấp Container):**
- CPU, Memory chi tiết cho từng container trong pod

**Cluster & Namespace Level:**
- Aggregated metrics theo namespace (không gian tên Kubernetes)

### EKS Enhanced Observability (Khả Quan Sát Nâng Cao)

Tính năng mới (2023) cung cấp thêm:
- **Control Plane metrics** (Mặt Phẳng Điều Khiển) — API server latency, etcd health
- **Kube State Metrics** — Pod phase, deployment rollout status
- **GPU metrics** — Cho ML workloads

```bash
# Bật Enhanced Observability
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --configuration-values '{"agent":{"config":{"logs":{"metrics_collected":{"kubernetes":{"enhanced_container_insights":true}}}}}}'
```

---

## Container Insights Metrics Quan Trọng

### Metrics Cần Monitor Thường Xuyên

| Metric                      | Namespace                | Ngưỡng Cảnh Báo Thường Dùng              |
| --------------------------- | ------------------------ | ----------------------------------------- |
| `CpuUtilized` (Task)        | ContainerInsights        | > 80% của CPU reservation                |
| `MemoryUtilized` (Task)     | ContainerInsights        | > 85% của memory reservation             |
| `RunningTaskCount`          | ContainerInsights        | Giảm đột ngột (task crash)               |
| `pod_cpu_utilization`       | ContainerInsights (EKS)  | > 80%                                     |
| `pod_memory_utilization`    | ContainerInsights (EKS)  | > 85%                                     |

### OOMKill Detection (Phát Hiện Bị Tắt Do Hết RAM)

OOMKill xảy ra khi container dùng nhiều RAM hơn limit, Linux kernel buộc terminate nó:

```sql
# Logs Insights — tìm OOMKill events trên EKS
fields @timestamp, @message
| filter @message like /OOMKilled|out of memory|memory limit exceeded/
| stats count() as oomCount by bin(1h)
| sort @timestamp desc
```

```bash
# kubectl — kiểm tra OOMKill
kubectl describe pod <pod-name> | grep -A5 "OOMKilled"
kubectl get events --field-selector reason=OOMKilling
```

### Crash Loop Detection (Phát Hiện Vòng Lặp Crash)

```sql
# Tìm container restart nhiều lần
fields @timestamp, @message
| filter @message like /Back-off restarting failed container/
| stats count() as restartCount by bin(5m)
```

### CPU Throttling (Bóp Nghẹt CPU)

CPU throttling xảy ra khi container đã dùng hết CPU limit — không bị kill nhưng chậm lại:

```sql
# EKS — tìm pods bị throttle
fields pod_name, container_name, pod_cpu_limit_total, pod_cpu_utilization
| filter pod_cpu_utilization > 95
| sort pod_cpu_utilization desc
```

---

## Application Insights — Tổng Quan

**CloudWatch Application Insights** (ra mắt 2019) là tính năng tự động phát hiện và chẩn đoán sự cố ứng dụng, đặc biệt cho các stack công nghệ phổ biến trên EC2 và ECS.

### Stack Được Hỗ Trợ

| Technology Stack         | Ví Dụ                                         |
| ------------------------ | --------------------------------------------- |
| **Windows IIS**          | .NET web applications trên Windows Server      |
| **.NET Core**            | ASP.NET Core trên Linux                        |
| **Java JVM**             | Spring Boot, Tomcat, JBoss, WebLogic           |
| **SAP HANA**             | SAP applications                               |
| **SQL Server**           | Windows SQL Server                             |
| **MySQL / PostgreSQL**   | Trên EC2                                       |
| **Kubernetes**           | EKS workloads                                  |

### Tại Sao Cần Application Insights?

Setup monitoring thủ công cho complex application stack là tedious (mất công):
- Phải biết metrics nào quan trọng cho từng technology
- Phải tạo alarm cho từng metrics
- Phải correlate log với metric khi có sự cố

Application Insights **tự động hóa** tất cả điều này.

---

## Application Insights — Cách Hoạt Động

### Quy Trình Tự Động

```
1. Discover Resources
   └── Application Insights scan Resource Group
       → Tìm EC2, RDS, ELB, Lambda... liên quan

2. Detect Technology Stack
   └── Phân tích metadata để nhận biết:
       → Đây là Java Spring Boot app
       → Database là MySQL RDS
       → Load balancer là ALB

3. Configure Monitoring Automatically
   └── Cài CloudWatch Agent với config đúng cho stack
   └── Bật metrics phù hợp
   └── Tạo log stream patterns
   └── Setup alarms trên critical metrics

4. ML-based Problem Detection
   └── Phân tích metrics và logs liên tục
   └── Phát hiện anomalies (bất thường)
   └── Correlate (tương quan) symptoms để tìm root cause

5. OpsItem Creation
   └── Tạo OpsItem trong SSM OpsCenter
   └── Include: affected resources, related metrics, relevant logs
   └── Đề xuất potential root cause
```

### Kích Hoạt Application Insights

```bash
# Tạo Application Group từ Resource Group hiện có
aws application-insights create-application \
  --resource-group-name my-app-resource-group \
  --ops-center-enabled \
  --cwe-monitor-enabled  # CloudWatch Events

# Hoặc tạo và discover tự động
aws application-insights create-application \
  --resource-group-name my-app-resource-group \
  --auto-config-enabled
```

### Ví Dụ Sự Cố Được Phát Hiện Tự Động

**Scenario 1: Java OOM (Out of Memory)**
```
Triệu Chứng Phát Hiện:
  ✗ JVM Heap Used > 90%
  ✗ GC (Garbage Collection — Dọn Rác Bộ Nhớ) pause time tăng
  ✗ Application log: "java.lang.OutOfMemoryError: Java heap space"
  ✗ Response time tăng → requests timeout

Application Insights:
  → Tạo OpsItem: "Java heap exhaustion detected"
  → Link đến JVM metrics, GC logs, application logs
  → Đề xuất: Increase heap size hoặc investigate memory leak
```

**Scenario 2: Database Connection Pool Exhaustion (Cạn Kiệt Kết Nối Database)**
```
Triệu Chứng Phát Hiện:
  ✗ RDS DatabaseConnections gần max
  ✗ Application log: "Unable to acquire JDBC Connection"
  ✗ Application response time tăng
  ✗ Error rate tăng

Application Insights:
  → Tạo OpsItem: "Database connection pool exhausted"
  → Correlate application errors với RDS connection count
  → Đề xuất: Increase connection pool size hoặc scale RDS
```

### OpsItem Dashboard

Mỗi sự cố phát hiện tạo **OpsItem** (Mục Vận Hành) trong SSM OpsCenter với:

```
OpsItem: "High JVM Heap Utilization"
  Status: Open
  Severity: 2 (High)
  
  Related Resources:
    - EC2: i-0abc123 (app server)
    - RDS: db-instance-prod (database)
  
  Related Insights:
    - JVM Heap Used: 94% (last 30 min)
    - GC Pause Time: 850ms avg (up from 50ms)
    - Request Error Rate: 2.3% (up from 0.1%)
  
  Related Log Events:
    - [10:35:22] OutOfMemoryError: Java heap space
    - [10:35:23] OutOfMemoryError: Java heap space
    - [10:35:23] Application restart initiated
  
  Recommended Resolutions:
    1. Increase JVM heap: -Xmx4g
    2. Check for memory leak with heap dump
    3. Consider horizontal scaling
```

---

## So Sánh Container Insights vs Application Insights

| Tiêu Chí                    | Container Insights                     | Application Insights                    |
| --------------------------- | -------------------------------------- | --------------------------------------- |
| **Mục đích chính**          | Monitor container infrastructure       | Monitor application health              |
| **Phù hợp với**             | ECS, EKS, Kubernetes                   | EC2, ECS, với specific tech stacks      |
| **Cấu hình**                | Enable một lần per cluster             | Setup per application                   |
| **Intelligence**            | Metrics và logs thu thập               | ML-based anomaly detection              |
| **Root Cause**              | Tự mình phân tích                     | Tự động đề xuất                         |
| **OpsCenter integration**   | Không                                  | Có                                      |
| **Stack awareness**         | Container-agnostic                     | Biết về Java, .NET, SQL...              |
| **Pricing**                 | Phí theo vEMF metrics                  | Thêm phí monitoring per resource        |

### Khi Nào Dùng Cái Nào

**Dùng Container Insights khi:**
- Đang chạy ECS hoặc EKS và cần visibility vào container-level metrics
- Muốn monitor pod health, restarts, resource utilization ở tất cả levels
- Team DevOps cần metrics để debug deployment và scaling issues

**Dùng Application Insights khi:**
- Đang chạy traditional apps (.NET, Java) trên EC2
- Muốn automatic monitoring setup mà không cần expert knowledge
- Cần ML-based anomaly detection và root cause suggestions
- Team ít kinh nghiệm AWS, muốn guided troubleshooting

**Dùng cả hai khi:**
- Java/Spring Boot chạy trên EKS → Container Insights cho infra, Application Insights cho app

---

## Câu Hỏi Phỏng Vấn

### Q1: Container Insights thu thập data như thế nào mà không ảnh hưởng app?

Container Insights dùng **sidecar pattern** hoặc **DaemonSet** tách biệt với application containers:
- **ECS**: CloudWatch Agent chạy như task riêng hoặc sidecar container
- **EKS**: CloudWatch Agent deploy như **DaemonSet** — một agent pod trên mỗi node, không trong application pods

Agent thu thập metrics từ cAdvisor (kubelet API) và Docker stats API — không inject code vào app.

### Q2: Sự khác nhau giữa Container Insights và Prometheus?

| Tiêu Chí           | Container Insights                  | Prometheus                              |
| ------------------ | ----------------------------------- | --------------------------------------- |
| **Setup**          | Một lệnh AWS CLI/addon              | Deploy Prometheus server, configure scraping |
| **Scraping**       | Push-based (agent gửi đến CW)      | Pull-based (Prometheus scrape targets)  |
| **Storage**        | CloudWatch (managed)                | Self-managed hoặc managed (Thanos, Cortex)|
| **Query**          | Logs Insights                       | PromQL                                  |
| **Kubernetes native**| Không (AWS-specific)              | Có — chuẩn CNCF                        |
| **Vendor lock-in** | AWS only                            | Portable                                |

**Thực tế:** Nhiều tổ chức dùng **Amazon Managed Service for Prometheus** + Grafana để có best of both.

### Q3: OOMKill khác với Container restart bình thường như thế nào?

**OOMKill**: Linux kernel force-kill process khi dùng RAM vượt limit. `exit code 137`. Không graceful — có thể corrupt state, in-flight requests drop.

**Normal restart**: Application crash do unhandled exception, process exit code khác 137. Kubernetes/ECS restart theo restart policy.

**Cách phân biệt trong kubectl:**
```bash
kubectl describe pod <pod-name>
# OOMKill:
#   Last State: Terminated
#     Reason: OOMKilled
#     Exit Code: 137
```

### Q4: Application Insights có thể thay thế APM tool như Datadog không?

**Phần lớn: Không.** Application Insights tốt cho:
- Automated problem detection trên specific stacks
- Integration với AWS ecosystem (OpsCenter, SSM)
- Cost-effective cho AWS-only shops

Nhưng thiếu so với APM (Application Performance Management — Quản Lý Hiệu Suất Ứng Dụng) tools:
- Distributed tracing (theo dõi request xuyên suốt services) — cần X-Ray riêng
- Code-level profiling (phân tích hiệu năng đến từng dòng code)
- Custom dashboards phức tạp
- Multi-cloud support

**Kết hợp tốt:** Application Insights + AWS X-Ray cho distributed tracing.

### Q5: Làm thế nào debug pod "Pending" mà không schedule được lên node nào?

```bash
# 1. Kiểm tra sự kiện
kubectl describe pod <pod-name> | grep -A 10 Events

# 2. Logs Insights với Container Insights
fields @timestamp, @message
| filter @message like /FailedScheduling|Insufficient cpu|Insufficient memory/
| sort @timestamp desc

# 3. Kiểm tra node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Thường gặp:
# - Node không đủ CPU/Memory → Tăng node group hoặc dùng instance type lớn hơn
# - Node có taints chưa matching tolerations của pod
# - Pod có nodeSelector không match node hiện có
# - PersistentVolume không available trong AZ của node
```

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [4-dashboards-widgets.md](./4-dashboards-widgets.md) | [6-cloudwatch-agent.md](./6-cloudwatch-agent.md)
