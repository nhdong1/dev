# RPO & RTO — Mục Tiêu Phục Hồi Sao Lưu

Hiểu rõ RPO và RTO là nền tảng để thiết kế bất kỳ chiến lược sao lưu và khắc phục thảm họa nào.

## RPO (Recovery Point Objective) — Mục Tiêu Điểm Phục Hồi

**"Mức mất dữ liệu tối đa chấp nhận được"**

Đo lường bằng **THỜI GIAN**: phút/giờ/ngày

### Ví dụ: RPO = 1 giờ

```
Có thể mất tối đa 1 giờ transaction
Cần sao lưu ít nhất mỗi giờ
Hoặc replication liên tục
```

### Ý Nghĩa Thực Tế

- Nếu thảm họa xảy ra lúc 3:45 CH, và backup cuối lúc 3:00 CH
- Mất 45 phút dữ liệu (chấp nhận nếu RPO = 1 giờ)
- Mất 45 phút dữ liệu (KHÔNG chấp nhận nếu RPO = 10 phút)

### Cách Đạt Được RPO

| Mục tiêu RPO  | Phương pháp                           | Chi phí        |
| ------------- | ------------------------------------- | -------------- |
| 24 giờ        | Sao lưu hàng ngày                     | Thấp           |
| 1-4 giờ       | Nhiều lần sao lưu mỗi ngày            | Trung bình     |
| 15-30 phút    | Sao lưu hàng giờ + transaction logs  | Trung bình     |
| < 15 phút     | Replication liên tục (WAL shipping)   | Cao            |
| Gần bằng không | Replication đồng bộ                  | Rất cao        |

---

## RTO (Recovery Time Objective) — Mục Tiêu Thời Gian Phục Hồi

**"Thời gian dừng hoạt động tối đa chấp nhận được"**

Đo lường bằng **THỜI GIAN** để khôi phục dịch vụ

### Ví dụ: RTO = 30 phút

```
CSDL phải online trong vòng 30 phút sau sự cố
Cần failover tự động HOẶC quy trình khôi phục nhanh
Yêu cầu backup có thể truy cập và đã được kiểm tra
```

### Ý Nghĩa Thực Tế

- Thảm họa xảy ra lúc 3:00 CH
- Tác động nghiệp vụ bắt đầu ngay (mất doanh thu, ảnh hưởng khách hàng, v.v.)
- Đến 3:30 CH, CSDL PHẢI online
- Mọi thứ từ thông báo đến khôi phục phải ≤ 30 phút

### Cách Đạt Được RTO

| Mục tiêu RTO  | Phương pháp                         | Chi phí        | Độ phức tạp   |
| ------------- | ----------------------------------- | -------------- | ------------- |
| 4-8 giờ       | Khôi phục thủ công từ backup        | Thấp           | Dễ            |
| 1-2 giờ       | Khôi phục tự động + warm standby   | Trung bình     | Vừa           |
| 15-30 phút    | HA failover đến replica             | Cao            | Phức tạp      |
| < 5 phút      | Active-active + load balancer       | Rất cao        | Rất phức tạp  |
| < 1 phút      | Sync replication + auto-DNS         | Cực cao        | Rất phức tạp  |

---

## RPO vs RTO — Đánh Đổi

```
RPO ← MẤT DỮ LIỆU     so với     THỜI GIAN CHẾT → RTO
│
↑ RPO chặt chẽ hơn = Tốn kém hơn
  (sao lưu/replication liên tục)

                    ↑ RTO chặt chẽ hơn = Tốn kém hơn
                      (HA/tự động hóa failover)

Nghiệp vụ quyết định đánh đổi chấp nhận được
```

---

## Ví Dụ Tác Động Nghiệp Vụ

### Ví dụ 1: Nền Tảng Thương Mại Điện Tử

```
Yêu cầu: Uptime 99.99% (52 phút/năm được phép ngừng)

Tác động 1 giờ ngừng hoạt động:
- Mất doanh thu 50.000 đô/giờ
- Thiệt hại uy tín
- Khiếu nại từ khách hàng

Thiết kế:
- RPO: 5 phút (replication liên tục)
- RTO: 10 phút (failover tự động sang hot standby)
- Chi phí: 500k đô/năm cho cơ sở hạ tầng
```

### Ví dụ 2: CSDL Báo Cáo Nội Bộ

```
Yêu cầu: Uptime 95% (21.6 giờ/năm được phép ngừng)

Tác động 4 giờ ngừng hoạt động:
- Báo cáo bị trễ, không ảnh hưởng doanh thu
- Các nhóm có thể làm việc offline
- Chấp nhận được

Thiết kế:
- RPO: 24 giờ (chỉ backup hàng ngày)
- RTO: 4 giờ (khôi phục thủ công, trong giờ hành chính)
- Chi phí: 20k đô/năm
```

### Ví dụ 3: CSDL Tuân Thủ PCI-DSS

```
Yêu cầu: Tuân thủ PCI-DSS, phải phục hồi trong 1 giờ

Tác động 1 giờ ngừng hoạt động:
- Transaction được xếp hàng, có thể thử lại
- Yêu cầu tuân thủ: Uptime 99.99%

Thiết kế:
- RPO: 15 phút (transaction log backups)
- RTO: 45 phút (khôi phục tự động với kiểm tra)
- Chi phí: 300k đô/năm
```

---

## Tính Toán Chi Phí

### Tính Toán Chi Phí RPO

```
RPO chặt chẽ hơn = Sao lưu thường xuyên hơn = Chi phí nhiều hơn:
- Lưu trữ backup
- Băng thông mạng
- License phần mềm backup
- Chi phí overhead replication
```

### Tính Toán Chi Phí RTO

```
RTO chặt chẽ hơn = Cơ sở hạ tầng tinh vi hơn = Chi phí nhiều hơn:
- Hệ thống dự phòng
- Thiết lập replication
- Công cụ failover tự động
- Giám sát & cảnh báo
```

---

## Checklist: Đồng Bộ RPO/RTO với Nghiệp Vụ

- [ ] Họp với stakeholders nghiệp vụ
- [ ] Tài liệu hóa mức mất dữ liệu tối đa chấp nhận (RPO)
- [ ] Tài liệu hóa thời gian ngừng tối đa chấp nhận (RTO)
- [ ] Tính toán tác động nghiệp vụ mỗi giờ ngừng
- [ ] Tính toán chi phí đạt được RPO/RTO
- [ ] Xác minh chi phí chấp nhận được (thường < chi phí tác động nghiệp vụ)
- [ ] Tài liệu hóa trong kế hoạch phục hồi thảm họa
- [ ] Kiểm tra quy trình phục hồi hàng quý
- [ ] Cập nhật khi yêu cầu nghiệp vụ thay đổi

---

## Điểm Mấu Chốt

> **RPO và RTO là quyết định nghiệp vụ, không phải kỹ thuật. Công nghệ triển khai quyết định, nhưng quyết định đến từ việc hiểu các đánh đổi chấp nhận được.**
