# S3 Cross-Region Replication cho Disaster Recovery

> CRR — Cross-Region Replication — Sao Chép Liên Vùng là công cụ cốt lõi để đạt RPO thấp cho dữ liệu object trên S3. Phần này hướng dẫn thiết kế, cấu hình và vận hành CRR trong bối cảnh DR thực tế.

---

## 1. Tổng Quan S3 Replication

### Hai Loại Replication

```
CRR — Cross-Region Replication — Sao Chép Liên Vùng:
├── Source bucket và destination bucket ở khác AWS Region
├── Mục đích: DR, compliance địa lý, giảm latency cho user toàn cầu
└── Chi phí: Tiền truyền dữ liệu liên vùng (inter-region data transfer)

SRR — Same-Region Replication — Sao Chép Cùng Vùng:
├── Source và destination trong cùng AWS Region
├── Mục đích: Log aggregation (tập hợp nhật ký), compliance, test/prod sync
└── Chi phí: Không tính phí data transfer (cùng region)
```

### Cách CRR Hoạt Động

```
Write Object → Source Bucket (us-east-1)
                │
                ├── S3 nhận object, lưu vào storage
                │
                ├── S3 Replication Service phát hiện object mới
                │   (asynchronous — không đồng bộ, không chặn write)
                │
                └── Sao chép sang Destination Bucket (eu-west-1)
                    ├── Giữ nguyên: metadata, ACL, tags, encryption
                    ├── Giữ nguyên: object key, ETag, version ID
                    └── Thời gian: Vài giây – vài phút (mặc định)
```

---

## 2. Điều Kiện Bắt Buộc để Dùng CRR

```
Yêu Cầu:
├── Versioning (Quản Lý Phiên Bản) phải được bật ở CẢ HAI bucket
│   (source và destination)
│
├── IAM Role (Vai Trò IAM) có quyền:
│   ├── s3:GetObject trên source
│   ├── s3:ReplicateObject trên destination
│   ├── s3:ReplicateDelete (nếu bật delete marker replication)
│   └── s3:ReplicateTags
│
├── Destination bucket phải tồn tại trước
│
└── Nếu destination dùng KMS encryption:
    └── IAM Role cần quyền kms:GenerateDataKey + kms:Decrypt
```

---

## 3. Cấu Hình CRR — Hướng Dẫn Từng Bước

### Bước 1: Bật Versioning

```bash
# Bật versioning trên source bucket
aws s3api put-bucket-versioning \
    --bucket my-source-bucket-us-east-1 \
    --versioning-configuration Status=Enabled

# Bật versioning trên destination bucket
aws s3api put-bucket-versioning \
    --bucket my-dr-bucket-eu-west-1 \
    --versioning-configuration Status=Enabled
```

### Bước 2: Tạo IAM Role cho Replication

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::my-source-bucket-us-east-1"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::my-source-bucket-us-east-1/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::my-dr-bucket-eu-west-1/*"
    }
  ]
}
```

### Bước 3: Cấu Hình Replication Rule

```bash
aws s3api put-bucket-replication \
    --bucket my-source-bucket-us-east-1 \
    --replication-configuration '{
        "Role": "arn:aws:iam::123456789:role/S3ReplicationRole",
        "Rules": [
            {
                "ID": "DR-Replication-All-Objects",
                "Status": "Enabled",
                "Filter": {
                    "Prefix": ""
                },
                "Destination": {
                    "Bucket": "arn:aws:s3:::my-dr-bucket-eu-west-1",
                    "StorageClass": "STANDARD_IA",
                    "ReplicationTime": {
                        "Status": "Enabled",
                        "Time": {"Minutes": 15}
                    },
                    "Metrics": {
                        "Status": "Enabled",
                        "EventThreshold": {"Minutes": 15}
                    }
                },
                "DeleteMarkerReplication": {
                    "Status": "Enabled"
                }
            }
        ]
    }'
```

---

## 4. S3 RTC — Replication Time Control — Kiểm Soát Thời Gian Sao Chép

### Tại Sao Cần RTC?

```
Vấn Đề Với CRR Thông Thường:
├── Không có SLA (Service Level Agreement) về thời gian sao chép
├── Đa số object được sao chép trong vài phút
├── Nhưng object lớn hoặc lúc traffic cao có thể mất vài giờ
└── Không thể cam kết RPO cụ thể với stakeholder

Giải Pháp — S3 RTC:
├── SLA: 99.99% object được sao chép trong 15 phút
├── Metrics được publish lên CloudWatch
├── Cảnh báo khi replication lag vượt ngưỡng
└── Chi phí thêm: ~$0.015/GB data replicated
```

### Kiến Trúc RTC

```
Source Object Upload
        │
        ▼
S3 RTC Engine
        │
        ├── Priority queue (hàng đợi ưu tiên) cho các object
        ├── Dedicated bandwidth (băng thông riêng) cho replication
        └── SLA monitor (giám sát SLA) — alert nếu chậm
        │
        ▼
Destination Bucket (trong 15 phút — 99.99% object)
        │
        └── CloudWatch Metric: ReplicationLatency
```

### Monitoring RTC với CloudWatch

```bash
# Tạo alarm khi replication lag > 10 phút (cảnh báo trước SLA)
aws cloudwatch put-metric-alarm \
    --alarm-name "S3-CRR-Replication-Lag-High" \
    --alarm-description "Replication lag exceeds 10 minutes — RPO at risk" \
    --metric-name ReplicationLatency \
    --namespace AWS/S3 \
    --dimensions Name=SourceBucket,Value=my-source-bucket-us-east-1 \
                 Name=DestinationBucket,Value=my-dr-bucket-eu-west-1 \
                 Name=RuleId,Value=DR-Replication-All-Objects \
    --statistic Maximum \
    --period 300 \
    --threshold 600 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:123456789:DR-Alert-Team
```

---

## 5. Kiến Trúc CRR cho DR — Các Mô Hình

### Mô Hình 1: Active-Passive DR (Chủ Động - Bị Động)

```
us-east-1 (Primary — Chính)          eu-west-1 (DR — Dự Phòng)
├── App servers (active)     CRR     ├── App servers (stopped)
├── S3 source bucket ──────────────▶ ├── S3 DR bucket (read-only)
├── RDS (active)                     └── RDS snapshot (restored khi cần)
└── Route 53: us-east-1

Failover Process (Quy Trình Chuyển Đổi Dự Phòng):
1. Detect failure (phát hiện sự cố) → CloudWatch alarm
2. Start app servers ở eu-west-1
3. Restore RDS từ snapshot mới nhất
4. Update Route 53 → trỏ sang eu-west-1
5. Xác nhận eu-west-1 hoạt động
```

### Mô Hình 2: Bidirectional Replication (Sao Chép Hai Chiều)

```
us-east-1 ◄─────────────────────────▶ eu-west-1
          CRR (từ us-east-1 sang)
          CRR (từ eu-west-1 sang)

Lưu Ý Quan Trọng — Tránh Replication Loop:
├── S3 không sao chép object đã được replicate (replica tag)
├── Dùng prefix filter hoặc tag filter để tránh circular replication
└── Cẩn thận với delete marker replication trong bidirectional setup
```

### Mô Hình 3: Multi-Region Fan-Out (Phân Tán Nhiều Vùng)

```
Source (us-east-1)
        │
        ├──── CRR ────▶ DR Region 1 (eu-west-1)
        ├──── CRR ────▶ DR Region 2 (ap-southeast-1)
        └──── CRR ────▶ Archive (us-west-2, Glacier IA)

Use Cases:
├── Regulatory: Dữ liệu phải ở trong EU và Asia-Pacific
├── Latency: Phục vụ user gần nhất
└── DR cấp cao: Mất một region vẫn còn hai region kia
```

---

## 6. Điểm Quan Trọng Cần Biết

### Những Gì CRR KHÔNG Sao Chép

```
CRR KHÔNG tự động sao chép:
├── Object đã tồn tại trước khi bật CRR
│   → Giải pháp: Dùng S3 Batch Replication (Sao Chép Hàng Loạt)
│
├── Object trong Glacier Flexible/Deep Archive
│   → Giải pháp: Phải restore trước, hoặc dùng lifecycle rule riêng
│
├── Object được mã hóa với SSE-C (Server-Side Encryption with Customer Keys)
│   → Giải pháp: Chuyển sang SSE-S3 hoặc SSE-KMS
│
└── Object xóa (nếu không bật DeleteMarkerReplication)
    → Quan trọng cho DR: Cần cân nhắc bật tính năng này
```

### Delete Marker Replication — Sao Chép Đánh Dấu Xóa

```
Kịch Bản:
├── User xóa object ở source → tạo delete marker
├── Nếu DeleteMarkerReplication = Enabled:
│   └── Delete marker được sao chép sang destination
│       → Object "xóa" ở cả hai bucket
│
└── Nếu DeleteMarkerReplication = Disabled:
    └── Object vẫn còn ở destination dù đã xóa ở source
        → Tốt cho DR (bảo vệ khỏi xóa nhầm)
        → Nhưng destination và source không sync hoàn toàn

Khuyến Nghị DR:
├── Bật cho compliance replication (đồng bộ đầy đủ)
└── Tắt nếu muốn destination là "backup bảo vệ khỏi xóa nhầm"
```

---

## 7. S3 Batch Replication — Sao Chép Dữ Liệu Cũ

```
Vấn Đề: CRR chỉ sao chép object MỚI tạo sau khi bật replication
         Dữ liệu cũ trước đó không được sao chép

Giải Pháp: S3 Batch Replication (Sao Chép Hàng Loạt)
```

```bash
# Bước 1: Tạo S3 Inventory (danh sách object) để biết cần sao chép gì
aws s3api put-bucket-inventory-configuration \
    --bucket my-source-bucket-us-east-1 \
    --id BackfillInventory \
    --inventory-configuration '{
        "Id": "BackfillInventory",
        "IsEnabled": true,
        "Destination": {
            "S3BucketDestination": {
                "Bucket": "arn:aws:s3:::my-inventory-bucket",
                "Format": "CSV"
            }
        },
        "Schedule": {"Frequency": "Daily"},
        "IncludedObjectVersions": "All",
        "OptionalFields": ["ReplicationStatus"]
    }'

# Bước 2: Tạo Batch Operations job dùng inventory làm input
aws s3control create-job \
    --account-id 123456789012 \
    --operation '{"S3ReplicateObject": {}}' \
    --report '{
        "Bucket": "arn:aws:s3:::my-report-bucket",
        "Format": "Report_CSV_20180820",
        "Enabled": true,
        "Prefix": "batch-replication-report",
        "ReportScope": "AllTasks"
    }' \
    --manifest-generator '{
        "S3JobManifestGenerator": {
            "SourceBucket": "arn:aws:s3:::my-source-bucket-us-east-1",
            "EnableManifestOutput": true,
            "Filter": {
                "ObjectReplicationStatuses": ["NONE", "FAILED"]
            }
        }
    }' \
    --role-arn arn:aws:iam::123456789012:role/BatchReplicationRole \
    --priority 10
```

---

## 8. Failover và Failback — Chuyển Đổi và Phục Hồi

### Failover — Chuyển Sang DR Region

```bash
#!/bin/bash
# DR Failover Runbook (Hướng Dẫn Chuyển Đổi DR)

echo "=== DR FAILOVER INITIATED ==="
echo "Primary Region: us-east-1"
echo "DR Region: eu-west-1"

# Bước 1: Xác nhận DR bucket có dữ liệu cập nhật
LAST_MODIFIED=$(aws s3 ls s3://my-dr-bucket-eu-west-1 \
    --recursive | sort | tail -1 | awk '{print $1, $2}')
echo "Last object modified in DR bucket: $LAST_MODIFIED"

# Bước 2: Tạm dừng write vào source (nếu source còn accessible)
# Để tránh split-brain (hai region cùng nhận write)

# Bước 3: Cập nhật DNS via Route 53
aws route53 change-resource-record-sets \
    --hosted-zone-id ZONEID \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "api.example.com",
                "Type": "CNAME",
                "TTL": 60,
                "ResourceRecords": [{"Value": "api-dr.eu-west-1.example.com"}]
            }
        }]
    }'

echo "=== FAILOVER COMPLETE — Monitor eu-west-1 ==="
```

### Failback — Quay Về Primary Region

```
Quy Trình Failback (Phục Hồi Về Primary):
1. Xác nhận primary region (us-east-1) đã ổn định
2. Bật CRR ngược chiều: eu-west-1 → us-east-1
   (sao chép dữ liệu phát sinh trong DR period)
3. Đợi replication đồng bộ hoàn toàn
4. Chuyển DNS về us-east-1
5. Tắt CRR ngược chiều
6. Resume normal operations
```

---

## 9. Chi Phí CRR — Ước Tính

```
Thành Phần Chi Phí CRR:
├── PUT request cho mỗi object replicated: $0.005/1,000 requests
├── Data Transfer Out (truyền dữ liệu ra): $0.02/GB (US → EU)
├── Storage tại destination: Theo storage class đã chọn
└── S3 RTC (nếu dùng): $0.015/GB replicated

Ví Dụ Tính Chi Phí (100GB/ngày, 1M objects/tháng):
├── Data Transfer: 100GB × 30 ngày × $0.02 = $60/tháng
├── PUT requests: 1M × $0.005/1,000 = $5/tháng
├── Storage tại DR (100GB): ~$2.3/tháng (Standard-IA)
├── RTC (nếu dùng): 3,000GB × $0.015 = $45/tháng
└── Tổng ước tính: ~$67–$112/tháng cho 3TB/tháng data
```

---

## 10. Checklist CRR cho DR

```
Thiết Kế:
□ Đã chọn DR region phù hợp (địa lý xa, không cùng natural disaster zone)
□ Đã quyết định CRR thông thường hay RTC (dựa trên RPO yêu cầu)
□ Đã xác định filter rule (toàn bộ bucket hay theo prefix/tag)
□ Đã quyết định storage class tại destination

Cấu Hình:
□ Versioning bật ở cả source và destination
□ IAM role có đủ quyền (kể cả KMS nếu dùng encryption)
□ Replication rule đã được test với object nhỏ
□ Delete marker replication: quyết định có bật hay không

Monitoring:
□ CloudWatch alarm cho ReplicationLatency
□ CloudWatch alarm cho ReplicationFailures
□ Dashboard theo dõi BytesPendingReplication
□ Alert channel (Slack, PagerDuty) kết nối với SNS

Vận Hành:
□ Runbook failover được document và test
□ Runbook failback được document
□ Team biết quy trình failover (đã diễn tập)
□ RTO/RPO thực tế đã đo và phù hợp với SLA
```

---

**File Tiếp Theo:** [3-ebs-snapshot-strategy.md](3-ebs-snapshot-strategy.md) — Chiến Lược Snapshot EBS cho DR
