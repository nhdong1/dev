# AWS Elastic Disaster Recovery (DRS) — Phục Hồi Thảm Họa Liên Tục

> **AWS Elastic Disaster Recovery** (DRS — Dịch Vụ Phục Hồi Thảm Họa Đàn Hồi) cho phép replication liên tục từ server on-premises hoặc cloud khác sang AWS, sẵn sàng failover (chuyển sang AWS) trong vài phút khi có thảm họa. Khác với MGN dùng để di chuyển vĩnh viễn, DRS là giải pháp **luôn chạy nền** để bảo vệ business continuity (tính liên tục của hoạt động kinh doanh).

## 📚 Mục Lục

1. [DRS Là Gì? Và Tại Sao Cần?](#drs-là-gì)
2. [Kiến Trúc DRS](#kiến-trúc-drs)
3. [RPO Và RTO — Chỉ Số Quan Trọng Nhất](#rpo-và-rto)
4. [Failover — Chuyển Sang AWS Khi Có Thảm Họa](#failover)
5. [Failback — Quay Về On-Premises Sau Thảm Họa](#failback)
6. [Drill — Kiểm Tra DR Định Kỳ](#drill)
7. [DRS Vs. MGN — Khi Nào Dùng Loại Nào](#drs-vs-mgn)
8. [Phí Dịch Vụ](#phí-dịch-vụ)
9. [Cấu Hình Thực Tế](#cấu-hình-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 DRS Là Gì? Và Tại Sao Cần?

### Vấn Đề Mà DRS Giải Quyết

```
Kịch bản thực tế không có DR plan:

22:47 — Lũ lụt, mất điện toàn bộ data center khu vực
22:48 — Tất cả server on-premises DOWN
22:49 — Users không truy cập được ứng dụng
...
Ngày 3 — Thuê hardware mới, restore từ backup cũ nhất: 3 ngày trước
Ngày 5 — Ứng dụng phục hồi, nhưng mất 3 ngày dữ liệu

Hậu quả:
├── Revenue loss: $X triệu (downtime × doanh thu/giờ)
├── Data loss: giao dịch 3 ngày không phục hồi được
├── Reputation damage: khách hàng mất tin tưởng
└── Regulatory penalty: vi phạm SLA, có thể bị phạt tiền
```

**DRS giải quyết bằng cách:**

- Replication liên tục (sub-second RPO — Recovery Point Objective — Mục Tiêu Điểm Phục Hồi dưới 1 giây)
- Failover nhanh (< 15 phút RTO — Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi)
- Không cần pre-provisioned hardware (chỉ trả phí khi failover xảy ra)
- Test DR định kỳ mà không ảnh hưởng production

### DRS Trong Bức Tranh Tổng Thể DR

```
Bốn chiến lược DR (từ đơn giản → phức tạp, từ rẻ → đắt):

Tier 1 — Backup & Restore:
├── RPO: vài giờ → vài ngày (tùy tần suất backup)
├── RTO: vài giờ → vài ngày (phải restore và rebuild)
├── Chi phí: rất thấp (chỉ trả phí lưu trữ backup)
└── Dùng cho: ứng dụng không quan trọng, tolerance downtime cao

Tier 2 — Pilot Light (Đèn Dẫn Đường — Tắt Nhưng Sẵn Sàng):
├── RPO: vài phút → vài giờ (sync định kỳ)
├── RTO: vài giờ (cần scale up từ minimal setup)
├── Chi phí: thấp (chỉ duy trì minimum infra)
└── Dùng cho: ứng dụng quan trọng nhưng có thể chịu vài giờ downtime

Tier 3 — Warm Standby (Dự Phòng Ấm — Chạy Nhưng Thu Nhỏ):
├── RPO: giây → phút
├── RTO: vài phút → vài chục phút (scale up từ reduced capacity)
├── Chi phí: trung bình (duy trì infra thu nhỏ)
└── Dùng cho: ứng dụng critical, chịu được một ít downtime

Tier 4 — Multi-Site Active-Active (Đa Địa Điểm Đang Hoạt Động):
├── RPO: 0 (không mất dữ liệu)
├── RTO: 0 (không có downtime — load balancer tự điều hướng)
├── Chi phí: rất cao (duy trì full capacity ở nhiều site)
└── Dùng cho: banking, healthcare, e-commerce lớn

→ AWS DRS phù hợp nhất với Tier 3 (Warm Standby),
  với RPO < 1 giây và RTO < 15 phút, ở mức chi phí hợp lý.
```

---

## 🏗️ Kiến Trúc DRS

### Sơ Đồ Tổng Quan

```
[SOURCE — Nguồn: On-Premises hoặc Cloud Khác]
    │ Block-level continuous replication
    │ (AWS Replication Agent — cùng technology với MGN)
    │
    ▼
[AWS DRS STAGING AREA — Vùng Dàn Dựng]
    │ Replication Server (EC2 tự động quản lý)
    │ EBS Staging Volumes (lưu data liên tục)
    │ Recovery Points (điểm phục hồi) mỗi vài giây
    │
    ├─── [BÌNH THƯỜNG — Không có thảm họa]
    │    └── Chỉ replication chạy nền, chi phí thấp
    │        (không có EC2 recovery instances đang chạy)
    │
    └─── [KHI CÓ THẢM HỌA — Trigger Failover]
         ▼
    [RECOVERY EC2 INSTANCES — Instances Phục Hồi]
         │ Launch trong VPC theo Recovery Launch Settings
         │ Khởi động từ Recovery Point (chọn thời điểm phục hồi)
         └── Chạy full capacity để thay thế on-premises
```

### Recovery Points — Điểm Phục Hồi

```
DRS lưu Recovery Points liên tục:

Thời gian thực (Continuous Recovery Points):
├── Mỗi vài giây một recovery point
├── Lưu trữ trong 24 giờ gần nhất
└── RPO sub-second: phục hồi đến trạng thái gần nhất có thể

Hourly Snapshots (Snapshot Mỗi Giờ — tùy chọn):
├── Snapshot mỗi giờ trong 7 ngày gần nhất
└── Hữu ích để phục hồi về trạng thái trước khi ransomware tấn công
    (vì continuous replication sẽ sync cả ransomware nếu không kịp dừng)

→ Khi failover, bạn CHỌN recovery point muốn phục hồi về:
  "Phục hồi về lúc 14:30:00 hôm qua" (trước khi xảy ra incident)
```

---

## ⏱️ RPO Và RTO — Chỉ Số Quan Trọng Nhất

### Định Nghĩa

| Chỉ Số | Tiếng Anh | Tiếng Việt | Ý Nghĩa |
| ------ | --------- | ---------- | ------- |
| **RPO** | Recovery Point Objective | Mục Tiêu Điểm Phục Hồi | Mất tối đa bao nhiêu dữ liệu? (tính theo thời gian) |
| **RTO** | Recovery Time Objective | Mục Tiêu Thời Gian Phục Hồi | Hệ thống offline tối đa bao lâu? |

### DRS Đạt Được Mức Nào?

```
AWS DRS:
├── RPO: Sub-second (dưới 1 giây)
│   └── Continuous block-level replication → gần như không mất dữ liệu
│
└── RTO: Thường < 15 phút
    ├── Khởi động EC2 recovery instances: ~3-5 phút
    ├── OS boot: ~2-3 phút
    ├── Application startup: 2-10 phút (tùy ứng dụng)
    └── Update DNS/routing: ~1-2 phút
```

### RPO vs RTO: Trade-offs

```
Ví dụ: Công ty tài chính, mất 1 giờ downtime = $500,000

Tình huống A: RPO = 4 giờ, RTO = 8 giờ (backup truyền thống)
├── Có thể mất 4 giờ giao dịch
├── Mất $4M revenue trong 8 giờ downtime
└── Chi phí DR infrastructure: $1,000/tháng

Tình huống B: RPO < 1 giây, RTO < 15 phút (AWS DRS)
├── Gần như không mất giao dịch
├── Mất tối đa $125K revenue (15 phút × $500K/giờ)
└── Chi phí DRS: ~$5,000-10,000/tháng

→ ROI rõ ràng: chi thêm ~$9,000/tháng để tránh rủi ro mất $4M+
```

---

## 🚨 Failover — Chuyển Sang AWS Khi Có Thảm Họa

### Các Loại Failover

```
1. Planned Failover (Failover Có Kế Hoạch):
├── Ví dụ: bảo trì data center theo lịch, nâng cấp phần cứng
├── Graceful shutdown on-premises → launch AWS instances → verify
└── Sau đó: failback về on-premises khi bảo trì xong

2. Unplanned Failover (Failover Khẩn Cấp):
├── Ví dụ: thiên tai, mất điện, tấn công mạng
├── On-premises đột ngột DOWN → trigger failover ngay lập tức
└── Chọn recovery point phù hợp và launch AWS instances
```

### Quy Trình Failover Trong DRS Console

```
Bước 1: Xác Định Sự Cố (Incident Detection)
├── Monitoring báo alert: on-premises servers not responding
├── Xác nhận: đây là thảm họa thực sự (không phải false alarm)
└── Kích hoạt DR runbook, thông báo stakeholders

Bước 2: Trigger Failover Trong DRS Console
→ AWS Console → Elastic Disaster Recovery → Source Servers
→ Chọn servers cần failover → Actions → Initiate Recovery

Bước 3: Chọn Recovery Point
├── "Use most recent" — dùng recovery point mới nhất (thường là tốt nhất)
├── Hoặc chọn một thời điểm cụ thể (ví dụ: trước khi ransomware tấn công)
└── DRS cảnh báo: dữ liệu sau recovery point sẽ bị mất

Bước 4: Launch Recovery EC2 Instances
├── DRS tạo EC2 instances từ recovery point đã chọn
├── Instances boot trong Recovery VPC theo Recovery Launch Settings
└── Thời gian: 5-10 phút

Bước 5: Verify Và Cập Nhật Routing
├── SSH/RDP vào instances, verify ứng dụng chạy
├── Update DNS, load balancer trỏ về AWS recovery instances
└── Monitor ứng dụng hoạt động bình thường

Bước 6: Thông Báo "Recovery Complete"
└── Hệ thống đã phục hồi — DRS vẫn tiếp tục replication (cho failback sau này)
```

### Recovery Launch Settings — Cài Đặt Khởi Chạy Phục Hồi

Tương tự Launch Template trong MGN, Recovery Launch Settings định nghĩa EC2 instances được tạo khi failover:

```
Cấu hình quan trọng:
├── Instance type: thường khác với on-premises (right-sized cho workload)
├── Target subnet: Recovery VPC subnet (thường khác Staging subnet)
├── Security Groups: phản ánh firewall rules on-premises
├── IAM Instance Profile: quyền cần thiết cho ứng dụng AWS
└── Public IP: chỉ gán nếu cần thiết (prefer private + load balancer)
```

---

## 🔄 Failback — Quay Về On-Premises Sau Thảm Họa

### Tại Sao Cần Failback?

```
Sau khi failover sang AWS, có thể muốn quay về on-premises vì:
├── Data center on-premises đã phục hồi sau thảm họa
├── Chi phí EC2 dài hạn cao hơn on-premises (đặc biệt workload ổn định)
├── Compliance/regulatory: dữ liệu phải ở on-premises
└── Hoặc quyết định: tiếp tục chạy trên AWS vĩnh viễn (không failback)
```

### Quy Trình Failback

```
DRS hỗ trợ failback về on-premises (hoặc về cloud khác):

Bước 1: Chuẩn Bị Môi Trường On-Premises
├── Đảm bảo server on-premises đã phục hồi (power, network...)
└── Cài AWS Replication Agent trên server on-premises target

Bước 2: Replication Ngược (AWS → On-Premises)
├── DRS replicate data từ AWS recovery instances về on-premises
└── Sync delta: chỉ sync những thay đổi xảy ra trong thời gian failover

Bước 3: Cutover Về On-Premises
├── Graceful shutdown AWS recovery instances
├── Verify on-premises instances đã có dữ liệu mới nhất
├── Update DNS/routing về on-premises
└── Verify ứng dụng hoạt động bình thường

Bước 4: Terminate Recovery EC2 Instances
├── Sau khi verify ổn định, xóa AWS recovery instances
└── DRS tiếp tục chế độ bảo vệ bình thường (on-premises → AWS replication)
```

> **Lưu ý thực tế:** Failback phức tạp hơn failover. Nhiều tổ chức thực hiện failover rồi quyết định **ở lại trên AWS** thay vì failback, kết hợp với việc hiện đại hóa ứng dụng trong khi đang chạy trên AWS.

---

## 🔍 Drill — Kiểm Tra DR Định Kỳ

### Drill Là Gì?

**DR Drill** (Kiểm Tra Phòng Ngừa Thảm Họa) — còn gọi là **failover test** — là việc thực hiện failover giả lập định kỳ để xác minh:

1. DR plan thực sự hoạt động (không chỉ tồn tại trên giấy)
2. RTO và RPO đạt được như cam kết
3. Team biết cách thực hiện failover khi áp lực thực tế

```
DRS hỗ trợ Drill mà không ảnh hưởng production:

Trong Drill:
├── Recovery instances được launch trong isolated test VPC
│   (không có kết nối ra ngoài — không ảnh hưởng production)
├── Replication tiếp tục bình thường trong suốt drill
└── Sau drill: terminate test instances, không có tác động gì

Khác với failover thực tế:
└── Failover thực: terminate on-premises, launch trong production VPC
```

### Tần Suất Drill Được Khuyến Nghị

| Loại Ứng Dụng | Tần Suất Drill | Ghi Chú |
| ------------- | -------------- | ------- |
| Mission-Critical (Cực Kỳ Quan Trọng) | Hàng tháng | Banking, healthcare, e-commerce lớn |
| Business-Critical (Rất Quan Trọng) | Hàng quý (3 tháng/lần) | ERP, CRM, internal systems |
| Standard (Tiêu Chuẩn) | Nửa năm/lần | Ứng dụng nội bộ ít quan trọng |
| Compliance-Required (Yêu Cầu Tuân Thủ) | Theo quy định | Ít nhất 1 lần/năm theo nhiều standards |

> **Thực tế đáng buồn:** Nhiều tổ chức có DR plan nhưng **chưa bao giờ test**. Khi thảm họa xảy ra mới phát hiện DR plan không hoạt động. DRS giúp test dễ dàng và không rủi ro — không có lý do để không drill định kỳ.

---

## ⚖️ DRS Vs. MGN — Khi Nào Dùng Loại Nào

### Bảng So Sánh Chi Tiết

| Tiêu Chí | AWS MGN | AWS DRS |
| -------- | ------- | ------- |
| **Mục đích** | Di chuyển server vĩnh viễn lên AWS | Bảo vệ liên tục, failover khi có thảm họa |
| **Thời gian hoạt động** | Project (có điểm kết thúc) | Ongoing — liên tục (không có điểm kết thúc) |
| **Hướng replication** | Source → AWS (một chiều) | Source → AWS, hỗ trợ failback AWS → Source |
| **Test mode** | Test cutover (server gốc vẫn chạy) | DR Drill (isolated test VPC) |
| **Failback** | Không có | ✅ Hỗ trợ failback |
| **Recovery Points** | Current state (điểm hiện tại) | Multi-point (nhiều điểm trong 24 giờ + hourly snapshots 7 ngày) |
| **Use case điển hình** | "Tôi muốn chuyển server này lên AWS" | "Tôi muốn bảo vệ server này, sẵn sàng failover bất kỳ lúc nào" |
| **Phí replication** | $0.042/server/giờ | $0.028/server/giờ |
| **Chi phí ongoing** | Tạm thời (giảm khi hoàn thành) | Liên tục (chi phí replication mãi mãi) |

### Có Thể Dùng Cả Hai Không?

```
Kịch bản kết hợp MGN + DRS:

Giai đoạn 1 — Migration (dùng MGN):
├── Rehost server lên EC2 dùng MGN
└── Sau cutover: EC2 là "server gốc" mới

Giai đoạn 2 — Bảo vệ sau migration (dùng DRS):
├── Cài DRS trên EC2 vừa migrate (bây giờ EC2 là "source")
├── DRS replicate EC2 sang region AWS thứ hai (cross-region DR)
└── Đây là pattern phổ biến: cross-region disaster recovery sau migration

→ Ví dụ: Server on-premises → (MGN) → EC2 ap-southeast-1
                                → (DRS) → EC2 us-east-1 (DR region)
```

---

## 💰 Phí Dịch Vụ

### Cơ Cấu Phí DRS

| Thành Phần | Phí | Ghi Chú |
| ---------- | --- | ------- |
| **DRS Replication** | $0.028/server/giờ | Tính từ khi cài agent, liên tục |
| **Replication Server** (EC2) | ~$0.02/giờ (t3.small) | AWS tự quản lý |
| **EBS Staging Volumes** | ~$0.08/GB/tháng | Lưu data replicated |
| **Recovery EC2 Instances** | Theo EC2 pricing | CHỈ tính khi đang failover hoặc drill |
| **Data Transfer** | $0.09/GB (internet) | Replication traffic |

### Ví Dụ Chi Phí Thực Tế

```
Kịch bản: Bảo vệ 20 server (500 GB mỗi server) liên tục

DRS Replication:
└── $0.028 × 20 server × 720 giờ/tháng = $403.20/tháng

Replication Servers (20 t3.small):
└── $0.0208 × 20 × 720 = $299.52/tháng

EBS Staging Volumes (20 × 500 GB):
└── $0.08 × 10,000 GB = $800/tháng

Data Transfer (initial + ongoing ~500 GB/tháng delta):
└── $0.09 × 500 GB = $45/tháng

Tổng ongoing: ~$1,548/tháng cho 20 server
→ ~$77/server/tháng để có RPO sub-second và RTO < 15 phút

Chi phí khi failover (chỉ tính khi xảy ra):
└── EC2 recovery instances: tính theo giờ chạy
    ví dụ: 20 m5.xlarge × $0.192/giờ × 24 giờ = $92.16/ngày
```

---

## 🚀 Cấu Hình Thực Tế

### Thiết Lập DRS Từ Đầu

```
Bước 1: Khởi Tạo DRS Service
→ AWS Console → Elastic Disaster Recovery → Get started
→ Chọn AWS Region làm DR target
→ AWS tạo IAM roles cần thiết tự động

Bước 2: Cấu Hình Staging Area
→ Replication Settings:
   ├── Replication Server instance type: t3.small (default)
   ├── Staging Area Subnet: private subnet (KHÔNG public)
   ├── Use AWS-managed CMK hoặc customer-managed KMS key
   └── Bandwidth throttling: giới hạn bandwidth nếu cần (GB/giây)

Bước 3: Cài Replication Agent
# Giống MGN — cùng installer:
sudo python3 aws-replication-installer-init.py \
  --region us-east-1 \
  --aws-access-key-id YOUR_KEY \
  --aws-secret-access-key YOUR_SECRET \
  --no-prompt

# Server xuất hiện trong DRS Console → Source Servers

Bước 4: Cấu Hình Recovery Launch Settings
→ DRS Console → Source Servers → Chọn server → Launch settings
→ Cấu hình instance type, subnet, security groups cho recovery instances
→ Thường chọn Recovery VPC khác với Staging VPC

Bước 5: Verify Replication
→ Theo dõi trong DRS Console:
   └── Data Lag: thời gian chênh lệch giữa source và staging
       (nên < 30 giây trong điều kiện bình thường)
```

### Bandwidth Throttling — Kiểm Soát Băng Thông

```
DRS cho phép giới hạn bandwidth sử dụng:

Tại sao cần throttling:
├── Tránh replication chiếm hết băng thông production network
└── Ứng dụng production cần băng thông ưu tiên hơn replication

Cấu hình:
├── Theo giờ: peak hours (8:00-18:00) → 50 Mbps, off-peak → 200 Mbps
└── DRS Console → Replication Settings → Bandwidth throttling

Lưu ý: throttling có thể làm tăng Data Lag —
nếu lag quá cao, RPO bị ảnh hưởng
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: AWS DRS khác gì với AWS MGN? Khi nào dùng từng loại?**

> MGN (Application Migration Service) là công cụ di chuyển vĩnh viễn — project có điểm bắt đầu và kết thúc rõ ràng, không hỗ trợ failback. DRS (Elastic Disaster Recovery) là giải pháp bảo vệ liên tục — luôn chạy nền, hỗ trợ nhiều recovery points trong lịch sử, có thể failback về on-premises. Dùng MGN khi muốn chuyển hẳn server lên AWS; dùng DRS khi cần bảo vệ server on-premises khỏi thảm họa trong khi vẫn giữ on-premises là primary.

**Q: RPO và RTO là gì? AWS DRS đạt được mức nào?**

> RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) là khoảng thời gian dữ liệu có thể bị mất — ví dụ RPO 1 giờ nghĩa là mất tối đa 1 giờ dữ liệu. RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) là thời gian hệ thống offline tối đa cho phép. AWS DRS đạt RPO sub-second (dưới 1 giây) nhờ continuous block-level replication, và RTO dưới 15 phút nhờ pre-staged data và automated EC2 launch.

**Q: DR Drill trong DRS hoạt động như thế nào? Có ảnh hưởng production không?**

> Khi trigger DR Drill trong DRS, recovery instances được launch trong một isolated test VPC — không có routing ra internet và không kết nối với production. Replication tiếp tục bình thường trong suốt drill. Sau khi drill xong, terminate test instances, không có tác động gì đến production. Đây là lý do DRS cho phép drill thường xuyên (monthly cho mission-critical) mà không cần maintenance window hay rủi ro gián đoạn.

**Q: Tại sao DRS lưu nhiều recovery points? Khi nào cần chọn recovery point cũ hơn?**

> Continuous recovery points (mỗi vài giây trong 24 giờ) cho phép phục hồi đến trạng thái gần nhất khi có hardware failure hoặc natural disaster. Nhưng khi có ransomware attack, continuous replication sẽ sync cả mã hóa ransomware — lúc này hourly snapshots (7 ngày) cho phép "point-in-time recovery" về thời điểm trước khi bị tấn công, giống như database point-in-time restore. Đây là lý do DRS lưu cả continuous và hourly snapshots.

**Q: Có thể dùng DRS để cross-region disaster recovery trong AWS không?**

> Hoàn toàn có thể. Sau khi migrate server lên EC2 (dùng MGN hoặc cách khác), có thể cài DRS agent trên EC2 đó để replicate sang AWS Region thứ hai. Ví dụ: Primary workload ở ap-southeast-1 (Singapore), DRS replicate liên tục sang us-east-1 (N. Virginia) hoặc ap-northeast-1 (Tokyo). Đây là pattern phổ biến sau khi đã hoàn thành migration — layer DR protection lên trên môi trường AWS.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
