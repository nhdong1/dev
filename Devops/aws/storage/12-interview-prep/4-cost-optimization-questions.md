# 💰 Câu Hỏi Phỏng Vấn — Tối Ưu Chi Phí AWS Storage

> Phần tối ưu chi phí xuất hiện trong phần lớn phỏng vấn senior/architect. Tài liệu này tổng hợp các câu hỏi thường gặp và cách trả lời có chiều sâu, thể hiện tư duy "cost-aware engineer".

---

## 🎯 Tại Sao Cost Optimization Quan Trọng Trong Phỏng Vấn?

Interviewer hỏi về cost vì:
1. **Cloud bills thực sự lớn:** Storage và data transfer thường chiếm 20-40% AWS bill
2. **Biết cost = biết trade-offs:** Engineer giỏi phải hiểu tại sao chọn giải pháp này, không phải giải pháp kia
3. **Thể hiện ownership:** Engineer quan tâm đến cost = quan tâm đến business, không chỉ technical elegance

---

## 📊 Phần 1: Câu Hỏi Nền Tảng

---

### Q1: Giải thích bảng giá S3. Có những thành phần chi phí nào?

**Câu trả lời:**

S3 tính phí theo 5 thành phần chính:

**1. Storage (Lưu trữ) — tính theo GB-month:**

| Storage Class | Giá (ap-southeast-1) | Ghi chú |
|--------------|----------------------|---------|
| Standard | $0.025/GB | Không minimum |
| Standard-IA | $0.019/GB | Minimum 30 ngày, minimum 128KB/object |
| One Zone-IA | $0.015/GB | Minimum 30 ngày |
| Glacier Instant | $0.005/GB | Minimum 90 ngày |
| Glacier Flexible | $0.004/GB | Minimum 90 ngày |
| Glacier Deep Archive | $0.002/GB | Minimum 180 ngày |
| Intelligent-Tiering | Theo tier hiện tại + $0.0025/1000 objects monitoring fee |

**2. Requests (Yêu cầu API):**

| Operation | Standard | IA | Glacier |
|-----------|----------|----|---------|
| PUT, COPY, POST | $0.005/1000 | $0.01/1000 | $0.05/1000 |
| GET, SELECT | $0.0004/1000 | $0.001/1000 | Tùy loại |

**3. Data Retrieval (Lấy dữ liệu ra khỏi Glacier):**
- Standard-IA: $0.01/GB
- Glacier Instant: $0.03/GB
- Glacier Flexible (Standard): $0.0025/GB + request fee

**4. Data Transfer Out (Truyền dữ liệu ra):**
- Vào S3 (ingress): Miễn phí
- Ra Internet: $0.09/GB (first 10TB/month)
- Sang CloudFront: Miễn phí
- Sang EC2 cùng region: Miễn phí

**5. Management Features:**
- S3 Inventory: $0.0025/million objects
- S3 Analytics: $0.10/million objects
- Object tagging: $0.01/10.000 tags

**Điểm quan trọng thường bị bỏ qua:**
- Minimum storage duration fees: Xóa object trong Standard-IA trước 30 ngày → vẫn bị tính 30 ngày
- Small object penalty: Object <128KB trong IA vẫn bị tính như 128KB
- Request costs: Với millions of tiny files, request cost > storage cost

---

### Q2: Intelligent-Tiering tiết kiệm tiền trong trường hợp nào? Khi nào KHÔNG nên dùng?

**Câu trả lời:**

Intelligent-Tiering (IT) — Phân Tầng Thông Minh — tự động chuyển object giữa các tier dựa trên access pattern, tính phí monitoring fee $0.0025/1000 objects.

**Tiết kiệm khi:**

```
Scenario: 1TB data, access pattern không biết trước

Không dùng IT:
  → Để Standard: $25/tháng
  → Nếu thực tế 70% không access → lãng phí $17.5/tháng

Dùng IT:
  → Sau 30 ngày: 70% chuyển IA tier → tiết kiệm 40% cho phần đó
  → Monitoring fee: 1TB / 128KB-min = ~8M objects × $0.0025/1000 = $20/tháng (nếu objects nhỏ)
  → Hoặc: 10.000 large objects × $0.0025/1000 = $0.025/tháng (nếu objects lớn)
```

**Hợp lý khi:**
- Objects có size > 128KB (để tránh minimum size penalty)
- Access pattern không dự đoán được — một số truy cập ít, một số truy cập nhiều
- Data tồn tại > 30 ngày
- Số lượng objects không quá nhiều (để monitoring fee không vượt savings)

**KHÔNG nên dùng khi:**

1. **Objects nhỏ (<128KB):** Monitoring fee trên mỗi object làm mất lợi thế
   ```
   1 triệu objects × 1KB = 1GB
   Monitoring: 1M × $0.0025/1000 = $2.5/tháng
   Storage Standard 1GB = $0.025/tháng
   → Monitoring fee > storage fee → KHÔNG DÙNG IT
   ```

2. **Data luôn được access thường xuyên:** IT sẽ giữ ở Frequent Access tier = giá như Standard, nhưng có thêm monitoring fee → tốn hơn

3. **Data chỉ cần lưu ngắn hạn (<30 ngày):** IT không kịp optimize

4. **Access pattern rõ ràng đã biết:** Biết trước rằng data cũ không bao giờ access → dùng lifecycle rule cụ thể, không cần IT monitoring fee

---

### Q3: Tại sao nhiều team quên mất "hidden costs" của S3?

**Câu trả lời — các chi phí thường bị bỏ qua:**

**1. Data Transfer Out (Tốn kém nhất thường bị bỏ qua):**
```
Backup 1TB từ EC2 → S3: Miễn phí (cùng region)
Restore 1TB từ S3 → EC2 khác region: $90
Serve 1TB từ S3 → Internet: $90
→ Giải pháp: CloudFront trước S3 cho public content (giảm data transfer cost)
```

**2. Request Amplification với small files:**
```
1 triệu ảnh thumbnail (1KB/file) — upload qua multipart (1 request/file):
1M PUT requests × $0.005/1000 = $5
Nếu tải về: 1M GET requests × $0.0004/1000 = $0.40

Nhưng nếu có S3 Batch operations trên 1M objects:
$0.25 per 1M operations + inventory cost → có thể đắt hơn expected
```

**3. Lifecycle transition requests:**
```
Mỗi object chuyển tầng (Standard → IA → Glacier) tốn 1 lifecycle request
1M objects × 2 transitions = 2M requests × $0.01/1000 = $20 (cho IA)
Với Glacier: $0.05/1000 → $100 cho 2M transitions
→ Với nhiều small objects, transition cost > storage savings
```

**4. Glacier retrieval costs không ngờ:**
```
Cần restore 10TB từ Glacier Flexible đột ngột:
  Bulk retrieval (5-12h): $0.0025/GB × 10.240GB = $25.60
  Standard retrieval (3-5h): $0.01/GB = $102.40
  Expedited retrieval (<5min): $0.03/GB = $307.20 + $10/1000 requests
→ Không plan cho retrieval cost → bill sốc khi DR test
```

---

## 📊 Phần 2: Câu Hỏi Nâng Cao

---

### Q4: So sánh chi phí giữa EBS gp3 và io2 cho database production. Khi nào io2 xứng đáng chi phí cao hơn?

**Câu trả lời:**

**Chi phí so sánh cho cùng cấu hình 1TB, 10.000 IOPS:**

| | gp3 | io2 |
|--|-----|-----|
| Storage | 1TB × $0.08 = $80/tháng | 1TB × $0.125 = $125/tháng |
| IOPS | 10.000 × $0.005 = $50/tháng | 10.000 × $0.065 = $650/tháng |
| **Tổng** | **$130/tháng** | **$775/tháng** |

**io2 đắt gấp 6 lần — khi nào worth it?**

**1. IOPS > 16.000:** gp3 capped ở 16.000 IOPS, io2 lên tới 64.000 (io2 Block Express: 256.000)

**2. Durability 99.999% vs 99.8-99.9%:** io2 có durability cao hơn gp3 (20x ít bị lỗi hơn)
   ```
   Với production database $10M/ngày revenue:
   gp3 failure rate 0.2% → expected downtime 7.3h/năm
   io2 failure rate 0.001% → expected downtime 0.05h/năm
   
   Nếu downtime = $100K/giờ → tiết kiệm ~$700K/năm vs io2 premium $7.740/năm
   → io2 worth it với mission-critical
   ```

**3. EBS Multi-Attach:** Chỉ io1/io2 hỗ trợ — cần cho Oracle RAC hoặc clustered workloads

**4. io2 Block Express với SAP HANA:** Yêu cầu specifically 256K IOPS — không có alternative

**Khi nào KHÔNG cần io2:**
- OLTP với <5.000 IOPS → gp3 đủ và rẻ hơn 5x
- Read-heavy workloads → thêm Read Replica rẻ hơn nâng cấp storage
- Dev/test environments → gp3 luôn đủ

---

### Q5: Bạn có 100TB log data 2 năm tuổi trong S3 Standard. Cost optimization plan là gì?

**Câu trả lời — Structured approach:**

**Bước 1: Phân tích access pattern trước khi làm bất cứ gì**

```bash
# Bật S3 Analytics cho bucket
aws s3api put-bucket-analytics-configuration \
  --bucket log-bucket \
  --id all-logs-analysis \
  --analytics-configuration '{"Id":"all-logs-analysis","StorageClassAnalysis":{}}'

# Đợi 30 ngày để có đủ data → xem report trong S3 Console
```

**Bước 2: Phân tích S3 Inventory**

```sql
-- Query Athena trên S3 Inventory
SELECT storage_class, 
       count(*) as object_count,
       sum(size)/1024/1024/1024 as size_gb,
       avg(DATEDIFF(day, from_iso8601_timestamp(last_modified_date), current_date)) as avg_age_days
FROM inventory_table
GROUP BY storage_class
ORDER BY size_gb DESC;
```

**Bước 3: Thiết kế lifecycle strategy**

Giả sử log data:
- 0-30 ngày: Debug log (access thường xuyên) → giữ Standard
- 31-90 ngày: Compliance check (access thỉnh thoảng) → Standard-IA
- 91-365 ngày: Audit (access hiếm) → Glacier Instant Retrieval
- 1-2 năm: Long-term (gần như không access) → Glacier Deep Archive

```json
{
  "Rules": [{
    "ID": "log-tiering",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER_IR"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "NoncurrentVersionTransitions": [
      {"NoncurrentDays": 7, "StorageClass": "GLACIER"}
    ],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
  }]
}
```

**Bước 4: Tính toán tiết kiệm**

```
Hiện tại: 100TB Standard = $2.500/tháng

Sau lifecycle:
  10TB recent (0-30 ngày): Standard = $250/tháng
  20TB (31-90 ngày): IA = $380/tháng (tiết kiệm 24%)
  30TB (91-365 ngày): Glacier IR = $150/tháng (tiết kiệm 80%)
  40TB (>1 năm): Deep Archive = $80/tháng (tiết kiệm 92%)

Tổng sau: $860/tháng — tiết kiệm 65% ($1.640/tháng)
Cộng transition cost một lần: ~$50
Break-even: Month 1 ✓
```

---

### Q6: Một DevOps engineer nói "Dùng S3 Intelligent-Tiering cho tất cả buckets để tự động tối ưu". Bạn đồng ý không?

**Câu trả lời — Không đồng ý hoàn toàn, đây là câu trả lời cần nuance:**

IT không phải silver bullet — giải pháp kỳ diệu. Cần đánh giá từng trường hợp:

**Trường hợp IT tốt:**
```
✓ Bucket chứa user-generated content (UGC) — không dự đoán được ai access gì
✓ Objects > 128KB
✓ Objects tồn tại > 30 ngày
✓ Cost của monitoring fee < expected savings
```

**Trường hợp IT không tốt:**

1. **Build artifacts và logs (nhiều small files):**
   ```
   10M objects × 10KB = 100GB
   Monitoring fee: 10M × $0.0025/1000 = $25/tháng
   Storage Standard 100GB = $2.5/tháng
   → Monitoring fee gấp 10 lần storage cost → IT đắt hơn Standard!
   ```

2. **Database backups với access pattern rõ ràng:**
   ```
   Backup hàng ngày → giữ 30 ngày → xóa
   Pattern: chỉ access trong 24h đầu (nếu cần restore)
   → Lifecycle rule cụ thể rẻ hơn IT monitoring fee
   ```

3. **Static assets của website:**
   ```
   CSS, JS files → access liên tục từ CloudFront origin
   → IT sẽ luôn ở Frequent Access tier = Standard price + monitoring fee
   → Tốn hơn Standard thuần
   ```

**Kết luận:** Đề xuất đúng hơn là "Dùng IT cho buckets mà bạn không biết access pattern, và test với pilot bucket trước. Với buckets có pattern rõ, dùng lifecycle rules cụ thể."

---

### Q7: Làm thế nào theo dõi và kiểm soát chi phí S3 theo team hoặc project?

**Câu trả lời:**

**1. Cost Allocation Tags — Thẻ Phân Bổ Chi Phí:**

```bash
# Tag bucket theo team và project
aws s3api put-bucket-tagging \
  --bucket my-bucket \
  --tagging 'TagSet=[{Key=Team,Value=backend},{Key=Project,Value=payments},{Key=Environment,Value=prod}]'

# Tag individual objects
aws s3api put-object-tagging \
  --bucket my-bucket --key report.pdf \
  --tagging 'TagSet=[{Key=CostCenter,Value=engineering}]'
```

Sau đó trong AWS Cost Explorer, filter và group by tag để xem chi phí theo team.

**2. S3 Storage Lens — Kính Phân Tích Lưu Trữ:**

S3 Storage Lens cung cấp dashboard tổng hợp cho toàn organization:
- Tổng storage per bucket, per account, per region
- % objects theo storage class
- Phát hiện unversioned buckets, incomplete multiparts
- Cost metrics: cost per GB theo từng account

**3. AWS Budgets — Ngân Sách AWS:**

```
Tạo budget theo tag:
  Team: backend, Project: payments
  Monthly limit: $500
  Alert: 80% ($400) → email engineering lead
  Alert: 100% ($500) → email CTO
```

**4. AWS Cost Anomaly Detection — Phát Hiện Bất Thường Chi Phí:**

Tự động phát hiện khi S3 cost tăng đột biến so với baseline:
- Không cần set threshold thủ công
- ML-based — biết seasonal patterns
- Alert qua SNS → Slack

---

## 📊 Phần 3: Tình Huống Thực Tế

---

### Q8: Khách hàng nói bill S3 tháng này tăng $5.000 đột ngột. Bạn debug như thế nào?

**Câu trả lời — Quy trình debug có hệ thống:**

**Bước 1: Xác định thành phần tăng (Cost Explorer)**

```
AWS Cost Explorer → Service: S3 → Group by "Usage Type"
Nhìn vào:
- StorageObjectCount (số objects)
- DataTransfer-Out-Bytes (data transfer ra)
- Requests (GET, PUT)
- StorageGlacierRestoreObjectCount (Glacier restore)
```

**Bước 2: Drill down theo bucket**

```
Cost Explorer → Group by "S3 Bucket"
→ Bucket nào tăng nhiều nhất?
→ Tăng storage hay requests?
```

**Bước 3: S3 Storage Lens hoặc Inventory**

```
Nếu storage tăng → Inventory để biết file nào mới được tạo
Nếu requests tăng → Server Access Logs để biết ai đang request gì
Nếu data transfer tăng → CloudFront logs hoặc VPC Flow Logs
```

**Bước 4: CloudTrail investigation**

```
Ai tạo/thay đổi gì trong period đó?
→ CloudTrail → Event history → Filter: S3 PutObject, PutBucketReplication
```

**Common root causes:**
- Developer test upload data lớn vào production bucket không xóa
- Replication bị bật bất ngờ → duplicate data
- Lifecycle rule bị disable → objects không chuyển xuống Glacier
- Application bug → infinite loop tạo objects
- Glacier restore hàng loạt không cần thiết

---

### Q9: Với một workload mới, bạn estimate chi phí EFS vs EBS như thế nào?

**Câu trả lời:**

**Framework so sánh:**

```
Thông tin cần biết:
1. Dung lượng storage cần (GB)
2. Throughput cần (MB/s)
3. IOPS cần
4. Số EC2 instances access (1 hay nhiều?)
5. Access pattern (random hay sequential?)
```

**EBS gp3 estimate (cho 1TB, 5.000 IOPS, 250 MB/s):**

```
Storage: 1.000GB × $0.08 = $80/tháng
IOPS: 5.000 IOPS × $0.005 = $25/tháng (3.000 IOPS miễn phí với gp3)
Throughput: 250 MB/s × $0.04 = $10/tháng (125 MB/s miễn phí với gp3)
Snapshot (20% của volume): 200GB × $0.05 = $10/tháng
Tổng: ~$125/tháng
```

**EFS estimate (cho 1TB, Elastic Throughput):**

```
Storage:
  - Standard tier: 500GB × $0.30 = $150/tháng (data nóng)
  - IA tier: 500GB × $0.016 = $8/tháng (sau 30 ngày không access)
Throughput (Elastic): pay-per-use
  - Estimated: 10GB/ngày read × $0.03/GB = $9/tháng
  - Write: 2GB/ngày × $0.06/GB = $3.60/tháng
Tổng: ~$170/tháng
```

**Kết luận so sánh:**

```
EBS: $125/tháng cho 1 EC2 — không chia sẻ được
EFS: $170/tháng cho nhiều EC2 — chia sẻ được

Nếu cần 3 EC2 chia sẻ storage:
- EBS: $125 × 3 = $375/tháng (3 volume riêng)
- EFS: $170/tháng (1 file system, 3 EC2 mount)
→ EFS rẻ hơn khi nhiều instances
```

---

## 🏆 Điểm Cốt Lõi Cần Nhớ

1. **S3 storage cost là phần nhỏ** — data transfer out và requests thường là "surprise"
2. **Minimum duration fees** — Standard-IA (30 ngày), Glacier (90/180 ngày) — tính toán lifecycle kỹ
3. **Small objects cần cẩn thận** — overhead per-object (monitoring fee, request fee) có thể vượt savings
4. **Measure trước, optimize sau** — S3 Analytics, Storage Lens, Cost Explorer là công cụ bắt buộc
5. **EFS rẻ hơn EBS khi scale** — nhiều instance chia sẻ 1 EFS vs nhiều EBS riêng lẻ
6. **io2 chỉ worth it khi cần >16K IOPS hoặc Multi-Attach** — 99% workloads dùng gp3

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
