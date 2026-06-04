# AWS Migration — Bài Toán Thiết Kế Kiến Trúc (Architecture Scenarios)

> Mỗi bài toán mô phỏng **system design interview thực tế** cho vị trí Solutions Architect hoặc Migration Engineer. Đọc đề bài → tự thiết kế → sau đó so sánh với giải pháp tham khảo.

## 📚 Mục Lục

- [Scenario 1: Enterprise Lift-and-Shift 500 Servers](#scenario-1-enterprise-lift-and-shift-500-servers)
- [Scenario 2: Zero-Downtime Oracle-to-Aurora Migration](#scenario-2-zero-downtime-oracle-to-aurora-migration)
- [Scenario 3: Hybrid Cloud Data Sync Architecture](#scenario-3-hybrid-cloud-data-sync-architecture)
- [Scenario 4: Disaster Recovery Strategy Với AWS](#scenario-4-disaster-recovery-strategy-với-aws)
- [Scenario 5: Offline Data Migration Cho Môi Trường Không Có Mạng](#scenario-5-offline-data-migration-cho-môi-trường-không-có-mạng)
- [Framework Trả Lời System Design](#framework-trả-lời-system-design)

---

## Scenario 1: Enterprise Lift-and-Shift 500 Servers

### Đề Bài

```
Khách hàng: Công ty logistics (vận tải) có 500 servers on-premises tại 2 datacenter
            ở Hà Nội và TP.HCM.

Yêu cầu:
  - Di chuyển toàn bộ lên AWS Region ap-southeast-1 (Singapore) trong 12 tháng
  - Hệ thống đặt hàng và tracking (theo dõi đơn hàng) phải hoạt động 24/7
  - Không muốn thay đổi code ứng dụng (budget thấp cho development)
  - Chi phí AWS phải thấp hơn on-premises hiện tại

Thông tin bổ sung (khi bạn hỏi):
  - 500 servers: 300 application, 150 database, 50 infrastructure (monitoring, DNS, v.v.)
  - Kết nối internet hiện tại: 500 Mbps tại HN, 300 Mbps tại HCM
  - Compliance: không có yêu cầu đặc biệt về data residency
  - Budget migration: $2 triệu USD
  - Team: 5 engineers nội bộ, có thể thuê 3 AWS migration partners

Hỏi: Thiết kế migration strategy và architecture.
```

### Clarifying Questions (Câu Hỏi Làm Rõ) Cần Hỏi

```
Trước khi thiết kế, hãy hỏi:
1. "Hệ thống nào là mission-critical (thiết yếu) nhất? Downtime ảnh hưởng doanh thu như thế nào?"
2. "Có dependency (phụ thuộc) nào giữa servers HN và HCM không?"
3. "Database engine nào đang dùng? (MySQL, Oracle, SQL Server?)"
4. "Có bất kỳ workload nào phụ thuộc phần cứng đặc biệt không? (GPU, FPGA?)"
5. "RPO và RTO của các hệ thống production là bao nhiêu?"
```

### Giải Pháp Tham Khảo

```
KIẾN TRÚC TỔNG QUAN:

[On-premises HN/HCM] ←→ [AWS Direct Connect 1 Gbps] ←→ [AWS ap-southeast-1]
                                                              ├── Production VPC
                                                              │   ├── EC2 instances (migrated)
                                                              │   ├── RDS instances
                                                              │   └── Load Balancers
                                                              └── Migration VPC
                                                                  └── MGN Staging Area

GIAI ĐOẠN 1 — ASSESS (Tuần 1-4):
  ┌─────────────────────────────────────────────────────────┐
  │ Công Cụ                  │ Mục Đích                     │
  ├─────────────────────────────────────────────────────────┤
  │ Application Discovery    │ Inventory 500 servers,        │
  │ Service (ADS)            │ map dependencies              │
  │ Migration Evaluator      │ Tính TCO, đề xuất instance    │
  │                          │ types right-sized             │
  │ AWS Migration Hub        │ Setup tracking dashboard       │
  └─────────────────────────────────────────────────────────┘

  → Phân loại theo 7Rs:
    Retire: ~50 servers (không còn dùng → tắt ngay)
    Rehost: ~350 servers dùng AWS MGN
    Replatform: ~50 DB servers → RDS managed
    Retain: ~50 servers (nếu có compliance hoặc latency issues)

GIAI ĐOẠN 2 — MOBILIZE (Tuần 5-10):
  - Thiết lập AWS Landing Zone: AWS Control Tower
  - Network:
    → AWS Direct Connect 1 Gbps từ HN (primary — chính)
    → Site-to-Site VPN từ HCM (backup — dự phòng) trong khi chờ Direct Connect HCM
  - Thiết lập DNS hybrid: Route 53 Resolver + on-premises DNS
  - AWS Migration Hub: tích hợp với MGN và DMS

GIAI ĐOẠN 3 — MIGRATION WAVES (Tuần 11-52):

  WAVE 1 (Tuần 11-14): Non-critical, infrastructure
    → Dev/test environments (không sợ mất)
    → Internal monitoring servers
    → File servers → AWS EFS hoặc S3

  WAVE 2 (Tuần 15-22): Business non-critical
    → Web servers, API gateways (không trực tiếp xử lý đơn hàng)
    → Reporting và analytics databases
    → Email, intranet

  WAVE 3 (Tuần 23-36): Business critical applications
    → Hệ thống tracking đơn hàng
    → Application servers core logistics
    → Database servers (MySQL → RDS MySQL — homogeneous, ít rủi ro)

  WAVE 4 (Tuần 37-48): Mission critical + special databases
    → Core order management system
    → Databases phức tạp
    → Payment gateway integrations

  WAVE 5 (Tuần 49-52): Cutover hoàn tất
    → Tắt on-premises servers đã migrate
    → Optimize (tối ưu) costs: Reserved Instances, Savings Plans
    → Đóng cửa hoặc bán lại datacenter equipment

CÔNG CỤ CHÍNH:
  Rehost (lift-and-shift):   AWS MGN (Application Migration Service)
  Replatform DB:             AWS DMS (homogeneous MySQL → RDS MySQL)
  Monitoring:                AWS Migration Hub + CloudWatch
  Network:                   Direct Connect (ưu tiên) + VPN (backup)
```

### Trade-offs Cần Nêu

```
TRADE-OFF 1: Direct Connect vs Internet
  Direct Connect: Đắt hơn (~$1,000-2,000/tháng) nhưng ổn định, thấp latency
  Internet: Rẻ hơn nhưng bandwidth không đảm bảo khi migrate hàng TB

TRADE-OFF 2: Migration Speed vs Risk
  Nhanh hơn → nhiều waves nhỏ hơn → nhiều coordination overhead
  Chậm hơn → ít rủi ro → nhưng datacenter costs kéo dài

TRADE-OFF 3: Rehost vs Replatform
  Rehost: Nhanh, ít rủi ro → nhưng không tối ưu chi phí cloud
  Replatform (RDS): Tốt hơn long-term nhưng cần test migration kỹ hơn
```

---

## Scenario 2: Zero-Downtime Oracle-to-Aurora Migration

### Đề Bài

```
Khách hàng: Ngân hàng thương mại cổ phần

Database hiện tại:
  - Oracle Database 19c, 8 TB, on-premises
  - 2,000 giao dịch/giây vào giờ cao điểm
  - 200 stored procedures, 50 triggers
  - Đang chạy Oracle Real Application Clusters (RAC — Cụm Ứng Dụng Thực)

Yêu cầu:
  - Target: Amazon Aurora PostgreSQL (tiết kiệm Oracle license)
  - RPO = 0 (không mất dữ liệu)
  - RTO = 30 phút (tối đa 30 phút downtime cho maintenance window)
  - Phải pass security audit (kiểm toán bảo mật) PCI-DSS trước khi go-live
  - Deadline: 6 tháng

Hỏi: Thiết kế migration architecture và plan chi tiết.
```

### Giải Pháp Tham Khảo

```
KIẾN TRÚC MIGRATION:

[Oracle RAC on-premises]
       │
       │ Oracle LogMiner (đọc redo logs)
       ↓
[AWS DMS Replication Instance r5.4xlarge]
       │
       ├── Full Load Task (giai đoạn 1)
       │   └→ [Aurora PostgreSQL (target)]
       │
       └── CDC Task (giai đoạn 2 — ongoing)
           └→ [Aurora PostgreSQL (target)]

BƯỚC 1: SCHEMA CONVERSION (4-6 tuần)

  Chạy AWS SCT (Schema Conversion Tool):
  - Analyze 200 stored procedures + 50 triggers
  - SCT report: ~65% tự động convert được
  - ~35% cần manual rewrite (viết lại thủ công)
  
  Phân công:
  - SCT auto-convert: Oracle DBA review và validate
  - Manual rewrite: Team gồm 2 Oracle DBA + 2 PostgreSQL DBA
  - Testing framework: pgTAP unit tests cho từng stored procedure
  
  Chuyển đổi đặc biệt cần chú ý:
    Oracle → PostgreSQL:
    - ROWNUM → ROW_NUMBER() OVER() hoặc LIMIT/OFFSET
    - CONNECT BY → WITH RECURSIVE (CTE đệ quy)
    - Oracle sequences → PostgreSQL sequences (tương tự nhưng cú pháp khác)
    - VARCHAR2 → VARCHAR
    - NUMBER → NUMERIC hoặc BIGINT tùy precision
    - SYSDATE → CURRENT_TIMESTAMP
    - NVL() → COALESCE()
    - DECODE() → CASE WHEN

BƯỚC 2: AURORA SETUP (1 tuần)

  - Aurora PostgreSQL cluster: 1 writer + 2 readers trong multi-AZ
  - Instance type: db.r6g.4xlarge (tương đương Oracle RAC về compute)
  - Storage: Aurora auto-scaling (bắt đầu từ 10 GB, tự động tăng)
  - Encryption: AWS KMS customer-managed key
  - Parameter group: tuning cho high-throughput OLTP
  - Enhanced Monitoring + Performance Insights bật

BƯỚC 3: DMS FULL LOAD (2-3 tuần)

  - DMS Replication Instance: r5.4xlarge (nhiều RAM cho large LOB columns)
  - Network: Direct Connect hoặc VPN với dedicated bandwidth
  - Task settings:
    → LOB (Large Object) mode: limited LOB (giới hạn kích thước LOB) để tăng tốc
    → Parallel load: 8 threads trên bảng lớn nhất
    → Table sort order: load bảng nhỏ trước, bảng lớn sau
  - Thời gian ước tính: 8 TB × 8 bits / throughput DMS ≈ 2-3 ngày

BƯỚC 4: CDC REPLICATION (4-6 tuần chạy song song)

  - DMS CDC task start from: SCN (System Change Number — Số Thay Đổi Hệ Thống)
    tại thời điểm bắt đầu full load
  - Monitor replication lag: target < 30 giây trong steady state
  - CloudWatch alarm: nếu lag > 60 giây → page on-call engineer

BƯỚC 5: VALIDATION (song song trong 4-6 tuần)

  Lớp 1: Row count validation tự động hàng ngày
  Lớp 2: Business logic testing: chạy 100 test cases (tình huống kiểm thử) tài chính
  Lớp 3: Performance testing: load test Aurora với synthetic 2,000 TPS (giao dịch/giây)
  Lớp 4: Security audit: PCI-DSS checklist — encryption, access control, audit logging

BƯỚC 6: CUTOVER (Maintenance Window — Cửa Sổ Bảo Trì)

  Chọn thời điểm: Thứ Bảy 2:00 AM (traffic thấp nhất)
  
  Timeline cutover (23:00 Thứ Sáu → 00:30 Thứ Bảy):
  
    23:00 — Tắt batch jobs (công việc lô) đêm
    01:00 — Monitor lag → phải < 5 giây để proceed (tiếp tục)
    01:45 — Bật application maintenance mode
    01:46 — Stop write transactions (dừng giao dịch ghi) vào Oracle
    01:47 — Chờ DMS drain final transactions (~2-3 phút)
    01:50 — FINAL VALIDATION: row count, business critical records
    01:53 — Cập nhật connection string trỏ Aurora PostgreSQL
    01:55 — Re-enable application
    01:55-02:15 — Smoke test + monitoring
    02:15 — Confirm cutover success → thông báo stakeholders
    
  Actual downtime: ~28 phút (trong RTO 30 phút)
  
  ROLLBACK PLAN (nếu validation fail hoặc critical error):
    → Revert connection string trỏ Oracle (~5 phút)
    → Oracle vẫn còn nguyên vẹn (read-only trong maintenance window)
    → Post-mortem (phân tích sau sự cố) để tìm nguyên nhân
    → Reschedule cutover sau khi fix

BƯỚC 7: POST-CUTOVER (4 tuần)

  - Giữ Oracle running read-only 2 tuần phòng rollback
  - Monitor performance Aurora: query plan comparison
  - Rightsizing Aurora instance type sau 2 tuần dữ liệu thực
  - Decommission (tắt) Oracle sau khi confident
```

### Điểm Đánh Giá Cao Trong Interview

```
✅ Đề cập Oracle RAC → Aurora không 1-1 (RAC là active-active, Aurora là 1 writer)
✅ Nhắc đến SCT limitation cho stored procedures
✅ Chọn r5.4xlarge (memory-optimized) cho DMS với LOB data
✅ PCI-DSS audit trước go-live — hiểu compliance requirements
✅ Rollback plan cụ thể với thời gian
✅ Giữ Oracle read-only (không shutdown ngay) sau cutover
```

---

## Scenario 3: Hybrid Cloud Data Sync Architecture

### Đề Bài

```
Khách hàng: Công ty bảo hiểm

Tình huống:
  - Hệ thống core (quản lý hợp đồng) PHẢI ở on-premises vì quy định Bộ Tài Chính
  - Muốn dùng AWS cho analytics, AI/ML, và customer portal (cổng khách hàng)
  - Cần đồng bộ dữ liệu từ on-premises lên AWS trong thời gian thực (near real-time)
  - Dữ liệu nhạy cảm: thông tin hợp đồng, sức khỏe khách hàng, tài chính

Yêu cầu:
  - Latency đồng bộ: < 5 phút từ khi update on-premises đến khi có trên AWS
  - Bảo mật: Mã hóa end-to-end, audit log đầy đủ
  - Analytics team cần query dữ liệu tổng hợp, không cần real-time
  - Customer portal cần dữ liệu gần thực (eventual consistency chấp nhận được)

Hỏi: Thiết kế hybrid data sync architecture.
```

### Giải Pháp Tham Khảo

```
KIẾN TRÚC HYBRID SYNC:

[On-premises Core System]
       │ 
       │ Change Data Capture (CDC)
       ↓
[AWS DMS — CDC Task]
       │
       │ Encrypted (TLS) qua Direct Connect
       ↓
[Amazon Kinesis Data Streams] → [AWS Lambda] → [S3 Data Lake]
       │                                              │
       │                              ┌───────────────┤
       ↓                              ↓               ↓
[Amazon RDS / Aurora]          [Amazon Athena]  [Amazon Redshift]
(Customer Portal DB)           (Ad-hoc query)   (Analytics DW)
                                                      │
                                                      ↓
                                               [Amazon QuickSight]
                                               (BI Dashboard)

THÀNH PHẦN VÀ LÝ DO CHỌN:

1. NETWORK LAYER (Tầng Mạng):
   → AWS Direct Connect: băng thông đảm bảo, latency thấp, không qua internet
   → Backup: Site-to-Site VPN khi Direct Connect có sự cố
   → Traffic encryption: IPSec trên VPN, MACsec trên Direct Connect

2. DATA CAPTURE (Thu Thập Dữ Liệu):
   → AWS DMS CDC: capture thay đổi từ on-premises Oracle/PostgreSQL
   → DMS ghi ra Amazon Kinesis Data Streams (real-time stream)
   → Kinesis: buffer (bộ đệm) 24 giờ — tránh mất data nếu downstream bị chậm

3. DATA PROCESSING (Xử Lý Dữ Liệu):
   → Lambda consumer (tiêu thụ từ Kinesis):
     - PII masking (che giấu thông tin cá nhân): hash sensitive fields
     - Data validation: kiểm tra business rules
     - Routing: ghi vào đúng destination (điểm đến)

4. STORAGE LAYER (Tầng Lưu Trữ):
   → S3 Data Lake: dữ liệu thô đã masked, partition by date/type
   → Aurora PostgreSQL: phục vụ customer portal (read-heavy)
   → Redshift: analytics workload (query phức tạp, aggregation)

5. SECURITY CONTROLS (Kiểm Soát Bảo Mật):
   → KMS Customer Managed Keys cho mọi data at rest
   → VPC endpoints: DMS, Kinesis, S3 không qua internet
   → AWS CloudTrail: mọi API call được log
   → Amazon Macie: tự động detect PII trong S3
   → AWS Security Hub: tổng hợp security findings

LATENCY ĐẠT ĐƯỢC:
  On-premises change → DMS capture → Kinesis → Lambda → Aurora/S3
  ≈ 30 giây đến 2 phút (well within yêu cầu < 5 phút)

TRADE-OFFS:
  → DMS CDC có giới hạn throughput (thông lượng) — nếu >100,000 events/giây cần scaling
  → Kinesis shard (mảnh) cần tính toán: 1 shard = 1 MB/s write, 2 MB/s read
  → Data consistency (tính nhất quán dữ liệu): CDC đảm bảo at-least-once delivery,
    Lambda cần idempotent (có thể xử lý lặp mà không gây lỗi)
```

---

## Scenario 4: Disaster Recovery Strategy Với AWS

### Đề Bài

```
Khách hàng: Chuỗi bán lẻ 500 cửa hàng toàn quốc

Hệ thống hiện tại:
  - POS (Point of Sale — Hệ Thống Thanh Toán Tại Điểm Bán) kết nối datacenter trung tâm HN
  - Database: SQL Server 2019, 3 TB
  - RPO yêu cầu: 15 phút
  - RTO yêu cầu: 2 giờ
  - Không có DR site (cơ sở phục hồi thảm họa) hiện tại

Tình huống: Datacenter HN bị ngập lụt → toàn bộ hệ thống mất điện

Hỏi: Thiết kế DR strategy dùng AWS với RTO 2 giờ và RPO 15 phút.
```

### Giải Pháp Tham Khảo

```
DR STRATEGY: WARM STANDBY (Chờ Ấm — Không Phải Full Replica Nhưng Không Tắt Hẳn)

Lý do chọn Warm Standby:
  - Hot Standby (chạy đầy đủ): Đắt ~200% so với on-premises
  - Cold Standby (chỉ backup): RTO quá dài (4-8 giờ → không đạt 2 giờ)
  - Warm Standby: Chi phí trung bình, RTO 1-2 giờ ✅

ARCHITECTURE:

[On-premises HN - Primary]           [AWS ap-southeast-1 - DR Site]
       │                                       │
[SQL Server 2019]                    [Amazon RDS SQL Server - Read Replica]
       │                                       │ (sync liên tục qua DMS CDC)
       │ AWS DMS CDC                           │
       └──────────────────────────────────────→ RPO = realtime lag ≈ < 5 phút
                                               │
[Application Servers]                [EC2 instances (stopped — đã dừng)]
       │                                       │ (AMI (Amazon Machine Image) cập nhật hàng tuần)
       │ AWS MGN continuous replication        │
       └──────────────────────────────────────→ Có thể start trong 30 phút

[DNS: POS terminals → HN IP]        [Route 53 Health Check → failover record]

THÀNH PHẦN AWS:

1. Amazon RDS SQL Server (Multi-AZ trong AWS):
   → Read replica của SQL Server on-premises
   → AWS DMS CDC replication: lag < 5 phút → RPO đạt được
   → Instance: db.r6.2xlarge (scaled down — thu nhỏ so với production)
   → Promote (nâng lên) thành standalone trong 5-10 phút khi failover

2. EC2 AMIs (Ảnh Máy — Snapshot Ứng Dụng):
   → Mỗi tuần: tạo AMI từ application servers trên-premises
   → Khi DR: launch EC2 từ AMI → patch data từ cách đây 1 tuần
   → Thời gian: ~30-45 phút để EC2 chạy và connect được

3. Amazon Route 53 (DNS Failover — Chuyển Đổi DNS):
   → Health check (kiểm tra sức khỏe) endpoint on-premises mỗi 30 giây
   → Khi on-premises fail: tự động failover DNS record trỏ AWS ELB
   → TTL (Time To Live — Thời Gian Sống): 60 giây → DNS propagation nhanh

4. AWS Direct Connect + VPN Backup:
   → Bình thường: Direct Connect cho DMS replication
   → Khi DR: kết nối từ AWS ra internet (POS terminals connect về AWS qua internet)

FAILOVER RUNBOOK (Hướng Dẫn Chuyển Đổi Khẩn Cấp):

  T+0:  Nhận alert: on-premises datacenter không phản hồi
  T+5:  Confirm (xác nhận) sự cố thực (không phải false alarm)
  T+10: Declare DR (tuyên bố thực hiện DR) — thông báo stakeholders
  T+15: Promote RDS Read Replica thành standalone writer
  T+20: Launch EC2 instances từ latest AMIs
  T+35: EC2 ready → update RDS connection string
  T+40: Run smoke tests trên AWS environment
  T+50: Update Route 53 records (hoặc đã auto-update qua health check)
  T+60: POS terminals tự kết nối lại qua DNS → AWS environment
  T+90: Full validation → tất cả 500 cửa hàng online

Tổng RTO: ~90 phút ✅ (yêu cầu 2 giờ)
RPO thực tế: ~5-10 phút (DMS CDC lag) ✅ (yêu cầu 15 phút)

CHI PHÍ DR (Warm Standby):
  - RDS SQL Server (scaled-down): ~$800/tháng
  - EC2 AMI storage: ~$50/tháng
  - DMS replication: ~$200/tháng
  - Direct Connect (phần DR): chia sẻ với production
  Tổng: ~$1,100/tháng — rẻ hơn nhiều so với có datacenter DR vật lý
```

---

## Scenario 5: Offline Data Migration Cho Môi Trường Không Có Mạng

### Đề Bài

```
Khách hàng: Công ty khai thác dầu khí (oil & gas)

Tình huống:
  - Có 30 giàn khoan offshore (ngoài khơi) trên biển
  - Mỗi giàn thu thập [5 TB] sensor data (dữ liệu cảm biến)/tháng
  - Kết nối internet: satellite (vệ tinh) 10 Mbps — không ổn định, đắt tiền
  - Cần phân tích ML (machine learning) trên cloud để dự đoán sự cố thiết bị
  - Data center trên đất liền có Direct Connect 10 Gbps về AWS

Yêu cầu:
  - Transfer 30 × 5 TB = 150 TB/tháng lên S3 để chạy ML pipeline
  - Chi phí transfer tối thiểu
  - Độ trễ phân tích: chấp nhận 1-2 ngày delay (không cần real-time)

Hỏi: Thiết kế data ingestion architecture.
```

### Giải Pháp Tham Khảo

```
KIẾN TRÚC MULTI-LAYER DATA INGESTION:

LAYER 1 — EDGE (Cạnh — Trên Giàn Khoan):
  
  [Sensor Data Collection]
         ↓
  [AWS Snowcone] trên mỗi giàn
         │ - Thu thập và buffer data
         │ - Edge processing: filter noise, compress (nén)
         │ - DataSync agent tích hợp
         │
  Tùy chọn truyền:
  ├── Satellite link (10 Mbps):
  │   → Upload delta (chỉ thay đổi) daily → ~1-2 GB/ngày (sau compress)
  │   → DataSync incremental sync: chỉ gửi file mới/thay đổi
  │   → Backup channel, không phải primary
  │
  └── Tàu tiếp vận (supply vessel — ghé mỗi 2 tuần):
      → Physical transport: tháo Snowcone, giao cho đất liền
      → Datacenter đất liền: connect Snowcone, DataSync về S3 qua Direct Connect
      → Throughput: Direct Connect 10 Gbps → 14 TB trong ~3 giờ

LAYER 2 — EDGE ONSHORE (Trên Đất Liền):
  
  [Datacenter Đất Liền HN]
         │
         │ Direct Connect 10 Gbps
         ↓
  [AWS ap-southeast-1]
         │
         ├── S3 Raw Data Bucket (dữ liệu thô từ Snowcone)
         │   └── S3 Intelligent-Tiering (tự động optimize cost)
         │
         └── DataSync từ Satellite uploads → S3

LAYER 3 — AWS CLOUD:
  
  [S3 Raw Data]
       │
       ↓
  [AWS Glue] → ETL (Extract Transform Load — Trích Xuất Chuyển Đổi Nạp)
       │        Làm sạch, chuẩn hóa sensor data
       ↓
  [S3 Processed Data] → [Amazon SageMaker]
                               │ ML training:
                               │ - Predictive maintenance (bảo trì dự đoán)
                               │ - Anomaly detection (phát hiện bất thường)
                               ↓
                        [Model Predictions]
                               │
                               ↓
                        [Amazon QuickSight Dashboard]
                        (Engineers xem kết quả dự đoán)

TÍNH TOÁN LOGISTICS (LÁT CẮT MỘT THÁNG):

  30 giàn × 5 TB raw = 150 TB/tháng
  
  Sau khi compress (nén dữ liệu sensor ~70%): ~45 TB thực
  
  Method A — Satellite upload:
    45 TB / (10 Mbps × 0.5 hiệu suất satellite) = 7,200 giờ = 300 ngày ❌ Không khả thi
  
  Method B — Physical Snowcone (primary):
    30 Snowcones × 14 TB SSD = 420 TB capacity — đủ cho 1 tháng data
    Tàu tiếp vận mỗi 2 tuần: upload data của 2 tuần qua Direct Connect → vài giờ ✅
  
  Method C — Hybrid (recommended — được khuyến nghị):
    Primary: Snowcone + tàu tiếp vận (2 tuần/lần)
    Secondary: Satellite DataSync upload critical alerts (cảnh báo khẩn cấp) ngay lập tức
    → Alert thực sự khẩn cần real-time: ~10 MB/ngày → satellite đủ dùng

CHI PHÍ ƯỚC TÍNH:
  Snowcone rental: 30 × $[300]/tháng = $9,000/tháng
  Satellite upload (alerts only): ~$200/tháng
  S3 storage 45 TB: ~$1,000/tháng (Intelligent-Tiering)
  Glue + SageMaker: ~$500/tháng (ML training)
  Tổng: ~$10,700/tháng
  
  So sánh: Nâng cấp satellite lên 1 Gbps cho 30 giàn = $[200,000]+/tháng ❌
```

---

## 📋 Framework Trả Lời System Design

### Bước 1: Clarifying Questions (2-3 phút)

```
LUÔN HỎI TRƯỚC KHI THIẾT KẾ:

  Requirements (Yêu cầu):
    - Scale: bao nhiêu servers/TB/users?
    - Performance: RPO, RTO, latency requirements?
    - Deadline: bao lâu phải hoàn thành?

  Constraints (Ràng buộc):
    - Budget: bao nhiêu?
    - Team: bao nhiêu người, kỹ năng gì?
    - Compliance: có GDPR/HIPAA/PCI-DSS không?
    - Existing infrastructure: có Direct Connect chưa?

  Priorities (Ưu tiên):
    - Cost vs Speed vs Safety — cái nào quan trọng nhất?
    - Nếu trade-off: business sẽ chọn gì?
```

### Bước 2: High-Level Architecture (5 phút)

```
Vẽ/mô tả kiến trúc tổng quan:
  1. Xác định các thành phần chính
  2. Luồng dữ liệu (data flow)
  3. Chọn AWS services phù hợp (có lý do)
  4. Network connectivity: Direct Connect / VPN / Internet

Nếu không chắc: "Approach của tôi là X vì Y. Bạn có muốn explore alternative Z không?"
```

### Bước 3: Deep Dive (5-10 phút)

```
Đi sâu vào phần quan trọng nhất:
  - Migration strategy (7Rs) nếu large-scale
  - CDC vs Full Load nếu database migration
  - Bandwidth calculation nếu data transfer
  - Failover và rollback plan

Luôn đề cập:
  ✅ Monitoring plan
  ✅ Rollback / disaster recovery
  ✅ Cost estimates (ước tính chi phí)
  ✅ Timeline (lịch trình)
```

### Bước 4: Trade-offs (2-3 phút)

```
Chủ động nêu trade-offs — đây là điểm phân biệt Senior vs Junior:

  "Approach này có ưu điểm [A, B] nhưng cũng có nhược điểm [C, D].
   Alternative là [Y] — phù hợp hơn nếu ưu tiên [Z]."

Ví dụ trade-offs thường gặp:
  - Speed vs Cost: rush migration tốn thêm tiền thuê contractor
  - Safety vs Timeline: test cutover kỹ hơn → delay nhưng ít rủi ro
  - Fully managed vs Custom: RDS đơn giản hơn EC2 + MySQL tự quản lý nhưng ít linh hoạt
```

### Bước 5: Scale & Evolution (1-2 phút)

```
Nêu cách kiến trúc scale theo thời gian:
  - "Nếu data tăng 10x, chúng ta sẽ..."
  - "Sau migration hoàn tất, bước modernization tiếp theo là..."
  - "Cost optimization sau 3 tháng chạy ổn định: Reserved Instances, rightsizing"
```

---

## 📊 Bảng Tổng Hợp AWS Services Theo Use Case

| Vấn Đề | AWS Service | Lý Do Chọn |
| ------- | ----------- | ----------- |
| Lift-and-shift server | AWS MGN | Block-level replication, không cần sửa code |
| DB migration cùng engine | AWS DMS (homogeneous) | Không cần SCT, đơn giản |
| DB migration khác engine | AWS DMS + SCT | SCT chuyển schema, DMS transfer data |
| Zero-downtime DB migration | AWS DMS + CDC | Full load rồi sync liên tục |
| Transfer file qua mạng nhanh | AWS DataSync | Multi-thread, checksum, scheduling |
| Transfer > 10 TB, mạng kém | Snowball Edge | Physical, 80 TB, không cần internet |
| Transfer < 14 TB, vùng xa | Snowcone | Portable, nhỏ gọn |
| Transfer exabyte | Snowmobile | Xe tải AWS |
| SFTP server managed | AWS Transfer Family | Không cần quản lý server |
| Kết nối on-premises liên tục | AWS Storage Gateway | Hybrid, không ngừng công việc |
| Discovery hạ tầng | ADS | Agentless hoặc agent-based |
| Tính TCO | Migration Evaluator | Báo cáo tự động so sánh |
| Theo dõi migration progress | AWS Migration Hub | Dashboard tập trung |
| DR cho on-premises | AWS DRS | Continuous replication + failover |
| Modernize mainframe | AWS Mainframe Modernization | Refactor hoặc replatform COBOL |
| Containerize Java/.NET | AWS App2Container | Tự động detect và containerize |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành
