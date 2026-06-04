# Migration Strategies — Chiến Lược Di Chuyển Database

> Lựa chọn chiến lược di chuyển đúng đắn là yếu tố quyết định thành công. Chiến lược sai có thể dẫn đến chi phí cao hơn, rủi ro lớn hơn, hoặc bỏ lỡ cơ hội tối ưu hóa. Tài liệu này phân tích 3 chiến lược chính — Lift-and-Shift (Nâng và Chuyển), Re-platform (Tái Nền Tảng), và Re-architect (Tái Kiến Trúc) — với trade-offs (đánh đổi) và decision framework (khung ra quyết định) rõ ràng.

## 📚 Mục Lục

1. [Tổng Quan 6R](#tổng-quan-6r)
2. [Lift-and-Shift (Rehost)](#lift-and-shift-rehost)
3. [Re-platform](#re-platform)
4. [Re-architect (Refactor)](#re-architect-refactor)
5. [Decision Framework](#decision-framework)
6. [Migration Paths Phổ Biến](#migration-paths-phổ-biến)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Tổng Quan 6R

AWS giới thiệu framework 6R để phân loại các chiến lược cloud migration:

```
6R Migration Strategies:

  Retire    — Tắt bỏ hệ thống không còn cần thiết
              Ví dụ: Hệ thống legacy không còn user nào dùng
  
  Retain    — Giữ nguyên on-premises, chưa migrate
              Ví dụ: Compliance yêu cầu data on-prem, sắp EOL
  
  Rehost    — Lift-and-shift: chuyển nguyên xi lên EC2
              Ví dụ: MySQL on-prem → MySQL trên EC2
  
  Replatform — Migrate lên managed service với thay đổi tối thiểu
               Ví dụ: MySQL on EC2 → MySQL on RDS
  
  Refactor  — Re-architect: thiết kế lại tận dụng cloud-native
               Ví dụ: Oracle monolith → Aurora PostgreSQL microservices
  
  Repurchase — Mua SaaS thay thế
               Ví dụ: Self-managed Redis → Redis Cloud
```

Với **database migration**, 3 chiến lược thực tế nhất là Rehost, Replatform, và Refactor.

---

## 🚛 Lift-and-Shift (Rehost — Tái Lưu Trữ)

### Định Nghĩa

Lift-and-Shift (Nâng và Chuyển) nghĩa là di chuyển database lên AWS EC2 mà **không thay đổi engine, schema, hay cách ứng dụng kết nối**. Database vẫn self-managed như on-premises.

### Kiến Trúc

```
On-Premises:                    AWS (Sau Lift-and-Shift):
──────────────────              ──────────────────────────────
┌─────────────────┐             ┌──────────────────────────────┐
│  App Server     │             │  VPC                         │
│  ┌───────────┐  │             │  ┌──────────┐  ┌──────────┐  │
│  │  App Code │  │             │  │  EC2     │  │  EC2     │  │
│  └───────────┘  │             │  │  App     │  │  MySQL   │  │
│       │         │    ────▶    │  │  Server  │  │  (EBS)   │  │
│  ┌───────────┐  │             │  └──────────┘  └──────────┘  │
│  │  MySQL    │  │             │       └──────────────┘        │
│  │  5.7      │  │             │                              │
│  └───────────┘  │             └──────────────────────────────┘
└─────────────────┘
```

### Khi Nào Dùng Lift-and-Shift?

```
✅ Phù hợp khi:
  - Deadline migration nhanh (< 1 tháng)
  - Team không có thời gian test re-platform/re-architect
  - Database có nhiều stored procedures phức tạp
  - Muốn giảm rủi ro — migrate minimal trước, optimize sau
  - Compliance/regulatory yêu cầu kiểm soát OS level
  - License Oracle/SQL Server đã trả tiền (BYOL — Bring Your Own License)

❌ Không phù hợp khi:
  - Muốn giảm operational overhead (vẫn phải tự patch, backup, HA setup)
  - Muốn tận dụng cloud-native features (Aurora auto-scaling, DynamoDB serverless)
  - Database lớn cần elasticity cao
```

### Trade-offs

| Ưu Điểm                                      | Nhược Điểm                                       |
| -------------------------------------------- | ------------------------------------------------- |
| Nhanh nhất — ít rủi ro nhất                  | Vẫn phải tự quản lý: patching, backup, HA         |
| Ít thay đổi code ứng dụng                    | Không tận dụng managed service features           |
| Dễ rollback — chuyển IP ngược lại on-prem   | Chi phí có thể không tối ưu (EC2 always-on)       |
| Tương thích 100% (same engine, same version) | Khó scale ngang (cần thêm sharding thủ công)      |

### Chi Phí So Sánh Với Re-platform

```
Lift-and-Shift (MySQL trên EC2 r5.4xlarge):
  EC2 r5.4xlarge:  ~$0.80/giờ = $576/tháng
  EBS gp3 2TB:     ~$160/tháng
  Snapshot backup: ~$50/tháng
  Total:           ~$786/tháng + ops time

Re-platform (MySQL trên RDS db.r5.4xlarge Multi-AZ):
  RDS Multi-AZ:    ~$1.20/giờ = $864/tháng
  Storage 2TB:     ~$230/tháng
  Automated backup: Included
  Total:           ~$1,094/tháng, KHÔNG cần ops time

→ Re-platform đắt hơn ~40% về infrastructure cost
→ Nhưng tiết kiệm ops time (backup, patching, failover setup)
→ Tổng TCO (Total Cost of Ownership — Tổng Chi Phí Sở Hữu) thường Re-platform rẻ hơn
```

---

## 🔄 Re-platform (Tái Nền Tảng)

### Định Nghĩa

Re-platform nghĩa là chuyển database lên **managed service** (dịch vụ có quản lý) của AWS với **ít hoặc không thay đổi code ứng dụng**. Engine database giữ nguyên, chỉ thay đổi cách deploy.

### Ví Dụ Re-platform

```
MySQL on-premises     → MySQL on RDS
MySQL on EC2          → MySQL on RDS
PostgreSQL on-prem    → PostgreSQL on RDS / Aurora PostgreSQL
                      (Aurora PostgreSQL tương thích hoàn toàn với PostgreSQL)
MySQL on EC2          → Aurora MySQL
                      (Aurora MySQL tương thích với MySQL 5.7/8.0)
Redis on EC2          → Amazon ElastiCache for Redis
MongoDB on EC2        → Amazon DocumentDB (tương thích MongoDB API)
```

### Kiến Trúc Aurora MySQL vs MySQL on EC2

```
MySQL on EC2 (Lift-and-Shift):         Aurora MySQL (Re-platform):
──────────────────────────────         ─────────────────────────────────
┌──────────────────────────┐           ┌────────────────────────────────┐
│  Primary EC2             │           │  Aurora Cluster                │
│  ┌──────────────────┐    │           │  ┌─────────┐  ┌─────────┐     │
│  │  MySQL 8.0       │    │           │  │Writer   │  │Reader   │     │
│  │  EBS gp3 2TB     │    │           │  │Instance │  │Instance │     │
│  └──────────────────┘    │           │  └─────────┘  └─────────┘     │
│                          │           │        ↕              ↕        │
│  Replica EC2 (thủ công): │           │  ┌──────────────────────────┐  │
│  ┌──────────────────┐    │           │  │  Aurora Shared Storage   │  │
│  │  MySQL 8.0       │    │           │  │  (6 copies, 3 AZs)       │  │
│  │  Binlog Replica  │    │           │  └──────────────────────────┘  │
│  └──────────────────┘    │           └────────────────────────────────┘
└──────────────────────────┘
                                       Managed: backup, patching, failover
Managed by: bạn                        Managed by: AWS
```

### Khi Nào Dùng Re-platform?

```
✅ Phù hợp khi:
  - Muốn giảm operational overhead ngay lập tức
  - Application đã dùng standard SQL — không có Oracle-specific syntax
  - Muốn Multi-AZ, automated backup, monitoring mà không tự setup
  - Muốn scale read replicas dễ dàng
  - Team nhỏ, không có DBA chuyên biệt

❌ Không phù hợp khi:
  - Database có nhiều Oracle/SQL Server specific features không tương thích
  - Cần customization OS-level (cài extension đặc biệt)
  - Cost là priority tuyệt đối (EC2 Reserved vẫn rẻ hơn RDS Reserved)
```

### Validation Khi Re-platform

Dù cùng engine, vẫn có thể có minor incompatibilities:

```
MySQL → Aurora MySQL:
  ✓ SQL syntax tương thích (MySQL 5.7/8.0)
  ✓ Application drivers không đổi
  ✗ Một số system variables khác (global/session)
  ✗ Không hỗ trợ MyISAM storage engine (chỉ InnoDB)
  ✗ Spatial functions một số khác
  → Chạy test suite đầy đủ trước cutover

PostgreSQL → Aurora PostgreSQL:
  ✓ SQL tương thích
  ✓ Extensions phần lớn được hỗ trợ
  ✗ Một số extensions không có trên Aurora (pg_partman, TimescaleDB)
  ✗ PostgreSQL version phải khớp major version
  → Kiểm tra danh sách supported extensions
```

---

## 🏗️ Re-architect (Refactor — Tái Kiến Trúc)

### Định Nghĩa

Re-architect (còn gọi là Refactor — Tái Cấu Trúc) là thiết kế lại database và đôi khi cả ứng dụng để tận dụng tối đa cloud-native capabilities. Thường đi kèm thay đổi engine (heterogeneous migration) và thay đổi data model.

### Ví Dụ Re-architect

```
Oracle OLTP monolith → Aurora PostgreSQL + microservices
MySQL + Redis → DynamoDB (single-table design)
MySQL reporting → Aurora + Redshift (OLTP + OLAP tách biệt)
Self-managed Elasticsearch → Amazon OpenSearch Service
```

### Use Case: Oracle → Aurora PostgreSQL

```
Trước (Oracle):                   Sau (Aurora PostgreSQL):
────────────────────────          ──────────────────────────────────
Oracle 19c Enterprise             Aurora PostgreSQL 15
  - 48 vCPU                         - Auto-scaling (2-128 vCPU)
  - 256 GB RAM                       - Storage auto-scale
  - $300,000/năm license             - ~$20,000-50,000/năm
  - DBA team 3 người                 - 1 DBA phần thời gian
  
  500+ stored procedures             ~50 stored procedures còn lại
  (nhiều logic trong DB)             (logic chuyển lên application)
  
  Tight coupling                     Loose coupling
  (app gọi SP trực tiếp)            (app gọi REST API)
```

### Use Case: MySQL → DynamoDB (Re-architect Sang NoSQL)

```
Khi nào phù hợp:
  - Access pattern đơn giản: get by user_id, get by order_id
  - Cần scale đến hàng tỷ records
  - Latency yêu cầu single-digit millisecond
  - Schema flexible (attributes thay đổi theo loại item)

MySQL schema:
  CREATE TABLE user_sessions (
    session_id VARCHAR(36) PRIMARY KEY,
    user_id    INT,
    data       JSON,
    expires_at DATETIME,
    created_at DATETIME
  );

DynamoDB equivalent:
  Table: UserSessions
  PK: session_id (String)
  Attributes: user_id, data (Map), expires_at (TTL)
  
  GSI: UserIdIndex (PK: user_id) → "Get all sessions for user"
  TTL attribute: expires_at → auto-delete expired sessions

Kết quả:
  - Không cần manage DB server
  - Auto-scale đến hàng triệu concurrent users
  - TTL tự xóa sessions hết hạn
  - Cost theo actual usage (on-demand mode)
```

### Khi Nào Dùng Re-architect?

```
✅ Phù hợp khi:
  - Oracle/SQL Server license quá đắt (ROI rõ ràng)
  - Application cần scale đột biến (Black Friday, viral events)
  - Team muốn modernize kỹ thuật nợ (technical debt)
  - Database schema phức tạp không tận dụng tốt engine hiện tại
  - Roadmap đã có time budget cho migration lớn (3-6 tháng)

❌ Không phù hợp khi:
  - Timeline gấp (< 2 tháng)
  - Team không có kinh nghiệm với engine đích
  - Hệ thống critical, risk tolerance thấp
  - Application phụ thuộc nặng vào vendor-specific features
```

---

## 🧭 Decision Framework

### Câu Hỏi Để Chọn Chiến Lược

```
1. Timeline (Thời Hạn):
   < 1 tháng     → Lift-and-Shift
   1-3 tháng     → Re-platform
   3-6+ tháng    → Re-architect

2. Risk Tolerance (Chịu Đựng Rủi Ro):
   Thấp          → Lift-and-Shift
   Trung          → Re-platform  
   Cao            → Re-architect

3. ROI (Return on Investment — Lợi Nhuận Đầu Tư):
   Cần nhanh     → Lift-and-Shift (ít benefit dài hạn)
   Balance        → Re-platform
   Long-term      → Re-architect (cao nhất nhưng tốn công nhất)

4. Team Capability (Năng Lực Nhóm):
   Ít kinh nghiệm cloud   → Lift-and-Shift / Re-platform
   Kinh nghiệm target DB  → Re-architect
   
5. Database Size & Complexity:
   < 100 GB, ít SP        → Bất kỳ chiến lược nào
   > 1 TB, nhiều SP       → Lift-and-Shift hoặc Re-platform
   Oracle packages/SP     → Re-platform (Aurora) hoặc Re-arch với SCT
```

### Decision Tree (Cây Quyết Định)

```
START
  │
  ▼
Cùng engine? (MySQL→MySQL, PG→PG)
  │ YES                          │ NO (Oracle→PG, SQL Server→MySQL)
  ▼                              ▼
Muốn managed service?     Có budget 3+ tháng?
  │ YES       │ NO           │ YES          │ NO
  ▼           ▼              ▼              ▼
Re-platform Lift-and-  Re-architect    Xem xét lại
(RDS/Aurora) Shift(EC2)  (với SCT)      timeline
```

### Ma Trận So Sánh

| Tiêu Chí              | Lift-and-Shift | Re-platform    | Re-architect   |
| --------------------- | -------------- | -------------- | -------------- |
| **Thời Gian**         | Ngắn nhất      | Trung bình     | Dài nhất       |
| **Rủi Ro**            | Thấp nhất      | Thấp–Trung     | Cao nhất       |
| **Chi Phí Ban Đầu**   | Thấp nhất      | Trung bình     | Cao nhất       |
| **Chi Phí Vận Hành**  | Cao nhất       | Trung bình     | Thấp nhất      |
| **Operational Burden**| Cao (self-mgd) | Thấp (managed) | Thấp (managed) |
| **Cloud Benefits**    | Ít nhất        | Trung bình     | Nhiều nhất     |
| **Khả Năng Scale**    | Giới hạn       | Tốt            | Tốt nhất       |
| **Code Change**       | Không          | Ít hoặc không  | Nhiều          |

---

## 🛤️ Migration Paths Phổ Biến

### Path 1: On-Premises MySQL → Aurora MySQL (Re-platform)

```
Bước 1: Assessment (1 tuần)
  - Kiểm tra MySQL version, storage engine (InnoDB only?)
  - Kiểm tra custom plugins, fulltext search
  - Load test để biết instance size cần thiết

Bước 2: Target Setup (2 ngày)
  - Tạo Aurora MySQL cluster
  - Config parameter groups
  - Setup security groups, VPC

Bước 3: DMS Migration (1-2 tuần)
  - Full Load + CDC
  - Validate data

Bước 4: Application Update (3-5 ngày)
  - Cập nhật connection string
  - Test integration
  
Bước 5: Cutover (2-4 giờ maintenance window)

Tổng: 3-4 tuần
Risk: Thấp
```

### Path 2: Oracle → Aurora PostgreSQL (Re-architect)

```
Bước 1: Assessment với SCT (1 tuần)
  - Chạy SCT report
  - Tính toán % objects cần xử lý thủ công
  - Estimate effort

Bước 2: Schema Conversion (3-4 tuần)
  - SCT automatic conversion
  - Manual fixes cho stored procedures, triggers
  - Code review
  
Bước 3: Application Changes (2-4 tuần)
  - Cập nhật SQL dialect (Oracle → PostgreSQL)
  - Cập nhật ORM config
  - Xử lý Oracle-specific features (ROWNUM, DUAL table, TO_DATE)

Bước 4: Testing (2-3 tuần)
  - Unit tests
  - Integration tests
  - Performance benchmarking

Bước 5: DMS Migration (1-2 tuần)
  - Full Load + CDC
  - Validate data

Bước 6: Cutover (4-8 giờ maintenance window)

Tổng: 10-14 tuần
Risk: Cao (nhiều thay đổi)
ROI: Rất cao (tiết kiệm Oracle license)
```

### Path 3: MySQL → DynamoDB (Re-architect sang NoSQL)

```
Quyết định phù hợp khi:
  - Table hiện tại chủ yếu là key-value lookups
  - Không có complex JOINs
  - Cần global scale

Các bước đặc biệt:
  1. Model DynamoDB table design mới (access patterns first)
  2. Viết migration script (không dùng DMS — DMS không hỗ trợ MySQL → DynamoDB tốt)
  3. Dual-write trong giai đoạn transition
  4. Gradual traffic shift (chuyển traffic dần dần)

Timeline: 6-12 tuần (bao gồm application refactor)
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Phân biệt Lift-and-Shift, Re-platform, và Re-architect trong database migration?**

> Lift-and-Shift (Rehost) chuyển database nguyên xi lên EC2, không thay đổi engine, nhanh và ít rủi ro nhưng vẫn phải tự quản lý. Re-platform chuyển lên managed service như RDS hoặc Aurora với cùng engine, giảm operational burden và tận dụng managed features như automated backup, Multi-AZ. Re-architect thiết kế lại hoàn toàn để tận dụng cloud-native capabilities, ví dụ Oracle sang Aurora PostgreSQL để tiết kiệm license, tốn công nhất nhưng ROI cao nhất dài hạn.

**Q: Khi nào nên chọn Re-platform thay vì Re-architect?**

> Re-platform phù hợp khi cần migrated nhanh (1-3 tháng), risk tolerance thấp, application đã dùng standard SQL không vendor-specific, hoặc team không có kinh nghiệm với engine đích. Re-architect phù hợp khi có ROI rõ ràng như tiết kiệm Oracle license, timeline đủ dài (3-6 tháng), team có capability, và cần tận dụng cloud-native scale.

**Q: Bạn đang migrate Oracle 19c sang Aurora PostgreSQL. Những Oracle-specific features nào thường gây vấn đề nhất?**

> Những thứ khó nhất: (1) Oracle Packages — PostgreSQL không có concept packages, phải tách thành individual functions/procedures; (2) CONNECT BY cho hierarchical queries — phải viết lại bằng WITH RECURSIVE; (3) Oracle DATE chứa cả time component trong khi PostgreSQL DATE không có — phải đổi sang TIMESTAMP; (4) ROWNUM — phải dùng ROW_NUMBER() window function hoặc LIMIT; (5) Oracle implicit type coercions khác PostgreSQL strict typing. SCT xử lý nhiều cái tự động nhưng packages và complex PL/SQL thường cần viết lại thủ công.

---

**Liên Kết:**
- [1-dms-overview.md](./1-dms-overview.md) — AWS DMS chi tiết
- [2-sct.md](./2-sct.md) — Schema Conversion Tool
- [3-cdc-online-migration.md](./3-cdc-online-migration.md) — Zero-downtime CDC
- [5-cutover-runbook.md](./5-cutover-runbook.md) — Cutover planning

**Cập Nhật Lần Cuối:** 2026-05-15
