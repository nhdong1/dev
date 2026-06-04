# Chiến Lược Snapshot EBS cho Disaster Recovery

> EBS Snapshot — Ảnh Chụp Nhanh EBS là cơ chế backup chính cho block storage trong AWS. Phần này hướng dẫn thiết kế tần suất, retention policy (chính sách lưu giữ), và tự động hóa snapshot để đạt RPO/RTO mục tiêu.

---

## 1. EBS Snapshot — Cơ Chế Hoạt Động

### Snapshot Là Gì

```
EBS Snapshot:
├── Point-in-time copy (bản sao tại thời điểm cụ thể) của EBS volume
├── Lưu trữ trong S3 (nhưng không hiển thị trong S3 bucket của bạn)
├── Incremental (gia tăng) — chỉ lưu dữ liệu thay đổi so với snapshot trước
└── Có thể sao chép sang region khác cho DR
```

### Incremental Snapshot — Cơ Chế Tiết Kiệm

```
Snapshot 1 (Full — Đầy Đủ):
Volume: [Block A] [Block B] [Block C] [Block D]
→ Snapshot lưu: A + B + C + D = 100GB

Snapshot 2 (Incremental — Gia Tăng):
Volume: [Block A'] [Block B] [Block C'] [Block D]
                           ↑ thay đổi        ↑ thay đổi
→ Snapshot chỉ lưu thêm: A' + C' = 20GB

Snapshot 3 (Incremental):
Volume: [Block A'] [Block B'] [Block C'] [Block D]
                   ↑ thay đổi
→ Snapshot chỉ lưu thêm: B' = 5GB

Tổng storage: 100 + 20 + 5 = 125GB
(thay vì 3 × 100GB = 300GB nếu dùng full snapshot)
```

### Restore từ Snapshot

```
Restore Process:
1. Tạo volume mới từ snapshot (specify region, AZ, volume type, size)
2. Volume được tạo ngay lập tức — nhưng dữ liệu lazy-loaded (tải lười biếng)
3. Lần đọc đầu tiên từ block chưa load → download từ S3 → latency cao
4. Giải pháp: Fast Snapshot Restore (FSR) hoặc pre-warm

Fast Snapshot Restore — FSR — Khôi Phục Nhanh:
├── Pre-load toàn bộ dữ liệu trước khi volume available
├── Xóa bỏ I/O penalty của lazy loading
├── Chi phí: $0.75/snapshot/AZ/giờ (đắt hơn đáng kể)
└── Giới hạn: Tối đa 5 snapshots/AZ được kích hoạt FSR đồng thời
```

---

## 2. Thiết Kế Tần Suất Snapshot

### Nguyên Tắc Cơ Bản

```
RPO Mục Tiêu → Tần Suất Snapshot Tối Thiểu

RPO = 1 giờ  → Snapshot mỗi 1 giờ
RPO = 4 giờ  → Snapshot mỗi 4 giờ
RPO = 24 giờ → Snapshot mỗi ngày (daily)

Lưu Ý:
├── RPO thực tế = tần suất snapshot + thời gian sao chép sang DR region
├── Nếu cross-region copy mất 30 phút → RPO thực = snapshot interval + 30 phút
└── Với dữ liệu thay đổi ít → snapshot hàng giờ = thực tế chỉ backup ít dữ liệu thêm
```

### Khuyến Nghị Theo Workload

```
Database (cơ sở dữ liệu) Production:
├── Snapshot: Mỗi 1–4 giờ
├── Retention: 7 ngày gần + 4 tuần gần + 12 tháng gần
├── Cross-region copy: Bật ngay sau mỗi snapshot
└── FSR: Kích hoạt cho DR region AZ

Web Server Application:
├── Snapshot: Mỗi ngày (daily)
├── Retention: 7 ngày gần + 4 tuần gần
├── Cross-region copy: Hàng ngày (hoặc hàng tuần nếu ít thay đổi)
└── FSR: Không cần thiết (RTO dài hơn)

Development/Staging (Phát Triển/Kiểm Thử):
├── Snapshot: Hàng ngày hoặc trước deploy
├── Retention: 7 ngày
├── Cross-region copy: Không cần
└── Dùng AWS Backup nếu muốn policy tập trung
```

---

## 3. AWS DLM — Data Lifecycle Manager — Quản Lý Vòng Đời Dữ Liệu

### DLM là Gì

```
AWS DLM — Data Lifecycle Manager:
├── Dịch vụ tự động tạo, giữ lại và xóa EBS snapshots
├── Cấu hình qua lifecycle policy (chính sách vòng đời)
├── Không tốn thêm phí (chỉ trả tiền storage snapshot)
└── Tích hợp với CloudWatch Events (Sự Kiện CloudWatch)
```

### Tạo DLM Policy

```bash
aws dlm create-lifecycle-policy \
    --description "Production DB — Hourly snapshot with 7-day retention" \
    --state ENABLED \
    --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
    --policy-details '{
        "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
        "ResourceTypes": ["VOLUME"],
        "TargetTags": [
            {"Key": "Environment", "Value": "production"},
            {"Key": "Backup", "Value": "hourly"}
        ],
        "Schedules": [
            {
                "Name": "Hourly-Snapshots",
                "CreateRule": {
                    "Interval": 1,
                    "IntervalUnit": "HOURS",
                    "Times": ["00:00"]
                },
                "RetainRule": {
                    "Count": 168
                },
                "CopyTags": true,
                "CrossRegionCopyRules": [
                    {
                        "TargetRegion": "eu-west-1",
                        "Encrypted": true,
                        "CmkArn": "arn:aws:kms:eu-west-1:123456789012:key/dr-key-id",
                        "CopyTags": true,
                        "RetainRule": {
                            "Interval": 7,
                            "IntervalUnit": "DAYS"
                        }
                    }
                ]
            },
            {
                "Name": "Daily-Snapshots",
                "CreateRule": {
                    "CronExpression": "cron(0 2 * * ? *)"
                },
                "RetainRule": {
                    "Count": 30
                },
                "CopyTags": true
            },
            {
                "Name": "Monthly-Snapshots",
                "CreateRule": {
                    "CronExpression": "cron(0 3 1 * ? *)"
                },
                "RetainRule": {
                    "Count": 12
                },
                "CopyTags": true
            }
        ]
    }'
```

### DLM — Chú Thích Retention Rule

```
RetainRule có hai cách:

1. Theo Count (Số Lượng):
   "RetainRule": {"Count": 168}
   → Giữ 168 snapshots gần nhất
   → Với snapshot mỗi giờ: 168 ÷ 24 = 7 ngày

2. Theo Age (Tuổi):
   "RetainRule": {"Interval": 30, "IntervalUnit": "DAYS"}
   → Giữ snapshot trong 30 ngày
   → Sau 30 ngày → tự động xóa
```

---

## 4. Multi-Volume Consistent Snapshot — Snapshot Nhất Quán Đa Volume

### Vấn Đề Với Snapshot Từng Volume

```
Kịch Bản Database Có Nhiều Volume:
├── /dev/xvda — OS volume (volume hệ điều hành)
├── /dev/xvdb — Data volume (volume dữ liệu)
└── /dev/xvdc — Log volume (volume nhật ký giao dịch)

Nếu snapshot từng volume tại các thời điểm khác nhau:
├── OS snapshot lúc 10:00:01
├── Data snapshot lúc 10:00:15  ← 14 giây sau
└── Log snapshot lúc 10:00:28   ← 27 giây sau OS

Kết Quả: 3 snapshot KHÔNG nhất quán với nhau
→ Restore có thể gây database corruption (hỏng cơ sở dữ liệu)
```

### Giải Pháp: Multi-Volume Crash-Consistent Snapshot

```bash
# Tạo snapshot cho nhiều volume CÙNG LÚC — crash-consistent
aws ec2 create-snapshots \
    --instance-specification InstanceId=i-1234567890abcdef0,ExcludeBootVolume=false \
    --description "Multi-volume consistent snapshot — prod-db-01" \
    --copy-tags-from-source volume \
    --tag-specifications '[
        {
            "ResourceType": "snapshot",
            "Tags": [
                {"Key": "Name", "Value": "prod-db-01-consistent-snapshot"},
                {"Key": "Environment", "Value": "production"},
                {"Key": "SnapshotType", "Value": "multi-volume-consistent"}
            ]
        }
    ]'
```

### Application-Consistent Snapshot — Snapshot Nhất Quán Ứng Dụng

```
Crash-Consistent:
├── Snapshot tại cùng thời điểm cho tất cả volume
├── Như rút điện đột ngột — dữ liệu trong memory bị mất
└── Phù hợp: Hệ thống có journaling/WAL (Write-Ahead Logging)

Application-Consistent:
├── Ứng dụng được thông báo trước khi snapshot
├── Flush buffer, commit pending transactions (commit giao dịch đang chờ)
├── Snapshot sau khi ứng dụng xác nhận trạng thái ổn định
└── Cách thực hiện: AWS VSS (Volume Shadow Copy Service) hoặc script

VSS (Volume Shadow Copy Service) cho Windows:
├── DLM hỗ trợ VSS snapshot native trên Windows EC2
└── Cần cài AWS VSS Provider trên instance

Script cho Linux:
```

```bash
#!/bin/bash
# Pre-snapshot script cho MySQL

# Flush và lock tables (khóa bảng và flush dữ liệu xuống disk)
mysql -e "FLUSH TABLES WITH READ LOCK;"

# Tạo snapshot
SNAPSHOT_ID=$(aws ec2 create-snapshot \
    --volume-id vol-1234567890abcdef0 \
    --description "App-consistent MySQL snapshot" \
    --query 'SnapshotId' --output text)

echo "Snapshot created: $SNAPSHOT_ID"

# Unlock tables ngay sau khi snapshot initiated
mysql -e "UNLOCK TABLES;"
echo "Tables unlocked — snapshot will complete asynchronously"
```

---

## 5. Cross-Region Snapshot Copy cho DR

### Tự Động Sao Chép Sang DR Region

```bash
# Sao chép snapshot sang DR region
aws ec2 copy-snapshot \
    --source-region us-east-1 \
    --source-snapshot-id snap-0abc123def456789 \
    --destination-region eu-west-1 \
    --description "DR Copy — prod-db — $(date +%Y-%m-%d)" \
    --encrypted \
    --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/dr-key-id
```

### Lambda Function Tự Động Cross-Region Copy

```python
import boto3
import os
from datetime import datetime

def lambda_handler(event, context):
    """
    Tự động copy EBS snapshot sang DR region khi snapshot hoàn thành.
    Trigger: CloudWatch Events rule trên EC2 snapshot state-change.
    """
    ec2_source = boto3.client('ec2', region_name='us-east-1')
    ec2_dr = boto3.client('ec2', region_name='eu-west-1')
    
    DR_REGION = 'eu-west-1'
    DR_KMS_KEY = os.environ['DR_KMS_KEY_ARN']
    
    for record in event.get('detail', {}).get('snapshot_id', []):
        snapshot_id = record
        
        # Lấy thông tin snapshot gốc
        snapshot = ec2_source.describe_snapshots(
            SnapshotIds=[snapshot_id]
        )['Snapshots'][0]
        
        # Chỉ copy snapshot có tag Backup=dr
        tags = {t['Key']: t['Value'] for t in snapshot.get('Tags', [])}
        if tags.get('Backup') != 'dr':
            print(f"Skipping {snapshot_id} — no DR tag")
            return
        
        # Copy sang DR region
        response = ec2_dr.copy_snapshot(
            SourceRegion='us-east-1',
            SourceSnapshotId=snapshot_id,
            Description=f"DR Copy — {snapshot['Description']}",
            Encrypted=True,
            KmsKeyId=DR_KMS_KEY
        )
        
        dr_snapshot_id = response['SnapshotId']
        print(f"DR copy initiated: {dr_snapshot_id}")
        
        # Thêm tag để tracking
        ec2_dr.create_tags(
            Resources=[dr_snapshot_id],
            Tags=[
                {'Key': 'SourceSnapshot', 'Value': snapshot_id},
                {'Key': 'SourceRegion', 'Value': 'us-east-1'},
                {'Key': 'CopiedAt', 'Value': datetime.utcnow().isoformat()},
                *snapshot.get('Tags', [])
            ]
        )
```

---

## 6. Restore từ Snapshot — Quy Trình Thực Tế

### Restore Toàn Bộ EC2 Instance

```bash
#!/bin/bash
# DR Restore Runbook cho EC2 + EBS

SNAPSHOT_ID="snap-0abc123def456789"
DR_REGION="eu-west-1"
DR_AZ="eu-west-1a"
INSTANCE_TYPE="m5.xlarge"

# Bước 1: Tạo volume từ snapshot tại DR region
VOLUME_ID=$(aws ec2 create-volume \
    --region $DR_REGION \
    --availability-zone $DR_AZ \
    --snapshot-id $SNAPSHOT_ID \
    --volume-type gp3 \
    --iops 3000 \
    --throughput 125 \
    --encrypted \
    --query 'VolumeId' --output text)

echo "Volume created: $VOLUME_ID"

# Bước 2: Đợi volume available
aws ec2 wait volume-available \
    --region $DR_REGION \
    --volume-ids $VOLUME_ID
echo "Volume available"

# Bước 3: Launch EC2 instance với volume này
INSTANCE_ID=$(aws ec2 run-instances \
    --region $DR_REGION \
    --image-id ami-dr-version \
    --instance-type $INSTANCE_TYPE \
    --block-device-mappings "[{
        \"DeviceName\": \"/dev/xvda\",
        \"Ebs\": {\"VolumeId\": \"$VOLUME_ID\"}
    }]" \
    --query 'Instances[0].InstanceId' --output text)

echo "Instance launched: $INSTANCE_ID"
echo "RTO checkpoint: $(date)"
```

---

## 7. Retention Policy — Chính Sách Lưu Giữ

### Quy Tắc Grandfather-Father-Son (GFS — Ông Nội - Cha - Con)

```
Retention Policy GFS Điển Hình:

Son (Con — Hàng Ngày):
├── Giữ 7 snapshots daily gần nhất
└── Snapshot hàng ngày → xóa sau 7 ngày

Father (Cha — Hàng Tuần):
├── Giữ 4 snapshots weekly gần nhất
└── Snapshot cuối tuần → xóa sau 4 tuần

Grandfather (Ông Nội — Hàng Tháng):
├── Giữ 12 snapshots monthly gần nhất
└── Snapshot đầu tháng → xóa sau 12 tháng

Tổng Storage Estimate (cho 100GB volume, thay đổi 5GB/ngày):
├── 7 daily × 5GB = 35GB
├── 4 weekly × 35GB = 140GB (đã dedup với daily)
├── 12 monthly × 150GB = 1.8TB (đã dedup)
└── Thực tế ít hơn nhiều nhờ incremental
```

### Retention cho Compliance (Tuân Thủ)

```
Yêu Cầu Pháp Lý Phổ Biến:
├── PCI-DSS (Payment Card Industry): Backup 1 năm, audit log 3 năm
├── HIPAA (Healthcare): Backup 6 năm
├── SOX (Sarbanes-Oxley — Đạo Luật Kế Toán): Financial records 7 năm
└── GDPR (Quy Định Bảo Vệ Dữ Liệu EU): "Không lâu hơn cần thiết"

Giải Pháp:
├── DLM policy với retention period dài (lên đến 100 năm)
├── Hoặc chuyển sang AWS Backup với Vault Lock cho WORM compliance
└── Tag snapshot với retention category để tracking
```

---

## 8. Monitoring Snapshot Health

```bash
# CloudWatch Alarm: Không có snapshot trong 25 giờ (hàng ngày expected)
aws cloudwatch put-metric-alarm \
    --alarm-name "EBS-Snapshot-Missing-24h" \
    --alarm-description "No EBS snapshot created in 25 hours" \
    --metric-name "NumberOfSnapshotsTaken" \
    --namespace "AWS/EBS" \
    --statistic Sum \
    --period 90000 \
    --threshold 1 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:123456789:DR-Alert

# Lambda kiểm tra snapshot age (tuổi snapshot) định kỳ
def check_snapshot_freshness():
    ec2 = boto3.client('ec2')
    
    snapshots = ec2.describe_snapshots(
        Filters=[
            {'Name': 'tag:Backup', 'Values': ['daily']},
            {'Name': 'status', 'Values': ['completed']}
        ],
        OwnerIds=['self']
    )['Snapshots']
    
    from datetime import datetime, timezone, timedelta
    now = datetime.now(timezone.utc)
    threshold = timedelta(hours=25)
    
    for snap in snapshots:
        age = now - snap['StartTime']
        if age > threshold:
            print(f"ALERT: Snapshot {snap['SnapshotId']} is {age} old!")
```

---

## 9. Chi Phí EBS Snapshot

```
Giá Snapshot:
├── $0.05/GB-month cho snapshot storage
├── Incremental — chỉ tính phần thay đổi
└── Cross-region copy: Phí data transfer + storage tại DR region

Ví Dụ Tính Chi Phí:
Volume: 500GB, thay đổi 10GB/ngày

Month 1:
├── Initial snapshot: 500GB × $0.05 = $25
└── 30 daily increments × 10GB × $0.05 = $15
Tổng: $40/tháng

Sau 6 tháng (với GFS retention):
├── 7 daily (70GB mới) = $3.5
├── 4 weekly (40GB mới) = $2
├── 6 monthly (60GB mới) = $3
└── Base + increments ≈ $50/tháng

Cross-Region Copy (sang eu-west-1):
├── Data Transfer: 10GB/ngày × 30 × $0.02 = $6/tháng
└── Storage DR: tương tự local = +$50/tháng
```

---

## 10. Checklist Snapshot Strategy

```
Thiết Kế:
□ Đã xác định RPO cho từng EBS volume (theo criticality tier)
□ Đã tính tần suất snapshot cần thiết
□ Đã chọn retention policy (số ngày/tuần/tháng)
□ Đã quyết định có cần cross-region copy không

Cấu Hình:
□ DLM policy đã tạo với đúng tags
□ IAM role cho DLM có đủ quyền
□ Cross-region copy policy đã bật (nếu cần DR)
□ Multi-volume consistent snapshot cho database

Vận Hành:
□ CloudWatch alarm khi snapshot không được tạo
□ Đã test restore từ snapshot (ít nhất hàng tháng)
□ Đã đo RTO thực tế của restore process
□ FSR bật cho DR snapshots quan trọng
□ Chi phí snapshot trong budget và được theo dõi
```

---

**File Tiếp Theo:** [4-aws-backup-service.md](4-aws-backup-service.md) — AWS Backup Service — Backup Tập Trung
