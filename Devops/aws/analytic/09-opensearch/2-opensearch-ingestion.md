# OpenSearch Ingestion — Thu Nạp Dữ Liệu vào Amazon OpenSearch

> Dữ liệu đi vào OpenSearch qua nhiều con đường: Kinesis Data Firehose (Vòi Dữ Liệu Kinesis), Logstash (Bộ Thu Thập Log), Fluent Bit (Thu Thập Log Nhẹ), Lambda (Hàm Không Máy Chủ) và OpenSearch Ingestion Pipeline. Chọn đúng phương thức ingest (Thu Nạp) ảnh hưởng trực tiếp đến hiệu suất, chi phí và độ phức tạp vận hành.

---

## 📚 Mục Lục

1. [Tổng Quan Các Phương Thức Ingest](#1-tổng-quan-các-phương-thức-ingest)
2. [Kinesis Data Firehose → OpenSearch](#2-kinesis-data-firehose--opensearch)
3. [AWS Lambda → OpenSearch](#3-aws-lambda--opensearch)
4. [Logstash — Bộ Thu Thập Log](#4-logstash--bộ-thu-thập-log)
5. [Fluent Bit — Thu Thập Log Nhẹ](#5-fluent-bit--thu-thập-log-nhẹ)
6. [OpenSearch Ingestion (OSI)](#6-opensearch-ingestion-osi)
7. [Bulk API — Nhập Hàng Loạt](#7-bulk-api--nhập-hàng-loạt)
8. [Index Template — Mẫu Chỉ Mục](#8-index-template--mẫu-chỉ-mục)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Các Phương Thức Ingest

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Nguồn Dữ Liệu                                   │
│  Application Logs │ IoT Events │ Clickstream │ DB Changes │ Metrics  │
└────────┬──────────────────────────────────────────────────┬─────────┘
         │                                                   │
         ▼                                                   ▼
┌─────────────────────────────┐         ┌────────────────────────────┐
│    Real-time Streaming       │         │    Batch / Near-real-time  │
│                             │         │                             │
│  Kinesis Data Streams (KDS) │         │  AWS Glue ETL               │
│    → Kinesis Firehose (KDF) │         │  S3 + Lambda trigger        │
│    → Lambda → Bulk API      │         │  Direct Bulk API            │
│                             │         │                             │
│  OpenSearch Ingestion (OSI) │         │                             │
└─────────────────────────────┘         └────────────────────────────┘
         │                                         │
         └──────────────────┬──────────────────────┘
                            ▼
                ┌─────────────────────┐
                │  Amazon OpenSearch  │
                │  Domain / Cluster   │
                └─────────────────────┘
```

### So Sánh Nhanh

| Phương Thức | Loại | Độ Trễ | Phức Tạp | Phù Hợp Nhất |
|------------|------|--------|----------|-------------|
| **Kinesis Firehose** | Managed | ~60 giây | Thấp | Log delivery có buffering |
| **Lambda** | Serverless | Thời gian thực | Trung bình | Event-driven, biến đổi nhẹ |
| **Logstash** | Self-managed | Thời gian thực | Cao | Pipeline phức tạp, nhiều nguồn |
| **Fluent Bit** | Agent | Thời gian thực | Thấp | Container/K8s log collection |
| **OSI Pipeline** | Managed | Thời gian thực | Thấp | Managed Logstash thay thế |
| **Bulk API trực tiếp** | Direct | Tức thì | Cao | Migration, backfill |

---

## 2. Kinesis Data Firehose → OpenSearch

### 2.1 Luồng Dữ Liệu

```
Source                  Kinesis               OpenSearch
──────                  ───────               ──────────
Application ──PUT──▶  Data Firehose  ──▶  OpenSearch Domain
IoT Device  ──PUT──▶  (Delivery      ──▶  Index: logs-YYYY-MM-DD
Kinesis KDS ──▶        Stream)
                           │
                      (Buffering)        Backup (Optional)
                      size: 5 MB  ──▶        │
                      time: 60s              ▼
                                         S3 Bucket
                                     (backup + failed)
```

### 2.2 Cấu Hình Firehose → OpenSearch

```
Delivery Stream Settings:
├── Destination:          Amazon OpenSearch Service
├── Domain ARN:           arn:aws:es:ap-southeast-1:123456789:domain/my-domain
├── Index:                logs          (hoặc dynamic với ${timestamp:yyyy-MM-dd})
├── Index rotation:       NoRotation / OneHour / OneDay / OneWeek / OneMonth
├── Buffer size:          1–100 MB      (flush khi đạt ngưỡng)
├── Buffer interval:      60–900 giây   (flush khi timeout)
├── Retry duration:       0–7200 giây
└── S3 backup:            All / Failed  (lưu backup hoặc chỉ lưu khi lỗi)
```

### 2.3 Index Rotation — Xoay Vòng Chỉ Mục

| Tùy Chọn | Index Pattern | Phù Hợp Cho |
|---------|---------------|------------|
| `NoRotation` | `logs` (cố định) | Dataset nhỏ, không cần rotation |
| `OneHour` | `logs-2024-01-15-10` | Khối lượng log rất lớn |
| `OneDay` | `logs-2024-01-15` | Log analytics thông thường |
| `OneWeek` | `logs-2024-w03` | Dữ liệu trung bình |
| `OneMonth` | `logs-2024-01` | Dữ liệu ít hơn |

### 2.4 Data Transformation — Biến Đổi Dữ Liệu

Trước khi đến OpenSearch, Firehose có thể gọi Lambda để biến đổi dữ liệu:

```
Kinesis Firehose
    │
    │ (Invoke Lambda cho mỗi micro-batch)
    ▼
Lambda Function
    ├── Parse JSON / CSV / log format
    ├── Thêm trường: { "ingested_at": "2024-01-15T10:00:00Z" }
    ├── Lọc records không cần thiết
    └── Trả về: { "result": "Ok", "data": "<base64 encoded JSON>" }
    │
    ▼
OpenSearch Domain
```

```python
# Lambda transformation example (Python)
import base64
import json

def lambda_handler(event, context):
    output = []
    for record in event['records']:
        payload = json.loads(base64.b64decode(record['data']))

        # Thêm trường metadata
        payload['processed_at'] = '2024-01-15T10:00:00Z'
        payload['environment']  = 'production'

        output.append({
            'recordId': record['recordId'],
            'result':   'Ok',
            'data':     base64.b64encode(
                            json.dumps(payload).encode('utf-8')
                        ).decode('utf-8')
        })
    return {'records': output}
```

---

## 3. AWS Lambda → OpenSearch

### 3.1 Kiến Trúc Event-Driven

Lambda phù hợp khi cần xử lý real-time với logic biến đổi phức tạp hơn Firehose cho phép:

```
DynamoDB Streams ──▶ Lambda ──▶ OpenSearch (sync DB thay đổi vào search index)
S3 Event         ──▶ Lambda ──▶ OpenSearch (index tài liệu mới upload)
SNS / SQS        ──▶ Lambda ──▶ OpenSearch (index event từ message queue)
API Gateway      ──▶ Lambda ──▶ OpenSearch (user-triggered indexing)
```

### 3.2 Lambda → OpenSearch với opensearch-py

```python
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

def get_opensearch_client():
    region    = 'ap-southeast-1'
    service   = 'es'
    host      = 'my-domain.ap-southeast-1.es.amazonaws.com'
    credentials = boto3.Session().get_credentials()
    awsauth   = AWS4Auth(
        credentials.access_key,
        credentials.secret_key,
        region, service,
        session_token=credentials.token
    )
    return OpenSearch(
        hosts=[{'host': host, 'port': 443}],
        http_auth=awsauth,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection
    )

def lambda_handler(event, context):
    client = get_opensearch_client()
    for record in event['Records']:
        document = {
            'order_id':   record['dynamodb']['NewImage']['order_id']['S'],
            'status':     record['dynamodb']['NewImage']['status']['S'],
            'amount':     float(record['dynamodb']['NewImage']['amount']['N']),
            'updated_at': record['dynamodb']['NewImage']['updated_at']['S']
        }
        # Index hoặc update document
        client.index(
            index='orders',
            id=document['order_id'],
            body=document
        )
```

---

## 4. Logstash — Bộ Thu Thập Log

### 4.1 Logstash Là Gì?

**Logstash** là công cụ thu thập, xử lý và chuyển tiếp dữ liệu mã nguồn mở của Elastic. Hoạt động theo mô hình pipeline: Input → Filter → Output.

```
┌──────────────────────────────────────────────┐
│              Logstash Pipeline               │
│                                              │
│  ┌─────────┐   ┌──────────┐   ┌──────────┐  │
│  │  Input  │──▶│  Filter  │──▶│  Output  │  │
│  │ Plugin  │   │  Plugin  │   │  Plugin  │  │
│  └─────────┘   └──────────┘   └──────────┘  │
│                                              │
│  Input:   file, beats, kafka, jdbc, http    │
│  Filter:  grok, mutate, date, geoip, json   │
│  Output:  opensearch, s3, kafka, stdout     │
└──────────────────────────────────────────────┘
```

### 4.2 Cấu Hình Logstash → OpenSearch

```ruby
# /etc/logstash/conf.d/nginx-to-opensearch.conf

input {
  file {
    path  => "/var/log/nginx/access.log"
    start_position => "beginning"
    sincedb_path   => "/dev/null"
    type           => "nginx"
  }
}

filter {
  # Grok — parse nginx log format
  grok {
    match => {
      "message" => '%{IPORHOST:client_ip} - %{DATA:user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:request} HTTP/%{NUMBER:http_version}" %{NUMBER:response_code} %{NUMBER:bytes}'
    }
  }
  # Chuyển đổi response_code sang integer
  mutate {
    convert => { "response_code" => "integer" }
    convert => { "bytes" => "integer" }
  }
  # Parse timestamp
  date {
    match   => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target  => "@timestamp"
    remove_field => ["timestamp"]
  }
  # Thêm GeoIP location (Vị Trí Địa Lý) từ IP
  geoip {
    source => "client_ip"
  }
}

output {
  opensearch {
    hosts               => ["https://my-domain.ap-southeast-1.es.amazonaws.com:443"]
    index               => "nginx-logs-%{+YYYY.MM.dd}"
    auth_type           => { type => 'aws_iam' }
    region              => "ap-southeast-1"
    service_name        => "es"
  }
}
```

### 4.3 Grok Pattern — Mẫu Phân Tích Log

**Grok** là plugin filter mạnh nhất của Logstash — parse (Phân Tích Cú Pháp) log dạng text thành các trường có cấu trúc:

```
Log dạng thô:
  203.0.113.5 - - [15/Jan/2024:10:30:45 +0700] "GET /api/products HTTP/1.1" 200 1234

Sau khi qua Grok:
  client_ip:     "203.0.113.5"
  method:        "GET"
  request:       "/api/products"
  response_code: 200
  bytes:         1234
  @timestamp:    2024-01-15T03:30:45Z
```

---

## 5. Fluent Bit — Thu Thập Log Nhẹ

### 5.1 Fluent Bit Là Gì?

**Fluent Bit** là log processor (Bộ Xử Lý Log) và forwarder (Bộ Chuyển Tiếp) siêu nhẹ — tối ưu cho môi trường container và Kubernetes. Dùng ít hơn 450KB RAM (so với Logstash ~300MB).

```
Kubernetes Cluster
├── Pod App 1 ──▶ Fluent Bit (DaemonSet) ──▶ OpenSearch
├── Pod App 2 ──▶ Fluent Bit              ──▶ (index: k8s-logs)
└── Pod App N ──▶ (chạy trên mỗi Node)   ──▶
```

### 5.2 Cấu Hình Fluent Bit → OpenSearch

```ini
# fluent-bit.conf

[SERVICE]
    Flush         5
    Log_Level     info
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     50MB

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Merge_Log           On
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[OUTPUT]
    Name            opensearch
    Match           kube.*
    Host            my-domain.ap-southeast-1.es.amazonaws.com
    Port            443
    TLS             On
    AWS_Auth        On
    AWS_Region      ap-southeast-1
    Index           k8s-logs
    Suppress_Type_Name On
```

### 5.3 Fluent Bit vs Logstash — Khi Nào Dùng Cái Nào?

| Tiêu Chí | Fluent Bit | Logstash |
|----------|-----------|---------|
| **Bộ nhớ** | ~1 MB | ~300 MB |
| **Môi trường** | Container, IoT, embedded | Server, on-premises |
| **Plugin** | Ít hơn nhưng đủ dùng | Rất phong phú |
| **Xử lý phức tạp** | Hạn chế | Mạnh (Grok, Ruby filter) |
| **Tốc độ triển khai** | Nhanh | Cần cấu hình nhiều hơn |
| **Khuyến nghị** | EKS/ECS log collection | Complex log transformation |

---

## 6. OpenSearch Ingestion (OSI)

### 6.1 OSI Là Gì?

**Amazon OpenSearch Ingestion** (OSI) — ra mắt 2023 — là dịch vụ managed pipeline dựa trên **OpenSearch Data Prepper** (Bộ Tiền Xử Lý Dữ Liệu OpenSearch), thay thế cho việc tự vận hành Logstash.

```
Nguồn (Source)          OSI Pipeline            Đích (Sink)
──────────────          ────────────            ─────────
Kinesis Stream ──▶  ┌────────────────┐  ──▶  OpenSearch Domain
S3 bucket      ──▶  │  Processor     │  ──▶  OpenSearch Serverless
CloudWatch     ──▶  │  (Grok, filter,│
Direct HTTP    ──▶  │   transform)   │
               ──▶  └────────────────┘
```

### 6.2 Cấu Hình OSI Pipeline

```yaml
# OSI Pipeline Configuration (YAML)
version: "2"
log-pipeline:
  source:
    http:
      path: "/log/ingest"

  processor:
    - grok:
        match:
          message:
            - '%{IPORHOST:client_ip} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:path}" %{NUMBER:status}'
    - date:
        from_time_received: true
        destination: "@timestamp"

  sink:
    - opensearch:
        hosts: ["https://my-domain.ap-southeast-1.es.amazonaws.com"]
        index: "app-logs-%{yyyy.MM.dd}"
        aws:
          sts_role_arn: "arn:aws:iam::123456789:role/osi-pipeline-role"
          region: "ap-southeast-1"
```

### 6.3 OSI vs Logstash Self-managed

| Tiêu Chí | OSI | Logstash Self-managed |
|----------|-----|-----------------------|
| **Quản lý** | AWS quản lý hoàn toàn | Tự quản lý server |
| **Scaling** | Auto-scale | Thủ công |
| **Giá** | OCU-based | EC2 instance giờ |
| **Tính năng** | Data Prepper plugins | Logstash plugins (nhiều hơn) |
| **Phù hợp** | Workload chuẩn AWS | Pipeline phức tạp, custom |

---

## 7. Bulk API — Nhập Hàng Loạt

### 7.1 Bulk API Là Gì?

Bulk API cho phép gửi nhiều thao tác (index, create, update, delete) trong một HTTP request duy nhất — hiệu quả hơn nhiều so với gửi từng request.

```bash
POST /my-index/_bulk
{ "index": { "_id": "1" } }
{ "name": "Sản phẩm A", "price": 100000 }
{ "index": { "_id": "2" } }
{ "name": "Sản phẩm B", "price": 200000 }
{ "update": { "_id": "1" } }
{ "doc": { "price": 90000 } }
{ "delete": { "_id": "3" } }
```

### 7.2 Tối Ưu Bulk Indexing

```
Bulk size khuyến nghị:
├── 5–15 MB per request (không theo số documents)
├── Số workers = số primary shards × 1-2
└── Tắt refresh trong quá trình ingest lớn:
    PUT /my-index/_settings
    { "index": { "refresh_interval": "-1" } }
    → Sau khi xong: đặt lại "1s"
```

**Python bulk indexing example:**

```python
from opensearchpy import OpenSearch, helpers

client = get_opensearch_client()  # từ ví dụ trước

def generate_documents(data_list):
    for item in data_list:
        yield {
            "_index": "products",
            "_id":    item["id"],
            "_source": item
        }

# Bulk index với helpers.bulk — tự động chia thành batches
success, failed = helpers.bulk(
    client,
    generate_documents(my_data),
    chunk_size=500,            # documents per request
    request_timeout=60
)
print(f"Indexed: {success}, Failed: {failed}")
```

---

## 8. Index Template — Mẫu Chỉ Mục

### 8.1 Index Template Là Gì?

**Index Template** (Mẫu Chỉ Mục) định nghĩa settings và mappings được tự động áp dụng cho tất cả index khớp với một pattern — không cần tạo thủ công từng index.

```
Index Template: "logs-*"
├── Áp dụng cho: logs-2024-01, logs-2024-02, logs-app-nginx, ...
├── Settings: 3 shards, 1 replica, ILM policy
└── Mappings: @timestamp là date, message là text, level là keyword
```

### 8.2 Tạo Index Template

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards":   3,
      "number_of_replicas": 1,
      "refresh_interval":   "30s",
      "index.lifecycle.name": "log-lifecycle-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp":    { "type": "date" },
        "message":       { "type": "text" },
        "level":         { "type": "keyword" },
        "service":       { "type": "keyword" },
        "client_ip":     { "type": "ip" },
        "response_code": { "type": "short" },
        "duration_ms":   { "type": "float" }
      }
    }
  }
}
```

### 8.3 Data Stream — Luồng Dữ Liệu

**Data Stream** (Luồng Dữ Liệu) là abstraction (Lớp Trừu Tượng) trên nhiều time-series indices — ẩn việc rotation khỏi client:

```
Client ghi vào: "logs"          (data stream name — không phải index thật)
    │
    ▼
Thực tế ghi vào: .ds-logs-000001  (backing index hiện tại)
    │
    │ Khi rollover:
    ▼
.ds-logs-000002 (ghi tiếp)
.ds-logs-000001 (read-only, theo ISM → UltraWarm → Delete)
```

---

## 9. Câu Hỏi Phỏng Vấn

### Q1: Chọn Kinesis Firehose hay Lambda để ingest vào OpenSearch? Khi nào dùng cái nào?

**Trả lời:**
- **Kinesis Firehose** phù hợp khi: Cần managed delivery, chấp nhận ~60 giây buffering, cần S3 backup tự động, transformation đơn giản qua Lambda inline. Chi phí thấp, ít code.
- **Lambda trực tiếp** phù hợp khi: Cần latency thấp hơn (< 1 giây), transformation phức tạp (join với DynamoDB, gọi external API), trigger từ nhiều nguồn (DynamoDB Streams, S3 Events, API Gateway).

### Q2: Tại sao Fluent Bit được ưa dùng hơn Logstash trong môi trường Kubernetes?

**Trả lời:** Fluent Bit tiêu thụ khoảng 450KB RAM so với ~300MB của Logstash — quan trọng khi chạy dạng DaemonSet (Tập Nền) trên mỗi node. Fluent Bit cũng có native Kubernetes filter — tự động thêm metadata như pod name, namespace, container name vào log. Logstash phù hợp hơn khi cần transformation phức tạp hoặc pipeline nhiều bước.

### Q3: Bulk size bao nhiêu là tối ưu cho indexing?

**Trả lời:** Không có con số tuyệt đối — phụ thuộc vào document size và cluster capacity. Nguyên tắc: test với bulk size 5 MB → 10 MB → 15 MB và đo throughput. Nếu quá lớn → timeout và retry overhead. Nếu quá nhỏ → nhiều HTTP overhead. Thêm vào đó: giảm `refresh_interval` về `-1` trong quá trình bulk load lớn để tắt real-time indexing tạm thời.

### Q4: Index Template và Index Policy (ISM) phối hợp thế nào?

**Trả lời:** Index Template định nghĩa **cấu trúc** (mapping, settings) được áp dụng khi index mới được tạo. ISM Policy định nghĩa **vòng đời** (chuyển từ hot sang warm, sau đó xóa). Trong template, khai báo `index.lifecycle.name` để tự động gắn ISM policy cho mọi index mới khớp pattern — không cần gắn thủ công.

---

## 🔗 Liên Kết Tiếp Theo

- [OpenSearch Fundamentals — Nền Tảng](./1-opensearch-fundamentals.md)
- [Bảo Mật OpenSearch — Fine-grained Access Control](./3-opensearch-security.md)
- [OpenSearch Module Overview](./README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
