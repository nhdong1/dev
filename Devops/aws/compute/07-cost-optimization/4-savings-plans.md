# 💼 AWS Savings Plans — Kế Hoạch Tiết Kiệm

> Hướng dẫn chi tiết về AWS Savings Plans — ba loại Compute SP, EC2 Instance SP, SageMaker SP, cách phân tích và mua, so sánh với Reserved Instances, và chiến lược kết hợp để tối đa hóa tiết kiệm.

---

## 📚 Mục Lục

1. [Savings Plans Là Gì?](#savings-plans-là-gì)
2. [Ba Loại Savings Plans](#ba-loại)
3. [Compute Savings Plans Chi Tiết](#compute-sp)
4. [EC2 Instance Savings Plans Chi Tiết](#ec2-instance-sp)
5. [SageMaker Savings Plans](#sagemaker-sp)
6. [So Sánh Savings Plans vs Reserved Instances](#so-sánh)
7. [Hướng Dẫn Mua Savings Plans](#hướng-dẫn-mua)
8. [Quản Lý Sau Khi Mua](#quản-lý)
9. [Chiến Lược Kết Hợp](#chiến-lược-kết-hợp)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📌 Savings Plans Là Gì? {#savings-plans-là-gì}

### Cơ Chế Hoạt Động

Savings Plans là một **commitment (cam kết)** theo dạng: "Tôi đồng ý chi tối thiểu **$X mỗi giờ** cho AWS Compute trong 1 hoặc 3 năm."

```
Cách áp dụng Savings Plans:

Giờ này bạn dùng:
├── 3 × m5.large On-Demand: $0.096 × 3 = $0.288/giờ
├── 2 × Lambda invocations cost: $0.050/giờ
└── Tổng: $0.338/giờ

Savings Plan của bạn: commit $0.200/giờ

Áp dụng:
├── SP cover $0.200/giờ với discounted rate
└── Phần dư $0.138/giờ → tính On-Demand rate

Kết quả: Phần được cover bởi SP tự động nhận discount 17-66%
```

### Ưu Điểm Chính

| Ưu Điểm                   | Mô Tả                                                |
| ------------------------- | ---------------------------------------------------- |
| **Đơn giản**              | Chỉ cam kết $ amount, không phải instance type      |
| **Tự động áp dụng**       | AWS tự match SP với eligible usage                  |
| **Multi-service (Compute SP)** | Áp dụng cho EC2 + Lambda + Fargate              |
| **Cross-region (Compute SP)** | Áp dụng cho bất kỳ region nào                  |
| **Flexible**              | Instance type thay đổi vẫn được cover               |

---

## 🗂️ Ba Loại Savings Plans {#ba-loại}

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPUTE SAVINGS PLANS                        │
│  Linh hoạt nhất — EC2 + Lambda + Fargate, bất kỳ region        │
│  Discount: 17-66% vs On-Demand                                  │
├─────────────────────────────────────────────────────────────────┤
│                 EC2 INSTANCE SAVINGS PLANS                      │
│  Tiết kiệm nhiều nhất — 1 instance family trong 1 region        │
│  Discount: tương đương Standard RI (lên đến 72%)                │
├─────────────────────────────────────────────────────────────────┤
│                  SAGEMAKER SAVINGS PLANS                        │
│  Dành cho ML workloads — SageMaker instance types               │
│  Discount: lên đến 64%                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ Compute Savings Plans Chi Tiết {#compute-sp}

### Phạm Vi Áp Dụng

Compute Savings Plans áp dụng cho **bất kỳ combination nào** của:
- **EC2 instances:** mọi instance family, size, OS, tenancy
- **AWS Lambda:** function compute costs (không bao gồm request costs)
- **AWS Fargate:** ECS và EKS Fargate tasks

Trên **bất kỳ AWS region** nào.

### Discount Rate (Tỷ Lệ Chiết Khấu)

```
Ví dụ Compute SP discount cho m5.large Linux, us-east-1:
  On-Demand:                  $0.096/giờ  (baseline 100%)
  Compute SP (1-yr No Upfront): $0.078/giờ (19% discount)
  Compute SP (1-yr All Upfront): $0.076/giờ (21% discount)
  Compute SP (3-yr No Upfront): $0.060/giờ (38% discount)
  Compute SP (3-yr All Upfront): $0.056/giờ (42% discount)
```

### Khi Nào Chọn Compute SP

```
✅ Tốt nhất khi:
├── Workload mix EC2 + Lambda + Fargate
├── Instance type có thể thay đổi qua thời gian
├── Cần apply discount cho nhiều regions
├── Đang trong quá trình cloud transformation
└── Team FinOps muốn đơn giản hóa commitment management

Ví dụ điển hình:
Công ty SaaS có:
  - 20 EC2 instances (mix m5, c5, r5 families)
  - Lambda functions xử lý async events
  - ECS Fargate cho microservices
→ Compute SP là lựa chọn tốt nhất (1 commitment cover tất cả)
```

### Application Order (Thứ Tự Áp Dụng)

```
Khi có nhiều SP/RI, AWS áp dụng theo thứ tự tối ưu nhất:

1. Zonal Reserved Instances (áp dụng trước, bao gồm capacity reservation)
2. Regional Reserved Instances
3. EC2 Instance Savings Plans (trong family/region commit)
4. Compute Savings Plans (linh hoạt nhất, áp dụng sau cùng)
5. On-Demand (phần còn lại)
```

---

## 2️⃣ EC2 Instance Savings Plans Chi Tiết {#ec2-instance-sp}

### Cam Kết Cụ Thể Hơn

EC2 Instance Savings Plans yêu cầu cam kết với:
- **Instance family** (dòng instance): ví dụ `m5` hoặc `c6i`
- **AWS Region** cụ thể: ví dụ `us-east-1`

Nhưng linh hoạt về:
- **Instance size:** m5.large, m5.xlarge, m5.2xlarge, ... đều được cover
- **OS:** Linux, Windows, RHEL, SUSE
- **Tenancy:** Default, Dedicated

### Discount Rate

```
EC2 Instance SP cho m5 family, us-east-1:
  On-Demand m5.large:               $0.096/giờ  (100%)
  EC2 ISP (1-yr No Upfront):        $0.060/giờ  (37.5% discount)
  EC2 ISP (1-yr All Upfront):       $0.058/giờ  (39.6% discount)
  EC2 ISP (3-yr No Upfront):        $0.037/giờ  (61.5% discount)
  EC2 ISP (3-yr All Upfront):       $0.034/giờ  (64.6% discount)

→ Tiết kiệm nhiều hơn Compute SP ~5-10% nhưng kém linh hoạt hơn
```

### Khi Nào Chọn EC2 Instance SP

```
✅ Tốt nhất khi:
├── Biết chắc sẽ dùng 1 instance family trong 1 region lâu dài
├── Workload chủ yếu là EC2, không có nhiều Lambda/Fargate
├── Muốn maximize discount hơn là flexibility
└── Sau khi đã rightsizing và ổn định instance family

Ví dụ điển hình:
Database servers: r6i family (Memory Optimized) tại us-east-1
Web servers: c6i family (Compute Optimized) tại us-east-1
→ 2 EC2 Instance SPs cho 2 families này, discount cao nhất
```

---

## 3️⃣ SageMaker Savings Plans {#sagemaker-sp}

### Phạm Vi Áp Dụng

Áp dụng cho Amazon SageMaker:
- **Notebook Instances** (Máy Chủ Notebook)
- **Training Jobs** (Công Việc Huấn Luyện)
- **Real-time Inference** (Suy Luận Thời Gian Thực)
- **Batch Transform** (Biến Đổi Hàng Loạt)
- **Processing Jobs** (Công Việc Xử Lý)
- **Data Wrangler** (Chuẩn Bị Dữ Liệu)

### Discount

Lên đến **64%** so với On-Demand SageMaker pricing, tùy instance type và region.

### Khi Nào Cần

```
Khi tổng chi phí SageMaker > $1,000/tháng → xem xét SageMaker SP.
ML team với training jobs thường xuyên → tiết kiệm đáng kể.
```

---

## ⚖️ So Sánh Savings Plans vs Reserved Instances {#so-sánh}

### Bảng So Sánh Chi Tiết

| Tiêu Chí                    | Compute SP      | EC2 Instance SP  | Standard RI       | Convertible RI    |
| --------------------------- | --------------- | ---------------- | ----------------- | ----------------- |
| **Cam kết**                 | $/giờ          | $/giờ + family + region | Instance type + region | Instance type + region |
| **Instance flexibility**    | Bất kỳ family  | Trong 1 family   | Size trong family | Có thể convert   |
| **Region flexibility**      | Bất kỳ         | 1 region         | Regional hoặc Zonal | Regional hoặc Zonal |
| **Lambda/Fargate**          | ✅ Có          | ❌ Không         | ❌ Không          | ❌ Không          |
| **Capacity Reservation**    | ❌ Không       | ❌ Không         | Chỉ Zonal RI      | Chỉ Zonal RI     |
| **Có thể bán lại**          | ❌ Không       | ❌ Không         | ✅ Có (Marketplace) | ❌ Không        |
| **Discount (1-yr, No Up)** | ~17-38%        | ~37-42%          | ~30-45%           | ~25-37%           |
| **Discount (3-yr, All Up)** | ~42-66%       | ~64-72%          | ~50-72%           | ~40-54%           |
| **Quản lý**                 | Đơn giản       | Đơn giản         | Phức tạp          | Phức tạp          |

### Quyết Định: Savings Plans hay Reserved Instances?

```
Dùng Savings Plans khi:
├── Workload mix EC2 + Lambda + Fargate → Compute SP
├── Không chắc instance family/region 1-3 năm tới
├── Muốn đơn giản hóa (1-2 commitments thay vì 20+ RIs)
└── Đang migrate giữa instance families

Dùng Reserved Instances khi:
├── Cần CAPACITY RESERVATION (Zonal RI) — SP không có tính năng này
├── BYOL Windows/SQL Server (RI với Dedicated Host)
├── Muốn tối đa discount cho 1 family cố định (Standard 3-yr RI)
└── Cần flexibility bán RI nếu không dùng (Standard RI Marketplace)
```

---

## 🛒 Hướng Dẫn Mua Savings Plans {#hướng-dẫn-mua}

### Bước 1: Thu Thập Baseline Data

```bash
# Xem current On-Demand spending
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Xem hourly On-Demand EC2 cost (baseline cho Savings Plans)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity HOURLY \
  --filter '{"Dimensions": {"Key": "PURCHASE_TYPE", "Values": ["On-Demand"]}}' \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```

### Bước 2: Xem AWS Recommendations

```bash
# Recommendations từ Cost Explorer (tốt nhất khi có 3+ tháng data)
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS \
  --query 'Recommendations[0].{
    EstimatedROI:EstimatedROI,
    CurrentCost:CurrentOnDemandSpend,
    EstimatedSavings:EstimatedSavingsAmount,
    HourlyCommitment:HourlyCommitmentToPurchase
  }'
```

### Bước 3: Xác Định Commitment Amount

```
Chiến lược an toàn — Mua 60-70% baseline:

Baseline hourly On-Demand spend: $100/giờ

Option A (An Toàn):  Commit $60/giờ  (60%)
  → Cover 60% cost với discount, 40% On-Demand
  → Ít rủi ro nếu usage giảm

Option B (Tối Ưu):   Commit $70/giờ  (70%)
  → Cover 70% cost, tối đa hóa savings

Option C (Aggressive): Commit $90/giờ (90%)
  → Tối đa savings nhưng rủi ro nếu workload scale down
  → Không khuyến nghị khi chưa ổn định

→ Quy tắc: Bao giờ cũng dùng "steady-state minimum" (mức tối thiểu ổn định)
  không phải average hay maximum.
```

### Bước 4: Chọn Term và Payment

```
Term (Thời Hạn):
  1 năm: ROI cao, linh hoạt hơn, phù hợp khi uncertain
  3 năm: ROI tốt nhất, dành cho core infrastructure stable

Payment Options (Tùy Chọn Thanh Toán):
  All Upfront: Discount tối đa, cần cash upfront
  Partial Upfront: Cân bằng giữa cash flow và savings
  No Upfront: Linh hoạt cash flow nhất, discount ít nhất

ROI comparison cho $100/giờ Compute SP (3-yr):
  No Upfront: Break-even ~2 tháng, save $1,800/tháng
  All Upfront: Break-even ~4 tháng (upfront cost), save $1,900/tháng
```

### Bước 5: Mua Savings Plans

```bash
# Xem available Savings Plans offerings
aws savingsplans describe-savings-plans-offerings \
  --product-type "EC2" \
  --savings-plans-types "COMPUTE" \
  --durations 31536000 \  # 1 năm tính bằng giây
  --payment-options "NO_UPFRONT"

# Mua Savings Plan
aws savingsplans purchase-savings-plan \
  --savings-plan-offering-id "<offering-id-from-above>" \
  --commitment "50.00" \
  --purchase-time "2024-01-01T00:00:00" \
  --upfront-payment-amount "0"

# Verify purchase
aws savingsplans describe-savings-plans \
  --states "active" \
  --query 'savingsPlans[*].{
    ID:savingsPlanId,
    Type:savingsPlanType,
    Commitment:commitment,
    End:end
  }' \
  --output table
```

---

## 📊 Quản Lý Sau Khi Mua {#quản-lý}

### Theo Dõi Coverage (Độ Bao Phủ)

```bash
# Coverage: % spending được cover bởi Savings Plans
aws ce get-savings-plans-coverage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --filter '{"Dimensions": {"Key": "SERVICE", "Values": ["Amazon EC2"]}}' \
  --query 'SavingsPlansCoverages[*].{
    TimePeriod:TimePeriod,
    CoverageHoursPercentage:Coverage.CoverageHoursPercentage,
    OnDemandCost:Coverage.OnDemandCost,
    SpendCoveredBySP:Coverage.SpendCoveredBySavingsPlans
  }'

# Mục tiêu: Coverage > 80%
# Coverage thấp = đang dùng nhiều On-Demand, cân nhắc mua thêm SP
```

### Theo Dõi Utilization (Mức Sử Dụng)

```bash
# Utilization: % commitment thực sự được sử dụng
aws ce get-savings-plans-utilization \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --query 'SavingsPlansUtilizationsByTime[*].{
    Period:TimePeriod.Start,
    Utilization:Utilization.UtilizationPercentage,
    UnusedCommitment:Savings.NetSavings
  }'

# Utilization thấp (< 80%) = bạn đang lãng phí phần commitment thừa
# Cần: giảm infra xuống hoặc terminate instances không dùng
```

### CloudWatch Alarm Cho SP Utilization

```bash
# Tạo alarm cảnh báo khi SP Utilization < 80%
aws cloudwatch put-metric-alarm \
  --alarm-name "SavingsPlanLowUtilization" \
  --alarm-description "Savings Plans utilization below 80%" \
  --namespace "AWS/Billing" \
  --metric-name "SavingsPlanUtilization" \
  --dimensions Name=SavingsPlanArn,Value="arn:aws:savingsplans::..." \
  --statistic Average \
  --period 86400 \
  --threshold 80 \
  --comparison-operator LessThanThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:123456789:cost-alerts" \
  --treat-missing-data notBreaching
```

---

## 🎯 Chiến Lược Kết Hợp {#chiến-lược-kết-hợp}

### Model 3 Lớp Tối Ưu

```
Kiến Trúc Tối Ưu Chi Phí:

┌─────────────────────────────────────────────────────────┐
│  LAYER 1: SAVINGS PLANS / RESERVED INSTANCES (60-70%)  │
│  Cho baseline (mức nền) capacity ổn định               │
│  Discount: 30-66%                                       │
├─────────────────────────────────────────────────────────┤
│  LAYER 2: ON-DEMAND (20-30%)                           │
│  Cho buffer capacity và unpredictable peaks             │
│  Trả giá On-Demand                                      │
├─────────────────────────────────────────────────────────┤
│  LAYER 3: SPOT INSTANCES (10-20%)                      │
│  Cho batch jobs, CI/CD, non-critical workers            │
│  Discount: 60-90%                                       │
└─────────────────────────────────────────────────────────┘

Kết quả: Tiết kiệm 40-60% so với all On-Demand
```

### Ví Dụ Cụ Thể: E-commerce Platform

```
Workload: 50 EC2 instances (mix m5 và c5), Lambda, ECS Fargate

Trước khi tối ưu (all On-Demand):
  50 × m5.xlarge avg @ $0.192 × 730 = $7,008/tháng
  Lambda + Fargate                   = $2,000/tháng
  Tổng:                              = $9,008/tháng

Sau khi tối ưu:
  Compute SP $60/giờ (1-yr, No Up):  $43,800/năm = $3,650/tháng (cover EC2 + Lambda + Fargate baseline)
  On-Demand cho peak + buffer:        $1,500/tháng
  Spot cho batch workers (10 instances): $400/tháng
  
  Tổng tối ưu: $5,550/tháng
  Tiết kiệm: $9,008 - $5,550 = $3,458/tháng (38%)
  Tiết kiệm hàng năm: $41,496
```

### Quyết Định Framework

```
Bước 1: Có cần capacity reservation không?
         Có → Zonal Reserved Instance (SP không có tính năng này)
         Không → tiếp tục bước 2

Bước 2: Workload có dùng Lambda/Fargate không?
         Có → Compute Savings Plans
         Không → tiếp tục bước 3

Bước 3: Instance family có thay đổi trong 1-3 năm không?
         Có thể thay đổi → Compute SP hoặc Convertible RI
         Ổn định → EC2 Instance SP hoặc Standard RI

Bước 4: Muốn khả năng bán lại nếu không dùng?
         Có → Standard Reserved Instance (Marketplace)
         Không → EC2 Instance SP (đơn giản hơn)
```

---

## 🎤 Câu Hỏi Phỏng Vấn {#câu-hỏi-phỏng-vấn}

**Q: Savings Plans và Reserved Instances khác nhau cơ bản ở điểm gì?**
> Reserved Instances cam kết với **một instance cụ thể** (type, size, region, OS). Savings Plans cam kết với **mức chi tiêu** ($/giờ). Savings Plans linh hoạt hơn nhiều — Compute SP áp dụng cho bất kỳ EC2 instance, Lambda, Fargate, và bất kỳ region. Trade-off: Standard RI tiết kiệm nhiều hơn một chút cho workload cố định, và RI có capacity reservation mà SP không có.

**Q: Tại sao không nên mua SP 100% của On-Demand spending?**
> Vì usage có thể giảm: scaling down, terminate instances, migrate services. Nếu mua 100% nhưng usage giảm 30%, bạn lãng phí 30% commitment. Khuyến nghị: commit 60-70% của "steady-state minimum" (mức tối thiểu ổn định), không phải average. Phần còn lại dùng On-Demand + Spot. Sau mỗi 6 tháng, review và top-up nếu cần.

**Q: Làm sao biết mình đang "waste" (lãng phí) Savings Plans đã mua?**
> Xem Savings Plans Utilization trong Cost Explorer. Nếu utilization < 80% trong nhiều ngày, bạn đang waste phần commitment dư. Nguyên nhân thường: scale down infra, terminate instances, workload thay đổi. Giải pháp: không thể hủy SP đã mua (hết hạn sau term), nhưng có thể tăng workload để tận dụng, hoặc accept sẽ không maximize SP value này.

**Q: Khi nào nên mua Savings Plans 3 năm vs 1 năm?**
> 3 năm cho infrastructure **cốt lõi và ổn định**: database servers, monitoring stack, core API services. 1 năm cho services **đang evolve**: khi đang migrate, khi technology stack chưa chắc. Rule of thumb: nếu bạn biết chắc 80% sẽ dùng infrastructure đó sau 3 năm → 3-yr All Upfront là ROI tốt nhất. Nếu không chắc → 1-yr No Upfront, flexibility quan trọng hơn maximum savings.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
