# EFS Intelligent-Tiering — Phân Tầng Thông Minh

> EFS Intelligent-Tiering (Phân Tầng Thông Minh) tự động di chuyển file giữa các storage class — Standard (Tiêu Chuẩn) và IA — Infrequent Access (Truy Cập Không Thường Xuyên) dựa trên pattern truy cập thực tế. Mục tiêu: giảm chi phí mà không cần quản lý thủ công.

---

## 📌 EFS Storage Classes — Các Lớp Lưu Trữ EFS

| Storage Class | Giá/GB/tháng (us-east-1) | Phí Truy Xuất | Dùng Cho |
|---------------|--------------------------|---------------|---------|
| **EFS Standard** | ~$0.30 | Miễn phí | Dữ liệu truy cập thường xuyên |
| **EFS Standard-IA** | ~$0.016 | ~$0.01/GiB đọc | Dữ liệu ít truy cập, đa-AZ |
| **EFS One Zone** | ~$0.16 | Miễn phí | Dev/test, một AZ |
| **EFS One Zone-IA** | ~$0.008 | ~$0.01/GiB đọc | Dữ liệu lạnh, một AZ |

> **EFS Standard** đắt gấp ~19 lần so với **EFS Standard-IA** — tiết kiệm lớn nếu có nhiều dữ liệu ít truy cập.

---

## 🔄 Cơ Chế Hoạt Động Intelligent-Tiering

### Luồng Di Chuyển Dữ Liệu

```
File được tạo/truy cập
         │
         ▼
  EFS Standard (Hot)
   (Đang hoạt động)
         │
         │  Không truy cập trong N ngày
         │  (lifecycle policy)
         ▼
EFS Standard-IA (Cold)
   (Lưu trữ lạnh)
         │
         │  Có truy cập lại
         ▼
  EFS Standard (Hot)
   (Trở về Standard)
```

### Lifecycle Policy — Chính Sách Vòng Đời

EFS hỗ trợ hai loại lifecycle policy:

#### 1. Transition to IA (Chuyển vào IA)

File không được truy cập trong khoảng thời gian đã đặt → tự động chuyển sang IA.

| Tùy Chọn | Mô Tả |
|-----------|-------|
| `AFTER_7_DAYS` | Sau 7 ngày không truy cập |
| `AFTER_14_DAYS` | Sau 14 ngày không truy cập |
| `AFTER_30_DAYS` | Sau 30 ngày không truy cập |
| `AFTER_60_DAYS` | Sau 60 ngày không truy cập |
| `AFTER_90_DAYS` | Sau 90 ngày không truy cập |

#### 2. Transition to Primary Storage (Chuyển về Standard)

Khi file trong IA được truy cập → tự động chuyển về Standard để tối ưu cho các lần truy cập tiếp theo.

| Tùy Chọn | Mô Tả |
|-----------|-------|
| `AFTER_1_ACCESS` | Sau lần đầu tiên truy cập (mặc định) |
| Tắt | File không bao giờ trở về Standard (tiết kiệm hơn nếu ít truy cập) |

---

## ⚙️ Cấu Hình Intelligent-Tiering

### Bật Khi Tạo File System

```bash
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --lifecycle-policies \
    TransitionToIA=AFTER_30_DAYS \
    TransitionToPrimaryStorageClass=AFTER_1_ACCESS \
  --tags Key=Name,Value=my-intelligent-tiering-efs
```

### Bật/Đổi Lifecycle Policy Trên File System Hiện Có

```bash
# Bật chuyển sang IA sau 30 ngày và chuyển về Standard khi truy cập
aws efs put-lifecycle-configuration \
  --file-system-id fs-xxxxxxxx \
  --lifecycle-policies \
    '[
      {"TransitionToIA": "AFTER_30_DAYS"},
      {"TransitionToPrimaryStorageClass": "AFTER_1_ACCESS"}
    ]'

# Chỉ bật chuyển sang IA (không chuyển về Standard)
aws efs put-lifecycle-configuration \
  --file-system-id fs-xxxxxxxx \
  --lifecycle-policies \
    '[{"TransitionToIA": "AFTER_14_DAYS"}]'

# Xem cấu hình hiện tại
aws efs describe-lifecycle-configuration \
  --file-system-id fs-xxxxxxxx

# Tắt Intelligent-Tiering
aws efs put-lifecycle-configuration \
  --file-system-id fs-xxxxxxxx \
  --lifecycle-policies '[]'
```

---

## 💰 Phân Tích Chi Phí Tiết Kiệm

### Ví Dụ Thực Tế

```
Scenario:
  Tổng dữ liệu: 10 TiB
  Dữ liệu hot (truy cập thường xuyên): 20% = 2 TiB
  Dữ liệu cold (ít truy cập): 80% = 8 TiB

Không dùng Intelligent-Tiering (tất cả Standard):
  Chi phí = 10.240 GiB × $0.30 = $3.072/tháng

Dùng Intelligent-Tiering:
  Standard: 2.048 GiB × $0.30 = $614
  IA:       8.192 GiB × $0.016 = $131
  Phí đọc IA (giả sử 100 GiB/tháng đọc từ IA):
            100 GiB × $0.01 = $1
  Tổng: $746/tháng

Tiết kiệm: $3.072 - $746 = $2.326/tháng (76% tiết kiệm)
```

### Break-Even Analysis (Phân Tích Điểm Hòa Vốn)

```
Intelligent-Tiering có lợi khi:
  File được truy cập < X lần/tháng

Chi phí Standard cho 1 GiB: $0.30/tháng
Chi phí IA cho 1 GiB: $0.016 + ($0.01 × số lần đọc)

Điểm hòa vốn:
  $0.016 + $0.01 × N = $0.30
  $0.01 × N = $0.284
  N = 28.4 lần đọc/tháng

→ Nếu 1 GiB dữ liệu được đọc < 28 lần/tháng → IA tiết kiệm hơn
→ Nếu 1 GiB dữ liệu được đọc > 28 lần/tháng → Standard tốt hơn
```

---

## 🔍 Hành Vi Quan Trọng Cần Hiểu

### File Nào Được Chuyển Vào IA?

```
✅ File không được đọc/ghi trong N ngày
✅ File với kích thước > 128 bytes (file quá nhỏ không được chuyển)

❌ Thư mục (directories) — không được chuyển
❌ File đang mở (open files)
❌ File symlinks (liên kết tượng trưng)
❌ File < 128 bytes
❌ Metadata (thông tin file) — luôn ở Standard
```

### Thời Gian Chuyển Đổi

```
Chuyển Standard → IA:
  - Quét định kỳ (thường ngay sau mỗi ngày UTC)
  - Không phải đúng N ngày — có thể trễ vài giờ

Chuyển IA → Standard (sau 1 lần truy cập):
  - Xảy ra gần như tức thì sau lần đọc
  - File sẽ ở Standard cho các lần truy cập tiếp theo
```

### Latency Khi Đọc Từ IA

```
EFS Standard:   ~1ms latency (đối với General Purpose mode)
EFS Standard-IA: Tương đương — không có first-byte latency cao như S3 Glacier
                 (EFS IA không cần "restore" như S3 Glacier)
```

> Đây là điểm khác biệt quan trọng: EFS IA **không có retrieval delay** (độ trễ truy xuất) như S3 Glacier. File trong EFS IA được đọc ngay lập tức.

---

## 📊 Giám Sát Intelligent-Tiering

### CloudWatch Metrics Quan Trọng

| Metric | Ý Nghĩa |
|--------|---------|
| `StorageBytes` với dimension `StorageClass=Standard` | Dung lượng ở Standard |
| `StorageBytes` với dimension `StorageClass=StandardIA` | Dung lượng ở IA |
| `MeteredIOBytes` với `StorageClass=StandardIA` | I/O từ IA (tính phí) |

### Xem Phân Bố Storage Class

```bash
# Xem storage classes breakdown
aws efs describe-file-systems \
  --file-system-id fs-xxxxxxxx \
  --query 'FileSystems[0].SizeInBytes'

# Output:
{
  "Value": 107374182400,        # Tổng (bytes)
  "Timestamp": "2026-05-16T00:00:00+00:00",
  "ValueInIA": 85899345920,     # Dữ liệu trong IA (bytes)
  "ValueInStandard": 21474836480 # Dữ liệu trong Standard (bytes)
}
```

### Dashboard CloudWatch

```bash
# Metric để theo dõi % dữ liệu trong IA
CloudWatch Metric Math:
  IA_Percentage = StorageBytes(IA) / StorageBytes(Total) × 100

# Nếu IA_Percentage thấp (<30%) với lifecycle 30 ngày
# → Dữ liệu đang được truy cập thường xuyên → IA không tiết kiệm nhiều
# → Cân nhắc tắt Intelligent-Tiering
```

---

## 🏗️ One Zone vs Standard Intelligent-Tiering

### One Zone-IA — Lựa Chọn Rẻ Nhất

| | Standard + Standard-IA | One Zone + One Zone-IA |
|-|------------------------|------------------------|
| **Durability** | 99.999999999% (đa-AZ) | 99.999999999% (một AZ) |
| **Availability** | 99.99% | 99.9% |
| **Standard price** | $0.30/GiB | $0.16/GiB |
| **IA price** | $0.016/GiB | $0.008/GiB |
| **Use case** | Production | Dev/test, reproducible data |

```bash
# Tạo One Zone file system với Intelligent-Tiering
aws efs create-file-system \
  --availability-zone-name us-east-1a \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --lifecycle-policies \
    TransitionToIA=AFTER_14_DAYS \
    TransitionToPrimaryStorageClass=AFTER_1_ACCESS
```

---

## 🔑 Các Chiến Lược Dùng EFS Intelligent-Tiering

### Chiến Lược 1: Maximize IA (Tối Đa Hóa Dữ Liệu IA)

```
Mục tiêu: Tiết kiệm chi phí tối đa
Cấu hình:
  - TransitionToIA: AFTER_7_DAYS (rất nhanh)
  - TransitionToPrimaryStorageClass: Tắt (không chuyển về Standard)
  
Phù hợp: Archive storage, backup, log retention
Rủi ro: Phí đọc từ IA tích lũy nếu truy cập nhiều
```

### Chiến Lược 2: Balanced (Cân Bằng) — Khuyến Nghị

```
Mục tiêu: Cân bằng chi phí và hiệu suất
Cấu hình:
  - TransitionToIA: AFTER_30_DAYS
  - TransitionToPrimaryStorageClass: AFTER_1_ACCESS
  
Phù hợp: Hầu hết workloads production
Lợi ích: Dữ liệu hot vẫn ở Standard; dữ liệu lạnh tự động xuống IA
```

### Chiến Lược 3: Conservative (Thận Trọng)

```
Mục tiêu: Đảm bảo hiệu suất, tiết kiệm vừa phải
Cấu hình:
  - TransitionToIA: AFTER_90_DAYS
  - TransitionToPrimaryStorageClass: AFTER_1_ACCESS
  
Phù hợp: Workload cần latency ổn định; không chắc pattern truy cập
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: EFS IA có latency cao hơn Standard không?**
A: Không đáng kể — EFS IA không yêu cầu restore như S3 Glacier. Latency đọc từ IA tương đương Standard (~1ms cho General Purpose). Điểm khác biệt là phí đọc ($0.01/GiB).

**Q: File tối thiểu bao nhiêu bytes mới được chuyển sang IA?**
A: 128 bytes. File nhỏ hơn luôn ở Standard.

**Q: Khi nào Intelligent-Tiering không tiết kiệm?**
A: Khi dữ liệu thường xuyên di chuyển giữa Standard và IA (truy cập liên tục nhưng không đều), phí đọc từ IA và phí chuyển tầng có thể vượt khoản tiết kiệm. Phân tích access pattern trước khi bật.

**Q: Intelligent-Tiering ảnh hưởng đến ứng dụng không?**
A: Không — ứng dụng không cần thay đổi gì. Mount path, file path đều giống nhau. AWS tự xử lý việc đọc từ đúng storage class.

---

## 🔗 Điều Hướng

- [← 3-throughput-modes.md](./3-throughput-modes.md) — Chế Độ Thông Lượng
- [→ 5-efs-vs-ebs-vs-s3.md](./5-efs-vs-ebs-vs-s3.md) — Bảng So Sánh Đầy Đủ
- [→ README.md](./README.md) — Tổng Quan EFS
