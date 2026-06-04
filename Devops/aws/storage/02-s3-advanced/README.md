# S3 Nâng Cao — Advanced S3 Features

> Các tính năng S3 nâng cao dành cho production: Replication (Sao chép), Object Lock (Khóa Đối Tượng), Event Notifications (Thông Báo Sự Kiện), và Batch Operations (Thao Tác Hàng Loạt).

## 📚 Mục Lục Module

| File | Nội Dung | Độ Quan Trọng |
| ---- | -------- | ------------- |
| [1-lifecycle-policies.md](./1-lifecycle-policies.md) | Lifecycle Policies — Chính Sách Vòng Đời tự động chuyển tầng lưu trữ | ⭐⭐⭐ |
| [2-cross-region-replication.md](./2-cross-region-replication.md) | CRR — Cross-Region Replication — Sao Chép Liên Vùng | ⭐⭐⭐ |
| [3-same-region-replication.md](./3-same-region-replication.md) | SRR — Same-Region Replication — Sao Chép Cùng Vùng | ⭐⭐ |
| [4-s3-object-lock.md](./4-s3-object-lock.md) | Object Lock — WORM (Write Once Read Many — Ghi Một Lần Đọc Nhiều Lần) | ⭐⭐⭐ |
| [5-s3-event-notifications.md](./5-s3-event-notifications.md) | Event Notifications — Thông Báo Sự Kiện tích hợp Lambda/SQS/SNS | ⭐⭐⭐ |
| [6-s3-batch-operations.md](./6-s3-batch-operations.md) | Batch Operations — Thao Tác Hàng Loạt trên hàng tỷ object | ⭐⭐ |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- Thiết kế Lifecycle Policy tiết kiệm 60–80% chi phí lưu trữ dài hạn
- Cấu hình Cross-Region Replication cho Disaster Recovery và compliance
- Giải thích sự khác biệt Governance Mode vs Compliance Mode trong Object Lock
- Xây dựng event-driven pipeline (kiến trúc hướng sự kiện) với S3 Event Notifications
- Thực thi Batch Operations trên hàng tỷ object không cần code tùy chỉnh

---

## 🗺️ Tổng Quan Module

### Nhóm 1: Quản Lý Vòng Đời Dữ Liệu

```
Lifecycle Policies
├── Transition Actions (Hành Động Chuyển Tầng)
│   ├── Standard → Standard-IA (sau 30 ngày)
│   ├── Standard-IA → Glacier Instant (sau 90 ngày)
│   └── Glacier → Deep Archive (sau 180 ngày)
└── Expiration Actions (Hành Động Hết Hạn)
    ├── Xóa object sau N ngày
    ├── Xóa phiên bản cũ
    └── Xóa incomplete multipart uploads
```

### Nhóm 2: Sao Chép Dữ Liệu

```
Replication (Sao Chép)
├── CRR — Cross-Region Replication (Liên Vùng)
│   ├── DR — Disaster Recovery (Khôi Phục Thảm Họa)
│   ├── Giảm latency cho người dùng địa lý khác nhau
│   └── Tuân thủ quy định lưu trữ dữ liệu theo vùng
└── SRR — Same-Region Replication (Cùng Vùng)
    ├── Log aggregation (Tổng Hợp Nhật Ký)
    ├── Test/prod environment sync
    └── Backup trong cùng region
```

### Nhóm 3: Bảo Vệ Dữ Liệu Bất Biến

```
Object Lock (Khóa Đối Tượng)
├── Retention Mode (Chế Độ Lưu Giữ)
│   ├── Governance Mode — Chỉ admin đặc biệt mới xóa được
│   └── Compliance Mode — Không ai xóa được kể cả root
└── Legal Hold (Giữ Pháp Lý)
    └── Bật/tắt độc lập với retention period
```

### Nhóm 4: Tích Hợp & Tự Động Hóa

```
Event Notifications (Thông Báo Sự Kiện)
├── Destinations (Đích Đến)
│   ├── SQS — Simple Queue Service (Hàng Đợi Đơn Giản)
│   ├── SNS — Simple Notification Service (Thông Báo Đơn Giản)
│   ├── Lambda Function
│   └── EventBridge (Cầu Nối Sự Kiện)
└── Trigger Events (Sự Kiện Kích Hoạt)
    ├── s3:ObjectCreated:* (object được tạo)
    ├── s3:ObjectRemoved:* (object bị xóa)
    └── s3:Replication:* (sự kiện replication)

Batch Operations (Thao Tác Hàng Loạt)
├── Copy objects (sao chép object)
├── Restore from Glacier (khôi phục từ Glacier)
├── Invoke Lambda per object
├── Replace ACLs
└── Apply Object Lock retention
```

---

## ⚡ Tóm Tắt Nhanh — Quick Reference

### Lifecycle Transitions — Thứ Tự Chuyển Tầng Hợp Lệ

```
Standard → Standard-IA → One Zone-IA → Glacier Instant → Glacier Flexible → Deep Archive
                    ↑
               Không thể đi ngược chiều
```

Thời gian tối thiểu ở mỗi tầng trước khi chuyển tiếp:
- Standard → Standard-IA: **tối thiểu 30 ngày**
- Standard-IA → Glacier Instant: **tối thiểu 90 ngày tổng**
- Glacier Instant → Glacier Flexible: **không giới hạn thêm**

### Replication — Điều Kiện Bắt Buộc

```
✅ Versioning phải được bật ở CŨNG bucket nguồn và bucket đích
✅ IAM role phải có quyền đọc nguồn và ghi đích
✅ Replication chỉ áp dụng cho object MỚI (sau khi bật)
❌ Object đã tồn tại KHÔNG được replicate tự động
   → Dùng S3 Batch Replication cho object cũ
```

### Object Lock — So Sánh Nhanh

| Đặc Điểm | Governance Mode | Compliance Mode |
| --------- | --------------- | --------------- |
| Ai có thể xóa? | Admin có quyền đặc biệt | **Không ai** (kể cả root) |
| Thay đổi retention period? | Có thể rút ngắn | Chỉ có thể kéo dài |
| Mục đích | Kiểm tra & thử nghiệm | Compliance cứng (SEC, FINRA) |
| Mức độ bảo vệ | Cao | Tuyệt đối |

---

## 🔗 Kết Nối Với Các Module Khác

| Module | Liên Quan |
| ------ | --------- |
| [01-s3-fundamentals/](../01-s3-fundamentals/) | Versioning là điều kiện tiên quyết cho Replication |
| [05-security/](../05-security/) | Object Lock liên quan đến compliance encryption |
| [06-cost-optimization/](../06-cost-optimization/) | Lifecycle policies là công cụ tối ưu chi phí chính |
| [08-disaster-recovery/](../08-disaster-recovery/) | CRR là nền tảng chiến lược DR của S3 |
| [07-monitoring/](../07-monitoring/) | CloudWatch metrics cho replication lag, batch jobs |

---

## 📖 Lộ Trình Học Module Này

```
Bước 1: Lifecycle Policies     → Tiết kiệm chi phí ngay lập tức
Bước 2: CRR/SRR                → Hiểu replication trước Object Lock
Bước 3: Object Lock            → Bảo vệ dữ liệu bất biến
Bước 4: Event Notifications    → Xây dựng event-driven architecture
Bước 5: Batch Operations       → Tự động hóa tác vụ quy mô lớn
```

**Thời gian ước tính:** 5–7 giờ để nắm vững toàn bộ module

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
