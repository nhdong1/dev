# 💰 Tối Ưu Chi Phí Lưu Trữ AWS — Tổng Quan

> Hướng dẫn toàn diện về chiến lược tối ưu chi phí cho S3, EBS và EFS — từ hiểu bảng giá đến thiết kế lifecycle rules và phân tích với Storage Lens.

## 📚 Mục Lục Chủ Đề

| File | Chủ Đề | Mức Độ |
|------|--------|--------|
| [1-s3-pricing-guide.md](./1-s3-pricing-guide.md) | Bảng Giá S3 — Storage, Requests, Data Transfer | Cơ bản |
| [2-ebs-pricing-guide.md](./2-ebs-pricing-guide.md) | Bảng Giá EBS — gp3 vs io2 vs st1 — Tính Toán | Trung cấp |
| [3-lifecycle-rules-design.md](./3-lifecycle-rules-design.md) | Thiết Kế Lifecycle Rules — Ví Dụ Thực Tế | Trung cấp |
| [4-intelligent-tiering-when.md](./4-intelligent-tiering-when.md) | Khi Nào Intelligent-Tiering Tiết Kiệm Chi Phí | Trung cấp |
| [5-cost-allocation-tags.md](./5-cost-allocation-tags.md) | Thẻ Phân Bổ Chi Phí — Phân Tích Theo Team/Project | Cơ bản |
| [6-s3-storage-lens.md](./6-s3-storage-lens.md) | Storage Lens Dashboard — Phân Tích Toàn Tổ Chức | Nâng cao |

---

## 🎯 Tại Sao Tối Ưu Chi Phí Lưu Trữ Quan Trọng

### Chi Phí Lưu Trữ Tăng Theo Thời Gian

```
Dữ liệu tích lũy theo thời gian:

Tháng 1:   ████ 10 TB   →  $230/tháng (S3 Standard)
Tháng 6:   ████████ 25 TB   →  $575/tháng
Tháng 12:  ██████████████ 50 TB  →  $1,150/tháng
Năm 2:     ████████████████████ 100 TB →  $2,300/tháng

Không tối ưu → Chi phí tăng tuyến tính với dữ liệu
Có lifecycle rules → Chi phí tăng chậm hơn nhiều
```

### Cơ Hội Tiết Kiệm Điển Hình

| Loại Tối Ưu | Tiết Kiệm Điển Hình | Độ Phức Tạp |
|-------------|---------------------|-------------|
| Lifecycle rules chuyển sang Glacier | 60–80% chi phí lưu trữ dữ liệu cũ | Thấp |
| Xóa object hết hạn | 100% của dữ liệu vô dụng | Thấp |
| EBS gp2 → gp3 migration | 20% giảm ngay lập tức | Rất thấp |
| Xóa EBS snapshot cũ | 100% của snapshot không cần | Thấp |
| S3 Intelligent-Tiering | 20–40% cho dữ liệu không đoán được | Thấp |
| Cost Allocation Tags | Không tiết kiệm trực tiếp nhưng tạo visibility | Trung bình |
| Storage Lens insights | Giúp phát hiện cơ hội tiết kiệm | Cao |

---

## 🗺️ Mô Hình Chi Phí AWS Storage

### Ba Trụ Cột Chi Phí S3

```
┌────────────────────────────────────────────────────┐
│              CHI PHÍ S3 = A + B + C                │
├────────────────┬───────────────┬───────────────────┤
│   A: Lưu Trữ  │  B: Requests  │  C: Data Transfer │
│   (Storage)   │  (API Calls)  │   (Egress)        │
│               │               │                   │
│ $0.023/GB/mon │ GET: $0.0004  │ $0.09/GB ra       │
│ (Standard)    │ per 1,000     │ internet          │
│               │               │                   │
│ Giảm bằng:    │ Giảm bằng:    │ Giảm bằng:        │
│ - Lifecycle   │ - Cache ở CDN │ - CloudFront      │
│ - Glacier     │ - Batch ops   │ - S3 Transfer     │
│ - Tiering     │               │   Acceleration    │
└────────────────┴───────────────┴───────────────────┘
```

### Chi Phí EBS — Cấu Phần

```
EBS Cost = Storage Provisioned + IOPS + Throughput + Snapshots

gp3 ($0.08/GB-month):
  ├── 3,000 IOPS miễn phí
  ├── $0.005/IOPS-month (>3,000 IOPS)
  └── $0.04/MB/s-month (>125 MB/s)

io2 ($0.125/GB-month):
  └── $0.065/IOPS-month (mọi IOPS)

Snapshot: $0.05/GB-month (dung lượng thực, incremental)
```

---

## 🏆 Chiến Lược Tối Ưu Chi Phí Theo Ưu Tiên

### Ưu Tiên 1 — Quick Wins (Thắng Lợi Nhanh, <1 tuần)

```
1. Chuyển EBS gp2 → gp3 (tiết kiệm 20% ngay)
2. Xóa EBS snapshot hơn 30 ngày không dùng
3. Xóa EBS volume ở trạng thái "available" (không gắn EC2)
4. Bật S3 lifecycle rule xóa object hết hạn
5. Xóa Elastic IP chưa gắn (không phải storage nhưng thường đi kèm)
```

### Ưu Tiên 2 — Medium Term (1–4 tuần)

```
1. Thiết kế lifecycle policy đa tầng cho S3
2. Đánh giá dữ liệu nào phù hợp Intelligent-Tiering
3. Triển khai Cost Allocation Tags
4. Cấu hình Cost Anomaly Detection
5. Review EFS throughput mode (Elastic thay Provisioned nếu burst không đều)
```

### Ưu Tiên 3 — Strategic (1–3 tháng)

```
1. Bật S3 Storage Lens và phân tích toàn tổ chức
2. Kiến trúc data lake với tiering đúng loại
3. Tính toán Reserved Instance cho EBS io2 workloads ổn định
4. Xem xét S3 Batch Operations để cleanup hàng loạt
5. Thiết lập FinOps (Financial Operations) framework
```

---

## 📊 So Sánh Chi Phí Dịch Vụ Lưu Trữ

### Giá Tham Khảo (us-east-1, 2026)

| Dịch Vụ | Giá Lưu Trữ | Ghi Chú |
|---------|-------------|---------|
| S3 Standard | $0.023/GB-month | Truy cập thường xuyên |
| S3 Standard-IA | $0.0125/GB-month | Phí retrieve $0.01/GB |
| S3 Glacier Instant | $0.004/GB-month | Phí retrieve $0.03/GB |
| S3 Glacier Flexible | $0.0036/GB-month | Retrieve 3–5 giờ |
| S3 Glacier Deep Archive | $0.00099/GB-month | Retrieve 12 giờ |
| EBS gp3 | $0.08/GB-month | Block storage |
| EBS io2 | $0.125/GB-month | + $0.065/IOPS |
| EFS Standard | $0.30/GB-month | NFS shared |
| EFS Standard-IA | $0.025/GB-month | Infrequent Access |

> **Lưu ý:** Giá có thể thay đổi theo vùng và theo thời gian. Luôn kiểm tra tại [aws.amazon.com/pricing](https://aws.amazon.com/pricing).

---

## 🔍 Framework Phân Tích Chi Phí

### Bước 1 — Baseline (Đường Cơ Sở)

```bash
# Xem chi phí S3 theo bucket
aws ce get-cost-and-usage \
  --time-period Start=2026-01-01,End=2026-02-01 \
  --granularity MONTHLY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Simple Storage Service"]}}' \
  --group-by '[{"Type":"DIMENSION","Key":"USAGE_TYPE"}]'

# Xem dung lượng từng bucket
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  xargs -I{} aws cloudwatch get-metric-statistics \
    --namespace AWS/S3 \
    --metric-name BucketSizeBytes \
    --dimensions Name=BucketName,Value={} Name=StorageType,Value=StandardStorage \
    --start-time 2026-01-01T00:00:00Z \
    --end-time 2026-02-01T00:00:00Z \
    --period 86400 \
    --statistics Average
```

### Bước 2 — Identify Opportunities (Tìm Cơ Hội)

```
Câu hỏi cần trả lời:
□ Bucket nào lớn nhất? Dữ liệu đó được truy cập bao nhiêu lần/ngày?
□ EBS volume nào có utilization thấp (<10 IOPS trung bình)?
□ Snapshot cũ nào >90 ngày vẫn tồn tại?
□ EFS mount nào không có traffic tuần qua?
□ Dữ liệu nào không được truy cập >30 ngày?
```

### Bước 3 — Implement & Measure (Triển Khai & Đo)

```
Sau mỗi thay đổi:
1. Đợi 1 billing cycle (tháng)
2. So sánh chi phí với baseline
3. Tính ROI (Return on Investment — Lợi nhuận trên vốn đầu tư)
4. Điều chỉnh nếu có tác động ngoài ý muốn
```

---

## ⚠️ Rủi Ro Khi Tối Ưu Chi Phí

### Rủi Ro Thường Gặp

| Rủi Ro | Hậu Quả | Phòng Tránh |
|--------|---------|-------------|
| Xóa data cần thiết | Mất dữ liệu vĩnh viễn | Test lifecycle trên bucket test trước |
| Chuyển sang Glacier quá sớm | Retrieve cost cao bất ngờ | Phân tích access pattern trước |
| Intelligent-Tiering cho object nhỏ | Phí monitoring > tiết kiệm | Chỉ dùng cho object >128KB |
| Giảm IOPS EBS đột ngột | Ứng dụng chậm/crash | Thay đổi từ từ, monitor |
| Xóa snapshot | Không restore được | Dùng DLM — Data Lifecycle Manager thay xóa thủ công |

### Kiểm Tra Trước Khi Tối Ưu

```
□ Backup hiện tại của dữ liệu có hoạt động không?
□ Ai là owner của dữ liệu? Họ đồng ý thay đổi chưa?
□ Có compliance requirement nào về retention không?
□ Dữ liệu có được dùng trong CI/CD pipeline không?
□ Có SLA nào về access time cho dữ liệu này không?
```

---

## 🔗 Điều Hướng

| Chủ Đề Tiếp Theo | Link |
|-----------------|------|
| Bảng giá S3 chi tiết | [1-s3-pricing-guide.md](./1-s3-pricing-guide.md) |
| Bảng giá EBS | [2-ebs-pricing-guide.md](./2-ebs-pricing-guide.md) |
| Thiết kế lifecycle rules | [3-lifecycle-rules-design.md](./3-lifecycle-rules-design.md) |
| Intelligent-Tiering khi nào | [4-intelligent-tiering-when.md](./4-intelligent-tiering-when.md) |
| Cost Allocation Tags | [5-cost-allocation-tags.md](./5-cost-allocation-tags.md) |
| S3 Storage Lens | [6-s3-storage-lens.md](./6-s3-storage-lens.md) |
| Monitoring chi phí | [../07-monitoring/README.md](../07-monitoring/README.md) |
| Bảo mật lưu trữ | [../05-security/README.md](../05-security/README.md) |

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
