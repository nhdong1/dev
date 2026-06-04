# Tối Ưu Chi Phí AWS AI/ML — Cost Optimization

> Chiến lược giảm chi phí SageMaker Training (Huấn Luyện) và Inference (Dự Đoán) từ 50-90% mà không ảnh hưởng đến hiệu suất

## 📋 Mục Lục

1. [Tổng Quan Chi Phí SageMaker](#tổng-quan-chi-phí-sagemaker)
2. [Tối Ưu Chi Phí Training](#tối-ưu-chi-phí-training)
3. [Tối Ưu Chi Phí Inference](#tối-ưu-chi-phí-inference)
4. [AWS Inferentia — Inf1 & Inf2](#aws-inferentia--inf1--inf2)
5. [Multi-Model Endpoint](#multi-model-endpoint-mme)
6. [Serverless Inference](#serverless-inference)
7. [SageMaker Savings Plans](#sagemaker-savings-plans)
8. [Giám Sát & Phân Tích Chi Phí](#giám-sát--phân-tích-chi-phí)
9. [Checklist Tối Ưu Chi Phí](#checklist-tối-ưu-chi-phí)

---

## Tổng Quan Chi Phí SageMaker

### Các Thành Phần Chi Phí Chính

SageMaker tính phí theo **pay-as-you-go** (trả theo mức sử dụng), không có chi phí tối thiểu. Các thành phần chi phí:

```
Chi Phí SageMaker:
├── Training Jobs           Instance giờ khi chạy training
├── Real-time Endpoints     Instance giờ khi endpoint đang chạy (kể cả idle)
├── Batch Transform         Instance giờ khi chạy batch job
├── Async Inference         Instance giờ + số request
├── Serverless Inference    Số GB-giây xử lý + số request
├── SageMaker Studio        Instance giờ cho notebook (JupyterServer/KernelGateway)
├── Feature Store           Storage + Ingestion + Retrieval
└── Model Registry          Storage metadata (không đáng kể)
```

### Bảng Giá Tham Khảo (us-east-1, 2026)

| Instance Type | Mục Đích | Giá/giờ (USD) | Ghi Chú |
|--------------|----------|---------------|---------|
| ml.m5.xlarge | Training/Hosting | ~0.23 | CPU, 4 vCPU, 16GB RAM |
| ml.m5.4xlarge | Training/Hosting | ~0.92 | CPU, 16 vCPU, 64GB RAM |
| ml.p3.2xlarge | GPU Training | ~3.83 | 1x V100 GPU |
| ml.p3.8xlarge | GPU Training | ~15.30 | 4x V100 GPU |
| ml.g4dn.xlarge | GPU Training/Inference | ~0.74 | 1x T4 GPU, tốt cho inference |
| ml.inf1.xlarge | Inference (Inferentia) | ~0.37 | AWS Inferentia chip |
| ml.inf2.xlarge | Inference (Inferentia2) | ~0.76 | AWS Inferentia2 chip |

> **Quy tắc vàng:** Endpoint Real-time tính phí 24/7 kể cả khi không có traffic → Xóa endpoint ngay sau khi không cần dùng.

---

## Tối Ưu Chi Phí Training

### 1. Managed Spot Training — Huấn Luyện Spot

**Spot Instances** (Instance Tạm Thời) là EC2 instances dư thừa của AWS, bán với giá thấp hơn 60-90%. Nhược điểm: AWS có thể thu hồi bất kỳ lúc nào (Spot Interruption — Gián Đoạn Spot).

**SageMaker Managed Spot Training** tự động xử lý việc checkpoint (điểm lưu) và tiếp tục từ checkpoint khi instance bị thu hồi.

```python
import sagemaker
from sagemaker.estimator import Estimator

estimator = Estimator(
    image_uri="763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:2.0-gpu-py310",
    role=sagemaker.get_execution_role(),
    instance_count=1,
    instance_type="ml.p3.2xlarge",

    # Bật Managed Spot Training
    use_spot_instances=True,
    max_run=3600,               # Thời gian chạy tối đa (giây) = 1 giờ
    max_wait=7200,              # Thời gian chờ tối đa kể cả interrupt (giây) = 2 giờ

    # Checkpoint để tiếp tục khi bị interrupt
    checkpoint_s3_uri="s3://my-bucket/checkpoints/",
    checkpoint_local_path="/opt/ml/checkpoints",  # Path trong container
)

estimator.fit({"train": "s3://my-bucket/data/train/"})

# Xem mức tiết kiệm
job_name = estimator.latest_training_job.name
client = boto3.client("sagemaker")
desc = client.describe_training_job(TrainingJobName=job_name)
savings = desc["BillableTimeInSeconds"]      # Thời gian thực tính phí
total = desc["TrainingTimeInSeconds"]         # Tổng thời gian training
print(f"Tiết kiệm: {(1 - savings/total)*100:.1f}%")
```

**Bao nhiêu tiết kiệm?**

| Kịch Bản | Không Spot | Có Spot | Tiết Kiệm |
|----------|-----------|---------|-----------|
| Training 10 giờ ml.p3.2xlarge | 38.30 USD | ~5.75 USD | ~85% |
| Training 100 giờ ml.p3.8xlarge | 1,530 USD | ~230 USD | ~85% |

**Khi nào KHÔNG dùng Spot Training:**
- Job cần hoàn thành trong deadline cứng (ví dụ: cần deploy trước 5 giờ chiều)
- Job ngắn < 5 phút (overhead checkpoint không đáng)
- Job không có checkpoint implementation (rủi ro mất toàn bộ progress)

---

### 2. Pipe Mode và FastFile Mode — Tối Ưu I/O Data

Khi data lớn (> 10GB), thời gian copy data từ S3 vào instance có thể chiếm 20-30% tổng thời gian training.

**File Mode (mặc định):** Copy toàn bộ data vào local disk trước khi training bắt đầu.

**Pipe Mode:** Stream data trực tiếp từ S3 vào training container dưới dạng Unix pipe — không cần copy trước.

**FastFile Mode:** Mount S3 như local filesystem (POSIX-compatible — Tương Thích POSIX) qua FUSE. Đọc files theo yêu cầu, không cần copy toàn bộ.

```python
from sagemaker.inputs import TrainingInput

# FastFile Mode — Khuyến nghị cho phần lớn use cases
train_input = TrainingInput(
    s3_data="s3://my-bucket/data/train/",
    input_mode="FastFile",          # hoặc "Pipe"
)

estimator.fit({"train": train_input})
```

| Mode | Thời Gian Startup | Throughput | Phù Hợp |
|------|------------------|-----------|---------|
| File Mode | Chậm (copy toàn bộ) | Nhanh | Data nhỏ < 5GB |
| Pipe Mode | Nhanh | Phụ thuộc S3 bandwidth | Data lớn, sequential read |
| FastFile Mode | Rất nhanh | Tốt | Data lớn, random access |

---

### 3. Distributed Training — Huấn Luyện Phân Tán

Thay vì chạy 1 instance ml.p3.8xlarge (4 GPU, ~15.30 USD/giờ) trong 10 giờ = **153 USD**, có thể dùng 4 instance ml.p3.2xlarge (1 GPU, ~3.83 USD/giờ) trong ~3 giờ = **46 USD** (tiết kiệm ~70%).

**SageMaker Data Parallelism** (Song Song Dữ Liệu — SDP): Chia batch data ra các GPU/instance, mỗi instance chạy một bản sao model, gradient (Độ Dốc) được tổng hợp.

```python
from sagemaker.pytorch import PyTorch
from sagemaker.debugger import TensorBoardOutputConfig

estimator = PyTorch(
    entry_point="train.py",
    role=sagemaker.get_execution_role(),
    instance_type="ml.p3.16xlarge",  # Multi-GPU instance
    instance_count=2,                  # 2 instances = 16 GPU tổng

    # Bật SageMaker Distributed Data Parallel
    distribution={
        "smdistributed": {
            "dataparallel": {
                "enabled": True
            }
        }
    },

    hyperparameters={
        "epochs": 50,
        "batch-size": 256,  # Sẽ được chia cho số GPU
    },
    framework_version="2.0",
    py_version="py310",
)
```

---

### 4. SageMaker Inference Recommender — Tìm Instance Tối Ưu

**Inference Recommender** (Công Cụ Gợi Ý Instance Inference) tự động benchmark (kiểm tra hiệu suất) model trên nhiều instance types và tìm ra instance tối ưu về cost-performance.

```python
import boto3

sm = boto3.client("sagemaker")

# Bước 1: Đăng ký model vào Model Registry
# (xem 09-mlops/2-model-registry.md)

# Bước 2: Chạy Inference Recommender job
response = sm.create_inference_recommendations_job(
    JobName="my-inference-recommender-job",
    JobType="Default",          # "Default" (nhanh) hoặc "Advanced" (toàn diện)
    RoleArn=role_arn,
    InputConfig={
        "ModelPackageVersionArn": model_package_arn,
        "Endpoints": [],  # Để trống = tự động chọn instance types để test
    },
)

# Bước 3: Xem kết quả sau ~30-90 phút
result = sm.describe_inference_recommendations_job(
    JobName="my-inference-recommender-job"
)
for rec in result["InferenceRecommendations"]:
    print(f"Instance: {rec['EndpointConfiguration']['InstanceType']}")
    print(f"  Cost/hour: ${rec['Metrics']['CostPerHour']:.3f}")
    print(f"  P99 Latency: {rec['Metrics']['MaxInvocations']}ms")
```

---

## Tối Ưu Chi Phí Inference

### Chi Phí Inference Là Vấn Đề Lớn Nhất

Training thường chạy một lần hoặc vài lần/tuần. **Inference endpoint chạy 24/7** — đây là nguồn chi phí lớn nhất trong production.

```
Ví dụ thực tế:
- 1 Real-time endpoint ml.m5.xlarge: 0.23 USD/giờ × 720 giờ/tháng = 165.6 USD/tháng
- Nếu có 10 models khác nhau → 1,656 USD/tháng chỉ cho hosting
- Với Multi-Model Endpoint: Cả 10 models trên 1 endpoint = 165.6 USD/tháng
- Tiết kiệm: 90%
```

---

## AWS Inferentia — Inf1 & Inf2

### Tổng Quan

**AWS Inferentia** là chip ML inference do AWS thiết kế riêng, được tối ưu cho việc chạy model đã huấn luyện (không dùng cho training). So với GPU (ml.p3/ml.g4dn), Inferentia cung cấp:

- **Giá thấp hơn 40-70%** so với GPU tương đương
- **Throughput** (Thông Lượng) cao hơn (số request/giây)
- **Latency thấp hơn** cho các model phổ biến

| Instance | Chip | GPU Tương Đương | Giá/giờ | Use Case |
|----------|------|-----------------|---------|---------|
| ml.inf1.xlarge | 1x Inferentia | ~ml.g4dn.xlarge | $0.37 | Model nhỏ, NLP |
| ml.inf1.6xlarge | 4x Inferentia | ~ml.g4dn.4xlarge | $1.84 | Model trung bình |
| ml.inf2.xlarge | 1x Inferentia2 | ~ml.g5.xlarge | $0.76 | LLM inference |
| ml.inf2.48xlarge | 12x Inferentia2 | ~ml.p4d.24xlarge | $12.98 | LLM lớn (Llama 2 70B) |

### Sử Dụng Inf2 với SageMaker

Model cần được **compile** (biên dịch) sang định dạng **AWS Neuron** (Runtime đặc biệt cho Inferentia) trước khi deploy lên Inf instance.

```python
# Bước 1: Compile model với AWS Neuron SDK
import torch
import torch_neuronx  # Neuron SDK cho PyTorch

model = MyModel()
model.load_state_dict(torch.load("model.pth"))
model.eval()

# Trace và compile model
example_input = torch.zeros(1, 128, dtype=torch.long)  # Ví dụ input shape
neuron_model = torch.neuronx.trace(model, example_input)
torch.jit.save(neuron_model, "model_neuron.pt")

# Bước 2: Deploy lên Inf2 instance
from sagemaker.pytorch import PyTorchModel

model = PyTorchModel(
    model_data="s3://my-bucket/model_neuron.tar.gz",
    role=role,
    framework_version="1.13",
    py_version="py39",
    entry_point="inference_neuron.py",
)

predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.inf2.xlarge",  # Inf2 instance
)
```

**Khi nào dùng Inferentia:**
- Model đã stable, không thay đổi thường xuyên (vì cần compile lại khi đổi model)
- Volume inference cao (> 1000 request/phút)
- Ưu tiên tiết kiệm chi phí và/hoặc cần throughput cao
- Model phổ biến: BERT, RoBERTa, ResNet, MobileNet, Llama 2 (trên Inf2)

---

## Multi-Model Endpoint (MME)

### Khái Niệm

**Multi-Model Endpoint** (MME — Endpoint Đa Mô Hình) cho phép nhiều models chạy trên một endpoint duy nhất. SageMaker tự động load/unload model vào bộ nhớ theo demand (yêu cầu).

```
Không có MME:
├── Model A: ml.m5.xlarge endpoint = 0.23 USD/giờ
├── Model B: ml.m5.xlarge endpoint = 0.23 USD/giờ
├── Model C: ml.m5.xlarge endpoint = 0.23 USD/giờ
└── Tổng: 0.69 USD/giờ = 496.80 USD/tháng

Với MME (3 models trên 1 endpoint):
└── 1x ml.m5.xlarge endpoint = 0.23 USD/giờ = 165.60 USD/tháng
    Tiết kiệm: 66%
```

### Triển Khai MME

```python
import boto3
import sagemaker
from sagemaker.multidatamodel import MultiDataModel

# Tất cả models phải lưu trong cùng S3 prefix
s3_model_data_prefix = "s3://my-bucket/multi-model/"

# Tạo MultiDataModel
mme = MultiDataModel(
    name="my-mme",
    model_data_prefix=s3_model_data_prefix,
    model=base_model,       # Model base xác định framework (PyTorch, XGBoost,...)
    sagemaker_session=sagemaker.Session(),
)

# Deploy endpoint
predictor = mme.deploy(
    initial_instance_count=1,
    instance_type="ml.m5.2xlarge",
)

# Thêm models vào MME (upload model artifact lên S3)
mme.add_model(model_data_source="s3://my-bucket/models/model_a.tar.gz", model_data_path="model_a.tar.gz")
mme.add_model(model_data_source="s3://my-bucket/models/model_b.tar.gz", model_data_path="model_b.tar.gz")

# Invoke với target model cụ thể
import json
response = predictor.predict(
    data={"text": "Classify this text"},
    target_model="model_a.tar.gz",  # Chỉ định model nào xử lý request
)
```

### Cơ Chế Load/Unload Model

```
Khi request đến với target_model="model_a.tar.gz":
1. SageMaker kiểm tra model_a đã ở trong bộ nhớ chưa?
2. Nếu chưa: Load model từ S3 → Cache vào /tmp
3. Xử lý request với model_a
4. Model_a ở trong cache cho request tiếp theo
5. Khi hết bộ nhớ: Unload model ít được dùng nhất (LRU — Least Recently Used)
```

**Lưu ý quan trọng:**
- **Cold start** (Khởi Động Nguội): Request đầu tiên với model chưa được cache sẽ bị chậm do phải load
- MME phù hợp khi nhiều models nhỏ và không phải tất cả đều được call đồng thời
- Mỗi model cùng framework (PyTorch, TensorFlow, XGBoost, v.v.)

---

## Serverless Inference

### Khái Niệm

**Serverless Inference** (Dự Đoán Không Máy Chủ) không có provisioned instance — AWS tự động scale (mở rộng) container lên khi có request và scale về 0 khi không có request. Tính phí theo GB-giây xử lý và số request, không phải theo giờ.

```
Chi phí Serverless Inference:
- $0.00001 per GB-second of computation
- $0.20 per 1,000 invocations
- Không có chi phí idle (không dùng = không tính tiền)

Ví dụ: 10,000 requests/ngày, mỗi request tốn 100ms, memory config 2GB
- Computation: 10,000 × 0.1s × 2GB × $0.00001 = $0.02/ngày
- Requests: 10,000 / 1,000 × $0.20 = $2.00/ngày
- Tổng: ~$2.02/ngày = ~$60/tháng

So với Real-time endpoint ml.m5.xlarge: $0.23 × 24 × 30 = $165.60/tháng
Tiết kiệm: 64% (nếu traffic thấp/intermittent)
```

### Triển Khai Serverless Endpoint

```python
from sagemaker.serverless import ServerlessInferenceConfig

# Cấu hình Serverless Inference
serverless_config = ServerlessInferenceConfig(
    memory_size_in_mb=2048,        # 1024, 2048, 3072, 4096, 5120, 6144 MB
    max_concurrency=5,              # Số request đồng thời tối đa (1-200)
    provisioned_concurrency=1,      # (Tùy chọn) Keep-warm để tránh cold start
)

# Deploy với Serverless config
predictor = model.deploy(
    serverless_inference_config=serverless_config,
)
```

### So Sánh: Khi Nào Dùng Gì?

| Loại Endpoint | Traffic | Latency Yêu Cầu | Chi Phí |
|--------------|---------|-----------------|---------|
| **Real-time** | Cao, ổn định | Thấp (< 100ms) | Cao nhưng predictable |
| **Serverless** | Thấp, sporadic (không đều) | Trung bình (chấp nhận cold start ~1-5s) | Thấp, pay-per-use |
| **Async Inference** | Trung bình, burst | Cao không quan trọng (job xử lý sau) | Trung bình |
| **Batch Transform** | Offline bulk | Không quan trọng | Thấp nhất |
| **MME** | Nhiều models nhỏ | Trung bình | Rất thấp per-model |

---

## SageMaker Savings Plans

**SageMaker Savings Plans** (Kế Hoạch Tiết Kiệm SageMaker) tương tự EC2 Savings Plans — cam kết sử dụng một mức nhất định (USD/giờ) trong 1 hoặc 3 năm để đổi lấy giảm giá 20-64%.

```
Ví dụ:
On-demand: ml.m5.xlarge = $0.23/giờ
1-year Savings Plan (no upfront): $0.164/giờ → tiết kiệm 28%
3-year Savings Plan (all upfront): $0.083/giờ → tiết kiệm 64%

Khi nào nên mua Savings Plans:
✅ Endpoint chạy ổn định 24/7 trong dài hạn
✅ Đã biết rõ workload pattern
❌ Workload mới, chưa biết sẽ scale thế nào
❌ Dùng Spot Training thường xuyên (Spot không áp dụng Savings Plans)
```

---

## Giám Sát & Phân Tích Chi Phí

### AWS Cost Explorer cho SageMaker

```python
import boto3

ce = boto3.client("ce", region_name="us-east-1")

# Lấy chi phí SageMaker theo service component trong 30 ngày qua
response = ce.get_cost_and_usage(
    TimePeriod={
        "Start": "2026-05-01",
        "End": "2026-06-01",
    },
    Granularity="MONTHLY",
    Filter={
        "Dimensions": {
            "Key": "SERVICE",
            "Values": ["Amazon SageMaker"],
        }
    },
    GroupBy=[
        {"Type": "DIMENSION", "Key": "USAGE_TYPE"},  # Chia theo loại sử dụng
    ],
    Metrics=["UnblendedCost"],
)

for group in response["ResultsByTime"][0]["Groups"]:
    usage_type = group["Keys"][0]
    cost = group["Metrics"]["UnblendedCost"]["Amount"]
    print(f"{usage_type}: ${float(cost):.2f}")
```

### Tag Strategy — Chiến Lược Tag Tài Nguyên

Sử dụng **tags** (nhãn) để phân bổ chi phí theo team, project, environment:

```python
estimator = Estimator(
    # ...
    tags=[
        {"Key": "Project", "Value": "recommendation-engine"},
        {"Key": "Team", "Value": "ml-platform"},
        {"Key": "Environment", "Value": "production"},
        {"Key": "CostCenter", "Value": "CC-1234"},
    ],
)
```

### SageMaker Cost Dashboard

Truy cập: **SageMaker Console → Governance → Cost dashboard**

Hiển thị:
- Chi phí theo training jobs, endpoints, notebooks
- Trend (xu hướng) 30/60/90 ngày
- Top 10 resource tốn kém nhất

---

## Checklist Tối Ưu Chi Phí

### Training Jobs

- [ ] **Bật Managed Spot Training** cho non-urgent jobs (tiết kiệm 60-90%)
- [ ] **Implement checkpoint** trong training code để recovery khi Spot interruption
- [ ] **Dùng FastFile Mode** cho data > 5GB để giảm startup time
- [ ] **Thử Distributed Training** cho large model thay vì 1 instance lớn đắt tiền
- [ ] **Dùng SageMaker Inference Recommender** để tìm instance type phù hợp
- [ ] **Xóa experiment artifact** không cần thiết trên S3

### Inference Endpoints

- [ ] **Xóa endpoint ngay** khi không cần dùng (rule #1 tránh lãng phí)
- [ ] **Đánh giá MME** nếu có nhiều models nhỏ cùng framework
- [ ] **Thử Serverless Inference** cho traffic thấp/không ổn định
- [ ] **Cân nhắc Inf1/Inf2 instances** cho model ổn định volume cao
- [ ] **Bật Auto Scaling** để scale down khi traffic thấp
- [ ] **Dùng Async Inference** cho non-real-time use cases

### Chung

- [ ] **Thiết lập AWS Budget alerts** khi chi phí vượt ngưỡng
- [ ] **Tag tất cả resources** để phân tích chi phí theo project/team
- [ ] **Xem xét SageMaker Savings Plans** nếu workload production ổn định
- [ ] **Dùng SageMaker Pricing Calculator** để ước tính trước khi scale

---

## 📌 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Làm thế nào để giảm chi phí SageMaker Training Job?**
> A: Sử dụng Managed Spot Training (tiết kiệm 60-90%), implement checkpoint để tránh mất progress khi Spot interruption, dùng FastFile Mode cho data lớn để giảm startup time, và xem xét Distributed Training để giảm wall-clock time.

**Q: Khi nào dùng Multi-Model Endpoint thay vì nhiều endpoints riêng?**
> A: Khi có nhiều models (5+) cùng framework nhưng không được gọi đồng thời tất cả, và traffic của mỗi model không đủ lớn để justify một dedicated endpoint. MME phù hợp cho multi-tenant applications (ứng dụng đa khách hàng) với model per-customer.

**Q: Serverless Inference phù hợp với use case nào?**
> A: Phù hợp nhất cho traffic thấp hoặc intermittent (không đều), không cần SLA latency thấp (chấp nhận cold start 1-5 giây), và muốn zero-cost khi không có traffic. Không phù hợp cho real-time API cần P99 latency < 200ms.

**Q: Inf2 instance có ưu điểm gì so với GPU instance cho inference?**
> A: Inf2 rẻ hơn 40-70% so với GPU tương đương, throughput cao hơn cho model phổ biến (BERT, ResNet, Llama 2). Nhược điểm: cần compile model sang AWS Neuron format trước khi deploy, không flexible bằng GPU khi thay đổi model thường xuyên.

---

**Liên Kết Liên Quan:**
- [Multi-Model Endpoint Chi Tiết](../02-sagemaker/3-sagemaker-inference.md)
- [SageMaker Training Jobs](../02-sagemaker/2-sagemaker-training.md)
- [SageMaker Inference Recommender](../02-sagemaker/3-sagemaker-inference.md)

**Cập Nhật Lần Cuối:** 2026-06-03
