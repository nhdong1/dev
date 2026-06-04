# Snowmobile — Di Chuyển Cả Datacenter Lên AWS

> AWS Snowmobile là một **container 45 foot (13,7m) gắn trên xe tải bán tải**, chứa hệ thống lưu trữ có dung lượng lên đến **100 Petabyte**. Đây là giải pháp di chuyển dữ liệu lớn nhất thế giới mà AWS cung cấp — dùng cho các công ty cần di chuyển toàn bộ datacenter lên đám mây.

---

## Tại Sao Snowmobile Tồn Tại?

```
Bài toán: Di chuyển 100 Exabyte (10 datacenter lớn)

Với mạng 10 Gbps:
  100 PB ÷ 10 Gbps = 92,6 ngày × 1.000 = 92.600 ngày = 253 NĂM

Với Snowmobile:
  100 PB / xe × 1.000 PB = 10 xe Snowmobile
  Thời gian thực tế: vài tháng thay vì hàng thế kỷ
```

**Ngưỡng quyết định:** Khi dữ liệu vượt **10 PB** → Snowmobile thường hiệu quả hơn nhiều Snowball Edge.

---

## Thông Số Kỹ Thuật

```
Thân xe:
  - Container dài: 45 foot (13,7 meter)
  - Xe kéo: Semi-truck (xe tải đầu kéo)
  - Nguồn điện: Generator diesel tích hợp + kết nối lưới điện
  - Nhiệt độ: Hệ thống làm mát chuyên dụng

Lưu trữ:
  - Dung lượng tối đa: 100 PB (Petabyte) mỗi xe
  - Công nghệ: Rack server với ổ đĩa SAS/SATA dày đặc
  - Giao thức: Xuất ra qua Multiple 40 GbE hoặc 1/10 GbE

Kết nối tại datacenter:
  - Kết nối qua cáp quang trực tiếp vào Switch nội bộ
  - Tốc độ tổng hợp: lên đến 1 Tbps (Terabit per second)
  - Cần phòng có nguồn điện 3-phase (điện 3 pha công nghiệp)

Bảo mật di chuyển:
  - GPS tracking (định vị vệ tinh) 24/7
  - Camera an ninh quanh xe
  - Đội bảo vệ vũ trang (tùy khu vực)
  - Alarm system (hệ thống báo động)
  - Chỉ hoạt động ban ngày (tùy địa điểm)
```

---

## Quy Trình Triển Khai

```
Bước 1 — Liên Hệ AWS Sales (Không tự order được)
  - Không có button "Order Snowmobile" trong Console
  - Cần gặp AWS Enterprise team
  - Thảo luận về site requirements (yêu cầu địa điểm)

Bước 2 — AWS Khảo Sát Datacenter
  - AWS engineer đến khảo sát thực địa
  - Kiểm tra: nguồn điện, kết nối mạng, đường vào xe
  - Ký NDA — Non-Disclosure Agreement — Thỏa Thuận Bảo Mật

Bước 3 — Chuẩn Bị Hạ Tầng Tại Chỗ
  - Cung cấp 150 kW điện 3-phase (điện 3 pha)
  - Cáp quang từ switch core đến vị trí đậu xe
  - Khu vực đậu xe đủ rộng + mái che (khuyến nghị)

Bước 4 — AWS Ship Snowmobile Đến
  - Xe tải đến theo lịch đã thỏa thuận
  - AWS engineer setup và test kết nối
  - Customer bắt đầu copy dữ liệu qua mạng cục bộ

Bước 5 — Copy Dữ Liệu (Tuần đến Tháng)
  - Tốc độ thực tế: ~1 Gbps × số cổng kết nối
  - Dùng AWS Snowmobile client để manage job
  - Có thể copy song song từ nhiều server

Bước 6 — Hoàn Thành và Vận Chuyển Về AWS
  - AWS seal (niêm phong) container
  - Xe tải đưa về AWS Region datacenter
  - AWS import dữ liệu vào S3 Buckets chỉ định
  - Customer nhận thông báo hoàn thành

Thời gian tổng: Vài tuần đến vài tháng (tùy khối lượng)
```

---

## Bảo Mật Đa Lớp

### Bảo Mật Dữ Liệu

```
Tất cả dữ liệu được mã hóa trước khi ghi vào ổ đĩa:
  - Encryption: AES-256 hardware encryption
  - Key management: AWS KMS — khóa không bao giờ rời datacenter AWS
  - Key rotation: Tự động sau mỗi job

Riêng biệt logic từng khách hàng:
  - Mỗi khách hàng có encryption key riêng
  - Không có khả năng cross-contamination (lây nhiễm chéo)
  - Sau khi import xong: xóa theo NIST 800-88
```

### Bảo Mật Vật Lý

```
GPS tracking:          Theo dõi vị trí 24/7 qua satellite
Cảm biến xâm nhập:    Alarm ngay khi có người cạy mở
Camera giám sát:       Quay liên tục toàn bộ ngoại thất xe
Bảo vệ vũ trang:      Đi kèm xe (tùy region, tùy hợp đồng)
Chain of custody:      Log từng hành động với timestamp
Xe dự phòng:          Không đi một mình — có xe hộ tống
Chỉ di chuyển ban ngày: Giảm rủi ro tai nạn và trộm cắp
```

---

## Use Cases Thực Tế

### 1. Di Chuyển Toàn Bộ Datacenter (Datacenter Exit — Thoát Khỏi Datacenter Vật Lý)

```
Kịch bản: Công ty tài chính quyết định đóng cửa 2 datacenter
Dữ liệu: 8 PB (media files, transaction logs, backup)
Giải pháp:
  - 1 Snowmobile (8 PB < 100 PB tối đa)
  - Copy 8 PB trong ~8 tuần với 10 Gbps
  - Import vào S3 với nhiều bucket theo business unit
  - Tiết kiệm $5M/năm chi phí datacenter
```

### 2. Migration Ngành Truyền Thông (Media & Entertainment)

```
Kịch bản: Hãng phim có 50 PB tape archive (băng lưu trữ)
Vấn đề: Tape đang xuống cấp, cần số hóa khẩn cấp
Giải pháp:
  - Số hóa tape → ổ đĩa NAS trước
  - Dùng Snowmobile copy 50 PB lên S3 Glacier Deep Archive
  - Chi phí lưu trữ giảm 90% so với tape management
```

### 3. Migration Chính Phủ (Government Data Center Consolidation)

```
Kịch bản: Cơ quan chính phủ hợp nhất 5 datacenter thành 1 AWS Region
Dữ liệu: 200 PB (hồ sơ công dân, ảnh vệ tinh, data khoa học)
Giải pháp:
  - 2 Snowmobile chạy song song (2 × 100 PB)
  - Mỗi xe phụ trách 1 agency
  - Hoàn thành trong 6 tháng
  - Tuân thủ FedRAMP — Federal Risk and Authorization Management Program
```

---

## Tính Chi Phí

```
Snowmobile không có public pricing — tất cả đều theo hợp đồng enterprise.

Các yếu tố ảnh hưởng giá:
  ├── Khối lượng dữ liệu (PB)
  ├── Thời gian thuê xe (ngày)
  ├── Khoảng cách vận chuyển
  ├── Yêu cầu bảo mật đặc biệt
  └── SLA — Service Level Agreement — Thỏa Thuận Mức Dịch Vụ

Ước tính ballpark (thường không public):
  - Rẻ hơn nhiều so với nâng cấp băng thông để transfer qua internet
  - Thường hoàn vốn trong vài tuần so với phương án mạng
```

---

## Hạn Chế Của Snowmobile

| Hạn Chế | Chi Tiết |
|---|---|
| Không tự order được | Cần liên hệ AWS Sales trực tiếp |
| Chỉ available ở một số region | Không phải tất cả AWS Region đều có |
| Yêu cầu hạ tầng đặc biệt | Điện 3-phase, cáp quang, không gian đậu xe |
| Không có compute | Không chạy EC2 như Snowball Edge |
| Một chiều | Chỉ import vào AWS, không export ra |
| Thời gian chuẩn bị dài | Vài tuần để lên lịch và deploy |

---

## Snowmobile vs Snowball Edge — Khi Nào Dùng Cái Nào?

```
Snowball Edge (nhiều thiết bị):
  ✅ < 10 PB
  ✅ Địa điểm không cho xe tải lớn vào
  ✅ Cần edge compute đồng thời
  ✅ Cần linh hoạt về lịch trình
  ✅ Budget cần dự đoán được (public pricing)

Snowmobile:
  ✅ > 10 PB
  ✅ Đang đóng cửa datacenter (DC exit)
  ✅ Có không gian và hạ tầng phù hợp
  ✅ Có team AWS Enterprise support
  ✅ Có thể chờ 2–4 tuần để lên lịch
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao có Snowmobile khi đã có nhiều Snowball Edge?**

> Logistics: thuê 125 Snowball Edge để copy 100 PB phức tạp hơn nhiều so với 1 Snowmobile. Với 1 xe, có 1 manifest, 1 team quản lý, 1 connection point. Đơn giản hóa vận hành ở quy mô exabyte.

**Q: Snowmobile có thể export dữ liệu từ AWS ra không?**

> Không. Snowmobile chỉ hỗ trợ **import** dữ liệu vào AWS. Nếu cần export lượng lớn, cần thảo luận riêng với AWS — thường không khả thi hoặc rất tốn kém.

**Q: Nếu xe Snowmobile bị tai nạn, dữ liệu có mất không?**

> Không mất data vì mã hóa AES-256 không thể đọc nếu không có KMS key (KMS key ở AWS, không trên xe). Nhưng dữ liệu vật lý có thể bị hư hại (ổ đĩa vỡ). AWS có SLA về việc này và thường có backup plan trong hợp đồng enterprise.

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-05-16  
**Liên Kết:** [README](./README.md) | [Snowball Edge](./2-snowball-edge.md) | [Snow vs DataSync](./4-snow-vs-datasync.md)
