# EFS Throughput Modes — Chế Độ Thông Lượng

> Throughput Mode (Chế Độ Thông Lượng) của EFS quyết định lượng băng thông I/O — Input/Output (Đầu Vào/Đầu Ra) tối đa mà file system có thể cung cấp. Không như Performance Mode, Throughput Mode **có thể thay đổi sau khi tạo** (sau mỗi 24 giờ).

---

## 📌 Ba Chế Độ Thông Lượng

| | Bursting | Provisioned | Elastic |
|-|----------|-------------|---------|
| **Tên Tiếng Việt** | Bùng Phát | Đã Cấp Phát | Linh Hoạt |
| **Throughput** | Scale theo dung lượng + credit | Cố định (đặt trước) | Tự động scale |
| **Chi Phí** | Tính theo lưu trữ | Thêm phí provisioned throughput | Thêm phí theo I/O thực tế |
| **Thay Đổi Được?** | Có | Có | Có |
| **AWS Khuyến Nghị** | Workload nhỏ, không ổn định | Throughput > burst allowance | Hầu hết workloads hiện đại |

---

## ⚡ Bursting Mode — Chế Độ Bùng Phát

### Cơ Chế Hoạt Động

Bursting Mode hoạt động theo mô hình **credit-based** (dựa trên tín dụng) — tương tự như EBS gp2:

```
Baseline Throughput (Thông lượng nền) = 50 KiB/s × mỗi GiB lưu trữ

Burst Throughput (Thông lượng bùng phát) = 100 MiB/s (tối thiểu)
  hoặc cao hơn nếu file system đủ lớn

Burst Credit Balance (Số dư tín dụng):
  - Tích lũy khi dùng dưới baseline
  - Tiêu thụ khi burst vượt baseline
  - Tối đa 2.1 TiB credit
```

### Công Thức Tính Burst

```
Dung lượng file system: 1 TiB (1.024 GiB)

Baseline throughput = 50 KiB/s × 1.024 GiB = 51.2 MiB/s
Burst throughput    = max(100 MiB/s, baseline × 2) = 102.4 MiB/s

Tốc độ tích lũy credit:
  = Burst throughput - Baseline throughput
  = 102.4 - 51.2 = 51.2 MiB/s dư
  → Mỗi giây idle, tích lũy được 51.2 MiB credit

Thời gian burst tối đa (full burst):
  = Total credit / (Burst - Baseline)
  = 2.1 TiB / 51.2 MiB/s ≈ 41.000 giây ≈ 11.4 giờ
```

### Burst Throughput Theo Dung Lượng

| Dung Lượng EFS | Baseline Throughput | Burst Throughput |
|----------------|--------------------|--------------------|
| 100 GiB | 5 MiB/s | 100 MiB/s (minimum) |
| 1 TiB | 50 MiB/s | 100 MiB/s |
| 10 TiB | 500 MiB/s | 500 MiB/s (= baseline) |
| 100 TiB | 5 GiB/s | 5 GiB/s |

> **Lưu ý:** Khi dung lượng > 2 TiB, baseline vượt burst minimum nên không còn credit mechanism.

### Phù Hợp Với Bursting

```
✅ File system nhỏ (< 1 TiB) với workload không liên tục
✅ Development environments — chỉ dùng thỉnh thoảng
✅ Backup storage — ghi batch ban đêm, idle ban ngày
✅ Khi chi phí là ưu tiên và throughput cao không cần liên tục
```

### Không Phù Hợp Với Bursting

```
❌ Workloads liên tục cao hơn baseline (sẽ cạn credit)
❌ File system nhỏ nhưng cần throughput cao bền vững
❌ Production workloads cần throughput đảm bảo (predictable)
```

### Giám Sát Burst Credits

```bash
# CloudWatch metric quan trọng cho Bursting mode
BurstCreditBalance:
  - Đơn vị: bytes
  - 2.1 TiB = 2.307.072.000.000 bytes (đầy)
  - Khi về 0 → throughput bị giới hạn xuống baseline

# Tạo alarm khi credit thấp
aws cloudwatch put-metric-alarm \
  --alarm-name "EFS-BurstCredit-Low" \
  --metric-name BurstCreditBalance \
  --namespace AWS/EFS \
  --dimensions Name=FileSystemId,Value=fs-xxxxxxxx \
  --period 300 \
  --threshold 1073741824000 \
  --comparison-operator LessThanThreshold \
  --statistic Minimum \
  --alarm-actions arn:aws:sns:us-east-1:123456789:alerts
```

---

## 🎯 Provisioned Throughput Mode — Chế Độ Đã Cấp Phát

### Cơ Chế Hoạt Động

Provisioned Throughput cho phép đặt trước một lượng throughput **cố định**, bất kể dung lượng file system. Dùng khi cần throughput cao hơn mức Bursting có thể cung cấp dựa trên dung lượng.

```
Bạn đặt: 500 MiB/s throughput
File system dung lượng: 100 GiB (baseline chỉ 5 MiB/s trong Bursting)

→ Với Provisioned: đảm bảo 500 MiB/s bất kể burst credit
```

### Giới Hạn Provisioned

| Tham Số | Giới Hạn |
|---------|----------|
| Tối đa Provisioned Throughput | 3 GiB/s (đọc) / 1 GiB/s (ghi) |
| Tối thiểu | 1 MiB/s |
| Thay đổi throughput | Sau mỗi 24 giờ |

### Chi Phí Provisioned Throughput

```
Chi phí = Phí lưu trữ + Phí throughput provisioned

Phí throughput provisioned (us-east-1):
  = $6.00/MiB/s/month (cho phần vượt quá baseline Bursting)

Ví dụ:
  Provisioned: 500 MiB/s
  File system: 100 GiB → Baseline bursting = 5 MiB/s
  
  Phần tính phí = 500 - 5 = 495 MiB/s
  Chi phí throughput/tháng = 495 × $6.00 = $2.970/tháng
```

### Phù Hợp Với Provisioned

```
✅ File system nhỏ nhưng cần throughput cao và bền vững
✅ Video encoding/streaming — cần throughput ổn định liên tục
✅ Database backups — throughput dự đoán được
✅ Workloads cần SLA throughput cụ thể
```

---

## 🔄 Elastic Throughput Mode — Chế Độ Linh Hoạt

### Cơ Chế Hoạt Động

Elastic Throughput (ra mắt 2022) là chế độ **serverless** — tự động scale throughput theo nhu cầu thực tế, không cần quản lý credit hay đặt trước.

```
Throughput tự động scale:
  - Đọc:  lên đến 3 GiB/s (per file system)
  - Ghi:  lên đến 1 GiB/s (per file system)

Tính phí:
  - Theo lượng dữ liệu thực sự đọc/ghi (GiB transferred)
  - Không tính phí khi idle
```

### Chi Phí Elastic Throughput

```
Chi phí = Phí lưu trữ + Phí I/O thực tế

Phí I/O (us-east-1):
  Đọc:  $0.03/GiB transferred (chỉ đọc từ EFS Standard IA và One Zone IA)
  Ghi:  $0.06/GiB transferred

Lưu ý: Đọc từ EFS Standard KHÔNG tính phí I/O trong Elastic mode
```

### Phù Hợp Với Elastic (AWS Khuyến Nghị)

```
✅ Hầu hết workloads hiện đại
✅ Workloads không dự đoán được (spiky traffic)
✅ Microservices, containerized apps
✅ Development environments
✅ Khi không muốn quản lý burst credits hay provisioned throughput
```

### Không Phù Hợp Với Elastic

```
❌ Workloads đọc rất nhiều từ IA storage class (tốn phí $0.03/GiB)
❌ Khi throughput cần tuyến tính với dung lượng (Bursting tốt hơn)
```

---

## 💰 So Sánh Chi Phí — Ví Dụ Thực Tế

### Scenario: File System 500 GiB, Cần 200 MiB/s Throughput Liên Tục

#### Với Bursting Mode

```
Baseline throughput = 50 KiB/s × 500 GiB = 25 MiB/s
Burst throughput    = 100 MiB/s (minimum)

→ Burst = 100 MiB/s < 200 MiB/s yêu cầu
→ Bursting KHÔNG đáp ứng được yêu cầu ❌
```

#### Với Provisioned Mode

```
Provisioned: 200 MiB/s
Baseline từ file system: 25 MiB/s

Phần tính phí = 200 - 25 = 175 MiB/s
Chi phí throughput = 175 × $6.00 = $1.050/tháng
Chi phí storage    = 500 GiB × $0.30 = $150/tháng
Tổng: $1.200/tháng
```

#### Với Elastic Mode

```
Giả sử workload:
  - 8 giờ/ngày đọc 200 MiB/s từ Standard storage
  - Không đọc từ IA storage

Chi phí đọc = 200 MiB/s × 3600s × 8h × 30 ngày
            = 200 × 3600 × 8 × 30 = 1.728.000 MiB/tháng
            = 1.688 GiB/tháng

Phí I/O = $0 (đọc từ Standard miễn phí trong Elastic)
Chi phí storage = 500 GiB × $0.30 = $150/tháng
Tổng: $150/tháng ✅ (rẻ hơn Provisioned rất nhiều)
```

---

## 📊 Bảng Quyết Định — Decision Matrix

| Tiêu Chí | Bursting | Provisioned | Elastic |
|----------|----------|-------------|---------|
| File system < 1 TiB, workload nhẹ | ✅ Tốt nhất | Đắt hơn cần | Tốt |
| File system lớn, workload đều đặn | ✅ Baseline cao | Phụ thuộc | ✅ |
| Cần throughput cao hơn baseline liên tục | ❌ | ✅ Tốt nhất | ✅ |
| Workload không dự đoán được (spiky) | Có thể hết credit | Lãng phí phí | ✅ Tốt nhất |
| Đọc nhiều từ IA storage | Không liên quan | Không liên quan | ❌ Tốn phí |
| Serverless / no management | ❌ Cần giám sát | ❌ Cần planning | ✅ Tốt nhất |

---

## 🔄 Chuyển Đổi Giữa Các Throughput Mode

Khác với Performance Mode, Throughput Mode **có thể thay đổi** nhưng phải chờ **24 giờ** giữa các lần thay đổi.

```bash
# Chuyển sang Elastic Throughput
aws efs update-file-system \
  --file-system-id fs-xxxxxxxx \
  --throughput-mode elastic

# Chuyển sang Provisioned Throughput (500 MiB/s)
aws efs update-file-system \
  --file-system-id fs-xxxxxxxx \
  --throughput-mode provisioned \
  --provisioned-throughput-in-mibps 500

# Chuyển sang Bursting
aws efs update-file-system \
  --file-system-id fs-xxxxxxxx \
  --throughput-mode bursting
```

---

## 📈 CloudWatch Metrics Quan Trọng

| Metric | Áp Dụng Cho | Ý Nghĩa |
|--------|------------|---------|
| `BurstCreditBalance` | Bursting | Số credit còn lại (bytes) |
| `PermittedThroughput` | Tất cả | Throughput tối đa hiện tại (MiB/s) |
| `MeteredIOBytes` | Tất cả | Lượng I/O thực tế (bytes/giây) |
| `TotalIOBytes` | Tất cả | Tổng I/O kể cả metadata |

### Giám Sát Throughput Utilization

```bash
# Xem PermittedThroughput — throughput tối đa được phép
aws cloudwatch get-metric-statistics \
  --namespace AWS/EFS \
  --metric-name PermittedThroughput \
  --dimensions Name=FileSystemId,Value=fs-xxxxxxxx \
  --start-time 2026-05-16T00:00:00Z \
  --end-time 2026-05-16T23:59:59Z \
  --period 300 \
  --statistics Average
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Khi nào dùng Provisioned thay vì Bursting?**
A: Khi throughput cần thiết vượt quá mức burst cho phép dựa trên dung lượng file system. Ví dụ: file system 100 GiB chỉ burst được 100 MiB/s, nhưng ứng dụng cần 500 MiB/s liên tục → cần Provisioned.

**Q: Elastic Throughput tính phí như thế nào?**
A: Tính theo lượng dữ liệu thực tế đọc/ghi (GiB transferred). Đọc từ Standard storage không tính phí trong Elastic mode; ghi tính $0.06/GiB; đọc từ IA tính $0.03/GiB.

**Q: BurstCreditBalance về 0 thì sao?**
A: Throughput bị giới hạn xuống baseline (50 KiB/s × GiB). Workload sẽ chậm đột ngột. Cần chuyển sang Provisioned hoặc Elastic, hoặc tăng dung lượng file system để tăng baseline.

**Q: Có thể thay đổi Throughput Mode bao lâu một lần?**
A: Tối thiểu mỗi 24 giờ thay đổi một lần.

---

## 🔗 Điều Hướng

- [← 2-performance-modes.md](./2-performance-modes.md) — Chế Độ Hiệu Suất
- [→ 4-efs-intelligent-tiering.md](./4-efs-intelligent-tiering.md) — Phân Tầng Thông Minh
- [→ README.md](./README.md) — Tổng Quan EFS
