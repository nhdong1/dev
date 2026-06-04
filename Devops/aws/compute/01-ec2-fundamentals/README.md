# EC2 Fundamentals — Nền Tảng EC2

> **EC2 — Elastic Compute Cloud — Máy Chủ Ảo Đám Mây** là dịch vụ compute cốt lõi của AWS, cho phép thuê máy chủ ảo theo nhu cầu với hàng trăm loại cấu hình phần cứng khác nhau.

## 📚 Mục Lục

1. [Tổng Quan EC2](#tổng-quan-ec2)
2. [Vòng Đời Instance](#vòng-đời-instance)
3. [Kiến Trúc Cơ Bản](#kiến-trúc-cơ-bản)
4. [Nội Dung Trong Thư Mục Này](#nội-dung-trong-thư-mục-này)
5. [Câu Hỏi Phỏng Vấn Nhanh](#câu-hỏi-phỏng-vấn-nhanh)

---

## Tổng Quan EC2

**EC2** ra đời năm 2006 và là nền tảng của hầu hết mọi kiến trúc AWS. Dù Lambda và Container đang phát triển mạnh, EC2 vẫn chiếm phần lớn workload thực tế vì:

- **Toàn quyền kiểm soát** — Hệ điều hành, cấu hình mạng, lưu trữ, phần mềm
- **Tính linh hoạt cao** — Hơn 750 loại instance với nhiều cấu hình CPU/RAM/GPU/Network
- **Hỗ trợ mọi workload** — Database, web server, machine learning, gaming, HPC
- **Tích hợp sinh thái AWS** — Kết nối trực tiếp với VPC, EBS, ELB, IAM, CloudWatch

### Khi Nào Dùng EC2 Thay Vì Lambda/ECS?

| Tình Huống                              | Lý Do Chọn EC2                              |
| --------------------------------------- | ------------------------------------------- |
| Ứng dụng legacy cần OS cụ thể          | Kiểm soát OS hoàn toàn                      |
| Database tự quản lý (MySQL, PostgreSQL) | Cần persistent storage và low latency        |
| Workload dài hạn > 15 phút             | Lambda bị giới hạn 15 phút timeout          |
| Cần GPU hoặc bare metal                | Loại instance đặc biệt chỉ có trên EC2      |
| Windows license phức tạp               | Quản lý license linh hoạt hơn               |
| Yêu cầu compliance cụ thể về OS       | Dedicated Host, kiểm soát hypervisor        |

---

## Vòng Đời Instance

```
             ┌─────────────────────────────────────────────┐
             │            EC2 Instance Lifecycle             │
             └─────────────────────────────────────────────┘

  [AMI]
    │
    ▼
[pending] ──────────────────────────────────────► [running]
    │                                                  │
    │                                         ┌────────┼────────┐
    │                                         ▼        ▼        ▼
    │                                    [reboot] [stopping] [hibernate]
    │                                              │
    │                                              ▼
    │                                         [stopped] ──► [starting]
    │                                              │              │
    │                                              ▼              ▼
    │                                        [terminated]    [running]
    │
    └── [launch failed] ──► [terminated]
```

### Các Trạng Thái Instance

| Trạng Thái   | Mô Tả                                    | Tính Phí CPU/RAM |
| ------------ | ---------------------------------------- | ---------------- |
| `pending`    | Đang khởi động, chưa sẵn sàng           | Không            |
| `running`    | Đang chạy, có thể kết nối               | Có               |
| `stopping`   | Đang tắt                                 | Không            |
| `stopped`    | Đã tắt, dữ liệu EBS vẫn còn             | Không (EBS có)   |
| `shutting-down` | Đang xóa                              | Không            |
| `terminated` | Đã xóa, không thể khôi phục            | Không            |
| `rebooting`  | Khởi động lại, giữ IP và dữ liệu       | Có               |

> **Lưu ý quan trọng:** Instance ở trạng thái `stopped` không tính phí CPU/RAM nhưng vẫn tính phí **EBS storage** và **Elastic IP** (nếu có).

---

## Kiến Trúc Cơ Bản

```
┌──────────────── AWS Region (us-east-1) ────────────────┐
│                                                          │
│  ┌─────────── VPC (Virtual Private Cloud) ───────────┐  │
│  │                                                    │  │
│  │  ┌──── Availability Zone A ────┐                  │  │
│  │  │                             │                  │  │
│  │  │  ┌─────── Subnet ────────┐  │                  │  │
│  │  │  │                       │  │                  │  │
│  │  │  │  ┌─────────────────┐  │  │                  │  │
│  │  │  │  │   EC2 Instance  │  │  │                  │  │
│  │  │  │  │                 │  │  │                  │  │
│  │  │  │  │  ┌───────────┐  │  │  │                  │  │
│  │  │  │  │  │    EBS    │  │  │  │                  │  │
│  │  │  │  │  │  (root +  │  │  │  │                  │  │
│  │  │  │  │  │   data)   │  │  │  │                  │  │
│  │  │  │  │  └───────────┘  │  │  │                  │  │
│  │  │  │  │                 │  │  │                  │  │
│  │  │  │  │  IAM Role       │  │  │                  │  │
│  │  │  │  │  Security Group │  │  │                  │  │
│  │  │  │  │  Key Pair       │  │  │                  │  │
│  │  │  │  └─────────────────┘  │  │                  │  │
│  │  │  └───────────────────────┘  │                  │  │
│  │  └─────────────────────────────┘                  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Các Thành Phần Cốt Lõi

| Thành Phần               | Mục Đích                                              |
| ------------------------ | ----------------------------------------------------- |
| **AMI** (Amazon Machine Image — Ảnh Máy Ảo) | Template để tạo instance      |
| **Instance Type**         | Cấu hình phần cứng (CPU, RAM, Network, Storage)      |
| **EBS** (Elastic Block Store — Lưu Trữ Khối) | Ổ đĩa mạng persistent       |
| **Security Group** (Nhóm Bảo Mật) | Firewall cấp instance, kiểm soát traffic  |
| **Key Pair** (Cặp Khóa)  | SSH/RDP authentication cho EC2                       |
| **IAM Role** (Vai Trò)   | Quyền truy cập AWS services từ bên trong instance    |
| **User Data** (Dữ Liệu Khởi Tạo) | Script chạy khi instance khởi động lần đầu |
| **Placement Group** (Nhóm Vị Trí) | Kiểm soát vị trí vật lý của instance        |
| **Elastic IP** (IP Tĩnh) | IP public cố định, không đổi khi restart             |

---

## Nội Dung Trong Thư Mục Này

| File                       | Chủ Đề                                           | Độ Khó |
| -------------------------- | ------------------------------------------------ | ------ |
| [1-instance-types.md](./1-instance-types.md) | Instance families, naming convention, chọn đúng loại | ⭐⭐ |
| [2-ami-storage.md](./2-ami-storage.md) | AMI, EBS volumes, Instance Store, Snapshots | ⭐⭐ |
| [3-security-keypairs.md](./3-security-keypairs.md) | Security Groups, Key Pairs, IAM Instance Profiles | ⭐⭐ |
| [4-user-data-metadata.md](./4-user-data-metadata.md) | User Data scripts, IMDS v1 vs v2 | ⭐⭐ |
| [5-placement-groups.md](./5-placement-groups.md) | Cluster, Spread, Partition Placement Groups | ⭐⭐⭐ |

### Lộ Trình Học Đề Xuất

```
1. Đọc file README này (bạn đang ở đây)
2. → 1-instance-types.md    (hiểu loại instance trước)
3. → 2-ami-storage.md       (storage là nền tảng)
4. → 3-security-keypairs.md (bảo mật cơ bản)
5. → 4-user-data-metadata.md (automation & bootstrap)
6. → 5-placement-groups.md  (tối ưu nâng cao)
```

---

## Câu Hỏi Phỏng Vấn Nhanh

### Câu Hỏi Thường Gặp

**Q: EC2 instance ở trạng thái `stopped` có tính phí không?**
> Không tính phí vCPU và RAM. Nhưng vẫn tính phí EBS volumes gắn vào instance và Elastic IP (nếu không được sử dụng sẽ bị tính phí).

**Q: Sự khác biệt giữa `stop` và `terminate` là gì?**
> `stop` — Tắt instance, dữ liệu EBS còn nguyên, có thể khởi động lại. `terminate` — Xóa vĩnh viễn instance và root EBS volume (data volumes tùy cấu hình `DeleteOnTermination`).

**Q: Khi instance bị `stop` rồi `start` lại, điều gì thay đổi?**
> Public IP (public DNS) thay đổi (trừ khi dùng Elastic IP). Private IP trong VPC thường giữ nguyên. Instance có thể chạy trên host vật lý khác.

**Q: Instance Store (lưu trữ tạm thời trên máy chủ vật lý) khác EBS (lưu trữ mạng bền vững) thế nào?**
> Instance Store: Nhanh hơn, nằm trên host vật lý, **mất dữ liệu khi stop/terminate**. EBS: Network-attached, **bền vững qua stop/start**, có thể snapshot. Dùng Instance Store cho cache, buffer, temp files.

---

## Liên Kết Tham Khảo

- [Tiếp theo: Instance Types →](./1-instance-types.md)
- [← Trở về: Compute Overview](../README.md)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/latest/userguide/)

---

**Cập Nhật:** 2026-05-14 | **Trạng Thái:** ✅ Hoàn thành
