# Competing Consumers — Mở Rộng Xử Lý Hàng Đợi Song Song

> Competing Consumers (Người Tiêu Dùng Cạnh Tranh) là pattern nhiều worker instance cùng xử lý một queue, cho phép scale out (mở rộng ngang) throughput (thông lượng) xử lý tin nhắn.

---

## 📚 Mục Lục

1. [Vấn Đề Cần Giải Quyết](#vấn-đề-cần-giải-quyết)
2. [Competing Consumers Pattern](#competing-consumers-pattern)
3. [SQS Competing Consumers](#sqs-competing-consumers)
4. [Kinesis — Partition-Based Consumers](#kinesis--partition-based-consumers)
5. [Autoscaling — Tự Động Mở Rộng](#autoscaling--tự-động-mở-rộng)
6. [SQS vs Kinesis: Consuming Model](#sqs-vs-kinesis-consuming-model)
7. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
8. [Anti-Patterns Cần Tránh](#anti-patterns-cần-tránh)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Cần Giải Quyết

### Queue Tích Tụ

```
Kịch bản:
- Black Friday sale: 10,000 orders/giây
- Order Processor: xử lý 100 orders/giây
- Queue tích lũy: 10,000 - 100 = 9,900 messages/giây
- Sau 1 giờ: 35,640,000 messages chưa xử lý ❌

Giải pháp đơn giản: Chạy 100 Order Processor instances
- Mỗi instance xử lý 100 orders/giây
- Tổng: 10,000 orders/giây = sạch queue
```

### Thách Thức

```
Nếu 100 instances cùng đọc từ một queue:
- Làm sao đảm bảo mỗi message chỉ xử lý bởi MỘT instance?
- Làm sao tránh 2 instances cùng nhận một message?
- Làm sao handle instance crash giữa chừng?
```

---

## Competing Consumers Pattern

### Nguyên Lý Hoạt Động

```
┌─────────────────────────────────────────────────────────────┐
│                       SQS Queue                             │
│  [MSG-1] [MSG-2] [MSG-3] [MSG-4] [MSG-5] [MSG-6] ...       │
└──────────────┬──────────────────────────────────────────────┘
               │
     ┌─────────┴──────────┐
     │                    │
┌────▼────┐         ┌─────▼───┐         ┌─────────┐
│Consumer │         │Consumer │         │Consumer │
│Instance │         │Instance │         │Instance │
│   #1    │         │   #2    │         │   #3    │
│         │         │         │         │         │
│ MSG-1 ✓ │         │ MSG-2 ✓ │         │ MSG-3 ✓ │
│ MSG-4 ✓ │         │ MSG-5 ✓ │         │ MSG-6 ✓ │
└─────────┘         └─────────┘         └─────────┘
```

### Cơ Chế SQS Visibility Timeout

```
Visibility Timeout (Thời Gian Ẩn) là chìa khóa:

1. Consumer #1 RECEIVE MSG-1
   → MSG-1 bị ẩn khỏi queue trong 30 giây (visibility timeout)
   → Consumer #2, #3 không thể nhận MSG-1

2. Consumer #1 XỬ LÝ MSG-1 (mất 5 giây)

3. Consumer #1 DELETE MSG-1
   → MSG-1 bị xóa vĩnh viễn khỏi queue

4. Nếu Consumer #1 CRASH sau bước 1:
   → Sau 30 giây, MSG-1 REAPPEARS (xuất hiện lại)
   → Consumer #2 hoặc #3 có thể nhận và xử lý
   → Đây là tại sao consumer phải IDEMPOTENT
```

---

## SQS Competing Consumers

### Long Polling — Nhận Tin Nhắn Hiệu Quả

```python
import boto3
import json
import threading

sqs = boto3.client('sqs')
QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/123/order-processing-queue'

def consumer_worker(worker_id: int):
    print(f"Worker {worker_id} started")
    
    while True:
        # Long polling: chờ tối đa 20 giây nếu không có message
        # → Giảm chi phí API calls so với short polling
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10,          # Batch tối đa 10 messages
            WaitTimeSeconds=20,              # Long polling 20 giây
            VisibilityTimeout=30,            # Ẩn 30 giây trong lúc xử lý
            AttributeNames=['ApproximateReceiveCount']
        )
        
        messages = response.get('Messages', [])
        
        for message in messages:
            receipt_handle = message['ReceiptHandle']
            body = json.loads(message['Body'])
            receive_count = int(message['Attributes']['ApproximateReceiveCount'])
            
            try:
                process_order(body, worker_id)
                
                # Xóa message sau khi xử lý thành công
                sqs.delete_message(
                    QueueUrl=QUEUE_URL,
                    ReceiptHandle=receipt_handle
                )
                
            except Exception as e:
                print(f"Worker {worker_id}: Error processing {body['order_id']}: {e}")
                
                # Nếu đã nhận nhiều lần → có thể là poison message (tin nhắn độc hại)
                if receive_count >= 3:
                    print(f"Poison message detected, sending to DLQ manually")
                    # SQS tự động gửi vào DLQ khi vượt maxReceiveCount
                
                # Không delete → message reappears sau visibility timeout
                # Có thể extend visibility nếu cần thêm thời gian xử lý
                if processing_will_take_longer:
                    sqs.change_message_visibility(
                        QueueUrl=QUEUE_URL,
                        ReceiptHandle=receipt_handle,
                        VisibilityTimeout=60  # Extend thêm 60 giây
                    )


def run_competing_consumers(num_workers: int = 5):
    """Chạy nhiều consumer instances song song."""
    threads = []
    for i in range(num_workers):
        t = threading.Thread(target=consumer_worker, args=(i,))
        t.daemon = True
        t.start()
        threads.append(t)
    
    for t in threads:
        t.join()
```

### SQS + Lambda — Serverless Competing Consumers

```
Lambda là cách đơn giản nhất để implement competing consumers:
- Lambda tự động scale (mở rộng) số instances theo queue depth
- Mỗi Lambda invocation xử lý một batch của messages
- Lambda Event Source Mapping quản lý polling và deletion

Cấu hình tối ưu:
- BatchSize: 1–10 (tùy thời gian xử lý mỗi message)
- MaximumConcurrency: giới hạn số Lambda instances đồng thời
- ReportBatchItemFailures: xử lý partial failure trong batch
```

```python
# Lambda handler với partial batch failure

def lambda_handler(event, context):
    failed_message_ids = []
    
    for record in event['Records']:
        message_id = record['messageId']
        body = json.loads(record['body'])
        
        try:
            process_order_idempotent(message_id, body)
        except Exception as e:
            print(f"Failed to process {message_id}: {e}")
            failed_message_ids.append(message_id)
    
    # Trả về danh sách messages THẤT BẠI
    # Lambda sẽ chỉ xóa messages THÀNH CÔNG
    # Messages thất bại sẽ reappear (xuất hiện lại)
    return {
        'batchItemFailures': [
            {'itemIdentifier': msg_id} for msg_id in failed_message_ids
        ]
    }
```

### Tính Toán Số Consumer Cần Thiết

```
Công thức:
  Số consumer = (Incoming rate × Processing time) / Batch size

Ví dụ:
  Incoming rate: 1,000 messages/giây
  Processing time per message: 0.5 giây
  Batch size: 10 messages

  Số consumer = (1,000 × 0.5) / 10 = 50 consumer instances

Với Lambda:
  Lambda MaximumConcurrency = 50
  Mỗi Lambda xử lý 10 messages/batch
  → 50 × 10 = 500 messages đang xử lý cùng lúc
  → 500 / 0.5s = 1,000 messages/giây ✅
```

---

## Kinesis — Partition-Based Consumers

### Kinesis Consumer Model Khác SQS

```
SQS:                          Kinesis:
Any consumer can get          Each shard has ONE consumer
any message                   processing at a time

┌──────────────────┐          ┌──────────────────────────┐
│ [M1][M2][M3][M4] │          │ Shard-1: [M1][M3][M5]   │
│      Queue       │          │ Shard-2: [M2][M4][M6]   │
└────────┬─────────┘          │ Shard-3: [M7][M8][M9]   │
         │                    └────┬────────┬────────────┘
  ┌──────┼──────┐                  │        │
  ▼      ▼      ▼             ┌────▼──┐ ┌───▼───┐
 C1     C2     C3            │Cons-1 │ │Cons-2 │ (Cons-3 for Shard-3)
 (any)  (any) (any)          │Shard-1│ │Shard-2│
                             └───────┘ └───────┘
```

**Hàm ý:**
- SQS: scale bằng cách thêm consumer instance (không giới hạn)
- Kinesis: scale bằng cách thêm shard (mỗi shard tối đa 2MB/s read)
- Kinesis đảm bảo ordering trong shard — SQS Standard không đảm bảo

### Kinesis Consumer Groups

```
Kinesis Data Streams:
  Shard-1  ──────┬──────► Consumer Group A (Lambda function #1)
                 └──────► Consumer Group B (Lambda function #2)
                          (Enhanced Fan-Out: mỗi nhóm 2MB/s)

Không như SQS — nhiều Consumer Group ĐỀU nhận cùng dữ liệu!
→ Consumer Group A: real-time alerts
→ Consumer Group B: archive to S3
→ Cả hai nhận cùng events từ Shard-1
```

```python
# Lambda với Kinesis trigger — xử lý theo shard
def lambda_handler(event, context):
    shard_id = event['Records'][0]['kinesis']['sequenceNumber'].split(':')[0]
    
    for record in event['Records']:
        # Kinesis records được deliver theo thứ tự trong shard
        partition_key = record['kinesis']['partitionKey']
        sequence_number = record['kinesis']['sequenceNumber']
        data = json.loads(record['kinesis']['data'])
        
        try:
            process_record_idempotent(sequence_number, data)
        except Exception as e:
            # Kinesis: lỗi sẽ STOP processing của shard đó
            # Cần cẩn thận với error handling
            print(f"Error: {e}")
            raise  # Re-raise để Lambda retry từ record này
```

---

## Autoscaling — Tự Động Mở Rộng

### ECS Autoscaling Dựa Trên Queue Depth

```
Chiến lược:
  Metric: ApproximateNumberOfMessagesVisible (SQS)
  Trigger: message count > 1000 → scale out
  Scale in: message count < 100 → scale in

CloudWatch Alarm → Auto Scaling Policy → ECS Service
```

```json
// Auto Scaling Policy
{
  "PolicyName": "SQSQueueDepthScaling",
  "PolicyType": "TargetTrackingScaling",
  "TargetTrackingScalingPolicyConfiguration": {
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [{"Name": "QueueName", "Value": "order-processing-queue"}],
      "Statistic": "Sum"
    },
    "TargetValue": 100.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }
}
```

### Lambda Autoscaling (Tự Động)

```
Lambda tự động scale dựa trên SQS queue depth:
- SQS event source mapping polling liên tục
- Khi queue tăng → Lambda tăng concurrency (đồng thời)
- Giới hạn: ReservedConcurrency (đồng thời dành riêng), AccountConcurrencyLimit

Cài đặt khuyến nghị:
  maxReceiveCount: 3–5 (trước khi gửi vào DLQ)
  visibilityTimeout: ≥ 6 × Lambda timeout (để tránh reappear khi đang xử lý)
  batchSize: 1 nếu processing nặng; 10 nếu processing nhẹ
```

### Kinesis Enhanced Fan-Out Autoscaling

```
Kinesis Standard:
  Pull-based: Consumer poll shard → 2MB/s CHIA SẺ giữa các consumer
  2 consumers → mỗi consumer 1MB/s

Kinesis Enhanced Fan-Out (Khuếch Tán Nâng Cao):
  Push-based: Kinesis PUSH data đến mỗi consumer → 2MB/s MỖI consumer
  2 consumers → mỗi consumer 2MB/s (không chia sẻ)
  Chi phí: $0.015 / shard-hour thêm
```

---

## SQS vs Kinesis: Consuming Model

| | SQS | Kinesis |
|---|---|---|
| **Model** | Competing consumers (cạnh tranh) | Per-shard consumer (một consumer mỗi shard) |
| **Ordering** | FIFO Queue: có; Standard: không | Có (trong cùng shard) |
| **Multiple consumers** | Một message chỉ đến một consumer | Nhiều consumer groups nhận cùng data |
| **Throughput** | Không giới hạn | 1MB/s per shard write, 2MB/s per shard read |
| **Scale** | Thêm consumer instances | Thêm shard |
| **Replay** | Không | Có (1–365 ngày) |
| **Use case** | Task queuing, job processing | Streaming analytics, multi-consumer |

---

## Ví Dụ Code Thực Tế

### Consumer Với Priority Queue (Hàng Đợi Ưu Tiên)

```python
# Nhiều queue với priority khác nhau
PRIORITY_QUEUES = {
    'HIGH': 'high-priority-orders-queue',
    'NORMAL': 'normal-orders-queue',
    'LOW': 'low-priority-orders-queue'
}

def priority_consumer():
    """
    Luôn ưu tiên xử lý queue cao trước.
    Nếu queue cao trống → xử lý queue thấp hơn.
    """
    while True:
        processed = False
        
        for priority in ['HIGH', 'NORMAL', 'LOW']:
            queue_url = PRIORITY_QUEUES[priority]
            
            response = sqs.receive_message(
                QueueUrl=queue_url,
                MaxNumberOfMessages=1,
                WaitTimeSeconds=1  # Short wait
            )
            
            if response.get('Messages'):
                msg = response['Messages'][0]
                process_order(json.loads(msg['Body']))
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=msg['ReceiptHandle']
                )
                processed = True
                break  # Sau khi xử lý → quay lại HIGH priority
        
        if not processed:
            time.sleep(1)  # Tất cả queue trống → đợi 1 giây
```

### Batch Processing Với SQS

```python
def batch_consumer():
    """Xử lý nhiều messages cùng lúc để tối ưu throughput."""
    while True:
        # Nhận tối đa 10 messages
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20
        )
        
        messages = response.get('Messages', [])
        if not messages:
            continue
        
        # Xử lý song song trong thread pool
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = {
                executor.submit(process_order, json.loads(msg['Body'])): msg
                for msg in messages
            }
            
            successful_receipts = []
            failed_receipts = []
            
            for future, msg in futures.items():
                try:
                    future.result()
                    successful_receipts.append(msg['ReceiptHandle'])
                except Exception:
                    failed_receipts.append(msg['ReceiptHandle'])
        
        # Batch delete (xóa theo lô) các messages thành công
        if successful_receipts:
            sqs.delete_message_batch(
                QueueUrl=QUEUE_URL,
                Entries=[
                    {'Id': str(i), 'ReceiptHandle': rh}
                    for i, rh in enumerate(successful_receipts)
                ]
            )
        
        print(f"Processed: {len(successful_receipts)}, Failed: {len(failed_receipts)}")
```

---

## Anti-Patterns Cần Tránh

### 1. Visibility Timeout Quá Ngắn

```
❌ Vấn đề:
  Processing time: 25 giây
  Visibility timeout: 30 giây
  → Consumer #1 đang xử lý
  → Sau 30 giây, message reappears
  → Consumer #2 nhận và cũng bắt đầu xử lý
  → Xử lý trùng lặp!

✅ Giải pháp:
  Visibility timeout = Processing time × 3 (buffer)
  Hoặc extend visibility timeout trong khi xử lý
  Lambda: visibilityTimeout = 6 × Lambda function timeout
```

### 2. Không Xử Lý Poison Message

```
❌ Vấn đề:
  Message có format sai (bad JSON, null fields)
  → Consumer lỗi → message reappears → Consumer lỗi lại → vòng lặp mãi
  
✅ Giải pháp:
  - Cấu hình DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) với maxReceiveCount = 3–5
  - Sau N lần nhận, SQS tự động chuyển sang DLQ
  - Monitor DLQ, alert khi có messages
  - Xử lý DLQ: manual investigation hoặc automated parsing fix
```

### 3. Consumer Không Idempotent

```
❌ Vấn đề:
  Consumer #1 xử lý MSG-1 → crash trước khi delete
  MSG-1 reappears → Consumer #2 xử lý lại → side effects nhân đôi

✅ Giải pháp: (xem file 3-idempotency.md)
  - Dùng MessageId làm idempotency key
  - Kiểm tra DynamoDB trước khi xử lý
  - Lambda Powertools @idempotent decorator
```

### 4. Xử Lý Quá Nhiều Messages Cùng Lúc

```
❌ Vấn đề:
  Lambda concurrency = 1000 (không giới hạn)
  Downstream database chỉ chịu được 100 connections đồng thời
  → Lambda scale lên 1000 → database bị overwhelm (quá tải)

✅ Giải pháp:
  - Lambda Reserved Concurrency = 50 (khớp với database limit)
  - RDS Proxy để connection pooling (gộp kết nối)
  - Xem xét batch size phù hợp
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Competing Consumers Pattern là gì và giải quyết vấn đề gì?

**Trả lời:**
> Competing Consumers là pattern chạy nhiều consumer instances cùng xử lý một queue. Khi một consumer nhận message, message bị ẩn (visibility timeout), các consumer khác không nhận được cùng message đó. Sau khi xử lý xong, consumer delete message.
>
> Pattern này giải quyết vấn đề queue backlog (ứ đọng): khi tốc độ produce nhanh hơn consume, thêm consumer instances để tăng throughput. Vì Lambda tự động scale dựa trên queue depth, đây thực sự là cách phổ biến nhất trên AWS.

### Câu 2: Tính Visibility Timeout phù hợp như thế nào?

**Trả lời:**
> Visibility Timeout phải đủ dài để consumer xử lý xong message. Nếu quá ngắn: message reappears trước khi consumer xong → xử lý trùng. Nếu quá dài: khi consumer crash, mất nhiều thời gian trước khi message được retry.
>
> Rule of thumb: Visibility Timeout = 3× thời gian xử lý trung bình. Với Lambda: visibility timeout phải ≥ 6× Lambda function timeout. Nếu processing time không ổn định, có thể extend visibility timeout trong khi xử lý bằng `change_message_visibility`.

### Câu 3: Khác biệt giữa Competing Consumers với SQS và Kinesis?

**Trả lời:**
> SQS Competing Consumers: nhiều consumer cùng "cạnh tranh" nhận từ một queue, mỗi message chỉ được xử lý bởi MỘT consumer. Không có ordering trong SQS Standard. Scale bằng thêm consumer instances.
>
> Kinesis: mỗi shard được đọc bởi một consumer tại một thời điểm (per shard consumer), đảm bảo ordering trong shard. Với Enhanced Fan-Out, nhiều consumer groups có thể đọc cùng shard độc lập — mỗi nhóm nhận toàn bộ data. Scale bằng thêm shard.
>
> Chọn SQS khi: cần competing consumers đơn giản, ordering không quan trọng.  
> Chọn Kinesis khi: cần ordering, cần replay, hoặc nhiều consumer groups độc lập.

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** competing-consumers, SQS, Kinesis, autoscaling, visibility-timeout, batch-processing, Lambda
