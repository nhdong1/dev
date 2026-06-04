# ☠️ Giám Sát DLQ — Phát Hiện và Xử Lý Tin Nhắn Lỗi

> Hướng dẫn toàn diện về giám sát Dead Letter Queue (DLQ — Hàng Đợi Thư Chết), phân tích poison message (tin nhắn độc hại), và chiến lược phục hồi cho hệ thống messaging AWS.

## 📚 Mục Lục

1. [DLQ Là Gì và Tại Sao Quan Trọng](#dlq-là-gì)
2. [DLQ Trong Từng Dịch Vụ AWS](#dlq-trong-từng-dịch-vụ-aws)
3. [Chiến Lược Giám Sát DLQ](#chiến-lược-giám-sát-dlq)
4. [Phân Tích Poison Message](#phân-tích-poison-message)
5. [Chiến Lược Tái Xử Lý — Message Replay](#chiến-lược-tái-xử-lý)
6. [Tự Động Hóa Xử Lý DLQ](#tự-động-hóa-xử-lý-dlq)
7. [Thiết Kế DLQ Đa Tầng](#thiết-kế-dlq-đa-tầng)
8. [Runbook — Sổ Tay Vận Hành](#runbook)

---

## DLQ Là Gì

**Dead Letter Queue** (DLQ — Hàng Đợi Thư Chết) là hàng đợi đặc biệt nhận những tin nhắn không thể xử lý thành công sau một số lần thử lại nhất định.

### Tin Nhắn Vào DLQ Khi Nào?

```
Kịch bản 1 — Consumer trả lỗi liên tục:
  Message → Queue → Consumer (lỗi lần 1)
                  → Consumer (lỗi lần 2)
                  → Consumer (lỗi lần 3 — hết maxReceiveCount)
                  → DLQ ✓

Kịch bản 2 — Visibility Timeout hết nhiều lần:
  Message → Consumer nhận nhưng không xử lý xong
          → Visibility Timeout hết → Message hiện lại
          → Xảy ra nhiều lần → DLQ ✓

Kịch bản 3 — Lambda timeout:
  Message → Lambda bị timeout → Message trả về queue
          → Retry nhiều lần → DLQ ✓
```

### Tại Sao DLQ Quan Trọng Trong Production?

```
Không có DLQ:
  Message lỗi → Thử lại mãi → Block queue → Các message khác bị delay
  → Sau khi hết retention period (4–14 ngày) → Message MẤT VĨNH VIỄN

Có DLQ:
  Message lỗi → Vào DLQ → Các message bình thường tiếp tục xử lý
  → DLQ lưu message lỗi để debug và replay (phát lại)
  → Không mất dữ liệu
```

---

## DLQ Trong Từng Dịch Vụ AWS

### SQS — Dead Letter Queue

**Cấu hình `maxReceiveCount`**: Số lần tối đa tin nhắn được nhận trước khi vào DLQ.

```bash
# Tạo DLQ trước
aws sqs create-queue --queue-name order-queue-dlq

# Lấy ARN của DLQ
DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-southeast-1.amazonaws.com/123456789/order-queue-dlq \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Tạo main queue với redrive policy (chính sách chuyển hướng)
aws sqs create-queue \
  --queue-name order-queue \
  --attributes "{
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\",
    \"VisibilityTimeout\": \"30\"
  }"
```

**Lựa Chọn `maxReceiveCount`:**

```
maxReceiveCount = 1: Không retry — ngay lập tức vào DLQ khi lỗi
  Dùng khi: Idempotent operation, lỗi do dữ liệu không phải hạ tầng

maxReceiveCount = 3 (khuyến nghị): Retry 3 lần trước khi vào DLQ
  Dùng khi: Lỗi tạm thời có thể tự giải quyết (network timeout, throttle)

maxReceiveCount = 10: Retry nhiều — ít vào DLQ hơn
  Dùng khi: Các operation có latency cao, hệ thống downstream không ổn định

LƯU Ý: Tăng maxReceiveCount → Tin nhắn lỗi chiếm tài nguyên lâu hơn
```

### SNS — Dead Letter Queue Cho Subscription

SNS DLQ dành cho subscription (đăng ký) — khi SNS không giao được tin nhắn đến subscriber:

```bash
# Tạo DLQ cho SNS subscription
aws sqs create-queue --queue-name sns-subscription-dlq

# Thiết lập DLQ cho SNS subscription
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:ap-southeast-1:123456789:order-topic:abc123 \
  --attribute-name RedrivePolicy \
  --attribute-value '{"deadLetterTargetArn": "arn:aws:sqs:ap-southeast-1:123456789:sns-subscription-dlq"}'
```

**SNS DLQ khác SQS DLQ:**

```
SQS DLQ: Xử lý khi CONSUMER thất bại (nhận nhưng không xử lý được)
SNS DLQ: Xử lý khi SNS không GIAO ĐƯỢC tin nhắn đến subscriber
  (HTTP endpoint down, SQS queue đầy, Lambda throttle...)
```

### EventBridge — Dead Letter Queue Cho Target

EventBridge DLQ lưu event khi target (đích) không thể nhận:

```bash
# Cấu hình DLQ cho EventBridge rule target
aws events put-targets \
  --rule process-order-rule \
  --event-bus-name default \
  --targets '[{
    "Id": "lambda-target",
    "Arn": "arn:aws:lambda:ap-southeast-1:123456789:function:order-processor",
    "DeadLetterConfig": {
      "Arn": "arn:aws:sqs:ap-southeast-1:123456789:eventbridge-dlq"
    },
    "RetryPolicy": {
      "MaximumRetryAttempts": 185,
      "MaximumEventAgeInSeconds": 86400
    }
  }]'
```

### Lambda Event Source Mapping DLQ

Khi Lambda được trigger bởi SQS/Kinesis, cấu hình DLQ ở cấp Event Source Mapping (Ánh Xạ Nguồn Sự Kiện):

```bash
aws lambda create-event-source-mapping \
  --function-name order-processor \
  --event-source-arn arn:aws:sqs:ap-southeast-1:123456789:order-queue \
  --batch-size 10 \
  --bisect-batch-on-function-error true \
  --destination-config '{
    "OnFailure": {
      "Destination": "arn:aws:sqs:ap-southeast-1:123456789:order-queue-dlq"
    }
  }'
```

**`bisect-batch-on-function-error`** (Chia Batch Khi Lỗi): Khi Lambda lỗi, tự động chia batch làm đôi và thử lại từng nửa — giúp cô lập poison message.

---

## Chiến Lược Giám Sát DLQ

### Alarm Ngay Khi DLQ Có Tin Nhắn

```bash
# Alarm mức CRITICAL — kích hoạt ngay khi DLQ không còn rỗng
aws cloudwatch put-metric-alarm \
  --alarm-name "DLQ-HasMessages-$(queue_name)" \
  --metric-name ApproximateNumberOfMessagesVisible \
  --namespace AWS/SQS \
  --dimensions Name=QueueName,Value=order-queue-dlq \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:critical-alert-topic \
  --treat-missing-data notBreaching
```

### Dashboard Giám Sát DLQ

```
Widget cần có trên dashboard DLQ:

1. DLQ Message Count (Số Lượng Tin Nhắn DLQ)
   Metric: ApproximateNumberOfMessagesVisible
   Hiển thị: Line chart theo thời gian

2. DLQ Message Age (Tuổi Tin Nhắn DLQ)
   Metric: ApproximateAgeOfOldestMessage
   Cảnh báo: Tin nhắn sắp hết TTL (Time To Live — Thời Gian Sống)

3. Main Queue vs DLQ Ratio (Tỷ Lệ)
   Math expression: dlq_count / main_count * 100
   Tỷ lệ lỗi > 1% là bất thường

4. Alarm Status Widget
   Hiển thị trạng thái tất cả DLQ alarm trong một view
```

### CloudWatch Log Insights Cho DLQ

```sql
-- Phân tích lỗi trong Lambda consumer để tìm nguyên nhân DLQ
fields @timestamp, @message, @requestId
| filter @message like /ERROR/ or @message like /Exception/
| parse @message /orderId=(?<orderId>[^ ]+)/
| stats count() as errorCount by orderId
| sort errorCount desc
| limit 20
```

---

## Phân Tích Poison Message

**Poison Message** (Tin Nhắn Độc Hại) là tin nhắn gây lỗi liên tục do dữ liệu không hợp lệ hoặc logic xử lý không tương thích — phân biệt với lỗi tạm thời do hạ tầng.

### Phân Biệt Poison Message vs Lỗi Tạm Thời

```
Lỗi Tạm Thời (Transient Error — Lỗi Thoáng Qua):
  - Network timeout → Retry thường thành công
  - Database throttle → Sau ít phút hết
  - Lambda cold start → Lần sau nhanh hơn
  Xử lý: Retry với exponential backoff (tăng dần thời gian chờ)

Poison Message:
  - JSON không hợp lệ → Mọi consumer đều parse lỗi
  - Thiếu field bắt buộc → Validation luôn thất bại
  - Giá trị không hợp lệ (số âm cho số lượng, ngày trong quá khứ...)
  - Version không tương thích (consumer cũ nhận event format mới)
  Xử lý: Cô lập và phân tích, KHÔNG retry mù quáng
```

### Quy Trình Phân Tích Tin Nhắn Trong DLQ

```python
import boto3
import json
from datetime import datetime

sqs = boto3.client('sqs')

def analyze_dlq(dlq_url: str, max_messages: int = 10):
    """Đọc và phân tích tin nhắn trong DLQ mà không xoá."""
    
    response = sqs.receive_message(
        QueueUrl=dlq_url,
        MaxNumberOfMessages=min(max_messages, 10),
        AttributeNames=['All'],
        MessageAttributeNames=['All'],
        VisibilityTimeout=300  # Giữ visible trong 5 phút để phân tích
    )
    
    messages = response.get('Messages', [])
    
    if not messages:
        print("DLQ đang rỗng — hệ thống hoạt động bình thường")
        return
    
    print(f"Tìm thấy {len(messages)} tin nhắn trong DLQ\n")
    
    for i, msg in enumerate(messages, 1):
        attrs = msg.get('Attributes', {})
        print(f"=== Tin nhắn #{i} ===")
        print(f"Message ID: {msg['MessageId']}")
        print(f"Số lần nhận: {attrs.get('ApproximateReceiveCount', 'N/A')}")
        print(f"Gửi lần đầu: {datetime.fromtimestamp(int(attrs.get('ApproximateFirstReceiveTimestamp', 0)) / 1000)}")
        print(f"Nội dung body (200 ký tự đầu):")
        
        body = msg['Body']
        try:
            # Thử parse JSON để dễ đọc hơn
            parsed = json.loads(body)
            print(json.dumps(parsed, indent=2, ensure_ascii=False)[:500])
        except json.JSONDecodeError:
            print(body[:200])
        
        # Kiểm tra các dấu hiệu poison message
        diagnose_message(body)
        print()
    
    # LƯU Ý: Không xoá tin nhắn sau khi phân tích
    # Visibility timeout sẽ hết và message hiện lại trong DLQ

def diagnose_message(body: str) -> dict:
    """Chẩn đoán nguyên nhân tin nhắn lỗi."""
    issues = []
    
    # Kiểm tra JSON hợp lệ
    try:
        data = json.loads(body)
    except json.JSONDecodeError as e:
        issues.append(f"JSON không hợp lệ: {e}")
        return issues
    
    # Kiểm tra các field bắt buộc
    required_fields = ['orderId', 'customerId', 'items', 'total']
    for field in required_fields:
        if field not in data:
            issues.append(f"Thiếu field bắt buộc: {field}")
    
    # Kiểm tra giá trị hợp lệ
    if 'total' in data and data['total'] <= 0:
        issues.append(f"Tổng tiền không hợp lệ: {data['total']}")
    
    if 'items' in data and len(data['items']) == 0:
        issues.append("Đơn hàng không có sản phẩm nào")
    
    if issues:
        print(f"Phát hiện {len(issues)} vấn đề:")
        for issue in issues:
            print(f"  ⚠️  {issue}")
    else:
        print("Không phát hiện vấn đề rõ ràng — có thể là lỗi hạ tầng tạm thời")
    
    return issues
```

---

## Chiến Lược Tái Xử Lý — Message Replay

### 1. Replay Thủ Công

Khi đã fix lỗi và muốn tái xử lý tin nhắn từ DLQ:

```python
import boto3

sqs = boto3.client('sqs')

def replay_dlq_to_main_queue(
    dlq_url: str,
    main_queue_url: str,
    max_messages: int = None,
    dry_run: bool = True
):
    """
    Chuyển tin nhắn từ DLQ về main queue để tái xử lý.
    dry_run=True: Chỉ đọc và in ra, không thực sự chuyển.
    """
    
    replayed = 0
    
    while True:
        response = sqs.receive_message(
            QueueUrl=dlq_url,
            MaxNumberOfMessages=10,
            VisibilityTimeout=30,
            WaitTimeSeconds=1
        )
        
        messages = response.get('Messages', [])
        if not messages:
            break
        
        for msg in messages:
            if max_messages and replayed >= max_messages:
                return replayed
            
            print(f"Replay: {msg['MessageId']}")
            
            if not dry_run:
                # Gửi lại vào main queue
                sqs.send_message(
                    QueueUrl=main_queue_url,
                    MessageBody=msg['Body'],
                    MessageAttributes=msg.get('MessageAttributes', {})
                )
                
                # Xoá khỏi DLQ sau khi đã chuyển thành công
                sqs.delete_message(
                    QueueUrl=dlq_url,
                    ReceiptHandle=msg['ReceiptHandle']
                )
            
            replayed += 1
    
    print(f"{'[DRY RUN] ' if dry_run else ''}Đã replay {replayed} tin nhắn")
    return replayed

# Sử dụng:
# Kiểm tra trước (dry run)
replay_dlq_to_main_queue(dlq_url, main_url, dry_run=True)

# Thực sự replay sau khi xác nhận
# replay_dlq_to_main_queue(dlq_url, main_url, dry_run=False)
```

### 2. SQS Redrive — Tính Năng Tích Hợp AWS

AWS cung cấp tính năng **Redrive** (Tái Định Tuyến) tích hợp sẵn — không cần code:

```bash
# Bật redrive từ DLQ về source queue
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:ap-southeast-1:123456789:order-queue-dlq \
  --destination-arn arn:aws:sqs:ap-southeast-1:123456789:order-queue \
  --max-number-of-messages-per-second 10  # Giới hạn tốc độ replay

# Kiểm tra tiến độ
aws sqs list-message-move-tasks \
  --source-arn arn:aws:sqs:ap-southeast-1:123456789:order-queue-dlq

# Dừng replay nếu cần
aws sqs cancel-message-move-task \
  --task-handle <task-handle-from-list>
```

### 3. Replay Có Chọn Lọc (Selective Replay)

Không phải mọi tin nhắn trong DLQ đều cần replay — một số là poison message thực sự:

```python
def selective_replay(dlq_url: str, main_queue_url: str, dead_letter_url: str):
    """
    Phân loại tin nhắn DLQ:
    - Lỗi tạm thời → Replay về main queue
    - Poison message → Chuyển sang permanent dead letter queue
    - Cần review thủ công → Giữ trong DLQ
    """
    
    messages = read_all_dlq_messages(dlq_url)
    
    for msg in messages:
        body = json.loads(msg['Body'])
        receive_count = int(msg['Attributes']['ApproximateReceiveCount'])
        
        issues = diagnose_message(msg['Body'])
        
        if not issues and receive_count <= 3:
            # Có thể là lỗi tạm thời — replay
            replay_to_main(msg, main_queue_url)
            delete_from_dlq(msg, dlq_url)
            
        elif issues:
            # Poison message — chuyển vào permanent dead letter (lưu trữ lâu dài)
            archive_poison_message(msg, dead_letter_url, issues)
            delete_from_dlq(msg, dlq_url)
            
        else:
            # Không chắc — để lại trong DLQ để review thủ công
            print(f"Cần review thủ công: {msg['MessageId']}")
```

---

## Tự Động Hóa Xử Lý DLQ

### Lambda Tự Động Phân Tích Khi DLQ Có Tin Nhắn

```python
# Lambda được trigger khi CloudWatch Alarm báo DLQ không rỗng

import boto3
import json

sqs = boto3.client('sqs')
sns = boto3.client('sns')

def handler(event, context):
    """Tự động phân tích DLQ và gửi báo cáo."""
    
    # Đọc tin nhắn từ DLQ (không xoá)
    response = sqs.receive_message(
        QueueUrl=DLQ_URL,
        MaxNumberOfMessages=10,
        VisibilityTimeout=300,
        AttributeNames=['All']
    )
    
    messages = response.get('Messages', [])
    if not messages:
        return
    
    # Phân tích nguyên nhân
    analysis = {
        'total_messages': len(messages),
        'poison_messages': [],
        'transient_errors': [],
        'unknown': []
    }
    
    for msg in messages:
        issues = diagnose_message(msg['Body'])
        receive_count = int(msg['Attributes']['ApproximateReceiveCount'])
        
        if issues:
            analysis['poison_messages'].append({
                'messageId': msg['MessageId'],
                'issues': issues,
                'receiveCount': receive_count
            })
        elif receive_count <= 3:
            analysis['transient_errors'].append(msg['MessageId'])
        else:
            analysis['unknown'].append(msg['MessageId'])
    
    # Gửi báo cáo
    report = f"""
DLQ ALERT: {analysis['total_messages']} tin nhắn trong DLQ

PHÂN TÍCH:
- Poison message (lỗi dữ liệu): {len(analysis['poison_messages'])}
- Lỗi tạm thời (có thể replay): {len(analysis['transient_errors'])}
- Cần review thủ công: {len(analysis['unknown'])}

CHI TIẾT POISON MESSAGE:
{json.dumps(analysis['poison_messages'], indent=2, ensure_ascii=False)}

HÀNH ĐỘNG ĐỀ XUẤT:
1. Review poison message và fix code/data nếu cần
2. Replay lỗi tạm thời sau khi hệ thống ổn định
3. Kiểm tra DLQ retention để tránh mất message
    """
    
    sns.publish(
        TopicArn=ALERT_TOPIC_ARN,
        Subject=f"DLQ Alert: {analysis['total_messages']} tin nhắn cần xử lý",
        Message=report
    )
    
    return analysis
```

---

## Thiết Kế DLQ Đa Tầng

Hệ thống production phức tạp nên có nhiều lớp DLQ:

```
Tầng 1: Main Queue
  maxReceiveCount = 3
  VisibilityTimeout = 30 giây
  Retention: 4 ngày
        ↓ (sau 3 lần lỗi)

Tầng 2: DLQ Tự Động Retry (Retry DLQ)
  → Lambda trigger mỗi 5 phút
  → Retry lên Main Queue nếu lỗi tạm thời
  → Chuyển xuống tầng 3 nếu vẫn lỗi sau 3 lần
  Retention: 7 ngày

Tầng 3: Permanent Dead Letter (Lưu Trữ Vĩnh Cửu)
  → Lưu tin nhắn lỗi thực sự không thể xử lý
  → Human review (con người xem xét) hàng ngày
  → Cần tạo task để fix và replay
  Retention: 14 ngày

Tầng 4: S3 Archive (Lưu Trữ S3)
  → Backup tất cả tin nhắn từ Permanent Dead Letter
  → Lưu vĩnh viễn để audit và compliance
  → Có thể replay từ S3 nếu cần thiết
```

### Terraform Cho DLQ Đa Tầng

```hcl
# Tầng 1: Main Queue
resource "aws_sqs_queue" "main" {
  name                       = "order-queue"
  visibility_timeout_seconds = 30
  message_retention_seconds  = 345600  # 4 ngày
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.retry_dlq.arn
    maxReceiveCount     = 3
  })
}

# Tầng 2: Retry DLQ
resource "aws_sqs_queue" "retry_dlq" {
  name                      = "order-queue-retry-dlq"
  message_retention_seconds = 604800  # 7 ngày
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.permanent_dlq.arn
    maxReceiveCount     = 3
  })
}

# Tầng 3: Permanent Dead Letter
resource "aws_sqs_queue" "permanent_dlq" {
  name                      = "order-queue-permanent-dlq"
  message_retention_seconds = 1209600  # 14 ngày
}

# Lambda retry processor cho tầng 2
resource "aws_lambda_event_source_mapping" "retry_processor" {
  event_source_arn = aws_sqs_queue.retry_dlq.arn
  function_name    = aws_lambda_function.dlq_retry_processor.arn
  batch_size       = 1  # Xử lý từng tin nhắn để dễ debug
  
  filter_criteria {
    filter {
      # Chỉ retry sau 5 phút
      pattern = jsonencode({
        # Custom filter nếu cần
      })
    }
  }
}
```

---

## Runbook — Sổ Tay Vận Hành

### Khi Nhận Alarm DLQ

```
BƯỚC 1 — Đánh Giá Mức Độ (5 phút)
  □ Xem số lượng tin nhắn trong DLQ
  □ Xem tốc độ tăng (đang tăng nhanh hay chậm?)
  □ Xem ApproximateAgeOfOldestMessage trong main queue
    → Nếu main queue đang tăng → Vấn đề nghiêm trọng hơn

BƯỚC 2 — Kiểm Tra Consumer (10 phút)
  □ Lambda Errors có tăng không?
  □ Lambda Throttles có không?
  □ CloudWatch Logs của consumer có lỗi gì?
  □ X-Ray traces có thấy pattern lỗi không?

BƯỚC 3 — Phân Tích Tin Nhắn DLQ (15 phút)
  □ Đọc 1–5 tin nhắn đầu trong DLQ (không xoá)
  □ Chạy script analyze_dlq() để chẩn đoán
  □ Phân loại: Poison message hay lỗi tạm thời?

BƯỚC 4 — Hành Động
  Nếu lỗi tạm thời (hạ tầng, network):
    □ Đợi hệ thống ổn định
    □ Replay từ DLQ về main queue (dùng SQS Redrive)
    □ Theo dõi để đảm bảo xử lý thành công

  Nếu poison message (lỗi dữ liệu):
    □ Cô lập message — đừng retry mù quáng
    □ Tìm nguyên nhân: Schema thay đổi? Bug mới? Data corrupt?
    □ Fix code hoặc dữ liệu
    □ Deploy fix
    □ Replay có chọn lọc

  Nếu lỗi code (logic mới):
    □ Rollback (khôi phục) deployment nếu cần
    □ Fix code
    □ Test kỹ
    □ Deploy lại
    □ Replay từ DLQ

BƯỚC 5 — Sau Sự Cố
  □ Viết post-mortem (báo cáo phân tích sự cố)
  □ Cập nhật runbook nếu cần
  □ Thêm validation để ngăn loại lỗi tương tự
  □ Xem xét tăng test coverage
```

### Lệnh Hay Dùng Khi Xử Lý DLQ

```bash
# Xem số lượng tin nhắn trong DLQ
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-southeast-1.amazonaws.com/123456789/order-queue-dlq \
  --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage

# Đọc 1 tin nhắn để xem nội dung (không xoá)
aws sqs receive-message \
  --queue-url https://sqs.ap-southeast-1.amazonaws.com/123456789/order-queue-dlq \
  --max-number-of-messages 1 \
  --visibility-timeout 300 \
  --attribute-names All

# Bắt đầu replay từ DLQ về main queue
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:ap-southeast-1:123456789:order-queue-dlq \
  --destination-arn arn:aws:sqs:ap-southeast-1:123456789:order-queue \
  --max-number-of-messages-per-second 5

# Xoá TẤT CẢ tin nhắn trong DLQ (dùng cẩn thận!)
# Chỉ dùng khi đã xác nhận là poison message không cần replay
aws sqs purge-queue \
  --queue-url https://sqs.ap-southeast-1.amazonaws.com/123456789/order-queue-dlq
```

---

## Tóm Tắt Best Practice

| Hành Động | Khuyến Nghị |
|---|---|
| **maxReceiveCount** | 3–5 cho hầu hết use case |
| **DLQ retention** | Ít nhất gấp đôi main queue retention |
| **Alarm ngưỡng** | Alarm ngay khi DLQ > 0 tin nhắn |
| **Không xoá ngay** | Đọc và phân tích trước khi xoá khỏi DLQ |
| **Replay từ từ** | Giới hạn tốc độ replay để tránh overwhelm consumer |
| **Dry run** | Luôn chạy dry run trước khi replay thực sự |
| **Post-mortem** | Viết phân tích sau mỗi sự cố DLQ đáng kể |
| **Monitoring** | Có dashboard riêng cho DLQ metrics |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Xem Thêm:**
- [1-key-metrics.md](1-key-metrics.md) — Metrics quan trọng cho monitoring
- [2-cloudwatch-alarms.md](2-cloudwatch-alarms.md) — Cấu hình alarm cho DLQ
- [3-xray-tracing.md](3-xray-tracing.md) — Trace để tìm nguyên nhân lỗi
- [../01-sqs/3-dead-letter-queue.md](../01-sqs/3-dead-letter-queue.md) — Kiến thức nền về DLQ
