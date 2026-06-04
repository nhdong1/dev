# Snowball Edge — Di Chuyển Dữ Liệu Quy Mô Lớn

> AWS Snowball Edge là thế hệ kế tiếp của Snowball gốc, bổ sung thêm khả năng **compute tại biên** — xử lý dữ liệu ngay trên thiết bị trước khi gửi về AWS. Có hai biến thể chính: **Storage Optimized** (tối ưu lưu trữ) và **Compute Optimized** (tối ưu tính toán), phục vụ hai nhóm use case khác nhau.

---

## Hai Biến Thể Snowball Edge

### Storage Optimized — Tối Ưu Lưu Trữ

```
Mục tiêu chính: Di chuyển dữ liệu lớn (data migration)

Thông số:
  - Dung lượng lưu trữ: 80 TB (khả dụng cho S3)
  - CPU: 40 vCPU (Intel Xeon D)
  - RAM: 80 GB
  - GPU: Không có
  - Cổng mạng: 10GbE × 3, 25GbE × 1
  - Trọng lượng: 22,3 kg

Use cases:
  - Migration 80TB+/lần (dùng nhiều thiết bị song song)
  - Backup kho lưu trữ từ on-premises lên S3
  - Di chuyển data warehouse lên S3 Data Lake
```

### Compute Optimized — Tối Ưu Tính Toán

```
Mục tiêu chính: Edge computing mạnh mẽ

Thông số:
  - Dung lượng lưu trữ: 28 TB NVMe (tốc độ cao hơn HDD)
  - CPU: 104 vCPU (Intel Xeon)
  - RAM: 416 GB
  - GPU: NVIDIA V100 (tùy chọn, 1 GPU)
  - Cổng mạng: 10GbE × 3, 25GbE × 1, 100GbE × 2
  - Trọng lượng: 22,3 kg

Use cases:
  - ML inference — suy luận mô hình AI tại biên
  - Xử lý video real-time (camera công nghiệp, drone)
  - High-performance computing ở địa điểm không có cloud
  - Genomics pipeline tại phòng lab di động
```

---

## Kiến Trúc Cluster — Cụm Thiết Bị

Snowball Edge hỗ trợ **clustering** — ghép nhiều thiết bị thành cụm:

```
Cluster Requirements:
  - Tối thiểu: 3 thiết bị
  - Tối đa: 16 thiết bị
  - Yêu cầu: Cùng loại thiết bị (Storage hoặc Compute)

Lợi ích cluster:
  ┌─────────────────────────────────────────┐
  │  3 × Storage Optimized (80TB mỗi cái)  │
  │  = 180TB usable storage (60TB parity)  │
  │  Redundant: 1 thiết bị hỏng vẫn OK     │
  └─────────────────────────────────────────┘

Distributed storage protocol: S3-compatible API
```

---

## Tính Năng Compute — Xử Lý Tại Biên

### EC2 Instances Trên Thiết Bị

```
Storage Optimized hỗ trợ:
  sbe1.small    — 1 vCPU,  1 GB RAM
  sbe1.medium   — 1 vCPU,  2 GB RAM
  sbe1.large    — 2 vCPU,  4 GB RAM
  sbe1.xlarge   — 4 vCPU,  7.5 GB RAM
  sbe1.2xlarge  — 8 vCPU,  15 GB RAM
  sbe1.4xlarge  — 16 vCPU, 30 GB RAM

Compute Optimized hỗ trợ thêm:
  sbe-c.xlarge  — 4 vCPU,  15 GB RAM
  sbe-c.2xlarge — 8 vCPU,  30 GB RAM
  sbe-c.4xlarge — 16 vCPU, 60 GB RAM
  sbe-g.xlarge  — 4 vCPU,  15 GB RAM, 1/4 NVIDIA V100
```

### AWS Lambda (Serverless Tại Biên)

```
Chạy Lambda functions không cần internet:
  - Trigger: S3 events, IoT Greengrass messages
  - Runtime: Python, Node.js, Java, Go
  - Timeout: Giống Lambda thông thường (15 phút max)

Ứng dụng:
  - Xử lý ảnh ngay khi upload lên thiết bị
  - Validate dữ liệu trước khi lưu
  - Gửi cảnh báo qua Greengrass nếu phát hiện bất thường
```

### NFS Interface — Giao Thức NFS

```
Snowball Edge hỗ trợ NFS mount:
  - Linux: mount -t nfs <device-ip>:/bucket-name /mnt/data
  - Dùng cp, rsync bình thường
  - Không cần cài AWS client

Hạn chế:
  - Throughput thấp hơn S3 API (~50% throughput)
  - Không hỗ trợ versioning trong quá trình copy
```

---

## Quy Trình Migration Thực Tế

### Kịch Bản: Di Chuyển 500TB

```
Cần bao nhiêu thiết bị?
500TB ÷ 72TB (usable per device after formatting) ≈ 7 thiết bị

Timeline:
  Ngày 1–2:   Đặt hàng 7 Snowball Edge Storage Optimized
  Ngày 10–12: Nhận thiết bị, kết nối vào mạng nội bộ
  Ngày 12–19: Copy dữ liệu (song song trên 7 thiết bị)
  Ngày 20:    Ship tất cả 7 thiết bị về AWS
  Ngày 27–30: AWS import vào S3, nhận thông báo hoàn thành

Tổng: ~30 ngày vs 536 ngày nếu dùng mạng 100Mbps
```

### Công Cụ Copy Dữ Liệu

```bash
# Cài AWS OpsHub (GUI) hoặc dùng AWS Snowball Edge Client (CLI)

# Cấu hình credentials
snowballEdge configure --endpoint https://<device-ip> \
  --manifest-file manifest.bin \
  --unlock-code <unlock-code>

# Copy toàn bộ thư mục
aws s3 cp /data/archive s3://mybucket/archive \
  --recursive \
  --endpoint-url https://<device-ip>:8443

# Copy với multipart upload cho file lớn
aws s3 cp /data/bigfile.tar.gz s3://mybucket/ \
  --endpoint-url https://<device-ip>:8443 \
  --expected-size 107374182400  # 100GB
```

---

## So Sánh Storage Optimized vs Compute Optimized

| Tiêu Chí | Storage Optimized | Compute Optimized |
|---|---|---|
| Lưu trữ | 80 TB HDD | 28 TB NVMe |
| CPU | 40 vCPU | 104 vCPU |
| RAM | 80 GB | 416 GB |
| GPU | ❌ Không | ✅ NVIDIA V100 (tùy chọn) |
| Mục tiêu | Data migration | Edge computing |
| Giá thuê | Thấp hơn | Cao hơn |
| Clustering | ✅ Hỗ trợ | ✅ Hỗ trợ |
| NFS | ✅ Có | ✅ Có |

---

## Bảo Mật Chi Tiết

```
Lớp 1 — Mã hóa dữ liệu:
  - AES-256-bit hardware encryption — mã hóa phần cứng
  - Khóa KMS — Key Management Service — không bao giờ ghi lên thiết bị
  - Dữ liệu không thể đọc nếu không có unlock code + manifest

Lớp 2 — Bảo vệ thiết bị vật lý:
  - TPM — Trusted Platform Module — Mô-đun Nền Tảng Đáng Tin Cậy
  - Secure boot — khởi động an toàn
  - Anti-tamper: phát hiện cạy mở → xóa key → dữ liệu thành rác
  - E-ink label tự động cập nhật trạng thái tracking

Lớp 3 — Kiểm soát truy cập:
  - Unlock code (29 ký tự) — riêng biệt với manifest
  - Manifest file (JSON) — chứa thông tin job
  - IAM role — kiểm soát quyền tạo job và nhận thiết bị

Sau khi AWS nhận thiết bị:
  - Xóa theo tiêu chuẩn NIST 800-88 — DoD 5220.22-M
  - Certificate of erasure (chứng nhận xóa) nếu cần
```

---

## Giá Thuê (Tham Khảo — us-east-1)

```
On-demand pricing:
  ┌────────────────────────────┬─────────────────────────────┐
  │ Storage Optimized          │ ~$300/lần dùng (10 ngày)    │
  │ Compute Optimized          │ ~$900/lần dùng (10 ngày)    │
  │ Compute Optimized + GPU    │ Liên hệ AWS Sales           │
  │ Sau 10 ngày                │ $30/ngày (Storage)          │
  │                            │ $40/ngày (Compute)          │
  │ Data transfer vào AWS      │ Miễn phí (ship về)          │
  │ Data transfer ra khỏi AWS  │ Tính theo S3 egress pricing │
  └────────────────────────────┴─────────────────────────────┘

Committed Use Discount (giảm giá cam kết dài hạn):
  - 1 năm: ~30% giảm
  - 3 năm: ~55% giảm
  - Dành cho edge deployment dài hạn (không phải migration một lần)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Khi nào dùng Storage Optimized vs Compute Optimized?**

> **Storage Optimized**: khi mục tiêu chính là di chuyển lượng lớn dữ liệu về AWS, cần tối đa storage capacity.  
> **Compute Optimized**: khi cần chạy ML inference, video processing, hoặc workload tính toán nặng ngay tại hiện trường, và dữ liệu sẽ được xử lý trước khi gửi về.

**Q: Có thể dùng Snowball Edge như NAS dài hạn không?**

> Về kỹ thuật có thể, nhưng không nên về kinh tế. Snowball Edge là sản phẩm migration/edge computing, không phải NAS. Chi phí sẽ cao hơn nhiều so với EFS hoặc NAS vật lý thực sự.

**Q: Snowball Edge cluster hoạt động như thế nào khi 1 thiết bị hỏng?**

> Cluster dùng cơ chế erasure coding — mã hóa xóa — tương tự RAID. Với cluster 3 thiết bị, có thể mất 1 thiết bị mà không mất dữ liệu. Cần gọi AWS để thay thế thiết bị hỏng.

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-05-16  
**Liên Kết:** [README](./README.md) | [Snowcone](./1-snowcone.md) | [Snowmobile](./3-snowmobile.md)
