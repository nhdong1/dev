# Circuit Breaker — Cầu Dao Ngắt Lỗi Dây Chuyền

> Circuit Breaker (Cầu Dao) ngăn hệ thống tiếp tục gọi đến một service đang bị lỗi hoặc chậm, tránh cascade failure (lỗi dây chuyền) lan rộng toàn hệ thống.

---

## 📚 Mục Lục

1. [Vấn Đề Cascade Failure](#vấn-đề-cascade-failure)
2. [Circuit Breaker Là Gì?](#circuit-breaker-là-gì)
3. [Ba Trạng Thái Của Circuit Breaker](#ba-trạng-thái-của-circuit-breaker)
4. [Circuit Breaker vs Retry](#circuit-breaker-vs-retry)
5. [Bulkhead Pattern — Vách Ngăn Cách Ly](#bulkhead-pattern--vách-ngăn-cách-ly)
6. [Triển Khai Trên AWS](#triển-khai-trên-aws)
7. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
8. [Metrics Và Monitoring](#metrics-và-monitoring)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Cascade Failure

### Kịch Bản Không Có Circuit Breaker

```
Hệ thống bình thường:
Order Service → Payment Service → Bank API
     ↓                ↓                ↓
   100ms            200ms             300ms

Bank API đột ngột bị chậm (30 giây/request):
- Payment Service gọi Bank API → chờ 30s → timeout
- Mỗi request Order → Payment Service tạo ra 1 thread chờ
- Payment Service hết thread pool (thread pool exhaustion — cạn kiệt luồng xử lý)
- Payment Service trả về 503 (Service Unavailable)
- Order Service retry → gọi thêm nhiều lần → tệ hơn
- Order Service cũng hết thread pool
- API Gateway timeout
- Toàn hệ thống sập ❌
```

```
Cascade Failure (Lỗi Dây Chuyền):

Bank API chậm
     ↓
Payment Service bị block
     ↓
Order Service bị block
     ↓
User Service bị block (vì cần check payment)
     ↓
Toàn bộ hệ thống không khả dụng
```

### Tại Sao Xảy Ra

```
Nguyên nhân:
1. Synchronous calls (Gọi Đồng Bộ): caller block và chờ response
2. Shared resources (Tài Nguyên Chia Sẻ): thread pool, connection pool
3. No timeout (Không Có Timeout): request chờ mãi mãi
4. Retry storms (Bão Thử Lại): nhiều client retry cùng lúc → tăng tải lên service đang yếu
```

---

## Circuit Breaker Là Gì?

Lấy ý tưởng từ circuit breaker (cầu dao) điện:
- Khi có quá tải → cầu dao tự ngắt
- Bảo vệ circuit (mạch điện) khỏi hỏng hóc
- Sau khi sửa → reset cầu dao để thử lại

```
Service A  →  [Circuit Breaker]  →  Service B

Khi Circuit Breaker CLOSED (Đóng — bình thường):
  A → [---] → B  (request qua được)

Khi Circuit Breaker OPEN (Mở — đang lỗi):
  A → [CB: OPEN] ✗  (fail fast — từ chối ngay, không gọi B)
       ↓
    Fallback response (phản hồi dự phòng)

Lợi ích:
- A không lãng phí tài nguyên chờ B
- B được nghỉ để tự hồi phục
- Hệ thống fail fast thay vì fail slow
```

---

## Ba Trạng Thái Của Circuit Breaker

### CLOSED — Đóng (Bình Thường)

```
Trạng thái:  CLOSED
Hành vi:     Cho tất cả request qua
Monitoring:  Theo dõi failure rate (tỷ lệ thất bại)

CLOSED ──────────────────────────────────►
  │                                      │
  │  failure_rate > threshold             │
  │  (tỷ lệ lỗi vượt ngưỡng)             │
  ▼                                      │
OPEN                                     │
```

**Ví dụ ngưỡng:**
- 5 lần thất bại trong 10 giây → mở cầu dao
- Hoặc failure rate > 50% trong 1 phút

### OPEN — Mở (Đang Lỗi)

```
Trạng thái:  OPEN
Hành vi:     Từ chối TẤT CẢ request — không gọi downstream
             Trả về fallback ngay lập tức (fail fast)
Thời gian:   Sau timeout (ví dụ: 60 giây) → chuyển sang HALF-OPEN

OPEN ─────────────────────────────────────►
  │                                       │
  │  timeout elapsed (hết thời gian chờ) │
  │                                       │
  ▼                                       │
HALF-OPEN                                │
```

**Trong OPEN state, trả về fallback:**
```
- Cached response (phản hồi đã lưu)
- Default value ("Service temporarily unavailable")
- Queue request for later processing (xếp hàng để xử lý sau)
- Redirect to backup service (chuyển hướng đến service dự phòng)
```

### HALF-OPEN — Nửa Mở (Đang Thử Phục Hồi)

```
Trạng thái:  HALF-OPEN
Hành vi:     Cho qua MỘT SỐ request thử nghiệm
             Nếu thành công → CLOSED
             Nếu thất bại → OPEN lại

HALF-OPEN ──────────────────────────────────►
  │  success                │  failure
  ▼                         ▼
CLOSED                    OPEN
```

### Biểu Đồ Chuyển Trạng Thái

```
                failure_rate > threshold
CLOSED ──────────────────────────────────► OPEN
  ▲                                         │
  │                                         │ after_timeout
  │                                         ▼
  │  probe success               HALF-OPEN ──► OPEN (probe fail)
  └─────────────────────────────────────────────
```

---

## Circuit Breaker vs Retry

Circuit Breaker và Retry bổ sung cho nhau, không thay thế nhau:

| | Retry (Thử Lại) | Circuit Breaker (Cầu Dao) |
|---|---|---|
| **Mục đích** | Xử lý lỗi tạm thời (transient) | Ngăn cascade failure |
| **Khi nào** | Lỗi ngẫu nhiên (network glitch) | Service bị lỗi kéo dài |
| **Hành vi** | Gọi lại nhiều lần | Dừng gọi ngay lập tức |
| **Ảnh hưởng** | Tăng tải lên downstream | Giảm tải lên downstream |

### Kết Hợp Retry + Circuit Breaker

```
Request → Circuit Breaker → Retry với Exponential Backoff → Service

OPEN: Không retry, fail fast ngay
CLOSED: Retry với backoff khi lỗi transient
HALF-OPEN: Retry tối đa 1 lần thử nghiệm
```

### Exponential Backoff (Tăng Dần Thời Gian Chờ)

```python
def retry_with_backoff(func, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)  # Jitter
            time.sleep(delay)
            # delays: ~1s, ~2s, ~4s
```

---

## Bulkhead Pattern — Vách Ngăn Cách Ly

**Bulkhead** (Vách Ngăn) — lấy ý tưởng từ vách ngăn tàu thủy: nếu một khoang bị thủng, nước không tràn sang khoang khác.

### Bulkhead Với Thread Pool Isolation (Cách Ly Luồng Xử Lý)

```
Không có Bulkhead:
┌──────────────────────────────────┐
│  Shared Thread Pool (100 threads)│
│  Payment calls: 80 threads (busy)│
│  Inventory calls: 20 threads     │
│  → Payment Service chậm → chiếm  │
│    hết 100 threads → Inventory   │
│    cũng không xử lý được         │
└──────────────────────────────────┘

Có Bulkhead:
┌──────────────────────────────────┐
│  Payment Thread Pool: 50 threads │
│  → Payment chậm → chỉ ảnh hưởng │
│    50 threads Payment            │
├──────────────────────────────────┤
│  Inventory Thread Pool: 30 threads│
│  → Inventory không bị ảnh hưởng  │
├──────────────────────────────────┤
│  Other Thread Pool: 20 threads   │
└──────────────────────────────────┘
```

### Bulkhead Với Lambda

```python
# Tách riêng Lambda function cho từng downstream dependency
# Không dùng chung Lambda để tránh resource contention (tranh giành tài nguyên)

Lambda: order-processor
  → Gọi payment-service-client (có circuit breaker)
  → Gọi inventory-service-client (có circuit breaker)
  → Mỗi client có timeout riêng, circuit breaker riêng
  
# Không chia sẻ connection pool giữa hai client
```

---

## Triển Khai Trên AWS

### Option 1: AWS Lambda + DynamoDB (Lưu Circuit State)

```python
import boto3
import json
from datetime import datetime, timedelta
from enum import Enum

class CircuitState(Enum):
    CLOSED = 'CLOSED'
    OPEN = 'OPEN'
    HALF_OPEN = 'HALF_OPEN'

dynamodb = boto3.resource('dynamodb')
state_table = dynamodb.Table('circuit-breaker-states')

class DynamoDBCircuitBreaker:
    def __init__(self, name: str, failure_threshold=5, 
                 timeout_seconds=60, success_threshold=2):
        self.name = name
        self.failure_threshold = failure_threshold
        self.timeout_seconds = timeout_seconds
        self.success_threshold = success_threshold

    def get_state(self) -> dict:
        response = state_table.get_item(Key={'circuit_name': self.name})
        if 'Item' not in response:
            return {
                'state': CircuitState.CLOSED.value,
                'failure_count': 0,
                'success_count': 0,
                'last_failure_time': None
            }
        return response['Item']

    def call(self, func, fallback=None):
        state = self.get_state()

        if state['state'] == CircuitState.OPEN.value:
            # Kiểm tra timeout
            last_failure = datetime.fromisoformat(state['last_failure_time'])
            if datetime.utcnow() - last_failure < timedelta(seconds=self.timeout_seconds):
                # Vẫn còn trong OPEN period → fail fast
                if fallback:
                    return fallback()
                raise CircuitOpenException(f"Circuit {self.name} is OPEN")
            else:
                # Timeout đã qua → chuyển HALF-OPEN
                self._set_state(CircuitState.HALF_OPEN)
                state['state'] = CircuitState.HALF_OPEN.value

        try:
            result = func()
            self._on_success(state)
            return result
        except Exception as e:
            self._on_failure(state)
            if fallback:
                return fallback()
            raise

    def _on_success(self, state):
        if state['state'] == CircuitState.HALF_OPEN.value:
            new_success = state.get('success_count', 0) + 1
            if new_success >= self.success_threshold:
                self._set_state(CircuitState.CLOSED, reset=True)
            else:
                state_table.update_item(
                    Key={'circuit_name': self.name},
                    UpdateExpression='SET success_count = :sc',
                    ExpressionAttributeValues={':sc': new_success}
                )

    def _on_failure(self, state):
        new_count = state.get('failure_count', 0) + 1
        now = datetime.utcnow().isoformat()

        if new_count >= self.failure_threshold:
            self._set_state(CircuitState.OPEN, failure_count=new_count, 
                          last_failure_time=now)
        else:
            state_table.update_item(
                Key={'circuit_name': self.name},
                UpdateExpression='SET failure_count = :fc, last_failure_time = :t',
                ExpressionAttributeValues={':fc': new_count, ':t': now}
            )

    def _set_state(self, new_state: CircuitState, reset=False, **kwargs):
        item = {
            'circuit_name': self.name,
            'state': new_state.value,
            'failure_count': 0 if reset else kwargs.get('failure_count', 0),
            'success_count': 0,
            'last_failure_time': kwargs.get('last_failure_time', ''),
            'updated_at': datetime.utcnow().isoformat()
        }
        state_table.put_item(Item=item)
```

### Option 2: API Gateway + Lambda Authorizer

```
API Gateway Circuit Breaker:
- API Gateway có built-in throttling (giới hạn tốc độ) 
- 429 Too Many Requests khi vượt quá rate limit
- Integration timeout (60s mặc định)

Kết hợp với:
- Lambda Authorizer: custom circuit breaker logic
- Step Functions: retry + circuit breaker cho workflow
```

### Option 3: AWS App Mesh + Envoy

```
App Mesh (Lưới Dịch Vụ) với Envoy proxy:
- Circuit breaker ở infrastructure level (tầng hạ tầng)
- Không cần code trong service
- Cấu hình qua AWS Console hoặc Terraform

VirtualRouter:
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 30s
    baseEjectionDuration: 30s
    maxEjectionPercent: 100
```

---

## Ví Dụ Code Thực Tế

### Dùng Circuit Breaker

```python
# Khởi tạo circuit breakers cho từng downstream service
payment_cb = DynamoDBCircuitBreaker(
    name='payment-service',
    failure_threshold=5,
    timeout_seconds=30
)

inventory_cb = DynamoDBCircuitBreaker(
    name='inventory-service',
    failure_threshold=3,
    timeout_seconds=60
)

def process_order(order_data: dict):
    # Gọi inventory với circuit breaker
    inventory_result = inventory_cb.call(
        func=lambda: inventory_service.reserve(order_data['items']),
        fallback=lambda: {'status': 'QUEUED', 'message': 'Will process when service recovers'}
    )

    if inventory_result['status'] == 'QUEUED':
        # Đưa vào retry queue (hàng đợi thử lại)
        sqs.send_message(
            QueueUrl=RETRY_QUEUE_URL,
            MessageBody=json.dumps(order_data),
            DelaySeconds=30
        )
        return {'status': 'ACCEPTED', 'note': 'Processing delayed'}

    # Gọi payment với circuit breaker
    try:
        payment_result = payment_cb.call(
            func=lambda: payment_service.charge(order_data['payment']),
            fallback=None  # Payment không có fallback — phải thành công
        )
    except CircuitOpenException:
        # Payment circuit open → từ chối order ngay
        return {'status': 'REJECTED', 'reason': 'Payment service unavailable'}

    return {'status': 'SUCCESS', 'order_id': order_data['order_id']}
```

### Circuit Breaker Đơn Giản Với Decorator

```python
import functools
import time
from collections import deque

class SimpleCircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = deque(maxlen=failure_threshold)
        self.state = 'CLOSED'
        self.last_failure_time = None

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return self._call(func, args, kwargs)
        return wrapper

    def _call(self, func, args, kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception(f"Circuit OPEN for {func.__name__}")

        try:
            result = func(*args, **kwargs)
            if self.state == 'HALF_OPEN':
                self.state = 'CLOSED'
                self.failures.clear()
            return result
        except Exception as e:
            self.failures.append(time.time())
            self.last_failure_time = time.time()
            if len(self.failures) >= self.failure_threshold:
                self.state = 'OPEN'
            raise

# Sử dụng như decorator
payment_cb = SimpleCircuitBreaker(failure_threshold=5, recovery_timeout=30)

@payment_cb
def call_payment_api(order_id, amount):
    return requests.post(PAYMENT_API, json={'order_id': order_id, 'amount': amount})
```

---

## Metrics Và Monitoring

### CloudWatch Metrics Quan Trọng

```python
def record_circuit_breaker_metrics(circuit_name: str, state: str, 
                                    success: bool, response_time_ms: float):
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_data(
        Namespace='CircuitBreaker',
        MetricData=[
            {
                'MetricName': 'CircuitState',
                'Dimensions': [{'Name': 'CircuitName', 'Value': circuit_name}],
                'Value': 1 if state == 'OPEN' else 0,
                'Unit': 'Count'
            },
            {
                'MetricName': 'RequestSuccess',
                'Dimensions': [{'Name': 'CircuitName', 'Value': circuit_name}],
                'Value': 1 if success else 0,
                'Unit': 'Count'
            },
            {
                'MetricName': 'ResponseTime',
                'Dimensions': [{'Name': 'CircuitName', 'Value': circuit_name}],
                'Value': response_time_ms,
                'Unit': 'Milliseconds'
            }
        ]
    )
```

### CloudWatch Alarms

```
Cảnh báo khi circuit OPEN:
  Metric: CircuitBreaker/CircuitState
  Condition: Sum > 0 trong 1 phút
  Action: SNS → PagerDuty / Email / Slack

Cảnh báo failure rate cao (trước khi circuit mở):
  Metric: CircuitBreaker/RequestSuccess
  Condition: Sum (fail) / Sum (total) > 20% trong 5 phút
  Action: Warning notification
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Circuit Breaker giải quyết vấn đề gì?

**Trả lời:**
> Circuit Breaker ngăn cascade failure (lỗi dây chuyền). Khi một downstream service bị chậm hoặc lỗi, nếu không có circuit breaker, caller sẽ block threads chờ response, dẫn đến thread pool exhaustion, cascade sang các service khác.
>
> Circuit breaker "ngắt mạch" khi phát hiện nhiều lỗi liên tiếp: thay vì gọi downstream và chờ timeout 30 giây, nó fail fast ngay lập tức — giảm tải lên service đang bị lỗi và trả response nhanh cho caller.

### Câu 2: Ba trạng thái của Circuit Breaker là gì?

**Trả lời:**
> - **CLOSED (Đóng):** Bình thường, request qua được. Đếm số lần lỗi. Nếu vượt ngưỡng → OPEN.
> - **OPEN (Mở):** Đang lỗi. Từ chối tất cả request, trả fallback ngay lập tức. Sau một khoảng thời gian (timeout) → HALF-OPEN để thử.
> - **HALF-OPEN (Nửa Mở):** Đang thử phục hồi. Cho qua một số request thử nghiệm. Nếu thành công → CLOSED. Nếu thất bại → OPEN lại.

### Câu 3: Triển khai Circuit Breaker như thế nào trên AWS Lambda?

**Trả lời:**
> Lambda là stateless nên không thể lưu circuit state trong memory (mỗi invocation có thể là instance khác nhau).
>
> Giải pháp: lưu circuit state trong DynamoDB (low latency, shared state). Mỗi Lambda invocation đọc state từ DynamoDB trước khi call downstream. Nếu state là OPEN → fail fast và trả fallback.
>
> Hoặc dùng AWS App Mesh với Envoy proxy — circuit breaker ở infrastructure level, không cần code trong Lambda. Phù hợp hơn khi chạy trên ECS/EKS.

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** circuit-breaker, cascade-failure, bulkhead, resilience, fault-tolerance, Lambda, DynamoDB
