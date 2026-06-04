# Kiến Thức Quản Trị Cơ Sở Dữ Liệu (DBA) - Tài Liệu Tiếng Việt

> Hướng dẫn toàn diện về Quản Trị Cơ Sở Dữ Liệu, bao gồm tất cả kỹ năng từ cơ bản đến nâng cao.

## Mục Lục

1. [Lộ Trình Học Tập](#lộ-trình-học-tập)
2. [Các Năng Lực Cốt Lõi](#các-năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)

---

## Lộ Trình Học Tập

### **Giai Đoạn 1: Kiến Thức Cơ Bản (Tuần 1-2)**

- [ ] Cơ bản về CSDL & Tính chất ACID
- [ ] Quản lý Transaction & Mức độ Isolation
- [ ] Chiến lược Index
- [ ] Giám sát cơ bản

### **Giai Đoạn 2: Kỹ Năng DBA Cốt Lõi (Tuần 3-6)**

- [ ] Sao lưu & Phục hồi (RPO/RTO)
- [ ] Tính Sẵn Sàng Cao & Nhân bản (Replication)
- [ ] Tối ưu Hiệu suất
- [ ] Bảo mật & Kiểm soát Truy cập

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7-10)**

- [ ] Di chuyển Schema & Quản lý Thay đổi
- [ ] Lập kế hoạch Dung lượng
- [ ] Xử lý Sự cố & Ứng phó
- [ ] Quản lý CSDL trên Cloud (AWS & Azure — SQL & NoSQL)

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] Tìm hiểu sâu về từng nền tảng
- [ ] Chiến lược nhân bản nâng cao
- [ ] Các mẫu thiết kế CSDL
- [ ] Tuân thủ & Quản trị

---

## Các Năng Lực Cốt Lõi

| Năng lực                       | Ưu tiên | Thời gian | Trạng thái |
| ------------------------------ | ------- | --------- | ---------- |
| **Sao lưu & Khắc phục Thảm họa** | ⭐⭐⭐ | 2 tuần   | -          |
| **Tối ưu Hiệu suất**           | ⭐⭐⭐  | 3 tuần   | -          |
| **Tính Sẵn Sàng Cao**          | ⭐⭐⭐  | 2 tuần   | -          |
| **Bảo mật & Tuân thủ**         | ⭐⭐⭐  | 2 tuần   | -          |
| **Nhân bản & Failover**        | ⭐⭐⭐  | 2 tuần   | -          |
| **Giám sát & Cảnh báo**        | ⭐⭐⭐  | 1 tuần   | -          |
| **Di chuyển Schema**           | ⭐⭐⭐  | 2 tuần   | -          |
| **Xử lý Sự cố**               | ⭐⭐⭐  | 2 tuần   | -          |
| **Lập kế hoạch Dung lượng**   | ⭐⭐    | 1 tuần   | -          |
| **CSDL trên Cloud**            | ⭐⭐    | 2 tuần   | -          |

---

## Tổng Quan Chủ Đề

### **1. Kiến Thức Cơ Bản** (`01-co-ban/`)

- Tính chất ACID & Transaction
- Mức độ Isolation của Transaction
- Index cơ bản (B-Tree, Hash, Full-Text)
- Các loại CSDL (RDBMS, NoSQL)
- Connection Pooling

### **2. Sao Lưu & Phục Hồi** (`02-sao-luu-phuc-hoi/`)

- **RPO vs RTO** — Yêu cầu nghiệp vụ
- Chiến lược sao lưu (Toàn bộ, Tăng dần, Chênh lệch)
- Lưu trữ WAL & Point-in-Time Recovery
- Kiểm tra khôi phục & Runbook
- Mã hóa sao lưu
- Giải pháp sao lưu trên Cloud

### **3. Tính Sẵn Sàng Cao & Nhân Bản** (`03-ha-replication/`)

- Các loại nhân bản (Single-Leader, Multi-Leader, Leaderless)
- Nhân bản Đồng bộ vs Bất đồng bộ
- Giám sát độ trễ nhân bản
- Cơ chế Failover
- Chiến lược Read Replica
- Ngăn ngừa Split-brain

### **4. Tối Ưu Hiệu Suất** (`04-toi-uu-hieu-suat/`)

- Phân tích truy vấn & EXPLAIN Plans
- Thiết kế & Tối ưu Index
- Autovacuum & Quản lý Bloat
- Thống kê & Kế hoạch truy vấn
- Bão hòa Connection Pool
- Phát hiện & Phân tích truy vấn chậm
- Tranh chấp khóa & Deadlock

### **5. Bảo Mật & Tuân Thủ** (`05-bao-mat-tuan-thu/`)

- Mô hình quyền tối thiểu
- Kiểm soát truy cập dựa trên vai trò (RBAC)
- Bảo mật cấp hàng (RLS)
- Mã hóa (Lưu trữ & Truyền tải)
- Ghi nhật ký kiểm tra
- Tuân thủ GDPR & PCI-DSS
- Quản lý bí mật & Vault

### **6. Di Chuyển Schema** (`06-schema-migrations/`)

- Migration chỉ tiến
- Mẫu Expand-Contract
- Triển khai không có thời gian chết
- Công cụ Migration (Flyway, Liquibase)
- Xác thực dữ liệu & Checksum
- Chiến lược rollback

### **7. Cloud Database — AWS & Azure** (`07-cloud-database/`)

- Tổng quan dịch vụ: RDS, Aurora, DynamoDB, ElastiCache vs Azure DB, Cosmos DB, Redis
- Setup SQL trên Cloud: networking, parameter group, Multi-AZ, RDS Proxy
- Setup NoSQL trên Cloud: DynamoDB partition key, Cosmos DB RU, Redis eviction
- Bảo mật & Mạng: VPC/VNet isolation, encryption, IAM auth, secrets rotation
- Quản lý & Vận hành: monitoring, scaling, maintenance, DR runbook
- Tối ưu Chi phí: Reserved Instances, right-sizing, storage type, data transfer

### **8. Phỏng Vấn** (`08-phong-van/`)

- Top 20 câu hỏi phỏng vấn DBA
- Tình huống thiết kế hệ thống
- Câu chuyện xử lý sự cố (phương pháp STAR)
- Thách thức tối ưu SQL
- Các mẫu thiết kế cho quy mô lớn

---

## Theo Nền Tảng CSDL

### **PostgreSQL**

```
Điểm mạnh: ACID, Extensions, JSONB, PostGIS, Replication
Lý tưởng cho: Truy vấn phức tạp, phân tích, dữ liệu địa không gian
```

### **MySQL/InnoDB**

```
Điểm mạnh: Độ tin cậy, Tốc độ, Hệ sinh thái, Replication
Lý tưởng cho: Ứng dụng web, CRUD đơn giản, tải nặng đọc
```

### **MongoDB**

```
Điểm mạnh: Mô hình tài liệu, Linh hoạt, Mở rộng ngang
Lý tưởng cho: Schema linh hoạt, phát triển nhanh
```

### **Redis**

```
Điểm mạnh: Độ trễ cực thấp, In-memory, Nhiều cấu trúc dữ liệu
Lý tưởng cho: Caching, sessions, rate limiting, pub/sub
```

---

## Ma Trận Kỹ Năng

### Người Mới (0-1 năm)

- [ ] Tính chất ACID và transaction
- [ ] Các loại index cơ bản
- [ ] Quy trình sao lưu đơn giản
- [ ] SQL cơ bản
- [ ] Khái niệm connection pooling

### Trung Cấp (1-3 năm)

- [ ] Thiết lập và giám sát replication
- [ ] Phân tích hiệu suất với EXPLAIN
- [ ] Điều chỉnh autovacuum
- [ ] Lưu trữ WAL
- [ ] Thiết kế kiến trúc HA
- [ ] Lập kế hoạch di chuyển schema

### Nâng Cao (3-5+ năm)

- [ ] Chiến lược multi-datacenter
- [ ] Kiến trúc sharding
- [ ] Xử lý sự cố & Post-mortem
- [ ] Mô hình lập kế hoạch dung lượng
- [ ] Tối ưu CSDL Cloud
- [ ] Khung tuân thủ quy định

---

## Hướng Dẫn Sử Dụng

### Tự Học

1. Bắt đầu với Lộ trình học tập
2. Học từng giai đoạn theo thứ tự
3. Thực hành bài tập
4. Xây dựng dự án portfolio

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào `08-phong-van/`
2. Nghiên cứu sâu về nền tảng mục tiêu
3. Chuẩn bị câu chuyện sự cố (STAR)
4. Luyện tập giải thích rõ ràng

---

**Cập nhật lần cuối:** 2026-04-26
**Phiên bản:** 1.0
