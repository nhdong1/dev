# Redis Persistence — Tính Bền Vững Dữ Liệu: RDB, AOF, Backup & Restore

> Redis là in-memory database (cơ sở dữ liệu trong RAM) — mặc định dữ liệu mất khi server restart. **Persistence** (Tính Bền Vững) cho phép Redis lưu dữ liệu xuống đĩa để khôi phục sau khi khởi động lại. ElastiCache cung cấp hai cơ chế: **RDB** (Redis Database Backup — Bản Sao Lưu Cơ Sở Dữ Liệu Redis) và **AOF** (Append-Only File — Tệp Chỉ Ghi Thêm).

---

## 🗺️ Tổng Quan Persistence Redis

```
                    Redis Persistence Options
                    (Tùy Chọn Bền Vững Redis)
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
           RDB               AOF          RDB + AOF
   (Periodic Snapshot)  (Write Log)     (Kết hợp cả hai)
   (Ảnh Chụp Định Kỳ)  (Nhật Ký Ghi)
            │                 │                 │
     Nhanh hơn         Toàn vẹn hơn       Tốt nhất
     File nhỏ hơn      File lớn hơn       Phức tạp hơn
     RPO cao hơn       RPO thấp hơn
     (có thể mất        (gần như
      vài phút data)    không mất data)
```

---

## 1️⃣ RDB — Redis Database Backup (Ảnh Chụp Định Kỳ)

### Cơ Chế Hoạt Động

RDB tạo **point-in-time snapshots** (ảnh chụp theo thời điểm) của toàn bộ dataset và lưu xuống đĩa dưới dạng file nhị phân `.rdb`:

```
Redis Memory (RAM)
┌─────────────────────────────────────────┐
│  user:1    → {name: "Alice", age: 30}   │
│  product:5 → {name: "Phone", price: 999}│
│  session:X → {user_id: 1001, ...}       │
│  ...10,000 more keys...                 │
└─────────────────┬───────────────────────┘
                  │
         [BGSAVE command or
          auto-trigger by config]
                  │
                  ▼
         ┌──────────────────┐
         │   Fork process   │  ← Redis fork() một child process
         │   (Child)        │    Parent tiếp tục phục vụ requests
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │  dump.rdb        │  ← File binary chứa snapshot
         │  (Disk/S3)       │    Kích thước nhỏ hơn AOF nhiều
         └──────────────────┘
```

### Cấu Hình RDB

```
# redis.conf — Cấu hình tự động snapshot
save 900 1      # Snapshot nếu ≥1 key thay đổi trong 900 giây (15 phút)
save 300 10     # Snapshot nếu ≥10 keys thay đổi trong 300 giây (5 phút)
save 60 10000   # Snapshot nếu ≥10,000 keys thay đổi trong 60 giây

# Tắt RDB hoàn toàn
save ""

# Tên file snapshot
dbfilename dump.rdb

# Thư mục lưu file
dir /var/lib/redis
```

### Thời Gian Tạo Snapshot

```
Ví dụ: Dataset 10GB trên server 32GB RAM
  ├─ fork() time: ~100-500ms (CPU-intensive)
  ├─ Snapshot time: ~30-60 giây
  └─ Trong suốt thời gian này: Redis vẫn phục vụ đầy đủ (COW — Copy-On-Write)
```

**Copy-On-Write (Sao Chép Khi Ghi):**

```
Parent (Redis vẫn chạy)   Child (Tạo snapshot)
         │                        │
         │   Cùng tham chiếu      │
         │   memory pages         │
         │◄───────────────────────│
         │
         │ Khi Parent ghi key mới:
         │ → Copy page riêng trước khi ghi
         │ → Child vẫn thấy dữ liệu cũ
         │ → Không ảnh hưởng lẫn nhau
```

### Ưu & Nhược Điểm RDB

| ✅ Ưu Điểm | ❌ Nhược Điểm |
|-----------|--------------|
| File nhỏ gọn — dễ backup/transfer | **RPO cao** — có thể mất dữ liệu từ lần snapshot cuối |
| Restore nhanh (load binary file) | fork() gây hiện tượng latency spike ngắn |
| Ảnh hưởng tối thiểu đến hiệu năng | Không phù hợp nếu cần "zero data loss" |
| Phù hợp cho disaster recovery | |

---

## 2️⃣ AOF — Append-Only File (Tệp Chỉ Ghi Thêm)

### Cơ Chế Hoạt Động

AOF ghi lại **mỗi write command** (lệnh ghi) vào một file log. Khi restart, Redis replay lại toàn bộ file để khôi phục state:

```
Redis nhận commands:
SET user:1 "Alice"       ─────► AOF File:
SET product:5 "Phone"           *3\r\n$3\r\nSET\r\n$6\r\nuser:1\r\n$5\r\nAlice\r\n
INCR counter:views               *3\r\n$3\r\nSET\r\n$9\r\nproduct:5\r\n...
DEL temp:session                 *2\r\n$4\r\nINCR\r\n$14\r\ncounter:views\r\n
...                              *2\r\n$3\r\nDEL\r\n$12\r\ntemp:session\r\n
                                 ... (growing forever)

Khi Restart:
Redis đọc AOF file → Replay từng command → Dataset được khôi phục
```

### AOF fsync Policies (Chính Sách Đồng Bộ Đĩa)

`fsync` quyết định khi nào dữ liệu từ OS buffer được flush xuống đĩa vật lý:

```
appendfsync always
  ├─ fsync mỗi write command
  ├─ An toàn nhất — không mất data
  └─ Chậm nhất — ~1000-3000 writes/giây

appendfsync everysec (Khuyến Nghị)
  ├─ fsync mỗi giây một lần
  ├─ Trade-off tốt — mất tối đa 1 giây data
  └─ ~10,000 writes/giây

appendfsync no
  ├─ OS tự quyết định khi nào flush (thường 30 giây)
  ├─ Nhanh nhất
  └─ Có thể mất nhiều data nếu crash
```

### AOF Rewrite — Nén File AOF

AOF file tăng liên tục theo thời gian. Rewrite tạo file mới gọn hơn:

```
File AOF cũ (1GB, nhiều redundant commands — lệnh thừa):
  SET counter 0
  INCR counter      ← tăng 10,000 lần
  INCR counter
  ... (9,998 lần INCR nữa)
  INCR counter

Sau BGREWRITEAOF (Rewrite Nền):
  File AOF mới (nhỏ hơn nhiều):
  SET counter 10000   ← Chỉ trạng thái cuối cùng
```

```
# Cấu hình auto-rewrite
auto-aof-rewrite-percentage 100   # Rewrite khi file tăng gấp đôi kích thước cũ
auto-aof-rewrite-min-size 64mb   # Chỉ rewrite khi file ≥ 64MB
```

### Ưu & Nhược Điểm AOF

| ✅ Ưu Điểm | ❌ Nhược Điểm |
|-----------|--------------|
| **RPO thấp** — mất tối đa 1 giây (everysec) | File lớn hơn RDB nhiều |
| Không mất data với `always` | Restore chậm hơn (phải replay từng lệnh) |
| File dạng text, dễ audit | Rewrite cần disk I/O |
| | Hiệu năng thấp hơn RDB với `always` policy |

---

## 3️⃣ RDB + AOF — Kết Hợp Tốt Nhất

ElastiCache hỗ trợ bật cả hai đồng thời để có lợi ích của cả hai:

```
RDB + AOF Combined (Kết Hợp):

  RDB: Tạo snapshot hàng ngày
       → Recovery nhanh từ snapshot
       → File nhỏ để lưu S3

  AOF: Ghi log mọi write command từ snapshot cuối
       → Nếu crash, load RDB + replay AOF commands từ đó
       → Mất tối đa 1 giây data (với everysec)

Ví dụ khôi phục:
  RDB snapshot lúc 02:00 AM
  AOF log từ 02:00 AM đến 11:45 AM (crash time)
  
  Recovery (Khôi Phục):
  1. Load RDB → state tại 02:00 AM
  2. Replay AOF → state tại ~11:44:59 AM
  → Mất < 1 giây data
```

---

## 4️⃣ ElastiCache Backup — Sao Lưu Trên AWS

### Automatic Backups (Sao Lưu Tự Động)

ElastiCache Redis (không phải Memcached) hỗ trợ sao lưu tự động hàng ngày:

```hcl
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "my-redis"
  
  # Backup Configuration (Cấu Hình Sao Lưu)
  snapshot_retention_limit = 7         # Giữ 7 ngày backup (tối đa 35 ngày)
  snapshot_window          = "02:00-03:00"  # Backup lúc 2-3 giờ sáng UTC
  
  # Không nên đặt backup window trùng với maintenance window
  maintenance_window = "sun:04:00-sun:05:00"
}
```

### Backup Storage (Lưu Trữ Backup)

```
ElastiCache Backup Flow:
  
  Redis Node (RAM) 
        │
        │ BGSAVE — Fork child process
        ▼
  dump.rdb file
        │
        │ Tự động upload
        ▼
  Amazon S3 (Lưu trữ an toàn)
        │
        │ S3 Lifecycle policy tự động xóa sau N ngày
        ▼
  Glacier (Lưu trữ dài hạn, tùy chọn)
```

### Manual Snapshots (Ảnh Chụp Thủ Công)

```bash
# Tạo manual snapshot qua AWS CLI
aws elasticache create-snapshot \
    --replication-group-id my-redis-cluster \
    --snapshot-name "pre-migration-backup-20260515" \
    --region ap-southeast-1

# Liệt kê snapshots
aws elasticache describe-snapshots \
    --replication-group-id my-redis-cluster

# Copy snapshot sang region khác (Cross-Region Backup)
aws elasticache copy-snapshot \
    --source-snapshot-name "pre-migration-backup-20260515" \
    --target-snapshot-name "cross-region-backup" \
    --target-bucket "my-elasticache-backups-us-east-1" \
    --region us-east-1
```

---

## 5️⃣ Restore — Khôi Phục Từ Backup

### Restore Toàn Bộ Cluster

```bash
# Restore từ snapshot — tạo cluster mới từ backup
aws elasticache create-replication-group \
    --replication-group-id "redis-restored" \
    --description "Restored from pre-migration backup" \
    --snapshot-name "pre-migration-backup-20260515" \
    --node-type "cache.r7g.large" \
    --engine "redis" \
    --engine-version "7.1" \
    --num-cache-clusters 3 \
    --automatic-failover-enabled \
    --subnet-group-name "redis-subnet-group" \
    --security-group-ids "sg-abc123"

# Lưu ý: Restore luôn tạo cluster MỚI, không overwrite cluster cũ
# Sau đó, cần update app config để trỏ vào cluster mới
```

### Seed New Cluster From Snapshot (Khởi Tạo Cluster Mới Từ Snapshot)

```
Use case thực tế:
  1. Pre-warming cache (Làm Nóng Cache Trước) cho môi trường mới
  2. Tạo staging environment (Môi Trường Staging) với production data
  3. Disaster Recovery — restore về production sau incident
  4. Database migration — move data sang region mới
```

---

## 6️⃣ RPO & RTO Cho ElastiCache

### RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục)

```
RPO = Bao nhiêu dữ liệu có thể mất khi disaster?

┌─────────────────────────────────────────────────────┐
│ Cấu hình             │ RPO          │ Trade-off      │
├─────────────────────────────────────────────────────┤
│ No persistence       │ Toàn bộ data │ Hiệu năng cao  │
│ RDB (daily)          │ Tối đa 24h   │ Tốt nhất cho   │
│                      │              │ nightly backup  │
│ RDB (mỗi 5 phút)     │ Tối đa 5 phút│ Balance        │
│ AOF (everysec)       │ Tối đa 1s    │ Hiệu năng OK   │
│ AOF (always)         │ ~0 data loss │ Chậm nhất      │
│ RDB + AOF (everysec) │ Tối đa 1s    │ Khuyến nghị    │
└─────────────────────────────────────────────────────┘
```

### RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục)

```
RTO = Bao lâu để hệ thống phục hồi?

┌─────────────────────────────────────────────────────┐
│ Kịch bản             │ RTO                           │
├─────────────────────────────────────────────────────┤
│ Node failure (Multi-AZ ON) │ 1-3 phút (failover)    │
│ Cluster restore từ RDB  │ Phụ thuộc dataset size    │
│   - 1GB dataset         │ ~2-5 phút                 │
│   - 50GB dataset        │ ~20-40 phút               │
│ AOF replay              │ Chậm hơn RDB 2-5x         │
└─────────────────────────────────────────────────────┘
```

---

## 7️⃣ Best Practices (Thực Hành Tốt Nhất)

### Cấu Hình Khuyến Nghị Theo Môi Trường

```
Development (Phát Triển):
  ├─ Persistence: OFF (tiết kiệm I/O)
  ├─ Backup: OFF
  └─ Lý do: Data có thể tái tạo, tiết kiệm cost

Staging (Môi Trường Test):
  ├─ Persistence: RDB mỗi 5 phút
  ├─ Backup: 1 ngày retention
  └─ Lý do: Cần test backup/restore procedures

Production — Cache Layer (Cache Thuần Túy):
  ├─ Persistence: RDB daily hoặc OFF
  ├─ Backup: 7 ngày retention
  └─ Lý do: Cache có thể rebuild từ DB nếu mất

Production — Session Store (Lưu Phiên):
  ├─ Persistence: RDB + AOF (everysec)
  ├─ Backup: 7-14 ngày retention
  └─ Lý do: Không muốn user bị đăng xuất khi incident

Production — Primary Data Store (Kho Dữ Liệu Chính):
  ├─ Persistence: AOF (always) + RDB daily
  ├─ Backup: 14-35 ngày retention
  └─ Lý do: RPO gần như 0, cần full durability
```

### Monitoring Persistence Health (Giám Sát Sức Khỏe Persistence)

```bash
# Kiểm tra trạng thái persistence qua redis-cli
redis-cli INFO persistence

# Output quan trọng:
# rdb_changes_since_last_save:0   ← Số changes chưa được snapshot
# rdb_last_save_time:1715785200   ← Unix timestamp của snapshot cuối
# rdb_last_bgsave_status:ok       ← Trạng thái snapshot cuối
# 
# aof_enabled:1                   ← AOF có bật không
# aof_rewrite_in_progress:0       ← Có đang rewrite không
# aof_last_rewrite_time_sec:45    ← Rewrite lần cuối mất bao nhiêu giây
# aof_current_size:1048576        ← Kích thước AOF hiện tại (bytes)
# aof_last_bgrewrite_status:ok    ← Trạng thái rewrite cuối
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Q: "Giải thích RDB vs AOF — khi nào dùng gì?"

**Trả lời mẫu:**
> "RDB tạo periodic snapshots toàn bộ dataset — file nhỏ, restore nhanh, nhưng có thể mất vài phút đến vài giờ data tùy tần suất snapshot. AOF ghi mọi write command — file lớn hơn, restore chậm hơn, nhưng với `everysec` policy chỉ mất tối đa 1 giây data.
>
> Trong production, tôi thường bật cả hai: RDB daily để có snapshot nhỏ gọn cho fast restore, AOF everysec để đảm bảo RPO < 1 giây. Với pure cache không quan trọng có thể tắt persistence hoàn toàn để tối ưu hiệu năng — cache loss chỉ nghĩa là cache miss, không mất data thật."

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
