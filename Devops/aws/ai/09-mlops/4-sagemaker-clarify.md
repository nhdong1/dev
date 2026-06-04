# SageMaker Clarify — Phát Hiện Thiên Kiến và Giải Thích AI

> **SageMaker Clarify** là công cụ giúp phát hiện **bias** (Thiên Kiến) trong dữ liệu và mô hình, đồng thời cung cấp **explainability** (Khả Năng Giải Thích) thông qua SHAP values (Shapley Additive Explanations — Giải Thích Đóng Góp Theo Shapley). Clarify là thành phần cốt lõi của **Responsible AI** (AI Có Trách Nhiệm) trên AWS.

---

## 📚 Mục Lục

1. [Tại Sao Cần Clarify](#tại-sao-cần)
2. [Bias Detection — Phát Hiện Thiên Kiến](#bias-detection)
3. [Pre-training Bias Analysis](#pre-training-bias)
4. [Post-training Bias Analysis](#post-training-bias)
5. [SHAP Explainability](#shap-explainability)
6. [Clarify Trong SageMaker Pipeline](#clarify-pipeline)
7. [Bias Metrics Chi Tiết](#bias-metrics)
8. [Câu Hỏi Phỏng Vấn](#phỏng-vấn)

---

## Tại Sao Cần Clarify {#tại-sao-cần}

### Vấn Đề Với ML "Black Box" (Hộp Đen)

```
Traditional Model:
  Input: [transaction_amount=500, age_group=young, merchant=online]
  Output: "FRAUD" (probability=0.89)
  Question: Tại sao? → ❌ Không biết

Với SageMaker Clarify:
  Input: [transaction_amount=500, age_group=young, merchant=online]
  Output: "FRAUD" (probability=0.89)
  SHAP Explanation:
    - transaction_amount: +0.35 (tăng khả năng fraud)
    - merchant=online:    +0.28 (tăng khả năng fraud)
    - age_group=young:    +0.12 (tăng khả năng fraud) ← Đây là bias không?
    - hour=2am:           +0.14 (tăng khả năng fraud)
```

### Ba Mục Đích Chính Của Clarify

1. **Pre-training Bias Analysis** (Phân Tích Thiên Kiến Trước Huấn Luyện):
   - Phát hiện bias trong dữ liệu training TRƯỚC khi train model
   - Ví dụ: training data có ít samples từ nhóm người cao tuổi → model sẽ kém chính xác với nhóm này

2. **Post-training Bias Analysis** (Phân Tích Thiên Kiến Sau Huấn Luyện):
   - Đánh giá xem model đã học được bias từ data hay thậm chí tạo ra bias mới
   - Ví dụ: model approve khoản vay ít hơn 25% cho phụ nữ dù có cùng credit score

3. **Explainability** (Khả Năng Giải Thích):
   - Giải thích TẠI SAO model đưa ra dự đoán cụ thể qua SHAP values
   - Hỗ trợ audit (kiểm toán), debug, và tin tưởng vào model

---

## Bias Detection — Phát Hiện Thiên Kiến {#bias-detection}

### Các Khái Niệm Bias

| Khái Niệm                                                  | Định Nghĩa                                                           |
| ---------------------------------------------------------- | -------------------------------------------------------------------- |
| **Facet** (Diện)                                           | Thuộc tính nhân khẩu học cần kiểm tra fairness (tuổi, giới tính, v.v.) |
| **Facet value** (Giá Trị Diện)                             | Nhóm cụ thể trong facet (ví dụ: facet=gender, value=female)         |
| **Label** (Nhãn)                                           | Kết quả của model (ví dụ: approved=1, rejected=0)                    |
| **Positive label value** (Giá Trị Nhãn Tích Cực)           | Kết quả được coi là có lợi (ví dụ: loan_approved=1)                  |
| **Reference group** (Nhóm Tham Chiếu)                     | Nhóm làm chuẩn so sánh (ví dụ: gender=male)                         |
| **Protected group** (Nhóm Được Bảo Vệ)                    | Nhóm cần kiểm tra không bị thiệt thòi (ví dụ: gender=female)        |

### Cấu Hình BiasConfig

```python
from sagemaker.clarify import BiasConfig

bias_config = BiasConfig(
    label_values_or_threshold=[1],      # Nhãn positive: 1 = loan approved (được duyệt)
    facet_name="gender",                # Kiểm tra fairness theo giới tính
    facet_values_or_threshold=[0],      # 0 = female (nhóm được bảo vệ)
    group_name="age_group",             # Phân tích thêm theo nhóm tuổi
)

# Có thể định nghĩa nhiều facets để kiểm tra
# BiasConfig chỉ hỗ trợ một facet mỗi lần chạy → chạy nhiều lần cho nhiều facets
```

---

## Pre-training Bias Analysis {#pre-training-bias}

### Phát Hiện Bias Trong Data Trước Khi Train

Pre-training analysis chỉ cần **dataset** — không cần model. Nó phân tích xem dữ liệu training có đại diện đều các nhóm không.

```python
from sagemaker.clarify import SageMakerClarifyProcessor, DataConfig, BiasConfig

# Tạo Clarify processor
clarify_processor = SageMakerClarifyProcessor(
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
    sagemaker_session=session,
)

# Cấu hình dữ liệu
data_config = DataConfig(
    s3_data_input_path="s3://my-bucket/training-data/train.csv",
    s3_output_path="s3://my-bucket/clarify-output/pre-training-bias/",
    label="loan_approved",              # Cột nhãn
    headers=[
        "age", "income", "credit_score",
        "gender", "loan_amount", "loan_approved",
    ],
    dataset_type="text/csv",
)

# Cấu hình bias
bias_config = BiasConfig(
    label_values_or_threshold=[1],      # loan_approved = 1 là positive
    facet_name="gender",
    facet_values_or_threshold=[0],      # 0 = female
)

# Chạy pre-training bias analysis
clarify_processor.run_pre_training_bias(
    data_config=data_config,
    bias_config=bias_config,
    methods="all",                      # Tính tất cả bias metrics
    wait=True,
    logs=True,
)
```

### Đọc Kết Quả Pre-training Bias

```json
// analysis.json — Ví dụ kết quả
{
  "pre_training_bias_metrics": {
    "facets": {
      "gender": {
        "0": {  // female group
          "metrics": [
            {
              "name": "CI",
              "description": "Class Imbalance (Mất Cân Bằng Lớp)",
              "value": 0.35    // Female chiếm ít hơn 35% so với male trong dataset
            },
            {
              "name": "DPL",
              "description": "Difference in Positive Proportions in Labels",
              "value": -0.12   // Female có 12% ít hơn tỷ lệ loan_approved=1
            },
            {
              "name": "JS",
              "description": "Jensen-Shannon Divergence",
              "value": 0.08    // Phân phối label khác nhau đáng kể giữa hai nhóm
            }
          ]
        }
      }
    }
  }
}
```

**Diễn Giải:**
- `CI = 0.35`: Dataset thiếu đại diện của nhóm female — chỉ chiếm 32.5% samples
- `DPL = -0.12`: Trong data, female được approve ít hơn male 12% — có thể là bias lịch sử
- `JS = 0.08`: Phân phối label giữa male/female khác nhau đáng kể

---

## Post-training Bias Analysis {#post-training-bias}

### Phát Hiện Bias Sau Khi Model Đã Được Train

Post-training analysis dùng **model predictions** để đánh giá model có đối xử công bằng không.

```python
from sagemaker.clarify import ModelConfig

# Cấu hình model để Clarify gọi predictions
model_config = ModelConfig(
    model_name=model_name,              # SageMaker model name
    instance_type="ml.m5.large",
    instance_count=1,
    accept_type="text/csv",
    content_type="text/csv",
)

# Cấu hình dữ liệu — lần này là validation set
data_config = DataConfig(
    s3_data_input_path="s3://my-bucket/validation/validation.csv",
    s3_output_path="s3://my-bucket/clarify-output/post-training-bias/",
    label="loan_approved",
    headers=[
        "age", "income", "credit_score",
        "gender", "loan_amount", "loan_approved",
    ],
    dataset_type="text/csv",
)

model_predicted_label_config = ModelPredictedLabelConfig(
    probability_threshold=0.5,          # Ngưỡng để classify là 1 (approved)
)

# Chạy post-training bias analysis
clarify_processor.run_post_training_bias(
    data_config=data_config,
    data_bias_config=bias_config,
    model_config=model_config,
    model_predicted_label_config=model_predicted_label_config,
    methods="all",
    wait=True,
)
```

### Kết Quả Post-training Bias — Ví Dụ

```json
{
  "post_training_bias_metrics": {
    "facets": {
      "gender": {
        "0": {  // female group predictions
          "metrics": [
            {
              "name": "DPPL",
              "description": "Difference in Positive Proportions in Predicted Labels",
              "value": -0.18   // Model approve female ÍT HƠN 18% so với male ← VẤN ĐỀ!
            },
            {
              "name": "DI",
              "description": "Disparate Impact (Tác Động Không Công Bằng)",
              "value": 0.72    // < 0.8 → CÓ THIÊN KIẾN ĐÁNG KỂ
            },
            {
              "name": "FPR",
              "description": "False Positive Rate (Tỷ Lệ Dương Tính Sai)",
              "value": 0.08    // Model từ chối đúng người vay female ít hơn male 8%
            },
            {
              "name": "FNR",
              "description": "False Negative Rate (Tỷ Lệ Âm Tính Sai)",
              "value": -0.15   // Model bỏ sót người vay tốt từ female nhiều hơn 15%
            }
          ]
        }
      }
    }
  }
}
```

**Kết Luận:** Model này CÓ bias đáng kể — cần điều tra nguyên nhân (từ data hay từ features) và remediate trước khi deploy.

---

## SHAP Explainability {#shap-explainability}

### SHAP Là Gì?

**SHAP** (SHapley Additive exPlanations — Giải Thích Đóng Góp Theo Shapley) là phương pháp giải thích đóng góp của từng feature vào một dự đoán cụ thể, dựa trên lý thuyết trò chơi (game theory).

```
Prediction = Base value + SHAP(feature_1) + SHAP(feature_2) + ... + SHAP(feature_n)
(Dự Đoán)   (Giá trị nền) (Đóng góp f1)    (Đóng góp f2)           (Đóng góp fn)

Ví dụ thực tế:
P(fraud) = 0.5 + 0.35 (amount=high) + 0.28 (merchant=online) - 0.14 (history=good)
         = 0.99 → Fraud detected
```

### Global Explainability vs Local Explainability

| Loại                                              | Giải Thích                                     | Dùng Cho                              |
| ------------------------------------------------- | ---------------------------------------------- | ------------------------------------- |
| **Global Explainability** (Giải Thích Toàn Cục)  | Feature importance trung bình toàn bộ dataset   | Hiểu model tổng thể, debug, audit     |
| **Local Explainability** (Giải Thích Cục Bộ)     | SHAP cho một prediction cụ thể                  | Giải thích cho từng khách hàng cụ thể |

### Chạy SHAP Analysis

```python
from sagemaker.clarify import SHAPConfig, ExplainabilityConfig

# Cấu hình SHAP
shap_config = SHAPConfig(
    baseline=[
        [
            # Baseline: giá trị trung bình (mean) của mỗi feature từ training data
            # Thứ tự phải khớp với headers
            35000.0,    # income (mean)
            680,        # credit_score (mean)
            0,          # gender (majority: male)
            15000.0,    # loan_amount (mean)
        ]
    ],
    num_samples=100,        # Nhiều hơn → chính xác hơn nhưng tốn nhiều compute hơn
    agg_method="mean_abs",  # Aggregate: mean_abs | median | mean_sq
    save_local_shap_values=True,   # Lưu SHAP cho từng sample (local explainability)
)

explainability_config = ExplainabilityConfig(shap_config=shap_config)

# Cấu hình dữ liệu — không cần cột label
data_config = DataConfig(
    s3_data_input_path="s3://my-bucket/validation/validation_no_label.csv",
    s3_output_path="s3://my-bucket/clarify-output/explainability/",
    headers=["income", "credit_score", "gender", "loan_amount"],
    dataset_type="text/csv",
)

# Chạy explainability analysis
clarify_processor.run_explainability(
    data_config=data_config,
    model_config=model_config,
    explainability_config=explainability_config,
    wait=True,
)
```

### Hiểu Kết Quả SHAP

```json
// Global feature importance (trung bình |SHAP| toàn dataset)
{
  "explanations": {
    "kernel_shap": {
      "label0": {
        "global_shap_values": {
          "credit_score": 0.423,    // Feature quan trọng nhất
          "income": 0.312,          // Quan trọng thứ hai
          "loan_amount": 0.198,     // Quan trọng thứ ba
          "gender": 0.067           // Quan trọng ít nhất — nhưng VẪN đóng góp!
        }
      }
    }
  }
}
```

**Diễn giải:** `gender` đóng góp 0.067 vào decision → model đang dùng giới tính như một predictive feature. Trong hầu hết use cases cho vay, đây là **illegal discrimination** (Phân Biệt Đối Xử Bất Hợp Pháp) — cần loại bỏ gender khỏi model.

### Local Explainability — Giải Thích Cho Từng Khách Hàng

```python
# Lấy SHAP values cho một sample cụ thể
import pandas as pd
import numpy as np

# Đọc local SHAP values
local_shap = pd.read_csv(
    "s3://my-bucket/clarify-output/explainability/explanations_shap/out.csv"
)

# Sample thứ 42 — khách hàng bị từ chối
sample_42_shap = local_shap.iloc[42]

print("Giải thích quyết định từ chối khoản vay của khách hàng #42:")
print(f"  credit_score: {sample_42_shap['credit_score']:.3f} (điểm tín dụng thấp, ảnh hưởng xấu)")
print(f"  loan_amount:  {sample_42_shap['loan_amount']:.3f} (khoản vay lớn, ảnh hưởng xấu)")
print(f"  income:       {sample_42_shap['income']:.3f} (thu nhập ổn, ảnh hưởng tốt)")
print(f"  gender:       {sample_42_shap['gender']:.3f} ← Không nên có ảnh hưởng!")

# Dùng thông tin này để giải thích với khách hàng tại sao bị từ chối
# và đề xuất cải thiện (ví dụ: cải thiện credit score)
```

---

## Clarify Trong SageMaker Pipeline {#clarify-pipeline}

### Tích Hợp Clarify Vào ML Pipeline

```python
from sagemaker.workflow.clarify_check_step import (
    ClarifyCheckStep,
    DataBiasCheckConfig,
    ModelBiasCheckConfig,
    ModelExplainabilityCheckConfig,
)
from sagemaker.workflow.check_job_config import CheckJobConfig

# Cấu hình chung cho Clarify check job
check_job_config = CheckJobConfig(
    role=role,
    instance_count=1,
    instance_type="ml.m5.large",
    volume_size_in_gb=20,
    sagemaker_session=pipeline_session,
)

# Bước 1: Kiểm tra data bias TRƯỚC khi train
data_bias_check_config = DataBiasCheckConfig(
    data_config=data_config,
    data_bias_config=bias_config,
    methods=["CI", "DPL", "JS"],        # Chỉ tính một số metrics để nhanh hơn
)

step_data_bias = ClarifyCheckStep(
    name="DataBiasCheck",
    clarify_check_config=data_bias_check_config,
    check_job_config=check_job_config,
    skip_check=False,
    fail_on_violation=True,             # Fail pipeline nếu phát hiện bias nghiêm trọng
    registered_model_name=model_package_group_name,
)

# Bước 2: Kiểm tra model bias SAU khi train
model_bias_check_config = ModelBiasCheckConfig(
    data_config=data_config,
    data_bias_config=bias_config,
    model_config=model_config,
    model_predicted_label_config=model_predicted_label_config,
    methods=["DPPL", "DI", "FPR", "FNR"],
)

step_model_bias = ClarifyCheckStep(
    name="ModelBiasCheck",
    clarify_check_config=model_bias_check_config,
    check_job_config=check_job_config,
    skip_check=False,
    fail_on_violation=True,
    registered_model_name=model_package_group_name,
)

# Bước 3: Explainability analysis
model_explainability_check_config = ModelExplainabilityCheckConfig(
    data_config=data_config,
    model_config=model_config,
    explainability_config=explainability_config,
)

step_explainability = ClarifyCheckStep(
    name="ModelExplainabilityCheck",
    clarify_check_config=model_explainability_check_config,
    check_job_config=check_job_config,
    skip_check=False,
    fail_on_violation=False,           # Không fail pipeline, chỉ log
    registered_model_name=model_package_group_name,
)

# Thứ tự trong pipeline:
# step_data_bias → step_train → step_model_bias → step_explainability → step_register
pipeline = Pipeline(
    name="FairMLPipeline",
    steps=[
        step_data_bias,
        step_train,          # step_train depends_on step_data_bias
        step_model_bias,     # depends_on step_train
        step_explainability, # depends_on step_model_bias
        step_register,       # depends_on step_explainability
    ],
)
```

---

## Bias Metrics Chi Tiết {#bias-metrics}

### Pre-training Metrics

| Metric                                                           | Công Thức (Đơn Giản)          | Ý Nghĩa                                              | Ngưỡng Nguy Hiểm |
| ---------------------------------------------------------------- | ----------------------------- | ---------------------------------------------------- | ----------------- |
| **CI** — Class Imbalance (Mất Cân Bằng Lớp)                     | (na - nd) / (na + nd)         | Nhóm được bảo vệ bị underrepresent trong data        | \|CI\| > 0.1      |
| **DPL** — Diff. Positive Proportions Labels                      | q_a - q_d                     | Chênh lệch tỷ lệ positive label giữa hai nhóm       | \|DPL\| > 0.1     |
| **JS** — Jensen-Shannon Divergence                               | JSD(P_a \|\| P_d)             | Sự khác biệt phân phối labels giữa hai nhóm         | JS > 0.1          |
| **KL** — Kullback-Leibler Divergence                             | KL(P_a \|\| P_d)              | Đo lường information gain nếu dùng P_d thay P_a     | KL > 0.2          |

### Post-training Metrics

| Metric                                                           | Công Thức (Đơn Giản)          | Ý Nghĩa                                              | Ngưỡng Nguy Hiểm |
| ---------------------------------------------------------------- | ----------------------------- | ---------------------------------------------------- | ----------------- |
| **DPPL** — Diff. Positive Proportions Predicted Labels           | q^_a - q^_d                   | Chênh lệch tỷ lệ dự đoán positive giữa hai nhóm     | \|DPPL\| > 0.1   |
| **DI** — Disparate Impact (Tác Động Không Công Bằng)             | q^_d / q^_a                   | Tỷ lệ approved giữa nhóm yếu thế / nhóm ưu thế      | DI < 0.8          |
| **AD** — Accuracy Difference (Chênh Lệch Độ Chính Xác)          | accuracy_a - accuracy_d       | Model chính xác hơn cho một nhóm                     | \|AD\| > 0.05    |
| **CDDPL** — Conditional Demographic Disparity                    | Conditional DPL               | DPPL sau khi kiểm soát confounding variables         | \|CDDPL\| > 0.1  |
| **FPR** — False Positive Rate Difference                         | FPR_a - FPR_d                 | Model false positive nhiều hơn với một nhóm          | \|FPR\| > 0.05   |
| **FNR** — False Negative Rate Difference                         | FNR_a - FNR_d                 | Model false negative nhiều hơn với một nhóm          | \|FNR\| > 0.05   |

*Ghi chú: a = nhóm ưu thế (advantaged), d = nhóm được bảo vệ (disadvantaged/facet group)*

### Cách Remediate Bias (Khắc Phục Thiên Kiến)

```
Bias Detected → Điều Tra Nguồn Gốc
       │
       ├── Data Bias? (Pre-training CI, DPL cao)
       │    └── Solutions:
       │         - Oversampling (Lấy Mẫu Quá Mức): Tăng samples của nhóm thiểu số
       │         - Undersampling (Lấy Mẫu Dưới Mức): Giảm samples của nhóm đa số
       │         - Synthetic data generation (Tạo DL Tổng Hợp): SMOTE technique
       │         - Reweighting: Gán sample weights cho classes
       │
       ├── Feature Bias? (SHAP của facet feature cao)
       │    └── Solutions:
       │         - Loại bỏ facet feature khỏi model
       │         - Dùng proxy feature removal (loại bỏ feature proxy)
       │         - Fairness constraints trong training objective
       │
       └── Model Bias? (Post-training DPPL, DI vượt ngưỡng)
            └── Solutions:
                 - Resampling training data
                 - Adversarial debiasing (Giải Thiên Kiến Đối Nghịch)
                 - Calibrate predictions per group
                 - Threshold adjustment per demographic group
```

---

## Câu Hỏi Phỏng Vấn {#phỏng-vấn}

**H: SHAP hoạt động như thế nào? Tại sao dùng SHAP thay vì feature importance đơn giản?**

> **SHAP** dựa trên lý thuyết trò chơi Shapley: tính đóng góp "công bằng" của mỗi feature bằng cách xem xét nó trong mọi tổ hợp có thể với các features khác.
>
> **Feature importance truyền thống** (ví dụ: Gain trong XGBoost) chỉ cho biết feature đó tổng thể quan trọng bao nhiêu — nhưng không cho biết nó ảnh hưởng thế nào trong từng prediction cụ thể.
>
> **SHAP** cho biết cả:
> - **Global**: Feature nào quan trọng nhất tổng thể
> - **Local**: Tại sao prediction CỤ THỂ này là +0.7 probability fraud (do feature nào đóng góp bao nhiêu)
> - **Direction**: Feature này tăng hay giảm probability (positive hay negative contribution)

**H: Disparate Impact là gì? DI = 0.7 có nghĩa gì?**

> **Disparate Impact** (Tác Động Không Công Bằng) đo tỷ lệ kết quả tích cực giữa nhóm yếu thế và nhóm tham chiếu:
> `DI = P(positive outcome | disadvantaged group) / P(positive outcome | reference group)`
>
> `DI = 0.7` nghĩa là: nhóm được bảo vệ nhận kết quả tích cực chỉ 70% so với nhóm tham chiếu. Ví dụ: female được approve loan với tỷ lệ 35% trong khi male là 50% → DI = 35/50 = 0.7.
>
> **Ngưỡng pháp lý ở Mỹ**: DI < 0.8 là "adverse impact" (Tác Động Bất Lợi) theo EEOC (Equal Employment Opportunity Commission — Ủy Ban Cơ Hội Bình Đẳng Việc Làm). Trong AI, ngưỡng này thường được áp dụng tương tự.

**H: Khi nào dùng pre-training bias analysis vs post-training?**

> - **Pre-training**: Luôn chạy trước khi train. Phát hiện vấn đề trong DATA sớm nhất — rẻ hơn vì không cần train model. Nếu data bị bias nghiêm trọng, fix data trước → tiết kiệm thời gian train.
> - **Post-training**: Chạy sau khi train để xác nhận model đã học bias từ data hay tạo ra bias mới. Kể cả khi data không có bias, model vẫn có thể học được patterns dẫn đến bias.
>
> **Best practice**: Chạy cả hai trong pipeline — pre-training để fix data, post-training để validate model fairness trước khi deploy.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
