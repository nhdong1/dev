# Disaster Recovery — Chiến Lược Khôi Phục Thảm Họa

> DR (Disaster Recovery — Khôi Phục Thảm Họa) là tập hợp các chính sách, công cụ và quy trình cho phép khôi phục hệ thống sau một sự kiện gián đoạn nghiêm trọng — từ lỗi phần cứng, tấn công ransomware, đến thảm họa tự nhiên ảnh hưởng toàn bộ AWS region. Một DR strategy tốt được thiết kế dựa trên RPO/RTO đã xác định và được kiểm tra định kỳ trước khi cần dùng thực sự.

---

## 🎯 Bốn Chiến Lược DR Cơ Bản

AWS định nghĩa 4 chiến lược DR theo thứ tự từ chi phí thấp đến cao (và RTO từ chậm đến nhanh):

```
                  Chi Phí & Độ Phức Tạp
                          ▲
    Multi-Site            │
    Active-Active ────────┤ RTO: < 1 phút
    (Đa Điểm              │ RPO: ~0
    Chủ-Chủ)              │ Chi phí: Rất cao (2x infrastructure)
                          │
    Warm Standby ─────────┤ RTO: < 15 phút
    (Dự Phòng Ấm)         │ RPO: < 1 phút
                          │ Chi phí: Cao (~50-75% production)
                          │
    Pilot Light ──────────┤ RTO: < 1 giờ
    (Đèn Phi Công)        │ RPO: < 5 phút
                          │ Chi phí: Trung bình (~10-20% production)
                          │
    Backup & Restore ─────┤ RTO: > 1 giờ
    (Sao Lưu & Khôi Phục) │ RPO: Vài giờ
                          │ Chi phí: Thấp nhất
                          │
                          ▼
                  RTO & Độ Phức Tạp Kỹ Thuật
```

---

## 📋 Chiến Lược 1 — Backup & Restore (Sao Lưu & Khôi Phục)

### Đặc Điểm

```
Phù Hợp: Non-critical systems (hệ thống không quan trọng), dev/staging
RPO: Vài giờ (phụ thuộc vào backup frequency)
RTO: 1-4+ giờ (thời gian restore snapshot)
Chi Phí: Thấp nhất — chỉ trả cho backup storage

Kiến Trúc:
  Region Chính (us-east-1)           DR Region (us-west-2)
  ┌───────────────────────┐          ┌──────────────────────┐
  │   RDS/Aurora Production│         │  Không có DB         │
  │                        │  Copy   │  running (chạy)      │
  │  Automated Backup ─────────────►│                      │
  │  Manual Snapshots ─────────────►│  Snapshots lưu ở đây│
  └───────────────────────┘          └──────────────────────┘
  
  Khi DR:
  1. Restore snapshot ở DR region (~1-4 giờ)
  2. Update DNS trỏ về DR region endpoint
  3. App hoạt động trở lại
```

### Triển Khai

```bash
# Script tự động copy snapshot sang DR region hàng ngày
#!/bin/bash
SOURCE_REGION="us-east-1"
DR_REGION="us-west-2"
DB_IDENTIFIER="prod-database"
DR_KMS_KEY="arn:aws:kms:us-west-2:123456789:key/xxxxx"

# Tìm snapshot mới nhất
LATEST_SNAPSHOT=$(aws rds describe-db-snapshots \
  --db-instance-identifier $DB_IDENTIFIER \
  --snapshot-type automated \
  --region $SOURCE_REGION \
  --query 'reverse(sort_by(DBSnapshots, &SnapshotCreateTime))[0].DBSnapshotIdentifier' \
  --output text)

# Copy sang DR region
COPY_ID="dr-copy-${LATEST_SNAPSHOT}-$(date +%Y%m%d)"
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier \
    "arn:aws:rds:${SOURCE_REGION}:123456789:snapshot:${LATEST_SNAPSHOT}" \
  --target-db-snapshot-identifier "$COPY_ID" \
  --kms-key-id "$DR_KMS_KEY" \
  --region "$DR_REGION"

echo "DR snapshot $COPY_ID đang được copy sang $DR_REGION"
```

---

## 📋 Chiến Lược 2 — Pilot Light (Đèn Phi Công)

### Đặc Điểm

```
Tên gọi: Giống ngọn lửa nhỏ luôn sẵn sàng châm lửa lớn khi cần
Phù Hợp: Business-critical apps có thể chịu RTO 1 giờ
RPO: < 5 phút (cross-region read replica lag)
RTO: 30-60 phút (promote replica + scale up)
Chi Phí: Trung bình (~$50-200/tháng thêm cho replica nhỏ)

Kiến Trúc:
  Region Chính (us-east-1)           DR Region (us-west-2)
  ┌───────────────────────┐          ┌──────────────────────┐
  │   RDS Primary         │          │   Cross-Region       │
  │   db.r6g.2xlarge      │─────────►│   Read Replica       │
  │   (Full production    │  Async   │   db.t3.medium       │
  │    capacity)          │  Repl.   │   (Nhỏ, chi phí thấp)│
  └───────────────────────┘          └──────────────────────┘
  
  Khi DR:
  1. Promote replica lên primary (5-10 phút)
  2. Scale up instance class (10-20 phút)
  3. Update DNS
  4. App hoạt động trở lại tại DR region
```

### Triển Khai

```bash
# Bước 1: Tạo cross-region read replica nhỏ để Pilot Light
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-db-pilot-light \
  --source-db-instance-identifier \
    arn:aws:rds:us-east-1:123456789:db:prod-database \
  --db-instance-class db.t3.medium \    # Instance nhỏ để tiết kiệm
  --destination-region us-west-2 \
  --region us-west-2 \
  --publicly-accessible false \
  --kms-key-id alias/rds-dr-key

# Khi DR xảy ra — Bước 1: Promote replica
aws rds promote-read-replica \
  --db-instance-identifier prod-db-pilot-light \
  --region us-west-2

# Đợi promotion hoàn thành
aws rds wait db-instance-available \
  --db-instance-identifier prod-db-pilot-light \
  --region us-west-2

# Khi DR xảy ra — Bước 2: Scale up instance (vì đang dùng t3.medium)
aws rds modify-db-instance \
  --db-instance-identifier prod-db-pilot-light \
  --db-instance-class db.r6g.2xlarge \  # Upgrade lên production class
  --apply-immediately \
  --region us-west-2
```

---

## 📋 Chiến Lược 3 — Warm Standby (Dự Phòng Ấm)

### Đặc Điểm

```
Phù Hợp: Mission-critical apps cần RTO < 15 phút
RPO: < 1 phút (near real-time replication)
RTO: 5-15 phút (đã có infrastructure sẵn sàng, chỉ cần failover)
Chi Phí: Cao — DR region chạy 50-75% capacity của production

Kiến Trúc:
  Region Chính (us-east-1)                DR Region (us-west-2)
  ┌─────────────────────────┐             ┌─────────────────────────┐
  │  Aurora Primary Cluster  │             │  Aurora Secondary       │
  │  Writer: r6g.4xlarge    │             │  Cluster (Global DB)    │
  │  Reader: r6g.2xlarge    │────────────►│  Reader: r6g.2xlarge   │
  │                          │  < 1 sec   │  (Smaller, read-only)   │
  │  App: 10 instances       │  repl. lag │  App: 3 instances       │
  │  (Full capacity)         │            │  (Scaled down, warm)    │
  └─────────────────────────┘             └─────────────────────────┘
  
  Khi DR:
  1. Trigger Aurora Global DB managed failover (< 1 phút)
  2. Auto Scaling Group ở DR region scale up app instances
  3. DNS failover tự động (Route 53 Health Check)
  4. App hoạt động trở lại với capacity thấp hơn ban đầu
  5. Scale up thêm khi cần
```

### Terraform Warm Standby Setup

```hcl
# Aurora Global Database cho Warm Standby
resource "aws_rds_global_cluster" "production" {
  global_cluster_identifier = "prod-global-cluster"
  engine                    = "aurora-mysql"
  engine_version            = "8.0.mysql_aurora.3.04.0"
  database_name             = "myapp"
  deletion_protection       = true
}

# Primary cluster ở us-east-1
resource "aws_rds_cluster" "primary" {
  provider = aws.us_east_1
  
  cluster_identifier        = "prod-cluster-primary"
  global_cluster_identifier = aws_rds_global_cluster.production.id
  engine                    = "aurora-mysql"
  engine_version            = "8.0.mysql_aurora.3.04.0"
  
  master_username = var.db_username
  master_password = var.db_password
  
  backup_retention_period     = 14
  preferred_backup_window     = "02:00-03:00"
  preferred_maintenance_window = "sun:03:30-sun:04:30"
  
  db_subnet_group_name   = aws_db_subnet_group.primary.name
  vpc_security_group_ids = [aws_security_group.rds_primary.id]
}

# Secondary cluster ở us-west-2 (DR region)
resource "aws_rds_cluster" "secondary" {
  provider = aws.us_west_2
  
  cluster_identifier        = "prod-cluster-secondary"
  global_cluster_identifier = aws_rds_global_cluster.production.id
  engine                    = "aurora-mysql"
  engine_version            = "8.0.mysql_aurora.3.04.0"
  
  # Secondary không có master credentials — dùng từ global cluster
  
  db_subnet_group_name   = aws_db_subnet_group.secondary.name
  vpc_security_group_ids = [aws_security_group.rds_secondary.id]
  
  depends_on = [aws_rds_cluster_instance.primary_writer]
}
```

---

## 📋 Chiến Lược 4 — Multi-Site Active-Active (Đa Điểm Chủ-Chủ)

### Đặc Điểm

```
Phù Hợp: Global apps cần 99.99%+ uptime, tài chính, e-commerce toàn cầu
RPO: ~0 (synchronous hoặc < 1 giây lag)
RTO: < 1 phút (tự động, không cần manual intervention)
Chi Phí: Rất cao — 2x+ infrastructure

Kiến Trúc với Aurora Global Database:
  us-east-1 (Primary)               us-west-2 (Secondary)           eu-west-1 (Secondary)
  ┌──────────────────┐               ┌──────────────────┐             ┌──────────────────┐
  │  Aurora Cluster  │               │  Aurora Cluster  │             │  Aurora Cluster  │
  │  Writer + Reader │──────────────►│  Reader only     │◄────────── │  Reader only     │
  │                  │  < 1 sec      │                  │  < 1 sec   │                  │
  │  Writes tại đây  │               │  Reads tại đây   │             │  Reads tại đây   │
  └──────────────────┘               └──────────────────┘             └──────────────────┘
  
  Route 53 → Latency-based routing (Định Tuyến Dựa Trên Độ Trễ)
  Reads → Đến region gần nhất
  Writes → Đến primary region
  
  Khi DR (primary region fail):
  1. Aurora Managed Planned Failover tự động promote secondary
  2. Route 53 health check phát hiện primary failed
  3. DNS tự động redirect về secondary region
  4. RTO < 1 phút
```

---

## 🗺️ DR Strategy Selection Framework (Khung Lựa Chọn Chiến Lược DR)

### Decision Tree (Cây Quyết Định)

```
                    Bắt đầu
                       │
            RPO yêu cầu < 5 phút?
              ┌────────┴─────────┐
              │ Không            │ Có
              ▼                  ▼
       RTO < 4 giờ?         RTO < 15 phút?
       ┌──────┴──────┐       ┌──────┴──────┐
       │ Không       │ Có    │ Không       │ Có
       ▼             ▼       ▼             ▼
   Backup      Pilot      Warm          Multi-Site
   & Restore   Light      Standby       Active-Active
   
   RPO: giờ   RPO: 5m   RPO: <1m      RPO: ~0
   RTO: giờ+  RTO: 1h   RTO: 15m      RTO: <1m
   Cost: $    Cost: $$  Cost: $$$     Cost: $$$$
```

---

## 🔄 Failover Procedures (Quy Trình Chuyển Đổi Dự Phòng)

### RDS Multi-AZ Automatic Failover

```
Trigger (Kích Hoạt) Tự Động Khi:
  - Primary instance hardware failure (lỗi phần cứng)
  - AZ (Availability Zone — Vùng Sẵn Sàng) outage
  - DB software crash
  - Network connectivity issue (vấn đề kết nối mạng)
  - Maintenance window OS patching

Quy Trình Tự Động:
  1. AWS phát hiện primary failure (~30 giây)
  2. Promote standby replica lên primary
  3. DNS endpoint được cập nhật (~60 giây)
  4. App cần reconnect (cần retry logic — logic thử lại)
  
  Tổng thời gian: 60-120 giây

Quy Trình Kiểm Tra (Testing):
  # Force failover để test
  aws rds reboot-db-instance \
    --db-instance-identifier my-production-db \
    --force-failover
```

### Aurora Cluster Failover

```
Aurora Failover Priority (Thứ Tự Ưu Tiên Failover):
  - Tier 0: Cao nhất — Promote ngay lập tức
  - Tier 15: Thấp nhất — Promote sau cùng
  
  Cấu hình trong Console: Instance → Modify → Failover priority

Quy Trình:
  1. Writer instance failure detected
  2. Aurora chọn reader instance có priority cao nhất
  3. Promote reader → writer (~15-30 giây)
  4. Cluster endpoint tự động cập nhật
  5. App reconnect qua cluster endpoint

# Set failover tier cho reader instance
aws rds modify-db-instance \
  --db-instance-identifier aurora-reader-1 \
  --promotion-tier 1    # Tier 1 = Được promote sớm nhất
```

### Aurora Global Database Managed Failover

```bash
# Failover Aurora Global Database sang DR region
# Dùng khi có planned maintenance hoặc DR drill

aws rds failover-global-cluster \
  --global-cluster-identifier prod-global-cluster \
  --target-db-cluster-identifier \
    arn:aws:rds:us-west-2:123456789:cluster:prod-cluster-secondary

# Theo dõi tiến trình failover
aws rds describe-global-clusters \
  --global-cluster-identifier prod-global-cluster \
  --query 'GlobalClusters[0].GlobalClusterMembers[*].{
    DBCluster: DBClusterArn,
    IsWriter: IsWriter,
    Status: GlobalWriteForwardingStatus
  }'
```

---

## 📊 DR Testing & Validation (Kiểm Tra & Xác Nhận DR)

### DR Drill Schedule (Lịch Diễn Tập DR)

```
Monthly DR Drill (Diễn Tập Hàng Tháng):
  - Test: Restore từ snapshot trong 2 giờ
  - Verify: Data integrity check (Kiểm tra tính toàn vẹn dữ liệu)
  - Measure: Thực tế RTO so với mục tiêu
  - Document: Kết quả và issues

Quarterly DR Drill (Diễn Tập Hàng Quý):
  - Test: Full failover sang DR region
  - Run: Application với DR database 30 phút
  - Test: Failback (Chuyển ngược lại region chính)
  - Verify: Không mất dữ liệu sau fail + failback

Annual DR Drill (Diễn Tập Hàng Năm):
  - Full production failover (nếu có thể, trong maintenance window)
  - Test all DR scenarios (tất cả kịch bản DR)
  - Review toàn bộ DR strategy
  - Update documentation
```

### DR Test Checklist

```markdown
## DR Drill Report — Quarterly Q2/2026

### Thông Tin
- Ngày: 2026-05-15
- Loại: Quarterly Failover Test
- Region chính: us-east-1
- DR Region: us-west-2
- DB: Aurora Global Cluster

### Pre-Drill Checks (Kiểm Tra Trước Diễn Tập)
- [x] Notify team và stakeholders 48 giờ trước
- [x] Verify DR region resources sẵn sàng
- [x] Kiểm tra latest snapshot trong DR region
- [x] Verify Route 53 health checks được cấu hình đúng
- [x] Backup current production state

### Failover Execution (Thực Hiện Chuyển Đổi Dự Phòng)
- [x] Trigger Aurora Global DB managed failover
- Start time: 10:00 AM UTC
- DB available ở DR region: 10:02 AM (+2 phút)  ✅ (RTO mục tiêu: 5 phút)
- App operational: 10:05 AM (+5 phút)           ✅

### RPO Measurement (Đo RPO)
- Last write ở primary: 09:59:58 AM
- First read after failover: 10:02:15 AM
- Dữ liệu cuối cùng trong DR: 09:59:58 AM
- RPO thực tế: < 1 giây                          ✅ (RPO mục tiêu: < 1 phút)

### Data Integrity Check
- [x] Row count match: production vs DR
- [x] Last transaction ID match
- [x] Application smoke test passed (kiểm tra cơ bản ứng dụng qua)

### Failback (Chuyển Ngược Lại Region Chính)
- Trigger failback: 10:35 AM
- Primary region operational: 10:37 AM (+2 phút) ✅
- Total drill time: 37 phút

### Issues Found
- Không có issues nghiêm trọng
- Minor: App connection pool (bộ nhớ đệm kết nối) cần 30 giây để warm up
  → Action: Implement connection pool pre-warming logic

### Next Drill
- Date: August 15, 2026 (Q3)
```

---

## 🔔 Route 53 Health Check & DNS Failover

### Cấu Hình DNS Failover Tự Động

```hcl
# Route 53 Health Check cho primary DB endpoint
resource "aws_route53_health_check" "primary_db" {
  fqdn              = aws_rds_cluster.primary.endpoint
  port              = 3306
  type              = "TCP"
  request_interval  = 30    # Check mỗi 30 giây
  failure_threshold = 3     # Fail sau 3 lần liên tiếp = 90 giây
  
  tags = { Name = "primary-db-health-check" }
}

# DNS record cho DB endpoint với failover routing
resource "aws_route53_record" "db_primary" {
  zone_id = var.hosted_zone_id
  name    = "db.myapp.internal"
  type    = "CNAME"
  ttl     = 60

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary_db.id
  records         = [aws_rds_cluster.primary.endpoint]
}

resource "aws_route53_record" "db_secondary" {
  zone_id = var.hosted_zone_id
  name    = "db.myapp.internal"
  type    = "CNAME"
  ttl     = 60

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"
  records        = [aws_rds_cluster.secondary.reader_endpoint]
}
```

---

## 💰 Chi Phí DR Theo Chiến Lược

```
Giả sử production DB: Aurora MySQL r6g.2xlarge, us-east-1
Base cost: ~$700/tháng

Backup & Restore:
  + Cross-region snapshot storage: ~$50/tháng
  Tổng DR cost: ~$50/tháng
  DR/Production ratio: ~7%

Pilot Light:
  + Cross-region replica t3.medium: ~$70/tháng
  + Cross-region data transfer: ~$30/tháng
  Tổng DR cost: ~$100/tháng
  DR/Production ratio: ~14%

Warm Standby:
  + Cross-region Aurora replica r6g.large: ~$350/tháng
  + App servers (scaled down): ~$150/tháng
  Tổng DR cost: ~$500/tháng
  DR/Production ratio: ~71%

Multi-Site Active-Active:
  + Full replica + app: ~$850/tháng (mỗi region thêm)
  Tổng DR cost: ~$850/tháng (per additional region)
  DR/Production ratio: ~121% (cao hơn production vì overhead)
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Disaster Recovery

**Q: Hãy giải thích 4 chiến lược DR của AWS?**
> Từ rẻ đến đắt và từ chậm đến nhanh: (1) Backup & Restore — chỉ giữ backup ở DR region, restore khi cần, RTO vài giờ. (2) Pilot Light — giữ một replica nhỏ ở DR region, promote và scale up khi cần, RTO ~1 giờ. (3) Warm Standby — giữ scaled-down version đang chạy ở DR region, chỉ cần scale up, RTO < 15 phút. (4) Multi-Site Active-Active — cả hai region đều đang chạy full capacity và nhận traffic, failover tự động < 1 phút.

**Q: Bạn thiết kế DR strategy cho e-commerce với RPO 5 phút, RTO 15 phút, budget $500/tháng, như thế nào?**
> Đề xuất Warm Standby với Aurora Global Database. Primary Aurora cluster ở us-east-1 với Global Database replicating sang us-west-2 với lag < 1 giây. Ở DR region duy trì một reader instance nhỏ hơn (r6g.large thay vì r6g.2xlarge). Khi DR, Aurora Global Managed Failover mất < 1 phút. RPO thực tế < 1 giây, RTO thực tế < 2 phút — vượt xa yêu cầu. Chi phí DR region ~$400/tháng, trong budget.

**Q: Tại sao phải test DR định kỳ? "Backup không được test là backup chưa tồn tại" nghĩa là gì?**
> DR plan trông tuyệt vời trên giấy nhưng thường thất bại khi thực hiện lần đầu trong tình huống khẩn cấp — do cấu hình thay đổi, script lỗi, permissions thiếu, hoặc dependencies không có ở DR region. Testing định kỳ đảm bảo bạn biết chính xác RTO/RPO thực tế (không phải lý thuyết), team biết quy trình, và tất cả automation scripts hoạt động. "Backup không test = backup chưa tồn tại" vì bạn không biết liệu backup có thực sự restore được cho đến khi thử.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
