# SageMaker Studio — IDE (Môi Trường Phát Triển Tích Hợp) cho ML

> SageMaker Studio là IDE (Integrated Development Environment — Môi Trường Phát Triển Tích Hợp) web-based đầu tiên được thiết kế chuyên biệt cho toàn bộ Machine Learning workflow — từ chuẩn bị dữ liệu, viết code, training, đến debugging và deployment, tất cả trong một giao diện thống nhất.

---

## 📚 Mục Lục

1. [SageMaker Studio là gì?](#sagemaker-studio-là-gì)
2. [Domain và User Profile](#domain-và-user-profile)
3. [JupyterLab và Kernel](#jupyterlab-và-kernel)
4. [SageMaker Studio Components](#sagemaker-studio-components)
5. [Pricing và Chi Phí](#pricing-và-chi-phí)
6. [Hands-on: Tạo Domain và Notebook](#hands-on)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## SageMaker Studio Là Gì?

SageMaker Studio là giao diện web duy nhất (single pane of glass) để làm việc với toàn bộ SageMaker ecosystem:

```
Browser (Trình Duyệt)
    │
    ▼
SageMaker Studio UI (JupyterLab-based)
    │
    ├── Notebooks                → Viết Python/R code
    ├── Experiments (Thí Nghiệm) → So sánh training runs
    ├── Pipelines (Đường Ống)    → Xây ML workflows tự động
    ├── Models (Mô Hình)         → Xem Model Registry
    ├── Endpoints (Điểm Đầu Vào)→ Quản lý deployments
    ├── Feature Store            → Xem/tạo features
    └── Data Wrangler            → Visual data transformation
```

### Ưu Điểm So Với Jupyter Local

| Tính Năng | Jupyter Local | SageMaker Studio |
|---|---|---|
| **Compute** (Tính Toán) | Laptop CPU/RAM | Bất kỳ instance AWS |
| **Collaboration** (Cộng Tác) | Không có | Share notebook, Git integration |
| **Training Jobs** | Chạy trong notebook | Submit managed training jobs |
| **Experiment Tracking** (Theo Dõi Thí Nghiệm) | Tự quản lý | Tích hợp sẵn |
| **Cost** (Chi Phí) | Phần cứng laptop | Trả theo giờ dùng |
| **GPU Access** | Cần mua | Thuê theo nhu cầu |

---

## Domain và User Profile

### SageMaker Domain

**Domain** là môi trường SageMaker Studio được tạo trong một AWS account, trong một Region (Vùng Địa Lý) cụ thể.

```
AWS Account
└── Region (VD: us-east-1)
    └── SageMaker Domain  ← MỖI REGION CÓ TỐI ĐA 1 DOMAIN (mặc định)
        ├── Domain ID: d-xxxxxxxxxx
        ├── Auth Mode: IAM hoặc IAM Identity Center (SSO)
        ├── VPC Settings: Public hoặc VPC-only
        ├── EFS Volume: Shared storage cho tất cả users
        └── User Profiles
            ├── User A (alice)
            ├── User B (bob)
            └── User C (charlie)
```

**Các Thành Phần Của Domain:**
- **EFS Volume** (Amazon Elastic File System — Hệ Thống File Đàn Hồi): Shared storage, mỗi user có thư mục riêng `/home/user-profile-name/`
- **VPC Configuration**: Kiểm soát network access cho Studio containers
- **Execution Role** (Vai Trò Thực Thi): IAM Role mặc định áp dụng cho tất cả users (override được ở User Profile)

### User Profile

Mỗi **User Profile** đại diện cho một người dùng trong Domain:

```python
# Tạo User Profile qua AWS CLI
aws sagemaker create-user-profile \
    --domain-id d-xxxxxxxxxx \
    --user-profile-name alice \
    --user-settings '{
        "ExecutionRole": "arn:aws:iam::123456789012:role/alice-sagemaker-role",
        "JupyterServerAppSettings": {
            "DefaultResourceSpec": {
                "InstanceType": "system"
            }
        }
    }'
```

**Phân Quyền User Profile:**
- Mỗi User Profile có IAM Execution Role riêng
- Role quyết định user được phép làm gì (đọc S3 bucket nào, chạy training job không, v.v.)
- Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu): chỉ cấp quyền thực sự cần thiết

### Auth Modes (Chế Độ Xác Thực)

| Mode | Mô Tả | Phù Hợp |
|---|---|---|
| **IAM** | Mỗi user login bằng IAM user/role | Nhỏ lẻ, lab |
| **IAM Identity Center** (SSO) | Single Sign-On qua AWS IAM Identity Center | Doanh nghiệp, nhiều users |

---

## JupyterLab và Kernel

### JupyterLab

SageMaker Studio dùng **JupyterLab** (phiên bản 3.x trở lên) làm IDE. Khi mở Studio, JupyterLab chạy trên một managed instance.

**Jupyter Server App** (Ứng Dụng Jupyter Server):
- Một light-weight instance luôn chạy khi Studio open
- Instance type: `system` (miễn phí) — chỉ chạy JupyterLab UI, không dùng cho tính toán nặng

### Kernel và Compute

**Kernel** là môi trường thực thi code trong notebook. Mỗi kernel chạy trên **Kernel Gateway App** — một container riêng trên instance mà bạn chọn.

```
SageMaker Studio UI (Browser)
    │
    ▼
Jupyter Server App (ml.t3.medium — luôn chạy)
    │ (kết nối kernel)
    ▼
Kernel Gateway App (Instance bạn chọn)
    ├── ml.t3.medium   → Phân tích nhẹ, viết code
    ├── ml.m5.4xlarge  → Xử lý data lớn
    └── ml.g4dn.xlarge → GPU, chạy deep learning model nhỏ
```

**Các Loại Kernel Phổ Biến:**

| Kernel Image | Framework | Dùng Cho |
|---|---|---|
| `Data Science 3.0` | Scikit-learn, Pandas, Numpy | General ML, tabular data |
| `TensorFlow 2.x` | TensorFlow, Keras | Deep learning |
| `PyTorch 2.x` | PyTorch | Deep learning, research |
| `MXNet 1.9` | Apache MXNet | Production inference |
| `Custom Image` | Bất kỳ | Docker image tùy chỉnh |

### Lifecycle Configurations (Cấu Hình Vòng Đời)

**Lifecycle Configuration** (LC) là shell script chạy tự động khi kernel khởi động — dùng để cài thư viện, set environment variables (biến môi trường), v.v.

```bash
#!/bin/bash
# Ví dụ Lifecycle Configuration cho Kernel Gateway
# Script này chạy khi kernel gateway app khởi động

set -eux

# Cài thêm thư viện Python
pip install --quiet \
    transformers==4.35.0 \
    datasets \
    boto3 \
    sagemaker

# Cài extension JupyterLab
jupyter labextension install @jupyter-widgets/jupyterlab-manager --no-build

echo "Lifecycle configuration completed!"
```

---

## SageMaker Studio Components

### 1. SageMaker Data Wrangler (Công Cụ Xử Lý Dữ Liệu Trực Quan)

Visual tool để:
- Import data từ S3, Athena, Redshift, Feature Store
- Apply 300+ built-in data transformations (biến đổi dữ liệu)
- Profile data (thống kê cơ bản, phân phối, outliers)
- Export pipeline thành Python code hoặc SageMaker Processing Job

```
Data Sources              Data Wrangler              Output
S3 ──────────────────────►                          ──► S3 (cleaned data)
Athena ──────────────────►  (Visual Pipeline)       ──► Feature Store
Redshift ────────────────►                          ──► SageMaker Pipeline
```

### 2. SageMaker Experiments (Theo Dõi Thí Nghiệm)

Tự động theo dõi và so sánh các training runs:

```python
import sagemaker
from sagemaker.experiments.run import Run

# Tạo và track experiment run
with Run(
    experiment_name="churn-prediction-v2",
    run_name="xgboost-depth5-eta02",
    sagemaker_session=session
) as run:
    # Log hyperparameters (siêu tham số)
    run.log_parameters({
        "max_depth": 5,
        "eta": 0.2,
        "n_estimators": 100
    })
    
    # Train model...
    model.fit(X_train, y_train)
    
    # Log metrics (chỉ số đánh giá)
    run.log_metric("train_accuracy", 0.94)
    run.log_metric("val_accuracy", 0.91)
    run.log_metric("auc_roc", 0.96)
```

**Studio UI** sẽ hiển thị bảng so sánh tất cả runs, giúp chọn model tốt nhất nhanh chóng.

### 3. SageMaker Debugger (Công Cụ Gỡ Lỗi Huấn Luyện)

Phát hiện và debug các vấn đề trong quá trình training mà không cần dừng job:

| Rule (Quy Tắc) | Phát Hiện |
|---|---|
| `vanishing_gradient` | Gradient tiến về 0, model không học được |
| `exploding_gradient` | Gradient quá lớn, NaN/Inf values |
| `overfit` | Val loss tăng khi train loss giảm |
| `poor_weight_initialization` | Khởi tạo weight không tốt |
| `class_imbalance` | Mất cân bằng class trong dữ liệu |

```python
from sagemaker.debugger import Rule, rule_configs

estimator = TensorFlow(
    ...,
    rules=[
        Rule.sagemaker(rule_configs.vanishing_gradient()),
        Rule.sagemaker(rule_configs.overfit()),
        Rule.sagemaker(rule_configs.loss_not_decreasing())
    ]
)
```

### 4. SageMaker Canvas (Học Máy Không Cần Code)

No-code ML dành cho business users (người dùng nghiệp vụ):
- Upload CSV, chọn cột target (mục tiêu dự đoán)
- SageMaker tự chọn model, train và evaluate
- Export predictions (dự đoán) ra CSV
- Kết nối với Tableau, QuickSight để visualize

### 5. SageMaker JumpStart (Bắt Đầu Nhanh)

Model hub tích hợp trong Studio:
- 300+ pre-trained models từ Hugging Face, PyTorch Hub, v.v.
- 1-click fine-tuning (tinh chỉnh) trên dữ liệu của bạn
- 1-click deployment lên SageMaker endpoint
- Foundation Models: Llama 2, Falcon, Stable Diffusion

---

## Pricing và Chi Phí

### Các Khoản Tính Phí Trong Studio

```
Chi Phí Studio = Jupyter Server App + Kernel Gateway Apps + Storage

Jupyter Server App:
  - Instance "system" → MIỄN PHÍ (AWS cung cấp)
  - ml.t3.medium → ~$0.0464/giờ (nếu chọn custom)

Kernel Gateway App (tính phí theo giờ instance chạy):
  - ml.t3.medium:   ~$0.0464/giờ
  - ml.m5.4xlarge:  ~$0.922/giờ
  - ml.g4dn.xlarge: ~$0.736/giờ (GPU T4)

EFS Storage (Lưu Trữ):
  - ~$0.30/GB/tháng cho home directories
```

> ⚠️ **Chi Phí Ẩn Thường Gặp:** Kernel Gateway App tiếp tục chạy (và tính phí) dù bạn đóng tab browser! Phải **Shut Down** (Tắt) app thủ công trong Studio → File → Shut Down.

### Tối Ưu Chi Phí Studio

1. **Shut Down Kernels When Not In Use**: Studio → Running Terminals and Kernels → Stop tất cả
2. **Dùng JumpStart Notebooks**: Tự động configure instance phù hợp nhất
3. **Auto-Shutdown Extension**: Cài extension tự động tắt kernel sau N phút idle
4. **Chọn Instance Phù Hợp**: Dùng `ml.t3.medium` cho code-writing, chỉ switch GPU khi cần training

---

## Hands-on: Tạo Domain và Chạy Notebook Đầu Tiên {#hands-on}

### Bước 1: Tạo SageMaker Domain

```bash
# Qua AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)
aws sagemaker create-domain \
    --domain-name "my-ml-domain" \
    --auth-mode IAM \
    --default-user-settings '{
        "ExecutionRole": "arn:aws:iam::ACCOUNT_ID:role/SageMakerExecutionRole"
    }' \
    --subnet-ids subnet-xxxx \
    --vpc-id vpc-xxxx
```

Hoặc qua AWS Console:
1. Vào SageMaker → Domains → Create Domain
2. Chọn "Quick setup" (Cài Đặt Nhanh) để tự động tạo VPC và Role

### Bước 2: Mở Studio

```bash
# Lấy pre-signed URL để vào Studio
aws sagemaker create-presigned-domain-url \
    --domain-id d-xxxxxxxxxx \
    --user-profile-name alice
```

Output trả về URL, mở trong browser để vào Studio.

### Bước 3: Notebook Đầu Tiên — Train Sklearn Model

```python
# Cell 1: Import libraries
import pandas as pd
import numpy as np
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
import sagemaker
import boto3

# Cell 2: Load và split data
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42
)

# Cell 3: Train model (chạy ngay trong notebook kernel — KHÔNG phải training job)
clf = RandomForestClassifier(n_estimators=100)
clf.fit(X_train, y_train)
print(f"Accuracy: {accuracy_score(y_test, clf.predict(X_test)):.2f}")

# Cell 4: Để train job lớn hơn, submit SageMaker Training Job thay vì chạy trực tiếp
# (Xem file 2-sagemaker-training.md)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: SageMaker Studio khác gì Jupyter Notebook thông thường?

**Trả lời tốt:**
> SageMaker Studio là IDE web-based được tích hợp trực tiếp với toàn bộ SageMaker ecosystem. Khác với Jupyter thông thường, Studio cho phép submit training jobs lên managed infrastructure thay vì chạy trong notebook — có nghĩa là tôi có thể viết code trên `ml.t3.medium` rẻ tiền nhưng submit training job lên `ml.p3.16xlarge` với 8 GPU. Ngoài ra, Studio tích hợp sẵn Experiments tracking, Model Registry, Pipelines và Debugger — những thứ này tôi phải tự setup nếu dùng Jupyter thuần. Cho production ML workflow, Studio tiết kiệm setup time đáng kể.

### Q2: Domain và User Profile là gì? Tại sao cần phân tách?

**Trả lời tốt:**
> Domain là môi trường SageMaker Studio cấp tổ chức — có shared EFS storage, VPC configuration, và default permissions. User Profile là account cá nhân trong Domain, mỗi user có IAM Execution Role riêng. Lý do phân tách: Data Scientist thực nghiệm cần quyền đọc nhiều S3 buckets, trong khi Production Engineer chỉ cần deploy models. Tách User Profile cho phép least-privilege IAM — mỗi người chỉ có quyền tối thiểu cần thiết. Trong thực tế, tôi sẽ tạo một Domain per team, và User Profile per member, với IAM Role tương ứng role của họ.

### Q3: Kernel Gateway App tính phí như thế nào? Làm sao tránh chi phí không mong muốn?

**Trả lời tốt:**
> Kernel Gateway App tính phí theo giờ instance chạy, kể cả khi bạn đóng tab browser! Đây là lỗi phổ biến nhất khiến hóa đơn AWS phát sinh ngoài ý muốn. Để tránh, tôi làm 3 việc: (1) Luôn Shut Down kernel trong Studio → Running Terminals and Kernels khi xong việc, (2) Cài auto-shutdown extension tự động tắt sau 60 phút idle, (3) Dùng lifecycle configuration để set idle timeout. Ngoài ra, chọn đúng instance size cũng quan trọng: viết code dùng `ml.t3.medium` ($0.046/giờ), chỉ switch `ml.g4dn.xlarge` ($0.74/giờ) khi thực sự cần GPU.

---

## 📊 Tóm Tắt

```
SageMaker Studio
├── Domain (1 per Region)
│   ├── EFS Shared Storage
│   ├── VPC Configuration
│   └── User Profiles (1 per person)
│       ├── IAM Execution Role
│       └── JupyterLab Environment
│           ├── Jupyter Server App (miễn phí "system" instance)
│           └── Kernel Gateway Apps (TÍNH PHÍ — nhớ shut down!)
│
├── Built-in Tools
│   ├── Data Wrangler (Visual ETL — Extract Transform Load)
│   ├── Experiments (Tracking training runs)
│   ├── Debugger (Phát hiện training issues)
│   ├── Canvas (No-code ML)
│   └── JumpStart (Pre-trained model hub)
```

---

**File tiếp theo:** [2-sagemaker-training.md](./2-sagemaker-training.md) — Training Jobs, Built-in Algorithms, Spot Training

**Cập Nhật Lần Cuối:** 2026-06-03
