# FSx for Windows File Server — Hệ Thống Tệp Windows Được Quản Lý

> FSx for Windows File Server — Dịch Vụ Hệ Thống Tệp Windows Được Quản Lý Hoàn Toàn — cung cấp shared file storage tương thích hoàn toàn với Windows, tích hợp Active Directory (AD — Dịch Vụ Thư Mục Quản Lý Danh Tính), và hỗ trợ SMB — Server Message Block — Giao Thức Chia Sẻ Tệp Windows.

---

## Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc](#kiến-trúc)
3. [Tích Hợp Active Directory](#tích-hợp-active-directory)
4. [SMB và DFS](#smb-và-dfs)
5. [Shadow Copies](#shadow-copies)
6. [Hiệu Suất và Sizing](#hiệu-suất-và-sizing)
7. [Bảo Mật](#bảo-mật)
8. [Chi Phí](#chi-phí)
9. [Use Cases Thực Tế](#use-cases-thực-tế)
10. [Điểm Kiểm Tra Phỏng Vấn](#điểm-kiểm-tra-phỏng-vấn)

---

## Tổng Quan

### FSx for Windows là gì?

FSx for Windows File Server là hệ thống tệp Windows Server được AWS quản lý hoàn toàn. Về mặt kỹ thuật, đây là Windows Server chạy NTFS — New Technology File System — Hệ Thống Tệp Công Nghệ Mới, phục vụ file qua giao thức SMB.

### Điểm Khác Biệt So Với EFS

| Tiêu Chí | FSx for Windows | EFS |
|---------|----------------|-----|
| Protocol | SMB (Windows native) | NFS — Network File System |
| OS | Windows & Linux | Linux only |
| Active Directory | Tích hợp native | Không |
| NTFS permissions | Có đầy đủ | Không |
| DFS Namespaces | Có | Không |
| Shadow Copies | Có (VSS) | Không |
| Windows ACL | Đầy đủ | Không |

### Khi Nào Dùng

```
✅ Nên dùng FSx for Windows khi:
- Workload Windows cần SMB shares
- Ứng dụng dùng Windows authentication (Active Directory)
- Migration lift-and-shift từ on-premises file servers
- Home directories cho Windows users
- SharePoint, SQL Server FileStream
- Container Windows trên ECS

❌ Không nên dùng khi:
- Workload Linux thuần túy → dùng EFS
- HPC / ML training → dùng FSx Lustre
- Cần multi-protocol → dùng FSx ONTAP
```

---

## Kiến Trúc

### Single-AZ Deployment (Triển Khai Một Vùng Khả Dụng)

```
VPC
└── AZ-A (us-east-1a)
    ├── Subnet
    │   ├── FSx File System (Primary)
    │   │   └── Standby server (cùng AZ)
    │   └── ENI — Elastic Network Interface — Giao Diện Mạng Linh Hoạt
    └── EC2 / Windows instances
```

**Ưu điểm**: Rẻ hơn ~50% so với Multi-AZ
**Nhược điểm**: Không chịu được mất AZ

### Multi-AZ Deployment (Triển Khai Đa Vùng Khả Dụng)

```
VPC
├── AZ-A (us-east-1a)
│   ├── Primary file server
│   └── ENI (IP tĩnh cho client)
└── AZ-B (us-east-1b)
    ├── Standby file server (mirror)
    └── ENI (failover IP)

Khi AZ-A lỗi:
→ Tự động failover sang AZ-B trong vài giây
→ Client tự reconnect qua DNS (không đổi IP/hostname)
```

**RTO** (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục): Thường < 30 giây khi failover tự động.

### Replication (Nhân Bản)

Tất cả dữ liệu được nhân bản đồng bộ:
- **Single-AZ**: Nhân bản giữa SSD tiers trong cùng AZ
- **Multi-AZ**: Nhân bản đồng bộ qua AZ — ghi dữ liệu chỉ xác nhận sau khi cả hai AZ đã nhận

---

## Tích Hợp Active Directory

### Active Directory là gì?

**AD — Active Directory — Dịch Vụ Thư Mục Quản Lý Danh Tính** là hệ thống xác thực và phân quyền của Microsoft. FSx for Windows tích hợp sâu với AD để:

- Xác thực người dùng qua Kerberos
- Phân quyền file dựa trên AD groups
- Single Sign-On — SSO — Đăng Nhập Một Lần

### Hai Loại AD Hỗ Trợ

#### 1. AWS Managed Microsoft AD (Khuyến Nghị)

```
AWS Cloud
├── AWS Directory Service (AWS Managed AD)
│   ├── Domain Controller 1 (AZ-A)
│   └── Domain Controller 2 (AZ-B)  ← High Availability tự động
└── FSx for Windows
    └── Join domain: corp.example.com
```

**Khi nào dùng**: Dự án mới trên AWS, không có on-premises AD

#### 2. Self-Managed AD (On-Premises hoặc EC2)

```
On-Premises / EC2
└── Domain Controller (tự quản lý)
    └── corp.example.com

AWS VPC
└── FSx for Windows
    └── Join domain qua Direct Connect / VPN
```

**Khi nào dùng**: Đã có AD on-premises, cần FSx tham gia domain hiện có

### Cấu Hình AD Trust (Ủy Thác AD)

Khi cần user từ nhiều domain khác nhau truy cập FSx:

```
Forest A (corp.example.com)  ←──Trust──→  Forest B (partner.example.com)
     └── FSx joins Forest A                    └── Users từ Forest B
                                                    cũng được truy cập
```

---

## SMB và DFS

### SMB — Server Message Block

**SMB** là giao thức mạng để chia sẻ file và tài nguyên trong môi trường Windows. FSx hỗ trợ SMB 2.0, 2.1, 3.0, và 3.1.1.

```powershell
# Mount FSx share trên Windows
net use Z: \\fs-0123456789abcdef.fsx.us-east-1.amazonaws.com\share

# Mount qua PowerShell
New-PSDrive -Name "Z" -PSProvider FileSystem `
    -Root "\\fs-0123456789abcdef.fsx.us-east-1.amazonaws.com\share" `
    -Persist
```

```bash
# Mount FSx share trên Linux qua Samba
sudo mount -t cifs //fs-xxx.fsx.us-east-1.amazonaws.com/share /mnt/fsx \
    -o username=admin,password=xxx,domain=corp.example.com
```

### DFS — Distributed File System — Hệ Thống Tệp Phân Tán

**DFS Namespaces** (Không Gian Tên DFS) cho phép tập hợp nhiều file shares từ nhiều server dưới một đường dẫn UNC — Universal Naming Convention — thống nhất.

```
Trước DFS (phức tạp):
├── \\server1\marketing
├── \\server2\engineering
└── \\server3\finance

Sau DFS (đơn giản):
└── \\corp.example.com\shares
    ├── marketing → trỏ đến \\fsx-A\marketing
    ├── engineering → trỏ đến \\fsx-B\engineering
    └── finance → trỏ đến \\fsx-C\finance
```

**DFS Replication** (Nhân Bản DFS): Tự động đồng bộ nội dung giữa nhiều FSx instances ở các region khác nhau.

---

## Shadow Copies

### Shadow Copies là gì?

**Shadow Copies** (Bản Sao Bóng) hay **VSS — Volume Shadow Copy Service — Dịch Vụ Sao Chép Bóng** là tính năng Windows cho phép tạo snapshot nhất quán của file system tại một thời điểm, mà không cần dừng ứng dụng.

### Cách Hoạt Động

```
T=0 (snapshot được tạo)
├── File A: version 1.0  ← được lưu trong shadow copy
├── File B: version 2.3  ← được lưu trong shadow copy
└── File C: version 1.5  ← được lưu trong shadow copy

T=1 (user vô tình xóa File B)
├── File A: version 1.0
├── File B: [đã xóa]
└── File C: version 1.5

Khôi phục từ shadow copy:
→ Chuột phải vào thư mục → Properties → Previous Versions
→ Chọn snapshot tại T=0
→ Restore File B: version 2.3 ✅
```

### Cấu Hình Shadow Copies

```json
{
  "ClientConfigurations": [
    {
      "Clock": "DEFAULT",
      "Schedule": "0 7,12,17 * * 1-5",
      "Retention": "1 1 1"
    }
  ]
}
```

**Giải thích schedule**: Chạy lúc 7h, 12h, 17h — thứ Hai đến thứ Sáu
**Retention**: Giữ lại theo ngày / tuần / tháng

---

## Hiệu Suất và Sizing

### Các Thông Số Quan Trọng

| Thông Số | Giải Thích | Giá Trị |
|---------|-----------|---------|
| **Storage capacity** (Dung lượng lưu trữ) | Tổng dung lượng | 32 GB – 65.536 GB |
| **Throughput capacity** (Dung lượng thông lượng) | Băng thông đọc/ghi | 8 – 2.048 MB/s |
| **IOPS** (Input/Output Operations Per Second — Số thao tác đọc/ghi mỗi giây) | Số thao tác đồng thời | Lên đến 1 triệu |

### Storage Tiers (Các Tầng Lưu Trữ)

```
SSD Storage (Ổ Đĩa Thể Rắn — Lưu Trữ Chính):
├── Latency: < 1ms
├── Dùng cho: Dữ liệu truy cập thường xuyên
└── Giá: ~$0.13/GB-month

HDD Storage (Ổ Đĩa Cứng — Lưu Trữ Thứ Cấp):
├── Latency: vài ms
├── Dùng cho: Dữ liệu truy cập không thường xuyên
└── Giá: ~$0.025/GB-month
```

### Throughput Scaling (Mở Rộng Thông Lượng)

Throughput capacity có thể thay đổi mà không cần downtime:

```bash
# AWS CLI — tăng throughput capacity
aws fsx update-file-system \
    --file-system-id fs-0123456789abcdef \
    --windows-configuration ThroughputCapacity=512
```

**Lưu ý**: Thay đổi throughput mất 5–30 phút, không ảnh hưởng đến hoạt động.

---

## Bảo Mật

### Mã Hóa (Encryption)

**Encryption at-rest** (Mã hóa khi lưu trữ):
- Tự động với AWS KMS — Key Management Service — Dịch Vụ Quản Lý Khóa
- Dùng AES-256
- Không thể tắt sau khi tạo

**Encryption in-transit** (Mã hóa khi truyền tải):
- SMB 3.0 encryption tự động khi client hỗ trợ
- Có thể enforce encryption cho tất cả kết nối

```powershell
# Kiểm tra encryption trên Windows client
Get-SmbConnection | Select-Object -Property ServerName, Encrypted
```

### Network Security (Bảo Mật Mạng)

```
FSx nằm trong VPC subnet:
├── Security Groups: Kiểm soát traffic vào/ra
│   ├── Inbound: Port 445 (SMB), 389/636 (LDAP/LDAPS cho AD)
│   └── Outbound: Không giới hạn (hoặc chỉ ra AD)
└── VPC — chỉ truy cập từ nội bộ VPC hoặc qua Direct Connect/VPN
```

### Backup Tự Động

- Backup tự động mỗi ngày, lưu 0–90 ngày
- Có thể tạo backup thủ công bất kỳ lúc nào
- Backup được mã hóa cùng key với file system

---

## Chi Phí

### Bảng Giá (Tham Khảo — Giá có thể thay đổi)

| Thành Phần | SSD | HDD |
|-----------|-----|-----|
| Storage | ~$0.13/GB-month | ~$0.025/GB-month |
| Throughput | ~$2.20/MBps-month | ~$2.20/MBps-month |
| Backup (S3) | ~$0.05/GB-month | ~$0.05/GB-month |
| Multi-AZ premium | +~50% | +~50% |

### Ví Dụ Tính Toán Chi Phí

**Kịch bản**: 5 TB SSD, 512 MB/s throughput, Single-AZ, 30 ngày backup

```
Storage:    5.000 GB × $0.13  = $650/month
Throughput: 512 MBps × $2.20  = $1.126/month
Backup:     5.000 GB × $0.05  = $250/month
                               ─────────────
Tổng:                          ~$2.026/month
```

### Tối Ưu Chi Phí

1. **Dùng HDD cho dữ liệu ít truy cập**: Tiết kiệm 80% chi phí storage
2. **Chọn Single-AZ nếu không cần HA**: Tiết kiệm ~50%
3. **Giảm throughput khi không cần**: Thay đổi được mà không downtime
4. **Retention backup ngắn hơn**: Backup là khoản chi phí ẩn lớn

---

## Use Cases Thực Tế

### 1. Home Directories (Thư Mục Cá Nhân) Cho Nhân Viên

```
Kịch bản: 500 nhân viên, mỗi người cần 50 GB home directory

Kiến trúc:
├── AWS Managed Microsoft AD
├── FSx for Windows (Multi-AZ, SSD, 25 TB)
└── Windows Clients (on-prem) ─── Direct Connect ──► FSx
    └── \\corp.example.com\home\username
```

**Lợi ích**: IT không cần quản lý file server vật lý, tự động backup, HA.

### 2. SharePoint Backend

```
SharePoint Server (EC2)
    └── Nội dung lưu trên FSx for Windows
        ├── Document libraries
        ├── Team sites
        └── User profiles
```

**Lưu ý**: SharePoint đòi hỏi NTFS permissions và VSS — FSx đáp ứng đủ.

### 3. SQL Server FileStream (Lưu Trữ Tệp SQL Server)

**FileStream** là tính năng SQL Server lưu trữ dữ liệu BLOB (Binary Large Object — Đối Tượng Nhị Phân Lớn) như file thay vì trong database.

```sql
-- SQL Server cấu hình FileStream trỏ đến FSx
EXEC sp_configure N'filestream access level', 2;
RECONFIGURE;

-- Tạo filegroup trên FSx share
ALTER DATABASE MyDB
    ADD FILEGROUP [FSxFG] CONTAINS FILESTREAM;

ALTER DATABASE MyDB
    ADD FILE (
        NAME = N'FileStreamData',
        FILENAME = N'\\fsx-xxx\share\filestream-data'
    )
    TO FILEGROUP [FSxFG];
```

### 4. Migration Lift-and-Shift (Di Chuyển Nguyên Trạng)

```
On-Premises                          AWS
┌──────────────────┐                ┌──────────────────┐
│ Windows File     │                │ FSx for Windows  │
│ Server           │ ─ DataSync ──► │ (same domain)    │
│ \\fileserver\hr  │                │ \\fsx-xxx\hr     │
└──────────────────┘                └──────────────────┘

Sau migration:
├── DFS Namespace chuyển hướng \\corp\hr → FSx
└── Users không biết đã migration
```

---

## Điểm Kiểm Tra Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: FSx for Windows khác EFS như thế nào?**
> A: FSx for Windows dùng SMB và tích hợp Active Directory, phù hợp Windows workloads. EFS dùng NFS, chỉ cho Linux, không có AD integration. Không thể dùng EFS cho ứng dụng Windows.

**Q: Multi-AZ FSx có đảm bảo không mất dữ liệu khi AZ lỗi không?**
> A: Có. Dữ liệu nhân bản đồng bộ sang AZ thứ hai. Ghi chỉ xác nhận khi cả hai AZ đã nhận. Failover tự động, thường < 30 giây, RTO thấp.

**Q: Làm sao backup FSx for Windows?**
> A: FSx tự động backup hàng ngày về S3. Ngoài ra có thể dùng VSS shadow copies cho point-in-time recovery (Khôi Phục Theo Thời Điểm Cụ Thể) ở mức file.

**Q: Có thể thay đổi storage capacity sau khi tạo không?**
> A: Có, có thể tăng dung lượng storage và throughput capacity mà không cần downtime. Không thể giảm dung lượng.

**Q: FSx for Windows có thể truy cập từ on-premises không?**
> A: Có, qua Direct Connect hoặc VPN Site-to-Site. Máy on-premises cần join cùng AD domain (hoặc có trust relationship).

### Bảng Tóm Tắt Nhanh

| Câu Hỏi | Trả Lời |
|---------|---------|
| Protocol | SMB 2.0 / 3.0 |
| OS | Windows & Linux (qua Samba) |
| AD Integration | Có — bắt buộc khi tạo |
| Storage tối đa | 64 TB (SSD), 65.536 TB (HDD) |
| Throughput tối đa | 2.048 MB/s |
| Backup | Tự động hàng ngày + VSS Shadow Copies |
| Encryption | AES-256 (KMS), không thể tắt |
| Multi-AZ | Có, tự động failover |
| Giá SSD | ~$0.13/GB-month + throughput |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Trạng Thái:** ✅ Hoàn thành
