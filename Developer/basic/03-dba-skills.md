# 20 câu hỏi phỏng vấn mock — kỹ năng DBA

Tài liệu này căn cứ vào kinh nghiệm **database administration** trên CV: backup/restore, replication, performance tuning, indexing, schema design, migration planning, monitoring cho **PostgreSQL**, **MySQL**, **MongoDB**, **Redis**, **SQL Server**; và các dự án **ICT**, **BlueSG**, **HospoPay**, v.v.

---

## 1. Bạn phân biệt **RPO** và **RTO** trong chiến lược backup như thế nào? Áp dụng ra sao với PostgreSQL vs SQL Server?

**Trả lời:**

- **RPO (Recovery Point Objective — mục tiêu điểm phục hồi):** khoảng thời gian dữ liệu có thể mất tối đa khi có sự cố (ví dụ chấp nhận mất tối đa 15 phút giao dịch).
- **RTO (Recovery Time Objective — mục tiêu thời gian phục hồi):** thời gian tối đa hệ thống được phép “tắt” trước khi phải hoạt động lại.

**PostgreSQL:** logical dump (**pg_dump**) thường phù hợp khi RPO lỏng hơn (snapshot theo lịch); **WAL archiving** + **PITR (Point-in-Time Recovery — phục hồi theo thời điểm)** giúp RPO chặt hơn.
**SQL Server:** **full + differential + transaction log backups** cho phép RPO rất nhỏ nếu backup log thường xuyên.

**Ưu/nhược & so sánh:**

- **Ưu pg_dump:** đơn giản, portable, tốt cho subset/schema cụ thể. **Nhược:** restore lớn chậm, RPO phụ thuộc lịch dump.
- **Ưu WAL/PITR (PG) vs log chain (SQL Server):** cả hai đều phục vụ RPO chặt; SQL Server có toolchain enterprise quen thuộc (SSMS, maintenance plans); PostgreSQL linh hoạt nhưng cần vận hành WAL/archive rõ ràng.

---

## 2. Khi nào nên dùng **logical backup** (pg_dump) thay vì **physical backup** / snapshot trên RDS?

**Trả lời:**
Dùng **pg_dump** khi cần portability giữa version, export subset, audit dữ liệu, hoặc migration sang môi trường khác. Snapshot/RDS automated backup phù hợp DR nhanh, RTO thấp, toàn cục instance.

**Ưu snapshot/RDS:** restore nhanh, ít tác động runtime nếu storage snapshot nhất quán. **Nhược:** phụ thuộc nhà cung cấp, khó “cherry-pick” bảng.
**Ưu pg_dump:** kiểm soát object-level. **Nhược:** lock/IO trên DB lớn nếu không tách read replica; restore lâu.

**So sánh tương đương:** tương tự **mysqldump** (MySQL) vs **Percona XtraBackup** / snapshot — trade-off logic vs physical.

---

## 3. Bạn thiết kế **chiến lược backup** cho hệ thống vừa có PostgreSQL vừa có SQL Server (như dự án ICT). Các bước và tài liệu hóa restore?

**Trả lời:**

1. Xác định **RPO/RTO** theo từng domain (user management, reporting…).
2. **PostgreSQL:** cửa sổ backup, retention; kết hợp logical dump khi phù hợp + **WAL** cho PITR nếu cần.
3. **SQL Server:** **full**, **differential**, **log** backup theo workload OLTP.
4. **Runbook restore:** bước xác định điểm restore, verify checksum, kiểm tra **referential integrity**, test trên môi trường non-prod định kỳ.
5. Giám sát job backup failed, dung lượng disk, thời lượng backup.

**Ưu:** tách policy theo engine đúng bản chất từng hệ. **Nhược:** phức tạp vận hành, cần người hiểu cả hai stack.

**So sánh:** SQL Server có **SSMS** và policy enterprise; PostgreSQL thường dùng scripting + **pgBackRest**/**Barman** (công cụ backup chuyên dụng) nếu scale lớn.

---

## 4. **Schema migration** “forward-only” và align với release là gì? Rủi ro nếu rollback application?

**Trả lời:**
**Forward-only migration** nghĩa là thay đổi schema chỉ tiến về phía trước theo version; không “quay lui” schema trên production. Rollback app thường kéo theo mismatch nếu DB đã đổi (cột rename, NOT NULL…).

**Kỹ thuật giảm rủi ro:** **backward-compatible** columns, **expand–contract pattern**, **dual-write** hoặc **feature flag** trong giai đoạn chuyển tiếp (như mô tả BlueSG/ICT).

**Ưu forward-only:** lịch sử rõ ràng, CI/CD đơn giản. **Nhược:** cần kỹ thuật tách phase, không “sửa nhanh” bằng revert SQL tùy tiện.

**So sánh công cụ:** **Flyway** / **Liquibase** (migration có version) vs ad-hoc script — tool giúp audit và ordering; ad-hoc nhanh nhưng dễ mất kiểm soát.

---

## 5. **Expand–contract** trong migration billing/subscription (PostgreSQL) hoạt động thế nào?

**Trả lời:**

1. **Expand:** thêm cột/bảng mới, nullable hoặc default an toàn; app cũ vẫn chạy.
2. **Migrate data:** backfill, dual-write nếu cần.
3. **Contract:** chuyển read sang schema mới, bỏ cột cũ khi không còn consumer.

**Ưu:** cutover an toàn, giảm downtime. **Nhược:** kéo dài thời gian triển khai, cần coordination NestJS/service.

**So sánh:** tương tự **blue-green** ở tầng app nhưng áp dụng cho schema evolution.

---

## 6. Bạn tối ưu **slow query** trên PostgreSQL: dùng metric/log nào và các bước tuning?

**Trả lời:**

- Bật/khai thác **pg_stat_statements** (extension thống kê câu lệnh), **slow query log**, **EXPLAIN (ANALYZE, BUFFERS)**.
- Theo dõi **lock waits**, **connection pool saturation** (pooler như **PgBouncer** nếu có).
- Điều chỉnh **autovacuum** và theo dõi **bloat** (phình bảng/index).

**Ưu pg_stat_statements:** tổng hợp theo fingerprint, dễ tìm “top N”. **Nhược:** cần bảo mật/normalize query text; reset stats cần quy trình.

**So sánh với MySQL:** **performance_schema** + **slow log**; SQL Server: **Query Store**, **DMVs**.

---

## 7. **Autovacuum** và **bloat** trên PostgreSQL: vì sao quan trọng và can thiệp thế nào?

**Trả lời:**
PostgreSQL dùng **MVCC (Multi-Version Concurrency Control)**; **VACUUM** dọn dead tuple, **ANALYZE** cập nhật statistics cho planner. Nếu autovacuum yếu → bloat index/table → chậm IO, index không hiệu quả.

**Can thiệp:** tune `autovacuum_vacuum_scale_factor`, cost limit, parallel vacuum (version mới), **REINDEX** có kế hoạch.

**Ưu autovacuum mặc định:** tự động. **Nhược:** workload nặng có thể cần tune aggressive.
**So sánh:** SQL Server không MVCC giống PG; vấn đề tương đương là **index fragmentation** + **statistics** — xử lý **REBUILD/REORGANIZE**, **UPDATE STATISTICS**.

---

## 8. Trên **SQL Server**, bạn dùng **statistics** và **index** để tuning như thế nào?

**Trả lời:**

- Xem missing indexes qua DMVs (cẩn trọng vì gợi ý không phải lúc nào cũng đúng).
- **UPDATE STATISTICS** hoặc để auto; kiểm tra **parameter sniffing** gây plan xấu.
- **Index maintenance:** rebuild vs reorganize theo fragmentation threshold.

**Ưu index phù hợp:** giảm latency read. **Nhược:** tăng chi phí write, storage, có thể làm chậm bulk load.

**So sánh với PostgreSQL:** PG ít “auto suggest” như một số tool SQL Server; thường dựa **EXPLAIN** và workload thực tế.

---

## 9. Khi nào nên đưa logic vào **stored procedure** / **function** gần data thay vì chỉ ở application (NestJS/.NET)?

**Trả lời:**
Phù hợp cho **reporting**, **batch job**, rule cần **atomic** và giảm round-trip, hoặc tái sử dụng từ nhiều consumer (ETL, BI).

**Ưu:** hiệu năng, transaction rõ ràng tại DB. **Nhược:** khó test/version theo CI của service, coupling với DB team/release, khó scale theo kiểu microservice “pure”.

**So sánh:** PostgreSQL functions/procedures vs **T-SQL** stored procedures — tư duy giống nhau; với MongoDB thường tránh SP, ưu tiên aggregation pipeline ở app hoặc **$function** hạn chế.

---

## 10. **Idempotency** trong luồng thanh toán liên quan SQL/procedure — bạn đảm bảo thế nào?

**Trả lời:**
Dùng **unique constraint** trên idempotency key, **UPSERT** (`ON CONFLICT` trong PostgreSQL), hoặc bảng ledger/outbox; procedure bọc transaction, kiểm tra trạng thái trước khi ghi side-effect.

**Ưu:** webhook/payment retry an toàn. **Nhược:** schema và index phải thiết kế sớm; race condition cần lock/isolation phù hợp.

**So sánh:** MySQL: `INSERT ... ON DUPLICATE KEY UPDATE`; SQL Server: **MERGE** (cẩn trọng bug history) hoặc pattern riêng.

---

## 11. Bạn thiết kế **index** cho workload tìm kiếm booking/rental (PostgreSQL): nguyên tắc gì?

**Trả lời:**

- Phân tích predicate (`WHERE`, `JOIN`, `ORDER BY`).
- **Composite index** đúng thứ tự cột theo selectivity và sort.
- Tránh index trùng; cân nhắc **partial index** nếu filter cố định (ví dụ `WHERE status = 'active'`).
- Sau thay đổi: đo bằng **EXPLAIN**, theo dõi **seq scan** bất thường.

**Ưu partial index:** nhỏ, nhanh. **Nhược:** planner phải match điều kiện chính xác.

**So sánh:** MongoDB dùng compound index + **ESR rule** (Equality, Sort, Range); khái niệm tương tự nhưng implementation khác.

---

## 12. **Replication lag** ảnh hưởng hệ thống thế nào? Bạn monitor và xử lý?

**Trả lời:**
Lag làm user đọc **replica** thấy dữ liệu cũ (stale read), vi phạm consistency mong đợi; với thanh toán/subscription có thể gây UX sai.

**Monitor:** replication lag metric (RDS/CloudWatch hoặc tương đương), **WAL** send/replay, alert ngưỡng.

**Xử lý:** giảm tải write nóng, tối ưu transaction lớn, scale IO, hoặc routing read critical về primary.

**Ưu read replica:** scale read, HA. **Nhược:** eventual consistency, vận hành lag.

**So sánh:** MySQL replication lag vs **PostgreSQL streaming replication** — bản chất tương tự; **MongoDB** secondary lag trong replica set.

---

## 13. **Connection pool saturation** — nguyên nhân và giải pháp trong kiến trúc microservices?

**Trả lời:**
Nguyên nhân: quá nhiều instance service × pool size lớn → vượt `max_connections`; long transaction; leak connection.

**Giải pháp:** pooler (**PgBouncer** transaction mode), giảm pool per instance, tối ưu query, circuit breaker, autoscaling hợp lý.

**Ưu pooler:** tiết kiệm connection. **Nhược:** transaction mode có hạn chế với prepared statements/session features.

**So sánh:** SQL Server thường dùng **connection pooling** ở driver + **RDS Proxy** tương tự ý tưởng.

---

## 14. Bạn role **DBA** vs **developer-owner schema** trong microservice: ranh giới trách nhiệm?

**Trả lời:**
Developer-owned: schema trong repo service, migration theo release. DBA/platform: chuẩn hóa backup, HA, limits, security, review DDL nguy hiểm, capacity.

**Ưu dev-owned:** velocity cao. **Nhược:** risk drift, cần guardrail (review, automated checks).

**So sánh mô hình:** **GitOps schema** vs centralized DBA team — tổ chức lớn thường hybrid.

---

## 15. **MySQL** (HospoPay) cho payment webhook: transaction isolation và deadlock bạn xử lý ra sao?

**Trả lời:**
Chọn isolation phù hợp (thường **READ COMMITTED** hoặc default InnoDB), giữ transaction ngắn, thứ tự lock nhất quán, retry có backoff cho deadlock.

**Ưu InnoDB row lock:** concurrency tốt. **Nhược:** deadlock vẫn xảy ra nếu truy cập bảng theo thứ tự khác nhau.

**So sánh:** PostgreSQL cũng có deadlock; diagnostic khác nhưng mindset giống.

---

## 16. **MongoDB** (Vision Direct / XO Sections): khi nào phù hợp và trade-off so với PostgreSQL cho ecommerce API?

**Trả lời:**
Mongo phù hợp document linh hoạt, nested data, scale horizontal sharding (khi cần), schema evolution nhanh.

**Ưu:** linh hoạt model, horizontal scaling story. **Nhược:** join phức tạp kém thuận tiện hơn SQL, multi-document transaction có overhead (đã cải thiện qua các version), cần discipline index.

**So sánh với PostgreSQL:** PG mạnh ACID quan hệ, constraint, reporting; Mongo mạnh khi access pattern theo document.

---

## 17. **Redis** trong stack ecommerce: use case và rủi ro vận hành?

**Trả lời:**
Use case: cache session/cart hot data, rate limit, pub/sub nhẹ, distributed lock (cẩn trọng TTL/fencing).

**Ưu:** latency cực thấp. **Nhược:** mất instance → mất cache (trừ persistence config), **cache stampede**, hot key.

**So sánh:** **Memcached** đơn giản pure cache; Redis đa năng hơn (data structure, persistence optional).

---

## 18. **Monitoring** DB trên cloud (CloudWatch / dashboard): metric tối thiểu bạn cảnh báo?

**Trả lời:**
CPU, free storage, **IOPS**, **read/write latency**, connections, replication lag, failed backup/maintenance, slow queries, error log spike.

**Ưu:** phát hiện sớm. **Nhược:** quá nhiều alert → noise; cần SLO-driven thresholds.

**So sánh:** On-prem có thể dùng **Prometheus** + **Grafana**; managed service tích hợp sẵn nhưng kém linh hoạt query.

---

## 19. **Security** ở tầng database trong hệ identity/IoT (Ory, microservices): bạn làm gì?

**Trả lời:**
Least privilege role per service, không dùng superuser trong app, **TLS** cho connection, rotate secret, audit DDL, tách schema theo service nếu feasible, **row-level security (RLS)** khi cần multi-tenant chặt (PostgreSQL).

**Ưu RLS (PG):** enforce tại DB. **Nhược:** phức tạp debug/perf nếu policy kém.

**So sánh:** SQL Server có **RLS** tương tự; MySQL hạn chế hơn tùy version/edition.

---

## 20. Trước khi **cutover** migration lớn production, checklist của bạn gồm những gì?

**Trả lời:**

1. Backup/snapshot + verified restore path.
2. Migration script idempotent/replay-safe trong giới hạn cho phép.
3. **Backward compatibility** hoặc feature flag.
4. So sánh row count/checksum mẫu, invariant business.
5. Rollback plan (app + data) rõ ràng — biết điều gì không thể rollback nếu forward-only.
6. Canary traffic / dry-run trên staging gần production volume.

**Ưu checklist:** giảm incident. **Nhược:** tốn thời gian; vẫn cần judgment khi trade-off tốc độ.

**So sánh công cụ:** automated pipeline (CI) chạy migration trên staging + integration test vs manual — nên kết hợp.

---

## Phụ lục — giải thích từ viết tắt / thuật ngữ

| Thuật ngữ     | Giải thích ngắn                                                      |
| ------------- | -------------------------------------------------------------------- |
| **RPO**       | Recovery Point Objective — mức mất dữ liệu chấp nhận được.           |
| **RTO**       | Recovery Time Objective — thời gian phục hồi tối đa cho phép.        |
| **WAL**       | Write-Ahead Log — nhật ký ghi trước của PostgreSQL.                  |
| **PITR**      | Point-in-Time Recovery — phục hồi DB đến một mốc thời gian.          |
| **DDL**       | Data Definition Language — lệnh định nghĩa schema (CREATE/ALTER…).   |
| **DML**       | Data Manipulation Language — INSERT/UPDATE/DELETE.                   |
| **MVCC**      | Multi-Version Concurrency Control — cơ chế đồng thời của PostgreSQL. |
| **OLTP**      | Online Transaction Processing — xử lý giao dịch tương tác.           |
| **ETL**       | Extract, Transform, Load — tích hợp dữ liệu.                         |
| **HA**        | High Availability — sẵn sàng cao.                                    |
| **DR**        | Disaster Recovery — phục hồi thảm họa.                               |
| **IAM / TLS** | Identity/access management / mã hóa truyền tải.                      |
| **RDS**       | Relational Database Service (AWS) — dịch vụ DB quan hệ quản trị.     |

---

_Tài liệu mang tính ôn tập phỏng vấn; câu trả lời mang tính mẫu và cần điều chỉnh theo kinh nghiệm thực tế từng dự án._
