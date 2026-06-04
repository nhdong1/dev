# ⚡ Lambda Fundamentals — Cấu Trúc & Mô Hình Thực Thi

> Hiểu sâu về anatomy (cấu trúc giải phẫu) của Lambda function: từ handler, runtime, execution role, đến vòng đời của execution environment — nền tảng để làm chủ Serverless trên AWS.

## 📚 Mục Lục

1. [Anatomy Của Lambda Function](#anatomy-của-lambda-function)
2. [Runtimes — Môi Trường Chạy](#runtimes--môi-trường-chạy)
3. [Handler — Điểm Vào Hàm](#handler--điểm-vào-hàm)
4. [Execution Role — Vai Trò Thực Thi](#execution-role--vai-trò-thực-thi)
5. [Execution Environment Lifecycle](#execution-environment-lifecycle)
6. [Cấu Hình Function](#cấu-hình-function)
7. [Environment Variables — Biến Môi Trường](#environment-variables--biến-môi-trường)
8. [Destination & Error Handling](#destination--error-handling)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🔬 Anatomy Của Lambda Function

Một Lambda function bao gồm các thành phần sau:

```
┌─────────────────────────────────────────────────────────────┐
│                    Lambda Function                           │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │   Function   │  │  Deployment  │  │   Configuration   │ │
│  │    Code      │  │   Package    │  │                   │ │
│  │              │  │  (ZIP/Image) │  │ - Memory: 512 MB  │ │
│  │ handler()    │  │              │  │ - Timeout: 30s    │ │
│  │ helper()     │  │ function.py  │  │ - Runtime: Py3.12 │ │
│  │              │  │ requirements │  │ - Arch: arm64     │ │
│  └──────────────┘  └──────────────┘  └───────────────────┘ │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │  Execution   │  │  Environment │  │     Triggers      │ │
│  │    Role      │  │  Variables   │  │   (Event Source   │ │
│  │  (IAM Role)  │  │ DB_HOST=...  │  │    Mapping)       │ │
│  │              │  │ API_KEY=...  │  │ - API Gateway     │ │
│  │ DynamoDB:Get │  │              │  │ - SQS Queue       │ │
│  │ S3:PutObject │  │              │  │ - S3 Bucket       │ │
│  └──────────────┘  └──────────────┘  └───────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 🖥️ Runtimes — Môi Trường Chạy

Runtime là môi trường để Lambda chạy code của bạn. AWS cung cấp **managed runtimes** (môi trường do AWS quản lý) và hỗ trợ **custom runtimes** (tự tạo).

### Managed Runtimes (Môi Trường Quản Lý Sẵn)

| Runtime                  | Phiên Bản         | Trạng Thái  | Ghi Chú                            |
| ------------------------ | ----------------- | ----------- | ---------------------------------- |
| **Python**               | 3.13, 3.12, 3.11  | ✅ Active   | Phổ biến nhất cho data/ML          |
| **Node.js**              | 22.x, 20.x, 18.x  | ✅ Active   | Phổ biến cho API/web               |
| **Java**                 | 21, 17, 11, 8     | ✅ Active   | Enterprise, SnapStart hỗ trợ       |
| **Go**                   | 1.x               | ✅ Active   | Hiệu năng cao, cold start thấp     |
| **.NET**                 | 8 (C#/F#)         | ✅ Active   | Microsoft ecosystem                |
| **Ruby**                 | 3.3, 3.2          | ✅ Active   | Web scripting                      |

### Custom Runtime (Runtime Tùy Chỉnh)

Dùng `provided.al2023` (Amazon Linux 2023) để tạo runtime riêng — hữu ích cho:
- Rust, Erlang, Elixir
- Ngôn ngữ tự tạo
- Phiên bản runtime cụ thể không có sẵn

```bash
# Ví dụ: Rust Lambda với custom runtime
cargo lambda build --release
cargo lambda deploy my-function
```

### Architecture (Kiến Trúc CPU)

Lambda hỗ trợ 2 kiến trúc:

| Kiến Trúc | Chi Phí          | Hiệu Năng        | Khi Dùng                  |
| --------- | ---------------- | ---------------- | ------------------------- |
| **x86_64** | Tiêu chuẩn       | Tốt              | Mặc định, binary phổ biến |
| **arm64** (Graviton2) | Rẻ hơn ~20% | Tốt hơn 19% | Workload mới, tiết kiệm chi phí |

---

## 🚪 Handler — Điểm Vào Hàm

Handler là hàm được Lambda gọi khi có event. Format: `file_name.function_name`.

### Python Handler

```python
import json
import boto3

# Init SDK client bên ngoài handler = được tái sử dụng giữa các warm invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def handler(event, context):
    """
    event   — JSON object từ event source (API GW, SQS, S3...)
    context — object chứa metadata về invocation
    """
    # Đọc thông tin từ context
    print(f"Request ID: {context.aws_request_id}")
    print(f"Thời gian còn lại: {context.get_remaining_time_in_millis()} ms")
    print(f"Function name: {context.function_name}")

    # Xử lý event
    user_id = event.get('pathParameters', {}).get('userId')

    try:
        response = table.get_item(Key={'userId': user_id})
        user = response.get('Item')

        if not user:
            return {
                'statusCode': 404,
                'body': json.dumps({'error': 'User not found'})
            }

        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(user)
        }
    except Exception as e:
        print(f"Error: {str(e)}")
        raise  # Lambda sẽ retry nếu là async invocation
```

### Node.js Handler

```javascript
const { DynamoDBClient, GetItemCommand } = require("@aws-sdk/client-dynamodb");

// Init client bên ngoài handler = reused across warm starts
const client = new DynamoDBClient({ region: process.env.AWS_REGION });

exports.handler = async (event, context) => {
    console.log("Event:", JSON.stringify(event, null, 2));
    console.log("Remaining time:", context.getRemainingTimeInMillis(), "ms");

    const userId = event.pathParameters?.userId;

    try {
        const command = new GetItemCommand({
            TableName: "Users",
            Key: { userId: { S: userId } }
        });
        const response = await client.send(command);

        if (!response.Item) {
            return {
                statusCode: 404,
                body: JSON.stringify({ error: "User not found" })
            };
        }

        return {
            statusCode: 200,
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(response.Item)
        };
    } catch (error) {
        console.error("Error:", error);
        throw error;
    }
};
```

### Java Handler

```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;

public class UserHandler implements
    RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    // Static field = initialized once per execution environment
    private static final DynamoDbClient dynamoDb = DynamoDbClient.create();

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {

        context.getLogger().log("Request ID: " + context.getAwsRequestId());

        String userId = event.getPathParameters().get("userId");
        // ... xử lý logic
        return new APIGatewayProxyResponseEvent()
            .withStatusCode(200)
            .withBody("{\"userId\": \"" + userId + "\"}");
    }
}
```

### Context Object — Các Thuộc Tính Quan Trọng

| Thuộc Tính                        | Mô Tả                                      |
| ---------------------------------- | ------------------------------------------ |
| `aws_request_id`                   | ID duy nhất của invocation                 |
| `function_name`                    | Tên function                               |
| `function_version`                 | Version đang chạy ($LATEST hoặc số)        |
| `memory_limit_in_mb`               | Memory được cấu hình                       |
| `get_remaining_time_in_millis()`   | Milliseconds còn lại trước khi timeout     |
| `log_group_name`                   | CloudWatch Log Group name                  |
| `log_stream_name`                  | CloudWatch Log Stream name                 |

---

## 🔐 Execution Role — Vai Trò Thực Thi

**Execution Role** (Vai Trò Thực Thi) là IAM Role được Lambda assume để:
- Ghi log vào CloudWatch Logs (bắt buộc)
- Truy cập các AWS services khác (DynamoDB, S3, SQS...)
- Assume role tạm thời khi function chạy

### Minimum Execution Role Policy (Policy Tối Thiểu)

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
    }
  ]
}
```

### Trust Policy (Chính Sách Tin Tưởng) — Bắt Buộc

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Best Practice: Least Privilege (Quyền Tối Thiểu)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudWatchLogs",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:123456789:log-group:/aws/lambda/my-function:*"
    },
    {
      "Sid": "AllowDynamoDBAccess",
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Users"
    },
    {
      "Sid": "AllowS3ReadAccess",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/uploads/*"
    }
  ]
}
```

### AWS Managed Policies Hữu Ích

| Policy                                        | Quyền Gồm                                  |
| --------------------------------------------- | ------------------------------------------- |
| `AWSLambdaBasicExecutionRole`                  | CloudWatch Logs (bắt buộc)                 |
| `AWSLambdaVPCAccessExecutionRole`              | CloudWatch Logs + VPC ENI management        |
| `AWSLambdaDynamoDBExecutionRole`               | CloudWatch Logs + DynamoDB Streams          |
| `AWSLambdaSQSQueueExecutionRole`               | CloudWatch Logs + SQS polling               |
| `AWSXRayDaemonWriteAccess`                     | Gửi traces đến X-Ray                        |

---

## 🔄 Execution Environment Lifecycle (Vòng Đời Môi Trường Thực Thi)

Đây là khái niệm quan trọng nhất để hiểu Lambda hoạt động:

```
                    ┌─────────────────────────────────────────┐
                    │         Execution Environment           │
                    │         (Môi Trường Thực Thi)           │
                    │                                         │
Phase 1: INIT       │  1. Download deployment package         │
(Khởi tạo)         │  2. Start runtime (Python/Node/Java...) │
                    │  3. Run initialization code             │
                    │     (global scope, static blocks)       │
                    │                                         │
Phase 2: INVOKE     │  4. Run handler() function              │
(Thực thi)         │  5. Return response                     │
                    │  6. Freeze environment                  │
                    │                                         │
Phase 3: SHUTDOWN   │  7. Extension shutdown hooks            │
(Tắt)              │  8. Release resources                   │
                    │  (Xảy ra sau thời gian idle)            │
                    └─────────────────────────────────────────┘
```

### Init Phase Chi Tiết

```
INIT Phase:
├── Extension Init    — Lambda Extensions khởi tạo
├── Runtime Init      — Runtime bootstrap (Python interpreter, JVM...)
└── Function Init     — Code trong global scope/static chạy một lần

VD Python:
  import boto3                          # Runtime Init
  client = boto3.client('dynamodb')     # Function Init ← chạy 1 lần/environment
  
  def handler(event, context):          # Invoke Phase ← chạy mỗi invocation
      response = client.get_item(...)   # Reuses client từ init!
```

### Tái Sử Dụng Execution Environment (Warm Start)

Sau khi invocation hoàn tất, execution environment bị **frozen** (đóng băng) và có thể được **reused** (tái sử dụng) cho invocation tiếp theo:

```python
# Đây là pattern ĐÚNG — tận dụng warm start:
import boto3

# Khởi tạo một lần, tái sử dụng nhiều lần
s3_client = boto3.client('s3')
db_connection = create_db_connection()  # connection pooling!

def handler(event, context):
    # Sử dụng lại s3_client và db_connection
    return s3_client.get_object(Bucket='my-bucket', Key='file.json')
```

```python
# Đây là pattern SAI — tốn thêm thời gian mỗi invocation:
def handler(event, context):
    # Tạo mới mỗi lần → chậm hơn, không cần thiết
    s3_client = boto3.client('s3')
    return s3_client.get_object(Bucket='my-bucket', Key='file.json')
```

### /tmp Storage — Lưu Trữ Tạm Thời

```python
import os
import json

CACHE_FILE = '/tmp/cached_data.json'

def handler(event, context):
    # Kiểm tra xem /tmp có dữ liệu cached không (từ warm invocation)
    if os.path.exists(CACHE_FILE):
        with open(CACHE_FILE, 'r') as f:
            return json.load(f)

    # Fetch data và cache vào /tmp
    data = fetch_expensive_data()
    with open(CACHE_FILE, 'w') as f:
        json.dump(data, f)
    
    return data
```

**Lưu ý:** `/tmp` data tồn tại giữa các warm invocations nhưng KHÔNG chia sẻ giữa các concurrent instances.

---

## ⚙️ Cấu Hình Function

### Memory & CPU

Lambda không cho phép cấu hình CPU riêng — CPU tỉ lệ thuận với Memory:

| Memory (MB) | vCPU (ảo)    | Thích Hợp Cho                         |
| ----------- | ------------ | -------------------------------------- |
| 128         | 0.125 vCPU   | Simple logic, không cần tính toán      |
| 512         | 0.5 vCPU     | API handlers phổ biến                  |
| 1,024       | 1 vCPU       | Xử lý files, JSON parsing              |
| 1,769       | 1 vCPU đầy đủ| Compute-intensive tasks                |
| 3,008       | 2 vCPU       | Parallel processing nhỏ                |
| 10,240      | 6 vCPU       | Machine learning inference, video      |

**Nguyên tắc:** Tăng memory giảm execution time, có thể giảm chi phí tổng thể vì cost = memory × time.

### Timeout (Thời Gian Chờ Tối Đa)

```
Min: 1 giây
Max: 900 giây (15 phút)
Default: 3 giây

Hướng dẫn:
- API handlers: 5-30 giây
- Async processing: 60-300 giây
- Batch jobs: 300-900 giây
```

### Deployment Package (Gói Triển Khai)

Lambda nhận code qua 2 dạng:

#### 1. ZIP Archive

```
Giới hạn:
- Upload trực tiếp: 50 MB (compressed)
- Qua S3:          250 MB (uncompressed)

Cấu trúc Python:
my-function/
├── lambda_function.py   # handler
├── requirements.txt
└── lib/                 # dependencies (cài vào đây, không dùng venv)

Cấu trúc Node.js:
my-function/
├── index.js             # handler
├── package.json
└── node_modules/        # dependencies
```

#### 2. Container Image (Hình Ảnh Container)

```
Giới hạn: 10 GB
Tốt cho: Dependencies lớn (ML models, complex libraries)

Dockerfile:
FROM public.ecr.aws/lambda/python:3.12

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY lambda_function.py .
CMD ["lambda_function.handler"]
```

---

## 🌍 Environment Variables — Biến Môi Trường

### Cách Sử Dụng

```python
import os

# Đọc environment variables
DB_HOST = os.environ.get('DB_HOST', 'localhost')
TABLE_NAME = os.environ['DYNAMODB_TABLE']  # Raise error nếu không có
LOG_LEVEL = os.environ.get('LOG_LEVEL', 'INFO')

def handler(event, context):
    print(f"Connecting to: {DB_HOST}")
    # ...
```

### Encryption (Mã Hóa)

Lambda tự động mã hóa environment variables **at rest** bằng AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa):

```
Mặc định: Dùng AWS managed key
Custom: Có thể dùng Customer Managed Key (CMK — Khóa Do Khách Hàng Quản Lý)
```

### Secrets — Không Hardcode trong Environment Variables

```python
import boto3
import json

# ✅ ĐÚNG: Lấy secret từ Secrets Manager khi cần
def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# ❌ SAI: Không đặt password/API key trực tiếp trong env var
# DB_PASSWORD = os.environ['DB_PASSWORD']  # Visible trong console!
```

---

## 📤 Destination & Error Handling (Đích Đến & Xử Lý Lỗi)

### Lambda Destinations (Đích Đến Lambda)

Dành cho **asynchronous invocations** (gọi bất đồng bộ) — cho phép định tuyến kết quả:

```
Lambda Function
      │
      ├─ On Success (Thành công) → SQS / SNS / EventBridge / Lambda
      └─ On Failure (Thất bại)  → SQS / SNS / EventBridge / Lambda
```

```json
{
  "DestinationConfig": {
    "OnSuccess": {
      "Destination": "arn:aws:sqs:us-east-1:123456789:success-queue"
    },
    "OnFailure": {
      "Destination": "arn:aws:sqs:us-east-1:123456789:dlq"
    }
  }
}
```

### Dead Letter Queue — DLQ (Hàng Đợi Thư Chết)

Khi Lambda async invocation thất bại sau tất cả retries, message được gửi đến DLQ:

```
Async Event → Lambda → Fail → Retry (2 lần) → DLQ (SQS/SNS)
                                   ↑
                            Cấu hình: 0-2 retries
```

### Retry Behavior (Hành Vi Retry)

| Invocation Type     | Retry Behavior                                    |
| ------------------- | ------------------------------------------------- |
| **Synchronous**     | Không retry — caller nhận error ngay              |
| **Asynchronous**    | Retry 2 lần với backoff (0s, 60s, 180s)           |
| **Poll-based** (SQS)| Giữ message trong queue đến visibility timeout    |

---

## 🔢 Versions & Aliases (Phiên Bản & Bí Danh)

### Versions (Phiên Bản)

```
$LATEST → Version không cố định (mutable)
1       → Version 1 (immutable — bất biến)
2       → Version 2 (immutable)

Tạo version:
aws lambda publish-version --function-name my-function
```

### Aliases (Bí Danh)

```
production → 2      (100% traffic)
staging    → 3      (100% traffic)
canary     → 2/3    (90% → v2, 10% → v3) ← Traffic splitting
```

```bash
# Tạo alias với weighted routing (phân chia traffic)
aws lambda create-alias \
  --function-name my-function \
  --name canary \
  --function-version 2 \
  --routing-config AdditionalVersionWeights={"3"=0.1}
```

### Ứng Dụng Thực Tế

```
CI/CD Pipeline:
1. Deploy code mới → $LATEST
2. Run integration tests trên $LATEST
3. Publish version (ví dụ: version 5)
4. Update alias 'production' sang version 5
5. Rollback dễ dàng: Update alias về version 4
```

---

## 📊 Monitoring Cơ Bản (Giám Sát)

### CloudWatch Metrics Tự Động

Lambda tự động gửi các metrics sau đến CloudWatch:

| Metric              | Mô Tả                                           |
| ------------------- | ----------------------------------------------- |
| `Invocations`       | Số lần được gọi                                  |
| `Duration`          | Thời gian thực thi (ms) — min, max, avg, p99     |
| `Errors`            | Số lần lỗi (exception, timeout, OOM)             |
| `Throttles`         | Số lần bị throttle do vượt concurrency limit     |
| `ConcurrentExecutions` | Số lần chạy song song tại một thời điểm      |
| `DeadLetterErrors`  | Lỗi khi gửi vào DLQ                             |
| `IteratorAge`       | Độ trễ xử lý stream (Kinesis/DynamoDB Streams)   |

### Structured Logging (Ghi Log Có Cấu Trúc)

```python
import json
import logging
import os

logger = logging.getLogger()
logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))

def handler(event, context):
    # Structured log — dễ query với CloudWatch Insights
    logger.info(json.dumps({
        "message": "Processing request",
        "requestId": context.aws_request_id,
        "userId": event.get("userId"),
        "action": "get_user"
    }))
    # ...
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Tại sao nên init AWS SDK client bên ngoài handler?**
> Client được khởi tạo một lần trong Function Init phase và được tái sử dụng (reuse) trong các warm invocations. Nếu init trong handler, mỗi invocation tốn thêm thời gian tạo connection — có thể tăng 10-100ms latency.

**Q: `/tmp` khác gì environment variables?**
> Environment variables là key-value string, max 4 KB tổng, dùng cho config. `/tmp` là filesystem 512 MB - 10 GB, dùng cho files tạm thời. Cả hai đều tồn tại giữa warm invocations nhưng không chia sẻ giữa concurrent instances.

**Q: Versions và Aliases giải quyết vấn đề gì trong production?**
> Versions tạo snapshot bất biến của function code + config. Aliases tạo pointer có tên (ví dụ: "production") đến version cụ thể. Kết hợp lại cho phép: canary deployment (traffic splitting), instant rollback mà không cần redeploy, và stable ARN cho downstream services dù code thay đổi.

**Q: Lambda Execution Role khác gì với Lambda Resource-Based Policy?**
> Execution Role: IAM Role Lambda *assume* để gọi AWS services khác (outbound). Resource-Based Policy: Chính sách gắn vào Lambda function cho phép *ai* có thể invoke function (inbound) — ví dụ: API Gateway, S3 bucket được phép gọi function.

**Q: Khi nào Lambda raise exception vs return error response?**
> Synchronous invocation: Nên return error response (HTTP 4xx/5xx) để caller xử lý. Async invocation: Raise exception sẽ trigger retry logic và DLQ — quan trọng để không mất event. Poll-based (SQS): Raise exception giữ message trong queue để retry.

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [2-event-sources.md](./2-event-sources.md) — Event Sources & Triggers
