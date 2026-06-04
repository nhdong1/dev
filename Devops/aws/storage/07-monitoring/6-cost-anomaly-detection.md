# Cost Anomaly Detection — Phát Hiện Bất Thường Chi Phí Lưu Trữ

> AWS Cost Anomaly Detection dùng Machine Learning — Học Máy — để tự động phát hiện chi phí tăng bất thường, gửi cảnh báo trước khi hóa đơn cuối tháng gây bất ngờ

---

## 📋 Tổng Quan

Chi phí AWS Storage có thể tăng đột biến vì nhiều lý do:

```
Vô tình upload data lớn bất thường
Lifecycle rule bị xóa → data không tự xóa → tích lũy
Application bug → loop tạo object vô tận
DDoS hay data theft → request/egress cost tăng
Misconfigured replication → sao chép vô hạn
Snapshot không được dọn → chi phí EBS snapshot
```

Cost Anomaly Detection — Phát Hiện Bất Thường Chi Phí — phân tích lịch sử chi phí và dùng ML để phát hiện khi chi phí lệch khỏi pattern bình thường.

---

## 🧠 Cơ Chế Hoạt Động

```
1. Thu thập:  AWS Cost Anomaly Detection đọc dữ liệu từ Cost & Usage Report
              (Báo Cáo Chi Phí & Sử Dụng) — cập nhật hàng ngày

2. Baseline:  ML model học pattern chi phí lịch sử
              - Theo tuần: thứ 2 thường nhiều hơn thứ 7
              - Theo tháng: cuối tháng thường cao hơn
              - Seasonal patterns: theo mùa, sự kiện đặc biệt

3. Phát hiện: So sánh chi phí hiện tại với predicted range (dải dự đoán)
              Nếu vượt ngưỡng → đây là anomaly — bất thường

4. Cảnh báo: Gửi alert qua SNS, email, Slack (qua SNS integration)
```

---

## 🏗️ Cấu Hình Cost Anomaly Detection

### Tạo Monitor — Màn Hình Giám Sát

Monitor là đơn vị theo dõi, xác định phạm vi giám sát:

```bash
# Monitor cho toàn bộ storage services
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "StorageServicesMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Monitor cho S3 cụ thể
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "S3Monitor",
    "MonitorType": "CUSTOM",
    "MonitorSpecification": {
      "And": [
        {
          "Dimensions": {
            "Key": "SERVICE",
            "Values": ["Amazon Simple Storage Service"]
          }
        }
      ]
    }
  }'
```

**Các MonitorType:**

| Loại | Mô Tả | Dùng Khi |
|------|-------|---------|
| `AWS_SERVICES` | Giám sát từng AWS service | Muốn tổng quan toàn account |
| `DIMENSIONAL` | Theo dimension: SERVICE, LINKED_ACCOUNT, REGION | Tổ chức có nhiều account |
| `CUSTOM` | Lọc tùy chỉnh theo tag, service, region | Theo dõi project cụ thể |

### Tạo Subscription — Đăng Ký Nhận Cảnh Báo

```bash
# Đầu tiên lấy MonitorArn từ bước trên
MONITOR_ARN=$(aws ce get-anomaly-monitors \
  --query 'AnomalyMonitors[?MonitorName==`StorageServicesMonitor`].MonitorArn' \
  --output text)

# Tạo subscription với SNS
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "StorageAnomalyAlert",
    "MonitorArnList": ["'"$MONITOR_ARN"'"],
    "Subscribers": [
      {
        "Address": "arn:aws:sns:us-east-1:123456789012:cost-alerts",
        "Type": "SNS"
      },
      {
        "Address": "ops-team@company.com",
        "Type": "EMAIL"
      }
    ],
    "Threshold": 20,
    "ThresholdExpression": {
      "Dimensions": {
        "Key": "ANOMALY_TOTAL_IMPACT_PERCENTAGE",
        "Values": ["20"],
        "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
      }
    },
    "Frequency": "DAILY"
  }'
```

**Tham số Subscription:**

| Tham Số | Ý Nghĩa |
|---------|---------|
| `Threshold` | Ngưỡng % tăng để kích hoạt alert |
| `ThresholdExpression` | Logic phức tạp hơn — % hoặc số tiền tuyệt đối |
| `Frequency` | `DAILY` (hàng ngày) hoặc `IMMEDIATE` (ngay lập tức) |
| `Subscribers` | SNS topic, email, chat (qua SNS → Lambda) |

---

## 📊 Loại Threshold — Ngưỡng Kích Hoạt

### 1. Percentage Threshold — Ngưỡng Theo Tỉ Lệ

```json
{
  "Threshold": 20
}
```

Kích hoạt khi chi phí vượt **20%** so với expected (dự đoán). Phù hợp cho chi phí ổn định.

### 2. Absolute Threshold — Ngưỡng Theo Số Tiền Tuyệt Đối

```json
{
  "ThresholdExpression": {
    "Dimensions": {
      "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
      "Values": ["100"],
      "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
    }
  }
}
```

Kích hoạt khi chi phí bất thường vượt **$100**. Phù hợp để tránh alert không quan trọng (ví dụ: $1 tăng lên $1.5 = 50% nhưng không đáng lo).

### 3. Combined Threshold — Kết Hợp

```json
{
  "ThresholdExpression": {
    "And": [
      {
        "Dimensions": {
          "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
          "Values": ["50"],
          "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
        }
      },
      {
        "Dimensions": {
          "Key": "ANOMALY_TOTAL_IMPACT_PERCENTAGE",
          "Values": ["10"],
          "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
        }
      }
    ]
  }
}
```

Kích hoạt khi **cả hai** điều kiện đều đúng: tăng > $50 VÀ > 10%. Giảm false alarms.

---

## 🔔 Cấu Hình Cảnh Báo Qua Slack

Kết hợp SNS → Lambda → Slack để nhận alert ngay trên Slack:

```python
import json
import urllib.request

def lambda_handler(event, context):
    """
    Lambda nhận SNS message từ Cost Anomaly Detection
    và gửi vào Slack channel
    """
    sns_message = json.loads(event['Records'][0]['Sns']['Message'])
    
    # Phân tích thông tin anomaly
    anomaly = sns_message.get('anomalyDetails', {})
    service = anomaly.get('dimensionValue', 'Unknown Service')
    impact = anomaly.get('totalImpact', {})
    max_impact = impact.get('maxImpact', 0)
    total_impact = impact.get('totalImpact', 0)
    
    # Tính % tăng
    expected = impact.get('totalExpectedSpend', 1)
    actual = impact.get('totalActualSpend', 0)
    pct_increase = ((actual - expected) / expected) * 100
    
    # Tạo Slack message
    slack_message = {
        "text": f":warning: *AWS Cost Anomaly Detected!*",
        "attachments": [
            {
                "color": "danger",
                "fields": [
                    {"title": "Service", "value": service, "short": True},
                    {"title": "Tổng chi phí bất thường", "value": f"${total_impact:.2f}", "short": True},
                    {"title": "Chi phí cao nhất 1 ngày", "value": f"${max_impact:.2f}", "short": True},
                    {"title": "% Tăng so với dự kiến", "value": f"{pct_increase:.1f}%", "short": True},
                    {"title": "Hành động", "value": "Kiểm tra AWS Cost Explorer ngay", "short": False}
                ]
            }
        ]
    }
    
    # Gửi vào Slack
    slack_webhook_url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    req = urllib.request.Request(
        slack_webhook_url,
        data=json.dumps(slack_message).encode('utf-8'),
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)
    
    return {'statusCode': 200}
```

---

## 🏷️ Monitor theo Cost Allocation Tags — Thẻ Phân Bổ Chi Phí

Phát hiện bất thường theo team, project hoặc môi trường:

```bash
# Monitor theo tag Environment
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "ProductionStorageMonitor",
    "MonitorType": "CUSTOM",
    "MonitorSpecification": {
      "And": [
        {
          "Tags": {
            "Key": "Environment",
            "Values": ["production"]
          }
        },
        {
          "Dimensions": {
            "Key": "SERVICE",
            "Values": [
              "Amazon Simple Storage Service",
              "Amazon Elastic Block Store",
              "Amazon Elastic File System"
            ]
          }
        }
      ]
    }
  }'
```

### Thiết Lập Tag Strategy — Chiến Lược Đánh Thẻ

```
Tag key: Environment    Value: production | staging | dev
Tag key: Team           Value: backend | data | security
Tag key: Project        Value: user-service | analytics | backup
Tag key: CostCenter     Value: eng-001 | ops-002
```

Khi tag được kích hoạt trong **AWS Cost Allocation Tags**, Cost Anomaly Detection có thể phân tích theo từng tag.

---

## 📈 Phân Tích Anomaly Đã Được Phát Hiện

```bash
# Xem các anomalies gần đây
aws ce get-anomalies \
  --date-interval '{
    "StartDate": "2026-05-01",
    "EndDate": "2026-05-16"
  }' \
  --query 'Anomalies[*].{
    Service:DimensionValue,
    StartDate:AnomalyStartDate,
    EndDate:AnomalyEndDate,
    ActualSpend:Impact.TotalActualSpend,
    ExpectedSpend:Impact.TotalExpectedSpend,
    Impact:Impact.TotalImpact
  }' \
  --output table

# Xem chi tiết một anomaly
ANOMALY_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
aws ce get-anomalies \
  --anomaly-ids $ANOMALY_ID
```

**Kết quả mẫu:**

```
Service                              StartDate    ActualSpend  ExpectedSpend  Impact
Amazon Simple Storage Service        2026-05-14   $1,240.50    $89.30         $1,151.20
Amazon Elastic Block Store           2026-05-12   $456.00      $120.00        $336.00
```

---

## 🔍 Quy Trình Điều Tra Khi Nhận Anomaly Alert

```
Bước 1: Xác nhận anomaly thật
  → Vào Cost Explorer → xem chi tiết service, region, tag
  → Có phải do deploy mới không? Release scheduled không?

Bước 2: Tìm nguyên nhân
  → S3: kiểm tra BucketSizeBytes, RequestMetrics
     Thủ phạm thường: unexpected large upload, replication loop
  → EBS: kiểm tra số snapshot, volume size
     Thủ phạm thường: automated snapshot quá nhiều
  → Data Transfer: kiểm tra BytesDownloaded
     Thủ phạm thường: public bucket bị scan, data leak

Bước 3: Giảm tác động ngay
  → S3: xóa object thừa, review lifecycle rules
  → EBS: xóa snapshot cũ, xóa volume không dùng
  → Data transfer: review CloudFront, đặt bucket restrictions

Bước 4: Phòng ngừa
  → S3 Budget với alarm
  → Lifecycle rules đúng chỗ
  → Athena query để identify top spenders
```

---

## 💰 AWS Budgets — Ngân Sách Chi Phí Bổ Sung

Cost Anomaly Detection phát hiện *bất thường*. AWS Budgets — Ngân Sách AWS — phát hiện khi *vượt ngưỡng cố định*. Cần dùng cả hai:

```bash
# Tạo budget cho S3 không vượt $500/tháng
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "S3-Monthly-Budget",
    "BudgetLimit": {
      "Amount": "500",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "Service": ["Amazon Simple Storage Service"]
    },
    "CostTypes": {
      "IncludeTax": false,
      "IncludeSubscription": false,
      "UseBlended": false
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "SNS",
          "Address": "arn:aws:sns:us-east-1:123456789012:cost-alerts"
        },
        {
          "SubscriptionType": "EMAIL",
          "Address": "ops@company.com"
        }
      ]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "SNS",
          "Address": "arn:aws:sns:us-east-1:123456789012:cost-critical"
        }
      ]
    }
  ]'
```

---

## 📊 S3 Storage Lens — Phân Tích Chi Phí Chuyên Sâu

S3 Storage Lens — Kính Phân Tích Lưu Trữ — cung cấp metrics giúp tìm chi phí lãng phí:

```bash
# Tạo Storage Lens configuration
aws s3control put-storage-lens-configuration \
  --account-id 123456789012 \
  --config-id my-cost-optimization-lens \
  --storage-lens-configuration '{
    "Id": "my-cost-optimization-lens",
    "IsEnabled": true,
    "Include": {
      "Buckets": ["arn:aws:s3:::*"]
    },
    "DataExport": {
      "S3BucketDestination": {
        "AccountId": "123456789012",
        "Arn": "arn:aws:s3:::my-lens-export-bucket",
        "Format": "CSV",
        "OutputSchemaVersion": "V_1",
        "Prefix": "lens-data/"
      }
    },
    "AccountLevel": {
      "ActivityMetrics": {"IsEnabled": true},
      "BucketLevel": {
        "ActivityMetrics": {"IsEnabled": true},
        "PrefixLevel": {
          "StorageMetrics": {
            "IsEnabled": true,
            "SelectionCriteria": {
              "MaxDepth": 5,
              "MinStorageBytesPercentage": 1.0
            }
          }
        }
      }
    }
  }'
```

**Metrics hữu ích từ Storage Lens để phát hiện lãng phí:**

| Metric | Chỉ Ra |
|--------|--------|
| `IncompleteMultipartUploadBytes` | Multipart upload bỏ dở — đang trả tiền vô ích |
| `NonCurrentVersionStorageBytes` | Old versions — cần lifecycle rule xóa |
| `DeleteMarkerStorageBytes` | Delete markers thừa — cần lifecycle rule xóa |
| `ExpiredObjectDeleteMarkerCount` | Objects hết hạn nhưng chưa được dọn |

---

## 📋 Tổng Hợp — Chiến Lược Monitoring Chi Phí Toàn Diện

```
Lớp 1: Real-time awareness
  → CloudWatch Billing Alarms (miễn phí, cảnh báo mức tổng)
  → AWS Budgets (cảnh báo theo service)

Lớp 2: Anomaly detection (phát hiện bất thường)
  → Cost Anomaly Detection per service
  → Cost Anomaly Detection per team/project tag

Lớp 3: Root cause analysis (phân tích nguyên nhân gốc)
  → Cost Explorer drill-down
  → S3 Storage Lens cho storage-specific insights

Lớp 4: Proactive optimization (tối ưu chủ động)
  → Storage Lens → tìm IncompleteMultipartUpload
  → CloudWatch S3 metrics → BucketSizeBytes trend
  → Athena query CloudTrail → tìm unexpected access pattern
```

---

## 📝 Câu Hỏi Phỏng Vấn

**Q: Hóa đơn tháng này tăng $5,000 so với tháng trước. Làm sao điều tra?**
A: Vào AWS Cost Explorer → phân tích theo service. Thường gặp nhất: S3 data transfer out (ai đó download nhiều), EBS snapshot tích lũy, hoặc S3 storage tăng do data pipeline. Nếu là S3: kiểm tra BucketSizeBytes và BytesDownloaded CloudWatch. Nếu là EBS: xem danh sách snapshot theo CreateTime.

**Q: Cost Anomaly Detection có hoạt động real-time không?**
A: Không. Dữ liệu Cost & Usage Report cập nhật hàng ngày, nên anomaly thường phát hiện sau 1-2 ngày. Để phát hiện nhanh hơn, kết hợp với CloudWatch Billing Alarm (cập nhật vài giờ một lần) hoặc AWS Budgets.

**Q: Threshold bao nhiêu % là hợp lý cho storage cost anomaly?**
A: Phụ thuộc vào độ biến động bình thường. Storage cost thường ổn định hơn compute cost, nên 20-30% là hợp lý. Nhưng nên thêm absolute threshold ít nhất $50-100 để tránh alert không quan trọng (ví dụ: $10 tăng lên $13 = 30% nhưng không đáng lo).

---

**Hoàn Thành Module 07:** Trở về [README.md](README.md) để xem tổng quan

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
