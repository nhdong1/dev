# Workload Management — Quản Lý Tải Công Việc

> Tổng quan về các loại workload trong Kubernetes: cách triển khai, quản lý vòng đời, và chọn đúng loại workload cho từng tình huống.

## Mục Lục

1. [Tổng Quan Workload](#tổng-quan-workload)
2. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
3. [Các Loại Workload](#các-loại-workload)
4. [Kiến Trúc Quan Hệ Giữa Các Object](#kiến-trúc-quan-hệ-giữa-các-object)
5. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
6. [Checklist Thực Chiến](#checklist-thực-chiến)

---

## Tổng Quan Workload

Trong Kubernetes, **workload** là thuật ngữ chỉ ứng dụng đang chạy trên cluster. Thay vì quản lý từng container riêng lẻ, Kubernetes cung cấp các **workload resource** (tài nguyên workload) để khai báo trạng thái mong muốn — hệ thống sẽ tự đảm bảo thực tế khớp với khai báo đó.

### Vì Sao Cần Các Loại Workload Khác Nhau?

| Loại Workload | Câu Hỏi Giải Quyết |
| ------------- | ------------------- |
| **Pod**       | Container chạy thế nào? Cần bao nhiêu tài nguyên? |
| **Deployment** | Làm sao để có nhiều bản sao, cập nhật không gián đoạn? |
| **StatefulSet** | Ứng dụng cần danh tính ổn định (database, Kafka)? |
| **DaemonSet** | Cần chạy đúng một instance trên mỗi node? |
| **Job**       | Tác vụ chạy một lần đến hoàn thành (batch processing)? |
| **CronJob**   | Tác vụ chạy theo lịch định kỳ (backup, report)? |

---

## Bản Đồ Quyết Định

```
Ứng dụng của bạn là gì?
│
├── Tác vụ một lần (batch, migration)
│   ├── Chạy ngay bây giờ → Job
│   └── Chạy theo lịch → CronJob
│
└── Dịch vụ chạy liên tục
    ├── Cần chạy trên mọi node (agent, log collector)
    │   └── DaemonSet
    │
    └── Ứng dụng web / microservice
        ├── Có trạng thái (database, message queue, cần lưu trữ ổn định)
        │   └── StatefulSet
        └── Không có trạng thái (stateless)
            └── Deployment
```

---

## Các Loại Workload

### Pod

**Pod** là đơn vị triển khai nhỏ nhất trong Kubernetes — một nhóm một hoặc nhiều container dùng chung mạng và lưu trữ.

> Hiếm khi tạo Pod trực tiếp. Dùng Deployment/StatefulSet/DaemonSet để quản lý Pod.

Tham khảo: [pod.md](./pod.md)

**Điểm cốt lõi:**
- Mỗi Pod có một địa chỉ IP riêng trong cluster
- Các container trong một Pod giao tiếp qua `localhost`
- Pod là **ephemeral** (ngắn hạn) — khi bị xoá, dữ liệu trong container mất
- **Init Container** chạy trước container chính, dùng để khởi tạo môi trường
- **Sidecar Container** chạy song song với container chính (log shipper, proxy)

---

### Deployment

**Deployment** quản lý một tập hợp Pod giống hệt nhau (**ReplicaSet** — tập bản sao) và hỗ trợ cập nhật rolling update.

Tham khảo: [deployment.md](./deployment.md)

**Điểm cốt lõi:**
- Duy trì số lượng Pod mong muốn (`replicas`)
- Hỗ trợ **Rolling Update** (cập nhật cuốn) — thay thế Pod dần dần, không có downtime
- Hỗ trợ **Rollback** (quay lại) về phiên bản trước
- Quản lý lịch sử qua `revisionHistoryLimit`
- Chiến lược: `RollingUpdate` (mặc định) hoặc `Recreate` (dừng rồi tạo mới)

---

### StatefulSet

**StatefulSet** như Deployment nhưng dành cho ứng dụng **stateful** (có trạng thái) — mỗi Pod có danh tính ổn định, thứ tự khởi động và lưu trữ riêng.

Tham khảo: [statefulset.md](./statefulset.md)

**Điểm cốt lõi:**
- Pod được đặt tên theo thứ tự: `pod-0`, `pod-1`, `pod-2`
- Thứ tự tạo và xoá Pod được đảm bảo
- Mỗi Pod có **PersistentVolumeClaim** (yêu cầu lưu trữ bền vững) riêng
- Dùng **Headless Service** (dịch vụ không có ClusterIP) để truy cập từng Pod qua DNS
- Phù hợp: MySQL, PostgreSQL, Redis Cluster, Kafka, Zookeeper, Elasticsearch

---

### DaemonSet

**DaemonSet** đảm bảo mỗi node (hoặc một tập node được chọn) chạy đúng **một bản sao** của một Pod.

Tham khảo: [daemonset.md](./daemonset.md)

**Điểm cốt lõi:**
- Khi thêm node mới, Pod được tự động tạo trên đó
- Khi node bị xoá, Pod cũng bị thu hồi
- Dùng cho: log agent (Fluentd, Filebeat), monitoring agent (Node Exporter), CNI plugin, kube-proxy
- Có thể dùng `nodeSelector` hoặc `affinity` để giới hạn node nào chạy

---

### Job

**Job** tạo một hoặc nhiều Pod để thực hiện tác vụ đến **khi hoàn thành** rồi dừng.

Tham khảo: [job-cronjob.md](./job-cronjob.md)

**Điểm cốt lõi:**
- Pod bị xoá sau khi Job hoàn thành (có thể cấu hình `ttlSecondsAfterFinished`)
- Hỗ trợ chạy song song: `parallelism` và `completions`
- Tự động retry nếu Pod thất bại (`backoffLimit`)
- Dùng cho: database migration, batch processing, export dữ liệu, gửi email hàng loạt

---

### CronJob

**CronJob** tạo Job theo lịch định kỳ, dùng cú pháp cron UNIX tiêu chuẩn.

Tham khảo: [job-cronjob.md](./job-cronjob.md)

**Điểm cốt lõi:**
- Cú pháp: `"0 2 * * *"` (mỗi ngày lúc 2 giờ sáng)
- Quản lý Job cũ qua `successfulJobsHistoryLimit` và `failedJobsHistoryLimit`
- `concurrencyPolicy`: `Allow`, `Forbid`, `Replace` — xử lý khi Job trước chưa xong
- `startingDeadlineSeconds` — bỏ qua nếu không thể khởi động đúng giờ

---

### Health Probes — Kiểm Tra Sức Khoẻ

Kubernetes dùng ba loại probe để kiểm tra trạng thái container:

Tham khảo: [health-probes.md](./health-probes.md)

| Probe | Mục Đích | Hành Động Khi Thất Bại |
| ----- | -------- | ----------------------- |
| **Liveness Probe** (kiểm tra sức sống) | Container có đang chạy không? | Restart container |
| **Readiness Probe** (kiểm tra sẵn sàng) | Container có sẵn sàng nhận traffic không? | Tạm ngừng gửi traffic |
| **Startup Probe** (kiểm tra khởi động) | Ứng dụng đã khởi động xong chưa? | Trì hoãn các probe khác |

---

## Kiến Trúc Quan Hệ Giữa Các Object

```
CronJob
  └── tạo → Job (theo lịch)
                └── tạo → Pod(s)

Deployment
  └── quản lý → ReplicaSet (bản hiện tại và lịch sử)
                  └── quản lý → Pod(s)

StatefulSet
  └── quản lý trực tiếp → Pod-0, Pod-1, Pod-2
                            └── mỗi Pod có PVC riêng

DaemonSet
  └── quản lý trực tiếp → Pod trên mỗi Node
```

**Lưu ý quan trọng:** Khi xoá Deployment/StatefulSet/DaemonSet, các Pod do chúng quản lý cũng sẽ bị xoá theo (`ownerReference` — tham chiếu sở hữu). Để giữ Pod lại khi xoá controller, dùng `--cascade=orphan`.

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Sự khác biệt giữa Deployment và StatefulSet là gì?

| Tiêu Chí | Deployment | StatefulSet |
| -------- | ---------- | ----------- |
| Danh tính Pod | Ngẫu nhiên (`pod-abc123`) | Ổn định theo thứ tự (`pod-0`, `pod-1`) |
| Lưu trữ | Dùng chung hoặc không có | Mỗi Pod có PVC riêng |
| Thứ tự khởi động | Tuỳ ý | Tuần tự (0, 1, 2...) |
| Headless Service | Không cần | Cần thiết |
| Dùng cho | Stateless app | Database, message queue |

### Câu 2: Khi nào dùng DaemonSet thay vì Deployment?

Dùng DaemonSet khi cần **mỗi node** có đúng một instance. Ví dụ: log collector (cần thu log từ mọi node), node monitoring agent, network plugin. Deployment phù hợp khi số lượng replica không gắn với số lượng node.

### Câu 3: Init Container khác Sidecar Container thế nào?

- **Init Container**: chạy **trước** container chính, phải hoàn thành trước khi container chính bắt đầu. Dùng để setup database schema, chờ dependency sẵn sàng.
- **Sidecar Container**: chạy **song song** với container chính trong suốt vòng đời Pod. Dùng để ship log, inject config, proxy traffic.

### Câu 4: Tại sao Pod không nên tạo trực tiếp mà dùng Deployment?

Pod tạo trực tiếp không được tự động restart khi crash, không có rolling update, không có scaling. Deployment đảm bảo số lượng bản sao mong muốn luôn được duy trì và cung cấp cơ chế cập nhật/rollback.

### Câu 5: Job khác Deployment thế nào?

Job chạy Pod đến khi hoàn thành rồi dừng — thiết kế cho tác vụ có điểm kết thúc. Deployment chạy Pod liên tục, tự restart nếu Pod dừng — thiết kế cho dịch vụ luôn sẵn sàng.

---

## Checklist Thực Chiến

### Trước Khi Deploy Workload Lên Production

- [ ] Đã đặt `resources.requests` và `resources.limits` cho mỗi container
- [ ] Đã cấu hình `livenessProbe` và `readinessProbe`
- [ ] Số lượng `replicas` ≥ 2 cho Deployment (high availability — tính sẵn sàng cao)
- [ ] Đã đặt `podAntiAffinity` để Pod phân tán trên nhiều node
- [ ] Đã cấu hình `PodDisruptionBudget (PDB)` để kiểm soát rolling update
- [ ] Đặt `terminationGracePeriodSeconds` đủ dài cho graceful shutdown
- [ ] Image đã pin version cụ thể (không dùng `latest`)
- [ ] Đã kiểm tra `revisionHistoryLimit` để giữ lịch sử rollback

### Debug Workload Bị Lỗi

```bash
# Kiểm tra trạng thái Pod
kubectl get pods -n <namespace>

# Xem mô tả chi tiết và events
kubectl describe pod <pod-name> -n <namespace>

# Xem log container
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Xem log container trước đó (khi bị restart)
kubectl logs <pod-name> -c <container-name> --previous

# Vào trong container để debug
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
```

---

## Điều Hướng Nhanh

| File | Nội Dung |
| ---- | -------- |
| [pod.md](./1-pod.md) | Pod lifecycle, multi-container, init container |
| [deployment.md](./2-deployment.md) | Rolling update, rollback, strategy |
| [statefulset.md](./3-statefulset.md) | Workload có trạng thái, headless service |
| [daemonset.md](./4-daemonset.md) | Chạy agent trên mọi node |
| [job-cronjob.md](./5-job-cronjob.md) | Tác vụ một lần và định kỳ |
| [health-probes.md](./6-health-probes.md) | Liveness, Readiness, Startup Probe |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
