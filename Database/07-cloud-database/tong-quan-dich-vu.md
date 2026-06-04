# Tổng Quan Dịch Vụ Database trên Cloud

## AWS vs Azure — Bảng So Sánh Nhanh

### SQL (Relational)

| Tính năng | AWS | Azure |
|-----------|-----|-------|
| PostgreSQL managed | RDS PostgreSQL / Aurora PostgreSQL | Azure Database for PostgreSQL (Flexible Server) |
| MySQL managed | RDS MySQL / Aurora MySQL | Azure Database for MySQL (Flexible Server) |
| SQL Server | RDS SQL Server | Azure SQL Database / SQL Managed Instance |
| Serverless SQL | Aurora Serverless v2 | Azure SQL Database Serverless |
| Giá rẻ dev/test | RDS t3.micro (Free Tier) | Azure SQL Basic (5 DTU ~$5/tháng) |

### NoSQL

| Loại | AWS | Azure |
|------|-----|-------|
| Document / Key-Value | DynamoDB | Cosmos DB (Core SQL API) |
| MongoDB compatible | DocumentDB | Cosmos DB (MongoDB API) |
| Cassandra compatible | Keyspaces | Cosmos DB (Cassandra API) |
| Redis / Cache | ElastiCache for Redis | Azure Cache for Redis |
| In-memory + persistent | MemoryDB for Redis | Azure Cache for Redis (Enterprise) |
| Graph | Neptune | Cosmos DB (Gremlin API) |
| Time-series | Timestream | Azure Data Explorer |
| Search | OpenSearch Service | Azure Cognitive Search |

---

## AWS RDS — Kiến Trúc Tổng Quan

```
                    ┌─────────────────────────────┐
                    │         Application          │
                    └──────────────┬──────────────┘
                                   │ (DNS Endpoint)
                    ┌──────────────▼──────────────┐
                    │      RDS Proxy (optional)    │  ← giảm connection overhead
                    └──────────────┬──────────────┘
               ┌───────────────────┼───────────────────┐
               │                   │                   │
    ┌──────────▼──────┐  ┌─────────▼────────┐  ┌──────▼──────────┐
    │   Primary (AZ-a) │  │ Read Replica (AZ-b)│  │ Read Replica   │
    │   Writer          │  │ Reader Endpoint   │  │ (cross-region) │
    └──────────┬────────┘  └──────────────────┘  └────────────────┘
               │ sync replication
    ┌──────────▼────────┐
    │  Standby (AZ-b)   │  ← Multi-AZ failover ~60s
    │  (không đọc được) │
    └───────────────────┘
```

### Phân biệt Multi-AZ vs Read Replica

| | Multi-AZ | Read Replica |
|--|----------|--------------|
| **Mục đích** | High Availability (failover) | Scale đọc |
| **Đọc được không?** | Không (standby thụ động) | Có |
| **Replication** | Synchronous | Asynchronous |
| **Chi phí** | ~2x instance | Tính riêng từng replica |
| **Tự động failover** | Có (~60s) | Không tự động |

---

## Aurora — Điểm Khác Biệt Quan Trọng

Aurora **không phải** RDS thông thường. Storage layer hoàn toàn khác:

```
┌──────────────────────────────────────────────────────────┐
│                    Aurora Cluster                        │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ Writer      │    │ Reader 1    │    │ Reader 2    │  │
│  │ Instance    │    │ Instance    │    │ Instance    │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│         └──────────────────┴──────────────────┘         │
│                             │                           │
│         ┌───────────────────▼──────────────────┐        │
│         │     Shared Distributed Storage        │        │
│         │   (6 copies across 3 AZs, 10GB chunks)│        │
│         └────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────┘
```

**Ưu điểm Aurora so với RDS:**
- Storage tự mở rộng đến 128 TB, không cần provision
- Failover nhanh hơn (~30s vs ~60s)
- Read Replica lag thấp hơn (thường < 100ms)
- Aurora Serverless v2: scale compute trong giây

**Nhược điểm:**
- Giá cao hơn RDS ~20%
- Không có Free Tier
- Chỉ hỗ trợ PostgreSQL và MySQL

---

## Azure Database for PostgreSQL — Flexible Server

```
Flexible Server (hiện tại, được khuyến nghị)
├── Single Zone: 1 instance, không HA
├── Zone Redundant HA:
│     Primary (Zone 1) ──sync──> Standby (Zone 2)
│     Failover: ~60-120s, automatic
└── Same Zone HA:
      Primary ──sync──> Standby (cùng zone, rẻ hơn)

Compute options:
  Burstable   (B series)  → dev/test, không liên tục
  General Purpose (D/E)   → production workloads
  Memory Optimized (E)    → heavy in-memory workloads
```

---

## DynamoDB — Kiến Trúc

```
┌─────────────────────────────────────────────────────┐
│                    DynamoDB Table                    │
│                                                      │
│  Partition Key (Hash Key) → xác định partition nào  │
│  Sort Key (Range Key)     → optional, sắp xếp trong │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │Partition1│  │Partition2│  │Partition3│  ...       │
│  │ (AZ-a)   │  │ (AZ-b)   │  │ (AZ-c)   │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                      │
│  Read modes:  Eventually Consistent (default, rẻ)   │
│               Strongly Consistent (2x RCU)           │
│               Transactional (2x RCU)                 │
└─────────────────────────────────────────────────────┘
```

---

## Cosmos DB — Multi-Model, Multi-Region

```
┌─────────────────────────────────────────────────────────┐
│                   Cosmos DB Account                     │
│                                                         │
│  APIs: Core SQL │ MongoDB │ Cassandra │ Gremlin │ Table │
│                                                         │
│  Consistency levels (5 cấp):                           │
│  Strong ──> Bounded Staleness ──> Session ──>          │
│  Consistent Prefix ──> Eventual                        │
│  (mạnh hơn = đắt hơn và chậm hơn)                     │
│                                                         │
│  Multi-region write (active-active):                   │
│  Region 1 (write) ←──sync──→ Region 2 (write)         │
│        ↓                           ↓                   │
│  Region 3 (read)             Region 4 (read)           │
└─────────────────────────────────────────────────────────┘
```

**Lưu ý quan trọng Cosmos DB:**
- Đơn vị tính phí là **RU (Request Unit)** — 1 RU ≈ đọc 1KB item
- Phải estimate RU/s khi provision (hoặc dùng Autoscale)
- Chọn Partition Key sai → hot partition → throttling
