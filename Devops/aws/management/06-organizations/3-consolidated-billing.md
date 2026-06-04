# Consolidated Billing — Hóa Đơn Hợp Nhất & Tối Ưu Chi Phí

> **Consolidated Billing** (Hóa Đơn Hợp Nhất) là tính năng của AWS Organizations cho phép tổng hợp chi phí của tất cả member accounts vào một hóa đơn duy nhất trong Management Account. Ngoài sự tiện lợi trong quản lý, Consolidated Billing mang lại lợi thế về **volume discount** (Chiết Khấu Theo Khối Lượng) và chia sẻ **Reserved Instances** (Phiên Bản Dự Trữ) và **Savings Plans** (Kế Hoạch Tiết Kiệm) giữa các accounts.

---

## 📚 Mục Lục

1. [Cơ Chế Consolidated Billing](#cơ-chế-consolidated-billing)
2. [Volume Discount — Chiết Khấu Theo Khối Lượng](#volume-discount)
3. [Reserved Instance Sharing — Chia Sẻ RI](#reserved-instance-sharing)
4. [Savings Plans Sharing](#savings-plans-sharing)
5. [Spot Instance Và Free Tier](#spot-instance-và-free-tier)
6. [Cost Allocation Tags — Phân Bổ Chi Phí Bằng Tag](#cost-allocation-tags)
7. [Billing Dashboard Và Báo Cáo](#billing-dashboard-và-báo-cáo)
8. [Quản Lý Chi Phí Đa Account](#quản-lý-chi-phí-đa-account)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cơ Chế Consolidated Billing

### Luồng Thanh Toán

```
Member Account A: Chi $500
Member Account B: Chi $1,200
Member Account C: Chi $800
         │
         ▼
   Management Account
   ├── Nhận hóa đơn hợp nhất: $2,500
   ├── Có thể xem breakdown theo từng account
   ├── Nhận discount dựa trên tổng usage
   └── Thanh toán bằng payment method của Management Account
```

### Lợi Ích Chính

```
1. Một hóa đơn duy nhất — không cần quản lý payment method riêng
2. Volume discount tự động áp dụng cho tổng usage
3. Reserved Instances / Savings Plans chia sẻ giữa accounts
4. AWS Free Tier độc lập cho mỗi account (không chia sẻ)
5. Xem và phân tích chi phí tập trung qua Cost Explorer
```

### Chu Kỳ Thanh Toán

```
Hàng tháng:
1. AWS tổng hợp usage của tất cả member accounts
2. Tính discount dựa trên tổng usage
3. Phát hành hóa đơn hợp nhất cho Management Account
4. Management Account có thể xem chi tiết từng account

Ngày cutoff: Ngày cuối tháng (UTC)
Hóa đơn: Khoảng ngày 3–5 tháng tiếp theo
```

---

## Volume Discount

### Nguyên Tắc Volume Discount

AWS áp dụng chiết khấu **lũy tiến** (tiered pricing) theo tổng lượng usage — sử dụng nhiều hơn trả ít hơn trên mỗi đơn vị.

```
Ví dụ: S3 Standard Storage

Tier 1: 0 – 50 TB/tháng    → $0.023/GB
Tier 2: 50 – 500 TB/tháng  → $0.022/GB  (Tiết kiệm ~4%)
Tier 3: > 500 TB/tháng     → $0.021/GB  (Tiết kiệm ~9%)
```

### Ảnh Hưởng Của Consolidated Billing Đến Volume Discount

```
Không có Organizations (standalone accounts):
  Account A: 30 TB → Tier 1: $0.023/GB × 30,720 GB = $707
  Account B: 40 TB → Tier 1: $0.023/GB × 40,960 GB = $942
  Tổng: $1,649

Có Organizations (consolidated):
  Tổng usage: 70 TB → vào Tier 2
  Account A + B: 50 TB × $0.023 + 20 TB × $0.022
               = $1,177 + $450 = $1,627
  
  → Tiết kiệm: $1,649 - $1,627 = $22/tháng
    (% tiết kiệm tăng khi usage lớn hơn)
```

### Dịch Vụ Có Volume Discount Quan Trọng

```
S3: Storage tiered pricing
    Transfer accelerated requests
    
Data Transfer (DataTransfer Out):
    0–10 TB/tháng: $0.09/GB
    10–40 TB/tháng: $0.085/GB
    40–100 TB/tháng: $0.07/GB
    > 150 TB/tháng: $0.05/GB

CloudFront: HTTPS requests, data transfer out
DynamoDB: Read/Write capacity units
Kinesis Data Streams: PUT payload units
AWS Shield Advanced: Chia sẻ cost cho toàn Organization
```

---

## Reserved Instance Sharing

### Reserved Instance (RI — Phiên Bản Dự Trữ) Là Gì?

**Reserved Instance** là cam kết sử dụng EC2 (hoặc RDS, ElastiCache, Redshift...) trong 1–3 năm để đổi lấy giảm giá đáng kể (lên đến 72% so với On-Demand).

### RI Sharing Trong Organizations

```
Management Account mua:
  1x m5.large Reserved Instance, ap-southeast-1, Linux, 1 năm

Member Account A: Đang chạy m5.large instance on-demand
→ Tự động nhận RI discount, không cần làm gì thêm

Member Account B: Đang chạy m5.large instance on-demand
→ Nếu Account A không dùng hết RI, Account B cũng được áp dụng
```

### Điều Kiện RI Sharing

```
RI được chia sẻ khi:
✅ Instance type, region, OS khớp
✅ Tenancy khớp (shared vs dedicated)
✅ RI Sharing được enable (mặc định ON)

RI KHÔNG chia sẻ khi:
❌ Account tự opt-out khỏi RI sharing
❌ RI là loại "Zonal" nhưng accounts ở AZ khác
❌ Instance type/OS không khớp
```

### Bật/Tắt RI Sharing

```bash
# Xem trạng thái RI sharing
aws organizations describe-organization --query 'Organization.AvailablePolicyTypes'

# Từ Management Account — tắt RI sharing cho member account cụ thể
# (Thực hiện qua AWS Cost Management Console hoặc API)
aws cur put-report-definition  # Liên quan đến Cost and Usage Report

# Từ Member Account — tự opt-out
# Đăng nhập vào Billing Console → Preferences → RI sharing → OFF
```

### Ưu Tiên Áp Dụng RI

```
Account mua RI → Ưu tiên dùng RI cho chính account đó trước
  │ Còn dư RI capacity?
  ▼
Chia sẻ cho member accounts khác trong Organization (theo thứ tự ngẫu nhiên)
```

---

## Savings Plans Sharing

### Savings Plans (Kế Hoạch Tiết Kiệm) Là Gì?

**Savings Plans** là cam kết chi tiêu tối thiểu ($/giờ) trong 1–3 năm, linh hoạt hơn RI — áp dụng cho nhiều instance type, region và services.

```
3 loại Savings Plans:
1. Compute Savings Plans — Linh hoạt nhất
   Áp dụng: EC2 (mọi type), Fargate, Lambda
   Giảm: Lên đến 66%

2. EC2 Instance Savings Plans — Ít linh hoạt, giảm nhiều hơn
   Áp dụng: EC2 trong một region + instance family cụ thể
   Giảm: Lên đến 72%

3. SageMaker Savings Plans
   Áp dụng: SageMaker instances
   Giảm: Lên đến 64%
```

### Savings Plans Sharing Trong Organizations

```
Hoạt động tương tự RI sharing:
1. Bất kỳ account nào trong Organization mua Savings Plans
2. Savings Plans tự động áp dụng cho usage phù hợp
   trong toàn bộ Organization
3. Account mua được ưu tiên dùng trước
4. Phần còn lại chia sẻ cho accounts khác

Ưu điểm so với RI:
- Không cần biết trước instance type
- Tự động áp dụng khi scale up/down
- Compute SP áp dụng cho cả Fargate và Lambda
```

### Disable Savings Plans Sharing

```bash
# Từ Management Account — disable sharing
aws savingsplans describe-savings-plans \
  --filters '[{"name": "state", "values": ["active"]}]'

# Opt-out account cụ thể (qua Billing Console)
# AWS Console → Billing → Savings Plans → Settings
# → Disable "Savings Plans sharing across my AWS organization"
```

---

## Spot Instance Và Free Tier

### AWS Free Tier Trong Organizations

```
⚠️ Free Tier KHÔNG chia sẻ giữa accounts

Mỗi account có Free Tier độc lập:
- Account A: 750 giờ EC2 t2.micro/tháng
- Account B: 750 giờ EC2 t2.micro/tháng  ← Riêng biệt

→ Lợi thế: Sandbox accounts vẫn có Free Tier đầy đủ
→ Nhược điểm: Cần theo dõi Free Tier usage từng account riêng
```

### Spot Instance Và Consolidated Billing

Spot Instances không có cơ chế sharing đặc biệt như RI hay Savings Plans — mỗi account tự đặt Spot bids và thanh toán riêng. Consolidated Billing chỉ tổng hợp chi phí.

---

## Cost Allocation Tags — Phân Bổ Chi Phí Bằng Tag

### Tag Policy Trong Organizations

```bash
# Tạo Tag Policy để chuẩn hóa tags
cat > tag-policy.json << 'EOF'
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["Production", "Staging", "Development", "Sandbox"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "s3:bucket", "rds:db"]
      }
    },
    "CostCenter": {
      "tag_key": {
        "@@assign": "CostCenter"
      }
    }
  }
}
EOF

aws organizations create-policy \
  --name "RequiredTags" \
  --type TAG_POLICY \
  --content file://tag-policy.json

aws organizations attach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id r-xxxx  # Gắn vào Root cho toàn Organization
```

### Kích Hoạt Cost Allocation Tags

```bash
# Kích hoạt user-defined tags để dùng trong Cost Explorer
# (Từ Management Account, Billing Console)
# Billing → Cost allocation tags → Activate

# Tags phổ biến cần activate:
# - Environment
# - Team
# - Project
# - CostCenter
# - Application
```

### Phân Tích Chi Phí Theo Tag

```
Cost Explorer filter by tag:
→ Xem tổng chi phí theo Environment (Production vs Dev)
→ Xem chi phí từng Team
→ Xem chi phí từng Project
→ Tạo budget alert cho CostCenter cụ thể
```

---

## Billing Dashboard Và Báo Cáo

### Cost and Usage Report (CUR — Báo Cáo Chi Phí Và Sử Dụng)

**CUR** là báo cáo chi tiết nhất của AWS — tích hợp với Athena, S3, QuickSight để phân tích chuyên sâu.

```bash
# Tạo CUR (Cost and Usage Report)
aws cur put-report-definition \
  --report-definition '{
    "ReportName": "organization-cur",
    "TimeUnit": "HOURLY",
    "Format": "Parquet",
    "Compression": "Parquet",
    "AdditionalSchemaElements": ["RESOURCES"],
    "S3Bucket": "my-cur-bucket",
    "S3Prefix": "cur/",
    "S3Region": "us-east-1",
    "RefreshClosedReports": true,
    "ReportVersioning": "OVERWRITE_REPORT"
  }'
```

### Cost Explorer (Khám Phá Chi Phí) Trong Organizations

```
Từ Management Account, Cost Explorer có thể:
├── Xem chi phí của toàn Organization
├── Breakdown theo từng member account
├── Filter theo service, region, tag, account
├── Forecast (Dự Báo) chi phí 12 tháng tới
├── Rightsizing Recommendations — gợi ý optimize instance type
└── Savings Plans Recommendations — gợi ý cam kết tiết kiệm

Từ Member Account:
└── Chỉ xem chi phí của account đó
    (trừ khi Management Account bật "Member accounts access to Billing")
```

### Bật Quyền Xem Billing Cho Member Accounts

```bash
# Từ Management Account
# AWS Console → Account → IAM user and role access to Billing information
# → Activate IAM access

# Hoặc qua CLI
aws iam create-account-alias --account-alias my-org-management

# Sau đó IAM user trong member account có thể xem
# billing của account họ (không phải toàn Organization)
```

---

## Quản Lý Chi Phí Đa Account

### Chiến Lược Budget Theo Cấp

```
Level 1 — Organization Budget (Management Account):
  $50,000/tháng cho toàn Organization
  Alert: 80%, 100%

Level 2 — Account Budget (từng account):
  app-prod: $15,000/tháng
  app-dev: $5,000/tháng
  Alert: 80%, 100%

Level 3 — Tag-based Budget:
  CostCenter CC-001: $10,000/tháng
  Project ProjectX: $3,000/tháng
```

### AWS Budgets Actions (Hành Động Budget Tự Động)

```bash
# Tạo budget với action tự động
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "app-prod-monthly",
    "BudgetLimit": {"Amount": "15000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{
      "SubscriptionType": "EMAIL",
      "Address": "ops-team@company.com"
    }]
  }]'
```

### AWS Cost Anomaly Detection (Phát Hiện Chi Phí Bất Thường)

```bash
# Tạo cost monitor cho toàn Organization
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "OrganizationMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Tạo subscription alert
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "DailyAnomalyAlert",
    "MonitorArnList": ["arn:aws:ce::123456789012:anomalymonitor/xxxx"],
    "Subscribers": [{
      "Address": "billing@company.com",
      "Type": "EMAIL"
    }],
    "Threshold": 100,
    "Frequency": "DAILY"
  }'
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Consolidated Billing mang lại lợi ích gì ngoài một hóa đơn duy nhất?**
> Ba lợi ích chính: (1) **Volume discount** — tổng usage của tất cả accounts được tính chung, đạt tier cao hơn và nhận chiết khấu cao hơn; (2) **RI/Savings Plans sharing** — Reserved Instance hoặc Savings Plans trong một account có thể áp dụng cho accounts khác trong Organization; (3) **Quản lý chi phí tập trung** — Cost Explorer và Budgets có thể xem toàn Organization.

**Q: AWS Free Tier có chia sẻ giữa các accounts trong Organization không?**
> **Không.** Mỗi account có Free Tier riêng hoàn toàn độc lập. Account A dùng hết Free Tier không ảnh hưởng Account B. Đây là lợi thế — sandbox accounts vẫn có Free Tier đầy đủ cho testing.

**Q: Nếu Account A mua Reserved Instance nhưng không dùng hết, Account B có được hưởng không?**
> **Có**, tự động. Khi Account A không dùng hết RI capacity, AWS tự động áp dụng RI discount cho instance phù hợp trong Organization (cùng type, region, OS). Account A vẫn được ưu tiên trước — phần dư mới chia sẻ. Tính năng này có thể tắt (opt-out) từng account.

### Nâng Cao

**Q: Khi nào nên mua RI ở Management Account vs Member Account?**
> Nếu usage ổn định **ở tất cả accounts** (không tăng giảm nhiều theo account cụ thể), nên mua RI ở Management Account hoặc account có usage cao nhất — RI sẽ tự động share cho accounts khác khi cần. Nếu một account cụ thể có baseline rất rõ ràng (ví dụ: database server luôn chạy 24/7), mua RI ở chính account đó để ownership rõ ràng hơn trong cost reporting.

**Q: Compute Savings Plans và EC2 Instance Savings Plans khác nhau thế nào, và trong Organizations nên dùng loại nào?**
> **EC2 Instance SP** — cam kết instance family + region cụ thể, giảm nhiều hơn (72%), nhưng cứng nhắc. **Compute SP** — linh hoạt hơn (mọi EC2 type, Fargate, Lambda), giảm ít hơn (66%), nhưng phù hợp Organizations vì usage mix thay đổi nhiều. Trong Organizations với nhiều team/workload đa dạng → **Compute Savings Plans** là lựa chọn an toàn hơn vì tự động áp dụng cho mọi instance type phù hợp.

---

## 🔗 Điều Hướng

| Trước | File Này | Tiếp Theo |
|-------|---------|-----------|
| [2-scp-policies.md](./2-scp-policies.md) | **3-consolidated-billing.md** | [4-delegated-admin.md](./4-delegated-admin.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
