# SageMaker Inference — Triển Khai Mô Hình ML Lên Production

> SageMaker cung cấp bốn loại inference (suy luận / dự đoán) khác nhau để đáp ứng mọi nhu cầu: từ real-time API với độ trễ mili-giây, đến batch processing hàng triệu records, đến async inference cho file lớn và serverless cho traffic thưa.

---

## 📚 Mục Lục

1. [Tổng Quan Inference Types](#tổng-quan-inference-types)
2. [Real-time Inference](#real-time-inference)
3. [Batch Transform](#batch-transform)
4. [Async Inference](#async-inference)
5. [Serverless Inference](#serverless-inference)
6. [Multi-Model Endpoint](#multi-model-endpoint)
7. [Inference Pipeline](#inference-pipeline)
8. [Auto Scaling Endpoints](#auto-scaling-endpoints)
9. [Model Serving Best Practices](#model-serving-best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Inference Types

### Bảng So Sánh 4 Loại Inference

| Tiêu Chí | Real-time | Batch Transform | Async | Serverless |
|---|---|---|---|---|
| **Độ Trễ (Latency)** | <100ms | Phút-giờ | Giây-phút | 100ms-giây |
| **Payload Size** | ≤6MB | Không giới hạn | ≤1GB | ≤4MB |
| **Chi Phí Khi Idle** | Tính phí 24/7 | Chỉ khi chạy | Chỉ khi xử lý | $0 khi idle |
| **Use Case** | Live API, chatbot | File xử lý hàng loạt | Video, audio, docs lớn | Traffic thưa/không đều |
| **Scaling** | Auto scaling | Parallel jobs | Queue-based | Tự động 0→N |

### Khi Nào Dùng Loại Nào?

```
Có traffic liên tục, cần độ trễ thấp?
    → Real-time Endpoint

Xử lý S3 files lớn theo lô (batch), không cần response ngay?
    → Batch Transform

File lớn >6MB, xử lý lâu (>1 phút), client có thể chờ?
    → Async Inference

Traffic không thường xuyên, không thể dự đoán, OK với cold start?
    → Serverless Inference
```

---

## Real-time Inference (Suy Luận Thời Gian Thực)

**Real-time Endpoint** (Điểm Cuối Thời Gian Thực) là HTTP(S) endpoint luôn sẵn sàng, respond trong mili-giây.

### Deploy Endpoint

```python
from sagemaker.sklearn import SKLearnModel
import sagemaker

# Cách 1: Deploy từ estimator sau training
predictor = estimator.deploy(
    initial_instance_count=1,
    instance_type='ml.c5.xlarge',
    endpoint_name='churn-predictor-v1'  # Tên endpoint
)

# Cách 2: Deploy từ model artifacts S3
model = SKLearnModel(
    model_data='s3://bucket/output/model.tar.gz',  # Model đã train
    role=role,
    framework_version='1.2-1',
    entry_point='inference.py'  # Script serving
)

predictor = model.deploy(
    initial_instance_count=1,
    instance_type='ml.c5.xlarge'
)
```

### Inference Script (Script Phục Vụ Dự Đoán)

```python
# inference.py — SageMaker gọi các hàm này khi serve
import os
import json
import joblib
import numpy as np

def model_fn(model_dir):
    """Load model từ /opt/ml/model"""
    return joblib.load(os.path.join(model_dir, 'model.joblib'))

def input_fn(request_body, request_content_type):
    """
    Deserialize (giải tuần tự hóa) input từ request.
    Chuyển HTTP request body thành Python object.
    """
    if request_content_type == 'application/json':
        data = json.loads(request_body)
        return np.array(data['features'])
    elif request_content_type == 'text/csv':
        return np.array([float(x) for x in request_body.split(',')])
    else:
        raise ValueError(f"Unsupported content type: {request_content_type}")

def predict_fn(input_data, model):
    """Chạy inference (dự đoán)"""
    prediction = model.predict(input_data)
    probabilities = model.predict_proba(input_data)
    return {'prediction': prediction.tolist(), 'probabilities': probabilities.tolist()}

def output_fn(prediction, response_content_type):
    """
    Serialize (tuần tự hóa) output để gửi response.
    Chuyển Python object thành HTTP response body.
    """
    if response_content_type == 'application/json':
        return json.dumps(prediction)
    raise ValueError(f"Unsupported content type: {response_content_type}")
```

### Gọi Endpoint

```python
import boto3
import json

runtime = boto3.client('sagemaker-runtime')

# Invoke endpoint (Gọi điểm cuối)
response = runtime.invoke_endpoint(
    EndpointName='churn-predictor-v1',
    ContentType='application/json',
    Accept='application/json',
    Body=json.dumps({'features': [5.1, 3.5, 1.4, 0.2]})
)

result = json.loads(response['Body'].read().decode())
print(f"Prediction: {result['prediction']}")
print(f"Confidence: {result['probabilities']}")
```

### Production Variants (Biến Thể Sản Xuất)

**Production Variants** cho phép chạy nhiều model versions trên cùng một endpoint — dùng cho A/B testing (Thử Nghiệm A/B) và canary deployment (Triển Khai Canary):

```python
from sagemaker.session import ProductionVariant

# Tạo endpoint với 2 model variants
variant_a = ProductionVariant(
    variant_name='model-v1',
    model_name='churn-model-v1',
    initial_instance_count=1,
    instance_type='ml.c5.xlarge',
    initial_weight=80  # 80% traffic đến model v1
)

variant_b = ProductionVariant(
    variant_name='model-v2',
    model_name='churn-model-v2',
    initial_instance_count=1,
    instance_type='ml.c5.xlarge',
    initial_weight=20  # 20% traffic đến model v2 (canary)
)

sm_client = boto3.client('sagemaker')
sm_client.create_endpoint_config(
    EndpointConfigName='churn-ab-test-config',
    ProductionVariants=[variant_a, variant_b]
)
```

---

## Batch Transform (Biến Đổi Theo Lô)

**Batch Transform** xử lý toàn bộ dataset lưu trên S3 cùng một lúc — không cần endpoint thường trực.

### Khi Nào Dùng Batch Transform

- Cần dự đoán cho toàn bộ dataset (VD: score toàn bộ khách hàng hàng tuần)
- Data đã có sẵn trên S3, không cần real-time response
- Tiết kiệm chi phí: instance chỉ chạy khi cần, tắt sau khi xong

### Ví Dụ Batch Transform

```python
from sagemaker.transformer import Transformer

# Tạo transformer từ model đã train
transformer = estimator.transformer(
    instance_count=1,
    instance_type='ml.m5.xlarge',
    output_path='s3://bucket/batch-predictions/',
    strategy='MultiRecord',           # 'SingleRecord' hoặc 'MultiRecord'
    assemble_with='Line',             # Ghép outputs bằng newline
    accept='text/csv'
)

# Chạy batch transform job
transformer.transform(
    data='s3://bucket/input-data/',   # Input S3 path (có thể là folder)
    content_type='text/csv',
    split_type='Line',                # Mỗi dòng là 1 record
    join_source='Input'               # Ghép input và output trong kết quả
)

# Chờ job hoàn thành
transformer.wait()
print(f"Output tại: {transformer.output_path}")
```

### Chiến Lược Xử Lý Record

| Strategy | Mô Tả | Dùng Khi |
|---|---|---|
| `SingleRecord` | 1 request = 1 record | Model chỉ nhận 1 sample/request |
| `MultiRecord` | 1 request = nhiều records | Model nhận batch, tăng throughput |

---

## Async Inference (Suy Luận Bất Đồng Bộ)

**Async Inference** (Bất Đồng Bộ) xử lý request lớn mà không block client — phù hợp cho video, audio, document dài.

### Cơ Chế Hoạt Động

```
Client                  SageMaker Async          Output S3
  │                          │                      │
  │── POST Request ─────────►│                      │
  │                          │ (Enqueue — Xếp Hàng) │
  │◄── 202 Accepted ─────────│                      │
  │    + Output Location     │                      │
  │                          │── Processing ────────►│
  │                          │   (giây đến phút)     │
  │                          │◄── Done ──────────────│
  │ (Client poll S3 hoặc     │                       │
  │  nhận SNS notification)  │                       │
  │── GET S3 Output ─────────────────────────────────►│
  │◄── Result ────────────────────────────────────────│
```

### Deploy Async Endpoint

```python
from sagemaker.async_inference import AsyncInferenceConfig

async_config = AsyncInferenceConfig(
    output_path='s3://bucket/async-outputs/',  # S3 lưu kết quả
    notification_config={
        'SuccessTopic': 'arn:aws:sns:...:async-success',   # SNS khi thành công
        'ErrorTopic': 'arn:aws:sns:...:async-error'        # SNS khi lỗi
    },
    max_concurrent_invocations_per_instance=4  # Số requests xử lý song song
)

model.deploy(
    instance_type='ml.m5.xlarge',
    async_inference_config=async_config
)
```

### Invoke Async Endpoint

```python
import boto3

runtime = boto3.client('sagemaker-runtime')

# Upload input lên S3 trước
s3 = boto3.client('s3')
s3.upload_file('large_video.mp4', 'my-bucket', 'inputs/video.mp4')

# Invoke async endpoint
response = runtime.invoke_endpoint_async(
    EndpointName='my-async-endpoint',
    InputLocation='s3://my-bucket/inputs/video.mp4',  # Input trên S3
    ContentType='video/mp4'
)

output_location = response['OutputLocation']  # S3 path kết quả
request_id = response['InferenceId']

print(f"Request submitted. Output will be at: {output_location}")
# Client có thể poll S3 hoặc chờ SNS notification
```

---

## Serverless Inference (Suy Luận Không Máy Chủ)

**Serverless Inference** tự động scale từ 0 đến N — không cần cấu hình instance, không tốn phí khi idle.

### Đặc Điểm

- **Cold Start** (Khởi Động Nguội): Lần đầu invoke sau khoảng idle dài có thể mất 5-30 giây
- **Không quản lý instance**: AWS tự động provision và teardown
- **Pricing**: Trả theo số requests và compute time (giây × GB RAM)
- **Max Concurrency** (Đồng Thời Tối Đa): Tối đa 200 concurrent invocations

### Deploy Serverless Endpoint

```python
from sagemaker.serverless import ServerlessInferenceConfig

serverless_config = ServerlessInferenceConfig(
    memory_size_in_mb=2048,    # RAM: 1024, 2048, 3072, 4096, 5120, 6144 MB
    max_concurrency=10          # Số requests xử lý đồng thời tối đa
)

predictor = model.deploy(
    serverless_inference_config=serverless_config
)
```

### Khi Nào Dùng Serverless

```
Traffic Pattern phù hợp với Serverless:
- Dev/test environment (môi trường phát triển/kiểm thử)
- Ứng dụng dùng ít, vài request/ngày
- Seasonal workloads (tải theo mùa) — VD: Black Friday
- Prototype trước khi commit real-time endpoint

Traffic Pattern KHÔNG phù hợp:
- Production API với SLA (Service Level Agreement) độ trễ thấp
- Continuous high traffic
- Cần response <100ms consistently
```

---

## Multi-Model Endpoint — MME (Điểm Cuối Đa Mô Hình)

**MME** host nhiều model trên cùng một endpoint — chia sẻ infrastructure, tiết kiệm chi phí đáng kể khi có nhiều model tương tự nhau.

### Use Case Điển Hình

```
Ví Dụ: 1000 model dự đoán churn cho 1000 khách hàng B2B khác nhau

Không có MME: 1000 endpoints × $0.10/giờ = $100/giờ = $2,400/ngày ❌

Với MME: 1 endpoint với 1 instance, load model on-demand
  → ~$0.10/giờ cho tất cả models ✅
  (Thực tế có thể cần 2-5 instances tùy traffic, nhưng vẫn rẻ hơn 100-500x)
```

### Cơ Chế Hoạt Động

```
Request: Target Model = "customer_123_model"
         │
         ▼
    MME Endpoint
    ├── Model Cache (Bộ Nhớ Đệm Model) trong RAM/disk
    │   ├── customer_001_model (cached — đã load)
    │   ├── customer_087_model (cached)
    │   └── customer_123_model → NOT CACHED
    │                           │
    │                           ▼
    │                   Load từ S3:
    │                   s3://models/customer_123/model.tar.gz
    │                           │
    │                           ▼
    └── Run inference (Chạy dự đoán)
```

### Deploy MME

```python
from sagemaker.multidatamodel import MultiDataModel

# Tất cả models lưu trong cùng S3 prefix (tiền tố S3)
model_data_prefix = 's3://bucket/models/'

# Mỗi model là 1 file .tar.gz trong prefix đó:
# s3://bucket/models/model_a.tar.gz
# s3://bucket/models/model_b.tar.gz

mme = MultiDataModel(
    name='multi-customer-models',
    model_data_prefix=model_data_prefix,
    model=base_model  # Base model config (container, role)
)

predictor = mme.deploy(
    initial_instance_count=1,
    instance_type='ml.c5.xlarge'
)
```

### Invoke Specific Model

```python
# Chỉ định model nào cần dùng qua TargetModel header
response = runtime.invoke_endpoint(
    EndpointName='multi-customer-models',
    ContentType='application/json',
    TargetModel='customer_123_model.tar.gz',  # Model cụ thể
    Body=json.dumps({'features': [...]})
)
```

---

## Inference Pipeline (Đường Ống Suy Luận)

**Inference Pipeline** chuỗi nhiều containers lại với nhau — tiền xử lý → model → hậu xử lý — trong một endpoint.

```
Request
  │
  ▼
Container 1: Preprocessing (Tiền Xử Lý)
  (normalize features, encode categoricals)
  │
  ▼
Container 2: ML Model (XGBoost)
  (prediction)
  │
  ▼
Container 3: Postprocessing (Hậu Xử Lý)
  (decode labels, format response)
  │
  ▼
Response
```

```python
from sagemaker.pipeline import PipelineModel

pipeline_model = PipelineModel(
    models=[
        preprocessing_model,  # Container 1: scikit-learn preprocessing
        xgboost_model,         # Container 2: XGBoost prediction
        postprocessing_model   # Container 3: custom postprocessing
    ],
    role=role
)

predictor = pipeline_model.deploy(
    initial_instance_count=1,
    instance_type='ml.c5.xlarge'
)
```

---

## Auto Scaling Endpoints (Tự Động Mở Rộng Endpoint)

### Application Auto Scaling

```python
import boto3

client = boto3.client('application-autoscaling')

# Đăng ký endpoint variant như scalable target (mục tiêu mở rộng)
client.register_scalable_target(
    ServiceNamespace='sagemaker',
    ResourceId='endpoint/churn-predictor-v1/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    MinCapacity=1,    # Tối thiểu 1 instance
    MaxCapacity=10    # Tối đa 10 instances
)

# Tạo scaling policy (chính sách mở rộng)
client.put_scaling_policy(
    PolicyName='invocations-scaling',
    ServiceNamespace='sagemaker',
    ResourceId='endpoint/churn-predictor-v1/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 1000.0,  # Mục tiêu: 1000 invocations/phút/instance
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
        },
        'ScaleInCooldown': 300,   # Chờ 5 phút trước khi scale in
        'ScaleOutCooldown': 60    # Chờ 1 phút trước khi scale out
    }
)
```

### Scaling Metrics Phổ Biến

| Metric | Mô Tả | Dùng Khi |
|---|---|---|
| `SageMakerVariantInvocationsPerInstance` | Số requests/phút/instance | Traffic-based scaling phổ biến nhất |
| `CPUUtilization` | % CPU của instances | CPU-bound models |
| `MemoryUtilization` | % RAM của instances | Memory-intensive models |
| `GPUUtilization` | % GPU utilization | Deep learning models |

---

## Model Serving Best Practices

### 1. Chọn Đúng Instance Type

```
Tabular ML (XGBoost, Linear Learner):
  → ml.c5.xlarge đến ml.c5.4xlarge (CPU-optimized)
  
NLP Models (BERT, transformer-based):
  → ml.g4dn.xlarge (GPU T4) hoặc ml.inf2.xlarge (Inferentia2)
  
Image Models:
  → ml.g4dn.xlarge hoặc ml.g4dn.2xlarge

Cost-optimized Production:
  → ml.inf2.xlarge (AWS Inferentia2 — rẻ hơn GPU 40-70%)
```

### 2. Model Optimization (Tối Ưu Mô Hình)

**Neo Compilation** — biên dịch model cho hardware cụ thể:

```python
compiled_model = estimator.compile_model(
    target_instance_family='ml_c5',  # Compile cho C5 instances
    input_shape={'data': [1, 3, 224, 224]},  # Input shape
    role=role,
    framework='pytorch',
    framework_version='1.8'
)
# Model sau khi compile chạy nhanh hơn 2-5x trên target hardware
```

### 3. Endpoint Health và Monitoring

```python
import boto3

sm = boto3.client('sagemaker')

# Kiểm tra endpoint status
response = sm.describe_endpoint(EndpointName='churn-predictor-v1')
print(f"Status: {response['EndpointStatus']}")
# Possible: Creating, Updating, InService, Deleting, Failed

# Xem CloudWatch metrics quan trọng:
# - Invocations: Tổng số requests
# - InvocationErrors: Số requests lỗi
# - ModelLatency: Thời gian model xử lý (ms)
# - OverheadLatency: Thời gian overhead của SageMaker (ms)
```

### 4. Blue/Green Deployment (Triển Khai Xanh/Lam)

Cập nhật endpoint không có downtime (thời gian chết):

```python
# Update endpoint để chuyển sang config mới
sm.update_endpoint(
    EndpointName='churn-predictor-v1',
    EndpointConfigName='churn-config-v2',  # Config mới với model mới
    DeploymentConfig={
        'BlueGreenUpdatePolicy': {
            'TrafficRoutingConfiguration': {
                'Type': 'LINEAR',                         # Chuyển traffic dần dần
                'LinearStepSize': {'Type': 'PERCENT', 'Value': 10},  # 10% mỗi bước
                'WaitIntervalInSeconds': 300              # Chờ 5 phút giữa mỗi bước
            },
            'TerminationWaitInSeconds': 600              # Chờ 10 phút trước khi xóa fleet cũ
        }
    }
)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Khi nào dùng Real-time, Batch Transform, Async và Serverless?

**Trả lời tốt:**
> Tôi chọn dựa trên 3 yếu tố: latency requirement (yêu cầu độ trễ), payload size (kích thước dữ liệu) và traffic pattern (mẫu traffic).
> - Real-time: Khi ứng dụng cần response trong mili-giây — live fraud detection, recommendation API, chatbot. Endpoint luôn chạy nên phù hợp traffic liên tục.
> - Batch Transform: Khi có sẵn tập data lớn trên S3 và không cần response ngay — chạy weekly scoring toàn bộ customer base, offline feature computation.
> - Async Inference: Khi payload lớn (video, audio) hoặc inference lâu (>1 phút) nhưng không muốn hold connection — NLP trên document dài, video analysis.
> - Serverless: Khi traffic rất thưa hoặc không đều — internal tools, dev/test environments, seasonal features. Lợi điểm lớn là $0 khi không có request, nhưng có cold start latency.

### Q2: Multi-Model Endpoint giải quyết bài toán gì? Khi nào KHÔNG phù hợp?

**Trả lời tốt:**
> MME giải quyết bài toán cost khi có nhiều model nhỏ cần host — ví dụ 1000 model personalized cho 1000 khách hàng B2B. Thay vì 1000 endpoints tốn $2400/ngày, dùng 1 MME với vài instances. MME load model từ S3 on-demand và cache trong bộ nhớ instance.
> Tuy nhiên MME không phù hợp khi: (1) Mỗi model cần dedicated GPU — MME chia sẻ resources; (2) Model quá lớn để cache nhiều cùng lúc — phải evict thường xuyên gây latency spike; (3) Tất cả models đều được gọi với tần suất cao tương đương nhau — cache hit rate thấp, lợi ích ít. Trong các trường hợp này, nên cân nhắc Multi-Container Endpoint hoặc nhiều single-model endpoints.

### Q3: Giải thích Auto Scaling cho SageMaker Endpoint. Metric nào quan trọng nhất?

**Trả lời tốt:**
> Auto Scaling cho Endpoint dùng Application Auto Scaling service, scale số lượng instances dựa trên CloudWatch metrics. Metric quan trọng nhất là `SageMakerVariantInvocationsPerInstance` — số invocations mỗi phút per instance. Tôi set target value bằng cách đo throughput của 1 instance (VD: 1000 req/phút), rồi configure scale out khi vượt quá. Scale in cooldown nên dài hơn scale out cooldown (VD: 5 phút vs 1 phút) để tránh thrashing — scale in/out liên tục. Ngoài ra cần set MinCapacity ≥ 1 để tránh scale to zero và gây cold start, trừ khi traffic thực sự có lúc hoàn toàn 0. Cho deep learning models, GPUUtilization cũng quan trọng vì đây thường là bottleneck.

---

## 📊 Tóm Tắt

```
SageMaker Inference
├── Real-time Endpoint
│   ├── Luôn chạy, tính phí 24/7
│   ├── Độ trễ <100ms
│   ├── Production Variants cho A/B test
│   └── Auto Scaling với Application Auto Scaling
│
├── Batch Transform
│   ├── Xử lý S3 files theo lô
│   ├── Instance chỉ chạy khi job chạy
│   └── Output lưu S3
│
├── Async Inference
│   ├── Queue-based, không block client
│   ├── Payload ≤1GB
│   └── SNS notification khi xong
│
├── Serverless Inference
│   ├── $0 khi idle
│   ├── Cold start 5-30s
│   └── Max 200 concurrent invocations
│
└── Advanced Patterns
    ├── Multi-Model Endpoint (MME) — chia sẻ infrastructure nhiều models
    ├── Inference Pipeline — chuỗi containers preprocessing → model → postprocessing
    └── Blue/Green Deployment — cập nhật không downtime
```

---

**File tiếp theo:** [4-sagemaker-autopilot.md](./4-sagemaker-autopilot.md) — AutoML và Hyperparameter Optimization Tự Động

**Cập Nhật Lần Cuối:** 2026-06-03
