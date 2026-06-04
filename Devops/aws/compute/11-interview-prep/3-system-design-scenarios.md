# 🏗️ 5 Kịch Bản Thiết Kế Hệ Thống AWS Compute Thực Tế

> 5 bài toán system design (thiết kế hệ thống) phổ biến trong phỏng vấn, với kiến trúc đề xuất chi tiết, decision rationale (lý do ra quyết định), và trade-offs (đánh đổi). Mỗi kịch bản đi kèm framework tư duy để bạn áp dụng tương tự.

## 📚 Framework Thiết Kế Hệ Thống

```
1. CLARIFY    — Làm rõ yêu cầu (5 phút)
2. ESTIMATE   — Ước lượng quy mô (3 phút)
3. DESIGN     — Kiến trúc tổng thể (10 phút)
4. DEEP DIVE  — Chi tiết các component quan trọng (10 phút)
5. TRADE-OFFS — Thảo luận đánh đổi và cải tiến (5 phút)
```

---

## Kịch Bản 1: Thiết Kế Hệ Thống Video Streaming (YouTube-like)

### Yêu Cầu

- **Functional:** Upload video, transcode, stream to users globally
- **Non-functional:** 10M DAU (Daily Active Users — Người Dùng Hoạt Động Hàng Ngày), 99.9% uptime, latency < 3 giây khi bắt đầu stream

### Ước Lượng Quy Mô

```
Upload: 10,000 videos/giờ, trung bình 500MB mỗi video → 5TB/giờ storage mới
Stream: 10M users × 4 giờ/ngày × 5MB/phút → 12PB data transfer/ngày
Transcode: mỗi video cần 5 phiên bản (360p, 480p, 720p, 1080p, 4K) → 5× CPU
```

### Kiến Trúc Đề Xuất

```
[User Upload]
    ↓
API Gateway + Lambda (Generate presigned S3 URL — URL Đã Ký Sẵn)
    ↓
S3 Raw Video Bucket (Bucket Video Thô)
    ↓ (S3 Event trigger)
SQS Queue (Hàng Đợi) → [Transcode Workers]
                         ECS Fargate Tasks với FFmpeg
                         (scale tự động theo queue depth)
                         ↓
                    S3 Transcoded Videos (Bucket Video Đã Mã Hóa)
                         ↓
                    CloudFront CDN Distribution
                    (Edge locations toàn cầu)
                         ↓
                    [User Stream]

[Metadata]
API Gateway → ECS (API Service) → RDS PostgreSQL (video metadata)
                                → ElastiCache Redis (popular video cache)
                                → DynamoDB (view counts, likes — high write)
```

### Chi Tiết Các Component Quan Trọng

**Transcode Workers (Nhân Công Mã Hóa Video):**
- Chọn **ECS Fargate với Spot** — transcode là CPU-intensive, stateless, fault-tolerant
- Spot vì nếu task bị interrupt, chỉ mất 1 video đang transcode (có thể retry)
- Fargate vì không muốn manage EC2 fleet cho batch workload
- Scale: SQS queue depth → Application Auto Scaling điều chỉnh task count

**Storage Strategy (Chiến Lược Lưu Trữ):**
```
S3 Standard       → Video mới (< 30 ngày)
S3 Standard-IA    → Video ít xem (30-365 ngày)
S3 Glacier        → Archive video cũ (> 1 năm)
S3 Lifecycle Policy → Tự động chuyển tầng
```

**CDN Strategy (Chiến Lược CDN):**
- CloudFront với 300+ edge locations toàn cầu
- Origin Shield (Khiên Nguồn) để giảm load lên S3
- Signed URLs/Cookies cho nội dung premium

### Trade-offs

| Quyết Định                  | Lý Do Chọn                          | Đánh Đổi                             |
| --------------------------- | ----------------------------------- | ------------------------------------ |
| Fargate cho transcode       | Không manage nodes, auto-scale      | Đắt hơn EC2 Spot cho steady load     |
| SQS cho transcode queue     | Đơn giản, reliable, at-least-once   | Không đảm bảo thứ tự (FIFO queue nếu cần) |
| CloudFront CDN              | Global distribution, thấp latency  | Chi phí data transfer, invalidation cost |
| RDS PostgreSQL              | ACID, complex queries               | Vertical scaling, single writer      |
| DynamoDB cho counters       | High write throughput               | Eventual consistency, phức tạp hơn  |

---

## Kịch Bản 2: Thiết Kế Hệ Thống Ride-sharing (Uber-like)

### Yêu Cầu

- **Functional:** Tìm xe gần nhất, match driver-rider, tracking real-time, pricing
- **Non-functional:** 5M concurrent users, < 1 giây match time, 99.99% uptime cho core matching

### Ước Lượng Quy Mô

```
Location updates: 5M drivers × 1 update/4 giây → 1.25M writes/giây
Match requests: 100,000 concurrent ride requests
Geospatial queries: "Find drivers within 2km" → cần geospatial index
```

### Kiến Trúc Đề Xuất

```
[Driver App]                    [Rider App]
    ↓ WebSocket                     ↓ HTTP/WebSocket
API Gateway + NLB              API Gateway + ALB
(WebSocket cho real-time)      (REST cho requests)
    ↓                               ↓
[Location Service]            [Matching Service]
ECS Fargate + ElastiCache     ECS Fargate
Redis Geo (lưu driver location)  → Query Redis Geo
(1 update/4 giây per driver)     → Find nearby drivers
                                  → Assign driver
    ↓                               ↓
[Notification Service]         [Ride Service]
Lambda + SNS/WebSocket         ECS Fargate + RDS Aurora
(Push to driver/rider)         (Ride lifecycle, pricing)

[Real-time Tracking]
API Gateway WebSocket → Lambda → ElastiCache Pub/Sub → Rider App
```

### Chi Tiết Location Service

**Redis Geo Commands (Lệnh Geo Redis):**
```python
# Driver update location
redis.geoadd("drivers", longitude, latitude, driver_id)

# Find drivers within 2km
nearby_drivers = redis.georadius(
    "drivers",
    rider_longitude, rider_latitude,
    2, "km",
    count=10,
    sort="ASC"  # Sắp xếp theo khoảng cách
)
```

**Tại sao ElastiCache Redis thay vì Database:**
- Redis GEOADD/GEORADIUS operations: O(N+log(N)) — cực kỳ nhanh
- In-memory → microsecond latency
- Millions of writes/giây — database sẽ không chịu được

**ECS Scaling cho Location Service:**
```
Target: CPU utilization 60%
Min tasks: 20 (đảm bảo capacity ngay cả khi traffic thấp)
Max tasks: 200
Scale-out: CPU > 60% trong 1 phút → thêm 20 tasks
Scale-in: CPU < 30% trong 5 phút → bớt 10 tasks
```

### Trade-offs

| Quyết Định                     | Lý Do Chọn                                | Đánh Đổi                                  |
| ------------------------------ | ----------------------------------------- | ----------------------------------------- |
| Redis cho location             | Geospatial native, in-memory latency      | Single point of failure nếu không replica |
| WebSocket cho real-time        | Bidirectional, low overhead               | Stateful connections, harder to scale     |
| NLB cho WebSocket              | TCP passthrough, giữ long-lived connection | Không có L7 routing                      |
| Aurora cho ride data           | Multi-AZ, auto-failover, MySQL compatible | Đắt hơn RDS MySQL                        |
| Lambda cho notifications       | Event-driven, auto-scale, no idle cost   | Cold start latency cho first notification |

---

## Kịch Bản 3: Thiết Kế Hệ Thống E-commerce Payment (Hệ Thống Thanh Toán)

### Yêu Cầu

- **Functional:** Process payment, refund, fraud detection
- **Non-functional:** Exactly-once processing (không duplicate payment), < 2 giây latency, 99.999% uptime (5 nines)

### Ước Lượng Quy Mô

```
Peak: 10,000 transactions/giây (Black Friday)
Normal: 500 transactions/giây
Fraud check: < 100ms per transaction
Data retention: 7 năm (compliance)
```

### Kiến Trúc Đề Xuất

```
[Client App]
    ↓ HTTPS
API Gateway (WAF enabled)
    ↓
[Payment API Service]
ECS Fargate (Blue/Green deploy)
Private Subnet, Multi-AZ
    ↓
[Idempotency Layer]
DynamoDB (idempotency keys với TTL 24 giờ)
    ↓              ↓
[Fraud Service]  [Payment Processor]
Lambda (real-time) ECS Fargate → External Gateway
(ML model inference) (Stripe/Braintree)
    ↓              ↓
[Event Bus]
EventBridge (Payment events: initiated, succeeded, failed, refunded)
    ↓                    ↓                    ↓
[Notification]    [Ledger Service]    [Analytics]
Lambda + SES/SNS  ECS + RDS Aurora    Kinesis → Redshift
```

### Chi Tiết Idempotency Implementation

**Vấn đề cần giải quyết:**
- Client có thể retry nếu không nhận được response
- Network timeout không có nghĩa là payment failed — payment có thể đã thành công
- Cần đảm bảo không charge customer 2 lần

```python
# Idempotency với DynamoDB
def process_payment(idempotency_key, payment_data):
    # Bước 1: Kiểm tra key đã tồn tại
    existing = dynamodb.get_item(
        TableName='idempotency-keys',
        Key={'key': idempotency_key}
    )
    
    if existing.get('Item'):
        # Key đã tồn tại → return kết quả cũ (không charge lần 2)
        return existing['Item']['result']
    
    # Bước 2: Reserve key với trạng thái "in-progress"
    dynamodb.put_item(
        TableName='idempotency-keys',
        Item={
            'key': idempotency_key,
            'status': 'in-progress',
            'ttl': int(time.time()) + 86400  # 24 giờ TTL
        },
        ConditionExpression='attribute_not_exists(#k)',  # Atomic check
        ExpressionAttributeNames={'#k': 'key'}
    )
    
    # Bước 3: Thực hiện payment
    result = call_payment_gateway(payment_data)
    
    # Bước 4: Cập nhật key với kết quả
    dynamodb.update_item(
        TableName='idempotency-keys',
        Key={'key': idempotency_key},
        UpdateExpression='SET #s = :status, #r = :result',
        ExpressionAttributeValues={':status': 'completed', ':result': result}
    )
    
    return result
```

### Deployment Strategy (Chiến Lược Triển Khai)

**Blue/Green Deployment với CodeDeploy:**
```
Tại sao Blue/Green cho payment service:
- Zero downtime deployment
- Instant rollback nếu có issue
- Test new version với percentage traffic trước
- Payment service không được có restart-in-place

Cấu hình:
- Traffic shift: 10% → 30% → 70% → 100% trong 15 phút
- Health check sau mỗi shift
- Auto rollback nếu error rate > 1%
```

### Trade-offs

| Quyết Định                   | Lý Do Chọn                               | Đánh Đổi                                 |
| ---------------------------- | ---------------------------------------- | ---------------------------------------- |
| DynamoDB idempotency         | Single-digit ms latency, TTL built-in   | Chi phí reads/writes, DynamoDB complexity |
| EventBridge cho events       | Decoupled services, audit trail          | Eventual consistency, replay complexity  |
| Aurora Multi-AZ              | ACID, sub-minute failover                | Đắt, sync replication latency            |
| Lambda cho fraud             | Auto-scale đột biến, isolate từ main flow | Cold start có thể ảnh hưởng latency     |
| Blue/Green deployment        | Safe rollback, zero downtime            | Cần 2x infrastructure trong deploy window|

---

## Kịch Bản 4: Thiết Kế Data Pipeline Xử Lý Log Thời Gian Thực

### Yêu Cầu

- **Functional:** Ingest (nhập) 100GB logs/giờ, real-time alerting khi error rate tăng, batch analytics
- **Non-functional:** < 30 giây từ log → alert, 99.9% data durability, query dữ liệu 90 ngày

### Ước Lượng Quy Mô

```
Ingest: 100GB/giờ ≈ 28MB/giây, ~500K events/giây
Real-time processing: latency < 30 giây
Storage: 100GB × 24 × 90 = 216TB cho 90 ngày
Query: Ad-hoc queries trên 90 ngày data
```

### Kiến Trúc Đề Xuất

```
[Application Servers / ECS / Lambda]
    ↓ CloudWatch Agent / Fluent Bit
[Log Aggregation]
Kinesis Data Streams (10 shards, 1MB/s per shard)
    ↓                           ↓
[Real-time Path]          [Batch Path]
Kinesis Data Analytics    Kinesis Firehose
(Apache Flink SQL)         (Buffer 5 phút)
    ↓                           ↓
Lambda (Alert logic)      S3 Data Lake
    ↓                     (Parquet format, partitioned by date)
SNS → PagerDuty               ↓
                          Glue Crawler (hàng ngày)
                              ↓
                          Glue Data Catalog
                              ↓
                          Athena (ad-hoc SQL queries)
                              ↓
                          QuickSight Dashboard
```

### Chi Tiết Real-time Alerting

**Kinesis Data Analytics (Flink SQL):**
```sql
-- Detect error rate > 5% trong 1 phút sliding window
SELECT
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) as window_start,
    service_name,
    COUNT(*) as total_requests,
    SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) as errors,
    ROUND(SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as error_rate
FROM log_stream
GROUP BY TUMBLE(event_time, INTERVAL '1' MINUTE), service_name
HAVING error_rate > 5.0
```

**Lambda Alert Handler:**
```python
def handle_alert(event, context):
    for record in event['Records']:
        alert_data = json.loads(base64.b64decode(record['kinesis']['data']))
        
        # Deduplicate alerts (tránh spam)
        alert_key = f"{alert_data['service_name']}-{alert_data['window_start']}"
        if not is_duplicate_alert(alert_key):
            sns.publish(
                TopicArn=ALERT_TOPIC,
                Subject=f"High Error Rate: {alert_data['service_name']}",
                Message=f"Error rate: {alert_data['error_rate']}% in last minute"
            )
            mark_alert_sent(alert_key)
```

**S3 Data Lake Optimization (Tối Ưu Data Lake):**
```
Partitioning (Phân Vùng):
s3://data-lake/logs/year=2026/month=05/day=15/hour=14/

File format: Parquet (cột, nén tốt cho analytics)
Compression: Snappy (cân bằng tốc độ đọc và tỷ lệ nén)
File size: 128MB+ mỗi file (tránh small file problem)
```

### Trade-offs

| Quyết Định               | Lý Do Chọn                                  | Đánh Đổi                                    |
| ------------------------ | ------------------------------------------- | ------------------------------------------- |
| Kinesis thay vì SQS      | Ordered, replayable, fan-out nhiều consumers | Đắt hơn, phức tạp hơn, shard management    |
| Flink cho real-time      | Windowed aggregations, exactly-once          | Steep learning curve, cost Kinesis Analytics|
| Athena cho queries       | Serverless, pay-per-query, no infrastructure | Không real-time, query time 1-60 giây       |
| Parquet format           | Column-oriented, 10x cheaper Athena queries  | Write-time overhead, không human-readable   |
| Firehose buffering 5 phút | Reduce S3 API calls, larger files           | 5 phút latency cho batch queries             |

---

## Kịch Bản 5: Thiết Kế Serverless API Backend Cho Mobile App

### Yêu Cầu

- **Functional:** User auth, profile CRUD, social feed, notifications push
- **Non-functional:** 1M MAU (Monthly Active Users — Người Dùng Hoạt Động Hàng Tháng), < 500ms API response, chi phí tối thiểu, auto-scale to 10x trong event

### Ước Lượng Quy Mô

```
Normal: 500 RPS
Peak event: 5,000 RPS (10x spike)
Data: 1M users × 5KB profile = 5GB user data
Feed: 1M users × 20 posts/ngày = 20M posts/ngày
```

### Kiến Trúc Đề Xuất

```
[Mobile App (iOS/Android)]
    ↓ HTTPS
Amazon Cognito (User Auth — Xác Thực Người Dùng)
    ↓ JWT token
API Gateway (REST / HTTP API)
    ↓
[Lambda Functions — per endpoint]
┌──────────────────────────────────┐
│ profile-get      → DynamoDB      │
│ profile-update   → DynamoDB      │
│ feed-get         → ElastiCache   │
│ post-create      → DynamoDB      │  
│                  → SQS (async)   │
│ follow-user      → DynamoDB      │
└──────────────────────────────────┘
    ↓ Async via SQS
[Background Workers — Lambda]
┌──────────────────────────────────┐
│ feed-fanout     → DynamoDB       │  (ghi post vào feed của followers)
│ push-notif      → SNS → FCM/APNS │  (Firebase/Apple Push)
│ image-resize    → S3 → Lambda    │  (resize avatar khi upload)
└──────────────────────────────────┘

[Storage]
DynamoDB   → User profiles, posts, follow relationships (high-scale NoSQL)
S3         → Images, videos (media storage)
ElastiCache → Social feed cache (giảm DynamoDB reads)
```

### Chi Tiết DynamoDB Design

**Single-table Design (Thiết Kế Một Bảng) cho Social App:**

```
Table: social-app
PK (Partition Key)   | SK (Sort Key)         | Attributes
---------------------|----------------------|---------------------------
USER#user123         | PROFILE              | name, bio, avatar_url
USER#user123         | FOLLOW#user456       | followed_at
USER#user123         | POST#2026-05-15-001  | content, media_url, likes
FEED#user456         | 2026-05-15T10:00#001 | post_id, author_id (fan-out)

Indexes (Chỉ Mục Phụ):
GSI1: post_id → find all reactions to a post
GSI2: author_id + created_at → paginate user's posts
```

**Fan-out Strategy (Chiến Lược Phát Sóng) cho Social Feed:**
```python
# Khi user A tạo post → ghi vào feed của tất cả followers
def fanout_post(post_id, author_id, content):
    followers = get_followers(author_id)  # Lấy danh sách followers
    
    if len(followers) <= 1000:
        # Push model: ghi vào feed mỗi follower ngay
        with dynamodb.batch_write() as batch:
            for follower_id in followers:
                batch.put_item(Item={
                    'PK': f'FEED#{follower_id}',
                    'SK': f'{timestamp}#{post_id}',
                    'post_id': post_id
                })
    else:
        # Pull model cho celebrity (>1000 followers): không fan-out
        # Feed query sẽ merge từ user's own posts + following posts
        pass
```

**Lambda Concurrency Management:**
```
profile-get:     Reserved = 100 (core feature, protect)
feed-get:        Reserved = 200 (most called endpoint)
post-create:     Reserved = 50  (write operations)
push-notif:      Reserved = 20  (async, less critical)
image-resize:    Reserved = 10  (background, không urgent)

Provisioned Concurrency:
feed-get:        Provisioned = 20 (hot path, loại bỏ cold start)
profile-get:     Provisioned = 10
```

### Chi Phí Ước Tính

```
Lambda:
- 1M users × 10 API calls/ngày × 30 ngày = 300M invocations/tháng
- Average duration 200ms, 512MB memory
- Chi phí: ~60 USD/tháng (rất rẻ!)

API Gateway HTTP API:
- 300M requests × $1/million = 300 USD/tháng

DynamoDB on-demand:
- ~1B reads + 100M writes/tháng ≈ 300 USD/tháng

ElastiCache (1 node r6g.large):
- ~150 USD/tháng

Tổng ước tính: ~850 USD/tháng cho 1M MAU
→ ~$0.00085/user/tháng — cực kỳ cost-effective!
```

### Trade-offs

| Quyết Định                   | Lý Do Chọn                                | Đánh Đổi                                     |
| ---------------------------- | ----------------------------------------- | -------------------------------------------- |
| Lambda thay vì ECS Fargate   | Pay-per-use, zero idle cost, auto-scale  | Cold start, 15 phút timeout, 10GB RAM limit  |
| DynamoDB thay vì RDS         | Serverless, auto-scale, single-digit ms  | No JOIN, eventual consistency, DynamoDB complexity |
| HTTP API thay vì REST API    | 71% rẻ hơn REST API, lower latency       | Ít tính năng hơn (no edge optimized)         |
| Fan-out async với SQS        | Decouple post creation từ feed update    | Lag vài giây trước khi followers thấy post   |
| Cognito cho auth             | Managed, MFA, OAuth integration           | Vendor lock-in, giá tăng theo users           |

---

## 📊 So Sánh Compute Platform Qua 5 Kịch Bản

| Kịch Bản                | Compute Chính           | Lý Do                                          |
| ----------------------- | ----------------------- | ---------------------------------------------- |
| Video Streaming         | ECS Fargate + Spot      | CPU-intensive batch, stateless, fault-tolerant |
| Ride-sharing            | ECS Fargate + NLB       | Long-running services, WebSocket, steady load  |
| Payment System          | ECS Fargate             | Long-running, strict latency, easy rollback    |
| Data Pipeline           | Kinesis + Lambda        | Event-driven, pay-per-event, auto-scale        |
| Mobile API Backend      | Lambda                  | Event-driven, variable load, zero idle cost    |

---

## 💡 Mẹo Thiết Kế Trong Phỏng Vấn

### Trước Khi Bắt Đầu Vẽ

```
1. Hỏi: "Hệ thống cần serve bao nhiêu users?"
2. Hỏi: "Read-heavy hay write-heavy?"
3. Hỏi: "Cần consistency mức nào? ACID hay eventual?"
4. Hỏi: "Budget constraints? Team size?"
5. Hỏi: "Global hay single region?"
```

### Khi Vẽ Kiến Trúc

```
1. Start với high-level: User → API → Database
2. Thêm scaling: Load Balancer, Auto Scaling
3. Thêm reliability: Multi-AZ, failover
4. Thêm performance: Cache, CDN
5. Thêm security: WAF, VPC, IAM
```

### Những Câu Hay Nói Trong System Design

- "Tôi sẽ dùng X vì Y, nhưng có thể thay bằng Z nếu..." (show trade-offs)
- "Đây là kiến trúc MVP, sau đó có thể cải thiện bằng..."
- "Điểm bottleneck (nút cổ chai) tiềm năng là X, tôi sẽ giải quyết bằng..."
- "Nếu budget là ưu tiên, tôi sẽ dùng... Nếu latency quan trọng hơn, tôi sẽ..."

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành — 5 kịch bản thiết kế hệ thống chi tiết
