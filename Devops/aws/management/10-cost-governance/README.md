# AWS Cost Governance — Quản Trị Chi Phí & FinOps

> **Cost Governance** (Quản Trị Chi Phí) là tập hợp công cụ, quy trình và chiến lược giúp doanh nghiệp kiểm soát, tối ưu hóa và dự báo chi phí trên AWS. Gắn liền với triết lý **FinOps** — Financial Operations (Vận Hành Tài Chính) — đưa trách nhiệm tài chính vào từng team kỹ thuật.

---

## 📚 Mục Lục Module

| File | Chủ Đề |
|------|--------|
| [1-budgets-alerts.md](1-budgets-alerts.md) | AWS Budgets — ngân sách, cảnh báo, hành động tự động |
| [2-cost-explorer.md](2-cost-explorer.md) | Cost Explorer — phân tích, lọc, dự báo, rightsizing |
| [3-anomaly-detection.md](3-anomaly-detection.md) | Cost Anomaly Detection — phát hiện chi tiêu đột biến với ML |
| [4-tagging-strategy.md](4-tagging-strategy.md) | Tag policies — chiến lược gắn nhãn và phân bổ chi phí |
| [5-savings-plans-ri.md](5-savings-plans-ri.md) | Savings Plans vs Reserved Instances — cam kết tiết kiệm |

---

## 🎯 Tại Sao Cost Governance Quan Trọng?

### Vấn Đề Thực Tế

```
Không có Cost Governance:
  Engineer deploy EC2 r6g.4xlarge cho test → quên tắt
  → $800/tháng mà không ai biết
  → Phát hiện sau 3 tháng khi xem hóa đơn: $2,400 đã mất

Có Cost Governance:
  AWS Budget cảnh báo khi EC2 cost vượt $200
  → Nhận alert qua email/Slack ngay ngày đầu
  → Tắt instance, tiết kiệm $2,200
```

### Ba Trụ Cột của FinOps

```
┌─────────────────────────────────────────────────┐
│                  FinOps Lifecycle               │
│                                                 │
│  INFORM          OPTIMIZE         OPERATE       │
│  (Thông Tin)     (Tối Ưu)        (Vận Hành)    │
│                                                 │
│  • Cost Explorer • Savings Plans  • Budgets     │
│  • Tags          • Reserved Ins.  • Alerts      │
│  • Reports       • Rightsizing    • Anomaly Det │
│  • Dashboards    • Spot Instances • Governance  │
└─────────────────────────────────────────────────┘
```

---

## 🗺️ Bản Đồ Công Cụ Cost Governance

### AWS Native Tools

```
                    ┌─────────────────┐
                    │  AWS Cost & Usage│
                    │  Report (CUR)   │◄── Raw data, S3, Athena
                    └────────┬────────┘
                             │ feeds
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌──────▼──────┐   ┌───────▼──────┐  ┌───────▼──────┐
   │   AWS Cost  │   │  AWS Budgets │  │  Cost Anomaly│
   │  Explorer   │   │  & Actions   │  │  Detection   │
   │ (Phân tích) │   │  (Giám sát)  │  │  (ML Alert)  │
   └──────┬──────┘   └───────┬──────┘  └───────┬──────┘
          │                  │                  │
          └──────────────────▼──────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Tag Strategy + │
                    │  Cost Alloc.    │◄── Phân bổ chi phí
                    │  Tags           │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Savings Plans  │
                    │  + Reserved Ins │◄── Giảm giá cam kết
                    └─────────────────┘
```

---

## 📊 Tổng Quan Các Dịch Vụ

### 1. AWS Budgets (Ngân Sách AWS)

**Mục đích:** Đặt ngưỡng chi phí và nhận cảnh báo chủ động.

**4 loại budget:**
- **Cost Budget** — Ngân sách theo tổng chi phí ($)
- **Usage Budget** — Ngân sách theo lượng sử dụng (GB, giờ, requests)
- **Reservation Budget** — Theo dõi utilization/coverage của RI
- **Savings Plans Budget** — Theo dõi utilization/coverage của Savings Plans

**Hành động tự động (Budget Actions):**
- Gắn IAM policy hạn chế quyền tạo tài nguyên
- Nhận email/SNS alert
- Kích hoạt SSM Automation runbook

---

### 2. AWS Cost Explorer (Khám Phá Chi Phí)

**Mục đích:** Phân tích, visualize và dự báo chi phí.

**Khả năng chính:**
- Lọc theo service, account, region, tag, usage type
- Dự báo (forecast) 12 tháng tới
- **Rightsizing Recommendations** — Gợi ý downsize EC2 đang dư thừa
- Xem chi phí theo **hourly granularity** (độ chi tiết theo giờ)

---

### 3. Cost Anomaly Detection (Phát Hiện Chi Phí Bất Thường)

**Mục đích:** Dùng ML để phát hiện chi tiêu đột biến bất thường.

**Cách hoạt động:**
- Học pattern chi tiêu lịch sử của từng service/account/tag
- So sánh chi tiêu hiện tại với baseline dự đoán
- Gửi alert khi phát hiện anomaly vượt ngưỡng

**Monitor types:**
- AWS Service Monitor — Theo service (EC2, S3...)
- Linked Account Monitor — Theo member account
- Cost Category Monitor — Theo cost category
- Tag Monitor — Theo tag key/value

---

### 4. Resource Tagging Strategy (Chiến Lược Gắn Nhãn Tài Nguyên)

**Mục đích:** Phân bổ chi phí chính xác theo team/project/environment.

**Các loại tag quan trọng:**
- **Cost Allocation Tags** — Được kích hoạt → hiển thị trong Cost Explorer
- **Tag Policies** (via Organizations) — Chuẩn hóa tên và giá trị tag
- **AWS Config Rules** — Phát hiện tài nguyên thiếu tag bắt buộc

**Tag chuẩn phổ biến:**
```
Environment: prod | staging | dev | test
Team:        platform | backend | frontend | data
Project:     user-auth | payment | analytics
CostCenter:  CC-1001 | CC-1002
Owner:       john.doe@company.com
```

---

### 5. Savings Plans & Reserved Instances (Cam Kết Tiết Kiệm)

**Savings Plans** (Kế Hoạch Tiết Kiệm):
- Cam kết chi tiêu theo $/giờ trong 1 hoặc 3 năm
- **Compute Savings Plans** — Linh hoạt nhất: áp dụng EC2, Fargate, Lambda
- **EC2 Instance Savings Plans** — Giảm nhiều hơn, ít linh hoạt hơn
- **SageMaker Savings Plans** — Dành riêng cho ML workloads

**Reserved Instances — RI** (Phiên Bản Đặt Trước):
- Cam kết instance type/region trong 1 hoặc 3 năm
- Giảm đến 72% so với On-Demand
- Có thể **bán lại** trên Reserved Instance Marketplace nếu không dùng hết

---

## 🏗️ Kiến Trúc Cost Governance Đa Account

```
┌─────────────────────────────────────────────────────┐
│               Management Account                    │
│                                                     │
│  ┌─────────────────┐    ┌───────────────────────┐  │
│  │   AWS Budgets   │    │   Cost Anomaly Det.   │  │
│  │  (Organization- │    │  (Organization-level) │  │
│  │   level budget) │    └───────────────────────┘  │
│  └─────────────────┘                               │
│  ┌─────────────────────────────────────────────┐   │
│  │          Cost Explorer (aggregated)          │   │
│  │   Filtered by: Account | OU | Tag | Service  │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │    Cost & Usage Report → S3 → Athena         │   │
│  │    → QuickSight Dashboard (FinOps Board)     │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
         ↑ Consolidated Billing từ Member Accounts
┌────────┬────────┬────────┬────────┬────────────────┐
│  Dev   │  Stg  │  Prod  │Security│   Shared Svcs  │
│Account │Account│Account │Account │    Account     │
└────────┴────────┴────────┴────────┴────────────────┘
```

---

## 🎯 FinOps Maturity Model (Mô Hình Trưởng Thành FinOps)

### Giai Đoạn 1: Crawl (Bắt Đầu)

- [ ] Bật Cost & Usage Report
- [ ] Kích hoạt Cost Allocation Tags cơ bản
- [ ] Tạo Budget alert cho toàn organization
- [ ] Xem Cost Explorer hàng tuần

### Giai Đoạn 2: Walk (Phát Triển)

- [ ] Tag Policy via Organizations
- [ ] Budget theo team/project
- [ ] Cost Anomaly Detection alerts
- [ ] Rightsizing review hàng tháng

### Giai Đoạn 3: Run (Trưởng Thành)

- [ ] Automated cost allocation reports
- [ ] Chargeback/Showback cho từng team
- [ ] Savings Plans coverage > 70%
- [ ] FinOps dashboard real-time (QuickSight)
- [ ] Unit economics tracking (cost per API call, cost per user)

---

## 🔑 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: "Bạn sẽ phát hiện chi phí AWS tăng đột ngột như thế nào?"

**Trả lời chuẩn:**
> "Tôi sẽ dùng kết hợp: Cost Anomaly Detection để phát hiện tự động qua ML, AWS Budgets để alert khi vượt ngưỡng đã định, và Cost Explorer để drill-down xem service/account nào tăng. Nếu có Organization, tôi bật Anomaly Monitor ở cả organization-level và linked-account level."

### Câu 2: "Savings Plans vs Reserved Instances — khác nhau gì?"

**Trả lời chuẩn:**
> "Savings Plans linh hoạt hơn: cam kết theo $/giờ, tự động áp dụng cho EC2, Fargate, Lambda không phân biệt region/instance family. Reserved Instances cam kết instance type cụ thể, giảm giá nhiều hơn nhưng ít linh hoạt hơn. Trong môi trường containerized hiện đại, Compute Savings Plans thường phù hợp hơn vì workload hay thay đổi instance type."

### Câu 3: "Làm thế nào enforce tagging policy cho toàn organization?"

**Trả lời chuẩn:**
> "Ba lớp phòng thủ: (1) Tag Policies via Organizations chuẩn hóa tên/giá trị tag; (2) AWS Config Rule `required-tags` phát hiện resource thiếu tag và báo non-compliant; (3) SCP chặn tạo resource nếu không có mandatory tags — đây là cách enforce mạnh nhất nhưng cần cẩn thận vì có thể block cả automation."

---

## 📋 Quick Reference

| Công Cụ | Dùng Khi | Free? |
|---------|----------|-------|
| Cost Explorer | Phân tích chi phí lịch sử | ✅ Miễn phí (hourly có phí) |
| AWS Budgets | Đặt ngưỡng & alert | ✅ 2 budgets đầu miễn phí |
| Cost Anomaly Detection | Phát hiện spike tự động | ✅ Miễn phí |
| Cost & Usage Report | Data chi tiết nhất | ✅ S3 storage có phí |
| Savings Plans | Giảm chi phí compute | N/A (cam kết) |
| Reserved Instances | Giảm chi phí instance cố định | N/A (cam kết) |

---

## 🔗 Điều Hướng

- **Trước đó:** [09-health-dashboard/](../09-health-dashboard/README.md)
- **Tiếp theo:** [11-interview-prep/](../11-interview-prep/README.md)
- **Xem thêm:** [06-organizations/3-consolidated-billing.md](../06-organizations/3-consolidated-billing.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
