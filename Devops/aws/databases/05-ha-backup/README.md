# High Availability & Backup — Tính Sẵn Sàng Cao & Sao Lưu Trên AWS

> HA (High Availability — Tính Sẵn Sàng Cao) và Backup (Sao Lưu) là hai trụ cột không thể thiếu trong vận hành database production. HA đảm bảo hệ thống luôn hoạt động dù có sự cố; Backup đảm bảo dữ liệu không bị mất và có thể khôi phục khi cần. Cùng nhau, chúng tạo nên nền tảng cho một chiến lược Disaster Recovery (DR — Khôi Phục Thảm Họa) toàn diện.

## 📚 Mục Lục Chủ Đề

| File | Nội Dung | Trạng Thái |
|------|----------|-----------|
| [README.md](./README.md) | Tổng quan HA & Backup, kiến trúc, lộ trình học | ✅ |
| [1-rpo-rto.md](./1-rpo-rto.md) | RPO, RTO — yêu cầu kinh doanh, thiết kế chiến lược | ✅ |
| [2-automated-backups.md](./2-automated-backups.md) | RDS Automated Backups, retention policy, cấu hình | ✅ |
| [3-snapshots.md](./3-snapshots.md) | Manual Snapshots, cross-region copy, lifecycle | ✅ |
| [4-pitr.md](./4-pitr.md) | Point-in-Time Recovery, quy trình khôi phục | ✅ |
| [5-disaster-recovery.md](./5-disaster-recovery.md) | DR strategies, multi-region failover, runbook | ✅ |

---

## 🎯 Tại Sao HA & Backup Quan Trọng?

### Chi Phí Của Downtime (Thời Gian Ngừng Hoạt Động)

```
Theo nghiên cứu của Gartner (2023):
  - Trung bình: $5,600/phút downtime
  - Tài chính/thương mại điện tử: $100,000+/phút
  - Mất dữ liệu: Ảnh hưởng pháp lý, mất niềm tin khách hàng
```

### Hai Chiều Bảo Vệ

```
┌─────────────────────────────────────────────────────────────────┐
│                    Bảo Vệ Database Production                    │
│                                                                  │
│   ┌──────────────────────────┐   ┌──────────────────────────┐   │
│   │   HIGH AVAILABILITY      │   │        BACKUP            │   │
│   │   (Tính Sẵn Sàng Cao)   │   │       (Sao Lưu)          │   │
│   │                          │   │                          │   │
│   │  Mục tiêu: Không bao     │   │  Mục tiêu: Phục hồi     │   │
│   │  giờ bị downtime         │   │  khi mất dữ liệu         │   │
│   │                          │   │                          │   │
│   │  Cơ chế:                 │   │  Cơ chế:                 │   │
│   │  - Multi-AZ Failover     │   │  - Automated Backups     │   │
│   │  - Read Replicas         │   │  - Manual Snapshots      │   │
│   │  - Aurora Cluster        │   │  - PITR                  │   │
│   │                          │   │  - Cross-region Copy     │   │
│   │  Phục hồi: Tự động       │   │  Phục hồi: Thủ công/     │   │
│   │  (giây đến phút)         │   │  bán tự động             │   │
│   └──────────────────────────┘   └──────────────────────────┘   │
│                                                                  │
│                 Cùng nhau tạo nên DR Strategy                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Kiến Trúc HA Tổng Quan

### RDS Multi-AZ (Đa Vùng Sẵn Sàng)

```
                   AWS Region (Vùng AWS)
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   AZ-a (Vùng Sẵn Sàng A)    AZ-b (Vùng Sẵn Sàng B)        │
│   ┌──────────────────┐       ┌──────────────────┐           │
│   │   RDS Primary    │       │   RDS Standby    │           │
│   │  (Read + Write)  │──────►│   (Synchronous   │           │
│   │                  │       │    Replica)       │           │
│   └──────────────────┘       └──────────────────┘           │
│           │  Automatic failover (60-120 giây)  ▲            │
│           └────────────────────────────────────┘            │
│                                                              │
│   DNS endpoint không thay đổi → App tự động kết nối lại    │
└──────────────────────────────────────────────────────────────┘
```

### Aurora Cluster (Cụm Aurora)

```
                   Aurora Cluster
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Writer Endpoint          Reader Endpoint                   │
│  (Điểm Truy Cập Ghi)      (Điểm Truy Cập Đọc)              │
│       │                         │                           │
│       ▼                         ▼                           │
│  ┌──────────┐         ┌──────────────────────┐             │
│  │  Primary │         │  Reader 1 | Reader 2 │             │
│  │ Instance │         │  (Read Replicas)      │             │
│  └────┬─────┘         └──────────────────────┘             │
│       │                          │                          │
│       └──────────┬───────────────┘                         │
│                  ▼                                          │
│     ┌─────────────────────────────────┐                    │
│     │   Aurora Shared Storage         │                    │
│     │   (Lưu Trữ Dùng Chung)         │                    │
│     │   6 bản sao × 2 AZ = 6 copies  │                    │
│     │   Tự lành lỗi (Self-healing)    │                    │
│     └─────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────┘
```

### Kiến Trúc Backup Toàn Diện

```
                    Backup Strategy (Chiến Lược Sao Lưu)
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Thời Gian Thực                  Hàng Ngày           Thủ Công   │
│  ┌──────────────┐         ┌──────────────┐     ┌──────────────┐ │
│  │ Transaction  │         │  Automated   │     │   Manual     │ │
│  │    Logs      │         │   Backup     │     │  Snapshot    │ │
│  │ (Nhật Ký GD) │         │  (Sao Lưu   │     │ (Ảnh Chụp)  │ │
│  │              │         │   Tự Động)   │     │              │ │
│  │ Cho phép     │         │              │     │ Giữ vĩnh    │ │
│  │ PITR         │         │ Retention:   │     │ viễn        │ │
│  │ (Khôi phục   │         │ 1-35 ngày    │     │             │ │
│  │ theo phút)   │         │              │     │             │ │
│  └──────┬───────┘         └──────┬───────┘     └──────┬──────┘ │
│         │                        │                     │        │
│         └────────────────────────┼─────────────────────┘        │
│                                  ▼                               │
│                    ┌─────────────────────────┐                  │
│                    │   S3 (Object Storage)    │                  │
│                    │   Cross-Region Copy      │                  │
│                    │   (Sao Chép Xuyên Vùng) │                  │
│                    └─────────────────────────┘                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 So Sánh Các Cơ Chế HA

| Cơ Chế | Failover Time | RPO | RTO | Chi Phí | Dùng Cho |
|--------|--------------|-----|-----|---------|---------|
| **RDS Multi-AZ** | 60-120 giây | ~0 | 1-2 phút | 2x instance | Production OLTP |
| **Aurora Multi-AZ** | 15-30 giây | ~0 | < 1 phút | 1.1x storage | High-traffic apps |
| **Read Replica Promoted** | 5-10 phút (thủ công) | Vài giây | 5-15 phút | Thấp hơn Multi-AZ | DR thứ cấp |
| **Cross-Region Replica** | 15-30 phút (thủ công) | Vài phút | 30-60 phút | Thêm egress | DR region khác |
| **Aurora Global Database** | < 1 phút (tự động) | < 1 giây | 1-2 phút | Premium | Global apps |

---

## 🔑 Khái Niệm Cốt Lõi

### RPO & RTO — Hai Chỉ Số Quan Trọng Nhất

```
Timeline (Dòng Thời Gian):

  Last Backup      Disaster      Recovery Complete
       │              │                │
       ▼              ▼                ▼
───────●──────────────●────────────────●──────────►
       │◄────────────►│◄───────────────►
            RPO              RTO
     (Recovery Point    (Recovery Time
       Objective)          Objective)
       "Tối đa mất        "Tối đa mất
        bao nhiêu          bao nhiêu
        dữ liệu?"          thời gian?"
```

- **RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục):** Khoảng thời gian dữ liệu tối đa có thể bị mất
- **RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục):** Thời gian tối đa để hệ thống hoạt động trở lại

### Các Loại Sao Lưu

| Loại | Mô Tả | Tần Suất | Lưu Trữ |
|------|--------|----------|---------|
| **Automated Backup** (Sao Lưu Tự Động) | Daily full backup + transaction logs | Hàng ngày | S3 (AWS quản lý) |
| **Manual Snapshot** (Ảnh Chụp Thủ Công) | Full backup do người dùng kích hoạt | Theo nhu cầu | S3 (User quản lý) |
| **PITR** (Point-in-Time Recovery) | Khôi phục theo thời điểm bất kỳ | Liên tục (5 phút) | S3 |

---

## 🗺️ Lộ Trình Học HA & Backup

### Bước 1 — Nền Tảng (Cần Biết)
1. Hiểu RPO/RTO và liên kết với business requirements
2. Nắm Multi-AZ hoạt động như thế nào và tại sao
3. Phân biệt Automated Backup và Manual Snapshot

### Bước 2 — Thực Hành (Kỹ Năng Cốt Lõi)
1. Cấu hình Automated Backup với retention policy phù hợp
2. Tạo và restore Manual Snapshot
3. Thực hiện PITR về một thời điểm cụ thể

### Bước 3 — Nâng Cao (Production-Ready)
1. Thiết kế DR strategy đầy đủ với RTO/RPO cụ thể
2. Cross-region backup và failover
3. Automation với AWS Backup, Lambda, EventBridge

---

## 💡 Những Điều Hay Bị Nhầm Lẫn

### Multi-AZ vs Read Replica — Mục Đích Khác Nhau

| | Multi-AZ | Read Replica |
|-|----------|-------------|
| **Mục đích chính** | HA — tránh downtime | Scale đọc — tăng throughput |
| **Đồng bộ** | Synchronous (Đồng Bộ) | Asynchronous (Bất Đồng Bộ) |
| **Standby có đọc được không?** | ❌ Không (chỉ failover) | ✅ Có |
| **Tự động failover** | ✅ Tự động | ❌ Phải promote thủ công |
| **Replica lag** | Không có (sync) | Có thể có vài giây |
| **Cross-region** | ❌ Cùng region | ✅ Cross-region được |

### Automated Backup vs Snapshot — Khi Nào Dùng Gì?

```
Automated Backup → Dùng cho: Khôi phục hàng ngày, PITR
                 → Bị xóa khi: Xóa DB instance
                 → Giữ tối đa: 35 ngày

Manual Snapshot → Dùng cho: Trước maintenance, milestone quan trọng
               → Bị xóa khi: Bạn xóa thủ công
               → Giữ: Vĩnh viễn (đến khi xóa)
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

1. **RPO và RTO là gì? Hệ thống của bạn thiết kế để đáp ứng RPO/RTO nào?**
2. **Phân biệt Multi-AZ và Read Replica — khi nào dùng gì?**
3. **Automated Backup khác Manual Snapshot như thế nào?**
4. **Giải thích Point-in-Time Recovery — khi nào cần dùng?**
5. **Thiết kế DR strategy cho ứng dụng có RPO = 1 phút, RTO = 5 phút?**
6. **Tại sao Backup không được test là Backup chưa tồn tại?**

---

## 🔗 Điều Hướng Nhanh

| Chủ Đề | File |
|--------|------|
| RPO, RTO và thiết kế chiến lược | [1-rpo-rto.md](./1-rpo-rto.md) |
| Automated Backups — cấu hình, retention | [2-automated-backups.md](./2-automated-backups.md) |
| Manual Snapshots — tạo, copy, restore | [3-snapshots.md](./3-snapshots.md) |
| Point-in-Time Recovery | [4-pitr.md](./4-pitr.md) |
| Disaster Recovery — chiến lược, runbook | [5-disaster-recovery.md](./5-disaster-recovery.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
