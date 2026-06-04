# Mẫu Kiến Trúc ML — ML Architecture Patterns

> Các mẫu kiến trúc (architecture patterns) chuẩn cho hệ thống ML production trên AWS: từ Batch Inference (Suy Luận Hàng Loạt), Online Serving (Phục Vụ Trực Tuyến), Shadow Deployment (Triển Khai Bóng), Canary Release (Phát Hành Canary) đến Multi-region Deployment (Triển Khai Đa Vùng)

## 📋 Mục Lục

1. [Tổng Quan Architecture Patterns](#tổng-quan-architecture-patterns)
2. [Batch Inference Architecture](#batch-inference-architecture)
3. [Real-time Online Serving](#real-time-online-serving)
4. [Async Inference Architecture](#async-inference-architecture)
5. [Shadow Deployment](#shadow-deployment)
6. [Canary Release & Blue/Green Deployment](#canary-release--bluegreen-deployment)
7. [A/B Testing với Multi-variant Endpoint](#ab-testing-với-multi-variant-endpoint)
8. [Multi-model & Multi-container Patterns](#multi-model--multi-container-patterns)
9. [Event-driven ML Architecture](#event-driven-ml-architecture)
10. [Multi-region ML Architecture](#multi-region-ml-architecture)
11. [Chọn Pattern Phù Hợp](#chọn-pattern-phù-hợp)

---

## Tổng Quan Architecture Patterns

### Decision Framework — Khung Ra Quyết Định

```
Câu hỏi 1: Latency yêu cầu là bao nhiêu?
├── < 100ms                   → Real-time Endpoint
├── 100ms - 5s                → Real-time hoặc Async Inference
├── 5s - 15 phút              → Async Inference
└── Giờ/ngày (batch)          → Batch Transform

Câu hỏi 2: Traffic pattern như thế nào?
├── Liên tục, dự đoán được    → Real-time + Auto Scaling
├── Burst (đột biến)           → Async Inference hoặc Serverless
├── Định kỳ (nightly/weekly)  → Batch Transform
└── Thấp, không đều           → Serverless Inference

Câu hỏi 3: Đang thay model hay thêm model mới?
├── Thay model (same schema)  → Blue/Green hoặc Canary
├── Test model mới             → Shadow Deployment → Canary → Blue/Green
├── Nhiều model variants       → A/B Testing với Multi-variant Endpoint
└── Nhiều models khác nhau    → Multi-Model Endpoint (MME)
```

---

## Batch Inference Architecture

### Khi Nào Dùng Batch Inference?

**Batch Inference** (Suy Luận Hàng Loạt) phù hợp khi:
- Không cần kết quả ngay lập tức (chạy offline)
- Cần xử lý hàng triệu records
- Kết quả được lưu vào database để dùng sau
- Ví dụ: Tính recommendation score cho tất cả users đêm hôm trước, chạy fraud detection batch hàng đêm

### Kiến Trúc Batch Inference Chuẩn

```
┌─────────────────────────────────────────────────────────────┐
│                    Batch Inference Pipeline                   │
│                                                               │
│  S3 (Input)                                                   │
│  raw_data.csv                                                 │
│       │                                                       │
│       ▼                                                       │
│  SageMaker Processing Job                                     │
│  (Tiền xử lý data)                                            │
│  - Feature engineering                                        │
│  - Normalize, encode                                          │
│       │                                                       │
│       ▼                                                       │
│  S3 (Processed)           SageMaker Model                     │
│  processed_data.csv ──────►                                   │
│                            Batch Transform Job                │
│                            - ml.m5.4xlarge × 5 instances      │
│                            - MaxConcurrentTransforms: 10      │
│                            - BatchStrategy: MultiRecord       │
│                       ─────►                                  │
│                       │    S3 (Output)                        │
│                       │    predictions.csv.out                │
│                       │                                       │
│                       ▼                                       │
│  Lambda → DynamoDB / Redshift / RDS                           │
│  (Load predictions vào database để app query)                 │
└─────────────────────────────────────────────────────────────┘
```

### Triển Khai Batch Transform Job

```python
import sagemaker
from sagemaker.transformer import Transformer

# Tạo Transformer từ model đã trained
transformer = Transformer(
    model_name="my-recommendation-model",
    instance_count=5,                    # Chạy song song 5 instances
    instance_type="ml.m5.4xlarge",
    strategy="MultiRecord",              # Gửi nhiều records trong 1 request
    max_concurrent_transforms=10,        # 10 requests song song per instance
    max_payload=6,                       # Max 6MB per request
    assemble_with="Line",                # Kết quả output mỗi record 1 dòng
    output_path="s3://my-bucket/output/",
    accept="text/csv",
)

# Chạy batch transform
transformer.transform(
    data="s3://my-bucket/input/",
    data_type="S3Prefix",
    content_type="text/csv",
    split_type="Line",                   # Split input theo từng dòng
    join_source="Input",                 # Ghép input và output trong kết quả
    compression_type="None",
)

transformer.wait()  # Chờ hoàn thành (có thể dùng pipeline step thay thế)

print(f"Output tại: {transformer.output_path}")
```

### Lên Lịch Batch Job với EventBridge

```python
import boto3

events = boto3.client("events")

# Chạy batch inference mỗi đêm lúc 2:00 AM UTC
events.put_rule(
    Name="nightly-batch-inference",
    ScheduleExpression="cron(0 2 * * ? *)",  # 2:00 AM UTC mỗi ngày
    State="ENABLED",
)

# Target: Lambda function kích hoạt SageMaker Batch Transform
events.put_targets(
    Rule="nightly-batch-inference",
    Targets=[
        {
            "Id": "trigger-batch-inference",
            "Arn": "arn:aws:lambda:us-east-1:123456789:function:trigger-batch-job",
            "Input": json.dumps({
                "input_s3": "s3://my-bucket/daily-data/",
                "output_s3": "s3://my-bucket/predictions/",
                "model_name": "recommendation-model-v3",
            }),
        }
    ],
)
```

---

## Real-time Online Serving

### Kiến Trúc Real-time Serving Chuẩn

```
                           Internet
                               │
                               ▼
                    ┌─────────────────┐
                    │   API Gateway   │  Rate limiting, Auth, CORS
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Lambda Function │  Request validation, Auth
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────────────────────┐
                    │   SageMaker Real-time Endpoint   │
                    │                                   │
                    │  ┌───────┐  ┌───────┐  ┌───────┐│
                    │  │Inst 1 │  │Inst 2 │  │Inst 3 ││  Auto Scaling
                    │  └───────┘  └───────┘  └───────┘│
                    └──────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  DynamoDB Cache │  Cache predictions để tránh repeat calls
                    └─────────────────┘
```

### Auto Scaling cho Real-time Endpoint

```python
import boto3

autoscaling = boto3.client("application-autoscaling")

endpoint_name = "my-production-endpoint"
variant_name = "AllTraffic"

# Đăng ký resource với Application Auto Scaling
autoscaling.register_scalable_target(
    ServiceNamespace="sagemaker",
    ResourceId=f"endpoint/{endpoint_name}/variant/{variant_name}",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    MinCapacity=1,       # Tối thiểu 1 instance
    MaxCapacity=10,      # Tối đa 10 instances
)

# Policy: Scale out khi InvocationsPerInstance > 100/phút
autoscaling.put_scaling_policy(
    PolicyName="scale-out-on-high-traffic",
    ServiceNamespace="sagemaker",
    ResourceId=f"endpoint/{endpoint_name}/variant/{variant_name}",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    PolicyType="TargetTrackingScaling",
    TargetTrackingScalingPolicyConfiguration={
        "TargetValue": 100.0,               # Target: 100 invocations/phút/instance
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance",
        },
        "ScaleOutCooldown": 60,             # Chờ 60s trước khi scale out lần nữa
        "ScaleInCooldown": 300,             # Chờ 5 phút trước khi scale in
    },
)
```

### Caching với ElastiCache

```python
import redis
import hashlib

redis_client = redis.Redis(host="my-elasticache.abc.0001.use1.cache.amazonaws.com")

def predict_with_cache(input_data: dict, endpoint_name: str, ttl_seconds=3600):
    # Tạo cache key từ input data
    cache_key = f"ml:{hashlib.md5(json.dumps(input_data, sort_keys=True).encode()).hexdigest()}"

    # Kiểm tra cache
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache hit — không cần gọi endpoint

    # Cache miss — gọi SageMaker endpoint
    runtime = boto3.client("sagemaker-runtime")
    response = runtime.invoke_endpoint(
        EndpointName=endpoint_name,
        ContentType="application/json",
        Body=json.dumps(input_data),
    )
    prediction = json.loads(response["Body"].read())

    # Lưu vào cache với TTL (Time-to-Live — Thời Gian Sống)
    redis_client.setex(cache_key, ttl_seconds, json.dumps(prediction))

    return prediction
```

---

## Async Inference Architecture

### Khi Nào Dùng Async Inference?

**Async Inference** (Suy Luận Bất Đồng Bộ) phù hợp:
- Request cần xử lý lâu (30 giây - 15 phút), ví dụ: phân tích video dài
- Input payload lớn (> 6MB), ví dụ: document dài, ảnh độ phân giải cao
- Không cần đợi kết quả ngay

```
Async Inference Flow:
1. Client gửi request → SageMaker enqueue → Trả về request ID ngay lập tức
2. SageMaker xử lý request trong background
3. Kết quả ghi vào S3 output location
4. (Tùy chọn) SNS notification khi xong
5. Client poll S3 location hoặc subscribe SNS để nhận kết quả
```

### Triển Khai Async Inference Endpoint

```python
from sagemaker.async_inference import AsyncInferenceConfig

# Cấu hình Async Inference
async_config = AsyncInferenceConfig(
    output_path="s3://my-bucket/async-output/",
    max_concurrent_invocations_per_instance=4,
    notification_config={
        "SuccessTopic": "arn:aws:sns:us-east-1:123456789:async-success",
        "ErrorTopic": "arn:aws:sns:us-east-1:123456789:async-error",
    },
)

# Deploy với Async config
async_predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.g4dn.xlarge",
    async_inference_config=async_config,
)

# Gửi request async
response = async_predictor.predict_async(
    data=large_video_bytes,
    input_path="s3://my-bucket/async-input/video-001.mp4",
)
output_path = response.output_path  # S3 path sẽ chứa kết quả

# Poll kết quả (hoặc dùng SNS để được notify)
import time
s3 = boto3.client("s3")

while True:
    try:
        result = s3.get_object(Bucket="my-bucket", Key=output_path.replace("s3://my-bucket/", ""))
        prediction = json.loads(result["Body"].read())
        break
    except s3.exceptions.NoSuchKey:
        time.sleep(5)  # Chờ 5s rồi poll lại
```

---

## Shadow Deployment

### Khái Niệm Shadow Deployment

**Shadow Deployment** (Triển Khai Bóng) cho phép test model mới bằng cách:
1. Copy 100% production traffic đến model mới (shadow model)
2. Production model vẫn trả lời cho user
3. Shadow model xử lý cùng request nhưng kết quả bị drop (không trả về user)
4. So sánh predictions và performance giữa hai models

```
Production Traffic:
User Request → [Endpoint] ─── 100% ──► Production Model → User Response
                            └── 100% ──► Shadow Model     → /dev/null (bỏ)

Thu thập metrics:
- Latency của shadow model
- Prediction differences (% requests có kết quả khác)
- Error rate của shadow model
```

### SageMaker Shadow Testing

```python
sm = boto3.client("sagemaker")

# Cấu hình Shadow Testing
response = sm.create_inference_experiment(
    Name="shadow-test-model-v2",
    Type="ShadowMode",           # Shadow Testing mode
    RoleArn=role_arn,
    EndpointName="production-endpoint",

    # Shadow variant — model cần test
    ShadowModeConfig={
        "SourceModelVariantName": "ProductionVariant",  # Copy traffic từ đây
        "ShadowModelVariants": [
            {
                "ShadowModelVariantName": "ShadowVariant",
                "SamplingPercentage": 100,  # Copy 100% traffic
            }
        ],
    },

    # Cấu hình production model
    ModelVariants=[
        {
            "ModelName": "production-model",
            "VariantName": "ProductionVariant",
            "InfrastructureConfig": {
                "InfrastructureType": "RealTimeInference",
                "RealTimeInferenceConfig": {
                    "InstanceType": "ml.m5.xlarge",
                    "InstanceCount": 2,
                },
            },
        },
        {
            "ModelName": "new-model-v2",           # Model cần test
            "VariantName": "ShadowVariant",
            "InfrastructureConfig": {
                "InfrastructureType": "RealTimeInference",
                "RealTimeInferenceConfig": {
                    "InstanceType": "ml.m5.xlarge",
                    "InstanceCount": 1,
                },
            },
        },
    ],

    Schedule={
        "StartTime": datetime.now() + timedelta(minutes=5),
        "EndTime": datetime.now() + timedelta(days=7),  # Test trong 7 ngày
    },
)
```

---

## Canary Release & Blue/Green Deployment

### Blue/Green Deployment — Triển Khai Xanh/Xanh

**Blue/Green Deployment** (Triển Khai Xanh Lá/Xanh Dương): Duy trì hai môi trường giống hệt nhau:
- **Blue**: Môi trường production hiện tại
- **Green**: Môi trường mới với model mới

Traffic được chuyển từ Blue sang Green, Blue được giữ lại để rollback nhanh.

```python
sm = boto3.client("sagemaker")

# Bước 1: Update endpoint với production variant mới (Green deployment)
sm.update_endpoint(
    EndpointName="production-endpoint",
    EndpointConfigName="endpoint-config-v2",  # Config với model mới

    DeploymentConfig={
        "BlueGreenUpdatePolicy": {
            "TrafficRoutingConfiguration": {
                "Type": "ALL_AT_ONCE",           # Chuyển toàn bộ traffic cùng lúc
                # Hoặc: "CANARY", "LINEAR" cho gradual rollout
            },
            "TerminationWaitInSeconds": 300,     # Giữ Blue thêm 5 phút trước khi terminate
            "MaximumExecutionTimeoutInSeconds": 1800,  # Timeout toàn bộ deployment
        },
        "AutoRollbackConfiguration": {
            "Alarms": [
                {
                    "AlarmName": "endpoint-error-rate-too-high",
                    # CloudWatch Alarm sẽ trigger auto-rollback nếu error rate cao
                }
            ]
        },
    },
)
```

### Canary Release — Phát Hành Từng Bước

**Canary Release** (Phát Hành Canary): Chuyển traffic dần dần từ model cũ sang model mới, giám sát theo từng bước.

```python
sm.update_endpoint(
    EndpointName="production-endpoint",
    EndpointConfigName="endpoint-config-v2",

    DeploymentConfig={
        "BlueGreenUpdatePolicy": {
            "TrafficRoutingConfiguration": {
                "Type": "CANARY",
                "CanarySize": {
                    "Type": "CAPACITY_PERCENT",
                    "Value": 10,   # Bước 1: Chuyển 10% traffic sang Green
                },
                "WaitIntervalInSeconds": 300,  # Chờ 5 phút, kiểm tra metrics
                # Sau 5 phút: Chuyển 100% nếu metrics OK, rollback nếu có alarm
            },
            "TerminationWaitInSeconds": 60,
        },
        "AutoRollbackConfiguration": {
            "Alarms": [
                {"AlarmName": "canary-4xx-error-rate"},
                {"AlarmName": "canary-p99-latency-high"},
            ]
        },
    },
)
```

```
Canary Release Timeline:
T=0:   Deploy new model, route 10% traffic → Green, 90% → Blue
T=5m:  Check CloudWatch metrics (error rate, latency, custom metrics)
       → OK: Route 100% → Green, terminate Blue sau 60s
       → ALARM: Auto-rollback 100% → Blue, investigate
```

---

## A/B Testing với Multi-variant Endpoint

### Khái Niệm A/B Testing trong ML

**A/B Testing** (Kiểm Tra A/B) chạy hai variants song song với traffic được chia để so sánh metrics thực tế.

```python
sm = boto3.client("sagemaker")

# Tạo Endpoint Configuration với 2 variants và traffic split
sm.create_endpoint_config(
    EndpointConfigName="ab-test-config",
    ProductionVariants=[
        {
            "VariantName": "ModelA",                  # Model hiện tại
            "ModelName": "model-a-xgboost",
            "InitialInstanceCount": 2,
            "InstanceType": "ml.m5.xlarge",
            "InitialVariantWeight": 70,               # 70% traffic
        },
        {
            "VariantName": "ModelB",                  # Model mới cần test
            "ModelName": "model-b-neural-network",
            "InitialInstanceCount": 1,
            "InstanceType": "ml.m5.xlarge",
            "InitialVariantWeight": 30,               # 30% traffic
        },
    ],
)

# Theo dõi metrics per variant
cloudwatch = boto3.client("cloudwatch")

# SageMaker tự động tách CloudWatch metrics theo VariantName
for variant in ["ModelA", "ModelB"]:
    metrics = cloudwatch.get_metric_statistics(
        Namespace="AWS/SageMaker",
        MetricName="Invocations",
        Dimensions=[
            {"Name": "EndpointName", "Value": "ab-test-endpoint"},
            {"Name": "VariantName", "Value": variant},
        ],
        StartTime=datetime.now() - timedelta(hours=24),
        EndTime=datetime.now(),
        Period=3600,
        Statistics=["Sum"],
    )
    print(f"{variant}: {sum(p['Sum'] for p in metrics['Datapoints'])} invocations")
```

### Điều Chỉnh Traffic Sau A/B Test

```python
# Sau khi A/B test, điều chỉnh traffic split dựa trên kết quả
sm.update_endpoint_weights_and_capacities(
    EndpointName="ab-test-endpoint",
    DesiredWeightsAndCapacities=[
        {
            "VariantName": "ModelA",
            "DesiredWeight": 0,    # Model cũ → 0% traffic
        },
        {
            "VariantName": "ModelB",
            "DesiredWeight": 100,  # Model mới → 100% traffic (winner)
        },
    ],
)
```

---

## Multi-model & Multi-container Patterns

### Multi-container Endpoint — Endpoint Đa Container

**Multi-container Endpoint** khác với MME (Multi-Model Endpoint — Endpoint Đa Mô Hình): Cho phép **pipeline inference** (Suy Luận Theo Pipeline) — request đi qua nhiều containers theo thứ tự, mỗi container xử lý một bước.

```
Use case: Preprocessing + Inference + Postprocessing trong một endpoint

Request → [Container 1: Preprocessing] → [Container 2: ML Model] → [Container 3: Postprocessing] → Response
          Normalize, encode              Predict probability         Apply business rules, format output
```

```python
sm = boto3.client("sagemaker")

# Tạo model với pipeline container sequence
sm.create_model(
    ModelName="pipeline-inference-model",
    Containers=[
        {
            "ContainerHostname": "preprocessing",
            "Image": "123456789.dkr.ecr.us-east-1.amazonaws.com/preprocessing:latest",
        },
        {
            "ContainerHostname": "inference",
            "Image": "763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-inference:2.0",
            "ModelDataUrl": "s3://my-bucket/models/model.tar.gz",
        },
        {
            "ContainerHostname": "postprocessing",
            "Image": "123456789.dkr.ecr.us-east-1.amazonaws.com/postprocessing:latest",
        },
    ],
    InferenceExecutionConfig={
        "Mode": "Serial",  # Sequential pipeline (mặc định)
        # Hoặc "Direct" để gọi container cụ thể qua TargetContainerHostname
    },
    ExecutionRoleArn=role_arn,
)
```

---

## Event-driven ML Architecture

### Kiến Trúc ML Hướng Sự Kiện

**Event-driven** (Hướng Sự Kiện) — xử lý ML inference khi có sự kiện xảy ra thay vì real-time API hoặc batch scheduling:

```
S3 Event-driven Inference:
┌─────────┐   Upload file    ┌─────────────┐  Trigger   ┌──────────────────┐
│ User    │──────────────►   │   S3 Bucket │─────────►  │  Lambda Function │
│ Upload  │                  │  (input)    │            │  invoke_endpoint │
└─────────┘                  └─────────────┘            └────────┬─────────┘
                                                                  │ InvokeEndpoint
                                                                  ▼
                                                    ┌──────────────────────┐
                                                    │  SageMaker Endpoint  │
                                                    └────────┬─────────────┘
                                                             │ Prediction
                                                             ▼
                                                    ┌──────────────────┐
                                                    │  DynamoDB/SNS/   │
                                                    │  SQS/EventBridge │
                                                    └──────────────────┘
```

```python
import json
import boto3

def lambda_handler(event, context):
    """Lambda trigger bởi S3 event khi có file upload mới"""
    runtime = boto3.client("sagemaker-runtime")
    s3 = boto3.client("s3")
    dynamodb = boto3.resource("dynamodb")

    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]

        # Đọc file từ S3
        obj = s3.get_object(Bucket=bucket, Key=key)
        content = obj["Body"].read()

        # Gọi SageMaker endpoint
        response = runtime.invoke_endpoint(
            EndpointName="document-classifier-endpoint",
            ContentType="application/pdf",
            Body=content,
        )
        prediction = json.loads(response["Body"].read())

        # Lưu kết quả vào DynamoDB
        table = dynamodb.Table("ml-predictions")
        table.put_item(Item={
            "file_key": key,
            "prediction": prediction["category"],
            "confidence": str(prediction["confidence"]),
            "timestamp": datetime.now().isoformat(),
        })

    return {"statusCode": 200, "body": "Processing complete"}
```

---

## Multi-region ML Architecture

### Tại Sao Cần Multi-region?

```
Mục tiêu Multi-region ML:
├── HA (High Availability — Tính Sẵn Sàng Cao): Nếu us-east-1 down, failover sang us-west-2
├── Low Latency: Deploy model gần user nhất (user EU → eu-west-1)
├── Data Residency: GDPR yêu cầu EU data ở trong EU
└── DR (Disaster Recovery — Khôi Phục Sau Thảm Họa): RPO, RTO thấp
```

### Kiến Trúc Active-Active Multi-region

```
                        ┌──────────────────┐
                        │   Route 53       │
                        │  Latency-based   │  Route đến region gần nhất
                        │  Routing         │
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
              ▼                  ▼                   ▼
    ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
    │   us-east-1      │ │  eu-west-1   │ │   ap-southeast-1 │
    │                  │ │              │ │                   │
    │ SageMaker        │ │ SageMaker    │ │ SageMaker         │
    │ Endpoint         │ │ Endpoint     │ │ Endpoint          │
    │                  │ │              │ │                   │
    │ Model v2.1       │ │ Model v2.1   │ │ Model v2.1        │
    └──────────────────┘ └──────────────┘ └──────────────────┘
              │                  │                   │
              └──────────────────┴───────────────────┘
                                 │
                        ┌────────▼─────────┐
                        │   S3 (Model      │
                        │   Artifacts)     │  Cross-region replication
                        │   us-east-1      │  sang eu-west-1 & ap-southeast-1
                        └──────────────────┘
```

### Replication Model Artifacts Cross-region

```python
s3 = boto3.client("s3")

# Bật S3 Cross-Region Replication (CRR — Sao Chép Đa Vùng)
s3.put_bucket_replication(
    Bucket="model-artifacts-us-east-1",
    ReplicationConfiguration={
        "Role": "arn:aws:iam::123456789:role/s3-crr-role",
        "Rules": [
            {
                "Status": "Enabled",
                "Filter": {"Prefix": "models/"},  # Chỉ replicate model artifacts
                "Destination": {
                    "Bucket": "arn:aws:s3:::model-artifacts-eu-west-1",
                    "StorageClass": "STANDARD",
                },
            },
            {
                "Status": "Enabled",
                "Filter": {"Prefix": "models/"},
                "Destination": {
                    "Bucket": "arn:aws:s3:::model-artifacts-ap-southeast-1",
                    "StorageClass": "STANDARD",
                },
            },
        ],
    },
)
```

---

## Chọn Pattern Phù Hợp

### Ma Trận Quyết Định

| Use Case | Latency | Traffic | Pattern Khuyến Nghị |
|---------|---------|---------|---------------------|
| Real-time recommendation | < 100ms | Cao, ổn định | Real-time Endpoint + Auto Scaling + Cache |
| Document processing | 5-60s | Trung bình, burst | Async Inference + SNS notification |
| Nightly fraud detection | Giờ | Batch định kỳ | Batch Transform + EventBridge schedule |
| Test model mới không rủi ro | N/A | Production traffic | Shadow Deployment |
| Gradually rollout model mới | N/A | 10% → 100% | Canary Release |
| So sánh 2 model versions | < 100ms | Chia 70/30 | Multi-variant A/B Test |
| 100 models ít được dùng | < 500ms | Thấp, không đều | Multi-Model Endpoint |
| Spike traffic không đoán được | < 100ms | Rất không đều | Serverless Inference |
| Global user base, low latency | < 50ms | Theo region | Multi-region Active-Active |

### Decision Tree Code

```python
def recommend_architecture(
    latency_ms: int,
    traffic_pattern: str,  # "constant", "burst", "batch", "intermittent"
    deployment_goal: str,   # "new_model", "ab_test", "shadow", "production"
    num_models: int = 1,
    user_regions: list = ["us-east-1"]
) -> str:

    if deployment_goal == "shadow":
        return "Shadow Deployment với SageMaker Shadow Testing"

    if deployment_goal == "ab_test":
        return "Multi-variant Endpoint với traffic split"

    if deployment_goal == "new_model" and traffic_pattern == "constant":
        return "Canary Release (10% → 100%) với Auto Rollback"

    if traffic_pattern == "batch":
        return "Batch Transform + EventBridge schedule"

    if latency_ms > 30000:  # > 30 giây
        return "Async Inference + SNS notification"

    if traffic_pattern == "intermittent":
        return "Serverless Inference (zero cost khi idle)"

    if num_models > 5:
        return "Multi-Model Endpoint (MME)"

    if len(user_regions) > 1:
        return "Multi-region Active-Active + Route 53 Latency Routing"

    return "Real-time Endpoint + Application Auto Scaling"
```

---

## 📌 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Giải thích sự khác nhau giữa Shadow Deployment, Canary Release và Blue/Green Deployment?**
> A: **Shadow Deployment** — Copy production traffic sang model mới, model mới xử lý nhưng không trả lời user; dùng để validate model mới mà không rủi ro. **Canary Release** — Chuyển dần dần (ví dụ: 10% → 100%) traffic sang model mới theo bước; dùng khi muốn rollout từng bước với monitoring. **Blue/Green** — Duy trì hai môi trường giống hệt nhau, chuyển 100% traffic cùng lúc nhưng giữ Blue để rollback nhanh; dùng khi muốn deploy nhanh và có rollback plan.

**Q: Khi nào dùng Async Inference thay vì Real-time Endpoint?**
> A: Async Inference khi: (1) Model cần processing time > 1 giây (video analysis, large documents); (2) Input payload > 6MB (SageMaker real-time limit); (3) User không cần kết quả ngay (email sau khi xử lý xong); (4) Traffic burst — Async buffer requests thay vì auto scale tốn kém.

**Q: Thiết kế kiến trúc ML cho một e-commerce recommendation system cần phục vụ 1 triệu user với latency < 50ms?**
> A: (1) Batch job chạy mỗi đêm với SageMaker Batch Transform — precompute recommendations cho tất cả users, lưu vào DynamoDB; (2) Real-time endpoint để compute real-time recommendations khi user có behavior mới (ít request hơn vì đã có precomputed); (3) ElastiCache Redis để cache recommendations — 50ms budget không đủ cho real-time inference từ đầu; (4) Auto Scaling cho real-time endpoint; (5) Multi-region nếu có users ở nhiều châu lục.

**Q: Làm thế nào implement A/B testing với SageMaker để so sánh hai model algorithms?**
> A: Tạo SageMaker Endpoint Config với hai ProductionVariants có InitialVariantWeight (ví dụ: 70/30). Deploy endpoint. CloudWatch tự động tách metrics theo VariantName. Sau khi chạy đủ thời gian và traffic, so sánh metrics kinh doanh (CTR, conversion, revenue per session). Winner variant được điều chỉnh về 100% với `update_endpoint_weights_and_capacities`.

---

**Liên Kết Liên Quan:**
- [SageMaker Inference Types chi tiết](../02-sagemaker/3-sagemaker-inference.md)
- [MLOps Pipelines](../09-mlops/1-sagemaker-pipelines.md)
- [Model Registry & Approval](../09-mlops/2-model-registry.md)
- [Cost Optimization](./1-cost-optimization.md)

**Cập Nhật Lần Cuối:** 2026-06-03
