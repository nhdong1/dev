# Chuẩn Bị Phỏng Vấn Kubernetes — Tổng Quan

> Module cuối cùng tổng hợp toàn bộ kiến thức Kubernetes, giúp bạn tự tin trả lời câu hỏi phỏng vấn, kể câu chuyện sự cố, và thiết kế hệ thống với K8s ở mọi cấp độ.

## Mục Lục

1. [Cấu Trúc Module](#cấu-trúc-module)
2. [Chiến Lược Ôn Tập](#chiến-lược-ôn-tập)
3. [Phân Loại Câu Hỏi Theo Cấp Độ](#phân-loại-câu-hỏi-theo-cấp-độ)
4. [Checklist Trước Phỏng Vấn](#checklist-trước-phỏng-vấn)
5. [Cách Trả Lời Câu Hỏi K8s](#cách-trả-lời-câu-hỏi-k8s)

---

## Cấu Trúc Module

```
12-interview-prep/
├── README.md                  Tổng quan (file này)
├── INTERVIEW_GUIDE.md         Top 30 câu hỏi + đáp án chi tiết
├── system-design.md           Thiết kế hệ thống với Kubernetes
├── star-stories.md            Câu chuyện sự cố theo phương pháp STAR
├── hands-on-scenarios.md      Bài tập thực hành tình huống thực chiến
└── 90-day-study-plan.md       Kế hoạch học 90 ngày có lịch chi tiết
```

---

## Chiến Lược Ôn Tập

### Nếu Chỉ Có 1 Tuần

```
Ngày 1–2: Đọc INTERVIEW_GUIDE.md — nắm top 30 câu hỏi
Ngày 3:   Ôn lại 01-architecture/ và 02-workload/
Ngày 4:   Ôn 03-networking/ và 06-security/
Ngày 5:   Luyện tập hands-on-scenarios.md
Ngày 6:   Chuẩn bị 2–3 câu chuyện STAR từ star-stories.md
Ngày 7:   Mock phỏng vấn — tự trả lời không nhìn tài liệu
```

### Nếu Có 1 Tháng

```
Tuần 1: Ôn lại nền tảng (01, 02, 03) + làm hands-on lab
Tuần 2: Ôn vận hành (06, 07, 08, 10) + luyện debug
Tuần 3: Ôn CI/CD + system design + câu chuyện STAR
Tuần 4: Mock phỏng vấn + gap analysis + ôn lại điểm yếu
```

### Nếu Có 3 Tháng

Theo [90-day-study-plan.md](./90-day-study-plan.md) — đầy đủ nhất.

---

## Phân Loại Câu Hỏi Theo Cấp Độ

### Junior Engineer (0–2 năm kinh nghiệm)

Câu hỏi tập trung vào **khái niệm cơ bản** và **thao tác kubectl**:

| Chủ Đề | Câu Hỏi Hay Gặp |
|--------|-----------------|
| Kiến trúc | Pod là gì? Deployment khác ReplicaSet thế nào? |
| Networking | ClusterIP vs NodePort vs LoadBalancer? |
| Storage | PV và PVC khác nhau thế nào? |
| Config | ConfigMap vs Secret — khi nào dùng cái nào? |
| Debug | Pod ở trạng thái Pending — nguyên nhân và cách xử lý? |

### Mid-level Engineer (2–4 năm kinh nghiệm)

Câu hỏi tập trung vào **vận hành** và **thiết kế**:

| Chủ Đề | Câu Hỏi Hay Gặp |
|--------|-----------------|
| Deployment | Cấu hình rolling update không có downtime? |
| Security | Thiết kế RBAC cho team 5 người? |
| Scaling | HPA dựa trên custom metric — cấu hình ra sao? |
| Monitoring | Cấu hình alert khi Pod restart nhiều lần? |
| Incident | Kể về sự cố bạn đã xử lý — phương pháp STAR |

### Senior Engineer (4+ năm kinh nghiệm)

Câu hỏi tập trung vào **trade-off**, **kiến trúc**, và **leadership**:

| Chủ Đề | Câu Hỏi Hay Gặp |
|--------|-----------------|
| Architecture | Thiết kế multi-cluster cho hệ thống global? |
| Cost | Tối ưu chi phí cluster K8s trên cloud? |
| Security | Implement zero-trust security trên K8s? |
| Reliability | Thiết kế SLO/SLA cho K8s-based service? |
| Leadership | Cách bạn xây dựng K8s platform cho team? |

---

## Checklist Trước Phỏng Vấn

### Kiến Thức Lý Thuyết

- [ ] Giải thích được luồng `kubectl apply` → Pod chạy (không nhìn tài liệu)
- [ ] Phân biệt được Deployment / StatefulSet / DaemonSet / Job
- [ ] Giải thích RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding
- [ ] Mô tả cách CoreDNS giải quyết service name
- [ ] Hiểu HPA hoạt động dựa trên metric như thế nào
- [ ] Biết các Access Mode của PersistentVolume
- [ ] Phân biệt liveness probe và readiness probe
- [ ] Hiểu Network Policy — ingress vs egress rule

### Kỹ Năng Thực Hành

- [ ] Tạo Deployment, Service, Ingress bằng YAML
- [ ] Debug Pod bằng `kubectl logs`, `describe`, `exec`
- [ ] Cấu hình HPA cho Deployment
- [ ] Tạo Role và RoleBinding cho ServiceAccount
- [ ] Kiểm tra kết nối giữa 2 Pod khác namespace
- [ ] Scale Deployment lên/xuống và kiểm tra rolling update
- [ ] Mount ConfigMap và Secret vào Pod

### Câu Chuyện Cá Nhân

- [ ] Chuẩn bị ≥ 2 câu chuyện sự cố theo phương pháp STAR
- [ ] Mỗi câu chuyện có kết quả đo lường được (số cụ thể)
- [ ] Có thể trả lời "bạn học được gì từ sự cố đó?"
- [ ] Chuẩn bị câu chuyện về cải tiến hiệu năng hoặc bảo mật

### System Design

- [ ] Thiết kế được hệ thống microservices cơ bản trên K8s
- [ ] Giải thích lý do chọn giải pháp (trade-off)
- [ ] Tính toán số lượng replica, resource request/limit
- [ ] Thiết kế monitoring và alerting stack

---

## Cách Trả Lời Câu Hỏi K8s

### Framework PREP (Point — Reason — Example — Point)

```
P (Point — Điểm chính):     Trả lời thẳng câu hỏi trong 1 câu
R (Reason — Lý do):         Giải thích tại sao / cơ chế hoạt động
E (Example — Ví dụ):        Ví dụ cụ thể hoặc tình huống thực tế
P (Point — Nhắc lại):       Tóm tắt lại điểm chính
```

**Ví dụ áp dụng PREP:**

*Câu hỏi: "HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) hoạt động thế nào?"*

```
P: HPA tự động tăng hoặc giảm số lượng Pod dựa trên metric hiện tại
   so với ngưỡng mong muốn.

R: Theo định kỳ 15 giây (mặc định), HPA controller đọc metric từ
   metrics-server hoặc custom metric adapter, tính toán số Pod cần thiết
   bằng công thức: desiredReplicas = ceil(currentReplicas × (currentMetric / desiredMetric))

E: Ví dụ: Deployment có 3 Pod, targetCPU = 50%. Nếu CPU hiện tại là 90%,
   HPA tính: ceil(3 × 90/50) = ceil(5.4) = 6 Pod. HPA sẽ scale lên 6 Pod.

P: Tóm lại, HPA là vòng lặp điều khiển tự động điều chỉnh số Pod
   để giữ metric gần với ngưỡng mong muốn.
```

### Kỹ Thuật "Show Don't Tell"

Thay vì nói chung chung, hãy đưa ra số cụ thể:

| Thay vì nói... | Hãy nói... |
|----------------|------------|
| "Chúng tôi cải thiện hiệu năng" | "Latency p99 giảm từ 800ms xuống 120ms" |
| "Chúng tôi scale ứng dụng tốt" | "HPA scale từ 3 lên 45 Pod trong 2 phút khi traffic tăng 10x" |
| "Sự cố xảy ra khá nặng" | "Downtime 23 phút, ảnh hưởng 12.000 user" |
| "Tôi sửa được nhanh" | "MTTR (Mean Time To Recovery) giảm từ 45 phút xuống 8 phút" |

---

## Những Lỗi Phổ Biến Khi Phỏng Vấn K8s

### Lỗi Kiến Thức

1. **Nhầm lẫn Deployment và ReplicaSet** — Deployment *quản lý* ReplicaSet, không phải thay thế
2. **Nhầm PV và PVC** — PV là tài nguyên storage, PVC là yêu cầu dùng storage
3. **Nói "Service = Load Balancer"** — ClusterIP không có load balancer ngoài, chỉ có kube-proxy
4. **Quên mention etcd** khi giải thích kiến trúc — etcd là trái tim của K8s

### Lỗi Trình Bày

1. **Trả lời quá chung chung** — thiếu ví dụ cụ thể, số liệu thực tế
2. **Không hỏi lại** khi câu hỏi mơ hồ — hãy làm rõ trước khi trả lời
3. **Nói "tôi không biết" và dừng lại** — hãy nói những gì bạn biết và suy luận có căn cứ
4. **Quá tập trung vào công cụ** — interviewer muốn nghe về vấn đề và quyết định

---

## Tài Nguyên Ôn Tập Nhanh

| Chủ Đề | File | Thời Gian |
|--------|------|-----------|
| Top 30 câu hỏi | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | 3–4 giờ |
| System Design | [system-design.md](./1-system-design.md) | 2–3 giờ |
| Câu chuyện STAR | [star-stories.md](./2-star-stories.md) | 1–2 giờ |
| Thực hành | [hands-on-scenarios.md](./3-hands-on-scenarios.md) | 4–6 giờ |
| Kế hoạch 90 ngày | [90-day-study-plan.md](./4-90-day-study-plan.md) | Tham khảo |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
