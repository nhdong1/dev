# 🌉 Storage Gateway & Hybrid Cloud — Lưu Trữ Đám Mây Lai

> Hướng dẫn toàn diện về AWS Storage Gateway và các giải pháp Hybrid Cloud Storage (Lưu Trữ Đám Mây Lai): kết nối hệ thống on-premises với AWS, đồng bộ dữ liệu tự động, và thay thế hạ tầng tape vật lý.

---

## 📚 Mục Lục Chủ Đề

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-file-gateway.md](1-file-gateway.md) | File Gateway — NFS/SMB → S3, use cases, cấu hình | ⭐⭐ |
| [2-volume-gateway.md](2-volume-gateway.md) | Volume Gateway — iSCSI block storage, Cached vs Stored mode | ⭐⭐⭐ |
| [3-tape-gateway.md](3-tape-gateway.md) | Tape Gateway — Virtual Tape Library, thay thế băng vật lý | ⭐⭐ |
| [4-datasync.md](4-datasync.md) | DataSync — đồng bộ tự động on-premises ↔ AWS, agent, schedule | ⭐⭐⭐ |
| [5-direct-connect-vs-vpn.md](5-direct-connect-vs-vpn.md) | Direct Connect vs VPN — kết nối mạng cho hybrid storage | ⭐⭐⭐ |

---

## 🎯 Tại Sao Hybrid Storage Quan Trọng

### Thực Tế Doanh Nghiệp

Hầu hết các doanh nghiệp lớn không thể chuyển toàn bộ hạ tầng lên cloud trong một ngày. Họ phải vận hành song song:

```
Trước Hybrid Cloud:
├── Datacenter on-premises — hàng chục năm đầu tư
├── NAS (Network Attached Storage — Lưu Trữ Gắn Qua Mạng) cũ
├── Tape library — hàng nghìn băng vật lý
├── Ứng dụng legacy không thể migrate ngay
└── Compliance (Tuân Thủ Pháp Lý) yêu cầu giữ dữ liệu on-premises

Thách Thức:
├── Chi phí datacenter tăng liên tục
├── Khó scale (mở rộng) nhanh theo nhu cầu
├── Backup chậm, phức tạp, tốn kém
├── DR (Disaster Recovery — Khôi Phục Thảm Họa) yếu
└── Quản lý tape library tốn nhân lực
```

### AWS Hybrid Storage Giải Quyết Gì

```
Sau Hybrid Cloud với AWS Storage Gateway:
├── On-premises apps vẫn dùng NFS/SMB/iSCSI như bình thường
├── Dữ liệu tự động replicate (sao chép) lên S3/Glacier
├── Scale dung lượng vô hạn — không lo đầy disk
├── Chi phí lưu trữ giảm 60-80% so với on-premises NAS
└── DR tự động — RTO (Recovery Time Objective) từ ngày → giờ
```

---

## 🏗️ Kiến Trúc AWS Hybrid Storage Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    ON-PREMISES ENVIRONMENT                       │
│                                                                  │
│  ┌──────────────┐   NFS/SMB   ┌─────────────────────────────┐  │
│  │  File Server │ ──────────► │    AWS Storage Gateway      │  │
│  │  (Linux/Win) │             │    (VM hoặc Hardware)       │  │
│  └──────────────┘             │                             │  │
│                               │  ┌─────────────────────┐   │  │
│  ┌──────────────┐   iSCSI     │  │   Local Cache       │   │  │
│  │   iSCSI App  │ ──────────► │  │   (Cache Cục Bộ)    │   │  │
│  │  (Database)  │             │  └─────────────────────┘   │  │
│  └──────────────┘             └──────────┬──────────────────┘  │
│                                          │                       │
│  ┌──────────────┐   VTL (Virtual         │                       │
│  │  Backup App  │   Tape Library)        │                       │
│  │  (CommVault) │ ──────────────────────►│                       │
│  └──────────────┘                        │                       │
└──────────────────────────────────────────┼─────────────────────┘
                                           │ HTTPS/TLS
                        ┌──────────────────▼──────────────────────┐
                        │              AWS CLOUD                   │
                        │                                          │
                        │  ┌──────────┐  ┌───────────────────┐   │
                        │  │   S3     │  │  S3 Glacier /     │   │
                        │  │ Bucket   │  │  Deep Archive     │   │
                        │  └──────────┘  └───────────────────┘   │
                        │                                          │
                        │  ┌──────────────────────────────────┐   │
                        │  │         AWS Backup               │   │
                        │  │   (Quản Lý Backup Tập Trung)     │   │
                        │  └──────────────────────────────────┘   │
                        └──────────────────────────────────────────┘
```

---

## 🔄 Các Loại AWS Storage Gateway

### So Sánh Nhanh

| Loại | Giao Thức | Backend AWS | Use Case Chính |
|------|-----------|-------------|----------------|
| **File Gateway** | NFS, SMB | S3 | File server, NAS replacement |
| **Volume Gateway (Cached)** | iSCSI | S3 + local cache | Block storage với cloud backup |
| **Volume Gateway (Stored)** | iSCSI | S3 (async backup) | Local block với cloud DR |
| **Tape Gateway** | VTL (iSCSI) | S3 Glacier | Thay thế tape library vật lý |

### Khi Nào Dùng Loại Nào

```
Hỏi: Ứng dụng đang dùng giao thức gì?

├── NFS hoặc SMB (file server) → File Gateway
│
├── iSCSI (block storage, database)
│   ├── Cần truy cập toàn bộ dataset cục bộ → Volume Gateway (Stored Mode)
│   └── OK với cache một phần cục bộ → Volume Gateway (Cached Mode)
│
└── Backup software với VTL interface → Tape Gateway
```

---

## 🌐 DataSync vs Storage Gateway

| Tiêu Chí | DataSync | Storage Gateway |
|----------|----------|-----------------|
| **Mục đích chính** | Di chuyển / đồng bộ dữ liệu | Hybrid storage liên tục |
| **Mô hình** | Batch transfer (theo lô) | Real-time / ongoing |
| **Giao thức** | NFS, SMB, S3, HDFS | NFS, SMB, iSCSI, VTL |
| **Use case điển hình** | Migration một lần, sync định kỳ | Ứng dụng chạy liên tục |
| **Tốc độ** | Tối ưu cho throughput cao | Tối ưu cho latency thấp |
| **Agent** | DataSync agent (VM) | Storage Gateway (VM/HW) |

---

## 🔌 Kết Nối Mạng Cho Hybrid Storage

```
Tùy chọn kết nối on-premises ↔ AWS:

1. Internet thông thường
   ├── Chi phí thấp nhất
   ├── Băng thông không đảm bảo
   └── Phù hợp: workload nhẹ, backup không quan trọng

2. AWS Site-to-Site VPN (Virtual Private Network — Mạng Riêng Ảo)
   ├── Mã hóa IPSec
   ├── Băng thông ~1.25 Gbps tối đa
   └── Phù hợp: workload trung bình, DR

3. AWS Direct Connect (Kết Nối Trực Tiếp)
   ├── Đường truyền vật lý riêng 1/10/100 Gbps
   ├── Độ trễ thấp, ổn định
   └── Phù hợp: workload nhạy cảm latency, migration lớn
```

---

## 💡 Kiến Trúc Tham Khảo — Typical Enterprise Hybrid

### Kịch Bản: Bệnh Viện 500 Giường

```
Yêu cầu:
- PACS (Picture Archiving and Communication System — Hệ Thống Lưu Trữ Hình Ảnh Y Tế) 
  cần NFS shared storage cho scan images
- Compliance: lưu ảnh 7 năm tối thiểu
- DR: RTO < 4 giờ, RPO < 1 giờ
- Không thể thay thế ứng dụng legacy ngay

Giải Pháp Hybrid:
┌─────────────────────────────────────────┐
│  On-Premises Hospital                    │
│                                          │
│  PACS System ──NFS──► File Gateway VM   │
│  HIS System ──iSCSI─► Volume Gateway   │
│  Backup SW ───VTL───► Tape Gateway      │
│                           │              │
│  Direct Connect 1Gbps ────┘              │
└──────────────────────────────────────────┘
                │
         AWS Cloud
         ├── S3 Standard (1 tháng gần nhất)
         ├── S3 Standard-IA (1-12 tháng)
         └── S3 Glacier Deep Archive (1-7 năm)

Chi Phí So Sánh:
On-premises NAS 200TB = ~$150,000/năm (phần cứng + bảo trì)
File Gateway + S3 = ~$25,000/năm → tiết kiệm 83%
```

---

## 📊 Bảng So Sánh Tổng Thể

| Dịch Vụ | Deployment | Giao Thức | Dung Lượng | Độ Trễ |
|----------|------------|-----------|-----------|--------|
| File Gateway | VM/HW on-prem | NFS v3/v4, SMB | Vô hạn (S3) | ms–s |
| Volume Gateway Cached | VM/HW on-prem | iSCSI | Vô hạn (S3) | ms (cached data) |
| Volume Gateway Stored | VM/HW on-prem | iSCSI | Tối đa 32TB/vol | ms (local) |
| Tape Gateway | VM/HW on-prem | VTL/iSCSI | Vô hạn (Glacier) | Giờ (restore) |
| DataSync | Agent VM | NFS, SMB, S3 | Không giới hạn | Không áp dụng |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

1. **Storage Gateway là gì và tại sao cần nó?**
   → Cầu nối giữa on-premises và AWS, cho phép ứng dụng cũ dùng cloud storage mà không cần thay đổi code

2. **Phân biệt File Gateway, Volume Gateway, Tape Gateway?**
   → Giao thức khác nhau: NFS/SMB vs iSCSI vs VTL; use case khác nhau: file server vs block vs backup

3. **Volume Gateway Cached vs Stored khác nhau thế nào?**
   → Cached: dữ liệu chính trên S3, cache hot data cục bộ. Stored: dữ liệu chính cục bộ, async backup lên S3

4. **Khi nào dùng DataSync thay vì Storage Gateway?**
   → DataSync cho migration/sync batch; Storage Gateway cho ongoing hybrid operations

5. **Direct Connect vs VPN cho hybrid storage?**
   → Direct Connect: dedicated line, thấp latency, cho production. VPN: rẻ hơn, cho backup/DR

---

## 🔗 Điều Hướng

| Cần | Đọc |
|-----|-----|
| File server lên cloud | [1-file-gateway.md](1-file-gateway.md) |
| Block storage hybrid | [2-volume-gateway.md](2-volume-gateway.md) |
| Thay thế tape vật lý | [3-tape-gateway.md](3-tape-gateway.md) |
| Di chuyển dữ liệu tự động | [4-datasync.md](4-datasync.md) |
| Chọn kết nối mạng | [5-direct-connect-vs-vpn.md](5-direct-connect-vs-vpn.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
