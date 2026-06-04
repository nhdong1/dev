# Kế Hoạch Học 90 Ngày — AWS AI/ML

> Lộ trình học chi tiết theo ngày và tuần, được thiết kế cho người có nền tảng lập trình nhưng chưa có kinh nghiệm sâu về AWS AI/ML. Mục tiêu cuối: tự tin phỏng vấn vị trí Senior Engineer liên quan đến AWS AI/ML.

---

## 📊 Tổng Quan 3 Giai Đoạn

| Giai Đoạn | Ngày | Nội Dung | Mục Tiêu Đầu Ra |
|-----------|------|----------|-----------------|
| **Giai Đoạn 1 — Nền Tảng** | 1-30 | AI/ML fundamentals, SageMaker cơ bản, Bedrock | Gọi được API, hiểu kiến trúc core |
| **Giai Đoạn 2 — Chuyên Sâu** | 31-60 | AI Services, MLOps, Advanced topics | Thiết kế được pipeline end-to-end |
| **Giai Đoạn 3 — Phỏng Vấn** | 61-90 | Luyện câu hỏi, system design, mock interview | Sẵn sàng phỏng vấn tự tin |

---

## ⚙️ Thiết Lập Trước Khi Bắt Đầu

### Môi Trường Học (Hoàn Thành Trước Ngày 1)

```bash
# 1. Tạo AWS Free Tier Account
# Truy cập: https://aws.amazon.com/free/

# 2. Cài đặt AWS CLI — Command Line Interface
# Windows: msi installer từ docs.aws.amazon.com
aws configure
# Nhập: AWS Access Key ID, Secret, Region (ap-southeast-1 hoặc us-east-1), output format (json)

# 3. Kiểm tra kết nối
aws sts get-caller-identity

# 4. Cài đặt Python và boto3 — AWS SDK cho Python
pip install boto3 awscli

# 5. Tạo S3 bucket cho lab exercises
aws s3 mb s3://your-name-aws-ai-lab-2026

# 6. Tạo IAM Role với các permissions cần thiết
# Attach policies: AmazonSageMakerFullAccess, AmazonRekognitionFullAccess,
# AmazonComprehendFullAccess, AmazonTranscribeFullAccess, BedrockFullAccess
```

### Công Cụ Cần Có

- [ ] **VS Code** với Python extension
- [ ] **AWS Console** — browser bookmarks cho SageMaker, Bedrock, Rekognition
- [ ] **Jupyter Notebook** — cho SageMaker Studio exercises
- [ ] **Postman** — test API calls (tùy chọn)
- [ ] **Notion / Obsidian** — ghi chú và flashcards

---

## 🌱 GIAI ĐOẠN 1: NỀN TẢNG (Ngày 1-30)

### Tuần 1 (Ngày 1-7): AI/ML Fundamentals — Nền Tảng

**Mục tiêu tuần:** Hiểu được landscape AI/ML và AWS strategy

#### Ngày 1 — Tổng Quan AI/ML

**Đọc:**
- `01-fundamentals/1-ai-ml-overview.md` — AI vs ML vs Deep Learning vs Generative AI
- `01-fundamentals/2-ml-types.md` — Supervised, Unsupervised, Reinforcement Learning

**Thực hành:**
```bash
# Gọi Comprehend API lần đầu tiên
aws comprehend detect-sentiment \
  --text "AWS AI services are incredibly powerful and easy to use!" \
  --language-code en
```

**Ghi chú cần nhớ:**
- AI ⊃ ML ⊃ Deep Learning
- Supervised: có label; Unsupervised: không label; Reinforcement: reward signal

---

#### Ngày 2 — ML Workflow và AWS AI Layers

**Đọc:**
- `01-fundamentals/3-ml-workflow.md` — Data Prep → Training → Evaluation → Deployment → Monitoring
- `01-fundamentals/4-aws-ai-layers.md` — Tầng 1 (AI Services), Tầng 2 (SageMaker), Tầng 3 (Framework)

**Thực hành:**
```bash
# Gọi Rekognition detect-labels
aws rekognition detect-labels \
  --image '{"S3Object":{"Bucket":"your-bucket","Name":"photo.jpg"}}' \
  --max-labels 10 \
  --min-confidence 70

# Upload một bức ảnh lên S3 trước:
aws s3 cp my-photo.jpg s3://your-name-aws-ai-lab-2026/
```

**Ghi chú:** Ba tầng AI AWS — biết chọn đúng tầng cho từng use case

---

#### Ngày 3 — Generative AI Concepts

**Đọc:**
- `01-fundamentals/5-generative-ai-concepts.md` — LLM, Foundation Model, Token, Prompt, RAG, Embedding

**Thực hành:**
- Vào **Amazon Bedrock Playground** (console.aws.amazon.com → Bedrock → Playground)
- Thử Claude 3 Sonnet: so sánh zero-shot vs few-shot prompt
- Thử Titan Text Express

**Ghi chú:** Token là gì? 1000 tokens ≈ 750 từ tiếng Anh

---

#### Ngày 4-5 — Amazon Rekognition Hands-on

**Đọc:**
- `04-computer-vision/1-rekognition-image.md`
- `04-computer-vision/4-textract.md`

**Thực hành (Python):**
```python
import boto3

rekognition = boto3.client('rekognition', region_name='us-east-1')

# Detect faces
response = rekognition.detect_faces(
    Image={'S3Object': {'Bucket': 'your-bucket', 'Name': 'portrait.jpg'}},
    Attributes=['ALL']
)
for face in response['FaceDetails']:
    print(f"Confidence: {face['Confidence']:.1f}%")
    print(f"Age range: {face['AgeRange']}")
    print(f"Emotions: {sorted(face['Emotions'], key=lambda x: x['Confidence'], reverse=True)[:3]}")

# Detect labels
response = rekognition.detect_labels(
    Image={'S3Object': {'Bucket': 'your-bucket', 'Name': 'scene.jpg'}},
    MaxLabels=10
)
for label in response['Labels']:
    print(f"{label['Name']}: {label['Confidence']:.1f}%")
```

**Mini project:** Viết script upload ảnh → detect labels → save kết quả vào DynamoDB

---

#### Ngày 6-7 — Amazon Comprehend Hands-on

**Đọc:**
- `05-nlp-text/1-comprehend-fundamentals.md`

**Thực hành (Python):**
```python
import boto3

comprehend = boto3.client('comprehend', region_name='us-east-1')

texts = [
    "I absolutely love this product! Best purchase ever.",
    "Terrible quality. Broke after 2 days. Very disappointed.",
    "It's okay, nothing special but does the job."
]

for text in texts:
    # Sentiment
    sentiment = comprehend.detect_sentiment(Text=text, LanguageCode='en')
    print(f"Text: {text[:50]}...")
    print(f"Sentiment: {sentiment['Sentiment']} (confidence: {sentiment['SentimentScore'][sentiment['Sentiment'].capitalize()]:.2f})")

    # Entities
    entities = comprehend.detect_entities(Text=text, LanguageCode='en')
    print(f"Entities: {[(e['Text'], e['Type']) for e in entities['Entities']]}")
    print("---")
```

**Cuối tuần 1:** Viết tóm tắt 1 trang những gì đã học, ghi lại các câu hỏi còn thắc mắc

---

### Tuần 2 (Ngày 8-14): Amazon SageMaker Cơ Bản

**Mục tiêu tuần:** Hiểu SageMaker architecture, chạy training job đầu tiên

#### Ngày 8-9 — SageMaker Studio và Tổng Quan

**Đọc:**
- `02-sagemaker/README.md` — Tổng quan SageMaker
- `02-sagemaker/1-sagemaker-studio.md` — Studio IDE

**Thực hành:**
1. Tạo SageMaker Domain trong AWS Console
2. Tạo User Profile
3. Mở JupyterLab từ SageMaker Studio
4. Explore giao diện: Experiments, Pipelines, Model Registry

**Lưu ý:** SageMaker Studio tính phí khi JupyterLab đang chạy — **nhớ shutdown khi không dùng**

---

#### Ngày 10-11 — SageMaker Training Job

**Đọc:**
- `02-sagemaker/2-sagemaker-training.md`

**Thực hành:**
```python
import sagemaker
from sagemaker import get_execution_role
from sagemaker.estimator import Estimator

role = get_execution_role()
session = sagemaker.Session()

# Train XGBoost built-in algorithm trên Iris dataset
xgb = sagemaker.estimator.Estimator(
    image_uri=sagemaker.image_uris.retrieve("xgboost", session.boto_region_name, "1.7-1"),
    role=role,
    instance_count=1,
    instance_type="ml.m5.xlarge",
    volume_size=5,
    output_path=f"s3://{session.default_bucket()}/output",
)

xgb.set_hyperparameters(
    objective="multi:softmax",
    num_class=3,
    num_round=100,
)

# Input channels
from sagemaker.inputs import TrainingInput
xgb.fit({
    "train": TrainingInput("s3://your-bucket/iris/train.csv", content_type="csv"),
    "validation": TrainingInput("s3://your-bucket/iris/val.csv", content_type="csv"),
})

print(f"Training job: {xgb.latest_training_job.name}")
```

---

#### Ngày 12-13 — SageMaker Inference

**Đọc:**
- `02-sagemaker/3-sagemaker-inference.md` — Real-time, Batch, Async, Serverless

**Thực hành:**
```python
# Deploy model vừa train lên real-time endpoint
predictor = xgb.deploy(
    initial_instance_count=1,
    instance_type="ml.t2.medium",
    endpoint_name="iris-xgb-endpoint"
)

# Gọi inference
import numpy as np
test_data = np.array([[5.1, 3.5, 1.4, 0.2]])  # Iris sample
result = predictor.predict(test_data)
print(f"Predicted class: {result}")

# QUAN TRỌNG: Xóa endpoint sau khi xong để tránh tính phí
predictor.delete_endpoint()
```

---

#### Ngày 14 — Ôn Tập và Quiz

**Tự kiểm tra:**
- [ ] Giải thích được SageMaker training job lifecycle
- [ ] Biết phân biệt 4 inference types
- [ ] Đã thực sự chạy training job và delete endpoint
- [ ] Hiểu tại sao phải delete endpoint

---

### Tuần 3 (Ngày 15-21): Amazon Bedrock và Generative AI

**Mục tiêu tuần:** Xây dựng được ứng dụng Generative AI đơn giản

#### Ngày 15-16 — Bedrock Foundation Models và Prompt Engineering

**Đọc:**
- `03-bedrock/1-foundation-models.md` — Claude, Llama 3, Titan, so sánh
- `03-bedrock/2-prompt-engineering.md` — Zero-shot, Few-shot, Chain-of-thought

**Thực hành (Python):**
```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

# Gọi Claude 3 Sonnet
def call_claude(prompt, max_tokens=500):
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_tokens,
        "messages": [{"role": "user", "content": prompt}]
    }
    response = bedrock.invoke_model(
        modelId="anthropic.claude-3-sonnet-20240229-v1:0",
        body=json.dumps(body)
    )
    return json.loads(response['body'].read())['content'][0]['text']

# Zero-shot
print(call_claude("What is the capital of Vietnam?"))

# Few-shot
few_shot_prompt = """
Classify sentiment:
'Great product!' → Positive
'Terrible quality' → Negative
'Amazon delivery was fast this time.' → ?
"""
print(call_claude(few_shot_prompt))

# Chain-of-thought
cot_prompt = """
Think step by step:
A model was trained on data from Jan 2023. Today is June 2026.
If a user asks about events in March 2026, what should the model do?
"""
print(call_claude(cot_prompt))
```

---

#### Ngày 17-18 — RAG Pipeline và Bedrock Knowledge Base

**Đọc:**
- `03-bedrock/3-rag-knowledge-base.md`

**Thực hành:**
1. Tạo Bedrock Knowledge Base trong Console
2. Connect S3 bucket với 3-5 PDF documents
3. Sync và test retrieve
4. Gọi RetrieveAndGenerate API

```python
bedrock_agent = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

response = bedrock_agent.retrieve_and_generate(
    input={"text": "What is the return policy?"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "YOUR_KB_ID",
            "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0"
        }
    }
)
print(response['output']['text'])
print("Sources:", [r['location']['s3Location']['uri'] for r in response['citations'][0]['retrievedReferences']])
```

---

#### Ngày 19-20 — Bedrock Agents và Guardrails

**Đọc:**
- `03-bedrock/4-bedrock-agents.md`
- Phần Guardrails trong `03-bedrock/README.md`

**Thực hành:**
- Tạo một Bedrock Agent đơn giản với 1 Action Group
- Tạo Guardrail với denied topic và content filter
- Test Guardrail với prompt vi phạm

---

#### Ngày 21 — Mini Project: Chatbot Q&A với RAG

**Mini project:** Xây chatbot đọc tài liệu từ S3, trả lời câu hỏi, kèm citations

```python
# Simple RAG chatbot CLI
import boto3, json

bedrock_agent = boto3.client('bedrock-agent-runtime', region_name='us-east-1')
KB_ID = "your-knowledge-base-id"
MODEL_ARN = "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0"

print("RAG Chatbot — gõ 'exit' để thoát\n")
while True:
    question = input("Câu hỏi: ")
    if question.lower() == 'exit':
        break
    response = bedrock_agent.retrieve_and_generate(
        input={"text": question},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {"knowledgeBaseId": KB_ID, "modelArn": MODEL_ARN}
        }
    )
    print(f"\nTrả lời: {response['output']['text']}\n")
```

---

### Tuần 4 (Ngày 22-30): Tổng Hợp Giai Đoạn 1

#### Ngày 22-23 — Speech AI: Transcribe và Polly

**Đọc:**
- `06-speech/1-transcribe-fundamentals.md`
- `06-speech/4-polly.md`

**Thực hành:**
```python
# Transcribe audio file
transcribe = boto3.client('transcribe', region_name='us-east-1')
transcribe.start_transcription_job(
    TranscriptionJobName='my-test-job',
    Media={'MediaFileUri': 's3://your-bucket/audio.mp3'},
    MediaFormat='mp3',
    LanguageCode='vi-VN'  # Tiếng Việt
)

# Polly TTS — Text-to-Speech
polly = boto3.client('polly', region_name='us-east-1')
response = polly.synthesize_speech(
    Text="Xin chào, đây là Amazon Polly đang nói tiếng Việt.",
    OutputFormat='mp3',
    VoiceId='Lea'  # Vietnamese voice
)
with open('output.mp3', 'wb') as f:
    f.write(response['AudioStream'].read())
```

---

#### Ngày 24-25 — NLP: Comprehend Nâng Cao và Translate

**Đọc:**
- `05-nlp-text/2-comprehend-custom.md` — Custom Classification, Custom NER
- `05-nlp-text/4-translate.md`

**Thực hành:**
```python
# Amazon Translate
translate = boto3.client('translate', region_name='us-east-1')
result = translate.translate_text(
    Text="Machine Learning is transforming every industry.",
    SourceLanguageCode='en',
    TargetLanguageCode='vi'
)
print(result['TranslatedText'])
```

---

#### Ngày 26-27 — Ôn Tập Giai Đoạn 1

**Hoạt động:**
1. Xem lại tất cả ghi chú từ tuần 1-3
2. Tự trả lời 10 câu đầu trong `INTERVIEW_GUIDE.md` (không nhìn gợi ý)
3. Viết lại mô tả kiến trúc cho mỗi dịch vụ đã học (1-2 câu mỗi cái)

**Self-assessment (Tự Đánh Giá):**
- [ ] Gọi được Rekognition, Comprehend, Transcribe, Polly API không cần tra docs
- [ ] Giải thích RAG pipeline trong 2 phút
- [ ] Chạy được SageMaker training job và deploy endpoint
- [ ] Xây được chatbot đơn giản với Bedrock Knowledge Base

---

#### Ngày 28-30 — Portfolio Project 1: End-to-End AI Pipeline

**Project:** Document Analysis Pipeline (Đường Ống Phân Tích Tài Liệu)

```
PDF Upload → S3
     ↓
EventBridge → Lambda
     ↓
Textract (trích xuất text từ PDF)
     ↓
Comprehend (sentiment, entities, key phrases)
     ↓
DynamoDB (lưu kết quả)
     ↓
API Gateway → Lambda → Query kết quả
```

**Deliverables:**
- Code Python trong GitHub
- README mô tả architecture
- Ảnh chụp màn hình kết quả

---

## 🔧 GIAI ĐOẠN 2: CHUYÊN SÂU (Ngày 31-60)

### Tuần 5 (Ngày 31-37): MLOps — Vận Hành Mô Hình ML

**Mục tiêu tuần:** Hiểu và implement MLOps pipeline cơ bản

#### Ngày 31-33 — SageMaker Pipelines

**Đọc:**
- `09-mlops/1-sagemaker-pipelines.md` — DAG, Steps, CI/CD cho ML

**Thực hành:**
```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.parameters import ParameterFloat

accuracy_threshold = ParameterFloat(name="AccuracyThreshold", default_value=0.80)

# Tạo pipeline 3 bước: Preprocess → Train → Evaluate
pipeline = Pipeline(
    name="ml-interview-prep-pipeline",
    parameters=[accuracy_threshold],
    steps=[preprocessing_step, training_step, evaluation_step]
)

pipeline.upsert(role_arn=role)
execution = pipeline.start()
execution.wait()
```

---

#### Ngày 34 — Model Registry

**Đọc:**
- `09-mlops/2-model-registry.md`

**Thực hành:**
- Tạo Model Group
- Register model từ training job
- Xem approval workflow: PendingManualApproval → Approved

---

#### Ngày 35-36 — SageMaker Model Monitor

**Đọc:**
- `09-mlops/3-model-monitor.md` — Data quality, Model quality, Bias drift

**Thực hành:**
- Setup DataQualityMonitor cho một endpoint
- Tạo baseline từ training data
- Gửi test data có drift → xem violation report

---

#### Ngày 37 — SageMaker Clarify

**Đọc:**
- `09-mlops/4-sagemaker-clarify.md` — Bias detection, SHAP

**Thực hành:**
- Chạy Clarify bias analysis trên adult income dataset (public dataset)
- Xem SHAP feature importance report

---

### Tuần 6 (Ngày 38-44): Advanced AI Services

**Mục tiêu tuần:** Nắm vững các AI Services chuyên biệt

#### Ngày 38-39 — Conversational AI: Amazon Lex

**Đọc:**
- `07-conversational-ai/1-lex-fundamentals.md`
- `07-conversational-ai/2-lex-advanced.md`

**Thực hành:**
- Tạo Lex bot "HotelBooking" với intents: BookRoom, CheckAvailability, Cancel
- Cấu hình slots: check-in date, check-out date, room type
- Kết nối fulfillment Lambda
- Test trên Lex console

---

#### Ngày 40-41 — Amazon Transcribe Call Analytics

**Đọc:**
- `06-speech/2-transcribe-call-analytics.md`

**Thực hành:**
- Start Call Analytics job với một audio file giả lập cuộc gọi tổng đài
- Xem sentiment per turn, detected issues, action items

---

#### Ngày 42-44 — Predictions: Forecast và Personalize

**Đọc:**
- `08-predictions/1-forecast-fundamentals.md`
- `08-predictions/3-personalize-fundamentals.md`

**Thực hành:**
- Import Amazon Personalize sample dataset (MovieLens)
- Train Solution với recipe `aws-user-personalization`
- Tạo Campaign và lấy recommendations

---

### Tuần 7 (Ngày 45-51): Security, Cost, Architecture Nâng Cao

**Mục tiêu tuần:** Hiểu production-grade considerations

#### Ngày 45-46 — Security và Compliance

**Đọc:**
- `10-advanced/2-security-compliance.md` — VPC, IAM, Encryption

**Thực hành:**
- Tạo SageMaker Domain trong VPC
- Cấu hình Interface Endpoint cho SageMaker API
- Review IAM least privilege cho SageMaker execution role

---

#### Ngày 47-48 — Cost Optimization

**Đọc:**
- `10-advanced/1-cost-optimization.md` — Spot, Inf2, MME, Serverless

**Thực hành:**
- So sánh giá: on-demand vs Spot training trên AWS Pricing Calculator
- Deploy 2 models trên cùng Multi-Model Endpoint
- Cấu hình Auto Scaling cho endpoint

---

#### Ngày 49-50 — Responsible AI

**Đọc:**
- `10-advanced/3-responsible-ai.md` — Fairness, SHAP, Guardrails

**Thực hành:**
- Cấu hình Bedrock Guardrail với denied topics và PII redaction
- Test Guardrail với các prompts khác nhau

---

#### Ngày 51 — ML Architecture Patterns

**Đọc:**
- `10-advanced/4-ml-architecture-patterns.md` — Batch, Shadow, Canary, A/B

**Ghi chú:** Hiểu khi nào dùng từng pattern deployment

---

### Tuần 8 (Ngày 52-60): Tổng Hợp và Portfolio Project 2

#### Ngày 52-55 — Review và Gaps Analysis

**Hoạt động:**
1. Trả lời tất cả 20 câu hỏi trong `INTERVIEW_GUIDE.md` (ghi lại bằng văn bản)
2. Tự chấm — highlight những câu trả lời chưa tốt
3. Tìm gaps: dịch vụ nào còn mờ? Quay lại đọc section tương ứng

---

#### Ngày 56-60 — Portfolio Project 2: MLOps Pipeline Đầy Đủ

**Project:** Fraud Detection (Phát Hiện Gian Lận) với MLOps

```
Data (S3) → SageMaker Pipeline
                  ├── ProcessingStep (feature engineering)
                  ├── TrainingStep (XGBoost)
                  ├── EvaluationStep (calculate F1, AUC)
                  ├── ClarifyStep (bias check)
                  └── ConditionStep → RegisterModelStep
                                          ↓
                                   Model Registry
                                          ↓ (manual approve)
                                   Deploy Endpoint
                                          ↓
                                   Model Monitor (hourly)
```

**Deliverables:**
- SageMaker Pipeline chạy được end-to-end
- Clarify bias report
- Model Monitor baseline + schedule
- README giải thích kiến trúc và các decisions

---

## 🎯 GIAI ĐOẠN 3: CHUẨN BỊ PHỎNG VẤN (Ngày 61-90)

### Tuần 9 (Ngày 61-67): Luyện Câu Hỏi Kỹ Thuật

**Mục tiêu tuần:** Trả lời tự tin, đúng điểm, trong 2-3 phút mỗi câu

#### Ngày 61-62 — Nhóm Câu Hỏi SageMaker

**Luyện tập:**
- Đặt timer 3 phút, trả lời to, record lại (voice recorder)
- Câu 1: AI Services vs SageMaker
- Câu 3: Inference types
- Câu 7: Spot Training
- Câu 8: Feature Store
- Câu 15: Hyperparameter Tuning

**Evaluation (Tự Đánh Giá):**
- Nghe lại recording — có nêu trade-offs không? Có ví dụ thực tế không?

---

#### Ngày 63-64 — Nhóm Câu Hỏi Generative AI & Bedrock

**Luyện tập:**
- Câu 2: RAG vs Fine-tuning
- Câu 5: Foundation Model là gì
- Câu 6: RAG pipeline chi tiết
- Câu 9: Prompt Engineering
- Câu 13: Guardrails
- Câu 17: Vector Database

---

#### Ngày 65-66 — Nhóm Câu Hỏi MLOps & Architecture

**Luyện tập:**
- Câu 4: Model drift
- Câu 10: Clarify bias + SHAP
- Câu 12: MLOps pipeline
- Câu 18: SageMaker Pipelines vs Step Functions
- Câu 19: Cost optimization

---

#### Ngày 67 — Câu Hỏi So Sánh Dịch Vụ

**Luyện tập:**
- Câu 11: Rekognition vs Textract
- Câu 14: Lex vs Bedrock
- Câu 16: Responsible AI
- Câu 20: End-to-end Gen AI design

---

### Tuần 10 (Ngày 68-74): System Design Practice

**Mục tiêu tuần:** Thiết kế fluently trong 45 phút, communicate clearly

#### Ngày 68-69 — Kịch Bản 1 & 2

**Luyện tập:**
- Đặt timer 45 phút
- Làm kịch bản 1 (Recommendation System): clarify 5 phút, design 35 phút, trade-offs 5 phút
- Ngày hôm sau: kịch bản 2 (RAG Chatbot)
- Đọc lại `system-design-scenarios.md` để so sánh với solution của mình

---

#### Ngày 70-71 — Kịch Bản 3 & 4

- Kịch bản 3 (Content Moderation)
- Kịch bản 4 (Contact Center Analytics)

---

#### Ngày 72-73 — Kịch Bản 5 và Custom Scenarios

- Kịch bản 5 (MLOps Platform)
- Tự nghĩ ra kịch bản mới dựa trên industry của mình

---

#### Ngày 74 — Ôn Lại Và Điền Gaps

- Xem lại các điểm yếu từ tuần 9-10
- Đọc lại các section còn mờ

---

### Tuần 11 (Ngày 75-81): Câu Hỏi Hành Vi và STAR Stories

**Mục tiêu tuần:** Có ít nhất 3 câu chuyện STAR ready

#### Ngày 75-76 — Chuẩn Bị STAR Stories

**Template STAR:**

```markdown
## Story 1: [Tên Project]

**Situation (Tình Huống):**
"Ở [công ty/dự án], chúng tôi cần giải quyết [vấn đề cụ thể]
với scale [N users/requests/data size]..."

**Task (Nhiệm Vụ):**
"Tôi chịu trách nhiệm thiết kế/xây dựng [component cụ thể]
với yêu cầu [latency/accuracy/cost] trong [thời gian]..."

**Action (Hành Động):**
"Tôi đã:
1. Chọn [dịch vụ AWS] vì [lý do trade-off]
2. Implement [kỹ thuật] để giải quyết [challenge]
3. Kết hợp [A] và [B] vì [lý do kỹ thuật]"

**Result (Kết Quả):**
"Kết quả: [metric cụ thể — accuracy X%, latency giảm Y%, cost giảm Z%]
Bài học: [điều rút ra]"
```

**Chuẩn bị ít nhất 3 stories:**
1. Lần bạn chọn AI Service có sẵn thay vì custom model (và tại sao)
2. Lần bạn gặp vấn đề về performance/accuracy trong production
3. Lần bạn phải học dịch vụ mới nhanh để giải quyết deadline

---

#### Ngày 77-78 — Câu Hỏi Hành Vi Phổ Biến

**Luyện tập nói to — không đọc script:**

- "Kể về lần bạn đưa ra quyết định technical khó, phải chọn giữa nhiều options?"
- "Mô tả một lần model/system của bạn fail trong production. Bạn xử lý thế nào?"
- "Làm thế nào bạn keep up với AWS AI services luôn ra tính năng mới?"
- "Khi nào bạn không đồng ý với quyết định của team? Bạn làm gì?"

---

#### Ngày 79-81 — Câu Hỏi Ngược Lại Cho Interviewer

**Chuẩn bị câu hỏi thể hiện sự nghiêm túc:**

```
Về kỹ thuật:
- "Team đang sử dụng SageMaker Pipelines hay Step Functions cho ML orchestration?"
- "Có model monitoring setup không? Cách handle model drift như thế nào?"
- "Team scale như thế nào khi load tăng đột biến? Multi-region không?"

Về văn hóa:
- "Data science và engineering team collaborate thế nào trong MLOps?"
- "Có culture of experimentation không? Tỷ lệ experiments được ship to production?"

Về growth:
- "Cơ hội để lead một AI initiative hoặc architecture decision là gì?"
- "Team có budget để thử nghiệm dịch vụ AI mới không?"
```

---

### Tuần 12 (Ngày 82-88): Mock Interviews và Final Review

**Mục tiêu tuần:** Simulate điều kiện phỏng vấn thực tế

#### Ngày 82-83 — Mock Interview 1 (Technical)

**Setup:**
- Nhờ bạn bè hoặc đồng nghiệp đóng vai interviewer
- Hoặc tự record, đặt timer, trả lời như phỏng vấn thật
- 45 phút: 15 phút câu hỏi kỹ thuật + 30 phút system design

**Đề mẫu:**
- Câu hỏi kỹ thuật: RAG vs Fine-tuning, Model drift, Inference types
- System design: "Thiết kế content moderation cho video platform"

---

#### Ngày 84-85 — Mock Interview 2 (Behavioral)

**Setup:**
- 45 phút behavioral questions
- Focus: STAR format, cụ thể, có metrics

**Đề mẫu:**
- "Tell me about a time you had to quickly learn a new AWS service to solve a problem"
- "Describe your approach when a production ML model starts degrading"
- "How do you prioritize between model accuracy and inference cost?"

---

#### Ngày 86-87 — Final Gap Analysis

**Hoạt động:**
1. Đọc lại toàn bộ `INTERVIEW_GUIDE.md`
2. Highlight câu nào vẫn còn yếu → ôn lại section tương ứng
3. Review bảng trade-offs ở cuối INTERVIEW_GUIDE
4. Xem lại pricing của top 5 dịch vụ (biết approximate cost)

---

#### Ngày 88 — Checklists Cuối

**Technical Checklist:**
- [ ] Giải thích được RAG pipeline từ indexing đến response trong 3 phút
- [ ] Phân biệt 4 SageMaker inference types và use case từng loại
- [ ] Mô tả model drift detection với SageMaker Model Monitor
- [ ] Giải thích SHAP explainability trong 2 phút
- [ ] Design được chatbot architecture với Bedrock + OpenSearch
- [ ] Biết khi nào dùng Lex vs Bedrock, Rekognition vs Textract
- [ ] Ước tính chi phí xấp xỉ cho Bedrock (per token) và SageMaker (per hour)

**Behavioral Checklist:**
- [ ] Có 3 STAR stories ready — không cần đọc notes
- [ ] Câu hỏi ngược lại cho interviewer (ít nhất 5 câu)
- [ ] Biết cách trả lời khi không biết câu hỏi ("Tôi chưa dùng dịch vụ đó nhưng từ architecture tôi hiểu...")

---

### Tuần 13 (Ngày 89-90): Ngày Trước Phỏng Vấn

#### Ngày 89 — Nhẹ Nhàng Ôn Lại

**Chỉ làm:**
- Đọc bảng tóm tắt trade-offs trong `INTERVIEW_GUIDE.md` (5 phút)
- Review 5 câu hỏi khó nhất của bạn (15 phút)
- Xem lại 1 system design scenario (15 phút)
- **Không học thêm nội dung mới**

---

#### Ngày 90 — Ngày Phỏng Vấn

**Sáng sớm:**
- [ ] Đọc lại phần "Chiến Thuật Trả Lời" trong README.md
- [ ] Nhẩm lại 3 STAR stories
- [ ] Nhẹ nhàng xem lại trade-offs chính

**Trong phỏng vấn:**
- [ ] Luôn clarify requirements trước system design
- [ ] Nói rõ assumptions (giả định)
- [ ] Đề cập trade-offs chủ động
- [ ] Nói "it depends" rồi giải thích cả 2 options
- [ ] Mention monitoring và security trong mọi design

---

## 📊 Theo Dõi Tiến Độ

### Weekly Review Template (Mỗi Chủ Nhật)

```markdown
## Tuần [N] Review — [Ngày]

### Hoàn Thành
- [x] ...

### Chưa Hoàn Thành
- [ ] ... (lý do)

### Điểm Mạnh Tuần Này
- ...

### Điểm Cần Cải Thiện
- ...

### Kế Hoạch Tuần Tới
- ...

### Câu Hỏi / Chỗ Còn Mờ
- ...
```

### Metrics Theo Dõi

| Tuần | Số Câu Hỏi Trả Lời Tốt / 20 | Projects Hoàn Thành | Thời Gian Học (giờ) |
|------|------------------------------|---------------------|---------------------|
| 1    |                              |                     |                     |
| 2    |                              |                     |                     |
| 3    |                              |                     |                     |
| 4    |                              |                     |                     |
| 5    |                              |                     |                     |
| 6    |                              |                     |                     |
| 7    |                              |                     |                     |
| 8    |                              |                     |                     |
| 9    |                              |                     |                     |
| 10   |                              |                     |                     |
| 11   |                              |                     |                     |
| 12   |                              |                     |                     |

---

## 💡 Mẹo Học Hiệu Quả

### Học Chủ Động — Active Recall

- **Không highlight** — highlight tạo cảm giác đã học nhưng không giúp nhớ
- **Feynman Technique:** Sau khi đọc, đóng sách và giải thích lại bằng lời của bạn
- **Flashcards:** Anki cho thuật ngữ (mặt trước: term; mặt sau: giải thích + use case)
- **Teach someone:** Giải thích RAG cho một người không biết ML

### Làm Thực Hành Trước, Đọc Sau

- Thử gọi API → xem lỗi → đọc docs → hiểu sâu hơn
- Đừng chỉ đọc code mẫu — tự gõ lại và modify

### Quản Lý AWS Cost

- Đặt **Budget Alert** — CloudWatch billing alarm khi cost > $50/tháng
- Chạy training jobs vào buổi tối → kiểm tra sáng
- **Luôn delete SageMaker endpoint** sau khi test xong
- SageMaker Studio: shutdown JupyterLab khi không dùng

### Nếu Bạn Bị Stuck

1. Đọc lại section trong knowledge base (thường giải thích đầy đủ)
2. Thử AWS Documentation official (docs.aws.amazon.com)
3. Xem AWS re:Invent talks trên YouTube (search: "[service name] deep dive re:Invent 2024")
4. AWS re:Post community Q&A

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành — Kế hoạch 90 ngày đầy đủ
