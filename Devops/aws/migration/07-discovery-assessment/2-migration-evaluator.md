# AWS Migration Evaluator — Phân Tích TCO & Xây Dựng Business Case

> **AWS Migration Evaluator** (trước đây có tên là **TSO Logic**) là dịch vụ giúp doanh nghiệp xây dựng **Migration Business Case** (hồ sơ kinh doanh cho việc di chuyển) bằng cách tính toán và so sánh **TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu)** giữa môi trường on-premises hiện tại và kiến trúc AWS đề xuất. Kết quả là báo cáo chi tiết cho phép C-level (cấp lãnh đạo: CTO, CFO, CEO) đưa ra quyết định có nên di chuyển lên AWS hay không, và tiết kiệm được bao nhiêu.

## 📚 Mục Lục (Table of Contents)

1. [Migration Evaluator Là Gì? Giải Quyết Gì?](#migration-evaluator-là-gì-giải-quyết-gì)
2. [Ba Cách Thu Thập Dữ Liệu](#ba-cách-thu-thập-dữ-liệu)
3. [Phân Tích TCO On-Premises](#phân-tích-tco-on-premises)
4. [Tính Toán Chi Phí AWS](#tính-toán-chi-phí-aws)
5. [Right-Sizing Recommendations](#right-sizing-recommendations)
6. [Migration Business Case Report](#migration-business-case-report)
7. [Savings Plans & Reserved Instances Trong Evaluator](#savings-plans--reserved-instances-trong-evaluator)
8. [Quy Trình Sử Dụng Migration Evaluator](#quy-trình-sử-dụng-migration-evaluator)
9. [Ví Dụ Kết Quả Thực Tế](#ví-dụ-kết-quả-thực-tế)
10. [Hạn Chế Và Điều Cần Lưu Ý](#hạn-chế-và-điều-cần-lưu-ý)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Migration Evaluator Là Gì? Giải Quyết Gì?

### Bối Cảnh Và Vấn Đề

```
TÌNH HUỐNG KINH DOANH:

Giám đốc CTO nói: "Tôi muốn di chuyển hạ tầng lên AWS."
CFO hỏi: "Chúng ta sẽ tiết kiệm hay tốn thêm tiền?"
CEO hỏi: "ROI — Return on Investment (Lợi Tức Đầu Tư) là bao nhiêu? Bao lâu có lãi?"

Để trả lời những câu hỏi này, bạn cần:
├── Chi phí on-premises hiện tại: THỰC TẾ (không phải ước tính)
│   ├── Hardware: servers, switches, storage (kho lưu trữ)
│   ├── Software: OS licenses, application licenses (bản quyền)
│   ├── Facilities: điện, làm mát, diện tích datacenter
│   └── People: IT staff (nhân lực IT), training
│
└── Chi phí AWS dự kiến: TÍNH ĐÚNG
    ├── Dựa trên utilization thực tế (không phải capacity tối đa)
    ├── Bao gồm tất cả dịch vụ: EC2, RDS, EBS, Network, Support
    ├── Với các discount options: Savings Plans, Reserved Instances
    └── Theo thời gian 3-5 năm (vì hardware renewal cycle)

Migration Evaluator giải quyết ĐÚNG bài toán này.
```

### Migration Evaluator vs. Tính Tay

```
┌────────────────────────┬───────────────────────────────────────────────────┐
│                        │                                                   │
│ Tính tay bằng Excel    │ AWS Migration Evaluator                           │
│                        │                                                   │
├────────────────────────┼───────────────────────────────────────────────────┤
│ Dữ liệu utilization:   │ Dữ liệu utilization:                             │
│ Thường dùng capacity   │ Từ ADS — utilization THỰC TẾ                     │
│ tối đa → oversizing    │ → right-sizing chính xác hơn 30-50%             │
├────────────────────────┼───────────────────────────────────────────────────┤
│ Giá AWS:               │ Giá AWS:                                         │
│ Phải tra tay, dễ sai   │ Tự động cập nhật từ AWS pricing API              │
│ hoặc outdated          │ → Luôn chính xác và mới nhất                    │
├────────────────────────┼───────────────────────────────────────────────────┤
│ Chi phí on-premises:   │ Chi phí on-premises:                             │
│ Dễ bỏ sót chi phí ẩn  │ Template có đủ categories → ít bỏ sót hơn       │
├────────────────────────┼───────────────────────────────────────────────────┤
│ Báo cáo:               │ Báo cáo:                                         │
│ Tự format, không       │ Professional PDF có thể trình C-level            │
│ chuyên nghiệp          │ với charts và visualizations                     │
├────────────────────────┼───────────────────────────────────────────────────┤
│ Thời gian:             │ Thời gian:                                       │
│ Vài tuần               │ Vài ngày đến 1 tuần                              │
└────────────────────────┴───────────────────────────────────────────────────┘
```

---

## 📥 Ba Cách Thu Thập Dữ Liệu

### Cách 1: Import Từ AWS Application Discovery Service (ADS)

```
CÁCH TỐT NHẤT — DỮ LIỆU CHÍNH XÁC NHẤT:

  AWS ADS (đã chạy 2-4 tuần)
  ├── Server inventory (danh sách server)
  ├── CPU/RAM utilization thực tế theo thời gian
  ├── Disk usage
  └── Network throughput
        │
        │ (tự động import)
        ▼
  AWS Migration Evaluator
  ├── Nhận diện server specs từ ADS
  ├── Dùng utilization THỰC TẾ để right-size
  └── Tính TCO AWS chính xác hơn

Ưu điểm:
✅ Dữ liệu utilization thực tế → right-sizing tốt nhất
✅ Ít công việc thủ công
✅ Dữ liệu nhất quán

Nhược điểm:
❌ Cần đã triển khai ADS từ trước (2-4 tuần)
```

### Cách 2: Agentless Collector Riêng Của Migration Evaluator

```
AGENTLESS COLLECTOR CỦA MIGRATION EVALUATOR:

Migration Evaluator có collector riêng (khác với ADS Collector)
├── Thu thập trực tiếp từ VMware vSphere
├── Windows Server (qua WMI — Windows Management Instrumentation)
└── SQL Server (qua SQL Server DMVs — Dynamic Management Views)

Cài đặt:
├── Download collector từ Migration Evaluator Console
├── Cài lên máy Windows Server (cần quyền admin)
├── Cấu hình credentials cho vCenter / WMI
└── Chạy 1-2 tuần

Ưu điểm:
✅ Không cần ADS riêng
✅ Thu thập SQL Server data chi tiết (license, edition, DTUs)
✅ Tốt cho environments với nhiều SQL Server

Nhược điểm:
❌ Chỉ hỗ trợ VMware + Windows, ít hơn ADS
❌ Dữ liệu dependency map kém hơn ADS Agent
```

### Cách 3: Upload File Excel/CSV Thủ Công

```
CÁCH THỦ CÔNG — KHI KHÔNG THỂ CÀI COLLECTOR:

Migration Evaluator cung cấp Excel template với các cột:

Thông tin bắt buộc:
├── Server name
├── Number of vCPUs (số nhân CPU ảo)
├── RAM (GB)
├── Storage (GB)
├── OS type (Windows / Linux)
└── Server role (Web, App, Database, etc.)

Thông tin tùy chọn (nhưng nên có để chính xác hơn):
├── Average CPU utilization (%)
├── Peak CPU utilization (%)
├── Average RAM utilization (%)
├── Disk I/O (IOPS, MB/s)
└── Network throughput (MB/s)

Thông tin chi phí on-premises (để tính TCO):
├── Hardware cost per server per year
├── Software licensing cost
├── Maintenance contract cost
├── Power consumption (kW per server)
└── Space (rack units)

Ưu điểm:
✅ Linh hoạt — hoạt động với bất kỳ môi trường nào
✅ Có thể thêm thông tin chi phí chi tiết

Nhược điểm:
❌ Tốn thời gian thu thập thủ công
❌ Dễ sai nếu không có monitoring data
❌ Thường dùng capacity tối đa → oversized → chi phí AWS bị ước tính cao hơn thực tế
```

---

## 💰 Phân Tích TCO On-Premises

### Các Danh Mục Chi Phí On-Premises

```
TCO ON-PREMISES — ĐẦY ĐỦ CÁC THÀNH PHẦN:

1. HARDWARE COSTS (Chi Phí Phần Cứng):
   ├── Server purchase cost (amortized — phân bổ theo vòng đời 3-5 năm)
   │   Ví dụ: Server $15,000, vòng đời 5 năm → $3,000/năm
   ├── Network equipment: switches, routers, load balancers
   ├── Storage arrays: SAN (Storage Area Network), NAS
   └── End-of-life replacement costs (chi phí thay thế khi hết vòng đời)

2. SOFTWARE LICENSING (Chi Phí Bản Quyền Phần Mềm):
   ├── Operating System: Windows Server licenses (đắt!), RHEL subscriptions
   ├── Database: Oracle Database, SQL Server Enterprise → rất đắt
   ├── Virtualization: VMware vSphere, ESXi licenses
   ├── Monitoring tools, backup software
   └── Application server licenses

3. FACILITIES COSTS (Chi Phí Cơ Sở Vật Chất):
   ├── Power (điện): chi phí điện × PUE (Power Usage Effectiveness)
   │   PUE trung bình datacenter on-premises: 1.5-2.0
   │   → Với 100 kW IT load: thực tế dùng 150-200 kW
   ├── Cooling (làm mát): thường bằng 30-50% chi phí điện
   ├── Physical space (diện tích): tiền thuê hoặc khấu hao tòa nhà
   └── Physical security: bảo vệ, kiểm soát ra vào

4. PEOPLE COSTS (Chi Phí Nhân LỰC):
   ├── Infrastructure engineers: quản lý servers, storage, network
   ├── DBAs (Database Administrators — Quản Trị Viên Cơ Sở Dữ Liệu)
   ├── Security team (chuyên gia bảo mật)
   ├── Helpdesk (bộ phận hỗ trợ)
   └── Training và certifications

5. OPERATIONAL COSTS (Chi Phí Vận Hành):
   ├── Maintenance contracts (hợp đồng bảo trì) với hardware vendors
   ├── Software support & maintenance fees
   ├── Backup media and offsite storage
   └── Disaster Recovery site costs (chi phí site DR)

TỔNG HỢP ĐIỂN HÌNH:
Chi phí on-premises thực sự thường CAO HƠN 40-60%
so với những gì team IT báo cáo lên ban lãnh đạo,
vì nhiều chi phí ẩn (hidden costs) bị bỏ qua.
```

---

## ☁️ Tính Toán Chi Phí AWS

### Các Thành Phần Chi Phí AWS

```
CHI PHÍ AWS TRONG MIGRATION EVALUATOR:

1. COMPUTE (EC2 Instances — Máy Chủ Ảo):
   ├── Instance type recommendations (đề xuất loại instance)
   │   Dựa trên: CPU cores cần + RAM cần + workload type
   ├── Pricing models được so sánh:
   │   ├── On-Demand (theo giờ, không cam kết)
   │   ├── 1-year Reserved Instances (tiết kiệm ~40% so với On-Demand)
   │   ├── 3-year Reserved Instances (tiết kiệm ~60%)
   │   └── Compute Savings Plans (linh hoạt hơn RI)
   └── Graviton (ARM-based) options nếu workload tương thích
       → Thêm 10-20% tiết kiệm so với x86

2. DATABASE (Amazon RDS / Aurora):
   ├── Đề xuất RDS engine tương đương
   │   ├── Oracle → RDS Oracle hoặc Aurora PostgreSQL (nếu dùng SCT)
   │   ├── SQL Server → RDS SQL Server hoặc Aurora MySQL/PostgreSQL
   │   └── MySQL/PostgreSQL → Amazon Aurora (thường rẻ hơn và mạnh hơn)
   ├── Multi-AZ pricing (Multi-AZ — Nhiều Vùng Khả Dụng, cho HA)
   └── Storage + IOPS costs

3. STORAGE:
   ├── EBS (Elastic Block Store — Lưu Trữ Khối Co Giãn) volumes
   │   Loại: gp3 (general purpose — đa mục đích), io2 (high IOPS)
   ├── S3 (Simple Storage Service) cho backups và archives
   └── EFS (Elastic File System) nếu cần shared file storage

4. NETWORKING:
   ├── Data transfer in: MIỄN PHÍ
   ├── Data transfer out to internet: tính phí (thường $0.09/GB)
   ├── Cross-AZ data transfer: $0.01/GB
   └── Direct Connect (kết nối chuyên dụng) nếu cần hybrid connectivity

5. SUPPORT PLAN:
   └── Business hoặc Enterprise Support Plan (thường 3-10% of spend)

6. MISC (Các Chi Phí Khác):
   ├── CloudWatch monitoring
   ├── AWS Backup
   └── KMS (Key Management Service — Dịch Vụ Quản Lý Khóa Mã Hóa)
```

---

## 📐 Right-Sizing Recommendations

### Cách Migration Evaluator Đề Xuất Instance Size

```
QUY TRÌNH RIGHT-SIZING TỰ ĐỘNG:

Input từ ADS hoặc Collector:
├── Số vCPU: 32
├── RAM: 128 GB
├── Average CPU utilization: 22%
├── Peak CPU utilization (P95): 65%
└── Average RAM utilization: 58%

Migration Evaluator tính toán:
├── CPU cần thực tế: 32 × 65% ÷ target_utilization(80%) ≈ 26 vCPU
│   → Làm tròn lên instance tiếp theo: 32 vCPU
├── RAM cần thực tế: 128 × 58% ÷ buffer(1.2) ≈ 62 GB
│   → Làm tròn lên: 64 GB
└── Right-sized instance: m5.8xlarge (32 vCPU, 128 GB)
    OR nếu RAM không cần nhiều: r5.2xlarge (8 vCPU, 64 GB)

Kết quả:
├── Server gốc: 32 vCPU, 128 GB RAM (đủ cho r5.4xlarge)
├── Right-sized: r5.2xlarge (8 vCPU, 64 GB)
└── Tiết kiệm: ~60% chi phí EC2 so với mapping 1:1
```

### Bảng Right-Sizing Ví Dụ

```
┌────────────────────┬──────────────────────┬─────────────────────┬───────────┐
│ Server On-Premises │ 1:1 Mapping (Sai)    │ Right-Sized (Đúng)  │ Tiết Kiệm │
├────────────────────┼──────────────────────┼─────────────────────┼───────────┤
│ 32 vCPU, 128 GB   │ r5.4xlarge           │ r5.2xlarge          │ ~50%      │
│ CPU avg 22%        │ $1.008/hr            │ $0.504/hr           │           │
├────────────────────┼──────────────────────┼─────────────────────┼───────────┤
│ 16 vCPU, 64 GB    │ m5.4xlarge           │ m5.2xlarge          │ ~50%      │
│ CPU avg 15%        │ $0.768/hr            │ $0.384/hr           │           │
├────────────────────┼──────────────────────┼─────────────────────┼───────────┤
│ 4 vCPU, 16 GB     │ m5.xlarge            │ m5.large            │ ~50%      │
│ CPU avg 8%         │ $0.192/hr            │ $0.096/hr           │           │
├────────────────────┼──────────────────────┼─────────────────────┼───────────┤
│ 8 vCPU, 16 GB     │ c5.2xlarge           │ c5.xlarge           │ ~50%      │
│ CPU avg 70%        │ $0.340/hr            │ $0.170/hr           │           │
│ (high CPU)         │ (đây là workload     │ (CPU-bound → c5     │           │
│                    │ phù hợp)             │ family đúng)        │           │
└────────────────────┴──────────────────────┴─────────────────────┴───────────┘

Trung bình: Right-sizing giúp tiết kiệm 30-50% chi phí EC2
so với ánh xạ 1:1 theo capacity tối đa on-premises.
```

---

## 📄 Migration Business Case Report

### Cấu Trúc Báo Cáo

```
MIGRATION BUSINESS CASE — CẤU TRÚC:

1. EXECUTIVE SUMMARY (Tóm Tắt Điều Hành):
   ├── Tổng tiết kiệm dự kiến 3 năm (con số lớn, nổi bật)
   ├── Thời gian hoàn vốn (payback period — tháng/năm để break-even)
   └── Khuyến nghị: Proceed (tiến hành) / Review Further (xem xét thêm)

2. CURRENT STATE ANALYSIS (Phân Tích Trạng Thái Hiện Tại):
   ├── Inventory: số servers, OS distribution, workload types
   ├── Chi phí on-premises chi tiết theo danh mục (hardware/software/facilities/people)
   └── Tổng TCO on-premises 3 năm và 5 năm

3. AWS PROPOSED ARCHITECTURE (Kiến Trúc AWS Đề Xuất):
   ├── EC2 instance recommendations với right-sizing
   ├── Database migration path (ví dụ: SQL Server → RDS SQL Server)
   └── Storage recommendations

4. AWS COST ANALYSIS (Phân Tích Chi Phí AWS):
   ├── Chi phí theo pricing model:
   │   ├── On-Demand: $X/năm (baseline)
   │   ├── 1-year Reserved: $Y/năm (savings $Z, X%)
   │   └── 3-year Reserved: $W/năm (savings $V, X%)
   └── Phân tích theo 1, 3, 5 năm với charts

5. COMPARISON & SAVINGS (So Sánh & Tiết Kiệm):
   ├── Waterfall chart: from on-premises cost to AWS cost
   ├── Annual savings breakdown (phân tích tiết kiệm theo năm)
   └── Cumulative savings over time (tiết kiệm tích lũy theo thời gian)

6. MIGRATION INVESTMENT (Đầu Tư Di Chuyển):
   ├── One-time migration costs (chi phí di chuyển một lần):
   │   ├── Professional services / consulting
   │   ├── Training
   │   └── Migration tools (MGN, DMS licenses)
   └── Run costs during parallel operation period (chạy song song)

7. ROI ANALYSIS (Phân Tích Lợi Tức Đầu Tư):
   ├── Break-even point (điểm hòa vốn)
   ├── 5-year NPV (Net Present Value — Giá Trị Hiện Tại Ròng)
   └── IRR (Internal Rate of Return — Tỷ Lệ Hoàn Vốn Nội Bộ)
```

---

## 💵 Savings Plans & Reserved Instances Trong Evaluator

### So Sánh Các Pricing Options

```
MIGRATION EVALUATOR PHÂN TÍCH CÁC PRICING OPTIONS:

┌───────────────────────┬─────────────────────────────────────────────────────┐
│ Pricing Option        │ Đặc Điểm Và Tiết Kiệm                              │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ On-Demand             │ • Không cam kết, linh hoạt                          │
│ (Theo Nhu Cầu)        │ • Baseline giá cao nhất                             │
│                       │ • Phù hợp: migration period, testing                │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ 1-year Reserved       │ • Cam kết 1 năm                                     │
│ Instances             │ • Tiết kiệm ~40% so với On-Demand                   │
│ (Phiên Bản Dự Trữ     │ • All Upfront (trả toàn bộ trước): tiết kiệm nhất  │
│ 1 Năm)                │ • No Upfront (không trả trước): linh hoạt hơn       │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ 3-year Reserved       │ • Cam kết 3 năm                                     │
│ Instances             │ • Tiết kiệm ~60% so với On-Demand                   │
│ (Phiên Bản Dự Trữ     │ • Phù hợp: workloads ổn định, dài hạn              │
│ 3 Năm)                │ • Risk: nếu workload thay đổi khó điều chỉnh        │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Compute Savings       │ • Linh hoạt: áp dụng cho bất kỳ instance family,   │
│ Plans                 │   region, OS                                        │
│ (Kế Hoạch Tiết Kiệm  │ • Tiết kiệm ~54-66% so với On-Demand               │
│ Tính Toán)            │ • Cam kết theo $/hour (không theo instance type)    │
│                       │ • RECOMMENDED (được khuyến nghị) cho hầu hết cases  │
└───────────────────────┴─────────────────────────────────────────────────────┘

KHUYẾN NGHỊ THỰC TẾ:
├── Năm đầu sau migration: On-Demand (chưa biết actual usage)
├── Sau 3-6 tháng ổn định: chuyển sang Compute Savings Plans 1-year
└── Sau 1-2 năm: có thể chuyển một phần sang 3-year RI cho workloads ổn định
```

---

## 🛠️ Quy Trình Sử Dụng Migration Evaluator

### Step-by-Step Guide

```
BƯỚC 1: TRUY CẬP MIGRATION EVALUATOR
├── AWS Console → Migration & Transfer → Migration Evaluator
└── Chọn "Create assessment" (Tạo đánh giá)

BƯỚC 2: CHỌN DATA SOURCE (Nguồn Dữ Liệu)
├── Option A: "Use existing ADS data" nếu đã có ADS
├── Option B: "Deploy Agentless Collector" nếu chưa có ADS
└── Option C: "Import manually" → tải Excel template

BƯỚC 3: ĐIỀN THÔNG TIN CHI PHÍ ON-PREMISES
├── Tab "On-premises costs":
│   ├── Server hardware cost per unit per year
│   ├── Networking equipment cost per year
│   ├── Power cost ($/kWh và PUE value)
│   ├── Facilities cost per rack unit per year
│   └── IT staff cost (số FTE và lương trung bình)
├── Tab "Software licensing":
│   ├── OS licenses (Windows Server cores)
│   ├── Database licenses (Oracle, SQL Server)
│   └── Other application licenses
└── Tab "Maintenance & support":
    ├── Hardware maintenance % of purchase price
    └── Software support annual fees

BƯỚC 4: CONFIGURE AWS ASSUMPTIONS (Cấu Hình Giả Thiết AWS)
├── Target region(s) (vùng AWS mục tiêu)
├── Discount program: On-Demand / Savings Plans / Reserved
├── Migration strategy per server group:
│   ├── Rehost (EC2) → tính EC2 cost
│   ├── Replatform to containers (ECS/EKS) → tính Fargate
│   └── Replatform database → tính RDS/Aurora
└── Graviton consideration (có muốn xem xét ARM-based Graviton không?)

BƯỚC 5: GENERATE REPORT
├── Nhấn "Generate business case report"
├── Chờ 10-30 phút (tùy số servers)
└── Download PDF report hoặc xem trực tiếp trong Console

BƯỚC 6: REVIEW VÀ TINH CHỈNH
├── Review right-sizing recommendations với infrastructure team
├── Điều chỉnh nếu có servers đặc biệt (GPU, high-memory workloads)
├── Thêm migration investment costs thủ công
└── Tạo final report để trình ban lãnh đạo
```

---

## 📊 Ví Dụ Kết Quả Thực Tế

### Dự Án: Công Ty SaaS 200 Nhân Viên

```
BỐI CẢNH:
├── Quy mô: 180 servers (150 VMs VMware + 30 bare-metal Windows)
├── Khối lượng DB: 15 SQL Server, 8 Oracle, 20 MySQL/PostgreSQL
├── Thời gian phân tích: 3 năm
└── Data source: ADS Agentless + manual input cho on-prem costs

KẾT QUẢ MIGRATION EVALUATOR:

CHI PHÍ ON-PREMISES (3 NĂM):
├── Hardware (servers, storage, networking):    $1,850,000
├── Software licensing (Windows, SQL Server,    $  720,000
│   VMware):
├── Facilities (power, cooling, co-location):   $  340,000
├── IT Infrastructure Staff (1.5 FTE):          $  450,000
├── Maintenance contracts:                       $  280,000
└── TỔNG ON-PREMISES:                           $3,640,000

CHI PHÍ AWS ĐỀ XUẤT (3 NĂM, Compute Savings Plans 1-year):
├── EC2 (after right-sizing: 180 → 145 instances): $ 680,000
├── RDS (SQL Server + Aurora thay Oracle):         $ 320,000
├── EBS + S3 Storage:                              $ 145,000
├── Data Transfer (outbound):                      $  55,000
├── Support Plan (Business):                       $  72,000
└── TỔNG AWS:                                      $1,272,000

SO SÁNH:
├── Tiết kiệm 3 năm: $3,640,000 - $1,272,000 = $2,368,000 (65%)
├── Migration investment (one-time): $280,000
├── Net savings 3 năm: $2,088,000
├── Break-even point: ~5 tháng
└── 5-year NPV (discount rate 10%): $3,200,000

RIGHT-SIZING BREAKDOWN:
├── Servers with significant oversizing: 82/180 (45%)
│   → Trung bình: instances giảm từ 16 vCPU → 8 vCPU
├── Servers recommended for Retire: 12
│   (chạy <5% utilization, không có traffic quan trọng)
└── Servers recommended for Graviton: 67
    → Thêm $85,000 tiết kiệm/năm (đã tính trong số trên)
```

---

## ⚠️ Hạn Chế Và Điều Cần Lưu Ý

### Điều Evaluator Không Tính Được Tự Động

```
NHỮNG GÌ MIGRATION EVALUATOR KHÔNG TỰ ĐỘNG LÀM:

1. Chi phí migration thực tế:
   ├── Professional services / consulting fees
   ├── Downtime costs (chi phí mất doanh thu khi downtime)
   ├── Staff retraining costs (chi phí đào tạo lại nhân lực)
   └── → Phải nhập thủ công

2. Application-specific licensing:
   ├── Oracle license trên AWS tính theo vCPU → có thể đắt hơn on-premises
   │   (Byol — Bring Your Own License có giới hạn vCPU)
   ├── Một số ISV applications có BYOL restrictions trên cloud
   └── → Cần kiểm tra riêng với vendor

3. Regulatory và compliance costs:
   ├── Chi phí audit, compliance certification
   └── Không tự động tính

4. Technical debt (nợ kỹ thuật):
   ├── Ứng dụng cũ cần refactor trước khi migrate
   └── Không tính được developer time

5. Organizational change costs:
   ├── Thay đổi quy trình vận hành
   └── Cultural change management
```

### Pitfalls Khi Dùng Evaluator

```
⚠️ CẠM BẪY PHỔ BIẾN:

1. Garbage In, Garbage Out (Dữ Liệu Vào Sai → Kết Quả Sai):
   Problem: Nhập chi phí on-premises không đầy đủ → TCO on-prem bị thấp
   → Tiết kiệm trông ít hơn thực tế
   Solution: Dùng checklist đầy đủ, kiểm tra với Finance team

2. 1:1 Mapping Không Right-Size:
   Problem: Không dùng ADS data → mapping theo capacity tối đa
   → AWS cost bị tính cao hơn thực tế
   Solution: LUÔN dùng utilization data từ ADS hoặc monitoring tools

3. Bỏ Qua Oracle / SQL Server Licensing:
   Problem: Oracle trên AWS tính theo socket/vCPU → có thể tốn hơn on-prem
   Solution: Xem xét migrate Oracle → Aurora PostgreSQL (với SCT)
   → Loại bỏ Oracle license hoàn toàn → tiết kiệm lớn nhất

4. Chỉ So Sánh Compute, Bỏ Qua Network:
   Problem: Data transfer egress cost bị underestimate
   Solution: Tính kỹ outbound data, cân nhắc Direct Connect cho heavy traffic

5. Trình Báo Cáo Mà Không Validate Với Technical Team:
   Problem: Finance & Executive chấp nhận con số, nhưng infra team không đồng ý
   Solution: Review right-sizing recommendations với infrastructure team TRƯỚC
   khi trình lên C-level
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: AWS Migration Evaluator dùng để làm gì?**

> Migration Evaluator giúp xây dựng **Migration Business Case** — tài liệu thuyết phục C-level để phê duyệt dự án migration bằng cách:
> 1. Tính **TCO on-premises** đầy đủ (bao gồm cả chi phí ẩn)
> 2. Tính **chi phí AWS dự kiến** với right-sizing dựa trên utilization thực tế
> 3. So sánh và tính **savings** theo 1-3-5 năm
> 4. Đề xuất **pricing model** phù hợp (On-Demand, Savings Plans, Reserved Instances)
> 5. Xuất **báo cáo chuyên nghiệp** có thể trình lên ban lãnh đạo

---

**Q: Right-sizing trong Migration Evaluator là gì và tại sao quan trọng?**

> **Right-sizing** là việc chọn EC2 instance size phù hợp với **utilization thực tế** của server on-premises, thay vì ánh xạ 1:1 theo capacity tối đa.
>
> Ví dụ: Server có 32 vCPU nhưng chỉ dùng trung bình 22% → cần khoảng 7 vCPU → chọn instance 8 vCPU thay vì 32 vCPU.
>
> Tại sao quan trọng: Trung bình các server on-premises chỉ dùng 15-25% capacity. Không right-size → trả tiền cho 4x tài nguyên không cần thiết → lãng phí 50-70% chi phí EC2.

---

**Q: Sự khác biệt giữa Reserved Instances và Compute Savings Plans là gì?**

> - **Reserved Instances** (Phiên Bản Dự Trữ): Cam kết với một **instance type cụ thể** (ví dụ: m5.xlarge tại us-east-1). Tiết kiệm cao nhất (~60% với 3-year All Upfront), nhưng kém linh hoạt — khó thay đổi nếu workload thay đổi.
>
> - **Compute Savings Plans** (Kế Hoạch Tiết Kiệm Tính Toán): Cam kết theo **mức chi tiêu $/giờ** thay vì instance cụ thể. Áp dụng tự động cho bất kỳ EC2 instance family, region, OS nào. Linh hoạt hơn, tiết kiệm ~54-66%.
>
> Khuyến nghị thực tế: Compute Savings Plans tốt hơn cho hầu hết workloads vì linh hoạt hơn — đặc biệt quan trọng trong 1-2 năm đầu sau migration khi workload có thể thay đổi.

---

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để xử lý Oracle Database licensing khi dùng Migration Evaluator?**

> Oracle licensing là điểm đau (pain point) lớn trong cloud migration:
> - Oracle trên AWS tính license theo **vCPU** (cứ 2 vCPU = 1 processor license)
> - Nếu server on-premises dùng Oracle Standard Edition 2 (SE2), có giới hạn 2 sockets → có thể tốn **nhiều hơn** khi lên EC2 với nhiều vCPU
>
> Migration Evaluator không tự động handle vấn đề này. Giải pháp tốt nhất:
> 1. **Migrate Oracle → Aurora PostgreSQL** hoặc PostgreSQL bằng SCT/DMS → loại bỏ Oracle license hoàn toàn → tiết kiệm lớn nhất
> 2. Nếu phải dùng Oracle: chọn instance type có **ít vCPU** như m5.large thay vì m5.4xlarge (Hyperthreading không tính trong Oracle licensing)
> 3. Tính license cost riêng và add vào báo cáo Evaluator thủ công

---

**Q: Migration Evaluator báo tiết kiệm 60%, nhưng CFO vẫn không tin. Bạn sẽ làm gì?**

> Các bước để tăng độ tin cậy của báo cáo:
> 1. **Validate dữ liệu đầu vào**: Mời Finance team kiểm tra chi phí on-premises trong báo cáo — thường họ có thêm số liệu chính xác
> 2. **Sensitivity analysis** (phân tích độ nhạy): Chạy scenario với giả định kém lạc quan hơn (ví dụ: tiết kiệm chỉ 40%, 30%) — nếu vẫn có lãi, càng thuyết phục
> 3. **Pilot migration**: Đề xuất di chuyển 10-15 servers đại diện trước → theo dõi chi phí thực tế 3 tháng → có số liệu thực để validate
> 4. **AWS Partner hoặc AWS team**: Yêu cầu AWS Solutions Architect review và sign-off báo cáo — có trọng lượng hơn với CFO
> 5. **Peer reference**: Tham khảo case studies của công ty tương tự đã migrate thành công

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 07-discovery-assessment
