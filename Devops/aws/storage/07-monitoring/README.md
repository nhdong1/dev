# 07 — Monitoring & Observability cho AWS Storage

> Giám sát và quan sát toàn diện cho S3, EBS, EFS — từ metrics cơ bản đến phát hiện bất thường chi phí

---

## 📋 Mục Lục

| File | Nội Dung | Thời Gian |
|------|----------|-----------|
| [1-cloudwatch-s3-metrics.md](1-cloudwatch-s3-metrics.md) | CloudWatch metrics cho S3 — BucketSizeBytes, NumberOfObjects, RequestMetrics | 30 phút |
| [2-cloudwatch-ebs-metrics.md](2-cloudwatch-ebs-metrics.md) | EBS metrics — VolumeReadOps, BurstBalance, VolumeQueueLength | 30 phút |
| [3-cloudwatch-efs-metrics.md](3-cloudwatch-efs-metrics.md) | EFS metrics — PermittedThroughput, BurstCreditBalance, ClientConnections | 25 phút |
| [4-s3-server-access-logging.md](4-s3-server-access-logging.md) | Nhật ký truy cập S3 — cấu hình, phân tích, Athena query | 25 phút |
| [5-cloudtrail-for-storage.md](5-cloudtrail-for-storage.md) | CloudTrail audit trail cho S3 API calls — event types, analysis | 30 phút |
| [6-cost-anomaly-detection.md](6-cost-anomaly-detection.md) | Cost Anomaly Detection — phát hiện bất thường, cảnh báo tự động | 25 phút |

**Tổng thời gian:** ~3 giờ

---

## 🎯 Tại Sao Monitoring Storage Quan Trọng

Monitoring — Giám sát — không chỉ là "xem metric cho vui". Với AWS storage, monitoring tốt giúp:

```
Phát hiện sự cố sớm:     Disk đầy, IOPS bão hòa, throughput giảm
Tối ưu chi phí:           Phát hiện bất thường chi phí trước khi hoá đơn đến
Bảo mật:                  Audit trail cho mọi API call lên storage
Tuân thủ (Compliance):    Bằng chứng ai đã làm gì với dữ liệu nào
Performance tuning:        Dữ liệu để cải thiện hiệu suất có căn cứ
```

---

## 🏗️ Kiến Trúc Monitoring AWS Storage

```
                    ┌─────────────────────────────────────┐
                    │         AWS Storage Services         │
                    │   S3  │  EBS  │  EFS  │  Glacier    │
                    └───────────────┬─────────────────────┘
                                    │ emit metrics/logs
                    ┌───────────────▼─────────────────────┐
                    │         Amazon CloudWatch            │
                    │  ┌─────────────┐ ┌───────────────┐  │
                    │  │   Metrics   │ │     Logs      │  │
                    │  │  (Chỉ số)   │ │  (Nhật ký)    │  │
                    │  └──────┬──────┘ └───────┬───────┘  │
                    │  ┌──────▼──────┐ ┌───────▼───────┐  │
                    │  │   Alarms    │ │  Log Insights │  │
                    │  │  (Cảnh báo) │ │  (Phân tích)  │  │
                    │  └──────┬──────┘ └───────────────┘  │
                    └─────────┼───────────────────────────┘
                              │ trigger
               ┌──────────────▼────────────────┐
               │         Actions (Hành Động)   │
               │  SNS   │  Lambda  │  Auto Fix  │
               └───────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │         AWS CloudTrail               │
                    │   Audit trail mọi API call          │
                    │   S3 GetObject / PutObject / Delete  │
                    └───────────────┬─────────────────────┘
                                    │ store logs
                    ┌───────────────▼─────────────────────┐
                    │    S3 Bucket (CloudTrail logs)       │
                    │         ↓ query via Athena           │
                    └─────────────────────────────────────┘
```

---

## 📊 Tổng Quan Công Cụ Monitoring

### CloudWatch — Trung Tâm Giám Sát

| Tính Năng | Mô Tả | Dùng Cho |
|-----------|-------|----------|
| **Metrics** — Chỉ số | Dữ liệu định lượng theo thời gian | IOPS, throughput, latency |
| **Alarms** — Cảnh báo | Kích hoạt hành động khi metric vượt ngưỡng | Thông báo, auto-scaling |
| **Dashboards** — Bảng điều khiển | Hiển thị nhiều metric cùng lúc | Toàn cảnh hệ thống |
| **Log Insights** — Phân tích log | Query log bằng ngôn ngữ riêng | Tìm lỗi, phân tích pattern |
| **Contributor Insights** | Tìm top contributors gây ra metric cao | Debug S3 hot key |

### CloudTrail — Kiểm Toán API

| Tính Năng | Mô Tả |
|-----------|-------|
| **Management Events** | API calls quản lý (CreateBucket, DeleteVolume) |
| **Data Events** | API calls cấp object (GetObject, PutObject) |
| **Insights Events** | Phát hiện bất thường trong API call pattern |

### S3 Server Access Logging — Nhật Ký Truy Cập

Ghi lại mọi request đến S3 bucket ở dạng file log text.

### Cost Anomaly Detection — Phát Hiện Bất Thường Chi Phí

Dùng Machine Learning — Học Máy để phát hiện chi phí bất thường.

---

## 🔑 Khái Niệm Cốt Lõi

### Metric Namespaces — Không Gian Tên Chỉ Số

```
AWS/S3          → Metrics cho S3
AWS/EBS         → Metrics cho EBS volumes
AWS/EFS         → Metrics cho EFS file systems
AWS/CostAnomaly → Phát hiện bất thường chi phí
```

### Metric Dimensions — Chiều Phân Loại Chỉ Số

Dimensions giúp lọc metric theo đối tượng cụ thể:

```
S3:  BucketName, StorageType
EBS: VolumeId
EFS: FileSystemId
```

### Alarm States — Trạng Thái Cảnh Báo

```
OK          → Metric trong ngưỡng bình thường
ALARM       → Metric vượt ngưỡng đã đặt
INSUFFICIENT_DATA → Chưa đủ dữ liệu để đánh giá
```

---

## ⚡ Quick Reference — Tham Chiếu Nhanh

### Metric Quan Trọng Nhất Theo Dịch Vụ

| Dịch Vụ | Metric Ưu Tiên | Ngưỡng Cảnh Báo Thường Dùng |
|---------|---------------|---------------------------|
| **S3** | BucketSizeBytes | Tùy ngân sách |
| **S3** | 4xxErrors | > 5% request |
| **EBS** | BurstBalance | < 20% → WARN, < 10% → CRITICAL |
| **EBS** | VolumeQueueLength | > 1 (steady state) |
| **EFS** | BurstCreditBalance | < 1TB |
| **EFS** | PercentIOLimit | > 80% |

### Công Thức IOPS Utilization (EBS)

```
IOPS Utilization = VolumeReadOps + VolumeWriteOps / ProvisionedIOPS × 100
```

Khi > 80% cần nâng IOPS hoặc chuyển volume type.

---

## 📚 Nội Dung Chi Tiết

1. **[CloudWatch S3 Metrics](1-cloudwatch-s3-metrics.md)** — Storage metrics, request metrics, replication metrics
2. **[CloudWatch EBS Metrics](2-cloudwatch-ebs-metrics.md)** — Volume performance, burst balance, queue length
3. **[CloudWatch EFS Metrics](3-cloudwatch-efs-metrics.md)** — Throughput, burst credits, connections
4. **[S3 Server Access Logging](4-s3-server-access-logging.md)** — Cấu hình, phân tích log, Athena
5. **[CloudTrail for Storage](5-cloudtrail-for-storage.md)** — Audit trail, event types, security investigation
6. **[Cost Anomaly Detection](6-cost-anomaly-detection.md)** — ML-based detection, alerts, thresholds

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

1. **"Làm thế nào để biết EBS volume sắp hết IOPS?"**
   → CloudWatch metric VolumeQueueLength và BurstBalance

2. **"Làm sao phát hiện ai đã xóa object trong S3?"**
   → CloudTrail Data Events cho S3

3. **"Cách thiết lập cảnh báo khi chi phí S3 tăng đột biến?"**
   → AWS Cost Anomaly Detection với SNS notification

4. **"EFS throughput metric nào quan trọng nhất?"**
   → PermittedThroughput và BurstCreditBalance

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
