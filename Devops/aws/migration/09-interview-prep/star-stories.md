# AWS Migration — Mẫu Câu Chuyện STAR

> **STAR** là phương pháp kể chuyện được dùng trong phỏng vấn behavioral (hành vi):
> - **S**ituation (Tình Huống): Bối cảnh, quy mô, áp lực
> - **T**ask (Nhiệm Vụ): Vai trò của bạn, mục tiêu cụ thể
> - **A**ction (Hành Động): Những gì **bạn** làm — công nghệ, quyết định, xử lý khó khăn
> - **R**esult (Kết Quả): Số liệu đo được, bài học rút ra, tác động kinh doanh

Mỗi câu chuyện dưới đây là **mẫu có thể điều chỉnh** — hãy thay số liệu và chi tiết theo kinh nghiệm thực tế của bạn.

---

## 📚 Mục Lục

- [Story 1: Database Migration Zero-Downtime](#story-1-database-migration-zero-downtime)
- [Story 2: Lift-and-Shift Hàng Loạt Server](#story-2-lift-and-shift-hàng-loạt-server)
- [Story 3: Di Chuyển Dữ Liệu Lớn Với Snow Family](#story-3-di-chuyển-dữ-liệu-lớn-với-snow-family)
- [Story 4: Xử Lý Sự Cố Trong Migration](#story-4-xử-lý-sự-cố-trong-migration)
- [Story 5: Thuyết Phục Stakeholder Về Migration Strategy](#story-5-thuyết-phục-stakeholder-về-migration-strategy)
- [Hướng Dẫn Tùy Chỉnh Câu Chuyện](#hướng-dẫn-tùy-chỉnh-câu-chuyện)

---

## Story 1: Database Migration Zero-Downtime

**Câu hỏi phỏng vấn kích hoạt:**
- "Kể về một lần bạn thực hiện database migration phức tạp."
- "Bạn đã xử lý zero-downtime migration như thế nào?"
- "Nói về một thách thức kỹ thuật khó nhất bạn từng đối mặt."

---

### Tình Huống (Situation)

```
Công ty [tên công ty / lĩnh vực: ngân hàng / e-commerce / logistics] có hệ thống
core chạy trên Oracle Database 12c on-premises tại datacenter riêng.
Database có dung lượng [ví dụ: 2 TB], phục vụ [ví dụ: 500,000 giao dịch/ngày].

Áp lực:
  - License Oracle hết hạn sau [3 tháng] — chi phí renew (gia hạn) quá cao
  - Business yêu cầu: KHÔNG được phép ngừng hệ thống trong giờ hành chính
  - Rủi ro: Database chứa dữ liệu tài chính — sai 1 record là vấn đề nghiêm trọng
```

### Nhiệm Vụ (Task)

```
Tôi là [Database Engineer / Cloud Migration Engineer] chịu trách nhiệm:
  - Thiết kế và thực thi kế hoạch migration Oracle → Amazon Aurora PostgreSQL
  - Đảm bảo zero data loss và downtime < 5 phút
  - Hoàn thành trong [8 tuần] với team [3 người]
```

### Hành Động (Action)

```
Tôi thực hiện theo quy trình sau:

TUẦN 1-2: ĐÁNH GIÁ (ASSESSMENT)
  → Chạy AWS SCT (Schema Conversion Tool) để phân tích Oracle schema
  → Kết quả: 72% code có thể tự động convert, 28% cần can thiệp thủ công
  → Phần thủ công chủ yếu: 15 stored procedures phức tạp dùng Oracle-specific
    PL/SQL cursors và Oracle-specific functions (CONNECT BY LEVEL, ROWNUM)

TUẦN 3-5: CHUYỂN ĐỔI SCHEMA & TESTING
  → Làm việc với DBA team để rewrite 15 stored procedures
  → Tôi viết automated test suite (bộ kiểm thử tự động) dùng pgTAP để so sánh
    output (kết quả) giữa Oracle và Aurora PostgreSQL
  → Phát hiện 3 edge cases (trường hợp đặc biệt) có kết quả khác nhau do NULL
    handling khác nhau giữa Oracle và PostgreSQL → fix trước khi production

TUẦN 6-7: DMS SETUP VÀ FULL LOAD
  → Tạo DMS Replication Instance r5.xlarge trong VPC cùng region
  → Bật Oracle LogMiner (phân tích redo logs) và tạo DMS task với mode:
    "Migrate existing data and replicate ongoing changes" (full load + CDC)
  → Full load hoàn tất sau [18 giờ] — trong khi production Oracle vẫn chạy bình thường
  → Monitor replication lag → ổn định ở [0.3-2 giây]

TUẦN 8: TEST CUTOVER VÀ PRODUCTION CUTOVER
  → Test cutover (không ảnh hưởng production) → application chạy đúng 100%
  → Production cutover window: [2 giờ sáng Chủ Nhật]
    - 02:00: Bật maintenance mode trên application
    - 02:01: Chờ DMS drain replication lag về 0
    - 02:04: Final row count validation — match 100%
    - 02:06: Cập nhật connection string → trỏ Aurora PostgreSQL
    - 02:08: Tắt maintenance mode, chạy smoke test
    - 02:12: Xác nhận hệ thống hoạt động bình thường
```

### Kết Quả (Result)

```
✅ Downtime thực tế: 12 phút (mục tiêu < 5 phút nhưng vẫn trong kế hoạch)
✅ Zero data loss — row count và checksums khớp 100%
✅ Tiết kiệm chi phí: Oracle license fee giảm $[150,000]/năm → Aurora cost $[18,000]/năm
✅ Performance cải thiện: Query p99 latency (độ trễ bách phân vị 99) giảm từ [120ms] xuống [45ms]
   nhờ Aurora storage engine tối ưu hơn

Bài học:
  - Không bao giờ bỏ qua test cutover — chúng tôi phát hiện edge case chỉ
    trong test cutover, nếu bỏ qua sẽ gây lỗi production
  - Invest (đầu tư) vào automated testing sớm — tiết kiệm nhiều giờ debug sau này
```

---

## Story 2: Lift-and-Shift Hàng Loạt Server

**Câu hỏi phỏng vấn kích hoạt:**
- "Bạn đã quản lý large-scale migration project (dự án di chuyển quy mô lớn) như thế nào?"
- "Kể về một lần bạn phải deliver (bàn giao) trong deadline gấp."
- "Làm thế nào bạn tổ chức công việc khi có nhiều workloads cần di chuyển?"

---

### Tình Huống (Situation)

```
Công ty [tên] quyết định đóng cửa 1 trong 2 datacenters để tiết kiệm chi phí.
Datacenter đó đang chạy [150 servers], bao gồm:
  - [80] web/application servers (Java, .NET, Node.js)
  - [40] database servers (MySQL, PostgreSQL, SQL Server)
  - [30] internal tools (monitoring, CI/CD, file servers)

Deadline: Phải đóng datacenter trong [6 tháng]
Áp lực: Hết hạn hợp đồng datacenter — không gia hạn được
```

### Nhiệm Vụ (Task)

```
Tôi là Lead Migration Engineer phụ trách:
  - Thiết kế migration strategy cho toàn bộ [150 servers]
  - Lên kế hoạch migration waves (đợt di chuyển)
  - Phối hợp với team dev, ops, và business stakeholders
  - Đảm bảo không có outage ảnh hưởng customer (khách hàng)
```

### Hành Động (Action)

```
GIAI ĐOẠN 1 — DISCOVERY (KHÁM PHÁ) [Tuần 1-3]:
  → Cài AWS Application Discovery Service agents trên tất cả [150 servers]
  → Thu thập 2 tuần traffic data để map dependencies chính xác
  → Kết quả: Phát hiện [12 nhóm dependency] (application clusters)
  → Quan trọng: Phát hiện [3 apps] có dependency ẩn không có trong documentation

GIAI ĐOẠN 2 — PHÂN LOẠI VÀ LÊN KẾ HOẠCH [Tuần 3-4]:
  → Phân loại theo 7Rs:
    - Retire: [15 servers] (apps không còn dùng) → tiết kiệm ngay lập tức
    - Rehost (MGN): [95 servers] → không cần sửa code, di chuyển nhanh
    - Replatform: [25 servers] → chuyển sang managed services (RDS, Elastic Beanstalk)
    - Retain: [15 servers] → giữ lại on-premises vì compliance nội địa
  → Chia thành [5 waves], mỗi wave [2-3 tuần], từ ít rủi ro đến nhiều rủi ro

GIAI ĐOẠN 3 — THỰC THI [Tuần 5-22]:
  Wave 1: Dev/test environments → thực hành quy trình, build confidence
  Wave 2: Internal tools (monitoring, file servers)
  Wave 3: Non-critical business apps
  Wave 4: Business-critical apps (cẩn thận hơn, test cutover kỹ)
  Wave 5: Databases và payment systems (rủi ro cao nhất, làm cuối cùng)

  Với mỗi server tôi dùng MGN:
    - Cài replication agent
    - Đợi sync hoàn tất
    - Test cutover → QA approve
    - Production cutover trong maintenance window

  Tôi build dashboard (bảng điều khiển) tùy chỉnh trên AWS Migration Hub để
  track (theo dõi) real-time progress cho tất cả stakeholders
```

### Kết Quả (Result)

```
✅ Hoàn thành trong [5.5 tháng] — trước deadline nửa tháng
✅ Zero unplanned outages trong suốt quá trình migration
✅ [135/150] servers thành công di chuyển (15 retained on-premises theo kế hoạch)
✅ Tiết kiệm: Đóng datacenter → tiết kiệm $[800,000]/năm chi phí cơ sở vật chất
✅ Bonus: Phát hiện [15 servers] không còn cần thiết → terminated → tiết kiệm thêm

Bài học:
  - Discovery phase (giai đoạn khám phá) là quan trọng nhất — đừng rút ngắn
  - Migration Hub dashboard giúp management (quản lý) trust (tin tưởng) vào tiến độ
  - Bắt đầu với wave dễ để team build confidence, tránh sai lầm ở production critical
```

---

## Story 3: Di Chuyển Dữ Liệu Lớn Với Snow Family

**Câu hỏi phỏng vấn kích hoạt:**
- "Kể về một lần bạn di chuyển lượng dữ liệu rất lớn."
- "Làm thế nào bạn giải quyết vấn đề khi không có đủ băng thông?"
- "Bạn đã tính toán trade-off (đánh đổi) kỹ thuật như thế nào?"

---

### Tình Huống (Situation)

```
Công ty media (truyền thông) tích lũy [500 TB] video archive (kho lưu trữ video)
trên NAS (Network Attached Storage — Lưu Trữ Gắn Mạng) on-premises trong [10 năm].
Họ quyết định đóng cửa văn phòng và di chuyển toàn bộ lên Amazon S3.

Thực trạng hạ tầng mạng:
  - Internet: 200 Mbps symmetrical
  - Tính toán: 500 TB × 8 bits / (200 Mbps × 0.7 hiệu suất) = 16,000,000 giây ≈ 185 ngày
  → Upload qua mạng sẽ mất 6 tháng — hoàn toàn không khả thi

Deadline: Phải trả văn phòng trong [3 tháng]
```

### Nhiệm Vụ (Task)

```
Tôi là Data Engineer phụ trách:
  - Đề xuất và thực thi phương án di chuyển 500 TB trong 3 tháng
  - Đảm bảo không mất dữ liệu
  - Tối ưu chi phí (budget $[50,000])
```

### Hành Động (Action)

```
ĐÁNH GIÁ VÀ QUYẾT ĐỊNH:
  → Tính toán: 3× Snowball Edge Storage Optimized (mỗi thiết bị 80 TB usable)
    = 240 TB/lần gửi → cần [3 lần gửi] × [3 Snowballs/lần] = 9 thiết bị tổng
  → Chi phí ước tính: $[200/thiết bị/10 ngày] + data transfer sau ingest
    = $[4,500] total device rental
  → So sánh với upload qua mạng: cần nâng cấp đường truyền + [185 ngày] → $[30,000]+
  → Decision (quyết định): Dùng Snowball Edge

CHUẨN BỊ DỮ LIỆU:
  → Audit (kiểm tra) toàn bộ file archive → loại bỏ duplicates (bản sao) = tiết kiệm [50 TB]
  → Tổ chức file theo prefix (tiền tố) S3 để dễ tìm kiếm sau này
  → Tạo manifest file (file danh sách) cho từng batch (lô) để verify sau

QUY TRÌNH VẬN CHUYỂN:
  → Batch 1: [240 TB] → 3× Snowball Edge → gửi cho AWS
  → Trong khi AWS ingest Batch 1 → chuẩn bị Batch 2
  → AWS ingest vào S3 Glacier Instant Retrieval (tier rẻ hơn Standard [60%])
  → Tôi tự viết script Python để:
    a) Copy file song song lên Snowball với [8 threads] (luồng)
    b) Tạo SHA256 checksum (kiểm tra tổng) cho mỗi file
    c) Verify checksum sau khi file xuất hiện trong S3

VALIDATION SAU INGEST:
  → So sánh file count (số lượng file) và total size per directory
  → Spot-check [5%] files bằng cách download (tải xuống) và so sánh checksum
  → Report: [99.997%] file khớp → [3 files] bị corrupt (hỏng) trong transfer → re-copy
```

### Kết Quả (Result)

```
✅ Hoàn thành trong [11 tuần] — trước deadline 1 tháng
✅ 500 TB (sau khi loại duplicate còn [450 TB]) trên S3 thành công
✅ Zero data loss (3 files corrupt được phát hiện và re-copied)
✅ Chi phí thực tế: $[28,000] — tiết kiệm $[22,000] so với budget
✅ Ongoing cost (chi phí tiếp theo): S3 Glacier = $[9,000]/năm so với NAS $[45,000]/năm

Bài học:
  - Tính toán "có nên dùng Snow hay không" là kỹ năng quan trọng — phải làm trước khi đề xuất
  - Automation cho checksum verification rất đáng đầu tư thời gian
  - Dọn dẹp duplicates trước migration → tiết kiệm cả tiền lẫn thời gian
```

---

## Story 4: Xử Lý Sự Cố Trong Migration

**Câu hỏi phỏng vấn kích hoạt:**
- "Kể về một lần migration gặp sự cố nghiêm trọng. Bạn xử lý thế nào?"
- "Bạn đã từng phải rollback (quay lui) chưa? Quy trình như thế nào?"
- "Kể về một sai lầm bạn đã mắc và bài học rút ra."

---

### Tình Huống (Situation)

```
Trong dự án migration e-commerce platform lên AWS, chúng tôi đang thực hiện
cutover (chuyển đổi) database PostgreSQL [1.2 TB] lên Amazon Aurora PostgreSQL.

Sau khi cutover [35 phút], monitoring (theo dõi) phát hiện:
  - Error rate (tỉ lệ lỗi) tăng từ [0.1%] lên [12%]
  - Specific error (lỗi cụ thể): "ERROR: operator does not exist: integer = text"
  - Ảnh hưởng: [15-20%] requests đến trang product listing bị fail

Đây là [11 giờ đêm thứ Sáu] — peak traffic (lưu lượng cao nhất) của tuần
```

### Nhiệm Vụ (Task)

```
Tôi là on-call engineer (kỹ sư trực cấp cứu) có 30 phút để quyết định:
  Tiếp tục debug và fix? hay Rollback ngay?
```

### Hành Động (Action)

```
PHÚT 0-5: ĐÁNH GIÁ NHANH (RAPID ASSESSMENT)
  → Xem CloudWatch dashboard: error rate 12%, nhưng [85%] traffic vẫn hoạt động bình thường
  → Đọc error log: "operator does not exist: integer = text" → type mismatch (không khớp kiểu dữ liệu)
  → Identify (xác định) affected queries (truy vấn bị ảnh hưởng): chỉ liên quan đến product_id
  → Root cause (nguyên nhân gốc rễ): PostgreSQL strict type checking (kiểm tra kiểu nghiêm ngặt)
    — Oracle tự động implicit cast (chuyển đổi ngầm) integer → text, Aurora không làm vậy

PHÚT 5-10: QUYẾT ĐỊNH
  → Báo cáo ngay với manager và business owner: "Tìm ra vấn đề, có thể fix trong 20 phút"
  → Decision framework (khung quyết định):
    + Bug isolated (lỗi cô lập)? → Có, chỉ ảnh hưởng product listing
    + Fix có rollback được không nếu fix sai? → Có
    + Rollback database mất bao lâu? → 10-15 phút, gây full outage (ngừng hoàn toàn)
  → Quyết định: TRY TO FIX (cố gắng sửa) trước rollback (vì bug isolated, fix đơn giản)

PHÚT 10-25: FIX
  → Tìm tất cả queries có implicit cast trong codebase: [8 queries] trong [3 services]
  → Fix: thêm explicit cast `::text` hoặc `::integer` vào queries
  → Deploy (triển khai) hotfix lên [3 services] — không cần touch database
  → Verify (kiểm tra): error rate về 0% sau 2 phút deploy

PHÚT 25-30: POST-INCIDENT (SAU SỰ CỐ)
  → Thông báo toàn bộ team và stakeholders
  → Enable CloudWatch alarm (cảnh báo) cho error rate
  → Note (ghi chú) vào runbook: "PostgreSQL strict type checking — audit all implicit casts trước migration"
```

### Kết Quả (Result)

```
✅ Issue resolved (vấn đề giải quyết) trong 25 phút — không cần rollback database
✅ Total impact: [35 phút] error rate 12% → khoảng [2,000] failed requests
✅ Database migration thành công — tiếp tục không bị gián đoạn

Bài học (tôi chia sẻ với team sau sự cố):
  1. Always audit implicit type casts khi migrate từ Oracle → PostgreSQL
     → Tôi thêm bước này vào checklist (danh sách kiểm tra) migration standard
  2. Decision tree cho "fix vs rollback" cần được chuẩn bị trước, không quyết định trong stress
  3. Isolated bug + simple fix → try to fix trước khi rollback toàn bộ
  4. Luôn có monitoring sẵn sàng ngay sau cutover

Sau incident (sự cố), tôi viết blog nội bộ về PostgreSQL type casting
và tổ chức brown bag session (buổi chia sẻ nội bộ không chính thức) cho team
```

---

## Story 5: Thuyết Phục Stakeholder Về Migration Strategy

**Câu hỏi phỏng vấn kích hoạt:**
- "Kể về một lần bạn phải thuyết phục người khác về quyết định kỹ thuật."
- "Bạn đã xử lý conflict (xung đột) với stakeholder như thế nào?"
- "Khi business requirements (yêu cầu kinh doanh) và kỹ thuật mâu thuẫn, bạn làm gì?"

---

### Tình Huống (Situation)

```
Team business muốn migrate toàn bộ [200 servers] lên AWS trong [2 tháng]
(quicker than usual — nhanh hơn bình thường) để đón kịp deadline hợp đồng mới.

Vấn đề: Team migration engineering (kỹ thuật di chuyển) của tôi ước tính
cần ít nhất [5 tháng] để làm đúng — đặc biệt có [30 databases] production
cần zero-downtime migration.

Business director (giám đốc kinh doanh) insist (nhấn mạnh): "Phải xong trong 2 tháng,
hoặc chúng ta mất hợp đồng $[2 triệu]"
```

### Nhiệm Vụ (Task)

```
Tôi là Senior Migration Engineer cần:
  - Đánh giá feasibility (tính khả thi) của deadline 2 tháng
  - Trình bày rõ risks (rủi ro) nếu rush (vội vàng)
  - Đề xuất giải pháp trung gian đáp ứng cả business và technical requirements
```

### Hành Động (Action)

```
BƯỚC 1: HIỂU BUSINESS CONSTRAINT (RÀNG BUỘC KINH DOANH) THỰC SỰ
  → Gặp riêng business director để hiểu: "Trong 2 tháng, cụ thể cần đạt gì?"
  → Phát hiện: Hợp đồng yêu cầu "ứng dụng chính phải chạy trên AWS" —
    không yêu cầu toàn bộ 200 servers

BƯỚC 2: PHÂN TÍCH VÀ CHUẨN BỊ DỮ LIỆU
  → Tôi tạo spreadsheet (bảng tính) với từng server:
    - Mức độ rủi ro (thấp/trung/cao)
    - Thời gian cần thiết để migrate an toàn
    - Liệu có thể rush không và hệ quả là gì
  → Kết quả: [50 servers] "core business" có thể migrate trong 2 tháng nếu tập trung
  → Các database production phức tạp cần thêm 3 tháng để làm đúng

BƯỚC 3: ĐỀ XUẤT PHÂN TẦNG (PHASED APPROACH)
  → Tôi chuẩn bị 3 options (lựa chọn) với risk và cost rõ ràng:

  Option A — Rush toàn bộ trong 2 tháng:
    → Chi phí: $X (thuê thêm 5 contractors — nhà thầu)
    → Risk: 40% khả năng ít nhất 1 production outage
    → Hệ quả tài chính nếu outage: $[500,000] lost revenue/giờ

  Option B — Phân tầng: Core apps trong 2 tháng, databases trong 5 tháng:
    → Chi phí: $X (chi phí bình thường)
    → Risk: < 5% khả năng outage
    → Đáp ứng hợp đồng vì "core apps" đã trên AWS

  Option C — Hybrid approach (cách tiếp cận lai):
    → Core apps lên AWS trong 2 tháng
    → Databases connect qua Direct Connect đến on-premises trong giai đoạn đầu
    → Migrate databases sau khi team có thêm kinh nghiệm

BƯỚC 4: PRESENT (TRÌNH BÀY) CHO LEADERSHIP (BAN LÃNH ĐẠO)
  → Trình bày không phải "không thể làm trong 2 tháng" mà là
    "đây là 3 cách đạt mục tiêu, với different risk profiles (hồ sơ rủi ro khác nhau)"
  → Tôi để business chọn — với đầy đủ thông tin để quyết định có trách nhiệm
```

### Kết Quả (Result)

```
✅ Business chọn Option B (phân tầng) — vì khi thấy risk của Option A rõ ràng,
   không ai muốn chịu trách nhiệm cho $500,000 potential loss
✅ Core apps migrated đúng 2 tháng → đáp ứng hợp đồng
✅ Databases migrated an toàn trong tháng 4 và 5 — zero downtime
✅ Hợp đồng $2 triệu được ký

Bài học:
  1. Khi conflict (xung đột) kỹ thuật vs business: KHÔNG tranh luận ai đúng —
     hãy đặt dữ liệu và options lên bàn để decision makers (người ra quyết định) chọn
  2. Hiểu đúng business constraint thực sự (không phải bề mặt) thường mở ra giải pháp
  3. Phân tầng approach thường là "win-win" trong migration — không phải all-or-nothing
  4. Risk quantification (định lượng rủi ro) bằng số tiền dễ thuyết phục hơn "đây là rủi ro kỹ thuật"
```

---

## 📝 Hướng Dẫn Tùy Chỉnh Câu Chuyện

### Bước 1: Điền Vào Template

```
Mỗi câu chuyện có dấu [  ] — đây là chỗ bạn điền thông tin thực tế:
  [tên công ty]    → "ngân hàng ABC" hoặc chỉ "công ty tài chính nơi tôi làm"
  [số liệu]        → số server, dung lượng data, thời gian, chi phí thực
  [role của bạn]   → đúng với title thực tế của bạn
```

### Bước 2: Thêm "Màu Sắc" Vào Câu Chuyện

```
Điều nhà tuyển dụng muốn nghe:
  ✅ Số liệu cụ thể (50 TB, 150 servers, $200K savings — tiết kiệm)
  ✅ Quyết định khó khăn và lý do bạn chọn cách đó
  ✅ Sai lầm và bài học rút ra (vulnerability — sự thành thật)
  ✅ Tác động đến business (không chỉ kỹ thuật)
  ✅ Collaboration (hợp tác) với người khác

Điều nên tránh:
  ❌ Câu chuyện quá hoàn hảo không có thách thức
  ❌ Chỉ nói "team chúng tôi" mà không rõ bạn cụ thể làm gì
  ❌ Câu chuyện quá dài (> 5 phút) hoặc lan man
  ❌ Chỉ nói kỹ thuật mà không đề cập business impact
```

### Bước 3: Luyện Tập Nói To

```
Mục tiêu: Kể câu chuyện trong 2-3 phút khi được hỏi bình thường,
          và 4-5 phút khi được hỏi follow-up (câu hỏi tiếp theo)

Cách luyện:
  1. Viết ra tay một lần để internalize (nội hóa) câu chuyện
  2. Kể cho đồng nghiệp nghe → xin feedback
  3. Record (ghi âm) bản thân → nghe lại
  4. Vary (thay đổi) độ chi tiết tùy cường độ câu hỏi của interviewer
```

### Bảng Câu Hỏi → Câu Chuyện Phù Hợp

| Câu Hỏi Phỏng Vấn | Dùng Story |
| ----------------- | ---------- |
| "Thách thức kỹ thuật khó nhất?" | Story 1 (DB migration) hoặc Story 4 (sự cố) |
| "Dự án migration lớn nhất bạn làm?" | Story 2 (150 servers) |
| "Xử lý dữ liệu lớn như thế nào?" | Story 3 (500 TB Snow Family) |
| "Thuyết phục stakeholder không đồng ý?" | Story 5 (thuyết phục) |
| "Sai lầm và bài học rút ra?" | Story 4 (sự cố) — honest version |
| "Làm việc dưới áp lực deadline?" | Story 2 hoặc Story 4 |
| "Trade-off quyết định kỹ thuật?" | Story 3 (Snow vs DataSync calculation) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành
