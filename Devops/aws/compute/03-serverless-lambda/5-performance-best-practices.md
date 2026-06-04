# 🚀 Performance & Best Practices — Tối Ưu Lambda

> Tối ưu Lambda performance bao gồm: giảm cold start latency, chọn đúng memory/timeout, tối ưu code, cấu hình VPC đúng cách, và tuân theo các best practices để xây dựng serverless applications production-ready.

## 📚 Mục Lục

1. [Cold Start Optimization — Tối Ưu Khởi Động Lạnh](#cold-start-optimization)
2. [SnapStart — Java Performance](#snapstart--java-performance)
3. [Memory & CPU Tuning](#memory--cpu-tuning)
4. [Lambda trong VPC — Cấu Hình Mạng](#lambda-trong-vpc)
5. [Code Optimization — Tối Ưu Code](#code-optimization)
6. [Connection Reuse — Tái Sử Dụng Kết Nối](#connection-reuse)
7. [Async Patterns — Mẫu Bất Đồng Bộ](#async-patterns)
8. [Security Best Practices](#security-best-practices)
9. [Cost Optimization — Tối Ưu Chi Phí](#cost-optimization)
10. [Production Checklist — Danh Sách Kiểm Tra](#production-checklist)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ❄️ Cold Start Optimization — Tối Ưu Khởi Động Lạnh

### Tại Sao Cold Start Xảy Ra?

```
Cold Start xảy ra khi:
1. Lambda function được gọi lần đầu tiên
2. Lambda cần tạo thêm instances do traffic tăng
3. Execution environment bị terminate sau thời gian idle

Cold Start Timeline:
[Download Package] → [Init MicroVM] → [Init Runtime] → [Init Code] → [Execute]
    ~50-200ms           ~50ms            ~100-500ms      ~100ms      ~your code
                                         ↑
                              Phần tốn thời gian nhất (JVM = 1-2s)
```

### Giảm Cold Start — Các Chiến Lược

#### 1. Chọn Runtime Nhanh

```
Cold start time (ước tính):
Go:         ~50ms   ⚡ Nhanh nhất
Python:     ~100ms  ⚡ Nhanh
Node.js:    ~150ms  ⚡ Nhanh
Ruby:       ~200ms  🟡 Trung bình
.NET 8:     ~400ms  🟡 Trung bình
Java 17:    ~1-2s   🔴 Chậm (JVM startup)
  └─ Với SnapStart: ~200ms ⚡
```

#### 2. Giảm Package Size (Kích Thước Gói)

```bash
# Python: Loại bỏ dependencies không dùng
pip install \
  --no-deps \         # Không cài transitive deps
  --only-binary=:all: \  # Dùng pre-compiled binaries
  -t ./package/ \
  requests

# Node.js: Dùng esbuild để bundle và minify
npx esbuild src/index.ts \
  --bundle \
  --minify \
  --target=node20 \
  --platform=node \
  --outfile=dist/index.js \
  --external:aws-sdk  # Exclude AWS SDK (đã có sẵn trong runtime)

# Loại bỏ files không cần thiết:
find ./package -type d -name "tests" -exec rm -rf {} +
find ./package -name "*.pyc" -delete
find ./package -name "*.md" -delete
```

#### 3. Tối Ưu Init Code (Code Khởi Tạo)

```python
# ❌ Tệ: Init nặng trong handler
def handler(event, context):
    import pandas as pd          # Import chậm mỗi lần!
    import numpy as np
    config = load_config()       # Network call mỗi lần!
    model = load_ml_model()      # Tốn 2s mỗi lần!
    ...

# ✅ Tốt: Init bên ngoài handler (chỉ chạy 1 lần per environment)
import pandas as pd      # Import ở module level = chạy 1 lần
import numpy as np

# Init outside handler = reused across warm invocations
config = load_config()    # 1 lần duy nhất
model = load_ml_model()   # 1 lần duy nhất

def handler(event, context):
    # Chỉ chứa request-specific logic
    data = pd.DataFrame(event['data'])
    return model.predict(data)
```

#### 4. Lazy Loading (Tải Lười Biếng) — Khi Không Phải Mọi Request Đều Cần

```python
_db_connection = None  # Lazy init

def get_db():
    global _db_connection
    if _db_connection is None or not _db_connection.is_connected():
        _db_connection = create_db_connection()
    return _db_connection

def handler(event, context):
    if event.get('action') == 'health_check':
        return {'status': 'ok'}  # Không cần DB
    
    db = get_db()  # Chỉ connect DB khi thực sự cần
    return db.query(...)
```

#### 5. Provisioned Concurrency (Đã đề cập ở file 4)

```bash
# Pre-warm 10 execution environments
aws lambda put-provisioned-concurrency-config \
  --function-name my-api \
  --qualifier production \
  --provisioned-concurrent-executions 10
```

#### 6. Warm-up Scheduled Invocations (Gọi Định Kỳ Để Giữ Warm)

```python
# CloudWatch Event Rule gọi Lambda mỗi 5 phút
# Event payload đặc biệt để Lambda biết đây là warm-up call

def handler(event, context):
    # Kiểm tra warm-up event
    if event.get('source') == 'warmup':
        print("Warm-up invocation - no action needed")
        return {'status': 'warmed'}
    
    # Business logic bình thường
    ...
```

```bash
# EventBridge rule để warm-up mỗi 5 phút
aws events put-rule \
  --name lambda-warmup \
  --schedule-expression "rate(5 minutes)"

aws events put-targets \
  --rule lambda-warmup \
  --targets Id=1,Arn=arn:aws:lambda:...:my-function,Input='{"source":"warmup"}'
```

---

## ☕ SnapStart — Java Performance

### SnapStart Là Gì?

**SnapStart** là tính năng của Lambda dành riêng cho **Java** runtime, giúp giảm cold start từ 2-10 giây xuống còn ~200ms:

```
Không có SnapStart:
INIT phase:
[JVM Start] → [Class Loading] → [JIT Compilation] → [App Init] → [Execute]
    ~500ms        ~500ms            ~500ms             ~1s           fast

Với SnapStart:
Build time:
[JVM Start] → [App Init] → [Snapshot] → [Store in cache]
                              ↑ Snapshot được tạo 1 lần

Khi có request:
[Restore Snapshot] → [Execute]
    ~200ms              fast
```

### Cách Hoạt Động

1. Khi publish version Lambda, AWS khởi tạo function và tạo **snapshot** (ảnh chụp)
2. Snapshot được cached (lưu vào bộ nhớ đệm) ở multiple AZs
3. Khi có request, Lambda restore từ snapshot thay vì init lại từ đầu

### Cấu Hình SnapStart

```bash
# Enable SnapStart khi tạo function (chỉ cho Java 11+)
aws lambda create-function \
  --function-name my-java-api \
  --runtime java21 \
  --handler com.example.Handler::handleRequest \
  --snap-start ApplyOn=PublishedVersions \
  --role arn:aws:iam::123456789:role/lambda-role \
  --zip-file fileb://function.jar \
  --memory-size 1024
```

### Lưu Ý Quan Trọng Với SnapStart

```java
// SnapStart hooks — Cleanup và Re-init sau khi restore
import com.amazonaws.services.lambda.crac.*;

public class Handler implements RequestHandler<APIGatewayEvent, Response>, CRaCEntryPoint {
    
    private DatabaseConnection dbConn;
    
    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        // Chạy TRƯỚC khi snapshot được tạo
        // Đóng connections, flush buffers, release resources
        dbConn.close();
        System.out.println("Closing connections before snapshot");
    }
    
    @Override
    public void afterRestore(Context<? extends Resource> context) {
        // Chạy SAU khi restore từ snapshot
        // Mở lại connections, re-init state
        dbConn = createNewConnection();
        System.out.println("Re-opening connections after restore");
    }
    
    @Override
    public Response handleRequest(APIGatewayEvent event, Context context) {
        return processRequest(event);
    }
}
```

```
Cảnh báo với SnapStart:
- Không lưu secrets/credentials trong snapshot (rotate after restore)
- Random number generators cần seed mới sau restore
- TCP connections không tồn tại qua snapshot → phải reconnect
- Sử dụng CRaC hooks (beforeCheckpoint/afterRestore) để xử lý
```

---

## 💾 Memory & CPU Tuning (Điều Chỉnh Bộ Nhớ & CPU)

### Nguyên Tắc Memory/CPU

```
Memory ↑ → vCPU ↑ → Execution time ↓ → Cost có thể ↓ (vì thời gian ngắn hơn)

Chi phí = Memory × Duration
  128 MB × 2s   = 256 GB-ms  → $0.00000426
  256 MB × 0.8s = 204 GB-ms  → $0.00000340  (rẻ hơn mặc dù nhiều RAM hơn!)
```

### Lambda Power Tuning Tool (Công Cụ Điều Chỉnh Hiệu Năng)

AWS cung cấp **Lambda Power Tuning** — Step Functions state machine để tự động tìm memory tối ưu:

```bash
# Deploy Lambda Power Tuning từ SAR (Serverless Application Repository)
aws serverlessrepo create-cloud-formation-change-set \
  --application-id arn:aws:serverlessrepo:us-east-1:451282441545:applications/aws-lambda-power-tuning \
  --stack-name lambda-power-tuning \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides '[{"Name":"lambdaResource","Value":"*"}]'

# Sau khi deploy, chạy state machine với config:
{
  "lambdaARN": "arn:aws:lambda:us-east-1:123456789:function:my-function",
  "powerValues": [128, 256, 512, 1024, 1769, 3008],
  "num": 50,
  "payload": {"key": "value"},
  "parallelInvocation": true,
  "strategy": "cost"  // hoặc "speed"
}

# Kết quả: Biểu đồ cost vs speed cho từng memory size
# → Chọn "knee of the curve" (điểm optimal)
```

### Hướng Dẫn Chọn Memory

```
128 MB:  Health checks, simple routing, no computation
256 MB:  Simple API calls, JSON parsing
512 MB:  Moderate computation, file processing nhỏ
1,024 MB: Standard workloads, moderate I/O
1,769 MB: Full 1 vCPU — compute-intensive tasks
3,008 MB: 2 vCPU — parallel processing, image resize
5,120 MB: 3 vCPU — video processing, data transformation
10,240 MB: 6 vCPU — heavy ML inference, large datasets
```

### Timeout Best Practices

```python
import os
import time

TIMEOUT_BUFFER_MS = 1000  # 1 giây buffer trước khi timeout

def handler(event, context):
    start_time = time.time()
    
    for item in large_list:
        # Kiểm tra thời gian còn lại
        remaining = context.get_remaining_time_in_millis()
        
        if remaining < TIMEOUT_BUFFER_MS:
            # Không đủ thời gian xử lý tiếp
            # Lưu checkpoint và trả về partial result
            save_checkpoint(item)
            return {
                'statusCode': 206,  # Partial Content
                'body': 'Processed partially, will continue from checkpoint'
            }
        
        process_item(item)
    
    return {'statusCode': 200, 'body': 'Done'}
```

---

## 🌐 Lambda Trong VPC — Cấu Hình Mạng

### Khi Nào Lambda Cần VPC?

```
Lambda MẶC ĐỊNH không trong VPC:
  ✅ Gọi public AWS APIs (DynamoDB, S3, SQS)
  ✅ Gọi external HTTP APIs
  ❌ Không thể kết nối RDS trong private subnet
  ❌ Không thể kết nối ElastiCache trong private subnet
  ❌ Không thể kết nối internal services trong VPC

Lambda TRONG VPC:
  ✅ Kết nối RDS, ElastiCache, private microservices
  ✅ Truy cập resources trong private subnet
  ❌ Không có internet access mặc định
  → Cần NAT Gateway để ra internet
  → Hoặc dùng VPC Endpoints cho AWS services
```

### Cấu Hình Lambda trong VPC

```bash
aws lambda create-function \
  --function-name db-function \
  --vpc-config SubnetIds=["subnet-private-1","subnet-private-2"],\
               SecurityGroupIds=["sg-lambda-to-rds"] \
  --role arn:aws:iam::123456789:role/lambda-vpc-role
```

```json
{
  "VpcConfig": {
    "SubnetIds": [
      "subnet-private-us-east-1a",
      "subnet-private-us-east-1b"
    ],
    "SecurityGroupIds": ["sg-lambda-function"]
  }
}
```

### VPC Networking Architecture (Kiến Trúc Mạng VPC)

```
VPC: 10.0.0.0/16
│
├── Public Subnets:
│   ├── 10.0.1.0/24 (us-east-1a) ─── NAT Gateway
│   └── 10.0.2.0/24 (us-east-1b) ─── NAT Gateway
│
└── Private Subnets:
    ├── 10.0.11.0/24 (us-east-1a)
    │   ├── Lambda ENI (Elastic Network Interface — Giao Diện Mạng Ảo)
    │   └── RDS Instance
    │
    └── 10.0.12.0/24 (us-east-1b)
        ├── Lambda ENI
        └── RDS Read Replica

Lambda → ENI → Private Subnet → RDS   ✅
Lambda → NAT GW → Internet            ✅ (nếu cần)
Lambda → VPC Endpoint → S3/DynamoDB   ✅ (không qua internet)
```

### ENI (Elastic Network Interface) — Trước Đây Là Vấn Đề Cold Start

Trước 2019, Lambda trong VPC tạo ENI mới mỗi cold start → +10 giây cold start!

**Hiện tại (2019+):** Lambda tạo ENIs từ subnet range và tái sử dụng — cold start chỉ thêm ~500ms.

### Security Groups cho Lambda trong VPC

```
Lambda Security Group (sg-lambda):
  Outbound:
    → Port 5432 → sg-rds (PostgreSQL)
    → Port 6379 → sg-redis (ElastiCache)
    → Port 443  → 0.0.0.0/0 (HTTPS internet qua NAT)

RDS Security Group (sg-rds):
  Inbound:
    ← Port 5432 ← sg-lambda (chỉ từ Lambda)
```

### VPC Endpoints — Tránh Đi Qua Internet

```bash
# Tạo S3 VPC Endpoint (Gateway type — miễn phí)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private-1 rtb-private-2

# Tạo DynamoDB VPC Endpoint (Gateway type — miễn phí)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids rtb-private-1 rtb-private-2

# Tạo Secrets Manager VPC Endpoint (Interface type — có phí)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids subnet-private-1 subnet-private-2 \
  --security-group-ids sg-vpce

# Kết quả: Lambda → VPC Endpoint → AWS Service (không qua internet)
# → Nhanh hơn, rẻ hơn (không qua NAT Gateway)
```

---

## ⚙️ Code Optimization — Tối Ưu Code

### Initialization Best Practices (Thực Hành Tốt Nhất Khi Khởi Tạo)

```python
import os
import boto3
import json
from functools import lru_cache

# 1. AWS SDK clients — init một lần
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])
s3 = boto3.client('s3')
ssm = boto3.client('ssm')

# 2. Config từ environment variables — nhanh hơn SSM mỗi request
REGION = os.environ['AWS_REGION']
BUCKET = os.environ['S3_BUCKET']
LOG_LEVEL = os.environ.get('LOG_LEVEL', 'INFO')

# 3. Config từ Parameter Store — cache kết quả
@lru_cache(maxsize=128)
def get_config(param_name):
    response = ssm.get_parameter(Name=param_name, WithDecryption=True)
    return response['Parameter']['Value']

# 4. ML models — load một lần, tốn nhiều memory nhưng fast sau đó
# model = load_model('/opt/ml/model.pkl')  # Từ Layer hoặc /tmp

def handler(event, context):
    # Handler chỉ chứa request-specific logic
    user_id = event['userId']
    config = get_config('/myapp/config')  # Cached sau lần đầu
    
    item = table.get_item(Key={'userId': user_id}).get('Item')
    return {'statusCode': 200, 'body': json.dumps(item)}
```

### Error Handling (Xử Lý Lỗi) Patterns

```python
import logging
import traceback
from typing import Any, Dict

logger = logging.getLogger()
logger.setLevel(logging.INFO)

class BusinessException(Exception):
    """Lỗi nghiệp vụ — không nên retry"""
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code
        super().__init__(message)

class RetryableException(Exception):
    """Lỗi tạm thời — nên retry"""
    pass

def handler(event: Dict, context: Any) -> Dict:
    try:
        result = process_request(event)
        return {'statusCode': 200, 'body': json.dumps(result)}
    
    except BusinessException as e:
        # Client error — không retry
        logger.warning(f"Business error: {e.message}")
        return {
            'statusCode': e.status_code,
            'body': json.dumps({'error': e.message})
        }
    
    except RetryableException as e:
        # System error — raise để Lambda retry (async) hoặc SQS retry
        logger.error(f"Retryable error: {str(e)}")
        raise
    
    except Exception as e:
        # Unexpected error — log full traceback
        logger.error(f"Unexpected error: {traceback.format_exc()}")
        # Raise để async invocations retry và tránh mất data
        raise
```

### Structured Logging (Ghi Log Có Cấu Trúc)

```python
import json
import logging
import os
import time

logger = logging.getLogger()
logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))

def log(level, message, **kwargs):
    """Structured JSON logging cho CloudWatch Insights"""
    log_entry = {
        'level': level,
        'message': message,
        'timestamp': time.time(),
        **kwargs
    }
    if level == 'ERROR':
        logger.error(json.dumps(log_entry))
    elif level == 'WARNING':
        logger.warning(json.dumps(log_entry))
    else:
        logger.info(json.dumps(log_entry))

def handler(event, context):
    log('INFO', 'Request received',
        request_id=context.aws_request_id,
        user_id=event.get('userId'),
        action='get_user'
    )
    
    start = time.time()
    result = process(event)
    duration = (time.time() - start) * 1000
    
    log('INFO', 'Request completed',
        request_id=context.aws_request_id,
        duration_ms=duration,
        status='success'
    )
    
    return result
```

---

## 🔄 Connection Reuse — Tái Sử Dụng Kết Nối

### HTTP Keep-Alive (Duy Trì Kết Nối)

```python
# Python requests — bật connection pooling
import requests
from requests.adapters import HTTPAdapter

# Session được tái sử dụng giữa warm invocations
session = requests.Session()
adapter = HTTPAdapter(pool_connections=5, pool_maxsize=20)
session.mount('https://', adapter)
session.mount('http://', adapter)

def handler(event, context):
    # Reuses existing TCP connection — không overhead TLS handshake
    response = session.get('https://api.example.com/data')
    return response.json()
```

```javascript
// Node.js — bật keep-alive
const https = require('https');
const axios = require('axios');

// Tạo agent với keepAlive = true (ngoài handler để reuse)
const httpsAgent = new https.Agent({ keepAlive: true });
const apiClient = axios.create({ httpsAgent });

exports.handler = async (event) => {
    // Reuses TCP connection
    const response = await apiClient.get('https://api.example.com/data');
    return response.data;
};
```

### Database Connection Pooling (Pool Kết Nối Database)

```python
# ❌ Không dùng trực tiếp khi Lambda scale lớn
# → Mỗi concurrent instance = 1 connection
# → 500 concurrent Lambda = 500 DB connections → RDS quá tải

# ✅ Giải pháp 1: RDS Proxy
# Lambda → RDS Proxy (pool 100 connections) → RDS
# RDS Proxy tái sử dụng connections cho nhiều Lambda instances

# ✅ Giải pháp 2: Giới hạn bằng Reserved Concurrency
# Reserved = 80 → tối đa 80 connections → RDS ok

# ✅ Giải pháp 3: Serverless-friendly databases
# DynamoDB — không có connection limit
# Aurora Serverless — scale automatically
# ElastiCache với connection pooling
```

---

## 🛡️ Security Best Practices (Thực Hành Tốt Nhất Bảo Mật)

### IAM Least Privilege (Quyền Tối Thiểu IAM)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Users",
      "Condition": {
        "StringEquals": {
          "dynamodb:LeadingKeys": "${aws:PrincipalTag/userId}"
        }
      }
    }
  ]
}
```

### Secrets Management (Quản Lý Bí Mật)

```python
import boto3
import json
import os
from functools import lru_cache

secrets_client = boto3.client('secretsmanager')

@lru_cache(maxsize=10)
def get_secret(secret_name):
    """Cache secrets để tránh gọi Secrets Manager mỗi invocation"""
    try:
        response = secrets_client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except Exception as e:
        raise RuntimeError(f"Cannot retrieve secret {secret_name}: {e}")

def handler(event, context):
    # Tên secret từ environment variable, không hardcode
    secret_name = os.environ['DB_SECRET_NAME']
    secret = get_secret(secret_name)  # Cached sau lần đầu
    
    db_host = secret['host']
    db_password = secret['password']
    # ...
```

### Input Validation (Xác Thực Đầu Vào)

```python
from pydantic import BaseModel, validator, ValidationError
import json

class CreateUserRequest(BaseModel):
    name: str
    email: str
    age: int
    
    @validator('name')
    def name_not_empty(cls, v):
        if not v.strip():
            raise ValueError('Name cannot be empty')
        return v.strip()
    
    @validator('age')
    def age_valid(cls, v):
        if not 0 < v < 150:
            raise ValueError('Age must be between 1 and 149')
        return v

def handler(event, context):
    try:
        body = json.loads(event.get('body', '{}'))
        request = CreateUserRequest(**body)
    except ValidationError as e:
        return {
            'statusCode': 400,
            'body': json.dumps({'errors': e.errors()})
        }
    
    # Dùng request.name, request.email đã được validate
```

---

## 💰 Cost Optimization — Tối Ưu Chi Phí

### Tối Ưu Duration (Thời Gian Thực Thi)

```
1. Tối ưu memory (đã nói ở trên)
2. Tránh blocking operations:
   - Dùng async/await trong Node.js
   - Parallel calls thay vì sequential

3. Caching thông minh:
   - Cache trong /tmp (warm invocations)
   - Cache config trong memory
   - Dùng ElastiCache/DynamoDB DAX cho dữ liệu thường xuyên đọc

4. Batch processing:
   - SQS batch size 10 thay vì 1
   - DynamoDB BatchGetItem thay vì GetItem × N
```

### Parallel Invocations (Gọi Song Song)

```python
import asyncio
import aioboto3

async def handler_async(event, context):
    session = aioboto3.Session()
    
    async with session.resource('dynamodb') as dynamodb:
        table = await dynamodb.Table('Users')
        
        # Gọi song song thay vì tuần tự
        user_ids = event['userIds']
        
        tasks = [
            table.get_item(Key={'userId': uid})
            for uid in user_ids
        ]
        
        results = await asyncio.gather(*tasks)
        users = [r.get('Item') for r in results]
        
        return {'users': users}

def handler(event, context):
    return asyncio.run(handler_async(event, context))
```

```javascript
// Node.js — Promise.all cho parallel calls
exports.handler = async (event) => {
    const userIds = event.userIds;
    
    // ❌ Tuần tự: 5 users × 100ms = 500ms
    // for (const id of userIds) { await getUser(id); }
    
    // ✅ Song song: max(100ms, 100ms, ...) = ~100ms
    const users = await Promise.all(
        userIds.map(id => getUser(id))
    );
    
    return { users };
};
```

### Function Consolidation (Gộp Functions)

```
Đừng micro-split quá mức:
❌ Tệ:
  - get-user-name function
  - get-user-email function
  - get-user-address function
  → 3 cold starts, 3 API GW configs, 3 IAM roles...

✅ Tốt:
  - user-service function (xử lý tất cả /users/* routes)
  → 1 cold start, 1 API GW, 1 IAM role

Nguyên tắc: Nhóm theo domain/service, không nhóm theo operation
```

---

## ✅ Production Checklist — Danh Sách Kiểm Tra

### Trước Khi Deploy

```
□ Function có execution role với least privilege không?
□ Secrets được lưu trong Secrets Manager/Parameter Store (không phải env var)?
□ Environment variables có sensitive data không?
□ Timeout được set hợp lý (không quá thấp, không quá cao)?
□ Memory được chọn dựa trên profiling thực tế?
□ Dead Letter Queue được cấu hình cho async functions?
□ CloudWatch Alarms cho Errors, Throttles, Duration?
□ Structured logging được implement?
□ Input validation được implement?
□ Versioning và Aliases được cấu hình?
```

### Reliability (Độ Tin Cậy)

```
□ Idempotency (Lũy Đẳng Tính): Gọi nhiều lần cùng kết quả?
  → Dùng idempotency key với DynamoDB conditional writes
  
□ Error handling phân biệt retryable vs non-retryable errors?

□ DLQ/Destination OnFailure cho async invocations?

□ SQS visibility timeout > Lambda timeout?
  → Nếu Lambda timeout = 30s, SQS visibility = 60s+
  
□ Function tested với partial failures?
```

### Performance

```
□ Cold start time được đo và chấp nhận?
  → < 1s cho API, < 5s cho background jobs
  
□ Provisioned Concurrency cho latency-critical APIs?

□ AWS SDK clients initialized outside handler?

□ Dependencies tối ưu (loại bỏ unused packages)?

□ Lambda Power Tuning đã chạy để tìm optimal memory?

□ VPC setup đúng nếu cần (private subnet + NAT/VPC Endpoints)?
```

### Observability (Khả Năng Quan Sát)

```
□ X-Ray tracing được enable?
□ Custom metrics cho business KPIs?
□ Log retention policy được set (không để mặc định "never expire")?
□ CloudWatch Dashboard cho function?
□ Alarms cho p99 duration, error rate, throttles?
□ Lambda Insights được enable?
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Giải thích các cách giảm cold start của Lambda?**
> (1) Chọn runtime nhanh như Go/Python thay vì Java (nếu không dùng SnapStart). (2) Giảm package size: loại bỏ unused dependencies, dùng tree-shaking. (3) Đặt init code (SDK clients, config) ngoài handler. (4) Dùng Provisioned Concurrency cho latency-critical APIs. (5) Với Java: dùng SnapStart để giảm cold start từ 2s xuống ~200ms.

**Q: Lambda trong VPC có cold start chậm hơn không?**
> Trước 2019: Có, vì Lambda phải tạo ENI mới mỗi cold start (~10s). Từ 2019 trở đi: AWS thay đổi cách quản lý ENI — Lambda tạo shared ENI pool trong subnet. Cold start trong VPC chỉ thêm ~500ms, chấp nhận được cho hầu hết use cases.

**Q: Khi nào Lambda cần trong VPC?**
> Khi cần kết nối đến resources trong private VPC: RDS, Aurora (không có public endpoint), ElastiCache, internal microservices, các EC2 instances trong private subnet. Không cần VPC khi chỉ gọi public AWS APIs (DynamoDB, S3 qua public endpoints) hoặc external HTTP APIs.

**Q: SnapStart giải quyết vấn đề gì và khi nào dùng?**
> SnapStart giải quyết JVM cold start problem — Java cold start thường 1-2 giây do JVM initialization. SnapStart tạo snapshot sau khi JVM và application đã initialized, restore từ snapshot mỗi cold start (~200ms). Dùng khi: Java 11+ functions có cold start vấn đề, latency-sensitive APIs, không muốn trả phí Provisioned Concurrency. Cần dùng CRaC hooks để manage stateful resources (DB connections) qua snapshot/restore cycle.

**Q: Làm thế nào handle database connections đúng cách với Lambda?**
> (1) RDS Proxy: Tốt nhất — Lambda kết nối đến Proxy, Proxy pool connections với RDS, Lambda có thể scale mạnh mà không overwhelm DB. (2) Reserved Concurrency: Giới hạn concurrent Lambda = giới hạn DB connections. (3) Connection init ngoài handler (reuse across warm starts). (4) Dùng serverless-compatible DB: DynamoDB (không connection limit), Aurora Serverless v2. (5) Implement connection timeout và retry logic.

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn thành
**Chủ Đề Tiếp Theo:** [../04-containers-ecs/README.md](../04-containers-ecs/README.md) — ECS & Container Orchestration
