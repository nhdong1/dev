# 01 — RDS Fundamentals — Nền Tảng Amazon RDS

> Amazon RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ) là dịch vụ cơ sở dữ liệu được quản lý hoàn toàn bởi AWS, cho phép bạn chạy các hệ quản trị cơ sở dữ liệu quan hệ phổ biến mà không cần tự quản lý cơ sở hạ tầng bên dưới.

## 📚 Mục Lục

1. [Tổng Quan RDS](#tổng-quan-rds)
2. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
3. [Kiến Trúc RDS](#kiến-trúc-rds)
4. [Khi Nào Dùng RDS](#khi-nào-dùng-rds)
5. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tổng Quan RDS

Amazon RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ) ra đời năm 2009, là một trong những dịch vụ cơ sở dữ liệu lâu đời nhất và phổ biến nhất của AWS.

### RDS Quản Lý Những Gì Thay Bạn?

| Nhiệm Vụ                                              | Không Dùng RDS (Tự Quản Lý) | Dùng RDS |
| ----------------------------------------------------- | ---------------------------- | -------- |
| Provisioning phần cứng                                | ✅ Bạn làm                   | ❌ AWS làm |
| Cài đặt OS (Hệ Điều Hành)                            | ✅ Bạn làm                   | ❌ AWS làm |
| Cài đặt và cấu hình database engine                  | ✅ Bạn làm                   | ❌ AWS làm |
| Automated backups (Sao Lưu Tự Động)                  | ✅ Bạn làm                   | ❌ AWS làm |
| Vá lỗi bảo mật (Patching)                            | ✅ Bạn làm                   | ❌ AWS làm |
| Scaling phần cứng (Mở Rộng)                          | ✅ Bạn làm                   | ❌ AWS làm |
| High availability (Tính Sẵn Sàng Cao) — Multi-AZ     | ✅ Bạn làm                   | ❌ AWS làm |
| Monitoring (Giám Sát) cơ bản                         | ✅ Bạn làm                   | ❌ AWS làm |
| Query optimization (Tối Ưu Truy Vấn)                 | ✅ Bạn làm                   | ✅ Bạn làm |
| Schema design (Thiết Kế Schema)                       | ✅ Bạn làm                   | ✅ Bạn làm |

### Lợi Ích Chính

- **Tiết kiệm thời gian vận hành:** Không cần lo về patching, backup, hardware
- **High Availability tích hợp:** Multi-AZ (Đa Vùng Sẵn Sàng) với failover tự động
- **Khả năng mở rộng:** Read Replicas (Bản Sao Đọc) và vertical scaling dễ dàng
- **Bảo mật:** Tích hợp VPC, IAM, KMS (Key Management Service — Dịch Vụ Quản Lý Khóa), SSL/TLS

---

## Các Thành Phần Cốt Lõi

### DB Instance (Máy Chủ Cơ Sở Dữ Liệu)

Đây là đơn vị tính toán cốt lõi của RDS — một môi trường database cô lập chạy trong cloud. Mỗi DB Instance:

- Chạy một **DB engine** (engine cơ sở dữ liệu) duy nhất
- Thuộc một **instance class** (lớp máy chủ) xác định CPU và RAM
- Gắn với một **storage** (lưu trữ) riêng
- Có một **endpoint** (điểm truy cập) dạng `mydb.xxxxxx.us-east-1.rds.amazonaws.com`

### DB Engine (Engine Cơ Sở Dữ Liệu)

RDS hỗ trợ 6 engine:

| Engine          | Phiên Bản Được Hỗ Trợ         | Trường Hợp Sử Dụng                |
| --------------- | ------------------------------ | ---------------------------------- |
| **MySQL**       | 8.0, 5.7                       | Web apps, CMS (Hệ Thống Quản Nội Dung), e-commerce |
| **PostgreSQL**  | 16, 15, 14, 13, 12             | Apps cần tính năng SQL nâng cao    |
| **MariaDB**     | 10.11, 10.6, 10.5              | Thay thế MySQL mã nguồn mở         |
| **Oracle**      | 19c, 21c                       | Enterprise apps cũ                 |
| **SQL Server**  | 2019, 2017, 2016, 2014         | Apps hệ sinh thái Microsoft        |
| **Db2**         | 11.5                           | Hệ thống IBM doanh nghiệp          |

> **Lưu ý:** Amazon Aurora (MySQL/PostgreSQL-compatible) là một dịch vụ riêng — xem `02-aurora/`.

### Storage (Lưu Trữ)

Ba loại storage:

| Loại Storage                    | IOPS (Tốc Độ Đọc/Ghi)      | Khi Nào Dùng                              |
| ------------------------------- | ---------------------------- | ----------------------------------------- |
| **gp3** (General Purpose SSD)  | 3,000–16,000 IOPS            | Hầu hết workloads — lựa chọn mặc định    |
| **io1/io2** (Provisioned IOPS) | 1,000–256,000 IOPS           | I/O-intensive (Nhiều Đọc/Ghi), OLTP quy mô lớn |
| **magnetic** (HDD — Ổ Đĩa Cứng Từ Tính) | Thấp (~100 IOPS)   | Không khuyến nghị — legacy (Cũ)          |

### Parameter Groups (Nhóm Tham Số)

- Cấu hình các tham số database engine (ví dụ: `max_connections`, `innodb_buffer_pool_size`)
- Áp dụng cho nhiều DB instances cùng lúc
- **Static parameters** (Tham Số Tĩnh): yêu cầu reboot instance
- **Dynamic parameters** (Tham Số Động): áp dụng ngay không cần reboot

### Option Groups (Nhóm Tùy Chọn)

- Thêm tính năng bổ sung cho database (ví dụ: Oracle APEX, SQL Server Transparent Data Encryption)
- Chỉ áp dụng cho một số engine nhất định

### Subnet Groups (Nhóm Mạng Con)

- Xác định tập hợp các subnets trong VPC mà RDS có thể deploy vào
- **DB Subnet Group** phải chứa ít nhất 2 subnets ở 2 AZ (Availability Zone — Vùng Sẵn Sàng) khác nhau

---

## Kiến Trúc RDS

### Triển Khai Đơn (Single-AZ)

```
                    ┌─────────────────────────────────┐
                    │           VPC (Mạng Ảo)          │
                    │  ┌──────────────────────────┐   │
                    │  │  Private Subnet (us-east-1a)  │
                    │  │  ┌────────────────────┐  │   │
  Application ──── │──│─►│   RDS DB Instance   │  │   │
                    │  │  │  (Primary/Chính)    │  │   │
                    │  │  └────────────────────┘  │   │
                    │  │           │               │   │
                    │  │  ┌────────────────────┐  │   │
                    │  │  │   EBS Storage       │  │   │
                    │  │  │ (gp3/io1/io2)      │  │   │
                    │  │  └────────────────────┘  │   │
                    │  └──────────────────────────┘   │
                    └─────────────────────────────────┘
```

### Triển Khai Multi-AZ (Đa Vùng Sẵn Sàng)

```
                    ┌──────────────────────────────────────────┐
                    │               VPC (Mạng Ảo)              │
                    │  ┌─────────────────┐  ┌───────────────┐  │
                    │  │ Subnet (us-east-1a) │ Subnet (1b)  │  │
                    │  │ ┌─────────────┐ │  │ ┌───────────┐ │  │
Application ──── ───│──│►│  Primary DB │ │  │ │ Standby   │ │  │
(qua DNS endpoint)  │  │ │  (Chính)    │ │  │ │ (Dự Phòng)│ │  │
                    │  │ └─────────────┘ │  │ └───────────┘ │  │
                    │  │       │         │  │       ▲        │  │
                    │  └───────│─────────┘  └───────│────────┘  │
                    │          └───── Sync Replication ─────────│
                    └──────────────────────────────────────────┘
```

**Điểm mấu chốt:** DNS endpoint KHÔNG thay đổi khi failover — ứng dụng không cần cập nhật connection string.

---

## Khi Nào Dùng RDS

### Nên Dùng RDS Khi

- Ứng dụng cần **ACID transactions** (Giao Dịch ACID — Tính Nguyên Tử, Nhất Quán, Cô Lập, Bền Vững)
- Dữ liệu có cấu trúc và quan hệ rõ ràng (relational data)
- Cần **SQL** đầy đủ với JOINs phức tạp
- Team đã quen với MySQL/PostgreSQL/SQL Server
- Workload OLTP (Online Transaction Processing — Xử Lý Giao Dịch Trực Tuyến)

### Nên Xem Xét Aurora Thay RDS Khi

- Cần hiệu năng cao hơn với chi phí hợp lý
- Cần global replication (sao chép toàn cầu)
- Workload có thể tận dụng Aurora Serverless v2

### Không Nên Dùng RDS Khi

- Cần millisecond latency (Độ Trễ Mili-giây) → dùng DynamoDB
- Dữ liệu key-value hoặc document → dùng DynamoDB
- OLAP (Online Analytical Processing — Xử Lý Phân Tích Trực Tuyến) / Data Warehouse → dùng Redshift
- Cần sub-millisecond reads → dùng ElastiCache

---

## Nội Dung Chi Tiết

| File                              | Nội Dung                                              | Trạng Thái |
| --------------------------------- | ----------------------------------------------------- | ---------- |
| [1-engine-types.md](1-engine-types.md)           | Engine types, so sánh, migration paths       | ✅ Hoàn thành |
| [2-instance-storage.md](2-instance-storage.md)   | Instance classes, storage types, IOPS        | ✅ Hoàn thành |
| [3-multi-az.md](3-multi-az.md)                   | Multi-AZ deployment, failover, replication   | ✅ Hoàn thành |
| [4-read-replicas.md](4-read-replicas.md)         | Read replicas, cross-region, use cases       | ✅ Hoàn thành |
| [5-parameter-groups.md](5-parameter-groups.md)   | Parameter groups, option groups, tuning      | ✅ Hoàn thành |
| [6-rds-proxy.md](6-rds-proxy.md)                 | RDS Proxy, connection pooling, Lambda        | ✅ Hoàn thành |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**Q: RDS Multi-AZ và Read Replica khác nhau như thế nào?**

| Tiêu Chí                | Multi-AZ (Đa Vùng Sẵn Sàng)          | Read Replica (Bản Sao Đọc)        |
| ----------------------- | ------------------------------------- | ---------------------------------- |
| **Mục đích chính**      | High Availability (Tính Sẵn Sàng Cao) | Cải thiện hiệu năng đọc           |
| **Đồng bộ hóa**         | Synchronous (Đồng Bộ)                 | Asynchronous (Bất Đồng Bộ)        |
| **Nhận traffic đọc**    | Không (chỉ standby)                   | Có (phục vụ read requests)         |
| **Failover**            | Tự động (60-120 giây)                 | Thủ công (promote)                 |
| **Cross-region**        | Không hỗ trợ                          | Có hỗ trợ                          |
| **Số lượng**            | 1 standby                             | Tối đa 5 (MySQL), 15 (Aurora)      |

**Q: RDS managed những gì? Bạn quản lý những gì?**
- **AWS quản lý:** Hardware, OS, engine patching, automated backup, Multi-AZ failover, storage scaling
- **Bạn quản lý:** Schema design, query optimization, indexes, application logic, parameter tuning

**Q: Khi nào nên dùng Multi-AZ?**
Production workloads cần HA — khi downtime là không chấp nhận được. Không dùng cho dev/test để tiết kiệm chi phí.

**Q: gp3 và io1 khác nhau thế nào? Khi nào dùng io1?**
- **gp3:** Mặc định, 3,000 IOPS cơ bản, chi phí thấp, phù hợp hầu hết workloads
- **io1:** Khi cần >16,000 IOPS, IOPS/storage độc lập nhau, dùng cho OLTP heavy I/O

### Câu Hỏi Nâng Cao

**Q: Giải thích cơ chế failover của RDS Multi-AZ.**

Khi primary instance gặp sự cố:
1. RDS phát hiện lỗi qua health check
2. DNS endpoint được trỏ sang standby instance (TTL ~5 giây)
3. Standby được promote thành primary
4. Thời gian failover: 60-120 giây thông thường

**Q: RDS Proxy giải quyết vấn đề gì?**

Lambda và các serverless functions tạo rất nhiều short-lived connections — connection pool không đủ. RDS Proxy duy trì connection pool bền vững, giảm tải trên database, tăng tốc failover.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [1-engine-types.md](1-engine-types.md) — Engine Types
- **Xem thêm:** [02-aurora/README.md](../02-aurora/README.md) — So sánh RDS vs Aurora
- **Tham khảo:** [05-ha-backup/README.md](../05-ha-backup/README.md) — HA & Backup

---

**Cập Nhật Lần Cuối:** 2026-05-15
