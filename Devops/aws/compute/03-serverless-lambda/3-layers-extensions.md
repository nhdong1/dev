# 📦 Lambda Layers & Extensions — Chia Sẻ Code & Tích Hợp Công Cụ

> Lambda Layers (Lớp Lambda) cho phép chia sẻ dependencies, custom runtimes, và utilities giữa nhiều functions. Lambda Extensions (Mở Rộng Lambda) cho phép tích hợp monitoring, security, và observability tools vào execution environment. Container Image mang lại linh hoạt tối đa.

## 📚 Mục Lục

1. [Lambda Layers — Lớp Chia Sẻ](#lambda-layers--lớp-chia-sẻ)
2. [Tạo và Sử Dụng Layer](#tạo-và-sử-dụng-layer)
3. [Lambda Extensions — Mở Rộng](#lambda-extensions--mở-rộng)
4. [Container Images — Hình Ảnh Container](#container-images--hình-ảnh-container)
5. [So Sánh Packaging Options](#so-sánh-packaging-options)
6. [Best Practices](#best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📚 Lambda Layers — Lớp Chia Sẻ

### Layer Là Gì?

Layer là một **ZIP archive** chứa libraries, custom runtimes, data, hoặc configuration files. Layer được mount vào `/opt` trong execution environment và chia sẻ giữa nhiều functions.

```
Không có Layer:
┌─────────────────────────────────────────────────────────┐
│  Function A ZIP                                         │
│  ├── function_a.py    (code của bạn)                    │
│  ├── boto3/           (500 KB)                          │
│  ├── pandas/          (15 MB)                           │
│  └── numpy/           (30 MB)                          │
│  Tổng: 45 MB                                            │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  Function B ZIP                                         │
│  ├── function_b.py                                      │
│  ├── boto3/           (DUP — 500 KB)                    │
│  ├── pandas/          (DUP — 15 MB)                     │
│  └── numpy/           (DUP — 30 MB)                    │
│  Tổng: 45 MB (lãng phí!)                               │
└─────────────────────────────────────────────────────────┘

Với Layer:
┌────────────────────────┐    ┌───────────────────────┐
│  Function A ZIP (3 KB) │    │  Function B ZIP (2 KB) │
│  └── function_a.py     │    │  └── function_b.py     │
└─────────┬──────────────┘    └──────────┬─────────────┘
          │                              │
          └──────────┬───────────────────┘
                     │ (shared layer)
          ┌──────────▼───────────┐
          │  Data Science Layer  │
          │  ├── pandas/         │
          │  ├── numpy/          │
          │  └── scipy/          │
          │  (45 MB, dùng chung) │
          └──────────────────────┘
```

### Giới Hạn Layer

| Giới Hạn                           | Giá Trị                          |
| ---------------------------------- | -------------------------------- |
| Layers per function                | Tối đa 5 layers                  |
| Unzipped size (code + layers)      | 250 MB tổng                      |
| Layer ZIP file size                | 50 MB (trực tiếp), 250 MB (qua S3) |
| Layer versions                     | Không giới hạn                   |
| Layer có thể dùng cross-account    | ✅ Có (với resource policy)       |

### Cấu Trúc Thư Mục Layer

Layer được mount vào `/opt/` — cần đặt đúng cấu trúc thư mục theo runtime:

```
Python:
/opt/
└── python/               ← Python thêm thư mục này vào sys.path
    ├── pandas/
    ├── numpy/
    └── my_utils.py

Node.js:
/opt/
└── nodejs/
    └── node_modules/     ← Node.js thêm vào module search path
        ├── lodash/
        └── axios/

Java:
/opt/
└── java/
    └── lib/              ← Add to classpath
        └── mylib.jar

Custom runtime:
/opt/
├── bootstrap             ← Custom runtime executable
└── lib/
    └── mylib.so
```

---

## 🛠️ Tạo và Sử Dụng Layer

### Tạo Python Layer

```bash
# Bước 1: Cài dependencies vào đúng cấu trúc thư mục
mkdir -p layer/python
pip install \
  pandas==2.1.0 \
  numpy==1.26.0 \
  requests==2.31.0 \
  -t layer/python/

# Bước 2: Nén lại
cd layer
zip -r ../data-science-layer.zip python/
cd ..

# Bước 3: Publish layer
aws lambda publish-layer-version \
  --layer-name data-science-layer \
  --description "Pandas, NumPy, Requests cho data processing" \
  --zip-file fileb://data-science-layer.zip \
  --compatible-runtimes python3.11 python3.12 \
  --compatible-architectures x86_64 arm64

# Output: Layer ARN
# arn:aws:lambda:us-east-1:123456789:layer:data-science-layer:1
```

### Tạo Utility Layer (Shared Code)

```bash
# Structure cho shared utilities
mkdir -p layer/python/utils
cat > layer/python/utils/__init__.py << 'EOF'
from .db import get_db_connection
from .auth import verify_token
from .logging import setup_logger
EOF

# Tạo từng utility module
cat > layer/python/utils/logging.py << 'EOF'
import logging
import json
import os

def setup_logger(name):
    logger = logging.getLogger(name)
    logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))
    return logger
EOF
```

### Gắn Layer vào Function

```bash
# Khi tạo function mới
aws lambda create-function \
  --function-name my-data-function \
  --runtime python3.12 \
  --handler index.handler \
  --role arn:aws:iam::123456789:role/lambda-role \
  --zip-file fileb://function.zip \
  --layers \
    arn:aws:lambda:us-east-1:123456789:layer:data-science-layer:1 \
    arn:aws:lambda:us-east-1:123456789:layer:utils-layer:3

# Hoặc update function đang chạy
aws lambda update-function-configuration \
  --function-name my-data-function \
  --layers \
    arn:aws:lambda:us-east-1:123456789:layer:data-science-layer:1
```

### Sử Dụng Layer Trong Code

```python
# Layer được mount vào /opt/python — import bình thường!
import pandas as pd
import numpy as np
from utils import setup_logger  # Từ utils layer
from utils.db import get_db_connection  # Từ utils layer

logger = setup_logger(__name__)

def handler(event, context):
    logger.info("Processing data")
    
    db = get_db_connection()
    data = db.query("SELECT * FROM orders LIMIT 1000")
    
    df = pd.DataFrame(data)
    result = df.groupby('product_id')['amount'].sum()
    
    return {
        'statusCode': 200,
        'body': result.to_json()
    }
```

### AWS Public Layers (Layer Công Khai Của AWS)

AWS cung cấp sẵn một số layers phổ biến:

```bash
# AWS Parameter and Secrets Lambda Extension
arn:aws:lambda:us-east-1:177933569100:layer:AWS-Parameters-and-Secrets-Lambda-Extension:11

# AWS AppConfig Lambda Extension
arn:aws:lambda:us-east-1:027255383542:layer:AWS-AppConfig-Extension:128

# Lambda Powertools for Python (Bộ Công Cụ Phát Triển Lambda)
arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV2:67
```

### Lambda Powertools — Bộ Công Cụ Phát Triển

Lambda Powertools (bây giờ là **AWS Lambda Powertools**) là library giúp implement production best practices dễ dàng:

```python
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

# Setup một lần ở module level
logger = Logger(service="user-service")
tracer = Tracer(service="user-service")
metrics = Metrics(namespace="MyApp", service="user-service")

@logger.inject_lambda_context(log_event=True)  # Auto log event + context
@tracer.capture_lambda_handler                  # Auto X-Ray tracing
@metrics.log_metrics(capture_cold_start_metric=True)  # Auto cold start metric
def handler(event: dict, context: LambdaContext) -> dict:
    # Logger tự động thêm request_id, function_name vào mỗi log
    logger.info("Processing user request", extra={"user_id": event.get("userId")})
    
    with tracer.capture_method:
        user = get_user(event["userId"])
    
    # Emit custom metric
    metrics.add_metric(name="UserFetched", unit=MetricUnit.Count, value=1)
    
    return {"statusCode": 200, "body": user}
```

---

## 🔌 Lambda Extensions — Mở Rộng

### Extensions Là Gì?

Extensions là các process chạy **song song** với Lambda function trong cùng execution environment. Chúng cho phép tích hợp các công cụ external mà không cần thay đổi function code:

```
Execution Environment
┌──────────────────────────────────────────────────────┐
│                                                      │
│  ┌────────────────────┐  ┌──────────────────────┐   │
│  │  Lambda Function   │  │  Lambda Extension    │   │
│  │  (Your Code)       │  │  (External Tool)     │   │
│  │                    │  │                      │   │
│  │  handler()         │  │  - Datadog Agent     │   │
│  │  business logic    │  │  - Dynatrace         │   │
│  │                    │  │  - New Relic         │   │
│  │                    │  │  - HashiCorp Vault   │   │
│  └────────────────────┘  └──────────────────────┘   │
│                │                       │             │
│           Telemetry API          Extension API       │
│                └───────────────────────┘             │
│                            │                         │
│                    Lambda Service                    │
└──────────────────────────────────────────────────────┘
```

### Extension Types (Loại Mở Rộng)

#### 1. Internal Extensions (Mở Rộng Bên Trong)

Chạy **trong** runtime process — không phải process riêng:
- Wrapper scripts
- Language-specific (ví dụ: JAVA_TOOL_OPTIONS)

#### 2. External Extensions (Mở Rộng Bên Ngoài)

Chạy như **process riêng** song song với function:
- Monitoring agents (Datadog, New Relic, Dynatrace)
- Security tools (HashiCorp Vault, Secrets Manager cache)
- Telemetry collectors

### Extension Lifecycle (Vòng Đời Extension)

```
Lambda Lifecycle với Extension:

INIT Phase:
1. Extension Init       ← Extension registers với Lambda service
2. Runtime Init         ← Python/Node/Java runtime khởi động  
3. Function Init        ← Global code chạy

INVOKE Phase:
4. Extension: Invoke event notification
5. Function: handler() runs
6. Function: Returns response
7. Extension: Post-invoke processing

SHUTDOWN Phase:
8. Extension: Shutdown signal (cleanup, flush buffers)
9. Environment released
```

### Datadog Extension — Ví Dụ Thực Tế

```bash
# Thêm Datadog Extension qua Layer
aws lambda update-function-configuration \
  --function-name my-function \
  --layers \
    arn:aws:lambda:us-east-1:464622532012:layer:Datadog-Extension:49 \
    arn:aws:lambda:us-east-1:464622532012:layer:Datadog-Python312:6 \
  --environment Variables="{
    DD_API_KEY_SECRET_ARN=arn:aws:secretsmanager:...:dd-api-key,
    DD_SITE=datadoghq.com,
    DD_SERVERLESS_LOGS_ENABLED=true,
    DD_TRACE_ENABLED=true
  }"
```

### Lambda Telemetry API — Thu Thập Dữ Liệu Giám Sát

Extensions có thể subscribe vào **Telemetry API** để nhận logs, metrics, và traces trực tiếp từ Lambda:

```python
# Extension code (chạy như subprocess)
import http.server
import json
import urllib.request

TELEMETRY_LISTENER_PORT = 4243

class TelemetryHandler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        body = self.rfile.read(content_length)
        events = json.loads(body)
        
        for event in events:
            event_type = event['type']
            
            if event_type == 'function':
                # Log line từ function
                process_function_log(event['record'])
            elif event_type == 'platform.report':
                # Lambda platform metrics
                flush_metrics_to_backend(event['record'])
        
        self.send_response(200)
        self.end_headers()

def register_extension():
    """Đăng ký extension với Lambda service"""
    response = urllib.request.urlopen(
        urllib.request.Request(
            f"http://{os.environ['AWS_LAMBDA_RUNTIME_API']}/2020-01-01/extension/register",
            data=json.dumps({"events": ["INVOKE", "SHUTDOWN"]}).encode(),
            headers={"Lambda-Extension-Name": "my-telemetry-extension"},
            method="POST"
        )
    )
    return response.headers["Lambda-Extension-Identifier"]
```

---

## 🐳 Container Images — Hình Ảnh Container

### Khi Nào Dùng Container Image?

| Trường Hợp                              | ZIP Deployment | Container Image |
| --------------------------------------- | -------------- | --------------- |
| Function code < 50 MB                  | ✅ Ưu tiên     | Có thể          |
| Dependencies lớn (ML model, FFmpeg)     | ❌ Khó          | ✅ Phù hợp     |
| Muốn test locally như production        | Hạn chế        | ✅ `docker run` |
| Cần custom OS packages (libvips, poppler)| ❌              | ✅ apt-get      |
| Image size > 250 MB                     | ❌ Không được  | ✅ Đến 10 GB   |
| Muốn reuse Dockerfile từ ECS/EKS        | ❌              | ✅ Tái sử dụng  |

### Tạo Container Image Lambda

#### Sử Dụng AWS Base Images

```dockerfile
# Python Lambda base image
FROM public.ecr.aws/lambda/python:3.12

# Copy requirements và cài packages
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY src/ .

# Set handler (file.function)
CMD ["app.handler"]
```

#### Build với ML Model (Mô Hình Machine Learning)

```dockerfile
FROM public.ecr.aws/lambda/python:3.12

# Cài system dependencies
RUN yum install -y \
    libgomp \
    && yum clean all

# Cài Python packages
COPY requirements.txt .
RUN pip install --no-cache-dir \
    torch==2.1.0+cpu \
    transformers==4.35.0 \
    --extra-index-url https://download.pytorch.org/whl/cpu

# Copy model (pre-downloaded để tránh download lúc cold start)
COPY models/ /opt/ml/models/

# Copy code
COPY src/ .

CMD ["inference.handler"]
```

#### Sử Dụng Custom Base Image

```dockerfile
# Bắt đầu từ image tùy chỉnh
FROM ubuntu:22.04

# Cài packages
RUN apt-get update && apt-get install -y \
    python3.12 \
    python3-pip \
    ffmpeg \
    poppler-utils \
    && rm -rf /var/lib/apt/lists/*

# Cài Lambda Runtime Interface Client (Giao Diện Thời Gian Chạy Lambda)
RUN pip3 install awslambdaric

# Copy code
WORKDIR /app
COPY src/ .
RUN pip3 install -r requirements.txt

# Entry point phải là Lambda RIC
ENTRYPOINT ["/usr/local/bin/python3", "-m", "awslambdaric"]
CMD ["app.handler"]
```

### Build & Push lên ECR

```bash
# Bước 1: Tạo ECR repository
aws ecr create-repository \
  --repository-name my-lambda-app \
  --image-scanning-configuration scanOnPush=true

# Bước 2: Login vào ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Bước 3: Build image
docker build -t my-lambda-app .

# Bước 4: Tag image
docker tag my-lambda-app:latest \
  123456789.dkr.ecr.us-east-1.amazonaws.com/my-lambda-app:latest

# Bước 5: Push lên ECR
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-lambda-app:latest

# Bước 6: Deploy Lambda với container image
aws lambda create-function \
  --function-name my-ml-function \
  --package-type Image \
  --code ImageUri=123456789.dkr.ecr.us-east-1.amazonaws.com/my-lambda-app:latest \
  --role arn:aws:iam::123456789:role/lambda-role \
  --timeout 60 \
  --memory-size 3008
```

### Test Container Locally (Kiểm Tra Locally)

```bash
# Chạy container local với Lambda RIE (Runtime Interface Emulator — Giả Lập Thời Gian Chạy)
docker run -p 9000:8080 \
  -e AWS_ACCESS_KEY_ID=test \
  -e AWS_SECRET_ACCESS_KEY=test \
  -e AWS_REGION=us-east-1 \
  my-lambda-app:latest

# Gọi thử function local
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d '{"pathParameters": {"userId": "123"}}'
```

### Multi-stage Build (Build Đa Giai Đoạn) — Tối Ưu Image Size

```dockerfile
# Stage 1: Build stage (Giai đoạn build)
FROM python:3.12-slim AS builder

WORKDIR /build

COPY requirements.txt .
RUN pip install --no-cache-dir --target /build/packages -r requirements.txt

# Stage 2: Runtime stage (Giai đoạn chạy) — chỉ copy những gì cần
FROM public.ecr.aws/lambda/python:3.12

# Copy chỉ installed packages từ build stage
COPY --from=builder /build/packages /opt/python/

# Copy source code
COPY src/ .

CMD ["app.handler"]
```

---

## ⚖️ So Sánh Packaging Options

| Tiêu Chí                    | ZIP (Direct)     | ZIP (via S3)     | Container Image  |
| --------------------------- | ---------------- | ---------------- | ---------------- |
| Kích thước tối đa           | 50 MB            | 250 MB           | 10 GB            |
| Cold start speed            | Nhanh nhất       | Nhanh            | Chậm hơn (~1-2s) |
| Build time                  | Rất nhanh        | Nhanh            | Chậm (docker build)|
| Local testing               | Lambda RIE       | Lambda RIE       | `docker run`     |
| Dependency management       | Lambda Layers    | Lambda Layers    | Dockerfile       |
| Custom OS packages          | ❌               | ❌               | ✅ apt/yum      |
| Reproductibility            | Khó              | Khó              | ✅ Hoàn toàn     |
| ECR scan (quét bảo mật)     | ❌               | ❌               | ✅ Tự động       |

### Khi Nào Dùng Cái Gì?

```
Function code < 10 MB, dependencies nhỏ?
  → ZIP Direct — đơn giản nhất

Dependencies 10-250 MB?
  → ZIP via S3 với Lambda Layers

Cần custom OS packages (FFmpeg, Poppler)?
  → Container Image

ML model lớn hoặc dependencies > 250 MB?
  → Container Image (up to 10 GB)

Muốn reuse CI/CD pipeline với Docker?
  → Container Image
```

---

## ✅ Best Practices

### Layers

```
1. Tách dependencies theo nhóm logic:
   - data-science-layer: pandas, numpy, scipy
   - utils-layer: shared code, helpers
   - vendors-layer: third-party SDKs

2. Version layers cẩn thận:
   - Đừng update layer version mà không test trước
   - Dùng alias để point đến layer version cụ thể

3. Chia sẻ layers cross-account qua Resource Policy:
   aws lambda add-layer-version-permission \
     --layer-name my-layer \
     --version-number 1 \
     --statement-id cross-account \
     --action lambda:GetLayerVersion \
     --principal 987654321  # Other AWS account ID

4. Đừng gộp quá nhiều vào 1 layer:
   - Mỗi function chỉ cần một phần nhỏ → lãng phí
   - Layer size lớn → cold start chậm hơn
```

### Container Images

```
1. Dùng AWS base images khi có thể:
   public.ecr.aws/lambda/python:3.12
   → Đã tối ưu cho Lambda, có Lambda RIC sẵn

2. Tối ưu image size:
   - Multi-stage build
   - .dockerignore để exclude files không cần
   - --no-cache-dir khi pip install

3. Pre-download models/assets vào image:
   → Tránh download lúc cold start
   → Tốn storage ECR nhưng fast cold start

4. Scan images thường xuyên:
   aws ecr start-image-scan \
     --repository-name my-lambda-app \
     --image-id imageTag=latest

5. Pin image versions:
   FROM public.ecr.aws/lambda/python:3.12.2024.01.01  # Pin cụ thể
   Không dùng: FROM public.ecr.aws/lambda/python:latest
```

### Extensions

```
1. Chỉ dùng extensions từ vendors tin cậy (Datadog, Dynatrace, New Relic)
2. Extensions thêm overhead vào Init và Shutdown phases
3. Đo impact của extension lên cold start trước khi production
4. Dùng Telemetry API thay vì intercept logs qua CloudWatch subscription
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Layer khác Extension thế nào?**
> Layer là ZIP archive chứa code/libraries, được mount vào `/opt` và import trong function code. Extension là process chạy song song với function trong execution environment, dùng cho monitoring/security tools. Layer phục vụ code sharing; Extension phục vụ observability và integration tools.

**Q: Tại sao Container Image có cold start chậm hơn ZIP?**
> Container image phải được pulled từ ECR và initialized, bao gồm cả filesystem setup. ZIP chỉ cần unzip nhỏ hơn nhiều. Tuy nhiên, AWS cache container layers nên sau lần đầu tiên cold start sẽ nhanh hơn đáng kể. Dùng Provisioned Concurrency để loại bỏ cold start nếu cần.

**Q: Lambda có thể chạy bao nhiêu layers cùng lúc?**
> Tối đa 5 layers per function. Tổng uncompressed size của function code + tất cả layers không được vượt quá 250 MB. Nếu cần nhiều hơn, dùng Container Image (up to 10 GB).

**Q: Khi nào nên tách code vào Layer thay vì để trong function ZIP?**
> Khi: (1) Nhiều functions cùng dùng chung library/code, (2) Library thay đổi ít hơn so với business logic (deploy nhanh hơn), (3) Library lớn (pandas, numpy) cần tách ra để function ZIP nhỏ hơn. Không nên tách khi: Library nhỏ và chỉ dùng trong một function.

**Q: Container Image mang lại lợi ích gì so với ZIP?**
> (1) Size lớn hơn (10 GB vs 250 MB) — phù hợp ML models, (2) Custom OS packages với apt/yum, (3) Reproduce chính xác environment với docker run local, (4) Reuse Dockerfile từ ECS/EKS, (5) ECR image scanning tự động, (6) Multi-stage build để tối ưu image size.

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [4-concurrency-throttling.md](./4-concurrency-throttling.md) — Concurrency & Throttling
