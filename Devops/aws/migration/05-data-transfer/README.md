# AWS Data Transfer Services — Truyền Tải Dữ Liệu Tổng Quan

> Sau khi lập kế hoạch và di chuyển server/database, bước tiếp theo thường là đồng bộ hóa dữ liệu liên tục hoặc thay thế các hệ thống truyền file cũ. AWS cung cấp ba dịch vụ cốt lõi: **AWS DataSync** — đồng bộ dữ liệu qua mạng tự động và nhanh chóng; **AWS Transfer Family** — managed SFTP/FTPS/FTP/AS2 (Giao Thức Truyền File Có Quản Lý) trên S3/EFS; và **AWS Storage Gateway** — cầu nối hybrid (kết nối lai) giữa on-premises và AWS cloud.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao Cần Dịch Vụ Truyền Tải Dữ Liệu?](#tại-sao-cần-dịch-vụ-truyền-tải-dữ-liệu)
2. [Ba Dịch Vụ Cốt Lõi](#ba-dịch-vụ-cốt-lõi)
3. [Bảng So Sánh Tổng Quan](#bảng-so-sánh-tổng-quan)
4. [Chọn Dịch Vụ Đúng](#chọn-dịch-vụ-đúng)
5. [Điều Hướng Tài Liệu](#điều-hướng-tài-liệu)

---

## 🎯 Tại Sao Cần Dịch Vụ Truyền Tải Dữ Liệu?

Di chuyển server và database mới giải quyết một phần bài toán migration. Nhưng thực tế production còn nhiều nhu cầu khác:

```
Nhu cầu truyền tải dữ liệu điển hình:

1. Đồng bộ file giữa on-premises và AWS Cloud liên tục:
   ├── NAS/SAN (Network-Attached Storage / Storage Area Network) tại datacenter
   ├── → Cần đẩy lên S3 hàng đêm hoặc real-time
   └── DataSync giải quyết điều này

2. Thay thế SFTP server nội bộ:
   ├── Đối tác B2B (Business-to-Business) gửi file qua SFTP mỗi ngày
   ├── Tự dựng SFTP server phải quản lý server, SSL certs, user accounts
   └── Transfer Family quản lý toàn bộ, file đẩy thẳng vào S3

3. Kết nối hybrid (lai) lâu dài:
   ├── Ứng dụng on-premises cần đọc/ghi vào storage AWS như local drive
   ├── Tape library (thư viện băng từ) cần lưu trữ băng ảo trên AWS
   └── Storage Gateway cung cấp giao thức NFS/SMB/iSCSI/VTL tương thích

4. Băng thông hạn chế hoặc không có mạng:
   └── Snow Family (xem 06-snow-family/) giải quyết trường hợp này
```

---

## 🛠️ Ba Dịch Vụ Cốt Lõi

### 1. AWS DataSync — Đồng Bộ Dữ Liệu Tự Động

**Vai trò:** Truyền file từ on-premises (NFS/SMB/S3-compatible/HDFS) lên AWS (S3/EFS/FSx) — hoặc giữa các storage AWS với nhau — một cách nhanh, an toàn, có kiểm tra toàn vẹn.

```
DataSync hoạt động theo nguyên lý:
├── DataSync Agent (VM cài tại on-premises) — Tác Nhân Thu Thập Dữ Liệu
│   └── Giống như "người vận chuyển" đứng tại datacenter của bạn
├── Location (Nguồn và Đích) — Vị Trí
│   ├── Source Location: NFS share, SMB share, S3, EFS, FSx, HDFS...
│   └── Destination Location: S3 bucket, EFS file system, FSx for Windows...
└── Task (Tác Vụ Đồng Bộ)
    ├── Định nghĩa: copy từ source location → destination location
    ├── Có thể lên lịch (schedule): chạy hàng giờ, hàng đêm, hàng tuần
    └── Tự động kiểm tra checksum (mã kiểm tra toàn vẹn) sau khi copy

Hiệu suất:
└── Nhanh hơn copy thủ công tới 10x nhờ parallel transfer (truyền song song)
    và network optimization (tối ưu mạng) tích hợp sẵn
```

Tài liệu chi tiết: [1-datasync-overview.md](./1-datasync-overview.md) và [2-datasync-advanced.md](./2-datasync-advanced.md)

---

### 2. AWS Transfer Family — Managed SFTP/FTPS/FTP/AS2

**Vai trò:** Cung cấp endpoint SFTP (SSH File Transfer Protocol — Giao Thức Truyền File Qua SSH), FTPS (FTP Secure — FTP Bảo Mật), FTP (File Transfer Protocol — Giao Thức Truyền File), AS2 (Applicability Statement 2 — Chuẩn Trao Đổi File B2B) mà **không cần quản lý server** — file được lưu thẳng vào S3 hoặc EFS.

```
Transfer Family hoạt động theo nguyên lý:
├── AWS tạo và quản lý server endpoint (hostname cố định)
│   └── Ví dụ: s-xxxxxxxx.server.transfer.ap-southeast-1.amazonaws.com
├── User kết nối qua SFTP/FTPS/FTP client thông thường
│   └── Filezilla, WinSCP, cyberduck — không cần thay đổi tool
├── AWS xác thực user:
│   ├── SSH key (khóa SSH) — dành cho SFTP
│   ├── Password qua AWS Secrets Manager hoặc custom identity provider
│   └── Tích hợp với Active Directory (qua AWS Directory Service)
└── File upload/download → ghi thẳng vào S3 bucket hoặc EFS

Use case điển hình:
├── Đối tác gửi báo cáo/đơn hàng qua SFTP hàng ngày
├── Thay thế SFTP server nội bộ tốn kém bảo trì
└── EDI (Electronic Data Interchange — Trao Đổi Dữ Liệu Điện Tử) với AS2
```

Tài liệu chi tiết: [3-transfer-family.md](./3-transfer-family.md)

---

### 3. AWS Storage Gateway — Cầu Nối Hybrid Storage

**Vai trò:** Cho phép ứng dụng on-premises dùng storage AWS như local storage bằng cách cung cấp giao thức NFS/SMB/iSCSI/VTL quen thuộc — trong khi dữ liệu thực tế được lưu trên AWS.

```
Storage Gateway có ba chế độ (gateway types):
├── File Gateway (S3 File Gateway) — Cổng File
│   ├── Cung cấp giao thức NFS (Network File System — Hệ Thống File Mạng) hoặc SMB
│   ├── Ứng dụng on-premises mount và dùng như NFS share bình thường
│   └── Dữ liệu thực tế lưu trên S3 (cache hot data tại on-premises)
│
├── Volume Gateway — Cổng Volume (iSCSI Block Storage)
│   ├── Cung cấp iSCSI (Internet Small Computer Systems Interface — Giao Thức Block Storage Qua Mạng)
│   ├── Hai chế độ:
│   │   ├── Stored Volumes: dữ liệu lưu on-premises, backup lên AWS EBS snapshots
│   │   └── Cached Volumes: dữ liệu lưu trên AWS S3, cache tại on-premises
│   └── Dùng cho backup block storage hoặc DR (Disaster Recovery — Phục Hồi Thảm Họa)
│
└── Tape Gateway — Cổng Băng Từ (VTL — Virtual Tape Library)
    ├── Giả lập VTL (Virtual Tape Library — Thư Viện Băng Từ Ảo) tương thích iSCSI
    ├── Phần mềm backup (Veeam, Commvault, Veritas NetBackup) "nghĩ" đang dùng tape thật
    └── Dữ liệu thực tế lưu trên S3 Glacier (lưu trữ lạnh, chi phí thấp)
```

Tài liệu chi tiết: [4-storage-gateway.md](./4-storage-gateway.md)

---

## 📊 Bảng So Sánh Tổng Quan

| Tiêu Chí | AWS DataSync | AWS Transfer Family | AWS Storage Gateway |
| -------- | ------------ | ------------------- | ------------------- |
| **Mục đích chính** | Đồng bộ/di chuyển file hàng loạt | Thay thế SFTP/FTP server | Hybrid storage — tích hợp on-premises + cloud |
| **Giao thức hỗ trợ** | NFS, SMB, S3, EFS, FSx, HDFS | SFTP, FTPS, FTP, AS2 | NFS, SMB, iSCSI, VTL |
| **Storage đích** | S3, EFS, FSx | S3, EFS | S3, EBS Snapshots, S3 Glacier |
| **Cần cài agent?** | ✅ Agent VM (trừ khi transfer trong AWS) | ❌ Không cần (fully managed) | ✅ Virtual appliance (VM) |
| **Scheduling** | ✅ Có thể lên lịch tự động | N/A (on-demand) | N/A (real-time cache) |
| **Kiểm tra toàn vẹn** | ✅ Checksum tự động | ❌ Không (phụ thuộc client) | ✅ Tích hợp |
| **Use case điển hình** | Di chuyển TB dữ liệu, backup định kỳ | B2B file exchange, SFTP managed | App on-premises cần cloud storage |
| **Giá** | Theo GB truyền | Theo giờ endpoint + GB | Theo loại gateway + lưu trữ |

---

## 🧭 Chọn Dịch Vụ Đúng

```
Câu hỏi quyết định:

Q1: Mục tiêu là gì?
├── Di chuyển/đồng bộ file giữa on-premises và AWS → DataSync
├── Thay thế SFTP/FTP server cho đối tác B2B → Transfer Family
└── Ứng dụng on-premises cần đọc/ghi cloud storage liên tục → Storage Gateway

Q2 (nếu chọn DataSync): Dữ liệu nhiều đến mức nào?
├── < 10 TB và có mạng tốt → DataSync qua internet
├── > 10 TB hoặc mạng yếu → DataSync + Direct Connect
└── > 100 TB, không có mạng → Snow Family (06-snow-family/)

Q3 (nếu chọn Transfer Family): Giao thức nào?
├── SFTP → đối tác dùng SSH key, phổ biến nhất
├── FTPS → đối tác cũ dùng FTP + TLS
├── FTP → môi trường nội bộ VPC (không mã hóa, không ra internet)
└── AS2 → EDI, trao đổi tài liệu thương mại điện tử B2B

Q4 (nếu chọn Storage Gateway): Ứng dụng cần giao thức nào?
├── Mount như NFS/SMB share → File Gateway
├── Block storage iSCSI cho server Windows/Linux → Volume Gateway
└── Phần mềm backup tape-based → Tape Gateway
```

### Kịch Bản Thực Tế (Decision Examples)

| Tình Huống | Dịch Vụ Nên Dùng | Lý Do |
| ---------- | ---------------- | ------ |
| Đồng bộ 50 TB từ NAS on-premises lên S3 hàng đêm | **DataSync** | File bulk transfer, có scheduling |
| Đối tác gửi file đơn hàng CSV qua SFTP mỗi sáng | **Transfer Family (SFTP)** | Managed SFTP, không tốn công bảo trì server |
| Server kế toán on-premises cần lưu file vào S3 | **Storage Gateway (File)** | Mount NFS/SMB, ứng dụng không cần thay đổi |
| Thay thế tape backup cho 5 server on-premises | **Storage Gateway (Tape)** | VTL tương thích Veeam/Commvault |
| EDI — trao đổi tài liệu với nhà cung cấp qua AS2 | **Transfer Family (AS2)** | Chuẩn AS2 cho B2B commerce |
| Copy file giữa S3 us-east-1 và S3 ap-southeast-1 | **DataSync (không cần agent)** | Transfer trong AWS, không cần agent |

---

## 🔗 Điều Hướng Tài Liệu

| File | Nội Dung |
| ---- | -------- |
| [1-datasync-overview.md](./1-datasync-overview.md) | DataSync kiến trúc, agent, location types, task configuration, scheduling |
| [2-datasync-advanced.md](./2-datasync-advanced.md) | Bandwidth throttling, filtering, data verification, monitoring, best practices |
| [3-transfer-family.md](./3-transfer-family.md) | Transfer Family: SFTP/FTPS/FTP/AS2, identity providers, S3/EFS mapping, security |
| [4-storage-gateway.md](./4-storage-gateway.md) | File Gateway, Volume Gateway, Tape Gateway — cài đặt, cache, performance |

---

## 🎓 Câu Hỏi Phỏng Vấn Điển Hình (Module Này)

1. Khi nào dùng DataSync thay vì Snow Family? Tiêu chí phân biệt?
2. Transfer Family khác gì so với tự dựng SFTP server trên EC2?
3. Storage Gateway File Gateway khác gì DataSync?
4. Tính thời gian truyền 50 TB qua đường 1 Gbps bằng DataSync.
5. Transfer Family lưu file ở đâu? Có thể kết hợp với Lambda để xử lý file không?
6. Tape Gateway phù hợp với phần mềm backup nào? Dữ liệu lưu trên AWS ở đâu?
7. Nếu băng thông thấp và cần chuyển 200 TB, bạn chọn giải pháp nào?

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
