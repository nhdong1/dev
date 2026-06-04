# AWS Well-Architected Migration Lens — Đánh Giá Kiến Trúc Di Chuyển Theo Best Practices

> **AWS Well-Architected Migration Lens** là bộ câu hỏi và hướng dẫn thực hành tốt nhất (best practices) được thiết kế riêng cho các dự án migration lên AWS. Đây là phần mở rộng (Lens — Góc Nhìn Chuyên Biệt) của **AWS Well-Architected Framework** — framework 6 trụ cột nổi tiếng của AWS. Migration Lens giúp các team xác định rủi ro (risks), khoảng thiếu hụt (gaps), và những điểm cần cải thiện **trước, trong, và sau** quá trình di chuyển, đảm bảo workload được migration một cách an toàn, hiệu quả và tuân theo best practices của AWS.

## 📚 Mục Lục (Table of Contents)

1. [Well-Architected Framework Là Gì?](#well-architected-framework-là-gì)
2. [Migration Lens — Tổng Quan Và Mục Đích](#migration-lens--tổng-quan-và-mục-đích)
3. [6 Pillars Trong Bối Cảnh Migration](#6-pillars-trong-bối-cảnh-migration)
4. [Ba Giai Đoạn Migration Trong Lens](#ba-giai-đoạn-migration-trong-lens)
5. [Pillar 1: Operational Excellence](#pillar-1-operational-excellence)
6. [Pillar 2: Security](#pillar-2-security)
7. [Pillar 3: Reliability](#pillar-3-reliability)
8. [Pillar 4: Performance Efficiency](#pillar-4-performance-efficiency)
9. [Pillar 5: Cost Optimization](#pillar-5-cost-optimization)
10. [Pillar 6: Sustainability](#pillar-6-sustainability)
11. [Quy Trình Thực Hiện Well-Architected Review](#quy-trình-thực-hiện-well-architected-review)
12. [High Risk Items — Xử Lý Rủi Ro Cao](#high-risk-items--xử-lý-rủi-ro-cao)
13. [Migration Readiness Assessment (MRA)](#migration-readiness-assessment-mra)
14. [Công Cụ AWS Well-Architected Tool](#công-cụ-aws-well-architected-tool)
15. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏗️ Well-Architected Framework Là Gì?

### Tổng Quan Framework

```
AWS WELL-ARCHITECTED FRAMEWORK:

AWS Well-Architected Framework là bộ hướng dẫn kiến trúc gồm 6 trụ cột (pillars)
được xây dựng từ kinh nghiệm thực tế của AWS với hàng nghìn khách hàng.

Mục đích: Giúp cloud architects (kiến trúc sư đám mây) xây dựng
hệ thống đáng tin cậy, an toàn, hiệu quả và tối ưu chi phí.

6 PILLARS (6 TRỤ CỘT):
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. Operational Excellence   (Xuất Sắc Vận Hành)                           │
│  2. Security                 (Bảo Mật)                                      │
│  3. Reliability              (Độ Tin Cậy)                                   │
│  4. Performance Efficiency   (Hiệu Quả Hiệu Năng)                          │
│  5. Cost Optimization        (Tối Ưu Chi Phí)                              │
│  6. Sustainability           (Bền Vững — Môi Trường)                        │
└─────────────────────────────────────────────────────────────────────────────┘

LENS (KÍNH / GÓC NHÌN CHUYÊN BIỆT):
AWS cung cấp nhiều Lens để áp dụng Framework cho lĩnh vực cụ thể:
├── Migration Lens ← CHÚNG TA ĐANG HỌC
├── Serverless Lens
├── SaaS Lens
├── IoT Lens
├── Machine Learning Lens
└── ... và nhiều Lens khác
```

### Tại Sao Cần Migration Lens Riêng?

```
WELL-ARCHITECTED FRAMEWORK THÔNG THƯỜNG vs MIGRATION LENS:

Framework thông thường hỏi:
"Hệ thống của bạn có auto-scaling không?"
"Bạn có multi-AZ deployment không?"
→ Phù hợp cho workloads ĐÃ chạy trên AWS

Migration Lens hỏi THÊM:
"Bạn đã test cutover chưa? Kế hoạch rollback là gì?"
"Dependencies giữa apps đã được xác định và migration wave đã lên kế hoạch chưa?"
"Bạn có runbook (tài liệu vận hành) cho từng app khi migration không?"
→ Phù hợp cho quá trình CHUYỂN ĐỔI từ on-premises lên AWS

Migration Lens tập trung vào RỦI RO ĐẶC THÙ của migration:
├── Downtime risk (rủi ro gián đoạn dịch vụ)
├── Data integrity risk (rủi ro mất hoặc sai dữ liệu)
├── Dependency management (quản lý phụ thuộc)
├── Security during migration (bảo mật trong quá trình chuyển đổi)
└── Cost explosion risk (rủi ro chi phí bùng nổ nếu không kiểm soát)
```

---

## 🔭 Migration Lens — Tổng Quan Và Mục Đích

### Khi Nào Chạy Migration Lens Review

```
THỜI ĐIỂM CHẠY WELL-ARCHITECTED REVIEW VỚI MIGRATION LENS:

PHASE 1: ASSESS (Đánh Giá) — cuối giai đoạn
└── Đánh giá kiến trúc mục tiêu đề xuất
    → Tìm gaps trước khi bắt đầu di chuyển
    → Lên kế hoạch remediation (khắc phục)

PHASE 2: MOBILIZE (Chuẩn Bị) — trong giai đoạn
└── Review landing zone (môi trường AWS nền tảng)
    → Đảm bảo nền tảng AWS sẵn sàng đón workloads
    → Kiểm tra security controls

PHASE 3: MIGRATE & MODERNIZE — sau từng wave (đợt)
└── Review sau mỗi application wave hoàn thành
    → Học từ bài học thực tế
    → Cải thiện cho wave tiếp theo

POST-MIGRATION — sau khi xong
└── Full review workloads đã migrate
    → Tối ưu hóa cho vận hành lâu dài
```

### Output Của Migration Lens Review

```
KẾT QUẢ SAU KHI CHẠY MIGRATION LENS REVIEW:

1. HIGH RISK ITEMS (Mục Rủi Ro Cao) — màu đỏ:
   Ví dụ: "Không có kế hoạch rollback nếu cutover thất bại"
   → Phải giải quyết TRƯỚC khi migration

2. MEDIUM RISK ITEMS (Mục Rủi Ro Trung Bình) — màu vàng:
   Ví dụ: "Chưa có monitoring đầy đủ cho migrated workloads"
   → Nên giải quyết trong vòng 30-90 ngày

3. IMPROVEMENT PLAN (Kế Hoạch Cải Thiện):
   Danh sách các action items (việc cần làm) với:
   ├── Mô tả vấn đề
   ├── Giải pháp đề xuất
   └── Links đến AWS documentation và best practices

4. MILESTONE TRACKER:
   Theo dõi tiến độ giải quyết các risk items theo thời gian
```

---

## 📐 6 Pillars Trong Bối Cảnh Migration

```
6 PILLARS + CÂU HỎI MIGRATION ĐẶC TRƯNG:

PILLAR 1: OPERATIONAL EXCELLENCE
"Bạn vận hành tốt không?"
→ Migration focus: Runbooks, cutover plans, rollback procedures

PILLAR 2: SECURITY
"Bạn có bảo vệ data và systems không?"
→ Migration focus: Data protection in transit, IAM least privilege,
                    encryption, network security

PILLAR 3: RELIABILITY
"Hệ thống có phục hồi từ sự cố không?"
→ Migration focus: Test cutover, DR plan, backup before migration,
                    multi-AZ design

PILLAR 4: PERFORMANCE EFFICIENCY
"Bạn dùng tài nguyên hiệu quả không?"
→ Migration focus: Right-sizing, performance testing, Auto Scaling setup

PILLAR 5: COST OPTIMIZATION
"Bạn trả đúng giá cho những gì dùng không?"
→ Migration focus: Avoid zombie resources, Reserved Instances/Savings Plans plan,
                    cost monitoring

PILLAR 6: SUSTAINABILITY
"Bạn giảm thiểu tác động môi trường không?"
→ Migration focus: Graviton instances, serverless options, regional selection
```

---

## ⚙️ Pillar 1: Operational Excellence

### Câu Hỏi Migration Lens Cho OE

```
OPERATIONAL EXCELLENCE — CÂU HỎI VÀ CHECKLIST:

OE 1: Bạn có hiểu đầy đủ về workloads sẽ được migrate không?
─────────────────────────────────────────────────────────────
□ Application inventory đầy đủ và up-to-date
□ Dependency map đã được validate với application owners
□ Tài liệu kiến trúc (architecture documentation) hiện tại
□ SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) requirements rõ ràng
□ Business criticality (mức độ quan trọng với kinh doanh) của từng app

OE 2: Bạn có tài liệu vận hành đầy đủ cho migration không?
───────────────────────────────────────────────────────────
□ Runbook (tài liệu vận hành từng bước) cho từng application migration
□ Cutover checklist (danh sách kiểm tra khi chuyển đổi)
□ Rollback procedure (quy trình quay lại) documented
□ Communication plan (kế hoạch truyền thông) — ai thông báo gì cho ai khi nào
□ Escalation path (đường leo thang vấn đề) nếu có sự cố

OE 3: Bạn có tổ chức migration team đúng cách không?
────────────────────────────────────────────────────
□ RACI matrix (Responsible/Accountable/Consulted/Informed) cho migration
□ War room (phòng kiểm soát) hoặc war channel (kênh liên lạc) trong Slack/Teams
□ On-call rotation (ca trực) trong ngày cutover
□ AWS Support plan phù hợp (Business hoặc Enterprise) đã được kích hoạt

OE 4: Bạn có monitoring đầy đủ ngay sau khi migrate không?
──────────────────────────────────────────────────────────
□ CloudWatch alarms cho CPU, RAM, disk, network thresholds
□ Application-level metrics (ví dụ: response time, error rate)
□ Log aggregation (tập hợp logs) vào CloudWatch Logs hoặc OpenSearch
□ Dashboard (bảng điều khiển) để xem tổng quan health sau migration
□ Alert routing (định tuyến cảnh báo) đến đúng team
```

### Ví Dụ Cutover Runbook Template

```
CUTOVER RUNBOOK — MẪU ĐƠN GIẢN:

Application: E-Commerce API Server
Migration Date: 2026-07-15, 22:00 GMT+7
Cutover Window: 22:00 - 02:00 (4 giờ)
Rollback Deadline: 01:00 (quyết định rollback trước 01:00 để kịp)

PRE-CUTOVER CHECKS (Kiểm Tra Trước Khi Chuyển Đổi):
T-48h:
□ [infra-team] Xác nhận MGN replication đang đồng bộ < 1 phút lag
□ [dba-team] Backup database lần cuối và verify restore
□ [app-team] Smoke test (kiểm tra nhanh) trên test instance đã migrate

T-2h:
□ [infra-team] Notify stakeholders (thông báo bên liên quan): migration bắt đầu lúc 22:00
□ [dba-team] Stop application writes, tạo final snapshot
□ [network-team] Chuẩn bị DNS change (thay đổi DNS)

CUTOVER STEPS (Các Bước Chuyển Đổi):
22:00 □ [app-team] Stop application on on-premises
22:05 □ [dba-team] Verify final DMS sync completed
22:10 □ [mgn-team] Launch target EC2 instance (MGN cutover)
22:20 □ [app-team] Configure application on new EC2 instance
22:30 □ [app-team] Run smoke tests on new EC2
22:45 □ [network-team] Update DNS to point to new EC2 IP
22:50 □ [app-team] Run full integration tests
23:00 □ [all] Monitor: CloudWatch metrics, application logs
23:30 □ [all] Decision point: GO (tiếp tục) / ROLLBACK (quay lại)

ROLLBACK PROCEDURE (Quy Trình Quay Lại):
01:00 rollback deadline
□ [network-team] Revert DNS to on-premises IP
□ [app-team] Restart on-premises application
□ [dba-team] Stop DMS task, assess data sync state
□ [mgn-team] Note: on-premises server vẫn chạy (không tắt)
□ [infra-team] Notify stakeholders: rollback executed
```

---

## 🔐 Pillar 2: Security

### Câu Hỏi Migration Lens Cho Security

```
SECURITY — CÂU HỎI VÀ CHECKLIST:

SEC 1: Dữ liệu có được bảo vệ trong quá trình migration không?
──────────────────────────────────────────────────────────────
□ Replication traffic (MGN, DMS) được mã hóa TLS in-transit
□ S3 buckets dùng cho migration data có encryption at-rest (mã hóa khi lưu)
□ Sensitive data (PII, PHI, financial data) được identify trước khi migrate
□ Data classification (phân loại dữ liệu) đã được thực hiện

SEC 2: IAM (Identity and Access Management — Quản Lý Danh Tính Và Truy Cập)
────────────────────────────────────────────────────────────────────────────
□ Migration tools (MGN agent, DMS replication instance) dùng IAM roles
  với least privilege (quyền tối thiểu cần thiết)
□ Không dùng root account hoặc admin user cho migration tasks
□ Break-glass (tài khoản khẩn cấp) accounts được tạo và secured
□ Temporary elevated permissions (quyền tạm thời cao hơn) cho migration
  có thời hạn cụ thể và được revoke sau

SEC 3: Network Security (Bảo Mật Mạng)
───────────────────────────────────────
□ Security Groups cho EC2 migrated instances restrict traffic đúng
□ NACLs (Network Access Control Lists — Danh Sách Kiểm Soát Truy Cập Mạng)
  được review
□ VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) design đảm bảo
  proper isolation (cách ly đúng cách)
□ No public-facing instances trừ khi cần thiết (load balancers qua ALB/NLB)
□ AWS Direct Connect hoặc VPN cho migration traffic
  (không qua public internet cho data nhạy cảm)

SEC 4: Compliance (Tuân Thủ Pháp Lý)
──────────────────────────────────────
□ Xác nhận AWS region phù hợp với data residency requirements
  (yêu cầu lưu trữ dữ liệu theo địa lý — ví dụ: dữ liệu EU phải ở eu-west-1)
□ AWS Config rules được bật để phát hiện non-compliant configurations
□ CloudTrail (theo dõi API calls) được bật ở tất cả regions
□ GuardDuty (phát hiện mối đe dọa) được kích hoạt
```

### Security Anti-Patterns Phổ Biến Trong Migration

```
⚠️ NHỮNG LỖI BẢO MẬT HAY GẶP KHI MIGRATION:

1. Hardcoded credentials (thông tin xác thực nhúng cứng trong code):
   Problem: Ứng dụng on-premises dùng database password hardcoded trong config file
   → Khi migrate lên AWS, file config này vẫn có password dạng plain text
   Solution: Migrate sang AWS Secrets Manager (dịch vụ quản lý bí mật)
   hoặc AWS Systems Manager Parameter Store

2. Overly permissive Security Groups:
   Problem: "Để nhanh, mở tất cả 0.0.0.0/0 vào, sau refine sau"
   → "Sau" thường không bao giờ xảy ra
   Solution: Đặt Security Groups đúng ngay từ đầu theo principle of least privilege

3. Public S3 buckets không cần thiết:
   Problem: Tạo S3 bucket để chứa migration artifacts, quên set private
   → Dữ liệu migration (có thể có server config, credentials) bị exposed
   Solution: Block Public Access by default, dùng S3 bucket policies chặt chẽ

4. Bỏ qua encryption for "temporary" data:
   Problem: "Data migration chỉ tạm thời, không cần mã hóa"
   → Vi phạm compliance requirements
   Solution: Luôn encrypt migration data, dù chỉ tạm thời

5. Không disable migration agent sau khi xong:
   Problem: MGN replication agent vẫn có IAM permissions rộng sau migration
   Solution: Revoke IAM permissions và uninstall agent sau khi cutover confirmed
```

---

## 🔄 Pillar 3: Reliability

### Câu Hỏi Migration Lens Cho Reliability

```
RELIABILITY — CÂU HỎI VÀ CHECKLIST:

REL 1: Kiến trúc đích có đảm bảo high availability (HA — Tính Khả Dụng Cao)?
──────────────────────────────────────────────────────────────────────────────
□ Web/App tiers: ít nhất 2 EC2 instances trong 2 AZs (Availability Zones — Vùng Khả Dụng)
□ Database: Multi-AZ RDS hoặc Aurora với read replica
□ Load balancer (ALB/NLB) đặt trước instances
□ Auto Scaling Groups (ASG — Nhóm Tự Động Co Giãn) cấu hình
□ Health checks (kiểm tra sức khỏe) được cấu hình đúng

REL 2: Bạn đã test cutover chưa?
─────────────────────────────────
□ Test cutover (chuyển đổi thử nghiệm) đã được thực hiện ít nhất 1 lần
□ Application hoạt động đúng trên AWS sau test cutover
□ Rollback từ test cutover đã được test
□ Performance benchmark (đo hiệu năng) trên AWS tương đương on-premises

REL 3: Backup và Recovery đã sẵn sàng chưa?
────────────────────────────────────────────
□ Backup on-premises đã được tạo TRƯỚC khi migration (không thể nhấn undo)
□ RDS automated backups được bật
□ EC2 AMI snapshots được schedule
□ AWS Backup policies được cấu hình
□ Recovery time tested (đo thời gian phục hồi thực tế, không chỉ lý thuyết)

REL 4: Dependency management (Quản Lý Phụ Thuộc)?
──────────────────────────────────────────────────
□ Migration wave plan đảm bảo di chuyển theo đúng thứ tự dependencies
□ Không di chuyển app layer trước data layer
□ Cross-wave dependencies (phụ thuộc giữa các đợt) được xử lý
  (ví dụ: app trên AWS kết nối tạm về DB on-premises trong hybrid period)
```

### Kiến Trúc HA Chuẩn Sau Migration

```
KIẾN TRÚC HA SAU MIGRATION — THIẾT KẾ CHUẨN:

                    INTERNET
                        │
              ┌─────────▼──────────┐
              │    Route 53        │ (DNS với health checks)
              │    (DNS Routing)   │
              └─────────┬──────────┘
                        │
              ┌─────────▼──────────┐
              │  Application Load  │
              │  Balancer (ALB)    │
              └────┬──────────┬────┘
                   │          │
         ┌─────────▼──┐  ┌───▼─────────┐
         │  EC2 (AZ-a) │  │ EC2 (AZ-b)  │ (Auto Scaling Group)
         │  Web + App  │  │ Web + App   │
         └─────────┬───┘  └─────┬───────┘
                   │            │
         ┌─────────▼────────────▼─────────┐
         │       Amazon Aurora            │
         │   Writer (AZ-a) ←→ Reader (AZ-b)│ (Multi-AZ, automatic failover)
         └────────────────────────────────┘

Với kiến trúc này:
├── AZ failure: Auto failover trong < 60 giây (Aurora), < 5 phút (RDS Multi-AZ)
├── Instance failure: Auto Scaling thay thế instance mới
├── AZ-level outage: Load Balancer tự chuyển traffic sang AZ còn lại
└── Region failure: Cần DR plan riêng (ví dụ: cross-region replication)
```

---

## ⚡ Pillar 4: Performance Efficiency

### Câu Hỏi Migration Lens Cho Performance

```
PERFORMANCE EFFICIENCY — CÂU HỎI VÀ CHECKLIST:

PERF 1: Right-sizing đã được thực hiện chưa?
─────────────────────────────────────────────
□ Dùng ADS + Migration Evaluator để right-size EC2 instances
□ Không ánh xạ 1:1 theo capacity tối đa on-premises
□ Xem xét Graviton (ARM-based) instances cho workloads tương thích
  → Tiết kiệm 10-20% chi phí với hiệu năng tương đương
□ Database instance sizing dựa trên actual metrics (không phải capacity)

PERF 2: Performance testing (Kiểm Tra Hiệu Năng) đã được thực hiện?
─────────────────────────────────────────────────────────────────────
□ Load testing (kiểm tra tải) trên AWS environment trước cutover
□ Baseline performance metrics trên on-premises đã được đo
□ So sánh response time, throughput trên AWS vs on-premises
□ Database query performance không bị degraded (suy giảm)

PERF 3: Kiến trúc có tận dụng AWS capabilities chưa?
──────────────────────────────────────────────────────
□ S3 Transfer Acceleration (tăng tốc truyền tải S3) nếu cần upload nhanh
□ CloudFront (CDN) cho static assets nếu web app
□ ElastiCache (Redis/Memcached) thay vì self-managed cache on EC2
□ Aurora Serverless nếu workload có traffic biến động lớn

PERF 4: Auto Scaling đã được cấu hình?
────────────────────────────────────────
□ Auto Scaling Groups với scaling policies đã setup
□ Target tracking policies (theo dõi mục tiêu): ví dụ giữ CPU ở 70%
□ Scheduled scaling (co giãn theo lịch) nếu có predictable peaks
□ Scale-in protection cho instances đang xử lý long-running jobs
```

---

## 💸 Pillar 5: Cost Optimization

### Câu Hỏi Migration Lens Cho Cost

```
COST OPTIMIZATION — CÂU HỎI VÀ CHECKLIST:

COST 1: Tránh chi phí không cần thiết trong giai đoạn migration?
─────────────────────────────────────────────────────────────────
□ Development/test environments được TẮTOFF ngoài giờ làm việc
  (tiết kiệm 60-70% so với chạy 24/7)
□ Không để MGN replication chạy lâu hơn cần thiết sau khi đã plan
□ Xóa temporary resources (tài nguyên tạm thời) sau khi migration xong:
  MGN replication servers, DMS replication instances, test EC2 instances
□ S3 buckets có migration data cần lifecycle policy hoặc xóa sau migration

COST 2: Kế hoạch tiết kiệm sau khi ổn định?
────────────────────────────────────────────
□ Có kế hoạch chuyển sang Savings Plans hoặc Reserved Instances
  sau 3-6 tháng ổn định
□ AWS Cost Explorer (Khám Phá Chi Phí) được bật để theo dõi
□ Budget alerts (cảnh báo ngân sách) được thiết lập
□ Cost allocation tags (thẻ phân bổ chi phí) để biết workload nào tốn bao nhiêu

COST 3: Quản lý chi phí on-premises sau migration?
───────────────────────────────────────────────────
□ Kế hoạch decommission (ngừng hoạt động) on-premises servers sau migration
  (QUAN TRỌNG: chi phí AWS + chi phí on-premises = tốn gấp đôi)
□ Timeline rõ ràng: sau X tháng on-premises được tắt
□ Lease/maintenance contracts được cancel đúng hạn

COST 4: Tối ưu storage costs?
──────────────────────────────
□ S3 Intelligent-Tiering cho data access patterns không rõ ràng
□ EBS volumes: gp2 → gp3 migration (gp3 rẻ hơn ~20%, hiệu năng tốt hơn)
□ EBS snapshots cũ được xóa (orphan snapshots — snapshot mồ côi)
□ Unattached EBS volumes (volume không gắn vào instance nào) được xóa
```

### Chi Phí Ẩn Sau Migration Cần Chú Ý

```
ZOMBIE RESOURCES (TÀI NGUYÊN ZOMBIE) — CHI PHÍ ẨN SAU MIGRATION:

Zombie resources là các tài nguyên AWS không được dùng nhưng vẫn bị tính phí.
Thường xuất hiện sau migration khi team không dọn dẹp đúng cách.

Loại phổ biến nhất:
┌────────────────────────────────┬──────────────────────────────────────────┐
│ Zombie Resource                │ Lý Do Xuất Hiện                          │
├────────────────────────────────┼──────────────────────────────────────────┤
│ Stopped EC2 instances          │ Test instance, "tắt tạm" nhưng quên xóa  │
│ (vẫn tính tiền EBS)            │                                          │
├────────────────────────────────┼──────────────────────────────────────────┤
│ Unattached EBS volumes         │ Xóa EC2 nhưng quên xóa volume đính kèm  │
├────────────────────────────────┼──────────────────────────────────────────┤
│ Old EBS snapshots              │ Manual snapshots không có retention policy│
├────────────────────────────────┼──────────────────────────────────────────┤
│ Unused Elastic IPs             │ Tạo EIP cho test, quên associate          │
│ (tính phí $0.005/hr nếu không  │                                          │
│ gắn vào running instance)      │                                          │
├────────────────────────────────┼──────────────────────────────────────────┤
│ Idle DMS replication instances │ Migration xong nhưng không xóa           │
├────────────────────────────────┼──────────────────────────────────────────┤
│ MGN replication servers        │ Cutover xong, quên finalize/terminate     │
└────────────────────────────────┴──────────────────────────────────────────┘

Giải pháp: AWS Cost Anomaly Detection + AWS Trusted Advisor + periodic cleanup
```

---

## 🌱 Pillar 6: Sustainability

### Câu Hỏi Migration Lens Cho Sustainability

```
SUSTAINABILITY (BỀN VỮNG — MÔI TRƯỜNG) — CÂU HỎI:

SUS 1: Tối thiểu hóa carbon footprint (dấu chân carbon)?
──────────────────────────────────────────────────────────
□ Xem xét AWS Graviton instances (ARM-based):
  → Tiết kiệm điện ~60% so với x86 equivalents
  → Ít nhiệt hơn → ít làm mát hơn
□ Chọn AWS regions có clean energy commitment:
  ví dụ: us-west-2 (Oregon), eu-west-1 (Ireland) — dùng nhiều năng lượng tái tạo
□ AWS Customer Carbon Footprint Tool (Công Cụ Theo Dõi Dấu Chân Carbon) được check

SUS 2: Tận dụng shared infrastructure (hạ tầng dùng chung)?
─────────────────────────────────────────────────────────────
□ Dùng managed services thay vì self-managed trên EC2:
  RDS thay vì MySQL on EC2, ElastiCache thay vì Redis on EC2
  → AWS quản lý và tối ưu utilization tốt hơn
□ Serverless (Lambda, Fargate) cho workloads không cần server chạy liên tục
  → Chỉ tốn tài nguyên khi thực sự xử lý request

SUS 3: Tối ưu hóa utilization?
───────────────────────────────
□ Right-sizing → không waste tài nguyên (vừa tiết kiệm tiền, vừa tiết kiệm điện)
□ Auto Scaling → chỉ chạy đủ số instances theo demand thực tế
□ Schedule downtime cho non-production environments
```

---

## 🔄 Ba Giai Đoạn Migration Trong Lens

### Migration Lens Structured Around 3 Phases

```
MIGRATION LENS CÓ CÂU HỎI RIÊNG CHO TỪNG PHASE:

PHASE 1: ASSESS
Câu hỏi tập trung vào:
├── Hiểu workloads: "Bạn có hiểu rõ tất cả workloads cần migrate không?"
├── TCO analysis: "Bạn có business case rõ ràng không?"
├── Risk identification: "Bạn đã xác định rủi ro chính chưa?"
└── Team readiness: "Team có kỹ năng cần thiết không?"

PHASE 2: MOBILIZE
Câu hỏi tập trung vào:
├── Landing zone: "AWS account structure và governance có sẵn sàng không?"
├── Connectivity: "Mạng on-premises đã kết nối an toàn với AWS chưa?"
├── Security baseline: "Security controls cơ bản đã được áp dụng chưa?"
└── Pilot readiness: "Bạn sẵn sàng cho pilot migrations chưa?"

PHASE 3: MIGRATE & MODERNIZE
Câu hỏi tập trung vào:
├── Migration execution: "Cutover được thực hiện đúng quy trình không?"
├── Validation: "Dữ liệu và ứng dụng được validate sau migration không?"
├── Optimization: "Workloads đã được tối ưu sau khi lên AWS chưa?"
└── Decommission: "On-premises được ngừng đúng timeline không?"
```

---

## 🛠️ Quy Trình Thực Hiện Well-Architected Review

### Step-by-Step Review Process

```
QUY TRÌNH THỰC HIỆN REVIEW:

BƯỚC 1: CHUẨN BỊ (1-2 ngày)
├── Xác định workload cần review (ví dụ: "E-Commerce Platform Migration")
├── Tập hợp đủ stakeholders: cloud architect, infra lead, security, app team lead
├── Chuẩn bị tài liệu: architecture diagrams, migration plan draft
└── Tạo workload trong AWS Well-Architected Tool (free — miễn phí)

BƯỚC 2: REVIEW SESSION (1-2 ngày)
├── Đi qua từng câu hỏi trong Migration Lens theo từng Pillar
├── Với mỗi câu hỏi: Có / Không / Không áp dụng
├── Ghi chú các gaps và concerns
└── Không phải tất cả câu hỏi đều áp dụng — dùng judgment

BƯỚC 3: IDENTIFY HIGH RISKS (vài giờ)
├── Tool tự động highlight High Risk Items
├── Phân loại: Critical (phải fix trước migration) vs Important (fix sớm)
└── Ưu tiên theo impact và effort

BƯỚC 4: REMEDIATION PLAN (1-2 ngày)
├── Với mỗi High Risk Item: lập action plan
│   ├── Mô tả gap
│   ├── Giải pháp cụ thể
│   ├── Owner (người chịu trách nhiệm)
│   └── Timeline (khi nào xong)
└── Review plan với leadership

BƯỚC 5: TRACK & RE-REVIEW
├── Track progress trên AWS Well-Architected Tool
├── Re-review sau mỗi migration wave hoặc sau 90 ngày
└── Milestone tracking để thấy improvement over time
```

---

## 🚨 High Risk Items — Xử Lý Rủi Ro Cao

### Top High Risk Items Phổ Biến Nhất

```
TOP 10 HIGH RISK ITEMS THƯỜNG GẶP TRONG MIGRATION REVIEWS:

HRI-1: Không có test cutover
───────────────────────────
Risk: Lần đầu cutover là production → rủi ro cao nhất
Mitigation: Thực hiện ít nhất 1 test cutover trước production cutover

HRI-2: Không có rollback plan
──────────────────────────────
Risk: Nếu cutover thất bại, không biết làm gì → downtime kéo dài
Mitigation: Documented rollback procedure với deadline rõ ràng

HRI-3: Dependency không được map đầy đủ
────────────────────────────────────────
Risk: Di chuyển App trước DB → App bị lỗi ngay sau migration
Mitigation: ADS + manual validation với app team

HRI-4: Backup không được tạo trước migration
──────────────────────────────────────────────
Risk: Nếu có lỗi, không thể recover dữ liệu trước migration
Mitigation: Full backup + verify restore TRƯỚC khi bắt đầu

HRI-5: IAM permissions quá rộng cho migration tools
───────────────────────────────────────────────────
Risk: Nếu credentials bị compromise → attacker có quyền rộng
Mitigation: Least privilege IAM roles, revoke sau khi xong

HRI-6: Không có monitoring sau migration
────────────────────────────────────────
Risk: Không biết khi nào hệ thống có vấn đề sau migration
Mitigation: CloudWatch dashboards và alarms trước cutover

HRI-7: On-premises không được decommission đúng hạn
───────────────────────────────────────────────────
Risk: Trả tiền cả AWS + on-premises → chi phí bùng nổ
Mitigation: Timeline decommission rõ ràng, được leadership approve

HRI-8: Data encryption không nhất quán
────────────────────────────────────────
Risk: Dữ liệu nhạy cảm không được mã hóa trong quá trình migrate
Mitigation: Enforce encryption in-transit và at-rest cho tất cả data

HRI-9: Performance không được test trước cutover
──────────────────────────────────────────────────
Risk: Application chậm hơn trên AWS do wrong instance type
Mitigation: Load testing trước cutover, so sánh với baseline

HRI-10: Team chưa có AWS skills đủ
────────────────────────────────────
Risk: Sự cố xảy ra không ai biết troubleshoot trên AWS
Mitigation: AWS training + thực hành trong lab + có AWS Support plan tốt
```

---

## 📋 Migration Readiness Assessment (MRA)

### MRA Là Gì?

```
MIGRATION READINESS ASSESSMENT (MRA — Đánh Giá Mức Độ Sẵn Sàng Di Chuyển):

MRA là một đánh giá toàn diện hơn, thường do AWS hoặc AWS Partner thực hiện.
Nó dựa trên Well-Architected Migration Lens nhưng có thêm:

├── Discovery workshop: 2-3 ngày với key stakeholders
├── People & Process readiness (sẵn sàng về con người và quy trình):
│   ├── Org structure phù hợp chưa?
│   ├── Cloud Center of Excellence (CCoE — Trung Tâm Xuất Sắc Đám Mây) có chưa?
│   └── Cloud skills gap assessment
├── Technology readiness: Kỹ thuật sẵn sàng chưa?
│   ├── Landing zone đã build chưa?
│   ├── Connectivity (Direct Connect/VPN) sẵn chưa?
│   └── Security baseline áp dụng chưa?
└── Output: Migration Readiness score (điểm sẵn sàng) cho từng domain

MRA THƯỜNG ĐƯỢC THỰC HIỆN KHI:
├── Enterprise migration lớn (100+ servers)
├── Government hoặc regulated industry (ngành có quy định chặt)
└── Khi cần AWS validation trước khi commit (cam kết) vào migration
```

---

## 🖥️ Công Cụ AWS Well-Architected Tool

### Cách Sử Dụng AWS Well-Architected Tool

```
AWS WELL-ARCHITECTED TOOL — HƯỚNG DẪN NHANH:

Truy cập:
└── AWS Console → Well-Architected Tool → Define workload

Tạo Workload:
├── Workload name: ví dụ "E-Commerce Migration 2026"
├── Description: mô tả ngắn
├── Industry type: chọn ngành
├── Environment: Pre-production hoặc Production
└── AWS Regions: region đang dùng

Chạy Review:
├── Chọn Lens: "Migration Lens"
├── Đi qua từng câu hỏi (thường 50-60 câu hỏi)
├── Mỗi câu: chọn "Best practices applied" (đã áp dụng) hoặc không
└── Ghi notes cho từng item

Xem Kết Quả:
├── Lens Dashboard: số High/Medium risks theo từng Pillar
├── High Risk Items list: danh sách cụ thể
└── Improvement Plan: với links đến tài liệu AWS

Chia Sẻ Và Track:
├── Share workload với team members
├── Create milestones (tạo mốc) sau mỗi sprint
└── Compare milestones để thấy improvement

CHI PHÍ: MIỄN PHÍ
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: AWS Well-Architected Migration Lens khác gì so với Well-Architected Framework thông thường?**

> **Well-Architected Framework** là bộ hướng dẫn chung cho bất kỳ workload nào chạy trên AWS — kiểm tra xem hệ thống có đang vận hành tốt, bảo mật, đáng tin cậy, hiệu quả và tối ưu chi phí không.
>
> **Migration Lens** là phần mở rộng tập trung vào **rủi ro đặc thù của quá trình di chuyển**:
> - Cutover planning và rollback procedures
> - Dependency management giữa applications
> - Data integrity trong quá trình migration
> - Security khi data đang di chuyển
> - Cost control để tránh trả cả on-premises lẫn AWS cùng lúc
>
> Bạn chạy Migration Lens ở cuối Assess phase để xác định gaps và risk items trước khi bắt đầu di chuyển thực sự.

---

**Q: Trong 6 pillars, pillar nào bạn cho là quan trọng nhất trong migration?**

> Câu trả lời phụ thuộc vào ngữ cảnh, nhưng trong migration tôi ưu tiên theo thứ tự:
>
> 1. **Reliability**: Vì cutover planning, rollback, và backup là make-or-break — nếu không có rollback plan và cutover thất bại, toàn bộ dự án có thể bị cancel.
>
> 2. **Security**: Data đang di chuyển là điểm dễ bị tấn công nhất. Dữ liệu nhạy cảm bị exposed trong migration là incident nghiêm trọng.
>
> 3. **Operational Excellence**: Không có runbooks và monitoring thì không thể troubleshoot khi có sự cố.
>
> 4. **Cost Optimization**: Zombie resources sau migration có thể làm chi phí AWS cao bất ngờ, ảnh hưởng đến business case đã cam kết với C-level.

---

**Q: High Risk Items trong Well-Architected Review là gì? Cần làm gì với chúng?**

> **High Risk Items** là các vấn đề mà nếu không xử lý, có khả năng cao gây ra sự cố nghiêm trọng trong migration: downtime không kiểm soát được, mất dữ liệu, security breach, hoặc chi phí bùng nổ.
>
> Quy trình xử lý:
> 1. **Phân loại**: Critical (phải fix TRƯỚC migration) vs Important (fix trong 30-90 ngày sau)
> 2. **Assign owner**: Mỗi item cần có người chịu trách nhiệm rõ ràng
> 3. **Đặt deadline**: Cam kết thời gian fix cụ thể
> 4. **Track progress**: Dùng AWS Well-Architected Tool để theo dõi
> 5. **Re-review**: Sau khi fix, re-review để xác nhận item đã được giải quyết

---

### Câu Hỏi Nâng Cao

**Q: Bạn cần thực hiện migration trong thời gian rất ngắn và không thể làm full Well-Architected Review. Bạn sẽ ưu tiên kiểm tra gì?**

> Nếu thời gian hạn chế, tôi sẽ ưu tiên kiểm tra 5 điều tối quan trọng:
>
> 1. **Backup verification**: Backup on-premises được tạo và restore test thành công — không thể bỏ qua
> 2. **Rollback plan**: Documented procedure rõ ràng — nếu cutover thất bại, làm gì trong vòng X phút
> 3. **Dependency check**: Ít nhất validate với app team rằng thứ tự wave plan đúng
> 4. **Basic security**: Security Groups không quá permissive, encryption enabled
> 5. **Monitoring**: CloudWatch alarms cho CPU, disk, error rates sẵn sàng trước cutover
>
> Các items khác có thể làm retrospectively nhưng 5 điểm trên là non-negotiable.

---

**Q: Làm thế nào để convince (thuyết phục) C-level chạy Migration Lens Review khi họ muốn "di chuyển nhanh nhất có thể"?**

> Tiếp cận bằng business language thay vì technical language:
>
> - **Risk quantification**: "Downtime 1 giờ của hệ thống thanh toán = mất $X doanh thu. Test cutover mất 4 giờ nhưng phòng ngừa rủi ro downtime 8-24 giờ."
>
> - **Industry precedent**: Dẫn case studies về migration failures do không test — ví dụ: airline reservation system downtime sau migration không test rollback.
>
> - **Insurance analogy**: "Well-Architected Review giống mua bảo hiểm — chi phí nhỏ (2-3 ngày) để bảo vệ investment lớn ($280K migration investment)"
>
> - **Phased approach**: Đề xuất review rút gọn — chỉ focus vào Reliability và Security pillars, 1 ngày thay vì full review 3 ngày. Better than nothing.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 07-discovery-assessment
