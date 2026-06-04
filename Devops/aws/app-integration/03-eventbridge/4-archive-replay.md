# Archive & Replay — Lưu Trữ và Phát Lại Sự Kiện

> **Archive** (Lưu Trữ) cho phép giữ lại bản sao của sự kiện để phân tích sau. **Replay** (Phát Lại) cho phép gửi lại các sự kiện đã lưu vào event bus — công cụ không thể thiếu để debug, recover (khôi phục) và test trong hệ thống event-driven.

---

## 📚 Mục Lục

1. [Archive — Lưu Trữ Sự Kiện](#1-archive--lưu-trữ-sự-kiện)
2. [Replay — Phát Lại Sự Kiện](#2-replay--phát-lại-sự-kiện)
3. [Use Cases Thực Tế](#3-use-cases-thực-tế)
4. [Pricing và Cost Optimization](#4-pricing-và-cost-optimization)
5. [Giám Sát Archive và Replay](#5-giám-sát-archive-và-replay)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Archive — Lưu Trữ Sự Kiện

### Archive Là Gì?

**Archive** là tính năng lưu bản sao của tất cả (hoặc một số) sự kiện đi qua event bus vào vùng lưu trữ được quản lý bởi AWS.

```
Sự kiện ──▶ Event Bus ──▶ Rules (xử lý bình thường)
                │
                └──▶ Archive (lưu bản sao)
                         │
                         └──▶ Replay (phát lại khi cần)
```

### Tạo Archive

```bash
# Tạo archive lưu tất cả sự kiện trong 30 ngày
aws events create-archive \
  --archive-name "orders-archive-30d" \
  --event-source-arn "arn:aws:events:ap-southeast-1:123456789:event-bus/orders-bus" \
  --retention-days 30 \
  --description "Archive 30 ngày cho orders bus"

# Tạo archive với filter — chỉ lưu sự kiện lỗi
aws events create-archive \
  --archive-name "payment-errors-archive" \
  --event-source-arn "arn:aws:events:ap-southeast-1:123456789:event-bus/payments-bus" \
  --retention-days 90 \
  --event-pattern '{
    "source": ["com.mycompany.payments"],
    "detail": {
      "status": ["FAILED", "ERROR"]
    }
  }'
```

### Cấu Hình Archive Qua Python

```python
import boto3

client = boto3.client('events')

# Tạo archive toàn bộ sự kiện — retention 60 ngày
response = client.create_archive(
    ArchiveName='all-events-60d',
    EventSourceArn='arn:aws:events:ap-southeast-1:123456789:event-bus/orders-bus',
    RetentionDays=60,  # 0 = lưu vĩnh viễn (tốn chi phí!)
    Description='Archive toàn bộ sự kiện 60 ngày'
)

print(f"Archive ARN: {response['ArchiveArn']}")
print(f"State: {response['State']}")

# Update retention period
client.update_archive(
    ArchiveName='all-events-60d',
    RetentionDays=90  # Tăng lên 90 ngày
)

# Xem thông tin archive
archive = client.describe_archive(ArchiveName='all-events-60d')
print(f"Events stored: {archive['EventCount']:,}")
print(f"Size: {archive['SizeBytes'] / 1024 / 1024:.2f} MB")
```

### Các Thông Số Archive

| Thông Số | Giá Trị | Ghi Chú |
|---|---|---|
| **Retention period** | 0 – 2,147,483,647 ngày | 0 = không giới hạn |
| **Archives per bus** | Không giới hạn | Nhưng tốn chi phí |
| **Event filter** | Tùy chọn | Như event pattern |
| **Encryption** | Có — AWS managed key | Hoặc customer managed KMS |

### Retention Policy (Chính Sách Lưu Giữ)

```
retention_days = 0  → Lưu mãi mãi (vĩnh viễn)
retention_days = 7  → Xóa sau 7 ngày
retention_days = 30 → Xóa sau 30 ngày
retention_days = 90 → Xóa sau 90 ngày (phổ biến nhất)

Best practice: Dùng retention ngắn nhất đáp ứng yêu cầu
                Mục đích debug: 7-30 ngày
                Mục đích compliance: 365+ ngày
                Mục đích audit: 2557 ngày (7 năm, theo một số quy định)
```

---

## 2. Replay — Phát Lại Sự Kiện

### Replay Là Gì?

**Replay** cho phép phát lại các sự kiện đã được archive vào event bus — như thể chúng vừa xảy ra lại:

```
Archive ──▶ Replay ──▶ Event Bus ──▶ Rules ──▶ Targets (Lambda, SQS...)
              │
              Có thể filter:
              - Theo thời gian: từ 2026-05-01 đến 2026-05-10
              - Theo event pattern
              - Đến bus khác (để test không ảnh hưởng production)
```

### Tạo Replay

```bash
# Phát lại tất cả sự kiện từ 1/5 đến 10/5/2026
aws events start-replay \
  --replay-name "replay-may-orders" \
  --source-arn "arn:aws:events:ap-southeast-1:123456789:archive/orders-archive-30d" \
  --event-start-time "2026-05-01T00:00:00Z" \
  --event-end-time "2026-05-10T23:59:59Z" \
  --destination '{
    "Arn": "arn:aws:events:ap-southeast-1:123456789:event-bus/orders-bus"
  }'

# Kiểm tra trạng thái replay
aws events describe-replay --replay-name "replay-may-orders"

# Dừng replay đang chạy
aws events cancel-replay --replay-name "replay-may-orders"
```

### Replay Với Python

```python
import boto3
from datetime import datetime, timezone

client = boto3.client('events')

def start_replay(
    archive_name: str,
    replay_name: str,
    start_time: datetime,
    end_time: datetime,
    target_bus_arn: str,
    event_filter: dict = None
):
    """Bắt đầu replay từ archive đến target bus."""

    params = {
        'ReplayName': replay_name,
        'SourceArn': f'arn:aws:events:ap-southeast-1:123456789:archive/{archive_name}',
        'EventStartTime': start_time,
        'EventEndTime': end_time,
        'Destination': {
            'Arn': target_bus_arn,
            'FilterArns': []
        }
    }

    if event_filter:
        # Replay chỉ các sự kiện khớp filter — áp dụng rule filter
        params['Destination']['FilterArns'] = event_filter

    response = client.start_replay(**params)
    return response['ReplayArn'], response['State']


def wait_for_replay(replay_name: str):
    """Chờ replay hoàn thành và báo cáo kết quả."""
    import time

    while True:
        replay = client.describe_replay(ReplayName=replay_name)
        state = replay['State']

        print(f"Replay state: {state}")

        if state in ['COMPLETED', 'CANCELLED', 'FAILED']:
            if state == 'COMPLETED':
                print(f"Replay completed successfully")
                print(f"Events replayed: {replay.get('EventLastReplayedTime')}")
            elif state == 'FAILED':
                print(f"Replay failed: {replay.get('StateReason')}")
            break

        time.sleep(30)  # Kiểm tra mỗi 30 giây


# Ví dụ sử dụng
replay_arn, state = start_replay(
    archive_name='orders-archive-30d',
    replay_name='debug-may-incident',
    start_time=datetime(2026, 5, 15, 10, 0, 0, tzinfo=timezone.utc),
    end_time=datetime(2026, 5, 15, 14, 0, 0, tzinfo=timezone.utc),
    target_bus_arn='arn:aws:events:ap-southeast-1:123456789:event-bus/orders-staging-bus'
)

print(f"Replay started: {replay_arn}, State: {state}")
wait_for_replay('debug-may-incident')
```

### Replay Destination Options (Tùy Chọn Đích Replay)

```
Option 1: Replay về cùng bus (production)
  ✅ Replay thực sự — consumers xử lý lại
  ❌ Nguy hiểm — có thể gây side effects (tác dụng phụ) như charge khách hàng 2 lần

Option 2: Replay về bus khác (staging/test)
  ✅ An toàn — test behavior mà không ảnh hưởng production
  ✅ Có thể dùng rule khác nhau để test code mới

Option 3: Replay về bus cùng account nhưng có rule filter
  ✅ Chỉ routes đến Lambda test, không đến Lambda production
```

---

## 3. Use Cases Thực Tế

### Use Case 1: Bug Fix và Reprocess (Xử Lý Lại)

```
Tình huống:
- Ngày 15/5: Bug trong Lambda khiến 10,000 đơn hàng bị xử lý sai
- Ngày 16/5: Bug đã được fix

Giải pháp với Archive & Replay:
1. Deploy Lambda đã fix
2. Replay sự kiện từ 15/5 vào cùng bus
3. Lambda mới xử lý lại đúng 10,000 đơn hàng
4. Tất cả dữ liệu được khôi phục

Không có Archive: Phải restore database từ backup
              → Tốn nhiều thời gian, mất dữ liệu mới hơn
```

### Use Case 2: Thêm Consumer Mới

```
Tình huống:
- Dịch vụ Analytics mới cần xử lý tất cả sự kiện đã xảy ra trong 30 ngày qua
- Không thể yêu cầu tất cả producers gửi lại

Giải pháp:
1. Triển khai Analytics service
2. Thêm rule cho Analytics Lambda
3. Replay 30 ngày sự kiện từ archive
4. Analytics service được backfill (điền đầy dữ liệu cũ)
```

### Use Case 3: Disaster Recovery (Khôi Phục Thảm Họa)

```
Tình huống:
- Database của Consumer service bị mất dữ liệu do lỗi migration
- Cần rebuild state từ đầu

Giải pháp với Event Sourcing + Archive:
1. Reset database về trạng thái ban đầu
2. Replay toàn bộ sự kiện từ archive
3. Database được rebuild từ chuỗi sự kiện
4. Dữ liệu được khôi phục hoàn toàn
```

### Use Case 4: Testing (Kiểm Thử)

```
Tình huống:
- Cần test Lambda mới với dữ liệu production thực tế
- Không muốn ảnh hưởng đến system production

Giải pháp:
1. Tạo staging event bus
2. Replay sự kiện production vào staging bus
3. Lambda staging xử lý sự kiện thực tế
4. So sánh kết quả với production
```

### Use Case 5: Compliance Audit (Kiểm Toán Tuân Thủ)

```
Tình huống:
- Cần chứng minh hệ thống đã xử lý đúng tất cả sự kiện trong quý

Giải pháp với Archive:
- Archive toàn bộ sự kiện 365+ ngày
- Truy xuất lịch sử đầy đủ khi cần
- Replay để verify (xác minh) behavior của hệ thống
```

---

## 4. Pricing và Cost Optimization

### Chi Phí Archive

```
Lưu trữ: $0.023 / GB / tháng
  → 1 GB archive: $0.023/tháng ~ 276 VND/tháng

Replay: $1.00 / triệu events phát lại
  → 1 triệu events: $1.00 ~ 25,000 VND
```

### Ước Tính Chi Phí

```python
def estimate_archive_cost(events_per_day: int, avg_event_size_kb: float, retention_days: int):
    """Ước tính chi phí archive EventBridge."""
    total_events = events_per_day * retention_days
    total_size_gb = (total_events * avg_event_size_kb) / (1024 * 1024)

    # Chi phí trung bình vì events được thêm dần
    avg_size_gb = total_size_gb / 2

    monthly_cost_usd = avg_size_gb * 0.023
    monthly_cost_vnd = monthly_cost_usd * 25000

    print(f"Events lưu trữ: {total_events:,}")
    print(f"Tổng dung lượng: {total_size_gb:.2f} GB")
    print(f"Chi phí archive: ${monthly_cost_usd:.2f}/tháng ({monthly_cost_vnd:,.0f} VND)")


# Ví dụ: 10,000 events/ngày, 1KB mỗi event, lưu 30 ngày
estimate_archive_cost(10000, 1.0, 30)
# → ~0.15 GB trung bình, ~$0.003/tháng — rất rẻ!
```

### Cost Optimization Tips

```
1. Đặt retention ngắn nhất đủ dùng:
   Debug: 7-14 ngày
   Operational: 30-90 ngày
   Compliance: theo quy định (365+ ngày)

2. Dùng event filter để chỉ archive sự kiện quan trọng:
   - Archive errors và warnings
   - Bỏ qua health check events

3. Compress data trước khi gửi vào EventBridge (detail field)

4. Dùng S3 Glacier cho long-term compliance archive
   (Export events ra S3 định kỳ, rẻ hơn EventBridge archive)
```

---

## 5. Giám Sát Archive và Replay

### CloudWatch Metrics Quan Trọng

```bash
# Số sự kiện được archive
aws cloudwatch get-metric-statistics \
  --namespace "AWS/Events" \
  --metric-name "ArchiveMatchedEventCount" \
  --dimensions Name=ArchiveName,Value=orders-archive-30d \
  --start-time "2026-05-18T00:00:00Z" \
  --end-time "2026-05-18T23:59:59Z" \
  --period 3600 \
  --statistics Sum
```

| Metric | Ý Nghĩa | Alarm Khi |
|---|---|---|
| `ArchiveMatchedEventCount` | Số sự kiện được archive | Đột ngột giảm về 0 |
| `ArchiveDroppedEventCount` | Sự kiện bị loại bỏ | > 0 (có vấn đề) |
| `ReplayMatchedEventCount` | Số sự kiện trong replay | Dùng để track tiến độ |
| `ReplayThrottledCount` | Replay bị throttle | > 0 |

### CloudWatch Alarm Cho Archive

```python
import boto3

cw = boto3.client('cloudwatch')

# Alarm khi không có sự kiện nào được archive trong 1 giờ
cw.put_metric_alarm(
    AlarmName='archive-no-events-1h',
    MetricName='ArchiveMatchedEventCount',
    Namespace='AWS/Events',
    Dimensions=[
        {'Name': 'ArchiveName', 'Value': 'orders-archive-30d'}
    ],
    Period=3600,
    EvaluationPeriods=1,
    Threshold=1,
    ComparisonOperator='LessThanThreshold',
    Statistic='Sum',
    AlarmDescription='Cảnh báo: Archive không nhận sự kiện trong 1 giờ',
    AlarmActions=['arn:aws:sns:...:ops-alerts']
)
```

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Archive và Replay giải quyết vấn đề gì mà SNS/SQS không có?**

> SNS không lưu tin nhắn sau khi deliver; SQS giữ message nhưng chỉ trong retention period và không có cơ chế replay có chọn lọc. EventBridge Archive cho phép: (1) lưu toàn bộ event history, (2) replay theo khoảng thời gian cụ thể, (3) replay đến bus khác để test. Điều này cực kỳ quan trọng cho: bug fix (reprocess), onboarding consumer mới (backfill), disaster recovery, và compliance audit.

**Q: Khi replay sự kiện về production bus, làm sao tránh duplicate side effects?**

> Đây là vấn đề idempotency (tính bất biến). Các biện pháp: (1) Consumer phải idempotent — kiểm tra idempotency key trước khi xử lý, (2) Replay về staging bus thay vì production, (3) Dùng DynamoDB conditional write với event ID để tránh xử lý lại, (4) Thêm metadata vào replay event để consumer phân biệt original vs replayed. EventBridge không tự xử lý vấn đề này — trách nhiệm của consumer.

**Q: Nên đặt retention period bao lâu?**

> Phụ thuộc vào mục đích: Debug/operational incidents: 7–30 ngày đủ để điều tra; Recovery SLA: phải dài hơn RTO (Recovery Time Objective) của hệ thống; Compliance/audit: theo quy định ngành (GDPR 6 năm, PCI-DSS 1 năm, một số quy định ngân hàng Việt Nam yêu cầu 5–10 năm). Trade-off: Retention dài hơn → tốn chi phí lưu trữ hơn.

---

**Liên Kết:** [3-schema-registry.md](3-schema-registry.md) | [5-eventbridge-pipes.md](5-eventbridge-pipes.md)
