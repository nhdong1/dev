# Scaling — Tự Động Mở Rộng Kubernetes

> Tổng quan về hệ thống tự động mở rộng (Auto Scaling) trong Kubernetes: từ HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang), VPA (Vertical Pod Autoscaler — Tự Động Điều Chỉnh Tài Nguyên Pod), Cluster Autoscaler (Tự Động Mở Rộng Node Cluster), KEDA (Kubernetes Event-Driven Autoscaling — Tự Động Mở Rộng Dựa Trên Sự Kiện), đến quản lý tài nguyên với ResourceQuota và LimitRange.

## Mục Lục

1. [Mô Hình Scaling Kubernetes](#mô-hình-scaling-kubernetes)
2. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
3. [Các Thành Phần Scaling](#các-thành-phần-scaling)
4. [So Sánh Các Cơ Chế Scaling](#so-sánh-các-cơ-chế-scaling)
5. [Ma Trận Tình Huống vs Giải Pháp](#ma-trận-tình-huống-vs-giải-pháp)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
7. [Checklist Scaling Production](#checklist-scaling-production)

---

## Mô Hình Scaling Kubernetes

Kubernetes có **3 chiều mở rộng** độc lập và bổ trợ cho nhau:

```
┌──────────────────────────────────────────────────────────────────┐
│                    CLUSTER SCALING                               │
│   Cluster Autoscaler — thêm/bớt Node khi cần                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   POD SCALING                              │  │
│  │   HPA — thêm/bớt số lượng Pod replica                     │  │
│  │   KEDA — scale theo sự kiện (queue, cron...)               │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │             CONTAINER RESOURCE SCALING               │  │  │
│  │  │   VPA — tăng/giảm CPU/memory của từng container      │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**3 chiều mở rộng:**
- **Horizontal Scaling (Mở Rộng Theo Chiều Ngang):** tăng số lượng Pod — phù hợp workload stateless
- **Vertical Scaling (Mở Rộng Theo Chiều Dọc):** tăng CPU/memory của Pod — phù hợp workload có trạng thái
- **Infrastructure Scaling (Mở Rộng Hạ Tầng):** tăng số node trong cluster — cần thiết khi node đầy

---

## Bản Đồ Quyết Định

```
Ứng dụng bị chậm hoặc không đủ tài nguyên?
│
├── Pod bị throttle CPU hoặc OOMKilled?
│   ├── Request/Limit đặt quá thấp, không đủ cho workload hiện tại
│   └── → VPA (Vertical Pod Autoscaler) — tự động điều chỉnh request/limit
│       └── Xem: 2-vpa.md
│
├── Traffic tăng đột biến, cần nhiều replica hơn?
│   ├── Scale theo CPU hoặc memory
│   └── → HPA (Horizontal Pod Autoscaler) — thêm Pod khi metric vượt ngưỡng
│       └── Xem: 1-hpa.md
│
├── Scale theo queue depth, số message, hoặc event bên ngoài?
│   ├── Kafka lag tăng, SQS queue dài, RabbitMQ message nhiều
│   └── → KEDA (Kubernetes Event-Driven Autoscaling)
│       └── Xem: 4-keda.md
│
├── Pod ở trạng thái Pending vì không đủ node?
│   ├── Cluster đã dùng hết tài nguyên, cần thêm node
│   └── → Cluster Autoscaler — tự động thêm node
│       └── Xem: 3-cluster-autoscaler.md
│
└── Namespace không kiểm soát được tài nguyên tiêu thụ?
    ├── Team dùng quá nhiều CPU/memory, không có giới hạn
    └── → ResourceQuota + LimitRange — đặt giới hạn cứng cho namespace
        └── Xem: 5-resource-management.md
```

---

## Các Thành Phần Scaling

### HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang)

**HPA** liên tục kiểm tra metric của workload và điều chỉnh số lượng replica. Controller loop mặc định chạy mỗi **15 giây**.

```
metrics-server / Prometheus
        │  cung cấp metric
        ▼
   HPA Controller
        │  so sánh metric với target
        │  tính toán desired replicas
        ▼
Deployment / StatefulSet / ReplicaSet
        │  scale up/down số Pod
        ▼
   Pod instances
```

**Hỗ trợ 3 loại metric:**
- `Resource` — CPU và memory của Pod (cần metrics-server)
- `Pods` — metric tuỳ chỉnh trên mỗi Pod (cần custom metrics API)
- `External` — metric từ hệ thống ngoài (cần external metrics API)

Tham khảo chi tiết: [1-hpa.md](./1-hpa.md)

---

### VPA — Vertical Pod Autoscaler (Tự Động Điều Chỉnh Tài Nguyên Pod)

**VPA** phân tích lịch sử tiêu thụ tài nguyên và đề xuất (hoặc tự động điều chỉnh) `resource.requests` và `resource.limits` của container.

```
Pod running
    │  báo cáo usage
    ▼
VPA Recommender
    │  phân tích lịch sử, tính toán recommendation
    ▼
VPA Updater + Admission Controller
    │  (nếu mode Auto/Recreate) evict Pod cũ, áp dụng request mới
    ▼
Pod mới chạy với resource được tối ưu
```

**4 chế độ VPA:**
- `Off` — chỉ đề xuất, không tự động áp dụng
- `Initial` — áp dụng khi Pod khởi động, không evict Pod đang chạy
- `Recreate` — evict Pod khi recommendation thay đổi đáng kể
- `Auto` — như Recreate, Kubernetes kiểm soát thời điểm cập nhật

Tham khảo chi tiết: [2-vpa.md](./2-vpa.md)

---

### Cluster Autoscaler — Tự Động Mở Rộng Node

**Cluster Autoscaler (CA)** giám sát Pod ở trạng thái `Pending` (không được schedule vì thiếu tài nguyên) và tự động thêm node. Ngược lại, khi node nhàn rỗi quá lâu, CA sẽ drain và xoá node để tiết kiệm chi phí.

```
Pod Pending (không đủ tài nguyên trên cluster)
        │
        ▼
Cluster Autoscaler phát hiện
        │  kiểm tra node group có thể thêm node không
        ▼
Thêm node mới (qua Cloud Provider API: EKS, GKE, AKS)
        │
        ▼
Scheduler đặt Pod lên node mới
        │
        ▼
Pod Running ✓

---

Node nhàn rỗi > 10 phút, utilization < 50%
        │
        ▼
Cluster Autoscaler đánh giá có thể drain node không
        │  kiểm tra PodDisruptionBudget, daemonset, local storage
        ▼
Drain node (evict Pod sang node khác)
        ▼
Xoá node — giảm chi phí cloud
```

Tham khảo chi tiết: [3-cluster-autoscaler.md](./3-cluster-autoscaler.md)

---

### KEDA — Kubernetes Event-Driven Autoscaling (Tự Động Mở Rộng Dựa Trên Sự Kiện)

**KEDA** mở rộng khả năng của HPA bằng cách cho phép scale dựa trên **sự kiện từ hệ thống bên ngoài** — Kafka lag, độ dài SQS queue, số message RabbitMQ, lịch cron, và hàng chục nguồn khác.

KEDA có thể **scale về 0 replica** — điều HPA không làm được — rất hữu ích cho workload theo lịch hoặc ít dùng.

```
External Event Source (Kafka, SQS, RabbitMQ, Prometheus...)
        │  KEDA Scaler kéo metric
        ▼
KEDA Operator
        │  tính toán desired replicas dựa trên trigger
        ▼
HPA object (KEDA tạo và quản lý HPA thay bạn)
        │
        ▼
Deployment / Job / StatefulSet scale up/down
```

Tham khảo chi tiết: [4-keda.md](./4-keda.md)

---

### Resource Management — Quản Lý Tài Nguyên

**Resource Request và Limit** là nền tảng của mọi cơ chế scaling — HPA dựa vào chúng để tính toán, Cluster Autoscaler dựa vào chúng để biết node có đủ chỗ không, VPA điều chỉnh chúng.

```
Container spec:
resources:
  requests:           ← Scheduler dùng để tìm node phù hợp
    cpu: "250m"       ← Kubelet đảm bảo container nhận được ít nhất 250m CPU
    memory: "256Mi"
  limits:             ← Kernel/container runtime enforce hard limit
    cpu: "500m"       ← CPU throttle nếu vượt quá 500m
    memory: "512Mi"   ← OOMKill nếu vượt quá 512Mi
```

**ResourceQuota** và **LimitRange** kiểm soát ở cấp namespace — đặt trần tổng tài nguyên và đặt default request/limit khi Pod không khai báo.

Tham khảo chi tiết: [5-resource-management.md](./5-resource-management.md)

---

## So Sánh Các Cơ Chế Scaling

| Cơ Chế | Scale Cái Gì | Dựa Trên | Giảm Về 0? | Phù Hợp |
| ------- | ------------ | -------- | ---------- | -------- |
| **HPA** | Số Pod replica | CPU, memory, custom metric | Không (min 1) | Stateless app với traffic web |
| **VPA** | CPU/memory của container | Lịch sử sử dụng thực tế | Không | Batch job, workload khó predict |
| **KEDA** | Số Pod replica | Event: queue, cron, DB... | Có ✓ | Worker queue, cron job, event processor |
| **Cluster Autoscaler** | Số Node | Pod Pending, node utilization | Không (min node group) | Kết hợp với HPA/KEDA |

> **Kết hợp phổ biến nhất:** HPA + Cluster Autoscaler — HPA thêm Pod khi traffic tăng, CA thêm Node khi cluster đầy.

---

## Ma Trận Tình Huống vs Giải Pháp

| Tình Huống | Dấu Hiệu | Giải Pháp | File |
| ---------- | --------- | --------- | ---- |
| API server chậm giờ cao điểm | CPU cao, latency tăng | HPA theo CPU/RPS | [1-hpa.md](./1-hpa.md) |
| Worker consume Kafka chậm | Consumer lag tăng | KEDA với Kafka trigger | [4-keda.md](./4-keda.md) |
| Pod bị OOMKilled liên tục | `memory limit` quá thấp | VPA để tính toán limit phù hợp | [2-vpa.md](./2-vpa.md) |
| Pod ở trạng thái Pending mãi | Node không đủ tài nguyên | Cluster Autoscaler + đúng request | [3-cluster-autoscaler.md](./3-cluster-autoscaler.md) |
| Team dùng hết tài nguyên cluster | Namespace chiếm nhiều CPU/memory | ResourceQuota cho namespace | [5-resource-management.md](./5-resource-management.md) |
| Pod mới không có resource request | Pod không biết cần bao nhiêu | LimitRange đặt default | [5-resource-management.md](./5-resource-management.md) |
| Batch job chỉ chạy ban đêm | Lãng phí replica suốt ngày | KEDA với cron trigger | [4-keda.md](./4-keda.md) |
| CPU throttled nhưng memory ổn | CPU limit quá thấp | VPA recommendation + điều chỉnh limit | [2-vpa.md](./2-vpa.md) |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**HPA và VPA khác nhau thế nào? Khi nào dùng cái nào?**

> **HPA (Horizontal Pod Autoscaler)** thêm/bớt số lượng Pod replica — phù hợp workload stateless có thể chạy nhiều instance song song như web API, worker service. **VPA (Vertical Pod Autoscaler)** điều chỉnh CPU/memory request của từng Pod — phù hợp workload không thể scale ngang dễ dàng (StatefulSet), hoặc dùng để tìm giá trị request phù hợp cho HPA. Trong production, thường dùng **HPA là chính, VPA ở mode Off để lấy recommendation** rồi điền vào manifest tĩnh.

**Tại sao không nên dùng HPA và VPA cùng lúc?**

> Mặc định HPA và VPA xung đột khi cùng quản lý `resources.requests` của cùng một Pod — VPA thay đổi request làm HPA tính toán sai (HPA dùng `requests` làm denominator). Từ K8s 1.20+, chỉ có thể dùng cả hai nếu HPA scale theo **custom metric hoặc external metric** thay vì CPU/memory, hoặc dùng VPA ở mode `Off` (chỉ đề xuất, không tự áp dụng).

**Cluster Autoscaler scale out khi nào? Scale in khi nào?**

> **Scale out:** khi có Pod ở trạng thái `Pending` vì không có node nào đủ tài nguyên (cpu/memory request). CA tìm node group phù hợp, tạo node mới qua cloud provider API. **Scale in:** khi một node có utilization < 50% trong 10 phút liên tiếp và tất cả Pod trên đó có thể chạy trên node khác (không vi phạm PodDisruptionBudget, không có local storage). CA drain node và xoá khỏi cloud. Scale in rủi ro hơn nên cooldown mặc định dài hơn (10 phút) so với scale out.

### Câu Hỏi Nâng Cao

**KEDA là gì và tại sao nó mạnh hơn HPA thuần tuý?**

> **KEDA (Kubernetes Event-Driven Autoscaling)** là operator mở rộng HPA bằng cách cung cấp hàng chục "scaler" cho phép scale dựa trên nguồn sự kiện đa dạng: Kafka consumer lag, SQS queue depth, RabbitMQ message count, Prometheus query, cron schedule, database row count... KEDA mạnh hơn HPA thuần ở 3 điểm: (1) **Scale về 0** — HPA không thể scale về 0, KEDA có thể giảm về 0 replica khi không có sự kiện, tiết kiệm tài nguyên tối đa; (2) **Trigger linh hoạt** — HPA chỉ dùng CPU/memory hoặc custom metric qua adapter riêng, KEDA tích hợp sẵn >50 scaler; (3) **ScaledJob** — scale Job tạm thời thay vì Deployment thường trực.

**Giải thích QoS class trong Kubernetes và ảnh hưởng đến scaling?**

> Kubernetes phân Pod thành 3 **QoS (Quality of Service — Chất Lượng Dịch Vụ) class** dựa trên cách khai báo request/limit: (1) **Guaranteed** — requests == limits cho mọi container; Pod này ít bị evict nhất khi node thiếu tài nguyên; (2) **Burstable** — có requests nhưng limits khác requests, hoặc chỉ có requests; bị evict khi node pressure trung bình; (3) **BestEffort** — không khai báo requests lẫn limits; bị evict đầu tiên. Ảnh hưởng đến scaling: Pod BestEffort không có requests — Scheduler không thể tính toán node phù hợp cho Cluster Autoscaler, dẫn đến scale out không hiệu quả. Luôn khai báo requests chính xác.

---

## Checklist Scaling Production

### Resource Request và Limit

- [ ] Mọi container đều khai báo `resources.requests` và `resources.limits`
- [ ] CPU limit không thấp hơn 2x CPU request (tránh throttle)
- [ ] Memory limit bằng hoặc gần với memory request (tránh OOMKill bất ngờ)
- [ ] Dùng VPA ở mode `Off` trong staging để lấy recommendation trước khi set cứng
- [ ] Không khai báo CPU limit quá cao hơn thực tế — ảnh hưởng Cluster Autoscaler packing

### HPA

- [ ] HPA có `minReplicas >= 2` cho production service (đảm bảo high availability)
- [ ] Target CPU utilization ở mức 60–70% (không phải 80–90%) để có buffer scale out
- [ ] `scaleDown.stabilizationWindowSeconds` đủ dài (300s+) để tránh flapping
- [ ] Kiểm tra `kubectl get hpa` để xem `TARGETS` — nếu `<unknown>` là metrics-server có vấn đề
- [ ] Với custom metric: verify metrics adapter hoạt động trước khi deploy HPA

### Cluster Autoscaler

- [ ] Node group có min/max rõ ràng — tránh vô tình scale vô hạn
- [ ] Đặt annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` cho Pod không nên bị drain
- [ ] Kiểm tra PodDisruptionBudget để CA không drain Pod gây downtime
- [ ] Bật `--balance-similar-node-groups` nếu có nhiều node group tương tự nhau
- [ ] Monitor CA logs: `kubectl logs -n kube-system -l app=cluster-autoscaler`

### KEDA

- [ ] Kiểm tra `ScaledObject status` sau khi deploy — xem Active và Ready condition
- [ ] Test scale về 0 ở staging trước khi áp dụng production
- [ ] TriggerAuthentication sử dụng Secret hoặc ServiceAccount — không hardcode credential
- [ ] Đặt `minReplicaCount: 1` cho production service không thể có downtime khi scale về 0
- [ ] Theo dõi KEDA operator log khi gặp sự cố scale

### ResourceQuota và LimitRange

- [ ] Mỗi production namespace có ResourceQuota giới hạn CPU và memory
- [ ] LimitRange đặt default request/limit cho Pod thiếu khai báo
- [ ] Alert khi namespace đạt 80% quota — tránh Pod Pending bất ngờ
- [ ] Kiểm tra quota hiện tại: `kubectl describe resourcequota -n <namespace>`

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
| ---- | -------- |
| [1-hpa.md](./1-hpa.md) | HPA cấu hình chi tiết, custom metric, behavior tuning |
| [2-vpa.md](./2-vpa.md) | VPA mode, recommender, kết hợp với HPA |
| [3-cluster-autoscaler.md](./3-cluster-autoscaler.md) | CA với EKS/GKE/AKS, node group, scale in/out |
| [4-keda.md](./4-keda.md) | KEDA ScaledObject, Kafka, SQS, cron trigger, scale to zero |
| [5-resource-management.md](./5-resource-management.md) | Request, Limit, QoS, ResourceQuota, LimitRange |
