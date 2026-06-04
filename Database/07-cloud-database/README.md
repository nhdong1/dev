# Quản Lý Database Trên Cloud (AWS & Azure)

> Hướng dẫn chi tiết thiết lập, quản lý và vận hành SQL/NoSQL database trên AWS và Azure — bao gồm các lưu ý quan trọng, best practice và các bẫy thường gặp.

---

## Mục Lục

| # | Chủ đề | File |
|---|--------|------|
| 1 | Tổng quan dịch vụ Cloud DB | `tong-quan-dich-vu.md` |
| 2 | Setup SQL Database trên Cloud | `setup-sql-cloud.md` |
| 3 | Setup NoSQL Database trên Cloud | `setup-nosql-cloud.md` |
| 4 | Bảo mật & Mạng | `bao-mat-va-mang.md` |
| 5 | Quản lý & Vận hành | `quan-ly-va-van-hanh.md` |
| 6 | Tối ưu Chi phí | `toi-uu-chi-phi.md` |

---

## Tại Sao Database trên Cloud?

### Ưu điểm
- **Managed service**: Cloud provider lo phần cứng, patching OS, HA infrastructure
- **Elasticity**: Scale up/down theo nhu cầu thực tế
- **Tích hợp sẵn**: Backup tự động, Multi-AZ, Read Replica chỉ vài click
- **SLA cao**: AWS RDS Multi-AZ đạt 99.95% uptime SLA

### Nhược điểm & Rủi ro cần biết
- **Vendor lock-in**: Aurora, Cosmos DB có cú pháp/tính năng riêng khó migrate
- **Chi phí ẩn**: Data transfer, IOPS provisioned, backup storage dễ đội ngân sách
- **Ít kiểm soát**: Không có OS access, một số cấu hình bị hạn chế
- **Latency mạng**: So với on-premise, thêm ~1-5ms network hop

---

## Lựa Chọn Dịch Vụ Nhanh

```
Cần SQL ACID + managed?
  ├─ PostgreSQL compatible → AWS Aurora PostgreSQL / Azure Database for PostgreSQL
  ├─ MySQL compatible      → AWS Aurora MySQL / Azure Database for MySQL
  └─ SQL Server            → AWS RDS SQL Server / Azure SQL Database

Cần NoSQL?
  ├─ Key-Value / Document  → AWS DynamoDB / Azure Cosmos DB
  ├─ Cache / Session       → AWS ElastiCache (Redis) / Azure Cache for Redis
  ├─ Wide-column           → AWS Keyspaces (Cassandra) / Azure Cosmos DB Cassandra
  └─ Search                → AWS OpenSearch / Azure Cognitive Search
```

---

**Cập nhật:** 2026-04-30
