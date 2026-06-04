# 11. Cost Optimization — Tối Ưu Chi Phí Database Trên AWS

> Hướng dẫn toàn diện về tối ưu chi phí cho các dịch vụ cơ sở dữ liệu AWS — từ Reserved Instances (Máy Chủ Đặt Trước), Right-sizing (Định Cỡ Phù Hợp) đến Storage Tiering (Phân Tầng Lưu Trữ) và lựa chọn pricing model (mô hình định giá) phù hợp.

## 📚 Mục Lục

1. [Tổng Quan — Tại Sao Cost Optimization Quan Trọng](#tổng-quan)
2. [Cấu Trúc Chi Phí AWS Database](#cấu-trúc-chi-phí)
3. [Công Cụ Quản Lý Chi Phí](#công-cụ-quản-lý-chi-phí)
4. [Các File Trong Section Này](#các-file-trong-section-này)
5. [Framework Tối Ưu Chi Phí](#framework-tối-ưu-chi-phí)
6. [Quick Wins — Tiết Kiệm Nhanh](#quick-wins)

---

## Tổng Quan

Chi phí database thường chiếm **30-50% tổng chi phí AWS** của một hệ thống production. Tối ưu chi phí không có nghĩa là cắt giảm chất lượng — mà là **chi đúng chỗ, đúng lúc, đúng lượng**.

### Ba Nguyên Tắc Cốt Lõi

```
1. Right-size  — Dùng đúng kích thước tài nguyên cần thiết
2. Right-type  — Chọn đúng loại dịch vụ/pricing model
3. Right-time  — Tắt/thu nhỏ những gì không dùng ngoài giờ cao điểm
```

### Chi Phí Ẩn Thường Gặp

| Hạng Mục                                | Mức Độ Ảnh Hưởng | Nhiều Người Bỏ Qua? |
| --------------------------------------- | ---------------- | ------------------- |
| Data transfer (Truyền Tải Dữ Liệu)      | Trung bình       | Có                  |
| Snapshot storage (Lưu Trữ Snapshot)     | Thấp-Trung bình  | Có                  |
| Multi-AZ standby (Dự Phòng Đa Vùng)    | Cao (x2 chi phí) | Không               |
| Aurora I/O operations (Thao Tác I/O)   | Rất cao          | Có                  |
| DynamoDB GSI (Chỉ Mục Phụ Toàn Cầu)   | Trung bình       | Có                  |
| Idle instances (Máy Chủ Nhàn Rỗi)      | Cao              | Có                  |

---

## Cấu Trúc Chi Phí

### RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ)

```
Chi Phí RDS = Instance + Storage + I/O + Backup + Data Transfer

Instance:      On-Demand hoặc Reserved (1 hoặc 3 năm)
Storage:       gp2/gp3/io1/io2 theo GB-month
I/O:           Chỉ với magnetic storage (io1/io2 không tính theo request)
Backup:        Miễn phí đến = database size; tính phí vượt mức
Data Transfer: Ra ngoài AWS và cross-region
```

### Aurora

```
Chi Phí Aurora = Instance + Storage + I/O + Backup + Data Transfer

Storage:    Tính theo GB-month thực sự dùng (không cần pre-provision)
I/O:        Standard mode: $0.20/1M requests — CHÚ Ý đây có thể rất cao
            I/O-Optimized mode: Storage cao hơn 25% nhưng không tính I/O
Backup:     100% database size miễn phí; tính phí vượt mức
```

### DynamoDB

```
Chi Phí DynamoDB = Read/Write + Storage + Streams + Backups + DAX

On-Demand:   $1.25/1M WRU (Write Request Unit — Đơn Vị Ghi)
             $0.25/1M RRU (Read Request Unit — Đơn Vị Đọc)
Provisioned: $0.00065/WCU/hour + $0.00013/RCU/hour
Storage:     $0.25/GB-month (first 25 GB free)
DAX:         Theo node type — tương tự ElastiCache
```

### ElastiCache

```
Chi Phí ElastiCache = Node hours + Backup + Data Transfer

On-Demand:  Theo node type (cache.r7g.large ~$0.166/hour)
Reserved:   1 năm: ~40% tiết kiệm | 3 năm: ~60% tiết kiệm
Backup:     $0.085/GB-month cho Redis backup
```

---

## Công Cụ Quản Lý Chi Phí

### AWS Cost Explorer (Khám Phá Chi Phí AWS)

```
Dùng để:
- Xem chi phí theo dịch vụ, theo ngày/tuần/tháng
- Phân tích xu hướng chi phí
- Tạo budget alerts (cảnh báo ngân sách)
- Xem Reserved Instance utilization (mức sử dụng máy chủ đặt trước)
```

### AWS Trusted Advisor (Cố Vấn Tin Cậy AWS)

```
Kiểm tra tự động:
- Idle RDS instances (Máy chủ RDS nhàn rỗi)
- Underutilized instances (Máy chủ dùng dưới mức)
- Reserved Instance recommendations (Gợi ý đặt trước)
- Security & performance checks (Kiểm tra bảo mật & hiệu năng)
```

### AWS Compute Optimizer (Trình Tối Ưu Tính Toán)

```
Phân tích và gợi ý:
- RDS instance right-sizing
- Memory/CPU utilization trends (Xu hướng sử dụng bộ nhớ/CPU)
- Savings estimate (Ước tính tiết kiệm)
```

### CloudWatch Metrics + Cost Allocation Tags (Nhãn Phân Bổ Chi Phí)

```bash
# Gắn tags để theo dõi chi phí theo team/project
aws rds add-tags-to-resource \
  --resource-name arn:aws:rds:... \
  --tags Key=Project,Value=ecommerce Key=Team,Value=backend Key=Env,Value=prod
```

---

## Các File Trong Section Này

| File                  | Nội Dung                                               | Đọc Khi Nào                      |
| --------------------- | ------------------------------------------------------ | --------------------------------- |
| **README.md**         | Tổng quan, framework, quick wins                       | Bắt đầu từ đây                    |
| **1-rds-pricing.md**  | RDS pricing models, Reserved Instances, storage tiers  | Tối ưu chi phí RDS                |
| **2-aurora-cost.md**  | Aurora I/O, serverless cost model, I/O-Optimized       | Tối ưu chi phí Aurora             |
| **3-dynamodb-cost.md**| On-Demand vs Provisioned, auto-scaling, TTL            | Tối ưu chi phí DynamoDB           |
| **4-elasticache-cost.md** | Pricing tiers, Reserved Nodes, sizing             | Tối ưu chi phí ElastiCache        |
| **5-rightsizing.md**  | Instance sizing, storage optimization, monitoring      | Right-sizing tổng thể             |

---

## Framework Tối Ưu Chi Phí

### Bước 1: Đo Lường Trước (Measure First)

```
Không thể tối ưu những gì không đo được.

Actions:
- Bật Cost Allocation Tags cho tất cả tài nguyên
- Tạo Cost Explorer dashboard theo team/project
- Thiết lập budget alerts ($X/tháng)
- Xem Trusted Advisor recommendations hàng tuần
```

### Bước 2: Identify Waste — Xác Định Lãng Phí

```
Checklist phát hiện lãng phí:

□ RDS instances CPU < 10% liên tục → Downsize
□ RDS instances kết nối = 0 nhiều giờ → Idle, xem xét xóa
□ Aurora Standard với I/O > 1M requests/ngày → Đổi sang I/O-Optimized
□ DynamoDB Provisioned với utilization < 20% → Đổi sang On-Demand hoặc giảm capacity
□ ElastiCache nodes memory < 30% → Downsize
□ Snapshots cũ > 90 ngày không cần thiết → Xóa
□ Dev/test environments chạy 24/7 → Schedule tắt ngoài giờ
```

### Bước 3: Optimize Pricing — Tối Ưu Mô Hình Định Giá

```
Priority thực hiện:

1. Reserved Instances cho production workloads ổn định
   → Tiết kiệm 40-65% so với On-Demand

2. Aurora Serverless cho dev/test và workloads không liên tục
   → Chỉ trả khi dùng

3. DynamoDB On-Demand cho traffic không dự đoán được
   → Không bao giờ bị throttle, trả theo dùng

4. ElastiCache Reserved Nodes cho Redis/Memcached production
   → Tiết kiệm 40-60%
```

### Bước 4: Right-size — Định Cỡ Phù Hợp

```
Quy trình right-sizing:

1. Thu thập 2-4 tuần metrics (CPU, Memory, IOPS, Connections)
2. Xác định peak usage (mức sử dụng đỉnh điểm)
3. Target: peak CPU < 70%, Memory free > 20%
4. Downsize nếu average CPU < 20% và peak < 40%
5. Upsize nếu thường xuyên > 80% CPU hoặc memory pressure
```

### Bước 5: Architecture Review — Xem Lại Kiến Trúc

```
Câu hỏi kiến trúc để tiết kiệm chi phí:

□ Có thể dùng DynamoDB thay vì RDS cho use case này không?
   DynamoDB serverless = không trả khi không dùng

□ Có cần Multi-AZ (Đa Vùng Sẵn Sàng) cho môi trường này không?
   Dev/test không cần Multi-AZ → tiết kiệm 50%

□ Caching layer (Tầng Cache) có giảm được database load không?
   ElastiCache cache hit 90% → giảm 90% DynamoDB/RDS reads

□ Read Replicas có thực sự cần thiết không?
   Xem xét xóa unused replicas

□ Aurora I/O-Optimized có rẻ hơn Standard không?
   Breakeven ≈ $0.10/GB storage cost khi I/O cao
```

---

## Quick Wins

### Tiết Kiệm Ngay Lập Tức (< 1 ngày)

```bash
# 1. Xóa snapshots cũ hơn 90 ngày không cần thiết
aws rds describe-db-snapshots --query \
  'DBSnapshots[?SnapshotCreateTime<=`2025-01-01`].[DBSnapshotIdentifier]' \
  --output text

# 2. Tìm RDS instances không có kết nối 7 ngày
# Xem CloudWatch metric: DatabaseConnections < 1 for 7 days

# 3. Tắt Enhanced Monitoring (Giám Sát Nâng Cao) nếu không cần
# Enhanced Monitoring: 60-second granularity ~ $0.01/hour/instance
# Chuyển về 0 (tắt) hoặc 300 seconds nếu không cần chi tiết

# 4. Giảm backup retention period cho dev instances
aws rds modify-db-instance \
  --db-instance-identifier mydb-dev \
  --backup-retention-period 1  # Giảm từ 7 ngày xuống 1 ngày
```

### Tiết Kiệm Trong Tuần (< 1 tuần)

```
1. Mua Reserved Instances cho production databases
   → Tiết kiệm 40-65% ngay

2. Tắt dev/test databases ngoài giờ (tối 18h - 8h sáng + cuối tuần)
   → Chạy 40% thời gian thay vì 100% = tiết kiệm 60%

3. Bật Storage Auto Scaling (Tự Động Co Giãn Lưu Trữ)
   → Không over-provision storage

4. Xem xét Aurora Serverless v2 cho môi trường không liên tục
```

### Tiết Kiệm Dài Hạn (1-3 tháng)

```
1. Audit và right-size tất cả instances dựa trên 30-day metrics
2. Thiết kế lại DynamoDB capacity planning với auto-scaling
3. Đánh giá Aurora I/O-Optimized vs Standard cho production
4. Cân nhắc chuyển cold data sang S3 + Athena
5. Implement caching strategy để giảm database reads
```

---

## Bảng So Sánh Savings Potential (Tiềm Năng Tiết Kiệm)

| Chiến Lược                                           | Tiết Kiệm Tiềm Năng | Độ Khó | Rủi Ro |
| ---------------------------------------------------- | ------------------- | ------ | ------ |
| Reserved Instances 3 năm (Máy Chủ Đặt Trước)        | 60-65%              | Thấp   | Thấp   |
| Reserved Instances 1 năm                             | 40-45%              | Thấp   | Thấp   |
| Tắt dev/test ngoài giờ (Schedule On/Off)            | 50-65%              | Thấp   | Thấp   |
| Right-sizing instances (Định Cỡ Phù Hợp)            | 20-40%              | Trung  | Trung  |
| Aurora I/O-Optimized (workload I/O cao)              | 20-40%              | Thấp   | Thấp   |
| DynamoDB On-Demand → Provisioned (traffic dự đoán)  | 30-70%              | Trung  | Trung  |
| ElastiCache Reserved Nodes (Nút Đặt Trước)          | 40-60%              | Thấp   | Thấp   |
| Caching để giảm database load                       | Gián tiếp 30-50%   | Cao    | Trung  |
| Xóa unused snapshots và replicas                   | 5-15%               | Thấp   | Thấp   |

---

## Lộ Trình Đọc

```
Bắt đầu:    README.md (file này) — hiểu framework tổng thể
Tiếp theo:  1-rds-pricing.md    — nếu dùng RDS/MySQL/PostgreSQL
            2-aurora-cost.md    — nếu dùng Aurora
            3-dynamodb-cost.md  — nếu dùng DynamoDB
            4-elasticache-cost.md — nếu dùng ElastiCache/Redis
Cuối cùng:  5-rightsizing.md    — áp dụng right-sizing thực tế
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Section:** 11/12 — Cost Optimization
