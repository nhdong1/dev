# Amazon EFS — Elastic File System (Hệ Thống Tệp Linh Hoạt)

> EFS là dịch vụ lưu trữ tệp được quản lý hoàn toàn (fully managed), có khả năng co giãn tự động (auto-scaling), dựa trên giao thức NFS — Network File System (Hệ Thống Tệp Mạng). EFS cho phép nhiều EC2 instance, container, và Lambda function cùng truy cập một file system chia sẻ đồng thời.

---

## 📚 Mục Lục Chủ Đề EFS

| File | Chủ Đề | Ưu Tiên |
|------|---------|---------|
| [1-mount-targets-and-access-points.md](./1-mount-targets-and-access-points.md) | Mount Targets & Access Points — Điểm Gắn Kết & Điểm Truy Cập | ⭐⭐⭐ |
| [2-performance-modes.md](./2-performance-modes.md) | Performance Modes — Chế Độ Hiệu Suất | ⭐⭐⭐ |
| [3-throughput-modes.md](./3-throughput-modes.md) | Throughput Modes — Chế Độ Thông Lượng | ⭐⭐⭐ |
| [4-efs-intelligent-tiering.md](./4-efs-intelligent-tiering.md) | EFS Intelligent-Tiering — Phân Tầng Thông Minh | ⭐⭐ |
| [5-efs-vs-ebs-vs-s3.md](./5-efs-vs-ebs-vs-s3.md) | EFS vs EBS vs S3 — Bảng So Sánh Đầy Đủ | ⭐⭐⭐ |

---

## 🎯 EFS Là Gì?

Amazon EFS cung cấp một **shared file system** (hệ thống tệp chia sẻ) theo chuẩn NFS v4.1/v4.0. Không như EBS chỉ gắn được vào một EC2 tại một thời điểm, EFS cho phép **hàng nghìn client** kết nối đồng thời.

### Đặc Điểm Chính

| Đặc Điểm | Chi Tiết |
|-----------|----------|
| **Protocol** | NFS v4.1 / v4.0 (Network File System) |
| **Scalability** (Khả năng co giãn) | Tự động scale từ 0 đến petabyte |
| **Availability** (Tính sẵn sàng) | Multi-AZ — dữ liệu trải đều trên nhiều Availability Zone |
| **Durability** (Độ bền) | 99.999999999% (11 chín) |
| **Access** (Truy cập) | Đồng thời từ nhiều EC2, ECS, EKS, Lambda |
| **OS** (Hệ điều hành) | Linux only — không hỗ trợ Windows natively |
| **Encryption** (Mã hóa) | At-rest (KMS) và in-transit (TLS) |

---

## 🏗️ Kiến Trúc Tổng Quan

```
                    ┌─────────────────────────────────────┐
                    │         Amazon EFS File System       │
                    │    (Hệ Thống Tệp EFS)               │
                    │                                      │
                    │  ┌──────────┐    ┌──────────┐       │
                    │  │ Standard │    │    IA    │       │
                    │  │  (Hot)   │ ←→ │  (Cold)  │       │
                    │  └──────────┘    └──────────┘       │
                    └──────────┬──────────────────────────┘
                               │ NFS v4.1
              ┌────────────────┼────────────────┐
              │                │                │
        ┌─────▼────┐    ┌──────▼───┐    ┌──────▼───┐
        │ Mount    │    │  Mount   │    │  Mount   │
        │ Target   │    │  Target  │    │  Target  │
        │  AZ-1a   │    │   AZ-1b  │    │   AZ-1c  │
        └─────┬────┘    └──────┬───┘    └──────┬───┘
              │                │                │
    ┌─────────▼──┐   ┌─────────▼──┐   ┌────────▼───┐
    │  EC2 / ECS │   │  EC2 / EKS │   │   Lambda   │
    │  instances │   │  pods      │   │  functions │
    └────────────┘   └────────────┘   └────────────┘
```

---

## 📊 Storage Classes (Lớp Lưu Trữ) EFS

| Storage Class | Mô Tả | Chi Phí (us-east-1) | Use Case |
|---------------|-------|---------------------|---------|
| **EFS Standard** | Đa-AZ, truy cập thường xuyên | ~$0.30/GB-month | Active data — dữ liệu đang dùng |
| **EFS Standard-IA** | Infrequent Access — Truy cập không thường xuyên, đa-AZ | ~$0.016/GB-month | Dữ liệu ít truy cập |
| **EFS One Zone** | Một AZ duy nhất | ~$0.16/GB-month | Dev/test, dữ liệu có thể tái tạo |
| **EFS One Zone-IA** | Một AZ, ít truy cập | ~$0.008/GB-month | Backup lạnh, archive |

> **Lưu ý:** EFS Standard-IA tính phí truy xuất ~$0.01/GB khi đọc dữ liệu.

---

## 🔐 Bảo Mật EFS

### Nhiều Lớp Kiểm Soát

```
Lớp 1: Network — Security Groups (kiểm soát kết nối NFS port 2049)
Lớp 2: File System — Resource-based Policy (chính sách tài nguyên)
Lớp 3: Access Points — POSIX identity enforcement (ép buộc danh tính POSIX)
Lớp 4: IAM — Client identity authorization (ủy quyền dựa trên IAM)
Lớp 5: Encryption — KMS at-rest, TLS in-transit
```

---

## ⚡ Performance Modes & Throughput (Chế Độ Hiệu Suất & Thông Lượng)

### Performance Modes (chọn khi tạo, không đổi được)

| Mode | IOPS | Latency | Dùng Khi |
|------|------|---------|---------|
| **General Purpose** (Đa Dụng) | ~35.000 | Thấp nhất (~1ms) | Mặc định — web servers, CMS |
| **Max I/O** (Tối Đa I/O) | Không giới hạn | Cao hơn (~10ms) | Big data, HPC, hàng nghìn client |

### Throughput Modes (có thể đổi sau khi tạo)

| Mode | Throughput | Dùng Khi |
|------|-----------|---------|
| **Bursting** (Bùng phát) | Scale theo dung lượng, credit-based | Workload không ổn định |
| **Provisioned** (Đã cấp phát) | Đặt cố định (MiB/s) | Cần throughput cao hơn dung lượng |
| **Elastic** (Linh hoạt) | Tự động scale 1–3 GiB/s | AWS khuyến nghị — serverless |

---

## 🔗 Tích Hợp Với Các Dịch Vụ AWS

```
EC2 ──────────────────────────────────────────┐
ECS (Elastic Container Service) ──────────────┤
EKS (Elastic Kubernetes Service) ─────────────┤──→ Amazon EFS
Lambda (với EFS mount) ───────────────────────┤
AWS Fargate ──────────────────────────────────┘

AWS Backup ──────────────────────────────────────→ EFS Backups
AWS DataSync ───────────────────────────────────→ EFS Migration
CloudWatch ──────────────────────────────────────→ EFS Metrics
```

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp về EFS

1. **Khi nào dùng EFS thay vì EBS?** → Khi cần shared file system cho nhiều instance/container
2. **EFS khác FSx for Windows như thế nào?** → EFS dùng NFS (Linux), FSx dùng SMB (Windows + Active Directory)
3. **Làm sao tối ưu chi phí EFS?** → Dùng Intelligent-Tiering để tự động chuyển dữ liệu lạnh sang IA
4. **EFS có thể dùng với Windows không?** → Không natively; cần dùng FSx for Windows
5. **Performance Mode có thể đổi sau khi tạo không?** → Không — phải chọn đúng từ đầu

---

## 🚀 Bắt Đầu Nhanh

```bash
# Tạo EFS file system
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --tags Key=Name,Value=my-efs

# Mount trên EC2 (cần nfs-utils)
sudo mount -t nfs4 \
  -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
  fs-xxxxxxxx.efs.ap-southeast-1.amazonaws.com:/ \
  /mnt/efs

# Mount qua EFS mount helper (khuyến nghị)
sudo mount -t efs -o tls fs-xxxxxxxx:/ /mnt/efs
```

---

## 📖 Điều Hướng

- **Trước:** [03-ebs/](../03-ebs/) — EBS Block Storage
- **Tiếp theo:** [05-security/](../05-security/) — Bảo Mật Lưu Trữ
- **So sánh:** [5-efs-vs-ebs-vs-s3.md](./5-efs-vs-ebs-vs-s3.md)

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
