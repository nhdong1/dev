# Quản Lý & Vận Hành Cloud Database

## 1. Monitoring — Các Metrics Quan Trọng

### AWS RDS — CloudWatch Metrics

```
Nhóm Compute:
  CPUUtilization        > 80% sustained  → scale up compute
  FreeableMemory        < 20% RAM        → scale up / tune
  SwapUsage             > 0              → RAM thiếu, scale ngay
  
Nhóm Storage:
  FreeStorageSpace      < 20%            → mở rộng storage
  DiskQueueDepth        > 1 sustained    → storage IOPS bottleneck
  ReadIOPS / WriteIOPS  (baseline & trend)
  ReadLatency / WriteLatency > 20ms      → storage vấn đề

Nhóm Database:
  DatabaseConnections   > 80% max_connections → thêm RDS Proxy
  ActiveTransactions                     → deadlock nếu tăng bất thường
  BlockedTransactions   > 0              → điều tra deadlock
  
Nhóm Replication:
  ReplicaLag            > 60s            → replica không kịp primary
  
Nhóm Network:
  NetworkReceiveThroughput
  NetworkTransmitThroughput

CloudWatch Alarms nên tạo:
  CPUUtilization > 80% for 5 minutes
  FreeStorageSpace < 10GB
  DatabaseConnections > (max_connections * 0.85)
  ReplicaLag > 30s (nếu có read replica)
  FreeableMemory < 256MB
```

### Performance Insights (AWS) — Bắt Buộc Bật

```
Performance Insights cho thấy:
  - DB Load theo thời gian (số active sessions)
  - Breakdown by: SQL, Wait, User, Host
  - Top SQL queries tiêu thụ nhiều nhất
  
DB Load > số vCPU → hệ thống bị overloaded

Retention:
  7 ngày: miễn phí
  2 năm:  $0.02/vCPU/month (khuyến nghị production)

Sử dụng:
  1. Mở Performance Insights
  2. Xem "Database load" chart
  3. Filter theo "Top SQL" → thấy query nào gây bottleneck
  4. Click vào query → xem execution plan
```

### Azure Monitor — Database Metrics

```
Azure Database for PostgreSQL:
  cpu_percent           > 80%       → scale compute
  memory_percent        > 85%       → scale / tune
  storage_percent       > 85%       → scale storage
  io_consumption_percent > 85%      → storage bottleneck
  connections_failed                → authentication / network issues
  active_connections    > max_pool  → connection exhaustion
  network_bytes_ingress / egress

Query Store (tương đương Performance Insights):
  Bật: pg_qs.query_capture_mode = ALL
  Xem: Azure Portal → Query Performance Insight
  Tự động track: top CPU, top duration, top executions

Azure Monitor Alerts:
  Tạo alert rules cho:
  - cpu_percent > 80% → action: notify + scale
  - storage_percent > 85% → notify team ngay
  - connections_failed > 100/min → security investigation
```

### Cosmos DB Metrics

```
Key metrics:
  Normalized RU Consumption (%) 
    > 100% → throttling → 429 errors
    Target: giữ < 70% để có buffer
  
  Server Side Latency
    P99 > 10ms (reads) → query cần tối ưu
    P99 > 15ms (writes) → check partition distribution
  
  Throttled Requests
    > 0 → cần tăng RU hoặc optimize queries
  
  Document Count per Partition
    Uneven distribution → hot partition → check partition key

Alerts cần thiết:
  NormalizedRUConsumption > 80%
  ServerSideLatency P99 > 10ms
  ThrottledRequests > 0 sustained
```

---

## 2. Scaling

### AWS RDS — Vertical Scaling (Scale Up)

```
Quy trình scale up instance:
1. Kiểm tra maintenance window
2. Multi-AZ: failover sang standby trước khi scale primary
   → Downtime: ~60s cho failover, sau đó scale standby không downtime
3. Single instance: có downtime (~5-15 phút)

Auto Scaling Storage (không downtime):
  Bật: Storage Autoscaling
  Threshold: scale khi còn < 10% free
  Maximum: đặt giới hạn (vd: 500GB)
  ⚠️ Storage chỉ tăng, không giảm được

Aurora Auto Scaling (Read Replicas):
  Bật Aurora Auto Scaling cho read replicas:
  Target metric: RDSReaderAverageCPUUtilization
  Min replicas: 1, Max replicas: 5
  Cooldown: 300s scale in, 60s scale out
```

### DynamoDB — Scaling

```
On-Demand: tự động scale, không cần làm gì

Provisioned + Auto Scaling:
  Target utilization: 70% (để có buffer)
  Min capacity: đủ cho base load
  Max capacity: đủ cho peak × 1.5
  
  ⚠️ DynamoDB Auto Scaling có độ trễ vài phút
     Nếu traffic tăng đột ngột → có thể bị throttle trước khi scale
     Fix: pre-warm bằng cách tăng capacity trước event lớn

Table-level scaling limits:
  Max 40,000 RCU và 40,000 WCU per table (soft limit, có thể xin tăng)
  Mỗi partition: max 3,000 RCU, 1,000 WCU
  → Partition nhiều → higher throughput limit
```

### Cosmos DB — Scaling

```
Manual Provisioned:
  Thay đổi RU/s bất kỳ lúc nào qua portal/API
  Min: 400 RU/s, Max: không giới hạn (tính theo storage)
  
Autoscale:
  Đặt Max RU/s, tự scale từ 10%-100%
  Tốt cho: traffic không đều
  Chi phí: tính theo Max RU/s đã set (dù dùng 10%)
  
Serverless:
  Trả đúng RU đã dùng
  Tốt cho: dev/test, intermittent workloads
  ⚠️ Không hỗ trợ multi-region, max 5000 RU/s per container

Scale Up Cosmos DB Account RU:
  Container level: thay đổi RU/s của từng container
  Database level: shared throughput chia cho nhiều containers
```

---

## 3. Maintenance & Patching

### AWS RDS Maintenance

```
Maintenance Window:
  Mặc định: random 30 phút mỗi tuần
  Khuyến nghị: đặt cụ thể vào cuối tuần, giờ thấp điểm
  Ví dụ: Sun:02:00-Sun:03:00 UTC
  
Pending Modifications:
  "Apply Immediately" vs "During Next Maintenance Window"
  Production: LUÔN chọn "During Next Maintenance Window"
  Trừ security patch critical
  
Auto Minor Version Upgrade:
  Tắt cho production (kiểm soát timing)
  Bật cho dev/staging (luôn mới nhất)

Major Version Upgrade:
  PostgreSQL 14 → 15 → 16 (không skip major version)
  Cần test application compatibility trước
  Quy trình: snapshot → restore sang version mới → test → failover
  
Blue/Green Deployments (AWS):
  - Tạo Green environment từ snapshot Blue
  - Test trên Green
  - Switch traffic: Blue → Green trong vài giây
  - Rollback: switch lại Blue nếu có vấn đề
  - Xóa Blue sau khi confirm
```

### Azure Database Maintenance

```
Maintenance window:
  Custom window: chọn ngày giờ cụ thể
  Thông báo: 7 ngày và 72 giờ trước
  
Minor/Major version updates:
  Minor: tự động trong maintenance window
  Major: phải thực hiện manual (vd: PostgreSQL 14 → 15)
  
Planned maintenance notification:
  Azure Service Health → Resource Health alerts
  Bật email/SMS notification cho maintenance events

High Availability và Maintenance:
  Zone Redundant HA: failover sang standby trước khi patch primary
  → Downtime ngắn hơn (<2 phút cho failover)
```

---

## 4. Backup & Restore

### AWS RDS Backup

```
Automated Backup:
  Retention: 1-35 ngày
  Stored trong S3 (managed, không thấy trong console)
  PITR: restore đến bất kỳ giây nào trong retention window
  
  Restore PITR:
  aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mydb \
    --target-db-instance-identifier mydb-restored \
    --restore-time 2026-04-29T10:30:00Z
  
  ⚠️ Restore tạo INSTANCE MỚI, không overwrite instance cũ
     Cần update application connection string sau khi restore

Manual Snapshots:
  Không expire (giữ nguyên đến khi xóa thủ công)
  Cross-region copy: aws rds copy-db-snapshot --destination-region
  Cross-account share: modify snapshot attributes

Aurora Backup:
  Storage-level backup, không ảnh hưởng performance
  Backtrack: "rewind" Aurora cluster trong vòng 72 giờ
    → Không cần restore, áp dụng trực tiếp lên cluster
    aws rds backtrack-db-cluster --db-cluster-identifier mycluster \
      --backtrack-to 2026-04-29T10:00:00Z
```

### RTO/RPO với Cloud Databases

```
Service         | RPO           | RTO
----------------|---------------|----------------
RDS Multi-AZ    | ~0 (sync)     | ~60s (auto failover)
RDS Single-AZ   | Last backup   | Minutes (restore)
Aurora          | ~0 (sync)     | ~30s (auto failover)
Aurora Serverless| ~0           | ~25s
DynamoDB Global | Seconds       | Seconds (auto)
Cosmos DB Multi | Config based  | <1s (active-active)

Azure DB Flex HA | ~0 (sync)    | ~60-120s (auto)
Azure DB Flex   | Last backup   | Minutes (restore)
```

---

## 5. Disaster Recovery

### Cross-Region DR — AWS

```
Strategy 1: Cross-Region Read Replica
  - Tạo Read Replica ở DR region (us-west-2)
  - RPO: vài giây (async replication lag)
  - RTO: 5-20 phút (promote replica)
  
  Promote khi failover:
  aws rds promote-read-replica \
    --db-instance-identifier mydb-replica-west2
  ⚠️ Sau promote: replica thành standalone, link bị xóa

Strategy 2: Cross-Region Backup Copy
  - Tự động copy backup sang DR region
  - RPO: time of last backup (vd: 1-24h)
  - RTO: 15-60 phút (restore từ backup)
  
  aws rds create-db-instance-read-replica \
    --db-instance-identifier mydb-dr \
    --source-db-instance-identifier arn:aws:rds:us-east-1:xxx:db:mydb \
    --destination-region us-west-2

Strategy 3: Aurora Global Database
  - 1 primary region, tối đa 5 secondary regions
  - Replication lag < 1s
  - RPO: < 1s
  - RTO: < 1 phút (managed failover)
```

### Cross-Region DR — Azure

```
Geo-redundant backup:
  - Backup tự động copy sang paired region
  - RPO: tùy backup frequency
  - Restore sang region khác nếu primary region down

Read Replica trong region khác:
  - Azure DB for PostgreSQL hỗ trợ cross-region replicas
  - Promote khi cần failover

Cosmos DB Global Distribution:
  - Enable multi-region trong vài click
  - Mỗi region serve reads locally
  - With multi-region writes: active-active tự động
```

---

## 6. Runbook: Xử Lý Sự Cố Thường Gặp

### CPU Cao (> 80%)

```
1. Mở Performance Insights / Query Store
2. Xem top queries theo CPU
3. Kiểm tra EXPLAIN ANALYZE cho top query
4. Actions:
   a. Missing index → tạo index (CONCURRENTLY để không lock)
   b. N+1 query → fix ở application code
   c. Lock contention → xem pg_locks, tìm blocking queries
   d. Nếu cần ngay: scale up instance (vài phút downtime nếu Single-AZ)
```

### Connection Exhausted

```
Triệu chứng: "too many connections" / connection timeout

1. Kiểm tra: SELECT count(*) FROM pg_stat_activity;
2. Xem connections theo application:
   SELECT application_name, count(*) FROM pg_stat_activity GROUP BY 1;
3. Kill idle connections cũ:
   SELECT pg_terminate_backend(pid) 
   FROM pg_stat_activity 
   WHERE state = 'idle' AND query_start < NOW() - INTERVAL '10 minutes';
4. Dài hạn:
   - Giảm connection pool size ở application
   - Thêm PgBouncer / RDS Proxy
   - Tăng max_connections (cần restart)
```

### Storage Gần Đầy

```
1. Ngay lập tức: 
   - AWS: modify instance → increase storage (không downtime)
   - Azure: scale up storage tier (không downtime)
2. Điều tra:
   SELECT pg_size_pretty(pg_database_size('mydb'));  -- database size
   SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
   FROM pg_catalog.pg_statio_user_tables ORDER BY 2 DESC LIMIT 10;
3. Dọn dẹp:
   - Xóa WAL archive cũ (nếu quản lý thủ công)
   - VACUUM FULL trên bảng bloat lớn (cần maintenance window)
   - Archive/delete data cũ
4. Dài hạn: bật Storage Autoscaling
```

### Replication Lag Tăng

```
1. Kiểm tra lag:
   -- Trên Primary:
   SELECT client_addr, state, sent_lsn, write_lsn, 
          (sent_lsn - write_lsn) AS lag_bytes
   FROM pg_stat_replication;
   
   -- AWS CloudWatch: ReplicaLag metric
2. Nguyên nhân:
   - Replica CPU/IO bị quá tải
   - Long-running transaction trên primary (blocking replication)
   - Network bandwidth
3. Fix:
   - Scale up replica instance
   - Kiểm tra long transactions: SELECT * FROM pg_stat_activity WHERE state = 'active'
   - Tăng max_wal_senders, wal_sender_timeout
```
