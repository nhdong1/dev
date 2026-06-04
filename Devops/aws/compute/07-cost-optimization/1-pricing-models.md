# 💳 AWS EC2 Pricing Models — Mô Hình Định Giá

> Hướng dẫn chi tiết về tất cả các mô hình định giá EC2 Compute — On-Demand (Trả Theo Nhu Cầu), Reserved Instances — RI (Phiên Bản Dự Trữ), Spot Instances (Phiên Bản Tạm Thời), Savings Plans (Kế Hoạch Tiết Kiệm), Dedicated Hosts (Máy Chủ Dành Riêng) — cùng framework lựa chọn đúng mô hình cho từng workload.

---

## 📚 Mục Lục

1. [Tổng Quan 5 Mô Hình Định Giá](#tổng-quan)
2. [On-Demand Instances](#on-demand)
3. [Reserved Instances — RI](#reserved-instances)
4. [Spot Instances](#spot-instances)
5. [Savings Plans](#savings-plans)
6. [Dedicated Hosts & Dedicated Instances](#dedicated)
7. [Framework Lựa Chọn Pricing Model](#framework)
8. [Ví Dụ Tính Toán Thực Tế](#ví-dụ-tính-toán)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Tổng Quan 5 Mô Hình Định Giá {#tổng-quan}

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SO SÁNH PRICING MODELS                           │
├───────────────┬──────────┬────────────┬────────────────────────────┤
│ Model         │ Tiết Kiệm│ Cam Kết    │ Dùng Cho                   │
├───────────────┼──────────┼────────────┼────────────────────────────┤
│ On-Demand     │ Baseline │ Không      │ Unpredictable, short-term   │
│ Reserved 1yr  │ 30-40%   │ 1 năm      │ Steady-state production     │
│ Reserved 3yr  │ 50-72%   │ 3 năm      │ Core infra, databases       │
│ Spot          │ 60-90%   │ Không      │ Batch, fault-tolerant       │
│ Savings Plans │ 17-66%   │ 1-3 năm    │ Flexible compute commitment │
│ Dedicated Host│ Giá cao  │ On-Demand  │ License compliance, BYOL    │
└───────────────┴──────────┴────────────┴────────────────────────────┘
```

---

## 1️⃣ On-Demand Instances {#on-demand}

### Định Nghĩa

On-Demand Instances cho phép trả tiền theo giờ (hoặc theo giây với Linux) mà không cần cam kết dài hạn. Đây là pricing model linh hoạt nhất nhưng đắt nhất.

### Đặc Điểm

- **Billing unit (Đơn Vị Tính Tiền):** Theo giây (Linux) hoặc theo giờ (Windows)
- **Minimum charge (Tính Phí Tối Thiểu):** 60 giây
- **Commitment (Cam Kết):** Không có
- **Interruption risk (Rủi Ro Gián Đoạn):** Không có
- **Availability (Sẵn Sàng):** Gần như luôn sẵn (trừ khi region hết capacity)

### Khi Nào Dùng On-Demand

```
✅ Phù Hợp:
├── Ứng dụng mới chưa biết usage pattern (chu kỳ sử dụng)
├── Workload ngắn hạn (< 1 năm) hoặc spike đột xuất
├── Dev/Test environments không dùng 24/7
├── Disaster Recovery (Phục Hồi Thảm Họa) standby instances
└── Bù đắp khi Spot bị interrupted

❌ Không Phù Hợp:
├── Production workload chạy 24/7 liên tục → dùng Reserved/Savings Plans
├── Batch processing có thể dừng được → dùng Spot
└── Workload ổn định lâu dài → lãng phí 30-72% chi phí
```

### Giá Điển Hình (us-east-1, 2024)

| Instance Type | vCPU | RAM    | On-Demand/giờ |
| ------------- | ---- | ------ | ------------- |
| t3.micro      | 2    | 1 GB   | $0.0104       |
| t3.medium     | 2    | 4 GB   | $0.0416       |
| m5.large      | 2    | 8 GB   | $0.096        |
| m5.xlarge     | 4    | 16 GB  | $0.192        |
| m5.4xlarge    | 16   | 64 GB  | $0.768        |
| c5.2xlarge    | 8    | 16 GB  | $0.34         |
| r5.2xlarge    | 8    | 64 GB  | $0.504        |

---

## 2️⃣ Reserved Instances — RI (Phiên Bản Dự Trữ) {#reserved-instances}

### Định Nghĩa

Reserved Instances là cam kết sử dụng EC2 trong 1 hoặc 3 năm để đổi lấy mức chiết khấu đáng kể so với On-Demand. RI không phải là một instance cụ thể — đây là một **billing discount** (chiết khấu thanh toán) áp dụng tự động cho On-Demand instances khớp với thuộc tính RI.

### Ba Loại RI Theo Flexibility (Linh Hoạt)

#### Standard Reserved Instances (RI Tiêu Chuẩn)
- Tiết kiệm **40-72%** so với On-Demand
- Cam kết: instance family, size, OS, tenancy, region cụ thể
- Có thể **modify** (thay đổi) Availability Zone, scope, size trong cùng family
- Có thể **sell** (bán) trên Reserved Instance Marketplace nếu không dùng nữa

#### Convertible Reserved Instances (RI Có Thể Chuyển Đổi)
- Tiết kiệm **31-54%** so với On-Demand (ít hơn Standard)
- Có thể **convert** (chuyển đổi) sang instance family/OS/tenancy khác
- Không bán được trên Marketplace
- Phù hợp khi không chắc chắn về instance type dài hạn

#### Scheduled Reserved Instances (RI Theo Lịch)
- Chạy trong time window cụ thể (ví dụ: Thứ 2-6, 9am-6pm)
- Tiết kiệm khoảng **5-10%** so với On-Demand
- Phù hợp cho batch jobs chạy theo lịch cố định
- Hiện tại ít phổ biến (thay thế bằng Scheduled Scaling trong ASG)

### Payment Options (Tùy Chọn Thanh Toán)

| Payment Option     | Mô Tả                          | Chiết Khấu   |
| ------------------ | ------------------------------ | ------------ |
| All Upfront        | Trả toàn bộ trước              | Tối đa       |
| Partial Upfront    | Một phần trước, còn lại hàng tháng | Trung bình |
| No Upfront         | Không trả trước, trả hàng tháng | Ít nhất     |

### Term Options (Thời Hạn)

- **1-year term (1 năm):** Tiết kiệm 30-45% — phù hợp nếu không chắc chắn 3 năm
- **3-year term (3 năm):** Tiết kiệm 50-72% — tối đa cho infrastructure cốt lõi

### RI Scope (Phạm Vi)

- **Regional (Theo Vùng):** RI áp dụng cho tất cả AZ trong region, size flexible trong family
- **Zonal (Theo AZ):** RI gắn với AZ cụ thể, không flexible nhưng reserve capacity

### Ví Dụ Tiết Kiệm — m5.large tại us-east-1

```
On-Demand:                  $0.096/giờ = $838/năm
1-yr No Upfront:            $0.060/giờ = $526/năm  → Tiết kiệm $312 (37%)
1-yr All Upfront:           $488 upfront             → Tiết kiệm $350 (42%)
3-yr No Upfront:            $0.038/giờ = $332/năm  → Tiết kiệm $507/năm (60%)
3-yr All Upfront:           $838 upfront ($279/năm)  → Tiết kiệm $559/năm (67%)
```

### Khi Nào Mua RI

```
✅ Mua Reserved Instances khi:
├── Instance đã chạy ổn định > 3 tháng với utilization > 80%
├── Workload production 24/7 không thay đổi trong 1-3 năm
├── Database servers (RDS cũng có RI)
├── Sau khi đã rightsizing để không RI sai size
└── Instance families không cần flexibility giữa families

❌ Không nên mua RI khi:
├── Đang trong giai đoạn scaling nhanh (instance type có thể thay đổi)
├── Chưa có 60-90 ngày data để xác định baseline
├── Workload seasonal (theo mùa) thay đổi nhiều
└── Khi Savings Plans linh hoạt hơn và cũng tiết kiệm tương đương
```

### AWS CLI — Kiểm Tra RI Đang Có

```bash
# Xem tất cả Reserved Instances đang active
aws ec2 describe-reserved-instances \
  --filters "Name=state,Values=active" \
  --query 'ReservedInstances[*].[ReservedInstancesId,InstanceType,InstanceCount,End,FixedPrice,UsagePrice]' \
  --output table

# Xem RI utilization (cần Cost and Usage Report được bật)
aws ce get-reservation-utilization \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY
```

---

## 3️⃣ Spot Instances (Phiên Bản Tạm Thời) {#spot-instances}

### Định Nghĩa

Spot Instances sử dụng EC2 capacity dư thừa (spare capacity) của AWS với giá thấp hơn 60-90% so với On-Demand. AWS có thể **reclaim (thu hồi)** Spot Instance bất kỳ lúc nào khi cần capacity, nhưng sẽ gửi **2-minute warning** (cảnh báo 2 phút) trước.

### Cơ Chế Hoạt Động

```
AWS Spot Market:
┌────────────────────────────────────────────────────────┐
│  Spot Price (Giá Spot) = giá thị trường theo supply/demand │
│                                                        │
│  Khi bạn request Spot:                                 │
│  ├── Spot Price < Spot Price Cap → Instance chạy       │
│  └── AWS cần capacity → 2-min warning → Instance stop  │
│                                                        │
│  Spot Price History (Lịch Sử Giá):                     │
│  ├── Thường ổn định trong nhiều giờ/ngày               │
│  └── Biến động mạnh khi demand tăng đột biến           │
└────────────────────────────────────────────────────────┘
```

### Interruption Rate (Tỷ Lệ Gián Đoạn) Theo Instance Type

AWS công bố Interruption Frequency (tần suất gián đoạn) cho mỗi Spot pool:
- **< 5%** — Rất ổn định (nhiều instance types lớn thường ở đây)
- **5-10%** — Ổn định trung bình
- **> 20%** — Hay bị interrupted (cần xử lý kỹ)

### Khi Nào Dùng Spot

```
✅ Phù Hợp (Fault-Tolerant Workloads):
├── Batch processing: ETL (Extract-Transform-Load), data analysis
├── CI/CD pipelines (build/test không cần 24/7)
├── Machine Learning training jobs
├── Web crawlers, video transcoding
├── Stateless web servers (đằng sau Load Balancer)
└── Big Data: Spark/Hadoop clusters

❌ Không Phù Hợp:
├── Primary databases (stateful, không chịu được interruption)
├── Real-time payment processing
├── Sessions/WebSocket servers cần connection liên tục
└── Anything với strict SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ)
```

### Spot Instance Request Types

| Type            | Mô Tả                                     |
| --------------- | ----------------------------------------- |
| One-time        | Request 1 lần, không tự re-launch khi bị interrupted |
| Persistent      | Tự re-request sau khi bị interrupted       |
| Spot Fleet      | Nhóm nhiều Spot/On-Demand, tự quản lý capacity |
| EC2 Fleet       | Thế hệ mới của Spot Fleet, nhiều tính năng hơn |

### Interruption Behaviors (Hành Vi Khi Bị Gián Đoạn)

```bash
# Khi tạo Spot request, chọn hành vi khi bị reclaimed:
# - terminate: Instance bị terminate (mặc định)
# - stop: Instance bị stop (có thể resume, nhưng chỉ cho persistent requests)
# - hibernate: RAM state được lưu vào EBS, restore khi có capacity

aws ec2 request-spot-instances \
  --instance-count 1 \
  --type "persistent" \
  --instance-interruption-behavior "stop" \
  --launch-specification file://spot-spec.json
```

### Kiểm Tra Spot Price Hiện Tại

```bash
# Xem giá Spot lịch sử 7 ngày cho m5.large
aws ec2 describe-spot-price-history \
  --instance-types m5.large \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -d '7 days ago' --utc +%FT%TZ) \
  --query 'SpotPriceHistory[*].[Timestamp,AvailabilityZone,SpotPrice]' \
  --output table
```

---

## 4️⃣ Savings Plans (Kế Hoạch Tiết Kiệm) {#savings-plans}

### Định Nghĩa

Savings Plans là mô hình định giá linh hoạt, cam kết mức chi tiêu **theo giờ** ($/hour) trong 1 hoặc 3 năm để đổi lấy chiết khấu. Khác với RI cam kết instance cụ thể, Savings Plans cam kết mức **spending rate** (tỷ lệ chi tiêu).

### Ba Loại Savings Plans

#### Compute Savings Plans (Linh Hoạt Nhất)
- Áp dụng cho: **EC2 + Lambda + Fargate**
- Áp dụng cho bất kỳ: instance family, size, region, OS, tenancy
- Tiết kiệm: **17-66%** so với On-Demand
- Tự động áp dụng cho usage phù hợp, giảm dần theo thứ tự tốt nhất

#### EC2 Instance Savings Plans (Tiết Kiệm Nhất)
- Áp dụng cho: EC2 trong **một instance family + region cụ thể** (ví dụ: m5 tại us-east-1)
- Linh hoạt về: size, OS, AZ trong family đó
- Tiết kiệm: **tương đương Standard RI** — cao nhất trong Savings Plans
- Phù hợp khi biết chắc sẽ dùng family nào lâu dài

#### SageMaker Savings Plans
- Áp dụng cho: Amazon SageMaker instances
- Linh hoạt về: instance type, size, region
- Tiết kiệm: đến **64%**

### So Sánh Savings Plans vs Reserved Instances

| Tiêu Chí                | Savings Plans              | Reserved Instances          |
| ----------------------- | -------------------------- | --------------------------- |
| Cam kết dạng gì         | $/giờ chi tiêu             | Instance type/region cụ thể |
| Flexibility             | Cao (Compute SP: mọi thứ) | Thấp (Standard RI)          |
| Tiết kiệm tối đa        | Tương đương RI (EC2 ISP)  | Cao nhất (Standard, 3-yr)   |
| Áp dụng cho Lambda/Fargate | Compute SP: Có           | Không                       |
| Marketplace             | Không bán được             | Standard RI có thể bán      |
| Quản lý                 | Đơn giản hơn               | Phức tạp hơn (nhiều RI)     |

### Khi Nào Mua Savings Plans vs RI

```
Chọn Savings Plans khi:
├── Dùng nhiều instance families và muốn dễ quản lý
├── Có Lambda/Fargate trong mix compute
├── Không chắc sẽ dùng instance family nào 1-3 năm tới
└── Muốn flexibility để đổi instance type không mất commitment value

Chọn Reserved Instances khi:
├── Biết chắc instance type/region sẽ dùng (databases)
├── Muốn tối đa hóa discount cho 1 workload cụ thể
├── Cần reserved capacity (Zonal RI) — Savings Plans không reserve capacity
└── Dùng Windows SQL Server/BYOL (Bring Your Own License — Mang Giấy Phép Riêng)
```

### Hướng Dẫn Mua Savings Plans

```bash
# Bước 1: Xem recommendations từ AWS Cost Explorer
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

# Bước 2: Kiểm tra current coverage
aws ce get-savings-plans-coverage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY

# Bước 3: Mua Savings Plans
aws savingsplans purchase-savings-plan \
  --savings-plan-offering-id <offering-id> \
  --commitment 10.00 \
  --upfront-payment-amount 0
```

---

## 5️⃣ Dedicated Hosts & Dedicated Instances (Máy Chủ Dành Riêng) {#dedicated}

### Dedicated Hosts (Máy Chủ Vật Lý Dành Riêng)

- Bạn nhận một **máy chủ vật lý hoàn toàn** dành riêng cho mình
- Kiểm soát instance placement (vị trí instance) trên host
- Phù hợp cho **BYOL (Bring Your Own License)** — SQL Server, Windows Server, Oracle
- Billing: Per-host (tính theo máy chủ), không phải per-instance
- Có thể mua Reserved (1-3 năm) để giảm chi phí

### Dedicated Instances (Phiên Bản Dành Riêng)

- Instances chạy trên hardware dành riêng cho **account của bạn**
- Không chia sẻ với AWS accounts khác nhưng có thể chia sẻ với instances khác trong cùng account
- Đắt hơn On-Demand khoảng $2/giờ/region + $0.01/giờ/instance
- Phù hợp cho regulatory compliance (tuân thủ quy định) yêu cầu physical isolation

### Khi Nào Dùng Dedicated

```
✅ Dùng Dedicated Hosts khi:
├── BYOL: Windows Server, SQL Server, Oracle (tính license theo core/socket)
├── Compliance yêu cầu single-tenant physical hardware
└── Cần visibility về physical server placement

✅ Dùng Dedicated Instances khi:
├── Compliance yêu cầu không chia sẻ hardware với accounts khác
├── Không có yêu cầu BYOL license
└── Security policy strict về physical isolation

❌ Không nên dùng khi:
└── Chỉ muốn security tốt hơn → IAM, VPC, Security Groups đủ rồi
```

---

## 🔍 Framework Lựa Chọn Pricing Model {#framework}

### Decision Tree (Cây Quyết Định)

```
Workload của bạn là gì?
│
├── Batch/Background có thể interrupt?
│   └── → SPOT INSTANCES (tiết kiệm 60-90%)
│
├── Chạy < 1 năm hoặc unpredictable?
│   └── → ON-DEMAND (trả đúng những gì dùng)
│
├── Event-driven, short duration?
│   └── → LAMBDA (pay-per-request, serverless)
│
├── Production, stable, chạy > 1 năm?
│   ├── Biết chắc instance type? → RESERVED INSTANCES (tối đa savings)
│   └── Muốn linh hoạt hoặc có Lambda/Fargate? → SAVINGS PLANS
│
└── License compliance hoặc BYOL?
    └── → DEDICATED HOSTS
```

### Chiến Lược Phân Lớp (Layered Strategy)

```
Production Architecture (Kiến Trúc Production):

[RESERVED/SAVINGS PLANS] — Baseline capacity (60-70% instances)
         +
[ON-DEMAND] — Buffer cho peak traffic (20-30% instances)
         +
[SPOT] — Batch, CI/CD, non-critical workers (10-20% capacity)

Kết quả: Tiết kiệm 40-60% so với all On-Demand
```

### Quy Tắc Thực Hành

| Câu Hỏi                              | Quy Tắc                                        |
| ------------------------------------ | ---------------------------------------------- |
| Khi nào mua RI/Savings Plans?        | Sau 60-90 ngày để có baseline usage data       |
| Mua bao nhiêu phần trăm?             | 60-70% baseline (không 100% vì usage thay đổi) |
| Trả trước hay không?                 | All Upfront nếu có tiền, ROI thường tốt hơn    |
| 1 năm hay 3 năm?                     | 1 năm nếu không chắc, 3 năm cho core infra     |

---

## 💰 Ví Dụ Tính Toán Thực Tế {#ví-dụ-tính-toán}

### Kịch Bản: Web Application với 10 m5.xlarge

```
Tình huống hiện tại (all On-Demand):
  10 × m5.xlarge × $0.192/giờ × 8,760 giờ/năm = $16,819/năm

Tối ưu với Layered Strategy (Chiến Lược Phân Lớp):
  6 × Compute Savings Plans (1-yr, No Upfront) @ $0.094/giờ × 8,760 = $4,944
  2 × On-Demand buffer                          @ $0.192/giờ × 8,760 = $3,364
  2 × Spot cho batch/workers                    @ $0.065/giờ × 8,760 = $1,139

Tổng chi phí tối ưu: $9,447/năm
Tiết kiệm: $16,819 - $9,447 = $7,372/năm (44%)
```

### Kịch Bản: Batch Processing với 50 c5.2xlarge, 8 giờ/ngày

```
Nếu dùng On-Demand:
  50 × c5.2xlarge × $0.34/giờ × 8 giờ × 365 ngày = $49,640/năm

Dùng Spot Instances:
  50 × c5.2xlarge × $0.12/giờ (Spot avg) × 8 × 365 = $17,520/năm

Tiết kiệm: $49,640 - $17,520 = $32,120/năm (65%)
```

---

## 🎤 Câu Hỏi Phỏng Vấn {#câu-hỏi-phỏng-vấn}

**Q: Khi nào nên dùng Spot thay vì On-Demand?**
> Khi workload fault-tolerant và có thể restart sau interruption. Ví dụ: batch ETL jobs, ML training, CI/CD builds. Lưu ý: cần xử lý 2-minute interruption warning và implement checkpointing cho long-running jobs.

**Q: Savings Plans có thể thay thế hoàn toàn Reserved Instances không?**
> Không hoàn toàn. Savings Plans (đặc biệt Compute SP) linh hoạt và dễ quản lý hơn, nhưng không reserve capacity — khi region đầy, bạn không đảm bảo được instance. Zonal RI vẫn cần khi cần capacity reservation. Với BYOL licensing, Dedicated Hosts + RI là lựa chọn tốt nhất.

**Q: Làm sao biết mình đang lãng phí RI không dùng hết?**
> Dùng `aws ce get-reservation-utilization` hoặc xem AWS Cost Explorer → Reserved Instance Utilization. Nếu utilization < 80% liên tục, cân nhắc bán RI trên Marketplace (Standard RI) hoặc convert sang Convertible RI rồi convert tiếp sang instance type khác đang dùng nhiều hơn.

**Q: Cách chọn 1-year vs 3-year term?**
> 1-year cho workloads đang evolve (tiến hóa) hoặc chưa chắc về technology stack 3 năm tới. 3-year cho core infrastructure ổn định: production databases, message brokers, monitoring servers. ROI của 3-year cao hơn nhưng rủi ro cao nếu architecture thay đổi.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
