# ⭐ STAR Stories — Mẫu Câu Chuyện Phỏng Vấn Behavioral

> 10 mẫu câu chuyện STAR (Situation — Task — Action — Result) về database incidents và achievements (Thành Tích) — giúp bạn trả lời tự tin câu hỏi kinh nghiệm trong phỏng vấn.

## Giới Thiệu STAR Framework

```
S — Situation (Tình Huống): Bối cảnh cụ thể, vấn đề gặp phải
T — Task (Nhiệm Vụ):        Trách nhiệm của bạn là gì
A — Action (Hành Động):     Bạn đã làm gì, từng bước cụ thể
R — Result (Kết Quả):       Số liệu cụ thể, impact thực tế
```

**Quan trọng:** Thay thế số liệu giả bằng con số thực từ kinh nghiệm của bạn. Các con số dưới đây chỉ mang tính minh họa.

---

## Mục Lục

1. [Database Performance — Xử lý Database Chậm](#story-1-xử-lý-database-chậm-lúc-cao-điểm)
2. [Connection Exhaustion — Hết Kết Nối Database](#story-2-xử-lý-connection-exhaustion)
3. [Database Migration — Zero Downtime](#story-3-migration-database-zero-downtime)
4. [Data Loss Recovery — Khôi Phục Dữ Liệu](#story-4-khôi-phục-dữ-liệu-bị-xóa-nhầm)
5. [Scaling — Xử Lý Traffic Spike](#story-5-xử-lý-traffic-spike-bất-ngờ)
6. [Security Incident — Bảo Mật](#story-6-phát-hiện-và-xử-lý-bảo-mật)
7. [Cost Optimization — Tối Ưu Chi Phí](#story-7-tối-ưu-chi-phí-database)
8. [DynamoDB Design — Thiết Kế Lại](#story-8-thiết-kế-lại-dynamodb-table)
9. [Multi-Region HA — Disaster Recovery](#story-9-thiết-lập-disaster-recovery)
10. [Monitoring Gap — Tạo Monitoring System](#story-10-xây-dựng-monitoring-từ-đầu)

---

## Story 1: Xử Lý Database Chậm Lúc Cao Điểm

**Câu hỏi dạng này:**
- "Kể về một lần bạn debug database performance issue"
- "Có bao giờ hệ thống bạn bị chậm? Bạn xử lý thế nào?"

---

**S — Situation:**
Trong một sprint, sau khi deploy feature mới cho trang product listing, hệ thống e-commerce của chúng tôi bắt đầu gặp vấn đề: response time trang chủ tăng từ 200ms lên 3-5 giây. Sự cố xảy ra vào giờ cao điểm buổi tối, ảnh hưởng đến khoảng 30% người dùng.

**T — Task:**
Với tư cách là backend engineer phụ trách database layer, tôi cần xác định nguyên nhân và khắc phục trong vòng 30 phút trước khi ảnh hưởng đến doanh thu.

**A — Action:**

*Bước 1 — Thu thập thông tin nhanh (5 phút):*
```
- Check CloudWatch: CPUUtilization đang ở 92%
- RDS Performance Insights: AAS (Average Active Sessions) = 45, vượt vCPU count (16)
- Top waits: "io/file/innodb/innodb_data_file" chiếm 78%
- Top SQL: một query mới trên bảng "products" với Full Table Scan
```

*Bước 2 — Xác định nguyên nhân gốc rễ (5 phút):*
```
- EXPLAIN cho query mới: type = ALL (Full Table Scan), rows = 2.3 triệu
- Phát hiện: feature mới có filter theo column "category_path" chưa có index
- Table products có 2.3 triệu rows, query này chạy mỗi page load
```

*Bước 3 — Giải pháp tức thời (10 phút):*
```
-- Tạo index không lock table (CREATE INDEX CONCURRENTLY)
CREATE INDEX CONCURRENTLY idx_products_category_path
ON products(category_path)
WHERE is_active = true;  -- Partial index (Chỉ Mục Một Phần)
```

*Bước 4 — Verify và monitor:*
```
- Sau index: query execution time từ 4.2s → 12ms
- CPUUtilization giảm từ 92% → 18% trong 3 phút
- Response time trở về < 200ms
```

*Bước 5 — Ngăn tái phát:*
```
- Thêm rule vào CI/CD: EXPLAIN plan check cho mọi query mới
- Thêm CloudWatch alarm khi CPUUtilization > 70%
- Thêm RDS Performance Insights dashboard vào team monitoring runbook
```

**R — Result:**
- Sự cố được giải quyết trong 18 phút
- Response time trở về bình thường (< 200ms)
- Implement được automated EXPLAIN check, ngăn được 3 similar issues trong 2 tháng tiếp theo
- Team adopted Performance Insights như standard monitoring tool

---

## Story 2: Xử Lý Connection Exhaustion

**Câu hỏi dạng này:**
- "Kể về một lần database của bạn không nhận được kết nối mới"
- "Bạn đã xử lý connection pool issue như thế nào?"

---

**S — Situation:**
Khi team migrate từ monolith (Ứng Dụng Đơn Khối) sang microservices (Dịch Vụ Vi Mô), mỗi service tạo connection pool riêng đến cùng RDS instance. Trong một buổi chiều, chúng tôi nhận được alert (Cảnh Báo): "Too many connections" — database từ chối kết nối mới, ảnh hưởng đến toàn bộ hệ thống.

**T — Task:**
Lead backend của nhóm, tôi phụ trách tìm giải pháp vừa giải quyết khủng hoảng hiện tại vừa ngăn tái phát trong kiến trúc microservices mới.

**A — Action:**

*Immediate fix (Khắc Phục Ngay):*
```
- Kill idle connections qua RDS console
- Giảm max_connections pool trong từng service từ 50 → 10
- Hệ thống ổn định sau 5 phút
```

*Root cause analysis (Phân Tích Nguyên Nhân):*
```
- RDS db.r5.xlarge: max_connections = 823
- 12 microservices × 50 connections = 600 connections thường xuyên
- Deploy mới tạo thêm instances → vượt giới hạn
- Problem: mỗi service không biết về total connection budget
```

*Long-term solution (Giải Pháp Dài Hạn) — RDS Proxy:*
```
1. Triển khai RDS Proxy (Lớp Trung Gian Kết Nối) trước RDS
2. RDS Proxy quản lý connection pool tập trung
3. Các services kết nối vào RDS Proxy thay vì RDS trực tiếp
4. RDS Proxy tái sử dụng connections, giảm total connections xuống ~50

Kết quả cấu hình:
  Trước: 12 services × 50 = 600 connections
  Sau:   RDS Proxy → RDS: 50 connections (tái sử dụng)
         12 services → RDS Proxy: unlimited (proxy handles multiplexing)
```

*Additional safeguard (Bảo Vệ Bổ Sung):*
```
- CloudWatch alarm: DatabaseConnections > 700 (85% of max)
- Alert team khi threshold approaching
- Document connection budget in Architecture Decision Record (ADR)
```

**R — Result:**
- Giải quyết connection exhaustion trong 5 phút
- RDS Proxy giảm actual connections từ 600 → 45 (-92%)
- Failover time giảm từ 120 giây → 40 giây (RDS Proxy keeps connections)
- Zero connection-related incidents trong 6 tháng tiếp theo

---

## Story 3: Migration Database Zero Downtime

**Câu hỏi dạng này:**
- "Bạn đã thực hiện database migration như thế nào?"
- "Kể về một lần bạn migrate production database không có downtime"

---

**S — Situation:**
Công ty cần migrate database từ on-premise (Tại Chỗ) MySQL 5.7 lên AWS Aurora MySQL 8.0. Database có 500GB data, ứng dụng phải hoạt động 24/7 với SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) 99.9% uptime.

**T — Task:**
Thiết kế và thực hiện migration plan với RPO (Recovery Point Objective) < 5 phút và RTO (Recovery Time Objective) < 10 phút.

**A — Action:**

*Phase 1 — Setup (2 tuần trước):*
```
1. Tạo Aurora MySQL cluster tại AWS
2. Cài đặt AWS DMS (Database Migration Service) Replication Instance
3. Initial load (Tải Lần Đầu) toàn bộ 500GB vào Aurora
4. Switch sang CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) mode
```

*Phase 2 — Parallel run (1 tuần):*
```
- DMS CDC đồng bộ mọi thay đổi từ source → Aurora (lag < 5 giây)
- Team validate (Xác Nhận) data consistency giữa hai systems
- Performance test (Kiểm Tra Hiệu Năng) trên Aurora với production-like load
- Rollback plan documented (Kế Hoạch Quay Lại Được Ghi Lại)
```

*Phase 3 — Cutover (Cắt Giảm) — Ngày thực hiện:*
```
Timeline (Dòng Thời Gian):
T+0:00 — Bật maintenance page cho write operations
T+0:02 — Verify DMS lag = 0 (no pending changes)
T+0:03 — Stop writes đến source DB
T+0:04 — Final check: row counts match trên cả hai DB
T+0:05 — Update application config để trỏ Aurora endpoint
T+0:06 — Enable writes trên Aurora
T+0:07 — Smoke test (Kiểm Tra Cơ Bản) pass
T+0:08 — Remove maintenance page, traffic live trên Aurora
Total cutover downtime: 8 phút
```

*Rollback triggers (Điều Kiện Quay Lại) — Nếu bất cứ điều nào:*
```
- Data inconsistency detected
- Performance degradation > 20%
- Error rate increase > 1%
→ Switch connection string back to source DB (2 phút)
```

**R — Result:**
- Migration hoàn thành với 8 phút planned downtime (dưới RTO 10 phút)
- Zero data loss
- Performance improvement: query latency giảm 35% trên Aurora
- Aurora failover time: 25 giây (so với 3-4 phút maintenance window trước đây)

---

## Story 4: Khôi Phục Dữ Liệu Bị Xóa Nhầm

**Câu hỏi dạng này:**
- "Có bao giờ bạn mất data không? Bạn xử lý thế nào?"
- "Kể về một tình huống khẩn cấp liên quan đến database"

---

**S — Situation:**
Một junior developer chạy nhầm DELETE statement (Lệnh Xóa) trên production database — xóa 45,000 order records của 3 ngày gần nhất do quên WHERE clause (Mệnh Đề Điều Kiện).

**T — Task:**
Khôi phục data bị mất càng nhanh càng tốt và không ảnh hưởng thêm đến production traffic.

**A — Action:**

*Bước 1 — Contain the damage (Khoanh Vùng Thiệt Hại):*
```
- Ngay lập tức revoke (Thu Hồi Quyền) EXECUTE permission của user đó
- Notify (Thông Báo) team lead và product manager
- Xác định time of incident: 14:32 UTC
```

*Bước 2 — Recovery plan:*
```
RDS PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm) availability:
- Transaction logs có từng 5 phút
- Restore point: 14:25 UTC (7 phút trước incident)

Plan: Restore vào separate instance, extract deleted records, re-insert
```

*Bước 3 — Execute recovery (Thực Hiện Khôi Phục):*
```
14:45 — Initiate PITR restore đến 14:25 UTC (vào new instance "recovery-db")
15:15 — Recovery instance available (30 phút)
15:20 — Extract deleted records từ recovery-db:
         SELECT * FROM orders
         WHERE created_at >= NOW() - INTERVAL 3 DAY
         INTO OUTFILE '/tmp/recovered_orders.csv'
15:25 — Validate (Xác Nhận) recovered data: 44,987 records (99.97% intact)
15:30 — Re-insert vào production db với status = 'recovered'
15:35 — Notify customer service team về affected orders
```

*Bước 4 — Prevention (Ngăn Chặn):*
```
- Implement: require confirmation phrase cho DELETE trên production
- Add: DELETE without WHERE clause blocked by SQL firewall rule
- Create: weekly "break glass" drill (Diễn Tập Khẩn Cấp) cho team
- Update: runbook với recovery procedures
```

**R — Result:**
- 99.97% data recovered (44,987/45,000 records) trong 63 phút
- 13 records không khôi phục được (orders tạo trong 7 phút window) — contacted customers manually
- Zero customer complaints về data loss
- Implemented prevention measures → zero similar incidents sau đó

---

## Story 5: Xử Lý Traffic Spike Bất Ngờ

**Câu hỏi dạng này:**
- "Kể về lần hệ thống bạn bị overload"
- "Bạn xử lý database scaling như thế nào?"

---

**S — Situation:**
Sản phẩm của chúng tôi được featured (Được Giới Thiệu) trên một tech blog lớn, traffic tăng đột biến 20x trong 30 phút. Database CPU đạt 100%, response time tăng từ 150ms lên 12 giây.

**T — Task:**
Giữ hệ thống hoạt động trong traffic spike và scale database để handle load mới.

**A — Action:**

*Tức thời — giảm database pressure:*
```
1. Enable aggressive caching (Bật Caching Mạnh Hơn):
   - Tăng TTL (Time-to-Live) của popular content từ 5 phút → 30 phút
   - Cache các trang landing page

2. Route reads sang Read Replicas:
   - Update application config để dùng replica endpoint
   - ~60% queries là reads → giảm 60% load trên primary

3. Graceful degradation (Xuống Cấp Nhẹ Nhàng):
   - Disable non-critical features: recommendations, analytics tracking
   - Reduce query complexity: bỏ tạm joins không cần thiết
```

*Sau 30 phút — scale database:*
```
- Add 2 Aurora Read Replicas (horizontal scale)
- Upgrade primary instance (vertical scale: r5.2xlarge → r5.4xlarge)
- Enable Aurora Auto Scaling cho Read Replicas cho future spikes
```

*Sau incident — cải thiện architecture:*
```
- Implement read/write splitting (Phân Tách Đọc/Ghi) chính thức
- Set up ElastiCache Redis cho product catalog cache layer
- Configure Aurora Auto Scaling policy:
  Target: CPUUtilization = 70%
  Min replicas: 1, Max replicas: 5
```

**R — Result:**
- Giữ được hệ thống hoạt động trong toàn bộ traffic spike
- Response time ổn định ở 300ms sau khi scale (vs 12 giây khi chưa xử lý)
- Retained (Giữ Lại) ~85% của extra traffic thay vì mất hết
- Sau khi implement auto-scaling: 2 traffic spikes tiếp theo handled tự động, zero manual intervention

---

## Story 6: Phát Hiện Và Xử Lý Bảo Mật

**Câu hỏi dạng này:**
- "Bạn có kinh nghiệm với database security không?"
- "Kể về lần bạn phát hiện security vulnerability trong hệ thống"

---

**S — Situation:**
Trong security audit định kỳ, tôi phát hiện RDS instance production đang expose port 3306 publicly (Công Khai) qua security group không đúng cấu hình — bất kỳ IP nào đều có thể connect.

**T — Task:**
Vá lỗ hổng bảo mật ngay lập tức mà không gây downtime và đánh giá xem có data breach (Rò Rỉ Dữ Liệu) không.

**A — Action:**

*Immediate security fix (Khắc Phục Bảo Mật Ngay):*
```
1. Restrict security group: 0.0.0.0/0:3306 → only app server security group
2. Enable RDS audit logging (Nhật Ký Kiểm Toán) ngay lập tức
3. Force rotation (Xoay Vòng Bắt Buộc) tất cả database credentials qua Secrets Manager
```

*Investigate potential breach (Điều Tra Vi Phạm Tiềm Ẩn):*
```
- Check CloudTrail (Lịch Sử CloudTrail): tìm unusual API calls
- Check RDS enhanced logs: có connection từ IPs lạ không?
- Check VPC Flow Logs: traffic pattern đến port 3306?
- Kết quả: không tìm thấy unauthorized access trong 30 ngày log
```

*Long-term hardening (Tăng Cường Bảo Mật Lâu Dài):*
```
1. Implement AWS Config Rule: "rds-instance-public-access-check"
   → Alert ngay khi RDS có public access
2. Enable AWS Security Hub: centralized security view
3. Move RDS vào private subnet (Mạng Con Riêng Tư) — không có internet route
4. Document security baseline trong Infrastructure as Code (Hạ Tầng Dưới Dạng Mã)
5. Quarterly security review checklist
```

**R — Result:**
- Lỗ hổng được vá trong 5 phút không downtime
- No data breach confirmed sau điều tra
- AWS Config rule ngăn được 2 misconfigurations tương tự trong 3 tháng tiếp theo
- Security posture score cải thiện trong Security Hub từ 45% → 89%

---

## Story 7: Tối Ưu Chi Phí Database

**Câu hỏi dạng này:**
- "Bạn đã bao giờ tối ưu chi phí hạ tầng chưa?"
- "Kể về lần bạn giảm chi phí mà không ảnh hưởng performance"

---

**S — Situation:**
AWS cost review cho thấy database chi phí tăng 40% trong Q3. Cần review và optimize mà không ảnh hưởng đến SLA.

**T — Task:**
Giảm database costs ít nhất 25% trong Q4.

**A — Action:**

*Cost analysis (Phân Tích Chi Phí):*
```
Breakdown chi phí:
- RDS On-Demand instances: 60% của total cost
- RDS storage: 20%
- DynamoDB On-Demand: 15%
- ElastiCache: 5%
```

*Optimization actions (Hành Động Tối Ưu):*

```
1. Reserved Instances cho RDS (tiết kiệm ~40%):
   - 3 production instances chạy 24/7
   - Mua 1-year Reserved Instances
   - Savings: $1,200/tháng

2. RDS Storage optimization:
   - Enable storage auto-scaling với max cap
   - Delete old snapshots (> 30 ngày) — chỉ giữ weekly
   - Savings: $300/tháng

3. DynamoDB: switch từ On-Demand → Provisioned + Auto Scaling:
   - Traffic ổn định → Provisioned rẻ hơn 70%
   - Configure Auto Scaling để handle spikes
   - Savings: $400/tháng

4. Right-sizing (Định Cỡ Phù Hợp) Dev/Test environments:
   - Dev: r5.2xlarge → r5.large (reduce 75%)
   - Test: chạy chỉ trong business hours, stop ngoài giờ
   - Savings: $600/tháng

5. DynamoDB TTL (Time-to-Live — Thời Gian Sống) cho old data:
   - Log/event tables: TTL = 90 ngày
   - Giảm storage 40%
```

**R — Result:**
- Total savings: $2,500/tháng (-31% so với baseline)
- Vượt target 25% cost reduction
- Zero performance degradation (Suy Giảm Hiệu Năng)
- Presented cost optimization framework cho team, adopted as quarterly practice

---

## Story 8: Thiết Kế Lại DynamoDB Table

**Câu hỏi dạng này:**
- "Bạn đã bao giờ thiết kế lại database schema không?"
- "Kể về một bài toán DynamoDB access pattern phức tạp"

---

**S — Situation:**
Ứng dụng quản lý dự án dùng DynamoDB với design ban đầu: một table per entity (users, projects, tasks). Sau 6 tháng, team gặp vấn đề: mỗi dashboard load cần 5-7 DynamoDB calls, latency tổng 800ms+.

**T — Task:**
Thiết kế lại DynamoDB schema để giảm dashboard load xuống còn 1-2 calls với latency < 100ms.

**A — Action:**

*Phân tích access patterns:*
```
Top 5 queries:
1. Get all projects for user → JOIN users + projects
2. Get all tasks for project → JOIN projects + tasks
3. Get task details + assignee info → JOIN tasks + users
4. Get user's active tasks → JOIN users + tasks + filter
5. Dashboard: all of the above
```

*Thiết kế lại với Single-Table Design (Thiết Kế Đơn Bảng):*
```
Entity prefix pattern (Mẫu Tiền Tố Thực Thể):
PK (Partition Key)   | SK (Sort Key)           | Data
─────────────────────────────────────────────────────────────
USER#user123         | PROFILE                 | user details
USER#user123         | PROJECT#proj456         | user-project relationship
USER#user123         | TASK#task789            | user's task assignment
PROJECT#proj456      | METADATA                | project details
PROJECT#proj456      | TASK#task789            | project's task
PROJECT#proj456      | MEMBER#user123          | project member
TASK#task789         | DETAILS                 | task details

GSI 1 (SK làm PK):
  → Query: "tất cả tasks của project X" = query PK=PROJECT#proj456, SK begins_with TASK#
  
GSI 2 (status + created_at):
  → Query: "active tasks của user X" = query PK=USER#user123, filter SK begins_with TASK#
```

*Migration strategy (Chiến Lược Di Chuyển):*
```
1. Deploy new table (không xóa old tables)
2. Backfill data từ old tables → new table
3. Update application để write vào cả hai tables
4. Gradually shift reads sang new table
5. After validation: cutover reads, remove old tables
```

**R — Result:**
- Dashboard load giảm từ 5-7 DynamoDB calls → 2 calls
- Latency giảm từ 800ms → 85ms (-89%)
- DynamoDB costs giảm 30% (ít reads hơn)
- Pattern được adopt làm standard cho các tables mới

---

## Story 9: Thiết Lập Disaster Recovery

**Câu hỏi dạng này:**
- "Hệ thống của bạn có DR plan không?"
- "Kể về lần bạn thiết kế high availability cho database"

---

**S — Situation:**
Sau một region-level outage (Gián Đoạn Toàn Vùng) khiến us-east-1 bị ảnh hưởng 4 giờ, ban lãnh đạo yêu cầu thiết lập DR plan với RTO < 15 phút. Hệ thống khi đó chỉ có Multi-AZ trong us-east-1.

**T — Task:**
Thiết kế và implement cross-region DR solution với budget được duyệt, đảm bảo RTO 15 phút và RPO 5 phút.

**A — Action:**

*Đánh giá options:*
```
Option 1: Cross-region Read Replica
  RTO: 15-30 phút (manual promote + DNS switch)
  RPO: < 1 phút
  Cost: +40% (read replica instance)

Option 2: Aurora Global Database
  RTO: < 1 phút (automatic failover)
  RPO: < 1 giây
  Cost: +80% (secondary region full instance + replication I/O)

Option 3: Automated backup cross-region
  RTO: 1-2 giờ (restore từ snapshot)
  RPO: 1-5 giờ
  Cost: +10% (snapshot storage only)

Decision: Option 1 (RTO 15-30 phút là đủ, optimize cost)
```

*Implementation:*
```
1. Create Aurora Read Replica tại us-west-2 (secondary region)
2. Terraform (Cơ Sở Hạ Tầng Dưới Dạng Mã) để auto-provision tất cả resources ở DR region
3. Create Route 53 (Dịch Vụ DNS) health check + failover routing
4. Write và test runbook (Tài Liệu Vận Hành):
   - Khi nào cần failover?
   - Steps to promote Read Replica
   - Update DNS, notify stakeholders
   - Rollback procedure

5. Test DR drill (Diễn Tập DR):
   - Simulate region failure mỗi quý
   - Track actual RTO achieved
```

*DR Drill Results (Kết Quả Diễn Tập):*
```
Drill 1: RTO = 28 phút (có bước thủ công chậm)
Drill 2: RTO = 19 phút (sau khi optimize runbook)
Drill 3: RTO = 12 phút (sau khi automate DNS switch)
```

**R — Result:**
- Achieved RTO 12 phút (< target 15 phút)
- RPO < 1 phút (Read Replica replication lag)
- Tự động hóa 70% failover steps → giảm human error
- Khi us-east-1 có partial outage lần sau, team failover trong 14 phút

---

## Story 10: Xây Dựng Monitoring Từ Đầu

**Câu hỏi dạng này:**
- "Team bạn monitor database như thế nào?"
- "Kể về lần bạn xây dựng observability (Khả Năng Quan Sát) cho hệ thống"

---

**S — Situation:**
Team joined dự án kế thừa: production database không có monitoring. Sự cố chỉ được phát hiện khi users report. Mục tiêu: xây dựng monitoring từ đầu.

**T — Task:**
Thiết lập comprehensive database monitoring (Giám Sát Cơ Sở Dữ Liệu Toàn Diện) trong 2 tuần sprint.

**A — Action:**

*Tuần 1 — Metrics & Alerts (Số Liệu & Cảnh Báo):*
```
CloudWatch Alarms (Cảnh Báo CloudWatch) cho RDS:
- CPUUtilization > 80% → WARNING; > 90% → CRITICAL
- FreeableMemory < 1GB → WARNING
- DatabaseConnections > 80% of max → WARNING
- ReadLatency + WriteLatency > 20ms → WARNING
- FreeStorageSpace < 20% → CRITICAL

CloudWatch Alarms cho DynamoDB:
- ConsumedReadCapacityUnits/ProvisionedReadCapacityUnits > 80%
- ThrottledRequests > 0 → WARNING
- SystemErrors > 0 → CRITICAL

ElastiCache Alarms:
- CacheHitRate < 85% → WARNING
- Evictions > 0 per minute → WARNING
- EngineCPUUtilization > 70% → WARNING
```

*Tuần 2 — Dashboards & Runbooks (Bảng Điều Khiển & Tài Liệu Vận Hành):*
```
CloudWatch Dashboard:
- Top-level: traffic, latency, error rates
- Database layer: RDS + DynamoDB + ElastiCache
- Drill-down: per-service, per-table metrics

Performance Insights Dashboard:
- AAS (Average Active Sessions) over time
- Top SQL by wait time
- Wait event breakdown

On-Call Runbook:
- Cho mỗi alert: triệu chứng, nguyên nhân thường gặp, action steps
- Links đến Performance Insights, CloudWatch
- Escalation path
```

**R — Result:**
- Mean Time to Detect (MTTD — Thời Gian Trung Bình Phát Hiện) incidents: từ 45 phút → 3 phút
- Mean Time to Resolve (MTTR — Thời Gian Trung Bình Giải Quyết): từ 3 giờ → 45 phút
- Phát hiện và fix 3 performance issues proactively (trước khi users report) trong tháng đầu
- Dashboard được adopt bởi toàn team, shared với leadership

---

## Hướng Dẫn Sử Dụng STAR Stories

### Customize Cho Kinh Nghiệm Thực Tế

1. **Thay số liệu:** Các con số trong template chỉ là minh họa. Thay bằng số thực từ kinh nghiệm của bạn.
2. **Thay công nghệ:** Nếu bạn dùng PostgreSQL thay MySQL, hoặc Redis Sentinel thay Cluster — điều chỉnh cho phù hợp.
3. **Thêm chi tiết kỹ thuật:** Interviewer technical thường muốn nghe lệnh/commands cụ thể bạn đã dùng.
4. **Honest về failures:** Nếu có bước sai trong quá trình xử lý, OK để mention — điều đó cho thấy bạn học từ mistakes.

### Tips Kể STAR Stories

```
✅ Bắt đầu với impact ngay: "Hệ thống bị chậm ảnh hưởng X% users"
✅ Nói về role cụ thể của bạn, không chỉ "chúng tôi"
✅ Có số liệu cụ thể: "giảm từ 3 giây xuống 150ms" tốt hơn "giảm nhiều"
✅ Mention cái bạn đã học được
✅ Kết thúc với preventive measures (Biện Pháp Phòng Ngừa) — interviewer muốn biết bạn ngăn tái phát

❌ Tránh kể story mà bạn không có vai trò rõ ràng
❌ Tránh story không có kết quả đo được
❌ Đừng kể quá dài (target 3-4 phút cho mỗi story)
❌ Đừng đổ lỗi hoàn toàn cho người khác
```

### Câu Hỏi Để Chuẩn Bị Story Cá Nhân

```
1. Database incident lớn nhất bạn đã xử lý là gì?
2. Bạn đã tối ưu query/database nào giúp cải thiện performance đáng kể?
3. Bạn đã migrate database nào chưa? Khó khăn gì?
4. Quyết định kiến trúc database nào bạn tự hào nhất?
5. Lần nào bạn phải học nhanh để giải quyết database problem?
6. Bạn đã tiết kiệm chi phí database như thế nào?
```
