# OpenSearch Fundamentals — Nền Tảng Amazon OpenSearch

> Nắm vững các khái niệm cốt lõi: Cluster (Cụm), Index (Chỉ Mục), Shard (Mảnh), Replica (Bản Sao), Mapping (Ánh Xạ) và Document (Tài Liệu) — nền tảng để thiết kế và vận hành OpenSearch hiệu quả.

---

## 📚 Mục Lục

1. [Kiến Trúc Cluster](#1-kiến-trúc-cluster)
2. [Index — Chỉ Mục](#2-index--chỉ-mục)
3. [Shards & Replicas](#3-shards--replicas)
4. [Node Types — Loại Node](#4-node-types--loại-node)
5. [Mapping & Data Types](#5-mapping--data-types)
6. [Query DSL — Ngôn Ngữ Truy Vấn](#6-query-dsl--ngôn-ngữ-truy-vấn)
7. [Index Lifecycle — Vòng Đời Chỉ Mục](#7-index-lifecycle--vòng-đời-chỉ-mục)
8. [OpenSearch Serverless](#8-opensearch-serverless)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Kiến Trúc Cluster

### 1.1 Cluster Là Gì?

**Cluster** (Cụm) là tập hợp một hoặc nhiều node (Node — Nút) OpenSearch cùng lưu trữ và xử lý dữ liệu. Mỗi cluster có một tên duy nhất (domain name trong AWS).

```
┌────────────────────────────── OpenSearch Domain ─────────────────────────┐
│                                                                            │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐        │
│   │  Dedicated      │   │  Data Node 1    │   │  Data Node 2    │        │
│   │  Master Node    │   │  (AZ-a)         │   │  (AZ-b)         │        │
│   │  (Không lưu     │   │  ┌───────────┐  │   │  ┌───────────┐  │        │
│   │   data — chỉ    │   │  │ Primary   │  │   │  │ Replica   │  │        │
│   │   quản lý       │   │  │ Shard 0   │  │   │  │ Shard 0   │  │        │
│   │   cluster)      │   │  ├───────────┤  │   │  ├───────────┤  │        │
│   └─────────────────┘   │  │ Replica   │  │   │  │ Primary   │  │        │
│                          │  │ Shard 1   │  │   │  │ Shard 1   │  │        │
│   ┌─────────────────┐   │  └───────────┘  │   │  └───────────┘  │        │
│   │  UltraWarm Node │   └─────────────────┘   └─────────────────┘        │
│   │  (Dữ liệu ít    │                                                     │
│   │   truy cập)     │   ┌─────────────────────────────────────────────┐  │
│   └─────────────────┘   │          OpenSearch Dashboards               │  │
│                          │  (Giao Diện Trực Quan — Tương Tự Kibana)    │  │
│                          └─────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Domain vs Cluster

Trong Amazon OpenSearch Service, **domain** (Miền) là khái niệm AWS tương đương với cluster trong OpenSearch thuần:

```
AWS Console:  "Create domain"   ←──→  Tạo OpenSearch Cluster
AWS Console:  "Domain endpoint" ←──→  Địa chỉ kết nối đến cluster
```

---

## 2. Index — Chỉ Mục

### 2.1 Index Là Gì?

**Index** (Chỉ Mục) là đơn vị lưu trữ logic — tương tự như một bảng (table) trong cơ sở dữ liệu quan hệ, nhưng linh hoạt hơn. Mỗi index chứa các **document** (Tài Liệu) có cấu trúc JSON.

```
Index: "products"
├── Document 1: { "id": "p001", "name": "Tai nghe Sony WH-1000XM5", "price": 8500000 }
├── Document 2: { "id": "p002", "name": "Laptop Dell XPS 13", "price": 35000000 }
└── Document 3: { "id": "p003", "name": "Chuột Logitech MX Master 3", "price": 2500000 }
```

### 2.2 Document — Tài Liệu

Document là đơn vị dữ liệu nhỏ nhất trong OpenSearch — một đối tượng JSON.

```json
{
  "_index": "products",
  "_id":    "p001",
  "_source": {
    "name":        "Tai nghe Sony WH-1000XM5",
    "category":    "Electronics",
    "price":       8500000,
    "tags":        ["wireless", "noise-cancelling", "premium"],
    "created_at":  "2024-01-15T10:30:00Z",
    "description": "Tai nghe chống ồn hàng đầu với 30 giờ pin"
  }
}
```

**Các trường metadata quan trọng:**

| Trường | Ý Nghĩa |
|--------|---------|
| `_index` | Index chứa document này |
| `_id` | Định danh duy nhất (tự động sinh hoặc do bạn chỉ định) |
| `_source` | Nội dung JSON gốc của document |
| `_score` | Điểm liên quan (relevance score) khi tìm kiếm |
| `_version` | Phiên bản document (tăng mỗi lần cập nhật) |

### 2.3 CRUD Operations — Thao Tác Cơ Bản

```bash
# Tạo Index
PUT /products
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 }
}

# Thêm Document (Index a Document)
PUT /products/_doc/p001
{ "name": "Tai nghe Sony", "price": 8500000 }

# Lấy Document theo ID
GET /products/_doc/p001

# Cập nhật một phần (Partial Update)
POST /products/_update/p001
{ "doc": { "price": 7900000 } }

# Xóa Document
DELETE /products/_doc/p001

# Xóa toàn bộ Index
DELETE /products
```

---

## 3. Shards & Replicas

### 3.1 Shard — Mảnh

**Shard** (Mảnh) là đơn vị phân phối vật lý. Mỗi index được chia thành nhiều shard, mỗi shard là một Apache Lucene index độc lập.

```
Index: "logs-2024-01"  (10 triệu documents)
│
├── Primary Shard 0  ──  Node 1  (3.3 triệu docs)
├── Primary Shard 1  ──  Node 2  (3.3 triệu docs)
└── Primary Shard 2  ──  Node 3  (3.4 triệu docs)
```

**Tại sao cần sharding?**
- **Horizontal scaling** (Mở Rộng Theo Chiều Ngang): Phân phối dữ liệu ra nhiều node
- **Parallel processing** (Xử Lý Song Song): Nhiều shard → query chạy song song → nhanh hơn

### 3.2 Replica — Bản Sao

**Replica shard** (Mảnh Bản Sao) là bản sao của primary shard, phục vụ hai mục đích:

1. **High Availability — Tính Sẵn Sàng Cao:** Khi node chứa primary shard bị lỗi, replica trở thành primary
2. **Read Throughput — Thông Lượng Đọc:** Query có thể được phục vụ bởi cả primary lẫn replica

```
Với number_of_replicas = 1:

Node 1 (AZ-a):   Primary-0  │  Replica-1  │  Replica-2
Node 2 (AZ-b):   Replica-0  │  Primary-1  │  Replica-2   (sai — sẽ sửa bên dưới)
Node 3 (AZ-c):   Replica-0  │  Replica-1  │  Primary-2

Quy tắc: Primary và Replica của cùng 1 shard KHÔNG được nằm trên cùng 1 node
```

### 3.3 Cách Tính Số Shard Phù Hợp

```
Công thức gốc (rule of thumb — quy tắc ngón tay cái):
    Số shard = (Dung lượng dữ liệu / 20-40 GB) + buffer

Ví dụ:
    Dữ liệu: 200 GB
    Target: 10 GB / shard → 20 primary shards

Lưu ý quan trọng:
    - Số primary shard KHÔNG thay đổi được sau khi tạo index (cần reindex)
    - Số replica shard CÓ THỂ thay đổi bất kỳ lúc nào
    - Quá nhiều shard nhỏ = overhead lớn (chi phí cluster state)
    - Quá ít shard lớn = query chậm, không tận dụng được parallel processing
```

### 3.4 Shard Sizing — Kích Thước Shard

| Kích Thước Shard | Đánh Giá |
|-----------------|---------|
| < 10 GB | Quá nhỏ — nhiều overhead |
| 10–30 GB | Tốt cho log/time-series data |
| 30–50 GB | Tốt cho general purpose |
| > 50 GB | Cần xem lại — query có thể chậm |

---

## 4. Node Types — Loại Node

### 4.1 Master Node — Node Chủ

**Dedicated Master Node** (Node Chủ Chuyên Dụng) chịu trách nhiệm quản lý cluster state (Trạng Thái Cụm), không lưu trữ dữ liệu.

```
Nhiệm vụ của Master Node:
├── Theo dõi danh sách nodes trong cluster
├── Quản lý index creation/deletion
├── Quyết định shard allocation (Phân Bổ Mảnh)
└── Duy trì cluster metadata

Khuyến nghị: Luôn dùng 3 dedicated master nodes cho production
(tránh "split-brain" — Tình Trạng Não Phân Đôi — khi cluster bị chia cắt)
```

### 4.2 Data Node — Node Dữ Liệu

Data Node lưu trữ dữ liệu thực tế và thực thi các query tìm kiếm.

```
Chọn instance type cho Data Node:
├── r6g.* (Memory-optimized) → Phù hợp khi cần nhiều RAM cho index caching
├── c6g.* (Compute-optimized) → Phù hợp khi query nhiều, ít dữ liệu
└── m6g.* (Balanced)          → Tổng hợp, phù hợp hầu hết workload
```

### 4.3 UltraWarm Node — Node Ấm

**UltraWarm** là tier lưu trữ dữ liệu ít truy cập hơn, dùng S3 làm backend (phía sau) thay vì EBS:

```
Hot Tier (Data Node)   ─ EBS ─ Query < 100ms
        │
        │ ISM Policy (sau 7 ngày)
        ▼
UltraWarm Tier         ─ S3  ─ Query 100ms–1s  ─ ~90% rẻ hơn
        │
        │ ISM Policy (sau 30 ngày)
        ▼
Cold Storage           ─ S3  ─ Cần attach trước khi query ─ Rẻ nhất
```

### 4.4 Coordinating Node — Node Điều Phối

Nhận request từ client, phân phối query đến data nodes, tổng hợp kết quả và trả về. Trong cluster nhỏ, data node kiêm luôn vai trò này.

---

## 5. Mapping & Data Types

### 5.1 Mapping Là Gì?

**Mapping** (Ánh Xạ) định nghĩa cách các trường trong document được lập chỉ mục và lưu trữ — tương tự schema trong database.

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "vietnamese"
      },
      "price": {
        "type": "double"
      },
      "category": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date",
        "format": "yyyy-MM-dd'T'HH:mm:ss'Z'"
      },
      "tags": {
        "type": "keyword"
      },
      "in_stock": {
        "type": "boolean"
      }
    }
  }
}
```

### 5.2 Text vs Keyword — Hai Kiểu Chuỗi Quan Trọng

| | `text` | `keyword` |
|--|--------|-----------|
| **Dùng cho** | Nội dung cần full-text search | Giá trị cần exact match / aggregation |
| **Analyzed** | Có (qua analyzer — bộ phân tích) | Không |
| **Ví dụ** | Tiêu đề bài viết, mô tả sản phẩm | Trạng thái đơn hàng, mã SKU, email |
| **Filter** | Match query | Term query |
| **Aggregation** | Không hỗ trợ | Có hỗ trợ |

**Ví dụ minh họa:**

```
Lưu trữ: "Tai nghe chống ồn Sony"

Nếu là text (analyzed):
    Tokens: ["tai", "nghe", "chong", "on", "sony"]
    → Tìm "Sony" ✅ | Tìm "tai nghe" ✅ | Aggregation ❌

Nếu là keyword (not analyzed):
    Lưu nguyên: "Tai nghe chống ồn Sony"
    → Tìm exact "Tai nghe chống ồn Sony" ✅ | Tìm "Sony" ❌ | Aggregation ✅
```

### 5.3 Dynamic Mapping — Ánh Xạ Tự Động

OpenSearch tự động suy luận kiểu dữ liệu khi nhận document đầu tiên:

```json
// Document gửi lên (không khai báo mapping trước):
{ "order_id": "ORD001", "amount": 1500.50, "created_at": "2024-01-15" }

// OpenSearch tự tạo mapping:
{
  "order_id":   { "type": "text" + "keyword" (multi-field) },
  "amount":     { "type": "float" },
  "created_at": { "type": "date" }
}
```

**Lưu ý:** Dynamic mapping tiện lợi nhưng có thể gây "mapping explosion" (Bùng Nổ Mapping) khi có quá nhiều trường động — nên dùng `dynamic: "strict"` trong production (Môi Trường Sản Xuất).

---

## 6. Query DSL — Ngôn Ngữ Truy Vấn

### 6.1 Cấu Trúc Query Cơ Bản

```json
POST /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "tai nghe" } }
      ],
      "filter": [
        { "term":  { "category": "Electronics" } },
        { "range": { "price": { "gte": 1000000, "lte": 10000000 } } }
      ],
      "must_not": [
        { "term": { "in_stock": false } }
      ]
    }
  },
  "sort": [
    { "_score": "desc" },
    { "price": "asc" }
  ],
  "from": 0,
  "size": 10
}
```

### 6.2 Các Loại Query Phổ Biến

| Query | Mục Đích | Ví Dụ |
|-------|---------|-------|
| `match` | Full-text search trên field `text` | Tìm sản phẩm theo tên |
| `term` | Exact match trên field `keyword` | Lọc theo trạng thái đơn hàng |
| `range` | So sánh số/ngày | Lọc giá từ 1M đến 10M |
| `bool` | Kết hợp nhiều điều kiện | must + filter + must_not |
| `multi_match` | Tìm trên nhiều field cùng lúc | Tìm từ khóa trong cả name và description |
| `fuzzy` | Tìm kiếm gần đúng (chịu lỗi chính tả) | "soni" → tìm được "sony" |
| `wildcard` | Tìm kiếm với ký tự đại diện | "tai*" → tai nghe, tai phone |
| `prefix` | Tìm kiếm tiền tố | "lap" → laptop |

### 6.3 Aggregations — Tổng Hợp Dữ Liệu

Aggregation (Tổng Hợp) tương tự GROUP BY trong SQL — phân tích dữ liệu thống kê:

```json
POST /orders/_search
{
  "size": 0,
  "aggs": {
    "revenue_by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "total_revenue": { "sum": { "field": "amount" } },
        "avg_order":     { "avg": { "field": "amount" } }
      }
    },
    "daily_orders": {
      "date_histogram": {
        "field":             "created_at",
        "calendar_interval": "day"
      },
      "aggs": {
        "order_count": { "value_count": { "field": "_id" } }
      }
    }
  }
}
```

---

## 7. Index Lifecycle — Vòng Đời Chỉ Mục

### 7.1 ISM — Index State Management (Quản Lý Trạng Thái Chỉ Mục)

ISM cho phép tự động hóa các hành động theo vòng đời của index — di chuyển dữ liệu từ hot sang warm sang cold, và cuối cùng xóa đi.

```
ISM Policy ví dụ cho Log Index:

Ngày 0-7:    Hot tier  → Ghi và đọc nhanh trên EBS
     │
     │ Sau 7 ngày (rollover hoặc age-based)
     ▼
Ngày 7-30:   UltraWarm → Di chuyển sang S3-backed storage
     │
     │ Sau 30 ngày
     ▼
Ngày 30-90:  Cold Storage → Ít truy cập nhất
     │
     │ Sau 90 ngày
     ▼
             DELETE → Xóa index tự động
```

**Cấu hình ISM Policy:**

```json
PUT _plugins/_ism/policies/log-lifecycle-policy
{
  "policy": {
    "description": "Vòng đời log index",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [
          {
            "state_name": "warm",
            "conditions": { "min_index_age": "7d" }
          }
        ]
      },
      {
        "name": "warm",
        "actions": [
          { "warm_migration": {} }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": { "min_index_age": "90d" }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          { "delete": {} }
        ],
        "transitions": []
      }
    ]
  }
}
```

### 7.2 Index Rollover — Xoay Vòng Chỉ Mục

Rollover tự động tạo index mới khi index hiện tại đạt ngưỡng kích thước hoặc số document — pattern phổ biến cho log pipeline:

```
logs-000001 (hiện tại, đang ghi)
    │
    │ Khi: size > 40GB HOẶC doc_count > 10M HOẶC age > 1 ngày
    ▼
logs-000002 (index mới tự tạo — ghi tiếp vào đây)
logs-000001 (chuyển sang read-only)
```

---

## 8. OpenSearch Serverless

### 8.1 Tổng Quan

**Amazon OpenSearch Serverless** (tháng 4/2023) loại bỏ hoàn toàn việc quản lý cluster — tự động scale, không cần cấu hình node:

```
OpenSearch Service (Có Quản Lý)    vs    OpenSearch Serverless
──────────────────────────────────────────────────────────────
Chọn instance type                       Không cần chọn instance
Cấu hình shard/replica                   Tự động quản lý
Scale thủ công (thêm node)              Auto-scale theo demand
Giá: giờ × số node                       Giá: OCU (giờ) × số lượng
Min ~$120/tháng (nhỏ nhất)              Min ~$700/tháng (2 OCU min)
```

### 8.2 OCU — OpenSearch Compute Unit (Đơn Vị Tính Toán OpenSearch)

```
OCU = 6 GB RAM + 2 vCPU

Tối thiểu: 2 OCU cho indexing + 2 OCU cho search = 4 OCU
Chi phí tối thiểu: ~$700/tháng

→ Serverless phù hợp cho workload không đều (spiky)
→ Không phù hợp cho cluster nhỏ chạy liên tục
```

### 8.3 Collections — Tuyển Tập

Serverless tổ chức data thành **collections** thay vì domain:

| Loại Collection | Mục Đích |
|----------------|---------|
| **Search** | Full-text search, low latency queries |
| **Time series** | Log analytics, metrics, time-based data |
| **Vector search** | Semantic search, AI/ML embeddings |

---

## 9. Câu Hỏi Phỏng Vấn

### Q1: Shard và Replica khác nhau thế nào? Tại sao cần cả hai?

**Trả lời:**
- **Primary Shard** (Mảnh Chính): Phân chia dữ liệu ra nhiều phần để lưu trữ trên nhiều node → horizontal scaling (Mở Rộng Theo Chiều Ngang). Số lượng primary shard cố định sau khi tạo index.
- **Replica Shard** (Mảnh Bản Sao): Bản sao của primary shard, luôn nằm trên node khác primary. Phục vụ hai mục đích: (1) failover (Chuyển Đổi Dự Phòng) khi node bị lỗi và (2) tăng read throughput (Thông Lượng Đọc) vì query có thể đến primary hoặc replica.

### Q2: Tại sao không nên tạo quá nhiều shard nhỏ?

**Trả lời:** Mỗi shard là một Lucene index — tốn RAM và CPU để duy trì. Với 1,000 shard × 10,000 documents mỗi shard, overhead cluster management lớn hơn benefit. AWS khuyến nghị mỗi shard khoảng 10–50 GB. Quá nhiều shard nhỏ gây "oversharding" — query phải tổng hợp kết quả từ quá nhiều shard, tăng latency (Độ Trễ).

### Q3: Sự khác biệt giữa `text` và `keyword` mapping type?

**Trả lời:**
- **`text`:** Được phân tích (analyzed) — tách thành tokens, lowercase, loại bỏ stopwords. Dùng cho `match` query (full-text search). Không thể dùng cho aggregation.
- **`keyword`:** Lưu nguyên giá trị — exact match. Dùng cho `term` query, sorting, aggregation. Ví dụ: status code, email, category.
- Trường hợp cần cả hai: dùng **multi-field mapping** với `field.keyword` subfield.

### Q4: Làm thế nào để tránh mapping explosion?

**Trả lời:** Mapping explosion xảy ra khi dữ liệu có quá nhiều field động (ví dụ log với key tùy ý). Giải pháp:
1. Đặt `"dynamic": "strict"` — từ chối document có field không khai báo trước
2. Đặt `"dynamic": false` — chấp nhận document nhưng không index field mới
3. Dùng `flattened` type cho object có nhiều key không dự đoán trước
4. Giới hạn `index.mapping.total_fields.limit` (mặc định 1,000)

### Q5: Khi nào nên dùng OpenSearch Serverless thay vì OpenSearch Service?

**Trả lời:**
- **Serverless phù hợp khi:** Workload không đều (spiky load), không muốn quản lý cluster, cần scale từ 0 nhanh, dev/test environment.
- **Managed Service phù hợp khi:** Workload ổn định, cần kiểm soát fine-grained (Tinh Tế) về instance type/shard, chi phí thấp hơn với cluster nhỏ chạy liên tục (~$120/tháng vs ~$700/tháng tối thiểu của Serverless).

---

## 🔗 Liên Kết Tiếp Theo

- [Thu Nạp Dữ Liệu — Kinesis, Logstash, Fluent Bit](./2-opensearch-ingestion.md)
- [Bảo Mật OpenSearch](./3-opensearch-security.md)
- [OpenSearch Module Overview](./README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
