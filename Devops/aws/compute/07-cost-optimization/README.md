# 💰 AWS Compute Cost Optimization — Tối Ưu Chi Phí Compute

> Hướng dẫn toàn diện về tối ưu chi phí AWS Compute — từ lựa chọn Pricing Model (Mô Hình Định Giá) phù hợp, chiến lược Spot Instance (Máy Chủ Tạm Thời), Rightsizing (Chọn Đúng Kích Cỡ) cho đến Savings Plans (Kế Hoạch Tiết Kiệm) và Cost Monitoring (Giám Sát Chi Phí) liên tục.

---

## 📚 Mục Lục

1. [Tại Sao Cost Optimization Quan Trọng?](#tại-sao-cost-optimization-quan-trọng)
2. [Tổng Quan Các Chiến Lược Tiết Kiệm](#tổng-quan-các-chiến-lược-tiết-kiệm)
3. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)
4. [Nguyên Tắc FinOps](#nguyên-tắc-finops)
5. [Quick Wins — Tiết Kiệm Ngay Lập Tức](#quick-wins)
6. [Công Thức Tính Tiết Kiệm](#công-thức-tính-tiết-kiệm)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn)

---

## 💡 Tại Sao Cost Optimization Quan Trọng?

### Thực Tế Ngành

Theo báo cáo của Flexera 2024:
- **35%** chi phí cloud bị lãng phí do over-provisioning (cấp phát dư thừa)
- **$147 tỷ** là tổng chi phí cloud toàn cầu năm 2023
- **68%** doanh nghiệp coi cost optimization là ưu tiên hàng đầu
- Trung bình có thể tiết kiệm **20-40%** chi phí cloud với các biện pháp đơn giản

### Chi Phí Compute Điển Hình

```
Infrastructure Cost Breakdown (Phân Bổ Chi Phí Hạ Tầng):
├── EC2 Instances:         40-50% tổng chi phí AWS
├── RDS/Databases:         20-25%
├── Data Transfer:         10-15%
├── Storage (S3/EBS):      10-15%
└── Các dịch vụ khác:      10-15%
```

### Tác Động Kinh Doanh

| Hành Động Tối Ưu                        | Tiết Kiệm Điển Hình |
| --------------------------------------- | ------------------- |
| Chuyển sang Reserved Instances 1 năm   | 30-40%              |
| Dùng Spot Instances cho batch jobs      | 60-90%              |
| Rightsizing EC2 instances               | 20-30%              |
| Tắt resources không dùng ngoài giờ     | 60-70% (môi trường dev) |
| Compute Savings Plans                   | 17-66%              |

---

## 🗺️ Tổng Quan Các Chiến Lược Tiết Kiệm

### Mô Hình Tối Ưu Chi Phí Theo Lớp

```
┌─────────────────────────────────────────────────────────────────┐
│            TẦNG 1: ĐÚNG PRICING MODEL (NGAY LẬP TỨC)           │
│  On-Demand → Reserved / Savings Plans → Spot                    │
│  Tiết kiệm: 17-90% tùy workload                                 │
├─────────────────────────────────────────────────────────────────┤
│            TẦNG 2: ĐÚNG KÍCH CỠ (TUẦN 1-4)                     │
│  Rightsizing + Terminate unused resources                        │
│  Tiết kiệm: 20-35% chi phí EC2                                  │
├─────────────────────────────────────────────────────────────────┤
│            TẦNG 3: TỐI ƯU KIẾN TRÚC (THÁNG 1-3)               │
│  Serverless migration + Container consolidation                  │
│  Tiết kiệm: 40-70% cho eligible workloads                       │
├─────────────────────────────────────────────────────────────────┤
│            TẦNG 4: FINOPS VĂN HÓA (LIÊN TỤC)                   │
│  Tagging + Chargeback + Accountability                           │
│  Tiết kiệm: 10-20% qua trách nhiệm giải trình                   │
└─────────────────────────────────────────────────────────────────┘
```

### Ma Trận Chiến Lược Theo Workload

| Loại Workload                    | Chiến Lược Tốt Nhất              | Tiết Kiệm  |
| -------------------------------- | -------------------------------- | ---------- |
| Production 24/7, predictable     | Reserved Instances / Savings Plans | 30-60%  |
| Batch processing, flexible time  | Spot Instances                   | 60-90%     |
| Dev/Test environments            | Tắt ngoài giờ + Spot             | 60-80%     |
| Spiky / unpredictable traffic    | On-Demand + Auto Scaling         | Linh hoạt  |
| Event-driven, short duration     | Lambda (pay-per-use)             | Rất cao    |
| Container workloads              | Fargate Spot + Savings Plans     | 40-70%     |

---

## 📁 Nội Dung Chi Tiết

### [1. Pricing Models — Mô Hình Định Giá](./1-pricing-models.md)

**Bao gồm:**
- **On-Demand** — Trả theo giờ, không cam kết
- **Reserved Instances — RI** — Cam kết 1-3 năm, giảm 30-72%
- **Spot Instances** — Dùng capacity dư thừa, giảm đến 90%
- **Savings Plans** — Cam kết mức chi tiêu/giờ, linh hoạt hơn RI
- **Dedicated Hosts/Instances** — Phần cứng dành riêng
- **Framework chọn pricing model** cho từng loại workload

### [2. Spot Strategy — Chiến Lược Spot](./2-spot-strategy.md)

**Bao gồm:**
- Cơ chế hoạt động của Spot Instances và Spot Interruption (Gián Đoạn Spot)
- **Instance Diversification** — Đa dạng hóa instance types để tăng availability
- **Interruption Handling** — Xử lý 2-phút warning trước khi bị thu hồi
- **Checkpointing** — Lưu tiến trình định kỳ để resume sau interruption
- Spot trong Auto Scaling Groups với Mixed Instance Policy
- **Spot với ECS, EKS** — Chiến lược triển khai container trên Spot

### [3. Rightsizing — Chọn Đúng Kích Cỡ](./3-rightsizing.md)

**Bao gồm:**
- **AWS Compute Optimizer** — Gợi ý rightsizing tự động dựa trên ML
- CloudWatch Utilization Analysis (Phân Tích Sử Dụng)
- Quy trình rightsizing: Đo lường → Phân tích → Hành động → Kiểm tra
- Nhận diện over-provisioned (cấp phát dư thừa) vs under-provisioned instances
- **Lambda rightsizing** — Memory và timeout optimization
- **ECS/EKS rightsizing** — CPU/Memory request & limits

### [4. Savings Plans — Kế Hoạch Tiết Kiệm](./4-savings-plans.md)

**Bao gồm:**
- **Compute Savings Plans** — Linh hoạt nhất, áp dụng cho EC2/Lambda/Fargate
- **EC2 Instance Savings Plans** — Tiết kiệm tối đa trong family cụ thể
- **SageMaker Savings Plans** — Dành cho ML workloads
- So sánh chi tiết Savings Plans vs Reserved Instances
- Hướng dẫn mua: Phân tích baseline, chọn term, Payment Options
- **Chiến lược kết hợp:** Savings Plans + Spot + On-Demand

### [5. Cost Monitoring — Giám Sát Chi Phí](./5-cost-monitoring.md)

**Bao gồm:**
- **AWS Cost Explorer** — Phân tích chi phí lịch sử và dự báo
- **AWS Budgets** — Cảnh báo và hành động tự động khi vượt ngưỡng
- **AWS Cost Anomaly Detection** — Phát hiện chi phí bất thường bằng ML
- **Trusted Advisor** — Kiểm tra 5 pillar bao gồm cost optimization
- **Tagging Strategy** — Tag resources để phân bổ chi phí chính xác
- **Cost Allocation Tags** — Chargeback/Showback cho teams/projects

---

## 📐 Nguyên Tắc FinOps

FinOps (Cloud Financial Operations — Quản Lý Tài Chính Đám Mây) là văn hóa và thực hành để tối đa hóa giá trị business từ cloud.

### Ba Giai Đoạn FinOps

```
INFORM (Thông Báo)
├── Ai đang tiêu bao nhiêu?
├── Chi tiêu tăng/giảm như thế nào?
└── Forecasting (Dự Báo) cho giai đoạn tiếp theo

OPTIMIZE (Tối Ưu)
├── Rightsizing recommendations
├── Terminate unused resources
└── Committed use discounts

OPERATE (Vận Hành)
├── Ownership: mỗi team chịu trách nhiệm chi phí
├── Automation: tự động hóa tối ưu
└── Continuous improvement loop
```

### Nguyên Tắc Cốt Lõi

1. **Teams cần biết chi phí của họ** — Visibility là nền tảng
2. **Quyết định dựa trên business value** — Không phải tiết kiệm tối đa mà là ROI tốt nhất
3. **Cloud variable cost** — Chi tiêu cloud cần flexible như doanh thu
4. **Accountability (Trách nhiệm giải trình)** — Team engineer, finance, business cùng chịu trách nhiệm
5. **Just-in-time provisioning** — Chỉ mua khi cần, không mua phòng xa

---

## ⚡ Quick Wins — Tiết Kiệm Ngay Lập Tức {#quick-wins}

### Tuần 1 — Không Rủi Ro

```bash
# 1. Tắt EC2 instances không dùng ngoài giờ (dev/test)
# Tiết kiệm: 70% chi phí môi trường dev

# 2. Xóa EBS snapshots cũ (> 90 ngày không dùng)
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[?StartTime<=`2024-01-01`].[SnapshotId,StartTime,VolumeSize]' \
  --output table

# 3. Tìm EBS volumes không gắn vào instance nào
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' \
  --output table

# 4. Tìm Elastic IPs không gắn vào instance
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' \
  --output table

# 5. Tìm Load Balancers không có targets
aws elbv2 describe-load-balancers --query 'LoadBalancers[*].[LoadBalancerName,DNSName]'
```

### Tuần 2-4 — Có Kế Hoạch

1. **Analyze CloudWatch metrics** — Tìm instances dùng < 10% CPU trung bình
2. **AWS Compute Optimizer** — Xem gợi ý rightsizing, đánh giá từng gợi ý
3. **Tag audit** — Kiểm tra resources chưa có tag, gán cost allocation tags
4. **Delete old AMIs** — AMIs không dùng + snapshots liên quan
5. **Review Lambda timeout/memory** — Dùng Lambda Power Tuning tool

### Tháng 1-3 — Cam Kết

1. **Mua Savings Plans** — Sau khi đã có 60-90 ngày data baseline
2. **Reserved Instances** — Cho databases và workloads cố định
3. **Spot cho batch** — Migrate batch processing sang Spot
4. **Serverless migration** — Đánh giá workloads phù hợp cho Lambda/Fargate

---

## 🔢 Công Thức Tính Tiết Kiệm

### Reserved Instance ROI

```
Monthly Saving = (On-Demand Hourly Rate - RI Effective Hourly Rate) × 730 giờ/tháng

Break-even Period = Upfront Payment / Monthly Saving

Ví dụ — m5.large tại us-east-1:
  On-Demand:       $0.096/giờ = $70.08/tháng
  1-yr No Upfront: $0.060/giờ = $43.80/tháng
  Monthly Saving:  $26.28/tháng (37.5%)
```

### Spot Savings

```
Spot Saving = (On-Demand Price - Spot Price) / On-Demand Price × 100%

Thực tế điển hình:
  On-Demand m5.large: $0.096/giờ
  Spot m5.large:      $0.030/giờ (biến động)
  Saving:             ~69%
```

### Compute Optimizer Potential Saving

```
If current: m5.xlarge (4 vCPU, 16 GB) at $0.192/giờ
Optimizer:  m5.large  (2 vCPU, 8 GB)  at $0.096/giờ (nếu CPU avg < 20%)

Monthly saving: ($0.192 - $0.096) × 730 = $70.08/tháng/instance
```

---

## 🎤 Câu Hỏi Phỏng Vấn Thường Gặp {#câu-hỏi-phỏng-vấn}

### Câu Hỏi Cơ Bản

**Q: Sự khác biệt giữa Reserved Instances và Savings Plans?**
> Reserved Instances cam kết với instance type + region + OS cụ thể, tiết kiệm tối đa. Savings Plans cam kết mức chi tiêu theo giờ ($x/giờ), linh hoạt áp dụng cho nhiều services và regions. Savings Plans dễ quản lý hơn nhưng Reserved Instances cho EC2 family cụ thể tiết kiệm nhiều hơn một chút.

**Q: Khi nào KHÔNG nên dùng Spot Instances?**
> Khi workload stateful (có state), không thể interrupted trong mid-process, hoặc cần latency guarantee. Ví dụ: primary database, session servers, real-time payment processing. Spot phù hợp cho stateless, fault-tolerant, batch workloads.

**Q: Làm sao biết EC2 instance của mình đang over-provisioned?**
> Dùng AWS Compute Optimizer (sau 14 ngày thu thập metric), kiểm tra CloudWatch Average CPU < 10-20%, Memory utilization < 30%, Network throughput thấp. Compute Optimizer tự động gợi ý instance type nhỏ hơn với cùng performance profile.

### Câu Hỏi Nâng Cao

**Q: Thiết kế cost optimization strategy cho startup đang scale nhanh?**
> (1) On-Demand + Auto Scaling cho giai đoạn đầu khi traffic unpredictable. (2) Sau 60-90 ngày có baseline, mua Compute Savings Plans cho 50-60% baseline usage. (3) Spot cho batch và CI/CD. (4) Review và rightsizing hàng tháng với Compute Optimizer. (5) Implement tagging và Cost Allocation ngay từ đầu.

**Q: Khi cần giảm chi phí 30% trong 30 ngày, bạn làm gì?**
> (1) Ngay lập tức: tắt resources không dùng, xóa orphaned EBS/Elastic IPs. (2) Tuần 1: rightsizing với Compute Optimizer, schedule dev/test tắt ngoài giờ. (3) Tuần 2: migrate batch jobs sang Spot. (4) Tuần 3-4: mua Savings Plans cho production baseline. Target: 30% là khả thi trong 30 ngày với các bước trên.

---

## 🔗 Điều Hướng

| File                                    | Nội Dung                              |
| --------------------------------------- | ------------------------------------- |
| [1-pricing-models.md](./1-pricing-models.md) | On-Demand, Reserved, Spot, Savings Plans |
| [2-spot-strategy.md](./2-spot-strategy.md)   | Spot interruption, diversification    |
| [3-rightsizing.md](./3-rightsizing.md)       | Compute Optimizer, utilization        |
| [4-savings-plans.md](./4-savings-plans.md)   | Compute vs EC2 vs SageMaker plans     |
| [5-cost-monitoring.md](./5-cost-monitoring.md) | Cost Explorer, Budgets, alerts      |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
