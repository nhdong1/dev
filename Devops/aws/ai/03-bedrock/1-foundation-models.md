# Foundation Models (Mô Hình Nền Tảng) trên Amazon Bedrock

> Foundation Model — Mô Hình Nền Tảng là các mô hình AI được huấn luyện trên tập dữ liệu khổng lồ và có thể được áp dụng cho nhiều tác vụ khác nhau mà không cần huấn luyện lại từ đầu. Amazon Bedrock cung cấp quyền truy cập vào các Foundation Models hàng đầu từ nhiều provider (nhà cung cấp) thông qua một API thống nhất.

---

## 🧠 Foundation Model Là Gì?

### Định Nghĩa

Foundation Model (còn gọi là **Large Language Model — LLM — Mô Hình Ngôn Ngữ Lớn**) được đặc trưng bởi:

- **Scale** (Quy Mô): Hàng tỷ đến hàng nghìn tỷ tham số (parameters), huấn luyện trên petabytes dữ liệu
- **General purpose** (Đa Mục Đích): Một model có thể làm nhiều tác vụ: viết, dịch, tóm tắt, lập trình, phân tích
- **Emergent capabilities** (Khả Năng Nổi Sinh): Các khả năng xuất hiện tự nhiên khi scale đủ lớn, không được lập trình trực tiếp
- **Transfer learning** (Học Chuyển Giao): Fine-tune (tinh chỉnh) cho domain cụ thể chỉ cần ít dữ liệu

### Kiến Trúc Transformer

Hầu hết Foundation Models hiện đại dựa trên kiến trúc **Transformer** (ra mắt 2017):

```
Input Text → Tokenization (Token Hóa) → Embedding (Nhúng Vector)
                                               │
                                               ▼
                              ┌────────────────────────────────┐
                              │    Transformer Layers           │
                              │  ┌──────────────────────────┐  │
                              │  │ Multi-Head Self-Attention │  │  ← Chú ý đa đầu
                              │  │ (Cơ Chế Chú Ý Đa Đầu)   │  │
                              │  └──────────────────────────┘  │
                              │  ┌──────────────────────────┐  │
                              │  │  Feed-Forward Network     │  │  ← Mạng nơ-ron tiếp truyền
                              │  └──────────────────────────┘  │
                              │        × N layers              │
                              └────────────────────────────────┘
                                               │
                                               ▼
                              Output Probability Distribution (Phân Phối Xác Suất)
                                               │
                                               ▼
                                    Next Token Prediction (Dự Đoán Token Tiếp Theo)
```

---

## 🏢 Các Provider Và Model Trên Bedrock

### 1. Anthropic — Claude Series

**Anthropic** là công ty AI an toàn (AI safety company), nổi tiếng với kỹ thuật **Constitutional AI** (AI Hiến Pháp) — huấn luyện model tuân theo các nguyên tắc đạo đức.

#### Dòng Claude 3

| Model | Context Window | Điểm Mạnh | Chi Phí Tương Đối |
|---|---|---|---|
| **Claude 3.5 Sonnet** | 200K tokens | Tốt nhất cân bằng chất lượng-tốc độ-chi phí | Trung bình |
| **Claude 3 Opus** | 200K tokens | Độ chính xác cao nhất, phân tích phức tạp | Đắt nhất |
| **Claude 3 Haiku** | 200K tokens | Nhanh nhất, rẻ nhất, phù hợp real-time | Rẻ nhất |

#### Khả Năng Claude

- **Multimodal** (Đa Phương Thức): Hiểu cả text và image (ảnh, biểu đồ, tài liệu scan)
- **Code generation** (Tạo Mã): Viết, debug, giải thích code nhiều ngôn ngữ
- **Long document analysis** (Phân Tích Tài Liệu Dài): Xử lý tài liệu 100+ trang trong một lần
- **Tool use** (Sử Dụng Công Cụ): Gọi external APIs và functions (hàm)
- **Multilingual** (Đa Ngôn Ngữ): Hỗ trợ tiếng Việt và nhiều ngôn ngữ khác

#### Ví Dụ Gọi Claude với Vision (Thị Giác)

```python
import boto3
import json
import base64

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def analyze_image_with_claude(image_path: str, question: str) -> str:
    """Phân tích ảnh với Claude 3 Vision."""
    # Đọc và encode ảnh thành base64
    with open(image_path, "rb") as f:
        image_data = base64.b64encode(f.read()).decode("utf-8")

    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": "image/jpeg",
                            "data": image_data
                        }
                    },
                    {
                        "type": "text",
                        "text": question
                    }
                ]
            }
        ]
    }

    response = bedrock.invoke_model(
        modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]
```

---

### 2. Meta — Llama Series

**Meta** (Facebook) phát triển dòng model **Llama** (Large Language Model Meta AI) theo hướng open-source có điều kiện.

#### Llama 3.1 — Thế Hệ Mới Nhất

| Model | Parameters (Tham Số) | Context Window | Use Case |
|---|---|---|---|
| **Llama 3.1 8B** | 8 tỷ | 128K tokens | Edge/embedded, chi phí thấp |
| **Llama 3.1 70B** | 70 tỷ | 128K tokens | Cân bằng chất lượng-chi phí |
| **Llama 3.1 405B** | 405 tỷ | 128K tokens | Gần Claude Opus về chất lượng |

#### Đặc Điểm Llama

- **Open weights** (Trọng Số Mở): Có thể download và chạy locally (cục bộ) hoặc fine-tune thoải mái
- **Strong coding** (Lập Trình Mạnh): Đặc biệt tốt với code generation và debugging
- **Function calling** (Gọi Hàm): Hỗ trợ structured output và tool use
- **Commercial license** (Giấy Phép Thương Mại): Llama 3 cho phép dùng thương mại (với điều kiện)

#### Ví Dụ Gọi Llama

```python
def call_llama(prompt: str, max_tokens: int = 512) -> str:
    """Gọi Llama 3.1 70B qua Bedrock."""
    body = {
        "prompt": f"<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\n{prompt}<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n",
        "max_gen_len": max_tokens,
        "temperature": 0.7,
        "top_p": 0.9
    }

    response = bedrock.invoke_model(
        modelId="meta.llama3-1-70b-instruct-v1:0",
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    return result["generation"]
```

---

### 3. Amazon — Titan Models

**Amazon Titan** là dòng model do AWS tự phát triển, tối ưu cho tích hợp native với AWS ecosystem.

#### Các Model Titan

| Model | Loại | Chiều Embedding | Use Case |
|---|---|---|---|
| **Titan Text Lite** | Text generation | N/A | Tóm tắt, phân loại, Q&A đơn giản |
| **Titan Text Express** | Text generation | N/A | Chatbot, content creation phức tạp hơn |
| **Titan Embeddings V2** | Text embeddings | 1,024 dimensions | RAG, semantic search, clustering |
| **Titan Multimodal Embeddings** | Text+Image embeddings | 1,024 dimensions | Tìm kiếm đa phương thức |
| **Titan Image Generator** | Image generation | N/A | Tạo và chỉnh sửa ảnh từ text |

#### Điểm Đặc Biệt Titan

- **AWS-native security** (Bảo Mật Gốc AWS): Cam kết mạnh nhất về không dùng data để train
- **Watermarking** (Đóng Dấu Bản Quyền): Titan Image tự động nhúng invisible watermark vào ảnh được tạo
- **Bedrock Knowledge Base integration** (Tích Hợp Knowledge Base): Titan Embeddings là default embedding model cho Bedrock Knowledge Base

#### Ví Dụ Tạo Embedding với Titan

```python
def get_titan_embedding(text: str, dimensions: int = 1024) -> list[float]:
    """Tạo embedding vector với Titan Embeddings V2."""
    body = {
        "inputText": text,
        "dimensions": dimensions,        # 256, 512, hoặc 1024
        "normalize": True                # Chuẩn hóa vector về đơn vị
    }

    response = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    return result["embedding"]
```

---

### 4. Stability AI — Stable Diffusion

**Stability AI** cung cấp các model **Stable Diffusion** (Khuếch Tán Ổn Định) cho image generation (tạo ảnh từ văn bản).

#### Stable Diffusion XL (SDXL)

```python
def generate_image_sdxl(prompt: str, negative_prompt: str = "") -> bytes:
    """Tạo ảnh với Stable Diffusion XL."""
    body = {
        "text_prompts": [
            {"text": prompt, "weight": 1.0},
            {"text": negative_prompt, "weight": -1.0}  # Tránh những gì không muốn
        ],
        "cfg_scale": 10,          # Mức độ tuân theo prompt (1-35)
        "steps": 50,              # Số bước khuếch tán (nhiều hơn = chất lượng cao hơn, chậm hơn)
        "width": 1024,
        "height": 1024,
        "seed": 42                # Seed cố định để reproduce kết quả
    }

    response = bedrock.invoke_model(
        modelId="stability.stable-diffusion-xl-v1",
        body=json.dumps(body)
    )

    result = json.loads(response["body"].read())
    # Decode base64 image data
    import base64
    image_bytes = base64.b64decode(result["artifacts"][0]["base64"])
    return image_bytes
```

---

### 5. Cohere — Command & Embed

**Cohere** tập trung vào enterprise NLP — đặc biệt xuất sắc trong embeddings cho RAG.

| Model | Loại | Đặc Điểm |
|---|---|---|
| **Command R+** | Text generation | RAG-optimized, citation support (hỗ trợ trích dẫn nguồn) |
| **Command R** | Text generation | Phiên bản nhẹ hơn của Command R+ |
| **Embed v3** | Embeddings | SOTA (State-of-the-Art — Tiên Tiến Nhất) về retrieval accuracy |

---

### 6. Mistral AI — Efficient Models

**Mistral AI** (Pháp) nổi tiếng với hiệu năng vượt trội so với size:

| Model | Parameters | Kiến Trúc Đặc Biệt |
|---|---|---|
| **Mistral Large** | ~70B | Dense transformer |
| **Mixtral 8x7B** | ~46.7B active (MoE) | MoE — Mixture of Experts (Hỗn Hợp Chuyên Gia) |
| **Mistral 7B** | 7B | Nhỏ nhưng mạnh tương đương Llama 13B |

**MoE — Mixture of Experts** (Hỗn Hợp Chuyên Gia): Mỗi token chỉ kích hoạt 2/8 expert networks, giảm compute nhưng giữ chất lượng.

---

## 📊 So Sánh Chi Tiết Các Foundation Models

### Theo Tác Vụ

| Tác Vụ | Best Model | Runner-up | Lý Do |
|---|---|---|---|
| **Complex reasoning** (Lý luận phức tạp) | Claude 3 Opus | GPT-4 level | Chính xác nhất trong phân tích sâu |
| **Code generation** (Tạo mã) | Claude 3.5 Sonnet | Llama 3.1 70B | Hiểu ngữ nghĩa code tốt |
| **Fast response** (Phản hồi nhanh) | Claude 3 Haiku | Llama 3.1 8B | Latency thấp, chi phí thấp |
| **Long document** (Tài liệu dài) | Claude 3.5 Sonnet | Claude 3 Opus | 200K context window |
| **Embeddings cho RAG** | Cohere Embed v3 | Titan Embeddings V2 | Retrieval accuracy cao nhất |
| **Image generation** (Tạo ảnh) | Stable Diffusion XL | Titan Image | Chất lượng ảnh tốt |
| **Image understanding** (Hiểu ảnh) | Claude 3.5 Sonnet | Claude 3 Opus | Vision capabilities mạnh |
| **Cost-sensitive workloads** (Tiết kiệm chi phí) | Llama 3.1 8B | Claude 3 Haiku | Chi phí thấp nhất |

### Theo Đặc Điểm Kỹ Thuật

| Đặc Điểm | Claude 3.5 Sonnet | Llama 3.1 70B | Titan Text Express |
|---|---|---|---|
| Context Window | 200K | 128K | 8K |
| Multimodal | ✅ (text + image) | ❌ | ❌ |
| Tool use / Function calling | ✅ | ✅ | ❌ |
| Streaming | ✅ | ✅ | ✅ |
| Fine-tuning trên Bedrock | ✅ | ✅ | ✅ |
| Giá (tương đối) | Trung bình | Thấp | Thấp nhất |

---

## 🔧 Model IDs Trên Bedrock

```python
# Danh sách Model IDs cho boto3
MODEL_IDS = {
    # Anthropic Claude
    "claude_3_5_sonnet":  "anthropic.claude-3-5-sonnet-20241022-v2:0",
    "claude_3_opus":      "anthropic.claude-3-opus-20240229-v1:0",
    "claude_3_haiku":     "anthropic.claude-3-haiku-20240307-v1:0",

    # Meta Llama
    "llama_3_1_8b":       "meta.llama3-1-8b-instruct-v1:0",
    "llama_3_1_70b":      "meta.llama3-1-70b-instruct-v1:0",
    "llama_3_1_405b":     "meta.llama3-1-405b-instruct-v1:0",

    # Amazon Titan
    "titan_text_lite":    "amazon.titan-text-lite-v1",
    "titan_text_express": "amazon.titan-text-express-v1",
    "titan_embed_v2":     "amazon.titan-embed-text-v2:0",
    "titan_image":        "amazon.titan-image-generator-v2:0",

    # Stability AI
    "sdxl":               "stability.stable-diffusion-xl-v1",

    # Cohere
    "command_r_plus":     "cohere.command-r-plus-v1:0",
    "cohere_embed_v3":    "cohere.embed-english-v3",

    # Mistral
    "mistral_large":      "mistral.mistral-large-2402-v1:0",
    "mixtral_8x7b":       "mistral.mixtral-8x7b-instruct-v0:1",
}
```

### Liệt Kê Models Từ API

```python
def list_available_models():
    """Liệt kê tất cả Foundation Models đang available trong region."""
    bedrock_client = boto3.client("bedrock", region_name="us-east-1")

    response = bedrock_client.list_foundation_models(
        byOutputModality="TEXT"  # Lọc chỉ text models: TEXT, IMAGE, EMBEDDING
    )

    for model in response["modelSummaries"]:
        print(f"{model['modelId']} — Provider: {model['providerName']}")
```

---

## 💡 Chiến Lược Chọn Model

### Decision Tree (Cây Quyết Định)

```
Bạn cần gì?
│
├─ Tạo/phân tích văn bản?
│   ├─ Ngân sách hạn chế hoặc cần phản hồi nhanh?
│   │   └─► Claude 3 Haiku hoặc Llama 3.1 8B
│   ├─ Cần chất lượng tốt, cân bằng chi phí?
│   │   └─► Claude 3.5 Sonnet (lựa chọn mặc định tốt nhất)
│   └─ Cần phân tích cực kỳ phức tạp, không giới hạn chi phí?
│       └─► Claude 3 Opus
│
├─ Cần embedding cho RAG / semantic search?
│   ├─ Cần độ chính xác retrieval cao nhất?
│   │   └─► Cohere Embed v3
│   └─ Cần tích hợp đơn giản với Bedrock Knowledge Base?
│       └─► Titan Embeddings V2
│
├─ Cần tạo ảnh?
│   └─► Stable Diffusion XL hoặc Titan Image Generator
│
├─ Cần phân tích ảnh / vision?
│   └─► Claude 3.5 Sonnet (Vision)
│
└─ Cần fine-tune với dữ liệu riêng?
    ├─ Muốn flexibility cao nhất?
    │   └─► Llama 3.1 (open weights)
    └─ Cần kết quả tốt nhất với ít data?
        └─► Claude (instruction tuning)
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Foundation Model khác gì với Traditional ML Model (Mô Hình ML Truyền Thống)?**

> Foundation Model được huấn luyện trên dữ liệu cực lớn và đa dạng, có thể xử lý nhiều tác vụ (general purpose). Traditional ML model thường được train cho một tác vụ cụ thể (classification, regression) với labeled data (dữ liệu có nhãn) nhỏ hơn nhiều.

**Q: Khi nào chọn Claude vs Llama?**

> Chọn **Claude** khi cần độ chính xác cao, tuân thủ instruction tốt, safety quan trọng, hoặc cần context window dài. Chọn **Llama** khi cần fine-tune tự do với dữ liệu riêng (open weights), chi phí là ưu tiên, hoặc cần deploy locally.

**Q: Embedding model (mô hình nhúng) khác gì với text generation model?**

> Embedding model chuyển văn bản thành vector số (representation — biểu diễn), dùng cho similarity search, clustering, RAG. Text generation model sinh ra văn bản mới dựa trên input. Trong RAG pipeline, bạn cần CẢ HAI: embedding để tìm kiếm, generation để tổng hợp câu trả lời.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
