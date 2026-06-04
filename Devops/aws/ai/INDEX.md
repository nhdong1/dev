# AWS Machine Learning & AI Services — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS AI/ML Services — từ nền tảng Machine Learning (Học Máy) đến Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh), MLOps (Vận Hành Mô Hình) và các AI Services quản lý hoàn toàn

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/ai/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình học & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/
│   ├── README.md                               ✅ Nền tảng AI/ML, khái niệm cốt lõi
│   ├── 1-ai-ml-overview.md                    ✅ AI vs ML vs DL vs Generative AI
│   ├── 2-ml-types.md                          ✅ Supervised, Unsupervised, Reinforcement
│   ├── 3-ml-workflow.md                       ✅ Data prep, Training, Evaluation, Deployment
│   ├── 4-aws-ai-layers.md                     ✅ Tầng AI Services, ML Services, Framework & Infra
│   └── 5-generative-ai-concepts.md           ✅ LLM, Foundation Model, Prompt, Token, RAG
│
├── 02-sagemaker/
│   ├── README.md                               ✅ Tổng quan SageMaker — ML Platform toàn diện
│   ├── 1-sagemaker-studio.md                  ✅ IDE, Domain, User Profile, JupyterLab, Canvas
│   ├── 2-sagemaker-training.md                ✅ Training jobs, Built-in algorithms, Spot training, AMT
│   ├── 3-sagemaker-inference.md               ✅ Real-time, Batch, Async, Serverless, MME, Auto Scaling
│   ├── 4-sagemaker-autopilot.md               ✅ AutoML, algorithm selection, HPO tự động, SHAP
│   └── 5-sagemaker-feature-store.md          ✅ Online/Offline store, Feature Group, Ingestion, Point-in-time
│
├── 03-bedrock/
│   ├── README.md                               ✅ Tổng quan Bedrock — Generative AI & Foundation Models
│   ├── 1-foundation-models.md                 ✅ Claude, Llama 3, Titan, Stable Diffusion, so sánh chi tiết
│   ├── 2-prompt-engineering.md                ✅ Zero-shot, Few-shot, Chain-of-thought, Templates
│   ├── 3-rag-knowledge-base.md                ✅ RAG, Bedrock Knowledge Base, Vector Store, OpenSearch
│   ├── 4-bedrock-agents.md                    ✅ Autonomous Agent, Tool use, Action Group, ReAct
│   └── 5-fine-tuning.md                      ✅ Continued pre-training, Instruction tuning, LoRA/PEFT
│
├── 04-computer-vision/
│   ├── README.md                               ✅ Tổng quan Computer Vision — Thị Giác Máy Tính trên AWS
│   ├── 1-rekognition-image.md                 ✅ Labels, Faces, Text, Moderation, PPE detection
│   ├── 2-rekognition-video.md                 ✅ Activities, Celebrities, Streaming video analysis
│   ├── 3-rekognition-custom-labels.md         ✅ Custom training, Auto-labeling, Model evaluation
│   └── 4-textract.md                         ✅ OCR, Analyze Document, Forms, Tables, Expense
│
├── 05-nlp-text/
│   ├── README.md                               ✅ Tổng quan NLP — Xử Lý Ngôn Ngữ Tự Nhiên trên AWS
│   ├── 1-comprehend-fundamentals.md           ✅ Sentiment, Entities, Key phrases, Language detection
│   ├── 2-comprehend-custom.md                 ✅ Custom Classification, Custom NER, training data
│   ├── 3-comprehend-medical.md                ✅ Medical entities, ICD-10-CM, RxNorm, PHI detection
│   └── 4-translate.md                        ✅ Real-time & batch translation, Custom terminology
│
├── 06-speech/
│   ├── README.md                               ✅ Tổng quan Speech AI — STT & TTS trên AWS
│   ├── 1-transcribe-fundamentals.md           ✅ Batch, Real-time, Custom Vocabulary, Custom Language Model
│   ├── 2-transcribe-call-analytics.md         ✅ Call analytics, Sentiment, Categories, Redaction
│   ├── 3-transcribe-medical.md                ✅ Medical transcription, Specialty models, PHI, HIPAA
│   └── 4-polly.md                            ✅ Neural TTS, SSML, Lexicons, Speech Marks, Caching
│
├── 07-conversational-ai/
│   ├── README.md                               ✅ Tổng quan Conversational AI — Chatbot & AI Assistant
│   ├── 1-lex-fundamentals.md                  ✅ Intent, Slot, Fulfillment, Multi-turn dialog
│   ├── 2-lex-advanced.md                      ✅ Custom slot types, Context, Lambda fulfillment
│   ├── 3-amazon-q-business.md                 ✅ Enterprise AI assistant, RAG nội bộ, Plugins
│   └── 4-amazon-q-developer.md              ✅ Code generation, Security scan, AWS knowledge
│
├── 08-predictions/
│   ├── README.md                               ✅ Tổng quan Prediction Services — Dự Báo & Gợi Ý
│   ├── 1-forecast-fundamentals.md             ✅ Dataset groups, Predictors, DeepAR+, AutoML
│   ├── 2-forecast-advanced.md                 ✅ What-if analysis, Explainability, Cold start
│   ├── 3-personalize-fundamentals.md          ✅ Interactions dataset, Recipe, Solution, Campaign
│   └── 4-lookout-anomaly-detection.md        ✅ Lookout for Metrics, Equipment, Vision
│
├── 09-mlops/
│   ├── README.md                               ✅ Tổng quan MLOps — Vận Hành Mô Hình ML trên AWS
│   ├── 1-sagemaker-pipelines.md               ✅ Pipeline steps, DAG, CI/CD cho ML, Triggers, Caching
│   ├── 2-model-registry.md                    ✅ Model versions, Approval workflow, Metadata, Lineage
│   ├── 3-model-monitor.md                     ✅ Data quality, Model quality, Bias drift, Feature attribution drift
│   ├── 4-sagemaker-clarify.md                 ✅ Bias detection, SHAP explainability, Pre/Post-training
│   └── 5-experiments-tracking.md             ✅ Experiment runs, Metrics, Artifacts, ML Lineage
│
├── 10-advanced/
│   ├── README.md                               ✅ Nâng cao: Kiến trúc, bảo mật, chi phí, Responsible AI
│   ├── 1-cost-optimization.md                 ✅ Spot training, Inf2 instances, Multi-model endpoint
│   ├── 2-security-compliance.md               ✅ VPC, Encryption, IAM, Private endpoint, Audit
│   ├── 3-responsible-ai.md                    ✅ Fairness, Transparency, Privacy, AWS AI principles
│   └── 4-ml-architecture-patterns.md         ✅ Batch inference, Online serving, Shadow deployment
│
├── 11-interview-prep/
│   ├── README.md                               ✅ Tổng quan chuẩn bị phỏng vấn, lộ trình, checklist
│   ├── INTERVIEW_GUIDE.md                      ✅ Top 20 câu hỏi phỏng vấn + gợi ý trả lời chi tiết
│   ├── system-design-scenarios.md             ✅ 5 kịch bản thiết kế hệ thống AI thực tế
│   └── 90-day-study-plan.md                  ✅ Kế hoạch học 90 ngày chi tiết theo ngày/tuần
```

---

## ✅ Đã Tạo

| Chủ Đề                                   | File                                                                         | Trạng Thái | Chất Lượng    |
| ---------------------------------------- | ---------------------------------------------------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**                 | README.md                                                                    | ✅         | Toàn diện     |
| **Chỉ Mục Đầy Đủ**                       | INDEX.md                                                                     | ✅         | Toàn diện     |
| **Nền Tảng AI/ML — Module Overview**     | 01-fundamentals/README.md                                                    | ✅         | Toàn diện     |
| **AI vs ML vs DL vs Generative AI**      | 01-fundamentals/1-ai-ml-overview.md                                          | ✅         | Toàn diện     |
| **Supervised, Unsupervised, RL**         | 01-fundamentals/2-ml-types.md                                                | ✅         | Toàn diện     |
| **ML Workflow: Data → Deploy → Monitor** | 01-fundamentals/3-ml-workflow.md                                             | ✅         | Toàn diện     |
| **AWS AI Ba Tầng Dịch Vụ**              | 01-fundamentals/4-aws-ai-layers.md                                           | ✅         | Toàn diện     |
| **LLM, Foundation Model, Prompt, RAG**   | 01-fundamentals/5-generative-ai-concepts.md                                  | ✅         | Toàn diện     |
| **SageMaker — ML Platform Overview**     | 02-sagemaker/README.md                                                       | ✅         | Toàn diện     |
| **SageMaker Studio, Domain, JupyterLab** | 02-sagemaker/1-sagemaker-studio.md                                           | ✅         | Toàn diện     |
| **Training Jobs, Algorithms, Spot**      | 02-sagemaker/2-sagemaker-training.md                                         | ✅         | Toàn diện     |
| **Real-time, Batch, Async, Serverless**  | 02-sagemaker/3-sagemaker-inference.md                                        | ✅         | Toàn diện     |
| **AutoML, Autopilot, SHAP**              | 02-sagemaker/4-sagemaker-autopilot.md                                        | ✅         | Toàn diện     |
| **Feature Store: Online/Offline**        | 02-sagemaker/5-sagemaker-feature-store.md                                    | ✅         | Toàn diện     |
| **Bedrock — Generative AI Overview**     | 03-bedrock/README.md                                                         | ✅         | Toàn diện     |
| **Foundation Models: Claude, Llama, Titan** | 03-bedrock/1-foundation-models.md                                         | ✅         | Toàn diện     |
| **Prompt Engineering: Zero/Few/CoT**     | 03-bedrock/2-prompt-engineering.md                                           | ✅         | Toàn diện     |
| **RAG & Bedrock Knowledge Base**         | 03-bedrock/3-rag-knowledge-base.md                                           | ✅         | Toàn diện     |
| **Bedrock Agents: Tool use, ReAct**      | 03-bedrock/4-bedrock-agents.md                                               | ✅         | Toàn diện     |
| **Fine-tuning: Instruction, LoRA/PEFT**  | 03-bedrock/5-fine-tuning.md                                                  | ✅         | Toàn diện     |
| **Computer Vision — Module Overview**    | 04-computer-vision/README.md                                                 | ✅         | Toàn diện     |
| **Rekognition Image: Labels, Faces, PPE**| 04-computer-vision/1-rekognition-image.md                                    | ✅         | Toàn diện     |
| **Rekognition Video: Stored & Streaming**| 04-computer-vision/2-rekognition-video.md                                    | ✅         | Toàn diện     |
| **Rekognition Custom Labels: Train & Eval** | 04-computer-vision/3-rekognition-custom-labels.md                         | ✅         | Toàn diện     |
| **Textract: OCR, Forms, Tables, Expense**| 04-computer-vision/4-textract.md                                             | ✅         | Toàn diện     |
| **NLP & Text AI — Module Overview**      | 05-nlp-text/README.md                                                        | ✅         | Toàn diện     |
| **Comprehend: Sentiment, Entities, Key phrases** | 05-nlp-text/1-comprehend-fundamentals.md                             | ✅         | Toàn diện     |
| **Comprehend Custom: Classification & NER** | 05-nlp-text/2-comprehend-custom.md                                        | ✅         | Toàn diện     |
| **Comprehend Medical: ICD-10, RxNorm, PHI** | 05-nlp-text/3-comprehend-medical.md                                       | ✅         | Toàn diện     |
| **Amazon Translate: Real-time & Batch**  | 05-nlp-text/4-translate.md                                                   | ✅         | Toàn diện     |
| **Speech AI — Module Overview**           | 06-speech/README.md                                                          | ✅         | Toàn diện     |
| **Transcribe: Batch, Streaming, CLM**    | 06-speech/1-transcribe-fundamentals.md                                       | ✅         | Toàn diện     |
| **Transcribe Call Analytics: Sentiment** | 06-speech/2-transcribe-call-analytics.md                                     | ✅         | Toàn diện     |
| **Transcribe Medical: PHI, HIPAA**       | 06-speech/3-transcribe-medical.md                                            | ✅         | Toàn diện     |
| **Polly: Neural TTS, SSML, Lexicons**    | 06-speech/4-polly.md                                                         | ✅         | Toàn diện     |
| **Conversational AI — Module Overview**  | 07-conversational-ai/README.md                                               | ✅         | Toàn diện     |
| **Lex: Intent, Slot, Fulfillment**       | 07-conversational-ai/1-lex-fundamentals.md                                   | ✅         | Toàn diện     |
| **Lex Advanced: Custom Slot, Context**   | 07-conversational-ai/2-lex-advanced.md                                       | ✅         | Toàn diện     |
| **Amazon Q Business: Enterprise RAG**    | 07-conversational-ai/3-amazon-q-business.md                                  | ✅         | Toàn diện     |
| **Amazon Q Developer: Code AI**          | 07-conversational-ai/4-amazon-q-developer.md                                 | ✅         | Toàn diện     |
| **Predictions — Module Overview**        | 08-predictions/README.md                                                     | ✅         | Toàn diện     |
| **Forecast: Dataset, Predictor, DeepAR+**| 08-predictions/1-forecast-fundamentals.md                                    | ✅         | Toàn diện     |
| **Forecast Advanced: What-if, Explain**  | 08-predictions/2-forecast-advanced.md                                        | ✅         | Toàn diện     |
| **Personalize: Recipe, Solution, Campaign** | 08-predictions/3-personalize-fundamentals.md                              | ✅         | Toàn diện     |
| **Lookout: Metrics, Equipment, Vision**  | 08-predictions/4-lookout-anomaly-detection.md                                | ✅         | Toàn diện     |
| **MLOps — Module Overview**              | 09-mlops/README.md                                                           | ✅         | Toàn diện     |
| **SageMaker Pipelines: DAG, CI/CD ML**   | 09-mlops/1-sagemaker-pipelines.md                                            | ✅         | Toàn diện     |
| **Model Registry: Versions, Approval**   | 09-mlops/2-model-registry.md                                                 | ✅         | Toàn diện     |
| **Model Monitor: Drift Detection**       | 09-mlops/3-model-monitor.md                                                  | ✅         | Toàn diện     |
| **Clarify: Bias & SHAP Explainability**  | 09-mlops/4-sagemaker-clarify.md                                              | ✅         | Toàn diện     |
| **Experiments: Tracking & ML Lineage**   | 09-mlops/5-experiments-tracking.md                                           | ✅         | Toàn diện     |
| **Advanced — Module Overview**           | 10-advanced/README.md                                                        | ✅         | Toàn diện     |
| **Cost Optimization: Spot, Inf2, MME**   | 10-advanced/1-cost-optimization.md                                           | ✅         | Toàn diện     |
| **Security: VPC, IAM, Encryption**       | 10-advanced/2-security-compliance.md                                         | ✅         | Toàn diện     |
| **Responsible AI: Fairness, SHAP, Privacy** | 10-advanced/3-responsible-ai.md                                           | ✅         | Toàn diện     |
| **ML Architecture Patterns: Batch, Shadow, Canary** | 10-advanced/4-ml-architecture-patterns.md                       | ✅         | Toàn diện     |
| **Interview Prep — Module Overview**         | 11-interview-prep/README.md                                               | ✅         | Toàn diện     |
| **Top 20 Câu Hỏi Phỏng Vấn AWS AI/ML**     | 11-interview-prep/INTERVIEW_GUIDE.md                                      | ✅         | Toàn diện     |
| **5 Kịch Bản System Design AI**             | 11-interview-prep/system-design-scenarios.md                              | ✅         | Toàn diện     |
| **Kế Hoạch Học 90 Ngày**                    | 11-interview-prep/90-day-study-plan.md                                    | ✅         | Toàn diện     |

---

## 🎯 Cần Tạo Theo Thứ Tự Ưu Tiên

### Ưu Tiên Cao — Core AI/ML Services

- [x] `01-fundamentals/README.md` — Nền tảng AI/ML: ML types, workflow, AWS AI layers ✅
- [x] `02-sagemaker/README.md` — SageMaker training, inference, Autopilot ✅
- [x] `02-sagemaker/1-sagemaker-studio.md` — Studio IDE, Domain, User Profile, JupyterLab ✅
- [x] `02-sagemaker/2-sagemaker-training.md` — Training Jobs, Built-in Algorithms, Spot, AMT ✅
- [x] `02-sagemaker/3-sagemaker-inference.md` — Real-time, Batch, Async, Serverless, MME ✅
- [x] `02-sagemaker/4-sagemaker-autopilot.md` — AutoML, HPO tự động, SHAP Explainability ✅
- [x] `02-sagemaker/5-sagemaker-feature-store.md` — Online/Offline Store, Feature Group, Ingestion ✅
- [x] `03-bedrock/README.md` — Tổng quan Bedrock, kiến trúc, Guardrails, Quick Start ✅
- [x] `03-bedrock/1-foundation-models.md` — Claude, Llama 3, Titan, Stable Diffusion, so sánh ✅
- [x] `03-bedrock/2-prompt-engineering.md` — Zero-shot, Few-shot, CoT, Templates, Injection ✅
- [x] `03-bedrock/3-rag-knowledge-base.md` — RAG pipeline, Knowledge Base, Vector Stores ✅
- [x] `03-bedrock/4-bedrock-agents.md` — Autonomous Agent, Action Group, ReAct, Memory ✅
- [x] `03-bedrock/5-fine-tuning.md` — Instruction tuning, Continued pre-training, LoRA/PEFT ✅
- [x] `04-computer-vision/README.md` — Rekognition Image, Video, Custom Labels, Textract ✅
- [x] `04-computer-vision/1-rekognition-image.md` — Labels, Faces, Text, Moderation, PPE ✅
- [x] `04-computer-vision/2-rekognition-video.md` — Stored & Streaming, Activities, Celebrities ✅
- [x] `04-computer-vision/3-rekognition-custom-labels.md` — Custom training, Evaluation, Precision/Recall ✅
- [x] `04-computer-vision/4-textract.md` — OCR, Forms, Tables, Expense, AnalyzeID ✅
- [x] `11-interview-prep/README.md` — Tổng quan, checklist, lộ trình ôn tập theo vai trò ✅
- [x] `11-interview-prep/INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn AWS AI/ML ✅
- [x] `11-interview-prep/system-design-scenarios.md` — 5 kịch bản thiết kế hệ thống AI ✅
- [x] `11-interview-prep/90-day-study-plan.md` — Kế hoạch học 90 ngày chi tiết ✅

### Ưu Tiên Trung Bình — Specialized AI Services

- [x] `05-nlp-text/README.md` — Comprehend, Comprehend Medical, Translate ✅
- [x] `05-nlp-text/1-comprehend-fundamentals.md` — Sentiment, Entities, Key phrases, Language detection ✅
- [x] `05-nlp-text/2-comprehend-custom.md` — Custom Classification, Custom NER, training data ✅
- [x] `05-nlp-text/3-comprehend-medical.md` — Medical entities, ICD-10-CM, RxNorm, PHI detection ✅
- [x] `05-nlp-text/4-translate.md` — Real-time & batch translation, Custom terminology ✅
- [x] `06-speech/README.md` — Transcribe, Transcribe Medical, Polly ✅
- [x] `06-speech/1-transcribe-fundamentals.md` — Batch, Real-time, Custom Vocabulary, Custom Language Model ✅
- [x] `06-speech/2-transcribe-call-analytics.md` — Call analytics, Sentiment, Categories, Redaction ✅
- [x] `06-speech/3-transcribe-medical.md` — Medical transcription, Specialty models, PHI, HIPAA ✅
- [x] `06-speech/4-polly.md` — Neural TTS, SSML, Lexicons, Speech Marks, Caching ✅
- [x] `07-conversational-ai/README.md` — Lex, Amazon Q Business, Amazon Q Developer ✅
- [x] `07-conversational-ai/1-lex-fundamentals.md` — Intent, Slot, Fulfillment, Multi-turn dialog ✅
- [x] `07-conversational-ai/2-lex-advanced.md` — Custom slot types, Context, Lambda fulfillment ✅
- [x] `07-conversational-ai/3-amazon-q-business.md` — Enterprise AI assistant, RAG nội bộ, Plugins ✅
- [x] `07-conversational-ai/4-amazon-q-developer.md` — Code generation, Security scan, AWS knowledge ✅
- [x] `09-mlops/README.md` — SageMaker Pipelines, Model Registry, Model Monitor ✅
- [x] `09-mlops/1-sagemaker-pipelines.md` — Pipeline DAG, Steps, CI/CD ML, Triggers ✅
- [x] `09-mlops/2-model-registry.md` — Model versions, Approval workflow, Metadata ✅
- [x] `09-mlops/3-model-monitor.md` — Data quality, Model quality, Bias drift, Feature attribution drift ✅
- [x] `09-mlops/4-sagemaker-clarify.md` — Bias detection, SHAP explainability, Pre/Post-training ✅
- [x] `09-mlops/5-experiments-tracking.md` — Experiment runs, Metrics, Artifacts, ML Lineage ✅

### Ưu Tiên Thấp — Advanced & Reference

- [x] `08-predictions/README.md` — Forecast, Personalize, Lookout ✅
- [x] `08-predictions/1-forecast-fundamentals.md` — Dataset groups, Predictors, DeepAR+, AutoML ✅
- [x] `08-predictions/2-forecast-advanced.md` — What-if analysis, Explainability, Cold start ✅
- [x] `08-predictions/3-personalize-fundamentals.md` — Interactions dataset, Recipe, Solution, Campaign ✅
- [x] `08-predictions/4-lookout-anomaly-detection.md` — Lookout for Metrics, Equipment, Vision ✅
- [x] `10-advanced/README.md` — Cost optimization, Security, Responsible AI ✅
- [x] `10-advanced/1-cost-optimization.md` — Spot, Inf2, MME, Serverless, Savings Plans ✅
- [x] `10-advanced/2-security-compliance.md` — VPC, IAM, Encryption, PII, Audit ✅
- [x] `10-advanced/3-responsible-ai.md` — Fairness, SHAP, Guardrails, Privacy ✅
- [x] `10-advanced/4-ml-architecture-patterns.md` — Batch, Shadow, Canary, A/B, Multi-region ✅
- [x] `11-interview-prep/system-design-scenarios.md` — Kịch bản thiết kế hệ thống AI ✅
- [ ] `GLOSSARY.md` — Thuật ngữ AI/ML
- [ ] `RESOURCES.md` — Tài liệu tham khảo

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Tự Học

```
1. Đọc README.md để nắm tổng quan
2. Chọn lộ trình (Beginner / Intermediate / Advanced)
3. Học từng module theo thứ tự 01 → 11
4. Thực hành hands-on trên AWS (nhớ xóa SageMaker endpoint sau khi xong)
5. Xây dựng mini AI project: ảnh → Rekognition → kết quả lưu DynamoDB
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md trước
2. Tập trung vào dịch vụ target role yêu cầu:
   - ML Engineer: 02-sagemaker, 09-mlops
   - AI App Developer: 03-bedrock, 04-computer-vision, 05-nlp-text, 07-conversational-ai
   - Data Scientist: 02-sagemaker (Studio, Autopilot, Experiments)
3. Chuẩn bị câu chuyện AI/ML project (STAR method)
4. Luyện giải thích trade-offs: AI Services vs Custom SageMaker, RAG vs Fine-tuning
```

### Cho Công Việc Thực Tế

```
Dùng như tài liệu tham khảo:
- Cần nhận diện ảnh/video: Xem 04-computer-vision/
- Cần phân tích văn bản: Xem 05-nlp-text/
- Cần xây chatbot: Xem 07-conversational-ai/
- Cần train custom model: Xem 02-sagemaker/
- Cần Generative AI app: Xem 03-bedrock/
- Cần MLOps pipeline: Xem 09-mlops/
```

### Cho System Design

```
1. Xác định yêu cầu: AI Service có sẵn đủ hay cần custom model?
2. Đọc So Sánh Nhanh trong README.md để chọn dịch vụ phù hợp
3. Thiết kế với nguyên tắc Well-Architected (bảo mật, reliability, cost)
4. Tham khảo 10-advanced/ cho security và cost optimization
```

---

## 📊 Ước Tính Thời Gian Học

| Module                                    | Thời Gian   | Độ Khó | Ưu Tiên  |
| ----------------------------------------- | ----------- | ------ | -------- |
| Fundamentals — Nền Tảng AI/ML             | 4-6 giờ     | ⭐     | Phải học |
| SageMaker — ML Platform                   | 10-15 giờ   | ⭐⭐⭐ | Phải học |
| Bedrock — Generative AI                   | 8-10 giờ    | ⭐⭐   | Phải học |
| Computer Vision — Rekognition & Textract  | 4-6 giờ     | ⭐⭐   | Phải học |
| NLP & Text — Comprehend & Translate       | 4-6 giờ     | ⭐⭐   | Nên học  |
| Speech AI — Transcribe & Polly            | 3-4 giờ     | ⭐     | Nên học  |
| Conversational AI — Lex & Amazon Q        | 4-6 giờ     | ⭐⭐   | Nên học  |
| Predictions — Forecast & Personalize      | 6-8 giờ     | ⭐⭐⭐ | Tùy chọn |
| MLOps — Pipelines & Monitoring            | 8-10 giờ    | ⭐⭐⭐ | Phải học |
| Advanced — Cost, Security, Responsible AI | 4-6 giờ     | ⭐⭐   | Nên học  |

**Tổng: 55-80 giờ cho kiến thức AWS AI/ML toàn diện**

---

## 🎓 Mức Độ Kỹ Năng Được Hỗ Trợ

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Giải thích được AI, Machine Learning (Học Máy), Deep Learning (Học Sâu) khác nhau thế nào
- [ ] Gọi Rekognition API (detect-labels) thành công qua AWS Console hoặc CLI
- [ ] Dùng Amazon Comprehend để phân tích cảm xúc (sentiment) của đoạn văn bản tiếng Anh
- [ ] Biết Foundation Model (Mô Hình Nền Tảng) là gì và các model nào có trên Bedrock
- [ ] Hiểu SageMaker dùng để làm gì và khi nào không cần SageMaker

**Thời gian đạt mức này:** 2-4 tuần

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Huấn luyện (train) model XGBoost với SageMaker built-in algorithm trên S3 data
- [ ] Triển khai (deploy) SageMaker endpoint và gọi từ Python Lambda function
- [ ] Xây dựng RAG pipeline (Tăng Cường Truy Xuất) đơn giản với Bedrock + S3 Knowledge Base
- [ ] Tạo Lex chatbot với 3 intents và tích hợp Fulfillment qua Lambda
- [ ] Thiết kế SageMaker Pipeline (Đường Ống ML) 3 bước: Preprocess → Train → Register

**Thời gian đạt mức này:** 2-3 tháng thực hành

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc MLOps platform (Nền Tảng MLOps) với auto-retraining và model drift monitoring
- [ ] Fine-tune (Tinh Chỉnh) Foundation Model trên Bedrock với proprietary dataset
- [ ] Thiết kế multi-model endpoint (Endpoint Đa Mô Hình) SageMaker cho tiết kiệm chi phí
- [ ] Triển khai Responsible AI framework với SageMaker Clarify bias detection
- [ ] Thiết kế kiến trúc Generative AI application production-ready với guardrails và monitoring

**Thời gian đạt mức này:** Học liên tục, 6-12 tháng kinh nghiệm thực chiến

---

## 🔗 Điều Hướng Nhanh

| Cần Gì                                    | Vị Trí                                                            |
| ----------------------------------------- | ----------------------------------------------------------------- |
| Tổng quan nhanh                           | [README.md](README.md)                                            |
| Nền tảng AI/ML                            | [01-fundamentals/README.md](01-fundamentals/README.md)            |
| Train & Deploy custom model               | [02-sagemaker/README.md](02-sagemaker/README.md)                  |
| Generative AI & LLM app                  | [03-bedrock/README.md](03-bedrock/README.md)                      |
| Nhận diện ảnh/video                       | [04-computer-vision/README.md](04-computer-vision/README.md)      |
| Phân tích văn bản & NLP                   | [05-nlp-text/README.md](05-nlp-text/README.md)                    |
| Chuyển đổi giọng nói & văn bản            | [06-speech/README.md](06-speech/README.md)                        |
| Xây dựng chatbot / AI assistant           | [07-conversational-ai/README.md](07-conversational-ai/README.md)  |
| Dự báo & gợi ý cá nhân hóa               | [08-predictions/README.md](08-predictions/README.md)              |
| MLOps & model monitoring                  | [09-mlops/README.md](09-mlops/README.md)                          |
| Câu hỏi phỏng vấn                         | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS AI/ML — Tiến Độ Của Tôi

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)
- [ ] AI vs ML vs Deep Learning vs Generative AI
- [ ] Machine Learning types (Supervised, Unsupervised, Reinforcement)
- [ ] ML Workflow (Data → Train → Evaluate → Deploy)
- [ ] AWS AI tầng dịch vụ: AI Services vs SageMaker
- [ ] Foundation Model, LLM, Prompt, Token là gì

### Giai Đoạn 2: Dịch Vụ Cốt Lõi (Tuần 3-6)
- [ ] SageMaker Training Job (custom + built-in algorithms)
- [ ] SageMaker Endpoint deployment (Real-time, Batch, Serverless)
- [ ] Bedrock Foundation Models (Claude, Titan, Llama)
- [ ] Bedrock RAG với Knowledge Base
- [ ] Rekognition Image / Video API
- [ ] Comprehend Sentiment + Entity detection

### Giai Đoạn 3: AI Services Chuyên Biệt (Tuần 7-10)
- [ ] Amazon Transcribe (batch + real-time)
- [ ] Amazon Polly (Neural TTS + SSML)
- [ ] Amazon Translate (real-time + batch)
- [ ] Amazon Textract (Document + Form + Table)
- [ ] Amazon Lex (Intent + Slot + Fulfillment)

### Giai Đoạn 4: MLOps & Nâng Cao (Tuần 11+)
- [ ] SageMaker Pipelines (CI/CD cho ML)
- [ ] SageMaker Model Monitor (Data drift, Model drift)
- [ ] SageMaker Clarify (Bias + Explainability)
- [ ] Bedrock Fine-tuning
- [ ] Responsible AI (Fairness, Transparency, Privacy)
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn phải có khả năng:

### ✅ Năng Lực Nền Tảng

- [ ] Chọn đúng dịch vụ AI AWS cho từng use case (AI Service có sẵn vs Custom SageMaker)
- [ ] Giải thích được RAG (Retrieval-Augmented Generation) và khi nào nên dùng Bedrock
- [ ] Ước tính chi phí cho một AI/ML workload trên AWS
- [ ] Thiết kế data flow đơn giản từ input → AI processing → output cho một bài toán thực tế

### ✅ Năng Lực Vận Hành

- [ ] Xử lý SageMaker training job failure (lỗi training)
- [ ] Phát hiện và xử lý model drift (Trôi Dạt Mô Hình) với SageMaker Model Monitor
- [ ] Debug Bedrock API throttling (Hạn Chế Tốc Độ) và latency issues
- [ ] Tối ưu chi phí SageMaker: chọn đúng instance, dùng Spot, multi-model endpoint

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời Top 20 câu hỏi phỏng vấn AWS AI/ML tự tin
- [ ] Thiết kế end-to-end AI pipeline khi được hỏi trong vòng 30 phút
- [ ] Giải thích trade-offs: AI Services vs SageMaker, RAG vs Fine-tuning, batch vs real-time inference
- [ ] Biết ít nhất 2 câu chuyện dự án AI/ML thực tế (STAR format)

---

## 🚀 Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ để nắm toàn cảnh AWS AI/ML
2. Chọn lộ trình học theo vai trò mục tiêu (ML Engineer / AI Developer / Data Scientist)
3. Tạo AWS Free Tier account nếu chưa có
4. Thử gọi Rekognition detect-labels API với một bức ảnh bất kỳ

### Ngắn Hạn (2-4 Tuần Tới)

1. Hoàn thành `01-fundamentals/` để vững nền tảng AI/ML
2. Học `02-sagemaker/` và chạy training job đầu tiên trên SageMaker
3. Khám phá Bedrock Playground: so sánh Claude vs Titan vs Llama 2
4. Thực hành `04-computer-vision/` — detect objects trong ảnh thực tế

### Trung Hạn (1-3 Tháng)

1. Hoàn thành tất cả core modules (01-07)
2. Xây dựng portfolio project: end-to-end AI application (ví dụ: RAG chatbot với Bedrock)
3. Học MLOps trong `09-mlops/` để biết deploy model production
4. Bắt đầu luyện câu hỏi phỏng vấn trong `11-interview-prep/`

### Dài Hạn (3-6 Tháng)

1. Nắm vững toàn bộ AWS AI/ML ecosystem
2. Thực hành trên dữ liệu thực tế và dự án thực chiến
3. Lấy chứng chỉ **AWS Certified Machine Learning — Specialty** hoặc **AWS Certified AI Practitioner**
4. Đóng góp vào open-source AI projects (Hugging Face, LangChain, v.v.)

---

## 💡 Mẹo Thực Hành

1. **Xóa SageMaker endpoint ngay sau khi xong:** Endpoint tính phí theo giờ kể cả khi không có traffic — đây là nguồn phát sinh chi phí lớn nhất
2. **Dùng AWS Free Tier thông minh:** Rekognition (5.000 ảnh/tháng), Comprehend (50.000 đơn vị/tháng), Transcribe (60 phút/tháng), Polly (5 triệu ký tự/tháng) đều có free tier
3. **Học bằng cách làm:** Đừng chỉ đọc — hãy thực sự gọi API với dữ liệu thực
4. **Hiểu chi phí mỗi dịch vụ:** SageMaker training tính phí instance/giờ; Bedrock tính phí per token — kiểm tra trước khi chạy large-scale
5. **Kết hợp dịch vụ:** Sức mạnh của AWS AI nằm ở tích hợp: S3 → Rekognition → Comprehend → DynamoDB là pipeline điển hình
6. **Kiểm tra Well-Architected:** Dùng AWS Machine Learning Lens để đánh giá kiến trúc AI của bạn
7. **Cẩn thận với Responsible AI:** Luôn test model cho bias và fairness trước khi production — dùng SageMaker Clarify

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Knowledge base này là tài liệu sống. Đóng góp được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho dịch vụ AI mới (AWS ra mắt dịch vụ AI mới liên tục)
- [ ] Ví dụ code Python boto3 thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm ML phức tạp
- [ ] Bổ sung hands-on exercises (bài tập thực hành) với từng dịch vụ

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 2.1
**Trạng Thái:** ✅ README.md & INDEX.md hoàn thành | ✅ 01-fundamentals hoàn thành (6 files) | ✅ 02-sagemaker hoàn thành (6 files) | ✅ 03-bedrock hoàn thành (6 files) | ✅ 04-computer-vision hoàn thành (5 files) | ✅ 05-nlp-text hoàn thành (5 files) | ✅ 06-speech hoàn thành (5 files) | ✅ 07-conversational-ai hoàn thành (5 files) | ✅ 08-predictions hoàn thành (5 files) | ✅ 09-mlops hoàn thành (6 files) | ✅ 10-advanced hoàn thành (5 files) | ✅ 11-interview-prep hoàn thành (4 files)
