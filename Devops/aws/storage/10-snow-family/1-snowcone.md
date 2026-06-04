# Snowcone — Thiết Bị Edge Computing Nhỏ Nhất

> AWS Snowcone là thiết bị Snow Family nhỏ gọn nhất, nặng chỉ **2,1 kg**, được thiết kế cho môi trường khắc nghiệt: chiến trường, tàu thuyền, mỏ khai thác, địa điểm thiếu điện và internet. Nó kết hợp khả năng **thu thập dữ liệu** và **xử lý tại biên** trong một hộp nhỏ hơn cả hộp giày.

---

## Thông Số Kỹ Thuật

| Thông Số | Snowcone HDD | Snowcone SSD |
|---|---|---|
| **Dung lượng lưu trữ** | 8TB HDD | 14TB SSD |
| **CPU** | 2 vCPU | 2 vCPU |
| **RAM** | 4 GB | 4 GB |
| **Cổng kết nối** | 2× 10GbE RJ-45 | 2× 10GbE RJ-45 |
| **Nguồn điện** | 45W (pin ngoài hoặc ổ cắm) | 45W |
| **Kích thước** | 227 × 148.6 × 82.65 mm | Tương đương |
| **Trọng lượng** | 2,1 kg | 2,1 kg |
| **Chứng nhận** | MIL-STD-810G (quân sự), IP65 (chống bụi/nước) | Tương đương |

---

## Edge Computing Trên Snowcone

Snowcone chạy được:

### EC2 — Amazon Elastic Compute Cloud

```
Instance type: sbe-c.large
- 2 vCPU
- 4 GB RAM
- Giới hạn: 1 instance cùng lúc

Dùng cho:
- Thu thập dữ liệu cảm biến IoT
- Xử lý video stream tại biên
- Chạy ứng dụng nhỏ không cần cloud
```

### AWS IoT Greengrass

```
Chạy Lambda functions tại biên:
- Xử lý dữ liệu trước khi gửi về cloud
- Giảm băng thông (chỉ gửi dữ liệu đã lọc)
- Hoạt động offline khi mất kết nối
```

---

## Các Cách Truyền Dữ Liệu Về AWS

### Cách 1: Ship Vật Lý (Offline Transfer)

```
Thu thập dữ liệu trên thiết bị
  ↓
Gói và ship về AWS Data Center
  ↓
AWS import vào S3 bucket chỉ định
  ↓
Nhận email xác nhận
```

**Thời gian:** 7–10 ngày làm việc (từ lúc ship đến khi dữ liệu vào S3)

### Cách 2: AWS DataSync qua Kết Nối Có Sẵn (Online Transfer)

```
Snowcone kết nối WiFi / LTE / 4G / vệ tinh
  ↓
AWS DataSync agent chạy sẵn trên thiết bị
  ↓
Tự động đồng bộ lên S3, EFS, FSx theo lịch
  ↓
Không cần ship thiết bị
```

**Yêu cầu:** Có kết nối internet (dù chậm cũng được — DataSync có retry tự động)

---

## Use Cases Điển Hình

### 1. Thu Thập Dữ Liệu Tại Hiện Trường (Field Data Collection)

```
Môi trường: Đoàn địa chất thăm dò dầu khí giữa sa mạc
Giải pháp:
  - Snowcone thu thập dữ liệu seismic (địa chấn)
  - Chạy ML model để lọc dữ liệu ngay tại chỗ
  - Ship về AWS mỗi 2–4 tuần
```

### 2. Sao Lưu Văn Phòng Chi Nhánh Nhỏ (Remote Office Backup)

```
Môi trường: Văn phòng chi nhánh 10 người, internet chậm
Giải pháp:
  - Snowcone làm NAS (Network Attached Storage) cục bộ
  - DataSync tự động backup lên S3 qua đêm
  - Không cần ship, không cần IT chuyên nghiệp
```

### 3. Hỗ Trợ Y Tế Khẩn Cấp (Emergency Medical Support)

```
Môi trường: Bệnh viện dã chiến, khu vực thiên tai
Giải pháp:
  - Lưu trữ hồ sơ bệnh nhân (DICOM images, EHR)
  - Chạy AI chẩn đoán hình ảnh offline
  - Đồng bộ về hệ thống trung tâm khi có kết nối
```

### 4. Giám Sát Hạ Tầng Từ Xa (Remote Infrastructure Monitoring)

```
Môi trường: Trạm điện gió ngoài khơi, tua-bin gió
Giải pháp:
  - Snowcone thu thập log từ cảm biến IoT
  - Xử lý anomaly detection tại biên
  - Ship hoặc upload qua satellite khi có cơ hội
```

---

## So Sánh Snowcone vs Snowball Edge

| Tiêu Chí | Snowcone | Snowball Edge |
|---|---|---|
| Dung lượng | 8–14 TB | 28–80 TB |
| Kích thước | Hộp giày (2,1 kg) | Vali nhỏ (22,3 kg) |
| Compute | 2 vCPU / 4GB RAM | 40–104 vCPU / 80–416GB RAM |
| Phù hợp | 1 người mang đi | Cần xe tải / forklift |
| Nguồn điện | Pin dự phòng hoặc USB-C | Cần ổ cắm điện |
| DataSync built-in | ✅ Có | ❌ Phải cài thêm |
| Giá thuê | Thấp hơn | Cao hơn |

---

## Giá Thuê (Tham Khảo)

```
Snowcone:
- On-demand: $0.03/GB dữ liệu transfer + phí thiết bị
- Phí thiết bị: ~$60/lần (10 ngày đầu miễn phí sau khi nhận)
- Sau 10 ngày: $15/ngày (Snowcone) | $30/ngày (Snowball Edge)

Lưu ý: Giá thực tế thay đổi theo region — kiểm tra AWS pricing calculator
```

---

## Bảo Mật

```
Mã hóa:       AES-256-bit (hardware encryption — mã hóa phần cứng)
Khóa:         AWS KMS — do KMS quản lý, không bao giờ lưu trên thiết bị
Anti-tamper:  Nếu mở nắp bất hợp lệ → dữ liệu tự xóa
Chain of custody: E-ink label — nhãn điện tử theo dõi hành trình thiết bị
Sau import:   AWS xóa sạch dữ liệu trên thiết bị theo NIST 800-88
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Khi nào chọn Snowcone thay vì DataSync thuần túy?**

> Chọn Snowcone khi: không có kết nối ổn định, dữ liệu quá lớn cho băng thông hiện có, hoặc cần xử lý tại biên (edge computing) mà không có server tại chỗ.

**Q: Snowcone có thể chạy Kubernetes không?**

> Không trực tiếp. Snowcone chỉ hỗ trợ EC2 instance (1 instance) và Greengrass. Không có EKS — Amazon Elastic Kubernetes Service — Dịch Vụ Kubernetes Quản Lý trên thiết bị này. Với Kubernetes tại biên, cần Snowball Edge Compute Optimized.

**Q: Làm sao đặt hàng Snowcone?**

> Qua AWS Management Console → Snow Family → Create Job. AWS ship thiết bị trong 7–10 ngày làm việc. Không cần mua, chỉ thuê theo lần dùng.

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-05-16  
**Liên Kết:** [README](./README.md) | [Snowball Edge](./2-snowball-edge.md) | [Snow vs DataSync](./4-snow-vs-datasync.md)
