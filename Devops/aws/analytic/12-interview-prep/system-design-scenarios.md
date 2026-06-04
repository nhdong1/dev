# 🏗️ System Design Scenarios — Kịch Bản Thiết Kế Hệ Thống AWS Analytics

> 5 kịch bản thiết kế hệ thống thực tế thường xuất hiện trong phỏng vấn cấp Senior/Staff Engineer — kèm framework phân tích, kiến trúc tham khảo và điểm cần bảo vệ.

## 📚 Mục Lục

1. [Real-time Analytics Pipeline — Phân Tích Sự Kiện Thời Gian Thực](#kịch-bản-1-real-time-analytics-pipeline)
2. [Data Lake trên AWS — Hồ Dữ Liệu Toàn Diện](#kịch-bản-2-data-lake-trên-aws)
3. [BI Platform — Nền Tảng Business Intelligence](#kịch-bản-3-bi-platform)
4. [Log Analytics System — Hệ Thống Phân Tích Log](#kịch-bản-4-log-analytics-system)
5. [Multi-tenant Analytics — Phân Tích Đa Khách Hàng](#kịch-bản-5-multi-tenant-analytics)

---

## Framework Giải Quyết System Design

Trước khi bắt đầu bất kỳ kịch bản nào, áp dụng framework 4 bước:

```
Bước 1 — CLARIFY (Làm Rõ Yêu Cầu) [5 phút]:
  - Volume: bao nhiêu events/records per giây/ngày?
  - Latency: real-time (< 1s), near-real-time (< 1 phút), batch (> giờ)?
  - Retention: lưu data bao lâu?
  - Users: ai query? bao nhiêu concurrent users?
  - Budget: có constraint không?
  - Availability: SLA là bao nhiêu? (99.9% = 8.7h downtime/năm)

Bước 2 — ESTIMATE (Ước Tính) [3 phút]:
  - Data volume: X events × Y bytes = Z GB/ngày
  - Throughput: peak vs average
  - Storage: 1 năm × Z GB = ? TB

Bước 3 — DESIGN (Thiết Kế) [15 phút]:
  - Vẽ high-level diagram (data flow, components)
  - Giải thích từng component chọn WHY
  - Đề cập trade-offs chủ động

Bước 4 — DEEP DIVE (Đào Sâu) [7 phút]:
  - Failure scenarios (khi nào fail, recover thế nào?)
  - Scaling bottlenecks (component nào bị bottleneck khi scale?)
  - Cost estimation (roughly)
  - Monitoring strategy
```

---

## Kịch Bản 1: Real-time Analytics Pipeline

### Đề Bài

> Thiết kế hệ thống analytics cho ứng dụng streaming media (như Netflix) — theo dõi viewing events (sự kiện xem) theo thời gian thực để cá nhân hóa đề xuất nội dung và phát hiện kỹ thuật số (content fraud detection — phát hiện gian lận nội dung).

### Bước 1 — Clarify

```
Bạn nên hỏi:
- Volume: 50 triệu users × 3 events/phút peak = ~2.5M events/phút = ~42,000 events/giây
- Event types: play, pause, seek, stop, buffering, error
- Mỗi event: ~500 bytes (user_id, content_id, timestamp, position, device)
- Latency yêu cầu:
  + Recommendation refresh: < 30 giây
  + Fraud detection alert: < 5 giây
  + Historical analytics (báo cáo): batch, vài giờ ok
- Retention: 2 năm historical, real-time data 7 ngày
- Users của analytics: data science team (~50), business analysts (~200), automated fraud system
```

### Bước 2 — Estimate

```
Throughput: 42,000 events/giây × 500 bytes = 21 MB/giây = 1.26 GB/phút
Daily volume: 21 MB/s × 86,400s = ~1.8 TB/ngày raw
Yearly: 1.8 TB × 365 = ~660 TB/năm (compressed Parquet ~150 TB)

Kinesis Shards needed:
  Write: ceil(21 MB/s ÷ 1 MB/s per Shard) = 21 Shards
  Với KPL aggregation (10x): ~3 Shards sufficient
```

### Bước 3 — Kiến Trúc

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INGESTION LAYER                              │
│  Mobile/Web Apps                                                    │
│      ↓ (HTTPS, batch events mỗi 5 giây)                            │
│  [API Gateway + Lambda]                                             │
│      ↓ (publish to KDS)                                             │
│  [Kinesis Data Streams] — 3 Shards (KPL aggregation)               │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
          ┌───────────┴──────────────┐
          ↓                          ↓
┌─────────────────┐        ┌──────────────────────┐
│  REAL-TIME PATH │        │   BATCH/STORAGE PATH │
│                 │        │                      │
│ [KDA — Flink]   │        │ [Kinesis Firehose]   │
│  Processing:    │        │  Buffer 5 phút       │
│  - Sessionize   │        │  Snappy compress     │
│  - Aggregate    │        │  Convert to Parquet  │
│  - Fraud score  │        │       ↓              │
│      ↓          │        │  [S3 Bronze Layer]   │
│  [ElastiCache   │        │  raw/year=../        │
│   Redis]        │        │       ↓ Glue ETL     │
│  - View counts  │        │  [S3 Silver Layer]   │
│  - User session │        │  cleaned/partitioned │
│  - Fraud flags  │        │       ↓ Glue ETL     │
│      ↓          │        │  [S3 Gold Layer]     │
│  [SNS/Lambda]   │        │  daily_active_users  │
│  - Fraud alert  │        │  content_performance │
│  - Email/Slack  │        │       ↓              │
└─────────────────┘        │  [Athena] [Redshift] │
                           │  [QuickSight]        │
                           └──────────────────────┘
```

**Giải Thích Từng Thành Phần:**

**API Gateway + Lambda (Ingestion):**
- Lambda batch events trước khi push vào KDS → giảm API calls
- Validate schema ngay tại đây — bad events → Dead Letter Queue (DLQ — Hàng Đợi Thư Chết)

**Kinesis Data Streams với KPL:**
- KPL — Kinesis Producer Library (Thư Viện Producer Kinesis) — aggregates nhiều records thành 1 PutRecord call
- Giảm 10x số Shards cần thiết → giảm 10x chi phí streaming

**Kinesis Data Analytics — Apache Flink:**
- Session window (Cửa Sổ Phiên): group events cùng user_id trong 30 phút inactivity gap
- Tumbling window 10 giây: aggregate viewership counts
- CEP (Complex Event Processing — Xử Lý Sự Kiện Phức Tạp): detect fraud patterns (xem cùng content từ 3+ IPs khác nhau trong 1 giờ)

**Redis cho Real-time State:**
- Sorted Set: top 10 trending content (cập nhật mỗi 10 giây)
- Hash: user session state (last_watched, watch_progress)
- TTL 24h: tự cleanup

**Medallion Architecture trên S3:**
- Bronze: raw JSON events
- Silver: deduplicated, sessionized, standardized timestamps
- Gold: daily_content_metrics, user_watch_patterns

### Bước 4 — Deep Dive

**Failure Scenarios (Kịch Bản Thất Bại):**

```
1. Kinesis Shard full (throttling):
   → KPL có built-in retry với exponential backoff
   → CloudWatch Alarm → Lambda auto-scale Shard count
   → Tradeoff: On-Demand mode giảm operational burden nhưng tốn hơn

2. Flink job crash:
   → KDA tự restart từ last checkpoint (Điểm Kiểm Tra)
   → Checkpoint mỗi 60 giây → max 60 giây data loss risk
   → Kinesis retention 7 ngày → có thể replay từ đầu

3. Redis down:
   → Real-time dashboard degraded (fallback sang static cache)
   → Redis ElastiCache Multi-AZ với automatic failover < 30 giây

4. S3 data corruption:
   → S3 Versioning (Phiên Bản Hóa) bật → rollback
   → Iceberg time travel → query tại point-in-time trước khi corrupt
```

**Cost Estimate:**
```
Kinesis KDS (3 Shards): 3 × $0.015 × 24h × 30 = $32/tháng
KDA Flink (4 KPU): 4 × $0.11 × 24h × 30 = $317/tháng
S3 (150 TB/năm): 150 TB × $0.023/GB = ~$3,500/tháng
Firehose: 1.8 TB/ngày × $0.029/GB = ~$1,566/tháng
→ Ước tính: ~$5,500/tháng cho streaming layer
```

**Điểm Cần Bảo Vệ:**
> "Tại sao KDS thay vì MSK?" → KDS tích hợp native với Firehose và Lambda, đơn giản hơn với team AWS-only. Nếu cần Kafka Streams ecosystem hoặc multi-cloud, chọn MSK.

---

## Kịch Bản 2: Data Lake trên AWS

### Đề Bài

> Thiết kế Data Lake (Hồ Dữ Liệu) toàn diện cho công ty fintech (tài chính công nghệ) — thu nhập dữ liệu từ 15 nguồn (databases, APIs, files), centralize governance (quản trị tập trung) và serve data cho data scientists và BI teams.

### Bước 1 — Clarify

```
Sources:
  - 5 RDBMS (PostgreSQL, MySQL) — 500M rows tổng
  - 3 REST APIs (payment providers) — webhook events
  - 7 file sources (CSV, JSON từ partners)

Volume: 50M records/ngày tổng, peak 1M records/giờ
Users: 30 data scientists (Python/SQL), 50 BI analysts (Tableau/QuickSight)
Compliance: GDPR (bảo vệ dữ liệu EU), PCI-DSS (bảo mật thẻ tín dụng)
  → Column-level encryption, row-level access control theo role
Retention: 7 năm (regulatory requirement — yêu cầu pháp lý)
```

### Bước 2 — Estimate

```
50M records × 1 KB average = 50 GB/ngày raw
Compressed Parquet: 50 GB × 0.1 = 5 GB/ngày (10:1 compression ratio)
7 năm: 5 GB × 365 × 7 = 12.8 TB Silver layer
Gold layer (aggregates): ~500 GB (much smaller)

S3 cost 7 năm:
  Silver (12.8 TB): frequent access first 30 ngày → S3 Standard
  Older data → S3 Infrequent Access → S3 Glacier for 5-7 năm old
```

### Bước 3 — Kiến Trúc

```
DATA SOURCES (15 nguồn)
├── RDBMS (5 databases)
│   └── [AWS DMS — Database Migration Service] → [S3 Bronze]
│       Full load + CDC (Change Data Capture — Thu Thập Thay Đổi)
│
├── REST APIs (3 webhooks)
│   └── [API Gateway → Lambda] → [Kinesis Firehose] → [S3 Bronze]
│       Real-time ingestion
│
└── Files (7 partners)
    └── [S3 Partner buckets → S3 Transfer Acceleration]
        → [EventBridge trigger] → [Lambda] → [S3 Bronze]

S3 DATA LAKE (Medallion Architecture)
├── Bronze Zone (Vùng Thô):
│   s3://datalake/bronze/{source}/{year}/{month}/{day}/
│   Format: JSON/CSV as-is, với thêm _ingested_at, _source
│   Encryption: SSE-KMS per data source
│   Retention: 7 năm, lifecycle → Glacier sau 90 ngày
│
├── Silver Zone (Vùng Sạch):
│   s3://datalake/silver/{domain}/{table}/year=../month=../
│   Format: Parquet + Snappy, partitioned by date
│   Processing: Glue ETL Jobs (chạy hourly)
│   Quality: Glue Data Quality rules
│   PII handling: Tokenize (thay thế PII bằng token) sensitive fields
│
└── Gold Zone (Vùng Tổng Hợp):
    s3://datalake/gold/{domain}/{metric}/
    Format: Parquet, denormalized
    Examples: daily_transaction_summary, user_risk_scores, merchant_analytics

GOVERNANCE LAYER
├── AWS Glue Data Catalog:
│   - Tất cả tables được register
│   - Schema versioning
│   - Business metadata (owner, SLA, PII tags)
│
├── AWS Lake Formation:
│   - Column-level security: mask PAN (card numbers) cho non-PCI roles
│   - Row-level security: data scientists chỉ thấy data domain của mình
│   - LF-Tags (Lake Formation Tags): classify tables theo sensitivity
│   - Data sharing cross-account: partner accounts
│
└── AWS Macie:
    - Scan S3 tự động detect PII mới
    - Alert khi tìm thấy unprotected sensitive data

SERVING LAYER
├── Amazon Athena: Data scientists, ad-hoc SQL
│   - Workgroups per team với budget limits
│   - Query Result Reuse bật
│
├── Amazon Redshift Spectrum:
│   - BI teams với Tableau/QuickSight
│   - Complex OLAP trên Gold layer
│
└── SageMaker: ML model training trên Silver data
```

### Bước 4 — Deep Dive

**Governance Quyết Định:**

```
PCI-DSS Compliance (Tuân Thủ PCI-DSS) — thẻ tín dụng:
  - Card numbers (PAN): KHÔNG bao giờ lưu raw
  - Tokenize ngay tại ingestion layer (Lambda)
  - Token → Payment provider vault
  - Lake Formation column-level masking cho analysts thấy last4 only

GDPR Right to Erasure (Quyền Xóa Dữ Liệu):
  - Problem: Parquet files khó delete individual records
  - Solution: Apache Iceberg (hoặc Delta Lake)
    → MERGE INTO để soft-delete: thêm is_deleted=true
    → Iceberg VACUUM để physically xóa sau 90 ngày
    → Cryptographic erasure: delete KMS key của user → data unreadable
```

**Data Quality Pipeline:**

```
Glue Data Quality Rules:
  - Completeness: customer_id NOT NULL > 99%
  - Uniqueness: transaction_id unique = 100%
  - Referential: merchant_id exists in merchants table
  - Range: amount BETWEEN 0 AND 1,000,000

Khi fail quality check:
  → Record ghi vào Quarantine zone (Vùng Cách Ly)
  → Alert qua SNS → data steward review
  → Không promote lên Silver cho đến khi fix
```

**Điểm Cần Bảo Vệ:**
> "Tại sao không dùng Redshift làm trung tâm thay vì S3 Data Lake?" → Với 15 nguồn heterogeneous (dị biệt), schema diversity lớn. S3 + Glue linh hoạt hơn. Redshift tốt cho analytics layer nhưng không phải raw storage layer.

---

## Kịch Bản 3: BI Platform

### Đề Bài

> Thiết kế BI Platform (Nền Tảng Business Intelligence) phục vụ 500 business analysts, với self-service analytics, embedded dashboards trong SaaS product, và thời gian refresh dữ liệu < 4 giờ.

### Bước 1 — Clarify

```
Users:
  - 500 internal analysts: ad-hoc query + dashboards
  - 10,000 external customers xem embedded dashboards (SaaS)
Concurrency: peak 100 internal + 1,000 external concurrent sessions
Data freshness: < 4 giờ (acceptable lag)
Data sources: Redshift (operational), S3 Data Lake, 3rd party SaaS APIs
Dashboard types: operational (refresh mỗi giờ), strategic (refresh hàng ngày)
```

### Bước 2 — Kiến Trúc

```
DATA FRESHNESS PIPELINE

[Sources]
├── Redshift (operational data) ──→ [Redshift RA3 Node]
│                                    Unload to S3 mỗi 4 giờ
│                                    ↓
└── S3 Data Lake Gold ─────────────→ [Glue ETL Job]
                                      Aggregate + denormalize
                                      ↓
                                   [S3 BI Layer]
                                   bi/year=../month=../
                                   Format: Parquet, highly optimized

BI SERVING LAYER

Internal Analysts (500 users):
  [Amazon Redshift Serverless]
    - Load BI Layer data với COPY command
    - Hoặc query S3 qua Redshift Spectrum
    - WLM: 3 queues (critical, standard, bulk)
    ↓
  [Amazon QuickSight Enterprise]
    - SPICE: cache data cho < 1 giây response
    - Refresh SPICE mỗi 4 giờ (khớp với data freshness SLA)
    - Row-level security: analyst chỉ thấy region của mình

External Customers (10,000 users — Embedded Analytics):
  [QuickSight Embedded Q&A + Dashboards]
    - Registered user embedding (mỗi customer có identity riêng)
    - Row-level security: customer chỉ thấy data của tenant mình
    - Custom domains: analytics.customerdomain.com
    - White-label styling (logo, màu sắc riêng)
    ↓
  [Multi-tenancy Implementation]:
    QuickSight User ↔ Redshift RLS (Row-Level Security)
    - Mỗi customer_id map to RLS policy
    - Auto-provisioning qua Lambda khi new customer signup

SELF-SERVICE ANALYTICS

For SQL-savvy analysts:
  [Amazon Athena]
  - Direct S3 query
  - Workgroup with $1,000/month budget limit per team
  - Named query templates (mẫu query sẵn)

For business users (không biết SQL):
  [QuickSight Q — NLQ (Natural Language Query)]
  - Hỏi bằng tiếng Anh: "Show revenue by region last quarter"
  - QuickSight Q interpret → tự generate visualization

GOVERNANCE

[Catalog]
  - Glue Catalog: technical metadata
  - QuickSight Dataset descriptions: business metadata

[Access Control]
  - QuickSight Groups → IAM Roles → Redshift roles
  - Column-level masking trong Redshift: analysts thấy masked PII
  - Audit log: CloudTrail + QuickSight access logs → S3 → Athena analysis
```

### Bước 4 — Deep Dive

**SPICE (Super-fast Parallel In-memory Calculation Engine) Strategy:**

```
Quyết Định: Dùng SPICE hay Direct Query?

SPICE (In-memory cache):
  ✅ Sub-second response cho dashboards
  ✅ Không tốn Redshift/Athena compute khi nhiều users query cùng lúc
  ✅ Tốt cho dashboards có lịch refresh cố định
  ❌ Giới hạn 25 GB per dataset (standard), 1 TB (enterprise)
  ❌ Data có thể stale giữa các lần refresh

Direct Query (kết nối thẳng Redshift/Athena):
  ✅ Real-time data (sub-minute freshness)
  ✅ Không giới hạn data size
  ❌ Chậm hơn (phụ thuộc Redshift/Athena performance)
  ❌ Tốn compute mỗi khi dashboard load

Strategy:
  - Executive dashboards (< 1 GB, refresh daily) → SPICE
  - Operational dashboards (real-time needed) → Direct Query + Redshift WLM priority
  - Ad-hoc exploration → Athena Direct Query với workgroup budget
```

**Embedded Analytics — Multi-tenancy:**

```python
# Lambda function: generate QuickSight embed URL per customer
import boto3

def get_embed_url(customer_id: str, dashboard_id: str) -> str:
    quicksight = boto3.client('quicksight')
    
    # Provision QuickSight user nếu chưa có
    user_arn = provision_quicksight_user(customer_id)
    
    # Register user vào Row-Level Security tag group
    quicksight.create_group_membership(
        GroupName=f"tenant_{customer_id}",
        MemberName=customer_id,
        AwsAccountId=ACCOUNT_ID,
        Namespace='default'
    )
    
    # Generate embed URL với 1 giờ expiry
    response = quicksight.generate_embed_url_for_registered_user(
        AwsAccountId=ACCOUNT_ID,
        SessionLifetimeInMinutes=60,
        UserArn=user_arn,
        ExperienceConfiguration={
            'Dashboard': {'InitialDashboardId': dashboard_id}
        }
    )
    return response['EmbedUrl']
```

**Điểm Cần Bảo Vệ:**
> "Tại sao QuickSight thay vì Tableau/Power BI?" → QuickSight embedded không cần client-side license per user, pay-per-session model rẻ hơn cho 10,000 external users. Native AWS → dễ integrate với Lake Formation security.

---

## Kịch Bản 4: Log Analytics System

### Đề Bài

> Thiết kế hệ thống phân tích log cho platform 200 microservices — thu nhập 500K log lines/giây, lưu trữ 90 ngày, hỗ trợ full-text search (tìm kiếm toàn văn bản) và alerting (cảnh báo) với latency < 30 giây từ log xuất hiện đến alert được gửi.

### Bước 1 — Clarify

```
Volume: 500K lines/giây × 200 bytes average = 100 MB/giây = 8.6 TB/ngày
Retention: 90 ngày hot (searchable), 1 năm cold (archive)
Use cases:
  - Developer debug (truy vết lỗi production) — ad-hoc search
  - Ops monitoring — pre-built dashboards, anomaly detection
  - Security audit — regex search trên access patterns
  - Cost analytics — parse và aggregate billing-related logs
Alert requirements: lỗi ERROR rate > 1% trong 30 giây → alert trong < 30 giây
```

### Bước 2 — Kiến Trúc

```
LOG SOURCES (200 microservices)
├── EC2/ECS: [CloudWatch Agent / Fluent Bit sidecar]
│   → Fluent Bit buffer locally, batch push mỗi 5 giây
│
└── Lambda: CloudWatch Logs tự động

INGESTION LAYER
├── [Kinesis Data Streams] — 100 Shards (100 MB/s write)
│   Hoặc: [Amazon Data Firehose] nếu không cần custom processing
│
└── [Amazon Data Firehose]
    - Transform: parse structured fields với Lambda
    - Compress: Gzip
    - Deliver to: S3 + OpenSearch

REAL-TIME SEARCH & ALERTING
[Amazon OpenSearch Service]
├── Cluster: 3 Master nodes (m6g.large) + 6 Data nodes (r6g.2xlarge)
│   → r6g cho RAM-optimized (logs cần RAM để index)
├── Index strategy:
│   - Daily index rotation: logs-2024-01-15
│   - Hot-warm-cold tiers:
│     Hot (SSD): 7 ngày → full search speed
│     Warm (HDD): 8-30 ngày → slower search, cheaper
│     Cold (S3 via UltraWarm): 31-90 ngày → query khi cần
├── OpenSearch Ingestion (OSI — Pipeline):
│   Kinesis → OSI → OpenSearch
│   + Parse JSON, extract fields
│   + Enrich: thêm service_name, environment tags
│
└── OpenSearch Dashboards:
    - Dev dashboards: error rates per service, p99 latency
    - Ops: infra health, capacity trends
    - Security: failed auth attempts, unusual access patterns

ALERTING PIPELINE
[OpenSearch Alerting Plugin]
├── Monitor: Check error_rate > 1% mỗi 30 giây
├── Trigger: Gửi SNS notification
└── Destination: PagerDuty / Slack / Email

LONG-TERM STORAGE (90 ngày → 1 năm)
[S3 Archive]
├── s3://logs-archive/{year}/{month}/{day}/{hour}/
├── Format: Gzip JSON (dùng Athena khi cần query historical)
├── Athena table trên S3 logs:
│   - Partition by year/month/day/hour
│   - SerDe (Serializer/Deserializer): Regex SerDe để parse log format
└── Lifecycle: S3 → Glacier Deep Archive sau 90 ngày
```

### Bước 4 — Deep Dive

**OpenSearch Shard Strategy:**

```
Index pattern: logs-YYYY-MM-DD
Data per day: 8.6 TB → compressed ~2.5 TB per index
Recommended shard size: 10-50 GB per shard

Number of primary shards = ceil(2,500 GB ÷ 30 GB per shard) = 84 shards
Round to next power of 2 for flexibility: 96 shards per index

Với 6 data nodes: 96 ÷ 6 = 16 primary shards per node
+ 1 replica: tổng 192 shards → 32 per node

Kiểm tra: 32 shards × 30 GB = 960 GB per node
r6g.2xlarge: 64 GB RAM, 1 TB SSD → OK
```

**Hot-Warm-Cold Tiering Cost:**

```
Hot (7 ngày, SSD): 7 × 2.5 TB = 17.5 TB
  r6g.2xlarge storage: $0.135/GB → $2,362/tháng

Warm (23 ngày, HDD): 23 × 2.5 TB = 57.5 TB
  UltraWarm (r6gd): $0.024/GB → $1,380/tháng

Cold S3 (60 ngày): 60 × 2.5 TB = 150 TB
  S3 + UltraWarm Cold: $0.007/GB → $1,050/tháng

Total OpenSearch: ~$4,792/tháng
vs all-hot: $12,150/tháng → tiết kiệm 60%
```

**Điểm Cần Bảo Vệ:**
> "Tại sao OpenSearch thay vì Athena cho log search?" → OpenSearch có inverted index (Chỉ Mục Đảo Ngược) — full-text search milliseconds. Athena scan Parquet — phù hợp cho structured analytics nhưng không cho free-text regex search trong vài GB logs. Kết hợp: OpenSearch cho hot search, Athena cho cold historical analysis.

---

## Kịch Bản 5: Multi-tenant Analytics

### Đề Bài

> Thiết kế Multi-tenant Analytics Platform (Nền Tảng Phân Tích Đa Khách Hàng) cho SaaS B2B — 500 enterprise customers, mỗi customer có analytics riêng, data hoàn toàn isolated, với option white-label (thương hiệu riêng) và bring-your-own-BI (BYOB — khách tự mang BI tool).

### Bước 1 — Clarify

```
Tenants: 500 enterprise customers, kích thước từ startup (1 GB) đến enterprise (10 TB)
Isolation: Data phải hoàn toàn isolated (không tenant nào thấy data tenant khác)
Compliance: SOC2, ISO 27001 → audit trail, encryption per tenant
BYOB: Một số customers muốn kết nối Tableau, Power BI trực tiếp
Pricing: Freemium (free với limit) → Enterprise (unlimited)
Operations: 2 người DevOps quản lý toàn bộ — phải tự động hóa cao
```

### Bước 2 — Kiến Trúc

```
TENANT ISOLATION STRATEGY — Chọn 1 trong 3 models:

Model A — Silo (Hoàn Toàn Tách Biệt):
  Mỗi tenant có AWS account riêng
  ✅ Isolation tốt nhất, dễ billing
  ❌ Expensive, khó manage 500 accounts
  → Dùng cho: Enterprise tier (top 10% customers)

Model B — Pool (Chia Sẻ Infrastructure):
  Tất cả tenants trên cùng infrastructure, row-level isolation
  ✅ Chi phí thấp nhất
  ❌ Risk: một tenant bug ảnh hưởng tenant khác (noisy neighbor)
  → Dùng cho: Freemium và SMB tier

Model C — Bridge (Hybrid — Lựa Chọn Của Chúng Ta):
  - Shared infrastructure với logical isolation
  - S3 prefix per tenant + Lake Formation per-tenant permissions
  - Redshift Serverless namespaces per tenant group
  → Dùng cho: Toàn bộ (tùy chỉnh level tùy tier)

KIẾN TRÚC TRIỂN KHAI

INGESTION (per tenant — isolated)
[Tenant App] → [API Gateway] → [Lambda]
  - Lambda check tenant_id từ JWT token
  - Validate data schema
  - Write to: s3://main-datalake/tenants/{tenant_id}/raw/
  - Publish event: SQS queue per tenant tier

PROCESSING
[Glue ETL per tenant] ← triggered by EventBridge
  - Medallion: Bronze → Silver → Gold per tenant prefix
  - Schedule: Enterprise: every 15m | SMB: hourly | Free: daily

ISOLATION LAYER — AWS Lake Formation
  - S3 bucket: 1 bucket, prefix per tenant
  - Lake Formation:
    - Glue Database per tenant: "tenant_{id}_db"
    - LF-Tags: tenant_id={id}
    - IAM role per tenant: only access their prefix + database

SERVING LAYER — 3 Options

Option 1: QuickSight (Default)
  - QuickSight namespace per tenant (Enterprise)
  - Shared namespace với RLS (Free/SMB)
  - Embedded URL generated by Lambda per user session
  - Custom domain: {tenant}.analytics.yourproduct.com

Option 2: BYOB — Redshift Data Sharing
  [Redshift Serverless Producer]
    Namespace per tenant group
  → [Redshift Data Share]
    Share specific schema to tenant's Redshift account
  ← [Tenant's own Redshift / Tableau]
    Query via datashare — data stays in your account

Option 3: BYOB — Athena Cross-account
  [Lake Formation cross-account grant]
    → Tenant's AWS account
    → Tenant runs Athena on their account
    → Data served from your S3 (charged back to them)

TENANT PROVISIONING AUTOMATION
[New Tenant Signup] → [EventBridge] → [Step Functions]
  Step 1: Create S3 prefix policy (prefix per tenant_id)
  Step 2: Create Glue databases (bronze/silver/gold per tenant)
  Step 3: Register Lake Formation permissions
  Step 4: Create IAM role + policy
  Step 5: Provision QuickSight namespace (Enterprise) hoặc add to shared (Free)
  Step 6: Create Glue ETL trigger schedule
  Step 7: Send welcome email với connection details
  Total: < 5 phút, zero human intervention
```

### Bước 4 — Deep Dive

**Cost Allocation (Phân Bổ Chi Phí) Per Tenant:**

```python
# Lambda: Track S3 operations per tenant với S3 request metrics
# Tag tất cả resources với tenant_id

# Cấu trúc cost allocation:
S3 Storage: s3:ListBucketByPrefix per tenant → đo storage
Glue Jobs: Tag JobRun với tenant_id → CloudWatch custom metric
Athena: Tag workgroup per tenant → CloudWatch per-workgroup metrics

# Billing dashboard:
AWS Cost Explorer + custom tags → API → Billing Report per tenant
→ Tính tiền đúng theo usage → Fair billing cho enterprise contracts
```

**Noisy Neighbor Prevention (Ngăn Hàng Xóm Gây Nhiễu):**

```
Glue Jobs:
  - Concurrent job limit per tenant: max 2 concurrent jobs
  - DPU cap per tenant per month
  - Priority queuing: Enterprise > SMB > Free

Athena:
  - Workgroup per tier với budget
  - Per-query DML data scanned limit

S3:
  - S3 request rate limit per prefix: không có native limit
  - Workaround: CloudWatch metric + Lambda throttle API access

Redshift Serverless:
  - Separate namespace per enterprise tenant
  - Max RPU setting per namespace → cost cap
```

**Điểm Cần Bảo Vệ:**
> "Tại sao không mỗi tenant 1 S3 bucket?" → Với 500 tenants, quản lý 500 buckets là nightmare. AWS soft limit là 100 buckets per account (có thể request tăng). Prefix + Lake Formation isolation cho phép scale đến hàng nghìn tenants trên 1 bucket một cách an toàn và dễ quản lý.

---

## 📋 Tổng Kết Framework

### Checklist System Design Interview

**Khi Bắt Đầu:**
- [ ] Hỏi volume (events/s, GB/day)
- [ ] Hỏi latency requirement (millisecond, second, minute, hour)
- [ ] Hỏi users (ai query, bao nhiêu concurrent)
- [ ] Hỏi retention (short-term vs long-term)
- [ ] Hỏi compliance (GDPR, PCI-DSS, HIPAA)

**Khi Thiết Kế:**
- [ ] Vẽ data flow từ source đến consumer
- [ ] Giải thích tại sao chọn mỗi dịch vụ
- [ ] Đề cập trade-offs chủ động
- [ ] Tính cost estimate gần đúng

**Khi Kết Thúc:**
- [ ] Nêu failure scenarios và recovery
- [ ] Nêu scaling bottlenecks
- [ ] Nêu monitoring strategy
- [ ] Hỏi: "Bạn có muốn tôi đào sâu vào phần nào không?"

### Common Trade-offs Cần Nhớ

| Quyết Định | Option A | Option B | Chọn Theo |
| ---------- | -------- | -------- | --------- |
| Streaming | Kinesis | MSK/Kafka | AWS-only vs Kafka ecosystem |
| Query | Athena | Redshift | Ad-hoc vs OLAP/BI |
| ETL | Glue | EMR | Serverless ease vs control |
| Search | OpenSearch | Athena | Full-text vs structured |
| BI | QuickSight | Tableau/PowerBI | AWS-native vs enterprise BI |
| Architecture | Lambda Arch | Kappa Arch | Accuracy vs simplicity |
| Storage | S3 only | S3 + Redshift | Flexibility vs performance |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Module:** 12 — Interview Prep
**Trạng Thái:** ✅ Hoàn thành — 5 kịch bản system design
