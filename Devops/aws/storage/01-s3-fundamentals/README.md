# S3 Fundamentals — Kiến Thức Nền Tảng Amazon S3

> Amazon S3 — Simple Storage Service — Dịch vụ Lưu trữ Đối tượng của AWS, cung cấp độ bền 99.999999999% (11 chín) và khả năng mở rộng gần như vô hạn.

---

## Tổng Quan

Amazon S3 là dịch vụ **object storage** (lưu trữ đối tượng) được quản lý hoàn toàn bởi AWS. S3 là nền tảng của nhiều kiến trúc cloud-native — từ backup đơn giản đến data lake quy mô petabyte.

### Tại Sao S3 Quan Trọng?

| Đặc Điểm | Chi Tiết |
|-----------|---------|
| **Durability** (Độ bền) | 99.999999999% — dữ liệu được lưu trên ít nhất 3 AZ |
| **Availability** (Tính sẵn sàng) | 99.99% cho Standard class |
| **Scalability** (Khả năng mở rộng) | Không giới hạn dung lượng, tự động scale |
| **Consistency** (Tính nhất quán) | Strong read-after-write consistency từ tháng 12/2020 |

---

## Các Khái Niệm Cốt Lõi

### 1. Bucket (Thùng chứa)

- Container cấp cao nhất chứa objects
- Tên bucket phải **unique toàn cầu** (globally unique) trên toàn bộ AWS
- Thuộc về một AWS Region cụ thể
- Giới hạn: 100 buckets mặc định mỗi account (có thể tăng lên 1000)

### 2. Object (Đối tượng)

- Đơn vị lưu trữ cơ bản trong S3
- Gồm: **data** (dữ liệu), **key** (tên định danh), **metadata** (siêu dữ liệu)
- Kích thước tối đa: **5TB** mỗi object
- Tải lên tối đa qua single PUT: **5GB** (dùng Multipart Upload cho file lớn hơn)

### 3. Key (Khóa định danh)

- Tên đầy đủ của object trong bucket
- Ví dụ: `logs/2026/05/app-server.log`
- Dấu `/` trong key tạo ra cấu trúc **prefix** (tiền tố) giống thư mục — nhưng thực chất S3 là flat namespace (không gian phẳng)

---

## Các File Trong Module Này

| File | Nội Dung | Độ Ưu Tiên |
|------|----------|-----------|
| [1-bucket-and-object-model.md](./1-bucket-and-object-model.md) | Bucket, object, key, metadata, ETag — cơ chế hoạt động | ⭐⭐⭐ |
| [2-storage-classes.md](./2-storage-classes.md) | 6 storage classes — so sánh chi phí và use case | ⭐⭐⭐ |
| [3-versioning-and-mfa-delete.md](./3-versioning-and-mfa-delete.md) | Quản lý phiên bản và xóa có xác thực | ⭐⭐⭐ |
| [4-presigned-urls.md](./4-presigned-urls.md) | URL tạm thời có chữ ký — tạo và sử dụng | ⭐⭐⭐ |
| [5-multipart-upload.md](./5-multipart-upload.md) | Tải lên nhiều phần cho file lớn (>100MB) | ⭐⭐ |

---

## Kiến Trúc S3 Ở Mức Cao

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Region (vd: ap-southeast-1)      │
│                                                         │
│   ┌──────────────────────────────────────────────────┐  │
│   │                  S3 Bucket                       │  │
│   │  Tên: my-company-data                            │  │
│   │                                                  │  │
│   │  ┌──────────────┐  ┌──────────────┐              │  │
│   │  │   Object     │  │   Object     │  ...         │  │
│   │  │  Key: a.jpg  │  │  Key: b.pdf  │              │  │
│   │  │  Size: 2MB   │  │  Size: 10MB  │              │  │
│   │  │  Class: STD  │  │  Class: IA   │              │  │
│   │  └──────────────┘  └──────────────┘              │  │
│   └──────────────────────────────────────────────────┘  │
│                                                         │
│   ┌──────────────────────────────────────────────────┐  │
│   │  Lưu trữ phân tán trên ≥3 AZ (Availability Zone)│  │
│   │  → Durability 99.999999999%                      │  │
│   └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Storage Classes — Tóm Tắt Nhanh

| Class | Truy Cập | Chi Phí Lưu Trữ | Dùng Cho |
|-------|----------|-----------------|---------|
| **Standard** | Thường xuyên | $$$ | Web assets, production data |
| **Standard-IA** | Không thường xuyên | $$ | Backup, logs cũ |
| **One Zone-IA** | Không thường xuyên, 1 AZ | $ | Data có thể tạo lại |
| **Intelligent-Tiering** | Không dự đoán được | $$–$ tự động | Dữ liệu access pattern không rõ |
| **Glacier Instant** | Lưu trữ lạnh, lấy ngay | $ | Archive nhưng cần tốc độ |
| **Glacier Flexible** | Lưu trữ lạnh, 1–12h | $$ | Compliance archive |
| **Glacier Deep Archive** | Lưu trữ lạnh nhất, 12–48h | ¢ | Long-term archive 7–10 năm |

> Chi tiết đầy đủ: [2-storage-classes.md](./2-storage-classes.md)

---

## Tính Năng Bảo Vệ Dữ Liệu

### Versioning (Quản lý phiên bản)
- Lưu nhiều phiên bản của cùng một object key
- Bảo vệ khỏi xóa nhầm hoặc ghi đè
- Kết hợp với **MFA Delete** để tăng cường bảo vệ

### Replication (Sao chép)
- **CRR** — Cross-Region Replication — Sao chép Liên Vùng: DR, latency thấp hơn cho người dùng xa
- **SRR** — Same-Region Replication — Sao chép Cùng Vùng: compliance, aggregation log

### Object Lock (Khóa đối tượng)
- **WORM** — Write Once Read Many — Ghi Một Lần Đọc Nhiều Lần
- Chống xóa hoặc ghi đè trong khoảng thời gian xác định

---

## Bảo Mật S3

```
Tầng 1: IAM Policies    → Ai được phép làm gì (identity-based)
Tầng 2: Bucket Policies → Quy tắc ở cấp bucket (resource-based)
Tầng 3: Block Public Access → Ngăn chặn public access toàn cầu
Tầng 4: Encryption      → SSE-S3 / SSE-KMS / SSE-C / CSE
Tầng 5: VPC Endpoints   → Truy cập không qua internet
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

1. **S3 khác EBS và EFS như thế nào?**
   - S3: object storage, không gắn vào EC2, truy cập qua HTTP/S3 API
   - EBS: block storage, gắn vào một EC2, low-latency
   - EFS: file system NFS, gắn vào nhiều EC2 đồng thời

2. **S3 đảm bảo durability như thế nào?**
   - Tự động replicate data trên tối thiểu 3 AZ trong cùng Region
   - Liên tục kiểm tra tính toàn vẹn dữ liệu (checksums)

3. **Khi nào dùng Intelligent-Tiering?**
   - Khi không biết trước access pattern của dữ liệu
   - Dữ liệu có giá trị > 128KB và lưu trữ > 30 ngày
   - Khi muốn tự động tối ưu chi phí mà không cần lifecycle rules

4. **Strong consistency trong S3 nghĩa là gì?**
   - Sau khi PUT hoặc DELETE thành công, mọi GET ngay sau đó đều nhận được dữ liệu mới nhất
   - Không còn eventual consistency (nhất quán cuối cùng) như trước tháng 12/2020

---

## Điều Hướng

- **Tiếp theo:** [1-bucket-and-object-model.md](./1-bucket-and-object-model.md) — Đi sâu vào cơ chế bucket và object
- **Module trước:** [../README.md](../README.md) — Tổng quan AWS Storage
- **Index:** [../INDEX.md](../INDEX.md) — Toàn bộ chỉ mục

---

**Cập Nhật Lần Cuối:** 2026-05-15
