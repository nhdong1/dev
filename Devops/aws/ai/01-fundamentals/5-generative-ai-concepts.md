# Generative AI Concepts — LLM, Foundation Model, Prompt, Token, RAG

> Nắm vững 5 khái niệm cốt lõi này là điều kiện tiên quyết để học Amazon Bedrock và các ứng dụng Generative AI hiện đại trên AWS.

---

## 1. Foundation Model — Mô Hình Nền Tảng

### Định Nghĩa

Foundation Model (FM — Mô Hình Nền Tảng) là **model AI quy mô lớn được huấn luyện trên lượng dữ liệu khổng lồ** bằng phương pháp self-supervised learning (học tự giám sát), có thể thích nghi với nhiều tác vụ khác nhau thông qua fine-tuning (tinh chỉnh) hoặc prompting (lập trình bằng prompt).

### Đặc Điểm Phân Biệt Foundation Model Với Traditional ML Model

| Tiêu Chí | Traditional ML Model | Foundation Model |
| --------- | -------------------- | ---------------- |
| **Phạm vi** | Một nhiệm vụ duy nhất | Đa nhiệm vụ (multi-task) |
| **Dữ liệu training** | Hàng nghìn đến triệu mẫu | Hàng tỉ mẫu (toàn bộ internet) |
| **Chi phí training** | Vài USD đến vài nghìn USD | Hàng triệu USD |
| **Thích nghi (Adaptation)** | Cần train lại từ đầu | Fine-tune nhẹ hoặc chỉ cần prompt |
| **Ví dụ** | XGBoost spam filter | GPT-4, Claude, Llama |
| **Ai train được** | Team ML nhỏ | Chỉ các tổ chức lớn (OpenAI, Anthropic, Meta, AWS) |

### Foundation Models Trên Amazon Bedrock

| Model | Nhà Cung Cấp | Mạnh Về |
| ----- | ------------ | ------- |
| **Claude 3.x** (Haiku, Sonnet, Opus) | Anthropic | Reasoning, analysis, safety |
| **Llama 3.x** | Meta | Open weights, customizable |
| **Titan Text / Embeddings** | Amazon | Tích hợp sâu với AWS ecosystem |
| **Mistral** | Mistral AI | Efficiency, European language |
| **Stable Diffusion XL** | Stability AI | Image generation (tạo hình ảnh) |
| **Cohere Command R+** | Cohere | RAG, enterprise use cases |
| **Jurassic-2** | AI21 Labs | Long-form writing |

### Paradigm Shift (Sự Thay Đổi Mô Hình) Của Foundation Models

```
TRƯỚC (Pre-Foundation Model era):
  Mỗi bài toán → Train một model riêng
  Spam filter ─────────────────── model A
  Sentiment analysis ──────────── model B
  Machine translation ─────────── model C
  Image captioning ────────────── model D

SAU (Foundation Model era):
  Một Foundation Model → Giải nhiều bài toán
  Foundation Model ─┬── Spam filter (prompt)
                    ├── Sentiment (prompt)
                    ├── Translation (prompt)
                    ├── Image captioning (prompt)
                    └── Code generation (prompt)
```

---

## 2. Large Language Model — LLM (Mô Hình Ngôn Ngữ Lớn)

### Định Nghĩa

LLM — Large Language Model (Mô Hình Ngôn Ngữ Lớn) là loại Foundation Model chuyên xử lý và tạo sinh văn bản (text), được huấn luyện để dự đoán token tiếp theo trong chuỗi.

### Cách LLM Hoạt Động — Next Token Prediction

```
Quá trình sinh văn bản:

Input: "Thủ đô của Việt Nam là"
                 ↓
LLM dự đoán token tiếp theo có xác suất cao nhất:
  "Hà" → 85%
  "TP" → 8%
  "Đà" → 2%
  ...
                 ↓
Chọn "Hà" → Input trở thành: "Thủ đô của Việt Nam là Hà"
                 ↓
Lại dự đoán token tiếp theo:
  "Nội" → 95%
  ...
                 ↓
Output: "Thủ đô của Việt Nam là Hà Nội"
```

**Mấu chốt:** LLM không "hiểu" theo nghĩa con người — nó thực hiện xác suất thống kê cực kỳ tinh vi dựa trên pattern từ hàng tỉ văn bản đã học.

### Kiến Trúc Transformer — Nền Tảng Của LLM

**Attention Mechanism (Cơ Chế Chú Ý):** Cho phép model "chú ý" (focus) vào các phần liên quan của input khi sinh output.

```
Input: "Con mèo đang ngủ vì nó mệt"

Khi xử lý "nó", model tính attention:
  "Con" → 5%
  "mèo" → 80%   ← chú ý cao vì "nó" = "mèo"
  "đang" → 3%
  "ngủ" → 7%
  "vì" → 2%
  "mệt" → 3%
```

### Các Thông Số Quan Trọng Của LLM

| Thông Số | Giải Thích | Ảnh Hưởng |
| --------- | ---------- | ---------- |
| **Parameters** (Tham Số) | Số lượng trọng số trong model | Lớn hơn → mạnh hơn, tốn tiền hơn |
| **Context Window** (Cửa Sổ Ngữ Cảnh) | Số token tối đa xử lý được | Nhỏ → quên "lịch sử" hội thoại |
| **Training Data Cutoff** (Ngày Khóa Dữ Liệu) | Dữ liệu training đến ngày nào | Không biết sự kiện sau ngày đó |

---

## 3. Token — Đơn Vị Xử Lý Của LLM

### Định Nghĩa

Token là **đơn vị xử lý cơ bản** của LLM — không phải chữ cái, cũng không phải từ hoàn chỉnh, mà là "mảnh" văn bản được xác định bởi thuật toán tokenization.

### Ví Dụ Tokenization

```
Văn bản: "Unhappiness is inevitable"

Tokenization ví dụ (tùy tokenizer):
  "Un" | "happiness" | " is" | " in" | "evitable"
    ↑ 5 tokens

Tokenization khác:
  "Unhappiness" | " is" | " inevitable"
    ↑ 3 tokens (tokenizer tốt hơn với từ dài)
```

**Quy tắc ước tính thực tế:**
- 1 token ≈ 0.75 từ tiếng Anh
- 1 token ≈ 0.5 từ tiếng Việt (vì tiếng Việt có nhiều syllable — âm tiết hơn)
- 1 trang văn bản ≈ 750 tokens

### Tại Sao Token Quan Trọng Với AWS?

**Bedrock tính phí theo token:**

```
Phí = (Input Tokens × giá input/1K token)
    + (Output Tokens × giá output/1K token)

Ví dụ (Claude 3 Haiku — 2024):
  Input: $0.25 / 1M tokens
  Output: $1.25 / 1M tokens

Bài toán thực tế:
  Prompt: 500 tokens + Context: 2.000 tokens = 2.500 input tokens
  Response: 300 output tokens
  
  Chi phí: (2.500 × $0.00000025) + (300 × $0.00000125)
         = $0.000625 + $0.000375
         = $0.001 per request
```

### Context Window — Cửa Sổ Ngữ Cảnh

```
Context Window = Số token tối đa model xử lý được trong 1 lần

Claude 3.5 Sonnet: 200.000 tokens (~150.000 từ, ~300 trang)
GPT-4o: 128.000 tokens
Llama 3.1: 128.000 tokens

Nếu vượt Context Window:
  → Model không thể đọc phần đầu của conversation
  → Phải dùng kỹ thuật như RAG hoặc sliding window
```

---

## 4. Prompt Engineering — Kỹ Thuật Lập Trình Prompt

### Định Nghĩa

Prompt Engineering (Kỹ Thuật Lập Trình Prompt) là nghệ thuật thiết kế đầu vào (prompt) cho LLM để nhận được đầu ra (output) chính xác và hữu ích nhất.

### Các Kỹ Thuật Prompting

#### Zero-shot Prompting (Không Có Ví Dụ)

Chỉ mô tả nhiệm vụ, không cho ví dụ.

```
Prompt: "Phân loại cảm xúc của câu sau: Positive, Negative, hoặc Neutral.
         Câu: 'Sản phẩm này tệ hại, tôi rất thất vọng.'"

Output: "Negative"
```

#### Few-shot Prompting (Vài Ví Dụ)

Cung cấp một số ví dụ (shots) trước khi hỏi câu thực.

```
Prompt:
  "Phân loại cảm xúc:
   
   Câu: 'Tôi rất thích sản phẩm này!' → Positive
   Câu: 'Giao hàng quá chậm.' → Negative
   Câu: 'Sản phẩm đã được giao.' → Neutral
   
   Câu: 'Màu sắc đẹp nhưng chất lượng kém.' → ?"

Output: "Negative"  ← chính xác hơn zero-shot với câu phức tạp
```

#### Chain-of-Thought — CoT (Chuỗi Suy Nghĩ)

Yêu cầu model giải thích từng bước suy luận trước khi đưa ra kết quả.

```
Prompt: "Giải bài toán sau. Hãy suy nghĩ từng bước:
         Một shop bán 15 áo với giá 200.000đ và 8 quần với giá 350.000đ.
         Tổng doanh thu là bao nhiêu?"

Output (với CoT):
  "Bước 1: Doanh thu từ áo = 15 × 200.000đ = 3.000.000đ
   Bước 2: Doanh thu từ quần = 8 × 350.000đ = 2.800.000đ
   Bước 3: Tổng = 3.000.000đ + 2.800.000đ = 5.800.000đ"
```

#### System Prompt (Prompt Hệ Thống)

Đặt "vai trò" và hành vi cho LLM trước khi bắt đầu conversation.

```
System Prompt:
  "Bạn là chuyên gia tư vấn tài chính của ngân hàng VietBank.
   - Chỉ trả lời câu hỏi liên quan đến sản phẩm ngân hàng
   - Không đưa ra lời khuyên đầu tư cụ thể
   - Luôn nhắc khách hàng tham khảo chuyên gia trước khi quyết định lớn
   - Trả lời bằng tiếng Việt, thân thiện và chuyên nghiệp"

User: "Lãi suất tiết kiệm hiện tại là bao nhiêu?"
Model: [trả lời đúng vai trò, không lạc đề]
```

### Các Tham Số Điều Chỉnh Output

| Tham Số | Giải Thích | Giá Trị Thấp | Giá Trị Cao |
| ------- | ---------- | ------------ | ----------- |
| **Temperature** (Nhiệt Độ) | Độ ngẫu nhiên của output | Nhất quán, dự đoán được | Sáng tạo, đa dạng |
| **Top-p** (Nucleus Sampling) | % xác suất tích lũy để chọn token | Output tập trung | Output đa dạng |
| **Max Tokens** | Giới hạn độ dài output | Output ngắn | Output dài |
| **Stop Sequences** | Ký tự/chuỗi báo dừng | — | Kiểm soát định dạng output |

```
Dùng Temperature thấp (0.0-0.3): Factual Q&A, code generation, structured output
Dùng Temperature cao (0.7-1.0): Sáng tác, brainstorming, creative writing
```

---

## 5. RAG — Retrieval-Augmented Generation (Tạo Sinh Tăng Cường Truy Xuất)

### Vấn Đề RAG Giải Quyết

LLM có 3 giới hạn cơ bản:

```
Giới Hạn 1: Knowledge Cutoff (Giới Hạn Thông Tin)
  → LLM không biết sự kiện sau ngày training cutoff
  → Claude không biết tin tức hôm nay

Giới Hạn 2: Hallucination (Ảo Giác)
  → LLM đôi khi "bịa" thông tin trông có vẻ đúng nhưng sai
  → Nguy hiểm trong y tế, pháp lý, tài chính

Giới Hạn 3: Private Knowledge (Kiến Thức Riêng Tư)
  → LLM không biết dữ liệu nội bộ của công ty bạn
  → Chính sách nội bộ, tài liệu kỹ thuật riêng
```

**RAG giải quyết cả 3 giới hạn** bằng cách cho LLM "tra cứu" dữ liệu thực trước khi trả lời.

### Kiến Trúc RAG

```
Phase 1: Indexing (Lập Chỉ Mục) — thực hiện trước

  Documents (Tài Liệu)
  ────────────────────
  PDF, Word, HTML, ...
         │
         ▼
  Chunking (Chia Nhỏ)
  Chia document thành đoạn nhỏ (~512 tokens mỗi đoạn)
         │
         ▼
  Embedding Model (Mô Hình Nhúng)
  Chuyển text → Vector số học
  "Chính sách hoàn tiền" → [0.2, -0.8, 0.5, ...]
         │
         ▼
  Vector Store (Kho Vector)
  Lưu vector + metadata
  (OpenSearch, Pinecone, pgvector...)


Phase 2: Retrieval & Generation (Truy Xuất & Sinh) — khi có câu hỏi

  User Question: "Chính sách hoàn tiền của công ty là gì?"
         │
         ▼
  Embed Question (Nhúng Câu Hỏi)
  → [0.19, -0.82, 0.48, ...]
         │
         ▼
  Semantic Search (Tìm Kiếm Ngữ Nghĩa)
  Tìm các chunks có vector gần nhất (cosine similarity)
  → Top 3-5 chunks liên quan nhất
         │
         ▼
  Augmented Prompt (Prompt Tăng Cường):
  "Dựa trên tài liệu sau:
   [Chunk 1: Chính sách hoàn tiền trong 30 ngày...]
   [Chunk 2: Điều kiện hoàn tiền: sản phẩm nguyên vẹn...]
   
   Câu hỏi: Chính sách hoàn tiền của công ty là gì?"
         │
         ▼
  LLM (Claude / Titan / ...)
         │
         ▼
  Answer: "Theo tài liệu, công ty hỗ trợ hoàn tiền trong 30 ngày
           nếu sản phẩm còn nguyên vẹn và..."
           ← Trích nguồn, không hallucinate
```

### RAG Trên Amazon Bedrock

**Amazon Bedrock Knowledge Bases** (Cơ Sở Kiến Thức Bedrock) tự động hóa toàn bộ quy trình RAG:

```
Bạn cần làm:
  1. Upload documents lên S3
  2. Tạo Knowledge Base trong Bedrock Console
  3. Chọn Embedding Model (Titan Embeddings V2)
  4. Chọn Vector Store (OpenSearch Serverless hoặc Aurora pgvector)
  5. Sync (đồng bộ) — Bedrock tự chunk, embed, index

AWS lo:
  - Chunking strategy
  - Embedding tự động
  - Vector storage
  - Retrieval khi query
  - Inject vào prompt
```

### RAG vs Fine-tuning — Khi Nào Dùng Gì?

| Tiêu Chí | RAG | Fine-tuning (Tinh Chỉnh) |
| --------- | --- | ------------------------ |
| **Dữ liệu thay đổi thường xuyên** | ✅ Tốt (chỉ update vector store) | ❌ Phải retrain |
| **Cần nguồn trích dẫn** | ✅ Dễ cung cấp nguồn | ❌ Khó |
| **Học phong cách/tone riêng** | ❌ Không tốt | ✅ Tốt hơn |
| **Chi phí** | Thấp (chỉ inference) | Cao (training + inference) |
| **Kiến thức domain sâu** | ✅ Tốt khi có đủ documents | ✅ Tốt khi train đủ examples |
| **Hallucination reduction** | ✅ Rất tốt | ✅ Một phần |

**Quy tắc thực tế:** Thử RAG trước — nếu không đủ (vì cần style/tone riêng, hoặc domain quá chuyên biệt) thì mới fine-tune.

---

## 6. Embedding — Vector Nhúng

### Định Nghĩa

Embedding (Vector Nhúng) là cách **biểu diễn văn bản (hoặc hình ảnh) dưới dạng vector số học** trong không gian nhiều chiều, sao cho các nội dung có nghĩa tương đồng nằm gần nhau trong không gian đó.

```
"Con mèo đang ngủ" → [0.2, -0.5, 0.8, 0.1, ..., 0.3]  (1.536 chiều)
"Mèo đang nghỉ ngơi" → [0.19, -0.48, 0.79, 0.12, ..., 0.31]
"Trời hôm nay đẹp" → [-0.8, 0.3, -0.1, 0.6, ..., -0.4]

Cosine Similarity (Độ Tương Đồng Cosin):
  "Con mèo..." vs "Mèo đang..." → 0.95 (rất gần)
  "Con mèo..." vs "Trời hôm nay..." → 0.12 (xa)
```

### Embedding Models Trên AWS

| Model | Nhà Cung Cấp | Chiều Vector | Use Case |
| ----- | ------------ | ------------ | --------- |
| **Titan Embeddings V2** | Amazon | 256/512/1024 | RAG, similarity search |
| **Cohere Embed v3** | Cohere | 1024 | Multilingual RAG |

### Ứng Dụng Embedding

- **RAG:** Tìm kiếm ngữ nghĩa (semantic search) trong vector store
- **Recommendation:** Tìm sản phẩm/nội dung tương tự
- **Deduplication (Loại Trùng Lặp):** Phát hiện câu hỏi tương tự
- **Clustering:** Nhóm tài liệu theo chủ đề

---

## 7. AI Agents — Tác Nhân AI

### Định Nghĩa

AI Agent (Tác Nhân AI) là LLM có khả năng **tự lên kế hoạch và thực hiện nhiều bước** để hoàn thành mục tiêu, bao gồm gọi external tools (công cụ ngoài).

### Kiến Trúc Agent

```
User: "Tìm giá vé máy bay Hà Nội → HCM ngày 15/7 và đặt vé rẻ nhất"

Agent (LLM):
  Suy nghĩ: "Tôi cần tìm giá vé, sau đó so sánh, rồi đặt vé"
       ↓
  Bước 1: Gọi Tool "search_flights(HAN, SGN, 2026-07-15)"
       → Kết quả: [VN123: $80, VJ456: $65, BL789: $72]
       ↓
  Bước 2: Gọi Tool "book_flight(VJ456)"
       → Kết quả: "Đặt thành công, mã đặt chỗ: ABC123"
       ↓
  Trả lời User: "Đã đặt vé Vietjet VJ456 giá $65,
                 mã đặt chỗ ABC123"
```

### Amazon Bedrock Agents (Tác Nhân Bedrock)

**Bedrock Agents** = Foundation Model (LLM) + Action Groups (Nhóm Hành Động) + Knowledge Bases

```
Action Groups:
  → Lambda functions (AWS) mà agent có thể gọi
  → Định nghĩa bằng OpenAPI schema
  
Ví dụ Action Groups cho Customer Service Agent:
  - get_order_status(order_id)
  - create_refund_request(order_id, reason)
  - search_knowledge_base(query)
  - escalate_to_human(ticket_id)
```

---

## 8. Các Giới Hạn Và Rủi Ro Của LLM

### Hallucination (Ảo Giác)

LLM có thể "bịa" thông tin tự tin: ngày tháng sai, trích dẫn không tồn tại, số liệu sai.

**Giải pháp:** RAG (có nguồn trích dẫn), prompt yêu cầu "nếu không chắc thì nói không biết", fact-checking bởi con người.

### Bias (Thiên Kiến)

Model học từ dữ liệu internet — mang theo thiên kiến xã hội, văn hóa, chính trị trong dữ liệu đó.

**Giải pháp:** SageMaker Clarify để phát hiện bias, Bedrock Guardrails (Rào Chắn Bedrock) để filter output.

### Prompt Injection (Tấn Công Qua Prompt)

Kẻ tấn công nhúng instruction vào input để override system prompt, khiến model làm điều không mong muốn.

**Giải pháp:** Input validation, Bedrock Guardrails, không expose system prompt, giới hạn action của agent.

### Knowledge Cutoff (Giới Hạn Thông Tin)

LLM không biết sự kiện xảy ra sau ngày training cutoff.

**Giải pháp:** RAG với dữ liệu real-time, kết hợp với web search tool trong agent.

---

## 9. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Foundation Model là gì? Tại sao nó quan trọng hơn traditional ML model?**

> Foundation Model được pre-train trên lượng dữ liệu khổng lồ, có thể thực hiện nhiều tác vụ khác nhau mà chỉ cần prompt hoặc fine-tune nhẹ. Trước đây mỗi bài toán cần một model riêng, tốn nhiều dữ liệu và thời gian. Với Foundation Model, một model có thể làm phân tích cảm xúc, dịch, tóm tắt, sinh code — giảm mạnh chi phí và thời gian phát triển.

**Q: RAG là gì? Tại sao dùng RAG thay vì fine-tuning?**

> RAG — Retrieval-Augmented Generation — cho phép LLM tra cứu dữ liệu thực tế trước khi trả lời, giúp giảm hallucination và cung cấp thông tin cập nhật/riêng tư. Dùng RAG thay vì fine-tuning khi: dữ liệu thay đổi thường xuyên, cần trích nguồn, chi phí training lớn. Chỉ fine-tune khi cần học phong cách/behavior riêng hoặc domain quá chuyên biệt.

**Q: Token là gì? Tại sao quan trọng khi dùng Bedrock?**

> Token là đơn vị xử lý của LLM, xấp xỉ 0.75 từ tiếng Anh. Bedrock tính phí theo số token (input + output), nên hiểu token giúp ước tính chi phí và tối ưu prompt (ngắn gọn hơn = rẻ hơn). Context window — số token tối đa — cũng ảnh hưởng đến khả năng xử lý tài liệu dài.

**Q: Hallucination trong LLM là gì? Cách xử lý?**

> Hallucination là hiện tượng LLM tự tin tạo ra thông tin sai lệch — ví dụ: trích dẫn bài báo không tồn tại, đưa ra số liệu sai. Xử lý bằng: (1) RAG để cung cấp ground truth facts; (2) Bedrock Guardrails để filter output; (3) Yêu cầu model cite nguồn; (4) Human-in-the-loop review cho quyết định quan trọng.

---

## 10. Tổng Kết — 5 Khái Niệm Cốt Lõi

```
1. Foundation Model (Mô Hình Nền Tảng)
   → Model khổng lồ, đa năng, train một lần dùng nhiều
   → AWS Bedrock cung cấp: Claude, Llama, Titan, Stable Diffusion...

2. LLM — Large Language Model (Mô Hình Ngôn Ngữ Lớn)
   → Foundation Model chuyên văn bản, dự đoán token tiếp theo
   → Cơ chế: Transformer + Attention

3. Token (Đơn Vị Xử Lý)
   → Mảnh văn bản (~0.75 từ/token)
   → Bedrock tính phí theo token, context window tính bằng token

4. Prompt Engineering (Kỹ Thuật Lập Trình Prompt)
   → Zero-shot, Few-shot, Chain-of-thought, System prompt
   → Kỹ năng thiết yếu để dùng LLM hiệu quả

5. RAG — Retrieval-Augmented Generation (Tạo Sinh Tăng Cường Truy Xuất)
   → Tra cứu tài liệu thực → inject vào prompt → LLM trả lời có nguồn
   → Bedrock Knowledge Bases tự động hóa RAG pipeline
```

---

## ➡️ Tiếp Theo

Bạn đã nắm vững nền tảng AI/ML! Bước tiếp theo:

- [02-sagemaker/README.md](../02-sagemaker/README.md) — Học Amazon SageMaker: train và deploy custom ML model
- [03-bedrock/README.md](../03-bedrock/README.md) — Học Amazon Bedrock: xây dựng ứng dụng Generative AI với Foundation Models
