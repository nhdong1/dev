# λ Lambda Debugging — Gỡ Lỗi Lambda Có Hệ Thống

> Hướng dẫn chẩn đoán và khắc phục các sự cố phổ biến nhất với AWS Lambda: timeout, OOM (Out of Memory — Hết Bộ Nhớ), lỗi quyền truy cập, cold start chậm, và throttling.

## 📚 Mục Lục

1. [Timeout Errors — Hết Thời Gian Thực Thi](#1-timeout-errors)
2. [Out of Memory — OOM](#2-out-of-memory)
3. [Permission Errors — Lỗi Quyền Truy Cập](#3-permission-errors)
4. [Cold Start Latency — Độ Trễ Khởi Động Lạnh](#4-cold-start-latency)
5. [Throttling — Bị Giới Hạn Số Lần Gọi](#5-throttling)
6. [VPC Lambda Issues](#6-vpc-lambda-issues)
7. [Debugging Workflow — Quy Trình Gỡ Lỗi](#7-debugging-workflow)

---

## 1. Timeout Errors

### Nhận Biết Timeout

```
CloudWatch Log message:
  "Task timed out after 30.00 seconds"
  "Status: timeout"

CloudWatch Metric:
  - Duration metric gần đúng bằng timeout limit
  - Errors metric tăng đột ngột
```

### Chẩn Đoán Timeout

```bash
# Xem duration của các invocations gần đây
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --filter-pattern "Task timed out" \
  --start-time $(date -d '1 hour ago' +%s000)

# CloudWatch Insights — phân tích duration
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @type = "REPORT"
    | stats
        avg(@duration) as avgDuration,
        max(@duration) as maxDuration,
        percentile(@duration, 95) as p95Duration,
        count(*) as invocations
      by bin(1h)
    | sort by @timestamp desc'
```

### Nguyên Nhân & Giải Pháp Timeout

#### A. External API Call Chậm

```python
import urllib3
import boto3

# Vấn đề: Không có connection timeout
http = urllib3.PoolManager()
response = http.request('GET', 'https://external-api.com/data')  # Có thể treo mãi

# Giải pháp: Luôn đặt timeout
http = urllib3.PoolManager(timeout=urllib3.Timeout(connect=2.0, read=5.0))
response = http.request('GET', 'https://external-api.com/data')
```

#### B. Database Connection Pool Không Được Quản Lý

```python
# Vấn đề: Mở connection mới mỗi invocation
def handler(event, context):
    conn = psycopg2.connect(...)  # Chậm, tốn thời gian
    # ... xử lý ...
    conn.close()

# Giải pháp: Reuse connection bên ngoài handler (connection pooling)
# Connection tồn tại trong execution environment giữa các invocations
import psycopg2
from psycopg2 import pool

connection_pool = None  # Module-level variable

def get_connection():
    global connection_pool
    if connection_pool is None:
        connection_pool = psycopg2.pool.SimpleConnectionPool(1, 5, ...)
    return connection_pool.getconn()

def handler(event, context):
    conn = get_connection()
    try:
        # ... xử lý ...
    finally:
        connection_pool.putconn(conn)
```

#### C. Lambda Trong VPC Không Truy Cập Được Internet

```
Triệu chứng: Lambda timeout khi gọi external API hoặc AWS services
Nguyên nhân: Lambda trong Private Subnet, không có NAT Gateway

Kiểm tra:
  1. Lambda execution trong Private Subnet không có default route ra internet
  2. Không có NAT Gateway cho subnet đó

Giải pháp:
  A. Thêm NAT Gateway vào VPC (Khuyến nghị cho external APIs)
  B. Dùng VPC Endpoints (Điểm Cuối VPC) cho AWS services:
     - S3 Gateway Endpoint (miễn phí)
     - DynamoDB Gateway Endpoint (miễn phí)
     - Secrets Manager Interface Endpoint
     - SQS Interface Endpoint
```

---

## 2. Out of Memory

### Nhận Biết OOM

```
Log message:
  "Runtime exited with error: signal: killed"
  "Process exited before completing request"

CloudWatch Metrics:
  - max_memory_used_mb tiệm cận memory_size
  - Errors metric tăng
  - Không có "Task timed out" trong logs — process bị kill đột ngột
```

### Phân Tích Memory Usage

```bash
# CloudWatch Insights — xem memory usage theo thời gian
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @type = "REPORT"
    | stats
        avg(@maxMemoryUsed) as avgMemMB,
        max(@maxMemoryUsed) as maxMemMB,
        avg(@memorySize) as configuredMemMB
      by bin(1h)'
```

### Tính Toán Memory Đúng

```
Nguyên tắc: Đặt memory limit cao hơn max_memory_used khoảng 20-30%
  Nếu max_memory_used = 450 MB → Đặt memory_size = 576 MB hoặc 768 MB

Lưu ý quan trọng:
  Lambda tính tiền theo GB-seconds:
  - 512 MB × 1 giây = 0.5 GB-seconds
  - 1024 MB × 0.4 giây = 0.41 GB-seconds
  → Nhiều memory hơn = execution nhanh hơn = có thể rẻ hơn!
```

```bash
# Cập nhật memory size
aws lambda update-function-configuration \
  --function-name my-function \
  --memory-size 1024  # Đơn vị: MB, giá trị từ 128 đến 10240

# Dùng Lambda Power Tuning (Tool tối ưu memory AWS cung cấp miễn phí)
# https://github.com/alexcasalboni/aws-lambda-power-tuning
# → Tự động test nhiều memory sizes và tìm điểm tối ưu cost/performance
```

### Memory Leak Trong Lambda

```python
# Vấn đề: Data tích lũy trong module-level variables
cache = {}  # Không bao giờ được clear

def handler(event, context):
    key = event['key']
    cache[key] = fetch_data(key)  # Cache tăng mãi theo thời gian
    return cache[key]

# Giải pháp: Giới hạn kích thước cache hoặc dùng TTL (Time to Live)
from functools import lru_cache
import time

cache = {}
CACHE_TTL = 300  # 5 phút

def get_cached(key):
    now = time.time()
    if key in cache and now - cache[key]['time'] < CACHE_TTL:
        return cache[key]['value']
    value = fetch_data(key)
    cache[key] = {'value': value, 'time': now}
    # Giới hạn kích thước cache để tránh memory leak
    if len(cache) > 1000:
        oldest = min(cache.keys(), key=lambda k: cache[k]['time'])
        del cache[oldest]
    return value
```

---

## 3. Permission Errors

### Các Lỗi Permission Phổ Biến

```
Error messages:
  "AccessDeniedException: User: arn:aws:sts::...assumed-role/...
   is not authorized to perform: s3:GetObject on resource: ..."

  "An error occurred (AccessDeniedException) when calling the
   InvokeFunction operation: ..."

  "Unable to import module 'lambda_function': No module named 'boto3'"
   (Đây không phải permission — đây là packaging issue)
```

### Chẩn Đoán Permission

```bash
# Xem IAM Role của Lambda function
aws lambda get-function-configuration \
  --function-name my-function \
  --query 'Role'
# Kết quả: arn:aws:iam::123456789:role/my-lambda-role

# Xem policies gắn với role
aws iam list-attached-role-policies \
  --role-name my-lambda-role

# Xem inline policies
aws iam list-role-policies --role-name my-lambda-role

# Test permission bằng CloudTrail — tìm "AccessDenied" events
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject \
  --start-time $(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --query 'Events[?contains(CloudTrailEvent, `AccessDenied`)]'
```

### Các Permission Thường Bị Quên

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    }
  ]
}
```

> **ec2:CreateNetworkInterface** là bắt buộc khi Lambda chạy trong VPC. Nếu thiếu, Lambda không thể attach vào VPC → invocation fail hoặc timeout.

### Resource-Based Policy (Chính Sách Dựa Trên Tài Nguyên)

```bash
# Xem resource-based policy của Lambda (cho phép ai invoke function này)
aws lambda get-policy --function-name my-function

# Ví dụ: Cho phép API Gateway invoke Lambda
aws lambda add-permission \
  --function-name my-function \
  --statement-id allow-apigw \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789:api-id/*/GET/path"

# Cho phép EventBridge (CloudWatch Events) invoke Lambda
aws lambda add-permission \
  --function-name my-function \
  --statement-id allow-eventbridge \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:us-east-1:123456789:rule/my-rule"
```

---

## 4. Cold Start Latency

### Đo Lường Cold Start

```bash
# CloudWatch Insights — tìm cold starts (Init Duration)
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @type = "REPORT"
    | filter @initDuration > 0  # initDuration > 0 nghĩa là có cold start
    | stats
        count(*) as coldStarts,
        avg(@initDuration) as avgInitMs,
        max(@initDuration) as maxInitMs,
        avg(@duration) as avgDurationMs
      by bin(1h)'
```

### Tối Ưu Cold Start Theo Runtime

```
Cold start time trung bình theo runtime (approx):
  Python 3.12:    < 200ms
  Node.js 20:     < 300ms
  Java 21 (SnapStart): < 1s (với SnapStart enabled)
  Java 21 (không SnapStart): 3-10 giây (!)
  .NET 8:         < 500ms
  Go:             < 100ms
```

**Chiến Lược Giảm Cold Start:**

```python
# 1. Giảm package size — import chỉ những gì cần thiết
# Xấu:
import boto3  # Load toàn bộ boto3

# Tốt hơn:
import boto3.s3  # Chỉ load S3 client

# 2. Move initialization ra ngoài handler
import boto3

# Initialization xảy ra một lần khi container khởi tạo
s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('my-table')

def handler(event, context):
    # Handler chỉ chứa business logic, không tái khởi tạo clients
    response = table.get_item(Key={'id': event['id']})
    return response['Item']
```

**Provisioned Concurrency** (Đồng Thời Dự Phòng — Loại Bỏ Cold Start):

```bash
# Bật Provisioned Concurrency cho function alias
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier production  # Lambda alias
  --provisioned-concurrent-executions 10  # Giữ 10 execution environments luôn warm

# Application Auto Scaling cho Provisioned Concurrency
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-function:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 5 \
  --max-capacity 50
```

**SnapStart** (Java — Khởi Động Nhanh Bằng Snapshot):

```bash
# Bật SnapStart cho Java Lambda (giảm cold start từ vài giây xuống dưới 1 giây)
aws lambda update-function-configuration \
  --function-name my-java-function \
  --snap-start ApplyOn=PublishedVersions

# Publish version để SnapStart có hiệu lực
aws lambda publish-version --function-name my-java-function
```

---

## 5. Throttling

### Nhận Biết Throttling

```
Error: TooManyRequestsException
HTTP Status: 429

CloudWatch Metric:
  - Throttles > 0 trong namespace AWS/Lambda
  - ConcurrentExecutions gần limit

Phân biệt hai loại throttling:
  A. Account-level concurrency throttle — Tổng concurrency toàn account đạt limit
  B. Function-level reserved concurrency throttle — Function đạt reserved limit của nó
  C. Burst limit throttle — Số lượng instances tăng quá nhanh trong thời gian ngắn
```

### Kiểm Tra Concurrency Limits

```bash
# Xem account-level concurrency limit
aws lambda get-account-settings \
  --query 'AccountLimit.[ConcurrentExecutions,UnreservedConcurrentExecutions]'

# Xem concurrency hiện tại của từng function
aws lambda list-functions \
  --query 'Functions[*].[FunctionName,ReservedConcurrentExecutions]'

# Xem throttling metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Throttles \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

### Xử Lý Throttling

```
Giải pháp tức thì:
  1. Tăng Reserved Concurrency nếu function quan trọng đang bị throttle
  2. Request limit increase từ AWS Support (account-level concurrency)

Giải pháp kiến trúc:
  1. Dùng SQS Queue (Hàng Đợi) trước Lambda → Tự động buffer requests
     - SQS Event Source Mapping tự động điều chỉnh concurrency
     - Throttled invocations được retry tự động
  
  2. Dùng SNS + SQS Fan-out Pattern (Mẫu Phân Tán)
     → Tránh spike đột ngột từ nhiều sources cùng lúc

  3. Implement exponential backoff trong client code
     → Retry với delay tăng dần khi nhận 429
```

```python
import boto3
import time
from botocore.exceptions import ClientError

lambda_client = boto3.client('lambda')

def invoke_with_retry(function_name, payload, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = lambda_client.invoke(
                FunctionName=function_name,
                Payload=payload
            )
            return response
        except ClientError as e:
            if e.response['Error']['Code'] == 'TooManyRequestsException':
                if attempt == max_retries - 1:
                    raise
                # Exponential backoff: 1s, 2s, 4s
                wait_time = (2 ** attempt)
                time.sleep(wait_time)
            else:
                raise
```

---

## 6. VPC Lambda Issues

### Lambda Trong VPC — Các Vấn Đề Thường Gặp

```
Vấn đề 1: Cold start tăng lên (trước 2019, giờ đã được cải thiện bằng HyperPlane ENI)
Vấn đề 2: Không truy cập được internet (thiếu NAT Gateway)
Vấn đề 3: ENI (Elastic Network Interface) limit của subnet

Kiểm tra ENI usage:
  aws ec2 describe-network-interfaces \
    --filters "Name=description,Values=AWS Lambda VPC ENI*" \
    --query 'NetworkInterfaces[*].[SubnetId,Status,Description]' | wc -l
```

### Cấu Hình VPC Lambda Đúng Cách

```bash
# Lambda cần ít nhất 2 private subnets trong 2 AZ khác nhau (HA)
# Subnets cần có NAT Gateway hoặc VPC Endpoints

# Xem VPC config hiện tại
aws lambda get-function-configuration \
  --function-name my-function \
  --query 'VpcConfig'

# Update VPC config
aws lambda update-function-configuration \
  --function-name my-function \
  --vpc-config '{
    "SubnetIds": ["subnet-private-1a", "subnet-private-1b"],
    "SecurityGroupIds": ["sg-lambda-sg"]
  }'
```

### Khi Nào Nên Dùng VPC Lambda

```
Dùng VPC Lambda KHI:
  ✅ Cần kết nối với RDS, ElastiCache trong private subnet
  ✅ Cần kết nối với resources chỉ có trong VPC
  ✅ Compliance yêu cầu network isolation

KHÔNG dùng VPC Lambda khi:
  ❌ Chỉ cần gọi AWS services (dùng VPC Endpoints thay thế)
  ❌ Chỉ cần gọi external APIs (Lambda không cần VPC để làm điều này)
  ❌ Performance là ưu tiên số 1 (VPC Lambda có cold start cao hơn dù nhỏ)
```

---

## 7. Debugging Workflow

### Quy Trình Gỡ Lỗi Lambda Có Hệ Thống

```
Lambda function báo lỗi/timeout?
│
BƯỚC 1: XEM LOGS
  aws logs filter-log-events \
    --log-group-name /aws/lambda/FUNCTION_NAME \
    --filter-pattern "ERROR" \
    --start-time $(date -d '1 hour ago' +%s000)
│
BƯỚC 2: PHÂN TÍCH ERROR TYPE
  ├─ "Task timed out" → Xem phần 1 (Timeout)
  ├─ "Runtime exited with error: signal: killed" → Xem phần 2 (OOM)
  ├─ "AccessDeniedException" → Xem phần 3 (Permissions)
  ├─ "TooManyRequestsException" → Xem phần 5 (Throttling)
  └─ "Init Duration" cao → Xem phần 4 (Cold Start)
│
BƯỚC 3: KIỂM TRA METRICS
  CloudWatch Lambda Metrics:
  - Duration: P50, P95, P99
  - Errors: Error rate (%)
  - Throttles: Throttle count
  - ConcurrentExecutions: Max concurrent
  - InitDuration: Cold start frequency
│
BƯỚC 4: TEST CÔ LẬP
  # Test thủ công với payload đơn giản
  aws lambda invoke \
    --function-name my-function \
    --payload '{"test": true}' \
    --log-type Tail \
    output.json | jq '.LogResult' | base64 -d
│
BƯỚC 5: X-RAY TRACING (nếu được bật)
  # X-Ray (Theo Dõi Phân Tán) cho thấy chính xác đoạn code nào chậm
  aws xray get-service-graph \
    --start-time $(date -d '1 hour ago' +%s) \
    --end-time $(date +%s)
```

### Lambda Power Tuning — Tối Ưu Memory/Cost

```
AWS Lambda Power Tuning là một Step Functions state machine
cho phép tự động test function ở nhiều memory sizes
và tìm cấu hình tối ưu về cost và/hoặc performance.

Deploy từ AWS Serverless Application Repository:
  https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:
  us-east-1:451282441545:applications~aws-lambda-power-tuning

Chạy với configs: [128, 256, 512, 1024, 2048, 3008] MB
Kết quả: Biểu đồ cost vs performance để chọn memory size tối ưu
```

---

## 📖 Tham Khảo Thêm

- [../03-serverless-lambda/4-concurrency-throttling.md](../03-serverless-lambda/4-concurrency-throttling.md) — Concurrency chi tiết
- [../03-serverless-lambda/5-performance-best-practices.md](../03-serverless-lambda/5-performance-best-practices.md) — Cold start optimization
- [../09-monitoring/4-xray-tracing.md](../09-monitoring/4-xray-tracing.md) — X-Ray distributed tracing

---

**Cập Nhật Lần Cuối:** 2026-05-15
