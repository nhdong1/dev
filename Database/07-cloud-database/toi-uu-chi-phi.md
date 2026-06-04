# Tối Ưu Chi Phí Cloud Database

## 1. Hiểu Cấu Trúc Chi Phí

### AWS RDS / Aurora

```
Chi phí RDS bao gồm:
  1. Instance hours         → lớn nhất (~50-60% tổng)
  2. Storage (GB/month)     → gp3: $0.115/GB, io1: $0.125/GB + IOPS fee
  3. I/O requests           → chỉ với gp2/io1 (gp3 không tính I/O)
  4. Backup storage         → miễn phí = storage size, tính phí phần dư
  5. Data transfer out      → $0.09/GB (first 10TB/month)
  6. RDS Proxy              → $0.015/vCPU/hour
  7. Performance Insights   → 7 ngày miễn phí, 2 năm: $0.02/vCPU/month

Aurora thêm:
  8. Aurora I/O             → $0.20/million requests (trừ Aurora I/O-Optimized)
  9. Aurora I/O-Optimized   → instance đắt hơn 25% nhưng không tính I/O fee
     → Break-even: khi I/O cost > 25% instance cost

Ví dụ tính phí Aurora:
  db.r6g.large (us-east-1):
    Standard:       $0.26/hour × 730h = $189/month instance
                  + $0.20/million I/O → $50-200/month (tùy workload)
    I/O-Optimized:  $0.325/hour × 730h = $237/month (không có I/O fee)
    
  → Nếu I/O cost > $48/month → chuyển I/O-Optimized
```

### DynamoDB

```
On-Demand:
  Writes: $1.25 per million WRU (Write Request Unit)
  Reads:  $0.25 per million RRU (Read Request Unit)
  Storage: $0.25/GB/month
  
Provisioned:
  WCU: $0.00065/hour per WCU = $0.47/WCU/month
  RCU: $0.00013/hour per RCU = $0.094/RCU/month
  
Break-even: On-Demand vs Provisioned
  On-Demand rẻ hơn khi utilization < ~20% thời gian
  Provisioned + Auto Scaling rẻ hơn khi traffic đều và có thể predict

Hidden costs:
  GSI: tốn RCU/WCU riêng (replica writes)
  DynamoDB Streams: $0.02/100,000 read requests
  Global Tables: 2x write cost + data transfer
  DAX: instance cost riêng
```

### Cosmos DB

```
Provisioned Throughput:
  100 RU/s = $0.008/hour = $5.76/month (single region)
  Multi-region write: giá × số regions

Autoscale:
  Charge = max RU/s provisioned × $0.012/100RU/hour
  (ngay cả khi không dùng)
  
Serverless:
  $0.25/million RU consumed
  Storage: $0.25/GB/month
  Tốt cho: < 5000 RU/s, sporadic traffic

Storage: $0.25/GB/month

So sánh: 1000 RU/s liên tục
  Provisioned: 1000 × $0.012/100 × 730h ≈ $87.6/month
  Serverless:  1000 RU/s = 86.4 billion RU/month = $21,600
  → Provisioned rẻ hơn rất nhiều khi traffic cao liên tục
```

---

## 2. Chiến Lược Tiết Kiệm Chi Phí

### Reserved Instances (AWS) — Tiết kiệm 30-60%

```
On-Demand: $0.26/hour db.r6g.large
           = $189.8/month

Reserved Instances:
  1-year No Upfront:    $0.162/hour = $118/month  (tiết kiệm 37%)
  1-year All Upfront:   $0.148/hour = $108/month  (tiết kiệm 43%)
  3-year All Upfront:   $0.103/hour = $75/month   (tiết kiệm 60%)

Khi nào mua Reserved:
  ✓ Instance chạy liên tục > 8 tháng/năm
  ✓ Đã stable, không cần thay đổi instance type thường xuyên
  ✗ Startup/new project: chưa biết cần bao nhiêu

Convertible RI:
  - Cho phép đổi instance type trong term
  - Tiết kiệm ít hơn Standard RI ~5-10%
  - Tốt khi không chắc về instance type tương lai
```

### Azure Reserved Capacity

```
Azure Database for PostgreSQL:
  On-Demand:  D4s v3 = ~$200/month
  1-year:     Tiết kiệm ~33%  → ~$134/month
  3-year:     Tiết kiệm ~60%  → ~$80/month

Cosmos DB Reserved Capacity:
  100 RU/s = $5.76/month On-Demand
  1-year Reserved: ~17% discount
  3-year Reserved: ~37% discount
```

### Right-Sizing

```
Quy trình right-sizing:
1. Thu thập 2-4 tuần CloudWatch/Azure Monitor metrics
2. Xác định peak CPU, Memory usage
3. Target: peak CPU ~60-70% (headroom cho spike)

Ví dụ:
  Hiện tại: db.r6g.2xlarge (8 vCPU, 64GB RAM)
  Metrics:  Peak CPU 40%, Peak Memory 35%
  
  Tính toán cần:
    CPU: 8 × 40% = 3.2 vCPU → cần 4 vCPU tối thiểu
    RAM: 64 × 35% = 22.4GB → cần 32GB tối thiểu
  
  Đề xuất: db.r6g.xlarge (4 vCPU, 32GB RAM)
    → Tiết kiệm 50% instance cost

Tools hỗ trợ:
  AWS: Compute Optimizer → đề xuất right-size tự động
  AWS: Cost Explorer → rightsizing recommendations
  Azure: Azure Advisor → rightsizing recommendations
```

### Dev/Test Cost Optimization

```
AWS:
  ✓ Tắt instance ngoài giờ làm (save ~65% chi phí)
    Lambda schedule: stop at 6PM, start at 8AM weekdays
    aws rds stop-db-instance --db-instance-identifier mydev-db
    ⚠️ RDS tự restart sau 7 ngày liên tục stopped
  
  ✓ Single-AZ cho non-production (tiết kiệm 50%)
  ✓ gp2/gp3 thay vì io1 cho dev
  ✓ Smaller instance + burst (db.t3.medium)
  ✓ Snapshot + delete: xóa instance ban đêm, restore từ snapshot sáng
    → Chỉ tốn phí snapshot storage ($0.095/GB/month)

Aurora Serverless v2 cho dev/test:
  ACU (Aurora Capacity Unit) min = 0.5 → scale to 0 sau idle
  Chi phí: $0.12/ACU/hour (minimum 0.5 ACU)
  Khi không dùng: ~$0.06/hour (rất rẻ)

Azure:
  ✓ Burstable tier (B series) cho dev
  ✓ Stop server ngoài giờ (Portal → Stop)
  ✓ Azure Dev/Test subscription pricing (giảm ~30%)
```

---

## 3. Storage Optimization

### Chọn Storage Type Đúng

```
AWS RDS Storage:
  
  gp2 (legacy):
    3 IOPS/GB, burst to 3000 IOPS
    Không dùng cho instance mới
  
  gp3 (khuyến nghị):
    3000 IOPS + 125MB/s baseline MIỄN PHÍ
    Provision thêm IOPS: $0.02/IOPS/month
    Provision thêm throughput: $0.04/MB-s/month
    Rẻ hơn gp2 ~20% cho cùng performance
  
  io1/io2:
    IOPS cao (64,000 IOPS với io2)
    $0.125/GB + $0.065/IOPS/month (io1)
    Chỉ cần khi thực sự cần > 16,000 IOPS

Migrate gp2 → gp3 (không downtime):
  aws rds modify-db-instance \
    --db-instance-identifier mydb \
    --storage-type gp3 \
    --apply-immediately
```

### DynamoDB Storage Optimization

```
TTL (Time to Live):
  Bật TTL → DynamoDB tự xóa expired items miễn phí
  Không tốn WCU để xóa, xóa trong vòng 48h sau expire
  
  Thiết lập:
    Attribute: "expiry_time" (Unix timestamp)
    Bật TTL trên attribute đó
  
  Usecase:
    Session data:     TTL = login_time + 30 days
    Cache entries:    TTL = now + 1 hour
    Temporary tokens: TTL = creation + 15 minutes
    Audit logs:       TTL = now + 90 days

Compress large attributes:
  Item > 400KB → không cho phép
  Item > 100KB → tăng WCU/RCU đáng kể
  → Compress JSON trước khi lưu (gzip ~70-80% reduction)
  → Lưu large binary vào S3, DynamoDB lưu S3 URL

Tiered storage:
  DynamoDB Standard:          $0.25/GB/month
  DynamoDB Standard-IA:       $0.10/GB/month (infrequently accessed)
  → Chọn Standard-IA cho data ít truy cập (old orders, archive)
```

---

## 4. Data Transfer Cost

```
Data Transfer thường bị bỏ qua nhưng có thể lớn!

AWS:
  Trong cùng AZ:   MIỄN PHÍ
  Cross-AZ:        $0.01/GB mỗi chiều
  Cross-Region:    $0.02-0.09/GB
  Internet out:    $0.09/GB (first 10TB)

⚠️ Vấn đề phổ biến:
  App ở AZ-a, RDS ở AZ-b → $0.02/GB data transfer
  → Luôn đặt app và DB cùng AZ cho primary connection
  → Read replica: có thể cross-AZ (nhưng tính phí)

Azure:
  Trong cùng Region:  MIỄN PHÍ (inbound)
  Cross-Region:       $0.02-0.08/GB
  Internet out:       $0.087/GB (first 10TB, Zone 1)

Tối ưu:
  1. Sử dụng VPC Endpoints / Private Endpoints
     → Traffic không đi qua internet, tránh NAT Gateway cost
  2. Compress data giữa app và DB
  3. Dùng read replicas cùng region/AZ với read-heavy services
```

---

## 5. Monitoring Chi Phí

### AWS Cost Monitoring

```
AWS Cost Explorer:
  Filter by: Service=RDS, Service=DynamoDB
  Group by: Usage Type → thấy breakdown chi tiết
  
AWS Budgets:
  Tạo budget: $500/month cho RDS
  Alert khi: 80% ($400) và 100% ($500) đạt ngưỡng
  Notify: email + SNS → Slack webhook

AWS Cost Anomaly Detection:
  Tự động phát hiện bất thường chi phí
  Alert khi DynamoDB tăng đột biến (vd: bug gây scan toàn bộ table)

Tagging strategy:
  Bắt buộc tag:
    Environment: production | staging | development
    Team:        backend | data | platform
    Project:     project-name
  → Filter cost theo team/project
```

### Azure Cost Monitoring

```
Azure Cost Management:
  Budgets: đặt ngưỡng cảnh báo
  Cost alerts: daily/monthly notifications
  Cost analysis: drill down theo service, resource group
  
Azure Advisor Cost recommendations:
  Tự động đề xuất:
    - Right-size underutilized resources
    - Purchase Reserved Instances
    - Delete unused resources
  
Tags:
  environment, team, project → allocate cost đúng center
```

---

## 6. Tóm Tắt — Quick Wins

```
✅ Ngay lập tức (không impact production):
   □ Migrate RDS gp2 → gp3 storage (tiết kiệm ~20%)
   □ Tắt dev/test databases ngoài giờ (tiết kiệm ~65%)
   □ Bật TTL cho DynamoDB data có thời hạn (giảm storage)
   □ Bật AWS Cost Anomaly Detection
   □ Xóa manual snapshots cũ không cần thiết

✅ Ngắn hạn (1-4 tuần, cần review):
   □ Right-size sau khi phân tích 4 tuần metrics
   □ Chuyển DynamoDB từ On-Demand → Provisioned + AutoScaling
   □ Aurora: tính toán Standard vs I/O-Optimized
   □ Xem xét Reserved Instances cho production (đã stable)

✅ Trung hạn (1-3 tháng):
   □ Mua Reserved Instances / Reserved Capacity
   □ Thiết kế DynamoDB Single Table Design (giảm số table, GSI)
   □ Implement tiered storage (Standard-IA cho cold data)
   □ Review và optimize cross-AZ data transfer
```
