# Vai trò Database Administrator (DBA) — Câu hỏi phỏng vấn

Hai mươi câu hỏi phỏng vấn thường gặp cho vị trí **Database Administrator (DBA)**, kèm câu trả lời có cấu trúc để bạn điều chỉnh theo kinh nghiệm (vendor **RDBMS (Relational Database Management System)**, cloud so với on‑prem, quy mô team).

---

## Câu 1: **DBA (Database Administrator)** làm gì hằng ngày, và khác **backend developer** thế nào?

**Trả lời:**

**DBA (Database Administrator)** chịu trách nhiệm **reliability, performance, security** và **lifecycle** của nền tảng database: provisioning, configuration, patching, backup, recovery testing, replication và **HA (High Availability)**, capacity planning, incident response, access governance, và phối hợp thay đổi cho schema và infrastructure.

**Backend developer** chủ yếu xây application logic và data access patterns; họ quan tâm correctness và query design, nhưng thường không chịu trách nhiệm **OS (Operating System)**‑level tuning, storage layout, cluster failover, hay tiêu chuẩn backup/**DR (Disaster Recovery)** toàn tổ chức.

**Overlap:** Cả hai đều troubleshoot slow query và indexing; **DBA (Database Administrator)** thường đưa ra standards, tooling và production guardrails trong khi developer ship features.

---

## Câu 2: Giải thích **RTO (Recovery Time Objective)**, **RPO (Recovery Point Objective)**, và chúng ảnh hưởng thế nào tới thiết kế backup và disaster recovery.

**Trả lời:**

- **RPO (Recovery Point Objective — mục tiêu điểm phục hồi):** Mức **data loss** tối đa chấp nhận được, đo theo thời gian (ví dụ: “chấp nhận mất tối đa 15 phút transaction”). Nó quyết định tần suất backup hoặc cách replicate (ví dụ: transaction log shipping mỗi N phút, synchronous replica sang **AZ (Availability Zone)** khác).

- **RTO (Recovery Time Objective — mục tiêu thời gian phục hồi):** Thời gian **downtime** tối đa chấp nhận để khôi phục dịch vụ (ví dụ: “database phải online trong 1 giờ”). Nó thúc đẩy automation, standby capacity, runbook, và việc dùng warm standby, active‑passive cluster, hay cloud multi‑**AZ (Availability Zone)** failover.

**Liên hệ thực tế:** Chỉ nightly backup rẻ có thể đáp ứng **RPO (Recovery Point Objective)** lỏng nhưng không đáp ứng **RPO (Recovery Point Objective)** chặt; **RTO (Recovery Time Objective)** chặt thường cần quy trình restore đã test và có thể cần automatic failover—không chỉ “đã có backup.”

---

## Câu 3: Trình bày cách bạn thiết kế backup strategy cho production **OLTP (Online Transaction Processing)** database.

**Trả lời:**

Câu trả lời tốt nên nhắc tới **layers** và **verification**:

1. **Full backup** theo lịch phù hợp với kích thước và tốc độ thay đổi (thường weekly hoặc daily với database nhỏ hơn).
2. **Incremental / differential** (nếu engine hỗ trợ) để rút ngắn recovery time giữa các lần full.
3. **Transaction log backup** (**SQL Server**) hoặc **WAL (Write-Ahead Log) archiving** (**PostgreSQL**) cho point‑in‑time recovery và **RPO (Recovery Point Objective)** chặt hơn.
4. Bản sao **offsite / immutable** để tăng khả năng chống ransomware (object lock, **WORM (Write Once, Read Many)**, account tách biệt).
5. **Retention** phù hợp compliance (legal hold, audit) so với chi phí.
6. **Restore test** định kỳ (tự động nếu được): chứng minh file backup hợp lệ và ghi lại thời gian restore thực tế (phục vụ **RTO (Recovery Time Objective)**).

Nên nhắc **encryption** in transit và at rest, và **least privilege** trên credential của backup storage.

---

## Câu 4: Bạn thực hiện và ghi chép restore test thế nào mà không rủi ro production?

**Trả lời:**

- Dùng **isolated environment** (instance riêng, **VPC (Virtual Private Cloud)**, hoặc restored clone) với dữ liệu **sanitized** nếu có **PII (Personally Identifiable Information)**.
- Restore sang **database name** hoặc server mới; không bao giờ ghi đè prod.
- Tự động hóa: script restore + smoke checks (database online, checksums, row count bảng quan trọng, application connectivity trên staging).
- Ghi **elapsed time** theo từng phase (restore, recovery, index rebuild nếu có) để kiểm chứng giả định **RTO (Recovery Time Objective)**.
- Sau major version upgrade hoặc storage migration, baseline lại thời gian restore.

---

## Câu 5: So sánh synchronous và asynchronous replication từ góc nhìn **DBA (Database Administrator)**.

**Trả lời:**

| Khía cạnh | Synchronous | Asynchronous |
|--------|-------------|----------------|
| **Data lag** | Không (commit chờ replica **ack (acknowledgment)**) | Replica có thể lag |
| **RPO (Recovery Point Objective)** | Mạnh hơn (không commit cho tới khi durable trên standby) | Yếu hơn nếu primary fail trước khi catch‑up |
| **Latency / throughput** | Commit latency cao hơn, nhạy với khoảng cách | Ít ảnh hưởng tới primary writes |
| **Split‑brain / failover** | Thường đi kèm quorum / fencing | Cần promotion rules cẩn thận |

Dùng **sync (synchronous)** khi mất một transaction đã commit là không chấp nhận được và budget latency cho phép (cùng metro, **RTT (Round-Trip Time)** thấp). Dùng **async (asynchronous)** cho cross‑region **DR (Disaster Recovery)** hoặc khi performance primary là ưu tiên hàng đầu, chấp nhận **small data loss** có thể xảy ra khi failover trừ khi bổ sung kiểm soát khác.

---

## Câu 6: Bạn thiết kế **HA (High Availability)** cho database quan trọng thế nào?

**Trả lời:**

Các lớp điển hình:

1. **Loại bỏ single point of failure:** Multi‑**AZ (Availability Zone)** / multi‑node cluster, networking và nguồn điện dự phòng (cloud che giấu phần lớn việc này).
2. **Automatic health checks** và **failover** với **quorum** để tránh dual primary (ví dụ: Patroni + etcd, **SQL Server AG (Always On Availability Group)** với witness, cloud **RDS (Relational Database Service)** Multi‑**AZ (Availability Zone)**).
3. **Connection routing:** **DNS (Domain Name System)**, proxy, hoặc driver‑level endpoints theo primary hiện tại.
4. **Failover** đã diễn tập: game day, controlled switchover trước khi patch.
5. **Observability:** replication lag, redo apply rate, disk latency, checkpoint pressure.

Thừa nhận trade‑off **CAP (Consistency, Availability, Partition tolerance — định lý Brewer)**: synchronous **HA (High Availability)** cải thiện consistency giữa các node nhưng có thể làm tổn hại availability khi partition.

---

## Câu 7: Mô tả phương pháp troubleshoot “slow database” của bạn.

**Trả lời:**

1. **Scope:** Một query, một app, cả instance, hay tầng storage?
2. **Time correlation:** Deployment, stats job, backup, batch **ETL (Extract, Transform, Load)**, antivirus, **VM (Virtual Machine)** noisy neighbor.
3. **Instance health:** **CPU (Central Processing Unit)**, memory pressure (rủi ro **OOM (Out Of Memory)**), **disk latency** và **IOPS (Input/Output Operations Per Second)**, **waits** (I/O, lock, latch).
4. **Top workloads:** Theo duration, **CPU (Central Processing Unit)**, reads, executions; xác định plan regression (bad stats, parameter sniffing, cardinality estimates).
5. **Concurrency:** blocking chain, deadlock, mismatch isolation level.
6. **Change control:** drop index gần đây, thay đổi parameter, spike cardinality.

Kết thúc bằng vòng **hypothesis → evidence → change → measure** và ưu tiên quan sát **non‑destructive** trước khi đổi global settings.

---

## Câu 8: Khác nhau thế nào giữa blocking và deadlock? Giảm thiểu từng loại ra sao?

**Trả lời:**

- **Blocking:** Một session giữ lock; session khác chờ. Thường bình thường trong thời gian ngắn; có vấn đề khi **long‑running transaction** hoặc **missing index** gây chờ lâu. Giảm thiểu: rút ngắn transaction, isolation phù hợp, tune query, thêm index cẩn thận, chỉ dùng lock hint khi thận trọng.

- **Deadlock:** Chờ vòng (A lock row 1, B lock row 2; A chờ row 2 của B, B chờ row 1 của A). Engine **detect** và **victim** một session. Giảm thiểu: **lock ordering** nhất quán trong app code, transaction nhỏ hơn, index phù hợp để giảm lock footprint, retry logic cho transaction bị victim.

---

## Câu 9: Bạn tiếp cận index design và index maintenance định kỳ thế nào?

**Trả lời:**

**Design:** Khớp index với predicate **selective** và key **JOIN**; tránh index rộng chồng chéo; cân nhắc **covering index** cho read path nóng nhưng cân **write amplification** và storage.

**Maintenance:** Theo dõi **fragmentation** (heap/index), **bloat** (**PostgreSQL**), **stale statistics**. Rebuild/reorganize (**SQL Server**) hoặc `REINDEX` / `pg_repack` (**PostgreSQL**) trong maintenance window hoặc online nếu được hỗ trợ. **Validate** bằng plan trước/sau và workload test, không theo lịch mê tín.

---

## Câu 10: Bạn đặt metric gì trên **DBA (Database Administrator)** dashboard cho production?

**Trả lời:**

Ví dụ nhóm theo mối quan tâm:

- **Availability:** uptime, replication state, backup thành công gần nhất, failover events.
- **Performance:** **CPU (Central Processing Unit)**, memory, batch requests/sec, buffer cache hit (kèm ngữ cảnh), disk latency/queue depth, top waits.
- **Capacity:** tăng trưởng kích thước database, log volume, connection count so với limit, áp lực tempdb hoặc work_mem.
- **Correctness / risk:** failed login, thay đổi permission, long transaction, số session bị block, checksum failure.

Gắn alert với **SLO (Service Level Objective)** và **runbook**, không chỉ threshold thô.

---

## Câu 11: Bạn xử lý security và least privilege cho database access thế nào?

**Trả lời:**

- **Roles** thay vì named user; **tách** role cho app runtime, migrations, read‑only reporting, và break‑glass admin.
- **Không shared password**; tích hợp **IAM (Identity and Access Management) / AD (Active Directory) / Kerberos** khi có thể; rotate secret qua vault.
- **Network:** private subnet, **TLS (Transport Layer Security)**, firewall rule giới hạn source **IP (Internet Protocol)**.
- **Auditing:** thay đổi **DDL (Data Definition Language)**, grant privilege, logon failure; ship log tới **SIEM (Security Information and Event Management)**.
- **Data protection:** encryption at rest, column‑level hoặc tokenization cho field nhạy cảm, masking ở non‑prod.

Nên nhắc **access review** định kỳ và revoke account không dùng.

---

## Câu 12: Quy trình của bạn khi apply database patch hoặc major version upgrade?

**Trả lời:**

1. **Đọc release notes** về breaking change, deprecated feature, hành vi optimizer.
2. **Test** trên clone với workload đại diện; chạy regression test và so sánh **query plan**.
3. **Backup / snapshot** trước thay đổi; định nghĩa **rollback** (restore so với chính sách downgrade).
4. **Maintenance window** hoặc cutover **blue/green** cho bước nhảy lớn.
5. **Post‑check:** version, extensions, job, replication health, application smoke test.
6. **Communicate** stakeholders và incident bridge nếu rủi ro cao.

Với managed cloud, phối hợp **maintenance window** và parameter group.

---

## Câu 13: Bạn capacity planning cho database đang tăng trưởng thế nào?

**Trả lời:**

- **Trend** tăng data và log theo tuần/tháng; seasonality (sự kiện marketing).
- **Headroom** cho index maintenance, batch job, và peak **QPS (Queries Per Second)**.
- **Storage performance** class (**IOPS (Input/Output Operations Per Second)**/throughput) so với size—tránh disk quá lớn nhưng chậm.
- **Compute** scaling: giới hạn vertical so với read replica / trigger sharding.
- Thảo luận **cost:** archiving, partition dữ liệu lạnh, compression, tiered storage.

Đầu ra: ngưỡng khi nào **scale up/out** hoặc bắt đầu dự án **archival**.

---

## Câu 14: Database corruption là gì, phát hiện thế nào, và bạn làm gì?

**Trả lời:**

**Corruption** là cấu trúc trên disk không hợp lệ hoặc không nhất quán (page torn, checksum sai). **Detection:** `CHECKDB` (**SQL Server**), `pg_checksums` / `pg_verifybackup` (**PostgreSQL**), scrubbing tầng storage, backup verification, bất thường ở application level.

**Response:** Tránh thay đổi hoảng loạn; **isolate** phạm vi (object nào); restore từ **good backup** nếu lan rộng; với hư hại hạn chế, page restore hoặc object‑level restore nếu có; mở vendor support kèm diagnostics. **Post‑incident:** root cause (firmware lỗi, **RAM (Random Access Memory)** hỏng, bug storage), cải thiện monitoring.

---

## Câu 15: Giải thích checkpoint, **WAL (Write-Ahead Log)**/redo log, và vì sao chúng quan trọng cho recovery và performance.

**Trả lời:**

**WAL (Write-Ahead Log — nhật ký ghi trước):** Thay đổi được append vào log bền **trước** khi data file phản ánh—cho phép **crash recovery** và **PITR (Point-In-Time Recovery)**.

**Checkpoint:** Flush dirty buffer xuống data file và đẩy một điểm để recovery work có giới hạn. Checkpoint **quá thưa** → recovery lâu và **WAL (Write-Ahead Log)** lớn; checkpoint **quá aggressive** → spike I/O và latency.

Việc **DBA (Database Administrator)**: tune setting liên quan checkpoint trong hướng dẫn vendor, đặt **WAL (Write-Ahead Log)** trên storage **fast durable**, theo dõi **checkpoint warning** và **log generation rate**.

---

## Câu 16: Bạn quản lý schema change an toàn trên production thế nào?

**Trả lời:**

- **Migrations as code** (Flyway, Liquibase, sqitch) có peer review; idempotent khi có thể.
- **Online vs offline:** **DDL (Data Definition Language)** lớn có thể cần tùy chọn **online rebuild** hoặc pattern **expand/contract** để tránh lock lâu.
- **Phased release:** thêm nullable column → backfill theo batch → thêm constraint → chuyển reads/writes.
- **Rollback plan:** feature flag, bước đảo ngược, hoặc chiến lược restore.
- **Coordination** với app team về lock type (`ACCESS EXCLUSIVE` so với concurrent operations).

---

## Câu 17: Connection pooling là gì, và **DBA (Database Administrator)** thường gặp vấn đề gì với nó?

**Trả lời:**

**Pooling** tái sử dụng kết nối **TCP (Transmission Control Protocol)** để giảm handshake overhead và tránh cạn max connections.

**Vấn đề:** Thundering herd sau failover, quá nhiều pool × replica = connection storm, **long session** giữ server‑side state, **transaction** mở xuyên **HTTP (Hypertext Transfer Protocol)** request gây blocking.

Góc **DBA (Database Administrator)**: right‑size `max_connections`, dùng **PgBouncer** / **RDS (Relational Database Service)** Proxy / gateway pattern, enforce **timeout**, giáo dục developer về pool sizing per instance.

---

## Câu 18: Bạn hỗ trợ compliance (ví dụ **GDPR (General Data Protection Regulation)**, **PCI (Payment Card Industry — thường gọi PCI DSS)**) từ góc database thế nào?

**Trả lời:**

- **Data inventory:** **PII (Personally Identifiable Information)**/payment data nằm đâu; gắn tag schema/column.
- **Retention và deletion:** legal hold so với yêu cầu xóa; partition drop so với row delete ở quy mô lớn.
- **Encryption:** at rest và in transit; key rotation qua **KMS (Key Management Service)**/**HSM (Hardware Security Module)**.
- **Access logging** và separation of duties (ai được đọc prod).
- **Non‑prod masking** để developer không dùng raw production **PII (Personally Identifiable Information)**.
- **Audit evidence:** backup retention, ai chạy **DDL (Data Definition Language)** nào.

---

## Câu 19: So sánh trách nhiệm vận hành: self‑managed **RDBMS (Relational Database Management System)** so với managed cloud (**RDS (Relational Database Service)**, **Cloud SQL**, **Azure SQL**).

**Trả lời:**

| Lĩnh vực | Self‑managed | Managed |
|------|----------------|---------|
| **OS (Operating System)** / engine patching | Bạn | Provider (với lựa chọn window của bạn) |
| Backup / **PITR (Point-In-Time Recovery)** | Bạn thiết kế và test | Built‑in; bạn vẫn test restore |
| **HA (High Availability)** / failover | Bạn (Patroni, **AG (Availability Group)**, v.v.) | Thường có tùy chọn Multi‑**AZ (Availability Zone)** / regional |
| Fine‑tuning kernel/**FS (File System)** | Bạn | Hạn chế / đã abstract |
| Cost control | **CapEx (Capital Expenditure)** + nhân công | **OpEx (Operating Expenditure)**; theo dõi **IOPS (Input/Output Operations Per Second)**, storage, egress |

Câu trả lời hay: **bạn vẫn giữ data ownership và architecture** trong cả hai; managed chuyển **undifferentiated heavy lifting**, không chuyển trách nhiệm kiểm chứng **RTO (Recovery Time Objective)**/**RPO (Recovery Point Objective)**.

---

## Câu 20: Kể về một database incident nghiêm trọng bạn đã xử lý (hoặc sẽ xử lý). Bạn học được gì?

**Trả lời:**

Dùng câu chuyện kiểu **STAR (Situation, Task, Action, Result)** nếu có kinh nghiệm thật; nếu không, mô tả scenario thực tế (ví dụ: replica lag → đọc stale data, failover thất bại, runaway query làm đầy disk).

Bao gồm:

1. **Detection** (alert, báo cáo user) và **triage** (scope, blast radius).
2. **Stabilization** (kill runaway query, throttle app, failover nếu an toàn, thêm disk khẩn cấp).
3. **Communication** với stakeholder và trung thực về timeline.
4. **Root cause** (thiếu index + deployment + thiếu alert trên log volume).
5. **Remediation** (sửa query, thêm index trên staging, thêm alert, cải thiện runbook).
6. **Prevention:** guardrail trong **CI (Continuous Integration)** cho migration nguy hiểm, automated restore test.

Người phỏng vấn tìm **quy trình bình tĩnh**, **nhận thức tác động khách hàng**, và **fix bền**, không phải heroics.

---

## Tóm tắt nhanh

Phỏng vấn **DBA (Database Administrator)** thường xoay quanh **backup**/**DR (Disaster Recovery)** (**RTO (Recovery Time Objective)**/**RPO (Recovery Point Objective)**), **HA (High Availability)**/replication, performance và lock, security và compliance, kỷ luật change và upgrade, và **operational maturity** (monitoring, automation, tested restore). Chuẩn bị một hai **concrete story** và nắm command **vendor‑specific** ở mức cao, kể cả khi hằng ngày chủ yếu dùng managed service.
