# AWS MGN — Test Cutover, Cutover Window Và Rollback

> Replication chỉ là bước chuẩn bị. Phần **quan trọng nhất** của một dự án migration là **cutover** — thời điểm bạn thực sự chuyển traffic từ server cũ sang EC2 trên AWS. Làm đúng thì downtime chỉ vài phút; làm sai thì rollback khẩn cấp lúc nửa đêm.

## 📚 Mục Lục

1. [Test Cutover — Thử Nghiệm Cắt Chuyển](#test-cutover)
2. [Production Cutover — Cắt Chuyển Chính Thức](#production-cutover)
3. [Cutover Window — Cửa Sổ Thực Hiện Cutover](#cutover-window)
4. [Rollback Strategy — Chiến Lược Quay Lui](#rollback-strategy)
5. [Post-Cutover Validation — Xác Thực Sau Cutover](#post-cutover-validation)
6. [Runbook Mẫu — Template Thực Hành](#runbook-mẫu)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🧪 Test Cutover — Thử Nghiệm Cắt Chuyển

### Test Cutover Là Gì?

**Test Cutover** — Thử Nghiệm Cắt Chuyển — là bước khởi động EC2 instance thử nghiệm từ dữ liệu đã replicate, **trong khi server gốc vẫn đang chạy bình thường**. Đây là cơ hội để kiểm tra ứng dụng trên môi trường AWS mà **không có rủi ro** với production.

```
Test Cutover vs Production Cutover:

TEST CUTOVER:
├── Server gốc: vẫn chạy, tiếp nhận traffic production
├── EC2 test instance: khởi động song song, dùng để kiểm tra
├── Replication: tiếp tục sau khi test instance được tạo
├── Kết quả: xóa test instance, không ảnh hưởng gì
└── Mục đích: xác minh ứng dụng hoạt động trên AWS trước khi commit

PRODUCTION CUTOVER:
├── Server gốc: sẽ dừng nhận traffic (downtime bắt đầu)
├── EC2 production instance: khởi động, nhận traffic
├── Replication: dừng sau cutover
├── Kết quả: không thể undo dễ dàng
└── Mục đích: di chuyển chính thức
```

### Quy Trình Test Cutover

```
Bước 1: Bắt đầu Test Cutover
→ MGN Console → Source Servers → Chọn server → Actions → Launch test instances
→ MGN tạo EC2 instance từ snapshot mới nhất của Staging Area
→ Instance chạy trong private network (thường không có public IP)

Bước 2: Kiểm Tra EC2 Test Instance
│
├── Connectivity test:
│   ├── SSH/RDP vào test instance (qua bastion host hoặc SSM Session Manager)
│   ├── Kiểm tra OS, services, processes đang chạy đúng không
│   └── Ping/telnet đến database, cache servers phụ thuộc
│
├── Application test:
│   ├── Chạy smoke test (kiểm tra nhanh chức năng chính)
│   ├── Kiểm tra log files (không có lỗi nghiêm trọng không?)
│   ├── Verify disk space, permissions, cron jobs
│   └── Test kết nối đến các dịch vụ AWS (S3, RDS, SQS...)
│
└── Performance test (tùy chọn):
    ├── Load test nhẹ để xác nhận instance type phù hợp
    └── So sánh response time với server gốc

Bước 3: Đánh Giá Kết Quả
├── OK → Ghi nhận vào migration runbook, lên kế hoạch production cutover
└── Có vấn đề → Fix launch template, fix config, test lại

Bước 4: Dọn Dẹp Test Instance
→ MGN Console → Source Servers → Actions → Terminate test instances
→ Replication tiếp tục sync delta changes từ server gốc
```

### Checklist Kiểm Tra Khi Test Cutover

```markdown
## Test Cutover Checklist — [Server Name]

### Kiểm Tra Cơ Bản
- [ ] EC2 instance khởi động thành công (trạng thái: running)
- [ ] Kết nối SSH/RDP thành công
- [ ] Hostname đúng
- [ ] Network interface có IP đúng subnet
- [ ] Disk volumes đầy đủ và mount đúng

### Kiểm Tra Dịch Vụ
- [ ] Tất cả services tự động start (systemd units / Windows Services)
- [ ] Ứng dụng lắng nghe đúng port (netstat -tlnp)
- [ ] Log files không có ERROR mức critical
- [ ] Cron jobs / Scheduled Tasks đã được cấu hình

### Kiểm Tra Kết Nối
- [ ] Kết nối đến database thành công
- [ ] Kết nối đến cache (Redis/Memcached) thành công
- [ ] API calls đến các service phụ thuộc hoạt động
- [ ] DNS resolution hoạt động (resolve internal hostnames)

### Kiểm Tra Bảo Mật
- [ ] Security group mở đúng port, không mở thừa
- [ ] IAM Instance Profile đúng role
- [ ] Secrets (API keys, DB passwords) truy cập được (từ Secrets Manager hoặc Parameter Store)

### Kiểm Tra Hiệu Năng
- [ ] CPU và RAM ở mức hợp lý khi idle
- [ ] Disk I/O không bất thường
- [ ] Response time ứng dụng tương đương server gốc
```

---

## 🚀 Production Cutover — Cắt Chuyển Chính Thức

### Các Bước Production Cutover Với MGN

```
Phase 1: Chuẩn Bị Trước Cutover (T-1 tuần)
├── Hoàn thành ít nhất 1 lần test cutover thành công
├── Thông báo đến tất cả stakeholders về maintenance window
├── Chuẩn bị runbook chi tiết với time-boxed steps
├── Xác nhận rollback plan đã được test
├── Chuẩn bị team on-call: backup DBA, network engineer, app owner
└── Chốt danh sách DNS records, load balancer rules cần cập nhật

Phase 2: Ngay Trước Cutover (T-1 giờ)
├── Verify replication lag < 30 giây (kiểm tra trong MGN Console)
├── Tắt các scheduled jobs/cron nếu cần
├── Thông báo đến users về maintenance (nếu có planned downtime)
└── Team on-call sẵn sàng

Phase 3: Thực Hiện Cutover (Trong Maintenance Window)
│
├── T+0: Bắt đầu cutover
│   ├── Dừng application server trên server gốc (graceful shutdown)
│   ├── Chờ in-flight requests hoàn thành
│   └── Verify: không còn active connections
│
├── T+2 phút: Finalize replication trong MGN
│   ├── MGN Console → Source Servers → Actions → Mark as "Ready for cutover"
│   └── MGN thực hiện final sync để bắt kịp thay đổi cuối cùng
│
├── T+5 phút: Launch production EC2 instance
│   ├── MGN Console → Source Servers → Actions → Launch cutover instances
│   └── EC2 instance khởi động (~2-3 phút)
│
├── T+8 phút: Cập nhật DNS / Load Balancer
│   ├── Route 53: cập nhật A record / CNAME trỏ sang EC2 IP/hostname mới
│   │   └── TTL đã giảm xuống 60 giây từ trước (chuẩn bị từ T-24 giờ)
│   └── Load Balancer: thêm EC2 mới, chờ health check pass, xóa server cũ
│
└── T+15 phút: Xác nhận hoạt động
    ├── Smoke test nhanh: truy cập ứng dụng, kiểm tra chức năng chính
    ├── Monitor CloudWatch metrics (CPU, errors, latency)
    └── Xác nhận → thông báo cutover thành công

Phase 4: Sau Cutover (T+24-48 giờ)
├── Monitor kỹ trong 24-48 giờ đầu
├── Cập nhật CMDB (Configuration Management Database) và tài liệu
├── Cleanup Staging Area trong MGN (tiết kiệm chi phí)
├── Lên kế hoạch decommission server gốc (thường sau 2-4 tuần)
└── Gỡ cài đặt Replication Agent trên server gốc
```

---

## ⏰ Cutover Window — Cửa Sổ Thực Hiện Cutover

### Định Nghĩa Maintenance Window Phù Hợp

**Maintenance Window** — Cửa Sổ Bảo Trì — là khoảng thời gian ứng dụng có thể bị gián đoạn để thực hiện cutover. Chọn sai thời điểm là một trong những sai lầm phổ biến nhất.

```
Tiêu chí chọn Maintenance Window:

1. Traffic thấp nhất
   ├── Ứng dụng B2C (Consumer): thường là 2-4 giờ sáng
   ├── Ứng dụng B2B (Business): cuối tuần (Thứ 7 hoặc Chủ Nhật)
   └── Hệ thống tài chính: cuối ngày giao dịch (sau 6 PM hoặc cuối tuần)

2. Đủ thời gian thực hiện
   ├── Mỗi server: tối thiểu 30-60 phút để cutover + verify
   ├── Rollback buffer: thêm 30 phút dự phòng cho mỗi server
   └── Ví dụ: 5 server → window tối thiểu 4-5 giờ

3. Đội ngũ đủ năng lực
   ├── Không cutover khi team key đang nghỉ phép
   ├── Có người backup sẵn sàng
   └── Tránh ngày trước kỳ nghỉ lễ dài

4. Không trùng sự kiện quan trọng
   ├── Không cutover trước hoặc trong peak business event (Black Friday, end of quarter)
   └── Không cutover trong khi đang có incident khác
```

### Time-Boxing — Giới Hạn Thời Gian Cho Từng Bước

```
Mẫu Time-Boxing cho 3 server:

22:00 — Bắt đầu maintenance window, roll call team
22:05 — Server 1: graceful shutdown application
22:07 — Server 1: finalize replication, launch EC2
22:12 — Server 1: verify, update DNS
22:20 — Server 1: smoke test → PASS ✅
22:25 — Server 2: bắt đầu ...
23:00 — Server 2: PASS ✅
23:30 — Server 3: PASS ✅
23:45 — Tổng kết, thông báo hoàn thành

00:00 — ROLLBACK DEADLINE: nếu đến thời điểm này chưa xong, thực hiện rollback
         (không cố gắng hoàn thành khi đã quá deadline)
```

> **Nguyên tắc vàng:** Đặt **hard rollback deadline** trước khi bắt đầu. Nếu cutover không hoàn thành trước deadline, rollback ngay không phân vân. Cố gắng tiếp tục sau deadline thường dẫn đến sự cố nghiêm trọng hơn.

---

## ↩️ Rollback Strategy — Chiến Lược Quay Lui

### Tại Sao Phải Có Rollback Plan Trước?

```
Tình huống cần rollback thường gặp:
├── Ứng dụng không start được trên EC2 (do cấu hình thiếu, driver lỗi...)
├── Hiệu năng EC2 tệ hơn server gốc (instance type không phù hợp)
├── Kết nối mạng/firewall vấn đề (security group sai, NACL chặn)
├── Third-party service từ chối kết nối từ IP AWS (whitelist IP issue)
└── Database sync chưa hoàn thành → data inconsistency (dữ liệu không nhất quán)
```

### MGN Rollback — Cơ Chế Quay Lui

**Điểm quan trọng:** MGN **không có "rollback button"** tự động. Khả năng rollback phụ thuộc vào việc bạn **giữ server gốc sống** cho đến khi confirm production hoạt động ổn định.

```
Chiến lược rollback đúng với MGN:

KHÔNG NÊN (sai lầm phổ biến):
└── Tắt/decommission server gốc ngay sau cutover

NÊN LÀM (best practice):
├── Giữ server gốc ở trạng thái powered-off (tắt điện nhưng không xóa)
├── Thời gian giữ: tối thiểu 24-48 giờ, lý tưởng 1-2 tuần
└── Nếu cần rollback:
    ├── Bật lại server gốc
    ├── Cập nhật DNS/LB trỏ về server gốc
    └── Rollback hoàn tất trong vài phút
```

### Các Tình Huống Rollback Cụ Thể

```
Tình huống 1: Phát hiện vấn đề TRONG maintenance window
├── Quyết định rollback ngay → không chờ xem thêm
├── Server gốc vẫn ở trạng thái graceful shutdown → bật lại
├── Khởi động services trên server gốc
├── Cập nhật DNS/LB về server gốc (TTL 60 giây đã set sẵn)
└── Thông báo rollback hoàn thành → điều tra nguyên nhân

Tình huống 2: Phát hiện vấn đề SAU maintenance window (< 48 giờ)
├── Server gốc đang powered-off nhưng còn nguyên
├── Power on server gốc
├── Verify services start lại bình thường
├── Cập nhật DNS/LB
└── Tổng thời gian rollback: 5-15 phút

Tình huống 3: Phát hiện vấn đề sau khi đã decommission server gốc
├── Đây là tình huống XẤU NHẤT
├── Phải debug EC2 instance trực tiếp (không có rollback đơn giản)
├── Hoặc restore từ EBS snapshot (nếu đã tạo backup)
└── Bài học: không bao giờ decommission server gốc trong vòng 1-2 tuần đầu
```

### DNS TTL — Yếu Tố Quyết Định Tốc Độ Rollback

```
DNS TTL ảnh hưởng thời gian propagation khi cập nhật DNS:

TTL mặc định (ví dụ 3600 giây = 1 giờ):
└── Sau khi cập nhật DNS, người dùng vẫn resolve về IP cũ tối đa 1 giờ
    → Rollback DNS mất 1 giờ để có hiệu lực hoàn toàn

Best practice trước cutover (T-24 giờ):
├── Giảm TTL xuống 60 giây (hoặc 300 giây) cho các record quan trọng
└── Sau khi đã propagate (sau 1 giờ kể từ khi giảm TTL):
    → Khi cần rollback DNS, chỉ mất 60 giây để propagate
    → Sau khi cutover thành công và ổn định: tăng TTL lại về mức ban đầu
```

---

## ✅ Post-Cutover Validation — Xác Thực Sau Cutover

### Ngay Sau Cutover (0-1 giờ)

```
Tier 1 — Smoke Test (Kiểm Tra Nhanh):
├── Truy cập được ứng dụng từ browser/API client
├── Đăng nhập thành công (authentication hoạt động)
├── Thực hiện được một transaction nhỏ (đọc/ghi dữ liệu)
└── Không có error 5xx trong CloudWatch Logs

Tier 2 — Health Check Monitoring:
├── CloudWatch: CPU, Memory, Network I/O trong ngưỡng bình thường
├── Application logs: không có exception rate bất thường
├── Database connections: ổn định, không có connection pool exhaustion
└── Load balancer health checks: tất cả instances healthy
```

### 24-48 Giờ Đầu

```
Tier 3 — Functional Testing (Kiểm Tra Chức Năng):
├── Chạy regression test suite (nếu có)
├── Kiểm tra các business flows quan trọng (checkout, payment, login...)
├── Verify batch jobs/cron chạy đúng lịch và thành công
└── Xác nhận integrations với third-party systems hoạt động

Tier 4 — Performance Baseline:
├── So sánh response time với baseline trước migration (dùng CloudWatch vs. APM tool cũ)
├── Xác nhận auto-scaling hoạt động đúng (nếu đã cấu hình)
└── Review cost dashboard: chi phí EC2 có trong dự kiến không?
```

### Sau 1-2 Tuần

```
Tier 5 — Cleanup & Handover:
├── Decommission server gốc (chỉ khi đã confident 100%)
├── Cleanup MGN Staging Area
├── Gỡ cài đặt Replication Agent
├── Cập nhật documentation và CMDB
└── Chia sẻ lessons learned với team
```

---

## 📄 Runbook Mẫu — Template Thực Hành

```markdown
# Cutover Runbook — [Tên Ứng Dụng]

**Ngày Cutover:** [DD/MM/YYYY]
**Maintenance Window:** [HH:MM — HH:MM] (múi giờ: ICT UTC+7)
**Rollback Deadline:** [HH:MM]
**Team On-Call:**
- Migration Lead: [Tên] — [SĐT]
- Application Owner: [Tên] — [SĐT]
- DBA: [Tên] — [SĐT] (nếu có DB dependency)
- Network/Security: [Tên] — [SĐT]

## Pre-Cutover Checklist (T-1 giờ)
- [ ] Xác nhận replication lag < 30 giây
- [ ] DNS TTL đã giảm xuống 60 giây (check bằng: dig +noall +answer yourdomain.com)
- [ ] Tất cả test cutover đã PASS
- [ ] Rollback plan đã review với toàn team
- [ ] Monitoring alerts đang active (CloudWatch, PagerDuty...)
- [ ] Communication channel (Slack/Teams) đã sẵn sàng

## Cutover Steps
| Bước | Hành Động | Owner | Time Box | Status |
|------|-----------|-------|----------|--------|
| 1 | Announce bắt đầu maintenance | Migration Lead | 5 phút | [ ] |
| 2 | Graceful shutdown app trên server gốc | App Owner | 5 phút | [ ] |
| 3 | Finalize replication (Mark as Ready for Cutover) | Migration Lead | 5 phút | [ ] |
| 4 | Launch production EC2 instance | Migration Lead | 5 phút | [ ] |
| 5 | Verify EC2 instance running + SSH OK | Migration Lead | 3 phút | [ ] |
| 6 | Update DNS record | Network Engineer | 2 phút | [ ] |
| 7 | Wait for DNS propagation (60 giây) | All | 1 phút | [ ] |
| 8 | Smoke test từ external | App Owner | 5 phút | [ ] |
| 9 | Announce cutover complete | Migration Lead | 2 phút | [ ] |

## Rollback Steps (nếu cần)
| Bước | Hành Động | Owner | Time Box |
|------|-----------|-------|----------|
| R1 | Announce rollback decision | Migration Lead | 1 phút |
| R2 | Power on server gốc | Ops Team | 3 phút |
| R3 | Verify services started | App Owner | 3 phút |
| R4 | Revert DNS record | Network Engineer | 2 phút |
| R5 | Smoke test trên server gốc | App Owner | 5 phút |
| R6 | Announce rollback complete | Migration Lead | 1 phút |

## Post-Cutover Actions
- [ ] Monitor 24 giờ đầu (check mỗi 2 giờ)
- [ ] Gửi báo cáo cutover cho stakeholders
- [ ] Lên kế hoạch decommission server gốc (sau 2 tuần)
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Test cutover trong MGN là gì? Tại sao phải làm nhiều lần?**

> Test cutover là bước khởi động EC2 instance từ dữ liệu đã replicate để kiểm tra ứng dụng trên AWS, trong khi server gốc vẫn đang chạy bình thường. Sau test, instance được xóa và replication tiếp tục. Làm nhiều lần vì: (1) lần đầu thường phát hiện vấn đề về cấu hình network, driver, service startup; (2) mỗi lần sửa launch template cần verify lại; (3) giúp team làm quen với quy trình cutover để execution lúc production được nhanh và tự tin hơn.

**Q: Làm thế nào để rollback nếu production cutover thất bại?**

> Rollback với MGN phụ thuộc vào việc giữ server gốc còn sống. Best practice: sau khi graceful shutdown app trên server gốc, chỉ power-off (không xóa hoặc decommission) và giữ trong ít nhất 24-48 giờ. Nếu cần rollback: bật server gốc lên, start services, cập nhật DNS/load balancer về server gốc (DNS TTL đã giảm xuống 60 giây từ trước) → rollback hoàn tất trong vài phút. Không có tính năng "undo cutover" tự động trong MGN.

**Q: Tại sao cần giảm DNS TTL trước khi cutover?**

> DNS TTL (Time to Live — Thời Gian Tồn Tại) xác định bao lâu bản ghi DNS được cache bởi DNS resolvers. TTL mặc định thường 1-24 giờ. Nếu giữ TTL cao trong khi cutover, sau khi bạn cập nhật DNS trỏ sang EC2 mới, nhiều người dùng vẫn resolve về IP cũ trong vài giờ — gây lỗi. Còn nếu cần rollback, DNS cũng mất vài giờ mới có hiệu lực. Giảm TTL xuống 60 giây trước 24 giờ: khi cập nhật DNS (dù cutover hay rollback), chỉ mất 60 giây để propagate toàn bộ. Sau cutover thành công, tăng TTL lại về mức bình thường.

**Q: Hard rollback deadline nghĩa là gì và tại sao quan trọng?**

> Hard rollback deadline là thời điểm cứng — nếu cutover không hoàn thành thành công trước thời điểm này, team thực hiện rollback ngay lập tức không bàn cãi. Ví dụ: maintenance window 22:00-01:00, rollback deadline là 00:30. Quan trọng vì: tránh tâm lý "cố thêm 15 phút nữa" kéo dài vô tận; đảm bảo service phục hồi trước giờ cao điểm sáng hôm sau; buộc team phải chuẩn bị kỹ hơn thay vì trông cậy vào việc "tự sửa trong window"; giảm rủi ro mệt mỏi dẫn đến quyết định sai trong đêm.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
