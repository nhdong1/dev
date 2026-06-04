# Module 11 — Chuẩn Bị Phỏng Vấn AWS AI/ML

> Hướng dẫn toàn diện để tự tin bước vào phỏng vấn vị trí liên quan đến AWS AI/ML — từ câu hỏi kỹ thuật, thiết kế hệ thống, đến bài tập tình huống hành vi

---

## 📁 Nội Dung Module

| File | Mô Tả | Trạng Thái |
| ---- | ------ | ---------- |
| `README.md` | Tổng quan module, lộ trình ôn tập, checklist | ✅ |
| `INTERVIEW_GUIDE.md` | Top 20 câu hỏi phỏng vấn AWS AI/ML + gợi ý trả lời chi tiết | ✅ |
| `system-design-scenarios.md` | 5 kịch bản thiết kế hệ thống AI thực tế (System Design) | ✅ |
| `90-day-study-plan.md` | Kế hoạch học 90 ngày chi tiết theo tuần và ngày | ✅ |

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn sẽ có khả năng:

- **Trả lời tự tin** Top 20 câu hỏi phỏng vấn AWS AI/ML phổ biến nhất
- **Thiết kế hệ thống AI** end-to-end trong vòng 45 phút khi được hỏi trong vòng phỏng vấn
- **Giải thích trade-offs** giữa các approach: AI Services vs SageMaker, RAG vs Fine-tuning, Real-time vs Batch
- **Kể câu chuyện dự án** theo phương pháp STAR (Situation — Task — Action — Result)
- **Ước tính chi phí** và đề xuất tối ưu cho một AI/ML workload trên AWS

---

## 🗂️ Loại Câu Hỏi Phỏng Vấn

### 1. Câu Hỏi Kỹ Thuật (Technical Questions)

Kiểm tra hiểu biết về dịch vụ AWS AI/ML:

- **Khái niệm cốt lõi:** Foundation Model (Mô Hình Nền Tảng) là gì? RAG — Retrieval-Augmented Generation (Tạo Sinh Tăng Cường Truy Xuất) hoạt động thế nào?
- **So sánh dịch vụ:** SageMaker Real-time vs Batch vs Async vs Serverless inference
- **Giới hạn & quota:** Rekognition xử lý tối đa bao nhiêu ảnh? Bedrock throttling xử lý thế nào?
- **Bảo mật:** Cách bảo vệ PII — Personally Identifiable Information (Thông Tin Định Danh Cá Nhân) trong pipeline AI

### 2. Câu Hỏi Thiết Kế Hệ Thống (System Design)

Yêu cầu bạn thiết kế kiến trúc AI/ML cho bài toán thực tế:

- Thiết kế hệ thống recommendation (gợi ý) cá nhân hóa cho 10 triệu người dùng
- Xây dựng RAG chatbot (chatbot tăng cường truy xuất) cho doanh nghiệp với dữ liệu nội bộ
- Kiến trúc content moderation pipeline (đường ống kiểm duyệt nội dung) tự động
- Thiết kế MLOps — ML Operations (Vận Hành Mô Hình) platform từ training đến production monitoring

### 3. Câu Hỏi Hành Vi (Behavioral Questions)

Đánh giá kinh nghiệm và cách giải quyết vấn đề thực tế:

- "Kể về một lần bạn phải chọn giữa AI Service có sẵn và xây dựng model custom"
- "Mô tả cách bạn xử lý khi model AI đưa ra kết quả không chính xác trong production"
- "Bạn đã làm gì khi phát hiện model có bias (thiên kiến) ảnh hưởng đến một nhóm người dùng?"

### 4. Câu Hỏi Tình Huống (Situational / Case Study)

- "Chi phí SageMaker training job đang quá cao. Bạn sẽ tối ưu thế nào?"
- "Bedrock API đang bị throttling (hạn chế tốc độ) lúc peak traffic. Xử lý thế nào?"
- "Model accuracy (độ chính xác) giảm đột ngột sau 2 tháng production. Nguyên nhân và cách xử lý?"

---

## ✅ Checklist Chuẩn Bị Phỏng Vấn

### Tuần Trước Phỏng Vấn

#### Kiến Thức Nền Tảng

- [ ] Giải thích được AI vs ML vs Deep Learning (Học Sâu) vs Generative AI (AI Tạo Sinh)
- [ ] Biết khi nào dùng AI Service có sẵn thay vì train custom model trên SageMaker
- [ ] Hiểu Foundation Model (Mô Hình Nền Tảng) và sự khác biệt với traditional ML model
- [ ] Giải thích được RAG — Retrieval-Augmented Generation và vector embeddings (nhúng vector)

#### SageMaker

- [ ] Mô tả training job lifecycle: data → S3 → training container → model artifact
- [ ] Phân biệt 4 loại inference: Real-time, Batch Transform, Async, Serverless
- [ ] Giải thích SageMaker Pipelines — DAG (Directed Acyclic Graph — Đồ Thị Vô Hướng Có Hướng) và CI/CD cho ML
- [ ] Biết khi nào dùng Spot training và cách xử lý checkpointing (lưu điểm kiểm tra)

#### Bedrock & Generative AI

- [ ] So sánh các Foundation Models: Claude, Llama 3, Titan, Stable Diffusion
- [ ] Giải thích Zero-shot, Few-shot, Chain-of-thought prompting
- [ ] Mô tả RAG pipeline: Query → Embed → Vector Search → Context → LLM
- [ ] Khi nào dùng RAG, khi nào Fine-tune (tinh chỉnh), khi nào dùng prompt engineering?

#### AI Services

- [ ] Rekognition: labels, faces, text, moderation, PPE — use case từng loại
- [ ] Comprehend: sentiment, entities, key phrases, custom classification
- [ ] Textract vs Rekognition — khi nào dùng cái nào để xử lý tài liệu
- [ ] Amazon Transcribe Custom Vocabulary vs Custom Language Model — khác nhau gì?

#### MLOps

- [ ] Mô tả model drift (trôi dạt mô hình) và cách SageMaker Model Monitor phát hiện
- [ ] SageMaker Clarify cho bias detection và SHAP — SHapley Additive exPlanations (Giải Thích Bằng Đóng Góp SHAP)
- [ ] Model Registry approval workflow — PendingManualApproval → Approved → Rejected

#### Kiến Trúc & Chi Phí

- [ ] Thiết kế được 1 end-to-end AI pipeline hoàn chỉnh (tối thiểu 3 dịch vụ AWS)
- [ ] Biết các kỹ thuật tối ưu chi phí: Spot, MME — Multi-Model Endpoint, Serverless
- [ ] Hiểu tradeoffs: latency vs throughput vs cost cho inference

### Ngày Trước Phỏng Vấn

- [ ] Đọc lại `INTERVIEW_GUIDE.md` — luyện trả lời to cho 5 câu khó nhất
- [ ] Chuẩn bị 2-3 câu chuyện STAR về dự án AI/ML thực tế
- [ ] Ôn lại 1 system design scenario từ `system-design-scenarios.md`
- [ ] Xem lại pricing model của các dịch vụ AI AWS chính
- [ ] Chuẩn bị câu hỏi ngược lại cho interviewer (về tech stack, scale, challenges)

---

## 🎓 Lộ Trình Ôn Tập Theo Vai Trò

### ML Engineer / MLOps Engineer

**Tập trung:** SageMaker training/deployment + MLOps + cost optimization

```
Ngày 1-3: SageMaker Training (02-sagemaker/2-sagemaker-training.md)
Ngày 4-5: SageMaker Inference types (02-sagemaker/3-sagemaker-inference.md)
Ngày 6-7: SageMaker Pipelines CI/CD ML (09-mlops/1-sagemaker-pipelines.md)
Ngày 8:   Model Registry & Monitor (09-mlops/2-model-registry.md + 3-model-monitor.md)
Ngày 9:   Clarify bias & SHAP (09-mlops/4-sagemaker-clarify.md)
Ngày 10:  Cost optimization & Architecture (10-advanced/)
Ngày 11+: Luyện System Design Scenarios
```

### AI Application Developer

**Tập trung:** Bedrock + AI Services (Rekognition, Comprehend, Lex, v.v.)

```
Ngày 1-2: Bedrock Foundation Models & Prompt Engineering (03-bedrock/1-2)
Ngày 3-4: RAG & Bedrock Agents (03-bedrock/3-4)
Ngày 5:   Computer Vision — Rekognition + Textract (04-computer-vision/)
Ngày 6:   NLP — Comprehend + Translate (05-nlp-text/)
Ngày 7:   Conversational AI — Lex + Amazon Q (07-conversational-ai/)
Ngày 8:   Speech AI — Transcribe + Polly (06-speech/)
Ngày 9-10: Luyện INTERVIEW_GUIDE + System Design
```

### Data Scientist

**Tập trung:** SageMaker Studio + Autopilot + Experiments + Responsible AI

```
Ngày 1-2: SageMaker Studio & Autopilot (02-sagemaker/1-4)
Ngày 3:   Feature Store (02-sagemaker/5-sagemaker-feature-store.md)
Ngày 4:   Experiments tracking & ML Lineage (09-mlops/5-experiments-tracking.md)
Ngày 5:   Clarify + Responsible AI (09-mlops/4-sagemaker-clarify.md + 10-advanced/3)
Ngày 6:   Forecast & Personalize (08-predictions/)
Ngày 7+:  Luyện câu hỏi kỹ thuật và behavioral
```

### Solution Architect / Cloud Architect (AI/ML)

**Tập trung:** Kiến trúc end-to-end + Security + Cost + Well-Architected

```
Ngày 1:   AWS AI layers — AI Services vs SageMaker vs Framework (01-fundamentals/)
Ngày 2-3: SageMaker tổng quan (02-sagemaker/README.md + 02-sagemaker/3-inference)
Ngày 4:   Bedrock kiến trúc (03-bedrock/README.md + RAG + Agents)
Ngày 5:   Security & Compliance cho AI/ML (10-advanced/2-security-compliance.md)
Ngày 6:   Cost Optimization + ML Architecture Patterns (10-advanced/1 + 4)
Ngày 7+:  Tập trung System Design Scenarios — 5 kịch bản trong system-design-scenarios.md
```

---

## 💡 Chiến Thuật Trả Lời Câu Hỏi

### Công Thức STAR cho Câu Hỏi Hành Vi

```
S — Situation (Tình Huống):
   "Ở công ty X, chúng tôi có bài toán Y với quy mô Z..."

T — Task (Nhiệm Vụ):
   "Nhiệm vụ của tôi là thiết kế / xây dựng / tối ưu..."

A — Action (Hành Động):
   "Tôi đã chọn SageMaker + Bedrock vì [lý do trade-off cụ thể]...
    Các bước thực hiện: 1) ... 2) ... 3) ..."

R — Result (Kết Quả):
   "Kết quả: accuracy tăng X%, latency giảm Yms, chi phí giảm Z%,
    bài học rút ra: ..."
```

### Công Thức Trả Lời Câu Hỏi So Sánh

```
Cấu trúc: "Depends on the use case — Tùy thuộc vào trường hợp sử dụng"

1. Nêu điểm khác biệt chính (1-2 câu)
2. Khi nào dùng A (2-3 bullet points)
3. Khi nào dùng B (2-3 bullet points)
4. Ví dụ thực tế từ kinh nghiệm của bạn (nếu có)
```

### Công Thức Thiết Kế Hệ Thống

```
Bước 1: Clarify requirements — Làm rõ yêu cầu (2-3 phút)
  - Scale: bao nhiêu request/s? bao nhiêu data?
  - Latency: real-time hay batch?
  - Accuracy vs cost tradeoff?

Bước 2: High-level architecture — Kiến trúc tổng quan (5 phút)
  - Vẽ data flow từ input → processing → output
  - Chọn dịch vụ AI phù hợp và giải thích lý do

Bước 3: Deep dive — Đào sâu vào phần quan trọng nhất (15 phút)
  - Training pipeline vs Inference pipeline
  - Storage: S3, Feature Store, Vector Store
  - Monitoring: Model Monitor, CloudWatch

Bước 4: Trade-offs & alternatives — Đánh đổi và phương án thay thế (5 phút)
  - Giải thích tại sao chọn approach này
  - Đề cập phương án thay thế đã cân nhắc

Bước 5: Cost & scale — Chi phí và khả năng mở rộng (3 phút)
  - Ước tính chi phí xấp xỉ
  - Điểm nút thắt cổ chai (bottleneck) tiềm năng
```

---

## ⚠️ Sai Lầm Phổ Biến Cần Tránh

### Về Kỹ Thuật

1. **Nhầm Rekognition và Textract:** Rekognition phát hiện object/face/text trong ảnh tự nhiên; Textract đọc văn bản có cấu trúc từ documents, forms, tables
2. **Không biết khi nào dùng Async inference:** Async phù hợp cho payload lớn (>6MB) hoặc model inference time dài (>60s) — không phải mọi trường hợp
3. **Confuse RAG vs Fine-tuning:** RAG thêm knowledge mới tại inference time; Fine-tuning thay đổi behavior/style của model thông qua training
4. **Bỏ qua Model Drift:** Interviewer rất hay hỏi về data drift (trôi dạt dữ liệu) vs concept drift (trôi dạt khái niệm) — phải phân biệt được
5. **Quên security khi design:** Luôn đề cập VPC, IAM roles, encryption at rest/in transit, PII handling

### Về Phong Cách Trả Lời

1. **Trả lời quá ngắn gọn:** Với câu hỏi so sánh, phải nêu cả trade-offs không chỉ câu trả lời "đúng"
2. **Không đặt câu hỏi làm rõ khi design:** Một senior engineer luôn hỏi về scale, requirements trước khi design
3. **Nói chắc chắn về thứ không biết:** Tốt hơn là nói "Tôi chưa dùng dịch vụ này nhưng từ kiến trúc tôi hiểu là..."
4. **Quên mention monitoring:** Mọi system design đều cần CloudWatch, model monitoring — đừng chỉ nói về training/inference

---

## 📚 Tài Liệu Tham Khảo Nhanh

| Chủ Đề | File Tham Khảo |
| ------- | -------------- |
| 20 câu hỏi phỏng vấn + gợi ý trả lời | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) |
| Kịch bản thiết kế hệ thống AI | [system-design-scenarios.md](./system-design-scenarios.md) |
| Kế hoạch học 90 ngày | [90-day-study-plan.md](./90-day-study-plan.md) |
| Nền tảng AI/ML | [../01-fundamentals/README.md](../01-fundamentals/README.md) |
| SageMaker toàn diện | [../02-sagemaker/README.md](../02-sagemaker/README.md) |
| Bedrock & Generative AI | [../03-bedrock/README.md](../03-bedrock/README.md) |
| MLOps & Model Monitor | [../09-mlops/README.md](../09-mlops/README.md) |
| Advanced: Cost, Security | [../10-advanced/README.md](../10-advanced/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành
