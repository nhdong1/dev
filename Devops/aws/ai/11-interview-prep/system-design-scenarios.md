# Kịch Bản Thiết Kế Hệ Thống AI — System Design Scenarios

> 5 kịch bản thiết kế hệ thống AI thực tế thường xuất hiện trong phỏng vấn kỹ thuật cấp Senior/Principal Engineer. Mỗi kịch bản có framework phân tích, kiến trúc đề xuất, trade-offs và điểm hỏi thêm của interviewer.

---

## 📋 Danh Sách Kịch Bản

| # | Tên Kịch Bản | Dịch Vụ Chính | Độ Phức Tạp |
|---|-------------|---------------|-------------|
| 1 | Hệ Thống Gợi Ý Sản Phẩm Thời Gian Thực | Personalize, SageMaker, Kinesis | ⭐⭐⭐ |
| 2 | RAG Chatbot Doanh Nghiệp | Bedrock, OpenSearch, Lambda | ⭐⭐⭐ |
| 3 | Đường Ống Kiểm Duyệt Nội Dung Tự Động | Rekognition, Comprehend, Step Functions | ⭐⭐ |
| 4 | Nền Tảng Phân Tích Giọng Nói Tổng Đài | Transcribe, Comprehend, Kinesis | ⭐⭐⭐ |
| 5 | MLOps Platform Tự Động Hóa Toàn Diện | SageMaker Pipelines, Model Monitor, CodePipeline | ⭐⭐⭐ |

---

## 🎯 Framework Phân Tích Kịch Bản (Áp Dụng Cho Mọi Câu Hỏi)

```
Bước 1 — Clarify Requirements (2-3 phút)
  □ Scale: bao nhiêu users? bao nhiêu requests/giây?
  □ Latency: real-time (< 100ms) hay near-real-time (< 5s) hay batch?
  □ Data volume: GB hay TB? Tần suất cập nhật data?
  □ Accuracy vs Latency vs Cost tradeoff — ưu tiên gì?
  □ Compliance: GDPR, HIPAA, PCI-DSS?
  □ Multi-region? HA requirement?

Bước 2 — High-Level Architecture (5 phút)
  □ Vẽ data flow: Input → Processing → Storage → Output
  □ Chọn dịch vụ và giải thích lý do
  □ Xác định điểm tích hợp giữa các dịch vụ

Bước 3 — Deep Dive (15 phút)
  □ Training pipeline (nếu có custom model)
  □ Inference pipeline
  □ Storage strategy
  □ Monitoring & observability

Bước 4 — Trade-offs (5 phút)
  □ Tại sao chọn approach này vs alternatives?
  □ Điểm yếu của kiến trúc này?
  □ Khi nào sẽ revisit kiến trúc?

Bước 5 — Cost & Scale (3 phút)
  □ Ước tính chi phí xấp xỉ
  □ Bottleneck tiềm năng khi scale 10x?
```

---

## 📐 Kịch Bản 1: Hệ Thống Gợi Ý Sản Phẩm Thời Gian Thực

### Đề Bài

> "Thiết kế hệ thống recommendation (gợi ý sản phẩm) cho một e-commerce platform — Nền Tảng Thương Mại Điện Tử với 5 triệu người dùng active, 2 triệu sản phẩm, target latency < 200ms cho mỗi request. Hệ thống cần học từ user behavior theo thời gian thực."

### Clarify Requirements

- **Scale:** 5M users, 2M items, peak: ~10.000 requests/giây (request per second) lúc flash sale
- **Latency:** < 200ms end-to-end
- **Data:** Click stream — luồng sự kiện click, purchase events, view history
- **Freshness:** Real-time behavior update trong vòng 5 phút
- **Goals:** Click-through rate — CTR (Tỷ Lệ Click) tăng, doanh thu tăng

### Kiến Trúc Đề Xuất

```
                    USER REQUEST
                         │
                    API Gateway
                         │
                    Lambda (< 5ms)
                    ┌────┴────────────────────────────────┐
                    │                                     │
              DynamoDB                          Amazon Personalize
           (User Features)               (Recommendation Engine)
                    │                                     │
                    └────────────┬────────────────────────┘
                                 │
                          Merge & Rank
                    (Business rules, price filter)
                                 │
                          Response: [item_id, score]

    ┌─────────────────────────────────────────────────────────────┐
    │                  REAL-TIME DATA PIPELINE                    │
    │                                                             │
    │  User Action → Kinesis Data Streams → Lambda               │
    │                                           │                │
    │                                   Personalize              │
    │                                PutEvents API               │
    │                              (incremental update)          │
    └─────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────┐
    │                BATCH TRAINING PIPELINE                      │
    │                                                             │
    │  S3 (historical data) → Personalize Dataset Group          │
    │  → Train Solution (HRNN — Hierarchical RNN recipe)         │
    │  → Deploy Campaign → Inference endpoint                    │
    └─────────────────────────────────────────────────────────────┘
```

### Chi Tiết Từng Thành Phần

**Amazon Personalize:**
- **Dataset Group:** Users dataset (attributes), Items dataset (category, price), Interactions dataset (clicks, purchases, views)
- **Recipe:** `aws-hrnn-coldstart` (xử lý cold start — người dùng hoặc sản phẩm mới) hoặc `aws-user-personalization`
- **Campaign:** Deploy với `minProvisionedTPS` (transactions per second tối thiểu) = 100
- **PutEvents API:** Gửi real-time events để model học ngay lập tức (incremental update)

**Kinesis Data Streams:**
- Nhận user events (click, add-to-cart, purchase) từ frontend với latency < 1 giây
- 5 shards — mỗi shard 1MB/s throughput

**DynamoDB (User Context Store):**
- Lưu real-time session context: location, device, cart contents
- Read latency < 1ms với DAX — DynamoDB Accelerator (Bộ Tăng Tốc DynamoDB) cache

**Lambda Orchestration:**
- Gọi song song Personalize + DynamoDB → merge kết quả → áp dụng business rules (filter sản phẩm hết hàng, filter theo category đang xem)

### Trade-offs

| Quyết Định | Chọn | Thay Thế | Lý Do |
|-----------|------|----------|-------|
| Recommendation engine | Amazon Personalize | SageMaker custom model | Personalize có built-in cold start, không cần ML expertise, managed |
| Real-time events | Kinesis | SQS | Kinesis phù hợp stream processing; SQS message queue không stream |
| User features | DynamoDB | ElastiCache Redis | DynamoDB dễ scale và durability hơn cho persistent features |
| Model refresh | Incremental via PutEvents | Full retrain hàng ngày | PutEvents đủ fresh, full retrain tốn kém |

### Interviewer Có Thể Hỏi Thêm

- *"Xử lý cold start — người dùng mới chưa có interaction như thế nào?"*
  → Personalize `aws-hrnn-coldstart` recipe, hoặc recommend popular items, hoặc dùng demographic-based filtering

- *"Nếu Personalize không đủ accuracy cho use case rất đặc thù?"*
  → Kết hợp: Personalize cho baseline, SageMaker custom model (two-tower neural network) cho re-ranking

- *"Tại sao không dùng SageMaker ngay từ đầu?"*
  → Time-to-market. Personalize MVP trong 2 tuần vs SageMaker custom cần 2-3 tháng

---

## 📐 Kịch Bản 2: RAG Chatbot Doanh Nghiệp (Enterprise RAG Chatbot)

### Đề Bài

> "Thiết kế chatbot Q&A cho doanh nghiệp 50.000 nhân viên, dựa trên tài liệu nội bộ (policies, procedures, HR documents) — tổng 2TB tài liệu PDF/Word. Chỉ nhân viên được xác thực mới truy cập được. Latency < 8 giây. Hỗ trợ tiếng Anh và tiếng Việt."

### Clarify Requirements

- **Users:** 50K employees, ~5K concurrent users giờ cao điểm
- **Documents:** 2TB, nhiều loại: PDF, Word, PowerPoint, có phân quyền (HR chỉ thấy HR docs)
- **Languages:** Tiếng Anh + tiếng Việt
- **Security:** SSO — Single Sign-On (Đăng Nhập Một Lần) với corporate identity provider
- **Audit:** Log mọi query cho compliance

### Kiến Trúc Đề Xuất

```
    Employee
        │ (SSO via Cognito + SAML federation)
    CloudFront (global CDN)
        │
    API Gateway + WAF (Web Application Firewall)
        │
    Lambda (Orchestration Layer)
        │
    ┌───┼──────────────────────────────────────────┐
    │                                              │
    │  1. Permission Check                    2. RAG Pipeline
    │     DynamoDB                               │
    │  (user → allowed_document_categories)      │
    │                                            │
    │                              Bedrock Knowledge Base
    │                                  (RAG Engine)
    │                                       │
    │                        OpenSearch Serverless
    │                         (Vector Store)
    │                                       │
    │                     Titan Embeddings (multi-lingual)
    │                                       │
    │                          Bedrock Claude 3 (LLM)
    │                                       │
    │                        Bedrock Guardrails
    │                      (content filter + PII redaction)
    └───────────────────────────────────────────────┘
        │
    Response + Citations (Nguồn Tài Liệu)
        │
    Employee

    ┌─────────────────────────────────────────────────────────────┐
    │                  DOCUMENT INGESTION PIPELINE                │
    │                                                             │
    │  SharePoint/S3 → EventBridge → Lambda → Textract           │
    │  → Chunk & Embed (Titan) → OpenSearch (với metadata:       │
    │    department, classification_level, last_updated)          │
    └─────────────────────────────────────────────────────────────┘
```

### Chi Tiết Kiến Trúc

**Permission-Aware RAG (RAG Có Phân Quyền):**
- Mỗi document chunk trong OpenSearch có metadata: `department`, `classification`, `allowed_roles`
- OpenSearch filter query: tìm chunks có `allowed_roles` chứa role của user đang hỏi
- Bảo đảm nhân viên HR không thấy tài liệu Finance và ngược lại

**Multilingual Support (Hỗ Trợ Đa Ngôn Ngữ):**
- Titan Embeddings v2 hỗ trợ 100+ ngôn ngữ — embed cả tiếng Việt lẫn tiếng Anh
- Claude 3 Sonnet hỗ trợ response tiếng Việt native
- Không cần translate trước — cross-lingual retrieval tự nhiên

**Document Ingestion:**
- Textract → trích xuất text từ scanned PDF, preserve bảng biểu
- Chunking: 512 tokens với 100-token overlap (phần trùng) để không mất context ở ranh giới chunk
- Metadata enrichment: department tag, document date, author → hỗ trợ filter và sorting

**Conversation Memory (Bộ Nhớ Hội Thoại):**
- DynamoDB lưu session history — mỗi session 24 giờ TTL — Time to Live (Thời Gian Sống)
- Inject N turns gần nhất vào context prompt → multi-turn conversation

**Audit & Compliance:**
- CloudTrail log mọi Bedrock API call
- Lambda log: user_id, query, retrieved_docs, response → CloudWatch Logs → S3 (long-term retention)
- Athena query on S3 logs cho audit report

### Trade-offs

| Quyết Định | Chọn | Thay Thế | Lý Do |
|-----------|------|----------|-------|
| Vector Store | OpenSearch Serverless | Pinecone | Native AWS, không cần external service, HIPAA compliant |
| Embedding | Titan Embeddings | OpenAI ada-002 | Không gửi data ra ngoài AWS, multilingual support tốt |
| LLM | Claude 3 Sonnet | Llama 3 | Claude tốt hơn cho factual Q&A với citations |
| Chunking | 512 tokens + overlap | Fixed character | Token-based + overlap đảm bảo context không bị cắt |

### Interviewer Có Thể Hỏi Thêm

- *"Khi document được cập nhật, sync thế nào?"*
  → EventBridge rule detect S3 PutObject event → trigger Lambda re-index → Bedrock Knowledge Base sync (incremental)

- *"Xử lý hallucination — model bịa thông tin như thế nào?"*
  → Guardrails grounding check, system prompt yêu cầu model chỉ trả lời dựa trên context, luôn show citations

- *"2TB documents — OpenSearch Serverless có scale được không?"*
  → OpenSearch Serverless auto-scale theo OCU — OpenSearch Compute Unit. 2TB text sau chunking ~200M vectors → cần ~4-8 OCU, ~$400-800/tháng

---

## 📐 Kịch Bản 3: Đường Ống Kiểm Duyệt Nội Dung Tự Động

### Đề Bài

> "Thiết kế hệ thống content moderation (kiểm duyệt nội dung) cho một mạng xã hội — 10 triệu ảnh và video upload mỗi ngày. Phải phát hiện: ảnh khiêu dâm, bạo lực, hate speech trong caption, và spam. SLA — Service Level Agreement (Thỏa Thuận Mức Dịch Vụ): 95% nội dung xử lý trong 10 giây."

### Kiến Trúc Đề Xuất

```
    User Upload
         │
    S3 (raw uploads)
         │
    EventBridge (S3 PutObject event)
         │
    Step Functions (Orchestration)
         │
    ┌────┼─────────────────────────────────────────────────────┐
    │    │                                                     │
    │  [Parallel State]                                        │
    │    ├── Lambda → Rekognition Image Moderation             │
    │    │            (unsafe content, violence)               │
    │    │                                                     │
    │    ├── Lambda → Rekognition Video (nếu là video)         │
    │    │            (activity detection, shot-by-shot)       │
    │    │                                                     │
    │    └── Lambda → Comprehend                               │
    │                 (caption sentiment, toxicity, entities)  │
    │                                                          │
    │  [Aggregate Results]                                     │
    │    │                                                     │
    │  ┌─┴──────────────────────────────────────────────┐     │
    │  │  Decision Engine (Lambda)                      │     │
    │  │  - Any confidence > 0.95 → Auto-reject         │     │
    │  │  - 0.70 < confidence ≤ 0.95 → Human review     │     │
    │  │  - All < 0.70 → Auto-approve                   │     │
    │  └─────────────────────────────────────────────────┘    │
    └─────────────────────────────────────────────────────────┘
         │
    ┌────┴────────────────────────────────┐
    │                                     │
    S3 (approved content)        SQS Queue (human review)
    + CloudFront CDN                  │
                             Human Reviewer Dashboard
                             (Amazon Augmented AI — A2I)
                             (Đánh Giá Của Con Người)
```

### Chi Tiết Từng Thành Phần

**Rekognition Image Moderation:**
- API: `DetectModerationLabels` → trả về confidence score cho: `Explicit Nudity`, `Violence`, `Visually Disturbing`, `Drugs`, v.v.
- Custom thresholds theo từng category — Violence có thể accept > Nudity

**Rekognition Video Moderation:**
- `StartContentModeration` (async) → nhận JobId → poll hoặc SNS notification khi done
- Analyze từng frame — đưa ra timestamps của frame vi phạm

**Amazon Comprehend:**
- `DetectSentiment`, `DetectToxicContent` (tiếng Anh) → phân tích caption, comment
- Custom Classifier (Bộ Phân Loại Tùy Chỉnh) cho hate speech theo ngôn ngữ cụ thể

**Amazon Augmented AI — A2I (Xem Xét Của Con Người):**
- Tích hợp workflow cho human review khi confidence ở vùng xám
- Reviewers gán nhãn → feedback loop cải thiện model qua Rekognition Custom Labels

**Step Functions Workflow:**
- Parallel state chạy Rekognition + Comprehend đồng thời → giảm latency
- Error handling: Rekognition timeout → fallback sang human review
- Timeout 30 giây per step → SLA 10 giây đạt được với parallel processing

### Trade-offs

| Quyết Định | Chọn | Thay Thế | Lý Do |
|-----------|------|----------|-------|
| Orchestration | Step Functions | SQS + Lambda chain | Step Functions có retry, timeout, parallel state built-in |
| Image moderation | Rekognition | SageMaker custom | Rekognition managed, đủ cho common cases; custom chỉ khi domain đặc thù |
| Human review | Amazon A2I | Custom dashboard | A2I tích hợp Rekognition/Textract, có workforce marketplace |
| Video processing | Rekognition Video | Transcribe + frame extraction | Rekognition Video native, không cần pipeline phức tạp |

---

## 📐 Kịch Bản 4: Nền Tảng Phân Tích Giọng Nói Tổng Đài

### Đề Bài

> "Thiết kế hệ thống phân tích cuộc gọi tổng đài — Contact Center Analytics cho 500 agents, 10.000 cuộc gọi mỗi ngày, mỗi cuộc gọi 5-30 phút. Yêu cầu: transcription real-time, phân tích cảm xúc, phát hiện compliance issues — vấn đề tuân thủ, dashboard cho supervisor."

### Kiến Trúc Đề Xuất

```
    Phone Call (Amazon Connect)
         │ (audio stream)
         │
    ┌────┴──────────────────────────────────────────────────────┐
    │              REAL-TIME STREAMING PATH                     │
    │                                                           │
    │  Kinesis Video Streams (audio) → Transcribe Streaming    │
    │  → Real-time transcript → Lambda                         │
    │  → Compliance Check (keyword detection)                  │
    │  → Alert if compliance phrase missing / detected         │
    │  → DynamoDB (real-time transcript store)                 │
    │  → WebSocket API → Supervisor Dashboard                  │
    └───────────────────────────────────────────────────────────┘

    ┌────────────────────────────────────────────────────────────┐
    │               POST-CALL ANALYTICS PATH                     │
    │                                                            │
    │  Recording (S3) → Transcribe Call Analytics               │
    │                    (phân tích chuyên sâu)                 │
    │                         │                                  │
    │               ┌─────────┴──────────────────────────┐      │
    │               │                                    │      │
    │        Speaker Diarization             Sentiment Analysis  │
    │   (Nhận Dạng Người Nói)             (per turn: agent/cust) │
    │               │                                    │      │
    │        Call Categories             Issue Detection         │
    │   (billing, technical, churn)     (frustration, escalation)│
    │               │                                    │      │
    │               └──────────────┬─────────────────────┘      │
    │                              │                             │
    │                    Kinesis Data Firehose                   │
    │                              │                             │
    │                         S3 (data lake)                     │
    │                              │                             │
    │               ┌──────────────┴────────────────┐           │
    │               │                               │           │
    │             Athena                       QuickSight        │
    │         (ad-hoc query)              (BI Dashboard)         │
    └────────────────────────────────────────────────────────────┘
```

### Chi Tiết Từng Thành Phần

**Amazon Transcribe Call Analytics:**
- Chuyên biệt cho contact center — tốt hơn Transcribe standard
- Built-in: speaker diarization, sentiment per turn, issue detection, action items
- PII Redaction — tự động ẩn số thẻ tín dụng, SSN trong transcript
- Call categories: định nghĩa keyword patterns → tự động tag cuộc gọi

**Real-time Compliance Monitoring:**
- Transcribe Streaming → Lambda → check against compliance script (agent có nói câu disclaimer chưa?)
- Alert qua SNS → Supervisor dashboard WebSocket nếu compliance phrase bị bỏ qua

**Post-Call Enrichment:**
- Comprehend → phân tích chủ đề cuộc gọi (topic modeling — mô hình chủ đề)
- Custom Comprehend Entity Recognizer → nhận dạng product names, error codes đặc thù của công ty

**Data Architecture:**
- S3 data lake với partition (phân vùng): `year/month/day/agent_id/`
- Glue Crawler → tự động cập nhật schema
- Athena → query ad-hoc cho compliance team
- QuickSight → supervisor dashboard: sentiment trend, issue breakdown, agent performance

### KPIs Có Thể Đo Được

- **Customer Sentiment Score** — Average per agent, per product, per day
- **First Call Resolution Rate** — Tỷ lệ giải quyết lần đầu — phát hiện từ "issue resolved" keywords
- **Agent Compliance Rate** — Tỷ lệ tuân thủ script
- **Average Handle Time** — Thời gian xử lý trung bình
- **Escalation Rate** — Tỷ lệ leo thang cuộc gọi

---

## 📐 Kịch Bản 5: MLOps Platform Tự Động Hóa Toàn Diện

### Đề Bài

> "Thiết kế MLOps — ML Operations platform cho một công ty fintech — công nghệ tài chính với 10 data scientists, 5 ML engineers. Platform cần hỗ trợ: training, evaluation, deployment, monitoring tự động; retraining khi phát hiện drift; audit trail (nhật ký kiểm toán) đầy đủ cho regulatory compliance."

### Kiến Trúc Đề Xuất

```
    ┌─────────────────────────────────────────────────────────────┐
    │                    DEVELOPMENT LAYER                        │
    │                                                             │
    │  SageMaker Studio (IDE) → Git (CodeCommit/GitHub)          │
    │  Jupyter Notebooks → SageMaker Experiments (tracking)      │
    └─────────────────────────────────────────────────────────────┘
                              │ Git push trigger
    ┌─────────────────────────┴───────────────────────────────────┐
    │                   CI/CD PIPELINE LAYER                      │
    │                                                             │
    │  CodePipeline ──────────────────────────────────           │
    │       │                                                     │
    │  CodeBuild (unit tests, lint, security scan)               │
    │       │ (on pass)                                           │
    │  SageMaker Pipelines (ML Pipeline)                         │
    │       │                                                     │
    │  ┌────┴──────────────────────────────────────────────────┐ │
    │  │  ProcessingStep → TrainingStep → EvaluationStep       │ │
    │  │       → ConditionStep (accuracy > 0.85?)              │ │
    │  │            → RegisterModelStep (PendingApproval)      │ │
    │  └───────────────────────────────────────────────────────┘ │
    └─────────────────────────────────────────────────────────────┘
                              │
    ┌─────────────────────────┴───────────────────────────────────┐
    │                   DEPLOYMENT LAYER                          │
    │                                                             │
    │  Model Registry (Approval: Manual hoặc Auto)               │
    │       │ (Approved)                                          │
    │  EventBridge → CodePipeline (Deployment)                   │
    │       │                                                     │
    │  SageMaker Endpoint (Blue/Green deployment)                │
    │  ├── Blue endpoint (production — hiện tại)                 │
    │  └── Green endpoint (new model — đang test)                │
    │       │ (shift traffic 10% → 50% → 100% khi stable)        │
    │  Route 53 weighted routing                                 │
    └─────────────────────────────────────────────────────────────┘
                              │
    ┌─────────────────────────┴───────────────────────────────────┐
    │                   MONITORING & FEEDBACK LAYER               │
    │                                                             │
    │  SageMaker Model Monitor (chạy hourly)                     │
    │  ├── DataQualityMonitor → compare with baseline             │
    │  ├── ModelQualityMonitor → precision, recall, F1            │
    │  ├── BiasMonitor → per protected attribute                 │
    │  └── FeatureAttributionMonitor → SHAP drift               │
    │                                                             │
    │  CloudWatch Alarms → SNS → (1) Alert team                 │
    │                           (2) Auto-trigger Retraining      │
    │                               Pipeline                     │
    └─────────────────────────────────────────────────────────────┘
```

### Chi Tiết Từng Thành Phần

**SageMaker Pipelines — Thiết Kế Pipeline:**

```python
# Pipeline steps minh họa
pipeline = Pipeline(
    name="fraud-detection-pipeline",
    parameters=[
        ParameterString(name="TrainingData", default_value="s3://..."),
        ParameterFloat(name="AccuracyThreshold", default_value=0.85),
    ],
    steps=[
        ProcessingStep(name="Preprocess", ...),
        TrainingStep(name="Train", ...),
        ProcessingStep(name="Evaluate", ...),
        ConditionStep(
            name="CheckAccuracy",
            conditions=[
                ConditionGreaterThanOrEqualTo(
                    left=accuracy_metric, right=0.85
                )
            ],
            if_steps=[RegisterModelStep(...)],
            else_steps=[FailStep(message="Accuracy below threshold")]
        )
    ]
)
```

**Model Registry & Approval Workflow:**
- **PendingManualApproval** — data scientist submit → ML lead review metrics + bias report → Approved/Rejected
- **Auto-approval** — có thể configure cho model update nhỏ (< 2% metric change) → không cần manual review
- Metadata lưu: training data S3 URI, metrics, hyperparameters, bias report, Clarify report

**Blue/Green Deployment:**
- Tạo `green` endpoint với model mới
- CloudWatch alarm monitor green: error rate, latency
- Sau 30 phút stable → shift production traffic (xem thêm: `UpdateEndpoint` API với `AutoRollbackConfig`)
- Nếu error rate tăng → tự động rollback về blue

**Retraining Trigger:**
- EventBridge rule: SageMaker Model Monitor violation → trigger SageMaker Pipeline
- Hoặc scheduled: chạy pipeline hàng tuần với fresh data từ Feature Store
- Hoặc data-driven: Glue job đếm new samples → khi đủ N samples mới → trigger pipeline

**Audit Trail cho Financial Compliance:**
- SageMaker ML Lineage Tracking → biết model version X được train từ dataset D, pipeline P, tại thời điểm T
- CloudTrail → mọi API call vào SageMaker, Bedrock, S3
- Model Registry approval history → ai approve, khi nào, với metrics gì
- Giữ lại trong S3 với Object Lock → không thể xóa trong 7 năm (regulatory requirement)

### Trade-offs

| Quyết Định | Chọn | Thay Thế | Lý Do |
|-----------|------|----------|-------|
| ML Orchestration | SageMaker Pipelines | MLflow + Airflow | Tích hợp native SageMaker, lineage tracking built-in |
| CI/CD | CodePipeline | GitHub Actions | Native AWS, IAM integration tốt hơn, no secrets to manage |
| Deployment strategy | Blue/Green | Rolling update | Blue/Green rollback nhanh hơn, zero downtime |
| Retraining trigger | EventBridge rule | Manual | Automation giảm toàn bộ manual overhead |

### Interviewer Có Thể Hỏi Thêm

- *"Làm thế nào để đảm bảo reproducibility — tái hiện kết quả training?"*
  → SageMaker ML Lineage Tracking lưu dataset version (S3 URI + version), hyperparameters, container image version. Bất kỳ lúc nào cũng clone và re-run pipeline.

- *"Khi retraining fail, rollback thế nào?"*
  → Model Registry giữ tất cả versions. Blue endpoint vẫn đang chạy → không có downtime. Alert team và investigate pipeline run trong Studio.

- *"Multi-model trong cùng team thì quản lý thế nào?"*
  → Mỗi model có pipeline riêng, model group riêng trong Model Registry. SageMaker Projects template (chuẩn hóa) cho mỗi model mới.

---

## 💡 Tips Trình Bày Trong Phỏng Vấn

### Luôn Bắt Đầu Bằng Câu Hỏi

```
"Trước khi design, tôi muốn làm rõ vài điểm:
1. Scale: expected load là bao nhiêu? Peak traffic?
2. Latency requirement: real-time (ms) hay near-real-time (giây)?
3. Existing infrastructure: team đang dùng gì? Có thể leverage không?
4. Budget constraints: có optimization requirement không?"
```

### Vẽ Diagram Ngay

- Dùng whiteboard hoặc shared doc
- Bắt đầu từ User → đến Output — không bắt đầu từ middle
- Label mỗi arrow với dữ liệu gì đang chảy qua

### Nói Rõ Trade-offs Trước Khi Được Hỏi

```
"Tôi chọn OpenSearch Serverless thay vì Pinecone vì:
- Native AWS → không cần manage external service
- IAM integration tốt
- Compromise: Pinecone có query performance tốt hơn ở scale cực lớn"
```

### Đề Cập Monitoring Chủ Động

```
"Với production system này, tôi sẽ monitor:
- Latency: P50, P95, P99 trên CloudWatch
- Error rate: 5xx errors từ API Gateway
- Model quality: SageMaker Model Monitor hàng giờ
- Cost: AWS Cost Anomaly Detection để phát hiện spike"
```

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành — 5 kịch bản thiết kế hệ thống đầy đủ
