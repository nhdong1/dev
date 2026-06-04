# Personal Health Dashboard (PHD) — Bảng Điều Khiển Sức Khỏe Cá Nhân

> **Personal Health Dashboard (PHD)** — nay được gọi là **AWS Health / Your Account Health** — cung cấp chế độ xem cá nhân hóa về sức khỏe dịch vụ AWS, hiển thị đúng những sự kiện ảnh hưởng đến tài nguyên và account của bạn, kèm hướng dẫn hành động cụ thể.

---

## 📚 Mục Lục

1. [PHD là gì và tại sao cần nó](#phd-là-gì-và-tại-sao-cần-nó)
2. [Các Loại Sự Kiện PHD](#các-loại-sự-kiện-phd)
3. [Cấu Trúc Sự Kiện Health](#cấu-trúc-sự-kiện-health)
4. [Truy Cập PHD](#truy-cập-phd)
5. [AWS Health API](#aws-health-api)
6. [Thực Hành: Kiểm Tra Events](#thực-hành-kiểm-tra-events)
7. [Tình Huống Thực Tế](#tình-huống-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## PHD là gì và tại sao cần nó

### Vấn Đề PHD Giải Quyết

Trước khi có PHD, khi AWS gặp sự cố:

```
Kỹ sư phải:
1. Theo dõi status.aws.amazon.com (thủ công)
2. Kiểm tra từng service trong console
3. Không biết có phải tài nguyên của mình bị ảnh hưởng không
4. Không có thông báo chủ động
```

Với PHD:

```
AWS chủ động thông báo:
"Instance i-0abc123 của bạn đang chạy trên phần cứng sẽ được retire vào 2026-06-01.
Vui lòng stop/start instance trước ngày đó.
Các instance bị ảnh hưởng: i-0abc123, i-0def456"
```

### Lợi Ích Cốt Lõi

| Lợi Ích                                       | Giải Thích                                                        |
| --------------------------------------------- | ----------------------------------------------------------------- |
| **Visibility cụ thể** (Khả năng quan sát cụ thể) | Biết chính xác tài nguyên nào bị ảnh hưởng, không phải đoán mò |
| **Proactive notification** (Thông báo chủ động) | Nhận thông báo trước khi sự cố xảy ra hoặc trước bảo trì       |
| **Recommended actions** (Hành động khuyến nghị) | AWS đưa ra hướng dẫn cụ thể phải làm gì                        |
| **90-day history** (Lịch sử 90 ngày)         | Xem lại sự kiện trong 3 tháng qua để phân tích                   |
| **EventBridge integration**                   | Tự động hóa phản hồi, không cần theo dõi thủ công               |

---

## Các Loại Sự Kiện PHD

### 1. Issue Events (Sự Kiện Sự Cố)

Xảy ra khi có vấn đề **đang diễn ra** ảnh hưởng đến dịch vụ AWS và tài nguyên của bạn.

**Ví dụ thực tế:**

| Event Code                                          | Mô Tả                                                            |
| --------------------------------------------------- | ---------------------------------------------------------------- |
| `AWS_EC2_OPERATIONAL_ISSUE`                         | EC2 API hoặc instance control plane gặp vấn đề                  |
| `AWS_RDS_OPERATIONAL_ISSUE`                         | RDS database connectivity hoặc performance degraded             |
| `AWS_LAMBDA_OPERATIONAL_ISSUE`                      | Lambda invocations bị lỗi hoặc latency cao bất thường           |
| `AWS_EKS_OPERATIONAL_ISSUE`                         | EKS control plane gặp sự cố                                     |
| `AWS_S3_OPERATIONAL_ISSUE`                          | S3 PUT/GET operations bị ảnh hưởng tại một region cụ thể        |
| `AWS_VPC_OPERATIONAL_ISSUE`                         | Kết nối mạng trong VPC bị gián đoạn                             |

**Vòng đời issue event:**
```
open → (đang điều tra) → (đang khắc phục) → closed
  │                                              │
  └── PHD hiển thị "ongoing"                    └── PHD cập nhật "resolved"
```

---

### 2. Scheduled Change Events (Sự Kiện Thay Đổi Có Lịch)

AWS thông báo **trước** về bảo trì hoặc thay đổi bắt buộc. Đây là loại sự kiện quan trọng nhất cần tự động hóa phản hồi.

**Ví dụ thực tế:**

| Event Code                                             | Mô Tả                                                           | Thời Gian Báo Trước |
| ------------------------------------------------------ | --------------------------------------------------------------- | ------------------- |
| `AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED`                | Phần cứng EC2 sẽ bị retire, instance phải được migrate         | 2–4 tuần           |
| `AWS_EC2_INSTANCE_REBOOT_SCHEDULED`                    | Instance cần reboot để áp dụng security patch                  | 2 tuần              |
| `AWS_EC2_INSTANCE_STOP_SCHEDULED`                      | Instance sẽ bị stop bởi AWS (hiếm gặp)                         | 2 tuần              |
| `AWS_RDS_MAINTENANCE_SCHEDULED`                        | RDS minor version upgrade hoặc OS patch                        | 1–2 tuần            |
| `AWS_CERTIFICATE_MANAGER_RENEWAL_SCHEDULED`            | ACM certificate sẽ auto-renew                                  | 45 ngày             |
| `AWS_ELASTICACHE_MAINTENANCE_SCHEDULED`                | ElastiCache cluster maintenance window                         | 1 tuần              |
| `AWS_LAMBDA_RUNTIME_DEPRECATION_SCHEDULED`             | Lambda runtime sắp kết thúc hỗ trợ (end-of-life)              | 6 tháng             |
| `AWS_RDS_SSL_CA_CERTIFICATE_EXPIRY`                    | Certificate CA của RDS sắp hết hạn, cần update connection code | 90 ngày             |

#### Ví Dụ Chi Tiết: EC2 Instance Retirement

```
Tình huống: AWS phát hiện phần cứng host i-0abc123 có dấu hiệu lỗi

AWS thông báo qua PHD:
┌─────────────────────────────────────────────────────────┐
│ Event Type: AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED       │
│ Status: upcoming                                        │
│ Service: EC2                                            │
│ Region: ap-southeast-1                                  │
│ Affected instances: i-0abc123 (my-web-server)           │
│                     i-0def456 (my-db-backup)            │
│                                                         │
│ Scheduled retirement date: 2026-06-15                   │
│                                                         │
│ Recommended action:                                     │
│   Stop and Start the instance (NOT reboot) to migrate   │
│   it to healthy hardware. This will change the          │
│   instance's public IP unless using Elastic IP.         │
└─────────────────────────────────────────────────────────┘

Hành động cần làm:
1. Stop instance (KHÔNG dùng Reboot)
2. Start lại — AWS tự động migrate sang phần cứng mới
3. Cập nhật DNS nếu không dùng Elastic IP (IP Đàn Hồi)
```

---

### 3. Account Notification Events (Sự Kiện Thông Báo Tài Khoản)

Thông báo liên quan đến account, billing, hoặc service limits — không ảnh hưởng trực tiếp đến infrastructure đang chạy.

**Ví dụ:**

| Event Code                                          | Mô Tả                                                          |
| --------------------------------------------------- | -------------------------------------------------------------- |
| `AWS_ABUSE_IOC_DETECTED`                            | Phát hiện dấu hiệu tài khoản bị compromise hoặc lạm dụng      |
| `AWS_ACCOUNT_IDENTITY_VERIFICATION`                 | Cần xác minh danh tính tài khoản                               |
| `AWS_SERVICE_LIMIT_INCREASE_REQUIRED`               | Gần đạt service quota, cần yêu cầu tăng                       |
| `AWS_BILLING_ANOMALY_DETECTED`                      | Phát hiện chi tiêu bất thường trong tài khoản                 |

---

## Cấu Trúc Sự Kiện Health

### Schema Đầy Đủ của Health Event

```json
{
  "version": "0",
  "id": "7bf73129-1428-4cd3-a780-95db273d1602",
  "detail-type": "AWS Health Event",
  "source": "aws.health",
  "account": "123456789012",
  "time": "2026-05-17T06:27:57Z",
  "region": "ap-southeast-1",
  "resources": [
    "arn:aws:ec2:ap-southeast-1:123456789012:instance/i-0abc123def456"
  ],
  "detail": {
    "eventArn": "arn:aws:health:ap-southeast-1::event/EC2/AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED/AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED_abc123",
    "service": "EC2",
    "eventTypeCode": "AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED",
    "eventTypeCategory": "scheduledChange",
    "startTime": "Wed, 17 May 2026 06:00:00 GMT",
    "endTime": "Mon, 15 Jun 2026 23:59:59 GMT",
    "eventDescription": [
      {
        "language": "en_US",
        "latestDescription": "Your Amazon EC2 instance(s) is scheduled for retirement..."
      }
    ],
    "affectedEntities": [
      {
        "entityValue": "i-0abc123def456",
        "tags": {
          "Name": "my-web-server",
          "Environment": "production"
        },
        "statusCode": "IMPAIRED"
      }
    ]
  }
}
```

### Trường Quan Trọng Cần Nhớ

| Trường                   | Ý Nghĩa                                                         |
| ------------------------ | --------------------------------------------------------------- |
| `service`                | Dịch vụ AWS bị ảnh hưởng (EC2, RDS, Lambda...)                 |
| `eventTypeCode`          | Mã sự kiện cụ thể, dùng để filter trong EventBridge            |
| `eventTypeCategory`      | issue / scheduledChange / accountNotification                   |
| `affectedEntities`       | Danh sách ARN tài nguyên bị ảnh hưởng trong account của bạn   |
| `resources`              | Tương tự affectedEntities, dùng trong EventBridge routing      |
| `eventDescription`       | Mô tả chi tiết và hướng dẫn hành động bằng tiếng Anh          |
| `statusCode`             | IMPAIRED / UNIMPAIRED / UNKNOWN (trạng thái tài nguyên)        |

---

## Truy Cập PHD

### Qua AWS Console

```
AWS Console → Health Dashboard → Your Account Health

Các tab chính:
├── Dashboard: Tổng quan sự kiện hiện tại
├── Event log: Toàn bộ lịch sử 90 ngày
└── Affected resources: Lọc theo tài nguyên cụ thể
```

### Qua AWS CLI

```bash
# Liệt kê tất cả events đang mở ảnh hưởng account
aws health describe-events \
  --filter '{"eventStatusCodes":["open","upcoming"]}' \
  --region us-east-1

# Xem chi tiết một event cụ thể
aws health describe-event-details \
  --event-arns "arn:aws:health:ap-southeast-1::event/EC2/AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED/..." \
  --region us-east-1

# Liệt kê tài nguyên bị ảnh hưởng
aws health describe-affected-entities \
  --filter '{"eventArns":["arn:aws:health:..."], "statusCodes":["IMPAIRED"]}' \
  --region us-east-1

# Xem events của toàn bộ organization (cần Organizations integration)
aws health describe-events-for-organization \
  --filter '{"eventStatusCodes":["open"]}' \
  --region us-east-1
```

> **Lưu ý:** Tất cả Health API calls phải gửi đến **us-east-1** dù tài nguyên ở region khác. Health là global service với single regional endpoint.

### Yêu Cầu IAM Permission

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "health:DescribeEvents",
        "health:DescribeEventDetails",
        "health:DescribeAffectedEntities",
        "health:DescribeAffectedAccountsForOrganization",
        "health:DescribeEventsForOrganization"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## AWS Health API

### Các Endpoint Chính

| API Method                                             | Mục Đích                                                      |
| ------------------------------------------------------ | ------------------------------------------------------------- |
| `DescribeEvents`                                       | Liệt kê events với filter (status, service, region...)       |
| `DescribeEventDetails`                                 | Xem mô tả chi tiết và recommended actions                     |
| `DescribeAffectedEntities`                             | Liệt kê tài nguyên bị ảnh hưởng bởi event                   |
| `DescribeEventTypes`                                   | Xem toàn bộ loại event có thể xảy ra                         |
| `DescribeEventsForOrganization`                        | Events của toàn bộ accounts trong organization                |
| `DescribeAffectedAccountsForOrganization`              | Accounts nào bị ảnh hưởng trong organization                 |
| `EnableHealthServiceAccessForOrganization`             | Bật Organizations integration                                 |

### Filter Hữu Ích

```python
import boto3

health = boto3.client('health', region_name='us-east-1')

# Lấy tất cả events scheduled change đang upcoming
response = health.describe_events(
    filter={
        'eventTypeCategories': ['scheduledChange'],
        'eventStatusCodes': ['upcoming'],
        'services': ['EC2', 'RDS']
    }
)

for event in response['events']:
    print(f"Event: {event['eventTypeCode']}")
    print(f"Service: {event['service']}")
    print(f"Region: {event['region']}")
    print(f"Start: {event['startTime']}")
    print(f"End: {event.get('endTime', 'TBD')}")
    print("---")
```

---

## Thực Hành: Kiểm Tra Events

### Script Python: Báo Cáo Health Events Hàng Ngày

```python
import boto3
from datetime import datetime, timezone
import json

def get_open_health_events():
    """Lấy tất cả Health events đang open/upcoming."""
    health = boto3.client('health', region_name='us-east-1')

    paginator = health.get_paginator('describe_events')
    all_events = []

    pages = paginator.paginate(
        filter={
            'eventStatusCodes': ['open', 'upcoming'],
            'eventTypeCategories': ['issue', 'scheduledChange']
        }
    )

    for page in pages:
        all_events.extend(page['events'])

    return all_events

def get_affected_resources(event_arn):
    """Lấy danh sách tài nguyên bị ảnh hưởng."""
    health = boto3.client('health', region_name='us-east-1')

    response = health.describe_affected_entities(
        filter={'eventArns': [event_arn]}
    )
    return response['entities']

def generate_report():
    events = get_open_health_events()

    if not events:
        print("✅ Không có Health events nào đang active.")
        return

    print(f"⚠️  Tìm thấy {len(events)} Health event(s) cần chú ý:\n")

    for event in events:
        print(f"📌 {event['eventTypeCode']}")
        print(f"   Service:  {event['service']}")
        print(f"   Region:   {event['region']}")
        print(f"   Category: {event['eventTypeCategory']}")
        print(f"   Status:   {event['statusCode']}")
        print(f"   Start:    {event['startTime']}")

        entities = get_affected_resources(event['eventArn'])
        if entities:
            print(f"   Affected resources ({len(entities)}):")
            for entity in entities[:5]:  # Hiển thị tối đa 5
                print(f"     - {entity['entityValue']}")
            if len(entities) > 5:
                print(f"     ... và {len(entities) - 5} tài nguyên khác")
        print()

if __name__ == '__main__':
    generate_report()
```

---

## Tình Huống Thực Tế

### Tình Huống 1: EC2 Retirement Discovery (Khám Phá Instance Sắp Retire)

**Bối cảnh:** Bạn phụ trách fleet 200 EC2 instances cho một ứng dụng thương mại điện tử.

**Sự kiện nhận được:**
```
Event: AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED
Affected: 12 instances tại ap-southeast-1
Retirement date: 2026-06-15 (còn 4 tuần)
```

**Kế hoạch phản hồi:**

```
Tuần 1: Phân tích tác động
├── Xác định 12 instances này thuộc service nào
├── Kiểm tra xem có instance nào chạy stateful workload không
└── Kiểm tra Elastic IP — nếu không có, IP sẽ thay đổi sau migrate

Tuần 2–3: Migrate instances không quan trọng trước
├── Stop → Start từng instance (KHÔNG reboot)
├── Xác nhận instance đã lên phần cứng mới
└── Cập nhật DNS nếu cần

Tuần 4: Migrate instances production
├── Chọn maintenance window (ít traffic nhất)
├── Thông báo team và stakeholders
└── Stop → Start và verify
```

**Script migrate hàng loạt:**

```bash
#!/bin/bash
# Danh sách instance IDs cần migrate
INSTANCES=("i-0abc123" "i-0def456" "i-0ghi789")

for instance_id in "${INSTANCES[@]}"; do
    echo "Migrating $instance_id..."

    # Stop instance
    aws ec2 stop-instances --instance-ids "$instance_id"
    aws ec2 wait instance-stopped --instance-ids "$instance_id"
    echo "  Stopped $instance_id"

    # Start instance (trigger migration to new hardware)
    aws ec2 start-instances --instance-ids "$instance_id"
    aws ec2 wait instance-running --instance-ids "$instance_id"
    echo "  Started $instance_id — migrated to new hardware"

    # Get new IP
    NEW_IP=$(aws ec2 describe-instances \
        --instance-ids "$instance_id" \
        --query 'Reservations[0].Instances[0].PublicIpAddress' \
        --output text)
    echo "  New IP: $NEW_IP"
    echo "---"
done
```

---

### Tình Huống 2: Lambda Runtime Deprecation

**Sự kiện nhận được:**
```
Event: AWS_LAMBDA_RUNTIME_DEPRECATION_SCHEDULED
Service: LAMBDA
Description: Python 3.8 runtime will reach end-of-life on 2026-10-14.
             New Lambda invocations will be blocked after this date.
```

**Kế hoạch phản hồi:**

```bash
# Bước 1: Tìm tất cả Lambda dùng runtime cũ
aws lambda list-functions \
  --query 'Functions[?Runtime==`python3.8`].[FunctionName, Runtime]' \
  --output table

# Bước 2: Update từng function lên Python 3.12
aws lambda update-function-configuration \
  --function-name my-function \
  --runtime python3.12

# Bước 3: Test sau khi update
aws lambda invoke \
  --function-name my-function \
  --payload '{"test": true}' \
  response.json
```

---

## Câu Hỏi Phỏng Vấn

### Q1: PHD khác với Service Health Dashboard ở điểm nào?

**Trả lời ngắn gọn:**

| Điểm khác              | Service Health Dashboard              | Personal Health Dashboard             |
| ---------------------- | ------------------------------------- | ------------------------------------- |
| Ai xem được?           | Công khai, bất kỳ ai                  | Chỉ account của bạn                  |
| Tài nguyên cụ thể?     | Không — chỉ biết service bị ảnh hưởng | Có — biết chính xác resource nào    |
| Hướng dẫn hành động?   | Không                                  | Có — recommended actions cụ thể     |
| EventBridge trigger?   | Không                                  | Có — tự động hóa phản hồi           |
| Yêu cầu đăng nhập?     | Không                                  | Có                                   |

---

### Q2: AWS Health API yêu cầu gì để sử dụng?

**Trả lời:** Cần hai điều kiện:
1. **AWS Business Support hoặc Enterprise Support** — không thể gọi API với Basic/Developer plan
2. **Gọi API từ region us-east-1** — Health là global service, endpoint duy nhất tại us-east-1

---

### Q3: Sự kiện `scheduledChange` vs `issue` khác nhau thế nào?

**Trả lời:**
- **issue**: Sự cố bất ngờ đang diễn ra, cần phản hồi ngay. Ví dụ: EC2 API degraded
- **scheduledChange**: Thay đổi có lịch, AWS thông báo trước để bạn chuẩn bị. Ví dụ: instance retirement sau 4 tuần

---

### Q4: Làm sao biết instance nào trong fleet bị retirement mà không check Console thủ công?

**Trả lời:** Dùng EventBridge bắt event `AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED`, trigger Lambda để:
- Parse danh sách `affectedEntities` từ event
- Ghi vào DynamoDB hoặc gửi email/Slack report
- Tự động khởi chạy runbook stop/start instance

---

**Điều Hướng:**
- ← [README.md — Tổng quan Health Dashboard](./README.md)
- → [2-eventbridge-integration.md — Tự Động Hóa với EventBridge](./2-eventbridge-integration.md)

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
