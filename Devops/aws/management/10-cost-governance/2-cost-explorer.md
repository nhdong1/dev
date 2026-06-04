# AWS Cost Explorer — Phân Tích & Dự Báo Chi Phí

> **AWS Cost Explorer** (Khám Phá Chi Phí) là công cụ trực quan hóa, phân tích và dự báo chi phí AWS. Cho phép drill-down theo service, account, tag, region và nhận **Rightsizing Recommendations** (Khuyến Nghị Điều Chỉnh Kích Thước Instance) để giảm chi phí.

---

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [Giao Diện & Các View Chính](#giao-diện--các-view-chính)
3. [Filtering & Grouping](#filtering--grouping)
4. [Forecasting — Dự Báo Chi Phí](#forecasting--dự-báo-chi-phí)
5. [Rightsizing Recommendations](#rightsizing-recommendations)
6. [Savings Plans Recommendations](#savings-plans-recommendations)
7. [Cost Explorer API](#cost-explorer-api)
8. [CUR — Cost and Usage Report](#cur--cost-and-usage-report)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🔑 Khái Niệm Cốt Lõi

### Cost Explorer là gì?

**Cost Explorer** cung cấp:
- **Phân tích lịch sử** — 12 tháng dữ liệu chi phí chi tiết
- **Granularity** — Daily (hàng ngày) hoặc Monthly (hàng tháng)
- **Forecasting** — Dự báo 12 tháng tương lai bằng ML
- **Filtering** — Lọc theo service, account, tag, region, purchase type
- **Recommendations** — Rightsizing và Savings Plans suggestions

### Cost Explorer vs AWS Budgets vs CUR

```
Cost Explorer:
  → "Tháng vừa rồi tôi chi bao nhiêu cho EC2? Trend như thế nào?"
  → Phân tích sau sự kiện (reactive analysis)
  → Interactive UI + API

AWS Budgets:
  → "Cảnh báo khi tôi sắp vượt $1,000 cho EC2"
  → Kiểm soát trước sự kiện (proactive control)

CUR — Cost and Usage Report (Báo Cáo Chi Phí & Sử Dụng):
  → Raw billing data, granularity đến giờ (hourly)
  → Export ra S3 → Athena → QuickSight cho custom analytics
  → Level detail cao nhất, dùng cho BI/finance team
```

---

## 🖥️ Giao Diện & Các View Chính

### Pre-Built Views (View Có Sẵn)

**AWS Cost Explorer** có các báo cáo mẫu sẵn:

| View                                | Mô Tả                                                   |
| ----------------------------------- | ------------------------------------------------------- |
| **Monthly costs by service**        | Chi phí theo tháng, phân tách theo từng service         |
| **Monthly costs by account**        | Chi phí theo tháng, phân tách theo account (multi-acct) |
| **Monthly EC2 running hours**       | Số giờ EC2 chạy — phát hiện waste                       |
| **Daily costs**                     | Chi phí hàng ngày — phát hiện anomaly thủ công          |
| **RI Utilization report**           | Tỷ lệ sử dụng Reserved Instances                        |
| **RI Coverage report**              | % usage được che bởi RIs                                |
| **Savings Plans Utilization**       | Tỷ lệ sử dụng Savings Plans đã mua                     |
| **Savings Plans Coverage**          | % usage được che bởi Savings Plans                      |

### Granularity Options (Độ Chi Tiết)

```
Monthly:  Tổng chi phí mỗi tháng — dùng cho trend analysis
Daily:    Chi phí mỗi ngày — dùng để phát hiện spike
Hourly:   Chỉ có qua API (không có trong Console UI)
          Cần tích hợp CUR để có hourly data trực quan
```

---

## 🔍 Filtering & Grouping

### Dimension Filters (Bộ Lọc Theo Chiều)

```
Filters có thể kết hợp:

Service          → EC2, RDS, S3, Lambda, CloudFront...
Linked Account   → Account ID cụ thể (trong org)
Region           → us-east-1, ap-southeast-1...
Usage Type       → BoxUsage:t3.medium, DataTransfer-Out...
Purchase Option  → On-Demand, Spot, Reserved, Savings Plans
Instance Type    → t3.medium, m5.large, r5.xlarge...
Tag              → team=backend, env=prod, project=apollo
Availability Zone→ us-east-1a, us-east-1b...
API Operation    → RunInstances, GetObject, Invoke...
```

### Group By (Nhóm Theo)

```
Group By cho phép phân tách chi phí theo một dimension:

Group by Service:
  EC2:      $3,200
  RDS:      $1,800
  S3:       $420
  Lambda:   $85
  Other:    $320

Group by Tag (team):
  backend:  $2,500
  frontend: $1,200
  data:     $2,125
  (untagged): $--

Group by Account:
  prod-account:    $4,500
  staging-account: $800
  dev-account:     $525
```

> **Lưu ý:** Chỉ group được theo **một dimension** tại một thời điểm trong Console. Dùng API để query multi-dimension.

### Saved Filters — Lưu Bộ Lọc

Lưu các filter thường dùng để tái sử dụng:
```
Saved Filter: "Backend Team Production EC2"
  Filter: service=EC2, account=prod-account, tag:team=backend
  → Mở nhanh mỗi khi cần review
```

---

## 📈 Forecasting — Dự Báo Chi Phí

### Cách Hoạt Động

Cost Explorer dùng **ML (Machine Learning)** phân tích:
- Pattern chi phí 12 tháng lịch sử
- Seasonality (tính mùa vụ)
- Trend (xu hướng)

Kết quả: **Forecasted cost** với **confidence interval** (khoảng tin cậy — 80% CI).

```
Ví dụ forecast:
  Tháng này (hết 15 ngày): $1,200 actual
  
  Forecast cuối tháng:
    Lower bound: $2,200
    Expected:    $2,450
    Upper bound: $2,700
    
  → Alert Budget khi Forecasted > $3,000 threshold vẫn OK
  → Alert Budget khi Forecasted > $2,500 → đang cận ngưỡng
```

### Forecast Accuracy (Độ Chính Xác Dự Báo)

- Chính xác hơn khi có **nhiều dữ liệu lịch sử**
- Kém chính xác với **workload không ổn định** hoặc **mới dùng < 3 tháng**
- **Forecasted Budget Alerts** tốt nhất khi dùng sau 6+ tháng hoạt động

---

## 💡 Rightsizing Recommendations

### Rightsizing là gì?

**Rightsizing** (Điều Chỉnh Kích Thước) là quá trình phân tích EC2 instances đang chạy và đề xuất:
- **Downsize** — Chuyển sang instance type nhỏ hơn nếu underutilized (sử dụng dưới mức)
- **Terminate** — Tắt instance không còn cần thiết

### Cách Cost Explorer Phân Tích

```
Cost Explorer xem xét (14 ngày gần nhất):
  • CPU utilization (max, avg)
  • Memory utilization (nếu cài CloudWatch Agent)
  • Network I/O
  • Storage I/O

Đề xuất:
  Instance hiện tại: m5.2xlarge ($0.384/hr)
    CPU avg: 8%
    Memory avg: 15%
    
  Đề xuất: m5.large ($0.096/hr)
    Tiết kiệm ước tính: $209/tháng (75%)
    Rủi ro: LOW (headroom đủ cho spike)
```

### Cách Kích Hoạt

1. Vào Cost Explorer → **Rightsizing recommendations**
2. Bật **CloudWatch Agent** để có memory metrics (quan trọng!)
3. Chọn **Consider all recommendations** (bao gồm cross-family changes)
4. Xem xét **Savings Opportunity** và **Risk Level**

> **Lưu ý:** Rightsizing không tự động thay đổi instance. Đây chỉ là **recommendation** (khuyến nghị) — cần người review và thực hiện thủ công hoặc qua SSM Automation.

### Risk Levels (Mức Độ Rủi Ro)

| Risk Level | Ý Nghĩa                                                  |
| ---------- | -------------------------------------------------------- |
| **Low**    | Instance rõ ràng oversized — an toàn để downsize         |
| **Medium** | Có một số spike nhưng instance vẫn có thể nhỏ hơn       |
| **High**   | Sử dụng khá nhiều — nên test kỹ trước khi downsize       |

---

## 💰 Savings Plans Recommendations

Cost Explorer phân tích usage và đề xuất loại và số tiền **Savings Plans** phù hợp.

```
Phân tích:
  EC2 On-Demand spend 30 ngày qua: $4,500/tháng
  Stable baseline (ổn định): ~$3,200/tháng
  
  Recommendation:
    Compute Savings Plans: $4.38/hr commitment
    Term: 1 năm, No Upfront
    
    Estimated savings: $856/tháng (19%)
    Payback period: 0 tháng (giảm ngay từ tháng đầu)
```

### Cách Đọc Recommendations

```
Savings Plans Recommendations page:
  
  ┌─────────────────────────────────────────────────────┐
  │ Recommended Commitment: $4.38/hour                  │
  │ Estimated Monthly Savings: $856 (19%)               │
  │                                                     │
  │ Based on: Last 30 days usage                        │
  │ Term: 1 year   Payment: No Upfront                  │
  │                                                     │
  │ Coverage if purchased: 87%                          │
  │ Remaining On-Demand: 13%                            │
  └─────────────────────────────────────────────────────┘
```

---

## 🔌 Cost Explorer API

### Khi Nào Dùng API

- **Tự động hóa báo cáo** — Cron job lấy chi phí → gửi Slack report
- **Custom dashboards** — Tích hợp vào internal tool
- **FinOps automation** — Trigger action dựa trên cost threshold

### GetCostAndUsage — Lệnh Phổ Biến Nhất

```python
import boto3

client = boto3.client('ce', region_name='us-east-1')

response = client.get_cost_and_usage(
    TimePeriod={
        'Start': '2026-04-01',
        'End': '2026-05-01'
    },
    Granularity='MONTHLY',
    Filter={
        'Tags': {
            'Key': 'team',
            'Values': ['backend'],
            'MatchOptions': ['EQUALS']
        }
    },
    GroupBy=[
        {
            'Type': 'DIMENSION',
            'Key': 'SERVICE'
        }
    ],
    Metrics=['BlendedCost', 'UsageQuantity']
)

for group in response['ResultsByTime'][0]['Groups']:
    service = group['Keys'][0]
    cost = group['Metrics']['BlendedCost']['Amount']
    print(f"{service}: ${float(cost):.2f}")
```

### Chi Phí API

```
$0.01 per API request (đắt hơn nhiều dịch vụ khác!)

Best practices:
  → Cache kết quả, không query real-time liên tục
  → Dùng CUR + Athena cho analytics phức tạp (rẻ hơn)
  → API phù hợp cho: alert, summary report, daily digest
```

---

## 📋 CUR — Cost and Usage Report

### CUR là gì?

**CUR — Cost and Usage Report** (Báo Cáo Chi Phí và Sử Dụng) là **nguồn dữ liệu chi tiết nhất** của AWS billing, xuất ra S3.

```
CUR vs Cost Explorer:
  CUR:           Raw data, hourly granularity, mọi dimension
  Cost Explorer: Aggregated, daily/monthly, UI + API

CUR dùng cho:
  → Finance team cần raw billing data
  → Tích hợp với BI tools (QuickSight, Tableau, Looker)
  → Custom cost allocation logic phức tạp
  → Audit chi phí chi tiết theo giờ
```

### Thiết Lập CUR

```
1. Billing Console → Cost & Usage Reports → Create report
2. Cấu hình:
   Report name: company-cur-hourly
   Include resource IDs: YES (quan trọng để filter theo resource)
   Data integration: Athena (tự tạo Glue catalog)
   S3 bucket: s3://company-cur-data/
   Compression: Parquet (tối ưu cho Athena query)
   
3. Query với Athena:
   SELECT line_item_product_code,
          line_item_resource_id,
          SUM(line_item_blended_cost) as total_cost
   FROM cur_database.cur_table
   WHERE line_item_usage_start_date >= DATE '2026-04-01'
     AND resource_tags_user_team = 'backend'
   GROUP BY 1, 2
   ORDER BY 3 DESC
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Cost Explorer và CUR khác nhau thế nào?**
> **Cost Explorer** — Công cụ phân tích trực quan với API, granularity tối đa là daily, phù hợp cho analysis nhanh và automated reporting. **CUR** — Raw billing data xuất ra S3, granularity hourly, chứa mọi field chi tiết, dùng cho custom BI analytics với Athena/QuickSight. Cả hai đều bắt nguồn từ cùng billing data nhưng CUR chi tiết hơn nhiều.

**Q: Rightsizing Recommendations cần điều kiện gì để chính xác?**
> Cần bật **CloudWatch Agent** để thu thập **memory metrics** — nếu chỉ có CPU thì không đủ để đánh giá. Cost Explorer dùng 14 ngày dữ liệu gần nhất. Memory metrics không được thu thập mặc định — phải cài CW Agent và cấu hình custom namespace.

**Q: Tại sao Cost Explorer API đắt và nên dùng thế nào?**
> $0.01/request, không hỗ trợ hourly granularity. Nên **cache kết quả** (lưu vào DynamoDB hoặc S3), chạy batch job hàng ngày thay vì real-time, và dùng **CUR + Athena** cho analytics phức tạp (rẻ hơn nhiều với large dataset).

**Q: Làm thế nào phân tích "untagged cost" — chi phí không có tag?**
> Trong Cost Explorer, dùng filter **Tag: key exists = false** hoặc group by Tag rồi tìm group có tên trống. Hoặc dùng CUR query: `WHERE resource_tags_user_team IS NULL`. Sau đó dùng Config rule `required-tags` để enforce tagging going forward.

---

**Điều Hướng:**
← [1-budgets-alerts.md](1-budgets-alerts.md) | → [3-anomaly-detection.md](3-anomaly-detection.md)

**Cập Nhật Lần Cuối:** 2026-05-17
