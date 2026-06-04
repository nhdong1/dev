# AWS AI Layers — Ba Tầng Dịch Vụ AI Của AWS

> AWS phân chia dịch vụ AI/ML thành 3 tầng rõ ràng. Hiểu đúng để chọn giải pháp phù hợp — và trả lời câu hỏi phỏng vấn về kiến trúc hệ thống AI một cách thuyết phục.

---

## 1. Kiến Trúc Ba Tầng (Three-Tier AI Architecture)

```
┌─────────────────────────────────────────────────────────────────────┐
│  TẦNG 1: AI Services (Dịch Vụ AI Sẵn Có)                          │
│  ─────────────────────────────────────────────────────────────────  │
│  Rekognition │ Comprehend │ Polly │ Transcribe │ Translate          │
│  Textract    │ Lex        │ Kendra│ Forecast   │ Personalize        │
│  Amazon Q    │ Bedrock    │ ...                                      │
│                                                                      │
│  ✅ Không cần ML expertise    ✅ Pay-per-use                         │
│  ✅ Tích hợp qua API đơn giản  ✅ Quản lý hoàn toàn bởi AWS         │
├─────────────────────────────────────────────────────────────────────┤
│  TẦNG 2: ML Services (Dịch Vụ Học Máy)                             │
│  ─────────────────────────────────────────────────────────────────  │
│  Amazon SageMaker (Studio, Training, Inference, Pipelines, ...)     │
│                                                                      │
│  ✅ Kiểm soát toàn bộ ML lifecycle   ✅ Custom model                │
│  ✅ Built-in algorithms              ⚠️ Cần ML expertise            │
├─────────────────────────────────────────────────────────────────────┤
│  TẦNG 3: ML Frameworks & Infrastructure (Khung & Hạ Tầng)          │
│  ─────────────────────────────────────────────────────────────────  │
│  TensorFlow │ PyTorch │ Apache MXNet trên EC2 GPU/Trainium/Inf2     │
│  AWS Neuron SDK │ ECS/EKS cho distributed training                  │
│                                                                      │
│  ✅ Kiểm soát tối đa, tùy chỉnh cao  ⚠️ Tự quản lý hạ tầng        │
│  ✅ Nghiên cứu & cutting-edge AI     ⚠️ Tốn kém nhất               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tầng 1 — AI Services (Dịch Vụ AI Sẵn Có)

### Định Nghĩa

AI Services là các dịch vụ AWS được **pre-trained (huấn luyện sẵn)** cho các tác vụ phổ biến. Bạn chỉ cần gọi API, không cần biết ML.

### Danh Sách Dịch Vụ AI Chính

#### Computer Vision — Thị Giác Máy Tính

| Dịch Vụ | Chức Năng | Use Case (Trường Hợp Dùng) |
| ------- | --------- | -------------------------- |
| **Amazon Rekognition** | Phân tích hình ảnh & video | Object detection, face recognition, content moderation |
| **Amazon Textract** | OCR thông minh | Đọc form, bảng, invoice từ tài liệu scan |

#### Natural Language Processing — Xử Lý Ngôn Ngữ Tự Nhiên

| Dịch Vụ | Chức Năng | Use Case |
| ------- | --------- | --------- |
| **Amazon Comprehend** | Phân tích văn bản | Sentiment analysis, entity recognition, topic modeling |
| **Amazon Comprehend Medical** | NLP y tế | Trích xuất thực thể y tế từ hồ sơ bệnh án |
| **Amazon Translate** | Dịch máy thần kinh | Dịch tự động đa ngôn ngữ |

#### Speech AI — AI Giọng Nói

| Dịch Vụ | Chức Năng | Use Case |
| ------- | --------- | --------- |
| **Amazon Transcribe** | Speech-to-Text (Giọng Nói → Văn Bản) | Phụ đề tự động, phiên âm cuộc họp |
| **Amazon Polly** | Text-to-Speech (Văn Bản → Giọng Nói) | Trợ lý giọng nói, accessibility |

#### Conversational AI — AI Đàm Thoại

| Dịch Vụ | Chức Năng | Use Case |
| ------- | --------- | --------- |
| **Amazon Lex** | Chatbot & Voice assistant | Customer service bot, IVR |
| **Amazon Kendra** | Intelligent Search (Tìm Kiếm Thông Minh) | Internal search engine doanh nghiệp |
| **Amazon Q Business** | Enterprise AI Assistant (Trợ Lý AI Doanh Nghiệp) | Hỏi đáp trên dữ liệu nội bộ |
| **Amazon Q Developer** | AI Lập Trình | Code generation, security scan |

#### Predictions — Dự Báo & Gợi Ý

| Dịch Vụ | Chức Năng | Use Case |
| ------- | --------- | --------- |
| **Amazon Forecast** | Time Series Forecasting (Dự Báo Chuỗi Thời Gian) | Dự báo nhu cầu, tồn kho |
| **Amazon Personalize** | Recommendation Engine (Hệ Thống Gợi Ý) | Gợi ý sản phẩm/nội dung cá nhân hóa |
| **Amazon Lookout for Metrics** | Anomaly Detection (Phát Hiện Bất Thường) | Phát hiện bất thường KPI business |
| **Amazon Lookout for Equipment** | Predictive Maintenance (Bảo Trì Dự Đoán) | Phát hiện sự cố máy móc |

#### Generative AI

| Dịch Vụ | Chức Năng | Use Case |
| ------- | --------- | --------- |
| **Amazon Bedrock** | Generative AI qua Foundation Models | Chatbot, RAG, content generation, code |
| **Amazon CodeWhisperer** → **Amazon Q Developer** | AI code assistant | Code completion, security scan |

### Khi Nào Dùng AI Services?

✅ Bài toán phổ biến: nhận diện ảnh, phân tích văn bản, dịch, giọng nói
✅ Cần triển khai nhanh (proof of concept, MVP)
✅ Team không có ML engineer
✅ Dữ liệu không đủ để train custom model
✅ Muốn pay-per-use, không muốn quản lý hạ tầng

❌ Khi cần nhận diện đối tượng rất đặc thù (ví dụ: loại vi khuẩn trong lab)
❌ Khi cần control hoàn toàn model behavior
❌ Khi yêu cầu data privacy cực cao (data không được ra khỏi VPC)

---

## 3. Tầng 2 — ML Services (Dịch Vụ Học Máy): Amazon SageMaker

### Định Nghĩa

SageMaker là nền tảng ML toàn diện — bao phủ toàn bộ ML workflow từ data preparation đến deployment và monitoring — nhưng bạn phải kiểm soát việc training và model design.

### SageMaker Components (Thành Phần SageMaker)

```
Amazon SageMaker
├── Studio (IDE)
│   └── JupyterLab + RStudio + Code Editor tích hợp
│
├── Data Preparation (Chuẩn Bị Dữ Liệu)
│   ├── Data Wrangler — GUI transform data
│   ├── Processing Jobs — code transform data
│   └── Feature Store — lưu và tái sử dụng features
│
├── Training (Huấn Luyện)
│   ├── Built-in Algorithms (Thuật Toán Tích Hợp): XGBoost, Linear Learner, DeepAR+, ...
│   ├── Custom Containers: TensorFlow, PyTorch, Scikit-learn, ...
│   ├── Automatic Model Tuning (AMT): HPO tự động
│   ├── Spot Training: tiết kiệm đến 90% chi phí training
│   └── Experiments: tracking runs, metrics, artifacts
│
├── Inference (Triển Khai & Suy Luận)
│   ├── Real-time Endpoint
│   ├── Batch Transform
│   ├── Async Endpoint
│   └── Serverless Endpoint
│
├── MLOps (Vận Hành ML)
│   ├── Pipelines — CI/CD cho ML
│   ├── Model Registry — versioning & approval
│   ├── Model Monitor — drift detection
│   └── Clarify — bias & explainability
│
└── Specialized Tools
    ├── Autopilot — AutoML
    ├── Canvas — no-code ML
    └── Ground Truth — data labeling
```

### SageMaker Built-in Algorithms (Thuật Toán Tích Hợp)

| Thuật Toán | Loại | Dùng Cho |
| ---------- | ---- | --------- |
| **XGBoost** | Supervised | Tabular data, classification, regression |
| **Linear Learner** | Supervised | Binary/multi-class classification, regression |
| **DeepAR+** | Supervised (time series) | Forecasting chuỗi thời gian |
| **K-Means** | Unsupervised | Clustering |
| **PCA** | Unsupervised | Dimensionality reduction |
| **Random Cut Forest** | Unsupervised | Anomaly detection |
| **BlazingText** | NLP | Word2Vec, text classification |
| **Image Classification** | Computer Vision | Binary/multi-class image classification |
| **Object Detection** | Computer Vision | Bounding box detection |

### Khi Nào Dùng SageMaker?

✅ Cần train custom model với dữ liệu riêng
✅ AI Service có sẵn không đủ chính xác cho domain cụ thể
✅ Cần kiểm soát toàn bộ model (architecture, hyperparameter, training data)
✅ Cần MLOps pipeline cho production (pipelines, model registry, monitoring)
✅ Team có ML engineer

❌ Khi AI Service có sẵn đã đủ — không cần phức tạp hóa
❌ Khi không có dữ liệu training
❌ Khi cần deploy nhanh (AI Service nhanh hơn nhiều)

---

## 4. Tầng 3 — ML Frameworks & Infrastructure (Khung & Hạ Tầng)

### Định Nghĩa

Tầng thấp nhất — bạn tự quản lý hạ tầng, chọn framework, tự cài đặt môi trường. AWS cung cấp hardware tối ưu.

### AWS AI/ML Hardware Chuyên Biệt

| Instance Type | Tên Chip | Tối Ưu Cho | So Với NVIDIA GPU |
| ------------- | -------- | ---------- | ----------------- |
| **trn1** | AWS Trainium | Training deep learning | Rẻ hơn đến 50% |
| **inf2** | AWS Inferentia 2 | Inference (deployment) | Rẻ hơn đến 70% |
| **p4d/p5** | NVIDIA A100/H100 | Training, research | Phổ biến, ecosystem lớn |
| **g5** | NVIDIA A10G | Inference, smaller training | Cân bằng giá/hiệu suất |

### AWS Neuron SDK

**AWS Neuron** là SDK để compile và chạy model Deep Learning trên chip Trainium và Inferentia. Tương thích với PyTorch, TensorFlow.

```python
# Ví dụ: compile model PyTorch cho Inferentia 2
import torch
import torch_neuronx

model = load_my_model()
traced_model = torch.jit.trace(model, example_inputs)
neuron_model = torch_neuronx.trace(traced_model, example_inputs)
neuron_model.save("model_neuron.pt")
```

### Khi Nào Dùng Tầng 3?

✅ Nghiên cứu AI/ML, cần kiểm soát tuyệt đối
✅ SageMaker không hỗ trợ framework/kỹ thuật cần dùng
✅ Distributed training quy mô lớn (training LLM từ đầu)
✅ Tối ưu chi phí inference cực độ với Inferentia

❌ Hầu hết doanh nghiệp thông thường — SageMaker đủ dùng
❌ Khi team không có DevOps/MLOps expertise

---

## 5. Decision Framework — Chọn Đúng Tầng

### Flowchart Quyết Định

```
Bắt đầu: Bạn có bài toán AI cần giải quyết
                    │
                    ▼
     Bài toán này có AI Service sẵn không?
     (Rekognition, Comprehend, Transcribe...)
                    │
         ┌──────────┴──────────┐
         │ Có                  │ Không
         ▼                     ▼
  AI Service đủ       Cần custom model
  chính xác?          (SageMaker / Tầng 3)
         │
    ┌────┴────┐
    │ Có     │ Không
    ▼         ▼
  DÙNG     Thêm Rekognition
  AI       Custom Labels
  SERVICE  hoặc Comprehend
           Custom Classification
               │
           Vẫn không đủ?
               │
               ▼
          DÙNG SAGEMAKER
          (custom training)
               │
    Cần kiểm soát tuyệt đối
    hoặc train LLM từ đầu?
               │
               ▼
         TẦNG 3 (EC2 GPU)
```

### Ma Trận So Sánh Ba Tầng

| Tiêu Chí | AI Services | SageMaker | Frameworks & Infra |
| --------- | ----------- | --------- | ------------------ |
| **ML Expertise Cần** | Không | Có | Nhiều |
| **Tốc Độ Triển Khai** | Rất nhanh (giờ) | Vừa (ngày-tuần) | Chậm (tuần-tháng) |
| **Kiểm Soát Model** | Không | Cao | Tuyệt đối |
| **Custom Training** | Hạn chế | Toàn diện | Toàn diện |
| **Quản Lý Hạ Tầng** | AWS quản lý | AWS hỗ trợ | Tự quản lý |
| **Chi Phí** | Pay-per-use | Instance + storage | Instance (raw) |
| **Phù Hợp Với** | App developer | ML Engineer | AI Researcher |

---

## 6. AWS AI Strategy — Chiến Lược AI Của AWS

### "Democratize AI" (Dân Chủ Hóa AI)

AWS xây dựng 3 tầng với triết lý:
- **Tầng 1 (AI Services):** Bất kỳ developer nào cũng có thể dùng AI
- **Tầng 2 (SageMaker):** ML Engineer có thể build custom ML system
- **Tầng 3 (Framework & Infra):** Nhà nghiên cứu có thể đổi mới AI

### Xu Hướng 2024-2026: Generative AI Shift

```
Trước 2022:
  Tầng 1 (AI Services) → Pre-trained model đơn nhiệm vụ
  Ví dụ: Rekognition chỉ làm Computer Vision

Sau 2022 (Generative AI Era):
  Bedrock (Tầng 1 mở rộng) → Foundation Models đa năng
  Ví dụ: Claude làm được: viết code, phân tích ảnh, tóm tắt văn bản...

Ảnh hưởng:
  → Bedrock đang "ăn" use case của nhiều AI Services chuyên biệt
  → Nhưng AI Services vẫn rẻ hơn và tối ưu hơn cho tác vụ cụ thể
```

---

## 7. Ví Dụ Thực Tế: Xây Dựng Hệ Thống Hỗ Trợ Khách Hàng

```
Yêu Cầu:
  - Chatbot tự động trả lời câu hỏi khách hàng
  - Phân tích cảm xúc khách hàng từ email
  - Dịch tự động sang tiếng Anh
  - Nhận diện tài liệu khách hàng gửi (CMND, hóa đơn)

Giải Pháp Theo Tầng:

Tầng 1 (AI Services) — dùng 4 dịch vụ:
  ├── Amazon Lex → chatbot hiểu intent & slot
  ├── Amazon Comprehend → phân tích sentiment email
  ├── Amazon Translate → dịch câu hỏi sang tiếng Anh
  └── Amazon Textract → đọc CMND, hóa đơn

Tầng 2 (SageMaker) — nếu cần thêm:
  └── SageMaker + Comprehend Custom Classification
      → Phân loại loại yêu cầu (hoàn tiền / kỹ thuật / tài khoản)
      khi Comprehend built-in không đủ chính xác

Kết quả:
  - Time-to-market: 2-4 tuần (AI Services, không cần train)
  - vs. Tự train: 3-6 tháng
  - Chi phí: Pay-per-use, không phải trả tiền training & serving infrastructure
```

---

## 8. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Khi nào dùng AI Service (Rekognition) vs Custom SageMaker model?**

> Dùng AI Service khi: bài toán phổ biến (nhận diện object, khuôn mặt, OCR cơ bản), cần triển khai nhanh, team không có ML expertise, dữ liệu không đủ để train. Dùng SageMaker khi: bài toán domain-specific (ví dụ: phát hiện khuyết tật linh kiện điện tử), AI Service không đủ chính xác sau khi thử, cần kiểm soát model hoàn toàn, hoặc cần MLOps pipeline phức tạp.

**Q: AWS AI Layer nào phù hợp cho startup với 2 developer?**

> Tầng 1 (AI Services) — lý do: (1) không cần ML expertise; (2) triển khai nhanh; (3) pay-per-use, không cần đầu tư infrastructure ban đầu; (4) AWS quản lý hoàn toàn uptime, scaling. Chỉ lên Tầng 2 khi AI Services không đáp ứng được yêu cầu accuracy cụ thể.

**Q: Bedrock thuộc tầng nào? Có khác gì các AI Services khác?**

> Bedrock thuộc Tầng 1 (AI Service), nhưng đặc biệt hơn vì: (1) cung cấp nhiều Foundation Models từ nhiều nhà cung cấp (Anthropic, Meta, Cohere, Amazon); (2) hỗ trợ fine-tuning model; (3) có thể build RAG và Agents phức tạp; (4) tính phí per-token thay vì per-request. Bedrock gần giống Tầng 1.5 — nhiều khả năng customize hơn AI Services thông thường nhưng vẫn không phải tự train từ đầu.

---

## ➡️ Tiếp Theo

[5-generative-ai-concepts.md](./5-generative-ai-concepts.md) — Các khái niệm cốt lõi của Generative AI: LLM, Foundation Model, Prompt, Token, RAG — chuẩn bị cho học Amazon Bedrock.
