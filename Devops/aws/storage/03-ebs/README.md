# EBS — Elastic Block Store — Lưu Trữ Khối Linh Hoạt

> EBS (Elastic Block Store — Lưu Trữ Khối Linh Hoạt) là dịch vụ block storage (lưu trữ khối) của AWS được thiết kế cho Amazon EC2 (Elastic Compute Cloud — Điện Toán Đám Mây Linh Hoạt). EBS cung cấp lưu trữ bền vững, hiệu suất cao và độ trễ thấp cho các workload (tải công việc) đòi hỏi khắt khe như database, ứng dụng enterprise và hệ điều hành.

---

## 📚 Mục Lục Module

| File | Nội Dung | Độ Ưu Tiên |
|------|----------|------------|
| [1-volume-types.md](./1-volume-types.md) | gp2/gp3, io1/io2, st1, sc1 — so sánh chi tiết | ⭐⭐⭐ |
| [2-snapshots-and-lifecycle.md](./2-snapshots-and-lifecycle.md) | Snapshot, DLM — Data Lifecycle Manager | ⭐⭐⭐ |
| [3-ebs-multi-attach.md](./3-ebs-multi-attach.md) | Multi-Attach — Gắn nhiều EC2 cùng lúc | ⭐⭐ |
| [4-performance-tuning.md](./4-performance-tuning.md) | IOPS, Throughput, Latency — tối ưu hiệu suất | ⭐⭐⭐ |
| [5-encryption-with-kms.md](./5-encryption-with-kms.md) | Mã hóa với KMS — Key Management Service | ⭐⭐⭐ |
| [6-ebs-vs-instance-store.md](./6-ebs-vs-instance-store.md) | EBS vs Instance Store — bền vững vs tạm thời | ⭐⭐⭐ |

---

## 🧠 EBS Là Gì?

EBS là **network-attached block storage** (lưu trữ khối gắn qua mạng) — hoạt động như ổ đĩa cứng vật lý nhưng được cung cấp qua mạng nội bộ AWS với độ trễ cực thấp (thường < 1ms).

### Đặc Điểm Cốt Lõi

```
┌─────────────────────────────────────────────────────────┐
│                    EBS Volume                           │
│                                                         │
│  ✅ Persistent  — Dữ liệu tồn tại khi EC2 dừng/khởi lại│
│  ✅ Replicated  — Tự nhân bản trong 1 Availability Zone │
│  ✅ Snapshots   — Sao lưu lên S3 bất cứ lúc nào         │
│  ✅ Encryption  — Mã hóa AES-256 với KMS                │
│  ✅ Resizable   — Tăng dung lượng không cần downtime     │
│  ❌ AZ-bound    — Chỉ dùng trong 1 AZ (trừ Multi-Attach) │
└─────────────────────────────────────────────────────────┘
```

### Cách EBS Kết Nối Với EC2

```
┌──────────────┐     NVMe/EBS Network      ┌──────────────┐
│  EC2 Instance│ ◄────────────────────────► │  EBS Volume  │
│  (us-east-1a)│                            │  (us-east-1a)│
└──────────────┘                            └──────────────┘
      │
      │  aws ebs attach-volume
      ▼
  Xuất hiện như /dev/xvda hoặc /dev/nvme0n1
  → mkfs, mount, dùng như ổ đĩa thông thường
```

---

## 📊 Tổng Quan Volume Types (Loại Volume)

| Loại | Công Nghệ | IOPS Tối Đa | Throughput Tối Đa | Use Case Chính |
|------|-----------|-------------|-------------------|----------------|
| **gp3** | SSD | 16.000 | 1.000 MB/s | Boot volume, ứng dụng chung |
| **gp2** | SSD | 16.000 | 250 MB/s | (Legacy) Boot volume |
| **io2 Block Express** | SSD NVMe | 256.000 | 4.000 MB/s | Database hiệu suất cao |
| **io1** | SSD | 64.000 | 1.000 MB/s | Database I/O cao |
| **st1** | HDD | 500 | 500 MB/s | Data warehouse, log |
| **sc1** | HDD | 250 | 250 MB/s | Cold data, lưu trữ rẻ |

> Chi tiết đầy đủ tại [1-volume-types.md](./1-volume-types.md)

---

## 🎯 Khi Nào Chọn EBS?

### Chọn EBS Khi

- Cần **persistent storage** (lưu trữ bền vững) cho EC2
- Ứng dụng yêu cầu **low latency** (độ trễ thấp < 1ms)
- **Database workload**: MySQL, PostgreSQL, Oracle, MongoDB
- **Boot volume** (volume khởi động) cho EC2 instance
- Cần **Snapshots** cho backup và DR (Disaster Recovery — Phục Hồi Thảm Họa)
- Dữ liệu không cần chia sẻ giữa nhiều EC2 đồng thời (trừ Multi-Attach)

### Không Chọn EBS Khi

- Cần **shared storage** cho nhiều EC2 đồng thời → Dùng **EFS**
- Dữ liệu lớn không cấu trúc, truy cập qua internet → Dùng **S3**
- Cần lưu trữ tạm thời tốc độ cao, chấp nhận mất dữ liệu → Dùng **Instance Store**
- Ứng dụng multi-AZ cần shared filesystem → Dùng **EFS** hoặc **FSx**

---

## 📐 Kiến Trúc EBS Trong Thực Tế

### Ứng Dụng Web Điển Hình

```
                    ┌─────────────────────────────┐
                    │        EC2 Instance          │
                    │  ┌──────────┐ ┌───────────┐  │
                    │  │ Root Vol │ │ Data Vol  │  │
                    │  │  gp3 8GB │ │  gp3 100GB│  │
                    │  │ /dev/sda │ │ /dev/sdb  │  │
                    │  └──────────┘ └───────────┘  │
                    └─────────────────────────────┘
                           │              │
               ┌───────────┘              └────────────┐
               ▼                                       ▼
       ┌──────────────┐                       ┌──────────────┐
       │ EBS gp3      │                       │ EBS gp3      │
       │ Root Volume  │                       │ Data Volume  │
       │ OS + App     │                       │ Database     │
       └──────────────┘                       └──────────────┘
               │                                       │
               └──────────────┬────────────────────────┘
                              ▼
                    ┌──────────────────┐
                    │  EBS Snapshots   │
                    │  (Stored in S3)  │
                    └──────────────────┘
```

### Database Production Pattern

```
                    ┌─────────────────────────────┐
                    │        RDS / EC2 DB          │
                    │                             │
                    │  ┌──────────────────────┐   │
                    │  │   io2 Block Express  │   │
                    │  │   500GB, 40.000 IOPS │   │
                    │  │   Encrypted with KMS │   │
                    │  └──────────────────────┘   │
                    └─────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Daily Snapshots   │
                    │   DLM Policy:       │
                    │   Retain 30 days    │
                    └─────────────────────┘
```

---

## 🔑 Khái Niệm Quan Trọng

### IOPS — Input/Output Operations Per Second (Thao Tác Đọc/Ghi Mỗi Giây)

- Đo số lượng thao tác I/O (đọc hoặc ghi) thực hiện được trong 1 giây
- IOPS cao → phù hợp với database (nhiều transaction nhỏ)
- Mỗi I/O của EBS = 16KB (block size — kích thước khối)

### Throughput (Thông Lượng)

- Đo lượng dữ liệu truyền được trong 1 giây (MB/s)
- Throughput cao → phù hợp với streaming, log, analytics (file lớn tuần tự)

### Latency (Độ Trễ)

- Thời gian hoàn thành 1 thao tác I/O
- EBS SSD: thường < 1ms
- EBS HDD: 2–5ms

### Durability (Độ Bền)

- EBS tự động nhân bản dữ liệu trong cùng AZ (Availability Zone — Vùng Khả Dụng)
- io2: 99.999% durability (5 chín) — cao nhất
- Snapshot lưu trên S3: 99.999999999% (11 chín)

---

## 💡 Tips Cho Phỏng Vấn

### Câu Hỏi Phổ Biến

**Q: Sự khác biệt giữa gp2 và gp3?**
> gp3 tốt hơn hoàn toàn: IOPS và Throughput độc lập với dung lượng, rẻ hơn 20%. Không có lý do dùng gp2 mới.

**Q: Khi nào chọn io2 thay vì gp3?**
> Khi cần > 16.000 IOPS hoặc cần Multi-Attach (Gắn Nhiều EC2) hoặc cần SLA 99.999% durability.

**Q: EBS snapshot hoạt động như thế nào?**
> Snapshot đầu tiên là full copy, các snapshot sau là incremental (chỉ lưu phần thay đổi). Tất cả snapshot đều lưu trên S3.

**Q: Tại sao EBS chỉ dùng được trong 1 AZ?**
> EBS nhân bản dữ liệu trong AZ để đảm bảo hiệu suất và độ trễ thấp. Để di chuyển sang AZ khác cần tạo Snapshot rồi restore.

---

## 📈 Giới Hạn Quan Trọng

| Giới Hạn | Giá Trị |
|----------|---------|
| Volume size tối đa | 64 TB (io2 Block Express) |
| IOPS tối đa / volume | 256.000 (io2 Block Express) |
| Throughput tối đa / volume | 4.000 MB/s (io2 Block Express) |
| Số volume / EC2 instance | 28 volumes |
| Snapshot tối đa | 100.000 / account |
| Volume có thể resize | Có (chỉ tăng, không giảm) |

---

## 🔗 Điều Hướng

- **Trước:** [02-s3-advanced/](../02-s3-advanced/) — S3 Nâng Cao
- **Tiếp theo:** [04-efs/](../04-efs/) — EFS — Elastic File System
- **Liên quan:** [05-security/](../05-security/) — Bảo Mật Lưu Trữ
- **Tham khảo:** [08-disaster-recovery/](../08-disaster-recovery/) — Disaster Recovery

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn thành
