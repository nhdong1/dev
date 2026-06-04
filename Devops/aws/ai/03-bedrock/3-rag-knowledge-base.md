# RAG & Bedrock Knowledge Base (Cơ Sở Tri Thức Tăng Cường Truy Xuất)

> RAG — Retrieval-Augmented Generation — Tạo Sinh Tăng Cường Truy Xuất là kỹ thuật kết hợp thông tin được truy xuất từ cơ sở dữ liệu riêng với khả năng tạo sinh (generation) của Foundation Model để trả lời câu hỏi chính xác, có nguồn gốc, và cập nhật theo thời gian thực.

---

## 🎯 Tại Sao Cần RAG?

### Vấn Đề Của Foundation Model "Thuần"

| Vấn Đề | Mô Tả | Ảnh Hưởng |
|---|---|---|
| **Knowledge cutoff** (Kiến Thức Lỗi Thời) | Model chỉ biết thông tin đến thời điểm training | Trả lời sai về sự kiện mới |
| **Hallucination** (Ảo Giác) | Model "bịa" thông tin nghe có vẻ đúng | Thông tin sai không thể tin cậy |
| **No domain knowledge** (Thiếu Kiến Thức Domain) | Model không biết tài liệu nội bộ của bạn | Không thể trả lời về policy, products nội bộ |
| **No source traceability** (Không Truy Xuất Nguồn) | Không biết câu trả lời lấy từ đâu | Khó kiểm chứng và debug |

### RAG Giải Quyết Như Thế Nào?

```
Không có RAG:
User: "Chính sách hoàn tiền của công ty tôi là gì?"
LLM: [Bịa đặt hoặc từ chối]

Có RAG:
User: "Chính sách hoàn tiền của công ty tôi là gì?"
  → Tìm kiếm trong policy documents
  → Tìm được: "policy_2026_refund.pdf, trang 5"
  → LLM: "Theo chính sách ngày 01/01/2026, khách hàng được hoàn tiền trong 30 ngày..."
         [Có nguồn cụ thể, thông tin mới nhất, chính xác]
```

---

## 🏗️ Kiến Trúc RAG Pipeline

### Tổng Quan

```
┌────────────────── Giai Đoạn Ingestion (Nhập Liệu) ──────────────────┐
│                                                                       │
│  Documents     Chunking      Embedding      Vector Store             │
│  (PDF, Word, ──► (Cắt Nhỏ) ──► (Nhúng) ──► (Kho Vector)            │
│   HTML, S3)     Strategy      Model         OpenSearch/              │
│                               Titan v2      Pinecone                 │
└───────────────────────────────────────────────────────────────────────┘

┌────────────────── Giai Đoạn Retrieval + Generation ─────────────────┐
│                                                                       │
│  User Query → Embed → Vector Search → Top-K Chunks                  │
│                                              │                       │
│              ┌───────────────────────────────┘                       │
│              ▼                                                        │
│  [System] + [Retrieved Chunks] + [User Query] → LLM → Answer        │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Các Bước Chi Tiết

#### Bước 1: Document Processing (Xử Lý Tài Liệu)

```
Nguồn dữ liệu:
- S3 (PDF, DOCX, HTML, TXT, Markdown)
- Confluence pages
- SharePoint
- Web crawling

  ↓ Text extraction (Trích xuất văn bản)

Raw text → [Chunking Strategy] → Chunks
```

#### Bước 2: Chunking (Cắt Nhỏ Tài Liệu)

Chunking là việc chia tài liệu dài thành các đoạn nhỏ (chunks) phù hợp để nhúng và tìm kiếm:

| Chiến Lược | Mô Tả | Phù Hợp Cho |
|---|---|---|
| **Fixed-size** (Cố Định) | Cắt theo số token cố định (VD: 512 tokens) | Tài liệu đồng nhất |
| **Overlapping** (Chồng Lấp) | Cắt cố định nhưng có overlap 10-20% | Tránh mất ngữ cảnh tại ranh giới |
| **Semantic** (Ngữ Nghĩa) | Cắt tại ranh giới câu/đoạn tự nhiên | Tài liệu có cấu trúc rõ |
| **Hierarchical** (Phân Cấp) | Giữ parent-child (cha-con) relationship | FAQ, tài liệu có sections |

```python
# Ví dụ chunking với overlap
def chunk_text(text: str, chunk_size: int = 512, overlap: int = 64) -> list[str]:
    """
    Cắt văn bản thành chunks với overlap để giữ ngữ cảnh.

    Args:
        chunk_size: Số ký tự mỗi chunk
        overlap: Số ký tự chồng lấp giữa các chunks
    """
    chunks = []
    start = 0

    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start += chunk_size - overlap  # Bước tiến = chunk_size - overlap

    return chunks
```

#### Bước 3: Embedding (Nhúng Vector)

Mỗi chunk được chuyển thành một vector số (embedding) để có thể tính toán độ tương đồng:

```python
import boto3
import json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def embed_chunk(text: str) -> list[float]:
    """Tạo embedding vector cho một chunk văn bản."""
    body = {"inputText": text}
    response = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps(body)
    )
    return json.loads(response["body"].read())["embedding"]
    # Trả về list 1024 floats biểu diễn ngữ nghĩa của text
```

#### Bước 4: Vector Store (Kho Lưu Trữ Vector)

Lưu embeddings vào database hỗ trợ vector search (tìm kiếm theo độ tương đồng):

```
Vector Search sử dụng:
- ANN — Approximate Nearest Neighbor (Hàng Xóm Gần Đúng)
- Thuật toán: HNSW (Hierarchical Navigable Small World), IVF
- Độ đo: Cosine Similarity (Độ Tương Đồng Cosin), Dot Product, Euclidean Distance
```

#### Bước 5: Retrieval (Truy Xuất)

Khi user đặt câu hỏi, embed câu hỏi và tìm top-K chunks tương đồng nhất:

```python
import numpy as np

def cosine_similarity(vec1: list[float], vec2: list[float]) -> float:
    """Tính độ tương đồng cosin giữa hai vectors."""
    a, b = np.array(vec1), np.array(vec2)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def retrieve_top_k(query: str, doc_embeddings: list, doc_texts: list, k: int = 3) -> list[str]:
    """Tìm k chunks liên quan nhất với câu hỏi."""
    query_emb = embed_chunk(query)
    scores = [cosine_similarity(query_emb, doc_emb) for doc_emb in doc_embeddings]
    top_k_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:k]
    return [doc_texts[i] for i in top_k_indices]
```

---

## 📦 Bedrock Knowledge Base — Managed RAG (RAG Được Quản Lý)

Amazon Bedrock Knowledge Base là dịch vụ **fully-managed RAG** — tự động xử lý toàn bộ ingestion pipeline:

```
Bạn chỉ cần:
1. Chỉ định S3 bucket chứa documents
2. Chọn embedding model (Titan v2 mặc định)
3. Chọn vector store (OpenSearch Serverless mặc định)

Bedrock tự động làm:
- Parse PDF, DOCX, HTML, TXT, Markdown
- Chunk documents
- Tạo embeddings
- Lưu vào vector store
- Sync khi documents thay đổi
```

### Tạo Knowledge Base Qua Console

```
AWS Console → Bedrock → Knowledge Bases → Create Knowledge Base
    ↓
1. Đặt tên, chọn IAM role
2. Chọn data source: S3 bucket
3. Chọn embedding model: amazon.titan-embed-text-v2:0
4. Chọn vector store: Amazon OpenSearch Serverless (tự tạo collection)
5. Review & Create
```

### Tạo Và Sử Dụng Knowledge Base Qua Python

```python
import boto3
import json

# Client cho Bedrock control plane (quản lý Knowledge Base)
bedrock_agent = boto3.client("bedrock-agent", region_name="us-east-1")

# Client cho Bedrock runtime (query Knowledge Base)
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

# ── Tạo Knowledge Base ──────────────────────────────────────────────
def create_knowledge_base(
    name: str,
    s3_bucket: str,
    role_arn: str
) -> str:
    """Tạo Bedrock Knowledge Base và trả về knowledge_base_id."""
    response = bedrock_agent.create_knowledge_base(
        name=name,
        roleArn=role_arn,
        knowledgeBaseConfiguration={
            "type": "VECTOR",
            "vectorKnowledgeBaseConfiguration": {
                "embeddingModelArn": "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0"
            }
        },
        storageConfiguration={
            "type": "OPENSEARCH_SERVERLESS",
            "opensearchServerlessConfiguration": {
                "collectionArn": "arn:aws:aoss:us-east-1:...",  # ARN của AOSS collection
                "vectorIndexName": "bedrock-knowledge-base-index",
                "fieldMapping": {
                    "vectorField": "bedrock-knowledge-base-default-vector",
                    "textField": "AMAZON_BEDROCK_TEXT_CHUNK",
                    "metadataField": "AMAZON_BEDROCK_METADATA"
                }
            }
        }
    )
    return response["knowledgeBase"]["knowledgeBaseId"]


# ── Sync Data Source (Đồng Bộ Nguồn Dữ Liệu) ──────────────────────
def sync_knowledge_base(knowledge_base_id: str, data_source_id: str):
    """Trigger sync để ingest documents mới từ S3."""
    bedrock_agent.start_ingestion_job(
        knowledgeBaseId=knowledge_base_id,
        dataSourceId=data_source_id
    )


# ── Query Knowledge Base (Truy Vấn) ───────────────────────────────
def query_knowledge_base(knowledge_base_id: str, query: str) -> dict:
    """
    Truy vấn Knowledge Base và nhận câu trả lời có trích dẫn nguồn.

    Returns:
        {
            "answer": "Câu trả lời...",
            "citations": [{"chunk_text": "...", "source_uri": "s3://..."}]
        }
    """
    response = bedrock_agent_runtime.retrieve_and_generate(
        input={"text": query},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": knowledge_base_id,
                "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0",
                "retrievalConfiguration": {
                    "vectorSearchConfiguration": {
                        "numberOfResults": 5  # Lấy top 5 chunks liên quan
                    }
                }
            }
        }
    )

    answer = response["output"]["text"]
    citations = []

    for citation in response.get("citations", []):
        for ref in citation.get("retrievedReferences", []):
            citations.append({
                "chunk_text": ref["content"]["text"],
                "source_uri": ref["location"]["s3Location"]["uri"]
            })

    return {"answer": answer, "citations": citations}


# ── Chỉ Retrieve (Không Generate) ─────────────────────────────────
def retrieve_only(knowledge_base_id: str, query: str, top_k: int = 5) -> list[dict]:
    """
    Chỉ tìm kiếm chunks liên quan, không generate câu trả lời.
    Dùng khi muốn tự control generation hoặc dùng model khác.
    """
    response = bedrock_agent_runtime.retrieve(
        knowledgeBaseId=knowledge_base_id,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {
                "numberOfResults": top_k,
                "overrideSearchType": "HYBRID"  # Kết hợp vector + keyword search
            }
        }
    )

    chunks = []
    for result in response["retrievalResults"]:
        chunks.append({
            "text": result["content"]["text"],
            "score": result["score"],
            "source": result["location"]["s3Location"]["uri"]
        })

    return chunks
```

---

## 🔍 Vector Stores Được Hỗ Trợ

### 1. Amazon OpenSearch Serverless (Mặc Định)

**Amazon OpenSearch Serverless** — OpenSearch (Tìm Kiếm Mở Rộng) Không Cần Máy Chủ:

```
Ưu điểm:
✅ Tích hợp native với Bedrock — zero configuration
✅ Serverless — tự động scale (mở rộng), không quản lý cluster
✅ Hỗ trợ cả vector search và keyword search (BM25)
✅ Hybrid search = vector + keyword (tốt nhất cho RAG)

Nhược điểm:
❌ Đắt hơn các alternatives khi data nhỏ
❌ Minimum billing = 2 OCUs (OpenSearch Compute Units)
```

### 2. Amazon Aurora PostgreSQL với pgvector

Dùng PostgreSQL extension **pgvector** để lưu vectors trong relational database:

```sql
-- Kích hoạt pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Tạo bảng lưu documents với embedding
CREATE TABLE documents (
    id          SERIAL PRIMARY KEY,
    content     TEXT,
    source_uri  TEXT,
    embedding   vector(1024),  -- 1024 dimensions cho Titan v2
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tạo index HNSW cho tìm kiếm nhanh
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Tìm 5 documents gần nhất
SELECT content, source_uri,
       1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

### 3. MongoDB Atlas

```python
# MongoDB Atlas Vector Search configuration (cấu hình)
atlas_vector_search_index = {
    "name": "vector_index",
    "type": "vectorSearch",
    "definition": {
        "fields": [
            {
                "type": "vector",
                "path": "embedding",
                "numDimensions": 1024,
                "similarity": "cosine"
            }
        ]
    }
}
```

### 4. Pinecone (Third-party)

```python
import pinecone

# Khởi tạo Pinecone index
pc = pinecone.Pinecone(api_key="your-api-key")
index = pc.Index("bedrock-rag-index")

# Upsert (Thêm/Cập Nhật) vectors
index.upsert(vectors=[
    {
        "id": "doc_001_chunk_0",
        "values": embedding_vector,  # 1024 floats
        "metadata": {
            "text": chunk_text,
            "source": "s3://my-bucket/policy.pdf",
            "page": 5
        }
    }
])

# Query
results = index.query(
    vector=query_embedding,
    top_k=5,
    include_metadata=True
)
```

### So Sánh Vector Stores

| Vector Store | Managed | Cost | Hybrid Search | Phù Hợp |
|---|---|---|---|---|
| OpenSearch Serverless | ✅ Full | $$$ | ✅ | Production, cần hybrid search |
| Aurora pgvector | ✅ (RDS) | $$ | ❌ | Đã dùng PostgreSQL, data nhỏ |
| MongoDB Atlas | ✅ | $$ | ✅ | Đã dùng MongoDB |
| Pinecone | ✅ | $$ | ❌ | Pure vector search, simple setup |
| OpenSearch managed | ✅ | $$ | ✅ | Cần kiểm soát cluster |

---

## 🔧 Advanced RAG Techniques (Kỹ Thuật RAG Nâng Cao)

### 1. Hybrid Search (Tìm Kiếm Kết Hợp)

Kết hợp **vector search** (hiểu ngữ nghĩa) + **BM25 keyword search** (tìm từ khóa chính xác):

```python
# Hybrid search với OpenSearch
def hybrid_search(knowledge_base_id: str, query: str) -> list[dict]:
    response = bedrock_agent_runtime.retrieve(
        knowledgeBaseId=knowledge_base_id,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {
                "numberOfResults": 10,
                "overrideSearchType": "HYBRID"  # Vector + BM25
            }
        }
    )
    return response["retrievalResults"]
```

### 2. Metadata Filtering (Lọc Theo Metadata)

Lọc kết quả tìm kiếm theo thuộc tính tài liệu:

```python
# Chỉ tìm trong tài liệu thuộc category "finance" và sau năm 2024
response = bedrock_agent_runtime.retrieve(
    knowledgeBaseId=knowledge_base_id,
    retrievalQuery={"text": "chính sách hoàn tiền"},
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "numberOfResults": 5,
            "filter": {
                "andAll": [
                    {
                        "equals": {
                            "key": "category",
                            "value": "finance"
                        }
                    },
                    {
                        "greaterThan": {
                            "key": "year",
                            "value": 2024
                        }
                    }
                ]
            }
        }
    }
)
```

### 3. Re-ranking (Xếp Hạng Lại)

Sau khi retrieve, dùng cross-encoder model để xếp hạng lại kết quả cho chính xác hơn:

```
Initial retrieval (Truy xuất ban đầu):
  Query → Top-100 chunks (bi-encoder, nhanh)

Re-ranking (Xếp hạng lại):
  (Query, Chunk) pairs → Cross-encoder → Scores → Top-5 chunks (chính xác hơn)

Chi phí: Chạy thêm model cross-encoder nhưng kết quả tốt hơn đáng kể
```

### 4. Contextual Compression (Nén Ngữ Cảnh)

Sau khi retrieve chunks, yêu cầu LLM trích xuất chỉ phần liên quan:

```python
def compress_retrieved_chunks(query: str, chunks: list[str]) -> list[str]:
    """
    Dùng LLM để trích xuất chỉ phần liên quan từ mỗi chunk.
    Giảm token count trước khi đưa vào final prompt.
    """
    compressed = []

    for chunk in chunks:
        prompt = f"""Từ đoạn văn bản sau, trích xuất CHỈ các câu liên quan đến câu hỏi:
        "{query}"

        Văn bản: {chunk}

        Nếu không có câu nào liên quan, trả lời: "KHÔNG LIÊN QUAN"

        Phần liên quan:"""

        # Dùng model nhỏ/rẻ để compress
        result = call_claude_haiku(prompt)
        if "KHÔNG LIÊN QUAN" not in result:
            compressed.append(result)

    return compressed
```

### 5. Query Transformation (Biến Đổi Câu Hỏi)

Cải thiện recall (khả năng tìm thấy) bằng cách biến đổi câu hỏi trước khi search:

```python
def multi_query_retrieval(knowledge_base_id: str, original_query: str) -> list[dict]:
    """
    Tạo nhiều biến thể của câu hỏi để tăng recall.
    Kỹ thuật: Multi-query retrieval
    """
    # Bước 1: Tạo 3 biến thể câu hỏi
    generate_prompt = f"""Tạo 3 cách diễn đạt khác nhau cho câu hỏi sau.
    Mỗi biến thể trên một dòng, không đánh số.

    Câu hỏi gốc: {original_query}

    Các biến thể:"""

    variants_text = call_claude_haiku(generate_prompt)
    all_queries = [original_query] + variants_text.strip().split("\n")

    # Bước 2: Retrieve cho mỗi variant
    all_results = {}  # Dùng dict để deduplicate (loại trùng lặp)

    for query in all_queries:
        results = retrieve_only(knowledge_base_id, query, top_k=3)
        for result in results:
            # Key = hash của text để deduplicate
            key = hash(result["text"])
            if key not in all_results:
                all_results[key] = result

    return list(all_results.values())
```

---

## 📊 RAG vs Fine-tuning — Khi Nào Chọn Cái Nào?

| Tiêu Chí | RAG | Fine-tuning |
|---|---|---|
| **Cập nhật kiến thức** | Dễ — thêm documents vào vector store | Khó — cần retrain |
| **Source traceability** (Truy xuất nguồn) | ✅ Biết câu trả lời từ đâu | ❌ Không biết |
| **Data requirements** (Yêu Cầu Dữ Liệu) | Ít — documents không cần labeled | Nhiều — cần nhiều Q&A pairs |
| **Kiến thức mới** | Cập nhật realtime | Cần training lại |
| **Hallucination** (Ảo Giác) | Thấp (grounded in documents) | Có thể cao hơn nếu data ít |
| **Cost** (Chi Phí) | Vector store + retrieval cost | Fine-tuning cost cao ban đầu |
| **Latency** (Độ Trễ) | Cao hơn (retrieval + generation) | Thấp hơn (generation only) |
| **Custom behavior/style** (Hành Vi Tùy Chỉnh) | ❌ Khó thay đổi style model | ✅ Dạy model nói theo cách riêng |
| **Domain-specific format** (Định Dạng Đặc Thù) | ❌ | ✅ Model học output format |

**Nguyên tắc chọn:**
- **Dùng RAG** khi: Cần trả lời dựa trên documents cụ thể, documents thay đổi, cần trace nguồn
- **Dùng Fine-tuning** khi: Cần model adopt phong cách/format đặc biệt, domain vocabulary khác lạ, không đủ runtime cho retrieval
- **Dùng cả hai** (RAG + Fine-tuned model): Best of both worlds nhưng phức tạp và đắt nhất

---

## 🔒 Bảo Mật RAG

### Phân Quyền Truy Cập Tài Liệu

```python
# Mỗi user chỉ được retrieve documents họ có quyền đọc
def secure_retrieve(
    knowledge_base_id: str,
    query: str,
    user_department: str
) -> list[dict]:
    """RAG có kiểm soát quyền truy cập theo department."""
    return retrieve_only(
        knowledge_base_id,
        query,
        # Filter chỉ lấy docs thuộc department của user
        # (Cần store metadata khi ingest)
    )

# Trong thực tế: implement trong Lambda authorizer hoặc
# dùng IAM conditions để restrict Knowledge Base access
```

### Bảo Mật Dữ Liệu Nhạy Cảm

```
Trước khi ingest vào Knowledge Base:
1. PII Scrubbing (Xóa Thông Tin Cá Nhân): Dùng Comprehend để detect và redact PII
2. Classification (Phân Loại): Tag documents theo sensitivity level
3. Encryption (Mã Hóa): S3 server-side encryption, OpenSearch encryption at rest
4. Access Control (Kiểm Soát Truy Cập): IAM policy cho Knowledge Base
5. Audit (Kiểm Tra): CloudTrail log tất cả retrieve calls
```

---

## 📐 Chunking Best Practices (Thực Hành Tốt Nhất)

```
Chunk size recommendations (khuyến nghị kích thước chunk):
- 256-512 tokens: Tốt cho Q&A, factual retrieval
- 512-1024 tokens: Tốt cho tóm tắt, phân tích
- 1024+ tokens: Tốt cho document synthesis

Overlap recommendations:
- 10-20% overlap: Giữ ngữ cảnh tại ranh giới
- VD: Chunk 512 tokens → Overlap 50-100 tokens

Metadata to store (metadata cần lưu):
- source_uri: S3 path để trích dẫn nguồn
- doc_title: Tiêu đề tài liệu
- page_number: Số trang
- created_at: Ngày tạo/cập nhật
- category/department: Để filter
- language: Ngôn ngữ tài liệu
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: RAG là gì? Tại sao quan trọng?**

> RAG — Retrieval-Augmented Generation là kỹ thuật kết hợp information retrieval (truy xuất thông tin) với text generation. Thay vì chỉ dựa vào kiến thức được baked in (nhúng sẵn) trong model, RAG cho phép model truy cập tài liệu mới nhất và trả lời có nguồn gốc rõ ràng, giảm hallucination.

**Q: Khi nào dùng RAG, khi nào dùng Fine-tuning?**

> Dùng **RAG** khi cần: dữ liệu cập nhật thường xuyên, cần cite source (trích dẫn nguồn), documents thay đổi, ít labeled data. Dùng **Fine-tuning** khi cần: model học style/format đặc biệt, domain vocabulary, latency thấp, không muốn thêm retrieval step.

**Q: Vector Database (Cơ Sở Dữ Liệu Vector) hoạt động như thế nào?**

> Mỗi document/chunk được chuyển thành embedding vector (tập hợp số thực). Vector database lưu các vectors này và tối ưu cho ANN search — tìm các vectors "gần nhất" với query vector theo cosine similarity hoặc dot product. Algorithms như HNSW và IVF cho phép tìm kiếm trong milliseconds dù có hàng triệu vectors.

**Q: Chunking strategy (chiến lược cắt nhỏ) ảnh hưởng thế nào đến chất lượng RAG?**

> Chunk quá nhỏ: Mất ngữ cảnh, model không có đủ thông tin để trả lời. Chunk quá lớn: Retrieval kém chính xác (1 chunk chứa nhiều chủ đề), tốn nhiều token. Optimal thường là 512 tokens với 10-15% overlap. Hierarchical chunking (lưu cả parent và child chunks) cho kết quả tốt nhất.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
