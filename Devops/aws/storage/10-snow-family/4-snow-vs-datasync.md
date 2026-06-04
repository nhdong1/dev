# Snow Family vs DataSync — Decision Tree Chọn Giải Pháp

> Đây là một trong những câu hỏi phỏng vấn phổ biến nhất về migration: **"Khi nào dùng Snow Family, khi nào dùng DataSync?"** Câu trả lời không phải là một hay kia mà thường là **kết hợp cả hai** — Snow để move bulk data, DataSync để sync incremental sau đó.

---

## Tổng Quan Hai Giải Pháp

### AWS DataSync

```
DataSync là dịch vụ truyền dữ liệu qua MẠNG:
  - Nguồn: NFS, SMB, S3, EFS, FSx, HDFS, object storage khác
  - Đích: S3, EFS, FSx for Windows, FSx for Lustre, FSx NetApp ONTAP
  - Protocol: Qua internet hoặc Direct Connect hoặc VPN
  - Agent: Phần mềm cài trên máy chủ on-premises

Điểm mạnh:
  ✅ Tự động hóa hoàn toàn
  ✅ Incremental sync (chỉ copy file thay đổi)
  ✅ Preserve metadata (giữ nguyên quyền file, timestamps)
  ✅ Không cần thiết bị vật lý
  ✅ Scheduling (lên lịch chạy định kỳ)
  ✅ Bandwidth throttling (giới hạn băng thông)
```

### Snow Family

```
Snow Family là thiết bị VẬT LÝ để chuyển dữ liệu:
  - Snowcone: 8–14 TB
  - Snowball Edge: 28–80 TB
  - Snowmobile: 100 PB

Điểm mạnh:
  ✅ Không phụ thuộc băng thông mạng
  ✅ Phù hợp dữ liệu cực lớn
  ✅ Edge computing (chạy code tại biên)
  ✅ Hoạt động offline hoàn toàn
  ✅ Bảo mật vật lý cao
```

---

## Decision Tree — Cây Quyết Định

```
                    BẮT ĐẦU
                       │
         Bao nhiêu dữ liệu cần migrate?
                       │
           ┌───────────┴───────────┐
         < 1 TB                  > 1 TB
           │                       │
     Dùng DataSync          Kết nối mạng ra sao?
     (đủ nhanh)                    │
                      ┌────────────┴────────────┐
                   Tốt                       Yếu/không có
                (>10 Mbps)                       │
                   │                      Mất bao lâu qua mạng?
         Deadline migration?                     │
                   │               ┌─────────────┴────────────┐
            ┌──────┴──────┐      < 1 tuần                > 1 tuần
          Gấp           Linh hoạt    │                       │
            │               │     DataSync              Snow Family
      Snow Family        DataSync   (OK)               (cần thiết)
      (nhanh hơn)       (đủ)
```

### Phiên Bản Chi Tiết Hơn

```
1. Băng thông thực tế < 100 Mbps? → Hỏi tiếp câu 2
2. Dữ liệu > 10 TB?               → Snow Family
3. Có kết nối mạng không ổn định?  → Snow Family  
4. Cần edge computing?             → Snowball Edge/Snowcone
5. Migration một lần?              → Snow Family
6. Sync liên tục sau migration?    → DataSync
7. Cần preserve file permissions?  → DataSync (tốt hơn)
8. Còn lại                        → DataSync
```

---

## So Sánh Chi Tiết

| Tiêu Chí | DataSync | Snow Family |
|---|---|---|
| **Cơ chế** | Qua mạng internet/DC | Thiết bị vật lý |
| **Tốc độ** | Phụ thuộc băng thông | Tốc độ cố định, không phụ thuộc mạng |
| **Dữ liệu phù hợp** | < 10 TB (thực tế), có thể hơn | Bất kỳ, tốt nhất > 10 TB |
| **Thiết lập** | Cài agent, cấu hình task | Order thiết bị, chờ ship |
| **Thời gian ready** | Vài giờ | 7–14 ngày (ship) |
| **Incremental sync** | ✅ Có (tự động) | ❌ Không (mỗi lần gửi là batch) |
| **Preserve metadata** | ✅ Tốt (NFS/SMB permissions) | ⚠️ Hạn chế hơn |
| **Edge computing** | ❌ Không | ✅ Có (Snowball/Snowcone) |
| **Offline hoàn toàn** | ❌ Cần mạng | ✅ Hoàn toàn offline |
| **Chi phí** | Theo GB transfer | Theo lần dùng + GB |
| **Tự động hóa** | ✅ Scheduling, triggers | ❌ Manual (từng batch) |
| **Encryption** | TLS in-transit + SSE at-rest | AES-256 hardware |

---

## Tính Toán Thực Tế: Nên Chọn Giải Pháp Nào?

### Công Thức Tính Thời Gian Qua Mạng

```
Thời gian (giờ) = Kích thước dữ liệu (GB) × 8
                  ─────────────────────────────
                  Băng thông thực tế (Gbps) × 3.600

Hiệu suất thực tế thường = 80% băng thông lý thuyết

Ví dụ:
  Dữ liệu: 50 TB = 51.200 GB
  Băng thông: 1 Gbps → thực tế 0.8 Gbps
  
  Thời gian = 51.200 × 8 / (0.8 × 3.600)
            = 409.600 / 2.880
            = 142 giờ ≈ 6 ngày

  → 6 ngày < 2 tuần (threshold tạm chấp nhận được)
  → DataSync khả thi nếu không cần parallel business operations
```

### Rule of Thumb (Quy Tắc Ngón Tay Cái)

```
Băng thông 1 Gbps:
  ≤ 1 TB   → DataSync (< 3 giờ)
  1–10 TB  → DataSync (vài giờ đến 1 ngày)
  10–80 TB → Gray zone — tính toán cụ thể
  > 80 TB  → Snowball Edge
  > 10 PB  → Snowmobile

Băng thông 100 Mbps:
  ≤ 100 GB → DataSync (< 3 giờ)
  100 GB–1 TB → DataSync (chấp nhận được)
  > 1 TB   → Cân nhắc Snow Family nghiêm túc
```

---

## Pattern Kết Hợp — Phương Pháp Hybrid Migration

Trong thực tế production, hai giải pháp thường dùng **cùng nhau**:

### Pattern 1: Bulk + Sync (Di Chuyển Hàng Loạt + Đồng Bộ)

```
Giai đoạn 1 — Bulk Migration (Di Chuyển Hàng Loạt):
  - Dùng Snow Family để copy 95% dữ liệu (historical data)
  - Thời gian: 2–4 tuần

Giai đoạn 2 — Cutover Preparation (Chuẩn Bị Chuyển Đổi):
  - Cài DataSync agent trên NAS/server on-premises
  - Sync incremental data (5% còn lại, dữ liệu mới)
  - Thời gian: song song với giai đoạn 1

Giai đoạn 3 — Cutover (Chuyển Đổi):
  - Tắt write access trên on-premises
  - Chạy DataSync lần cuối (final sync)
  - Chuyển ứng dụng trỏ vào S3/EFS
  - Downtime: vài phút đến vài giờ

Giai đoạn 4 — Ongoing Sync (Đồng Bộ Liên Tục):
  - DataSync tiếp tục sync nếu có on-premises còn lại
  - Dần dần tắt hẳn on-premises
```

### Pattern 2: Edge Collect + Cloud Process (Thu Thập Biên + Xử Lý Đám Mây)

```
Dùng cho IoT, thiết bị thực địa:

Snowcone/Snowball Edge tại biên:
  - Thu thập raw data từ sensors, camera
  - Xử lý sơ bộ (filter noise, aggregate)
  - Lưu kết quả đã xử lý

Ship thiết bị về AWS (hàng tuần/tháng):
  - AWS import vào S3

DataSync cho real-time alerts:
  - Chỉ gửi anomaly/critical events qua mạng
  - Bandwidth nhỏ, dữ liệu có chọn lọc
```

---

## Use Cases Theo Từng Giải Pháp

### Nên Dùng DataSync Khi

```
✅ Di chuyển NFS/SMB shares cần giữ permissions
✅ Sync liên tục giữa on-premises và AWS
✅ Đồng bộ giữa S3 buckets khác region
✅ Backup có lịch trình tự động
✅ Mạng đủ tốt (>100 Mbps) và deadline không gấp
✅ Dữ liệu thường xuyên thay đổi (active dataset)
✅ Cần audit log chi tiết từng file
```

### Nên Dùng Snow Family Khi

```
✅ Không có internet hoặc mạng rất chậm (<10 Mbps)
✅ Dữ liệu lớn và mạng không thực tế (rule: 1 tuần+)
✅ Cần edge computing đồng thời
✅ Migration one-time (lịch sử, không thay đổi)
✅ Môi trường không có kết nối (tàu, sa mạc, quân đội)
✅ Compliance yêu cầu dữ liệu không đi qua internet
✅ Dữ liệu quá nhạy cảm để gửi qua WAN
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Công ty có 200TB dữ liệu, đường truyền 1Gbps. Dùng gì?**

> Tính toán: 200TB ÷ (0.8 × 1Gbps) = ~2.000 giây × 8 ÷ 3.600 = ~4,5 giờ mỗi TB → tổng **khoảng 37 ngày**. Đây là ngưỡng gray zone. Đề xuất: dùng **Snowball Edge** (3 thiết bị × 80TB) để move 95% data trong 2-3 tuần, sau đó DataSync để sync incremental. Cutover trong 1 cuối tuần.

**Q: DataSync có thể thay thế hoàn toàn Snow Family không?**

> Không. DataSync không hoạt động ở nơi không có mạng, không hỗ trợ edge computing, và không thực tế cho dữ liệu cực lớn (>100TB) trên đường truyền chậm. Snow Family giải quyết các vấn đề vật lý mà software không thể.

**Q: Sau khi dùng Snowball migrate xong, có cần DataSync không?**

> Thường có, vì: (1) dữ liệu tiếp tục được tạo ra on-premises trong quá trình ship thiết bị, (2) DataSync sync incremental changes trước cutover, (3) nếu vẫn duy trì hybrid, cần DataSync để đồng bộ liên tục.

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-05-16  
**Liên Kết:** [README](./README.md) | [Snowball Edge](./2-snowball-edge.md) | [OpsHub](./5-opshub-management.md)
