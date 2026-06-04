# Aurora Cost Optimization — Tối Ưu Chi Phí Aurora

> Hiểu chi phí Aurora (Cơ Sở Dữ Liệu Đám Mây Hiệu Năng Cao) và cách tối ưu — từ I/O costs (chi phí I/O), Aurora Serverless v2, đến việc lựa chọn đúng storage mode tiết kiệm hàng nghìn USD mỗi tháng.

## 📚 Mục Lục

1. [Cấu Trúc Giá Aurora](#cấu-trúc-giá-aurora)
2. [Aurora Standard vs I/O-Optimized](#aurora-standard-vs-io-optimized)
3. [Aurora Serverless v2 — Cost Model](#aurora-serverless-v2--cost-model)
4. [Aurora vs RDS — So Sánh Chi Phí](#aurora-vs-rds--so-sánh-chi-phí)
5. [Reserved Instances cho Aurora](#reserved-instances-cho-aurora)
6. [Aurora Global Database — Chi Phí](#aurora-global-database--chi-phí)
7. [Chiến Lược Tối Ưu Aurora](#chiến-lược-tối-ưu-aurora)

---

## Cấu Trúc Giá Aurora

```
Tổng Chi Phí Aurora = Instance Hours
                    + Storage (per GB-month, tự động co giãn)
                    + I/O Requests (Standard mode) hoặc không (I/O-Optimized)
                    + Backup Storage vượt mức
                    + Data Transfer
                    + Tùy chọn: Aurora Serverless ACU hours, Global Database replication
```

### Điểm Khác Biệt So Với RDS

```
Aurora storage khác RDS:
- Tự động tăng theo 10 GB increments (bước tăng 10 GB)
- Chỉ trả cho storage thực sự dùng (không cần pre-provision)
- Storage được chia sẻ giữa tất cả instances trong cluster
- Không bao giờ cần "resize" storage thủ công

I/O tính khác:
- Aurora Standard: Tính $0.20/1M I/O requests (QUAN TRỌNG: đây là cost ẩn lớn!)
- Aurora I/O-Optimized: Không tính I/O nhưng storage cao hơn 25%
```

---

## Aurora Standard vs I/O-Optimized

### Aurora Standard

```
Giá (us-east-1, Aurora MySQL):
- Storage: $0.10/GB-month
- I/O:     $0.20/1M requests

Phù hợp cho:
- Workloads I/O thấp (ít read/write operations)
- Development và testing
- Applications với < 25% chi phí là I/O
- Khi I/O cost < 25% tổng Aurora cost
```

### Aurora I/O-Optimized

```
Giá (us-east-1, Aurora MySQL):
- Storage: $0.225/GB-month (cao hơn 125% so với Standard)
- I/O:     MIỄN PHÍ (không tính I/O requests)

Phù hợp cho:
- I/O-intensive workloads (khối lượng công việc nhiều I/O)
- I/O cost > 25% tổng Aurora cost
- OLTP systems với nhiều reads/writes liên tục
- Databases với nhiều concurrent transactions
```

### Breakeven Analysis — Phân Tích Điểm Hòa Vốn

```
Giả sử: 1 TB storage

Aurora Standard:
- Storage cost: 1,000 GB × $0.10 = $100/tháng
- I/O cost: X × $0.20/1M

Aurora I/O-Optimized:
- Storage cost: 1,000 GB × $0.225 = $225/tháng
- I/O cost: $0

Breakeven:
$225 = $100 + (X/1,000,000 × $0.20)
$125 = X × $0.0000002
X = 625,000,000 I/O requests/tháng
  ≈ 20.8 million I/O requests/ngày
  ≈ 240 I/O requests/giây

→ Nếu database có > 240 IOPS trung bình, I/O-Optimized rẻ hơn
→ Nếu database có > 25% chi phí là I/O, chuyển sang I/O-Optimized
```

### Cách Kiểm Tra I/O Cost Hiện Tại

```bash
# Dùng AWS Cost Explorer để xem I/O cost tách biệt
# Filter by: Service = Amazon Aurora
# Group by: Usage Type
# Tìm: "Aurora:StorageIOUsage"

# Dùng CloudWatch để ước tính I/O requests
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name VolumeReadIOPs \
  --dimensions Name=DBClusterIdentifier,Value=my-aurora-cluster \
  --start-time 2026-04-15T00:00:00Z \
  --end-time 2026-05-15T00:00:00Z \
  --period 2592000 \
  --statistics Sum \
  --output json

# Cộng VolumeReadIOPs + VolumeWriteIOPs để có tổng I/O
```

### Chuyển Đổi Giữa Standard và I/O-Optimized

```bash
# Chuyển sang I/O-Optimized (không downtime với Aurora)
aws rds modify-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --storage-type aurora-iopt1 \
  --apply-immediately

# Chuyển về Standard
aws rds modify-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --storage-type aurora \
  --apply-immediately

# Lưu ý: Có thể chuyển tối đa 1 lần mỗi 30 ngày
```

---

## Aurora Serverless v2 — Cost Model

### Cách Tính Phí Aurora Serverless v2

```
ACU — Aurora Capacity Unit (Đơn Vị Năng Lực Aurora):
- 1 ACU = ~2 GB RAM + CPU tương ứng
- Minimum: 0.5 ACU (có thể scale về 0 nếu pause)
- Maximum: 256 ACU

Giá:
- $0.06/ACU-hour (us-east-1, Aurora MySQL Serverless v2)
- Storage: $0.10/GB-month (Standard) hoặc $0.225 (I/O-Optimized)

Ví dụ:
- Chạy ở 2 ACU liên tục 24/7:
  2 × $0.06 × 24 × 30 = $86.40/tháng (chỉ instance cost)
- Provisioned db.r6g.large tương đương: $175/tháng
  → Serverless tiết kiệm ~51% nếu traffic đều đặn ở mức thấp
```

### Min/Max ACU Configuration — Cấu Hình Năng Lực Tối Thiểu/Tối Đa

```
min_capacity = 0.5 ACU:
- Không chấp nhận connections khi không có load
- Cold start ~15-30 giây khi traffic vào đột ngột
- Tiết kiệm tối đa cho idle periods

min_capacity = 1-2 ACU:
- Luôn có capacity để xử lý requests ngay
- Không bị cold start
- Phù hợp cho dev/test, staging

min_capacity = 4+ ACU:
- Đảm bảo capacity cho production workloads
- Scale nhanh khi traffic tăng

Khuyến nghị:
- Dev/test:     min=0.5, max=4 ACU
- Staging:      min=1, max=16 ACU
- Production:   min=2, max=64+ ACU (theo yêu cầu thực tế)
```

### Aurora Serverless v2 vs Provisioned Aurora — Khi Nào Dùng Gì

```
Dùng Serverless v2 khi:
✓ Traffic không đều — có peak và valley rõ ràng
✓ Dev/test environments (idle nhiều giờ)
✓ Staging environments
✓ Applications mới chưa có traffic baseline
✓ Workloads theo giờ hành chính (8h-18h)
✓ Muốn đơn giản hóa capacity planning

Dùng Provisioned Aurora khi:
✓ Traffic ổn định và dự đoán được 24/7
✓ Production OLTP với SLA latency nghiêm ngặt
✓ Đã có Reserved Instances
✓ Peak/average ratio < 2x (không cần scale nhiều)

So sánh chi phí ví dụ (traffic 10h/ngày full load, 14h idle):
- Provisioned db.r6g.large 24/7: $175/tháng
- Serverless v2 (10h @4 ACU + 14h @0.5 ACU):
  (4 × $0.06 × 10 × 30) + (0.5 × $0.06 × 14 × 30)
  = $72 + $12.60 = $84.60/tháng (~52% tiết kiệm)
```

### Auto Pause với Serverless v1 (Cũ — Chỉ Tham Khảo)

```
Aurora Serverless v1 (deprecated — không còn khuyến nghị):
- Có thể pause hoàn toàn (0 ACU, scale to zero)
- Cold start 25-30 giây khi resume
- Chỉ hỗ trợ MySQL 5.7 và PostgreSQL 10.x

Aurora Serverless v2 (hiện tại):
- Minimum 0.5 ACU (không scale về 0, trừ khi cluster paused)
- Warm start < 1 giây
- Hỗ trợ Aurora MySQL 8.0+, PostgreSQL 14+
- Có thể pause cluster thủ công hoặc qua API
```

---

## Aurora vs RDS — So Sánh Chi Phí

### Khi Aurora Đắt Hơn RDS

```
Aurora luôn đắt hơn RDS ở mức instance nhỏ:

db.t3.medium MySQL RDS: ~$60/tháng (Single-AZ)
db.t3.medium Aurora:    ~$65/tháng (không có t3.medium cho Aurora)
Nhỏ nhất Aurora:        db.t3.small → $35/tháng (Aurora)

Vấn đề thực tế:
- Aurora không có instance types nhỏ như RDS (không có db.t3.micro cho Aurora)
- Minimum viable Aurora instance: db.t3.small (~$35/tháng) cho dev

Nếu so sánh cùng instance type, Aurora đắt hơn khoảng 20-30%
nhưng bù lại bằng:
- Không cần Multi-AZ riêng (Aurora tự replicate 6 copies)
- Storage I/O throughput cao hơn nhiều
- Read Replica nhanh hơn (replication lag < 10ms vs RDS hàng giây)
```

### Khi Aurora Rẻ Hơn RDS (Total Cost of Ownership)

```
Ví dụ: Production MySQL 500 GB, 10,000 IOPS yêu cầu, Multi-AZ

RDS io1 Multi-AZ:
- Instance db.r6g.xlarge × 2 (Multi-AZ): $700/tháng
- Storage io1 500 GB: 500 × $0.125 = $62.50/tháng
- IOPS 10,000: 10,000 × $0.10 = $1,000/tháng
- Total RDS: ~$1,762.50/tháng

Aurora MySQL Standard:
- Instance db.r6g.xlarge × 1 (Aurora tự HA): $350/tháng
- Storage 500 GB: 500 × $0.10 = $50/tháng
- I/O (10,000 IOPS × 3600s × 24h × 30d = 25.9B requests):
  25,920M × $0.20/1M = $5,184/tháng  ← RẤT CAO!

Aurora MySQL I/O-Optimized:
- Instance: $350/tháng
- Storage 500 GB: 500 × $0.225 = $112.50/tháng
- I/O: $0
- Total Aurora I/O-Opt: ~$462.50/tháng

Kết luận: Aurora I/O-Optimized rẻ hơn RDS io1 Multi-AZ ~$1,300/tháng!
```

---

## Reserved Instances cho Aurora

### Aurora Reserved Instances Hoạt Động Như Thế Nào

```
Giống RDS Reserved Instances:
- 1-year term: ~40% savings
- 3-year term: ~60% savings
- No Upfront / Partial Upfront / All Upfront

Áp dụng cho Aurora provisioned instances
(Không áp dụng cho Aurora Serverless v2 — tính theo ACU-hour)

Ví dụ tiết kiệm (db.r6g.xlarge Aurora MySQL):
- On-Demand: ~$350/tháng
- 1yr RI No Upfront: ~$220/tháng (-37%)
- 3yr RI All Upfront: ~$140/tháng (-60%)
```

### Savings Plans Thay Thế RI

```
Compute Savings Plans (Kế Hoạch Tiết Kiệm Tính Toán):
- Cam kết $X/giờ chi tiêu trong 1 hoặc 3 năm
- Áp dụng tự động cho RDS, Aurora, và nhiều dịch vụ khác
- Flexibility hơn RI — không cần chọn instance type trước

Database-specific RI vẫn cho phần trăm tiết kiệm cao hơn Compute Savings Plans
cho RDS/Aurora cụ thể.
```

---

## Aurora Global Database — Chi Phí

### Cấu Trúc Chi Phí Global Database

```
Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu):
- Primary region: Bình thường như Aurora cluster thông thường
- Secondary regions: Instance cost + Storage cost + Replication cost

Replication cost:
- $0.20/GB data replicated đến secondary region
- Ví dụ: 100 GB/ngày write volume × $0.20 = $20/ngày = $600/tháng (chỉ replication!)

Total cost ví dụ (1 primary + 1 secondary):
- Primary cluster (db.r6g.large): $175/tháng
- Secondary cluster (db.r6g.large reader): $175/tháng
- Replication (50 GB/ngày): $300/tháng
- Storage (secondary trả riêng): ~$50/tháng
- Total: ~$700/tháng so với single-region ~$175/tháng
```

### Tối Ưu Chi Phí Global Database

```
Chiến lược tiết kiệm:

1. Dùng nhỏ hơn ở secondary region:
   Primary: db.r6g.2xlarge
   Secondary: db.r6g.large (đủ để phục vụ reads và failover)

2. Không cần replica writer ở secondary region:
   Secondary chỉ cần reader instances
   Promote lên writer chỉ khi failover

3. Cân nhắc alternative: Cross-region Read Replica thay vì Global Database
   - Rẻ hơn (~$0.02/GB replication vs $0.20/GB)
   - Nhưng replication lag cao hơn (seconds vs milliseconds)
   - Failover thủ công, không tự động

4. Tính toán kỹ trước khi deploy:
   Global Database chỉ justify khi:
   - Cần < 1 giây replication lag
   - Cần automated failover < 1 phút
   - Compliance yêu cầu multi-region data residency
```

---

## Chiến Lược Tối Ưu Aurora

### 1. Audit I/O Usage — Kiểm Tra Mức Sử Dụng I/O

```bash
# CloudWatch Metrics cần theo dõi
Metrics quan trọng:
- VolumeReadIOPs    — Số read I/O operations
- VolumeWriteIOPs   — Số write I/O operations
- AuroraVolumeBytesLeftTotal — Dung lượng trống

# Tính I/O cost hàng tháng từ metrics:
# Total I/O requests/tháng × $0.20 / 1,000,000

# Rule of thumb: Nếu I/O cost > 25% tổng Aurora cost
# → Chuyển sang I/O-Optimized
```

### 2. Right-size Instances — Định Cỡ Phù Hợp

```
Sử dụng AWS Compute Optimizer:
- Phân tích 14-30 ngày metrics
- Gợi ý instance type phù hợp
- Ước tính % tiết kiệm

Targets:
- Average CPU < 40% → cân nhắc downsize
- CPU peak < 70% → có thể downsize 1 cấp
- Memory free > 30% → có thể downsize

Chú ý Aurora-specific:
- Buffer Pool cache hit ratio > 99% là mục tiêu
- Nếu < 95% → cần thêm memory (upsize)
```

### 3. Optimize Reader Instances — Tối Ưu Reader

```
Aurora cluster có thể có đến 15 reader instances

Vấn đề thường gặp:
- Nhiều readers nhưng utilization thấp
- Readers có instance type lớn hơn cần thiết

Tối ưu:
- Monitor reader utilization riêng biệt
- Dùng Auto Scaling cho Aurora Readers:
  aws rds register-scalable-target \
    --service-namespace rds \
    --resource-id cluster:my-aurora-cluster \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --min-capacity 1 \
    --max-capacity 5

- Scale out chỉ khi cần, scale in khi traffic giảm
- Readers nhỏ hơn writer nếu chỉ dùng cho read-heavy queries
```

### 4. Aurora Parallel Query — Tối Ưu Analytics

```
Aurora Parallel Query (Truy Vấn Song Song Aurora):
- Tính năng đẩy query processing xuống storage layer
- Giảm I/O từ storage lên instance (tiết kiệm I/O cost)
- Tăng tốc analytical queries lên 3-50x

Bật Parallel Query:
aws rds modify-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --enable-cloudwatch-logs-exports '["parallelquery"]'

Lưu ý chi phí:
- Parallel Query có thể tăng I/O cost (đọc nhiều data hơn)
- Nhưng tiết kiệm được compute cost và thời gian query
- Trade-off: I/O cost ↑ nhưng instance cost ↓ và query time ↓
```

### 5. Backup Optimization — Tối Ưu Sao Lưu

```
Aurora Backup:
- Tự động backup storage = 100% cluster size miễn phí
- Vượt mức: $0.021/GB-month

Tối ưu:
- Set backup retention phù hợp với RPO thực tế
- Dev: 1-3 ngày
- Staging: 3-7 ngày
- Production: 7-14 ngày (không nhất thiết phải 35 ngày)

Backtrack (Aurora MySQL):
- Cho phép "rewind" cluster lên đến 72 giờ (không cần restore)
- Phí: $0.012/GB-hour để enable
- Chỉ bật nếu thực sự cần tính năng này
```

### 6. Aurora Clones — Tiết Kiệm Storage Cho Dev/Test

```
Aurora Fast Clone (Nhân Bản Nhanh):
- Tạo bản sao database gần như ngay lập tức
- Clone KHÔNG tạo bản sao storage — dùng copy-on-write
- Chỉ tính phí cho data thay đổi sau khi clone

Chi phí tiết kiệm:
- Không cần restore từ snapshot (tiết kiệm thời gian)
- Clone 1 TB database: gần như $0 ban đầu
- vs Restore từ snapshot: trả đầy đủ 1 TB storage ngay

Use case:
- Tạo dev/test environment từ production data
- Load testing (Kiểm Tra Tải)
- Thử nghiệm migration trước khi apply lên production

aws rds restore-db-cluster-to-point-in-time \
  --db-cluster-identifier my-aurora-clone \
  --source-db-cluster-identifier my-aurora-prod \
  --restore-type copy-on-write \
  --use-latest-restorable-time
```

---

## Tổng Kết: Aurora Cost Decision Tree

```
Aurora cluster của bạn:

1. I/O cost > 25% total Aurora cost?
   Có → Chuyển sang I/O-Optimized
   Không → Giữ Standard

2. Traffic uniform 24/7?
   Có → Dùng Provisioned + Reserved Instances
   Không → Cân nhắc Serverless v2

3. Production với SLA cao?
   Có → Provisioned + RI + tắt unnecessary readers
   Không → Serverless v2 với auto-scaling

4. Cần multi-region?
   RPO < 1s → Global Database (đắt hơn)
   RPO > 1s → Cross-region Read Replica (rẻ hơn)

5. Có development/test clusters?
   Có → Aurora Serverless v2, min=0.5 ACU, auto-pause hoặc schedule tắt
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**File:** 11-cost-optimization/2-aurora-cost.md
