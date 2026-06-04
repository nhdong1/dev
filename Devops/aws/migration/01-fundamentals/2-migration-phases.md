# Ba Giai Đoạn AWS Migration — Assess → Mobilize → Migrate & Modernize

> Mọi dự án migration AWS đều trải qua **3 giai đoạn chính**. Hiểu rõ từng giai đoạn giúp lập kế hoạch thực tế, tránh các sai lầm phổ biến và đảm bảo migration thành công.

## 📚 Mục Lục

1. [Tổng Quan Ba Giai Đoạn](#tổng-quan-ba-giai-đoạn)
2. [Giai Đoạn 1 — Assess (Đánh Giá)](#giai-đoạn-1--assess-đánh-giá)
3. [Giai Đoạn 2 — Mobilize (Chuẩn Bị)](#giai-đoạn-2--mobilize-chuẩn-bị)
4. [Giai Đoạn 3 — Migrate & Modernize (Di Chuyển & Hiện Đại Hóa)](#giai-đoạn-3--migrate--modernize-di-chuyển--hiện-đại-hóa)
5. [Migration Waves — Lập Kế Hoạch Theo Đợt](#migration-waves--lập-kế-hoạch-theo-đợt)
6. [Checklist Theo Từng Giai Đoạn](#checklist-theo-từng-giai-đoạn)
7. [Sai Lầm Phổ Biến](#sai-lầm-phổ-biến)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Tổng Quan Ba Giai Đoạn

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS MIGRATION JOURNEY                        │
│                                                                 │
│  GIAI ĐOẠN 1        GIAI ĐOẠN 2        GIAI ĐOẠN 3            │
│  ┌──────────┐       ┌──────────┐       ┌──────────────────┐    │
│  │  ASSESS  │──────►│ MOBILIZE │──────►│ MIGRATE &        │    │
│  │ (Đánh Giá│       │(Chuẩn Bị)│       │ MODERNIZE        │    │
│  │          │       │          │       │(Di Chuyển &      │    │
│  │          │       │          │       │ Hiện Đại Hóa)    │    │
│  └──────────┘       └──────────┘       └──────────────────┘    │
│                                                                 │
│  Thời gian:         Thời gian:         Thời gian:               │
│  2-8 tuần           3-6 tháng          6-24 tháng               │
│                                                                 │
│  Đầu ra chính:      Đầu ra chính:      Đầu ra chính:           │
│  Business case      Landing Zone       Workloads trên AWS      │
│  Migration plan     Migration Factory  Cost optimization       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 Giai Đoạn 1 — Assess (Đánh Giá)

### Mục Tiêu

> Hiểu rõ **môi trường hiện tại**, đánh giá **business case**, và lập **migration portfolio** trước khi bắt đầu di chuyển bất kỳ workload nào.

### Các Hoạt Động Chính

#### 1.1 Discovery — Khám Phá Hạ Tầng

**Công cụ:** AWS Application Discovery Service (ADS)

```
Agentless Discovery (Không cài agent):
├── Dùng AWS Agentless Discovery Connector — VM appliance
├── Kết nối VMware vCenter để thu thập metadata
├── Thu thập: hostname, IP, OS, CPU, RAM, disk usage
└── Không xem được chi tiết process-level

Agent-based Discovery (Cài agent):
├── Cài AWS Discovery Agent trên từng server
├── Thu thập chi tiết hơn: network connections, process list
├── Có thể xác định dependencies giữa các ứng dụng
└── Phù hợp: khi cần dependency mapping chính xác
```

**Đầu ra:**
- Danh sách đầy đủ servers và ứng dụng
- Dependency map — sơ đồ phụ thuộc giữa các ứng dụng
- Dữ liệu sử dụng tài nguyên (CPU/RAM/Storage utilization)

#### 1.2 Portfolio Assessment — Đánh Giá Danh Mục

**Công cụ:** AWS Migration Hub, AWS Migration Evaluator

```
Đánh giá từng ứng dụng theo:
├── Business Value (Giá trị kinh doanh): Critical / Important / Low
├── Technical Complexity (Độ phức tạp kỹ thuật): High / Medium / Low
├── Migration Effort (Nỗ lực di chuyển): Weeks / Months
├── Age of Application (Tuổi ứng dụng): Legacy / Modern
└── Compliance Requirements (Yêu cầu tuân thủ): Regulated / Standard
```

**Phân Loại Theo 7Rs:** (xem file `1-7r-strategies.md`)

#### 1.3 TCO Analysis — Phân Tích Tổng Chi Phí

**Công cụ:** AWS Migration Evaluator, AWS Pricing Calculator

```
So sánh chi phí:
├── On-premises hiện tại: phần cứng + điện + nhân sự + data center
└── AWS: EC2 + RDS + networking + support

Mục tiêu: Chứng minh business case với con số cụ thể
```

#### 1.4 Risk Assessment — Đánh Giá Rủi Ro

```
Loại Rủi Ro            │ Ví Dụ                  │ Giảm Thiểu
───────────────────────┼────────────────────────┼──────────────────────
Technical Risk         │ Ứng dụng legacy không  │ Spike test sớm
                       │ chạy được trên cloud   │
Data Risk              │ Mất dữ liệu trong      │ Backup + validation
                       │ quá trình transfer      │
Compliance Risk        │ Dữ liệu phải ở VN      │ Chọn đúng AWS Region
                       │                         │ hoặc Retain
Organizational Risk    │ Team chưa biết cloud   │ Training plan
```

### Đầu Ra Giai Đoạn 1

- [ ] **Discovery Report** — Báo cáo toàn bộ hạ tầng
- [ ] **Migration Portfolio** — Danh sách workload + chiến lược 7Rs
- [ ] **Business Case** — Phân tích TCO và ROI
- [ ] **Risk Register** — Đăng ký rủi ro và kế hoạch giảm thiểu
- [ ] **Migration Wave Plan** — Lịch trình di chuyển theo đợt

---

## 🔧 Giai Đoạn 2 — Mobilize (Chuẩn Bị)

### Mục Tiêu

> Xây dựng **nền tảng kỹ thuật (Landing Zone)**, **quy trình vận hành (Migration Factory)**, và **năng lực đội ngũ** để sẵn sàng di chuyển quy mô lớn.

### Các Hoạt Động Chính

#### 2.1 Landing Zone — Thiết Lập Môi Trường AWS Nền Tảng

> **Landing Zone** là môi trường AWS đã được thiết lập sẵn với đầy đủ governance, security, networking — sẵn sàng để nhận workload migration.

**Công cụ:** AWS Control Tower, AWS Organizations

```
Landing Zone bao gồm:
├── Account Structure (Cấu trúc tài khoản):
│   ├── Management Account (Tài khoản quản lý)
│   ├── Security Account (Log Archive + Audit)
│   ├── Shared Services Account (DNS, AD, monitoring)
│   └── Workload Accounts (dev/staging/prod riêng biệt)
│
├── Networking (Mạng):
│   ├── Transit Gateway — Hub-and-spoke network
│   ├── VPC per environment (dev/staging/prod)
│   ├── Direct Connect hoặc VPN về on-premises
│   └── DNS resolution (Route 53 Resolver)
│
├── Security & Compliance (Bảo mật & Tuân thủ):
│   ├── AWS IAM Identity Center (SSO tập trung)
│   ├── AWS Security Hub (tổng hợp findings)
│   ├── AWS GuardDuty (phát hiện mối đe dọa)
│   ├── AWS CloudTrail (audit log)
│   └── SCP — Service Control Policies (giới hạn quyền)
│
└── Operations (Vận hành):
    ├── Centralized Logging (CloudWatch Logs + S3)
    ├── Monitoring (CloudWatch dashboards)
    └── Cost Management (AWS Budgets, Cost Explorer)
```

#### 2.2 Network Connectivity — Kết Nối Mạng

```
Lựa Chọn Kết Nối On-Premises ↔ AWS:

Option 1: AWS Direct Connect (Đường Truyền Riêng)
├── Băng thông: 1 Gbps, 10 Gbps, 100 Gbps
├── Latency thấp, ổn định
├── Thời gian provisioning: 4-8 tuần
└── Chi phí: Cao hơn VPN, phù hợp enterprise

Option 2: VPN (Virtual Private Network)
├── Trên internet public, mã hóa TLS/IPSec
├── Nhanh setup (vài giờ)
├── Băng thông: giới hạn theo đường internet
└── Phù hợp: Pilot migration, dự phòng

Khuyến nghị: Dùng VPN trong Mobilize phase,
sau đó upgrade lên Direct Connect cho production migration
```

#### 2.3 Migration Factory — Quy Trình Di Chuyển Lặp Lại

> **Migration Factory** là tập hợp quy trình, công cụ, và đội ngũ được tiêu chuẩn hóa để di chuyển workload một cách nhất quán và có thể lặp lại.

```
Migration Factory Structure:
├── Wave Planning Team: Lập lịch và quản lý migration waves
├── Migration Engineering Team: Thực hiện kỹ thuật migration
├── Testing Team: Kiểm tra sau migration
└── Cutover Team: Thực hiện và theo dõi cutover

Quy Trình Chuẩn (Runbook):
1. Pre-migration: Backup, inventory check, notify stakeholders
2. Replication: Cài agent, initial sync, verify replication
3. Test Cutover: Launch test instance, run test cases
4. Production Cutover: Maintenance window, DNS change, cutover
5. Post-migration: Validate, monitor 48h, decommission source
```

#### 2.4 Pilot Migration — Di Chuyển Thí Điểm

> Di chuyển **2-5 workload đơn giản** để kiểm tra quy trình, công cụ, và năng lực đội ngũ trước khi triển khai đại trà.

```
Tiêu chí chọn workload pilot:
├── Không phải production critical
├── Không có compliance đặc biệt
├── Kích thước vừa phải (không quá lớn, không quá nhỏ)
└── Có thể rollback dễ dàng

Mục đích pilot:
├── Kiểm tra Landing Zone và kết nối mạng
├── Xác nhận runbook migration hoạt động đúng
├── Đào tạo thực tế cho Migration Factory team
└── Phát hiện vấn đề trước khi gặp ở workload quan trọng
```

#### 2.5 Training & Enablement — Đào Tạo Đội Ngũ

```
Đào tạo cần thiết:
├── AWS Cloud Practitioner (toàn đội)
├── AWS Solutions Architect Associate (cloud architects)
├── Kỹ năng cụ thể:
│   ├── AWS MGN operations (cho migration engineers)
│   ├── AWS DMS + SCT (cho DBAs)
│   └── AWS Security + Compliance (cho security team)
└── CloudFormation / Terraform (cho infrastructure as code)
```

### Đầu Ra Giai Đoạn 2

- [ ] **Landing Zone** — Môi trường AWS sẵn sàng
- [ ] **Network Connectivity** — Direct Connect / VPN hoạt động
- [ ] **Migration Runbooks** — Quy trình chuẩn cho từng loại workload
- [ ] **Pilot Migration Complete** — 2-5 workload đã migrate thành công
- [ ] **Trained Team** — Đội ngũ đã sẵn sàng
- [ ] **Migration Factory** — Đội ngũ + công cụ + quy trình thiết lập xong

---

## 🚀 Giai Đoạn 3 — Migrate & Modernize (Di Chuyển & Hiện Đại Hóa)

### Mục Tiêu

> **Di chuyển toàn bộ workload** lên AWS theo kế hoạch, đồng thời **bắt đầu hiện đại hóa** để tận dụng tối đa cloud.

### Các Hoạt Động Chính

#### 3.1 Wave Execution — Thực Hiện Theo Đợt

```
Migration Wave (Đợt Di Chuyển) = Nhóm workload di chuyển cùng nhau

Wave 1 (Pilot - đã làm ở Mobilize):
├── 2-5 workload không critical
└── Mục tiêu: Kiểm tra quy trình

Wave 2-N (Production Waves):
├── Nhóm theo: application dependencies, business unit, risk level
├── Mỗi wave: 5-20 workload/tuần (tuỳ năng lực đội)
├── Sequence trong mỗi wave:
│   ├── Replication (1-5 ngày)
│   ├── Test Cutover (1-2 ngày)
│   ├── Sign-off từ application owner
│   └── Production Cutover (maintenance window)
└── Theo dõi tập trung: AWS Migration Hub
```

#### 3.2 Cutover Process — Quy Trình Cắt Chuyển

```
Trước Cutover (T-7 ngày):
├── Thông báo stakeholders
├── Xác nhận backup đầy đủ
├── Kiểm tra replication lag < 30 giây
└── Chuẩn bị rollback plan

Maintenance Window (Cửa Sổ Bảo Trì — thường 2-4 giờ đêm):
├── T-0: Stop writes trên source (maintenance mode)
├── T+5: Chờ replication delta sync hoàn thành
├── T+10: Verify data trên AWS
├── T+15: Chạy smoke tests trên AWS instance
├── T+30: Cutover DNS / load balancer → AWS
└── T+60: Monitor, confirm application healthy

Sau Cutover (T+48 giờ):
├── Monitor CloudWatch metrics
├── Kiểm tra error rates, latency
├── Xác nhận với business stakeholders
└── Decommission source server (sau T+30 ngày)
```

#### 3.3 AWS Migration Hub — Theo Dõi Tập Trung

```
AWS Migration Hub cung cấp:
├── Dashboard tập trung: % hoàn thành toàn bộ migration
├── Theo dõi từng server/application
├── Tích hợp với: MGN, DMS, và partner tools
└── Migration status: Not Started → In Progress → Migrated
```

#### 3.4 Post-Migration Optimization — Tối Ưu Sau Di Chuyển

```
Tuần 1-4 sau migration:
├── Right-sizing: Phân tích CloudWatch → Chọn đúng EC2 size
│   (Nhiều workload khi Rehost bị over-provisioned)
├── Reserved Instances / Savings Plans: Cam kết để tiết kiệm 30-60%
├── Storage optimization: gp2 → gp3, Intelligent-Tiering cho S3
└── Network optimization: NAT Gateway → VPC Endpoints cho AWS services

Tháng 1-6 sau migration:
├── Modernization planning: Đánh giá workload nào sẽ Replatform/Refactor
├── Database migration: Oracle/SQL Server → Aurora PostgreSQL (giảm license)
├── Containerization: Ứng dụng phù hợp → ECS/EKS
└── Serverless candidates: Functions phù hợp → Lambda
```

#### 3.5 Modernization — Hiện Đại Hóa (Song Song Hoặc Sau Migration)

```
Ưu tiên modernization theo ROI:
1. Database license elimination (Oracle → Aurora): ROI rất cao
2. Containerization (ECS/EKS): Giảm ops overhead
3. Managed services (ElastiCache, MSK): Giảm management
4. Serverless (Lambda): Giảm chi phí compute idle time
5. Microservices (từ monolith): Tăng tốc độ delivery
```

### Đầu Ra Giai Đoạn 3

- [ ] **100% Workload Migrated** — Tất cả workload đã lên AWS
- [ ] **Data Center Decommissioned** — Đóng cửa on-premises
- [ ] **Cost Savings Realized** — Tiết kiệm chi phí đã đo lường
- [ ] **Modernization Roadmap** — Kế hoạch hiện đại hóa tiếp theo
- [ ] **Operations Model** — Vận hành AWS thuần thục

---

## 🌊 Migration Waves — Lập Kế Hoạch Theo Đợt

### Nguyên Tắc Nhóm Workload Vào Waves

```
Nhóm theo Dependencies (Ưu tiên cao nhất):
├── Các ứng dụng phụ thuộc nhau phải cùng một wave
├── Database phải migrate TRƯỚC ứng dụng sử dụng nó
│   (hoặc cùng wave với cơ chế replication)
└── Dùng dependency map từ ADS để xác định

Nhóm theo Business Function:
├── Ứng dụng cùng business domain → cùng wave
├── Dễ coordinate downtime với cùng business owner
└── Testing tích hợp thuận tiện hơn

Nhóm theo Risk Level:
├── Wave đầu: workload thấp rủi ro, ít critical
├── Wave giữa: workload trung bình
└── Wave cuối: workload critical nhất (đã có kinh nghiệm)
```

### Ví Dụ Migration Wave Plan

```
WAVE 1 — Pilot (Tuần 1-2):
├── Internal blog (WordPress)
├── Dev/Test environment
└── Mục tiêu: Kiểm tra quy trình

WAVE 2 — Non-Critical (Tuần 3-6):
├── HR portal
├── Internal ticketing system
├── Document management
└── 10 workload, 2-3 workload/tuần

WAVE 3 — Business Applications (Tuần 7-14):
├── CRM system
├── ERP modules non-critical
├── Reporting/Analytics platform
└── 20 workload, 3-4 workload/tuần

WAVE 4 — Core Systems (Tuần 15-24):
├── Production databases (cần DMS cho zero-downtime)
├── Core ERP
├── Main e-commerce platform
└── 10 workload, cần nhiều validation hơn

WAVE 5 — Final Cleanup (Tuần 25-26):
├── Decommission on-premises
├── Cancel data center contract
└── Post-migration review
```

---

## ✅ Checklist Theo Từng Giai Đoạn

### Giai Đoạn 1 — Assess Checklist

```
Discovery:
- [ ] Cài ADS connector hoặc agent trên tất cả servers
- [ ] Thu thập data tối thiểu 2-4 tuần để có baseline
- [ ] Export dependency map
- [ ] Xác nhận danh sách đầy đủ (không bỏ sót ứng dụng nào)

Portfolio Assessment:
- [ ] Phân loại 100% workload theo 7Rs
- [ ] Xác định Critical / Important / Low priority cho mỗi ứng dụng
- [ ] Tính TCO on-premises hiện tại
- [ ] Build business case với ROI rõ ràng

Planning:
- [ ] Migration Wave Plan được approve bởi management
- [ ] Risk register đã hoàn thiện
- [ ] Executive sponsorship đã có
```

### Giai Đoạn 2 — Mobilize Checklist

```
Landing Zone:
- [ ] AWS Organizations và Control Tower setup
- [ ] Account structure đúng (Management/Security/Shared/Workload)
- [ ] SCPs (Service Control Policies) đã cấu hình
- [ ] Centralized logging đang hoạt động

Networking:
- [ ] VPN hoặc Direct Connect hoạt động
- [ ] VPCs và subnets đã tạo đúng
- [ ] Security Groups cơ bản đã setup
- [ ] DNS resolution on-premises ↔ AWS hoạt động

Migration Factory:
- [ ] Runbook chuẩn cho Rehost (MGN) đã viết và test
- [ ] Runbook chuẩn cho Replatform đã viết và test
- [ ] Team đã được phân công roles rõ ràng
- [ ] Communication plan cho cutover đã có

Pilot:
- [ ] 2-5 workload pilot đã migrate thành công
- [ ] Bài học kinh nghiệm (lessons learned) đã ghi lại
- [ ] Runbook đã cập nhật dựa trên pilot
```

### Giai Đoạn 3 — Migrate & Modernize Checklist

```
Per-Wave:
- [ ] Wave plan được approve
- [ ] Stakeholders được thông báo
- [ ] Backup đầy đủ trước khi bắt đầu
- [ ] Test cutover đã thực hiện và pass
- [ ] Rollback plan đã viết rõ
- [ ] Cutover window đã lên lịch
- [ ] Post-migration testing đã pass
- [ ] Application owner sign-off

Post-Migration:
- [ ] Right-sizing analysis thực hiện sau 2 tuần
- [ ] Reserved Instances purchased cho workload stable
- [ ] Cost tracking đang theo dõi savings
- [ ] Modernization candidates đã xác định
```

---

## ⚠️ Sai Lầm Phổ Biến

### Giai Đoạn 1

```
❌ Sai: Bỏ qua giai đoạn Assess, nhảy thẳng vào migrate
✅ Đúng: Đầu tư 2-8 tuần khám phá kỹ trước — tiết kiệm nhiều hơn về sau

❌ Sai: Chỉ đếm chi phí server khi tính TCO on-premises
✅ Đúng: Tính đủ: server + điện + cooling + data center + nhân sự + license

❌ Sai: Phân loại 7Rs ngồi bàn một mình
✅ Đúng: Workshop với application owners và business stakeholders
```

### Giai Đoạn 2

```
❌ Sai: Landing Zone quá đơn giản (1 account cho tất cả)
✅ Đúng: Multi-account với proper isolation và governance

❌ Sai: Pilot chọn workload quan trọng nhất
✅ Đúng: Pilot workload đơn giản, có thể rollback dễ

❌ Sai: Thiếu Direct Connect → Replication quá chậm
✅ Đúng: Thiết lập kết nối đủ băng thông trước khi bắt đầu
```

### Giai Đoạn 3

```
❌ Sai: Cutover vào ban ngày business hours
✅ Đúng: Luôn cutover vào maintenance window (đêm/cuối tuần)

❌ Sai: Không có rollback plan
✅ Đúng: Rollback plan chi tiết LUÔN cần có trước mỗi cutover

❌ Sai: Decommission on-premises ngay sau cutover
✅ Đúng: Giữ source servers ít nhất 30 ngày làm fallback

❌ Sai: Quên right-sizing sau migration
✅ Đúng: Phân tích CloudWatch 2-4 tuần sau migration để resize
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Ba giai đoạn migration AWS là gì?**

> A: **Assess** (2-8 tuần) — Khám phá hạ tầng, phân loại workload theo 7Rs, tính TCO, lập migration portfolio và business case.
>
> **Mobilize** (3-6 tháng) — Thiết lập Landing Zone, kết nối mạng, Migration Factory (quy trình + công cụ + đội ngũ), đào tạo team, thực hiện pilot migration.
>
> **Migrate & Modernize** (6-24 tháng) — Thực hiện migration theo waves, theo dõi bằng Migration Hub, tối ưu sau migration (right-sizing, Reserved Instances), hiện đại hóa dần (containerize, serverless, managed services).

**Q: Migration Wave là gì và tại sao cần chia waves?**

> A: Migration Wave là nhóm workload được di chuyển cùng nhau trong một đợt. Lý do chia waves:
> 1. **Dependency management**: Ứng dụng phụ thuộc nhau phải migrate cùng đợt hoặc đúng thứ tự
> 2. **Risk management**: Bắt đầu với workload ít rủi ro, tích lũy kinh nghiệm trước khi xử lý workload critical
> 3. **Team capacity**: Đội migration có giới hạn năng lực — waves giúp phân bổ đều theo thời gian
> 4. **Business continuity**: Không migrate tất cả cùng lúc, giảm thiểu rủi ro toàn hệ thống

**Q: Landing Zone là gì?**

> A: Landing Zone là môi trường AWS đã được cấu hình đầy đủ với governance, security, networking trước khi di chuyển workload. Gồm: multi-account structure (Control Tower), centralized logging (CloudTrail), security controls (GuardDuty, Security Hub), networking (Transit Gateway, VPC), và IAM Identity Center cho SSO. Landing Zone giúp đảm bảo mọi workload migrate vào đều tuân thủ các tiêu chuẩn bảo mật và compliance ngay từ đầu.

**Q: Tại sao cần Pilot Migration?**

> A: Pilot migration (với 2-5 workload không critical) giúp:
> 1. Kiểm tra Landing Zone và kết nối mạng hoạt động đúng
> 2. Validate runbook migration trước khi dùng cho workload quan trọng
> 3. Đào tạo thực tế cho Migration Factory team
> 4. Phát hiện vấn đề bất ngờ (unexpected issues) ở quy mô nhỏ trước
> 5. Tăng sự tự tin của team và stakeholders

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
