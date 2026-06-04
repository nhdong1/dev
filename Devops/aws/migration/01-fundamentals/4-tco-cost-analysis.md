# TCO và Phân Tích Chi Phí Migration — Total Cost of Ownership

> **TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu)** là nền tảng để xây dựng business case cho migration. Bài này hướng dẫn cách tính TCO thực tế, so sánh on-premises vs AWS, và hiểu mô hình định giá AWS.

## 📚 Mục Lục

1. [TCO Là Gì?](#tco-là-gì)
2. [Chi Phí On-Premises Đầy Đủ](#chi-phí-on-premises-đầy-đủ)
3. [Mô Hình Chi Phí AWS](#mô-hình-chi-phí-aws)
4. [So Sánh TCO — Ví Dụ Thực Tế](#so-sánh-tco--ví-dụ-thực-tế)
5. [Công Cụ Tính TCO của AWS](#công-cụ-tính-tco-của-aws)
6. [Các Chiến Lược Tối Ưu Chi Phí AWS](#các-chiến-lược-tối-ưu-chi-phí-aws)
7. [Chi Phí Ẩn Khi Migration](#chi-phí-ẩn-khi-migration)
8. [ROI và Payback Period](#roi-và-payback-period)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 💡 TCO Là Gì?

> **TCO (Total Cost of Ownership — Tổng Chi Phí Sở Hữu)** = Tổng tất cả chi phí để sở hữu và vận hành một hệ thống trong một khoảng thời gian (thường 3-5 năm), bao gồm cả chi phí trực tiếp và gián tiếp.

### Tại Sao TCO Quan Trọng Trong Migration?

```
Sai lầm phổ biến:
├── So sánh: "Server on-premises giá $10,000 vs EC2 giá $300/tháng"
│   → Kết luận: On-premises rẻ hơn
└── Nhưng bỏ quên: điện, làm mát, nhân sự, downtime, license...

TCO đầy đủ:
├── On-premises: $10,000 server + $5,000 điện/năm + $8,000 nhân sự/năm + ...
│   → 3 năm: $10,000 + (3 × $15,000) = $55,000
└── AWS EC2: $300/tháng × 36 = $10,800 + networking
    → 3 năm: ~$15,000 (Reserved Instance)

Thực tế: AWS thường tiết kiệm 20-40% TCO 3 năm
(Số liệu từ AWS Migration Evaluator)
```

---

## 🏢 Chi Phí On-Premises Đầy Đủ

### Nhóm Chi Phí 1 — Hardware (Phần Cứng)

```
├── Server hardware: $3,000 - $50,000 / server
│   (Blade servers, rack servers, storage arrays)
├── Networking hardware: Switches, routers, load balancers
├── Storage: SAN, NAS (có thể 5-10x giá server)
├── UPS (Uninterruptible Power Supply — Nguồn Dự Phòng)
├── Khấu hao phần cứng: Thường 3-5 năm
└── Thay thế khi hỏng: Emergency replacement chi phí cao
```

### Nhóm Chi Phí 2 — Data Center Infrastructure (Hạ Tầng Data Center)

```
Power (Điện):
├── Tiêu thụ điện server: Thường 200-400W/server
├── Overhead của data center: PUE — Power Usage Effectiveness
│   PUE = Tổng điện data center / Điện IT equipment
│   PUE tốt: 1.2-1.5 | PUE kém: 2.0+
│   → Nếu server dùng 100 kWh, data center thực tế dùng 120-200 kWh
├── Chi phí điện ở VN: ~2,000 VND/kWh
└── Ví dụ: 100 servers × 300W × 24h × 365 ngày × 1.5 PUE
         = 394,200 kWh/năm × 2,000 VND ≈ 789 triệu VND/năm

Cooling (Làm Mát):
├── Thường chiếm 30-40% tổng điện data center
└── Chạy quanh năm dù server idle

Colocation (Thuê Không Gian Data Center):
├── Giá: $500 - $2,000 / rack / tháng (VN thấp hơn)
├── Cross-connect (Kết nối chéo): $200 - $500/tháng per connection
└── Bandwidth: $1 - $10/Mbps/tháng

Nếu tự có data center:
├── Chi phí xây dựng: Hàng triệu USD
├── Bảo trì tòa nhà
└── Security (bảo vệ vật lý)
```

### Nhóm Chi Phí 3 — Software & License (Phần Mềm và Giấy Phép)

```
OS License:
├── Windows Server: $1,000 - $6,000 / server (tùy version)
└── RHEL: $800 - $2,500 / server / năm

Database License:
├── Oracle Database Enterprise: $47,500 / processor core
│   (1 server × 8 cores = $380,000)
├── SQL Server Enterprise: $14,256 / core
└── MySQL / PostgreSQL: Miễn phí (open source)

Virtualization:
├── VMware vSphere: $1,000 - $4,000 / CPU socket
├── VMware vCenter: $4,000+
└── Hyper-V: Included với Windows Server

Backup Software:
└── Veeam, Commvault: $500 - $2,000 / server / năm

Monitoring:
└── Nagios, Zabbix, Datadog: $0 - $500 / server / năm
```

### Nhóm Chi Phí 4 — Personnel (Nhân Sự)

> Đây là chi phí **thường bị đánh giá thấp nhất** khi so sánh on-premises vs cloud.

```
System Administrators (Quản Trị Hệ Thống):
├── Lương: $60,000 - $150,000 / năm (thị trường VN thấp hơn)
├── Tỷ lệ server/admin: Thông thường 50-100 servers/admin
│   → 500 servers = 5-10 sysadmins
├── On-call 24/7: Thêm 20-30% premium hoặc thuê thêm người
└── Đào tạo: $2,000 - $5,000 / người / năm

DBA (Database Administrators):
├── Lương: $80,000 - $160,000 / năm
└── Oracle DBA: Đặc biệt đắt và khó tuyển dụng

Network Engineers:
└── Quản lý switches, routers, firewalls

Overhead tổ chức:
├── HR, management overhead: Thêm 20-30% chi phí nhân sự
└── Office space cho IT team
```

### Nhóm Chi Phí 5 — Chi Phí Rủi Ro và Ẩn

```
Downtime Cost (Chi Phí Gián Đoạn):
├── E-commerce: $1,000 - $100,000 / phút downtime
├── Banking: $5,600 / phút (Gartner estimate)
└── On-premises MTTR (Mean Time To Recover): Thường 4-8 giờ

Disaster Recovery (Khắc Phục Thảm Họa):
├── DR site on-premises: Gấp đôi chi phí hardware + data center
├── Backup media: Tape, disk ($500 - $5,000 / server / năm)
└── DR testing: 2-4 lần/năm, mỗi lần vài ngày downtime

Security:
├── Firewall hardware
├── Security software (antivirus, DLP, SIEM)
└── Security personnel và audits

Opportunity Cost (Chi Phí Cơ Hội):
└── Kỹ sư tài năng làm việc maintain on-premises
    thay vì build new features
```

---

## ☁️ Mô Hình Chi Phí AWS

### Pay-As-You-Go (Trả Theo Nhu Cầu Thực Tế)

> AWS tính phí theo giờ (hoặc giây), chỉ trả cho những gì dùng.

```
On-Demand Pricing:
├── Không cam kết, không upfront payment
├── Linh hoạt nhất — scale up/down bất cứ lúc nào
├── Chi phí cao nhất trong các model
└── Phù hợp: Dev/test, workload không dự đoán được
```

### Reserved Instances (RI) — Cam Kết Để Tiết Kiệm

> **Reserved Instances (RI) — Phiên Bản Đặt Trước** = Cam kết dùng EC2/RDS trong 1 hoặc 3 năm để đổi lấy giá thấp hơn.

```
Các loại RI:

Standard RI:
├── Tiết kiệm: 40-60% so với On-Demand
├── Nhược điểm: Không thể đổi instance type sau khi mua
└── Phù hợp: Workload stable, biết chắc instance type cần dùng

Convertible RI:
├── Tiết kiệm: 30-54% so với On-Demand
├── Có thể đổi sang instance type khác
└── Phù hợp: Khi không chắc chắn về instance type tương lai

Payment Options (Hình Thức Thanh Toán):
├── All Upfront (Trả Toàn Bộ Trước): Tiết kiệm nhiều nhất
├── Partial Upfront (Trả Một Phần Trước): Cân bằng giữa tiết kiệm và linh hoạt
└── No Upfront (Không Trả Trước): Tiết kiệm ít nhất nhưng không cần vốn

Ví dụ EC2 m5.xlarge:
├── On-Demand: $0.192/giờ → $1,682/năm
├── 1-year RI (No Upfront): $0.116/giờ → $1,016/năm (tiết kiệm 40%)
└── 3-year RI (All Upfront): $0.076/giờ → $666/năm (tiết kiệm 60%)
```

### Savings Plans — Cam Kết Linh Hoạt Hơn RI

```
Compute Savings Plans:
├── Cam kết chi tiêu $X/giờ trong 1 hoặc 3 năm
├── Áp dụng tự động cho EC2, Fargate, Lambda
├── Có thể đổi Region, OS, instance family
└── Tiết kiệm: Tới 66% so với On-Demand

EC2 Instance Savings Plans:
├── Cam kết với specific instance family và Region
└── Tiết kiệm: Tới 72%

Khi nào dùng Savings Plans vs RI?
├── Savings Plans: Khi muốn linh hoạt hơn, hoặc dùng Fargate/Lambda
└── RI: Khi biết chắc instance type và Region trong dài hạn
```

### Spot Instances — Tận Dụng Tài Nguyên Dư

```
Spot Instances:
├── Dùng EC2 capacity dư thừa của AWS
├── Tiết kiệm: Tới 90% so với On-Demand
├── Nhược điểm: Có thể bị terminate với báo trước 2 phút
└── Phù hợp: Batch jobs, data processing, CI/CD, stateless workload
           KHÔNG phù hợp: Production database, stateful apps
```

### Free Tier — Miễn Phí Học Tập

```
AWS Free Tier (12 tháng đầu):
├── EC2: 750 giờ/tháng t2.micro hoặc t3.micro
├── RDS: 750 giờ/tháng db.t2.micro (MySQL, PostgreSQL, MariaDB, Oracle SE1, SQL Server)
├── S3: 5 GB storage, 20,000 GET, 2,000 PUT requests
├── Lambda: 1 triệu requests/tháng
└── CloudFront: 50 GB data transfer/tháng

Always Free (Luôn Miễn Phí):
├── Lambda: 1 triệu requests/tháng
├── DynamoDB: 25 GB storage
└── CloudWatch: 10 metrics, 10 alarms
```

---

## 📊 So Sánh TCO — Ví Dụ Thực Tế

### Ví Dụ: Doanh Nghiệp 50 Servers

```
SCENARIO: Công ty 200 nhân viên, 50 servers on-premises
Server specs: Dual CPU 8-core, 64 GB RAM, 2 TB SSD

============================================================
CHI PHÍ ON-PREMISES (3 năm)
============================================================

Hardware:
├── 50 servers × $8,000/server = $400,000
├── Storage arrays = $150,000
├── Networking hardware = $50,000
└── Tổng hardware: $600,000

Data Center (Colocation):
├── 10 racks × $1,000/rack/tháng × 36 tháng = $360,000
├── Điện + cooling (ước tính): $180,000
└── Bandwidth: $36,000

License:
├── Windows Server: 50 × $3,000 = $150,000
├── SQL Server Standard: 5 instances × $10,000 = $50,000
└── VMware: $80,000

Nhân Sự:
├── 3 Sysadmins × $80,000/năm × 3 năm = $720,000
└── 1 DBA × $100,000/năm × 3 năm = $300,000

Backup & DR:
└── $60,000

TỔNG ON-PREMISES 3 NĂM: $2,536,000
Chi phí/năm: ~$845,000

============================================================
CHI PHÍ AWS (3 năm) — Rehost + Replatform
============================================================

EC2 (50 servers → m5.2xlarge Reserved Instance 3-year):
├── 50 × $0.252/giờ × 8,760 giờ × 3 năm (All Upfront)
└── Thực tế với 3yr RI: 50 × $3,800/năm × 3 = $570,000

RDS (Thay thế SQL Server bằng Aurora PostgreSQL):
├── 5 instances db.r5.xlarge × $2,000/năm × 3 = $30,000
└── Không cần SQL Server license!

Storage (EBS):
└── 50 × 2TB × $0.10/GB/tháng × 36 = $360,000

Networking (Data Transfer):
└── $60,000

Support (Business Support):
└── 3% monthly bill ≈ $50,000

AWS Migration (one-time):
├── MGN: Miễn phí 90 ngày
├── Professional Services: $100,000
└── Training: $30,000

TỔNG AWS 3 NĂM: ~$1,200,000
Chi phí/năm: ~$400,000

============================================================
KẾT QUẢ
============================================================
Tiết kiệm 3 năm: $2,536,000 - $1,200,000 = $1,336,000 (53%)
ROI: Hoàn vốn sau ~18 tháng
```

### Điều Chỉnh Theo Thực Tế

> Số liệu trên là ước tính. Thực tế phụ thuộc nhiều vào:
> - Vị trí địa lý (chi phí nhân sự, điện)
> - Tối ưu sau migration (right-sizing)
> - Chọn Reserved Instances đúng
> - Loại bỏ workload không cần thiết (Retire)

---

## 🛠️ Công Cụ Tính TCO của AWS

### AWS Migration Evaluator

```
Cách sử dụng:
1. Thu thập data từ on-premises (ADS hoặc RVTools export)
2. Upload vào Migration Evaluator
3. Nhận báo cáo PDF với:
   ├── Projected 3-year savings
   ├── Recommended EC2/RDS instance types
   ├── Breakdown chi phí chi tiết
   └── Executive summary để present với leadership

Ưu điểm:
├── Dùng data thực tế từ môi trường của bạn
├── Tự động right-sizing dựa trên utilization
└── Credibility cao với management
```

### AWS Pricing Calculator

```
URL: https://calculator.aws/

Cách dùng:
1. Chọn dịch vụ (EC2, RDS, S3...)
2. Nhập cấu hình
3. Nhận ước tính chi phí tháng/năm

Hạn chế:
└── Chỉ tính phía AWS, không so sánh với on-premises
    → Dùng kết hợp với Migration Evaluator

Phù hợp:
├── Ước tính chi phí solution mới
└── So sánh cấu hình EC2 khác nhau
```

### AWS Cost Explorer

```
Dùng sau khi đã lên AWS:
├── Phân tích chi phí thực tế theo service/account/tag
├── Phát hiện anomalies (chi phí bất thường)
├── Xem usage trends
└── Nhận recommendations: Reserved Instances, Savings Plans
```

---

## 💰 Các Chiến Lược Tối Ưu Chi Phí AWS

### Chiến Lược 1 — Right-Sizing (Chọn Đúng Kích Thước)

```
Vấn đề phổ biến sau Rehost:
└── On-premises thường over-provisioned (cấp phát dư để "an toàn")
    Ví dụ: Server 32 CPU nhưng trung bình chỉ dùng 10%

Cách right-size:
1. Chạy workload trên AWS 2-4 tuần
2. Xem CloudWatch metrics (CPU, Memory, Network)
3. Nếu average CPU < 10% và peak < 30% → Downsize
4. AWS Compute Optimizer: Tự động recommend instance type phù hợp

Tiết kiệm tiềm năng: 20-40% sau right-sizing
```

### Chiến Lược 2 — Reserved Instances / Savings Plans

```
Workflow đề xuất:
1. Tháng 1-3: Dùng On-Demand (linh hoạt, kiểm tra workload)
2. Sau 3 tháng: Phân tích usage patterns
3. Purchase RI/Savings Plans cho workload stable
4. Tiết kiệm ngay 30-60%

Nguyên tắc:
├── Chỉ RI cho workload chạy > 70% thời gian
├── Dùng Savings Plans nếu có nhiều instance types khác nhau
└── Đừng mua RI cho dev/test (nên dùng On-Demand hoặc Spot)
```

### Chiến Lược 3 — Spot Instances cho Workload Phù Hợp

```
Workload phù hợp với Spot:
├── Batch processing (xử lý hàng loạt)
├── CI/CD pipelines (tự động build/test)
├── Data analytics (Spark, EMR)
├── Rendering, media transcoding
└── Machine Learning training

Cách triển khai an toàn:
├── Dùng Spot Fleet hoặc Auto Scaling Group với mix Spot + On-Demand
├── Thiết kế ứng dụng để tolerate interruption (checkpoint, graceful shutdown)
└── Tiết kiệm: 60-90%
```

### Chiến Lược 4 — Storage Optimization

```
S3 Storage Classes:
├── S3 Standard: Data truy cập thường xuyên
├── S3 Standard-IA (Infrequent Access): Data truy cập ít → 40% rẻ hơn
├── S3 Glacier Instant Retrieval: Archive nhưng cần lấy nhanh
├── S3 Glacier Flexible Retrieval: Archive, lấy trong 3-5 giờ
└── S3 Glacier Deep Archive: Archive dài hạn, lấy trong 12 giờ → Rẻ nhất

S3 Intelligent-Tiering:
└── Tự động chuyển data giữa tiers dựa trên access patterns
    → Không cần quản lý thủ công

EBS:
├── gp2 → gp3: Tiết kiệm 20% với performance tốt hơn
├── Delete orphaned snapshots (snapshots không còn ai dùng)
└── Move cold data sang S3 hoặc EFS Infrequent Access
```

### Chiến Lược 5 — Database Optimization

```
Tiết kiệm lớn nhất thường đến từ:

1. Loại bỏ Oracle / SQL Server License:
   Oracle Enterprise → Aurora PostgreSQL
   SQL Server → Aurora MySQL hoặc Aurora PostgreSQL
   → Tiết kiệm: $47,500/core (Oracle) hoặc $14,256/core (SQL Server)

2. RDS Reserved Instances:
   └── 40-60% tiết kiệm so với On-Demand RDS

3. Aurora Serverless v2:
   └── Dev/test databases chỉ chạy khi cần → Gần bằng 0 khi idle
```

---

## ⚠️ Chi Phí Ẩn Khi Migration

### Chi Phí Migration (One-Time)

```
Chi phí thực hiện migration:
├── AWS Professional Services hoặc Partner: $50,000 - $500,000
├── Internal team time: 3-12 tháng kỹ sư
├── Training và certification: $2,000 - $5,000 / người
└── Downtime trong cutover (nếu có): Lost revenue

Chi phí testing và validation:
├── QA testing trên môi trường mới
├── Performance testing
└── User acceptance testing (UAT)
```

### Chi Phí Data Transfer (Networking)

```
AWS Data Transfer Out (Egress):
├── Data từ AWS ra internet: $0.09/GB (giảm dần theo volume)
├── Data giữa Regions: $0.02/GB
└── Data cùng Region, cùng AZ: Miễn phí

Lưu ý quan trọng:
├── Migration ban đầu: Data IN = Miễn phí
├── Sau migration, data ra nhiều → Tính egress
└── Tính toán egress cost khi thiết kế architecture

VPN / Direct Connect:
├── Trong migration: VPN ~ $30-50/tháng (Site-to-Site VPN)
└── Long-term: Direct Connect = $0.02-0.08/GB + port fee
```

### Chi Phí Vận Hành Mới

```
Một số chi phí mới sau migration:
├── AWS Support Plan: Developer ($29/tháng), Business (3% bill), Enterprise
├── Training liên tục (cloud evolves nhanh)
├── Công cụ quản lý cloud: CloudHealth, Spot.io, CloudCheckr
└── Security tools: GuardDuty, Security Hub, Inspector
```

---

## 📈 ROI và Payback Period

### Công Thức Tính ROI

```
ROI (Return on Investment — Tỷ Suất Hoàn Vốn):

ROI = (Net Benefits / Migration Cost) × 100%

Net Benefits = (On-premises TCO 3yr) - (AWS TCO 3yr) - (Migration Cost)

Payback Period (Thời Gian Hoàn Vốn):
= Migration Cost / Monthly Savings
```

### Ví Dụ Tính ROI

```
Từ ví dụ 50 servers ở trên:
├── Migration cost (one-time): $130,000 (Professional Services + Training)
├── Monthly savings: $1,336,000 / 36 tháng = $37,111/tháng
└── Payback Period: $130,000 / $37,111 = ~3.5 tháng

ROI trong 3 năm:
= ($1,336,000 - $130,000) / $130,000 × 100% = 928%
```

### Lợi Ích Phi Tài Chính (Intangible Benefits)

```
Khó định lượng nhưng thực sự có giá trị:
├── Time to market nhanh hơn (không chờ procurement phần cứng)
├── Khả năng experiment với chi phí thấp
├── Tăng cường bảo mật và compliance
├── Business continuity tốt hơn (Multi-AZ, disaster recovery)
├── Developer productivity (không lo ops, focus vào features)
└── Khả năng thu hút talent (engineers thích làm việc với cloud)
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: TCO là gì và tại sao quan trọng trong migration?**

> A: TCO (Total Cost of Ownership — Tổng Chi Phí Sở Hữu) là tổng tất cả chi phí để vận hành hệ thống trong một khoảng thời gian, bao gồm cả chi phí ẩn. Quan trọng vì nhiều tổ chức so sánh sai: chỉ nhìn chi phí EC2 so với giá server, bỏ qua điện, làm mát, nhân sự, license, downtime. Khi tính đầy đủ TCO, AWS thường tiết kiệm 20-50% so với on-premises trong vòng 3 năm.

**Q: Những chi phí on-premises nào thường bị bỏ sót?**

> A: Các chi phí hay bị bỏ sót:
> 1. **Điện và làm mát (Cooling)**: Thường 50-100% chi phí điện server
> 2. **Nhân sự (Personnel)**: Sysadmins, DBAs — thường là chi phí lớn nhất
> 3. **Over-provisioning**: Server mua to để "phòng khi cần" nhưng idle 90%
> 4. **Downtime cost**: On-premises MTTR thường 4-8 giờ, mỗi giờ = lost revenue
> 5. **DR site**: Thường phải nhân đôi infrastructure cho disaster recovery
> 6. **Software license**: Đặc biệt Oracle — $47,500/core

**Q: Sự khác biệt giữa Reserved Instances và Savings Plans?**

> A:
> - **Reserved Instances**: Cam kết với specific EC2 instance type và Region. Standard RI không đổi type được, Convertible RI đổi được. Tiết kiệm tới 72%.
> - **Savings Plans**: Cam kết chi tiêu $X/giờ. Compute Savings Plans áp dụng cho EC2 + Fargate + Lambda, linh hoạt hơn RI vì tự động áp dụng cho bất kỳ instance type nào. Tiết kiệm tới 66%.
>
> Chọn RI khi biết chắc instance type dài hạn. Chọn Savings Plans khi dùng nhiều loại instance hoặc muốn bao gồm Lambda/Fargate.

**Q: Làm thế nào để right-size EC2 sau migration?**

> A: Quy trình right-sizing:
> 1. Chạy workload trên AWS 2-4 tuần để có real data
> 2. Xem CloudWatch: CPU avg/max, Memory, Network I/O
> 3. Nếu CPU avg < 10%, peak < 30% → Instance quá lớn, downsize
> 4. Dùng **AWS Compute Optimizer** để nhận recommendations tự động
> 5. Test performance sau khi downsize
> 6. Kết quả điển hình: Tiết kiệm thêm 20-40% chi phí

**Q: Khi nào nên dùng Spot Instances?**

> A: Spot Instances phù hợp với workload có thể bị interrupt (ngắt đột ngột với báo trước 2 phút):
> - Batch processing, data analytics (EMR, Spark)
> - CI/CD build pipelines
> - Machine learning training
> - Rendering, media processing
>
> Không phù hợp: Production databases, stateful apps, workloads không thể restart.
> Tiết kiệm: 60-90% so với On-Demand. Thường combine Spot + On-Demand/Reserved trong Auto Scaling Group để đảm bảo availability.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
