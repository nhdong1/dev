# 🛡️ Disaster Recovery — Khôi Phục Sau Thảm Họa cho AWS Storage

> Hướng dẫn toàn diện về chiến lược DR (Disaster Recovery — Khôi Phục Thảm Họa) cho các dịch vụ lưu trữ AWS: thiết kế RPO/RTO, sao chép đa vùng, backup tập trung và tuân thủ quy định.

---

## 📚 Mục Lục Chủ Đề

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-rpo-rto-for-storage.md](1-rpo-rto-for-storage.md) | RPO/RTO — ý nghĩa, công thức thiết kế cho storage | ⭐⭐ |
| [2-s3-crr-for-dr.md](2-s3-crr-for-dr.md) | S3 Cross-Region Replication cho DR — kiến trúc & cấu hình | ⭐⭐⭐ |
| [3-ebs-snapshot-strategy.md](3-ebs-snapshot-strategy.md) | Chiến lược snapshot EBS — tần suất, retention, tự động hóa | ⭐⭐ |
| [4-aws-backup-service.md](4-aws-backup-service.md) | AWS Backup — backup tập trung, cross-account, vault lock | ⭐⭐⭐ |
| [5-backup-vault-lock.md](5-backup-vault-lock.md) | Backup Vault Lock — WORM cho backup, tuân thủ compliance | ⭐⭐⭐ |

---

## 🎯 Tại Sao Disaster Recovery Quan Trọng

### Thực Tế Rủi Ro

Mỗi hệ thống production đều phải đối mặt với các rủi ro không thể tránh khỏi:

```
Rủi Ro Tự Nhiên:
├── Thiên tai (lũ lụt, động đất, bão) → xóa sổ toàn bộ Availability Zone
├── Mất điện kéo dài → dữ liệu tạm thời bị mất
└── Hỏa hoạn trong datacenter → mất toàn bộ on-premises

Rủi Ro Con Người:
├── Xóa nhầm dữ liệu quan trọng (human error — lỗi con người)
├── Lỗi deployment — deploy code bug xóa dữ liệu
├── Ransomware — mã độc tống tiền mã hóa dữ liệu
└── Insider threat — nhân viên nội bộ cố tình phá hoại

Rủi Ro Kỹ Thuật:
├── Hardware failure — lỗi phần cứng bất ngờ
├── Software bug — bug trong ứng dụng gây mất/hỏng dữ liệu
├── Database corruption — cơ sở dữ liệu bị hỏng cấu trúc
└── Network partition — phân vùng mạng gây mất liên lạc
```

### Chi Phí Downtime

```
Tác Động Kinh Doanh:
├── E-commerce: ~$220,000/phút downtime (Amazon, 2018 estimate)
├── Fintech: mất giao dịch + phạt regulatory (quy định pháp lý)
├── Healthcare: rủi ro bệnh nhân + vi phạm HIPAA
└── SaaS B2B: mất khách hàng + phạt SLA (Service Level Agreement)
```

---

## 🏗️ Kiến Trúc DR Tổng Quan

### Bốn Chiến Lược DR (Từ Rẻ → Đắt, Từ Chậm → Nhanh)

```
1. Backup & Restore (Sao Lưu & Khôi Phục)
   ├── RTO: 24 giờ – vài ngày
   ├── RPO: 24 giờ
   ├── Chi phí: Thấp nhất
   └── Phù hợp: Hệ thống không yêu cầu cao về uptime

2. Pilot Light (Đèn Tín Hiệu — Hệ Thống Tối Thiểu)
   ├── RTO: 1–4 giờ
   ├── RPO: Vài phút – 1 giờ
   ├── Chi phí: Thấp (chỉ duy trì core services)
   └── Phù hợp: Production vừa, có thể chịu downtime ngắn

3. Warm Standby (Dự Phòng Nóng — Chạy Ở Quy Mô Nhỏ)
   ├── RTO: 15–60 phút
   ├── RPO: Vài giây – vài phút
   ├── Chi phí: Trung bình (~50% chi phí production)
   └── Phù hợp: Dịch vụ quan trọng, SLA 99.9%

4. Multi-Site Active/Active (Nhiều Điểm Hoạt Động Đồng Thời)
   ├── RTO: < 1 phút (thường là giây)
   ├── RPO: 0 hoặc gần 0
   ├── Chi phí: Cao nhất (~100% chi phí thêm)
   └── Phù hợp: Mission-critical (Hệ thống Thiết Yếu), SLA 99.99%+
```

### DR cho AWS Storage Services

```
Dịch Vụ        | Cơ Chế DR Chính                        | RPO Đạt Được
S3             | Cross-Region Replication (CRR)          | Gần thời gian thực
EBS            | Snapshots + AMI cross-region copy       | 1 giờ – 24 giờ
EFS            | Backup với AWS Backup, replication      | 1 giờ – 24 giờ
RDS/Aurora     | Read Replicas + Automated Backups       | 5 phút – 1 giờ
DynamoDB       | Global Tables (Bảng Toàn Cầu)           | < 1 giây
```

---

## 📐 Framework Thiết Kế DR

### Bước 1: Xác Định Mức Độ Quan Trọng (Criticality Tier)

```
Tier 1 — Mission Critical (Thiết Yếu):
├── Ảnh hưởng trực tiếp đến doanh thu hoặc an toàn
├── RPO: < 15 phút | RTO: < 1 giờ
└── Ví dụ: Payment data (dữ liệu thanh toán), core database

Tier 2 — Business Critical (Quan Trọng Kinh Doanh):
├── Gián đoạn nghiêm trọng nhưng không ngay lập tức
├── RPO: 1–4 giờ | RTO: 2–8 giờ
└── Ví dụ: User profiles, transaction logs

Tier 3 — Business Important (Quan Trọng Vừa):
├── Gián đoạn gây bất tiện nhưng có workaround
├── RPO: 4–24 giờ | RTO: 24–72 giờ
└── Ví dụ: Analytics data, reporting

Tier 4 — Administrative (Quản Trị):
├── Không ảnh hưởng đến khách hàng ngay
├── RPO: 24 giờ+ | RTO: Tuần
└── Ví dụ: Log archive, old backups
```

### Bước 2: Ánh Xạ Dịch Vụ → Chiến Lược

```python
def choose_dr_strategy(rto_hours, rpo_hours, budget_multiplier):
    if rto_hours < 1 and rpo_hours < 0.25:
        return "Multi-Site Active/Active"
    elif rto_hours < 4 and rpo_hours < 1:
        return "Warm Standby"
    elif rto_hours < 24 and rpo_hours < 4:
        return "Pilot Light"
    else:
        return "Backup & Restore"
```

### Bước 3: Kiểm Tra và Diễn Tập (DR Testing)

```
Tần Suất Kiểm Tra:
├── Hàng tuần: Backup integrity check (kiểm tra tính toàn vẹn backup)
├── Hàng tháng: Restore test (kiểm tra phục hồi) cho dữ liệu mẫu
├── Hàng quý: Full failover drill (diễn tập chuyển đổi dự phòng đầy đủ)
└── Hàng năm: Full DR simulation (mô phỏng thảm họa toàn diện)

Thước Đo Thành Công:
├── MTTR — Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình
├── MTBF — Mean Time Between Failures — Thời Gian Trung Bình Giữa Các Sự Cố
├── RTO thực tế vs RTO mục tiêu
└── RPO thực tế vs RPO mục tiêu
```

---

## 🔗 Liên Kết Nhanh

| Cần Làm | Đọc File |
|---------|----------|
| Tính RPO/RTO cho hệ thống của tôi | [1-rpo-rto-for-storage.md](1-rpo-rto-for-storage.md) |
| Setup S3 replication cho DR | [2-s3-crr-for-dr.md](2-s3-crr-for-dr.md) |
| Tự động backup EBS | [3-ebs-snapshot-strategy.md](3-ebs-snapshot-strategy.md) |
| Backup tập trung nhiều dịch vụ | [4-aws-backup-service.md](4-aws-backup-service.md) |
| Tuân thủ quy định về backup bất biến | [5-backup-vault-lock.md](5-backup-vault-lock.md) |

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp

1. **"Thiết kế DR cho hệ thống với RPO = 1 giờ, RTO = 4 giờ"** → Xem chiến lược Pilot Light + S3 CRR
2. **"Làm thế nào bảo vệ dữ liệu khỏi ransomware?"** → Xem Backup Vault Lock + Object Lock
3. **"S3 CRR và S3 Replication Time Control khác nhau như thế nào?"** → Xem file 2
4. **"AWS Backup quản lý gì và không quản lý gì?"** → Xem file 4
5. **"Tại sao không chỉ dùng S3 versioning cho DR?"** → Thảo luận RPO/RTO limitations

---

**Phần Tiếp Theo:** [09-storage-gateway/](../09-storage-gateway/) — Hybrid Cloud Storage
**Phần Trước:** [07-monitoring/](../07-monitoring/) — Monitoring & Observability
