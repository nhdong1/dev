# AWS Migration — Top 20 Câu Hỏi Phỏng Vấn & Câu Trả Lời Mẫu

> Bộ câu hỏi được chắt lọc từ các cuộc phỏng vấn thực tế cho vị trí **Cloud Migration Engineer**, **Solutions Architect**, và **DBA** tại các công ty sử dụng AWS. Mỗi câu trả lời mẫu được thiết kế để **trả lời trong 2-4 phút**.

## 📚 Mục Lục

- [Nhóm 1: Chiến Lược Migration (Q1–Q5)](#nhóm-1-chiến-lược-migration)
- [Nhóm 2: Di Chuyển Cơ Sở Dữ Liệu (Q6–Q10)](#nhóm-2-di-chuyển-cơ-sở-dữ-liệu)
- [Nhóm 3: Truyền Tải Dữ Liệu & Snow Family (Q11–Q14)](#nhóm-3-truyền-tải-dữ-liệu--snow-family)
- [Nhóm 4: Di Chuyển Ứng Dụng (Q15–Q17)](#nhóm-4-di-chuyển-ứng-dụng)
- [Nhóm 5: Vận Hành & Rủi Ro (Q18–Q20)](#nhóm-5-vận-hành--rủi-ro)

---

## Nhóm 1: Chiến Lược Migration

---

### Q1: Giải thích 7R Migration Strategies. Khi nào dùng chiến lược nào?

**Tại sao hỏi:** Câu hỏi nền tảng nhất — 100% interviews cho vị trí cloud migration hỏi câu này.

**Câu trả lời mẫu:**

7Rs là framework (khung phân loại) để phân loại cách di chuyển từng workload lên AWS. Từ ít nỗ lực nhất đến nhiều nỗ lực nhất:

```
7R STRATEGIES — XẾP THEO MỨC ĐỘ TRANSFORMATION (BIẾN ĐỔI):

1. RETIRE (Loại Bỏ)
   → Tắt ứng dụng không còn cần thiết
   → Ví dụ: Hệ thống báo cáo legacy không ai dùng sau khi có BI mới
   → Tiết kiệm 10-20% chi phí ngay lập tức

2. RETAIN (Giữ Nguyên / Keep On-Premises)
   → Giữ lại on-premises vì lý do compliance, latency, hoặc chưa sẵn sàng
   → Ví dụ: Database chứa PII (Personally Identifiable Information — Thông Tin Nhận Dạng Cá Nhân) phải ở datacenter nội địa theo luật
   → Xem xét lại sau 6-12 tháng

3. REHOST (Lift-and-Shift — Nâng Và Chuyển)
   → Di chuyển VM nguyên trạng lên EC2, không sửa code
   → Công cụ: AWS MGN (Application Migration Service)
   → Ví dụ: 200 web servers Java cần di chuyển nhanh trong 3 tháng
   → Nhanh nhất, rủi ro thấp nhất, nhưng chưa tối ưu chi phí cloud

4. RELOCATE (Di Chuyển Sang Cloud Khác / VMware → AWS)
   → Di chuyển VMware vCenter environment lên AWS VMware Cloud
   → Không cần chuyển đổi OS hay application
   → Ít phổ biến, dành cho tổ chức phụ thuộc nặng VMware

5. REPURCHASE (Replace — Thay Bằng SaaS)
   → Bỏ application cũ, mua SaaS (Software as a Service) thay thế
   → Ví dụ: CRM on-premises → Salesforce; Email server → Microsoft 365
   → Không phải "migrate" theo nghĩa truyền thống

6. REPLATFORM (Lift-Tinker-and-Shift — Nâng Tinh Chỉnh Rồi Chuyển)
   → Di chuyển với một vài thay đổi để tận dụng cloud, không viết lại
   → Ví dụ: MySQL on EC2 → RDS MySQL (managed); Tomcat → Elastic Beanstalk
   → Tiết kiệm vận hành, cần ít nỗ lực hơn Refactor

7. REFACTOR / RE-ARCHITECT (Tái Kiến Trúc)
   → Viết lại hoặc thiết kế lại từ đầu theo kiến trúc cloud-native
   → Ví dụ: Monolith → Microservices trên ECS/EKS; On-premises DB → Aurora Serverless
   → Tốn nhiều thời gian và công sức nhất, nhưng lợi ích dài hạn cao nhất
```

**Cách chọn chiến lược:**

```
→ Deadline gấp, không thể sửa code?         → REHOST
→ Ứng dụng không còn dùng?                  → RETIRE
→ DB phụ thuộc compliance nội địa?           → RETAIN
→ Muốn tận dụng managed service (RDS, EKS)?  → REPLATFORM
→ Muốn cloud-native, scale tốt, dài hạn?     → REFACTOR
→ Phần mềm có SaaS tương đương?              → REPURCHASE
```

**Điểm thêm khi trả lời:** Trên thực tế, một dự án enterprise thường dùng *hỗn hợp nhiều Rs* cho các workload khác nhau. Ví dụ: 60% Rehost, 30% Replatform, 10% Retire.

---

### Q2: Ba giai đoạn migration của AWS là gì? Mỗi giai đoạn làm gì?

**Câu trả lời mẫu:**

AWS định nghĩa 3 giai đoạn di chuyển cloud:

```
GIAI ĐOẠN 1: ASSESS (Đánh Giá)
─────────────────────────────────
Mục tiêu: Hiểu rõ hạ tầng hiện tại và xác định business case (lý do kinh doanh)

Hoạt động chính:
  - Chạy AWS Application Discovery Service (ADS) để inventory (kiểm kê) servers
  - Phân tích dependency mapping (bản đồ phụ thuộc giữa các ứng dụng)
  - Chạy AWS Migration Evaluator để tính TCO (Total Cost of Ownership — Tổng Chi Phí Sở Hữu)
  - Phân loại workloads theo 7Rs
  - Xác định migration waves (đợt di chuyển)

Kết quả: Business case + migration portfolio (danh mục di chuyển)
Thời gian: 2-8 tuần

GIAI ĐOẠN 2: MOBILIZE (Chuẩn Bị)
──────────────────────────────────
Mục tiêu: Xây dựng nền tảng cloud và chuẩn bị đội ngũ

Hoạt động chính:
  - Thiết kế landing zone (vùng hạ cánh — môi trường AWS chuẩn hóa)
  - Thiết lập network connectivity: Direct Connect hoặc VPN
  - Training (đào tạo) đội ngũ về AWS services
  - Thực hiện Proof of Concept (PoC — Bằng Chứng Khái Niệm) cho workload phức tạp
  - Lập migration runbooks (tài liệu hướng dẫn thực hiện từng bước)

Kết quả: Landing zone sẵn sàng, đội ngũ có kỹ năng
Thời gian: 2-6 tháng

GIAI ĐOẠN 3: MIGRATE & MODERNIZE (Di Chuyển & Hiện Đại Hóa)
──────────────────────────────────────────────────────────────
Mục tiêu: Thực thi migration theo waves (đợt), rồi optimize (tối ưu)

Hoạt động chính:
  - Di chuyển theo waves từ thấp đến cao rủi ro
  - Thực hiện test cutover (kiểm thử chuyển đổi) trước production cutover
  - Validation (kiểm tra tính toàn vẹn) dữ liệu sau migration
  - Rightsizing (điều chỉnh kích thước tài nguyên) để tối ưu chi phí
  - Modernize workloads phù hợp (containerize, managed DB, v.v.)

Kết quả: Workloads chạy trên AWS, chi phí tối ưu
Thời gian: 6-24 tháng tùy quy mô
```

---

### Q3: Sự khác biệt giữa RPO và RTO trong ngữ cảnh migration là gì?

**Câu trả lời mẫu:**

```
RPO — Recovery Point Objective (Mục Tiêu Điểm Phục Hồi)
→ "Mất tối đa bao nhiêu dữ liệu là chấp nhận được?"
→ Ví dụ: RPO = 1 giờ → nếu hệ thống crash, chấp nhận mất tối đa 1 giờ dữ liệu
→ Kỹ thuật đạt RPO thấp: CDC liên tục (DMS + CDC), synchronous replication
→ RPO = 0 → zero data loss → đắt nhất, cần synchronous replication

RTO — Recovery Time Objective (Mục Tiêu Thời Gian Phục Hồi)
→ "Hệ thống có thể ngừng hoạt động tối đa bao lâu?"
→ Ví dụ: RTO = 4 giờ → sau sự cố, phải restore (khôi phục) trong 4 giờ
→ RTO thấp → cần active-active hoặc warm standby

Liên quan đến migration:
  - Giai đoạn cutover (chuyển đổi) = planned downtime (ngừng hoạt động có kế hoạch)
  - Cần thiết kế để downtime < RTO của business
  - Với DMS + CDC: Full load (tải đầy đủ) rồi CDC → cutover window chỉ vài phút
  - Với MGN: Test cutover trước → cutover chính thức thường < 1 giờ
```

**Ví dụ thực tế:**

```
Ngân hàng: RPO = 0, RTO = 15 phút → Cần synchronous replication + hot standby
E-commerce: RPO = 15 phút, RTO = 1 giờ → DMS CDC + Pilot Light strategy
Internal tool: RPO = 24 giờ, RTO = 8 giờ → Nightly backup đủ dùng
```

---

### Q4: Làm thế nào bạn tính toán và trình bày TCO (Total Cost of Ownership) cho dự án migration?

**Câu trả lời mẫu:**

```
TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu)
= Chi phí on-premises hiện tại SO SÁNH VỚI chi phí AWS dự kiến

CHI PHÍ ON-PREMISES CẦN TÍNH:
1. Hardware (Phần Cứng):
   - Server purchase/lease (mua/thuê server): $X/năm
   - Storage arrays (thiết bị lưu trữ): $X/năm
   - Network equipment (thiết bị mạng): $X/năm

2. Facility (Cơ Sở Vật Chất):
   - Datacenter space (mặt bằng): $X/m²/năm
   - Power & cooling (điện & làm lạnh): $X/kWh/năm
   - Physical security (bảo mật vật lý): $X/năm

3. Software (Phần Mềm):
   - OS licensing (bản quyền hệ điều hành): $X/server/năm
   - Database licensing (bản quyền CSDL): Oracle, SQL Server rất đắt
   - Backup software (phần mềm sao lưu): $X/năm

4. Labor (Nhân Lực):
   - Sysadmin, DBA, network engineer
   - On-call (trực cấp cứu), maintenance windows

5. Hidden Costs (Chi Phí Ẩn):
   - Refresh cycle (chu kỳ thay mới) mỗi 3-5 năm
   - Unplanned downtime cost (chi phí ngừng hoạt động không lên kế hoạch)

CÔNG CỤ AWS HỖ TRỢ: AWS Migration Evaluator
→ Cài agent thu thập dữ liệu 1-2 tuần
→ Tự động so sánh và đề xuất right-sizing
→ Báo cáo TCO với assumption rõ ràng
```

**Điểm thêm:** Nhấn mạnh rằng TCO không chỉ là tiết kiệm tiền — còn là **agility (linh hoạt)**: deploy (triển khai) trong phút thay vì hàng tuần, scale (mở rộng) tự động, không cần dự phòng capacity (công suất) trước.

---

### Q5: Migration Wave Planning (Lập Kế Hoạch Di Chuyển Theo Đợt) là gì? Bạn phân loại workload vào đợt nào?

**Câu trả lời mẫu:**

Migration wave là nhóm workload được di chuyển cùng nhau trong một khoảng thời gian xác định.

```
TIÊU CHÍ PHÂN LOẠI WORKLOAD VÀO WAVE:

Wave 1 — "Pilot / Easy Wins" (Thí Điểm / Chiến Thắng Dễ):
  ✅ Workload không quan trọng (non-critical)
  ✅ Ít dependency (phụ thuộc) với hệ thống khác
  ✅ Không có compliance yêu cầu đặc biệt
  ✅ Team quen thuộc với công nghệ
  Ví dụ: Dev environment, nội bộ tools, static websites

Wave 2 — "Core Business Non-Critical" (Nghiệp Vụ Chính Nhưng Không Quan Trọng):
  → Website công khai, hệ thống báo cáo, email server
  → Downtime window ngắn vẫn chấp nhận được

Wave 3 — "Business Critical" (Quan Trọng Với Nghiệp Vụ):
  → ERP, CRM, databases chính
  → Cần test cutover kỹ, rollback plan rõ ràng

Wave N — "Mission Critical" (Sứ Mệnh Tối Quan Trọng):
  → Core banking, real-time trading platform, payment gateways
  → Di chuyển cuối cùng sau khi đội ngũ đã có kinh nghiệm từ các wave trước

NGUYÊN TẮC CHUNG:
  - Bắt đầu với những gì ít rủi ro nhất để build confidence (xây dựng niềm tin)
  - Mỗi wave không quá 20-30 applications để dễ quản lý
  - Test cutover cho từng app trước production cutover
  - Rollback plan phải sẵn sàng cho mọi wave
```

---

## Nhóm 2: Di Chuyển Cơ Sở Dữ Liệu

---

### Q6: Giải thích sự khác nhau giữa Homogeneous và Heterogeneous Database Migration. AWS DMS hỗ trợ như thế nào?

**Tại sao hỏi:** Câu hỏi DMS số 1 — nhà tuyển dụng muốn biết bạn hiểu khi nào cần SCT.

**Câu trả lời mẫu:**

```
HOMOGENEOUS MIGRATION (Di Chuyển Đồng Nhất):
  → Source DB engine = Target DB engine
  → Ví dụ: MySQL → Amazon RDS MySQL
           PostgreSQL → Amazon Aurora PostgreSQL
           Oracle → Oracle on EC2
  → Schema không cần chuyển đổi
  → AWS DMS có thể dùng trực tiếp, không cần SCT
  → Đơn giản hơn, ít rủi ro hơn

HETEROGENEOUS MIGRATION (Di Chuyển Dị Cấu Trúc):
  → Source DB engine ≠ Target DB engine
  → Ví dụ: Oracle → Amazon Aurora PostgreSQL
           SQL Server → Amazon RDS MySQL
           Sybase → Amazon Aurora PostgreSQL
  → Schema phải được chuyển đổi trước (data types khác nhau, syntax khác)
  → Cần AWS SCT (Schema Conversion Tool) TRƯỚC khi dùng DMS

QUY TRÌNH HETEROGENEOUS:
  Bước 1: Chạy SCT để phân tích source schema
  Bước 2: SCT tự động convert những gì có thể (khoảng 60-70%)
  Bước 3: DBA (Database Administrator — Quản Trị Viên CSDL) xử lý manual phần còn lại
          (stored procedures, triggers, custom types thường không tự convert được)
  Bước 4: DMS thực hiện full load (tải đầy đủ) với schema đã convert
  Bước 5: DMS CDC (Change Data Capture) để đồng bộ ongoing changes (thay đổi liên tục)
  Bước 6: Cutover khi replication lag (độ trễ sao chép) về 0
```

**Điểm thêm:** SCT không hỗ trợ 100% tự động cho stored procedures, triggers, và custom database logic — phần này thường cần 20-40% effort manual.

---

### Q7: CDC — Change Data Capture hoạt động như thế nào trong AWS DMS? Khi nào cần dùng?

**Câu trả lời mẫu:**

```
CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu):
  → Cơ chế đọc transaction log (nhật ký giao dịch) của database để capture
    các thay đổi (INSERT, UPDATE, DELETE) theo thời gian thực
  → Không phải backup — là stream liên tục của các thay đổi

CƠ CHẾ HOẠT ĐỘNG TRONG DMS:

  Phase 1 — Full Load (Tải Đầy Đủ):
    Source DB ──────────────────────────→ Target DB
    (Copy toàn bộ dữ liệu hiện tại)
    Thời gian: vài giờ đến vài ngày tùy kích thước

  Phase 2 — CDC Ongoing (Đồng Bộ Liên Tục):
    Source DB transaction log → DMS reads → Target DB
    Replication lag (độ trễ): thường < 1 giây với kết nối tốt

  Phase 3 — Cutover (Chuyển Đổi):
    Khi lag ≈ 0 → tắt ứng dụng trỏ source → verify (kiểm tra) target → chuyển traffic

CÁCH DMS ĐỌC TRANSACTION LOG:
  - MySQL / Aurora MySQL: Binary log (binlog) với format ROW
  - PostgreSQL: Logical replication với wal2json hoặc pglogical
  - Oracle: LogMiner (phân tích archive logs / redo logs)
  - SQL Server: SQL Server CDC feature hoặc MS-Replication

KHI NÀO CẦN CDC:
  ✅ Zero-downtime migration (hoặc minimal downtime < 5 phút)
  ✅ Source DB đang hoạt động production trong khi migrate
  ✅ Dataset lớn (full load mất nhiều giờ, cần đồng bộ trong thời gian đó)
  ❌ Không cần nếu: DB nhỏ, có thể chấp nhận downtime vài giờ
```

---

### Q8: Chiến lược zero-downtime database migration là gì? Mô tả các bước cụ thể.

**Câu trả lời mẫu:**

```
ZERO-DOWNTIME DATABASE MIGRATION STRATEGY:

Yêu cầu: Application vẫn chạy bình thường trong suốt quá trình migration.

CÁC BƯỚC:

Bước 1 — Chuẩn Bị (1-2 tuần trước):
  - Đảm bảo source DB đã bật binary logging / WAL / CDC feature
  - Tạo DMS Replication Instance trong cùng Region
  - Tạo source endpoint và target endpoint
  - Nếu heterogeneous: chạy SCT, fix manual conversions

Bước 2 — Full Load (Tải Ban Đầu):
  - DMS tạo migration task với mode "Migrate existing data and replicate ongoing changes"
  - DMS copy toàn bộ data → target DB (app vẫn chạy bình thường trên source)
  - Thời gian: vài giờ → vài ngày tuỳ dataset size

Bước 3 — CDC Replication (Đồng Bộ Liên Tục):
  - Sau khi full load xong, DMS tự động switch sang CDC mode
  - Mọi INSERT/UPDATE/DELETE trên source → được apply vào target trong giây lát
  - Monitor (theo dõi) replication lag dashboard

Bước 4 — Validation (Kiểm Tra Tính Toàn Vẹn):
  - Chạy row count comparison (so sánh số hàng) trên các bảng chính
  - Spot-check (kiểm tra mẫu ngẫu nhiên) một số records quan trọng
  - Application-level testing trên target DB (dùng read replica)

Bước 5 — Cutover Window (Cửa Sổ Chuyển Đổi):
  a) Chờ lag về 0 (hoặc < 1 giây)
  b) Tắt write traffic vào source DB (maintenance mode hoặc read-only mode)
  c) Chờ DMS drain (xử lý hết) các thay đổi cuối cùng
  d) Final validation trên target
  e) Cập nhật connection strings (chuỗi kết nối) của application
  f) Bật ứng dụng trỏ target DB
  g) Monitor 15-30 phút rồi tắt DMS task

Tổng downtime: thường < 5 phút (chỉ bước b-f)
```

---

### Q9: AWS SCT (Schema Conversion Tool) là gì? Giới hạn của nó là gì?

**Câu trả lời mẫu:**

```
AWS SCT — Schema Conversion Tool (Công Cụ Chuyển Đổi Schema):
  → Tool (công cụ) cài đặt trên máy tính local (Windows/Mac/Linux)
  → Kết nối vào source DB, phân tích toàn bộ schema
  → Tự động convert (chuyển đổi) DDL (Data Definition Language — Ngôn Ngữ Định Nghĩa Dữ Liệu)
    sang syntax của target DB engine

NHỮNG GÌ SCT LÀM TỐT:
  ✅ Table definitions (định nghĩa bảng): data types, indexes, constraints
  ✅ View (khung nhìn) đơn giản
  ✅ SQL SELECT queries đơn giản trong stored procedures
  ✅ Assessment report: % code có thể convert tự động vs cần manual

GIỚI HẠN CỦA SCT:
  ❌ Stored procedures phức tạp: logic DB-specific (Oracle PL/SQL, SQL Server T-SQL)
     thường chỉ convert được 50-70% tự động
  ❌ Triggers (bộ kích hoạt): logic phức tạp cần viết lại
  ❌ Proprietary functions (hàm độc quyền): Oracle CONNECT BY, SQL Server PIVOT, v.v.
  ❌ Database links (liên kết database): Oracle DB Links không có tương đương thẳng
  ❌ Cursors (con trỏ) phức tạp
  ❌ Dynamic SQL (SQL động): khó phân tích tự động

THỰC TẾ:
  → Phần chuyển đổi tự động: ~60-70% effort (nỗ lực)
  → Phần manual DBA phải xử lý: ~30-40% effort, thường là phần phức tạp nhất
  → Rule of thumb (quy tắc ngón tay cái): SCT report hiện % conversion action items —
    "Simple" (~10% effort), "Medium" (~50% effort), "Complex" (~100% effort) per item
```

---

### Q10: Làm thế nào kiểm tra tính toàn vẹn dữ liệu (Data Integrity Validation) sau khi migration?

**Câu trả lời mẫu:**

```
DATA VALIDATION STRATEGY (Chiến Lược Kiểm Tra Dữ Liệu) — 3 lớp:

LỚP 1: STRUCTURAL VALIDATION (Kiểm Tra Cấu Trúc)
  - Row count per table (số hàng mỗi bảng): source vs target phải bằng nhau
  - Schema diff (so sánh schema): tên cột, kiểu dữ liệu, indexes, constraints
  - Công cụ: DMS built-in data validation task, hoặc custom SQL scripts

LỚP 2: DATA SAMPLING (Lấy Mẫu Dữ Liệu)
  - Random sample (mẫu ngẫu nhiên) 1-5% records từ các bảng lớn
  - Kiểm tra các bảng quan trọng 100% (financial transactions, user accounts)
  - Hash comparison (so sánh hash): checksum (tổng kiểm tra) trên từng bảng nhỏ
  - Công cụ: DMS có thể enable data validation trong task settings

LỚP 3: APPLICATION-LEVEL TESTING (Kiểm Thử Ở Tầng Ứng Dụng)
  - Chạy smoke test (kiểm thử khói — test nhanh các chức năng chính)
  - Test business-critical flows: login, create order, payment, v.v.
  - Performance test: so sánh query time trên source vs target
  - Báo cáo bất kỳ data discrepancy (sự không khớp dữ liệu) trước cutover

DMS DATA VALIDATION FEATURE:
  → Bật trong task settings: "Enable data validation"
  → DMS tự động compare source vs target row by row (từng hàng)
  → Report bất kỳ mismatch (không khớp) vào CloudWatch Logs
  → Chi phí thêm vì query cả source và target

ROLLBACK TRIGGER (Điều Kiện Kích Hoạt Quay Lui):
  → Nếu validation fail > X% → không cutover → điều tra và fix
  → Có rollback plan rõ ràng: connection strings trỏ lại source trong < 5 phút
```

---

## Nhóm 3: Truyền Tải Dữ Liệu & Snow Family

---

### Q11: Khi nào dùng AWS DataSync và khi nào dùng Snow Family? Bạn tính toán như thế nào?

**Tại sao hỏi:** Câu hỏi "comparison" (so sánh) phổ biến nhất về data transfer.

**Câu trả lời mẫu:**

```
QUY TẮC CHỌN NHANH:

  "Nếu tải lên tất cả dữ liệu qua mạng mất > 1 tuần → dùng Snow Family"

CÔNG THỨC TÍNH THỜI GIAN TRANSFER:

  Thời gian = Kích thước dữ liệu / (Băng thông × Hiệu suất thực tế)

  Ví dụ:
    Dữ liệu: 50 TB = 50 × 1024 GB = 51,200 GB = 51,200 × 8 Gb = 409,600 Gb
    Băng thông: 1 Gbps
    Hiệu suất thực tế: 80% (luôn nhân 0.7-0.8 vì overhead — chi phí mạng)

    Thời gian = 409,600 Gb / (1 Gbps × 0.8) = 512,000 giây ≈ 5.9 ngày

    → Gần 1 tuần → biên giới → cân nhắc Snow hoặc dùng DataSync với Direct Connect

CHỌN DATASYNC KHI:
  ✅ Dữ liệu < 10 TB với băng thông tốt (> 100 Mbps)
  ✅ Cần đồng bộ định kỳ (recurring sync — không phải one-time)
  ✅ Cần data validation tự động (checksum verification)
  ✅ Môi trường có Direct Connect hoặc VPN ổn định
  ✅ Dữ liệu thay đổi liên tục (delta sync)
  ✅ Source: NFS, SMB, S3, HDFS, EFS, FSx

CHỌN SNOW FAMILY KHI:
  ✅ Dữ liệu rất lớn (> 10-20 TB) hoặc tải lên sẽ mất nhiều tuần qua mạng
  ✅ Băng thông kém hoặc không ổn định (vùng sâu, vùng xa, offshore platforms)
  ✅ One-time migration (di chuyển một lần)
  ✅ Cần edge computing (xử lý dữ liệu tại chỗ) trong khi thu thập
  ✅ Data center shutdown (đóng cửa trung tâm dữ liệu) hoàn toàn

  Snowcone   → < 14 TB, portable (cầm tay), harsh environments (môi trường khắc nghiệt)
  Snowball Edge → 80-210 TB, standard migration, có thể cluster
  Snowmobile → > 10 PB, chỉ dùng khi cực kỳ lớn (AWS gửi xe tải đến)
```

---

### Q12: AWS DataSync khác gì so với S3 sync command? Khi nào dùng DataSync?

**Câu trả lời mẫu:**

```
SO SÁNH DATASYNC VS aws s3 sync:

aws s3 sync (AWS CLI):
  - Chạy trên 1 luồng đơn (single thread)
  - Không có checksum verification tích hợp
  - Không có bandwidth throttling (giới hạn băng thông) tự động
  - Không có scheduling (lên lịch) tích hợp
  - Không monitor được trong CloudWatch
  - Dùng cho: copy thủ công, số lượng file nhỏ

AWS DataSync:
  - Multi-thread parallel transfer (truyền song song nhiều luồng): nhanh hơn 10x
  - Built-in checksum verification (kiểm tra tổng kiểm tra tích hợp) — đảm bảo dữ liệu không bị hỏng
  - Bandwidth throttling: tránh ảnh hưởng production traffic
  - Scheduling: hourly/daily/weekly, hoặc theo sự kiện
  - CloudWatch integration: metrics, alarms, logs đầy đủ
  - Hỗ trợ nhiều source: NFS, SMB, HDFS, S3, EFS, FSx, Azure Blob, Google Cloud Storage
  - Preserve metadata (bảo toàn metadata): timestamps, permissions (quyền), ownership
  - Agent-based: cài DataSync agent on-premises (trên VM hoặc bare-metal)

KHI NÀO DÙNG DATASYNC:
  → Di chuyển file shares (NFS/SMB) từ on-premises lên EFS hoặc S3
  → Đồng bộ định kỳ giữa on-premises storage và AWS
  → Di chuyển lớn (hàng chục TB) cần kiểm tra toàn vẹn dữ liệu
  → Cần audit trail (nhật ký kiểm tra) đầy đủ cho compliance
```

---

### Q13: Mô tả quy trình dùng Snowball Edge để di chuyển 80 TB dữ liệu từ on-premises lên S3.

**Câu trả lời mẫu:**

```
QUY TRÌNH SNOWBALL EDGE MIGRATION — 6 BƯỚC:

Bước 1 — ORDER (Đặt Hàng):
  - Đặt Snowball Edge qua AWS Console → chọn Storage Optimized (80 TB)
  - Chỉ định S3 bucket đích và IAM role
  - AWS ship (giao hàng) thiết bị trong 2-5 ngày làm việc

Bước 2 — RECEIVE & SETUP (Nhận & Thiết Lập):
  - Unbox (mở hộp) và kết nối điện + RJ45 ethernet (không dùng WiFi)
  - Dùng LCD màn hình trước để lấy credentials (thông tin đăng nhập)
  - Unlock bằng AWS OpsHub (ứng dụng desktop) hoặc Snowball client CLI
  - OpsHub connect qua local network (mạng nội bộ)

Bước 3 — DATA COPY (Sao Chép Dữ Liệu):
  Cách 1 — Snowball Client CLI:
    snowball cp -r /data/source s3://<bucket>/<prefix>
  Cách 2 — S3 compatible interface (giao diện tương thích S3):
    Mount Snowball như S3 endpoint → dùng aws s3 cp chỉ đến local IP
  Cách 3 — NFS mount (điểm gắn kết NFS):
    Snowball hỗ trợ NFS → mount như share thông thường → copy bình thường

  Lưu ý:
    - Tất cả data được mã hóa AES-256 ngay khi ghi vào thiết bị
    - Encryption key (khóa mã hóa) do AWS KMS (Key Management Service) quản lý
    - KHÔNG BAO GIỜ lưu key cục bộ trên thiết bị

Bước 4 — RETURN (Trả Thiết Bị):
  - Dùng E-ink label trên thiết bị → địa chỉ AWS datacenter tự động cập nhật
  - Giao cho carrier (đơn vị vận chuyển) — AWS trả tiền ship về
  - Tracking available (theo dõi được) qua AWS Console

Bước 5 — INGEST (Nhập Dữ Liệu Vào AWS):
  - AWS datacenter nhận thiết bị → ingest data vào S3 bucket đích
  - Thường mất 1-3 ngày làm việc
  - Notification (thông báo) qua SNS hoặc email khi hoàn tất

Bước 6 — VERIFY (Xác Minh):
  - So sánh S3 object count (số lượng object) vs số file gốc
  - Check S3 checksums vs local manifests
  - AWS sẽ xóa sạch thiết bị trước khi tái sử dụng (NIST 800-88 compliant)
```

---

### Q14: Giải thích AWS Transfer Family. Khi nào nên dùng thay vì tự dựng SFTP server?

**Câu trả lời mẫu:**

```
AWS Transfer Family:
  → Managed service cung cấp SFTP / FTPS / FTP / AS2 endpoints
  → Kết nối trực tiếp đến Amazon S3 hoặc Amazon EFS
  → Không cần quản lý server, patching (vá lỗi), scaling

GIAO THỨC HỖ TRỢ:
  - SFTP (SSH File Transfer Protocol): Port 22, phổ biến nhất
  - FTPS (FTP Secure — FTP Bảo Mật): Port 21 với TLS
  - FTP (File Transfer Protocol): Không mã hóa, chỉ dùng trong VPC nội bộ
  - AS2 (Applicability Statement 2): B2B EDI (Electronic Data Interchange — Trao Đổi Dữ Liệu Điện Tử) file exchange

KHI NÀO DÙNG TRANSFER FAMILY:
  ✅ Thay thế on-premises SFTP servers (truyền tải file an toàn với đối tác B2B)
  ✅ Partners yêu cầu SFTP (họ không thể thay đổi workflow)
  ✅ Không muốn quản lý server infrastructure
  ✅ Cần high availability (HA — Tính Sẵn Sàng Cao) tự động
  ✅ File cần lưu vào S3 hoặc EFS mà không cần trung gian

TỰ DỰNG SFTP SERVER (EC2 + OpenSSH) KHI:
  → Cần logic xử lý file phức tạp (custom pre/post processing)
  → Cần custom authentication backend rất đặc thù
  → Chi phí quá nhỏ để justify managed service
  → Đã có team sysadmin sẵn

SO SÁNH CHI PHÍ:
  Transfer Family: ~$0.30/giờ/endpoint + $0.04/GB transfer
  EC2 t3.medium SFTP: ~$0.042/giờ (nhưng phải quản lý, patch, HA, v.v.)
  → Với workload nhỏ: EC2 rẻ hơn
  → Với workload production: Transfer Family tiết kiệm operational overhead (gánh nặng vận hành)
```

---

## Nhóm 4: Di Chuyển Ứng Dụng

---

### Q15: AWS MGN (Application Migration Service) hoạt động như thế nào? Mô tả quy trình từ A đến Z.

**Câu trả lời mẫu:**

```
AWS MGN — Application Migration Service (thay thế AWS SMS — Server Migration Service):
  Mục đích: Rehost (lift-and-shift) máy chủ vật lý hoặc ảo lên Amazon EC2

QUY TRÌNH MGN — 5 GIAI ĐOẠN:

GIAI ĐOẠN 1 — INSTALL REPLICATION AGENT (Cài Đặt Agent Sao Chép):
  - Cài AWS Replication Agent trên source server (Windows hoặc Linux)
  - Agent kết nối đến AWS MGN Replication Server trong VPC của bạn
  - Tất cả traffic (lưu lượng) mã hóa TLS trong transit (khi truyền)

GIAI ĐOẠN 2 — CONTINUOUS REPLICATION (Sao Chép Liên Tục):
  - Agent đọc tất cả block-level changes (thay đổi cấp khối) từ source disk
  - Data được replicate (sao chép) liên tục vào staging area (vùng dàn dựng) trong AWS
  - Staging area dùng EC2 t3.small instances (rất rẻ) + EBS volumes
  - Lag thường < 1 phút sau initial sync (đồng bộ ban đầu)

GIAI ĐOẠN 3 — CONFIGURE LAUNCH TEMPLATE (Cấu Hình Template Khởi Chạy):
  - Trong MGN Console: định nghĩa instance type (loại instance), VPC, subnet, security groups
  - Mapping disk types (loại ổ đĩa): source disk → EBS volume type (gp3, io1, v.v.)
  - Tùy chỉnh post-launch script (script chạy sau khi khởi chạy): install agents, config changes

GIAI ĐOẠN 4 — TEST CUTOVER (Kiểm Thử Chuyển Đổi):
  - Bắt buộc phải làm trước production cutover
  - MGN tạo test instance trong isolation (cô lập — không ảnh hưởng production)
  - Team QA (Quality Assurance — Đảm Bảo Chất Lượng) kiểm tra app functionality
  - Performance test, dependency check (kiểm tra phụ thuộc)
  - Finalize test instance sau khi pass

GIAI ĐOẠN 5 — PRODUCTION CUTOVER (Chuyển Đổi Production):
  - Lên lịch trong maintenance window
  - Source server: drain connections (thoát kết nối), stop application
  - MGN tạo production EC2 instance từ latest replication point
  - Update DNS / load balancer trỏ đến EC2 mới
  - Monitor rồi disconnect source server (giữ lại vài tuần phòng rollback)
  - Sau khi confident: archive source server
```

---

### Q16: Sự khác biệt giữa AWS MGN và AWS DRS (Elastic Disaster Recovery)?

**Câu trả lời mẫu:**

```
                    AWS MGN                    AWS DRS (Elastic Disaster Recovery)
USE CASE         Di chuyển lên cloud          Phục hồi thảm họa (DR)
                 (Migration)                  (Disaster Recovery)

MỤC TIÊU        Chuyển workload lên EC2       Khởi động lại nhanh sau sự cố
                 là việc chính thức            trên AWS làm site DR phụ

NGUỒN            On-premises, VM, other       On-premises (primary site)
                 cloud                         chạy production

REPLICATION      Liên tục → cutover 1 lần     Liên tục → multi-failover được
PATTERN          rồi dừng                      (failback về on-premises cũng được)

KHI CUTOVER      Xong → source không cần      Sau DR failover: có thể failback
                  nữa                          (chuyển trở lại) về on-premises
                                               khi on-premises khôi phục xong

BILLING          Trả cho staging EC2 nhỏ      Trả cho staging EC2 nhỏ
                 trong quá trình sync          + trả thêm khi failover chạy
                 + EC2 đích sau cutover        full-size instances

CÙNG            - Cùng replication agent
CÔNG NGHỆ:      - Cùng block-level replication
                - Cùng test cutover mechanism

TÓM TẮT:
  MGN: "Tôi muốn chuyển lên AWS vĩnh viễn"
  DRS: "Tôi muốn AWS làm DR site phòng khi datacenter chính gặp sự cố"
```

---

### Q17: Bạn sẽ làm thế nào để rollback (quay lui) nếu migration thất bại?

**Câu trả lời mẫu:**

```
ROLLBACK STRATEGY (Chiến Lược Quay Lui) — Phải lên kế hoạch TRƯỚC khi cutover:

NGUYÊN TẮC CƠ BẢN:
  "Đừng cutover nếu bạn không biết cách rollback trong < 30 phút"

CHO DATABASE MIGRATION (DMS):
  ✅ Giữ DMS task chạy sau cutover ít nhất 24-48 giờ
  ✅ Cấu hình DMS reverse task (task ngược): target → source replication
     (để nếu rollback, dữ liệu từ AWS có thể replicate trở lại on-premises)
  ✅ Giữ source DB trong read-only mode (không shutdown) trong vài giờ đầu
  ✅ Rollback procedure:
     a) Put application in maintenance mode (chế độ bảo trì)
     b) Switch connection string trở lại source DB
     c) Verify data consistency (kiểm tra tính nhất quán dữ liệu)
     d) Re-open application
     Thời gian: < 10 phút nếu đã chuẩn bị

CHO APPLICATION MIGRATION (MGN):
  ✅ Giữ source server ON (bật) trong 1-2 tuần sau cutover
  ✅ Rollback procedure:
     a) Stop traffic đến EC2 mới (remove từ load balancer)
     b) Update DNS trỏ về source server IP cũ
     c) Re-enable application trên source server
     Thời gian: < 15 phút

CHO DATA TRANSFER (DataSync / Snowball):
  ✅ Không xóa source data cho đến khi verify xong target
  ✅ Giữ source intact (nguyên vẹn) ít nhất 30 ngày sau migration

ROLLBACK DECISION TREE (Cây Quyết Định Rollback):
  Phát hiện vấn đề sau cutover?
  → Nhỏ / isolated: fix trên AWS, không rollback
  → Ảnh hưởng nhiều users: rollback ngay + investigate
  → Data corruption (hỏng dữ liệu): rollback ngay, không cần họp
```

---

## Nhóm 5: Vận Hành & Rủi Ro

---

### Q18: Làm thế nào bạn monitor (theo dõi) một cuộc migration đang diễn ra? Tools nào bạn dùng?

**Câu trả lời mẫu:**

```
MONITORING MIGRATION — 3 LỚP:

LỚP 1: AWS MIGRATION HUB (Theo Dõi Tập Trung):
  - Dashboard tổng quan: progress của mọi servers/databases
  - Integration với MGN, DMS, Server Migration Service
  - Notification khi phase hoàn tất hoặc có error

LỚP 2: SERVICE-SPECIFIC MONITORING (Theo Dõi Theo Dịch Vụ):

  DMS Task Monitoring:
    - Replication lag (độ trễ sao chép): target < 60 giây
    - Table statistics: rows loaded, errors
    - CloudWatch metrics: CDCLatencySource, CDCLatencyTarget
    - DMS console: Table Statistics tab → xem progress từng table

  MGN Monitoring:
    - Replication lag per server
    - Data replicated (dữ liệu đã sao chép) vs total disk
    - CloudWatch: SourceServerLag, ReplicationBytesTransferred

  DataSync Monitoring:
    - Files transferred vs total files
    - Bytes transferred, transfer rate (tốc độ truyền)
    - CloudWatch: BytesTransferred, FilesPrepared, FilesTransferred

LỚP 3: ALERTING & INCIDENT RESPONSE (Cảnh Báo & Xử Lý Sự Cố):
  - CloudWatch Alarms: cảnh báo khi lag > threshold (ngưỡng)
  - SNS notification: email/Slack/PagerDuty khi cần
  - Migration runbook: ai làm gì khi alarm nào kích hoạt

METRICS QUAN TRỌNG CẦN WATCH:
  Trước cutover: Replication lag → phải ≈ 0
  Sau cutover: Application error rate, latency, throughput
  Post-migration: Cost (chi phí), performance → compare với baseline
```

---

### Q19: Những rủi ro phổ biến nhất trong migration project là gì? Cách giảm thiểu?

**Câu trả lời mẫu:**

```
TOP RỦI RO VÀ CÁCH GIẢM THIỂU:

RỦI RO 1: UNDISCOVERED DEPENDENCIES (Phụ Thuộc Chưa Khám Phá)
  Vấn đề: App A di chuyển trước, nhưng App B ở on-premises vẫn gọi vào App A
           → network path thay đổi → outage (ngừng hoạt động) bất ngờ
  Giảm thiểu:
    ✅ Chạy ADS (Application Discovery Service) để map dependencies
    ✅ Network flow analysis (phân tích luồng mạng) 2-4 tuần trước khi migration
    ✅ Group apps với tight dependencies vào cùng 1 wave

RỦI RO 2: DATA CORRUPTION / LOSS (Hỏng / Mất Dữ Liệu)
  Vấn đề: Silent data corruption trong quá trình transfer
  Giảm thiểu:
    ✅ Enable data validation trong DMS task
    ✅ Checksum verification trong DataSync
    ✅ Row count comparison trước cutover
    ✅ Application-level testing sau migration

RỦI RO 3: PERFORMANCE DEGRADATION (Giảm Hiệu Suất)
  Vấn đề: App chạy chậm hơn trên AWS so với on-premises
  Giảm thiểu:
    ✅ Right-sizing trước khi migrate (dùng Migration Evaluator)
    ✅ Kiểm tra network latency: Direct Connect vs VPN vs internet
    ✅ Kiểm tra DB performance: storage type (gp2 vs gp3 vs io1), IOPS
    ✅ Load test trên test environment trước khi production cutover

RỦI RO 4: SECURITY / COMPLIANCE GAP (Lỗ Hổng Bảo Mật / Tuân Thủ)
  Vấn đề: Cấu hình security group quá rộng, encryption at rest (mã hóa khi lưu trữ) chưa bật
  Giảm thiểu:
    ✅ Security review với AWS Well-Architected Migration Lens
    ✅ Enforce encryption: EBS, RDS, S3 encryption bật từ đầu
    ✅ Least privilege IAM (IAM quyền tối thiểu)
    ✅ Config Rules kiểm tra compliance liên tục

RỦI RO 5: CUTOVER OVERRUN (Quá Hạn Cửa Sổ Chuyển Đổi)
  Vấn đề: Cutover mất lâu hơn dự kiến → downtime quá RTO
  Giảm thiểu:
    ✅ Thực hiện test cutover ít nhất 1-2 lần trước
    ✅ Có cutover runbook chi tiết từng bước với thời gian ước tính
    ✅ Rollback trigger: nếu quá X phút → rollback ngay, không cố gắng tiếp
    ✅ Dry run (chạy thử) toàn bộ quy trình với team
```

---

### Q20: Sau khi migration xong, bạn làm gì để tối ưu chi phí (Cost Optimization)?

**Câu trả lời mẫu:**

```
POST-MIGRATION COST OPTIMIZATION — 5 HÀNH ĐỘNG NGAY LẬP TỨC:

HÀNH ĐỘNG 1: RIGHTSIZING (Điều Chỉnh Kích Thước Tài Nguyên)
  - Chạy AWS Compute Optimizer sau 2 tuần để có đủ data
  - Thường giảm 20-30% cost bằng cách downsize over-provisioned instances
  - Ví dụ: m5.2xlarge → m5.large nếu CPU < 10% và RAM < 30%

HÀNH ĐỘNG 2: SAVINGS PLANS / RESERVED INSTANCES (Gói Tiết Kiệm / Instance Dự Trữ)
  - Sau 1 tháng chạy ổn định → mua Compute Savings Plans
  - Tiết kiệm 40-66% so với On-Demand cho workload ổn định
  - Không commit (cam kết) quá 60% usage — giữ 40% On-Demand cho linh hoạt

HÀNH ĐỘNG 3: STORAGE OPTIMIZATION (Tối Ưu Lưu Trữ)
  - S3 Intelligent-Tiering: tự động chuyển object ít truy cập sang tier rẻ hơn
  - EBS: xem xét gp3 (rẻ hơn 20% so với gp2 với performance cao hơn)
  - Snapshot lifecycle policy: xóa snapshot (ảnh chụp) cũ tự động
  - RDS: reserved instances cho databases chạy 24/7

HÀNH ĐỘNG 4: TURN OFF WHAT YOU DON'T NEED (Tắt Những Gì Không Cần)
  - Dev/test environments: schedule shutdown ngoài giờ làm việc (tiết kiệm 60%)
  - Instance Scheduler: tự động start/stop theo lịch
  - Xóa unattached EBS volumes (ổ đĩa không gắn vào instance)
  - Xóa unused Elastic IPs (IP tĩnh không dùng) — bị tính phí

HÀNH ĐỘNG 5: MONITORING & GOVERNANCE (Theo DÕI & Quản TRỊ)
  - AWS Cost Explorer: phân tích chi phí theo service, tag, account
  - AWS Budgets: cảnh báo khi chi phí vượt ngưỡng
  - Tag policy: tag mọi resource với environment, team, project
  - Định kỳ review: monthly FinOps meeting (họp tối ưu tài chính hàng tháng)

KỲ VỌNG TIẾT KIỆM SAU 3 THÁNG:
  → Rightsizing: -20 đến -30%
  → Savings Plans: -40 đến -60% trên committed usage
  → Storage + cleanup: -10 đến -15%
  → Tổng: thường giảm 30-50% so với on-premises TCO
```

---

## 📝 Câu Hỏi Thường Gặp Thêm (Bonus Questions)

Những câu hỏi này ít phổ biến hơn nhưng xuất hiện ở vòng senior/principal:

```
B1: "Làm thế nào bạn migrate một Oracle database 5 TB với zero downtime?"
    → Trả lời: DMS + SCT + CDC, cutover window < 5 phút

B2: "Thiết kế Hybrid Cloud (đám mây lai) architecture khi vừa on-premises vừa AWS"
    → Trả lời: Direct Connect + Storage Gateway + Route 53 DNS-based routing

B3: "Bạn sẽ prioritize (ưu tiên) như thế nào khi 10 workload cùng cần migrate?"
    → Trả lời: Risk matrix (ma trận rủi ro) × Business value × Technical complexity

B4: "Compliance requirement ảnh hưởng thế nào đến migration strategy?"
    → Trả lời: Data residency, encryption, audit logging, PCI-DSS/HIPAA requirements

B5: "Giải thích AWS Well-Architected Migration Lens"
    → Trả lời: 6 pillars apply to migration: Operational Excellence, Security, Reliability,
               Performance Efficiency, Cost Optimization, Sustainability
```

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành
