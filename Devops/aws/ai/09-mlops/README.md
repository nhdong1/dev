# MLOps trên AWS — Vận Hành Mô Hình Học Máy Toàn Diện

> **MLOps** (Machine Learning Operations — Vận Hành Học Máy) là tập hợp các thực hành, quy trình và công cụ nhằm đưa mô hình ML từ giai đoạn thực nghiệm vào môi trường production (Sản Xuất) một cách đáng tin cậy, có thể tái lặp và có thể quan sát được. AWS cung cấp bộ công cụ MLOps đầy đủ thông qua Amazon SageMaker.

---

## 📚 Mục Lục

1. [MLOps Là Gì và Tại Sao Quan Trọng](#mlops-là-gì)
2. [Thách Thức Của ML Trong Production](#thách-thức)
3. [Bộ Công Cụ MLOps Của AWS](#bộ-công-cụ-aws)
4. [Kiến Trúc MLOps End-to-End](#kiến-trúc)
5. [Vòng Đời Mô Hình ML](#vòng-đời)
6. [So Sánh Công Cụ MLOps](#so-sánh)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#phỏng-vấn)

---

## MLOps Là Gì và Tại Sao Quan Trọng {#mlops-là-gì}

### Định Nghĩa

**MLOps** kết hợp Machine Learning (Học Máy) + DevOps (Development Operations — Kết Hợp Phát Triển và Vận Hành) + Data Engineering (Kỹ Thuật Dữ Liệu):

```
DevOps:  Code → Build → Test → Deploy → Monitor
DataOps: Data → Validate → Process → Store → Govern
MLOps:   Data + Code + Model → Train → Evaluate → Deploy → Monitor → Retrain
```

### Lý Do Cần MLOps

| Vấn Đề Không Có MLOps                          | Giải Pháp Với MLOps                              |
| ----------------------------------------------- | ------------------------------------------------- |
| Model train xong không biết đã deploy chưa      | Pipeline tự động hóa toàn bộ quy trình           |
| Không biết model đang dùng version nào          | Model Registry (Kho Mô Hình) theo dõi version    |
| Không phát hiện khi model giảm chất lượng       | Model Monitor (Giám Sát Mô Hình) cảnh báo drift  |
| Không thể tái lặp kết quả thực nghiệm           | Experiments Tracking (Theo Dõi Thực Nghiệm)      |
| Không ai biết tại sao model đưa ra dự đoán đó   | Clarify (Giải Thích) với SHAP explainability      |
| Khó biết model có thiên kiến không              | Bias Detection (Phát Hiện Thiên Kiến) tự động    |

### Ba Cột Trụ Của MLOps

```
┌─────────────────────────────────────────────────────┐
│                   MLOps Platform                    │
├──────────────┬──────────────────┬───────────────────┤
│ Reproducibility│   Automation   │   Observability   │
│ (Tái Lặp Được)│  (Tự Động Hóa) │  (Khả Năng QS)    │
│               │                │                   │
│ - Version cả  │ - Pipeline CI/ │ - Model Monitor   │
│   data, code  │   CD cho ML    │ - Logging, Tracing│
│   và model    │ - Auto retrain │ - Alerting drift  │
│ - Experiment  │ - Auto deploy  │ - Lineage Graph   │
│   tracking    │   sau approval │   (Đồ Thị Nguồn   │
│               │                │   Gốc)            │
└──────────────┴──────────────────┴───────────────────┘
```

---

## Thách Thức Của ML Trong Production {#thách-thức}

### 1. Model Drift (Trôi Dạt Mô Hình)

Có hai loại drift chính:

**Data Drift** (Trôi Dạt Dữ Liệu — còn gọi là Covariate Shift):
- Phân phối dữ liệu đầu vào thay đổi theo thời gian
- Ví dụ: model nhận diện gian lận train trên dữ liệu 2022, nhưng hành vi gian lận 2024 đã thay đổi

**Concept Drift** (Trôi Dạt Khái Niệm):
- Mối quan hệ giữa đầu vào và đầu ra thay đổi
- Ví dụ: model dự báo giá nhà không còn chính xác sau biến động thị trường bất thường

### 2. Training-Serving Skew (Lệch Huấn Luyện-Phục Vụ)

Sự khác biệt giữa dữ liệu dùng khi training và dữ liệu thực tế khi serving:
- Feature engineering khác nhau giữa training pipeline và inference pipeline
- Data preprocessing (Tiền Xử Lý Dữ Liệu) không nhất quán

### 3. Model Lineage (Nguồn Gốc Mô Hình)

Không thể truy vết:
- Model được train từ data nào, code version nào, hyperparameter nào?
- Ai đã approve và deploy model đó?

### 4. Reproducibility (Khả Năng Tái Lặp)

Không thể tái tạo lại kết quả từ 6 tháng trước vì:
- Data đã thay đổi (không có versioning)
- Code đã thay đổi (không có experiment tracking)
- Môi trường thay đổi (library versions)

---

## Bộ Công Cụ MLOps Của AWS {#bộ-công-cụ-aws}

### Amazon SageMaker MLOps Suite

```
┌──────────────────────────────────────────────────────────────────┐
│                    SageMaker MLOps Ecosystem                     │
├──────────────┬──────────────┬──────────────┬─────────────────────┤
│   Pipelines  │   Registry   │   Monitor    │    Clarify          │
│  (Đường Ống) │  (Kho Mô    │  (Giám Sát)  │   (Giải Thích)      │
│              │   Hình)      │              │                     │
│ - DAG-based  │ - Versioning │ - Data qual. │ - Bias detection    │
│   workflow   │ - Approval   │ - Model qual.│ - SHAP values       │
│ - CI/CD cho  │   workflow   │ - Bias drift │ - Pre/Post train    │
│   ML         │ - Metadata   │ - Feature    │   bias report       │
│ - Trigger    │   lineage    │   attrib.    │                     │
│   tự động    │              │   drift      │                     │
├──────────────┴──────────────┴──────────────┴─────────────────────┤
│                     Experiments Tracking                         │
│              (Theo Dõi Thực Nghiệm)                              │
│     - Run tracking   - Metrics   - Artifacts   - Lineage Graph   │
└──────────────────────────────────────────────────────────────────┘
```

### Các File Trong Module Này

| File                           | Nội Dung                                                 | Độ Khó |
| ------------------------------ | -------------------------------------------------------- | ------ |
| `1-sagemaker-pipelines.md`     | Pipeline DAG, bước CI/CD ML, Triggers, Caching          | ⭐⭐⭐ |
| `2-model-registry.md`          | Model versions, Approval workflow, Metadata, Lineage     | ⭐⭐   |
| `3-model-monitor.md`           | Data quality, Model quality, Bias drift, Alerts          | ⭐⭐⭐ |
| `4-sagemaker-clarify.md`       | Bias detection, SHAP explainability, Pre/Post-training   | ⭐⭐⭐ |
| `5-experiments-tracking.md`    | Experiment runs, Metrics, Artifacts, ML Lineage          | ⭐⭐   |

---

## Kiến Trúc MLOps End-to-End {#kiến-trúc}

### Luồng CI/CD Cho ML (Machine Learning CI/CD Flow)

```
                    ┌──────────────────────────────────────────────┐
                    │         Source Control (Git / CodeCommit)    │
                    │         (Kiểm Soát Phiên Bản Nguồn)          │
                    └────────────────────┬─────────────────────────┘
                                         │ Push/PR trigger
                                         ▼
                    ┌──────────────────────────────────────────────┐
                    │         SageMaker Pipeline (Trigger)         │
                    │         (Đường Ống SageMaker - Kích Hoạt)    │
                    └────────────────────┬─────────────────────────┘
                                         │
          ┌──────────────────────────────┼──────────────────────────────┐
          ▼                              ▼                              ▼
┌─────────────────┐          ┌─────────────────┐            ┌──────────────────┐
│   Data          │          │   Model         │            │   Model          │
│   Processing    │          │   Training &    │            │   Evaluation     │
│   (Xử Lý       │  ──────►  │   HPO           │  ────────► │   (Đánh Giá     │
│   Dữ Liệu)     │          │   (Tinh Chỉnh    │            │   Mô Hình)      │
└─────────────────┘          │   Tham Số)      │            └──────────────────┘
                             └─────────────────┘                      │
                                                                       │ Đạt ngưỡng?
                                                                       ▼
                                                          ┌─────────────────────┐
                                                          │   Model Registry    │
                                                          │   Registration      │
                                                          │   (Đăng Ký Kho      │
                                                          │   Mô Hình)          │
                                                          └──────────┬──────────┘
                                                                     │
                                                                     │ Manual Approval
                                                                     ▼
                                                          ┌─────────────────────┐
                                                          │   Deploy to         │
                                                          │   Staging → Prod    │
                                                          │   (Triển Khai)      │
                                                          └──────────┬──────────┘
                                                                     │
                                                                     ▼
                                                          ┌─────────────────────┐
                                                          │  Model Monitor +    │
                                                          │  Clarify            │
                                                          │  (Giám Sát &        │
                                                          │   Giải Thích)       │
                                                          └──────────┬──────────┘
                                                                     │ Drift detected
                                                                     ▼
                                                          ┌─────────────────────┐
                                                          │   Auto Retrain      │
                                                          │   Trigger           │
                                                          │   (Kích Hoạt Lại    │
                                                          │   Huấn Luyện)       │
                                                          └─────────────────────┘
```

### Môi Trường Triển Khai (Deployment Environments)

```
Development  →  Staging (Dàn Dựng)  →  Production (Sản Xuất)
    │                  │                       │
Experiment         Shadow Deploy          Blue/Green Deploy
tracking           (Triển Khai Bóng)      (Triển Khai Xanh/Xanh)
(Theo dõi thực     traffic split          A/B testing, canary
nghiệm)            10% → prod             rollout
```

---

## Vòng Đời Mô Hình ML {#vòng-đời}

### ML Model Lifecycle (Vòng Đời Mô Hình ML)

```
1. DATA PREPARATION (Chuẩn Bị Dữ Liệu)
   ├── Feature Store: Tạo và quản lý features
   ├── Data Wrangler: Làm sạch và biến đổi dữ liệu
   └── Ground Truth: Gán nhãn dữ liệu

2. EXPERIMENTATION (Thực Nghiệm)
   ├── SageMaker Experiments: Theo dõi runs, metrics
   ├── SageMaker Studio: IDE thực nghiệm
   └── Autopilot: AutoML (tự động chọn thuật toán)

3. TRAINING & TUNING (Huấn Luyện & Tinh Chỉnh)
   ├── SageMaker Training: Managed training jobs
   ├── Automatic Model Tuning (AMT): HPO tự động
   └── Spot Training: Tiết kiệm chi phí đến 90%

4. EVALUATION (Đánh Giá)
   ├── Model quality metrics (precision, recall, F1, AUC)
   ├── SageMaker Clarify: Kiểm tra bias trước khi deploy
   └── Threshold gate: Chỉ tiến nếu đạt ngưỡng chất lượng

5. REGISTRATION (Đăng Ký)
   ├── Model Registry: Lưu version, metadata, lineage
   ├── Approval workflow: Human review trước khi deploy
   └── Model Card: Tài liệu hóa mô hình

6. DEPLOYMENT (Triển Khai)
   ├── Real-time Endpoint: Inference theo thời gian thực
   ├── Batch Transform: Inference hàng loạt
   ├── Shadow deployment: Test model mới song song
   └── Blue/Green / Canary rollout: Giảm thiểu rủi ro

7. MONITORING (Giám Sát)
   ├── Data Quality Monitor: Kiểm tra phân phối dữ liệu
   ├── Model Quality Monitor: Theo dõi accuracy, F1
   ├── Bias Drift Monitor: Phát hiện thiên kiến mới
   └── Feature Attribution Drift: SHAP value thay đổi

8. RETRAINING (Huấn Luyện Lại)
   ├── Trigger: Tự động khi phát hiện drift
   ├── New data: Cập nhật với dữ liệu mới
   └── Quay lại bước 1 — vòng lặp liên tục
```

---

## So Sánh Công Cụ MLOps {#so-sánh}

### AWS SageMaker vs Công Cụ MLOps Khác

| Tính Năng                          | SageMaker MLOps    | MLflow             | Kubeflow            |
| ---------------------------------- | ------------------ | ------------------ | ------------------- |
| **Pipeline Orchestration**         | ✅ Native          | ✅ MLflow Projects | ✅ KFP              |
| **Model Registry**                 | ✅ Managed         | ✅ MLflow Registry | ✅ KFServing        |
| **Experiment Tracking**            | ✅ SageMaker Exp.  | ✅ MLflow Tracking | ⚠️ Phần 3 tools     |
| **Model Monitoring**               | ✅ Built-in        | ❌ Cần add-on      | ⚠️ Phần 3 tools     |
| **Bias Detection**                 | ✅ Clarify         | ❌                 | ❌                  |
| **Tích Hợp AWS**                   | ✅ Native S3, IAM  | ⚠️ Cần cấu hình   | ⚠️ Cần cấu hình     |
| **Managed Infrastructure**         | ✅ Fully managed   | ❌ Tự quản lý      | ❌ Tự quản lý       |
| **Chi Phí**                        | Trả theo dùng      | Mã nguồn mở        | Mã nguồn mở         |
| **Độ Phức Tạp**                    | Trung bình         | Thấp               | Cao                 |

### Khi Nào Dùng Gì

```
SageMaker MLOps: Khi đã dùng AWS ecosystem, cần managed solution
MLflow:          Khi cần portable, multi-cloud, open-source
Kubeflow:        Khi đã có Kubernetes cluster, cần tùy chỉnh cao
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp {#phỏng-vấn}

### Câu Hỏi Cơ Bản

**H: MLOps khác DevOps như thế nào?**

> MLOps mở rộng DevOps bằng cách thêm quản lý dữ liệu, model versioning, experiment tracking và model monitoring. Khác biệt chính: trong phần mềm thông thường, chỉ code thay đổi; trong ML, cả data, code và model weights đều thay đổi và đều cần versioning và testing.

**H: Model drift là gì? Có mấy loại?**

> **Model drift** là hiện tượng hiệu suất mô hình giảm theo thời gian. Có hai loại chính:
> - **Data drift** (Covariate Shift — Dịch Chuyển Hiệp Biến): Phân phối dữ liệu đầu vào thay đổi — model vẫn đúng về lý thuyết nhưng thực tế không gặp dữ liệu kiểu này
> - **Concept drift** (Trôi Dạt Khái Niệm): Mối quan hệ giữa đầu vào và nhãn thay đổi — tức là khái niệm học máy cần học đã thay đổi

**H: Khi nào cần retrain model?**

> Retrain khi: (1) phát hiện data drift hoặc concept drift qua monitoring, (2) model quality metrics giảm xuống dưới ngưỡng, (3) có dữ liệu labeled mới đáng kể, (4) theo lịch định kỳ (ví dụ: mỗi tháng với dữ liệu freshness cao). Không nên retrain mù quáng — cần đánh giá xem retrain có thực sự cải thiện không.

### Câu Hỏi Nâng Cao

**H: Thiết kế MLOps pipeline cho model phân loại gian lận giao dịch?**

> 1. **Data layer**: Feature Store lưu user/transaction features, Glue crawl data từ DWH
> 2. **Training pipeline**: SageMaker Pipeline — ProcessingStep → TrainingStep → EvaluationStep → RegisterStep
> 3. **Deployment**: Blue/Green với traffic splitting 10% → canary → 100%
> 4. **Monitoring**: Data Quality Monitor (phân phối transaction amount, merchant category), Model Quality Monitor (precision/recall so với ground truth từ fraud team), Bias Monitor (phát hiện nếu model unfair với demographic groups)
> 5. **Retraining trigger**: EventBridge khi Monitor phát hiện violation, kích hoạt Pipeline chạy lại

**H: Shadow deployment là gì? Khi nào dùng?**

> **Shadow deployment** (Triển Khai Bóng) là kỹ thuật chạy model mới song song với model đang sản xuất: traffic thực đến cả hai, nhưng chỉ model cũ phục vụ response thực tế, model mới xử lý nhưng output bị bỏ qua — dùng để so sánh hiệu suất trong điều kiện production thực tế trước khi chính thức chuyển đổi. Dùng khi muốn validate model mới với production traffic mà không có rủi ro.

---

## 📁 Điều Hướng Module

| File                           | Nội Dung                           | Đọc Khi Cần                       |
| ------------------------------ | ---------------------------------- | ---------------------------------- |
| `1-sagemaker-pipelines.md`     | Pipeline DAG, steps, CI/CD ML      | Tự động hóa quy trình ML          |
| `2-model-registry.md`          | Versioning, approval, metadata     | Quản lý version và govern mô hình |
| `3-model-monitor.md`           | Drift detection, data quality      | Theo dõi mô hình sau khi deploy   |
| `4-sagemaker-clarify.md`       | Bias, SHAP explainability          | Fairness và explainability         |
| `5-experiments-tracking.md`    | Experiment runs, lineage           | Theo dõi thực nghiệm và tái lặp   |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
