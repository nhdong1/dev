# ElastiCache Cost Optimization — Tối Ưu Chi Phí ElastiCache

> Hướng dẫn tối ưu chi phí Amazon ElastiCache (Bộ Nhớ Đệm Phân Tán) — bao gồm Reserved Nodes (Nút Đặt Trước), right-sizing (định cỡ phù hợp), chiến lược caching (chiến lược bộ nhớ đệm), và so sánh Redis vs Memcached về mặt chi phí.

## 📚 Mục Lục

1. [Cấu Trúc Giá ElastiCache](#cấu-trúc-giá-elasticache)
2. [Reserved Nodes — Tiết Kiệm Lớn Nhất](#reserved-nodes)
3. [Right-sizing ElastiCache](#right-sizing-elasticache)
4. [Redis vs Memcached — So Sánh Chi Phí](#redis-vs-memcached--so-sánh-chi-phí)
5. [ElastiCache Serverless](#elasticache-serverless)
6. [Caching Effectiveness — Hiệu Quả Cache](#caching-effectiveness)
7. [Chi Phí Ẩn Thường Bỏ Qua](#chi-phí-ẩn-thường-bỏ-qua)
8. [Chiến Lược Tối Ưu Toàn Diện](#chiến-lược-tối-ưu-toàn-diện)

---

## Cấu Trúc Giá ElastiCache

```
Tổng Chi Phí ElastiCache = Node Hours
                         + Backup Storage (Redis only)
                         + Data Transfer
                         + ElastiCache Serverless (nếu dùng)
```

### Node Pricing — Giá Node (us-east-1)

| Node Type         | vCPU | RAM   | On-Demand/giờ | On-Demand/tháng |
| ----------------- | ---- | ----- | ------------- | ---------------- |
| cache.t4g.micro   | 2    | 0.5 GB | $0.016       | ~$11.52          |
| cache.t4g.small   | 2    | 1.4 GB | $0.034       | ~$24.48          |
| cache.t4g.medium  | 2    | 3.1 GB | $0.068       | ~$48.96          |
| cache.r7g.large   | 2    | 13.1 GB| $0.166       | ~$119.52         |
| cache.r7g.xlarge  | 4    | 26.3 GB| $0.332       | ~$239.04         |
| cache.r7g.2xlarge | 8    | 52.6 GB| $0.664       | ~$478.08         |

> **Lưu ý:** r-series (memory optimized — tối ưu bộ nhớ) thường được dùng cho Redis production. t-series cho dev/test hoặc workloads nhỏ.

### Backup Storage (Redis Only — Chỉ Cho Redis)

```
Redis backup pricing:
- $0.085/GB-month cho backup storage

Ví dụ:
- Redis cluster 13 GB data × $0.085 = $1.10/tháng (không đáng kể)
- Backup storage chi phí thấp, không phải điểm tối ưu chính
```

### ElastiCache Serverless Pricing

```
Không tính theo nodes, tính theo:
- ECPUs (ElastiCache Processing Units — Đơn Vị Xử Lý ElastiCache):
  $0.0034/ECPU
- Storage: $0.125/GB-hour (khoảng $90/GB-month)

Khi nào dùng Serverless:
- Variable workloads không đều
- Không muốn quản lý node sizing
- Traffic có peaks và valleys rõ ràng

Khi Serverless ĐẮT HƠN standard:
- Workload ổn định, usage cao
- Production với consistent high throughput
```

---

## Reserved Nodes

### ElastiCache Reserved Nodes (Nút Đặt Trước)

```
Tương tự RDS Reserved Instances:
- 1-year term: ~40% savings
- 3-year term: ~55-60% savings

Payment options:
- No Upfront: Trả hàng tháng, tiết kiệm ít hơn
- Partial Upfront: Trả 50% trước, 50% hàng tháng
- All Upfront: Trả toàn bộ, tiết kiệm nhiều nhất
```

### Bảng Tiết Kiệm Ví Dụ (cache.r7g.large, us-east-1, Redis)

| Loại                     | Giá Hiệu Dụng/Giờ | Giá/Tháng | Tiết Kiệm |
| ------------------------ | ----------------- | --------- | --------- |
| On-Demand                | $0.166            | ~$119.52  | —         |
| 1yr No Upfront RI        | ~$0.105           | ~$75.60   | ~37%      |
| 1yr All Upfront RI       | ~$0.096           | ~$69.12   | ~42%      |
| 3yr No Upfront RI        | ~$0.074           | ~$53.28   | ~55%      |
| 3yr All Upfront RI       | ~$0.067           | ~$48.24   | ~60%      |

### Khi Nào Mua Reserved Nodes

```
Mua RI khi:
✓ Production Redis cluster chạy 24/7 ổn định
✓ Node type không thay đổi trong 1-3 năm
✓ Cache hit rate cao (> 80%) — cluster đang hoạt động hiệu quả
✓ Budget cho phép trả trước

Không mua RI khi:
✗ Dev/test clusters (tắt thường xuyên)
✗ Không chắc về node type (có thể cần upsize/downsize)
✗ Đang thử nghiệm Redis vs Memcached
✗ Workload mới chưa có baseline
```

### Quy Trình Mua Reserved Nodes

```
Bước 1: Thu thập 30-90 ngày metrics
- Node memory utilization
- CPU utilization
- Cache hit rate
- Connections count

Bước 2: Xác nhận workload ổn định
- Không có migration plan
- Node type đang dùng là phù hợp

Bước 3: Mua Reserved Nodes
AWS Console → ElastiCache → Reserved Nodes → Purchase
hoặc:
aws elasticache purchase-reserved-cache-nodes-offering \
  --reserved-cache-nodes-offering-id <offering-id> \
  --cache-node-count 3  # Số nodes cần mua

Bước 4: Monitor utilization hàng tháng
- Reserved node không waste nếu cluster vẫn chạy
- Chỉ waste nếu cluster bị xóa trước khi hết term
```

---

## Right-sizing ElastiCache

### Metrics Cần Theo Dõi Để Right-size

```
Memory metrics:
- FreeableMemory (Bộ Nhớ Trống): Phải luôn > 0
  Nếu = 0 → Eviction xảy ra → Upsize ngay
- BytesUsedForCache (Byte Dùng Cho Cache): 
  Nếu < 40% node memory → Downsize được

CPU metrics:
- EngineCPUUtilization (Redis engine CPU):
  > 80% → Bottleneck, cân nhắc upsize hoặc scale out
  < 20% thường xuyên → Downsize

Cache performance:
- CacheHits / (CacheHits + CacheMisses) = Cache Hit Rate
  > 90% → Cache đang hoạt động tốt
  < 70% → Review caching strategy trước khi upsize

Eviction metrics:
- Evictions > 0 thường xuyên → node memory không đủ → Upsize
```

### Decision Framework Right-sizing

```
Scenario 1: Memory cao nhưng CPU thấp
- BytesUsedForCache > 80% nhưng EngineCPU < 20%
→ Upsize sang node có nhiều RAM hơn (ví dụ: r7g.large → r7g.xlarge)

Scenario 2: CPU cao nhưng Memory thấp
- EngineCPU > 80% nhưng memory usage < 60%
→ Scale out (thêm nodes) hoặc dùng cluster mode
→ Phân tán reads across multiple nodes

Scenario 3: Cả Memory và CPU đều thấp
- Memory < 40%, CPU < 20% consistently
→ Downsize an toàn (tiết kiệm 50% nếu giảm 1 cấp)

Scenario 4: Evictions cao
- Evictions > 0 mỗi giờ
→ Upsize memory ngay, không delay
→ Evictions = cache miss → database hit → performance degradation
```

### Chiến Lược Scale Out vs Scale Up

```
Scale Up (Tăng Cấu Hình Node — Vertical Scaling):
- Tăng instance size: r7g.large → r7g.xlarge
- Đơn giản hơn, ít complexity
- Phù hợp cho single-shard Redis

Scale Out (Thêm Nodes — Horizontal Scaling):
- Thêm read replicas (Bản sao đọc) để scale reads
- Shard với Redis Cluster Mode
- Phân tán data và load across nodes
- Phức tạp hơn nhưng linh hoạt hơn

Chi phí so sánh:
Scale Up: 1 × r7g.xlarge = $239/tháng
Scale Out: 2 × r7g.large = $239/tháng (tương đương chi phí nhưng HA tốt hơn)

→ Scale Out thường tốt hơn cho production vì:
  - Read replicas cho HA
  - Không có single point of failure
  - Có thể add/remove replicas linh hoạt hơn
```

---

## Redis vs Memcached — So Sánh Chi Phí

### Chi Phí Trực Tiếp

```
Redis và Memcached dùng cùng node types và pricing
→ Không có sự khác biệt về giá node

Tuy nhiên Redis cần thêm:
- Backup storage: $0.085/GB-month (nhỏ)
- Monitoring data: Không đáng kể

Memcached không có:
- Persistence (tính bền vững) → không cần backup
- Replication → không có standby
→ Memcached rẻ hơn Redis nếu chỉ tính raw node cost và không cần HA
```

### Chi Phí Gián Tiếp — Tổng Chi Phí Thực

```
Redis với HA (High Availability — Tính Sẵn Sàng Cao):
- Primary + 1 replica = 2 nodes
- Automatic failover < 60 giây
- Cost: 2 × node_price

Memcached Multi-Node:
- Không có replication — mỗi node là standalone
- Nếu node fail → data trên node đó mất
- Cost: N × node_price (N nodes cho capacity)

Ví dụ 13 GB cache:
Redis Cluster (1 primary + 1 replica):
  2 × r7g.large = 2 × $119.52 = $239.04/tháng
  → HA với automatic failover

Memcached 2 nodes (13 GB tổng):
  2 × r7g.large = 2 × $119.52 = $239.04/tháng
  → Không có HA, mỗi node chứa independent data

→ Cùng chi phí nhưng Redis có HA tốt hơn nhiều
→ Chọn Redis trừ khi có lý do cụ thể cần Memcached
```

---

## ElastiCache Serverless

### Cách Tính Phí Serverless

```
ElastiCache Serverless:
- Không cần chọn node type
- Scale tự động từ MB đến TB
- Tính theo ECPUs (ElastiCache Processing Units) và Storage

Pricing:
- $0.0034/ECPU
- $0.125/GB-hour stored (~$90/GB/tháng!)

Ví dụ Serverless:
1 GB data, 1M requests/giờ (vừa read vừa write):
- Storage: 1 GB × $0.125/GB-hour × 720h = $90/tháng
- ECPUs: ước tính 0.5M ECPU/giờ × 720h × $0.0034 = $1,224/tháng
Total Serverless: ~$1,314/tháng

So sánh với cache.r7g.large ($119.52/tháng):
→ Serverless đắt hơn RẤT NHIỀU cho workload ổn định cao!
```

### Khi Nào Serverless Cost-Effective

```
Serverless rẻ hơn khi:
- Traffic rất thấp và intermittent (không liên tục)
- Không biết capacity cần thiết
- Workload burst ngắn rồi idle dài

Ví dụ Serverless cost-effective:
Workload: 100 requests/phút trong 2h/ngày, còn lại idle
Storage: 100 MB

Monthly cost Serverless:
- Storage: 0.1 GB × $0.125 × 720h = $9/tháng
- ECPUs: thấp do traffic thấp = ~$5/tháng
Total: ~$14/tháng

vs cache.t4g.micro ($11.52/tháng) — On-Demand tương đương
→ Serverless xấp xỉ ngang, nhưng không cần manage nodes

Kết luận:
- Serverless tốt cho dev/test, spiky workloads thực sự nhỏ
- Provisioned nodes tốt hơn cho bất kỳ workload production nào
```

---

## Caching Effectiveness — Hiệu Quả Cache

### Tại Sao Cache Hit Rate Quan Trọng Về Chi Phí

```
Cache ROI công thức:
Database cost saved = Cache Miss Rate × Database Read Cost
                    - Cache Miss Rate × Database Read Cost
                    + Cache Hit Rate × Database Read Cost
                   
ROI = (Database reads saved × Database cost/read) - Cache monthly cost

Ví dụ:
- 10M reads/ngày trước khi có cache
- Cache hit rate: 90%
- DynamoDB read cost: $0.25/M RRU

Không có cache:
10M × 30 ngày × $0.25/M = $75/tháng (DynamoDB reads)

Với cache (r7g.large $119.52 + Reserved 3yr = ~$60/tháng):
DynamoDB: 10M × 10% miss rate × 30 × $0.25/M = $7.50/tháng
Cache: $60/tháng (reserved node)
Total: $67.50/tháng

Tiết kiệm: $75 - $67.50 = $7.50/tháng ← KHÔNG đáng!

Với DynamoDB cost cao hơn ($500/tháng):
With cache: $50 (10% DynamoDB) + $60 (cache) = $110/tháng
Without cache: $500/tháng
Tiết kiệm: $390/tháng ← ĐÁNG RẤT NHIỀU!

→ Cache ROI tỷ lệ thuận với database cost saved
→ Cache chỉ cost-effective khi database reads đắt đủ
```

### Tối Ưu Cache Hit Rate

```
Caching strategies ảnh hưởng đến chi phí:

1. Lazy Loading (Cache-Aside — Tải Lười):
   - Cache miss → query database → store in cache
   - Cache hit rate tăng dần theo thời gian
   - Phù hợp cho data ít thay đổi

2. Write-Through (Ghi Xuyên Qua):
   - Write database → write cache cùng lúc
   - Cache luôn fresh, hit rate cao
   - Tốn thêm write bandwidth

3. TTL Optimization (Tối Ưu Thời Gian Sống):
   - TTL quá ngắn → nhiều cache misses → database pressure
   - TTL quá dài → stale data (dữ liệu lỗi thời)
   - Cân bằng theo data volatility (tính biến động dữ liệu)

Tối ưu TTL:
- Static data (dữ liệu tĩnh): TTL 24h-7d
- Semi-static (bán tĩnh): TTL 1-4h
- Dynamic data (dữ liệu động): TTL 1-15 phút
- Real-time data: Không nên cache hoặc TTL < 30 giây
```

---

## Chi Phí Ẩn Thường Bỏ Qua

### 1. Cross-AZ Data Transfer (Truyền Dữ Liệu Xuyên Vùng)

```
ElastiCache nodes trong AZ khác với ứng dụng:
- Mỗi request: $0.02/GB (từ EC2 đến ElastiCache cross-AZ)
- 1 GB cache reads/giờ cross-AZ:
  1 GB × 24h × 30d × $0.02 = $14.40/tháng

→ Đặt ElastiCache và ứng dụng trong cùng AZ
→ Sử dụng subnet group (nhóm mạng con) phù hợp
```

### 2. Replica Lag Costs Indirectly (Chi Phí Gián Tiếp Từ Replica Lag)

```
Redis Replication Lag (Độ Trễ Sao Chép) không tính phí trực tiếp
nhưng có thể dẫn đến:
- Stale reads → ứng dụng đọc data cũ
- Read từ primary thay vì replica để đảm bảo freshness
  → Tăng primary load → cần upsize

Monitor: ReplicationLag metric
Target: < 10ms cho most applications
```

### 3. Backup Chưa Cần Thiết

```
Redis backup:
- Không bắt buộc nếu ElastiCache chỉ dùng làm pure cache
- Pure cache: Nếu mất data → warm up lại từ database
  → Không cần backup → tiết kiệm $0.085/GB-month

Cần backup khi:
- Dùng Redis như primary data store (sessions, rate limiting counters)
- Muốn tránh cold cache sau restart (warm cache)

Tắt backup cho pure cache:
aws elasticache modify-replication-group \
  --replication-group-id my-redis-cluster \
  --snapshot-retention-limit 0  # 0 = Tắt backup
```

### 4. Enhanced Monitoring Granularity

```
Enhanced Monitoring (Giám Sát Nâng Cao) cho ElastiCache:
- Không như RDS, ElastiCache Enhanced Monitoring không tính phí riêng
- CloudWatch metrics có thể tăng chi phí nếu custom metrics

Tuy nhiên:
- CloudWatch Metrics chuẩn: Miễn phí (15 months retention)
- CloudWatch Dashboards: $3/dashboard/tháng
- CloudWatch Alarms: $0.10/alarm-month (10 alarms miễn phí)
→ Không đáng lo ngại, chi phí rất nhỏ
```

---

## Chiến Lược Tối Ưu Toàn Diện

### Tier Hóa Cache Data — Cache Tiering

```
Thay vì 1 lớn ElastiCache node:

Option 1: Application-level cache (In-process caching — Cache Trong Process):
- Dùng Caffeine (Java), LRU cache trong memory của app
- Không tốn ElastiCache cost
- Chỉ tốt cho data ít thay đổi, stateless apps

Option 2: ElastiCache + Application cache:
- L1 cache: In-process (milliseconds, giới hạn bởi heap JVM/Python RAM)
- L2 cache: ElastiCache (sub-ms, shared across instances)
- L3: Database
→ Giảm ElastiCache requests, tiết kiệm ECPU cost

Option 3: ElastiCache tiered storage (Redis 7+ với tiered storage):
- Hot data trong memory (nhanh)
- Cool data trên SSD (chậm hơn nhưng rẻ hơn)
→ Cho phép dùng node nhỏ hơn cho same data size
```

### Schedule Dev/Test ElastiCache

```
ElastiCache không có built-in "stop" như RDS
nhưng có thể:

Option 1: Xóa và tạo lại
- Xóa cluster trước giờ nghỉ, tạo lại khi cần
- Phức tạp, cần automation

Option 2: Scale down về cache.t4g.micro ngoài giờ
- Giảm cost từ $119.52 → $11.52 ngoài giờ
- Scale up lại khi cần

Option 3: Dùng ElastiCache Serverless cho dev
- Trả theo actual usage
- Idle time gần $0

Ví dụ tiết kiệm với schedule:
Development: cache.r7g.large → 0.5 ACU Serverless
Chạy 10h/ngày thay vì 24h + scale:
- Before: $119.52/tháng
- After: ~$15/tháng (Serverless, low traffic)
Tiết kiệm: ~$100/tháng
```

### Đánh Giá Liệu Có Cần ElastiCache Không

```
Câu hỏi quan trọng trước khi thêm ElastiCache:

1. Database có đang chịu tải reads cao không?
   Nếu không → Không cần cache

2. Read/write ratio là bao nhiêu?
   < 80% reads → Cache ít hiệu quả

3. Data có đủ tính lặp lại không (temporal locality)?
   Same data được đọc nhiều lần → Cache tốt
   Mỗi request đọc data khác nhau → Cache ít hiệu quả

4. Database hiện tại có đủ tài nguyên không?
   Nếu RDS CPU < 30% → Thêm cache chưa cần thiết ngay

5. Chi phí cache có thấp hơn chi phí database scale-up không?
   cache.r7g.large: $119.52/tháng
   RDS db.r6g.large → db.r6g.xlarge: thêm $175/tháng
   → Cache rẻ hơn nếu giải quyết được vấn đề
```

---

## Checklist Tối Ưu Chi Phí ElastiCache

```
Setup ban đầu:
□ Chọn đúng node type dựa trên data size + throughput requirements
□ Bật Reserved Nodes cho production clusters
□ Đặt ElastiCache và ứng dụng trong cùng AZ
□ Tắt backup cho pure cache deployments
□ Đặt eviction policy phù hợp (allkeys-lru cho pure cache)

Hàng tuần:
□ Monitor cache hit rate — target > 85%
□ Check FreeableMemory — luôn phải > 0
□ Check Evictions — cần phải = 0 thường xuyên
□ Review CPU utilization

Hàng tháng:
□ Đánh giá node size dựa trên actual usage
□ Tính ROI của cache: Database cost saved vs Cache cost
□ Review Reserved Nodes utilization

Hàng quý:
□ Đánh giá có cần thêm hoặc bớt replica không
□ Xem xét upgrade node generation (ví dụ r6g → r7g, tốt hơn và thường cùng giá)
□ Review caching strategy — TTL values có hợp lý không?
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**File:** 11-cost-optimization/4-elasticache-cost.md
