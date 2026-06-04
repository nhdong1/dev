# 2 — Aurora Serverless v2 — Aurora Không Máy Chủ Phiên Bản 2

> Aurora Serverless v2 (Aurora Không Máy Chủ Phiên Bản 2) tự động scale (mở rộng) capacity (năng lực) lên hoặc xuống theo đơn vị ACU (Aurora Capacity Units — Đơn Vị Năng Lực Aurora) trong vòng mili-giây, không cần downtime (thời gian ngừng hoạt động). Phiên bản này giải quyết triệt để vấn đề của Serverless v1 — scaling chậm và không tương thích với nhiều tính năng Aurora.

## 📚 Mục Lục

1. [Aurora Serverless v2 Là Gì?](#aurora-serverless-v2-là-gì)
2. [ACU — Aurora Capacity Units](#acu--aurora-capacity-units)
3. [Scaling Mechanics — Cơ Chế Tự Động Co Giãn](#scaling-mechanics--cơ-chế-tự-động-co-giãn)
4. [So Sánh v1 vs v2](#so-sánh-v1-vs-v2)
5. [Serverless v2 vs Provisioned — Khi Nào Dùng Gì](#serverless-v2-vs-provisioned--khi-nào-dùng-gì)
6. [Cấu Hình và Tối Ưu](#cấu-hình-và-tối-ưu)
7. [Chi Phí Aurora Serverless v2](#chi-phí-aurora-serverless-v2)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Aurora Serverless v2 Là Gì?

Aurora Serverless v2 là tùy chọn capacity type (loại năng lực) cho Aurora cluster — thay vì chọn một instance type cố định (như `r6g.large`), bạn chỉ cần đặt **min ACU** (năng lực tối thiểu) và **max ACU** (năng lực tối đa), Aurora tự scale giữa hai giá trị này.

### Vấn Đề Serverless v2 Giải Quyết

```
Provisioned Aurora (Aurora Được Cung Cấp Cố Định):
  Black Friday: Traffic × 10  → Instance bị quá tải hoặc phải resize thủ công
  Sau 2 giờ sáng: Traffic × 0 → Instance vẫn chạy và tốn tiền

Aurora Serverless v2:
  Black Friday: Traffic × 10  → ACU tự tăng trong mili-giây
  Sau 2 giờ sáng: Traffic × 0 → ACU tự giảm xuống minimum, tiết kiệm chi phí
```

---

## ACU — Aurora Capacity Units

### ACU Là Gì?

Mỗi **ACU** (Aurora Capacity Unit — Đơn Vị Năng Lực Aurora) bao gồm:
- **2 GB RAM** (bộ nhớ)
- **CPU tương ứng** (xấp xỉ 1 vCPU)
- **Network bandwidth** (băng thông mạng) tương ứng

### Phạm Vi ACU

| Cấu Hình               | Min ACU | Max ACU | RAM tương đương        | Phù Hợp Cho                   |
| ---------------------- | ------- | ------- | ---------------------- | ------------------------------ |
| **Dev/Test nhỏ**       | 0.5     | 4       | 1 GB – 8 GB            | Môi trường phát triển          |
| **Ứng Dụng Trung Bình**| 2       | 32      | 4 GB – 64 GB           | Web apps, microservices        |
| **Production Lớn**     | 8       | 128     | 16 GB – 256 GB         | OLTP quy mô cao                |
| **Enterprise**         | 16      | 256     | 32 GB – 512 GB         | Workload cực lớn               |

> **Lưu ý:** Min ACU 0.5 (Aurora MySQL) hoặc 0.5 (Aurora PostgreSQL). Max ACU tối đa là 256 ACU (512 GB RAM).

### Tự Động Pause (Tạm Dừng) — Chỉ Có Ở v1

Aurora Serverless **v2 KHÔNG có auto-pause** (tạm dừng tự động). Khác với v1 có thể pause hoàn toàn sau N phút không có traffic, v2 luôn chạy ở minimum ACU. Lý do: pause/resume gây ra cold start (khởi động nguội) 20–30 giây — không chấp nhận được cho production.

---

## Scaling Mechanics — Cơ Chế Tự Động Co Giãn

### Cách Scaling Hoạt Động

```
Traffic Pattern (Mẫu Lưu Lượng):

  Requests/giây
       │
  500  │              ████████
  400  │           ███        ███
  300  │         ██              ██
  200  │       ██                  ██
  100  │     ██                      ██
   50  │████                            ████
    0  └───────────────────────────────────── Thời gian
       06:00  08:00  10:00  14:00  18:00  22:00

  ACU được cấp phát:
  256  │              ████████
  128  │           ███        ███
   64  │         ██              ██
   32  │       ██                  ██
   16  │     ██                      ██
    4  │████                            ████
    0  └───────────────────────────────────── Thời gian
       (Matching traffic pattern gần như tức thì)
```

### Scale-Up (Tăng Năng Lực) vs Scale-Down (Giảm Năng Lực)

| Chiều Scaling                      | Tốc Độ          | Cơ Chế                                                  |
| ---------------------------------- | --------------- | ------------------------------------------------------- |
| **Scale-up** (tăng ACU)            | Mili-giây       | Thêm RAM/CPU trong cùng instance — không restart        |
| **Scale-down** (giảm ACU)          | Từ từ hơn       | Chờ connections idle, giải phóng dần — không restart    |
| **Scale across instances**         | Không áp dụng   | v2 scale per-instance, không thêm/bớt instances         |

### Scaling Triggers (Kích Hoạt Co Giãn)

Aurora Serverless v2 monitor (giám sát) liên tục:

```
Tăng ACU khi:
  - CPU utilization (mức sử dụng CPU) > ngưỡng
  - Memory pressure (áp lực bộ nhớ) cao
  - Connection count (số kết nối) tăng
  - Replication lag tăng (nếu có Readers)

Giảm ACU khi:
  - Tất cả các chỉ số trên giảm xuống
  - Aurora tự đánh giá workload pattern
  - Giảm từ từ để tránh thrashing (dao động liên tục)
```

---

## So Sánh v1 vs v2

| Tính Năng                                              | Serverless v1       | Serverless v2         |
| ------------------------------------------------------ | ------------------- | --------------------- |
| **Tốc độ scale-up**                                    | 15–30 giây          | Mili-giây             |
| **Auto-pause** (Tạm Dừng Tự Động)                     | Có (cold start)     | Không có              |
| **Multi-AZ** (Đa Vùng Sẵn Sàng)                       | Không               | Có                    |
| **Read Replicas** (Bản Sao Đọc)                        | Không               | Có (tối đa 15)        |
| **Global Database** (Cơ Sở Dữ Liệu Toàn Cầu)         | Không               | Có                    |
| **Performance Insights** (Thông Tin Hiệu Năng)        | Không               | Có                    |
| **Backtrack** (Quay Lại Theo Thời Gian)               | Không               | Có (MySQL)            |
| **Granularity ACU** (Độ Chi Tiết ACU)                 | Bước 2 ACU          | Bước 0.5 ACU          |
| **Tương thích tính năng Aurora đầy đủ**               | Một phần            | Đầy đủ                |
| **Trạng Thái**                                         | Deprecated (Cũ)     | Hiện tại (khuyến dùng)|

---

## Serverless v2 vs Provisioned — Khi Nào Dùng Gì

### Dùng Serverless v2 Khi

```
✅ Workload có peak traffic không đều (đỉnh ban ngày, thấp ban đêm)
✅ Dev/Test environments (Môi Trường Phát Triển/Kiểm Thử) — chỉ dùng khi test
✅ Ứng dụng mới chưa biết traffic pattern
✅ Môi trường staging (dàn dựng) cần scale như production nhưng thấp hơn
✅ Multiple microservices (vi dịch vụ) với traffic không đồng đều
✅ Cần tiết kiệm chi phí vào giờ thấp điểm
```

### Dùng Provisioned Khi

```
✅ Workload ổn định, traffic dự đoán được
✅ Cần maximum predictable performance (hiệu năng dự đoán được tối đa)
✅ Real-time analytics với queries liên tục
✅ Tổng chi phí thấp hơn khi traffic high liên tục 24/7
✅ Cần instance type cụ thể (memory-optimized, compute-optimized)
```

### Công Thức Đánh Giá Chi Phí

```
Serverless v2 rẻ hơn khi:
  Giờ idle (ACU × $0.12/ACU-hour × giờ idle) < Chi phí provisioned idle

Ví dụ:
  Provisioned r6g.large: $0.26/giờ × 24 giờ = $6.24/ngày
  Serverless v2 min 0.5 ACU: $0.12 × 0.5 × 8 giờ idle + $0.12 × 8 ACU × 16 giờ peak
  = $0.48 idle + $15.36 peak = $15.84/ngày (ĐẮT HƠN nếu peak dài)

  → Dùng Serverless v2 khi idle time (thời gian rảnh) > 60% tổng thời gian
```

---

## Cấu Hình và Tối Ưu

### Thiết Lập Min/Max ACU Hợp Lý

**Min ACU** — ảnh hưởng đến:
- Kích thước buffer pool (bộ nhớ đệm) tối thiểu
- Warm cache (cache nóng) khi traffic thấp
- Chi phí tối thiểu hàng giờ

```
Hướng dẫn chọn Min ACU:
  - Dev/Test:           0.5 ACU (tiết kiệm tối đa)
  - Staging:            2 ACU  (có warm cache nhỏ)
  - Production nhỏ:     4 ACU  (buffer pool đủ dùng)
  - Production lớn:     8+ ACU (tránh cold cache sau low-traffic period)
```

**Max ACU** — ảnh hưởng đến:
- Giới hạn chi phí tối đa
- Bảo vệ tránh runaway scaling (mở rộng không kiểm soát)
- Đảm bảo performance ceiling (trần hiệu năng) dự đoán được

```
Hướng dẫn chọn Max ACU:
  - Bắt đầu với mức bạn ước tính cần cho peak × 1.5 (hệ số dự phòng)
  - Monitor CloudWatch metric: ServerlessDatabaseCapacity
  - Tăng Max ACU nếu thấy capacity thường xuyên đạt max
  - Đặt CloudWatch Alarm khi capacity > 80% Max ACU
```

### Kết Hợp Serverless v2 Với Provisioned Trong Cùng Cluster

Aurora cho phép **mixed cluster** (cụm hỗn hợp) — Writer provisioned + Readers serverless hoặc ngược lại:

```
Ví dụ 1: Writer ổn định + Readers linh hoạt
  Writer: r6g.2xlarge (provisioned) — traffic write ổn định
  Reader 1: serverless v2 (min 2, max 64) — handling report queries
  Reader 2: serverless v2 (min 2, max 32) — handling app read traffic

Ví dụ 2: Toàn bộ serverless
  Writer: serverless v2 (min 4, max 64)
  Reader 1: serverless v2 (min 2, max 32)
```

---

## Chi Phí Aurora Serverless v2

### Cách Tính Tiền

```
Chi phí = ACU-hours × $0.12/ACU-hour (giá tham khảo us-east-1)

Ví dụ:
  8:00–20:00 (12 giờ peak): 16 ACU → 16 × 12 = 192 ACU-hours
  20:00–08:00 (12 giờ off): 2 ACU  → 2  × 12 = 24  ACU-hours
  Tổng/ngày: 216 ACU-hours × $0.12 = $25.92/ngày

  So với provisioned r6g.4xlarge (16 vCPU, 128 GB): $1.28/giờ × 24 = $30.72/ngày
  → Serverless v2 tiết kiệm ~16% trong ví dụ này
```

### Chi Phí Storage (Lưu Trữ) — Không Liên Quan Đến Serverless/Provisioned

- Storage: $0.10/GB-month (giống mọi Aurora cluster)
- I/O: Tùy chọn Standard ($0.20/1M I/O requests) hoặc I/O-Optimized ($0.225/GB-month, không tính I/O)

---

## Câu Hỏi Phỏng Vấn

### Q1: Aurora Serverless v2 khác v1 ở điểm gì quan trọng nhất?

**Trả lời:** Điểm khác biệt quan trọng nhất là tốc độ và tính năng. v1 scale trong 15–30 giây (quá chậm cho traffic spikes đột ngột) và thiếu nhiều tính năng Aurora (không có Multi-AZ, Global Database, Performance Insights). v2 scale trong mili-giây nhờ thay đổi memory/CPU trong cùng instance thay vì spin up instance mới, và hỗ trợ đầy đủ tính năng Aurora. v1 đã bị deprecated — mọi workload mới nên dùng v2.

### Q2: Tại sao Aurora Serverless v2 không có auto-pause?

**Trả lời:** Auto-pause yêu cầu cold start khi nhận request đầu tiên sau khi pause — Aurora Serverless v1 mất 20–30 giây để resume, không chấp nhận được cho production. Aurora Serverless v2 không có auto-pause: instance luôn chạy ở minimum ACU. Nếu muốn tiết kiệm tối đa khi không dùng (ví dụ dev/test environment), tắt cluster hoàn toàn ngoài giờ làm việc bằng AWS Lambda schedule.

### Q3: Khi nào Serverless v2 KHÔNG phải là lựa chọn tốt?

**Trả lời:** (1) Workload ổn định 24/7 — provisioned rẻ hơn vì bạn luôn trả minimum ACU dù có traffic hay không; (2) Workload cần instance type đặc biệt như memory-optimized r6gd hoặc compute-optimized c6g; (3) Ứng dụng cực kỳ latency-sensitive không muốn bất kỳ scaling overhead nào (dù scale là mili-giây); (4) Khi budget cứng: max ACU giới hạn performance ceiling, trong khi provisioned cho performance dự đoán được hoàn toàn.

### Q4: Làm sao biết Min/Max ACU đang đặt có hợp lý không?

**Trả lời:** Monitor metric `ServerlessDatabaseCapacity` trong CloudWatch. Đặt alarm: nếu capacity thường xuyên đạt Max ACU → tăng Max ACU để tránh bị throttled. Nếu capacity không bao giờ vượt 30% Max ACU → giảm Max để giới hạn chi phí. Với Min ACU: nếu thấy latency spike khi traffic tăng đột ngột sau low-traffic period → tăng Min ACU để buffer pool không quá nhỏ.

---

## 🔗 Điều Hướng

| Trước                                                | Tiếp Theo                                    |
| ---------------------------------------------------- | -------------------------------------------- |
| [1-aurora-architecture.md](./1-aurora-architecture.md) | [3-aurora-global.md](./3-aurora-global.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
