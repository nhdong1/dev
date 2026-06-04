# CloudWatch Metrics cho Amazon S3

> CloudWatch — Dịch Vụ Giám Sát Đám Mây — cung cấp metrics để theo dõi hiệu suất, dung lượng và lỗi của S3 bucket

---

## 📋 Tổng Quan

Amazon S3 tích hợp sẵn với CloudWatch để cung cấp hai loại metrics:

| Loại Metric | Mô Tả | Cần Bật? | Chi Phí |
|-------------|-------|----------|---------|
| **Storage Metrics** — Chỉ số Lưu trữ | BucketSizeBytes, NumberOfObjects | Không — tự động | Miễn phí |
| **Request Metrics** — Chỉ số Yêu cầu | GetRequests, PutRequests, Errors | Có — opt-in | Tính phí theo metric |

---

## 📦 Storage Metrics — Chỉ Số Lưu Trữ

Storage metrics được CloudWatch thu thập **tự động mỗi ngày một lần** (daily). Không cần cấu hình thêm.

### BucketSizeBytes — Kích Thước Bucket

Tổng dung lượng dữ liệu được lưu trong bucket, tính theo bytes.

```
Namespace:  AWS/S3
Metric:     BucketSizeBytes
Unit:       Bytes
Frequency:  Hàng ngày (không real-time)
Dimensions:
  - BucketName: tên bucket cụ thể
  - StorageType: loại lưu trữ
```

**StorageType dimension values:**

| Giá Trị StorageType | Ý Nghĩa |
|---------------------|---------|
| `StandardStorage` | S3 Standard |
| `IntelligentTieringFAStorage` | Intelligent-Tiering Frequent Access |
| `IntelligentTieringIAStorage` | Intelligent-Tiering Infrequent Access |
| `IntelligentTieringAIAStorage` | Intelligent-Tiering Archive Instant Access |
| `StandardIAStorage` | Standard-Infrequent Access |
| `OneZoneIAStorage` | One Zone-IA |
| `ReducedRedundancyStorage` | Reduced Redundancy (deprecated) |
| `GlacierStorage` | Glacier Flexible Retrieval |
| `DeepArchiveStorage` | Glacier Deep Archive |
| `AllStorageTypes` | Tổng tất cả loại |

**Ví dụ AWS CLI:**

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=my-data-bucket \
               Name=StorageType,Value=StandardStorage \
  --start-time 2026-05-01T00:00:00Z \
  --end-time 2026-05-16T00:00:00Z \
  --period 86400 \
  --statistics Average
```

### NumberOfObjects — Số Lượng Object

Tổng số object được lưu trong bucket, tính theo từng storage class.

```
Namespace:  AWS/S3
Metric:     NumberOfObjects
Unit:       Count
Frequency:  Hàng ngày
Dimensions:
  - BucketName
  - StorageType: AllStorageTypes (bắt buộc dùng giá trị này)
```

> **Lưu ý:** Metric này chỉ chính xác khi dùng StorageType=AllStorageTypes. AWS không cung cấp count theo từng storage class.

---

## ⚡ Request Metrics — Chỉ Số Yêu Cầu

Request metrics theo dõi từng HTTP request đến S3 bucket theo **thời gian thực** (1 phút). Phải bật thủ công và tính phí CloudWatch chuẩn.

### Bật Request Metrics

**Qua Console:**
1. Vào S3 Console → chọn bucket
2. Tab **Metrics** → **Request metrics**
3. Tạo filter: `EntireBucket` (toàn bucket) hoặc prefix cụ thể
4. Lưu — metrics xuất hiện sau ~15 phút

**Qua AWS CLI:**

```bash
aws s3api put-bucket-metrics-configuration \
  --bucket my-data-bucket \
  --id EntireBucket \
  --metrics-configuration '{"Id":"EntireBucket","Filter":{}}'
```

**Qua CloudFormation — Hạ Tầng Dưới Dạng Mã:**

```yaml
MyBucketMetrics:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: my-data-bucket
    MetricsConfigurations:
      - Id: EntireBucket
```

### Các Request Metrics Quan Trọng

#### Metrics cho GET Requests

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `GetRequests` | Count | Số request GET thành công |
| `BytesDownloaded` | Bytes | Tổng dữ liệu tải xuống |
| `FirstByteLatency` | Milliseconds | Thời gian đến byte đầu tiên (Time to First Byte — TTFB) |
| `TotalRequestLatency` | Milliseconds | Tổng thời gian xử lý request |

#### Metrics cho PUT/POST Requests

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `PutRequests` | Count | Số request PUT thành công |
| `PostRequests` | Count | Số request POST thành công |
| `BytesUploaded` | Bytes | Tổng dữ liệu tải lên |

#### Metrics cho DELETE Requests

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `DeleteRequests` | Count | Số request DELETE thành công |

#### Metrics cho Errors

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `4xxErrors` | Count | Lỗi phía client (403 Forbidden, 404 Not Found) |
| `5xxErrors` | Count | Lỗi phía server — hiếm gặp với S3 |

> **4xxErrors quan trọng:** Tỉ lệ 4xx cao thường chỉ ra:
> - Bucket policy sai → 403 Forbidden
> - Application request object không tồn tại → 404 Not Found
> - Credential hết hạn → 403

#### Metrics cho List Operations

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `ListRequests` | Count | Số request LIST objects |
| `HeadRequests` | Count | Số request HEAD (kiểm tra metadata) |

---

## 🔄 Replication Metrics — Chỉ Số Sao Chép

Khi dùng CRR — Cross-Region Replication — Sao Chép Liên Vùng hoặc SRR — Same-Region Replication, có thêm metrics:

```
Namespace:  AWS/S3
```

| Metric | Ý Nghĩa |
|--------|---------|
| `ReplicationLatency` | Độ trễ sao chép — giây |
| `BytesPendingReplication` | Dữ liệu chờ sao chép — bytes |
| `OperationsPendingReplication` | Số operation chờ sao chép |
| `OperationsFailedReplication` | Số operation sao chép thất bại |

**Bật Replication Metrics:**

```bash
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/replication-role",
    "Rules": [{
      "Id": "ReplicateAll",
      "Status": "Enabled",
      "Metrics": {
        "Status": "Enabled",
        "EventThreshold": {"Minutes": 15}
      },
      "ReplicationTime": {
        "Status": "Enabled",
        "Time": {"Minutes": 15}
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket"
      }
    }]
  }'
```

> **S3 RTC — Replication Time Control — Kiểm Soát Thời Gian Sao Chép:** SLA 99.99% object được sao chép trong 15 phút. Khi bật RTC, metrics sẽ theo dõi objects chưa được sao chép sau 15 phút.

---

## 🚨 CloudWatch Alarms cho S3

### Alarm Tỉ Lệ Lỗi 4xx

```json
{
  "AlarmName": "S3-HighErrorRate-4xx",
  "AlarmDescription": "Cảnh báo khi tỉ lệ lỗi 4xx vượt 5%",
  "MetricName": "4xxErrors",
  "Namespace": "AWS/S3",
  "Dimensions": [
    {"Name": "BucketName", "Value": "my-data-bucket"},
    {"Name": "FilterId", "Value": "EntireBucket"}
  ],
  "Period": 300,
  "EvaluationPeriods": 3,
  "Threshold": 50,
  "ComparisonOperator": "GreaterThanThreshold",
  "Statistic": "Sum",
  "AlarmActions": ["arn:aws:sns:us-east-1:123456789012:storage-alerts"]
}
```

**Tạo alarm qua CLI:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "S3-HighErrorRate-4xx" \
  --alarm-description "4xx error rate too high" \
  --metric-name 4xxErrors \
  --namespace AWS/S3 \
  --dimensions Name=BucketName,Value=my-data-bucket Name=FilterId,Value=EntireBucket \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:storage-alerts
```

### Alarm Latency Cao

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "S3-HighLatency" \
  --metric-name TotalRequestLatency \
  --namespace AWS/S3 \
  --dimensions Name=BucketName,Value=my-data-bucket Name=FilterId,Value=EntireBucket \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 500 \
  --comparison-operator GreaterThanThreshold \
  --statistic p99 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:storage-alerts
```

> **p99** — 99th percentile — phân vị thứ 99: 99% request hoàn thành trong thời gian dưới ngưỡng này. Dùng percentile thay vì Average để phát hiện outliers.

---

## 📊 CloudWatch Dashboard cho S3

### Tạo Dashboard Bằng CLI

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "S3-Storage-Overview" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "Bucket Size (GB) theo Storage Class",
          "metrics": [
            ["AWS/S3", "BucketSizeBytes", "BucketName", "my-bucket",
             "StorageType", "StandardStorage", {"label": "Standard"}],
            ["AWS/S3", "BucketSizeBytes", "BucketName", "my-bucket",
             "StorageType", "StandardIAStorage", {"label": "Standard-IA"}],
            ["AWS/S3", "BucketSizeBytes", "BucketName", "my-bucket",
             "StorageType", "GlacierStorage", {"label": "Glacier"}]
          ],
          "view": "timeSeries",
          "period": 86400,
          "stat": "Average"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "Request Errors",
          "metrics": [
            ["AWS/S3", "4xxErrors", "BucketName", "my-bucket",
             "FilterId", "EntireBucket", {"label": "4xx Errors", "color": "#FF9900"}],
            ["AWS/S3", "5xxErrors", "BucketName", "my-bucket",
             "FilterId", "EntireBucket", {"label": "5xx Errors", "color": "#D13212"}]
          ],
          "view": "timeSeries",
          "period": 300,
          "stat": "Sum"
        }
      }
    ]
  }'
```

---

## 🔍 S3 Storage Lens — Kính Phân Tích Lưu Trữ

S3 Storage Lens là tính năng phân tích S3 toàn tổ chức (organization-wide), cung cấp insight sâu hơn CloudWatch metrics thuần.

### So Sánh CloudWatch vs Storage Lens

| Tiêu Chí | CloudWatch | S3 Storage Lens |
|----------|-----------|-----------------|
| **Phạm vi** | Theo từng bucket | Toàn bộ organization (tổ chức) |
| **Granularity** — Độ chi tiết | Real-time (1 phút) | Daily (hàng ngày) |
| **Metrics** | Kỹ thuật — IOPS, latency | Business — cost, usage pattern |
| **Free tier** | Storage metrics miễn phí | 29 free metrics |
| **Advanced** | Custom metrics tốn phí | 35+ advanced metrics |
| **Dùng cho** | Alarms, operational | Cost optimization, governance |

### Metrics nổi bật trong Storage Lens

```
StorageBytes             → Tổng dung lượng
ObjectCount              → Số object
AveragObjectSize         → Kích thước trung bình
IncompleteMultipartUploadBytes → Multipart upload chưa hoàn thành (lãng phí tiền)
NonCurrentVersionStorageBytes  → Dung lượng non-current versions
```

> **Tip tiết kiệm tiền:** Storage Lens thường phát hiện `IncompleteMultipartUploadBytes` lớn — đây là tiền bị lãng phí vì multipart upload không hoàn thành. Thêm lifecycle rule để xóa incomplete uploads sau 7 ngày.

---

## 🛠️ Thực Hành Tốt — Best Practices

### 1. Bật Request Metrics cho bucket quan trọng

```bash
# Bật cho tất cả bucket production
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text); do
  aws s3api put-bucket-metrics-configuration \
    --bucket $bucket \
    --id EntireBucket \
    --metrics-configuration '{"Id":"EntireBucket","Filter":{}}'
  echo "Enabled metrics for: $bucket"
done
```

### 2. Thiết lập Alarm Tỉ Lệ Lỗi

Tính tỉ lệ 4xx so với tổng request dùng CloudWatch Metric Math — Toán Học Chỉ Số:

```
ErrorRate = 4xxErrors / (GetRequests + PutRequests + ...) × 100
```

Dùng CloudWatch Metric Math expression:

```
m1 = 4xxErrors
m2 = AllRequests
errorRate = m1/m2*100
```

### 3. Monitor Replication Lag

Khi dùng CRR, theo dõi `ReplicationLatency` và đặt alarm khi > 15 phút (900 giây).

### 4. Dùng Contributor Insights

CloudWatch Contributor Insights — Phân Tích Đóng Góp Viên — giúp xác định top IP/user-agent/key nào gây ra nhiều 4xx nhất:

```bash
aws cloudwatch put-insight-rule \
  --rule-name "S3-Top4xxKeys" \
  --rule-state ENABLED \
  --rule-definition '{
    "Schema": "CloudWatchLogRule/1",
    "LogGroupNames": ["aws-s3-access-logs"],
    "Contributions": {
      "Keys": ["$.key"],
      "Metrics": [{"Name": "ErrorCount", "ValueOf": "$.statuscode", "MatchExpressions": ["4[0-9][0-9]"]}]
    }
  }'
```

---

## 📝 Câu Hỏi Phỏng Vấn

**Q: S3 có real-time metrics không? Tại sao?**
A: Phụ thuộc loại metric. Storage metrics (BucketSizeBytes) chỉ cập nhật hàng ngày. Request metrics cập nhật mỗi phút nhưng phải bật thủ công. S3 không phải block storage nên không có metrics sub-minute như EBS.

**Q: Làm sao biết bucket nào tốn nhiều chi phí nhất?**
A: Dùng S3 Storage Lens với cost-effective mode, hoặc CloudWatch BucketSizeBytes kết hợp với Cost Allocation Tags — Thẻ Phân Bổ Chi Phí.

**Q: 4xxErrors cao có nghĩa là gì?**
A: Lỗi phía client — bucket policy sai (403), key không tồn tại (404), hoặc presigned URL hết hạn (403). Cần kiểm tra CloudTrail để biết caller cụ thể.

---

**Tiếp Theo:** [2-cloudwatch-ebs-metrics.md](2-cloudwatch-ebs-metrics.md) — EBS metrics và BurstBalance

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
