# SageMaker Model Registry — Kho Lưu Trữ và Quản Lý Mô Hình ML

> **SageMaker Model Registry** (Kho Mô Hình SageMaker) là dịch vụ quản lý vòng đời mô hình ML tập trung: đăng ký các phiên bản (versions) mô hình, lưu metadata, theo dõi lineage (nguồn gốc), và quản lý approval workflow (Quy Trình Phê Duyệt) trước khi deploy vào production.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#khái-niệm)
2. [Model Package Group](#model-package-group)
3. [Đăng Ký Model](#đăng-ký-model)
4. [Approval Workflow](#approval-workflow)
5. [Model Metadata và Lineage](#metadata-và-lineage)
6. [Deploy Từ Registry](#deploy-từ-registry)
7. [Tích Hợp Với CI/CD](#tích-hợp-cicd)
8. [Câu Hỏi Phỏng Vấn](#phỏng-vấn)

---

## Khái Niệm Cơ Bản {#khái-niệm}

### Model Registry Là Gì

**Model Registry** giải quyết câu hỏi trong production ML:

- Model nào đang chạy trong production? (Version tracking — Theo Dõi Phiên Bản)
- Model đó được train từ data nào, code nào? (Lineage — Nguồn Gốc)
- Ai đã review và approve model đó? (Governance — Quản Trị)
- Hiệu suất của model đó ở thời điểm deploy là bao nhiêu? (Metrics storage — Lưu Số Liệu)

### Các Khái Niệm Chính

| Khái Niệm                                            | Định Nghĩa                                                                        |
| ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Model Package Group** (Nhóm Gói Mô Hình)          | Container logic gom tất cả versions của một model (ví dụ: FraudDetectionModel)   |
| **Model Package** (Gói Mô Hình)                      | Một version cụ thể của model, gồm artifacts, metadata và approval status         |
| **Model Artifacts** (Tạo Phẩm Mô Hình)              | File model.tar.gz chứa weights và serialized model lưu trên S3                   |
| **Approval Status** (Trạng Thái Phê Duyệt)           | PendingManualApproval / Approved / Rejected — quyết định model có được deploy không |
| **Model Card** (Thẻ Mô Hình)                         | Tài liệu hóa mô hình: mục đích, performance, bias, rủi ro                       |
| **Model Lineage** (Nguồn Gốc Mô Hình)               | Đồ thị liên kết: Dataset → Training Job → Model → Endpoint                       |

### Luồng Tổng Quan

```
Train Model  →  Register Version  →  Review & Approve  →  Deploy
(Huấn Luyện)   (Đăng Ký Phiên Bản)  (Xem Xét & Phê Duyệt) (Triển Khai)
       │               │                     │                    │
  S3 Artifacts    Model Package       Approval Status        Endpoint
  (Tạo phẩm)    (Gói Model v1.3)     (Approved/Rejected)   (Real-time)
```

---

## Model Package Group {#model-package-group}

### Tạo Model Package Group

```python
import boto3
import sagemaker

sm_client = boto3.client("sagemaker")

# Tạo Model Package Group để chứa tất cả versions của fraud detection model
model_package_group_name = "FraudDetectionModelGroup"

sm_client.create_model_package_group(
    ModelPackageGroupName=model_package_group_name,
    ModelPackageGroupDescription=(
        "Nhóm chứa tất cả versions của mô hình phát hiện gian lận giao dịch. "
        "Train với XGBoost trên dữ liệu giao dịch thẻ tín dụng."
    ),
    Tags=[
        {"Key": "Project", "Value": "FraudDetection"},
        {"Key": "Team", "Value": "MLPlatform"},
        {"Key": "Environment", "Value": "Production"},
    ],
)
```

### Xem Danh Sách Model Package Groups

```python
response = sm_client.list_model_package_groups(
    SortBy="CreationTime",
    SortOrder="Descending",
)

for group in response["ModelPackageGroupSummaryList"]:
    print(f"Group: {group['ModelPackageGroupName']}")
    print(f"  ARN: {group['ModelPackageGroupArn']}")
    print(f"  Status: {group['ModelPackageGroupStatus']}")
    print()
```

---

## Đăng Ký Model {#đăng-ký-model}

### Cách 1: Đăng Ký Từ Estimator (Sau Training)

```python
from sagemaker.model_metrics import MetricsSource, ModelMetrics
from sagemaker.drift_check_baselines import DriftCheckBaselines

# Định nghĩa metrics cho model
model_metrics = ModelMetrics(
    model_statistics=MetricsSource(
        s3_uri=f"{evaluation_output_uri}/evaluation.json",
        content_type="application/json",
    ),
    model_constraints=MetricsSource(
        s3_uri=f"{constraints_uri}/model_constraints.json",
        content_type="application/json",
    ),
)

# Định nghĩa drift check baselines (Đường Cơ Sở Kiểm Tra Trôi Dạt)
drift_check_baselines = DriftCheckBaselines(
    model_statistics=MetricsSource(
        s3_uri=f"{baseline_uri}/statistics.json",
        content_type="application/json",
    ),
    model_constraints=MetricsSource(
        s3_uri=f"{baseline_uri}/constraints.json",
        content_type="application/json",
    ),
)

# Đăng ký model package
model_package = estimator.register(
    content_types=["text/csv"],
    response_types=["text/csv"],
    inference_instances=["ml.m5.large", "ml.m5.xlarge"],
    transform_instances=["ml.m5.large"],
    model_package_group_name=model_package_group_name,
    approval_status="PendingManualApproval",    # Chờ người review
    description="XGBoost v1.7 trained on 2024 Q1 transaction data. AUC=0.923",
    model_metrics=model_metrics,
    drift_check_baselines=drift_check_baselines,
    customer_metadata_properties={
        "training_data_date": "2024-01-01",
        "dataset_version": "v2.3",
        "training_duration_minutes": "45",
        "feature_count": "28",
    },
)

print(f"Model Package ARN: {model_package.model_package_arn}")
```

### Cách 2: Đăng Ký Trực Tiếp Qua Boto3

```python
model_package_arn = sm_client.create_model_package(
    ModelPackageGroupName=model_package_group_name,
    ModelPackageDescription="XGBoost Fraud Detection v1.3",
    InferenceSpecification={
        "Containers": [
            {
                "Image": xgboost_image_uri,
                "ModelDataUrl": "s3://my-bucket/models/model.tar.gz",
            }
        ],
        "SupportedContentTypes": ["text/csv"],
        "SupportedResponseMIMETypes": ["text/csv"],
        "SupportedRealtimeInferenceInstanceTypes": ["ml.m5.large"],
        "SupportedTransformInstanceTypes": ["ml.m5.large"],
    },
    ModelApprovalStatus="PendingManualApproval",
    ModelMetrics={
        "ModelQuality": {
            "Statistics": {
                "S3Uri": f"{evaluation_uri}/evaluation.json",
                "ContentType": "application/json",
            }
        }
    },
    CustomerMetadataProperties={
        "auc": "0.923",
        "precision": "0.891",
        "recall": "0.876",
    },
)["ModelPackageArn"]
```

### Cách 3: Đăng Ký Từ SageMaker Pipelines (RegisterModel Step)

```python
from sagemaker.workflow.step_collections import RegisterModel

step_register = RegisterModel(
    name="RegisterFraudModel",
    estimator=xgb_estimator,
    model_data=step_train.properties.ModelArtifacts.S3ModelArtifacts,
    content_types=["text/csv"],
    response_types=["text/csv"],
    inference_instances=["ml.m5.large"],
    transform_instances=["ml.m5.large"],
    model_package_group_name=model_package_group_name,
    approval_status="PendingManualApproval",
    model_metrics=model_metrics,
    drift_check_baselines=drift_check_baselines,
)
```

---

## Approval Workflow {#approval-workflow}

### Các Trạng Thái Phê Duyệt

```
PendingManualApproval  →  Approved   →  Deploy to Production
(Chờ Phê Duyệt)          (Đã Phê Duyệt) (Triển Khai)
                    ↘
                      Rejected
                      (Bị Từ Chối)
```

### Phê Duyệt Thủ Công Qua Boto3

```python
# Approve model để sẵn sàng deploy
sm_client.update_model_package(
    ModelPackageArn=model_package_arn,
    ModelApprovalStatus="Approved",
    ApprovalDescription=(
        "Model reviewed by ML team. AUC=0.923 meets production threshold 0.85. "
        "Bias metrics passed fairness checks. Approved for production deployment."
    ),
)

# Từ chối model với lý do cụ thể
sm_client.update_model_package(
    ModelPackageArn=model_package_arn,
    ModelApprovalStatus="Rejected",
    ApprovalDescription=(
        "Model rejected: Recall for high-risk transactions (class 1) = 0.72, "
        "below minimum threshold 0.80. Needs retraining with rebalanced dataset."
    ),
)
```

### Tự Động Hóa Approval Với EventBridge

Khi model được approve → tự động trigger deployment pipeline:

```python
# EventBridge rule lắng nghe khi model package được approve
{
    "source": ["aws.sagemaker"],
    "detail-type": ["SageMaker Model Package State Change"],
    "detail": {
        "ModelPackageGroupName": ["FraudDetectionModelGroup"],
        "ModelApprovalStatus": ["Approved"]
    }
}
# Target: Lambda function gọi SageMaker deployment pipeline hoặc update endpoint
```

### Quy Trình Phê Duyệt Đa Cấp (Multi-stage Approval)

```
Data Scientist  →  ML Lead Review  →  Risk/Compliance  →  Production Deploy
(Tạo Model)       (Kiểm Tra Kỹ     (Kiểm Tra Tuân Thủ)   (Triển Khai)
                   Thuật)
     │                  │                   │
Submit for          Approve/Reject      Final Approve
Review              with comments       for Prod
```

```python
# Pattern: Dùng model package tags để track multi-stage approval
sm_client.add_tags(
    ResourceArn=model_package_arn,
    Tags=[
        {"Key": "ml-lead-review", "Value": "approved-2024-01-15"},
        {"Key": "compliance-review", "Value": "pending"},
    ],
)
```

---

## Model Metadata và Lineage {#metadata-và-lineage}

### Model Card (Thẻ Mô Hình)

Model Card là tài liệu toàn diện về mô hình — bắt buộc cho Responsible AI (AI Có Trách Nhiệm).

```python
from sagemaker.model_card import (
    ModelCard,
    ModelOverview,
    IntendedUses,
    BusinessDetails,
    TrainingDetails,
    EvaluationDetails,
    AdditionalInformation,
)

model_card = ModelCard(
    name="FraudDetectionModelCard",
    status="Draft",
    model_overview=ModelOverview(
        model_description=(
            "Mô hình XGBoost phân loại gian lận giao dịch thẻ tín dụng. "
            "Train trên dữ liệu 2023 Q4, gồm 284,807 giao dịch."
        ),
        model_artifact=[model_artifact_s3_uri],
        algorithm_type="XGBoost",
        model_creator="MLPlatform Team",
    ),
    intended_uses=IntendedUses(
        purpose_of_model="Phát hiện giao dịch gian lận theo thời gian thực",
        intended_uses="Áp dụng cho tất cả giao dịch thẻ tín dụng > 100 USD",
        factors_affecting_model_efficiency=(
            "Hiệu suất giảm với merchant categories chưa có trong training data."
        ),
        risk_rating="High",              # Mức độ rủi ro: Low / Medium / High
        explanations_for_risk_rating=(
            "Quyết định từ chối giao dịch ảnh hưởng trực tiếp đến khách hàng. "
            "Cần monitoring liên tục và quy trình appeal (khiếu nại) rõ ràng."
        ),
    ),
    training_details=TrainingDetails(
        training_observations=(
            "Train với weighted loss để xử lý class imbalance (mất cân bằng lớp): "
            "99.83% giao dịch hợp lệ, 0.17% gian lận."
        ),
    ),
    evaluation_details=[
        EvaluationDetails(
            name="Hold-out Test Set Evaluation",
            evaluation_observation=(
                "AUC: 0.923 | Precision: 0.891 | Recall: 0.876 | F1: 0.883"
            ),
        )
    ],
    additional_information=AdditionalInformation(
        ethical_considerations=(
            "Model được test cho bias theo nhóm tuổi, khu vực địa lý. "
            "Không phát hiện disparate impact (Tác Động Không Công Bằng) đáng kể."
        ),
        caveats_and_recommendations=(
            "Retrain mỗi quý hoặc khi fraud rate thay đổi > 20%."
        ),
    ),
)

model_card.create()
print(f"Model Card ARN: {model_card.arn}")
```

### Truy Vết Lineage (Lineage Tracking)

SageMaker tự động ghi lại lineage cho các artifacts trong SageMaker Pipelines:

```python
from sagemaker.lineage.context import Context
from sagemaker.lineage.artifact import Artifact
from sagemaker.lineage.association import Association
from sagemaker.lineage.query import LineageQuery, LineageFilter, LineageEntityEnum

# Truy vết ngược: Model này được tạo từ Training Job nào?
lineage_query = LineageQuery(sagemaker_session=session)

# Tìm tất cả artifacts liên quan đến model package này
associations = lineage_query.query(
    start_arns=[model_package_arn],
    direction="Ascendants",               # Ngược về phía trước (nguồn gốc)
    include_edges=True,
    filters=LineageFilter(
        entities=[
            LineageEntityEnum.ARTIFACT,   # Training data, processed data
            LineageEntityEnum.CONTEXT,    # Training job context
            LineageEntityEnum.ACTION,     # Model registration action
        ]
    ),
)

for association in associations.edges:
    print(f"Source: {association.source_arn}")
    print(f"  ─→ Destination: {association.destination_arn}")
    print(f"     Type: {association.association_type}")
```

### Xem Tất Cả Versions Của Một Model Group

```python
response = sm_client.list_model_packages(
    ModelPackageGroupName=model_package_group_name,
    SortBy="CreationTime",
    SortOrder="Descending",     # Mới nhất trước
)

for pkg in response["ModelPackageSummaryList"]:
    print(f"Version: {pkg['ModelPackageVersion']}")
    print(f"  ARN: {pkg['ModelPackageArn']}")
    print(f"  Status: {pkg['ModelApprovalStatus']}")
    print(f"  Created: {pkg['CreationTime'].strftime('%Y-%m-%d %H:%M')}")
    print()

# Output ví dụ:
# Version: 5
#   ARN: arn:aws:sagemaker:us-east-1:123456:model-package/frauddetection/5
#   Status: Approved
#   Created: 2024-01-15 09:30
#
# Version: 4
#   ARN: arn:aws:sagemaker:us-east-1:123456:model-package/frauddetection/4
#   Status: Rejected
#   Created: 2024-01-10 14:22
```

---

## Deploy Từ Registry {#deploy-từ-registry}

### Deploy Model Package Đã Approved

```python
from sagemaker import ModelPackage

# Lấy model package mới nhất đã approved
approved_packages = sm_client.list_model_packages(
    ModelPackageGroupName=model_package_group_name,
    ModelApprovalStatus="Approved",
    SortBy="CreationTime",
    SortOrder="Descending",
)

latest_approved_arn = approved_packages["ModelPackageSummaryList"][0]["ModelPackageArn"]

# Deploy lên endpoint
model = ModelPackage(
    role=role,
    model_package_arn=latest_approved_arn,
    sagemaker_session=session,
)

predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.m5.large",
    endpoint_name="fraud-detection-prod",
    data_capture_config=DataCaptureConfig(
        enable_capture=True,
        sampling_percentage=100,
        destination_s3_uri="s3://my-bucket/data-capture/fraud-detection/",
    ),
)
```

### Chiến Lược Triển Khai An Toàn

```python
# Blue/Green Deployment (Triển Khai Xanh/Xanh): Không downtime
sm_client.create_endpoint_config(
    EndpointConfigName="fraud-detection-green-config",
    ProductionVariants=[
        {
            "VariantName": "GreenVariant",
            "ModelName": new_model_name,
            "InitialInstanceCount": 1,
            "InstanceType": "ml.m5.large",
            "InitialVariantWeight": 0,    # Bắt đầu với 0% traffic
        }
    ],
)

# Cập nhật endpoint: thêm Green variant với 0% traffic
sm_client.update_endpoint(
    EndpointName="fraud-detection-prod",
    EndpointConfigName="fraud-detection-green-config",
    DeploymentConfig={
        "BlueGreenUpdatePolicy": {
            "TrafficRoutingConfiguration": {
                "Type": "LINEAR",                        # Tăng dần tuyến tính
                "LinearStepSize": {
                    "Value": 10,
                    "Type": "CAPACITY_PERCENT",          # 10% mỗi bước
                },
                "WaitIntervalInSeconds": 300,           # Chờ 5 phút giữa mỗi bước
            },
            "TerminationWaitInSeconds": 600,            # Chờ 10 phút trước khi xóa Blue
        }
    },
)
```

---

## Tích Hợp Với CI/CD {#tích-hợp-cicd}

### Quy Trình Hoàn Chỉnh

```
Git Push (code/config thay đổi)
      │
      ▼
CodePipeline / GitHub Actions
      │
      ▼
SageMaker Pipeline chạy (Train → Evaluate → Register)
      │
      ▼
Model Package tạo với status = PendingManualApproval
      │
      ▼
Notification (SNS/Email): "Model v5 sẵn sàng để review"
      │
      ▼
ML Engineer review metrics trên SageMaker Studio
      │
      ├── Approve → EventBridge → Lambda → Deploy Endpoint
      │
      └── Reject  → Comment lý do → Thông báo team
```

### Lambda Tự Động Deploy Khi Model Approved

```python
import boto3
import json

def lambda_handler(event, context):
    """
    Lambda function được trigger bởi EventBridge khi model approved.
    Tự động deploy model mới lên staging endpoint.
    """
    detail = event["detail"]
    model_package_arn = detail["ModelPackageArn"]
    model_package_group = detail["ModelPackageGroupName"]

    if detail["ModelApprovalStatus"] != "Approved":
        return {"statusCode": 200, "body": "Not an approval event, skipping"}

    sm_client = boto3.client("sagemaker")

    # Tạo endpoint config mới từ model package
    endpoint_config_name = f"{model_package_group}-staging-config-{context.aws_request_id[:8]}"

    sm_client.create_endpoint_config(
        EndpointConfigName=endpoint_config_name,
        ProductionVariants=[{
            "VariantName": "AllTraffic",
            "ModelName": _create_model_from_package(sm_client, model_package_arn),
            "InitialInstanceCount": 1,
            "InstanceType": "ml.m5.large",
            "InitialVariantWeight": 1,
        }],
    )

    # Cập nhật staging endpoint
    try:
        sm_client.update_endpoint(
            EndpointName=f"{model_package_group}-staging",
            EndpointConfigName=endpoint_config_name,
        )
        print(f"Updated staging endpoint with new model {model_package_arn}")
    except sm_client.exceptions.ResourceNotFound:
        sm_client.create_endpoint(
            EndpointName=f"{model_package_group}-staging",
            EndpointConfigName=endpoint_config_name,
        )
        print(f"Created new staging endpoint")

    return {"statusCode": 200, "body": f"Deployed {model_package_arn} to staging"}
```

---

## Câu Hỏi Phỏng Vấn {#phỏng-vấn}

**H: Model Registry giải quyết vấn đề gì trong ML production?**

> Model Registry giải quyết **governance** (quản trị) và **version control** (kiểm soát phiên bản) cho ML models:
> 1. **Traceability** (Truy Xuất Nguồn Gốc): Biết chính xác model nào đang chạy trong production, được train từ data nào, code nào
> 2. **Approval gate** (Cổng Phê Duyệt): Ngăn model chưa được review vào production — giảm rủi ro
> 3. **Rollback** (Quay Lại Phiên Bản Cũ): Nhanh chóng deploy lại version trước nếu model mới có vấn đề
> 4. **Compliance** (Tuân Thủ): Lưu metadata, Model Card cho audit requirements — đặc biệt quan trọng trong tài chính, y tế

**H: Sự khác biệt giữa Model Package và Model Package Group?**

> - **Model Package Group** (Nhóm Gói Mô Hình): Là container logic cho một "model identity" — ví dụ "FraudDetectionModel". Chứa nhiều versions theo thời gian.
> - **Model Package** (Gói Mô Hình): Một version cụ thể trong group — ví dụ "FraudDetectionModel v3". Chứa artifacts, metrics, approval status, metadata cho version đó.
>
> Tương tự như Git Repository (Group) vs Git Commit (Package).

**H: Khi nào nên dùng PendingManualApproval vs Approved ngay?**

> - **PendingManualApproval**: Dùng cho production models — luôn cần human review để đảm bảo model đạt chất lượng, không có bias không mong muốn, và đội ML hiểu tại sao model lại cho kết quả như vậy
> - **Approved ngay**: Dùng cho staging/dev models khi đang thử nghiệm nhanh, hoặc trong automated testing pipeline nơi tests đã đủ nghiêm ngặt thay thế human review

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
