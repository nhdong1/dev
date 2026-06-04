# Responsible AI — AI Có Trách Nhiệm trên AWS

> Hướng dẫn triển khai AI Có Trách Nhiệm (Responsible AI) theo các nguyên tắc của AWS: Fairness (Công Bằng), Explainability (Khả Năng Giải Thích), Privacy (Quyền Riêng Tư), Safety (An Toàn), Controllability (Khả Năng Kiểm Soát), Veracity & Robustness (Tính Trung Thực & Bền Vững), và Governance (Quản Trị)

## 📋 Mục Lục

1. [Tổng Quan Responsible AI](#tổng-quan-responsible-ai)
2. [AWS AI Service Cards](#aws-ai-service-cards)
3. [Fairness — Công Bằng](#fairness--công-bằng)
4. [Explainability — Khả Năng Giải Thích](#explainability--khả-năng-giải-thích)
5. [Privacy — Quyền Riêng Tư](#privacy--quyền-riêng-tư)
6. [Safety & Robustness — An Toàn & Bền Vững](#safety--robustness--an-toàn--bền-vững)
7. [Governance & Controllability — Quản Trị & Kiểm Soát](#governance--controllability--quản-trị--kiểm-soát)
8. [Amazon Bedrock Guardrails](#amazon-bedrock-guardrails)
9. [SageMaker Clarify cho Responsible AI](#sagemaker-clarify-cho-responsible-ai)
10. [Responsible AI Framework Thực Tế](#responsible-ai-framework-thực-tế)
11. [Checklist Responsible AI](#checklist-responsible-ai)

---

## Tổng Quan Responsible AI

### Tại Sao Responsible AI Quan Trọng?

Hệ thống AI có thể gây hại thực sự nếu không được thiết kế và triển khai cẩn thận:

```
Ví dụ thực tế về AI bias (thiên kiến AI):
- Hệ thống tuyển dụng từ chối ứng viên nữ vì model được train trên data lịch sử thiên về nam giới
- Mô hình tín dụng từ chối vay vì mã bưu chính (proxy cho chủng tộc/thu nhập)
- Nhận diện khuôn mặt với độ chính xác thấp hơn cho người da đen và phụ nữ
- LLM tạo ra thông tin sai (hallucination — ảo giác AI) được trình bày như sự thật
```

**Responsible AI không phải là luxury — đây là yêu cầu bắt buộc** cho production AI systems, đặc biệt trong các lĩnh vực: tài chính, y tế, tuyển dụng, pháp lý.

### AWS Responsible AI Principles — 8 Nguyên Tắc

AWS cam kết phát triển AI theo 8 nguyên tắc:

| Nguyên Tắc | Định Nghĩa | Công Cụ AWS |
|------------|------------|-------------|
| **Fairness** (Công Bằng) | Model đối xử công bằng với mọi nhóm dân số | SageMaker Clarify |
| **Explainability** (Khả Năng Giải Thích) | Có thể giải thích tại sao model đưa ra quyết định | SageMaker Clarify + SHAP |
| **Privacy & Security** (Riêng Tư & Bảo Mật) | Bảo vệ dữ liệu cá nhân trong ML lifecycle | Macie, Comprehend PII, KMS |
| **Safety** (An Toàn) | AI không tạo ra output gây hại | Bedrock Guardrails |
| **Controllability** (Khả Năng Kiểm Soát) | Con người có thể giám sát và kiểm soát AI | SageMaker Model Monitor |
| **Veracity & Robustness** (Trung Thực & Bền Vững) | AI hoạt động ổn định, không bị adversarial attacks | Model Monitor, A/B Testing |
| **Transparency** (Minh Bạch) | Rõ ràng về cách thức và giới hạn của AI | AI Service Cards, Model Cards |
| **Governance** (Quản Trị) | Quy trình kiểm soát, audit và trách nhiệm | SageMaker Model Registry, CloudTrail |

---

## AWS AI Service Cards

**AI Service Cards** (Thẻ Thông Tin Dịch Vụ AI) là tài liệu minh bạch mô tả cách mỗi AWS AI service hoạt động, giới hạn của nó, và các use cases không nên dùng.

```
Amazon Rekognition Face Analysis — AI Service Card highlights:
- Intended use: Identity verification, photo organization
- NOT intended use: Sole basis for law enforcement decisions
- Limitations: Lower accuracy for darker skin tones, accuracy varies by age
- Fairness metrics: Available in documentation

Amazon Comprehend — AI Service Card highlights:
- Intended use: Text analysis, sentiment, NLP tasks
- Limitations: Better accuracy for English vs other languages
- Privacy: Text sent to API for analysis — consider data residency
```

Truy cập: https://aws.amazon.com/machine-learning/responsible-machine-learning/

---

## Fairness — Công Bằng

### Khái Niệm Bias trong ML

**Bias** (Thiên Kiến) trong ML xuất hiện ở nhiều giai đoạn:

```
Sources of Bias (Nguồn Thiên Kiến):
├── Historical bias          Data phản ánh bất bình đẳng lịch sử
├── Representation bias      Nhóm thiểu số bị underrepresented trong data
├── Measurement bias         Cách đo lường không nhất quán giữa các nhóm
├── Aggregation bias         Mô hình một model cho mọi nhóm không phù hợp
└── Evaluation bias          Benchmark dataset không đại diện cho target population
```

### Fairness Metrics — Chỉ Số Đo Công Bằng

```python
# SageMaker Clarify tính toán các fairness metrics sau:

# 1. Class Imbalance (CI) — Mất Cân Bằng Lớp
# = |n_privileged - n_unprivileged| / (n_privileged + n_unprivileged)
# CI = 0: Hoàn toàn cân bằng
# CI = ±1: Hoàn toàn mất cân bằng

# 2. Difference in Positive Proportions in Labels (DPL)
# = P(y=1 | a=0) - P(y=1 | a=1)  (a = sensitive attribute như giới tính)
# DPL = 0: Công bằng
# |DPL| > 0.1: Đáng lo ngại

# 3. Disparate Impact (DI) — Tác Động Chênh Lệch
# = P(y=1 | a=0) / P(y=1 | a=1)
# DI = 1: Công bằng
# DI < 0.8: Vi phạm "80% rule" của EEOC (US employment law)

# 4. Equal Opportunity Difference (EOD)
# = TPR(a=0) - TPR(a=1)  (TPR = True Positive Rate)
# Đo xem model có cơ hội bằng nhau phát hiện positive cases giữa các nhóm không
```

### Chạy Pre-training Bias Analysis

```python
import sagemaker
from sagemaker.clarify import (
    SageMakerClarifyProcessor,
    BiasConfig,
    DataConfig,
)

clarify_processor = SageMakerClarifyProcessor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.xlarge",
    sagemaker_session=sagemaker.Session(),
)

# Cấu hình dữ liệu
data_config = DataConfig(
    s3_data_input_path="s3://my-bucket/training-data.csv",
    s3_output_path="s3://my-bucket/clarify-output/",
    label="loan_approved",           # Cột nhãn (target)
    headers=["age", "gender", "income", "loan_approved"],
    dataset_type="text/csv",
)

# Cấu hình bias — chỉ định sensitive attribute (thuộc tính nhạy cảm)
bias_config = BiasConfig(
    label_values_or_threshold=[1],   # Positive label = được duyệt vay
    facet_name="gender",             # Sensitive attribute cần kiểm tra
    facet_values_or_threshold=["female"],  # Nhóm cần bảo vệ
)

# Chạy pre-training bias analysis
clarify_processor.run_pre_training_bias(
    data_config=data_config,
    bias_config=bias_config,
    methods="all",   # Tính tất cả metrics: CI, DPL, KL, KS, CDDL, v.v.
)
```

### Post-training Bias Analysis — Sau Khi Huấn Luyện

```python
from sagemaker.clarify import ModelConfig, ModelPredictedLabelConfig

# Cấu hình model đã trained
model_config = ModelConfig(
    model_name="my-loan-approval-model",
    instance_type="ml.m5.xlarge",
    instance_count=1,
    accept_type="text/csv",
    content_type="text/csv",
)

# Cấu hình cách đọc prediction từ model output
predictions_config = ModelPredictedLabelConfig(
    probability_threshold=0.5,  # Ngưỡng để phân loại positive
)

# Chạy post-training bias analysis
clarify_processor.run_post_training_bias(
    data_config=data_config,
    data_bias_config=bias_config,
    model_config=model_config,
    model_predicted_label_config=predictions_config,
    methods="all",  # DPPL, DI, DCA, DCR, RD, DAR, DRR, AD, CDDPL, TE, FT
)
```

---

## Explainability — Khả Năng Giải Thích

### Tại Sao Cần Explainability?

```
Yêu cầu pháp lý:
- EU AI Act: Hệ thống AI high-risk phải giải thích được quyết định
- GDPR Article 22: Quyền giải thích khi có automated decision
- US Fair Credit Reporting Act: Phải giải thích lý do từ chối tín dụng

Yêu cầu kinh doanh:
- Bác sĩ cần biết tại sao AI chẩn đoán bệnh X, không chỉ xem kết quả
- Quản lý rủi ro cần audit trail cho quyết định tự động
- Debug model: Hiểu feature nào quan trọng để cải thiện model
```

### SHAP — SHapley Additive exPlanations

**SHAP** (SHapley Additive exPlanations — Giải Thích Cộng Gộp Shapley) là phương pháp giải thích model dựa trên lý thuyết game Shapley values từ toán học:

- **SHAP value dương** (+): Feature này đẩy prediction lên cao hơn baseline
- **SHAP value âm** (−): Feature này kéo prediction xuống thấp hơn baseline
- **Magnitude**: Giá trị tuyệt đối càng lớn, feature càng quan trọng

```python
from sagemaker.clarify import (
    SageMakerClarifyProcessor,
    DataConfig,
    ModelConfig,
    SHAPConfig,
)

# Cấu hình SHAP analysis
shap_config = SHAPConfig(
    baseline=[
        # Baseline = giá trị "trung bình" hoặc "thiếu" cho mỗi feature
        # Dùng mean hoặc median của training data
        [35, 0, 50000]  # [age, gender(0=female), income]
    ],
    num_samples=100,    # Số samples để ước tính SHAP values
    agg_method="mean_abs",  # Cách tổng hợp SHAP: mean_abs, median, mean_sq
    use_logit=False,
)

# Chạy explainability analysis
clarify_processor.run_explainability(
    data_config=data_config,
    model_config=model_config,
    explainability_config=shap_config,
)

# Kết quả: global_explanations.json và local_explanations.json
# Global: Feature importance tổng thể cho toàn bộ dataset
# Local: Giải thích cho từng prediction cụ thể
```

**Đọc kết quả SHAP:**

```python
import json
import matplotlib.pyplot as plt

# Đọc global explanations
with open("global_explanations.json") as f:
    global_exp = json.load(f)

# mean_abs_shap = giá trị tuyệt đối trung bình của SHAP values
features = list(global_exp["explanations"]["kernel_shap"].keys())
shap_values = [
    global_exp["explanations"]["kernel_shap"][f]["mean_abs_shap"]
    for f in features
]

# Vẽ bar chart feature importance
plt.barh(features, shap_values)
plt.xlabel("Mean |SHAP value|")
plt.title("Feature Importance theo SHAP")
plt.tight_layout()
plt.savefig("shap_feature_importance.png")
```

### Model Cards — Thẻ Thông Tin Mô Hình

**Model Cards** (Thẻ Thông Tin Mô Hình) là tài liệu chuẩn hóa mô tả một ML model: mục đích, hiệu suất, giới hạn, và ethical considerations (Cân Nhắc Đạo Đức):

```python
import sagemaker
from sagemaker.model_card import (
    ModelCard,
    ModelOverview,
    TrainingDetails,
    EvaluationDetails,
    IntendedUses,
)

# Tạo Model Card cho model loan approval
model_card = ModelCard(
    name="loan-approval-model-v2",
    status="Draft",
    model_overview=ModelOverview(
        model_description="Model phê duyệt khoản vay dựa trên thông tin tài chính",
        model_id="loan-approval-xgboost-v2",
        model_version="2.0",
        problem_type="BinaryClassification",
        algorithm_type="XGBoost",
    ),
    intended_uses=IntendedUses(
        purpose_of_model="Hỗ trợ quyết định phê duyệt vay cá nhân",
        intended_uses="Screening ban đầu, không phải quyết định cuối cùng",
        factors_affecting_model_efficiency="Tỷ lệ vỡ nợ thay đổi theo chu kỳ kinh tế",
        risk_rating="High",           # High-risk vì ảnh hưởng tài chính cá nhân
        explanations_for_risk_rating="Quyết định tín dụng ảnh hưởng trực tiếp đến cuộc sống",
    ),
    sagemaker_session=sagemaker.Session(),
)

model_card.create()
```

---

## Privacy — Quyền Riêng Tư

### Privacy trong ML Lifecycle

```
Giai Đoạn Thu Thập Data:
├── Chỉ thu thập data thực sự cần thiết (data minimization)
├── Có consent (sự đồng ý) từ người dùng
└── Khai báo mục đích sử dụng rõ ràng

Giai Đoạn Training:
├── Anonymize/pseudonymize PII trước khi training
├── Dùng differential privacy (riêng tư vi phân) nếu cần
└── Không log raw training data chứa PII

Giai Đoạn Inference:
├── Không lưu trữ user queries chứa PII lâu dài
├── Audit log cho các requests nhạy cảm
└── User có thể request xóa data (GDPR right to erasure)

Giai Đoạn Deployment:
├── Model artifact không chứa raw training data
├── Cẩn thận với model inversion attacks
└── Monitor cho membership inference attacks
```

### Differential Privacy — Riêng Tư Vi Phân

**Differential Privacy** (DP) là kỹ thuật toán học thêm noise (nhiễu) có kiểm soát vào model training để ngăn chặn việc suy ra thông tin về individual trong training set:

```python
# Sử dụng TensorFlow Privacy với SageMaker
# (Ví dụ conceptual — không phải production code đầy đủ)

import tensorflow as tf
import tensorflow_privacy

optimizer = tensorflow_privacy.DPKerasSGDOptimizer(
    l2_norm_clip=1.0,       # Giới hạn độ lớn của gradient mỗi sample
    noise_multiplier=1.1,   # Lượng noise thêm vào gradient
    num_microbatches=256,   # Số microbatches per batch
    learning_rate=0.015
)

model.compile(
    optimizer=optimizer,
    loss=tf.keras.losses.CategoricalCrossentropy(),
    metrics=["accuracy"]
)

# Epsilon (ε) = privacy budget: nhỏ hơn = riêng tư hơn nhưng accuracy giảm
# ε < 1: Rất riêng tư, thường dùng cho sensitive data
# ε < 10: Riêng tư hợp lý
# ε > 10: Riêng tư yếu
```

---

## Safety & Robustness — An Toàn & Bền Vững

### Adversarial Robustness — Khả Năng Chống Tấn Công

**Adversarial attacks** (Tấn Công Đối Nghịch): Thay đổi nhỏ input không thể nhận thấy bằng mắt người nhưng khiến model đưa ra dự đoán sai hoàn toàn.

```
Ví dụ adversarial attack:
- Thêm pixel noise vào ảnh "mèo" → Model nhận diện thành "tên lửa"
- Thêm từ đặc biệt vào email spam → Qua được spam filter
- Thay đổi nhỏ trong loan application data → Từ reject thành approve

Cách phòng chống:
├── Adversarial training: Thêm adversarial examples vào training data
├── Input validation: Validate và normalize inputs trước khi predict
├── Model ensembling: Dùng nhiều models, tấn công một model khó ảnh hưởng tất cả
└── SageMaker Model Monitor: Phát hiện unusual input patterns
```

### Hallucination trong LLM — Ảo Giác AI

**Hallucination** (Ảo Giác AI) xảy ra khi LLM (Large Language Model — Mô Hình Ngôn Ngữ Lớn) tạo ra thông tin nghe có vẻ tự tin nhưng sai sự thật.

```
Chiến lược giảm hallucination:
├── RAG (Retrieval-Augmented Generation): Ground model responses vào facts từ knowledge base
├── Temperature = 0: Giảm randomness, model đưa ra deterministic output
├── System prompt instructions: Hướng dẫn model thừa nhận khi không biết
├── Bedrock Guardrails: Block hoặc filter toxic/incorrect content
└── Human-in-the-loop: Yêu cầu human review cho high-stakes decisions
```

---

## Governance & Controllability — Quản Trị & Kiểm Soát

### Human-in-the-Loop — Con Người Trong Vòng Kiểm Soát

**HITL** (Human-in-the-Loop — Con Người Trong Vòng Kiểm Soát): Thiết kế hệ thống để con người có thể can thiệp, override, hoặc audit quyết định AI.

```python
import boto3

a2i = boto3.client("sagemaker-a2i-runtime")

# Amazon Augmented AI (A2I) — Tích hợp Human Review vào ML pipeline
# Khi model confidence thấp hoặc với random sample → Gửi đến human reviewer

def invoke_with_human_review(endpoint_name, input_data, confidence_threshold=0.85):
    runtime = boto3.client("sagemaker-runtime")
    sm_a2i = boto3.client("sagemaker-a2i-runtime")

    # Gọi model inference
    response = runtime.invoke_endpoint(
        EndpointName=endpoint_name,
        ContentType="application/json",
        Body=json.dumps(input_data),
    )
    prediction = json.loads(response["Body"].read())
    confidence = prediction["confidence"]

    if confidence < confidence_threshold:
        # Confidence thấp → Gửi đến human reviewer
        a2i_response = sm_a2i.start_human_loop(
            HumanLoopName=f"review-{uuid.uuid4()}",
            FlowDefinitionArn=flow_definition_arn,
            HumanLoopInput={
                "InputContent": json.dumps({
                    "input": input_data,
                    "model_prediction": prediction,
                    "confidence": confidence,
                })
            },
        )
        return {"status": "pending_human_review", "loop_arn": a2i_response["HumanLoopArn"]}
    else:
        return {"status": "auto_approved", "prediction": prediction}
```

### SageMaker Model Registry cho Governance

```python
sm = boto3.client("sagemaker")

# Approval workflow: Model phải được approve trước khi deploy production
# Chỉ model có status "Approved" mới được deploy production

# Senior ML Engineer/Data Science Lead review và approve
sm.update_model_package(
    ModelPackageArn=model_package_arn,
    ModelApprovalStatus="Approved",       # hoặc "Rejected"
    ApprovalDescription="Đã review bias metrics, SHAP explanations. Model đạt tiêu chuẩn Responsible AI. DPL = 0.02 (acceptable). Feature importance hợp lý.",
)
```

---

## Amazon Bedrock Guardrails

### Guardrails là gì?

**Amazon Bedrock Guardrails** (Rào Cản Bedrock) là lớp kiểm soát an toàn đặt trước và sau Foundation Model để:
- Block toxic content (Nội Dung Độc Hại)
- Prevent prompt injection (Tấn Công Tiêm Nhiễm Prompt)
- Enforce topic restrictions (Hạn Chế Chủ Đề)
- Redact PII trong input/output
- Grounding check (Kiểm Tra Tính Có Căn Cứ) — phát hiện hallucination

```python
import boto3

bedrock = boto3.client("bedrock")

# Tạo Guardrail cho enterprise chatbot
response = bedrock.create_guardrail(
    name="enterprise-chatbot-guardrail",
    description="Guardrail cho customer service chatbot",

    # Chặn nội dung có hại
    contentPolicyConfig={
        "filtersConfig": [
            {"type": "SEXUAL", "inputStrength": "HIGH", "outputStrength": "HIGH"},
            {"type": "VIOLENCE", "inputStrength": "MEDIUM", "outputStrength": "HIGH"},
            {"type": "HATE", "inputStrength": "HIGH", "outputStrength": "HIGH"},
            {"type": "INSULTS", "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
            {"type": "MISCONDUCT", "inputStrength": "MEDIUM", "outputStrength": "HIGH"},
            {"type": "PROMPT_ATTACK", "inputStrength": "HIGH", "outputStrength": "NONE"},
        ]
    },

    # Chặn các chủ đề nhạy cảm với business
    topicPolicyConfig={
        "topicsConfig": [
            {
                "name": "Tư Vấn Pháp Lý",
                "definition": "Yêu cầu tư vấn pháp lý hoặc đưa ra ý kiến pháp luật",
                "examples": [
                    "Tôi có thể kiện công ty không?",
                    "Hợp đồng này có hợp pháp không?",
                ],
                "type": "DENY",
            },
            {
                "name": "Thông Tin Sức Khỏe",
                "definition": "Tư vấn y tế hoặc chẩn đoán bệnh",
                "examples": [
                    "Tôi nên uống thuốc gì?",
                    "Triệu chứng này có phải ung thư không?",
                ],
                "type": "DENY",
            },
        ]
    },

    # Redact PII trong input và output
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": "EMAIL", "action": "ANONYMIZE"},
            {"type": "PHONE", "action": "ANONYMIZE"},
            {"type": "CREDIT_DEBIT_CARD_NUMBER", "action": "BLOCK"},
            {"type": "AWS_ACCESS_KEY", "action": "BLOCK"},
        ],
        "regexesConfig": [
            {
                "name": "Vietnam CMND/CCCD",
                "description": "Số CMND hoặc CCCD Việt Nam",
                "pattern": r"\b\d{9}|\d{12}\b",
                "action": "ANONYMIZE",
            }
        ],
    },

    # Grounding: Câu trả lời phải dựa trên context được cung cấp
    groundingPolicyConfig={
        "filtersConfig": [
            {"type": "GROUNDING", "threshold": 0.7},  # Ngưỡng grounding score
            {"type": "RELEVANCE", "threshold": 0.7},  # Ngưỡng relevance score
        ]
    },

    blockedInputMessaging="Xin lỗi, tôi không thể trả lời câu hỏi này.",
    blockedOutputsMessaging="Câu trả lời không phù hợp đã bị chặn.",
)

guardrail_id = response["guardrailId"]

# Sử dụng Guardrail khi invoke model
bedrock_runtime = boto3.client("bedrock-runtime")
response = bedrock_runtime.invoke_model(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    guardrailIdentifier=guardrail_id,
    guardrailVersion="DRAFT",
    body=json.dumps({
        "messages": [{"role": "user", "content": user_message}],
        "max_tokens": 1000,
    }),
)
```

---

## SageMaker Clarify cho Responsible AI

### Tổng Hợp Các Capabilities

```
SageMaker Clarify:
├── Pre-training Bias Detection    Bias trong training data trước khi train
├── Post-training Bias Detection   Bias trong model predictions sau khi train
├── SHAP Explainability            Feature importance toàn cục và cục bộ
├── Partial Dependence Plots       Mối quan hệ giữa feature và prediction
└── Model Monitor Integration      Phát hiện bias drift trong production
```

### Tích Hợp Clarify vào SageMaker Pipeline

```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep
from sagemaker.clarify import SageMakerClarifyProcessor

clarify_processor = SageMakerClarifyProcessor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.xlarge",
    sagemaker_session=pipeline_session,
)

# Bước Clarify trong pipeline: chạy sau training, trước deployment
clarify_step = clarify_processor.run_bias(
    data_config=data_config,
    bias_config=bias_config,
    model_config=model_config,
    model_predicted_label_config=predictions_config,
    pre_training_methods="all",
    post_training_methods="all",
    job_name="clarify-bias-check",
    wait=False,
    logs=False,
)

# Pipeline sẽ fail nếu bias vượt ngưỡng — cần human review trước khi deploy
```

---

## Responsible AI Framework Thực Tế

### Quy Trình Triển Khai Responsible AI

```
Giai Đoạn 1: Problem Framing (Xác Định Bài Toán)
├── Xác định potential harms (tác hại tiềm năng)
├── Xác định sensitive attributes (giới tính, chủng tộc, tuổi, v.v.)
├── Định nghĩa fairness metrics phù hợp với context
└── Thiết lập threshold chấp nhận được cho bias

Giai Đoạn 2: Data Preparation (Chuẩn Bị Dữ Liệu)
├── Audit training data cho class imbalance
├── Kiểm tra representation của các nhóm dân số
├── Redact PII không cần thiết
└── Document data sources và potential biases

Giai Đoạn 3: Training & Evaluation
├── Chạy Pre-training Bias Analysis với Clarify
├── Train model
├── Chạy Post-training Bias Analysis
├── Generate SHAP explanations
└── Review Model Card

Giai Đoạn 4: Deployment Review
├── Human review của bias reports
├── Approval gate trong Model Registry
├── A/B testing với diverse user groups
└── Deploy với monitoring enabled

Giai Đoạn 5: Production Monitoring
├── Bias drift monitoring với Model Monitor
├── Regular fairness audits
├── User feedback mechanism
└── Incident response plan khi phát hiện bias
```

---

## Checklist Responsible AI

### Fairness

- [ ] **Xác định sensitive attributes** (giới tính, tuổi, vùng địa lý, v.v.) liên quan đến bài toán
- [ ] **Chạy Pre-training Bias Analysis** với Clarify — kiểm tra Class Imbalance và DPL
- [ ] **Chạy Post-training Bias Analysis** — kiểm tra Disparate Impact, Equal Opportunity
- [ ] **Thiết lập threshold** và reject model nếu bias vượt ngưỡng chấp nhận
- [ ] **Monitor bias drift** trong production với SageMaker Model Monitor

### Explainability

- [ ] **Generate SHAP explanations** cho model — global và local
- [ ] **Tạo Model Card** với performance metrics phân tách theo demographic groups
- [ ] **Giải thích được** tại sao model đưa ra quyết định cụ thể (cho auditors, regulators)
- [ ] **Feature importance** được review bởi domain expert (ví dụ: doctor review cho medical AI)

### Privacy

- [ ] **Scan training data** với Macie/Comprehend PII trước khi training
- [ ] **Redact/anonymize PII** không cần thiết trong training data
- [ ] **Cân nhắc Differential Privacy** cho highly sensitive data (medical, financial)
- [ ] **Data retention policy**: Không lưu training data lâu hơn cần thiết
- [ ] **Right to erasure**: Có quy trình xóa data của individual nếu yêu cầu

### Safety (đặc biệt cho GenAI)

- [ ] **Triển khai Bedrock Guardrails** cho LLM applications
- [ ] **Test adversarial inputs** và prompt injection attacks
- [ ] **Human-in-the-loop** cho high-stakes decisions
- [ ] **RAG** để giảm hallucination và ground responses vào facts
- [ ] **Rate limiting** và **abuse prevention** cho API

### Governance

- [ ] **Approval workflow** trong Model Registry trước khi deploy production
- [ ] **Audit trail** cho tất cả model deployment và updates
- [ ] **Incident response plan** khi phát hiện bias hoặc harmful outputs
- [ ] **Regular responsible AI review** (quarterly hoặc khi model thay đổi lớn)

---

## 📌 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Giải thích sự khác nhau giữa Pre-training Bias và Post-training Bias?**
> A: Pre-training Bias xuất hiện trong training data trước khi model được train — ví dụ: tập dữ liệu tuyển dụng có 80% nam giới (Class Imbalance). Post-training Bias là bias trong predictions của model đã train — ví dụ: model có Disparate Impact thấp hơn 0.8 với nhóm female (phụ nữ được approve ít hơn 80% so với nam giới). Cần kiểm tra cả hai: pre-training bias ảnh hưởng đến post-training bias nhưng không phải 1-1.

**Q: SHAP values là gì và tại sao quan trọng?**
> A: SHAP (SHapley Additive exPlanations) dùng lý thuyết game để phân bổ "công lao" của mỗi feature vào prediction. SHAP value của feature X cho prediction Y = mức đóng góp của X để đẩy prediction từ baseline lên Y. Quan trọng vì: (1) Giải thích được cho regulators và end-users; (2) Debug model — nếu một feature không nên ảnh hưởng (như race) mà có SHAP value cao, cần điều tra; (3) Feature selection để cải thiện model.

**Q: Bedrock Guardrails giải quyết vấn đề gì?**
> A: Guardrails là lớp safety giữa user và Foundation Model, giải quyết: (1) Toxic content — block hate speech, violence; (2) Topic restrictions — ngăn model tư vấn pháp lý/y tế ngoài phạm vi; (3) PII protection — redact số điện thoại, email trong input/output; (4) Prompt injection — ngăn user override system prompt; (5) Grounding — phát hiện hallucination bằng cách kiểm tra response có căn cứ vào context không.

**Q: Khi nào cần Human-in-the-Loop và làm thế nào implement trên AWS?**
> A: Cần HITL khi: model confidence thấp (< threshold), quyết định high-stakes (medical, legal, financial), random audit sample, user escalation request. Implement với Amazon Augmented AI (A2I): tạo Human Review Flow Definition, set trigger conditions (confidence < 0.85), route đến human reviewers qua Amazon Mechanical Turk, internal workforce hoặc private workforce.

---

**Liên Kết Liên Quan:**
- [SageMaker Clarify chi tiết](../09-mlops/4-sagemaker-clarify.md)
- [Model Monitor Bias Drift](../09-mlops/3-model-monitor.md)
- [Security & Compliance](./2-security-compliance.md)
- [Bedrock Agents & Guardrails](../03-bedrock/4-bedrock-agents.md)

**Cập Nhật Lần Cuối:** 2026-06-03
