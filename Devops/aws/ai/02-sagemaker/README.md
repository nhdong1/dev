# Amazon SageMaker — ML Platform (Nền Tảng Học Máy) Toàn Diện

> Amazon SageMaker là nền tảng ML (Machine Learning — Học Máy) fully-managed (được quản lý hoàn toàn) của AWS, cung cấp mọi công cụ cần thiết để xây dựng, huấn luyện (train), điều chỉnh (tune) và triển khai (deploy) mô hình học máy ở quy mô sản xuất (production scale) — không cần quản lý hạ tầng.

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| `README.md` | Tổng quan SageMaker, kiến trúc, so sánh components | ✅ |
| `1-sagemaker-studio.md` | Studio IDE, Domain, User Profile, JupyterLab | ✅ |
| `2-sagemaker-training.md` | Training Job, Built-in algorithms, Distributed training, Spot | ✅ |
| `3-sagemaker-inference.md` | Real-time, Batch Transform, Async, Serverless Inference | ✅ |
| `4-sagemaker-autopilot.md` | AutoML, algorithm selection, HPO tự động, Explainability | ✅ |
| `5-sagemaker-feature-store.md` | Online/Offline Feature Store, Feature Group, Ingestion | ✅ |

---

## 🎯 SageMaker Là Gì?

Amazon SageMaker là dịch vụ ML platform (nền tảng học máy) fully-managed ra mắt năm 2017, cho phép:

- **Build** (Xây Dựng): Chuẩn bị dữ liệu, tạo features, viết code ML trong môi trường IDE tích hợp
- **Train** (Huấn Luyện): Chạy training jobs trên managed infrastructure với built-in hoặc custom algorithms
- **Tune** (Tinh Chỉnh): Tối ưu hyperparameters (siêu tham số) tự động với AMT
- **Deploy** (Triển Khai): Host model trên endpoints với nhiều inference modes
- **Monitor** (Giám Sát): Phát hiện data drift (trôi dạt dữ liệu) và model degradation (suy giảm mô hình)

### Vị Trí Trong Hệ Sinh Thái AWS AI

```
┌─────────────────────────────────────────────────────┐
│  Tầng 3: AI Services (Dịch Vụ AI Được Quản Lý)     │
│  Rekognition │ Comprehend │ Polly │ Transcribe │ Lex │
├─────────────────────────────────────────────────────┤
│  Tầng 2: ML Services (Dịch Vụ ML)                  │
│               Amazon SageMaker  ◄── BẠN ĐANG Ở ĐÂY │
├─────────────────────────────────────────────────────┤
│  Tầng 1: ML Framework & Infrastructure              │
│  TensorFlow │ PyTorch │ MXNet │ EC2 │ GPU Instances  │
└─────────────────────────────────────────────────────┘
```

**Quy tắc chọn SageMaker:** Khi AI Services không đáp ứng được yêu cầu (cần custom model, cần kiểm soát hoàn toàn model architecture, dữ liệu đặc thù của doanh nghiệp).

---

## 🏗️ Kiến Trúc SageMaker

### Các Thành Phần Chính

```
                    ┌─────────────────────┐
                    │   SageMaker Studio  │  ← IDE (Môi Trường Phát Triển Tích Hợp)
                    │   (Môi Trường IDE)  │    JupyterLab, RStudio, VS Code
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌───────────────┐  ┌─────────────────┐  ┌────────────────┐
  │   Processing  │  │    Training     │  │   Inference    │
  │     Jobs      │  │     Jobs        │  │   Endpoints    │
  │ (Tiền Xử Lý) │  │  (Huấn Luyện)  │  │  (Triển Khai)  │
  └───────┬───────┘  └────────┬────────┘  └───────┬────────┘
          │                   │                   │
          └────────────┬──────┘                   │
                       ▼                          │
              ┌─────────────────┐                 │
              │  SageMaker      │                 │
              │  Pipelines      │─────────────────┘
              │ (ML CI/CD)      │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
  ┌──────────────┐ ┌──────────┐ ┌──────────────┐
  │Model Registry│ │Clarify   │ │Model Monitor │
  │(Kho Mô Hình) │ │(Giải Thích│ │(Giám Sát)    │
  └──────────────┘ └──────────┘ └──────────────┘
```

### Data Flow (Luồng Dữ Liệu) Điển Hình

```
Amazon S3          SageMaker          SageMaker         Application
(Lưu Trữ)         Training           Model             (Ứng Dụng)
     │                 │                 │                  │
     │── Raw Data ────▶│                 │                  │
     │                 │── Training ────▶│                  │
     │◀── Model ───────│                 │                  │
     │                 │                 │                  │
     │── Model ────────────────────────▶│                  │
     │                 │                 │── Endpoint ─────▶│
     │                 │                 │◀── Request ──────│
     │                 │                 │─── Prediction ──▶│
```

---

## 📦 Các Components Chính

### 1. SageMaker Studio (Studio Học Máy)

IDE web-based đầy đủ tính năng cho toàn bộ ML workflow:

- **JupyterLab**: Viết code Python/R để phân tích data và build model
- **Domain**: Môi trường SageMaker Studio của một team/organization
- **User Profile**: Account cá nhân trong Domain với IAM Role riêng
- **Canvas**: No-code ML (Học Máy Không Cần Lập Trình) cho business users

### 2. SageMaker Training (Huấn Luyện)

Chạy training jobs trên managed infrastructure:

- **Built-in Algorithms** (Thuật Toán Tích Hợp): XGBoost, Linear Learner, K-Means, v.v.
- **Custom Containers**: Mang Docker image riêng (BYOC — Bring Your Own Container)
- **Distributed Training** (Huấn Luyện Phân Tán): Data parallelism, Model parallelism
- **Managed Spot Training** (Huấn Luyện Spot): Tiết kiệm đến 90% chi phí

### 3. SageMaker Inference (Suy Luận / Dự Đoán)

Bốn loại inference phù hợp từng use case:

| Loại | Độ Trễ | Chi Phí | Use Case |
|---|---|---|---|
| **Real-time** (Thời Gian Thực) | <100ms | Cao | APIs, interactive apps |
| **Batch Transform** (Biến Đổi Theo Lô) | Phút-giờ | Thấp | Xử lý file lớn offline |
| **Async** (Bất Đồng Bộ) | Giây-phút | Trung bình | File lớn, xử lý dài |
| **Serverless** (Không Máy Chủ) | Trung bình | Rất thấp | Traffic thưa, unpredictable |

### 4. SageMaker Autopilot (Học Máy Tự Động)

AutoML (Automated Machine Learning — Học Máy Tự Động) end-to-end:
- Tự động phân tích data, chọn thuật toán, tinh chỉnh hyperparameters
- Tạo notebooks giải thích toàn bộ quy trình (explainability)
- Phù hợp cho data scientists muốn baseline nhanh

### 5. SageMaker Feature Store (Kho Đặc Trưng)

Quản lý tập trung features (đặc trưng) giữa các model:
- **Online Store** (Kho Trực Tuyến): Độ trễ thấp, phục vụ real-time inference
- **Offline Store** (Kho Ngoại Tuyến): S3-backed, phục vụ training và batch inference

---

## ⚡ SageMaker vs Các Dịch Vụ AI Khác

| Tiêu Chí | SageMaker | AI Services (Rekognition, Comprehend…) |
|---|---|---|
| **Kiểm Soát Model** | Hoàn toàn | Không (AWS quản lý) |
| **Dữ Liệu Đào Tạo** | Dữ liệu của bạn | Mô hình được pre-trained (đã huấn luyện sẵn) |
| **Chuyên Môn ML** | Cần expertise | Không cần |
| **Thời Gian Setup** | Tuần-tháng | Giờ |
| **Chi Phí** | Cao hơn, linh hoạt | Thấp hơn, theo dùng |
| **Tùy Biến** | Vô hạn | Hạn chế (Custom Labels, Custom Classification) |
| **Use Case** | Domain-specific, proprietary data | General purpose, quick deployment |

**Quy Tắc Ngón Tay Cái:**
- AI Services → Dùng khi bài toán phổ biến, không có dữ liệu đặc thù
- SageMaker → Dùng khi cần custom model, dữ liệu độc quyền, hoặc AI Services không đủ accuracy

---

## 🔧 Các SageMaker Instance Types (Loại Máy Chủ)

### Training Instances

| Family | GPU/CPU | Dùng Cho |
|---|---|---|
| `ml.m5.xlarge` | CPU | Nhẹ, tabular data |
| `ml.p3.2xlarge` | GPU (V100) | Deep learning training |
| `ml.p4d.24xlarge` | GPU (A100 x8) | Large model training |
| `ml.trn1.2xlarge` | Trainium | Cost-efficient deep learning |

### Inference Instances

| Family | GPU/CPU | Dùng Cho |
|---|---|---|
| `ml.t3.medium` | CPU | Dev/test, low traffic |
| `ml.c5.xlarge` | CPU | Production, high throughput |
| `ml.g4dn.xlarge` | GPU (T4) | GPU inference, NLP models |
| `ml.inf2.xlarge` | Inferentia2 | Cost-efficient production inference |

**Mẹo Chi Phí:** `ml.inf2` (Inferentia2 — chip suy luận của AWS) rẻ hơn GPU 40-70% cho inference workloads phổ biến.

---

## 💰 Mô Hình Tính Phí SageMaker

### Các Nguồn Chi Phí Chính

```
Chi Phí SageMaker = Training + Hosting + Storage + Processing

Training:   Số giờ × giá instance (VD: ml.p3.2xlarge ~ $3.06/giờ)
Hosting:    Số giờ × giá endpoint instance (tính 24/7 kể cả idle)
Storage:    EBS volume gắn với training instance
S3 Storage: Lưu trữ data, model artifacts
```

### Chiến Lược Tiết Kiệm Chi Phí

1. **Spot Training** (Huấn Luyện Spot): Dùng EC2 Spot Instances, tiết kiệm đến 90% — phải xử lý interruption (gián đoạn)
2. **Serverless Inference**: Cho traffic thưa — trả theo request, không tốn phí khi idle
3. **Multi-Model Endpoint** (Endpoint Đa Mô Hình): Nhiều model trên 1 endpoint, chia sẻ infrastructure
4. **Elastic Inference** (Suy Luận Đàn Hồi): Gắn GPU nhỏ vào CPU instance thay vì thuê full GPU instance
5. **Inference Recommender**: Tool tự động benchmarking instance phù hợp nhất về giá/performance

> ⚠️ **Cảnh báo chi phí quan trọng:** SageMaker real-time endpoint tính phí 24/7 kể cả khi KHÔNG có request. Luôn xóa endpoint sau khi thực hành!

---

## 🔐 Security (Bảo Mật) SageMaker

### Các Lớp Bảo Mật

1. **IAM Roles** (Vai Trò IAM): Mỗi training job, endpoint cần IAM Role với quyền tối thiểu
2. **VPC** (Virtual Private Cloud — Đám Mây Riêng Ảo): Chạy SageMaker trong VPC để tách biệt network
3. **Encryption at Rest** (Mã Hóa Lúc Lưu Trữ): KMS keys cho S3, EBS, EFS
4. **Encryption in Transit** (Mã Hóa Khi Truyền): TLS cho data movement
5. **Private Link** (Liên Kết Riêng Tư): Truy cập SageMaker API qua VPC Endpoint, không qua internet
6. **Network Isolation** (Cô Lập Mạng): Training container không có internet access

### IAM Best Practices

```json
{
  "Permissions cần thiết cho Training Job": [
    "s3:GetObject (đọc training data)",
    "s3:PutObject (lưu model artifacts)",
    "ecr:GetDownloadUrlForLayer (pull Docker image)",
    "logs:CreateLogGroup (ghi CloudWatch logs)"
  ]
}
```

---

## 🚀 Quick Start — Bắt Đầu Nhanh

### Ví Dụ Đơn Giản: Train XGBoost Model

```python
import boto3
import sagemaker
from sagemaker.xgboost import XGBoost

# Khởi tạo SageMaker session
session = sagemaker.Session()
role = sagemaker.get_execution_role()  # IAM Role cho SageMaker

# Định nghĩa estimator (bộ ước lượng)
estimator = XGBoost(
    entry_point='train.py',        # Script huấn luyện của bạn
    framework_version='1.5-1',
    instance_type='ml.m5.xlarge',  # Loại instance
    instance_count=1,
    role=role,
    hyperparameters={
        'max_depth': 5,
        'eta': 0.2,
        'objective': 'binary:logistic',
        'num_round': 100
    }
)

# Bắt đầu training job
estimator.fit({
    'train': 's3://my-bucket/train/',   # Dữ liệu huấn luyện trên S3
    'validation': 's3://my-bucket/val/' # Dữ liệu kiểm tra
})

# Deploy lên real-time endpoint
predictor = estimator.deploy(
    initial_instance_count=1,
    instance_type='ml.c5.xlarge'
)

# Gọi prediction
result = predictor.predict([[5.1, 3.5, 1.4, 0.2]])
print(result)

# XÓA ENDPOINT SAU KHI XONG để tránh tốn phí!
predictor.delete_endpoint()
```

---

## 📋 Checklist Nắm Vững SageMaker

### Nền Tảng
- [ ] Giải thích SageMaker là gì và khi nào cần dùng (vs AI Services)
- [ ] Hiểu luồng data: S3 → Training → Model Artifacts → S3 → Endpoint
- [ ] Biết các instance types phổ biến và chi phí tương đối
- [ ] Nắm quy tắc bảo mật: IAM Role, VPC, encryption

### Training
- [ ] Chạy training job với built-in XGBoost algorithm
- [ ] Sử dụng Spot Training để tiết kiệm chi phí
- [ ] Hiểu Managed Spot Training checkpoint và interruption handling
- [ ] Biết khi nào dùng distributed training

### Inference
- [ ] Deploy real-time endpoint và gọi từ Python
- [ ] Sử dụng Batch Transform cho xử lý file S3 lớn
- [ ] Biết sự khác nhau giữa Real-time / Async / Serverless / Batch
- [ ] Multi-model endpoint — tại sao và khi nào dùng

### Nâng Cao
- [ ] SageMaker Pipelines — CI/CD cho ML
- [ ] SageMaker Model Monitor — phát hiện data drift
- [ ] SageMaker Clarify — bias detection và explainability
- [ ] Feature Store — khi nào thực sự cần

---

## 🔗 Điều Hướng

| Chủ Đề | File |
|---|---|
| Studio IDE | [1-sagemaker-studio.md](./1-sagemaker-studio.md) |
| Training Jobs | [2-sagemaker-training.md](./2-sagemaker-training.md) |
| Inference & Endpoints | [3-sagemaker-inference.md](./3-sagemaker-inference.md) |
| AutoML / Autopilot | [4-sagemaker-autopilot.md](./4-sagemaker-autopilot.md) |
| Feature Store | [5-sagemaker-feature-store.md](./5-sagemaker-feature-store.md) |
| Module trước: Nền Tảng AI/ML | [../01-fundamentals/](../01-fundamentals/) |
| Module tiếp: Bedrock | [../03-bedrock/](../03-bedrock/) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
