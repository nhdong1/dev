# 1 — Aurora Architecture — Kiến Trúc Aurora

> Aurora được xây dựng trên triết lý "the log is the database" — thay vì truyền toàn bộ data pages giữa các nodes, Aurora chỉ truyền redo log records (bản ghi nhật ký ghi lại) và để distributed storage layer (lớp lưu trữ phân tán) tự dựng lại dữ liệu. Kiến trúc này loại bỏ phần lớn network I/O giữa compute và storage.

## 📚 Mục Lục

1. [Shared Storage Volume — Ổ Lưu Trữ Dùng Chung](#shared-storage-volume--ổ-lưu-trữ-dùng-chung)
2. [Aurora Cluster — Cụm Aurora](#aurora-cluster--cụm-aurora)
3. [Cluster Endpoints — Điểm Truy Cập Cụm](#cluster-endpoints--điểm-truy-cập-cụm)
4. [Write Path — Đường Ghi Dữ Liệu](#write-path--đường-ghi-dữ-liệu)
5. [Read Path — Đường Đọc Dữ Liệu](#read-path--đường-đọc-dữ-liệu)
6. [Aurora Storage Auto-Scaling](#aurora-storage-auto-scaling)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Shared Storage Volume — Ổ Lưu Trữ Dùng Chung

### Cấu Trúc 6-Way Replication (Sao Chép 6 Chiều)

```
                  Aurora Cluster Volume
    ┌─────────────────────────────────────────────────────┐
    │                                                      │
    │   us-east-1a          us-east-1b          us-east-1c│
    │   ┌──────────┐        ┌──────────┐        ┌────────┐│
    │   │ Segment 1│        │ Segment 3│        │Segment5││
    │   │ (Copy 1) │        │ (Copy 3) │        │(Copy 5)││
    │   ├──────────┤        ├──────────┤        ├────────┤│
    │   │ Segment 2│        │ Segment 4│        │Segment6││
    │   │ (Copy 2) │        │ (Copy 4) │        │(Copy 6)││
    │   └──────────┘        └──────────┘        └────────┘│
    │                                                      │
    │   Mỗi segment = 10 GB (Protection Group)            │
    └─────────────────────────────────────────────────────┘
              ▲                    ▲
              │                   │
        ┌─────┴────┐        ┌─────┴──────┐
        │  Writer  │        │  Reader(s) │
        │ Instance │        │  Instance  │
        └──────────┘        └────────────┘
```

### Quorum — Nguyên Tắc Đa Số

Aurora dùng **quorum-based writes và reads** (ghi và đọc dựa trên đa số):

| Hoạt Động                     | Cần Bao Nhiêu Nodes | Tại Sao                                          |
| ----------------------------- | ------------------- | ------------------------------------------------ |
| **Write** (Ghi)               | 4/6 thành công      | Đảm bảo durability (tính bền vững) — mất 1 AZ vẫn OK |
| **Read** (Đọc)                | 3/6 phản hồi        | Đảm bảo đọc được version mới nhất               |
| **Vẫn hoạt động khi mất**     | Cả 1 AZ (2 copies)  | 4 copies còn lại đủ để write và read            |

### Tính Toán Fault Tolerance (Khả Năng Chịu Lỗi)

```
6 copies, cần 4/6 để write:
  → Mất 2 copies → vẫn write được (4/6 còn lại)
  → Mất 1 AZ hoàn toàn (2 copies) → vẫn write được (4/6 còn lại)
  → Mất 3 copies → KHÔNG write được nhưng vẫn đọc được (3/6 cho read)
```

### Protection Groups (Nhóm Bảo Vệ)

Storage volume được chia thành các **protection groups** (nhóm bảo vệ) 10 GB:

- Mỗi protection group có 6 storage nodes (2 per AZ)
- Khi một storage node lỗi, Aurora tự sửa chữa từ các copies còn lại
- **Peer-to-peer repair** (sửa chữa ngang hàng): Storage nodes tự repair lẫn nhau, không tốn bandwidth của writer instance

---

## Aurora Cluster — Cụm Aurora

### Kiến Trúc Cluster Đầy Đủ

```
                    Aurora DB Cluster
    ┌─────────────────────────────────────────────────────────────┐
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │              Shared Cluster Volume                   │  │
    │  │          (6 copies, tự động tăng 10GB/lần)          │  │
    │  └───────────────────┬──────────────────────────────────┘  │
    │                      │ (tất cả instances đọc cùng volume)  │
    │                      │                                      │
    │  ┌───────────────────┼──────────────────────────┐          │
    │  │                   │                           │          │
    │  ▼                   ▼                           ▼          │
    │ ┌──────────┐    ┌──────────┐               ┌──────────┐   │
    │ │ Writer   │    │ Reader 1 │    . . .       │ Reader15 │   │
    │ │ (1 cái)  │    │          │               │  (tối đa)│   │
    │ └──────────┘    └──────────┘               └──────────┘   │
    │       ▲               ▲                          ▲          │
    │       │               │                          │          │
    │  Cluster           Reader                   Instance        │
    │  Endpoint          Endpoint                 Endpoint        │
    └─────────────────────────────────────────────────────────────┘
```

### Thành Phần Cluster

**Writer Instance (Máy Chủ Ghi):**
- Luôn có đúng 1 Writer trong cluster
- Xử lý tất cả INSERT, UPDATE, DELETE, DDL
- Khi Writer lỗi, Aurora tự promote (nâng cấp) một Reader thành Writer mới

**Reader Instances (Máy Chủ Đọc):**
- 0 đến 15 Reader instances
- Chỉ xử lý SELECT queries
- Chia sẻ cùng storage với Writer — không có replication lag thực sự (chỉ micro-lag ~mili-giây)
- Mỗi Reader trong AZ khác nhau cho HA tốt nhất

---

## Cluster Endpoints — Điểm Truy Cập Cụm

### Các Loại Endpoints

```
Cluster Endpoint (Điểm Truy Cập Cụm):
  mydb.cluster-xxxxxxxx.us-east-1.rds.amazonaws.com
  → Luôn trỏ đến Writer instance
  → Dùng cho write operations và read-write transactions
  → Tự động cập nhật khi có failover

Reader Endpoint (Điểm Truy Cập Đọc):
  mydb.cluster-ro-xxxxxxxx.us-east-1.rds.amazonaws.com
  → Load balance (Cân Bằng Tải) giữa tất cả Reader instances
  → Dùng cho read-only queries — scale out reads tự động
  → Nếu không có Reader, trỏ về Writer

Instance Endpoint (Điểm Truy Cập Máy Chủ):
  mydb.xxxxxxxx.us-east-1.rds.amazonaws.com  (Writer)
  mydb-reader-1.xxxxxxxx.us-east-1.rds.amazonaws.com (Reader 1)
  → Kết nối trực tiếp đến một instance cụ thể
  → Dùng khi cần kiểm soát routing chi tiết (debug, analytics query nặng)

Custom Endpoint (Điểm Truy Cập Tùy Chỉnh):
  mydb.cluster-custom-xxxxxxxx.us-east-1.rds.amazonaws.com
  → Tự định nghĩa nhóm instances
  → Ví dụ: nhóm large instances cho reports, nhóm small instances cho OLTP
```

### Khi Nào Dùng Endpoint Nào

| Trường Hợp Sử Dụng                                     | Endpoint Phù Hợp             |
| ------------------------------------------------------- | ----------------------------- |
| Ứng dụng chính — write + read                          | Cluster Endpoint              |
| Read-heavy workload — scaling reads                    | Reader Endpoint               |
| Reporting / analytics queries nặng                     | Custom Endpoint (large instances) |
| Debug một instance cụ thể                              | Instance Endpoint             |
| Blue/Green deployment (Triển Khai Xanh/Xanh)          | Custom Endpoint               |

---

## Write Path — Đường Ghi Dữ Liệu

### Quy Trình Ghi Truyền Thống (RDS)

```
1. Application gửi INSERT/UPDATE
2. Writer ghi vào buffer pool (bộ nhớ đệm)
3. Writer ghi redo log vào disk
4. Writer ghi data pages vào disk
5. Writer gửi binlog đến Standby và Read Replicas
6. Standby/Replicas replay binlog
7. Xác nhận trả về application
   → Nhiều I/O, latency cao
```

### Quy Trình Ghi Aurora

```
1. Application gửi INSERT/UPDATE
2. Writer ghi vào buffer pool
3. Writer tạo redo log records
4. Writer gửi log records đến 6 storage nodes
   (chỉ log — KHÔNG gửi data pages)
5. Đợi 4/6 storage nodes xác nhận (quorum write)
6. Xác nhận trả về application
   → Storage nodes tự áp dụng log và tạo data pages
   → Network I/O giảm ~7.7 lần so với RDS
```

### Minh Họa Giảm I/O

```
RDS MySQL ghi 1 transaction:
  Data page  = ~16 KB
  Redo log   = ~0.1–1 KB
  Binlog     = ~0.1–1 KB
  Tổng gửi đi cho replication: ~16–18 KB per transaction

Aurora ghi 1 transaction:
  Redo log   = ~0.1–1 KB
  Tổng gửi đi cho replication: ~0.1–1 KB per transaction
  → Tiết kiệm ~15–17 KB (giảm ~7.7 lần)
```

---

## Read Path — Đường Đọc Dữ Liệu

### Reader Instance Đọc Dữ Liệu Như Thế Nào

Reader instances không nhận data từ Writer. Thay vào đó:

1. Reader nhận **page cache notifications** (thông báo cache trang) từ storage
2. Reader gọi trực tiếp storage để lấy data pages
3. Storage nodes trả về page đã được áp dụng redo log mới nhất
4. Reader cache page vào buffer pool của mình

```
Writer:      Ghi redo log → Storage nodes
                              ↓
Storage:     Áp dụng log → Data pages sẵn sàng
                              ↓
Readers:     Đọc data pages từ Storage → Cache vào buffer pool
             (Không cần nhận gì từ Writer)
```

**Kết quả:** Lag chỉ là thời gian storage nodes áp dụng log — thường ~10–20 mili-giây.

### Isolation Levels (Mức Cách Ly) Đặc Biệt

Aurora MySQL Reader dùng **read view snapshot isolation** (cô lập ảnh chụp theo tầm nhìn đọc):

- Mỗi query nhìn thấy snapshot tại thời điểm query bắt đầu
- Đảm bảo consistent reads (đọc nhất quán) ngay cả khi Writer đang ghi
- Sử dụng **MVCC** (Multi-Version Concurrency Control — Kiểm Soát Đồng Thời Đa Phiên Bản)

---

## Aurora Storage Auto-Scaling

### Cơ Chế Tự Động Tăng

```
Storage tự động tăng — KHÔNG cần can thiệp thủ công:

  Bắt đầu:     10 GB (tối thiểu)
  Tăng thêm:   Theo 10 GB increments khi cần
  Tối đa:      128 TB
  
  Chi phí: Trả theo GB thực sự dùng ($0.10/GB-month thay đổi theo region)
           Không trả cho storage đã provision nhưng chưa dùng
```

### So Sánh Với RDS Storage

| Đặc Điểm                                        | RDS (EBS)                    | Aurora                        |
| ----------------------------------------------- | ---------------------------- | ----------------------------- |
| **Khởi tạo**                                    | Chọn size khi tạo            | Tự động từ 10 GB              |
| **Tăng storage**                                | Modify instance, downtime    | Tự động — không downtime      |
| **Giảm storage**                                | KHÔNG giảm được              | KHÔNG giảm được               |
| **Chi phí**                                     | Trả cho size đã provision    | Trả cho storage thực sự dùng  |
| **Tối đa**                                      | 64 TB                        | 128 TB                        |
| **Replication**                                 | 1 bản (EBS Multi-AZ: 2 bản) | 6 bản mặc định                |

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích "log is the database" trong Aurora

**Trả lời:** Trong database truyền thống, để ghi dữ liệu bạn cần ghi cả data pages và redo log. Aurora thay đổi điều này: Writer chỉ gửi redo log records đến storage layer, và storage layer tự áp dụng log để tạo data pages khi cần. Điều này giảm network I/O ~7.7 lần so với MySQL truyền thống. Storage trở thành database thực sự — nó lưu trữ, áp dụng log, và phục vụ data pages. Compute layer (Writer/Readers) chỉ là processing tier (lớp xử lý).

### Q2: Tại sao Aurora cần 4/6 storage nodes cho write?

**Trả lời:** Đây là quorum write để đảm bảo durability. Với 6 copies trên 3 AZs (2 per AZ), cần 4/6 để chịu được mất một AZ hoàn toàn (2 copies) và vẫn có ít nhất 4 copies xác nhận. Nếu chỉ cần 3/6, mất 1 AZ sẽ chỉ còn 4 copies nhưng chưa biết 3 cái nào xác nhận — không đủ guarantee. 4/6 đảm bảo mọi write được ghi trên ít nhất 2 AZs.

### Q3: Custom Endpoint dùng để làm gì?

**Trả lời:** Custom Endpoint cho phép chia nhỏ nhóm Readers cho các workload khác nhau. Ví dụ: tạo một custom endpoint "analytics" trỏ đến các r5.4xlarge instances dành cho report queries nặng, và một custom endpoint "app" trỏ đến các r5.large instances cho OLTP queries. Điều này tránh việc analytics queries tiêu thụ CPU của instances đang phục vụ user-facing traffic.

### Q4: Aurora Reader Endpoint có đảm bảo load balance tuyệt đối không?

**Trả lời:** Không hoàn toàn. Reader Endpoint dùng DNS-based load balancing (cân bằng tải qua DNS) — nó chọn Reader ngẫu nhiên khi thiết lập connection mới, không round-robin (luân phiên) từng query. Nếu application dùng connection pool (gộp kết nối), các connections sẽ được phân phối khi tạo mới, nhưng một connection hiện có luôn trỏ đến một Reader cố định. Để load balance tốt hơn, dùng RDS Proxy (Proxy RDS) trước Aurora.

---

## 🔗 Điều Hướng

| Trước                            | Tiếp Theo                                      |
| -------------------------------- | ---------------------------------------------- |
| [README.md](./README.md)         | [2-aurora-serverless.md](./2-aurora-serverless.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
