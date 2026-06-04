# 🤖 AWS Machine Learning & AI Services — Lộ Trình Học Toàn Diện

> Hướng dẫn đầy đủ về AWS AI/ML Services — từ nền tảng Machine Learning (Học Máy), Amazon SageMaker (Nền Tảng ML Toàn Diện), Amazon Bedrock (Generative AI — Trí Tuệ Nhân Tạo Tạo Sinh) đến các AI Services được quản lý hoàn toàn như Rekognition (Nhận Diện Hình Ảnh), Comprehend (Xử Lý Ngôn Ngữ Tự Nhiên), Polly (Chuyển Văn Bản Thành Giọng Nói), Transcribe (Chuyển Giọng Nói Thành Văn Bản) và các chiến lược MLOps (ML Operations — Vận Hành Mô Hình Học Máy) hiện đại.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Dịch Vụ](#tổng-quan-các-dịch-vụ)
4. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
5. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
6. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng — Fundamentals (Tuần 1-2)**

- [ ] Machine Learning (Học Máy) — Supervised, Unsupervised, Reinforcement Learning
- [ ] Deep Learning (Học Sâu) — Neural Networks (Mạng Thần Kinh Nhân Tạo), CNN, RNN, Transformer
- [ ] ML Workflow (Quy Trình Học Máy) — Data Prep, Training, Evaluation, Deployment
- [ ] AWS AI Strategy — Tầng dịch vụ AI: AI Services, ML Services, ML Framework & Infrastructure
- [ ] Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh) — LLM (Large Language Model — Mô Hình Ngôn Ngữ Lớn), Foundation Model (Mô Hình Nền Tảng)

### **Giai Đoạn 2: Dịch Vụ Cốt Lõi — Core Services (Tuần 3-6)**

- [ ] Amazon SageMaker — Training (Huấn Luyện), Tuning (Tinh Chỉnh), Deployment (Triển Khai)
- [ ] Amazon Bedrock — Foundation Models, Prompt Engineering (Kỹ Thuật Lập Trình Prompt), RAG
- [ ] Amazon Rekognition (Nhận Diện Hình Ảnh & Video) — Object Detection, Facial Analysis
- [ ] Amazon Comprehend (Hiểu Ngôn Ngữ Tự Nhiên) — Sentiment Analysis, Entity Recognition

### **Giai Đoạn 3: Dịch Vụ AI Chuyên Biệt — Specialized AI (Tuần 7-10)**

- [ ] Amazon Polly (Chuyển Văn Bản Thành Giọng Nói) — TTS, SSML, Neural Voice
- [ ] Amazon Transcribe (Chuyển Giọng Nói Thành Văn Bản) — STT, Custom Vocabulary
- [ ] Amazon Translate (Dịch Thuật Máy Thần Kinh) — Neural Machine Translation
- [ ] Amazon Textract (Trích Xuất Tài Liệu) — OCR, Form & Table Extraction
- [ ] Amazon Lex (AI Đàm Thoại) — Chatbot, Slot Filling, Intent Recognition
- [ ] Amazon Kendra (Tìm Kiếm Thông Minh) — Intelligent Enterprise Search

### **Giai Đoạn 4: Chuyên Sâu — Specialization (Tuần 11+)**

- [ ] MLOps (ML Operations — Vận Hành Mô Hình) — SageMaker Pipelines, Model Registry, CI/CD
- [ ] Amazon Forecast (Dự Báo Chuỗi Thời Gian) — AutoML, Time Series Forecasting
- [ ] Amazon Personalize (Hệ Thống Gợi Ý) — Recommendation Engine
- [ ] Responsible AI (AI Có Trách Nhiệm) — Bias Detection (Phát Hiện Thiên Kiến), Explainability (Khả Năng Giải Thích)

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                          | Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------------------- | ------- | --------- | ---------- |
| **SageMaker — Train & Deploy ML Models**          | ⭐⭐⭐  | 3 tuần    | -          |
| **Bedrock — Generative AI & LLM**                 | ⭐⭐⭐  | 2 tuần    | -          |
| **Rekognition — Computer Vision**                 | ⭐⭐⭐  | 1 tuần    | -          |
| **Comprehend — NLP & Text Analysis**              | ⭐⭐⭐  | 1 tuần    | -          |
| **Transcribe & Polly — Speech AI**                | ⭐⭐    | 1 tuần    | -          |
| **Textract — Document AI**                        | ⭐⭐    | 1 tuần    | -          |
| **Lex — Conversational AI**                       | ⭐⭐    | 1 tuần    | -          |
| **MLOps — SageMaker Pipelines**                   | ⭐⭐⭐  | 2 tuần    | -          |
| **Forecast & Personalize — Predictions**          | ⭐⭐    | 2 tuần    | -          |
| **Responsible AI — Bias & Explainability**        | ⭐⭐    | 1 tuần    | -          |

---

## 🗂️ Tổng Quan Các Dịch Vụ

### So Sánh Nhanh — Chọn Dịch Vụ Phù Hợp

| Dịch Vụ                    | Loại                          | Tốt Nhất Cho                                        | Mô Hình Tính Phí              |
| -------------------------- | ----------------------------- | --------------------------------------------------- | ----------------------------- |
| **SageMaker**              | ML Platform                   | Custom ML model training & deployment               | Instance giờ + Storage        |
| **Bedrock**                | Generative AI / Foundation AI | Ứng dụng AI tạo sinh, LLM, RAG pipeline             | Tokens đầu vào/ra             |
| **Rekognition**            | Computer Vision               | Nhận diện khuôn mặt, phát hiện object, video        | Ảnh/video phân tích           |
| **Comprehend**             | NLP                           | Phân tích cảm xúc, phân loại văn bản, NER           | Ký tự phân tích               |
| **Transcribe**             | Speech-to-Text                | Chuyển âm thanh thành văn bản, call analytics       | Giây âm thanh                 |
| **Polly**                  | Text-to-Speech                | Tổng hợp giọng nói từ văn bản                        | Ký tự tổng hợp                |
| **Translate**              | Machine Translation           | Dịch đa ngôn ngữ tự động theo thời gian thực        | Ký tự dịch                    |
| **Textract**               | Document AI / OCR             | Trích xuất text, form, bảng từ tài liệu scan        | Trang tài liệu                |
| **Lex**                    | Conversational AI             | Chatbot, voice assistant, IVR                       | Request text/voice            |
| **Kendra**                 | Intelligent Search            | Tìm kiếm doanh nghiệp thông minh                    | Index hour + Query            |
| **Forecast**               | Time Series AI                | Dự báo nhu cầu, tồn kho, tài chính                  | Data points + forecast        |
| **Personalize**            | Recommendation AI             | Gợi ý sản phẩm, nội dung cá nhân hóa               | TPS provisioned + event       |
| **Amazon Q**               | Generative AI Assistant       | AI trợ lý doanh nghiệp, code, tài liệu             | User/month                    |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. Fundamentals — Nền Tảng AI/ML** (`01-fundamentals/`)

- AI/ML Overview (Tổng Quan AI/ML) — Phân biệt AI, ML, Deep Learning, Generative AI
- Machine Learning Types (Các Loại Học Máy) — Supervised (Có Giám Sát), Unsupervised (Không Giám Sát), Reinforcement (Học Tăng Cường)
- ML Workflow (Quy Trình Học Máy) — Data Collection, Feature Engineering, Training, Evaluation, Deployment
- AWS AI Layers (Các Tầng AI Của AWS) — AI Services, ML Services, Framework & Infrastructure
- Generative AI Concepts (Khái Niệm AI Tạo Sinh) — LLM (Large Language Model), Foundation Model, Prompt, Token

### 📁 **2. Amazon SageMaker — ML Platform** (`02-sagemaker/`)

- **SageMaker Studio** (Studio Học Máy) — IDE cho toàn bộ ML workflow
- **SageMaker Training** (Huấn Luyện SageMaker) — Built-in algorithms, custom containers, Spot training
- **SageMaker Inference** (Triển Khai Dự Đoán) — Real-time endpoint, Batch transform, Async inference, Serverless
- **SageMaker Autopilot** (Học Máy Tự Động) — AutoML, tự động chọn thuật toán và tinh chỉnh siêu tham số
- **SageMaker Feature Store** (Kho Đặc Trưng) — Quản lý và tái sử dụng features giữa các model

### 📁 **3. Amazon Bedrock — Generative AI** (`03-bedrock/`)

- **Foundation Models (Mô Hình Nền Tảng)** — Claude, Llama 2, Titan, Stable Diffusion, Jurassic-2
- **Prompt Engineering (Kỹ Thuật Lập Trình Prompt)** — Zero-shot, Few-shot, Chain-of-thought
- **RAG — Retrieval-Augmented Generation (Tạo Sinh Tăng Cường Truy Xuất)** — Knowledge base, vector search
- **Amazon Bedrock Agents** (Tác Nhân Bedrock) — Autonomous agent, tool use, multi-step reasoning
- **Fine-tuning (Tinh Chỉnh Mô Hình)** — Continued pre-training, instruction tuning với dữ liệu của bạn

### 📁 **4. Computer Vision — Thị Giác Máy Tính** (`04-computer-vision/`)

- **Amazon Rekognition Image** — Object detection (Phát Hiện Đối Tượng), Label detection, Face comparison
- **Amazon Rekognition Video** — Activity detection (Phát Hiện Hoạt Động), Celebrity recognition, Content moderation
- **Amazon Rekognition Custom Labels** — Huấn luyện model nhận diện đối tượng tùy chỉnh
- **Amazon Textract** (Trích Xuất Tài Liệu) — OCR, Form fields, Table extraction, Expense analysis

### 📁 **5. NLP & Text AI — Xử Lý Ngôn Ngữ Tự Nhiên** (`05-nlp-text/`)

- **Amazon Comprehend** (Hiểu Ngôn Ngữ) — Sentiment analysis (Phân Tích Cảm Xúc), Entity recognition (Nhận Dạng Thực Thể), Key phrase extraction
- **Amazon Comprehend Medical** (Comprehend Y Tế) — Trích xuất thực thể y tế, ICD-10-CM/RxNorm
- **Amazon Translate** (Dịch Thuật Máy Thần Kinh) — Real-time & batch translation, custom terminology
- **Comprehend Custom Classification** (Phân Loại Tùy Chỉnh) — Huấn luyện bộ phân loại văn bản riêng

### 📁 **6. Speech AI — AI Giọng Nói** (`06-speech/`)

- **Amazon Transcribe** (Nhận Dạng Giọng Nói) — Real-time & batch STT, Custom Vocabulary, Speaker diarization
- **Amazon Transcribe Medical** (Transcribe Y Tế) — Phiên âm y tế chuyên biệt
- **Amazon Transcribe Call Analytics** — Phân tích cuộc gọi tổng đài, sentiment, issue detection
- **Amazon Polly** (Chuyển Văn Bản Thành Giọng Nói) — Standard & Neural TTS, SSML (Speech Synthesis Markup Language), Lexicons

### 📁 **7. Conversational AI — AI Đàm Thoại** (`07-conversational-ai/`)

- **Amazon Lex** (Xây Dựng Chatbot) — Intent (Mục Đích), Slot (Khe), Fulfillment, Multi-turn conversation
- **Amazon Lex với Connect** — Contact Center AI, IVR (Interactive Voice Response) thông minh
- **Amazon Q Business** (Trợ Lý AI Doanh Nghiệp) — Enterprise AI assistant, RAG trên dữ liệu nội bộ
- **Amazon Q Developer** (Trợ Lý AI Lập Trình) — Code generation (Tạo Mã), Code review, Security scan

### 📁 **8. Predictions — Dự Báo & Gợi Ý** (`08-predictions/`)

- **Amazon Forecast** (Dự Báo Chuỗi Thời Gian) — AutoML forecasting, DeepAR+, Predictor, Explainability
- **Amazon Personalize** (Hệ Thống Gợi Ý Cá Nhân Hóa) — User-item interactions, Recipe (Công Thức), Campaign (Chiến Dịch)
- **Amazon Lookout for Metrics** (Phát Hiện Bất Thường Số Liệu) — Anomaly detection không cần ML expertise
- **Amazon Lookout for Equipment** (Phát Hiện Bất Thường Thiết Bị) — Predictive maintenance (Bảo Trì Dự Đoán)

### 📁 **9. MLOps — Vận Hành Mô Hình ML** (`09-mlops/`)

- **SageMaker Pipelines** (Đường Ống ML) — CI/CD cho ML, bước Data Processing, Training, Evaluation, Deployment
- **SageMaker Model Registry** (Kho Mô Hình) — Versioning, approval workflow, metadata
- **SageMaker Model Monitor** (Giám Sát Mô Hình) — Data drift (Trôi Dạt Dữ Liệu), concept drift, bias detection
- **SageMaker Clarify** (Giải Thích AI) — Feature importance, bias analysis, explainability report
- **SageMaker Experiments** (Thí Nghiệm ML) — Tracking runs, metrics, artifacts

### 📁 **10. Advanced & Architecture — Nâng Cao & Kiến Trúc** (`10-advanced/`)

- **Cost Optimization (Tối Ưu Chi Phí)** — SageMaker Spot training, Inference Recommender, right-sizing
- **Security & Compliance (Bảo Mật & Tuân Thủ)** — VPC isolation, encryption, IAM, SageMaker role ARN
- **Responsible AI (AI Có Trách Nhiệm)** — Fairness (Sự Công Bằng), Transparency (Minh Bạch), Privacy (Quyền Riêng Tư)
- **ML Architecture Patterns** — Batch inference, real-time serving, hybrid multi-model endpoints

### 📁 **11. Interview Prep — Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 20 AWS AI/ML Interview Questions (20 Câu Hỏi Phỏng Vấn Hàng Đầu)
- System Design Scenarios (Kịch Bản Thiết Kế Hệ Thống AI) — Recommendation, NLP pipeline, Generative AI app
- STAR Stories (Câu Chuyện Theo Phương Pháp STAR) — ML project & incident templates
- Kế Hoạch Học 90 Ngày có lộ trình chi tiết

---

## 🎓 Theo Dịch Vụ AI/ML

### **Amazon SageMaker — ML Platform**

```
Điểm Mạnh: End-to-end ML workflow, built-in algorithms, managed infrastructure
Phù Hợp Cho: Custom model training, hyperparameter tuning, production deployment
Học Ở: 02-sagemaker/, 09-mlops/
```

### **Amazon Bedrock — Generative AI**

```
Điểm Mạnh: Truy cập nhiều Foundation Models qua API, không quản lý hạ tầng
Phù Hợp Cho: Chatbot, content generation, RAG-based Q&A, AI agents
Học Ở: 03-bedrock/, 07-conversational-ai/
```

### **Amazon Rekognition — Computer Vision**

```
Điểm Mạnh: Dịch vụ hoàn toàn managed, không cần ML expertise
Phù Hợp Cho: Content moderation, face verification, video surveillance
Học Ở: 04-computer-vision/
```

### **Amazon Comprehend — NLP**

```
Điểm Mạnh: Pre-trained NLP models, hỗ trợ nhiều ngôn ngữ, custom training
Phù Hợp Cho: Customer feedback analysis, document classification, medical records
Học Ở: 05-nlp-text/
```

### **Speech AI — Transcribe & Polly**

```
Điểm Mạnh: High accuracy STT/TTS, custom vocabulary & voice, streaming support
Phù Hợp Cho: Contact center, voice apps, subtitles, accessibility
Học Ở: 06-speech/
```

### **MLOps — SageMaker Pipelines**

```
Điểm Mạnh: Native tích hợp toàn bộ SageMaker, visual pipeline editor, model monitoring
Phù Hợp Cho: Production ML deployment, automated retraining, governance
Học Ở: 09-mlops/
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                          | Thư Mục                                                      | Ưu Tiên         |
| ------------------------------- | ------------------------------------------------------------ | --------------- |
| Bắt đầu từ đây                  | [README.md](./README.md)                                     | Start here      |
| Nền tảng AI/ML                  | [01-fundamentals/](./01-fundamentals/)                       | Nền tảng        |
| ML Platform tổng hợp            | [02-sagemaker/](./02-sagemaker/)                             | Quan trọng      |
| Generative AI & LLM             | [03-bedrock/](./03-bedrock/)                                 | Quan trọng      |
| Computer Vision                 | [04-computer-vision/](./04-computer-vision/)                 | Thực dụng       |
| NLP & Text Analysis             | [05-nlp-text/](./05-nlp-text/)                               | Thực dụng       |
| Speech AI                       | [06-speech/](./06-speech/)                                   | Thực dụng       |
| Conversational AI               | [07-conversational-ai/](./07-conversational-ai/)             | Nâng cao        |
| Predictions & Recommendations   | [08-predictions/](./08-predictions/)                         | Nâng cao        |
| MLOps & Model Management        | [09-mlops/](./09-mlops/)                                     | Quan trọng      |
| Câu hỏi phỏng vấn               | [11-interview-prep/INTERVIEW_GUIDE.md](./11-interview-prep/) | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Giải thích sự khác biệt giữa AI, Machine Learning (Học Máy) và Deep Learning (Học Sâu)
- [ ] Gọi Amazon Rekognition API để phát hiện nhãn trong một bức ảnh
- [ ] Dùng Amazon Comprehend để phân tích cảm xúc (sentiment) của đoạn văn bản
- [ ] Sử dụng Amazon Transcribe để chuyển file âm thanh thành văn bản
- [ ] Hiểu Foundation Model (Mô Hình Nền Tảng) là gì và Bedrock cung cấp những model nào

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Huấn luyện (train) một custom model trên SageMaker với built-in algorithm
- [ ] Triển khai (deploy) SageMaker real-time endpoint và tích hợp vào ứng dụng
- [ ] Xây dựng RAG pipeline (Tăng Cường Truy Xuất) với Bedrock Knowledge Base và vector store
- [ ] Tạo Lex chatbot với intents (mục đích), slots (khe) và multi-turn dialog
- [ ] Thiết kế SageMaker Pipeline (Đường Ống ML) với bước training và deployment tự động

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc MLOps platform (Nền Tảng MLOps) end-to-end với model monitoring và retraining
- [ ] Fine-tune (Tinh Chỉnh) Foundation Model trên Bedrock với dữ liệu doanh nghiệp
- [ ] Thiết kế multi-model endpoint (Endpoint Đa Mô Hình) SageMaker tiết kiệm chi phí
- [ ] Triển khai Responsible AI (AI Có Trách Nhiệm) với SageMaker Clarify
- [ ] Thiết kế kiến trúc Generative AI application production-ready với RAG, guardrails, monitoring

---

## 🚀 Bắt Đầu

### Bước 1: Đặt Mục Tiêu Học Tập

```
Chọn hướng đi phù hợp:
- ML Engineer: Tập trung SageMaker Training + Deployment + MLOps
- AI Application Developer: Tập trung Bedrock + Rekognition + Comprehend + Lex
- Data Scientist: Tập trung SageMaker Studio + Autopilot + Experiments
- Full-stack AI Engineer: Nắm vững tất cả tầng AI Services và SageMaker
```

### Bước 2: Chuẩn Bị Môi Trường Lab

```bash
# Cài đặt AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)
aws configure

# Kiểm tra kết nối
aws sts get-caller-identity

# Gọi thử Rekognition API phát hiện nhãn ảnh (cần có file ảnh local)
aws rekognition detect-labels \
  --image '{"S3Object":{"Bucket":"my-bucket","Name":"photo.jpg"}}' \
  --max-labels 10

# Gọi thử Comprehend API phân tích cảm xúc
aws comprehend detect-sentiment \
  --text "AWS AI services are incredibly powerful!" \
  --language-code en
```

### Bước 3: Học + Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Thực hành trên AWS Console hoặc CLI (30-45 phút)
3. Gọi API qua Python boto3 SDK (Bộ Công Cụ Phát Triển) (15 phút)
4. Xem lại checklist cost — xóa endpoint và resources không dùng (5 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation (Tình Huống): Bài toán AI/ML cần giải quyết
- Task (Nhiệm Vụ): Model / pipeline / ứng dụng AI bạn phải xây dựng
- Action (Hành Động): Dịch vụ AWS AI, kiến trúc và kỹ thuật bạn đã dùng
- Result (Kết Quả): Accuracy (Độ Chính Xác), latency, cost savings đạt được & bài học
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"Hands-On Machine Learning"** by Aurélien Géron — Thực hành ML với Scikit-Learn & TensorFlow
- **"Deep Learning"** by Ian Goodfellow et al. — Nền tảng học sâu toàn diện
- **"Designing Machine Learning Systems"** by Chip Huyen — ML Production Engineering
- **"Building LLM Applications"** — Ứng dụng thực tế với Large Language Models

### Tài Liệu Chính Thức AWS

- [Amazon SageMaker Developer Guide](https://docs.aws.amazon.com/sagemaker/)
- [Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/)
- [Amazon Rekognition Developer Guide](https://docs.aws.amazon.com/rekognition/)
- [Amazon Comprehend Developer Guide](https://docs.aws.amazon.com/comprehend/)
- [Amazon Transcribe Developer Guide](https://docs.aws.amazon.com/transcribe/)
- [Amazon Lex Developer Guide](https://docs.aws.amazon.com/lex/)

### Blog & Nguồn Học Thêm

- AWS Machine Learning Blog (aws.amazon.com/blogs/machine-learning/)
- AWS re:Invent AI/ML talks (YouTube)
- fast.ai — Practical Deep Learning for Coders
- Hugging Face Documentation — Open-source AI models & datasets

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Danh Mục

#### SageMaker & ML Training

- [ ] SageMaker Training Job vs SageMaker Autopilot — khi nào dùng cái nào?
- [ ] Giải thích Hyperparameter Tuning (Tinh Chỉnh Siêu Tham Số) với SageMaker AMT (Automatic Model Tuning)?
- [ ] Spot Training (Huấn Luyện Spot) là gì? Lợi ích và rủi ro?
- [ ] SageMaker Inference types (Loại Dự Đoán): Real-time vs Batch vs Async vs Serverless — khi nào dùng?

#### Generative AI & Bedrock

- [ ] Foundation Model (Mô Hình Nền Tảng) là gì? Khác gì với traditional ML model?
- [ ] RAG (Retrieval-Augmented Generation — Tạo Sinh Tăng Cường Truy Xuất) là gì? Tại sao dùng?
- [ ] Prompt Engineering (Kỹ Thuật Lập Trình Prompt) — Zero-shot, Few-shot, Chain-of-thought là gì?
- [ ] Fine-tuning (Tinh Chỉnh) model trên Bedrock khác gì RAG? Khi nào chọn approach nào?

#### AI Services (Rekognition, Comprehend, v.v.)

- [ ] Rekognition Custom Labels vs Rekognition built-in — khi nào cần custom?
- [ ] Amazon Comprehend phân tích những gì? Khác gì với Bedrock/LLM?
- [ ] Textract vs Rekognition — cái nào dùng để đọc tài liệu có bảng biểu?
- [ ] Khi nào dùng Amazon Lex thay vì xây dựng chatbot với Bedrock trực tiếp?

#### MLOps & Architecture

- [ ] MLOps (ML Operations) là gì? Các thành phần chính trong SageMaker MLOps?
- [ ] Model drift (Trôi Dạt Mô Hình) là gì? Cách phát hiện và xử lý?
- [ ] Feature Store (Kho Đặc Trưng) giải quyết vấn đề gì trong ML production?
- [ ] Thiết kế end-to-end ML pipeline cho bài toán recommendation (gợi ý sản phẩm)?

Xem `11-interview-prep/` để có hướng dẫn đầy đủ Q&A.

---

## ✅ Checklist Tự Đánh Giá

Trước khi phỏng vấn hoặc nhận vai trò mới, xác nhận:

- [ ] Có thể giải thích sự khác biệt giữa AI Services và SageMaker custom training
- [ ] Có thể thiết kế end-to-end ML pipeline đơn giản với ít nhất 2 dịch vụ AWS AI
- [ ] Có thể gọi Rekognition API và giải thích kết quả confidence score (Điểm Tin Cậy)
- [ ] Có thể giải thích RAG (Retrieval-Augmented Generation) và khi nào nên dùng Bedrock
- [ ] Có thể chọn dịch vụ AI phù hợp (Transcribe vs Polly vs Comprehend vs Textract) cho từng bài toán
- [ ] Có thể mô tả MLOps workflow (Quy Trình MLOps) với SageMaker Pipelines
- [ ] Có thể ước tính chi phí xấp xỉ cho một AI/ML workload trên AWS
- [ ] Có thể giải thích Responsible AI (AI Có Trách Nhiệm) và cách AWS hỗ trợ qua Clarify
- [ ] Có thể thiết kế HA (High Availability — Tính Sẵn Sàng Cao) cho SageMaker inference endpoint
- [ ] Có thể xử lý sự cố phổ biến: SageMaker training failure, Bedrock throttling, Rekognition accuracy issues

---

## 📞 Hỗ Trợ & Tài Nguyên

### Học Tập

- [AWS Free Tier](https://aws.amazon.com/free/) — Rekognition, Comprehend, Transcribe, Polly có free tier
- [AWS Skill Builder](https://skillbuilder.aws/) — Khóa học chính thức AWS ML & AI
- [SageMaker Examples (GitHub)](https://github.com/aws/amazon-sagemaker-examples) — Jupyter notebook thực hành
- [AWS AI/ML Blog](https://aws.amazon.com/blogs/machine-learning/) — Cập nhật mới nhất về AI services

### Công Cụ

- **SageMaker Studio** — IDE toàn diện cho ML (Jupyter-based)
- **AWS boto3** — Python SDK để gọi tất cả AI services
- **Amazon Bedrock Playground** — Thử các Foundation Models trực tiếp trên Console
- **SageMaker Canvas** — No-code ML (Học Máy Không Cần Lập Trình) cho business users
- **LangChain + Bedrock** — Framework xây dựng ứng dụng LLM

### Chứng Chỉ Liên Quan

- **AWS Certified Machine Learning — Specialty** — Chứng chỉ chuyên ngành ML của AWS
- **AWS Certified AI Practitioner** (mới 2024) — Chứng chỉ nền tảng AI/ML cho mọi người

### Cộng Đồng

- r/MachineLearning & r/aws (Reddit)
- AWS re:Post — Machine Learning category
- Hugging Face Community Forum
- AWS ML Community (Discord, Slack)

---

## 📋 Cách Sử Dụng Hướng Dẫn Này

### Cho Mục Đích Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành hands-on: gọi API, xây dựng mini app AI
4. **Chú ý chi phí** — SageMaker endpoint tính phí theo giờ kể cả khi không có traffic — xóa sau khi xong

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep/](./11-interview-prep/)
2. Nắm sâu SageMaker + Bedrock + ít nhất 2 AI Services (Rekognition/Comprehend phổ biến nhất)
3. Chuẩn bị câu chuyện xây dựng AI application theo phương pháp STAR
4. Luyện giải thích trade-offs: Custom SageMaker vs AI Services, Bedrock RAG vs Fine-tuning

### Cho Công Việc Thực Tế

1. Xác định bài toán: Cần model custom hay AI Service có sẵn đủ dùng?
2. Tham khảo [02-sagemaker/](./02-sagemaker/) cho custom ML, [03-bedrock/](./03-bedrock/) cho Generative AI
3. Dùng [09-mlops/](./09-mlops/) để thiết lập CI/CD và monitoring cho model production
4. Áp dụng [10-advanced/](./10-advanced/) để tối ưu chi phí và bảo mật

---

## 🗺️ Các Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/ để nắm vững nền tảng AI/ML
├─ 4️⃣  Tạo AWS Free Tier account và thử Rekognition / Comprehend với ảnh/văn bản của bạn
├─ 5️⃣  Thực hành Bedrock Playground: so sánh Claude, Llama 2, Titan
├─ 6️⃣  Xây dựng mini AI app: S3 ảnh → Rekognition → kết quả lưu DynamoDB
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Maintainer:** Backend Interview Prep
