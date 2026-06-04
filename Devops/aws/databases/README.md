# 🗄️ AWS Database Services — Lộ Trình Học Toàn Diện

> Hướng dẫn đầy đủ về AWS Database Services — từ RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ) và Aurora (Cơ Sở Dữ Liệu Đám Mây Hiệu Năng Cao) đến DynamoDB (Cơ Sở Dữ Liệu NoSQL Serverless), ElastiCache (Bộ Nhớ Đệm Phân Tán), Redshift (Data Warehouse — Kho Dữ Liệu), và các chiến lược lựa chọn, tối ưu hóa cơ sở dữ liệu trên AWS.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Dịch Vụ](#tổng-quan-các-dịch-vụ)
4. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
5. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
6. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng — Fundamentals (Tuần 1-2)**

- [ ] RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ) — Cơ bản, Engine Types (Loại Engine)
- [ ] Multi-AZ (Multi Availability Zone — Đa Vùng Sẵn Sàng) & Read Replicas (Bản Sao Đọc)
- [ ] Aurora (Cơ Sở Dữ Liệu Đám Mây Hiệu Năng Cao) — Architecture (Kiến Trúc), Cluster (Cụm)
- [ ] DynamoDB (Cơ Sở Dữ Liệu NoSQL Serverless) — Tables, Items, Attributes
- [ ] Mô Hình Định Giá — On-Demand vs Provisioned (Theo Yêu Cầu vs Được Cung Cấp Sẵn)

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi — Core Skills (Tuần 3-6)**

- [ ] Aurora Serverless (Aurora Không Máy Chủ) & Global Database (Cơ Sở Dữ Liệu Toàn Cầu)
- [ ] DynamoDB — Indexes (Chỉ Mục), Streams (Luồng Dữ Liệu), Transactions (Giao Dịch)
- [ ] ElastiCache (Bộ Nhớ Đệm Phân Tán) — Redis vs Memcached, Replication (Sao Chép)
- [ ] Backup & Restore (Sao Lưu & Khôi Phục), PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm)
- [ ] Database Security (Bảo Mật Cơ Sở Dữ Liệu) — IAM, Encryption (Mã Hóa), VPC

### **Giai Đoạn 3: Vận Hành Nâng Cao — Advanced Operations (Tuần 7-10)**

- [ ] Redshift (Data Warehouse — Kho Dữ Liệu) — Cluster, Distribution (Phân Phối Dữ Liệu)
- [ ] Database Migration Service — DMS (Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu)
- [ ] Schema Conversion Tool — SCT (Công Cụ Chuyển Đổi Schema)
- [ ] Performance Tuning (Tối Ưu Hiệu Năng) & Monitoring (Giám Sát) với CloudWatch
- [ ] Cost Optimization (Tối Ưu Chi Phí) — Reserved Instances, Storage Tiers

### **Giai Đoạn 4: Chuyên Sâu — Specialization (Tuần 11+)**

- [ ] Neptune (Cơ Sở Dữ Liệu Đồ Thị) & DocumentDB (Cơ Sở Dữ Liệu Tài Liệu)
- [ ] Keyspaces (Dịch Vụ Cassandra Trên AWS) & Timestream (Cơ Sở Dữ Liệu Chuỗi Thời Gian)
- [ ] Multi-Region Architecture (Kiến Trúc Đa Vùng) & Disaster Recovery (Khôi Phục Thảm Họa)
- [ ] Advanced DynamoDB Patterns (Các Mẫu DynamoDB Nâng Cao)

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                            | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| --------------------------------------------------- | ---------- | --------- | ---------- |
| **RDS & Aurora — Cơ Sở Dữ Liệu Quan Hệ**           | ⭐⭐⭐      | 2 tuần    | -          |
| **DynamoDB — NoSQL Serverless**                     | ⭐⭐⭐      | 2 tuần    | -          |
| **High Availability — Tính Sẵn Sàng Cao**          | ⭐⭐⭐      | 1 tuần    | -          |
| **Backup & Recovery — Sao Lưu & Khôi Phục**        | ⭐⭐⭐      | 1 tuần    | -          |
| **Security — Bảo Mật Cơ Sở Dữ Liệu**               | ⭐⭐⭐      | 1 tuần    | -          |
| **ElastiCache — Bộ Nhớ Đệm Phân Tán**              | ⭐⭐⭐      | 1 tuần    | -          |
| **Performance Tuning — Tối Ưu Hiệu Năng**          | ⭐⭐⭐      | 2 tuần    | -          |
| **Redshift — Data Warehouse**                       | ⭐⭐        | 1 tuần    | -          |
| **DMS — Di Chuyển Cơ Sở Dữ Liệu**                  | ⭐⭐        | 1 tuần    | -          |
| **Cost Optimization — Tối Ưu Chi Phí**              | ⭐⭐        | 1 tuần    | -          |

---

## 🗂️ Tổng Quan Các Dịch Vụ

### **Cơ Sở Dữ Liệu Quan Hệ (Relational Databases)**

| Dịch Vụ        | Mô Tả                                               | Trường Hợp Dùng Tốt Nhất              |
| -------------- | --------------------------------------------------- | -------------------------------------- |
| **RDS**        | Managed SQL databases — Cơ sở dữ liệu SQL Managed  | OLTP (Xử Lý Giao Dịch Trực Tuyến)     |
| **Aurora**     | Cloud-native high-performance DB — Hiệu năng cao   | OLTP quy mô lớn, cần HA tốt           |
| **Aurora Serverless** | Auto-scaling Aurora — Aurora tự động co giãn | Workload không liên tục, dev/test      |

### **Cơ Sở Dữ Liệu NoSQL**

| Dịch Vụ           | Mô Tả                                                      | Trường Hợp Dùng Tốt Nhất                   |
| ----------------- | ---------------------------------------------------------- | ------------------------------------------ |
| **DynamoDB**      | Serverless NoSQL — key-value & document                    | Millisecond latency (Độ Trễ Mili-giây)     |
| **DocumentDB**    | MongoDB-compatible managed DB — Tương thích MongoDB       | Migrate từ MongoDB                         |
| **Neptune**       | Graph database — Cơ sở dữ liệu đồ thị                     | Social graphs (Đồ Thị Xã Hội), fraud       |
| **Keyspaces**     | Managed Apache Cassandra — Cassandra Managed               | Wide-column workloads (Dữ Liệu Cột Rộng)  |
| **Timestream**    | Time-series database — Cơ sở dữ liệu chuỗi thời gian      | IoT, metrics, telemetry (Đo Từ Xa)        |

### **Bộ Nhớ Đệm & Tìm Kiếm (Cache & Search)**

| Dịch Vụ           | Mô Tả                                                      | Trường Hợp Dùng Tốt Nhất                   |
| ----------------- | ---------------------------------------------------------- | ------------------------------------------ |
| **ElastiCache Redis** | In-memory cache — Bộ nhớ đệm trong RAM             | Session, rate limiting, pub/sub            |
| **ElastiCache Memcached** | Simple distributed cache — Cache phân tán       | Simple caching, multi-threaded             |
| **OpenSearch**    | Search & analytics engine — Công cụ tìm kiếm               | Full-text search, log analytics            |

### **Phân Tích Dữ Liệu (Analytics)**

| Dịch Vụ     | Mô Tả                                                              | Trường Hợp Dùng Tốt Nhất      |
| ----------- | ------------------------------------------------------------------ | ------------------------------ |
| **Redshift** | Cloud data warehouse — Kho dữ liệu đám mây                       | OLAP (Phân Tích Dữ Liệu Lớn)  |
| **Athena**  | Serverless query on S3 — Truy vấn không máy chủ trên S3           | Ad-hoc analytics, data lake    |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. RDS Fundamentals — Nền Tảng RDS** (`01-rds-fundamentals/`)

- Tổng quan RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ)
- Engine Types (Loại Engine): MySQL, PostgreSQL, MariaDB, Oracle, SQL Server
- Instance Classes (Lớp Máy Chủ) & Storage Types (Loại Lưu Trữ)
- Parameter Groups (Nhóm Tham Số) & Option Groups (Nhóm Tùy Chọn)
- Multi-AZ Deployment (Triển Khai Đa Vùng Sẵn Sàng)
- Read Replicas (Bản Sao Đọc) & Replication (Sao Chép)

### 📁 **2. Aurora — Cơ Sở Dữ Liệu Đám Mây Cao Cấp** (`02-aurora/`)

- Aurora Architecture (Kiến Trúc Aurora) — Shared Storage (Lưu Trữ Dùng Chung)
- Aurora Cluster (Cụm Aurora) — Writer & Reader Endpoints (Điểm Truy Cập Ghi & Đọc)
- Aurora Serverless v2 (Aurora Không Máy Chủ Phiên Bản 2) — Auto Scaling
- Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu) — Cross-Region Replication
- Aurora vs RDS — Trade-offs (Đánh Đổi) & Cost Comparison (So Sánh Chi Phí)
- Failover (Chuyển Đổi Dự Phòng) & High Availability (Tính Sẵn Sàng Cao)

### 📁 **3. DynamoDB — NoSQL Serverless** (`03-dynamodb/`)

- DynamoDB Data Model (Mô Hình Dữ Liệu) — Tables, Partition Key, Sort Key
- Read/Write Capacity Modes (Chế Độ Đọc/Ghi): On-Demand vs Provisioned
- GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu) & LSI (Local Secondary Index — Chỉ Mục Phụ Cục Bộ)
- DynamoDB Streams (Luồng Dữ Liệu) & Lambda Integration (Tích Hợp Lambda)
- Transactions (Giao Dịch) & ACID trên DynamoDB
- DynamoDB Accelerator — DAX (Bộ Nhớ Đệm DynamoDB) — In-Memory Caching
- Access Patterns (Mẫu Truy Cập) & Single-Table Design (Thiết Kế Đơn Bảng)

### 📁 **4. ElastiCache — Bộ Nhớ Đệm Phân Tán** (`04-elasticache/`)

- Redis vs Memcached — Khi nào dùng gì
- Redis Cluster Mode (Chế Độ Cluster Redis) & Replication Groups (Nhóm Sao Chép)
- ElastiCache for Redis — Persistence (Tính Bền Vững Dữ Liệu): RDB & AOF
- Caching Strategies (Chiến Lược Cache): Lazy Loading, Write-Through, Write-Around
- Session Caching (Lưu Cache Session) & Rate Limiting (Giới Hạn Tốc Độ)
- ElastiCache Security (Bảo Mật ElastiCache) — VPC, Encryption, Auth

### 📁 **5. High Availability & Backup — Tính Sẵn Sàng Cao & Sao Lưu** (`05-ha-backup/`)

- RDS Multi-AZ vs Read Replicas (Đa Vùng vs Bản Sao Đọc) — Khác biệt then chốt
- RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) & RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục)
- Automated Backups (Sao Lưu Tự Động) & Manual Snapshots (Ảnh Chụp Thủ Công)
- PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm)
- Cross-Region Backup (Sao Lưu Xuyên Vùng) & Disaster Recovery (Khôi Phục Thảm Họa)
- Failover Mechanisms (Cơ Chế Chuyển Đổi Dự Phòng) — Automated & Manual

### 📁 **6. Security — Bảo Mật Cơ Sở Dữ Liệu** (`06-security/`)

- VPC Integration (Tích Hợp VPC) — Private Subnets (Mạng Con Riêng Tư), Security Groups
- IAM Authentication (Xác Thực IAM) cho RDS & DynamoDB
- Encryption at Rest (Mã Hóa Khi Lưu Trữ) — KMS (Key Management Service — Dịch Vụ Quản Lý Khóa)
- Encryption in Transit (Mã Hóa Khi Truyền Tải) — TLS/SSL
- Secrets Manager (Quản Lý Bí Mật) & Parameter Store (Kho Tham Số) cho Credentials
- Audit Logging (Nhật Ký Kiểm Toán) — CloudTrail, Database Activity Streams
- Compliance (Tuân Thủ) — PCI-DSS, HIPAA, GDPR

### 📁 **7. Performance Tuning — Tối Ưu Hiệu Năng** (`07-performance-tuning/`)

- RDS Performance Insights (Thông Tin Hiệu Năng RDS) — Query Analysis (Phân Tích Truy Vấn)
- Enhanced Monitoring (Giám Sát Nâng Cao) & CloudWatch Metrics (Số Liệu CloudWatch)
- Slow Query Log (Nhật Ký Truy Vấn Chậm) & EXPLAIN Plans
- Connection Pooling (Gộp Kết Nối) — RDS Proxy (Proxy RDS)
- DynamoDB Performance (Hiệu Năng DynamoDB) — Hot Partitions (Phân Vùng Nóng), Throttling (Giới Hạn)
- Index Optimization (Tối Ưu Chỉ Mục) — GSI Design Best Practices
- Storage Auto Scaling (Tự Động Co Giãn Lưu Trữ) & Instance Sizing (Định Cỡ Máy Chủ)

### 📁 **8. Database Migration — Di Chuyển Cơ Sở Dữ Liệu** (`08-migration/`)

- AWS DMS (Database Migration Service — Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu) — Tổng quan
- SCT (Schema Conversion Tool — Công Cụ Chuyển Đổi Schema) — Heterogeneous Migration (Di Chuyển Khác Loại)
- Homogeneous Migration (Di Chuyển Cùng Loại) vs Heterogeneous Migration
- CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) — Online Migration (Di Chuyển Trực Tuyến)
- Migration Strategies (Chiến Lược Di Chuyển) — Lift-and-Shift, Re-platform, Re-architect
- Cutover Planning (Kế Hoạch Cắt Giảm) & Rollback Strategy (Chiến Lược Quay Lại)

### 📁 **9. Monitoring & Observability — Giám Sát & Quan Sát** (`09-monitoring/`)

- CloudWatch Metrics (Số Liệu CloudWatch) — Key Database KPIs (Chỉ Số Hiệu Suất Chính)
- RDS Performance Insights (Thông Tin Hiệu Năng RDS) Dashboard
- Enhanced Monitoring (Giám Sát Nâng Cao) — OS-level Metrics (Số Liệu Cấp Hệ Điều Hành)
- DynamoDB CloudWatch Alarms (Cảnh Báo CloudWatch DynamoDB)
- Database Activity Streams (Luồng Hoạt Động Cơ Sở Dữ Liệu) — Audit & Compliance
- Alerting Thresholds (Ngưỡng Cảnh Báo) & SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ)

### 📁 **10. Advanced Topics — Chủ Đề Nâng Cao** (`10-advanced/`)

- Redshift (Kho Dữ Liệu) — Architecture (Kiến Trúc), Distribution Keys (Khóa Phân Phối)
- Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu) — Multi-Region Active-Active
- DynamoDB Global Tables (Bảng Toàn Cầu DynamoDB) — Multi-Region Replication
- Neptune (Cơ Sở Dữ Liệu Đồ Thị) — Graph Queries với Gremlin/SPARQL
- DocumentDB (Cơ Sở Dữ Liệu Tài Liệu) — MongoDB Migration Path
- Timestream (Chuỗi Thời Gian) — IoT & Metrics Use Cases

### 📁 **11. Cost Optimization — Tối Ưu Chi Phí** (`11-cost-optimization/`)

- RDS Reserved Instances (Máy Chủ Đặt Trước) — 1-year vs 3-year Savings
- Aurora Serverless — Cost vs Performance Trade-off (Đánh Đổi Chi Phí vs Hiệu Năng)
- DynamoDB On-Demand vs Provisioned — Khi nào nên dùng gì
- Storage Tiering (Phân Tầng Lưu Trữ) — Aurora I/O-Optimized vs Standard
- Right-sizing (Định Cỡ Phù Hợp) Database Instances
- ElastiCache Reserved Nodes (Nút Đặt Trước ElastiCache) & Savings Plans

### 📁 **12. Interview Prep — Chuẩn Bị Phỏng Vấn** (`12-interview-prep/`)

- Top 20 câu hỏi phỏng vấn AWS Database thường gặp
- System Design Scenarios (Kịch Bản Thiết Kế Hệ Thống) với Database
- Trade-off Discussions (Thảo Luận Đánh Đổi) — SQL vs NoSQL, RDS vs DynamoDB
- Real-world Incident Stories (Câu Chuyện Sự Cố Thực Tế) theo phương pháp STAR
- Architecture Diagrams (Sơ Đồ Kiến Trúc) & Design Patterns (Mẫu Thiết Kế)

---

## 🎓 Theo Dịch Vụ AWS

### **Amazon RDS**

```
Điểm mạnh: Managed SQL, Multi-AZ, Read Replicas, PITR
Phù hợp: OLTP workloads, ứng dụng dùng SQL truyền thống
Covered in: 01-rds-fundamentals, 05-ha-backup, 06-security, 07-performance-tuning
```

### **Amazon Aurora**

```
Điểm mạnh: 5x faster than MySQL, 3x faster than PostgreSQL, Auto-healing storage
Phù hợp: High-traffic OLTP, cần HA tốt và global replication
Covered in: 02-aurora, 05-ha-backup, 10-advanced
```

### **Amazon DynamoDB**

```
Điểm mạnh: Serverless, single-digit millisecond latency, unlimited scale
Phù hợp: Key-value/document data, gaming, IoT, real-time apps
Covered in: 03-dynamodb, 07-performance-tuning, 10-advanced
```

### **Amazon ElastiCache**

```
Điểm mạnh: Sub-millisecond latency, Redis/Memcached compatible
Phù hợp: Session management, caching, pub/sub, leaderboards
Covered in: 04-elasticache
```

### **Amazon Redshift**

```
Điểm mạnh: Petabyte-scale analytics, columnar storage, S3 integration
Phù hợp: OLAP, data warehousing, BI analytics
Covered in: 10-advanced/redshift
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                          | Thư Mục                                                     | Độ Ưu Tiên    |
| ------------------------------- | ----------------------------------------------------------- | ------------- |
| Bắt đầu từ đây                  | [README.md](./README.md)                                    | Start here    |
| RDS cơ bản                      | [01-rds-fundamentals](./01-rds-fundamentals/)               | Bắt buộc      |
| Aurora deep dive                | [02-aurora](./02-aurora/)                                   | Bắt buộc      |
| DynamoDB patterns               | [03-dynamodb](./03-dynamodb/)                               | Bắt buộc      |
| Backup & HA                     | [05-ha-backup](./05-ha-backup/)                             | Bắt buộc      |
| Câu hỏi phỏng vấn               | [12-interview-prep](./12-interview-prep/)                   | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng

### Beginner — Mới Bắt Đầu (0-1 năm)

- [ ] Phân biệt các loại cơ sở dữ liệu AWS (SQL vs NoSQL)
- [ ] Hiểu RDS Multi-AZ và Read Replicas
- [ ] Biết cơ bản DynamoDB (Table, Partition Key, Sort Key)
- [ ] Cấu hình Automated Backup (Sao Lưu Tự Động)
- [ ] Bảo mật cơ bản với VPC và Security Groups

### Intermediate — Trung Cấp (1-3 năm)

- [ ] Thiết kế Aurora Cluster với Global Database
- [ ] DynamoDB access patterns & single-table design
- [ ] ElastiCache Redis replication & caching strategies
- [ ] RDS Performance Insights & slow query analysis
- [ ] RDS Proxy (Proxy RDS) cho connection pooling
- [ ] DMS (Database Migration Service) cho database migration

### Advanced — Nâng Cao (3-5+ năm)

- [ ] Multi-region active-active architecture (Kiến Trúc Đa Vùng Chủ-Chủ)
- [ ] DynamoDB Global Tables & conflict resolution (Giải Quyết Xung Đột)
- [ ] Redshift architecture & query optimization
- [ ] Capacity planning & cost modeling (Lập Kế Hoạch Năng Lực)
- [ ] Compliance frameworks (Khung Tuân Thủ) — HIPAA, PCI-DSS, GDPR
- [ ] Zero-downtime database migration strategies

---

## 🚀 Bắt Đầu Như Thế Nào

### Bước 1: Xác Định Mục Tiêu Học Tập

```
Chọn hướng phát triển:
- Generalist (Tổng Quát) — Biết tất cả dịch vụ AWS database
- Specialist (Chuyên Sâu) — Chuyên sâu RDS/Aurora hoặc DynamoDB
- Data Engineer (Kỹ Sư Dữ Liệu) — Redshift, Athena, data pipelines
- Solutions Architect (Kiến Trúc Sư Giải Pháp) — Lựa chọn DB cho hệ thống
```

### Bước 2: Thiết Lập Môi Trường Thực Hành

```bash
# Tạo RDS instance miễn phí (Free Tier — Tầng Miễn Phí)
# db.t3.micro: 750 giờ/tháng miễn phí

# Tạo DynamoDB table miễn phí
# 25 GB storage + 25 WCU + 25 RCU/tháng miễn phí

# Tạo ElastiCache (cache.t3.micro) trong VPC
```

### Bước 3: Học + Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Tạo thử tài nguyên trên AWS Console (30 phút)
3. Thực hành kỹ năng bằng hands-on lab (30-60 phút)
4. Xem lại checklist và ghi chú (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện theo phương pháp STAR:
- Situation  — Tình huống
- Task       — Nhiệm vụ
- Action     — Hành động đã thực hiện
- Result     — Kết quả đạt được
```

---

## 📖 Tài Liệu Tham Khảo

### Đọc Thiết Yếu

- **AWS Database Documentation** — Tài liệu chính thức AWS
- **"DynamoDB Book"** by Alex DeBrie — DynamoDB chuyên sâu
- **"Designing Data-Intensive Applications"** by Martin Kleppmann — Nền tảng hệ thống
- **AWS re:Invent Database Sessions** — Video học chuyên sâu

### Tài Liệu Chính Thức

- [Amazon RDS Documentation](https://docs.aws.amazon.com/rds/)
- [Amazon Aurora Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- [Amazon DynamoDB Documentation](https://docs.aws.amazon.com/dynamodb/)
- [Amazon ElastiCache Documentation](https://docs.aws.amazon.com/elasticache/)
- [Amazon Redshift Documentation](https://docs.aws.amazon.com/redshift/)

### Blog & Bài Viết Chất Lượng

- AWS Database Blog (blog.awsdba.com)
- The DynamoDB Book — alexdebrie.com
- AWS Architecture Center — aws.amazon.com/architecture
- AWS Well-Architected Framework — Database Pillar

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Chủ Đề

#### RDS & Aurora

- [ ] Phân biệt RDS Multi-AZ và Read Replica — khi nào dùng gì?
- [ ] Aurora khác RDS ở điểm gì? Tại sao nhanh hơn?
- [ ] Giải thích Aurora Serverless v2 — use case phù hợp?
- [ ] RPO và RTO là gì? Thiết kế DR strategy (Chiến Lược Khôi Phục Thảm Họa) cho RDS?

#### DynamoDB

- [ ] Partition Key (Khóa Phân Vùng) tốt là gì? Tại sao quan trọng?
- [ ] GSI vs LSI — khi nào dùng loại nào?
- [ ] Giải thích hot partition (Phân Vùng Nóng) và cách tránh
- [ ] DynamoDB vs RDS — khi nào chọn DynamoDB?

#### ElastiCache

- [ ] Redis vs Memcached — khác biệt chính và khi nào dùng gì?
- [ ] Lazy Loading vs Write-Through caching — trade-offs?
- [ ] Thiết kế session management với ElastiCache Redis

#### Thiết Kế Hệ Thống

- [ ] Thiết kế hệ thống e-commerce — lựa chọn database nào cho từng thành phần?
- [ ] Thiết kế leaderboard realtime với DynamoDB + ElastiCache
- [ ] Giải thích kiến trúc multi-region active-active database

Xem `12-interview-prep/` để có full Q&A guide.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc khi đảm nhận vị trí mới, kiểm tra:

- [ ] Có thể giải thích sự khác biệt giữa RDS, Aurora, DynamoDB không có ghi chú
- [ ] Có thể thiết kế backup strategy cho RPO/RTO đã cho
- [ ] Có thể phân tích slow query với RDS Performance Insights
- [ ] Có thể thiết kế DynamoDB table với access patterns phù hợp
- [ ] Có thể giải thích Multi-AZ vs Read Replica
- [ ] Biết khi nào nên dùng ElastiCache Redis vs Memcached
- [ ] Hiểu Aurora Global Database và use case
- [ ] Có thể thực hiện database migration với DMS
- [ ] Biết cách tối ưu chi phí với Reserved Instances và Right-sizing
- [ ] Hiểu compliance requirements (PCI-DSS, HIPAA, GDPR) trên AWS

---

## 📞 Công Cụ & Tài Nguyên

### Công Cụ AWS

- **AWS Console** — Giao Diện Quản Trị Web
- **AWS CLI** (Command Line Interface — Giao Diện Dòng Lệnh) — Quản lý via terminal
- **AWS CloudFormation** — Infrastructure as Code (Hạ Tầng Dưới Dạng Mã)
- **AWS CDK** (Cloud Development Kit — Bộ Công Cụ Phát Triển Đám Mây) — IaC với ngôn ngữ lập trình

### Công Cụ Bên Thứ Ba

- **DBeaver** — Universal Database Client (Client Cơ Sở Dữ Liệu Đa Năng)
- **NoSQLWorkbench** — DynamoDB visual design tool
- **Datadog / Grafana** — Database monitoring & alerting
- **Flyway / Liquibase** — Database schema migration tools

### Cộng Đồng

- AWS re:Post (forum AWS chính thức)
- r/aws (Reddit AWS Community)
- AWS Developers Slack
- Stack Overflow — tag `amazon-rds`, `dynamodb`, `amazon-aurora`

---

## 📋 Cách Sử Dụng Hướng Dẫn Này

### Cho Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành trên AWS Console (Free Tier nếu có thể)
4. Xây dựng portfolio project

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [12-interview-prep](./12-interview-prep/)
2. Nghiên cứu sâu dịch vụ liên quan đến vị trí mục tiêu
3. Chuẩn bị câu chuyện sự cố theo phương pháp STAR
4. Luyện tập giải thích concepts rõ ràng

### Cho Công Việc Thực Tế

1. Tham khảo [07-performance-tuning](./07-performance-tuning/) khi có vấn đề hiệu năng
2. Dùng [09-monitoring](./09-monitoring/) để thiết lập giám sát
3. Theo [06-security](./06-security/) để đảm bảo bảo mật
4. Dùng [08-migration](./08-migration/) khi cần di chuyển database

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này hoàn chỉnh
├─ 2️⃣  Chọn lộ trình học (Beginner/Intermediate/Advanced)
├─ 3️⃣  Bắt đầu với 01-rds-fundamentals/
├─ 4️⃣  Thiết lập tài khoản AWS Free Tier để thực hành
├─ 5️⃣  Hoàn thành bài tập thực hành cho từng chủ đề
├─ 6️⃣  Xây dựng portfolio project sử dụng nhiều AWS database services
└─ 7️⃣  Chuẩn bị phỏng vấn với 12-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
