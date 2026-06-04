# SageMaker Pipelines — CI/CD Cho Machine Learning

> **SageMaker Pipelines** là dịch vụ orchestration (Điều Phối) workflow ML được tích hợp sẵn trong Amazon SageMaker. Nó cho phép tạo, tự động hóa và quản lý các quy trình ML end-to-end dưới dạng DAG (Directed Acyclic Graph — Đồ Thị Không Có Chu Trình Có Hướng), tương tự CI/CD pipeline trong phát triển phần mềm nhưng chuyên biệt cho ML.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#khái-niệm-cơ-bản)
2. [Các Loại Step](#các-loại-step)
3. [Pipeline DAG và Luồng Thực Thi](#dag-và-luồng)
4. [Parameters và Conditions](#parameters-và-conditions)
5. [Pipeline Triggers](#pipeline-triggers)
6. [Caching và Tối Ưu](#caching)
7. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#phỏng-vấn)

---

## Khái Niệm Cơ Bản {#khái-niệm-cơ-bản}

### SageMaker Pipelines Là Gì

**SageMaker Pipelines** là ML workflow orchestration tool với các đặc điểm:

- **DAG-based** (Dựa Trên Đồ Thị Không Chu Trình): Các bước liên kết thành đồ thị, hỗ trợ thực thi song song
- **Native SageMaker integration**: Tích hợp nội bộ với SageMaker Training, Processing, Registry
- **Lineage tracking** (Theo Dõi Nguồn Gốc): Tự động ghi lại metadata và lineage cho mọi run
- **Step caching** (Lưu Kết Quả Bước): Tái dùng kết quả của bước chưa thay đổi để tiết kiệm thời gian/chi phí
- **Visual DAG editor**: Xem trực quan trên SageMaker Studio

### Kiến Trúc Tổng Quan

```
┌──────────────────────────────────────────────────────────────────┐
│                    SageMaker Pipeline                            │
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌───────────┐  │
│  │Processing│───►│Training  │───►│Evaluation│───►│Condition  │  │
│  │  Step    │    │  Step    │    │  Step    │    │  Step     │  │
│  │(Xử Lý)  │    │(Huấn     │    │(Đánh Giá)│    │(Điều Kiện)│  │
│  └──────────┘    │Luyện)    │    └──────────┘    └─────┬─────┘  │
│                  └──────────┘                          │        │
│                                               ┌────────┴──────┐ │
│                                               │               │ │
│                                        ┌──────▼──────┐ ┌─────▼─┴────┐ │
│                                        │ Register    │ │   Fail     │ │
│                                        │ Model Step  │ │   Step     │ │
│                                        │(Đăng Ký)   │ │(Thất Bại) │ │
│                                        └─────────────┘ └────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Khái Niệm Chính

| Khái Niệm                                | Định Nghĩa                                                            |
| ---------------------------------------- | --------------------------------------------------------------------- |
| **Pipeline**                             | Toàn bộ workflow ML, gồm nhiều steps kết nối với nhau                |
| **Step** (Bước)                          | Một đơn vị công việc trong pipeline (processing, training, v.v.)     |
| **DAG** (Directed Acyclic Graph)         | Đồ Thị Không Có Chu Trình Có Hướng — cách tổ chức các bước          |
| **Pipeline Run** (Lần Chạy Pipeline)     | Một lần thực thi pipeline với bộ tham số cụ thể                      |
| **Pipeline Parameter** (Tham Số Pipeline)| Giá trị đầu vào có thể thay đổi mỗi lần chạy (ví dụ: learning rate) |
| **Step Cache** (Bộ Nhớ Đệm Bước)        | Kết quả được lưu lại và tái dùng nếu inputs không thay đổi          |
| **Lineage** (Nguồn Gốc)                  | Lịch sử và mối quan hệ giữa data, code, model trong một run          |

---

## Các Loại Step {#các-loại-step}

### 1. ProcessingStep (Bước Xử Lý)

Dùng SageMaker Processing Job để chạy code xử lý dữ liệu, feature engineering, model evaluation.

```python
from sagemaker.workflow.steps import ProcessingStep
from sagemaker.sklearn import SKLearnProcessor

sklearn_processor = SKLearnProcessor(
    framework_version="1.2-1",
    instance_type="ml.m5.large",
    instance_count=1,
    role=role,
)

step_process = ProcessingStep(
    name="PreprocessData",               # Tên bước, duy nhất trong pipeline
    processor=sklearn_processor,
    inputs=[
        ProcessingInput(
            source=raw_data_s3_uri,      # Dữ liệu thô từ S3
            destination="/opt/ml/processing/input",
        )
    ],
    outputs=[
        ProcessingOutput(
            output_name="train",
            source="/opt/ml/processing/train",
        ),
        ProcessingOutput(
            output_name="validation",
            source="/opt/ml/processing/validation",
        ),
    ],
    code="preprocess.py",
    cache_config=cache_config,           # Bật step caching
)
```

**Dùng Khi Nào:**
- Tiền xử lý dữ liệu (chuẩn hóa, mã hóa, xử lý missing values)
- Feature engineering (Kỹ Thuật Đặc Trưng)
- Đánh giá model và tạo báo cáo
- Chạy các script Python tùy chỉnh

### 2. TrainingStep (Bước Huấn Luyện)

Khởi chạy SageMaker Training Job với code và hyperparameters cụ thể.

```python
from sagemaker.workflow.steps import TrainingStep
from sagemaker.xgboost import XGBoost

xgb_estimator = XGBoost(
    entry_point="train.py",
    framework_version="1.7-1",
    instance_type="ml.m5.xlarge",
    instance_count=1,
    role=role,
    hyperparameters={
        "max_depth": max_depth_param,    # Pipeline Parameter
        "eta": eta_param,
        "num_round": num_round_param,
    },
    use_spot_instances=True,             # Dùng Spot để tiết kiệm chi phí
    max_wait=7200,
)

step_train = TrainingStep(
    name="TrainModel",
    estimator=xgb_estimator,
    inputs={
        "train": TrainingInput(
            s3_data=step_process.properties.ProcessingOutputConfig
                .Outputs["train"].S3Output.S3Uri,
            content_type="text/csv",
        ),
    },
    cache_config=cache_config,
)
```

### 3. TuningStep (Bước Tinh Chỉnh Tham Số)

Chạy Hyperparameter Optimization (HPO — Tối Ưu Siêu Tham Số) với SageMaker Automatic Model Tuning.

```python
from sagemaker.workflow.steps import TuningStep
from sagemaker.tuner import HyperparameterTuner, IntegerParameter, ContinuousParameter

tuner = HyperparameterTuner(
    estimator=xgb_estimator,
    objective_metric_name="validation:auc",
    hyperparameter_ranges={
        "max_depth": IntegerParameter(3, 10),
        "eta": ContinuousParameter(0.01, 0.3),
    },
    max_jobs=20,
    max_parallel_jobs=4,
)

step_tuning = TuningStep(
    name="HPO",
    tuner=tuner,
    inputs={
        "train": TrainingInput(s3_data=train_uri),
        "validation": TrainingInput(s3_data=val_uri),
    },
)
```

### 4. ModelStep (Bước Tạo Model)

Tạo SageMaker Model object từ kết quả training để chuẩn bị cho deployment.

```python
from sagemaker.workflow.model_step import ModelStep

best_model = Model(
    image_uri=image_uri,
    model_data=step_train.properties.ModelArtifacts.S3ModelArtifacts,
    sagemaker_session=pipeline_session,
    role=role,
)

step_create_model = ModelStep(
    name="CreateModel",
    step_args=best_model.create(instance_type="ml.m5.large"),
)
```

### 5. TransformStep (Bước Biến Đổi Hàng Loạt)

Chạy Batch Transform để inference trên tập dữ liệu lớn.

```python
from sagemaker.workflow.steps import TransformStep

transformer = Transformer(
    model_name=step_create_model.properties.ModelName,
    instance_type="ml.m5.xlarge",
    instance_count=1,
    output_path=batch_output_s3_uri,
)

step_transform = TransformStep(
    name="BatchTransform",
    transformer=transformer,
    inputs=TransformInput(data=test_data_s3_uri),
)
```

### 6. ConditionStep (Bước Điều Kiện)

Phân nhánh pipeline dựa trên điều kiện — chỉ đăng ký model nếu đạt ngưỡng chất lượng.

```python
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.functions import JsonGet

# Đọc AUC từ evaluation report
auc_score = JsonGet(
    step_name=step_evaluate.name,
    property_file=evaluation_report,
    json_path="binary_classification_metrics.auc.value",
)

# Chỉ đăng ký nếu AUC >= 0.85
condition_auc = ConditionGreaterThanOrEqualTo(
    left=auc_score,
    right=0.85,
)

step_condition = ConditionStep(
    name="CheckModelQuality",
    conditions=[condition_auc],
    if_steps=[step_register],        # Đăng ký model nếu AUC đạt
    else_steps=[step_fail],          # Thất bại nếu không đạt
)
```

### 7. RegisterModel / ModelStep (Bước Đăng Ký Model)

Đăng ký model vào SageMaker Model Registry.

```python
from sagemaker.workflow.step_collections import RegisterModel

step_register = RegisterModel(
    name="RegisterModel",
    estimator=xgb_estimator,
    model_data=step_train.properties.ModelArtifacts.S3ModelArtifacts,
    content_types=["text/csv"],
    response_types=["text/csv"],
    inference_instances=["ml.m5.large"],
    transform_instances=["ml.m5.large"],
    model_package_group_name="FraudDetectionModelGroup",
    approval_status="PendingManualApproval",   # Chờ phê duyệt thủ công
    model_metrics=model_metrics,
)
```

### 8. FailStep (Bước Thất Bại)

Kết thúc pipeline với trạng thái failure và message tùy chỉnh.

```python
from sagemaker.workflow.fail_step import FailStep

step_fail = FailStep(
    name="ModelQualityFailed",
    error_message=Join(
        on=" ",
        values=["AUC score", auc_score, "does not meet threshold 0.85"],
    ),
)
```

---

## Pipeline DAG và Luồng Thực Thi {#dag-và-luồng}

### Cách Kết Nối Steps

SageMaker Pipelines tự động suy ra thứ tự thực thi dựa trên **data dependencies** (Phụ Thuộc Dữ Liệu) — không cần định nghĩa thứ tự thủ công.

```python
# Pipeline tự biết step_train phải chạy sau step_process
# vì step_train dùng output của step_process
step_train = TrainingStep(
    name="TrainModel",
    inputs={
        "train": TrainingInput(
            # Tham chiếu đến output của step_process → tự động tạo dependency
            s3_data=step_process.properties.ProcessingOutputConfig
                .Outputs["train"].S3Output.S3Uri
        )
    },
    ...
)
```

### Thực Thi Song Song (Parallel Execution)

Nếu hai steps không có dependency, chúng sẽ chạy song song:

```
step_process_train ──┐
                     ├──► step_train ──► step_evaluate
step_process_val  ───┘
```

```python
# step_process_train và step_process_val chạy song song
# step_train chờ cả hai hoàn thành
step_train = TrainingStep(
    depends_on=[step_process_train, step_process_val],  # Explicit dependency
    ...
)
```

### Tạo và Chạy Pipeline

```python
from sagemaker.workflow.pipeline import Pipeline

pipeline = Pipeline(
    name="FraudDetectionPipeline",
    parameters=[
        max_depth_param,
        eta_param,
        num_round_param,
        training_instance_type,
    ],
    steps=[
        step_process,
        step_train,
        step_evaluate,
        step_condition,    # Nếu condition pass → step_register, else → step_fail
    ],
    sagemaker_session=pipeline_session,
)

# Tạo hoặc cập nhật pipeline definition trên AWS
pipeline.upsert(role_arn=role)

# Chạy pipeline với parameters tùy chỉnh
execution = pipeline.start(
    parameters={
        "MaxDepth": 6,
        "Eta": 0.1,
        "NumRound": 100,
    }
)

# Đợi pipeline hoàn thành (blocking)
execution.wait()

# Xem list steps đã chạy
execution.list_steps()
```

---

## Parameters và Conditions {#parameters-và-conditions}

### Pipeline Parameters (Tham Số Pipeline)

Cho phép thay đổi giá trị mỗi lần chạy pipeline mà không cần sửa definition.

```python
from sagemaker.workflow.parameters import (
    ParameterInteger, ParameterFloat, ParameterString, ParameterBoolean
)

# Tham số kiểu số nguyên
max_depth_param = ParameterInteger(
    name="MaxDepth",
    default_value=6,
)

# Tham số kiểu số thực
eta_param = ParameterFloat(
    name="Eta",
    default_value=0.1,
)

# Tham số kiểu chuỗi
training_instance_type = ParameterString(
    name="TrainingInstanceType",
    default_value="ml.m5.xlarge",
)

# Tham số kiểu boolean
use_spot = ParameterBoolean(
    name="UseSpotInstances",
    default_value=True,
)
```

### Conditions (Điều Kiện)

```python
from sagemaker.workflow.conditions import (
    ConditionGreaterThan,
    ConditionGreaterThanOrEqualTo,
    ConditionLessThan,
    ConditionEquals,
    ConditionAnd,
    ConditionOr,
)

# Điều kiện kết hợp AND
combined_condition = ConditionAnd(conditions=[
    ConditionGreaterThanOrEqualTo(left=auc_score, right=0.85),
    ConditionLessThan(left=inference_latency, right=100),  # < 100ms
])
```

---

## Pipeline Triggers {#pipeline-triggers}

### 1. Manual Trigger (Kích Hoạt Thủ Công)

```python
execution = pipeline.start()
```

### 2. EventBridge Schedule (Lịch Định Kỳ)

Chạy pipeline tự động theo lịch — ví dụ: mỗi tuần retrain với dữ liệu mới.

```python
import boto3

events_client = boto3.client("events")

# Tạo rule chạy mỗi thứ Hai lúc 2 giờ sáng UTC
events_client.put_rule(
    Name="WeeklyMLRetrainSchedule",
    ScheduleExpression="cron(0 2 ? * MON *)",   # Every Monday at 2am UTC
    State="ENABLED",
)

# Target là SageMaker Pipeline
events_client.put_targets(
    Rule="WeeklyMLRetrainSchedule",
    Targets=[{
        "Id": "SageMakerPipelineTarget",
        "Arn": f"arn:aws:sagemaker:{region}:{account}:pipeline/FraudDetectionPipeline",
        "RoleArn": events_role_arn,
        "SageMakerPipelineParameters": {
            "PipelineParameterList": [
                {"Name": "MaxDepth", "Value": "6"},
            ]
        },
    }],
)
```

### 3. S3 Event Trigger (Kích Hoạt Khi Có Dữ Liệu Mới)

Tự động chạy pipeline khi có file mới trong S3.

```
S3 PutObject → S3 Event Notification → EventBridge → Lambda → Pipeline.start()
(File Mới)     (Thông Báo Sự Kiện)   (Cầu Sự Kiện) (Hàm)   (Chạy Pipeline)
```

### 4. Model Monitor Alert Trigger (Kích Hoạt Từ Cảnh Báo Monitor)

Tự động retrain khi Model Monitor phát hiện drift:

```
Model Monitor violation → CloudWatch Alarm → SNS Topic → Lambda → Pipeline.start()
(Vi Phạm Monitor)         (Cảnh Báo)         (Chủ Đề)    (Hàm)   (Retrain)
```

---

## Caching và Tối Ưu {#caching}

### Step Caching (Lưu Đệm Bước)

Step caching tái sử dụng kết quả của bước trước nếu inputs không thay đổi — tiết kiệm thời gian và chi phí đáng kể.

```python
from sagemaker.workflow.steps import CacheConfig

# Bật caching, expire sau 30 ngày
cache_config = CacheConfig(
    enable_caching=True,
    expire_after="P30D",   # ISO 8601 duration: 30 Days
)

# Áp dụng cho step
step_process = ProcessingStep(
    name="PreprocessData",
    ...
    cache_config=cache_config,   # Step này sẽ được cache
)
```

**Cache hoạt động khi:** Cùng inputs (S3 URIs, hyperparameters, code hash) → tái dùng output từ lần chạy trước.

**Cache bị bỏ qua khi:** Inputs thay đổi, cache đã expire, hoặc force_rerun=True.

### Pipeline Parallelism (Song Song Hóa)

```python
# Tách preprocessing thành nhiều jobs nhỏ chạy song song
feature_steps = []
for feature_group in feature_groups:
    step = ProcessingStep(
        name=f"Process_{feature_group}",
        ...
    )
    feature_steps.append(step)

# Merge step chạy sau khi tất cả feature steps hoàn thành
step_merge = ProcessingStep(
    name="MergeFeatures",
    depends_on=feature_steps,   # Chờ tất cả feature steps
    ...
)
```

---

## Ví Dụ Thực Tế {#ví-dụ-thực-tế}

### Pipeline Hoàn Chỉnh: Phân Loại Gian Lận

```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.parameters import ParameterFloat, ParameterString
from sagemaker.workflow.steps import (
    ProcessingStep, TrainingStep
)
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.functions import JsonGet
from sagemaker.workflow.fail_step import FailStep

# 1. Tham số Pipeline
auc_threshold = ParameterFloat(name="AucThreshold", default_value=0.85)
training_instance = ParameterString(
    name="TrainingInstance", default_value="ml.m5.xlarge"
)

# 2. Bước xử lý dữ liệu
step_process = ProcessingStep(
    name="PrepareData",
    processor=sklearn_processor,
    code="preprocess.py",
    inputs=[ProcessingInput(source=raw_data_uri, destination="/opt/ml/processing/input")],
    outputs=[
        ProcessingOutput(output_name="train", source="/opt/ml/processing/train"),
        ProcessingOutput(output_name="val", source="/opt/ml/processing/val"),
    ],
    cache_config=CacheConfig(enable_caching=True, expire_after="P7D"),
)

# 3. Bước huấn luyện
step_train = TrainingStep(
    name="TrainXGBoost",
    estimator=xgb_estimator,
    inputs={
        "train": TrainingInput(
            s3_data=step_process.properties.ProcessingOutputConfig.Outputs["train"].S3Output.S3Uri
        ),
        "validation": TrainingInput(
            s3_data=step_process.properties.ProcessingOutputConfig.Outputs["val"].S3Output.S3Uri
        ),
    },
)

# 4. Bước đánh giá
step_evaluate = ProcessingStep(
    name="EvaluateModel",
    processor=sklearn_processor,
    code="evaluate.py",
    inputs=[
        ProcessingInput(
            source=step_train.properties.ModelArtifacts.S3ModelArtifacts,
            destination="/opt/ml/processing/model",
        ),
    ],
    outputs=[
        ProcessingOutput(
            output_name="evaluation",
            source="/opt/ml/processing/evaluation",
        )
    ],
    property_files=[evaluation_report],
)

# 5. Đọc AUC từ evaluation report
auc_score = JsonGet(
    step_name=step_evaluate.name,
    property_file=evaluation_report,
    json_path="binary_classification_metrics.auc.value",
)

# 6. Điều kiện: chỉ đăng ký nếu AUC đạt ngưỡng
step_condition = ConditionStep(
    name="CheckAUC",
    conditions=[ConditionGreaterThanOrEqualTo(left=auc_score, right=auc_threshold)],
    if_steps=[step_register],
    else_steps=[FailStep(name="AUCTooLow", error_message="AUC below threshold")],
)

# 7. Tạo pipeline
pipeline = Pipeline(
    name="FraudDetectionPipeline",
    parameters=[auc_threshold, training_instance],
    steps=[step_process, step_train, step_evaluate, step_condition],
)

pipeline.upsert(role_arn=role)
```

---

## Câu Hỏi Phỏng Vấn {#phỏng-vấn}

**H: SageMaker Pipelines khác gì với Airflow hay Step Functions trong ML workflow?**

> - **SageMaker Pipelines**: Chuyên biệt cho ML, native tích hợp SageMaker (Training Jobs, Processing Jobs, Model Registry), có step caching, lineage tự động, visual editor trong Studio. Phù hợp nhất khi toàn bộ stack là AWS.
> - **Apache Airflow (MWAA)**: Linh hoạt hơn, hỗ trợ nhiều loại tác vụ hơn (không chỉ ML), nhưng cần setup và quản lý nhiều hơn. Phù hợp khi cần kết hợp ML steps với data pipeline phức tạp.
> - **AWS Step Functions**: Orchestration tổng quát hơn, có thể gọi bất kỳ AWS service nào. Phù hợp khi cần tích hợp ML vào business workflow phức tạp.

**H: Step caching trong SageMaker Pipelines hoạt động thế nào?**

> SageMaker tính một **cache key** dựa trên: (1) code script hash, (2) container image URI, (3) input data S3 URIs, (4) hyperparameters và arguments. Nếu cache key giống lần chạy trước và chưa expire, SageMaker tái dùng output từ lần chạy đó thay vì chạy lại. Đặc biệt hữu ích cho ProcessingStep xử lý dữ liệu tốn thời gian — nếu data không thay đổi thì không cần xử lý lại.

**H: Làm sao để handle pipeline failure và retry?**

> - Dùng **FailStep** để kết thúc pipeline có kiểm soát khi điều kiện không thỏa
> - Cấu hình **retry policies** trên từng step để tự động retry khi có transient error
> - Dùng **CloudWatch Alarms** kết hợp **SNS** để nhận alert khi pipeline fail
> - Implement **Dead Letter Queue** (Hàng Đợi Thư Chết) để capture và xử lý các events không thể process

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
