# EFS Performance Modes — Chế Độ Hiệu Suất

> Performance Mode (Chế Độ Hiệu Suất) của EFS quyết định khả năng xử lý I/O — Input/Output (Đầu Vào/Đầu Ra) đồng thời. Đây là cấu hình **không thể thay đổi sau khi tạo file system** — cần chọn đúng từ đầu.

---

## 📌 Hai Chế Độ Hiệu Suất

| | General Purpose | Max I/O |
|-|-----------------|---------|
| **Tên Tiếng Việt** | Đa Dụng | Tối Đa I/O |
| **IOPS** (Hoạt động I/O/giây) | ~35.000 IOPS | Không giới hạn (scale ngang) |
| **Latency** (Độ trễ) | Rất thấp — ~1ms | Cao hơn — ~10ms trở lên |
| **Concurrency** (Đồng thời) | Hàng trăm client | Hàng nghìn client |
| **Throughput Modes** | Tất cả | Tất cả |
| **Recommended** (Khuyến nghị) | Hầu hết workloads | HPC, big data, media |
| **Có thể đổi không?** | Không | Không |

---

## ⚡ General Purpose Mode — Chế Độ Đa Dụng

### Đặc Điểm Kỹ Thuật

- **IOPS tối đa:** ~35.000 IOPS tổng hợp (aggregate)
- **Latency:** Thấp nhất có thể (~1ms per operation)
- **Metadata operations:** Tối ưu cho thao tác metadata (tạo/xóa/liệt kê file)
- **Mặc định:** AWS khuyến nghị cho phần lớn workloads

### Phù Hợp Với

```
✅ Web servers (máy chủ web) — latency quan trọng hơn throughput
✅ CMS — Content Management System (Hệ Thống Quản Lý Nội Dung) như WordPress
✅ Home directories (thư mục home người dùng) — nhiều user, file nhỏ
✅ Development environments (môi trường phát triển)
✅ Container workloads — ECS, EKS với vài trăm pods
✅ Application logs (nhật ký ứng dụng) — ghi nhiều file nhỏ
✅ Git repositories (kho mã nguồn) — nhiều thao tác metadata
```

### Giới Hạn General Purpose

```
❌ Hàng nghìn EC2 instances đồng thời → chạm giới hạn IOPS
❌ Big data analytics (phân tích dữ liệu lớn) với hàng triệu file
❌ HPC — High Performance Computing workloads yêu cầu IOPS cao
```

---

## 🚀 Max I/O Mode — Chế Độ Tối Đa I/O

### Đặc Điểm Kỹ Thuật

- **IOPS:** Scale ngang — không có giới hạn cứng (distributed architecture)
- **Latency:** Cao hơn General Purpose — do thêm tầng phân tán (distributed layer)
- **Concurrency:** Tối ưu cho hàng nghìn client đồng thời
- **Overhead:** Thêm latency cho mỗi operation vì phải điều phối phân tán

### Phù Hợp Với

```
✅ Big data analytics (phân tích dữ liệu lớn) — Spark, Hadoop, EMR
✅ HPC — High Performance Computing (Tính Toán Hiệu Suất Cao) — genomics, simulation
✅ Media processing (xử lý media) — video rendering, transcoding
✅ Machine learning (học máy) — training datasets với hàng nghìn workers
✅ Hàng nghìn EC2 instances cùng truy cập một file system
```

### Không Phù Hợp Với

```
❌ Web applications thông thường — latency cao gây chậm response
❌ Ứng dụng nhạy cảm với latency (databases, real-time APIs)
❌ Workloads metadata-heavy (nhiều thao tác ls, stat, open/close file nhỏ)
```

---

## 📊 So Sánh Chi Tiết

### Latency Profile (Phân Tích Độ Trễ)

```
General Purpose:
  Read  latency: ~1.0ms (99th percentile)
  Write latency: ~1.5ms (99th percentile)
  Metadata ops:  ~0.5ms (rất nhanh)

Max I/O:
  Read  latency: ~5–10ms+ (higher due to coordination)
  Write latency: ~10ms+
  Metadata ops:  ~3–5ms
```

### Throughput So Sánh (Với Cùng Provisioned Throughput)

```
                    General Purpose        Max I/O
Sequential Read:    ~3 GB/s               ~3 GB/s (tương đương)
Sequential Write:   ~3 GB/s               ~3 GB/s
Random Read IOPS:   ~35K ops/s            Scale horizontally
Metadata IOPS:      ~35K ops/s            Lower per-op, higher aggregate
Concurrent clients: ~500                  10.000+
```

---

## 🔍 Phân Biệt Qua Ví Dụ Thực Tế

### Ví Dụ 1 — CMS Platform (Nền Tảng Quản Lý Nội Dung)

```
Scenario: WordPress farm với 50 web servers, mỗi server phục vụ
          người dùng đọc/ghi bài viết, hình ảnh.

Workload characteristics:
  - Nhiều file nhỏ (PHP files, CSS, JS, ảnh nhỏ)
  - Nhiều thao tác metadata (open, stat, read)
  - Latency quan trọng (user-facing)
  - Concurrent clients: 50 EC2 instances

→ Chọn: General Purpose ✅
  Lý do: Latency thấp quan trọng hơn; metadata operations nhiều;
          50 clients không chạm giới hạn IOPS của General Purpose
```

### Ví Dụ 2 — Genomics Analysis Pipeline (Pipeline Phân Tích Bộ Gen)

```
Scenario: 5.000 EC2 spot instances chạy song song phân tích
          genome sequences, mỗi instance đọc/ghi file lớn ~1GB.

Workload characteristics:
  - File lớn (genome sequences)
  - Sequential I/O (đọc/ghi tuần tự)
  - Hàng nghìn clients đồng thời
  - Latency không quá quan trọng (batch job)

→ Chọn: Max I/O ✅
  Lý do: Cần aggregate IOPS cao cho hàng nghìn clients;
          sequential I/O nên latency cao hơn chấp nhận được
```

### Ví Dụ 3 — Container Orchestration (Điều Phối Container)

```
Scenario: EKS cluster với 200 pods, shared config files và logs.

Workload characteristics:
  - Mix file nhỏ (configs) và vừa (logs)
  - 200 concurrent pods
  - Latency quan trọng (microservices)

→ Chọn: General Purpose ✅
  Lý do: 200 pods không chạm giới hạn 35K IOPS;
          latency thấp quan trọng cho microservices
```

---

## 🧮 Khi Nào Chuyển Từ General Purpose Sang Max I/O?

AWS cung cấp CloudWatch metric **PercentIOLimit** — Phần Trăm Giới Hạn I/O để giám sát.

```
PercentIOLimit = (Actual IOPS / Max IOPS for GP mode) × 100

Nếu PercentIOLimit thường xuyên > 80% → Cân nhắc Max I/O
```

### CloudWatch Alarm (Cảnh Báo CloudWatch) Cho PercentIOLimit

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EFS-GP-IO-Saturation" \
  --alarm-description "EFS General Purpose tiệm cận giới hạn IOPS" \
  --metric-name PercentIOLimit \
  --namespace AWS/EFS \
  --dimensions Name=FileSystemId,Value=fs-xxxxxxxx \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --statistic Average \
  --alarm-actions arn:aws:sns:us-east-1:123456789:storage-alerts
```

> **Lưu ý:** Metric `PercentIOLimit` **chỉ có ý nghĩa** với General Purpose mode. Max I/O không có giới hạn tương đương.

---

## ⚠️ Lưu Ý Quan Trọng

### Performance Mode Không Thể Thay Đổi

```
❌ Không thể đổi Performance Mode sau khi tạo file system.

Giải pháp nếu cần đổi:
1. Tạo EFS file system mới với Performance Mode mong muốn
2. Dùng AWS DataSync để migrate dữ liệu
3. Update mount points của tất cả clients
4. Xóa file system cũ
```

### EFS Elastic Throughput Và General Purpose

```
Khuyến nghị từ AWS (2023 trở đi):
- Dùng General Purpose + Elastic Throughput Mode cho hầu hết use cases
- Max I/O chỉ khi thực sự cần > 35K IOPS với hàng nghìn clients
- Elastic Throughput tự scale throughput nên không cần Max I/O chỉ vì throughput
```

---

## 📋 Decision Tree — Cây Quyết Định Chọn Performance Mode

```
Số lượng concurrent clients (client đồng thời) của bạn?
│
├── < 1.000 clients?
│   │
│   ├── Latency có quan trọng (< 5ms)? → General Purpose ✅
│   └── Latency không quan trọng (batch)? → Vẫn nên dùng General Purpose ✅
│
└── > 1.000 clients đồng thời?
    │
    ├── Sequential I/O (đọc/ghi tuần tự file lớn)?
    │   └── → Max I/O ✅
    │
    └── Mixed workload?
        └── → Đánh giá PercentIOLimit; bắt đầu với General Purpose
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: General Purpose vs Max I/O khác nhau thế nào?**
A: General Purpose có latency thấp hơn (~1ms) nhưng giới hạn ~35K IOPS; Max I/O có latency cao hơn (~10ms+) nhưng scale không giới hạn cho hàng nghìn clients. Chọn General Purpose cho hầu hết workloads trừ khi có > 1.000 clients đồng thời cần aggregate IOPS rất cao.

**Q: Có thể đổi Performance Mode sau khi tạo không?**
A: Không — phải migrate sang file system mới dùng DataSync.

**Q: Performance Mode ảnh hưởng đến chi phí không?**
A: Không — Performance Mode không ảnh hưởng giá. Chi phí EFS tính theo dung lượng lưu trữ và throughput provisioned (nếu dùng Provisioned mode).

**Q: PercentIOLimit = 100% nghĩa là gì?**
A: File system đang bị IO-bound (bị nghẽn I/O) — đang đạt giới hạn IOPS của General Purpose mode. Cần xem xét Max I/O hoặc giảm số clients.

---

## 🔗 Điều Hướng

- [← 1-mount-targets-and-access-points.md](./1-mount-targets-and-access-points.md) — Mount Targets & Access Points
- [→ 3-throughput-modes.md](./3-throughput-modes.md) — Chế Độ Thông Lượng
- [→ README.md](./README.md) — Tổng Quan EFS
