# 🔍 Amazon OpenSearch Service — Search & Log Analytics (Tìm Kiếm & Phân Tích Log)

> Amazon OpenSearch Service là dịch vụ tìm kiếm và phân tích được quản lý hoàn toàn bởi AWS, dựa trên OpenSearch (nhánh mã nguồn mở tách ra từ Elasticsearch). Hỗ trợ full-text search (Tìm Kiếm Toàn Văn), log analytics (Phân Tích Log), APM (Application Performance Monitoring — Giám Sát Hiệu Suất Ứng Dụng) và real-time dashboards (Bảng Điều Khiển Thời Gian Thực) thông qua OpenSearch Dashboards.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Trạng Thái |
|------|--------|------------|
| [1-opensearch-fundamentals.md](./1-opensearch-fundamentals.md) | Clusters, Indices, Shards, Replicas — Nền Tảng | ✅ |
| [2-opensearch-ingestion.md](./2-opensearch-ingestion.md) | Thu Nạp Dữ Liệu — Kinesis, Logstash, Fluent Bit | ✅ |
| [3-opensearch-security.md](./3-opensearch-security.md) | Fine-grained Access Control, Encryption — Bảo Mật | ✅ |

---

## 🎯 OpenSearch Là Gì?

**Amazon OpenSearch Service** (trước đây là Amazon Elasticsearch Service) là dịch vụ được quản lý giúp triển khai, vận hành và mở rộng OpenSearch cluster mà không cần quản lý server. OpenSearch được xây dựng dựa trên Apache Lucene — search engine (Công Cụ Tìm Kiếm) mã nguồn mở nổi tiếng nhất.

### Đặc Điểm Nổi Bật

| Đặc Điểm | Chi Tiết |
|----------|----------|
| **Full-text Search** | Tìm kiếm toàn văn bản với relevance scoring (Điểm Liên Quan) |
| **Real-time Indexing** | Lập chỉ mục dữ liệu trong vài giây — truy vấn ngay |
| **Log Analytics** | Phân tích log từ ứng dụng, hệ thống, network |
| **Managed Service** | AWS quản lý patching, backup, scaling, HA |
| **OpenSearch Dashboards** | Giao diện trực quan hóa tích hợp sẵn (tương tự Kibana) |
| **Multi-AZ** | Triển khai đa vùng khả dụng cho high availability |

---

## 🏗️ Kiến Trúc Tổng Quan

```
                    ┌──────────────────────────────────────────────┐
                    │         Amazon OpenSearch Domain              │
                    │                                               │
 Nguồn Dữ Liệu     │  ┌──────────┐   ┌──────────┐  ┌──────────┐  │
 ─────────────      │  │  Node 1  │   │  Node 2  │  │  Node 3  │  │
 Kinesis Firehose ──┤  │ (Master) │   │  (Data)  │  │  (Data)  │  │
 Logstash        ──▶│  └──────────┘   └──────────┘  └──────────┘  │
 Fluent Bit      ──▶│       │                                       │
 Lambda          ──▶│  ┌────▼──────────────────────────────────┐   │
 Direct API      ──▶│  │         Indices (Chỉ Mục)             │   │
                    │  │  ┌──────────┐   ┌──────────────────┐  │   │
                    │  │  │ Primary  │   │   Replica Shards  │  │   │
                    │  │  │  Shards  │   │   (Bản Sao)      │  │   │
                    │  │  └──────────┘   └──────────────────┘  │   │
                    │  └───────────────────────────────────────┘   │
                    │                                               │
                    │  ┌────────────────────────────────────────┐  │
                    │  │     OpenSearch Dashboards (Kibana)      │  │
                    │  │  Visualize │ Discover │ Dev Tools       │  │
                    │  └────────────────────────────────────────┘  │
                    └──────────────────────────────────────────────┘
```

---

## 🗂️ Use Cases — Trường Hợp Sử Dụng

### 1. Log Analytics — Phân Tích Log

```
Application Logs ──▶ Kinesis Firehose ──▶ OpenSearch
System Logs      ──▶ Fluent Bit        ──▶ Domain
Security Logs    ──▶ Logstash          ──▶ (Index: logs-*)
                                             │
                                             ▼
                                    OpenSearch Dashboards
                                    (Xem log, alert, filter)
```

**Ứng dụng điển hình:**
- Centralized logging (Ghi Log Tập Trung) cho microservices
- Error tracking (Theo Dõi Lỗi) và debugging
- Security audit logs (Log Kiểm Tra Bảo Mật)

### 2. Full-text Search — Tìm Kiếm Toàn Văn Bản

```
Catalog/Product DB ──▶ ETL ──▶ OpenSearch Index
                                    │
                               User Query: "tai nghe khong day"
                                    │
                                    ▼
                            Kết quả có relevance score
                            (Điểm Liên Quan), fuzzy match
```

**Ứng dụng điển hình:**
- E-commerce product search (Tìm Kiếm Sản Phẩm)
- Document search (Tìm Kiếm Tài Liệu) trên nội dung lớn
- Knowledge base search (Tìm Kiếm Cơ Sở Tri Thức)

### 3. Observability — Khả Năng Quan Sát

```
Metrics + Traces + Logs (3 trụ cột Observability)
    │
    ├── AWS X-Ray traces ──▶ OpenSearch Trace Analytics
    ├── CloudWatch Metrics ──▶ OpenSearch (qua Kinesis)
    └── Application Logs ──▶ OpenSearch (qua Fluent Bit)
```

### 4. Security Analytics — Phân Tích Bảo Mật

```
VPC Flow Logs ──▶ OpenSearch ──▶ Threat Detection Dashboards
CloudTrail    ──▶             ──▶ Anomaly Alerts
WAF Logs      ──▶             ──▶ Compliance Reports
```

---

## 🆚 So Sánh Nhanh — OpenSearch vs Các Dịch Vụ Khác

| Tiêu Chí | OpenSearch | CloudWatch Logs | Athena + S3 |
|----------|-----------|-----------------|-------------|
| **Full-text Search** | Xuất sắc | Không hỗ trợ | Giới hạn |
| **Indexing Speed** | Thời gian thực | Thời gian thực | Batch |
| **Query Language** | DSL / SQL | Insights QL | SQL |
| **Visualization** | OpenSearch Dashboards | Có (CloudWatch) | Cần QuickSight |
| **Chi phí** | Instance-based | Pay-per-use | Pay-per-query |
| **Tốt nhất cho** | Search + Log Analytics | AWS Native Logs | Ad-hoc S3 Analysis |

---

## 💰 Mô Hình Tính Phí

| Thành Phần | Cách Tính Phí |
|-----------|---------------|
| **Instance hours** | Giờ chạy của mỗi node (Data/Master/UltraWarm/Cold) |
| **EBS storage** | GB lưu trữ trên Elastic Block Store |
| **UltraWarm storage** | GB trên S3-backed warm tier (rẻ hơn EBS ~90%) |
| **Cold storage** | GB trên S3 cold tier (rẻ nhất, query chậm hơn) |
| **Data transfer** | Dữ liệu truyền ra ngoài region |
| **Multi-AZ** | Số node nhân đôi → chi phí nhân đôi |

### Tiers Lưu Trữ

```
Hot Tier   ─ EBS-based ─ Query nhanh nhất  ─ Đắt nhất
UltraWarm  ─ S3-backed  ─ Query tương đối  ─ ~90% rẻ hơn Hot
Cold Tier  ─ S3-based   ─ Cần attach trước ─ Rẻ nhất
```

---

## 🔗 Điều Hướng Module

| Chủ Đề | File |
|--------|------|
| Clusters, Indices, Shards, Replicas | [1-opensearch-fundamentals.md](./1-opensearch-fundamentals.md) |
| Thu nạp dữ liệu — Kinesis, Logstash, Fluent Bit | [2-opensearch-ingestion.md](./2-opensearch-ingestion.md) |
| Bảo mật — Fine-grained access control | [3-opensearch-security.md](./3-opensearch-security.md) |
| Module trước: QuickSight | [../08-quicksight/README.md](../08-quicksight/README.md) |
| Module sau: MSK — Managed Kafka | [../10-msk/README.md](../10-msk/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
