# SageMaker Model Monitor — Giám Sát Mô Hình ML Trong Production

> **SageMaker Model Monitor** (Giám Sát Mô Hình SageMaker) là dịch vụ tự động giám sát chất lượng dữ liệu, chất lượng mô hình, thiên kiến và feature attribution (Đóng Góp Đặc Trưng) của các SageMaker endpoints trong production — phát hiện sự trôi dạt (drift) và cảnh báo khi model không còn hoạt động đúng như kỳ vọng.

---

## 📚 Mục Lục

1. [Tổng Quan và Bốn Loại Monitor](#tổng-quan)
2. [Data Quality Monitor](#data-quality-monitor)
3. [Model Quality Monitor](#model-quality-monitor)
4. [Bias Drift Monitor](#bias-drift-monitor)
5. [Feature Attribution Drift Monitor](#feature-attribution-drift)
6. [Cấu Hình Alerting](#alerting)
7. [Khi Nào Cần Retrain](#khi-nào-retrain)
8. [Câu Hỏi Phỏng Vấn](#phỏng-vấn)

---

## Tổng Quan và Bốn Loại Monitor {#tổng-quan}

### Vì Sao Cần Model Monitor?

```
Training Time (Lúc Huấn Luyện)    Serving Time (Lúc Phục Vụ)
─────────────────────────────      ─────────────────────────
Data: 2023 Q1-Q3 transactions  →   Data: 2024 Q2 transactions
Economy: Pre-recession             Economy: Recession period
Fraud patterns: Card-not-present   Fraud patterns: Account takeover
Model AUC: 0.923                   Model AUC: 0.71 (degraded!)
                                               ↑
                          Model Monitor phát hiện điều này
```

Model không "biết" khi nào chúng bắt đầu hoạt động kém — chỉ có monitoring mới phát hiện được.

### Bốn Loại Monitor

| Loại Monitor                                           | Phát Hiện                                                  | Yêu Cầu Ground Truth |
| ------------------------------------------------------ | ---------------------------------------------------------- | --------------------- |
| **Data Quality Monitor** (Giám Sát Chất Lượng DL)     | Data drift, missing values, schema violations              | Không                 |
| **Model Quality Monitor** (Giám Sát Chất Lượng MH)    | Accuracy, Precision, Recall, F1 giảm theo thời gian       | **Có** (labels thực) |
| **Bias Drift Monitor** (Giám Sát Trôi Dạt Thiên Kiến) | Model thiên kiến với demographic groups theo thời gian     | Có (nếu có thể)       |
| **Feature Attribution Drift** (Trôi Dạt Đóng Góp ĐT)  | SHAP values thay đổi — các features quan trọng đã khác đi | Không                 |

### Kiến Trúc Model Monitor

```
┌──────────────────────────────────────────────────────────────────┐
│                    SageMaker Endpoint                            │
│                                                                  │
│  Request ──►│                    │──► Response                   │
│             │   Model            │                               │
│             │                   │                               │
└─────────────┼───────────────────┼───────────────────────────────┘
              │    Data Capture    │
              │  (Thu Thập DL)     │
              ▼                   ▼
        S3 Capture Store (Requests & Responses)
              │
              ▼
     ┌─────────────────────┐
     │  Monitor Schedule   │  ← Chạy theo lịch (mỗi giờ/ngày)
     │  (Lịch Giám Sát)    │
     └──────────┬──────────┘
                │
                ▼
     ┌─────────────────────┐      ┌─────────────────────┐
     │  Baseline Constraints│      │  Monitoring Reports │
     │  (Ràng Buộc Cơ Sở)  │ ◄──► │  (Báo Cáo Giám Sát) │
     └─────────────────────┘      └──────────┬──────────┘
                                             │ Violation?
                                             ▼
                                  ┌─────────────────────┐
                                  │  CloudWatch Alarm   │
                                  │  SNS Notification   │
                                  │  (Cảnh Báo)         │
                                  └─────────────────────┘
```

---

## Data Quality Monitor {#data-quality-monitor}

### Khái Niệm

**Data Quality Monitor** so sánh phân phối dữ liệu đầu vào hiện tại với baseline (Đường Cơ Sở) từ training data. Không cần ground truth labels — hoạt động ngay sau khi deploy.

**Phát Hiện:**
- **Data drift** (Trôi Dạt Dữ Liệu): Phân phối feature thay đổi (mean, std, percentiles)
- **Schema violations** (Vi Phạm Lược Đồ): Feature bị missing, kiểu dữ liệu sai
- **Missing values** (Giá Trị Thiếu): Tỷ lệ null tăng bất thường
- **Out-of-range values** (Giá Trị Ngoài Phạm Vi): Giá trị vượt min/max trong training data

### Bước 1: Bật Data Capture Trên Endpoint

```python
from sagemaker.model_monitor import DataCaptureConfig

# Cấu hình thu thập dữ liệu request/response
data_capture_config = DataCaptureConfig(
    enable_capture=True,
    sampling_percentage=100,        # Thu thập 100% requests (giảm xuống 10-20% cho high-traffic)
    destination_s3_uri="s3://my-bucket/data-capture/fraud-detection/",
    capture_options=[
        {"CaptureMode": "Input"},   # Thu thập request data
        {"CaptureMode": "Output"},  # Thu thập response data
    ],
    csv_content_types=["text/csv"],
    json_content_types=["application/json"],
)

# Deploy endpoint với data capture
predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.m5.large",
    endpoint_name="fraud-detection-prod",
    data_capture_config=data_capture_config,
)
```

### Bước 2: Tạo Baseline Từ Training Data

```python
from sagemaker.model_monitor import DefaultModelMonitor
from sagemaker.model_monitor.dataset_format import DatasetFormat

# Tạo monitor object
data_quality_monitor = DefaultModelMonitor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
    volume_size_in_gb=20,
    max_runtime_in_seconds=3600,
)

# Tạo baseline từ training data — sinh ra statistics.json và constraints.json
baseline_result = data_quality_monitor.suggest_baseline(
    baseline_dataset="s3://my-bucket/training-data/train.csv",
    dataset_format=DatasetFormat.csv(header=True),
    output_s3_uri="s3://my-bucket/baseline/data-quality/",
    wait=True,
    logs=True,
)

# Baseline tạo ra:
# - statistics.json: mean, std, min, max, percentiles cho mỗi feature
# - constraints.json: ràng buộc mà data phải thỏa mãn
print(f"Baseline statistics: {baseline_result.baseline_statistics().body_dict}")
print(f"Baseline constraints: {baseline_result.suggested_constraints().body_dict}")
```

### Bước 3: Tạo Monitoring Schedule (Lịch Giám Sát)

```python
from sagemaker.model_monitor import CronExpressionGenerator

data_quality_monitor.create_monitoring_schedule(
    monitor_schedule_name="fraud-detection-data-quality",
    endpoint_input="fraud-detection-prod",
    output_s3_uri="s3://my-bucket/monitoring-reports/data-quality/",
    statistics=baseline_result.baseline_statistics(),
    constraints=baseline_result.suggested_constraints(),
    schedule_cron_expression=CronExpressionGenerator.hourly(),   # Chạy mỗi giờ
    enable_cloudwatch_metrics=True,
)
```

### Hiểu Kết Quả

```json
// Ví dụ violations trong monitoring report
{
  "violations": [
    {
      "feature_name": "transaction_amount",
      "constraint_check_type": "baseline_drift_check",
      "description": "Inferred baseline data type violation. Detected: Integral, Expected: Fractional"
    },
    {
      "feature_name": "merchant_category",
      "constraint_check_type": "missing_column_check",
      "description": "Column merchant_category is missing"
    },
    {
      "feature_name": "hour_of_day",
      "constraint_check_type": "baseline_drift_check",
      "description": "Data distribution has drifted. KL divergence: 0.234 > threshold: 0.1"
    }
  ]
}
```

---

## Model Quality Monitor {#model-quality-monitor}

### Khái Niệm

**Model Quality Monitor** so sánh predictions của model với **ground truth labels** (Nhãn Thực Tế) thực tế — đo lường accuracy, precision, recall, F1 theo thời gian.

**Yêu Cầu:** Phải có cơ chế thu thập ground truth sau khi prediction xảy ra.

```
Time=T:    Request → Model → Prediction "NOT FRAUD" (ID: txn-12345)
                                        ↓
Time=T+7d: Customer reports fraud  →  Ground Truth "FRAUD" (ID: txn-12345)
                                        ↓
                            Model Quality Monitor so sánh → model sai
```

### Setup Model Quality Monitor

```python
from sagemaker.model_monitor import ModelQualityMonitor
from sagemaker.model_monitor.dataset_format import DatasetFormat

model_quality_monitor = ModelQualityMonitor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
    volume_size_in_gb=20,
    max_runtime_in_seconds=1800,
)

# Tạo baseline từ validation set
model_quality_monitor.suggest_baseline(
    baseline_dataset="s3://my-bucket/validation/validation_with_predictions.csv",
    dataset_format=DatasetFormat.csv(header=True),
    output_s3_uri="s3://my-bucket/baseline/model-quality/",
    problem_type="BinaryClassification",       # Hoặc MulticlassClassification, Regression
    inference_attribute="prediction",          # Cột chứa prediction của model
    probability_attribute="probability",       # Cột chứa probability score
    ground_truth_attribute="label",            # Cột chứa ground truth
)

# Tạo monitoring schedule
model_quality_monitor.create_monitoring_schedule(
    monitor_schedule_name="fraud-detection-model-quality",
    endpoint_input="fraud-detection-prod",
    ground_truth_input="s3://my-bucket/ground-truth/",   # Ground truth được upload định kỳ
    output_s3_uri="s3://my-bucket/monitoring-reports/model-quality/",
    problem_type="BinaryClassification",
    inference_attribute="0",                   # Với binary: "0" là output column
    probability_threshold_attribute=0.5,       # Ngưỡng để classify là positive
    schedule_cron_expression=CronExpressionGenerator.daily(),   # Hằng ngày
)
```

### Merging Ground Truth (Hợp Nhất Nhãn Thực Tế)

Ground truth thường đến trễ hơn prediction — cần upload định kỳ:

```python
import boto3
import json
from datetime import datetime

def upload_ground_truth(transaction_ids, actual_labels, s3_bucket, s3_prefix):
    """
    Upload ground truth labels để Model Quality Monitor so sánh với predictions.
    """
    s3_client = boto3.client("s3")
    
    ground_truth_records = []
    for txn_id, label in zip(transaction_ids, actual_labels):
        ground_truth_records.append({
            "groundTruthData": {
                "data": str(label),             # "0" = không gian lận, "1" = gian lận
                "encoding": "CSV",
            },
            "eventMetadata": {
                "eventId": txn_id,              # ID phải khớp với captured request ID
            },
            "eventVersion": "0",
        })
    
    # Format: một record JSON mỗi dòng (JSON Lines format)
    content = "\n".join([json.dumps(r) for r in ground_truth_records])
    
    timestamp = datetime.now().strftime("%Y/%m/%d/%H")
    s3_key = f"{s3_prefix}/{timestamp}/ground_truth.jsonl"
    
    s3_client.put_object(
        Bucket=s3_bucket,
        Key=s3_key,
        Body=content.encode("utf-8"),
    )
```

---

## Bias Drift Monitor {#bias-drift-monitor}

### Khái Niệm

**Bias Drift Monitor** (Giám Sát Trôi Dạt Thiên Kiến) phát hiện liệu model có bắt đầu đối xử không công bằng với một nhóm người dùng theo thời gian — ví dụ: model phê duyệt khoản vay ít hơn cho một nhóm tuổi cụ thể.

```
Training Time:  Model approve rate nam/nữ ≈ 48%/52% (công bằng)
After 6 months: Model approve rate nam/nữ = 55%/38% (bắt đầu thiên kiến!)
                                                ↑
                                 Bias Drift Monitor phát hiện
```

### Setup Bias Drift Monitor

```python
from sagemaker.model_monitor import BiasAnalysisConfig, ClarifyBiasMonitor

bias_analysis_config = BiasAnalysisConfig(
    bias_config=BiasConfig(
        label_values_or_threshold=[1],      # Nhãn positive (gian lận = 1)
        facet_name="customer_age_group",    # Attribute cần kiểm tra fairness
        facet_values_or_threshold=[0],      # Nhóm cần bảo vệ (0 = nhóm trẻ)
    ),
    headers=[
        "transaction_amount", "merchant_category", 
        "hour_of_day", "customer_age_group", "prediction"
    ],
    label="prediction",
)

clarify_bias_monitor = ClarifyBiasMonitor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
    volume_size_in_gb=20,
)

clarify_bias_monitor.create_monitoring_schedule(
    monitor_schedule_name="fraud-detection-bias-drift",
    endpoint_input="fraud-detection-prod",
    ground_truth_input="s3://my-bucket/ground-truth/",
    analysis_config=bias_analysis_config,
    output_s3_uri="s3://my-bucket/monitoring-reports/bias/",
    schedule_cron_expression=CronExpressionGenerator.daily(),
)
```

### Các Chỉ Số Bias Được Theo Dõi

| Chỉ Số                                                   | Định Nghĩa                                                              | Cờ Nguy Hiểm    |
| --------------------------------------------------------- | ----------------------------------------------------------------------- | --------------- |
| **DI** — Disparate Impact (Tác Động Không Công Bằng)     | Tỷ lệ kết quả tích cực giữa nhóm được bảo vệ / nhóm tham chiếu        | < 0.8 hoặc > 1.25 |
| **DPPL** — Diff. in Positive Proportions (Chênh Lệch TP) | Chênh lệch tỷ lệ dự đoán positive giữa hai nhóm                        | |> 0.1|          |
| **FPR** — False Positive Rate Difference (Chênh Lệch FPR)| Model false positive nhiều hơn với nhóm được bảo vệ không?             | |> 0.05|         |
| **FNR** — False Negative Rate Difference (Chênh Lệch FNR)| Model bỏ sót (false negative) nhiều hơn với nhóm không?                | |> 0.05|         |

---

## Feature Attribution Drift Monitor {#feature-attribution-drift}

### Khái Niệm

**Feature Attribution Drift** (Trôi Dạt Đóng Góp Đặc Trưng) theo dõi sự thay đổi trong SHAP values (Shapley Additive Explanations — Giải Thích Đóng Góp Theo Shapley) của từng feature theo thời gian. Nếu feature tầm quan trọng thay đổi đột ngột → dữ liệu đầu vào hoặc behavior của người dùng đã thay đổi căn bản.

```
Training Time:
  Feature Importance:
  1. transaction_amount     (SHAP = 0.45) — Quan trọng nhất
  2. hour_of_day            (SHAP = 0.23)
  3. merchant_category      (SHAP = 0.18)

After 6 months:
  Feature Importance (DRIFTED!):
  1. merchant_category      (SHAP = 0.51) — Bây giờ quan trọng nhất
  2. transaction_amount     (SHAP = 0.28)
  3. customer_location_new  (SHAP = 0.15) — Feature mới xuất hiện nhiều!
```

Drift này gợi ý: Fraud patterns đã thay đổi — cần retrain.

### Setup Feature Attribution Drift Monitor

```python
from sagemaker.model_monitor import (
    ClarifyExplainabilityMonitor,
    ExplainabilityAnalysisConfig,
    ModelConfig,
    SHAPConfig,
)

shap_config = SHAPConfig(
    baseline=[
        [
            # Baseline values cho mỗi feature (thường là mean từ training data)
            500.0,    # transaction_amount
            14,       # hour_of_day
            "retail", # merchant_category
            # ... các features khác
        ]
    ],
    num_samples=100,    # Số samples để tính SHAP (nhiều hơn = chính xác hơn nhưng chậm hơn)
    agg_method="mean_abs",   # Aggregate SHAP: mean_abs, median, mean_sq
)

explainability_analysis_config = ExplainabilityAnalysisConfig(
    explainability_config=shap_config,
    model_config=ModelConfig(
        model_name=model_name,
        instance_type="ml.m5.large",
        instance_count=1,
        content_type="text/csv",
        accept_type="text/csv",
    ),
    headers=feature_names,
)

clarify_explainability_monitor = ClarifyExplainabilityMonitor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
)

clarify_explainability_monitor.create_monitoring_schedule(
    monitor_schedule_name="fraud-detection-explainability-drift",
    endpoint_input="fraud-detection-prod",
    output_s3_uri="s3://my-bucket/monitoring-reports/explainability/",
    analysis_config=explainability_analysis_config,
    schedule_cron_expression=CronExpressionGenerator.daily(),
    enable_cloudwatch_metrics=True,
)
```

---

## Cấu Hình Alerting {#alerting}

### CloudWatch Alarms (Cảnh Báo CloudWatch)

Model Monitor tự động đẩy metrics lên CloudWatch — dễ dàng tạo alarm:

```python
import boto3

cloudwatch = boto3.client("cloudwatch")

# Alarm khi có data quality violation
cloudwatch.put_metric_alarm(
    AlarmName="FraudDetection-DataQuality-Violation",
    AlarmDescription="Phát hiện data drift trên endpoint fraud-detection-prod",
    MetricName="feature_baseline_drift_check",
    Namespace="/aws/sagemaker/Endpoints/data-metrics",
    Dimensions=[
        {"Name": "Endpoint", "Value": "fraud-detection-prod"},
        {"Name": "MonitoringSchedule", "Value": "fraud-detection-data-quality"},
    ],
    Statistic="Sum",
    Period=3600,          # 1 giờ
    EvaluationPeriods=1,
    Threshold=1,          # Alarm nếu có >= 1 violation
    ComparisonOperator="GreaterThanOrEqualToThreshold",
    AlarmActions=[
        "arn:aws:sns:us-east-1:123456:MLOps-Alerts",   # SNS topic để notify
    ],
    TreatMissingData="notBreaching",
)
```

### SNS Notification và Auto-Remediation

```
CloudWatch Alarm
      │
      ▼
SNS Topic "MLOps-Alerts"
      │
      ├──► Email/Slack notification → ML Engineer
      │
      └──► Lambda Function
                │
                ├── Nếu Data Drift: Trigger SageMaker Pipeline (retrain)
                │
                └── Nếu Model Quality Drift: Page on-call + create Jira ticket
```

```python
# Lambda function: Tự động trigger retrain khi drift phát hiện
def auto_remediate_drift(event, context):
    """Auto-trigger retraining pipeline khi model monitor phát hiện drift."""
    alarm_name = event["AlarmName"]
    
    if "DataQuality" in alarm_name or "ModelQuality" in alarm_name:
        sm_client = boto3.client("sagemaker")
        
        # Kích hoạt pipeline retrain với data mới nhất
        execution = sm_client.start_pipeline_execution(
            PipelineName="FraudDetectionPipeline",
            PipelineParameters=[
                {
                    "Name": "TrainingDataUri",
                    "Value": get_latest_training_data_uri(),  # Hàm lấy data mới nhất
                },
            ],
            ClientRequestToken=context.aws_request_id,
        )
        
        print(f"Triggered retraining pipeline: {execution['PipelineExecutionArn']}")
```

---

## Khi Nào Cần Retrain {#khi-nào-retrain}

### Quyết Định Retrain

```
┌──────────────────────────────────────────────────────────────────┐
│                    Khi Nào Retrain?                              │
├──────────────────────────────────────────────────────────────────┤
│ 1. MODEL QUALITY DROPS (Chất Lượng Giảm)                        │
│    Recall < 0.80 (ngưỡng định nghĩa sẵn) → RETRAIN NGAY         │
│                                                                  │
│ 2. DATA DRIFT DETECTED (Phát Hiện Data Drift)                    │
│    > 30% features bị drift → Điều Tra trước, có thể retrain     │
│    > 50% features bị drift → RETRAIN gần chắc chắn cần          │
│                                                                  │
│ 3. CONCEPT DRIFT (Trôi Dạt Khái Niệm)                           │
│    Ground truth labels thay đổi căn bản → RETRAIN               │
│                                                                  │
│ 4. BIAS DRIFT (Trôi Dạt Thiên Kiến)                              │
│    Disparate Impact < 0.8 → ĐIỀU TRA + Có thể retrain           │
│                                                                  │
│ 5. SCHEDULED RETRAINING (Huấn Luyện Theo Lịch)                   │
│    Mỗi tháng/quý với dữ liệu mới nhất (data freshness)          │
└──────────────────────────────────────────────────────────────────┘
```

### Chiến Lược Retrain

| Chiến Lược                              | Mô Tả                                             | Khi Nào Dùng                              |
| --------------------------------------- | -------------------------------------------------- | ----------------------------------------- |
| **Full Retrain** (Huấn Luyện Lại Đầy Đủ) | Train lại từ đầu với toàn bộ data mới             | Concept drift nghiêm trọng                |
| **Incremental Retrain** (Tăng Dần)      | Fine-tune model hiện tại với data mới             | Data drift nhẹ, muốn nhanh               |
| **Window Retrain** (Cửa Sổ Thời Gian)   | Train chỉ trên N tháng gần nhất                   | Data có tính thời vụ (seasonality)        |
| **Continual Learning** (Học Liên Tục)   | Update model online với từng batch data nhỏ       | High-volume, fast-changing data           |

---

## Câu Hỏi Phỏng Vấn {#phỏng-vấn}

**H: Sự khác biệt giữa Data Drift và Concept Drift?**

> - **Data drift** (Covariate shift — Dịch Chuyển Hiệp Biến): Phân phối dữ liệu đầu vào X thay đổi — P(X) thay đổi nhưng P(Y|X) vẫn giữ nguyên. Model về lý thuyết vẫn đúng nhưng gặp ít hoặc không gặp dữ liệu kiểu này trong training. Ví dụ: model nhận diện ảnh train trên ảnh chụp ban ngày, nhưng bây giờ nhận nhiều ảnh chụp ban đêm.
> - **Concept drift** (Trôi Dạt Khái Niệm): Mối quan hệ P(Y|X) thay đổi — cùng input nhưng output đúng bây giờ khác trước. Ví dụ: spam filter train năm 2020, nhưng cách spam email viết năm 2024 đã hoàn toàn khác.

**H: Làm sao biết model cần retrain hay chỉ cần adjust threshold?**

> - Nếu **data drift** nhẹ: Thử điều chỉnh decision threshold trước (không cần retrain)
> - Nếu **concept drift**: Bắt buộc retrain — threshold không giúp được
> - Nếu **model quality giảm** nhưng data distribution vẫn tương tự: Kiểm tra serving infrastructure (preprocessing) trước, sau đó mới retrain
> - Nếu **bias drift**: Điều tra xem bias từ data mới hay từ feature set thay đổi — có thể cần cả hai điều chỉnh

**H: SageMaker Model Monitor tốn bao nhiêu chi phí?**

> Monitor chạy SageMaker Processing Jobs theo schedule → tính phí instance giờ. Để tối ưu: (1) Giảm sampling_percentage từ 100% xuống 10-20% cho high-traffic endpoints, (2) Chạy schedule hằng ngày thay vì hằng giờ nếu không cần real-time alerting, (3) Dùng instance nhỏ hơn (ml.m5.large đủ cho hầu hết use cases), (4) Bật CloudWatch metric filter thay vì chạy full Processing Job cho simple checks.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
