# RDS Pricing Models & Reserved Instances — Mô Hình Định Giá RDS

> Hiểu đầy đủ cấu trúc giá của Amazon RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ) để đưa ra quyết định mua sắm và thiết kế hạ tầng tiết kiệm chi phí nhất.

## 📚 Mục Lục

1. [Cấu Trúc Giá RDS](#cấu-trúc-giá-rds)
2. [On-Demand vs Reserved Instances](#on-demand-vs-reserved-instances)
3. [Storage Pricing — Giá Lưu Trữ](#storage-pricing)
4. [Backup & Snapshot Pricing — Giá Sao Lưu](#backup--snapshot-pricing)
5. [Data Transfer Pricing — Giá Truyền Dữ Liệu](#data-transfer-pricing)
6. [Reserved Instances Strategy — Chiến Lược Đặt Trước](#reserved-instances-strategy)
7. [Multi-AZ Cost Considerations — Chi Phí Multi-AZ](#multi-az-cost-considerations)
8. [Practical Examples — Ví Dụ Thực Tế](#practical-examples)

---

## Cấu Trúc Giá RDS

Chi phí RDS bao gồm nhiều thành phần — hiểu rõ từng phần giúp kiểm soát chi phí hiệu quả:

```
Tổng Chi Phí RDS = Instance Hours
                 + Storage (GB-month)
                 + I/O Requests (chỉ với magnetic, không áp dụng cho gp3/io1)
                 + Backup Storage vượt mức
                 + Data Transfer
                 + Tùy chọn: Multi-AZ, Read Replicas, RDS Proxy, Enhanced Monitoring
```

### Thành Phần Chi Phí Chi Tiết

| Thành Phần                            | Tính Theo         | Ghi Chú                                          |
| ------------------------------------- | ----------------- | ------------------------------------------------ |
| DB Instance                           | Giờ chạy          | Khác nhau theo engine, instance type, region      |
| Storage — gp2/gp3                     | GB-month          | SSD thông dụng, gp3 rẻ hơn gp2 ~20%             |
| Storage — io1/io2                     | GB-month + IOPS   | SSD hiệu năng cao, IOPS tính riêng               |
| Storage — magnetic                    | GB-month + I/O    | Ít dùng, rẻ nhất nhưng chậm                      |
| Automated Backup (Sao Lưu Tự Động)   | GB-month          | Miễn phí đến = database size; tính phí vượt mức |
| Manual Snapshot (Ảnh Chụp Thủ Công)  | GB-month          | Tính phí toàn bộ                                 |
| Data Transfer Out (Truyền Ra Ngoài)  | GB                | Trong cùng AZ miễn phí; ra internet tính phí    |
| Multi-AZ Standby                      | Nhân đôi instance | Không có thêm phí storage                        |
| Read Replica                          | Instance + Storage| Tính riêng như instance độc lập                  |

---

## On-Demand vs Reserved Instances

### On-Demand Instances (Máy Chủ Theo Yêu Cầu)

```
Đặc điểm:
- Trả theo giờ sử dụng thực tế
- Không cam kết, linh hoạt tối đa
- Giá cao nhất trong các options

Phù hợp cho:
- Môi trường dev/test ngắn hạn
- Workloads không dự đoán được
- Thử nghiệm trước khi commit Reserved
- Spiky traffic (Traffic đột biến theo mùa)
```

### Reserved Instances (Máy Chủ Đặt Trước)

RI — Reserved Instances cho phép cam kết dùng 1 hoặc 3 năm để đổi lấy mức giá thấp hơn đáng kể.

```
Hai loại term (kỳ hạn):
- 1 năm:  Tiết kiệm ~40% so với On-Demand
- 3 năm:  Tiết kiệm ~60-65% so với On-Demand

Ba payment options (tùy chọn thanh toán):
- No Upfront (Không Trả Trước):   Trả hàng tháng, tiết kiệm ít nhất
- Partial Upfront (Trả Một Phần): Trả một phần trước + hàng tháng
- All Upfront (Trả Toàn Bộ):     Trả hết trước, tiết kiệm nhiều nhất
```

### Bảng So Sánh Tiết Kiệm (Ví dụ db.r6g.large MySQL, us-east-1)

| Loại                           | Giá/Giờ  | Giá/Tháng  | Tiết Kiệm |
| ------------------------------ | -------- | ---------- | --------- |
| On-Demand                      | ~$0.240  | ~$175      | Chuẩn     |
| 1yr No Upfront RI              | ~$0.151  | ~$110      | ~37%      |
| 1yr All Upfront RI             | ~$0.138  | ~$101      | ~42%      |
| 3yr No Upfront RI              | ~$0.106  | ~$77       | ~56%      |
| 3yr All Upfront RI             | ~$0.095  | ~$70       | ~60%      |

> **Lưu ý:** Giá trên là ước tính minh họa, thay đổi theo thời gian và region. Luôn kiểm tra AWS Pricing Calculator trước khi quyết định.

### Quy Tắc Quyết Định Reserved Instances

```
Mua 1-năm RI khi:
✓ Instance đã chạy > 3 tháng liên tục
✓ Khả năng chạy ít nhất 12 tháng
✓ Engine và instance type ổn định (không có kế hoạch nâng cấp lớn)
✓ Workload đủ ổn định để dự đoán

Mua 3-năm RI khi:
✓ Production database cốt lõi, rất ít thay đổi architecture
✓ Đã chạy > 12 tháng không thay đổi instance type
✓ Team có kế hoạch dùng AWS ít nhất 3 năm

Không mua RI khi:
✗ Dev/test instances tắt thường xuyên
✗ Không chắc về scale hoặc architecture thay đổi
✗ Workload mới chưa có baseline usage
```

### Convertible Reserved Instances (RI Có Thể Chuyển Đổi)

```
Flexible RI option:
- Cho phép thay đổi instance family, OS, tenancy trong kỳ hạn
- Tiết kiệm ít hơn ~10-15% so với Standard RI
- Phù hợp khi có khả năng thay đổi instance type

Ví dụ: db.r6g.large → db.r7g.large trong kỳ hạn 3 năm
```

---

## Storage Pricing

### General Purpose SSD — gp2 và gp3

```
gp2 (cũ):
- Giá: ~$0.115/GB-month (us-east-1)
- IOPS: 3 IOPS/GB, tối đa 16,000 IOPS tự động
- Không tính phí IOPS riêng

gp3 (mới, khuyến nghị):
- Giá: ~$0.092/GB-month (us-east-1) — rẻ hơn gp2 ~20%
- IOPS baseline: 3,000 IOPS (đủ cho hầu hết workloads)
- IOPS thêm: $0.02/IOPS-month (cho 3,001-16,000 IOPS)
- Throughput thêm: $0.04/MiBps-month

→ Khuyến nghị: Migrate từ gp2 sang gp3 để tiết kiệm ~20% storage cost
```

### Provisioned IOPS SSD — io1 và io2

```
io1:
- Giá storage: ~$0.125/GB-month
- Giá IOPS:    $0.10/IOPS-month
- Dùng khi cần IOPS > 16,000 hoặc latency cực thấp nhất quán

io2 (Block Express):
- Giá storage: ~$0.125/GB-month
- IOPS tier 1 (1-32,000):   $0.10/IOPS-month
- IOPS tier 2 (32,001-64,000): $0.065/IOPS-month
- IOPS tier 3 (> 64,000):   $0.046/IOPS-month

Ví dụ tính toán io1:
100 GB storage + 10,000 IOPS = (100 × $0.125) + (10,000 × $0.10)
                              = $12.50 + $1,000 = $1,012.50/tháng
→ io1 rất đắt! Chỉ dùng khi thực sự cần IOPS cao
```

### Storage Auto Scaling — Tự Động Mở Rộng Lưu Trữ

```
Cấu hình Storage Auto Scaling:
- Không tính phí riêng cho tính năng này
- Chỉ trả cho storage thực sự dùng
- Tránh over-provisioning storage ban đầu

Best practice:
- Bật Auto Scaling với threshold (ngưỡng) 10% free space
- Set Maximum Storage Threshold để kiểm soát chi phí
- Storage chỉ tăng, không thể giảm tự động
  → Cần snapshot + restore để giảm storage
```

### Migrate gp2 → gp3 (Zero Downtime)

```bash
# Kiểm tra instance đang dùng storage type gì
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,StorageType,AllocatedStorage]' \
  --output table

# Migrate sang gp3 (không có downtime với Multi-AZ)
aws rds modify-db-instance \
  --db-instance-identifier mydb-prod \
  --storage-type gp3 \
  --apply-immediately

# Ước tính tiết kiệm: 1TB storage × ($0.115 - $0.092) × 12 tháng = ~$276/năm
```

---

## Backup & Snapshot Pricing

### Automated Backups (Sao Lưu Tự Động)

```
Chính sách:
- Storage miễn phí = 100% kích thước database
- Ví dụ: DB 500 GB → 500 GB backup storage miễn phí
- Vượt mức: $0.095/GB-month

Retention Period (Chu Kỳ Lưu Giữ):
- Default: 7 ngày
- Range: 0-35 ngày
- Ngày 0: Tắt automated backup hoàn toàn

Tối ưu chi phí:
- Dev/test: Set retention = 1-3 ngày → giảm chi phí backup vượt mức
- Production: Keep 7-14 ngày tùy yêu cầu
- Không giữ 35 ngày nếu RPO chỉ yêu cầu 7 ngày
```

### Manual Snapshots (Ảnh Chụp Thủ Công)

```
Tính phí:
- Tính phí cho toàn bộ snapshot storage
- $0.095/GB-month (us-east-1)
- Cross-region copy: thêm $0.020/GB cho data transfer

Best practices để tiết kiệm:
- Xóa snapshots cũ không cần thiết
- Thiết lập lifecycle policy tự động xóa
- Chỉ giữ checkpoint snapshots quan trọng (trước major migration)
```

```bash
# Xóa snapshots cũ hơn 90 ngày
aws rds describe-db-snapshots \
  --snapshot-type manual \
  --query 'DBSnapshots[?SnapshotCreateTime<=`2025-02-15`].DBSnapshotIdentifier' \
  --output text | tr '\t' '\n' | while read snap; do
    echo "Deleting: $snap"
    aws rds delete-db-snapshot --db-snapshot-identifier "$snap"
  done
```

---

## Data Transfer Pricing

### Trong Cùng Region (Same Region)

```
Trong cùng AZ (Availability Zone — Vùng Sẵn Sàng):
- EC2 → RDS: MIỄN PHÍ
- RDS → EC2: MIỄN PHÍ

Giữa các AZ khác nhau (Cross-AZ):
- $0.02/GB mỗi chiều
- Áp dụng khi: app ở AZ-a, RDS ở AZ-b

→ Best practice: Đặt EC2/ECS và RDS trong cùng AZ để tránh cross-AZ charge
```

### Cross-Region (Xuyên Vùng)

```
Read Replica cross-region replication:
- $0.02/GB data được replicate
- Ví dụ: 1 TB/ngày replicate × $0.02 = $20/ngày = $600/tháng

Cross-region snapshot copy:
- $0.02/GB cho mỗi lần copy
- Snapshot 500 GB copy sang us-west-2 = $10
```

### Internet Transfer Out (Truyền Ra Internet)

```
Tier pricing:
- First 1 GB/month: MIỄN PHÍ
- Up to 10 TB/month: $0.09/GB
- Next 40 TB:        $0.085/GB
- Next 100 TB:       $0.07/GB

→ Ứng dụng nên kết nối RDS qua private network (VPC), không qua internet
→ Dùng VPC Endpoints cho AWS services khác
```

---

## Reserved Instances Strategy

### Tính ROI (Return on Investment — Tỷ Suất Hoàn Vốn) Cho RI

```python
# Công thức tính breakeven cho Reserved Instance

on_demand_monthly = 175  # USD/tháng
ri_1yr_monthly = 110     # USD/tháng (No Upfront)
ri_1yr_upfront = 1,200   # USD trả trước (All Upfront)

monthly_savings_no_upfront = on_demand_monthly - ri_1yr_monthly  # $65/tháng
breakeven_no_upfront = 0  # Tiết kiệm ngay từ tháng đầu

ri_1yr_all_upfront_monthly = ri_1yr_upfront / 12  # $100/tháng equivalent
monthly_savings_all_upfront = on_demand_monthly - ri_1yr_all_upfront_monthly  # $75/tháng
# Nhưng cần trả $1,200 trước
breakeven_all_upfront = ri_1yr_upfront / monthly_savings_all_upfront  # 16 tháng
# All Upfront không breakeven trong 1 năm nếu so với No Upfront → chọn kỳ 3 năm
```

### RI Portfolio Management (Quản Lý Danh Mục RI)

```
Nguyên tắc quản lý:

1. Stagger renewals (Gia hạn so le):
   - Không mua tất cả RI cùng ngày
   - Rải đều 3-6 tháng để có flexibility

2. Match với actual usage:
   - Dùng Cost Explorer → Reserved Instance Utilization
   - Target: RI utilization > 90%
   - RI utilization < 70% → waste, cân nhắc modify hoặc sell

3. RI Marketplace:
   - Có thể bán lại Standard RI không dùng trên AWS Marketplace
   - Convertible RI không bán lại được nhưng có thể exchange

4. Size flexibility:
   - RI áp dụng cho cùng instance family trong cùng region
   - db.r6g.large RI có thể áp dụng cho 2 × db.r6g.medium
     (Theo normalized unit — đơn vị chuẩn hóa)
```

### Quy Trình Mua RI Trong Thực Tế

```
Bước 1: Phân tích 30-90 ngày usage
  - Dùng Cost Explorer → Reserved Instance Recommendations
  - Xem Savings Plans recommendations

Bước 2: Xác nhận workload ổn định
  - CPU/Memory usage patterns
  - Không có migration plan trong 12 tháng
  - Engine version ổn định

Bước 3: Chọn payment option phù hợp
  - Có cash flow → All Upfront (tiết kiệm tối đa)
  - Muốn linh hoạt → No Upfront
  - Cân bằng → Partial Upfront

Bước 4: Mua RI
  - AWS Console → RDS → Reserved Instances → Purchase
  - Chú ý: Region và engine phải khớp
  - DB Instance Class phải khớp (hoặc equivalent units)

Bước 5: Monitor utilization hàng tháng
  - Cost Explorer → Reservations → Utilization Report
  - Alert nếu utilization < 80%
```

---

## Multi-AZ Cost Considerations

### Chi Phí Multi-AZ

```
Multi-AZ Deployment (Triển Khai Đa Vùng Sẵn Sàng):
- Phí instance: x2 (trả cho cả primary và standby)
- Phí storage: KHÔNG x2 (storage chỉ tính một lần)
- Phí I/O: KHÔNG x2

Ví dụ:
Single AZ: db.r6g.large = $175/tháng
Multi-AZ:  db.r6g.large = $350/tháng

→ Multi-AZ thêm ~$175/tháng nhưng mang lại:
  - Automatic failover < 60 giây
  - Maintenance có thể thực hiện không downtime
  - Protection khỏi AZ failure
```

### Khi Nào Không Cần Multi-AZ

```
Không cần Multi-AZ (và tiết kiệm 50%):
- Dev/test environments
- Batch processing databases không cần uptime cao
- Reporting databases với tolerable downtime
- Databases được replicate từ production

Luôn cần Multi-AZ:
- Production OLTP databases
- Bất kỳ database nào có SLA uptime > 99%
- Databases trong regulated environments (PCI, HIPAA)
```

### Multi-AZ vs Read Replica — Trade-off Chi Phí

```
Multi-AZ:
- Chi phí: +100% instance cost
- Mục đích: High Availability (Tính Sẵn Sàng Cao), không cho read scaling
- Automatic failover

Read Replica (Bản Sao Đọc):
- Chi phí: Thêm một instance đầy đủ (instance + storage)
- Mục đích: Scale reads, reporting, analytics
- Không tự động failover (cần thủ công promote)

→ Có thể dùng Read Replica làm DR target
  nhưng cần tắt Multi-AZ nếu muốn tiết kiệm
  → Chấp nhận RTO (Recovery Time Objective) dài hơn
```

---

## Practical Examples

### Ví Dụ 1: Production MySQL RDS — Tối Ưu Chi Phí

```
Tình huống hiện tại:
- db.r6g.2xlarge, Multi-AZ, gp2 100GB, 7-day backup retention
- Region: us-east-1
- Chạy 24/7, On-Demand pricing

Chi phí hiện tại (ước tính):
- Instance On-Demand: ~$700/tháng × 2 (Multi-AZ) = $1,400/tháng
- Storage gp2 100GB: $11.50/tháng
- Backup (100GB free tier): $0
- Total: ~$1,411/tháng

Tối ưu:
1. Mua 1-năm Reserved Instance (No Upfront): ~$440/tháng × 2 = $880/tháng
2. Migrate storage sang gp3: $9.20/tháng (tiết kiệm $2.30)
3. Tổng sau tối ưu: ~$889/tháng

Tiết kiệm: $1,411 - $889 = $522/tháng = $6,264/năm (~37%)
```

### Ví Dụ 2: Dev/Test Environment — Tiết Kiệm Tối Đa

```
Tình huống:
- db.t3.medium, Single AZ, gp3 20GB
- Dùng giờ hành chính: 8h-18h weekdays (50h/tuần vs 168h/tuần)

Chi phí On-Demand 24/7:
- Instance: ~$60/tháng
- Storage: ~$1.84/tháng
- Total: ~$62/tháng

Tối ưu với scheduling (tắt ngoài giờ):
- Chạy 50/168 giờ = 30% thời gian
- Instance cost: $60 × 30% = $18/tháng
- Storage (luôn tính): $1.84/tháng
- Total: ~$20/tháng

Tiết kiệm: $42/tháng = $504/năm (~68% tiết kiệm)
```

```bash
# Tự động tắt/bật RDS instance dùng EventBridge + Lambda
# hoặc dùng AWS Instance Scheduler solution

# Tắt instance
aws rds stop-db-instance --db-instance-identifier mydb-dev

# Bật instance
aws rds start-db-instance --db-instance-identifier mydb-dev

# Lưu ý: RDS tự động bật lại sau 7 ngày dừng liên tục
```

### Ví Dụ 3: Quyết Định io1 vs gp3

```
Yêu cầu: PostgreSQL database cần 8,000 IOPS, 200 GB storage

Tính với io1:
- Storage: 200 GB × $0.125 = $25/tháng
- IOPS: 8,000 × $0.10 = $800/tháng
- Total io1: $825/tháng

Tính với gp3:
- Storage: 200 GB × $0.092 = $18.40/tháng
- IOPS baseline 3,000 miễn phí
- IOPS thêm: (8,000 - 3,000) × $0.02 = $100/tháng
- Total gp3: $118.40/tháng

Kết luận: gp3 rẻ hơn $706/tháng ($8,472/năm)!
→ io1 chỉ cần thiết khi cần > 64,000 IOPS hoặc sub-millisecond latency nhất quán
```

---

## Checklist Tối Ưu Chi Phí RDS

```
Hàng tuần:
□ Xem Trusted Advisor — Idle RDS Instances
□ Kiểm tra RI utilization trong Cost Explorer

Hàng tháng:
□ Review CloudWatch metrics — CPU, connections, IOPS
□ Xem Cost Explorer — database cost trends
□ Xóa unused snapshots cũ hơn 90 ngày

Hàng quý:
□ Right-sizing review dựa trên 90-day metrics
□ Đánh giá mua thêm RI nếu có instances On-Demand chạy ổn định
□ Xem xét nâng cấp từ gp2 sang gp3

Hàng năm:
□ Audit toàn bộ RDS instances — cần thiết không?
□ Renew hoặc mua mới RI sắp hết hạn
□ Đánh giá kiến trúc — có thể chuyển sang Aurora Serverless không?
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**File:** 11-cost-optimization/1-rds-pricing.md
