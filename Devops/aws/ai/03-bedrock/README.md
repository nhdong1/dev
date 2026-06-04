# Amazon Bedrock — Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh) Toàn Diện

> Amazon Bedrock là dịch vụ fully-managed (được quản lý hoàn toàn) của AWS, cung cấp quyền truy cập vào các Foundation Models (Mô Hình Nền Tảng) hàng đầu từ nhiều nhà cung cấp AI thông qua một API duy nhất — không cần quản lý hạ tầng, không cần huấn luyện mô hình từ đầu.

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| `README.md` | Tổng quan Bedrock, kiến trúc, so sánh Foundation Models | ✅ |
| `1-foundation-models.md` | Claude, Llama 3, Titan, Stable Diffusion — so sánh chi tiết | ✅ |
| `2-prompt-engineering.md` | Zero-shot, Few-shot, Chain-of-thought, Prompt Templates | ✅ |
| `3-rag-knowledge-base.md` | RAG, Bedrock Knowledge Base, Vector Store, OpenSearch Serverless | ✅ |
| `4-bedrock-agents.md` | Autonomous Agent, Tool use, Action Group, Orchestration | ✅ |
| `5-fine-tuning.md` | Continued Pre-training, Instruction Tuning, PEFT, LoRA | ✅ |

---

## 🎯 Amazon Bedrock Là Gì?

Amazon Bedrock là dịch vụ Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh) ra mắt năm 2023, cho phép:

- **Access** (Truy Cập): Gọi nhiều Foundation Models từ AWS và third-party providers qua một API thống nhất
- **Customize** (Tùy Chỉnh): Fine-tune (tinh chỉnh) hoặc RAG (tăng cường với dữ liệu riêng) mà không cần quản lý infrastructure
- **Build** (Xây Dựng): Tạo ứng dụng Generative AI với Bedrock Agents, Knowledge Bases, Guardrails
- **Govern** (Quản Trị): Kiểm soát nội dung, bảo mật dữ liệu, audit với Guardrails và CloudTrail

### Vị Trí Trong Hệ Sinh Thái AWS AI

```
┌─────────────────────────────────────────────────────────────────┐
│  Tầng 3: AI Services (Dịch Vụ AI Được Quản Lý)                 │
│  Rekognition │ Comprehend │ Polly │ Transcribe │ Lex            │
├─────────────────────────────────────────────────────────────────┤
│  Amazon Bedrock — Generative AI  ◄── BẠN ĐANG Ở ĐÂY            │
│  Foundation Models │ RAG │ Agents │ Fine-tuning │ Guardrails    │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 2: ML Services                                            │
│                      Amazon SageMaker                           │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 1: ML Framework & Infrastructure                          │
│  TensorFlow │ PyTorch │ EC2 │ GPU/Trainium Instances            │
└─────────────────────────────────────────────────────────────────┘
```

**Quy tắc chọn Bedrock:** Khi cần Generative AI capabilities (khả năng AI tạo sinh) mà không muốn tự huấn luyện và quản lý Foundation Model — đây là lựa chọn nhanh nhất và tiết kiệm chi phí nhất.

---

## 🏗️ Kiến Trúc Amazon Bedrock

### Các Thành Phần Chính

```
┌────────────────────────────────────────────────────────────┐
│                    Amazon Bedrock                          │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Foundation Models (Mô Hình Nền Tảng)      │  │
│  │  Claude (Anthropic) │ Llama (Meta) │ Titan (AWS)    │  │
│  │  Stable Diffusion   │ Jurassic (AI21) │ Command     │  │
│  └──────────────────────────────────────────────────────┘  │
│           │                    │                    │       │
│           ▼                    ▼                    ▼       │
│  ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  │
│  │  Knowledge   │  │  Bedrock Agents   │  │  Fine-tuning│  │
│  │  Bases (RAG) │  │  (Tác Nhân AI)    │  │  (Tinh Chỉnh│  │
│  │              │  │                   │  │   Mô Hình)  │  │
│  └──────────────┘  └───────────────────┘  └─────────────┘  │
│           │                    │                    │       │
│           └────────────────────┼────────────────────┘       │
│                                ▼                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Bedrock Guardrails (Rào Cản)            │  │
│  │  Content filtering │ PII redaction │ Topic denial    │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │   Ứng Dụng Của Bạn       │
              │  (Lambda, API Gateway,   │
              │   ECS, EC2, v.v.)        │
              └──────────────────────────┘
```

---

## 🤖 Foundation Models Có Sẵn Trên Bedrock

### Tổng Quan Các Provider (Nhà Cung Cấp)

| Provider | Model | Loại | Điểm Mạnh |
|---|---|---|---|
| **Anthropic** | Claude 3.5 Sonnet, Claude 3 Opus, Claude 3 Haiku | Text/Vision | Độ chính xác cao, an toàn, context dài |
| **Meta** | Llama 3.1 8B, 70B, 405B | Text | Open-source, tùy biến linh hoạt |
| **Amazon** | Titan Text, Titan Embeddings, Titan Image | Text/Image | Native AWS, tích hợp sâu |
| **Stability AI** | Stable Diffusion XL | Image | Tạo ảnh chất lượng cao |
| **Cohere** | Command R+, Embed | Text/Embeddings | RAG-optimized, embeddings xuất sắc |
| **AI21 Labs** | Jamba | Text | Kiến trúc SSM-Transformer hybrid |
| **Mistral AI** | Mistral Large, Mixtral 8x7B | Text | Hiệu năng/chi phí tối ưu |

### Chọn Model Theo Use Case (Trường Hợp Sử Dụng)

| Use Case | Model Khuyến Nghị | Lý Do |
|---|---|---|
| Chatbot tổng quát | Claude 3.5 Sonnet | Cân bằng chất lượng-tốc độ-chi phí |
| Phân tích tài liệu phức tạp | Claude 3 Opus | Context window dài nhất, độ chính xác cao |
| Phản hồi nhanh / low-latency | Claude 3 Haiku, Llama 3.1 8B | Chi phí thấp, tốc độ cao |
| RAG & semantic search | Titan Embeddings, Cohere Embed | Embedding chất lượng cao |
| Tạo ảnh từ văn bản | Stable Diffusion XL, Titan Image | Chuyên dụng cho image generation |
| Xử lý code | Claude 3.5 Sonnet, Llama 3.1 | Hiểu sâu ngữ nghĩa code |

---

## 🔑 Các Khái Niệm Cốt Lõi

### 1. Inference (Suy Luận / Dự Đoán)

Bedrock hỗ trợ ba chế độ inference:

| Chế Độ | Mô Tả | Tính Phí | Phù Hợp Cho |
|---|---|---|---|
| **On-demand** (Theo Yêu Cầu) | Gọi API ngay, trả tiền theo token | Per token | Dev, test, traffic không đều |
| **Provisioned Throughput** (Thông Lượng Dự Phòng) | Đặt trước capacity (năng lực), đảm bảo TPS | Per hour | Production, SLA nghiêm ngặt |
| **Batch Inference** (Suy Luận Hàng Loạt) | Xử lý lượng lớn request offline | Giảm ~50% | Bulk processing, phân tích văn bản lớn |

### 2. Context Window (Cửa Sổ Ngữ Cảnh)

Context window là số lượng token tối đa mà model có thể "nhìn thấy" trong một lần inference — bao gồm cả prompt (câu hỏi/hướng dẫn) và response (câu trả lời):

```
[System Prompt] + [Conversation History] + [User Input] + [Retrieved Docs] = Tổng Token
       ↑                    ↑                    ↑               ↑
   Hướng dẫn          Lịch sử hội thoại     Câu hỏi hiện tại   Tài liệu RAG
```

- **Claude 3**: Lên tới 200,000 tokens (~150,000 từ)
- **Llama 3.1**: Lên tới 128,000 tokens
- **Titan Text**: Lên tới 32,000 tokens

### 3. Token (Đơn Vị Văn Bản)

Token là đơn vị xử lý cơ bản của LLM:
- ~1 token ≈ 4 ký tự tiếng Anh (hoặc ~0.75 từ)
- Tiếng Việt thường tốn nhiều token hơn tiếng Anh do encoding
- Chi phí Bedrock tính theo **input tokens** (token đầu vào) + **output tokens** (token đầu ra)

### 4. Temperature & Sampling (Nhiệt Độ & Lấy Mẫu)

| Tham Số | Giá Trị | Ảnh Hưởng |
|---|---|---|
| `temperature` (nhiệt độ) | 0.0 → 1.0 | 0 = deterministic (ổn định), 1 = creative (sáng tạo) |
| `top_p` (lấy mẫu nucleus) | 0.0 → 1.0 | Kiểm soát sự đa dạng của vocabulary |
| `top_k` (lấy mẫu top-k) | 1 → 500 | Giới hạn số token được xem xét mỗi bước |
| `max_tokens` (token tối đa) | Tùy model | Giới hạn độ dài output |

---

## 💰 Mô Hình Chi Phí

### On-demand Pricing (Giá Theo Nhu Cầu)

Chi phí tính theo **1,000 input tokens** và **1,000 output tokens**:

| Model | Input ($/1K tokens) | Output ($/1K tokens) |
|---|---|---|
| Claude 3 Haiku | ~$0.00025 | ~$0.00125 |
| Claude 3.5 Sonnet | ~$0.003 | ~$0.015 |
| Claude 3 Opus | ~$0.015 | ~$0.075 |
| Llama 3.1 8B | ~$0.0003 | ~$0.0006 |
| Titan Text Lite | ~$0.0003 | ~$0.0004 |

> **Lưu ý chi phí:** Output token đắt hơn input token 3-5 lần. Hãy tối ưu output length khi cần kiểm soát chi phí.

### Provisioned Throughput (Thông Lượng Dự Phòng)

- Tính phí theo **Model Units (MU)** per hour
- Mỗi MU đảm bảo một mức TPS (Transactions Per Second — Số Giao Dịch Mỗi Giây) nhất định
- Phù hợp khi traffic ổn định và cần latency thấp nhất quán

---

## 🔒 Bảo Mật & Governance (Quản Trị)

### Bedrock Guardrails (Rào Cản Nội Dung)

Guardrails cho phép kiểm soát input/output của Foundation Models:

```
User Input → [Guardrails Check] → Foundation Model → [Guardrails Check] → Response
                    ↓                                         ↓
              Block nếu vi phạm                       Block/redact nếu vi phạm
```

**Các loại rào cản:**
- **Content filters** (Bộ Lọc Nội Dung): Chặn hate speech, violence, sexual content
- **Denied topics** (Chủ Đề Bị Từ Chối): Ngăn model thảo luận về chủ đề cụ thể (VD: cạnh tranh)
- **Word filters** (Bộ Lọc Từ): Chặn từ ngữ cụ thể
- **PII redaction** (Ẩn Danh Thông Tin Cá Nhân): Tự động xóa/thay thế PII — Personally Identifiable Information
- **Grounding check** (Kiểm Tra Cơ Sở): Phát hiện hallucination (ảo giác — thông tin bịa đặt)

### Bảo Mật Dữ Liệu

- **Không dùng dữ liệu của bạn để train model:** AWS cam kết dữ liệu gửi đến Bedrock không được dùng cải thiện model của bên thứ ba
- **Encryption** (Mã Hóa): Data in-transit qua TLS 1.2+, data at-rest qua AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa)
- **VPC PrivateLink**: Gọi Bedrock API trong VPC mà không qua internet công cộng
- **IAM**: Kiểm soát truy cập chi tiết tới từng model, action

---

## 🆚 So Sánh: Bedrock vs SageMaker

| Tiêu Chí | Amazon Bedrock | Amazon SageMaker |
|---|---|---|
| **Mục đích chính** | Dùng Foundation Models có sẵn | Train/deploy custom ML models |
| **ML expertise cần** | Thấp — chủ yếu API calls | Cao — cần biết ML, data science |
| **Thời gian ra mắt** | Nhanh (giờ → ngày) | Chậm hơn (ngày → tuần) |
| **Kiểm soát model** | Hạn chế — dùng model của provider | Hoàn toàn — tự chọn architecture |
| **Chi phí** | Per token (không có idle cost) | Per instance-hour (có idle cost) |
| **Use case** | GenAI apps, chatbot, content gen | Custom model training, AutoML, MLOps |
| **Customization** | Fine-tuning, RAG, Prompt engineering | Full retraining, custom architectures |

**Nguyên tắc chọn:**
- Dùng **Bedrock** khi: Cần Generative AI nhanh, không có dữ liệu labeled (được gán nhãn) đủ lớn để train, muốn minimal infrastructure management
- Dùng **SageMaker** khi: Cần custom model với dữ liệu riêng, cần kiểm soát hoàn toàn architecture, tabular data / time series / computer vision không phải LLM

---

## 🚀 Quick Start — Gọi Bedrock API

### Cài Đặt & Xác Thực

```python
import boto3
import json

# Khởi tạo Bedrock Runtime client (bộ xử lý yêu cầu Bedrock)
bedrock = boto3.client(
    service_name="bedrock-runtime",
    region_name="us-east-1"  # Bedrock sẵn sàng ở nhiều region
)
```

### Gọi Claude 3 (Anthropic Messages API)

```python
def call_claude(prompt: str, max_tokens: int = 1024) -> str:
    """Gọi Claude 3 Sonnet qua Bedrock API."""
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_tokens,
        "messages": [
            {
                "role": "user",
                "content": prompt
            }
        ]
    }

    response = bedrock.invoke_model(
        modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]

# Sử dụng
answer = call_claude("Giải thích RAG là gì trong 3 câu.")
print(answer)
```

### Gọi Titan Embeddings (Để Tạo Vector)

```python
def create_embedding(text: str) -> list[float]:
    """Tạo embedding vector (vectơ nhúng) từ văn bản."""
    body = {"inputText": text}

    response = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    return result["embedding"]  # Danh sách 1536 float

# Ví dụ tính cosine similarity (độ tương đồng cosin)
import numpy as np

vec1 = create_embedding("Machine learning là học máy")
vec2 = create_embedding("Deep learning là học sâu")
similarity = np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))
print(f"Độ tương đồng: {similarity:.4f}")
```

### Streaming Response (Phản Hồi Theo Luồng)

```python
def call_claude_streaming(prompt: str):
    """Gọi Claude với streaming — nhận token từng phần như ChatGPT."""
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": prompt}]
    }

    response = bedrock.invoke_model_with_response_stream(
        modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
        body=json.dumps(body)
    )

    for event in response["body"]:
        chunk = json.loads(event["chunk"]["bytes"])
        if chunk.get("type") == "content_block_delta":
            print(chunk["delta"]["text"], end="", flush=True)
```

---

## 📊 Kiến Trúc Ứng Dụng Generative AI Điển Hình

### Chatbot với Bedrock + Lambda + API Gateway

```
User Request
    │
    ▼
API Gateway  →  Lambda Function
                    │
                    ├──► DynamoDB (lưu conversation history — lịch sử hội thoại)
                    │
                    ├──► Bedrock Knowledge Base (RAG search — tìm kiếm RAG)
                    │
                    └──► Bedrock Runtime (gọi Foundation Model)
                              │
                              ▼
                    Response → API Gateway → User
```

### RAG Pipeline (Đường Ống Tăng Cường Truy Xuất)

```
1. Ingestion Phase (Giai Đoạn Nhập Liệu):
   Documents → Chunking (cắt nhỏ) → Embedding → Vector Store (OpenSearch/Pinecone)

2. Retrieval Phase (Giai Đoạn Truy Xuất):
   Query → Embedding → Vector Search → Top-K Relevant Chunks

3. Generation Phase (Giai Đoạn Tạo Sinh):
   [System Prompt] + [Retrieved Chunks] + [User Query] → Foundation Model → Answer
```

---

## 🔗 Điều Hướng Module

| Chủ Đề | File |
|---|---|
| Foundation Models — so sánh chi tiết | [1-foundation-models.md](1-foundation-models.md) |
| Prompt Engineering — kỹ thuật tạo prompt | [2-prompt-engineering.md](2-prompt-engineering.md) |
| RAG & Knowledge Base — tăng cường dữ liệu riêng | [3-rag-knowledge-base.md](3-rag-knowledge-base.md) |
| Bedrock Agents — tác nhân AI tự động | [4-bedrock-agents.md](4-bedrock-agents.md) |
| Fine-tuning — tinh chỉnh mô hình | [5-fine-tuning.md](5-fine-tuning.md) |

---

## 🎯 Câu Hỏi Phỏng Vấn Nhanh

1. **Bedrock khác SageMaker thế nào?** → Bedrock dùng Foundation Models có sẵn qua API; SageMaker để train/deploy custom models
2. **Khi nào dùng RAG thay vì Fine-tuning?** → RAG khi dữ liệu thay đổi thường xuyên hoặc cần source traceability (truy xuất nguồn); Fine-tuning khi cần thay đổi hành vi/style của model
3. **Guardrails giải quyết vấn đề gì?** → Kiểm soát nội dung độc hại, bảo vệ thông tin nhạy cảm, đảm bảo model không đi lạc chủ đề
4. **Provisioned Throughput khi nào cần?** → Khi cần SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) về latency và throughput cho production traffic ổn định

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
