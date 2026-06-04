# SageMaker Autopilot — AutoML (Học Máy Tự Động) End-to-End

> SageMaker Autopilot là tính năng AutoML (Automated Machine Learning — Học Máy Tự Động) của AWS: bạn chỉ cần cung cấp dữ liệu và chỉ ra cột mục tiêu (target column), Autopilot tự động phân tích data, chọn thuật toán, tinh chỉnh hyperparameters và tạo model tốt nhất — hoàn toàn transparent và có thể giải thích.

---

## 📚 Mục Lục

1. [Autopilot Là Gì?](#autopilot-là-gì)
2. [Các Bước Autopilot Thực Hiện](#các-bước-autopilot-thực-hiện)
3. [Problem Types — Loại Bài Toán](#problem-types)
4. [Autopilot Modes — Chế Độ Chạy](#autopilot-modes)
5. [Explainability — Khả Năng Giải Thích](#explainability)
6. [Autopilot vs Manual Training](#autopilot-vs-manual-training)
7. [Hands-on: Tạo Autopilot Job](#hands-on)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Autopilot Là Gì?

SageMaker Autopilot là **AutoML** solution — tự động hóa toàn bộ pipeline ML từ raw data đến deployed model, đồng thời vẫn **transparent** (minh bạch): bạn có thể xem toàn bộ code Autopilot tạo ra, chỉnh sửa và tái sử dụng.

### Điểm Khác Biệt Quan Trọng So Với Các AutoML Khác

| Tính Năng | SageMaker Autopilot | Các AutoML Khác |
|---|---|---|
| **Transparency** (Minh Bạch) | ✅ Sinh ra notebooks giải thích từng bước | ❌ Black box |
| **Editable Notebooks** | ✅ Có thể chỉnh sửa pipeline | ❌ Không |
| **Explainability** (Giải Thích AI) | ✅ SHAP values tích hợp | Tùy |
| **Custom Objectives** | ✅ Tự chọn metric tối ưu | Hạn chế |
| **Enterprise Integration** | ✅ Tích hợp MLflow, SageMaker Pipelines | ❌ |

---

## Các Bước Autopilot Thực Hiện

```
Bước 1: DATA ANALYSIS (Phân Tích Dữ Liệu)
   └── Tự động phân tích schema, statistics, missing values,
       class distribution, correlations

Bước 2: FEATURE ENGINEERING (Kỹ Thuật Đặc Trưng)
   └── Tạo data transformations:
       - Missing value imputation (Điền Giá Trị Bị Thiếu)
       - Categorical encoding (Mã Hóa Biến Phân Loại)
       - Numerical normalization (Chuẩn Hóa Số)
       - Feature combination

Bước 3: MODEL SELECTION + HPO (Chọn Mô Hình + Tối Ưu Siêu Tham Số)
   └── Thử nhiều thuật toán song song:
       - Linear models (Mô Hình Tuyến Tính)
       - Tree-based: XGBoost, Random Forest
       - Deep Learning: MLP (Multilayer Perceptron)
       Hyperparameter tuning tự động với Bayesian Optimization

Bước 4: LEADERBOARD (Bảng Xếp Hạng)
   └── Xếp hạng tất cả models theo metric tối ưu
       (VD: AUC, F1, RMSE, MSE)

Bước 5: BEST MODEL DEPLOYMENT
   └── Deploy model tốt nhất lên SageMaker Endpoint
       (hoặc để user tự deploy)
```

### Artifacts Được Sinh Ra

```
Output của Autopilot Job:
├── SageMaker-AutoML-Candidate-SageMaker-Notebooks/
│   ├── SageMaker AutoML Data Exploration Notebook.ipynb
│   │     (Khám phá và phân tích dữ liệu)
│   └── SageMaker AutoML Candidate Definition Notebook.ipynb
│         (Mỗi candidate pipeline: preprocessing + model)
│
└── output/
    ├── best-model/model.tar.gz
    └── leaderboard.csv (Bảng xếp hạng tất cả candidates)
```

---

## Problem Types — Loại Bài Toán {#problem-types}

Autopilot hỗ trợ 5 loại bài toán:

### 1. Binary Classification (Phân Loại Nhị Phân)

Phân loại 2 lớp — Yes/No, Churn/Retain, Fraud/Normal:

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='churn',   # Cột cần dự đoán
    problem_type='BinaryClassification',
    job_objective={'MetricName': 'F1'},  # Tối ưu F1 score
)
```

**Metrics phù hợp:** AUC, F1, Accuracy, Balanced Accuracy (cho imbalanced data — dữ liệu mất cân bằng)

### 2. Multiclass Classification (Phân Loại Đa Lớp)

Phân loại nhiều lớp — phân loại cảm xúc (tích cực/tiêu cực/trung tính), phân loại sản phẩm:

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='product_category',
    problem_type='MulticlassClassification',
    job_objective={'MetricName': 'Accuracy'}
)
```

### 3. Regression (Hồi Quy)

Dự đoán giá trị số liên tục — giá nhà, doanh thu, nhiệt độ:

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='house_price',
    problem_type='Regression',
    job_objective={'MetricName': 'MSE'}  # Mean Squared Error
)
```

**Metrics phù hợp:** MSE (Mean Squared Error — Sai Số Bình Phương Trung Bình), RMSE, MAE, R²

### 4. Time Series Forecasting (Dự Báo Chuỗi Thời Gian)

Dự báo giá trị tương lai dựa trên lịch sử:

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='sales',
    problem_type='Forecasting',
    forecasting_config={
        'ForecastFrequency': 'D',      # Tần suất: D=ngày, W=tuần, M=tháng
        'ForecastHorizon': 30,          # Dự báo 30 kỳ tới
        'ForecastQuantiles': ['0.1', '0.5', '0.9'],  # Phân vị dự báo
        'TimeSeriesConfig': {
            'TimestampAttributeName': 'date',
            'ItemIdentifierAttributeName': 'product_id'
        }
    }
)
```

### 5. Text Classification (Phân Loại Văn Bản) — NLP

Phân loại email spam, sentiment analysis:

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='label',
    problem_type='TextClassification',
    text_config={
        'TextColumns': ['review_text', 'title'],  # Cột văn bản
        'Language': 'en'
    }
)
```

---

## Autopilot Modes — Chế Độ Chạy {#autopilot-modes}

### Auto Mode (Chế Độ Tự Động) — Mặc Định

Autopilot tự chọn strategy dựa trên kích thước dataset và time budget:

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='label',
    mode='AUTO'  # Mặc định
)
```

### HPO Mode (Hyperparameter Optimization — Tối Ưu Siêu Tham Số)

Tập trung thử nhiều hyperparameter combinations, chất lượng cao hơn nhưng lâu hơn:

- Chạy nhiều training jobs với Bayesian Optimization
- Phù hợp khi có thời gian và budget
- Tạo candidate notebooks đầy đủ

### Ensembling Mode (Chế Độ Kết Hợp Mô Hình)

Kết hợp nhiều base models để tạo model mạnh hơn:

- Dùng AutoGluon framework
- Ensemble các models: bagging, stacking, boosting
- Thường cho kết quả tốt nhất nhưng chậm hơn HPO

```python
autopilot = AutoML(
    role=role,
    target_attribute_name='label',
    mode='ENSEMBLING'
)
```

---

## Explainability — Khả Năng Giải Thích {#explainability}

Autopilot tích hợp **SageMaker Clarify** để tạo explainability report (báo cáo giải thích).

### SHAP Values (SHapley Additive exPlanations — Giải Thích Đóng Góp Đặc Trưng)

SHAP là phương pháp giải thích model: đo lường mức độ đóng góp của từng feature (đặc trưng) vào từng dự đoán cụ thể.

```
Ví dụ: Dự đoán churn cho khách hàng A

Prediction: CHURN (90% probability)

SHAP Feature Contributions (Đóng Góp Của Từng Đặc Trưng):
├── last_purchase_days_ago = 120 days  → +0.35 (tăng xác suất churn mạnh)
├── total_purchases = 2               → +0.22 (tăng xác suất churn)
├── support_tickets = 5               → +0.18 (tăng xác suất churn)
├── avg_order_value = $150            → -0.08 (giảm xác suất churn một chút)
└── region = "North"                  → -0.02 (ảnh hưởng nhỏ)
```

### Bật Explainability Trong Autopilot

```python
from sagemaker.automl.automl import AutoML, AutoMLInput
from sagemaker.automl.candidate_estimator import CandidateEstimator

autopilot = AutoML(
    role=role,
    target_attribute_name='churn',
    generate_candidate_definitions_only=False
)

# Sau khi job hoàn thành, tạo explainability report
best_candidate = autopilot.best_candidate()
autopilot.create_model_insights_with_shap(
    target_attribute_name='churn',
    s3_input_data='s3://bucket/test-data.csv',
    wait=True
)
```

### Global vs Local Explainability

| Loại | Mô Tả | Use Case |
|---|---|---|
| **Global** (Toàn Cục) | Feature importance trung bình trên toàn dataset | Hiểu model nói chung |
| **Local** (Cục Bộ) | SHAP values cho một prediction cụ thể | Giải thích cho từng customer, từng quyết định |

---

## Autopilot vs Manual Training

### Khi Nào Dùng Autopilot

✅ **Phù Hợp:**
- Cần baseline model nhanh (trong 2-4 giờ)
- Data Scientist không chuyên sâu về thuật toán
- Bài toán tabular data chuẩn (classification, regression)
- Muốn explore nhiều algorithms trước khi commit
- Business team muốn tự chạy ML không cần ML Engineer
- Cần explainability report cho stakeholders (người liên quan)

❌ **Không Phù Hợp:**
- Custom deep learning architectures (kiến trúc học sâu tùy chỉnh)
- Computer vision, NLP complex tasks
- Cần kiểm soát chi tiết preprocessing pipeline
- Data quá đặc thù (time series phức tạp, text không chuẩn)
- Cần production model với SLA nghiêm ngặt về latency

### So Sánh Thời Gian và Effort

```
Bài Toán: Churn Prediction với 500k records

Manual Training:
  Data Analysis & Preprocessing:  1-2 ngày
  Model Experimentation:          2-3 ngày
  Hyperparameter Tuning:          1-2 ngày
  Model Evaluation & Selection:   1 ngày
  ─────────────────────────────────────
  Tổng:                           5-8 ngày

SageMaker Autopilot:
  Cấu hình job:                   30 phút
  Autopilot running time:         2-4 giờ (tự động)
  Review results & deploy:        1-2 giờ
  ─────────────────────────────────────
  Tổng:                           4-6 giờ
  
Chất lượng model: Autopilot thường đạt 85-95% chất lượng
của manual tuning. Manual vẫn tốt hơn cho complex cases.
```

---

## Hands-on: Tạo Autopilot Job {#hands-on}

### Bước 1: Chuẩn Bị Data

```python
import pandas as pd
import boto3
import sagemaker

# Load và upload data lên S3
df = pd.read_csv('customer_churn.csv')
print(df.head())
print(f"Shape: {df.shape}")
print(f"Target distribution:\n{df['churn'].value_counts(normalize=True)}")

# Upload lên S3
session = sagemaker.Session()
bucket = session.default_bucket()

train_data_s3 = session.upload_data(
    'customer_churn.csv',
    bucket=bucket,
    key_prefix='autopilot/input'
)
print(f"Data uploaded to: {train_data_s3}")
```

### Bước 2: Cấu Hình và Chạy Autopilot Job

```python
from sagemaker.automl.automl import AutoML

role = sagemaker.get_execution_role()

autopilot = AutoML(
    role=role,
    target_attribute_name='churn',          # Cột target
    problem_type='BinaryClassification',
    job_objective={'MetricName': 'F1'},     # Metric tối ưu
    max_candidates=20,                       # Thử tối đa 20 candidates
    max_runtime_per_training_job_in_seconds=600,  # 10 phút/job
    total_job_runtime_in_seconds=7200,       # Tổng job ≤ 2 giờ
    mode='ENSEMBLING',                       # Chế độ ensemble
    generate_candidate_definitions_only=False,
    output_path=f's3://{bucket}/autopilot/output/'
)

# Bắt đầu job
autopilot.fit(
    inputs=train_data_s3,
    wait=False,           # Không chờ — job chạy background
    logs=False
)

print(f"Job name: {autopilot.latest_auto_ml_job.name}")
```

### Bước 3: Theo Dõi Tiến Độ

```python
import time

while True:
    job_status = autopilot.describe_auto_ml_job()
    status = job_status['AutoMLJobStatus']
    secondary = job_status.get('AutoMLJobSecondaryStatus', '')
    
    print(f"Status: {status} | Secondary: {secondary}")
    
    if status in ['Completed', 'Failed', 'Stopped']:
        break
    
    time.sleep(60)  # Kiểm tra mỗi 1 phút

print(f"Final status: {status}")
```

### Bước 4: Xem Leaderboard (Bảng Xếp Hạng)

```python
# Lấy tất cả candidates được thử
candidates = autopilot.list_candidates(
    sort_by='FinalObjectiveMetricValue',
    sort_order='Descending',
    max_results=10
)

print("Top 10 Candidates (Ứng Viên):")
for i, candidate in enumerate(candidates):
    name = candidate['CandidateName']
    metric = candidate['FinalAutoMLJobObjectiveMetric']
    print(f"{i+1}. {name}: {metric['MetricName']} = {metric['Value']:.4f}")

# Lấy model tốt nhất
best = autopilot.best_candidate()
print(f"\nBest Model: {best['CandidateName']}")
print(f"Best F1: {best['FinalAutoMLJobObjectiveMetric']['Value']:.4f}")
```

### Bước 5: Deploy Model Tốt Nhất

```python
# Deploy best model lên endpoint
predictor = autopilot.deploy(
    initial_instance_count=1,
    instance_type='ml.m5.xlarge',
    endpoint_name='churn-autopilot-endpoint'
)

# Test prediction
import json
test_data = {
    'features': [
        [120, 2, 5, 150, 'North'],  # Sample 1
        [7, 15, 0, 500, 'South']    # Sample 2
    ]
}

response = predictor.predict(test_data)
print(f"Predictions: {response}")

# Xóa endpoint sau khi xong để tránh tốn phí!
predictor.delete_endpoint()
```

---

## Câu Hỏi Phỏng Vấn

### Q1: SageMaker Autopilot khác gì với các AutoML solutions khác như H2O hay AutoGluon?

**Trả lời tốt:**
> Điểm khác biệt lớn nhất của Autopilot là tính **transparent** (minh bạch). Khi Autopilot hoàn thành, nó sinh ra Jupyter notebooks giải thích từng bước: data analysis notebook và candidate definition notebook — tôi có thể đọc, chỉnh sửa và tái sử dụng code đó. Đây là AutoML có thể "inspect" được, không phải black box. Ngoài ra, Autopilot tích hợp sẵn với toàn bộ SageMaker ecosystem: tự động track experiments, đưa best model vào Model Registry, tích hợp SageMaker Clarify để tạo explainability report với SHAP values. Về thuật toán, Autopilot dùng ensemble mode với AutoGluon và HPO mode với Bayesian Optimization — chất lượng tốt cho tabular data. Hạn chế là chỉ hỗ trợ tabular, time series và text classification đơn giản; không hỗ trợ custom neural architectures.

### Q2: SHAP values là gì và tại sao quan trọng trong enterprise ML?

**Trả lời tốt:**
> SHAP (SHapley Additive exPlanations) là phương pháp giải thích dựa trên game theory (lý thuyết trò chơi): nó tính toán mức đóng góp của từng feature vào mỗi dự đoán cụ thể. Ví dụ, với model churn prediction: SHAP có thể nói "khách hàng này có 90% xác suất rời bỏ vì không mua hàng 120 ngày (+0.35 SHAP), 5 ticket hỗ trợ (+0.18 SHAP), chỉ mua 2 lần (+0.22 SHAP)."
> Trong enterprise quan trọng vì nhiều lý do: (1) **Regulatory compliance** — GDPR và một số ngành tài chính yêu cầu AI phải có thể giải thích được; (2) **Trust building** — stakeholders và customers muốn hiểu tại sao AI đưa ra quyết định; (3) **Model debugging** — nếu model sai, SHAP giúp tìm feature nào gây sai; (4) **Feature selection** — SHAP global importance giúp loại bỏ features ít ý nghĩa, đơn giản hóa model.

### Q3: Khi nào AutoML như Autopilot KHÔNG đủ và cần custom model?

**Trả lời tốt:**
> Autopilot hoạt động tốt cho tabular data chuẩn nhưng không đủ trong các trường hợp: (1) **Complex architectures** — custom transformer models, multi-modal inputs (văn bản + ảnh + số); (2) **Domain-specific preprocessing** — NLP với tiếng Việt cần custom tokenizer, y tế cần xử lý DICOM images; (3) **Strict latency SLA** — Autopilot optimize cho accuracy, không phải inference latency; đôi khi cần distill hoặc prune model thủ công; (4) **Custom loss functions** — business-specific objectives như asymmetric loss (lỗi false negative đắt hơn false positive); (5) **Continual learning** — model cần cập nhật liên tục với data stream. Trong thực tế, tôi thường dùng Autopilot để tạo **baseline nhanh** — nếu kết quả đã đủ tốt cho business yêu cầu thì dùng luôn, nếu chưa thì lấy best candidate notebook của Autopilot làm điểm khởi đầu để cải thiện thêm.

---

## 📊 Tóm Tắt

```
SageMaker Autopilot
├── Flow: Raw Data → Analysis → Feature Engineering → HPO → Leaderboard → Deploy
│
├── Problem Types (Loại Bài Toán)
│   ├── BinaryClassification (Phân Loại Nhị Phân)
│   ├── MulticlassClassification (Phân Loại Đa Lớp)
│   ├── Regression (Hồi Quy)
│   ├── Forecasting (Dự Báo Chuỗi Thời Gian)
│   └── TextClassification (Phân Loại Văn Bản)
│
├── Modes (Chế Độ)
│   ├── AUTO — tự chọn strategy
│   ├── HPO — Bayesian Optimization nhiều candidates
│   └── ENSEMBLING — AutoGluon ensemble
│
├── Outputs (Đầu Ra)
│   ├── Data Exploration Notebook
│   ├── Candidate Definition Notebooks (code pipeline)
│   ├── Leaderboard (bảng xếp hạng)
│   └── Best Model + SHAP Explainability Report
│
└── Best For (Phù Hợp Nhất)
    ├── Baseline model nhanh
    ├── Tabular data classification/regression
    └── Khi cần explainability cho stakeholders
```

---

**File tiếp theo:** [5-sagemaker-feature-store.md](./5-sagemaker-feature-store.md) — Feature Store cho ML Production

**Cập Nhật Lần Cuối:** 2026-06-03
