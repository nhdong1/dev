# AWS Application Migration Service (MGN) & DRS — Tổng Quan

> Sau khi đã lập kế hoạch và khám phá môi trường, bước tiếp theo là **thực sự di chuyển server**. AWS MGN — Application Migration Service — là dịch vụ chính để rehost server lên EC2 với downtime tối thiểu, trong khi AWS DRS — Elastic Disaster Recovery — phục vụ cho việc bảo vệ liên tục chống thảm họa.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao Dùng MGN Thay Vì Chỉ Cài Lại?](#tại-sao-dùng-mgn)
2. [Hai Dịch Vụ Trong Nhóm Này](#hai-dịch-vụ-trong-nhóm-này)
3. [Luồng Migration Điển Hình Với MGN](#luồng-migration-điển-hình)
4. [So Sánh MGN và DRS](#so-sánh-mgn-và-drs)
5. [Điều Hướng Tài Liệu](#điều-hướng-tài-liệu)

---

## 🎯 Tại Sao Dùng MGN?

Cài lại ứng dụng từ đầu trên EC2 (gọi là **lift-and-shift thủ công**) gặp nhiều vấn đề:

```
Vấn đề khi cài lại thủ công:
├── Cần tái tạo toàn bộ cấu hình OS, phần mềm, dependencies
├── Không biết chính xác server đang có gì (undocumented configs)
├── Downtime dài: tắt server cũ → cài lại → kiểm tra → bật server mới
├── Rủi ro: bỏ sót cấu hình quan trọng → ứng dụng lỗi sau cutover
└── Khó rollback nếu có sự cố
```

**AWS MGN — Application Migration Service giải quyết bằng cách:**

- **Replication liên tục (Continuous Replication):** Sao chép block-level từ server nguồn lên AWS staging area theo thời gian thực
- **Test trước, cutover sau:** Test ứng dụng trên AWS mà server gốc vẫn chạy bình thường
- **Downtime tối thiểu:** Cắt chuyển (cutover) chỉ mất vài phút
- **Tự động hóa:** Launch templates định nghĩa sẵn cấu hình EC2 target

---

## 🛠️ Hai Dịch Vụ Trong Nhóm Này

### 1. AWS MGN — Application Migration Service

**Vai trò:** Rehost — nâng và chuyển (lift-and-shift) máy chủ vật lý, máy ảo và cloud lên EC2 Amazon.

```
MGN hoạt động theo nguyên lý:
├── Cài AWS Replication Agent trên server nguồn
├── Agent sao chép block-by-block liên tục lên Staging Area (S3/EBS staging)
├── Khi sẵn sàng: launch EC2 instance từ dữ liệu đã replicate
│   ├── Test cutover (thử nghiệm, server gốc vẫn chạy)
│   └── Production cutover (cắt chuyển chính thức)
└── Hoàn tất: tắt server gốc, dọn dẹp staging
```

**Điểm mạnh:**
- Hỗ trợ rộng: Windows, Linux, VMware, Hyper-V, physical servers, cloud khác
- Replication block-level → giữ nguyên 100% OS + data + configuration
- Không cần refactor code — phù hợp chiến lược Rehost (R2)

Tài liệu chi tiết: [1-mgn-overview.md](./1-mgn-overview.md) | [2-mgn-cutover.md](./2-mgn-cutover.md)

---

### 2. AWS Elastic Disaster Recovery (DRS)

**Vai trò:** Phục hồi thảm họa liên tục — Disaster Recovery — cho server on-premises và cloud khác sang AWS.

```
DRS khác MGN ở mục đích sử dụng:
├── MGN  → Di chuyển vĩnh viễn (migration project, có điểm kết thúc)
└── DRS  → Bảo vệ liên tục (luôn chạy nền, sẵn sàng failover bất kỳ lúc nào)

Kịch bản DRS điển hình:
├── Server on-premises đang chạy bình thường
├── DRS liên tục replication lên AWS (RPO — Recovery Point Objective — vài giây)
├── Khi có sự cố (natural disaster, hardware failure):
│   ├── Trigger failover → EC2 instances khởi động trong phút
│   └── RTO — Recovery Time Objective — < 15 phút
└── Khi ổn định: failback về on-premises (hoặc tiếp tục chạy trên AWS)
```

Tài liệu chi tiết: [3-elastic-disaster-recovery.md](./3-elastic-disaster-recovery.md)

---

## 🔄 Luồng Migration Điển Hình Với MGN

```
Giai đoạn MIGRATE — Thứ tự thực hiện với MGN:

Bước 1: Chuẩn bị (1-2 ngày)
├── Tạo tài khoản dịch vụ MGN trong AWS Console
├── Tải và cài AWS Replication Agent trên server nguồn
└── Cấu hình Launch Template: instance type, subnet, security group...

Bước 2: Replication khởi tạo — Initial Sync (vài giờ đến vài ngày)
├── Agent sao chép toàn bộ dữ liệu lên Staging Area (EBS volumes)
├── Sau initial sync: replication chuyển sang chế độ continuous (delta changes only)
└── Theo dõi tiến trình trong MGN Console hoặc Migration Hub

Bước 3: Test Cutover — Thử Nghiệm Cắt Chuyển (nhiều lần)
├── Launch test EC2 instance từ dữ liệu đã replicate
├── Kiểm tra ứng dụng, connectivity, performance
├── Server gốc VẪN ĐANG CHẠY — không ảnh hưởng production
└── Xóa test instance → tiếp tục replication

Bước 4: Production Cutover — Cắt Chuyển Chính Thức (downtime ~phút)
├── Chọn thời điểm maintenance window
├── Finalize replication (đồng bộ thay đổi cuối cùng)
├── Launch production EC2 instance
├── Cập nhật DNS/load balancer trỏ sang EC2 mới
└── Xác nhận ứng dụng hoạt động → tắt server gốc

Bước 5: Dọn dẹp (Cleanup)
├── Xóa Staging Area (tiết kiệm chi phí)
├── Gỡ cài đặt Replication Agent trên server gốc
└── Archive hoặc decommission (loại bỏ) server vật lý
```

---

## 📊 So Sánh MGN và DRS

| Tiêu Chí | AWS MGN | AWS DRS |
| -------- | ------- | ------- |
| **Mục đích chính** | Di chuyển vĩnh viễn lên AWS | Phục hồi thảm họa (luôn chạy nền) |
| **Thời gian sử dụng** | Project có điểm bắt đầu/kết thúc | Liên tục (ongoing) |
| **Phí replication** | $0.042/server/giờ (staging) | $0.028/server/giờ (continuous) |
| **Test cutover** | ✅ Hỗ trợ đầy đủ | ✅ Hỗ trợ drill (test failover) |
| **Failback** | Không có (migration một chiều) | ✅ Failback về on-premises |
| **RPO — Recovery Point Objective** | N/A (migration, không phải DR) | Dưới 1 giây |
| **RTO — Recovery Time Objective** | Vài phút (cutover) | < 15 phút (failover) |
| **Khi nào dùng** | Khi muốn chuyển hẳn lên AWS | Khi cần DR song song với on-premises |

> **Quy tắc nhớ nhanh:** MGN = chuyển nhà vĩnh viễn. DRS = mua nhà thứ hai để phòng thiên tai.

---

## 🔗 Điều Hướng Tài Liệu

| File | Nội Dung |
| ---- | -------- |
| [1-mgn-overview.md](./1-mgn-overview.md) | MGN kiến trúc, Replication Agent, Launch Templates, supported platforms |
| [2-mgn-cutover.md](./2-mgn-cutover.md) | Test cutover, production cutover window, rollback strategy, post-cutover validation |
| [3-elastic-disaster-recovery.md](./3-elastic-disaster-recovery.md) | DRS: continuous replication, failover, failback, RPO/RTO, so sánh với MGN |

---

## 🎓 Câu Hỏi Phỏng Vấn Điển Hình (Module Này)

1. AWS MGN là gì? Nó thay thế dịch vụ nào trước đây?
2. Sự khác biệt giữa MGN và AWS DRS là gì? Khi nào dùng từng loại?
3. Giải thích quá trình test cutover và tại sao nó quan trọng.
4. RPO và RTO là gì? DRS đạt được mức nào?
5. Nếu cutover thất bại, bạn rollback như thế nào với MGN?

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
