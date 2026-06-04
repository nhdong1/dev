# Fine-tuning (Tinh Chỉnh Mô Hình) trên Amazon Bedrock

> Fine-tuning — Tinh Chỉnh Mô Hình là quá trình tiếp tục huấn luyện một Foundation Model đã được pre-trained (huấn luyện trước) trên tập dữ liệu riêng của bạn, để model học style, terminology (thuật ngữ), và behavior (hành vi) đặc thù theo domain hoặc use case cụ thể.

---

## 🎯 Khi Nào Cần Fine-tuning?

### So Sánh Các Approaches

```
Prompt Engineering (Không cần train):
  Ưu: Nhanh, rẻ, dễ thay đổi
  Nhược: Model không "học" được — phải viết prompt dài mỗi lần
  Dùng khi: Thay đổi cách trình bày, thêm context

RAG (Retrieval-Augmented Generation):
  Ưu: Dữ liệu luôn mới nhất, có source traceability (truy xuất nguồn)
  Nhược: Latency cao hơn, cần vector store
  Dùng khi: Cần trả lời dựa trên documents cụ thể thay đổi thường xuyên

Fine-tuning (Tinh Chỉnh):
  Ưu: Model học sâu, output format nhất quán, latency thấp
  Nhược: Cần nhiều data, chi phí training, khó update kiến thức mới
  Dùng khi: Cần model adopt style/format đặc biệt, domain vocabulary khác lạ
```

### Dấu Hiệu Cần Fine-tuning

| Dấu Hiệu | Ví Dụ | Giải Pháp |
|---|---|---|
| Prompt quá dài và lặp đi lặp lại | Mỗi call cần 2,000 tokens system prompt | Fine-tune → embed knowledge vào model |
| Output không nhất quán dù prompt rõ | JSON format thỉnh thoảng sai | Fine-tune với ví dụ JSON đúng |
| Model không hiểu jargon nội bộ | "Giao dịch T+2", "Lệnh ATC" — thuật ngữ chứng khoán | Fine-tune với domain vocabulary |
| Cần style/tone cụ thể | Luôn dùng "Quý khách" và bullet points ngắn | Fine-tune với exemplars |
| Few-shot không đủ | Cần 10+ examples nhưng vẫn sai | Fine-tune thay vì few-shot |

---

## 📚 Các Phương Pháp Fine-tuning Trên Bedrock

### 1. Instruction Tuning (Tinh Chỉnh Theo Hướng Dẫn)

Loại fine-tuning phổ biến nhất — dạy model làm theo instructions với examples cụ thể:

```
Dataset format (Định Dạng Tập Dữ Liệu):
{
  "prompt": "Phân loại email sau: [nội dung email]",
  "completion": "CATEGORY: Sales\nPRIORITY: High\nACTION: Forward to sales team"
}
```

**Phù hợp cho:**
- Phân loại văn bản (text classification)
- Tóm tắt theo định dạng cụ thể
- Dịch thuật chuyên ngành
- Trả lời câu hỏi theo format cố định

### 2. Continued Pre-training (Tiếp Tục Huấn Luyện Trước)

Tiếp tục pre-training với dữ liệu raw text (văn bản thô) trong domain cụ thể — không cần labeled data:

```
Input: Văn bản không có nhãn từ domain của bạn
VD: 10,000 trang tài liệu y khoa tiếng Việt
    10,000 trang báo cáo tài chính
    50,000 email hỗ trợ khách hàng

Kết quả: Model hiểu sâu hơn về domain vocabulary và concepts
```

**Khác biệt:**
- **Continued Pre-training**: Dữ liệu raw text, không có Q&A pairs — model học vocabulary và knowledge
- **Instruction Tuning**: Cần prompt-completion pairs — model học cách làm theo instructions

### 3. PEFT — Parameter-Efficient Fine-Tuning (Tinh Chỉnh Hiệu Quả Tham Số)

Thay vì update toàn bộ weights (trọng số) của model (tốn kém), PEFT chỉ update một phần nhỏ:

```
Full fine-tuning:
  Update ALL 70 billion parameters
  Chi phí: Rất cao, cần nhiều GPU RAM
  Kết quả: Tốt nhất

PEFT (LoRA):
  Update ~0.1% parameters (adapter layers)
  Chi phí: Thấp hơn 10-100x
  Kết quả: Gần tương đương full fine-tuning
```

**Các PEFT methods được Bedrock hỗ trợ:**
- **LoRA** — Low-Rank Adaptation (Thích Nghi Hạng Thấp): Thêm adapter matrices nhỏ vào attention layers
- **QLoRA** — Quantized LoRA: LoRA kết hợp với quantization (nén mô hình), tiết kiệm memory hơn

---

## 🔧 Cấu Hình Fine-tuning Trên Bedrock

### Các Model Hỗ Trợ Fine-tuning

| Model | Phương Pháp | Data Requirements |
|---|---|---|
| **Amazon Titan Text** | Instruction tuning, Continued pre-training | 32 → 10,000+ examples |
| **Meta Llama 3.1** | Instruction tuning | 32 → 10,000+ examples |
| **Anthropic Claude** | Instruction tuning (giới hạn) | Liên hệ AWS |
| **Cohere Command** | Instruction tuning | 32 → 10,000+ examples |

### Định Dạng Dataset (Tập Dữ Liệu)

#### Instruction Tuning Dataset Format

File JSONL (JSON Lines — mỗi dòng là một JSON object):

```jsonl
{"prompt": "Tóm tắt email sau trong 1 câu:\n\nGửi anh/chị,\nChúng tôi xin thông báo rằng hệ thống bảo trì định kỳ sẽ diễn ra vào thứ 7, 10/7/2026 từ 2:00-4:00 sáng...", "completion": "Thông báo bảo trì hệ thống thứ 7 ngày 10/7 từ 2-4 giờ sáng."}
{"prompt": "Tóm tắt email sau trong 1 câu:\n\nKính gửi quý khách,\nĐơn hàng #ORD-2456 của quý khách đã được xác nhận và sẽ giao trong 3-5 ngày làm việc...", "completion": "Xác nhận đơn hàng #ORD-2456 sẽ giao trong 3-5 ngày làm việc."}
{"prompt": "Tóm tắt email sau trong 1 câu:\n\nHi team,\nNhắc nhở sprint review diễn ra lúc 2pm thứ Sáu tuần này. Vui lòng chuẩn bị demo...", "completion": "Nhắc nhở sprint review 2pm thứ Sáu, cần chuẩn bị demo."}
```

#### Continued Pre-training Dataset Format

File TXT hoặc JSONL với raw text:

```jsonl
{"text": "Điều khoản bảo hiểm nhân thọ theo nghĩa rộng bao gồm các sản phẩm như: bảo hiểm nhân thọ trọn đời, bảo hiểm nhân thọ có kỳ hạn, bảo hiểm hỗn hợp..."}
{"text": "Quy trình xử lý bồi thường bảo hiểm: Bước 1, người được bảo hiểm thông báo sự cố trong vòng 24-48 giờ. Bước 2, cung cấp hồ sơ theo yêu cầu..."}
{"text": "Tỷ lệ phí bảo hiểm được tính dựa trên: tuổi, giới tính, tình trạng sức khỏe, số tiền bảo hiểm, thời hạn hợp đồng..."}
```

### Yêu Cầu Về Dữ Liệu

```
Tối thiểu tuyệt đối: 32 examples (cho kiểm tra)
Recommended (Khuyến Nghị):
  - Instruction tuning đơn giản: 1,000+ examples
  - Instruction tuning phức tạp: 10,000+ examples
  - Continued pre-training: 100,000+ tokens (nhiều hơn càng tốt)

Chất lượng quan trọng hơn số lượng:
  ✅ 1,000 examples chất lượng cao
  ❌ 10,000 examples nhiễu loạn, sai format
```

---

## 🚀 Thực Hiện Fine-tuning Qua Python SDK

### Bước 1: Upload Dataset Lên S3

```python
import boto3
import json

s3 = boto3.client("s3", region_name="us-east-1")
BUCKET = "my-fine-tuning-data"

def prepare_and_upload_dataset(
    training_data: list[dict],
    validation_data: list[dict]
) -> tuple[str, str]:
    """
    Upload training và validation data lên S3.

    Args:
        training_data: List of {"prompt": "...", "completion": "..."}
        validation_data: Tập validation (10-20% của training data)

    Returns:
        (training_s3_uri, validation_s3_uri)
    """
    def write_jsonl_to_s3(data: list[dict], key: str) -> str:
        content = "\n".join(json.dumps(item, ensure_ascii=False) for item in data)
        s3.put_object(
            Bucket=BUCKET,
            Key=key,
            Body=content.encode("utf-8"),
            ContentType="application/jsonl"
        )
        return f"s3://{BUCKET}/{key}"

    train_uri = write_jsonl_to_s3(training_data, "fine-tuning/train.jsonl")
    val_uri = write_jsonl_to_s3(validation_data, "fine-tuning/validation.jsonl")

    return train_uri, val_uri
```

### Bước 2: Tạo Fine-tuning Job

```python
bedrock = boto3.client("bedrock", region_name="us-east-1")

def create_fine_tuning_job(
    job_name: str,
    train_s3_uri: str,
    val_s3_uri: str,
    output_s3_uri: str,
    role_arn: str
) -> str:
    """
    Tạo Bedrock fine-tuning job.

    Returns:
        job_arn — ARN của job để theo dõi
    """
    response = bedrock.create_model_customization_job(
        jobName=job_name,
        customModelName=f"{job_name}-model",
        roleArn=role_arn,

        # Model gốc để fine-tune
        baseModelIdentifier="amazon.titan-text-express-v1",

        # Loại customization
        customizationType="FINE_TUNING",  # hoặc "CONTINUED_PRE_TRAINING"

        # Dataset
        trainingDataConfig={"s3Uri": train_s3_uri},
        validationDataConfig={
            "validators": [{"s3Uri": val_s3_uri}]
        },

        # Nơi lưu model sau training
        outputDataConfig={"s3Uri": output_s3_uri},

        # Hyperparameters (Siêu Tham Số)
        hyperParameters={
            "epochCount": "3",           # Số lần duyệt qua toàn bộ dataset
            "batchSize": "8",            # Số examples mỗi gradient update (cập nhật gradient)
            "learningRate": "0.00005",   # Tốc độ học — thấp để không overfit (quá khớp)
            "learningRateWarmupSteps": "100"  # Khởi động learning rate từ từ
        }
    )

    return response["jobArn"]


def monitor_fine_tuning_job(job_arn: str):
    """Theo dõi trạng thái fine-tuning job."""
    import time

    while True:
        response = bedrock.get_model_customization_job(jobIdentifier=job_arn)
        status = response["status"]

        print(f"Status: {status}")

        if status == "Completed":
            custom_model_arn = response["outputModelArn"]
            print(f"✅ Fine-tuning hoàn thành! Model ARN: {custom_model_arn}")
            return custom_model_arn
        elif status == "Failed":
            failure_message = response.get("failureMessage", "Unknown error")
            print(f"❌ Fine-tuning thất bại: {failure_message}")
            return None
        elif status in ["InProgress", "Stopping"]:
            time.sleep(60)  # Kiểm tra lại sau 1 phút
```

### Bước 3: Dùng Custom Model (Mô Hình Đã Tinh Chỉnh)

```python
def invoke_custom_model(custom_model_arn: str, prompt: str) -> str:
    """
    Gọi custom fine-tuned model (cần tạo Provisioned Throughput trước).
    Note: Custom models không hỗ trợ on-demand — cần Provisioned Throughput.
    """
    bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

    body = {
        "inputText": prompt,
        "textGenerationConfig": {
            "maxTokenCount": 512,
            "temperature": 0.3,
            "topP": 0.9
        }
    }

    # Dùng Provisioned Throughput ARN (tạo ở bước sau)
    response = bedrock_runtime.invoke_model(
        modelId=provisioned_throughput_arn,
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    return result["results"][0]["outputText"]


def create_provisioned_throughput(custom_model_arn: str, model_units: int = 1) -> str:
    """
    Tạo Provisioned Throughput (Thông Lượng Dự Phòng) cho custom model.
    Custom models bắt buộc dùng provisioned throughput, không hỗ trợ on-demand.

    Args:
        model_units: Số Model Units, mỗi unit = một mức throughput nhất định
    """
    response = bedrock.create_provisioned_model_throughput(
        provisionedModelName="my-custom-model-prod",
        modelId=custom_model_arn,
        modelUnits=model_units
    )
    return response["provisionedModelArn"]
```

---

## 📊 Hyperparameter Tuning (Tinh Chỉnh Siêu Tham Số)

### Các Hyperparameters Quan Trọng

| Hyperparameter | Giải Thích | Giá Trị Khuyến Nghị | Ảnh Hưởng |
|---|---|---|---|
| **epochCount** (Số Epoch) | Số lần duyệt toàn bộ training data | 2-5 | Nhiều epoch → model học sâu hơn nhưng dễ overfit |
| **batchSize** (Kích Thước Batch) | Số examples mỗi lần cập nhật weights | 4, 8, 16 | Batch nhỏ → gradient noise nhưng generalize tốt hơn |
| **learningRate** (Tốc Độ Học) | Mức độ cập nhật weights mỗi bước | 1e-5 to 1e-4 | Cao → học nhanh nhưng unstable; Thấp → ổn định nhưng chậm |
| **warmupSteps** (Bước Khởi Động) | Số bước tăng learning rate từ 0 lên | 10-100 | Tránh instability đầu training |

### Learning Rate Schedule (Lịch Trình Tốc Độ Học)

```
Với warmup (khởi động):

Learning Rate
     |          /‾‾‾‾‾‾\
     |         /        \___________
     |        /
     |_______/
     0    warmup    decay end
           steps

- Phase 1 (Warmup): Tăng lr từ 0 đến target
- Phase 2 (Peak): Giữ lr ổn định
- Phase 3 (Decay): Giảm lr để converge (hội tụ)
```

### Dấu Hiệu Overfitting (Quá Khớp) vs Underfitting (Dưới Khớp)

```
Overfitting (Quá Khớp) — Training loss thấp, Validation loss cao:
  ✅ Giải pháp:
  - Giảm epochCount
  - Tăng dataset size
  - Giảm batchSize
  - Thêm data augmentation

Underfitting (Dưới Khớp) — Cả hai loss đều cao:
  ✅ Giải pháp:
  - Tăng epochCount
  - Tăng learningRate nhẹ
  - Kiểm tra chất lượng data
  - Tăng model capacity (dùng model lớn hơn)
```

---

## 🆚 Fine-tuning vs RAG — Bảng Quyết Định Toàn Diện

```
                   Fine-tuning                    RAG
                        │                          │
Kiến thức thay đổi?  Không/ít ◄────────────────► Thường xuyên
                                                   │
                        │                         RAG
Cần cite nguồn?      Không ◄────────────────► Có
                                                   │
                        │                         RAG
Có labeled data?   Có, nhiều ◄────────────────► Ít/không
                        │
                  Fine-tuning
                        │
Latency critical?   Rất quan trọng ◄───────────► Có thể chấp nhận thêm
                        │                         200-500ms
                  Fine-tuning
                        │
Cần custom style?    Quan trọng ◄──────────────► Không bắt buộc
                        │
                  Fine-tuning

Kết hợp tốt nhất: Fine-tuned model + RAG
→ Model học domain vocabulary qua fine-tuning
→ Model truy cập thông tin mới nhất qua RAG
```

---

## 💰 Chi Phí Fine-tuning

### Cấu Trúc Chi Phí

| Giai Đoạn | Chi Phí | Ví Dụ |
|---|---|---|
| **Training** | Per token processed | Titan Text: $0.008/1K tokens |
| **Storage** | S3 standard rates | ~$0.023/GB/tháng |
| **Provisioned Throughput** | Per Model Unit per hour | ~$8-20/MU/giờ tùy model |

### Ước Tính Chi Phí

```python
def estimate_fine_tuning_cost(
    num_examples: int,
    avg_tokens_per_example: int,
    num_epochs: int,
    cost_per_1k_tokens: float = 0.008  # Titan Text
) -> dict:
    """Ước tính chi phí fine-tuning."""
    total_tokens = num_examples * avg_tokens_per_example * num_epochs
    training_cost = (total_tokens / 1000) * cost_per_1k_tokens

    return {
        "total_training_tokens": total_tokens,
        "estimated_training_cost_usd": round(training_cost, 2),
        "note": "Chi phí provisioned throughput tính riêng theo giờ"
    }

# Ví dụ:
estimate = estimate_fine_tuning_cost(
    num_examples=5000,
    avg_tokens_per_example=500,
    num_epochs=3
)
# → training_cost: ~$30 cho 7.5M tokens
```

---

## 🔬 Đánh Giá Mô Hình Sau Fine-tuning

### Evaluation Metrics (Chỉ Số Đánh Giá)

```python
def evaluate_model_quality(
    test_cases: list[dict],
    model_arn: str
) -> dict:
    """
    Đánh giá chất lượng fine-tuned model trên test set.

    test_cases: [{"prompt": "...", "expected": "..."}]
    """
    import difflib

    results = {
        "exact_match": 0,
        "similarity_avg": 0,
        "total": len(test_cases)
    }
    similarities = []

    for case in test_cases:
        actual = invoke_custom_model(model_arn, case["prompt"])
        expected = case["expected"]

        # Exact match (Khớp Chính Xác)
        if actual.strip() == expected.strip():
            results["exact_match"] += 1

        # Sequence similarity (Độ Tương Đồng Chuỗi) — 0.0 đến 1.0
        sim = difflib.SequenceMatcher(None, actual, expected).ratio()
        similarities.append(sim)

    results["exact_match_rate"] = results["exact_match"] / results["total"]
    results["similarity_avg"] = sum(similarities) / len(similarities)

    return results
```

### Validation Loss Theo Dõi

```python
def get_training_metrics(job_arn: str) -> dict:
    """Lấy training và validation loss theo từng epoch."""
    job = bedrock.get_model_customization_job(jobIdentifier=job_arn)

    metrics = {}
    for epoch_metric in job.get("trainingMetrics", {}).get("trainingLoss", []):
        step = epoch_metric["step"]
        loss = epoch_metric["value"]
        metrics[f"train_loss_step_{step}"] = loss

    for val_metric in job.get("validationMetrics", []):
        metrics["validation_loss"] = val_metric["validationLoss"]

    return metrics
```

---

## 📋 Quy Trình Fine-tuning Hoàn Chỉnh (Checklist)

```
Phase 1: Chuẩn Bị Dữ Liệu (Data Preparation)
  □ Thu thập 1,000-10,000+ examples chất lượng cao
  □ Định nghĩa rõ prompt format và completion format
  □ Tách 80% train / 10% validation / 10% test
  □ Review sample 50 examples thủ công để kiểm tra chất lượng
  □ Upload lên S3 dưới dạng JSONL

Phase 2: Training
  □ Chọn base model phù hợp (Titan, Llama, v.v.)
  □ Bắt đầu với hyperparameters mặc định
  □ Monitor training loss và validation loss
  □ Nếu overfit: giảm epoch, nếu underfit: tăng epoch

Phase 3: Đánh Giá (Evaluation)
  □ Chạy test set trên fine-tuned model
  □ So sánh với base model không fine-tune
  □ So sánh với few-shot prompting approach
  □ Kiểm tra edge cases và failure modes

Phase 4: Deployment (Triển Khai)
  □ Tạo Provisioned Throughput với số model units phù hợp
  □ Tích hợp vào ứng dụng với fallback logic
  □ Set up CloudWatch monitoring cho latency và errors
  □ Lên kế hoạch re-training khi data mới tích lũy
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Fine-tuning, RAG, và Prompt Engineering — khi nào chọn cái nào?**

> - **Prompt Engineering**: Luôn thử trước — nhanh, rẻ, không cần data. Đủ cho nhiều use cases.
> - **RAG**: Khi cần truy cập knowledge cụ thể, thay đổi thường xuyên, hoặc cần cite nguồn.
> - **Fine-tuning**: Khi cần model adopt style/format/vocabulary đặc biệt, prompt engineering không đủ, và có đủ data chất lượng.
> - **RAG + Fine-tuning**: Best of both worlds — model học domain, vẫn có current knowledge.

**Q: LoRA — Low-Rank Adaptation là gì? Tại sao quan trọng?**

> LoRA thêm các ma trận nhỏ (low-rank decomposition) vào attention layers của model. Thay vì update toàn bộ billions of parameters (tốn kém), LoRA chỉ train các ma trận adapter nhỏ này (~0.1% params). Kết quả: Chi phí giảm 10-100x, thời gian training ngắn hơn nhiều, chất lượng gần tương đương full fine-tuning.

**Q: Overfitting (quá khớp) trong fine-tuning là gì? Phát hiện và xử lý thế nào?**

> Overfitting là khi model "học thuộc" training data nhưng không generalize (tổng quát hóa) tốt trên data mới. Phát hiện: training loss thấp nhưng validation loss cao hoặc tăng. Xử lý: giảm số epochs, tăng kích thước tập training, early stopping (dừng sớm khi validation loss tăng).

**Q: Custom model trên Bedrock có thể dùng on-demand inference không?**

> Không. Custom fine-tuned models bắt buộc phải dùng **Provisioned Throughput** — phải đặt trước capacity và trả phí theo giờ. Đây là điểm khác biệt quan trọng so với base Foundation Models (có thể dùng on-demand pay-per-token).

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
