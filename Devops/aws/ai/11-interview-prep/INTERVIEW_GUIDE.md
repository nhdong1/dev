# Top 20 Câu Hỏi Phỏng Vấn AWS AI/ML — Hướng Dẫn Trả Lời

> Tổng hợp 20 câu hỏi phổ biến nhất trong phỏng vấn vị trí liên quan đến AWS AI/ML, kèm gợi ý trả lời chi tiết, điểm cần nhấn mạnh và sai lầm cần tránh

---

## 📋 Bảng Câu Hỏi Theo Chủ Đề

| # | Câu Hỏi | Chủ Đề | Mức Độ |
|---|---------|--------|--------|
| 1 | AI Services vs SageMaker — khi nào dùng cái nào? | Architecture | ⭐⭐ |
| 2 | RAG vs Fine-tuning — trade-offs? | Bedrock/Generative AI | ⭐⭐⭐ |
| 3 | SageMaker inference types — Real-time vs Batch vs Async vs Serverless | SageMaker | ⭐⭐ |
| 4 | Model drift là gì? Phát hiện và xử lý thế nào? | MLOps | ⭐⭐⭐ |
| 5 | Foundation Model là gì? Khác gì traditional ML model? | Generative AI | ⭐ |
| 6 | Giải thích RAG pipeline từ đầu đến cuối | Bedrock | ⭐⭐⭐ |
| 7 | Spot Training — lợi ích, rủi ro, cách xử lý checkpoint? | SageMaker | ⭐⭐ |
| 8 | Feature Store giải quyết vấn đề gì? Online vs Offline Store? | SageMaker | ⭐⭐⭐ |
| 9 | Prompt Engineering — Zero-shot, Few-shot, Chain-of-thought? | Bedrock | ⭐⭐ |
| 10 | SageMaker Clarify — bias detection và SHAP explainability? | MLOps/Responsible AI | ⭐⭐⭐ |
| 11 | Rekognition vs Textract — khi nào dùng cái nào? | AI Services | ⭐ |
| 12 | MLOps pipeline — các thành phần chính? | MLOps | ⭐⭐⭐ |
| 13 | Bedrock Guardrails — tại sao cần? Cấu hình thế nào? | Bedrock | ⭐⭐ |
| 14 | Amazon Lex vs Bedrock — xây chatbot dùng cái nào? | Conversational AI | ⭐⭐ |
| 15 | Hyperparameter Tuning với SageMaker AMT — hoạt động thế nào? | SageMaker | ⭐⭐ |
| 16 | Responsible AI — fairness, transparency, privacy trên AWS? | Responsible AI | ⭐⭐ |
| 17 | Vector Database là gì? Vai trò trong RAG pipeline? | Bedrock/Architecture | ⭐⭐⭐ |
| 18 | SageMaker Pipelines vs Step Functions — khi nào dùng cái nào? | MLOps | ⭐⭐⭐ |
| 19 | Tối ưu chi phí SageMaker — các kỹ thuật chính? | Cost Optimization | ⭐⭐ |
| 20 | Thiết kế end-to-end Generative AI application production-ready | System Design | ⭐⭐⭐ |

---

## 📖 Câu Hỏi & Gợi Ý Trả Lời Chi Tiết

---

### Câu 1: AI Services vs SageMaker — Khi Nào Dùng Cái Nào?

**Câu hỏi:** "Khi nào bạn sẽ dùng một AI Service có sẵn của AWS (như Rekognition, Comprehend, Transcribe) thay vì train custom model trên SageMaker?"

**Gợi ý trả lời:**

> "Đây là một trade-off (đánh đổi) cơ bản. Tôi sẽ chọn **AI Service có sẵn** khi:
>
> 1. **Use case phổ biến** — bài toán nhận diện khuôn mặt, phân tích cảm xúc, chuyển giọng nói thành văn bản... mà model pre-trained đã giải quyết tốt
> 2. **Không có ML expertise** trong team, hoặc cần MVP — Minimum Viable Product (Sản Phẩm Khả Thi Tối Thiểu) nhanh
> 3. **Không có labeled data** — dữ liệu gán nhãn riêng để train
> 4. **Chi phí vận hành thấp hơn** — không cần quản lý infrastructure, không trả tiền idle endpoint
>
> Tôi sẽ chọn **SageMaker custom model** khi:
> 1. **Domain quá đặc thù** — ảnh y tế, văn bản pháp lý chuyên ngành, ngôn ngữ hiếm
> 2. **AI Service hiện tại không đủ accuracy** — đã thử và kết quả không đạt yêu cầu
> 3. **Cần kiểm soát hoàn toàn** — model weights, inference latency, privacy
> 4. **Có đủ dữ liệu và budget** để training và maintainance
>
> **Ví dụ thực tế:** Phát hiện logo thương hiệu trong ảnh — Rekognition Custom Labels nhanh hơn SageMaker vì có auto-labeling và training giao diện đơn giản, phù hợp với team không có ML engineer."

**Điểm cần nhấn mạnh:**
- Luôn mention "it depends on" — không có câu trả lời tuyệt đối
- Đề cập đến tốc độ phát triển (time-to-market) và chi phí dài hạn

---

### Câu 2: RAG vs Fine-tuning — Trade-offs?

**Câu hỏi:** "Khi nào bạn sẽ dùng RAG — Retrieval-Augmented Generation (Tạo Sinh Tăng Cường Truy Xuất) và khi nào Fine-tune (tinh chỉnh) Foundation Model?"

**Gợi ý trả lời:**

> "RAG và Fine-tuning giải quyết hai vấn đề khác nhau về cơ bản:
>
> **RAG** phù hợp khi:
> - Cần cung cấp **knowledge mới hoặc cập nhật thường xuyên** mà Foundation Model chưa biết (sau cutoff date — ngày huấn luyện cuối)
> - Cần **trích dẫn nguồn** (citations) — người dùng cần biết thông tin lấy từ đâu
> - Dữ liệu **nhạy cảm hoặc proprietary** (độc quyền) — không muốn đưa vào training
> - Cần deploy **nhanh** — không cần training pipeline
>
> **Fine-tuning** phù hợp khi:
> - Muốn thay đổi **tone/style** của model (ví dụ: luôn trả lời ngắn gọn như FAQ)
> - Model cần học **format output đặc biệt** (ví dụ: JSON cụ thể, code theo convention nội bộ)
> - Cần **domain knowledge sâu** mà RAG không đủ context window để cung cấp
> - Cần **giảm latency** — inference không cần retrieval step
>
> **Tôi thường khuyến nghị RAG trước** — ít tốn kém, dễ cập nhật data, dễ debug. Chỉ Fine-tune khi RAG không đủ.
>
> **Kết hợp cả hai:** Fine-tuned model + RAG là pattern mạnh nhất — model học domain language, RAG cung cấp facts cụ thể."

**Điểm cần nhấn mạnh:**
- RAG giải quyết vấn đề **knowledge** — Fine-tuning giải quyết vấn đề **behavior/style**
- Đề cập chi phí: Fine-tuning trên Bedrock tốn $0.0008/1K tokens training + compute

---

### Câu 3: SageMaker Inference Types — Real-time vs Batch vs Async vs Serverless

**Câu hỏi:** "Giải thích 4 loại inference của SageMaker và khi nào dùng mỗi loại?"

**Gợi ý trả lời:**

> | Loại | Latency | Use Case | Giới Hạn |
> |------|---------|----------|----------|
> | **Real-time** | < 60s, synchronous | API inference, live recommendation | Payload ≤ 6MB, không idle tốt |
> | **Batch Transform** | Hours | Offline scoring, bulk predictions | Không real-time |
> | **Async** | Minutes | Large payload, long inference | Queue-based, không instant |
> | **Serverless** | Cold start ~1s | Sporadic traffic, cost-sensitive | Payload ≤ 4MB, không consistent |
>
> **Chọn theo bài toán:**
> - **Real-time:** E-commerce product recommendation khi user đang browse → cần < 100ms
> - **Batch Transform:** Xử lý 1 triệu transaction cuối tháng để detect fraud → không cần real-time
> - **Async Inference:** Phân tích video dài 1 giờ, transcription → model chạy lâu, queue kết quả vào S3
> - **Serverless:** Internal tool chạy ban ngày, idle ban đêm → không trả tiền khi không có traffic"

**Điểm cần nhấn mạnh:**
- Serverless có **cold start** (~1-5 giây đầu) — không phù hợp cho latency-sensitive apps
- Batch Transform **không có endpoint** — chạy xong tự xóa, không tốn tiền

---

### Câu 4: Model Drift — Phát Hiện và Xử Lý Thế Nào?

**Câu hỏi:** "Model drift (trôi dạt mô hình) là gì? Làm thế nào để phát hiện và xử lý?"

**Gợi ý trả lời:**

> "Model drift xảy ra khi **performance (hiệu suất) của model giảm theo thời gian** trong production. Có 2 loại:
>
> **Data Drift (Trôi Dạt Dữ Liệu):** Input distribution thay đổi — ví dụ model nhận diện sản phẩm được train trên ảnh studio, nhưng production nhận ảnh điện thoại chụp tay.
>
> **Concept Drift (Trôi Dạt Khái Niệm):** Mối quan hệ giữa input và output thay đổi — ví dụ model churn prediction (dự đoán rời bỏ) train trước COVID, sau COVID hành vi khách hàng thay đổi hoàn toàn.
>
> **Phát hiện bằng SageMaker Model Monitor:**
> - `DataQualityMonitor` — so sánh statistical baseline (phân phối baseline) với production data
> - `ModelQualityMonitor` — theo dõi accuracy, precision, recall theo thời gian (cần ground truth labels)
> - `BiasMonitor` — phát hiện bias drift (thiên lệch trôi dạt) cho các protected attributes
> - `FeatureAttributionMonitor` — SHAP value drift — feature tầm quan trọng thay đổi thế nào
>
> **Xử lý:**
> 1. **Alert sớm** — CloudWatch alarm khi metrics vượt ngưỡng
> 2. **Retraining pipeline** — SageMaker Pipelines trigger tự động khi phát hiện drift
> 3. **Shadow deployment** — deploy model mới song song, so sánh output trước khi replace
> 4. **A/B testing** — route một phần traffic sang model mới để validate"

---

### Câu 5: Foundation Model Là Gì? Khác Gì Traditional ML?

**Câu hỏi:** "Foundation Model — Mô Hình Nền Tảng là gì? Tại sao nó quan trọng?"

**Gợi ý trả lời:**

> "Foundation Model là **model quy mô lớn được pre-train trên lượng dữ liệu khổng lồ** (text, images, code), có thể fine-tune hoặc prompt để thực hiện nhiều tác vụ khác nhau mà không cần train lại từ đầu.
>
> **Khác biệt chính với traditional ML:**
>
> | Khía Cạnh | Traditional ML | Foundation Model |
> |-----------|----------------|-----------------|
> | **Training data** | Task-specific (cụ thể) | General, massive scale |
> | **Capabilities** | Một nhiệm vụ | Đa nhiệm (multi-task) |
> | **Customization** | Retrain toàn bộ | Prompt / Fine-tune nhẹ |
> | **Infrastructure** | Quản lý được | Cần GPU cực lớn |
>
> **Tại sao quan trọng:**
> - **Transfer learning (Học Chuyển Giao) cực mạnh** — một model giải quyết nhiều bài toán
> - **Giảm chi phí data labeling** — không cần labeled data cho mỗi task
> - **Emergence** — khả năng xuất hiện tự nhiên không được train cụ thể (reasoning, code)
>
> **Trên AWS:** Amazon Bedrock cung cấp Claude (Anthropic), Llama 3 (Meta), Titan (Amazon), Stable Diffusion qua API — không cần quản lý infrastructure."

---

### Câu 6: Giải Thích RAG Pipeline Từ Đầu Đến Cuối

**Câu hỏi:** "Mô tả chi tiết cách RAG — Retrieval-Augmented Generation hoạt động, từng bước từ indexing đến response."

**Gợi ý trả lời:**

> "RAG có 2 giai đoạn chính: **Indexing (Lập Chỉ Mục)** và **Querying (Truy Vấn)**.
>
> **Giai Đoạn 1 — Indexing (chuẩn bị dữ liệu):**
> 1. Load documents (PDF, Word, web pages) → S3 hoặc nguồn data
> 2. Chunk (chia nhỏ) documents thành đoạn 512-1024 tokens
> 3. Embed (nhúng) mỗi chunk thành vector bằng embedding model (Titan Embeddings, OpenAI)
> 4. Lưu vectors vào Vector Store — có thể là OpenSearch Serverless, Pinecone, pgvector
>
> **Giai Đoạn 2 — Querying (khi user hỏi):**
> 1. User gửi query (câu hỏi)
> 2. Embed query thành vector bằng **cùng embedding model**
> 3. Tìm kiếm vector gần nhất (nearest neighbor) trong Vector Store → lấy ra K chunks liên quan nhất
> 4. Ghép chunks vào Context Prompt: `[Retrieved Context] + [User Question]`
> 5. Gửi prompt đến LLM (Claude, Titan, Llama) để generate response (tạo câu trả lời)
>
> **Trên AWS:** Amazon Bedrock Knowledge Base tự động hóa toàn bộ pipeline này — kết nối S3, embedding model, OpenSearch Serverless."

---

### Câu 7: Spot Training — Lợi Ích, Rủi Ro, Checkpoint?

**Câu hỏi:** "Spot Training (Huấn Luyện Spot) trên SageMaker là gì? Rủi ro và cách xử lý?"

**Gợi ý trả lời:**

> "Spot Training dùng **EC2 Spot Instances — các instance dự phòng** được AWS bán với giá thấp hơn On-demand 60-90%.
>
> **Lợi ích:** Tiết kiệm chi phí training rất lớn — training job 8 giờ giảm từ $100 xuống $20-30.
>
> **Rủi ro:** Spot Instance có thể bị **interrupted (gián đoạn)** bất kỳ lúc nào khi AWS cần capacity. Job training bị dừng giữa chừng.
>
> **Cách xử lý Interruption:**
> 1. **Checkpointing (Lưu Điểm Kiểm Tra)** — SageMaker lưu model state định kỳ vào S3. Khi job restart, load checkpoint và tiếp tục từ đó, không mất toàn bộ training
> 2. **`MaxWaitTimeInSeconds`** — set thời gian tối đa chờ Spot capacity
> 3. **`checkpoint_s3_uri`** — chỉ định S3 path cho checkpoints
>
> **Khi nào dùng Spot:**
> - Jobs có thể restart được (không cần real-time deadline)
> - Jobs dài (> 1 giờ) — tiết kiệm nhiều hơn
> - Không nên dùng cho production inference endpoint"

---

### Câu 8: Feature Store — Online vs Offline Store?

**Câu hỏi:** "SageMaker Feature Store giải quyết vấn đề gì? Online Store và Offline Store khác nhau thế nào?"

**Gợi ý trả lời:**

> "Feature Store (Kho Đặc Trưng) giải quyết 3 vấn đề lớn trong ML production:
> 1. **Feature skew (Lệch Đặc Trưng)** — features dùng lúc training khác lúc inference
> 2. **Duplicate computation** — mỗi team tính cùng một feature theo cách khác nhau
> 3. **Reproducibility (Khả Năng Tái Hiện)** — không thể tái tạo lại training set tại thời điểm T
>
> **Online Store:**
> - Lưu trữ: **DynamoDB** — low latency
> - Latency: single-digit milliseconds (mili giây một chữ số)
> - Dùng cho: **Real-time inference** — lấy features nhanh khi có request
>
> **Offline Store:**
> - Lưu trữ: **S3** (Parquet format) — đây là S3 bucket của bạn
> - Dùng cho: **Model training, batch scoring** — query với Athena hoặc Spark
> - Đặc biệt: hỗ trợ **Point-in-time query** — lấy feature value tại thời điểm T trong quá khứ để training không bị data leakage (rò rỉ dữ liệu)
>
> **Pattern thực tế:** User clicks được ingested (nhập) real-time vào Online Store (cho inference), và đồng thời replicate sang Offline Store (cho training job hàng tuần)."

---

### Câu 9: Prompt Engineering — Zero-shot, Few-shot, Chain-of-thought?

**Câu hỏi:** "Giải thích các kỹ thuật Prompt Engineering cơ bản với ví dụ?"

**Gợi ý trả lời:**

> "**Zero-shot prompting:** Hỏi thẳng không có ví dụ:
> ```
> Classify the sentiment: 'The product broke after 2 days.'
> ```
>
> **Few-shot prompting:** Cung cấp vài ví dụ (shots) để hướng dẫn model:
> ```
> 'Amazing quality!' → Positive
> 'Terrible experience' → Negative
> 'The product broke after 2 days.' → ?
> ```
>
> **Chain-of-thought (Chuỗi Suy Luận) prompting:** Yêu cầu model suy luận từng bước:
> ```
> 'Giải thích từng bước tại sao đây là vấn đề về security, sau đó đưa ra kết luận'
> ```
>
> **Các kỹ thuật nâng cao:**
> - **System prompt:** Định nghĩa role của model — 'Bạn là chuyên gia bảo mật AWS...'
> - **Temperature (Nhiệt Độ):** 0 = deterministic (quyết định); 1 = creative (sáng tạo)
> - **Top-K / Top-P sampling (Lấy Mẫu):** Kiểm soát diversity của output
> - **Prompt injection prevention:** Validate input user, dùng Bedrock Guardrails để filter
>
> **Best practices:** Specific, structured, with clear output format. Tệ: 'Tóm tắt'. Tốt: 'Tóm tắt trong 3 bullet points ngắn gọn, mỗi bullet ≤ 15 từ'."

---

### Câu 10: SageMaker Clarify — Bias Detection và SHAP?

**Câu hỏi:** "SageMaker Clarify làm gì? Giải thích bias detection và SHAP explainability?"

**Gợi ý trả lời:**

> "SageMaker Clarify có 2 tính năng chính:
>
> **1. Bias Detection (Phát Hiện Thiên Lệch):**
> - **Pre-training bias:** Phân tích dataset trước khi train — kiểm tra có imbalance (mất cân bằng) về class labels cho các nhóm không?
> - **Post-training bias:** Phân tích model predictions — model có đưa ra predictions kém hơn cho một nhóm dân số (giới tính, tuổi, sắc tộc) không?
> - Metrics: DPL — Difference in Positive Proportions (Chênh Lệch Tỷ Lệ Tích Cực), FDR, FPR, TPR theo nhóm
>
> **2. SHAP — SHapley Additive exPlanations (Giải Thích Bằng Đóng Góp Shapley):**
> - Giải thích **tại sao model đưa ra prediction cụ thể đó**
> - SHAP value của mỗi feature = đóng góp của feature đó vào prediction
> - Ví dụ: Model từ chối khoản vay → SHAP cho thấy 'số năm làm việc' đóng góp -0.3 (giảm xác suất approve), 'thu nhập' đóng góp +0.6
>
> **Tại sao quan trọng:**
> - Regulatory compliance (Tuân Thủ Quy Định) — GDPR, Equal Credit Opportunity Act yêu cầu giải thích được quyết định
> - Debug model — hiểu tại sao model fail
> - Trust — business stakeholders tin tưởng model hơn khi hiểu lý do"

---

### Câu 11: Rekognition vs Textract — Khi Nào Dùng Cái Nào?

**Câu hỏi:** "Amazon Rekognition và Amazon Textract đều xử lý images. Khác nhau thế nào?"

**Gợi ý trả lời:**

> "Đây là câu hỏi hay về hiểu biết dịch vụ:
>
> **Amazon Rekognition:**
> - Dùng cho **ảnh tự nhiên (natural images)** — không phải tài liệu
> - Use cases: Phát hiện object/scene, nhận diện khuôn mặt, so sánh face (face comparison), content moderation (kiểm duyệt nội dung), celebrity recognition, PPE — Personal Protective Equipment (Thiết Bị Bảo Hộ Cá Nhân) detection
> - `detect-text` API có thể đọc text trong ảnh tự nhiên nhưng **không hiểu cấu trúc tài liệu**
>
> **Amazon Textract:**
> - Dùng cho **tài liệu có cấu trúc (structured documents)** — forms, tables, invoices, contracts
> - Hiểu layout tài liệu: form fields (key-value pairs), table rows/columns, check boxes
> - APIs: `AnalyzeDocument` (forms+tables), `DetectDocumentText` (raw OCR), `AnalyzeExpense` (hóa đơn), `AnalyzeID` (chứng minh thư)
>
> **Quy tắc chọn:**
> - Ảnh kho hàng → phát hiện sản phẩm → **Rekognition**
> - Scan hợp đồng → trích xuất ngày ký, các điều khoản → **Textract**
> - Ảnh đường phố với biển quảng cáo → đọc text trên biển → **Rekognition detect-text**
> - Form PDF đăng ký y tế → trích xuất tên, ngày sinh, số bảo hiểm → **Textract AnalyzeDocument**"

---

### Câu 12: MLOps Pipeline — Các Thành Phần Chính?

**Câu hỏi:** "Mô tả một MLOps — ML Operations pipeline đầy đủ. Các thành phần chính là gì?"

**Gợi ý trả lời:**

> "MLOps pipeline đầy đủ bao gồm:
>
> **1. Data Pipeline (Đường Ống Dữ Liệu):**
> - Data ingestion (nhập dữ liệu) → S3
> - Data validation — AWS Glue Data Quality hoặc Great Expectations
> - Feature engineering → SageMaker Processing Job
> - Feature store → SageMaker Feature Store
>
> **2. Training Pipeline (Đường Ống Huấn Luyện) — SageMaker Pipelines:**
> - `ProcessingStep` → preprocessing
> - `TrainingStep` → train model
> - `EvaluationStep` → calculate metrics
> - `ConditionStep` → nếu accuracy > threshold → tiếp tục register
> - `RegisterModelStep` → lưu vào Model Registry với status PendingManualApproval
>
> **3. Deployment Pipeline (Đường Ống Triển Khai):**
> - Manual hoặc auto approval trong Model Registry → Approved
> - CI/CD trigger (GitHub Actions hoặc CodePipeline) → deploy endpoint mới
> - Blue/Green deployment hoặc Canary release (phát hành theo tỷ lệ dần dần)
>
> **4. Monitoring (Giám Sát):**
> - SageMaker Model Monitor → data drift, model quality, bias drift
> - CloudWatch → endpoint metrics (latency, error rate, invocations)
> - Alert → trigger retraining pipeline nếu drift vượt ngưỡng
>
> Toàn bộ pipeline này là **fully automated (tự động hoàn toàn)** — khi có data mới đủ lượng, pipeline tự trigger, train, evaluate, và đề xuất deploy."

---

### Câu 13: Bedrock Guardrails — Tại Sao Cần?

**Câu hỏi:** "Amazon Bedrock Guardrails là gì? Khi nào và cách cấu hình?"

**Gợi ý trả lời:**

> "Bedrock Guardrails là layer bảo vệ (safety layer) giữa user và Foundation Model, giúp **đảm bảo AI response an toàn, phù hợp với policy doanh nghiệp**.
>
> **Các tính năng chính:**
> - **Content Filters (Bộ Lọc Nội Dung):** Chặn hate speech, violence, sexual content — có thể cấu hình mức độ từ Low đến High
> - **Denied Topics (Chủ Đề Bị Từ Chối):** Định nghĩa topic không được phép — ví dụ chatbot ngân hàng không được tư vấn cổ phiếu
> - **Word Filters (Bộ Lọc Từ):** Danh sách từ/cụm từ bị chặn (competitors, offensive terms)
> - **PII Redaction (Che Giấu PII):** Tự động ẩn số CMND, email, số điện thoại trong response
> - **Grounding Check:** Kiểm tra response có dựa trên context được cung cấp không (chống hallucination — ảo giác AI)
>
> **Khi nào cần:**
> - Mọi Generative AI app public-facing đều cần Guardrails
> - App cho trẻ em → strict content filter
> - Chatbot doanh nghiệp → denied topics + PII redaction
> - Customer service bot → grounding check để không hallucinate thông tin sản phẩm
>
> **Cách áp dụng:** Attach Guardrail ID vào API call — áp dụng cho cả user input và model output."

---

### Câu 14: Amazon Lex vs Bedrock — Xây Chatbot Dùng Cái Nào?

**Câu hỏi:** "Nếu cần xây một chatbot, bạn sẽ chọn Amazon Lex hay Amazon Bedrock? Lý do?"

**Gợi ý trả lời:**

> "Đây phụ thuộc vào yêu cầu của chatbot:
>
> **Chọn Amazon Lex khi:**
> - Chatbot có **task-oriented, structured conversation flow** — đặt vé máy bay, kiểm tra số dư tài khoản
> - Cần **tight control over dialog** — biết trước user sẽ nói gì, validate slot values (số ghế, ngày bay)
> - Tích hợp **Amazon Connect** cho contact center, IVR — Interactive Voice Response (Phản Hồi Giọng Nói Tương Tác)
> - Cần **compliance** — Lex có SOC, PCI DSS
>
> **Chọn Amazon Bedrock khi:**
> - Chatbot cần **open-ended conversation** — Q&A tự do, không có dialog flow cố định
> - Cần tích hợp **knowledge base (RAG)** — chatbot tư vấn dựa trên tài liệu nội bộ
> - Cần **generation capability** — tạo email, tóm tắt tài liệu
> - Cần **multi-turn memory** phức tạp, reasoning
>
> **Kết hợp cả hai:** Lex làm **dialog management** (quản lý hội thoại) → route intent phức tạp sang Bedrock để generate response. Ví dụ: Lex nhận dạng intent 'book_flight', nhưng user hỏi 'recommend me the best route for family with kids' → Lex call Lambda → Lambda call Bedrock để generate recommendation."

---

### Câu 15: Hyperparameter Tuning với SageMaker AMT?

**Câu hỏi:** "Giải thích SageMaker AMT — Automatic Model Tuning (Tinh Chỉnh Mô Hình Tự Động) hoạt động thế nào?"

**Gợi ý trả lời:**

> "SageMaker AMT tự động tìm **tập hyperparameters tốt nhất** để tối ưu một objective metric (ví dụ: validation accuracy).
>
> **Cách hoạt động:**
> 1. Định nghĩa **hyperparameter ranges** — ví dụ learning_rate: [0.001, 0.1], epochs: [10, 100]
> 2. Chọn **strategy (chiến lược):**
>    - `Bayesian (Bayes)` — học từ kết quả trước để chọn tham số tiếp theo thông minh hơn (mặc định, hiệu quả nhất)
>    - `Random Search` — ngẫu nhiên trong range
>    - `Grid Search` — duyệt tất cả combinations (chậm)
>    - `Hyperband` — early stopping jobs kém để tập trung vào jobs tốt
> 3. SageMaker chạy **parallel training jobs** (đồng thời nhiều jobs)
> 4. Kết thúc → trả về best hyperparameters
>
> **Giới hạn quan trọng:**
> - `MaxNumberOfTrainingJobs` — tổng số jobs chạy
> - `MaxParallelTrainingJobs` — số jobs song song (trade-off: nhiều song song = nhanh hơn nhưng kém efficient hơn với Bayesian)
>
> **Best practice:** Bắt đầu với phạm vi rộng (1 log scale), sau đó narrow down. Dùng Early Stopping để tiết kiệm cost."

---

### Câu 16: Responsible AI — Fairness, Transparency, Privacy?

**Câu hỏi:** "Bạn hiểu Responsible AI là gì? AWS hỗ trợ Responsible AI thế nào?"

**Gợi ý trả lời:**

> "Responsible AI là tập hợp nguyên tắc đảm bảo hệ thống AI **công bằng, minh bạch, an toàn và tôn trọng quyền riêng tư**.
>
> **Các trụ cột chính:**
> - **Fairness (Công Bằng):** Model không phân biệt đối xử với protected groups — SageMaker Clarify bias detection
> - **Transparency (Minh Bạch):** Giải thích được tại sao model đưa ra quyết định — SHAP explainability
> - **Privacy (Quyền Riêng Tư):** Không dùng PII vào training hoặc expose trong output — Macie detect PII, Comprehend Medical PHI redaction
> - **Robustness (Mạnh Mẽ):** Model hoạt động ổn định khi có adversarial inputs — Bedrock Guardrails
> - **Accountability (Trách Nhiệm Giải Trình):** Audit trail đầy đủ — SageMaker ML Lineage, CloudTrail
>
> **AWS Tools:**
> - SageMaker Clarify → bias + explainability
> - Bedrock Guardrails → content filtering, PII redaction
> - SageMaker Model Monitor → detect drift và degradation
> - Amazon Macie → scan S3 cho PII trong training data
> - CloudTrail → audit mọi API call
>
> **Trong thực tế:** Tôi sẽ chạy Clarify bias analysis trước khi deploy production model, đặt thresholds cụ thể (FPR < 0.05 cho nhóm minority), và set up automated re-evaluation mỗi tháng."

---

### Câu 17: Vector Database Là Gì? Vai Trò Trong RAG?

**Câu hỏi:** "Vector Database — Cơ Sở Dữ Liệu Vector là gì? Tại sao RAG cần nó?"

**Gợi ý trả lời:**

> "Vector Database lưu trữ và tìm kiếm **vector embeddings (nhúng vector)** — biểu diễn dạng số của text/images.
>
> **Tại sao cần trong RAG:**
> Khi user hỏi 'Chính sách hoàn tiền là gì?', ta không thể tìm bằng keyword search vì document có thể ghi 'refund policy' hoặc 'money-back guarantee'. Cần **semantic search (tìm kiếm ngữ nghĩa)** — tìm nội dung có ý nghĩa tương đồng, không phải giống từ ngữ.
>
> **Cách Vector Search hoạt động:**
> 1. Text → Embedding Model → Vector [0.2, -0.5, 0.8, ...] (768 hoặc 1536 chiều)
> 2. Lưu vector vào Vector Store cùng metadata
> 3. Khi query: embed query → tìm vectors gần nhất bằng **cosine similarity hoặc dot product**
>
> **Các Vector Store trên AWS:**
> - **OpenSearch Serverless** với k-NN plugin — native AWS, fully managed
> - **Amazon Aurora pgvector** — SQL database + vector search
> - **RDS PostgreSQL pgvector** — self-managed
> - **Third-party:** Pinecone, Weaviate, Qdrant (chạy trên EC2 hoặc EKS)
>
> **Amazon Bedrock Knowledge Base** tự động tích hợp OpenSearch Serverless — không cần setup thủ công."

---

### Câu 18: SageMaker Pipelines vs Step Functions — Khi Nào Dùng?

**Câu hỏi:** "SageMaker Pipelines và AWS Step Functions đều orchestrate workflows. Khác nhau gì?"

**Gợi ý trả lời:**

> "**SageMaker Pipelines:**
> - Được thiết kế **đặc biệt cho ML workflows**
> - Native integration với mọi SageMaker API (Training, Processing, Transform, Register)
> - Có visual DAG — Directed Acyclic Graph (Đồ Thị Không Chu Trình Có Hướng) editor trong Studio
> - Tracking tự động: lưu run history, parameters, metrics, artifacts
> - **ML Lineage** — biết model đến từ data nào, pipeline nào
> - Caching — step đã chạy không cần chạy lại nếu input không đổi
>
> **AWS Step Functions:**
> - General-purpose workflow orchestrator (điều phối workflow đa mục đích)
> - Kết hợp bất kỳ AWS service — Lambda, ECS, DynamoDB, SQS, Glue, **và SageMaker**
> - Phức tạp hơn để setup cho pure ML workflow
> - Phù hợp khi pipeline kết hợp nhiều bước không phải ML (ví dụ: gửi email, cập nhật database)
>
> **Khi nào dùng:**
> - Pipeline thuần ML: Train → Evaluate → Register → Deploy → **SageMaker Pipelines**
> - Pipeline phức tạp: Data fetch từ API → Transform → ML → Notify via SNS → Update RDS → **Step Functions**
> - Nhiều team: ML team dùng SageMaker Pipelines, Platform team orchestrate higher-level với Step Functions gọi SageMaker Pipelines bên trong"

---

### Câu 19: Tối Ưu Chi Phí SageMaker — Các Kỹ Thuật Chính?

**Câu hỏi:** "SageMaker bill đang rất cao. Bạn sẽ làm gì để tối ưu chi phí?"

**Gợi ý trả lời:**

> "Tôi sẽ phân tích theo từng phần của SageMaker:
>
> **Training Cost Optimization:**
> - **Spot Training** — tiết kiệm 60-90% với checkpointing
> - **Instance right-sizing** — dùng SageMaker Inference Recommender để test nhiều instance types
> - **Distributed training** — chia nhỏ training job trên nhiều instances nhỏ thay vì 1 instance lớn
> - **SageMaker Experiments** — track experiments để tránh duplicate training
>
> **Inference Cost Optimization:**
> - **Serverless Inference** — cho traffic thấp hoặc sporadic (không đều), trả tiền per-invocation
> - **Multi-Model Endpoint — MME (Endpoint Đa Mô Hình)** — host nhiều models trên 1 endpoint → giảm số endpoint cần duy trì
> - **Multi-Container Endpoint** — nhiều containers xử lý một request sequentially (pipeline inference)
> - **Auto Scaling** — scale out khi load tăng, scale in (xuống) khi idle → không trả tiền cho capacity dư
> - **Graviton instances** (g4dn, m7g) — inference rẻ hơn GPU khi model nhỏ
>
> **Storage Optimization:**
> - S3 lifecycle policies — chuyển old model artifacts sang S3 Glacier
> - EFS — chỉ dùng khi cần shared file system giữa training instances
>
> **General:**
> - SageMaker Savings Plans — cam kết usage để giảm giá
> - AWS Cost Explorer + SageMaker tagging → identify top spenders"

---

### Câu 20: Thiết Kế End-to-End Generative AI Application Production-Ready

**Câu hỏi:** "Thiết kế một Generative AI application production-ready phục vụ chatbot Q&A dựa trên tài liệu nội bộ doanh nghiệp, 10.000 người dùng, 99.9% uptime."

**Gợi ý trả lời:**

> "**Requirements Clarification:**
> - 10K users, 1K concurrent? Tần suất query: ~100 req/s?
> - Document corpus size: 10GB? 1TB?
> - Latency requirement: < 5s end-to-end?
> - Multi-language support?
>
> **Kiến Trúc Tổng Quan:**
>
> ```
> User → CloudFront → API Gateway → Lambda (Orchestration)
>                                      ↓
>                           Bedrock Knowledge Base
>                                      ↓
>                         OpenSearch Serverless (Vector Store)
>                                      ↓
>                           Bedrock (Claude) ← Guardrails
>                                      ↓
>                                  Response
> ```
>
> **Chi Tiết Từng Layer:**
>
> *Data Ingestion (Nhập Dữ Liệu):*
> - Documents → S3 → Bedrock Knowledge Base sync tự động
> - Scheduled sync (đồng bộ theo lịch) mỗi ngày hoặc event-driven khi có file mới
>
> *Vector Search:*
> - OpenSearch Serverless — managed, auto-scale, không cần quản lý cluster
> - k-NN index với FAISS — Fast Library for Approximate Nearest Neighbor
>
> *LLM Layer:*
> - Amazon Bedrock → Claude 3 Sonnet (balance giữa quality và cost)
> - Bedrock Guardrails → content filter + denied topics + PII redaction
> - Conversation memory → DynamoDB (lưu session history)
>
> *API Layer:*
> - API Gateway + Lambda → stateless, auto-scale
> - Cognito authentication (xác thực) → chỉ authenticated users
> - Rate limiting → 10 req/s per user
>
> *Observability (Quan Sát):*
> - CloudWatch → latency, error rate, token usage
> - X-Ray tracing → debug slow requests
> - Bedrock CloudWatch metrics → model invocations, throttling
>
> *High Availability:*
> - Multi-AZ — Availability Zone (Vùng Khả Dụng) cho OpenSearch Serverless
> - Bedrock: managed service, AWS handles HA
> - API Gateway + Lambda: inherently serverless, no single point of failure
>
> *Cost Estimate:*
> - Bedrock Claude 3 Sonnet: $3/1M input tokens, $15/1M output tokens
> - OpenSearch Serverless: ~$200/month for 2 OCU
> - Total: ~$500-2000/month tùy usage"

---

## 🎯 Câu Hỏi Bonus — Thường Xuất Hiện Cuối Phỏng Vấn

### Câu Hỏi Nhanh (1-2 phút)

**Q: Amazon Bedrock có thể train model từ đầu không?**
> Không. Bedrock cung cấp Foundation Models đã được pre-trained bởi providers (Anthropic, Meta, AWS). Bạn chỉ có thể Fine-tune với dữ liệu riêng, không train from scratch. Muốn train from scratch thì dùng SageMaker.

**Q: Khi nào cần Amazon Kendra thay vì Bedrock Knowledge Base?**
> Kendra là intelligent search engine có nhiều connectors (SharePoint, Confluence, Salesforce) và hỗ trợ incremental indexing tốt hơn. Bedrock Knowledge Base phù hợp hơn khi muốn LLM generate responses từ knowledge. Kendra + Bedrock có thể kết hợp: Kendra search → Bedrock generate.

**Q: SageMaker Neo là gì?**
> SageMaker Neo compile model để chạy trên specific hardware (Intel, ARM, NVIDIA) — tối ưu performance và giảm latency inference. Hữu ích cho Edge deployment (IoT, embedded devices).

**Q: Amazon Lookout for Metrics khác gì SageMaker Model Monitor?**
> Lookout for Metrics phát hiện bất thường (anomalies) trong business metrics (doanh thu, lượt click) — không cần ML expertise. Model Monitor giám sát ML model performance và data drift trong SageMaker endpoint.

---

## 📊 Bảng Tóm Tắt Trade-offs Quan Trọng

| Quyết Định | Chọn A | Chọn B | Khi Nào |
|-----------|--------|--------|---------|
| **RAG vs Fine-tuning** | RAG | Fine-tuning | RAG: knowledge mới, cần citations. Fine-tuning: behavior/style |
| **Real-time vs Batch** | Real-time | Batch Transform | Real-time: user-facing. Batch: offline bulk processing |
| **AI Service vs SageMaker** | AI Service | SageMaker | AI Service: common use case, no ML expertise. SageMaker: custom, special domain |
| **Lex vs Bedrock Chatbot** | Amazon Lex | Bedrock | Lex: structured dialog, IVR. Bedrock: open-ended, RAG-based |
| **Spot vs On-demand Training** | Spot | On-demand | Spot: long jobs, can checkpoint. On-demand: tight deadline, critical |
| **Serverless vs Real-time Endpoint** | Serverless | Real-time | Serverless: sporadic traffic. Real-time: consistent high traffic |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành — 20 câu hỏi đầy đủ
