# 🚀 Module 10: Advanced AWS AI/ML — Nâng Cao & Kiến Trúc

> Kiến thức nâng cao về tối ưu chi phí (Cost Optimization), bảo mật tuân thủ (Security & Compliance), AI Có Trách Nhiệm (Responsible AI) và các mô hình kiến trúc ML production-grade trên AWS

## 📚 Mục Lục Module

| File | Chủ Đề | Trạng Thái |
|------|---------|------------|
| [1-cost-optimization.md](./1-cost-optimization.md) | Tối Ưu Chi Phí: Spot, Inf2, Multi-model endpoint | ✅ |
| [2-security-compliance.md](./2-security-compliance.md) | Bảo Mật & Tuân Thủ: VPC, IAM, Encryption, Audit | ✅ |
| [3-responsible-ai.md](./3-responsible-ai.md) | AI Có Trách Nhiệm: Fairness, Transparency, Privacy | ✅ |
| [4-ml-architecture-patterns.md](./4-ml-architecture-patterns.md) | Mẫu Kiến Trúc ML: Batch, Online, Shadow, Canary | ✅ |

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có khả năng:

- **Tối ưu chi phí** SageMaker training và inference tối thiểu 50-80% bằng Spot Instances, Managed Spot Training, và Multi-model endpoints
- **Thiết kế kiến trúc bảo mật** cho ML workload với VPC isolation, Private Link, IAM least-privilege và encryption end-to-end
- **Triển khai Responsible AI** framework bao gồm phát hiện bias, explainability và data privacy trong pipeline ML production
- **Chọn đúng kiến trúc pattern** (Batch Inference, Online Serving, Shadow Deployment, Canary Release) cho từng use case

---

## 📊 Tổng Quan Nội Dung

### 1. Cost Optimization — Tối Ưu Chi Phí

**Vấn đề:** SageMaker Training Job trên instance ml.p3.2xlarge tốn ~3.8 USD/giờ. Nếu chạy 100 giờ/tháng = 380 USD/tháng. Với Spot Training có thể tiết kiệm 60-90%.

**Các chiến lược chính:**

```
Training Cost Optimization:
├── Managed Spot Training          Tiết kiệm 60-90% chi phí training
├── Pipe Mode / FastFile Mode      Giảm thời gian I/O data loading
├── SageMaker Savings Plans        Cam kết sử dụng để giảm giá
└── Distributed Training           Training song song để rút ngắn thời gian

Inference Cost Optimization:
├── Multi-Model Endpoint (MME)     Nhiều model chạy trên 1 endpoint
├── Multi-Container Endpoint       Nhiều container trong 1 endpoint
├── Serverless Inference           Pay-per-request, không trả phí idle
├── Inf2 / Inf1 Instances          AWS Inferentia chip — inference giá rẻ
└── Auto Scaling                   Scale down về 0 khi không có traffic
```

### 2. Security & Compliance — Bảo Mật & Tuân Thủ

**Ba trụ cột bảo mật ML trên AWS:**

```
Identity & Access (Danh Tính & Quyền Truy Cập):
├── IAM Roles for SageMaker        Mỗi component có role riêng
├── Resource-based Policies        Giới hạn truy cập vào S3, ECR
└── Service Control Policies       Kiểm soát ở cấp Organization

Network Isolation (Cô Lập Mạng):
├── VPC-only Training              Training không ra internet
├── PrivateLink Endpoints          Kết nối dịch vụ AWS qua mạng nội bộ
└── Security Groups & NACLs        Tường lửa cấp instance và subnet

Data Protection (Bảo Vệ Dữ Liệu):
├── Encryption at Rest             KMS key cho S3, EBS, EFS
├── Encryption in Transit          TLS cho tất cả API calls
└── SageMaker Data Wrangler        Phát hiện PII trong dataset training
```

### 3. Responsible AI — AI Có Trách Nhiệm

**Bốn nguyên tắc AWS Responsible AI:**

| Nguyên Tắc | Mô Tả | Công Cụ AWS |
|------------|-------|-------------|
| **Fairness — Công Bằng** | Model không phân biệt đối xử | SageMaker Clarify |
| **Explainability — Khả Năng Giải Thích** | Hiểu model ra quyết định thế nào | SageMaker Clarify + SHAP |
| **Privacy — Quyền Riêng Tư** | Bảo vệ dữ liệu cá nhân | Comprehend PII, Macie |
| **Robustness — Tính Bền Vững** | Model hoạt động ổn định, chống adversarial | SageMaker Model Monitor |

### 4. ML Architecture Patterns — Mẫu Kiến Trúc ML

**Chọn pattern phù hợp:**

```
Theo yêu cầu latency (độ trễ):
├── Real-time (< 100ms)     → SageMaker Real-time Endpoint
├── Near-real-time (< 5s)   → SageMaker Async Inference
├── Batch (giờ/ngày)        → SageMaker Batch Transform

Theo chiến lược deployment (triển khai):
├── Blue/Green Deployment   → SageMaker Endpoint Update
├── Shadow Deployment       → SageMaker Shadow Testing
├── Canary Release          → SageMaker Traffic Shifting
└── A/B Testing             → SageMaker Multi-variant Endpoint
```

---

## 🔗 Liên Kết Module

| Phụ Thuộc Vào | Module |
|---------------|--------|
| Kiến thức training cần có | [02-sagemaker/](../02-sagemaker/) |
| MLOps pipeline | [09-mlops/](../09-mlops/) |
| Bias & Explainability chi tiết | [09-mlops/4-sagemaker-clarify.md](../09-mlops/4-sagemaker-clarify.md) |
| Model Monitor | [09-mlops/3-model-monitor.md](../09-mlops/3-model-monitor.md) |

---

## ⏱️ Thời Gian Học

| File | Thời Gian Đọc | Thực Hành |
|------|---------------|-----------|
| 1-cost-optimization.md | 45 phút | 60 phút (thử MME trên console) |
| 2-security-compliance.md | 60 phút | 60 phút (tạo VPC endpoint) |
| 3-responsible-ai.md | 45 phút | 30 phút (chạy Clarify job) |
| 4-ml-architecture-patterns.md | 60 phút | 90 phút (thử Shadow Testing) |
| **Tổng** | **~4 giờ** | **~4 giờ** |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành (5 files)
