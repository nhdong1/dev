# RPO & RTO cho AWS Storage — Thiết Kế Mục Tiêu Khôi Phục

> RPO — Recovery Point Objective — Mục Tiêu Điểm Khôi Phục và RTO — Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục là hai thước đo nền tảng trong mọi kế hoạch DR (Disaster Recovery — Khôi Phục Thảm Họa). Phần này giải thích cách tính toán, áp dụng vào AWS Storage và thiết kế kiến trúc phù hợp.

---

## 1. RPO — Recovery Point Objective — Mục Tiêu Điểm Khôi Phục

### Định Nghĩa

```
RPO = Lượng dữ liệu tối đa có thể chấp nhận bị mất, tính bằng thời gian

Câu hỏi cốt lõi:
"Nếu thảm họa xảy ra ngay lúc này, chúng ta có thể chấp nhận
 mất dữ liệu của bao nhiêu phút/giờ/ngày qua?"
```

### Minh Họa

```
Timeline:
08:00 ─── Backup lần cuối
  │
  ├── 09:00 ─── Giao dịch diễn ra bình thường
  ├── 10:00 ─── Giao dịch tiếp tục
  ├── 11:00 ─── Giao dịch tiếp tục
  │
11:30 ─── THẢM HỌA XẢY RA (sự cố)
  │
  └── Khôi phục từ backup 08:00
      → Mất 3.5 giờ dữ liệu
      → RPO thực tế = 3.5 giờ

Nếu SLA yêu cầu RPO ≤ 1 giờ → KHÔNG ĐẠT
```

### RPO theo Loại Dữ Liệu

| Loại Dữ Liệu | RPO Điển Hình | Lý Do |
|--------------|---------------|-------|
| Giao dịch tài chính | < 1 giây | Mất tiền trực tiếp |
| Hồ sơ khách hàng | < 15 phút | Ảnh hưởng kinh doanh lớn |
| Nội dung CMS | < 1 giờ | Có thể nhập lại thủ công |
| Log phân tích | < 24 giờ | Không ảnh hưởng realtime |
| Archive tuân thủ | 24 giờ | Ít thay đổi, có thể chấp nhận |

---

## 2. RTO — Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục

### Định Nghĩa

```
RTO = Thời gian tối đa cho phép từ khi thảm họa xảy ra
      đến khi hệ thống trở lại hoạt động bình thường

Câu hỏi cốt lõi:
"Hệ thống có thể ngừng hoạt động tối đa bao lâu mà
 doanh nghiệp vẫn có thể tồn tại được?"
```

### Các Giai Đoạn Của RTO

```
Thảm Họa Xảy Ra
     │
     ├─ Phát Hiện (Detection)
     │   Thời gian: 5–30 phút
     │   Phụ thuộc: Monitoring, alerting
     │
     ├─ Đánh Giá (Assessment)
     │   Thời gian: 15–60 phút
     │   Phụ thuộc: Runbook rõ ràng, team experience
     │
     ├─ Kích Hoạt DR (DR Activation)
     │   Thời gian: 15 phút – 4 giờ
     │   Phụ thuộc: Chiến lược DR, mức độ tự động hóa
     │
     ├─ Khôi Phục Dữ Liệu (Data Recovery)
     │   Thời gian: 30 phút – 24 giờ
     │   Phụ thuộc: Lượng dữ liệu, storage class, bandwidth
     │
     └─ Kiểm Tra & Xác Nhận (Validation)
         Thời gian: 15–60 phút
         Phụ thuộc: Test cases, approval process

Tổng RTO = Tổng tất cả giai đoạn trên
```

---

## 3. Mối Quan Hệ RPO vs RTO vs Chi Phí

### Đánh Đổi Kinh Điển

```
         Chi phí
            ▲
            │           ● Multi-Site Active/Active
            │                (RPO≈0, RTO<1min)
            │         ● Warm Standby
            │              (RPO<1h, RTO<1h)
            │       ● Pilot Light
            │            (RPO<4h, RTO<4h)
            │     ● Backup & Restore
            │          (RPO<24h, RTO<24h)
            │
            └──────────────────────────────▶
                 RPO/RTO giảm (tốt hơn)

Quy tắc: RPO và RTO giảm 10x → chi phí tăng 3–5x
```

### Bảng Chiến Lược

| Chiến Lược | RPO | RTO | Chi Phí Tương Đối |
|-----------|-----|-----|-------------------|
| Backup & Restore | 24 giờ | 24 giờ | 1× |
| Pilot Light | 1–4 giờ | 1–4 giờ | 3–5× |
| Warm Standby | 15 phút | 30–60 phút | 10–15× |
| Multi-Site Active/Active | < 1 giây | < 1 phút | 20–50× |

---

## 4. RPO/RTO cho Từng Dịch Vụ AWS Storage

### Amazon S3 — Simple Storage Service

```
Cơ Chế Built-in:
├── 11 chín (99.999999999%) durability — độ bền trong cùng region
├── 3 AZ (Availability Zone — Vùng Khả Dụng) replication mặc định
└── Không tự động cross-region replication

RPO với Cross-Region Replication (CRR — Sao Chép Liên Vùng):
├── Đa số object: Sao chép trong vài giây đến vài phút
├── Với S3 RTC (Replication Time Control — Kiểm Soát Thời Gian Sao Chép):
│   └── 99.99% object được sao chép trong 15 phút → RPO ≤ 15 phút
└── Không có RTC: RPO = vài phút đến vài giờ (không đảm bảo)

RTO với Cross-Region failover:
├── Read: Ngay lập tức — chuyển DNS hoặc cập nhật endpoint
├── Write: Phụ thuộc Route 53 TTL (Time To Live) + health check
└── Tổng RTO điển hình: 5–15 phút
```

### Amazon EBS — Elastic Block Store

```
Giới Hạn Built-in:
├── EBS volume gắn với 1 Availability Zone duy nhất
├── Không tự động cross-AZ hoặc cross-region replication
└── Dữ liệu mất nếu toàn bộ AZ bị ảnh hưởng

RPO với Snapshots (Ảnh Chụp Nhanh):
├── Snapshot mỗi 1 giờ → RPO = 1 giờ
├── Snapshot mỗi 24 giờ → RPO = 24 giờ
└── EBS Multi-Volume Consistent Snapshots → RPO nhất quán

RTO với Snapshot Restore:
├── Tạo volume mới từ snapshot: 1–5 phút
├── Fast Snapshot Restore (FSR — Khôi Phục Nhanh): Dưới 1 phút
└── Sao chép snapshot sang region khác: 15–60 phút tùy kích thước
```

### Amazon EFS — Elastic File System

```
Cơ Chế Built-in:
├── Multi-AZ trong cùng region — tự động
├── 11 chín durability (Standard storage class)
└── Không tự động cross-region replication

RPO với AWS Backup:
├── Backup policy linh hoạt (hourly/daily/weekly)
├── RPO điển hình: 1–24 giờ tùy tần suất backup
└── EFS Replication (tính năng native): RPO ~ vài phút

RTO với Restore:
├── Restore EFS backup: 1–4 giờ tùy kích thước
└── EFS Replication failover: < 5 phút
```

---

## 5. Thiết Kế RPO/RTO Thực Tế — Ví Dụ Case Study

### Case Study 1: E-Commerce Platform

```
Yêu Cầu Kinh Doanh:
├── Không mất đơn hàng đã xác nhận
├── Khách hàng chờ tối đa 30 phút
└── Budget: $5,000/tháng thêm cho DR

Phân Tích:
├── RPO = 0 cho order data (không chấp nhận mất đơn hàng)
├── RTO = 30 phút
└── RPO = 1 giờ cho product catalog (có thể tái tạo)

Giải Pháp:
├── RDS Aurora Global Database → RPO < 1 giây cho orders
├── S3 CRR với RTC → RPO ≤ 15 phút cho product images
├── DynamoDB Global Tables → RPO < 1 giây cho session data
└── Warm Standby ở us-west-2 khi primary là us-east-1
```

### Case Study 2: Financial Institution (Tổ Chức Tài Chính)

```
Yêu Cầu Regulatory (Pháp Lý):
├── MAS TRM (Monetary Authority of Singapore)
├── RPO ≤ 4 giờ cho tất cả hệ thống
├── RTO ≤ 4 giờ cho hệ thống critical
└── Diễn tập DR hàng năm, có documentation

Giải Pháp AWS:
├── S3 CRR với S3 Object Lock (Khóa Đối Tượng) → compliance backup
├── EBS snapshots mỗi giờ với AWS DLM (Data Lifecycle Manager)
├── AWS Backup với cross-account (liên tài khoản) và vault lock
└── CloudTrail logs nhân rộng sang S3 DR bucket → audit trail
```

---

## 6. Công Thức Tính RPO/RTO

### Tính RPO Tối Ưu

```
RPO tối ưu = min(
    Chi phí mất dữ liệu mỗi phút × Phút mất dữ liệu,
    Chi phí hạ tầng DR thêm mỗi tháng
)

Ví dụ:
├── Mất $1,000/phút downtime
├── DR với RPO = 1 giờ tốn $2,000/tháng thêm
├── Trung bình 1 sự cố/năm, mất 3 giờ dữ liệu
│   → Chi phí mất dữ liệu/năm = $1,000 × 180 phút = $180,000
├── Chi phí DR/năm = $2,000 × 12 = $24,000
└── Kết luận: Đầu tư DR có ROI dương rõ ràng
```

### Kiểm Tra RTO Thực Tế

```bash
# Đo thời gian restore EBS snapshot thực tế
START=$(date +%s)

# Tạo volume từ snapshot
aws ec2 create-volume \
    --snapshot-id snap-0abc123def456 \
    --availability-zone us-east-1a \
    --volume-type gp3

# Đợi volume available
aws ec2 wait volume-available --volume-ids vol-newvolumeid

END=$(date +%s)
echo "Restore time: $((END - START)) seconds"
# Ghi lại → cập nhật RTO trong runbook
```

---

## 7. DR Testing — Kiểm Tra Kế Hoạch DR

### Tại Sao Phải Test?

> "Một kế hoạch DR chưa được kiểm tra không phải là kế hoạch — đó là giả thuyết."

```
Vấn Đề Thường Gặp Khi Không Test:
├── Backup tồn tại nhưng không restore được (corrupted — bị hỏng)
├── RTO ước tính 2 giờ nhưng thực tế mất 12 giờ
├── Team không biết runbook ở đâu
├── Credential (thông tin xác thực) hết hạn khi cần dùng khẩn
└── Dependencies (phụ thuộc) không được document — DB lên trước app
```

### Lịch Test Khuyến Nghị

```
Hàng Ngày (Automated):
├── Verify backup completion (xác nhận backup hoàn thành)
├── Check replication lag (độ trễ sao chép) < threshold
└── Alert nếu RPO bị vi phạm

Hàng Tuần:
├── Restore 1 file ngẫu nhiên từ S3 backup
├── Xác nhận snapshot EBS được tạo đúng lịch
└── Review CloudWatch DR metrics

Hàng Tháng:
├── Restore database vào môi trường isolated (cô lập)
├── Test failover DNS sang DR region
└── Đo RTO thực tế và so sánh với mục tiêu

Hàng Quý:
├── Full DR drill (diễn tập DR đầy đủ) — failover hoàn toàn
├── Cập nhật runbook dựa trên kết quả
└── Review và điều chỉnh RPO/RTO targets
```

---

## 8. Checklist Thiết Kế RPO/RTO

```
□ Đã phân loại dữ liệu theo criticality tier (4 tầng)
□ Đã tính chi phí downtime mỗi giờ cho từng tier
□ Đã chọn chiến lược DR phù hợp với budget
□ Đã implement và test backup cho từng dịch vụ
□ Đã document RTO từng bước (detection → restore → validation)
□ Đã test restore thực tế, không chỉ lý thuyết
□ Đã setup monitoring cho replication lag và backup failures
□ Đã có runbook rõ ràng, team biết cách thực hiện
□ Đã schedule DR drill định kỳ (hàng quý)
□ Đã đáp ứng yêu cầu regulatory nếu có (HIPAA, PCI-DSS, MAS TRM)
```

---

**File Tiếp Theo:** [2-s3-crr-for-dr.md](2-s3-crr-for-dr.md) — Triển khai S3 Cross-Region Replication cho DR
