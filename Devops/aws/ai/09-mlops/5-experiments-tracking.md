# SageMaker Experiments — Theo Dõi Thực Nghiệm và ML Lineage

> **SageMaker Experiments** (Theo Dõi Thực Nghiệm SageMaker) là dịch vụ quản lý và theo dõi các thực nghiệm (experiments) ML: ghi lại metrics (số liệu), parameters (tham số), artifacts (tạo phẩm) và relationships (mối quan hệ) giữa các runs. **ML Lineage** (Nguồn Gốc ML) tự động xây dựng đồ thị truy vết toàn bộ nguồn gốc của mô hình từ data đến deployment.

---

## 📚 Mục Lục

1. [Tại Sao Cần Experiment Tracking](#tại-sao-cần)
2. [Khái Niệm Cơ Bản](#khái-niệm)
3. [Tạo và Quản Lý Experiments](#tạo-experiments)
4. [Logging Metrics và Parameters](#logging)
5. [So Sánh Runs](#so-sánh-runs)
6. [Artifacts và Datasets](#artifacts)
7. [ML Lineage Tracking](#lineage)
8. [Tích Hợp Với SageMaker Pipelines](#tích-hợp-pipeline)
9. [Câu Hỏi Phỏng Vấn](#phỏng-vấn)

---

## Tại Sao Cần Experiment Tracking {#tại-sao-cần}

### Vấn Đề Không Có Experiment Tracking

```
Data Scientist ngày thứ Hai:
  "Tôi train XGBoost với max_depth=6, eta=0.1 → AUC = 0.91"
  "Tôi train LightGBM với num_leaves=31 → AUC = 0.93!"

Data Scientist ngày thứ Sáu:
  "Ủa... LightGBM đó dùng data version nào nhỉ?
   Code preprocessing có thay đổi không?
   Hyperparameters chính xác là gì?
   Tại sao kết quả hôm nay khác hôm qua?" → Không ai biết!
```

### Với SageMaker Experiments

```python
# Tất cả được ghi lại tự động và có thể truy vết:
run.log_parameter("max_depth", 6)
run.log_parameter("eta", 0.1)
run.log_metric("auc", 0.91, step=50)    # Theo epoch/step
run.log_file("training-data", s3_uri)   # Data version

# 6 tháng sau: "Model tốt nhất của Q1 2024 là Run #17 trong Experiment fraud-detection
# Dùng data version v2.3, XGBoost, max_depth=6, eta=0.1, AUC=0.923"
```

---

## Khái Niệm Cơ Bản {#khái-niệm}

### Hierarchy (Phân Cấp)

```
Experiment (Thực Nghiệm)
├── Run Group (Nhóm Runs)  [tùy chọn — để nhóm related runs]
│   ├── Run 1 (Lần Chạy 1)
│   │   ├── Parameters: {max_depth: 6, eta: 0.1}
│   │   ├── Metrics: {auc: 0.91, loss: [0.8, 0.6, 0.4]}  ← theo time series
│   │   └── Artifacts: {model: s3://..., data: s3://...}
│   └── Run 2
│       ├── Parameters: {max_depth: 8, eta: 0.05}
│       ├── Metrics: {auc: 0.93, loss: [0.75, 0.55, 0.38]}
│       └── Artifacts: {model: s3://..., data: s3://...}
└── Run Group B
    └── Run 3 ...
```

| Khái Niệm                                        | Định Nghĩa                                                              |
| ------------------------------------------------ | ----------------------------------------------------------------------- |
| **Experiment** (Thực Nghiệm)                     | Container logic gom tất cả runs liên quan đến một mục tiêu             |
| **Run** (Lần Chạy)                               | Một lần train cụ thể với bộ hyperparameters cụ thể                     |
| **Run Group** (Nhóm Runs)                        | Nhóm các runs liên quan (ví dụ: cùng data, khác hyperparameters)       |
| **Parameter** (Tham Số)                          | Giá trị đầu vào cố định trong một run (hyperparameter)                  |
| **Metric** (Số Liệu)                             | Giá trị đầu ra được đo lường (AUC, loss, accuracy) — có thể time series|
| **Artifact** (Tạo Phẩm)                          | File liên quan đến run: model weights, dataset, evaluation report       |
| **Tag** (Nhãn)                                   | Key-value metadata tự do để filter và search runs                       |

---

## Tạo và Quản Lý Experiments {#tạo-experiments}

### Tạo Experiment

```python
import sagemaker
from sagemaker.experiments.run import Run
from sagemaker.session import Session

session = Session()

# Tạo experiment (hoặc dùng lại nếu đã tồn tại)
experiment_name = "fraud-detection-v2"

# Cách 1: Dùng context manager (được khuyến nghị)
with Run(
    experiment_name=experiment_name,
    run_name="xgboost-max-depth-6",    # Tên mô tả cho run này
    sagemaker_session=session,
) as run:
    # Mọi thứ log trong block này được gán cho run này
    run.log_parameter("algorithm", "XGBoost")
    run.log_parameter("max_depth", 6)
    run.log_parameter("eta", 0.1)
    
    # ... train model ...
    
    run.log_metric("auc", 0.921)
    run.log_metric("precision", 0.887)
    run.log_metric("recall", 0.876)

# Cách 2: Explicit load (dùng khi cần pass run giữa nhiều functions)
run = Run(
    experiment_name=experiment_name,
    run_name="lightgbm-baseline",
    sagemaker_session=session,
)
run.__enter__()    # Bắt đầu run context
```

### Load Existing Run

```python
from sagemaker.experiments.run import load_run

# Load run đang chạy (ví dụ: trong training script)
with load_run(sagemaker_session=session) as run:
    # Khi chạy trong SageMaker Training Job,
    # SageMaker tự inject experiment context vào environment
    run.log_metric("train_loss", train_loss, step=epoch)
```

---

## Logging Metrics và Parameters {#logging}

### Log Parameters (Tham Số)

```python
with Run(experiment_name="fraud-detection", run_name="run-001") as run:
    # Log single parameter
    run.log_parameter("max_depth", 6)
    run.log_parameter("eta", 0.1)
    run.log_parameter("num_round", 100)
    run.log_parameter("use_spot_instances", True)
    run.log_parameter("training_data_version", "v2.3")
    run.log_parameter("feature_count", 28)
    
    # Log nhiều parameters cùng lúc
    run.log_parameters({
        "subsample": 0.8,
        "colsample_bytree": 0.8,
        "eval_metric": "auc",
        "scale_pos_weight": 577,   # Class weight cho imbalanced data
    })
```

### Log Metrics (Số Liệu)

```python
with Run(experiment_name="fraud-detection", run_name="run-001") as run:
    for epoch in range(num_epochs):
        train_loss = compute_train_loss()
        val_auc = compute_val_auc()
        
        # Log metric theo step/epoch — tạo time series chart
        run.log_metric(name="train_loss", value=train_loss, step=epoch)
        run.log_metric(name="validation_auc", value=val_auc, step=epoch)
    
    # Log final metrics (không có step → scalar)
    run.log_metric("final_auc", 0.923)
    run.log_metric("final_precision", 0.891)
    run.log_metric("final_recall", 0.876)
    run.log_metric("final_f1", 0.883)
    run.log_metric("inference_latency_ms", 12.3)
    run.log_metric("training_time_seconds", 1842)
```

### Log Artifacts (Tạo Phẩm)

```python
with Run(experiment_name="fraud-detection", run_name="run-001") as run:
    # Log S3 URI của model
    run.log_artifact(
        name="model",
        value="s3://my-bucket/models/xgboost-v1/model.tar.gz",
        media_type="model/xgboost",
        is_output=True,        # True = output artifact, False = input artifact
    )
    
    # Log dataset URI (input)
    run.log_artifact(
        name="training_dataset",
        value="s3://my-bucket/data/v2.3/train.csv",
        media_type="text/csv",
        is_output=False,
    )
    
    # Upload file trực tiếp (không cần upload S3 thủ công)
    run.log_file(
        file_path="evaluation_report.json",  # File local
        name="evaluation_report",
        media_type="application/json",
        is_output=True,
    )
    
    # Log thư mục (ví dụ: confusing matrix plots)
    run.log_file(
        file_path="plots/",
        name="evaluation_plots",
        is_output=True,
    )
```

### Log Từ Training Script (Trong SageMaker Training Job)

```python
# train.py — chạy bên trong SageMaker Training Job container
import argparse
import xgboost as xgb
from sagemaker.experiments.run import load_run

def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--max-depth", type=int, default=6)
    parser.add_argument("--eta", type=float, default=0.1)
    return parser.parse_args()

def train(args):
    # load_run() tự động lấy experiment context từ environment variables
    # được SageMaker inject khi khởi chạy training job
    with load_run() as run:
        # Log hyperparameters
        run.log_parameters({
            "max_depth": args.max_depth,
            "eta": args.eta,
        })
        
        # Train XGBoost với callback
        dtrain = xgb.DMatrix("/opt/ml/input/data/train/train.csv")
        dval = xgb.DMatrix("/opt/ml/input/data/validation/val.csv")
        
        eval_results = {}
        
        def log_callback(env):
            """Log metrics sau mỗi iteration."""
            for name, value in env.evaluation_result_list:
                metric_name = name.split("-")[1]  # "train-auc" → "auc"
                run.log_metric(
                    name=f"train_{metric_name}" if "train" in name else f"val_{metric_name}",
                    value=value,
                    step=env.iteration,
                )
        
        model = xgb.train(
            params={"max_depth": args.max_depth, "eta": args.eta, "eval_metric": "auc"},
            dtrain=dtrain,
            evals=[(dtrain, "train"), (dval, "validation")],
            num_boost_round=100,
            callbacks=[log_callback],
            evals_result=eval_results,
        )
        
        # Log final metrics
        run.log_metric("final_val_auc", eval_results["validation"]["auc"][-1])
        
        # Save và log model
        model.save_model("/opt/ml/model/xgboost.model")

if __name__ == "__main__":
    train(parse_args())
```

---

## So Sánh Runs {#so-sánh-runs}

### So Sánh Trong SageMaker Studio

SageMaker Studio cung cấp UI trực quan để so sánh runs — chọn nhiều runs và xem bảng comparison.

### So Sánh Qua Python SDK

```python
from sagemaker.experiments.experiment import Experiment

# Lấy tất cả runs của experiment
experiment = Experiment.load(
    experiment_name="fraud-detection-v2",
    sagemaker_session=session,
)

# Liệt kê tất cả runs với metrics
runs_df = experiment.dataframe()

print(runs_df[[
    "run_name",
    "parameters.max_depth",
    "parameters.eta",
    "metrics.final_val_auc - Last",
    "metrics.training_time_seconds - Last",
    "start_time",
]].sort_values("metrics.final_val_auc - Last", ascending=False))

# Output ví dụ:
# run_name                max_depth  eta   auc    train_time  start_time
# lightgbm-num-leaves-63   N/A      N/A   0.934   2341s       2024-01-15 10:22
# xgboost-max-depth-8      8       0.05  0.923   1842s       2024-01-15 09:15
# xgboost-max-depth-6      6       0.1   0.912   1654s       2024-01-15 08:30
# xgboost-default          6       0.3   0.891   1234s       2024-01-14 16:00
```

### Tìm Best Run

```python
import pandas as pd

runs_df = experiment.dataframe()

# Tìm run có AUC cao nhất
best_run = runs_df.loc[
    runs_df["metrics.final_val_auc - Last"].idxmax()
]

print(f"Best Run: {best_run['run_name']}")
print(f"AUC: {best_run['metrics.final_val_auc - Last']:.4f}")
print(f"Parameters: max_depth={best_run['parameters.max_depth']}, eta={best_run['parameters.eta']}")

# Dùng best run để register model
best_run_obj = Run.load(
    run_name=best_run["run_name"],
    experiment_name="fraud-detection-v2",
    sagemaker_session=session,
)

# Lấy model artifact từ best run
model_artifact_uri = best_run["artifact.model - Last"]
print(f"Best model at: {model_artifact_uri}")
```

---

## Artifacts và Datasets {#artifacts}

### Dataset Tracking (Theo Dõi Bộ Dữ Liệu)

Tracking dataset version là quan trọng để đảm bảo reproducibility (Tái Lặp Được):

```python
from sagemaker.experiments.artifact import Artifact

# Đăng ký dataset như một artifact có thể tái dùng
training_dataset = Artifact.create(
    artifact_name="fraud-detection-training-v2.3",
    artifact_type="DataSet",
    source_uri="s3://my-bucket/data/v2.3/train.csv",
    source_types=[{"SourceIdType": "S3ETag", "Value": "abc123def456"}],  # Content hash
    properties={
        "record_count": "284807",
        "feature_count": "28",
        "positive_class_ratio": "0.0017",
        "date_range": "2024-01-01 to 2024-03-31",
        "preprocessing_version": "preprocess_v1.2.py",
    },
    tags=[
        {"Key": "version", "Value": "v2.3"},
        {"Key": "environment", "Value": "production"},
    ],
    sagemaker_session=session,
)

print(f"Dataset artifact ARN: {training_dataset.artifact_arn}")
```

### Liên Kết Artifact Với Run

```python
from sagemaker.lineage.association import Association

# Tạo liên kết: Dataset → Run (input relationship)
Association.create(
    source_arn=training_dataset.artifact_arn,
    destination_arn=run_arn,
    association_type="ContributedTo",    # Dataset contributed to this run
    sagemaker_session=session,
)
```

---

## ML Lineage Tracking {#lineage}

### Lineage Graph (Đồ Thị Nguồn Gốc)

**ML Lineage** ghi lại toàn bộ mối quan hệ giữa các entities trong ML workflow:

```
Data Collection   Training Job     Model         Endpoint
(Thu Thập Dữ)    (Công Việc HLuận) (Mô Hình)     (Điểm Cuối)
     │                 │               │               │
   Artifact         Context          Artifact        Context
   (raw_data)       (training)       (model_v3)      (prod_endpoint)
     │                 │               │               │
     └────────────────►│───────────────►│───────────────►│
                  "ContributedTo"  "Model"          "Endpoint"

SageMaker tự động tạo Lineage graph này cho tất cả SageMaker objects!
```

### Loại Entities Trong Lineage

| Entity Type                          | Mô Tả                                               | Ví Dụ                           |
| ------------------------------------ | --------------------------------------------------- | --------------------------------|
| **Artifact** (Tạo Phẩm)              | File, dataset, model artifact trên S3               | train.csv, model.tar.gz         |
| **Context** (Ngữ Cảnh)              | Container logic cho activities (experiment, endpoint)| Training Job, Endpoint          |
| **Action** (Hành Động)               | Sự kiện xảy ra (training, deployment, approval)     | ModelApproval, EndpointDeploy   |
| **Association** (Liên Kết)           | Mối quan hệ giữa các entities                       | "ContributedTo", "DerivedFrom"  |

### Truy Vấn Lineage

```python
from sagemaker.lineage.query import (
    LineageQuery,
    LineageFilter,
    LineageEntityEnum,
    LineageSourceEnum,
)

lineage_query = LineageQuery(sagemaker_session=session)

# Câu hỏi: Model này được tạo từ data nào?
result = lineage_query.query(
    start_arns=[model_artifact_arn],
    direction="Ascendants",              # Truy ngược về nguồn gốc
    include_edges=True,
    filters=LineageFilter(
        entities=[LineageEntityEnum.ARTIFACT],
        sources=[LineageSourceEnum.DATASET],
    ),
)

print("Datasets dùng để train model này:")
for vertex in result.vertices:
    print(f"  - {vertex.arn}: {vertex.lineage_source}")
```

```python
# Câu hỏi: Dataset này đã được dùng trong những model nào?
result = lineage_query.query(
    start_arns=[training_dataset.artifact_arn],
    direction="Descendants",             # Truy tìm các models sinh ra từ data này
    include_edges=True,
)

print("Các models được train từ dataset này:")
for vertex in result.vertices:
    if vertex.lineage_entity == "Model":
        print(f"  - {vertex.arn}")
```

### Lineage Tự Động Trong SageMaker Pipelines

Khi dùng SageMaker Pipelines, lineage được ghi lại tự động — không cần code thêm:

```
Pipeline Run
├── ProcessingStep "PrepareData"
│   ├── Input:  s3://bucket/raw/transactions.csv  → Artifact (raw_data)
│   └── Output: s3://bucket/processed/train.csv   → Artifact (processed_train)
│                        ↓ (DerivedFrom)
├── TrainingStep "TrainXGBoost"
│   ├── Input:  Artifact (processed_train)
│   └── Output: s3://bucket/models/model.tar.gz   → Artifact (model_v3)
│                        ↓ (DerivedFrom)
└── RegisterModel "RegisterFraudModel"
    ├── Input:  Artifact (model_v3)
    └── Output: Model Package → Artifact (model_package_v3)

Tất cả mối quan hệ này được SageMaker ghi vào Lineage Store tự động!
```

---

## Tích Hợp Với SageMaker Pipelines {#tích-hợp-pipeline}

### Tự Động Ghi Experiment Từ Pipeline

```python
from sagemaker.workflow.pipeline_context import PipelineSession
from sagemaker.experiments.run import Run

pipeline_session = PipelineSession()

# Training step tự động tạo run trong experiment khi pipeline chạy
xgb_estimator = XGBoost(
    entry_point="train.py",
    framework_version="1.7-1",
    instance_type=training_instance_param,
    role=role,
    hyperparameters={
        "max-depth": max_depth_param,
        "eta": eta_param,
    },
    # Tất cả metrics, params từ train.py sẽ được log vào experiment này
    environment={
        "SAGEMAKER_EXPERIMENT_NAME": "fraud-detection-v2",
    },
)

step_train = TrainingStep(
    name="TrainModel",
    estimator=xgb_estimator,
    inputs={"train": train_input},
    # experiment_config tự động inherit từ Pipeline
)
```

### Liên Kết Pipeline Run Với Experiment

```python
# Khi start pipeline, associate với experiment
execution = pipeline.start(
    experiment_config={
        "ExperimentName": "fraud-detection-v2",
        "TrialName": f"pipeline-run-{datetime.now().strftime('%Y%m%d-%H%M%S')}",
        "TrialComponentDisplayName": "pipeline-execution",
    }
)
```

---

## Câu Hỏi Phỏng Vấn {#phỏng-vấn}

**H: Experiment tracking giải quyết vấn đề gì trong ML development?**

> Experiment tracking giải quyết ba vấn đề chính:
>
> 1. **Reproducibility** (Tái Lặp Được): Ghi lại chính xác code version, data version, hyperparameters và môi trường → có thể tái tạo lại kết quả bất kỳ run nào
> 2. **Comparison** (So Sánh): Dễ dàng so sánh hàng trăm experiments để tìm model tốt nhất mà không cần ghi chú thủ công
> 3. **Collaboration** (Cộng Tác): Team có thể xem kết quả của nhau, tránh làm trùng công việc đã được thực hiện

**H: Lineage tracking khác gì với experiment tracking?**

> - **Experiment tracking**: Theo dõi *quá trình thực nghiệm* — hyperparameters, metrics theo thời gian, artifacts của một run. Tập trung vào "chúng ta đã thử gì và kết quả ra sao?"
> - **ML Lineage**: Theo dõi *mối quan hệ nhân quả* — dataset nào → training job nào → model nào → endpoint nào. Tập trung vào "artifact này đến từ đâu và được dùng ở đâu?"
>
> Chúng bổ sung cho nhau: Experiment tracking để *tìm* model tốt nhất, Lineage để *audit* và *truy vết* model đó sau khi deploy.

**H: Tại sao ghi lại data version trong experiments quan trọng?**

> Khi model hoạt động kém trong production, câu hỏi đầu tiên thường là: "Model này được train trên data nào?" Nếu không ghi lại data version:
> - Không thể tái tạo lại training để debug
> - Không biết data có bị contaminated (nhiễm bẩn) không
> - Không thể kiểm tra xem data drift bắt đầu từ khi nào
>
> Data versioning + experiment tracking = đảm bảo bạn có thể *luôn luôn* trả lời câu hỏi "model này được train như thế nào" dù sau 1 năm.

**H: SageMaker Experiments vs MLflow — khi nào chọn cái nào?**

> - **SageMaker Experiments**: Chọn khi toàn bộ ML stack đang dùng AWS, muốn native integration với SageMaker Training Jobs, Pipelines, Registry mà không cần thêm infrastructure. Lineage tracking tự động.
> - **MLflow**: Chọn khi cần portable solution chạy được trên nhiều cloud hoặc on-premises, hoặc team quen thuộc với MLflow. Flexible hơn nhưng cần self-host hoặc dùng managed version (Databricks, Azure ML).
>
> **Không nên dùng cả hai** đồng thời cho cùng một project — gây phân tán context và khó maintain.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
