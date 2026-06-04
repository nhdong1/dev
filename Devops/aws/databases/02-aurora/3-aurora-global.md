# 3 — Aurora Global Database — Cơ Sở Dữ Liệu Toàn Cầu

> Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu Aurora) cho phép một Aurora cluster trải rộng qua nhiều AWS Regions (vùng địa lý) với replication lag (độ trễ sao chép) dưới 1 giây, hỗ trợ disaster recovery (khôi phục thảm họa) xuyên vùng và cho phép người dùng toàn cầu đọc từ region gần nhất.

## 📚 Mục Lục

1. [Kiến Trúc Global Database](#kiến-trúc-global-database)
2. [Storage-Level Replication — Sao Chép Tầng Lưu Trữ](#storage-level-replication--sao-chép-tầng-lưu-trữ)
3. [Primary vs Secondary Regions](#primary-vs-secondary-regions)
4. [Failover Xuyên Vùng — Cross-Region Failover](#failover-xuyên-vùng--cross-region-failover)
5. [Write Forwarding — Chuyển Tiếp Ghi](#write-forwarding--chuyển-tiếp-ghi)
6. [Use Cases — Trường Hợp Sử Dụng](#use-cases--trường-hợp-sử-dụng)
7. [Giới Hạn và Lưu Ý](#giới-hạn-và-lưu-ý)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Global Database

### Tổng Quan Kiến Trúc

```
Aurora Global Database
┌────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  PRIMARY REGION (Vùng Chính) — us-east-1                          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                  Aurora Cluster (Primary)                    │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐  │  │
│  │  │ Writer   │  │ Reader 1 │  │    Shared Storage Vol     │  │  │
│  │  │ instance │  │ instance │  │  (6 copies, 3 AZs)        │  │  │
│  │  └──────────┘  └──────────┘  └──────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              │ Storage-level replication           │
│                              │ (< 1 giây typical lag)              │
│                              │                                      │
│  SECONDARY REGION 1 (Vùng Phụ 1) — eu-west-1                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                  Aurora Cluster (Secondary)                  │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐  │  │
│  │  │ Reader   │  │ Reader   │  │    Shared Storage Vol     │  │  │
│  │  │ (có thể  │  │          │  │  (6 copies — replication  │  │  │
│  │  │ promote) │  │          │  │   từ Primary)             │  │  │
│  │  └──────────┘  └──────────┘  └──────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│  SECONDARY REGION 2 (Vùng Phụ 2) — ap-southeast-1                │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                  Aurora Cluster (Secondary)                  │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘

Tối đa: 1 Primary + 5 Secondary regions
```

### Đặc Điểm Chính

| Thuộc Tính                                              | Giá Trị                           |
| ------------------------------------------------------- | ---------------------------------- |
| **Số Secondary Regions**                                | Tối đa 5                          |
| **Replication lag** (Độ Trễ Sao Chép)                  | < 1 giây (typical ~100–200 ms)    |
| **RTO** (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục) | < 1 phút cho planned failover   |
| **RPO** (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) | Gần như 0 (< 1 giây dữ liệu có thể mất) |
| **Engines tương thích**                                 | Aurora MySQL, Aurora PostgreSQL   |
| **Hỗ trợ Serverless v2**                               | Có                                |

---

## Storage-Level Replication — Sao Chép Tầng Lưu Trữ

### Tại Sao Nhanh Hơn Binlog Replication

Khác với RDS Read Replicas dùng binlog replication (sao chép nhật ký nhị phân) — phải parse log, replay từng transaction — Aurora Global Database dùng **storage-level replication** (sao chép tầng lưu trữ):

```
RDS Cross-Region Read Replica:
  Primary → ghi data → tạo binlog → truyền binlog qua internet
         → Secondary phải parse và replay từng transaction
         Lag: nhiều giây đến phút

Aurora Global Database:
  Primary storage nodes → sao chép storage redo log records
         → Secondary storage nodes nhận và áp dụng
         Lag: < 1 giây (thường 100–200 ms tùy khoảng cách)
```

### Cơ Chế Hoạt Động Chi Tiết

```
1. Writer (us-east-1) nhận transaction
2. Writer gửi redo log đến Primary storage nodes (4/6 quorum)
3. Primary storage nodes XÁC NHẬN write trả về Writer
4. [Song song] Primary storage nodes STREAM log đến Secondary storage nodes
5. Secondary storage nodes áp dụng log — không cần Primary confirm
6. Secondary Reader instances đọc dữ liệu đã được cập nhật

→ Bước 4 là ASYNC (bất đồng bộ) — Primary không chờ Secondary
→ Điều này nghĩa là Secondary có thể lag tối đa ~1 giây
```

---

## Primary vs Secondary Regions

### Primary Region (Vùng Chính)

- **Có đúng 1 Writer instance** và tối đa 15 Reader instances
- Xử lý **tất cả write operations**
- Cluster endpoint trỏ vào Writer
- Nếu toàn bộ Primary region lỗi → phải manually promote (thủ công nâng cấp) Secondary

### Secondary Regions (Vùng Phụ)

- **Chỉ có Reader instances** (mặc định — không có Writer)
- Nhận replication từ Primary qua storage-level replication
- Users trong region đó có thể **đọc local** (đọc cục bộ) với latency thấp
- Có thể được **promoted thành Primary** khi cần disaster recovery

```
Tình huống đọc tối ưu:
  User tại Frankfurt → kết nối đến eu-west-1 Secondary → đọc local
  Latency: ~1–5 ms thay vì ~100 ms nếu đọc từ us-east-1

Tình huống ghi:
  User tại Frankfurt → vẫn phải ghi đến us-east-1 Primary
  Latency: ~100 ms (round-trip transatlantic)
  → Hoặc dùng Write Forwarding để ẩn điều này với ứng dụng
```

---

## Failover Xuyên Vùng — Cross-Region Failover

### Các Loại Failover

**Planned Failover** (Chuyển Đổi Dự Phòng Có Kế Hoạch) — khi bảo trì hoặc di chuyển:
```
1. Dừng writes trên Primary
2. Đợi Secondary đuổi kịp lag (~giây)
3. Promote Secondary thành Primary mới
4. Primary cũ trở thành Secondary
RTO: < 1 phút — không mất dữ liệu
```

**Unplanned Failover** (Chuyển Đổi Dự Phòng Khẩn Cấp) — khi Primary region lỗi:
```
1. Phát hiện Primary không phản hồi
2. Chọn Secondary region để promote (thủ công hoặc tự động*)
3. Detach (tách) Secondary khỏi Global Database
4. Promote thành standalone cluster rồi thành Primary mới
RTO: ~1 phút
RPO: Phụ thuộc lag tại thời điểm lỗi (thường < 1 giây)

*Managed Planned Failover được hỗ trợ; full automatic failover cần Aurora Global Database với RDS Global Cluster
```

### Switchover vs Failover

| Thuật Ngữ                              | Khi Dùng                                | Dữ Liệu Mất  | Thời Gian   |
| --------------------------------------- | --------------------------------------- | ------------ | ----------- |
| **Switchover** (Chuyển Đổi)            | Có kế hoạch — maintenance, migration   | Không        | ~1 phút     |
| **Failover** (Chuyển Đổi Dự Phòng)    | Khẩn cấp — region lỗi                  | < 1 giây     | 1–2 phút    |
| **Detach + Promote** (Tách + Nâng Cấp) | Disaster Recovery khẩn cấp             | Có thể mất   | 1–2 phút    |

---

## Write Forwarding — Chuyển Tiếp Ghi

### Tính Năng Write Forwarding

**Write Forwarding** (Chuyển Tiếp Ghi) cho phép ứng dụng kết nối đến Secondary region **gửi write queries** — Aurora tự động forward (chuyển tiếp) về Primary region và trả kết quả:

```
Không có Write Forwarding:
  App (Frankfurt) → phải tự detect write vs read
                  → writes: kết nối đến us-east-1 Primary
                  → reads: kết nối đến eu-west-1 Secondary

Với Write Forwarding:
  App (Frankfurt) → kết nối một endpoint duy nhất (eu-west-1)
                  → Aurora tự forward writes đến us-east-1
                  → reads được xử lý local tại eu-west-1
```

### Giới Hạn Write Forwarding

```
Không hỗ trợ:
  - DDL statements (CREATE, ALTER, DROP) — phải chạy trực tiếp trên Primary
  - XA transactions (Giao Dịch XA)
  - Stored procedures với side effects trên Primary
  
Latency tăng thêm:
  - Write forwarded phải round-trip về Primary → latency = RTT (Round-Trip Time) giữa 2 regions
  - Frankfurt → us-east-1: ~100 ms extra per write
  - Phù hợp cho read-heavy apps với occasional writes (ghi không thường xuyên)
```

---

## Use Cases — Trường Hợp Sử Dụng

### 1. Global Read Performance (Hiệu Năng Đọc Toàn Cầu)

```
Ứng dụng có users ở nhiều lục địa:
  - Primary: us-east-1 (New York) — xử lý writes
  - Secondary: eu-west-1 (Ireland) — phục vụ European users đọc
  - Secondary: ap-southeast-1 (Singapore) — phục vụ Asian users đọc
  
Lợi ích: Users đọc local, latency giảm từ 200ms xuống còn 5ms
```

### 2. Disaster Recovery (Khôi Phục Thảm Họa)

```
Yêu cầu business: RPO < 1 giây, RTO < 2 phút
Giải pháp: Aurora Global Database với 1 Secondary
  - Normal operation: Primary tại us-east-1
  - us-east-1 lỗi: Promote Secondary tại us-west-2
  - RTO: ~1 phút, RPO: < 1 giây lag
```

### 3. Blue/Green Cross-Region Migration

```
Di chuyển traffic từ us-east-1 sang eu-central-1 không downtime:
  1. Tạo Global Database với eu-central-1 là Secondary
  2. Đợi Secondary catch up hoàn toàn
  3. Switchover: eu-central-1 trở thành Primary
  4. Cập nhật DNS để users kết nối eu-central-1
  5. Remove us-east-1 khỏi Global Database
```

---

## Giới Hạn và Lưu Ý

### Giới Hạn Kỹ Thuật

```
- Secondary regions là read-only (chỉ đọc) mặc định
- Không hỗ trợ cross-region transactions — mỗi transaction chỉ commit trên một region
- Không hỗ trợ RDS Proxy cho Global Database writer endpoint
- DDL không thể dùng Write Forwarding — phải kết nối trực tiếp Primary
- Maximum 5 Secondary regions
```

### Chi Phí Global Database

```
Chi phí thêm ngoài chi phí cluster thông thường:
  - Replication I/O: $0.20/1M I/O replicated
  - Cross-region data transfer: $0.02–0.09/GB (phụ thuộc regions)
  
Ví dụ 1 TB data replicated/tháng:
  - I/O: ~10M I/Os × $0.20/M = $2
  - Data transfer: 1000 GB × $0.02 = $20
  Tổng thêm: ~$22/tháng ngoài chi phí cluster
```

### Replication Lag Trong Thực Tế

```
Yếu tố ảnh hưởng lag:
  - Khoảng cách vật lý giữa regions (ảnh hưởng lớn nhất)
  - Khối lượng writes trên Primary
  - Network conditions giữa AWS regions
  
Typical values:
  us-east-1 → eu-west-1: ~50–100 ms
  us-east-1 → ap-southeast-1: ~150–200 ms
  us-east-1 → ap-northeast-1 (Tokyo): ~120–180 ms
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Aurora Global Database khác RDS Cross-Region Read Replica như thế nào?

**Trả lời:** Hai điểm khác biệt chính: (1) **Cơ chế replication** — RDS dùng binlog replication (bất đồng bộ, thường lag giây đến phút), Aurora Global Database dùng storage-level replication (lag < 1 giây). (2) **Failover capability** — RDS Cross-Region Replica cần thời gian dài để promote và không đảm bảo RPO thấp; Aurora Global Database có thể promote Secondary thành Primary trong ~1 phút với RPO < 1 giây, phù hợp cho DR thực sự.

### Q2: Khi Primary region bị lỗi hoàn toàn, bạn làm gì với Aurora Global Database?

**Trả lời:** Quy trình là: (1) Phát hiện Primary region lỗi qua CloudWatch alarms; (2) Vào AWS Console hoặc dùng CLI, chọn Secondary cluster cần promote; (3) Thực hiện "Promote" — AWS tách cluster ra khỏi Global Database và nâng thành standalone Primary; (4) Cập nhật application connection strings hoặc DNS để trỏ vào cluster mới. Toàn bộ quá trình tốn ~1–2 phút. Dữ liệu có thể mất tối đa bằng lag tại thời điểm lỗi (thường < 1 giây).

### Q3: Write Forwarding có thể thay thế hoàn toàn việc kết nối trực tiếp Primary không?

**Trả lời:** Không. Write Forwarding chỉ forward DML (Data Manipulation Language — Ngôn Ngữ Thao Tác Dữ Liệu) như INSERT, UPDATE, DELETE. DDL (CREATE, ALTER, DROP) phải chạy trực tiếp trên Primary cluster. Ngoài ra, Write Forwarding thêm latency bằng round-trip time giữa Secondary và Primary — không phù hợp cho applications cần write latency thấp. Dùng Write Forwarding chủ yếu để đơn giản hóa connection logic trong ứng dụng read-heavy với occasional writes.

### Q4: Với RPO = 0, bạn có thể đạt được với Aurora Global Database không?

**Trả lời:** Không hoàn toàn. Aurora Global Database có replication bất đồng bộ — Primary xác nhận write mà không chờ Secondary. Do đó luôn có khả năng mất dữ liệu < 1 giây nếu Primary region lỗi. Để đạt RPO = 0 thực sự, bạn cần synchronous replication (sao chép đồng bộ) — nhưng điều này sẽ tăng write latency đáng kể (thêm 100–200 ms per write cho transatlantic). Trong thực tế, RPO < 1 giây thường được chấp nhận cho hầu hết business requirements.

---

## 🔗 Điều Hướng

| Trước                                              | Tiếp Theo                                    |
| -------------------------------------------------- | -------------------------------------------- |
| [2-aurora-serverless.md](./2-aurora-serverless.md) | [4-aurora-vs-rds.md](./4-aurora-vs-rds.md)  |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
