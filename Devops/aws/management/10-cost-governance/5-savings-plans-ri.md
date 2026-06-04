# Savings Plans & Reserved Instances — Cam Kết Tiết Kiệm Chi Phí

> **Savings Plans** (Kế Hoạch Tiết Kiệm) và **Reserved Instances — RI** (Phiên Bản Đặt Trước) là hai cơ chế cam kết sử dụng đổi lấy giảm giá lên đến **72% so với On-Demand**. Hiểu rõ sự khác biệt để chọn đúng công cụ cho từng workload.

---

## 📚 Mục Lục

1. [Tổng Quan Commitment Models](#tổng-quan-commitment-models)
2. [Savings Plans — Chi Tiết](#savings-plans--chi-tiết)
3. [Reserved Instances — Chi Tiết](#reserved-instances--chi-tiết)
4. [So Sánh Trực Tiếp](#so-sánh-trực-tiếp)
5. [Payment Options — Tùy Chọn Thanh Toán](#payment-options--tùy-chọn-thanh-toán)
6. [Chiến Lược Commitment](#chiến-lược-commitment)
7. [Monitoring Coverage & Utilization](#monitoring-coverage--utilization)
8. [RI Marketplace](#ri-marketplace)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Tổng Quan Commitment Models

### Tại Sao Cần Commitment?

```
Pricing tiers cho cùng một EC2 instance:

On-Demand (không cam kết):
  m5.xlarge: $0.192/hr = $139/tháng

Spot Instance (không cam kết, có thể bị thu hồi):
  m5.xlarge: ~$0.060/hr = $44/tháng (tiết kiệm 69%)
  Nhưng: Không dùng được cho stateful workloads

Savings Plans 1 năm No Upfront (cam kết $/hr):
  m5.xlarge: $0.134/hr = $97/tháng (tiết kiệm 30%)

Reserved Instance 1 năm No Upfront (cam kết instance):
  m5.xlarge: $0.116/hr = $84/tháng (tiết kiệm 40%)

Reserved Instance 3 năm All Upfront:
  m5.xlarge: ~$0.053/hr = $39/tháng (tiết kiệm 72%)
```

### Quy Tắc Vàng Chọn Commitment

```
Chạy < 1 năm hoặc không ổn định?  → On-Demand / Spot
Chạy 1–3 năm, instance type thay đổi? → Savings Plans
Chạy 1–3 năm, instance type cố định?  → Reserved Instances (giảm giá nhiều hơn)
Batch/fault-tolerant workload?         → Spot Instances
```

---

## 💡 Savings Plans — Chi Tiết

### Khái Niệm

**Savings Plans** cam kết **số tiền spend ($/hr)** thay vì instance cụ thể. Giảm giá áp dụng tự động cho usage phù hợp.

```
Cam kết: "Tôi sẽ chi ít nhất $5.00/hr trong 1 năm"
AWS áp dụng giảm giá tự động lên mọi eligible usage
đến khi đạt $5.00/hr, phần còn lại tính On-Demand
```

### 3 Loại Savings Plans

#### 1. Compute Savings Plans (Kế Hoạch Tiết Kiệm Compute)

```
Phạm vi áp dụng:
  ✅ EC2 (mọi instance family, region, OS)
  ✅ AWS Fargate (serverless containers)
  ✅ AWS Lambda (serverless functions)

Linh hoạt nhất:
  → Đổi instance type tự do (t3 → m5 → r6)
  → Đổi region tự do (us-east-1 → ap-southeast-1)
  → Đổi OS (Linux → Windows — nhưng discount khác nhau)
  → Đổi tenancy (shared → dedicated)

Giảm giá:
  → Lên đến 66% so với On-Demand EC2
  → Kém hơn EC2 Instance Savings Plans ~5%

Dùng khi:
  → Kiến trúc containerized (ECS/EKS/Fargate)
  → Hay thay đổi instance types/regions
  → Muốn 1 commitment cover nhiều workloads
```

#### 2. EC2 Instance Savings Plans (Kế Hoạch Tiết Kiệm EC2 Instance)

```
Phạm vi áp dụng:
  ✅ EC2 — cùng instance family VÀ region

Ít linh hoạt hơn:
  → Phải chỉ định: instance family + region
  → Ví dụ: "m5 trong us-east-1"
  → Được đổi: size (m5.xlarge ↔ m5.2xlarge), OS, tenancy

Giảm giá:
  → Lên đến 72% so với On-Demand (cao nhất)

Dùng khi:
  → Workload ổn định về instance family và region
  → Muốn maximize savings
  → Không có kế hoạch migrate region trong 1–3 năm
```

#### 3. SageMaker Savings Plans

```
Phạm vi áp dụng:
  ✅ Amazon SageMaker (ML training, inference)

Linh hoạt:
  → Áp dụng mọi SageMaker instance types/regions

Giảm giá:
  → Lên đến 64% so với On-Demand SageMaker

Dùng khi:
  → ML workloads ổn định trên SageMaker
```

### Cách Savings Plans Áp Dụng

```
Ví dụ: Compute Savings Plans $5.00/hr

Usage trong 1 giờ:
  EC2 m5.xlarge (us-east-1):  $0.192/hr On-Demand
  ECS Fargate (us-east-1):    $0.04048 vCPU/hr
  Lambda:                     $0.0000000167/request
  
  Total On-Demand would be:   $6.50/hr

Savings Plans áp dụng:
  Discounted rate đến $5.00/hr commitment
  Phần còn lại ($1.50/hr): tính On-Demand
  
Điều quan trọng:
  → Savings Plans KHÔNG cam kết instance nào cụ thể
  → AWS tự động áp dụng discount theo thứ tự ưu tiên
     (highest discount first)
```

---

## 🔒 Reserved Instances — Chi Tiết

### Khái Niệm

**Reserved Instances** cam kết một **instance type cụ thể** (family + size + region + OS) để đổi lấy giảm giá tối đa.

```
Cam kết: "Tôi sẽ dùng m5.xlarge Linux us-east-1 trong 1 năm"
→ Giảm giá từ $0.192/hr xuống $0.116/hr (1 năm No Upfront)
```

### Scope của Reserved Instances

#### Regional RIs (RI Theo Region)

```
Scope: Toàn bộ region (us-east-1)
  
Linh hoạt trong region:
  → Áp dụng cho mọi AZ trong region
  → Instance size flexibility: m5.xlarge RI cover 2x m5.large
    (hoặc 0.5x m5.2xlarge)
  → Không cần chỉ định AZ trước
  
Recommended cho hầu hết use cases
```

#### Zonal RIs (RI Theo Availability Zone)

```
Scope: Một AZ cụ thể (us-east-1a)

Ít linh hoạt hơn:
  → Phải dùng đúng AZ đã chọn
  → Không có size flexibility

Nhưng có thêm một lợi ích:
  → Capacity reservation (đặt trước capacity)
  → Đảm bảo AWS có instance sẵn trong AZ đó
  
Dùng khi:
  → Cần guaranteed capacity trong AZ cụ thể
  → Disaster recovery requirements
```

### Convertible Reserved Instances vs Standard Reserved Instances

| Tiêu Chí                      | Standard RI                   | Convertible RI                     |
| ----------------------------- | ----------------------------- | ---------------------------------- |
| **Giảm giá**                  | Lên đến 72%                   | Lên đến 66%                        |
| **Đổi instance type**         | Không (phải mua lại)          | Có thể đổi sang equal/higher value |
| **Đổi OS**                    | Không                         | Có                                 |
| **Đổi tenancy**               | Không                         | Có                                 |
| **Bán trên RI Marketplace**   | Có                            | Không                              |
| **Phù hợp**                   | Instance type cực kỳ ổn định  | Có thể thay đổi nhưng muốn giảm giá|

### RI Instance Size Flexibility (Linh Hoạt Kích Thước)

```
1 x m5.2xlarge RI = 2 x m5.xlarge RI = 4 x m5.large RI

Ví dụ: Mua 1 x m5.2xlarge Regional RI
  Scenario 1: Chạy 1 x m5.2xlarge → 100% covered
  Scenario 2: Chạy 2 x m5.xlarge  → 100% covered
  Scenario 3: Chạy 1 x m5.xlarge  → 50% covered, 50% On-Demand

Normalization factor (Hệ Số Chuẩn Hóa):
  nano=0.25, micro=0.5, small=1, medium=2, large=4,
  xlarge=8, 2xlarge=16, 4xlarge=32, 8xlarge=64...
```

---

## ⚖️ So Sánh Trực Tiếp

### Savings Plans vs Reserved Instances

| Tiêu Chí                    | Savings Plans (Compute)       | Reserved Instances (Standard)  |
| --------------------------- | ----------------------------- | ------------------------------ |
| **Cam kết**                 | $/hr (tiền spend)             | Instance type cụ thể           |
| **Phạm vi**                 | EC2 + Fargate + Lambda        | Chỉ EC2 (hoặc RDS/ElastiCache) |
| **Linh hoạt instance**      | Tối đa (cross-family, region) | Thấp (cùng family, region)     |
| **Giảm giá tối đa EC2**     | ~66%                          | ~72%                           |
| **Có thể bán lại**          | Không                         | Có (Standard RI Marketplace)   |
| **Capacity reservation**    | Không                         | Có (Zonal RI)                  |
| **Dịch vụ ngoài EC2**       | Lambda, Fargate                | RDS, ElastiCache, Redshift...  |
| **Đề xuất AWS**             | Workload thay đổi             | Workload cố định               |

### Quyết Định Chọn Loại

```
if workload_uses_fargate_or_lambda:
    → Compute Savings Plans (RI không cover được)
    
elif instance_family_fixed AND region_fixed:
    → EC2 Instance Savings Plans HOẶC Standard RIs
    → RIs giảm giá nhiều hơn ~6%
    
elif instance_family_changes OR might_migrate_region:
    → Compute Savings Plans (linh hoạt hơn)
    
elif need_capacity_reservation:
    → Zonal Reserved Instances
    
elif need_RDS_or_ElastiCache_savings:
    → Reserved Instances (Savings Plans không cover RDS)
```

---

## 💳 Payment Options — Tùy Chọn Thanh Toán

### 3 Tùy Chọn Thanh Toán

| Payment Option        | Mô Tả                                      | Giảm Giá | Cash Flow |
| --------------------- | ------------------------------------------ | -------- | --------- |
| **All Upfront**       | Trả toàn bộ ngay — giảm giá tối đa        | Cao nhất | Tệ nhất   |
| **Partial Upfront**   | Trả trước một phần + phần còn lại hàng tháng | Trung bình | Trung bình |
| **No Upfront**        | Không trả trước, trả hàng tháng — linh hoạt nhất | Thấp nhất | Tốt nhất |

### Tính Toán ROI (Return on Investment — Lợi Tức Đầu Tư)

```
Ví dụ: EC2 m5.xlarge Linux us-east-1, 1 năm

On-Demand:          $0.192/hr × 8,760 hr = $1,682/năm
No Upfront 1yr:     $0.116/hr × 8,760 hr = $1,016/năm  → tiết kiệm $666
Partial Upfront 1yr: $562 upfront + $0.06/hr = $1,089/năm → tiết kiệm $593
All Upfront 1yr:    $1,004 (trả hết) = $1,004/năm        → tiết kiệm $678

All Upfront 3yr:    $0.053/hr × 26,280 hr = $1,393/3năm  → tiết kiệm $3,653!
```

### Break-Even Analysis (Phân Tích Điểm Hòa Vốn)

```
Câu hỏi: "Nếu tôi mua RI 1 năm nhưng chỉ dùng 8 tháng, có lợi không?"

All Upfront 1yr RI: $1,004 cố định
On-Demand cho 8 tháng: $0.192 × 24 × 245 = $1,128

Break-even: ~7.1 tháng (dùng 7+ tháng là có lợi)

No Upfront 1yr RI: $0.116 × 24 × 245 = $681 (8 tháng)
On-Demand (8 tháng): $1,128

→ No Upfront luôn sinh lợi nếu dùng > 1 ngày!
  (Chỉ commit đến kỳ tiếp theo, không trả trước)
```

---

## 📐 Chiến Lược Commitment

### FinOps Layered Commitment Strategy

```
Tầng 1: Base Commitment (Cam Kết Nền Tảng)
  → Savings Plans / RIs cho ~60–70% baseline usage
  → Usage ổn định, luôn cần, không bao giờ tắt
  → Dùng 1–3 năm All/Partial Upfront để maximize savings

Tầng 2: Variable Buffer (Buffer Biến Thiên)
  → On-Demand cho 20–30% usage thêm khi cần
  → Workload có thể tăng/giảm theo mùa

Tầng 3: Spiky Workloads (Workload Đột Biến)
  → Spot Instances cho batch jobs, CI/CD, data processing
  → Tiết kiệm 50–90% nhưng chịu được interruption

Total coverage: Giảm chi phí trung bình 30–50%
```

### Quy Trình Mua Commitment

```
BƯỚC 1: Phân Tích 30–90 ngày usage qua Cost Explorer
  → Xác định baseline (usage luôn có)
  → Identify stable instance types/families

BƯỚC 2: Chạy Savings Plans Recommendations
  Cost Explorer → Savings Plans → Recommendations
  → AWS tính toán optimal commitment amount
  
BƯỚC 3: Chọn commitment amount bảo thủ
  → Bắt đầu với 70% recommendations
  → Tránh over-commit: lãng phí tiền cam kết

BƯỚC 4: Bắt đầu với term ngắn
  → 1 năm No Upfront trước để validate
  → Sau 6 tháng: review utilization/coverage
  → Nếu ổn: upgrade sang 3 năm hoặc All Upfront

BƯỚC 5: Monitor và adjust hàng quý
  → SP/RI Utilization Report: target > 90%
  → SP/RI Coverage Report: target > 70%
```

### Savings Plans Queue (Hàng Đợi Mua)

```
Thứ tự AWS áp dụng discount (ưu tiên discount cao nhất trước):

1. EC2 Instance Savings Plans → áp dụng trước
2. Compute Savings Plans     → áp dụng tiếp
3. On-Demand                 → phần còn lại

→ Mua EC2 Instance SP cho workload cố định (discount cao hơn)
→ Mua Compute SP cho phần linh hoạt
→ Để On-Demand cho phần không thể dự đoán
```

---

## 📊 Monitoring Coverage & Utilization

### 4 Metrics Quan Trọng

```
1. SP/RI Utilization (Mức Độ Sử Dụng):
   = Commitment đã được dùng / Tổng commitment đã mua
   Target: > 90%
   Nếu thấp: Đang lãng phí — mua nhiều hơn cần

2. SP/RI Coverage (Độ Bao Phủ):
   = On-Demand cost được cover bởi SP/RI / Tổng On-Demand cost
   Target: 60–80% (không nên 100% vì cần headroom)
   Nếu thấp: Chưa mua đủ SP/RI, đang trả nhiều On-Demand

3. Net Savings Amount ($):
   = On-Demand would-have-been cost - Actual cost với SP/RI
   
4. Effective Savings Rate (%):
   = Net Savings / On-Demand would-have-been cost
```

### Xem Trong Cost Explorer

```
Cost Explorer → Savings Plans
  → Utilization report:
     Filter by: date range, SP type
     See: Daily utilization %, trended over time
     
  → Coverage report:
     See: % On-Demand covered vs uncovered
     Group by: Service, Instance Type, Region
     → Identify gaps: service/region nào cần thêm coverage
```

### AWS Budgets cho SP/RI Monitoring

```yaml
# Budget alert khi SP utilization xuống thấp
BudgetType: SAVINGS_PLANS_UTILIZATION
CoveredBy: COMPUTE_SAVINGS_PLANS
TimeUnit: MONTHLY
BudgetLimit:
  Amount: 90    # Muốn ít nhất 90% utilization
  Unit: PERCENTAGE
AlertThreshold: 80   # Alert khi < 80%
AlertDirection: LOWER_THAN
```

---

## 🏪 RI Marketplace

**RI Marketplace** (Thị Trường RI) cho phép bán **Standard Reserved Instances** chưa dùng hết trước khi hết hạn.

### Khi Nào Dùng RI Marketplace

```
Tình huống bán:
  → Đã mua RI nhưng workload thay đổi (migrate region)
  → Sau khi chuyển sang container/serverless, EC2 RI thừa
  → Muốn upgrade lên instance type mới
  
Tình huống mua:
  → Cần RI nhưng muốn term ngắn hơn 1 năm
  → Người khác bán RI còn 3–6 tháng với giá rẻ hơn
  → Test commitment trước khi mua mới
```

### Hạn Chế

```
Chỉ Standard RIs (không bán được Convertible RIs)
Phí AWS: 12% của tổng giá bán
Người mua có thể set giá cao hơn On-Demand → không ai mua
Thực tế: Marketplace ít liquid, giá cạnh tranh thấp
```

> **Alternative tốt hơn:** Nếu không muốn bán, hãy dùng Convertible RIs để exchange lấy loại khác — linh hoạt hơn Marketplace.

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Savings Plans vs Reserved Instances — khi nào chọn cái nào?**
> **Savings Plans** phù hợp khi workload thay đổi instance type/region, hoặc dùng Fargate/Lambda (RI không cover). **Reserved Instances** phù hợp khi instance type và region hoàn toàn cố định (giảm giá nhiều hơn ~6%), cần capacity reservation, hoặc cần savings cho RDS/ElastiCache/Redshift (Savings Plans không cover). Trong kiến trúc container/serverless hiện đại, **Compute Savings Plans thường là lựa chọn tốt hơn**.

**Q: Utilization và Coverage khác nhau thế nào?**
> **Utilization** — "Đã mua bao nhiêu thì đang dùng bao nhiêu?" (target > 90%). Nếu thấp: đang lãng phí commitment đã mua. **Coverage** — "Bao nhiêu % total usage được cover bởi SP/RI?" (target 60–80%). Nếu thấp: đang trả On-Demand nhiều hơn cần thiết, nên mua thêm.

**Q: Có nên mua All Upfront 3 năm không?**
> Phụ thuộc cash flow và confidence về workload. All Upfront 3 năm tiết kiệm ~72% nhưng trả tiền ngay — nếu workload thay đổi trong 3 năm, mất linh hoạt. Best practice: Bắt đầu với **1 năm No Upfront** để validate baseline; sau 6–12 tháng utilization ổn định > 90%, xem xét **3 năm Partial/All Upfront** cho portion rất ổn định.

**Q: Làm thế nào chia sẻ Savings Plans trong multi-account?**
> Savings Plans mua trong **Management Account (Payer Account — Tài Khoản Thanh Toán)** tự động chia sẻ cho tất cả member accounts trong Organization theo Consolidated Billing. Discount áp dụng automatically — không cần cấu hình thêm. Có thể disable sharing per account nếu muốn phân bổ chi phí độc lập.

---

**Điều Hướng:**
← [4-tagging-strategy.md](4-tagging-strategy.md) | [README.md](README.md)

**Cập Nhật Lần Cuối:** 2026-05-17
