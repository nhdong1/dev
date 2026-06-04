# Snow Family — Di Chuyển Dữ Liệu Quy Mô Lớn

> AWS Snow Family là bộ thiết bị vật lý chuyên dụng giúp di chuyển dữ liệu khổng lồ từ môi trường on-premises lên AWS khi băng thông mạng không đủ hoặc quá tốn kém. Ngoài migration, Snow Family còn hỗ trợ **edge computing** — xử lý dữ liệu tại chỗ ở các vùng xa, biên giới, hoặc không có kết nối mạng ổn định.

---

## Tại Sao Cần Snow Family?

Giả sử bạn có **500TB dữ liệu** cần đưa lên AWS:

```
Kết nối 1 Gbps lý thuyết → thực tế ~100 Mbps hiệu quả
500TB ÷ 100 Mbps = 46.296.296 giây ≈ 536 ngày ≈ 1,5 NĂM
```

Còn với Snowball Edge:

```
80TB/thiết bị → cần 7 thiết bị
Thời gian vật lý: đặt hàng + thu thập dữ liệu + ship = ~2–3 tuần
```

**Quy tắc ngón tay cái:** Nếu migration qua mạng mất hơn 1 tuần → cân nhắc Snow Family.

---

## Tổng Quan Các Thiết Bị

| Thiết Bị | Dung Lượng | Compute | Use Case Chính |
|---|---|---|---|
| **Snowcone** | 8TB HDD / 14TB SSD | 2 vCPU, 4GB RAM | Edge nhỏ, IoT, hiện trường |
| **Snowball Edge Storage Optimized** | 80TB | 40 vCPU, 80GB RAM | Migration lớn, backup |
| **Snowball Edge Compute Optimized** | 28TB NVMe | 104 vCPU, 416GB RAM | ML inference, video xử lý |
| **Snowmobile** | 100PB (xe container) | N/A | Di chuyển cả datacenter |

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Snow Family                          │
├─────────────┬──────────────────┬──────────────────┬────────────┤
│  Snowcone   │  Snowball Edge   │  Snowball Edge   │ Snowmobile │
│  8/14 TB    │  Storage (80TB)  │  Compute (28TB)  │  100 PB    │
│  2vCPU/4GB  │  40vCPU/80GB    │  104vCPU/416GB   │  (xe tải)  │
├─────────────┴──────────────────┴──────────────────┴────────────┤
│                     Quản Lý Bằng OpsHub                         │
├─────────────────────────────────────────────────────────────────┤
│   Dữ liệu được ship vật lý về AWS → import vào S3 / EBS        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quy Trình Sử Dụng Chung

```
1. Tạo Job trong AWS Console (Snow Family)
      ↓
2. AWS ship thiết bị đến địa chỉ của bạn (7–10 ngày làm việc)
      ↓
3. Kết nối thiết bị vào mạng nội bộ
      ↓
4. Dùng AWS OpsHub hoặc CLI để copy dữ liệu vào thiết bị
      ↓
5. Ship thiết bị trở lại AWS (prepaid shipping label có sẵn)
      ↓
6. AWS import dữ liệu vào S3 bucket (hoặc EBS snapshot)
      ↓
7. Nhận thông báo hoàn thành qua email / SNS
```

> **Bảo mật:** Tất cả dữ liệu trên thiết bị được mã hóa AES-256 với khóa do AWS KMS — Key Management Service — Dịch Vụ Quản Lý Khóa quản lý. Thiết bị có cơ chế **anti-tamper** — tự xóa dữ liệu nếu bị cạy mở.

---

## Nội Dung Chi Tiết

| File | Chủ Đề |
|---|---|
| [1-snowcone.md](./1-snowcone.md) | Snowcone — thiết bị nhỏ nhất, edge computing di động |
| [2-snowball-edge.md](./2-snowball-edge.md) | Snowball Edge — hai variant Storage và Compute |
| [3-snowmobile.md](./3-snowmobile.md) | Snowmobile — di chuyển 100PB, container xe tải |
| [4-snow-vs-datasync.md](./4-snow-vs-datasync.md) | Snow Family vs DataSync — decision tree chọn lựa |
| [5-opshub-management.md](./5-opshub-management.md) | OpsHub — giao diện GUI quản lý thiết bị Snow |

---

## Điểm Cốt Lõi Để Nhớ

- **Snow Family = vận chuyển vật lý** — không truyền qua mạng internet
- **Mã hóa bắt buộc** — AES-256, KMS managed, không thể tắt
- **Edge Computing** — chạy EC2 instance hoặc Lambda ngay trên thiết bị
- **OpsHub** — giao diện GUI thay thế cho CLI, dễ dùng hơn
- **Snowmobile** — cần liên hệ AWS Sales, không tự order được qua console

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-05-16
