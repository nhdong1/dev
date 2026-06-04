# Trusted Advisor — 5 Check Categories Chi Tiết

> Đây là phần chi tiết về **5 lĩnh vực kiểm tra** (check categories) của AWS Trusted Advisor, bao gồm danh sách đầy đủ các checks quan trọng, ngưỡng cảnh báo, và cách diễn giải kết quả để đưa ra hành động khắc phục phù hợp.

---

## 📚 Mục Lục

1. [Cách Đọc Kết Quả Check](#cách-đọc-kết-quả-check)
2. [Category 1: Cost Optimization — Tối Ưu Chi Phí](#category-1-cost-optimization--tối-ưu-chi-phí)
3. [Category 2: Performance — Hiệu Suất](#category-2-performance--hiệu-suất)
4. [Category 3: Security — Bảo Mật](#category-3-security--bảo-mật)
5. [Category 4: Fault Tolerance — Khả Năng Chịu Lỗi](#category-4-fault-tolerance--khả-năng-chịu-lỗi)
6. [Category 5: Service Limits — Giới Hạn Dịch Vụ](#category-5-service-limits--giới-hạn-dịch-vụ)
7. [Ma Trận Ưu Tiên Xử Lý](#ma-trận-ưu-tiên-xử-lý)
8. [Checks Phổ Biến Nhất Trong Phỏng Vấn](#checks-phổ-biến-nhất-trong-phỏng-vấn)

---

## Cách Đọc Kết Quả Check

### Trạng Thái (Status)

```
✅  OK (Green)      — Không có vấn đề, đáp ứng best practice
⚠️  Warning (Yellow)— Có thể có vấn đề, nên điều tra
❌  Error (Red)     — Có vấn đề rõ ràng, cần hành động ngay
🔵  Not Available  — Check không thể thực thi (thiếu quyền, chưa dùng)
```

### Tần Suất Refresh

```
Auto-refresh: Một lần/tuần (mỗi thứ Tư, theo múi giờ UTC)
On-demand refresh (Business+ Support):
  - Gọi API: RefreshTrustedAdvisorCheck
  - Console: Nút "Refresh" trên từng check
  - Rate limit: Không quá 1 lần/5 phút mỗi check
```

### Cơ Chế Phân Loại Tài Nguyên

```
Mỗi check trả về danh sách tài nguyên (resources), mỗi tài nguyên có:
{
  "status": "warning" | "ok" | "error",
  "region": "ap-southeast-1",
  "resourceId": "i-0abc123def456",
  "metadata": [...] ← Dữ liệu cụ thể tùy check
}
```

---

## Category 1: Cost Optimization — Tối Ưu Chi Phí

### Mục Tiêu
Phát hiện tài nguyên đang lãng phí tiền hoặc có cơ hội chuyển sang pricing model tiết kiệm hơn.

---

### Check: Idle EC2 Instances (Instances EC2 Không Hoạt Động)

**Điều kiện kích hoạt Warning:**
```
CPU utilization (Mức Sử Dụng CPU) ≤ 10%
TRONG: ít nhất 4 trong 14 ngày gần nhất
```

**Dữ liệu trả về:**
| Cột                      | Ý Nghĩa                               |
| ------------------------ | ------------------------------------- |
| Instance ID              | ID của EC2 instance                   |
| Instance Name            | Tag Name nếu có                       |
| Instance Type            | VD: m5.large                          |
| Estimated Monthly Savings| Số tiền tiết kiệm nếu tắt instance   |
| 14-day avg CPU           | CPU trung bình 14 ngày                |

**Hành Động Khuyến Nghị:**
```
Tùy mục đích instance:
├─ Instance thực sự idle → Terminate (xóa)
├─ Dùng không thường xuyên → Dùng Stop/Start theo lịch
│   (Lambda + EventBridge scheduler)
├─ Luôn bật nhưng tải thấp → Downsize instance type
└─ Workload burst → Migrate sang Spot Instances
```

---

### Check: Underutilized EBS Volumes (EBS Volumes Sử Dụng Thấp)

**EBS — Elastic Block Store — Lưu Trữ Khối Đàn Hồi**

**Điều kiện kích hoạt Warning:**
```
Volume không được gắn vào bất kỳ instance nào
HOẶC
Volume gắn nhưng:
  Read/Write IOPS ≤ 1 trong 7 ngày liên tiếp
```

**Lưu ý:** Volume trạng thái `available` (không gắn) vẫn tính phí đầy đủ.

**Hành Động Khuyến Nghị:**
```
├─ Volume không gắn → Tạo snapshot → Delete volume
├─ Volume gắn nhưng không dùng → Kiểm tra application
└─ gp2 volume → Migrate sang gp3 (rẻ hơn 20%, hiệu suất tốt hơn)
```

---

### Check: Unassociated Elastic IP Addresses (EIP Không Được Gán)

**EIP — Elastic IP Address — Địa Chỉ IP Đàn Hồi**

**Điều kiện kích hoạt Error:**
```
EIP đang tồn tại nhưng:
  KHÔNG gắn vào running instance
  HOẶC gắn vào instance nhưng instance đang Stopped
```

**Chi phí:** $0.005/giờ ≈ $3.6/tháng mỗi EIP không dùng (tính từ 2024)

**Hành Động Khuyến Nghị:**
```
├─ EIP thực sự không cần → Release (giải phóng)
├─ Cần giữ để dùng sau → Gắn vào running instance
└─ Dùng DNS thay vì EIP cố định khi có thể (Route 53 Alias)
```

---

### Check: Idle RDS DB Instances (RDS Không Hoạt Động)

**RDS — Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ**

**Điều kiện kích hoạt Warning:**
```
Số kết nối đến DB = 0
TRONG: ít nhất 7 ngày liên tiếp
```

**Hành Động Khuyến Nghị:**
```
├─ Môi trường dev/test → Dùng RDS scheduled start/stop
│   (Stop tối đa 7 ngày → tự khởi động lại, cần cơ chế tự stop lại)
├─ Không còn cần → Create snapshot → Delete instance
└─ Cần dùng không thường xuyên → Xem xét Aurora Serverless v2
```

---

### Check: Amazon EC2 Reserved Instance Optimization

**RI — Reserved Instance — Instance Đặt Trước**

**Phân tích dựa trên:**
```
Usage patterns 30 ngày gần nhất:
├─ Instance type được dùng thường xuyên
├─ Bao nhiêu giờ/ngày instance chạy
└─ So sánh On-Demand cost vs RI cost
```

**Trả về:**
- Loại RI được khuyến nghị mua (1-year hoặc 3-year)
- Ước tính tiết kiệm hàng tháng nếu mua RI

---

### Tóm Tắt Cost Optimization Checks

| Check                                    | Cấp Độ Alert | Tiết Kiệm Tiềm Năng |
| ---------------------------------------- | :----------: | :------------------: |
| Idle EC2 Instances                       | ⚠️ Warning   | Cao                  |
| Underutilized EBS Volumes                | ⚠️ Warning   | Trung bình           |
| Unassociated Elastic IP Addresses        | ❌ Error     | Thấp (nhưng dễ fix)  |
| Idle RDS DB Instances                    | ⚠️ Warning   | Rất cao              |
| Underutilized Redshift Clusters          | ⚠️ Warning   | Rất cao              |
| Reserved Instance Optimization           | ⚠️ Warning   | Cao                  |
| Savings Plans Optimization               | ⚠️ Warning   | Cao                  |

---

## Category 2: Performance — Hiệu Suất

### Mục Tiêu
Phát hiện tài nguyên đang bị quá tải hoặc cấu hình chưa tối ưu gây ảnh hưởng đến trải nghiệm người dùng.

---

### Check: High Utilization Amazon EC2 Instances

**Điều kiện kích hoạt Warning:**
```
CPU utilization > 90%
TRONG: ít nhất 4 ngày trong 14 ngày gần nhất
```

**Lưu ý:** Đây là điều ngược lại của Idle EC2 Instances check — cùng một instance type nhưng đánh giá ngưỡng khác nhau.

**Hành Động Khuyến Nghị:**
```
├─ Scale Up: Chuyển sang instance type lớn hơn
├─ Scale Out: Thêm instance + ALB + Auto Scaling Group
│   (ASG — Auto Scaling Group — Nhóm Tự Động Mở Rộng)
├─ Optimize: Profile application để tìm bottleneck thực sự
└─ Cache: Thêm ElastiCache (Redis/Memcached) để giảm tải CPU
```

---

### Check: Large Number of Rules in EC2 Security Group

**Điều kiện kích hoạt Warning:**
```
Security Group có:
  > 50 rules (inbound + outbound cộng lại)
```

**Ảnh hưởng đến hiệu suất:**
```
Kernel phải xử lý mọi packet qua tất cả rules theo thứ tự
→ Nhiều rules → Latency tăng → Throughput giảm
→ Đặc biệt ảnh hưởng đến high-throughput applications
```

**Hành Động Khuyến Nghị:**
```
├─ Gộp IP ranges vào CIDR blocks rộng hơn khi có thể
├─ Dùng Security Group reference thay vì IP cụ thể
│   (Cho phép traffic từ SG-A thay vì list IP của SG-A)
├─ Tách workload vào Security Groups riêng biệt
└─ Xem xét AWS Network Firewall hoặc VPC prefix lists
```

---

### Check: CloudFront Header Forwarding and Cache Hit Ratio

**Cache Hit Ratio — Tỉ Lệ Trả Về Từ Cache**

**Điều kiện kích hoạt Warning:**
```
Cache hit ratio < 80%
(tức là > 20% requests phải đi lên origin)
```

**Nguyên nhân phổ biến:**
```
Forwards quá nhiều headers → mỗi tổ hợp header = cache key riêng biệt
→ Cache bị phân mảnh → hit rate thấp → origin bị tải nhiều

Ví dụ vấn đề:
  Forward: Accept-Language, User-Agent, Accept-Encoding
  → Mỗi browser/OS/ngôn ngữ = cache entry riêng
  → 1000 user khác nhau = 1000 cache miss
```

**Hành Động Khuyến Nghị:**
```
├─ Chỉ forward headers cần thiết cho application logic
├─ Dùng Cache Policies và Origin Request Policies rõ ràng
├─ Tách static content (CSS/JS/images) ra distribution riêng
│   với zero header forwarding
└─ Xem xét Lambda@Edge cho dynamic customization
```

---

### Check: EBS Provisioned IOPS (SSD) Overprovisioned

**IOPS — Input/Output Operations Per Second — Số Thao Tác I/O Mỗi Giây**

**Điều kiện kích hoạt Warning:**
```
Provisioned IOPS (io1/io2) được cấp phát
NHƯNG thực tế dùng < 20% provisioned IOPS trong 14 ngày
```

**Hành Động Khuyến Nghị:**
```
├─ Giảm provisioned IOPS xuống mức thực tế dùng × 1.2 (buffer)
├─ Migrate từ io1/io2 sang gp3 nếu IOPS < 16,000
│   gp3: 3,000 IOPS base + tùy chỉnh riêng, rẻ hơn io1/io2
└─ Monitor trước khi thay đổi để tránh gây ra throttling
```

---

### Check: Amazon Route 53 Alias Resource Record Sets

**Điều kiện kích hoạt Warning:**
```
Route 53 record point đến:
  - ELB endpoint
  - CloudFront distribution
  - S3 website endpoint
NHƯNG dùng A record thay vì Alias record
```

**Lý do quan trọng:**
```
A Record thông thường:
  DNS query → trả về IP cố định
  → IP thay đổi (ELB scale) → DNS cache outdate → lỗi kết nối

Alias Record (đặc biệt của Route 53):
  DNS query → Route 53 resolve trực tiếp sang IPs hiện tại của ELB
  → Không bị cache issue
  → Không tốn phí DNS query thêm
  → Hỗ trợ Zone Apex (ví dụ: example.com thay vì www.example.com)
```

---

## Category 3: Security — Bảo Mật

### Mục Tiêu
Phát hiện cấu hình không an toàn có thể dẫn đến data breach, unauthorized access, hoặc vi phạm compliance.

---

### Check: MFA on Root Account ⭐ (Quan Trọng Nhất)

**MFA — Multi-Factor Authentication — Xác Thực Đa Yếu Tố**

**Điều kiện kích hoạt Error:**
```
Root account của AWS account CHƯA bật MFA
```

**Tại sao nghiêm trọng nhất:**
```
Root account có quyền làm mọi thứ, không thể bị SCP giới hạn
→ Nếu root credentials bị lộ mà không có MFA
→ Attacker có thể:
   - Xóa toàn bộ tài nguyên
   - Tạo IAM admin user ẩn
   - Thay đổi billing information
   - Không thể thu hồi access kịp thời
```

**Hành Động:**
```
1. Đăng nhập bằng root account (chỉ làm việc này khi cần thiết)
2. My Security Credentials → Multi-factor authentication (MFA)
3. Assign MFA device (Virtual MFA: Google Authenticator hoặc phần cứng: YubiKey)
4. Lưu backup codes an toàn
```

---

### Check: IAM Access Key Rotation

**Điều kiện kích hoạt Warning/Error:**
```
Warning: Access key không được xoay vòng trong 90 ngày
Error:   Access key không được xoay vòng trong 180 ngày
```

**Best Practice:**
```
├─ Xoay vòng (rotate) key mỗi 90 ngày
├─ Dùng IAM Roles thay vì Access Keys khi có thể
│   (EC2 Instance Profile, Lambda Execution Role, ECS Task Role)
├─ Dùng AWS Secrets Manager để rotate tự động
└─ Monitor việc dùng key cũ qua CloudTrail trước khi disable
```

**Quy trình Rotate an toàn:**
```
1. Tạo key MỚI (Create new access key)
2. Cập nhật ứng dụng dùng key mới
3. Verify ứng dụng chạy tốt với key mới
4. Disable key CŨ (chưa xóa, để rollback nếu cần)
5. Theo dõi 1-2 ngày xem có lỗi không
6. Delete key CŨ
```

---

### Check: Amazon S3 Bucket Permissions

**Điều kiện kích hoạt Warning/Error:**
```
Warning: Bucket ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập)
         cho phép public READ
Error:   Bucket ACL hoặc Policy cho phép public READ + WRITE
```

**Phân biệt Public Access:**
```
"Public" theo Trusted Advisor có nghĩa là bất kỳ ai trên internet đều có thể:
  - READ: Download file từ bucket
  - WRITE: Upload/Delete file vào bucket (cực kỳ nguy hiểm)
```

**Hành Động:**
```
Nếu bucket CẦN public (static website hosting):
  ├─ Chỉ cho phép s3:GetObject (READ only)
  ├─ Không bao giờ cho phép public WRITE
  └─ Dùng CloudFront + OAC thay vì direct public access

Nếu bucket KHÔNG cần public:
  ├─ S3 Block Public Access → Enable tất cả 4 settings
  ├─ Xóa public ACL
  └─ Xem xét S3 Object Ownership = Bucket owner enforced
```

---

### Check: Security Groups — Unrestricted Access

**Điều kiện kích hoạt Error (mỗi port riêng):**

| Port | Protocol | Nguồn Nguy Hiểm            | Lý Do                         |
| ---- | -------- | -------------------------- | ----------------------------- |
| 22   | TCP      | 0.0.0.0/0 hoặc ::/0       | SSH brute force attack        |
| 3389 | TCP      | 0.0.0.0/0 hoặc ::/0       | RDP attack vào Windows        |
| 3306 | TCP      | 0.0.0.0/0 hoặc ::/0       | MySQL/MariaDB public access   |
| 5432 | TCP      | 0.0.0.0/0 hoặc ::/0       | PostgreSQL public access      |
| 1433 | TCP      | 0.0.0.0/0 hoặc ::/0       | MSSQL public access           |
| 27017| TCP      | 0.0.0.0/0 hoặc ::/0       | MongoDB public access         |

**Hành Động:**
```
├─ Thay 0.0.0.0/0 bằng IP cụ thể hoặc range của công ty
├─ Dùng SSM Session Manager thay SSH (không cần port 22)
│   (SSM — AWS Systems Manager — Quản Lý Hệ Thống AWS)
├─ Dùng RDS endpoint bên trong VPC private subnet
│   (VPC — Virtual Private Cloud — Đám Mây Riêng Ảo)
└─ VPN hoặc AWS Direct Connect cho admin access
```

---

### Check: AWS CloudTrail Logging

**Điều kiện kích hoạt Warning/Error:**
```
Warning: CloudTrail Trail tồn tại nhưng logging bị tắt
Error:   Không có Trail nào được tạo trong account
```

**Hành Động:**
```
├─ Bật CloudTrail multi-region trail (không chỉ single-region)
├─ Bật log file validation để phát hiện tampered logs
│   (Log File Validation — Xác Thực Tính Toàn Vẹn File Log)
├─ Gửi logs vào S3 + CloudWatch Logs để query được
└─ Cân nhắc Organization Trail nếu dùng AWS Organizations
```

---

### Check: IAM Password Policy

**Điều kiện kích hoạt Warning nếu thiếu một trong các yêu cầu:**
```
✅ Độ dài tối thiểu ≥ 8 ký tự
✅ Yêu cầu ít nhất 1 ký tự viết hoa
✅ Yêu cầu ít nhất 1 ký tự viết thường
✅ Yêu cầu ít nhất 1 số
✅ Yêu cầu ít nhất 1 ký tự đặc biệt
✅ Không cho phép dùng lại password gần nhất
✅ Password expiration (khuyến nghị 90 ngày)
```

---

### Checks Bảo Mật Khác Quan Trọng

| Check                              | Mức Độ  | Mô Tả Ngắn                                       |
| ---------------------------------- | :-----: | ------------------------------------------------- |
| EBS Public Snapshots               | ❌ Error | Snapshot chia sẻ công khai — bất kỳ ai đọc được  |
| RDS Public Snapshots               | ❌ Error | DB snapshot chia sẻ công khai                    |
| Exposed Access Keys                | ❌ Error | Key bị lộ trên public source (GitHub scan)        |
| AWS Well-Architected High Risk     | ⚠️ Warning| High risk issues trong Well-Architected review   |

---

## Category 4: Fault Tolerance — Khả Năng Chịu Lỗi

### Mục Tiêu
Phát hiện single point of failure — điểm đơn lẻ có thể gây ra downtime cho toàn hệ thống.

---

### Check: Amazon RDS Multi-AZ ⭐

**Multi-AZ — Multiple Availability Zone — Nhiều Vùng Khả Dụng**

**Điều kiện kích hoạt Warning:**
```
RDS instance production (ước tính theo size) đang chạy
NHƯNG không bật Multi-AZ
```

**Giải thích Multi-AZ:**
```
Single-AZ:
  ┌──────────────────────────────┐
  │  AZ-1a                       │
  │  ┌─────────────────────────┐ │
  │  │ RDS Primary (Read/Write)│ │
  │  └─────────────────────────┘ │
  └──────────────────────────────┘
  Nếu AZ-1a lỗi → Database DOWN hoàn toàn

Multi-AZ:
  ┌──────────────────────────────┐  ┌──────────────────────────────┐
  │  AZ-1a                       │  │  AZ-1b                       │
  │  ┌─────────────────────────┐ │  │  ┌─────────────────────────┐ │
  │  │ RDS Primary (Read/Write)│◄───sync─►RDS Standby (Passive) │ │
  │  └─────────────────────────┘ │  │  └─────────────────────────┘ │
  └──────────────────────────────┘  └──────────────────────────────┘
  Nếu AZ-1a lỗi → Failover tự động sang AZ-1b trong 60-120 giây
```

**Lưu ý:** Multi-AZ Standby KHÔNG phục vụ read traffic — dùng Read Replica cho read scaling.

---

### Check: Amazon RDS Backups

**Điều kiện kích hoạt Warning:**
```
RDS instance có:
  Backup retention period = 0 ngày
  (tức là automated backups bị TẮT hoàn toàn)
```

**Hậu quả nghiêm trọng:**
```
Backup tắt → Không có automated backup
→ Nếu database bị corrupt hoặc xóa nhầm:
   → Không thể restore từ point-in-time
   → Không thể undo transaction sai
   → Data loss hoàn toàn kể từ lần backup thủ công cuối
```

**Hành Động:**
```
├─ Bật automated backup: retention ≥ 7 ngày (khuyến nghị 35 ngày)
├─ Test restore định kỳ để verify backup chạy được
└─ Với production DB: Dùng Multi-AZ + backup + cross-region snapshot
```

---

### Check: Amazon EBS Snapshots

**Điều kiện kích hoạt Warning:**
```
EBS volume gắn vào running instance
NHƯNG không có snapshot nào trong 7 ngày gần nhất
```

**Best Practice:**
```
├─ Dùng Amazon Data Lifecycle Manager (DLM) để tự động snapshot
│   (DLM — Data Lifecycle Manager — Quản Lý Vòng Đời Dữ Liệu)
├─ Schedule: Hàng ngày cho production, hàng tuần cho non-prod
├─ Retention: 7-30 ngày tùy RPO yêu cầu
│   (RPO — Recovery Point Objective — Mục Tiêu Điểm Khôi Phục)
└─ Cross-region copy cho DR (Disaster Recovery — Khôi Phục Thảm Họa)
```

---

### Check: Amazon EC2 Availability Zone Balance

**Điều kiện kích hoạt Warning:**
```
Auto Scaling Group hoặc nhóm instances:
  Phân bố không đều giữa các AZ
  VD: AZ-1a: 8 instances, AZ-1b: 2 instances, AZ-1c: 0 instances
```

**Vấn đề:**
```
Nếu AZ-1a (có 8 instances) lỗi:
  → Chỉ còn 2 instances ở AZ-1b
  → Capacity giảm 80% thay vì giảm 33% (balanced)
  → Có thể không đủ capacity để phục vụ traffic
```

**Hành Động:**
```
├─ Bật ASG Rebalancing: Auto Scaling tự điều chỉnh AZ balance
├─ Đảm bảo mỗi AZ có đủ EC2 capacity để launch
└─ Dùng "Balance" strategy thay vì "Lowest Price" với Spot
```

---

### Check: Elastic Load Balancer Optimization

**Điều kiện kích hoạt Warning:**
```
ELB (Elastic Load Balancer — Cân Bằng Tải Đàn Hồi) có:
  - Cross-zone load balancing bị tắt
  - Chỉ enable ở 1 AZ (không dùng multi-AZ)
  - Health check configured quá lỏng
```

---

### Check: Amazon Route 53 High TTL Resource Records

**TTL — Time To Live — Thời Gian Tồn Tại (DNS cache)**

**Điều kiện kích hoạt Warning:**
```
Route 53 record có TTL > 900 giây (15 phút) cho:
  - Health-checked records
  - Failover routing records
```

**Vấn đề với TTL cao khi failover:**
```
TTL = 3600 giây (1 giờ):
  Primary endpoint lỗi → Route 53 muốn chuyển sang Secondary
  NHƯNG: DNS resolvers và client đang cache IP của Primary 1 giờ
  → Trong 1 giờ tiếp theo: Tiếp tục gọi đến Primary đang lỗi
  → Downtime = đến khi DNS cache expire

TTL = 60 giây:
  Primary endpoint lỗi → Route 53 failover
  Sau tối đa 60 giây: DNS resolvers cập nhật → Chuyển sang Secondary
  → Downtime ≤ 60 giây + health check interval
```

**Hành Động:**
```
├─ Giảm TTL của failover records xuống 60–120 giây
├─ Config Route 53 Health Checks: Interval 10s, threshold 2 failures
└─ Test failover định kỳ để verify hoạt động đúng
```

---

## Category 5: Service Limits — Giới Hạn Dịch Vụ

### Mục Tiêu
Cảnh báo sớm khi tài nguyên đang tiếp cận giới hạn (quota) của dịch vụ để tránh bị throttle hoặc không thể tạo thêm tài nguyên.

---

### Cách Trusted Advisor Tính Ngưỡng

```
Xanh (OK):      Sử dụng < 80% quota
Vàng (Warning): Sử dụng ≥ 80% và < 100% quota
Đỏ (Error):     Sử dụng = 100% quota (đã chạm giới hạn)
```

---

### Các Service Limits Checks Phổ Biến

| Service              | Giới Hạn Được Kiểm Tra                            | Default Quota Điển Hình   |
| -------------------- | ------------------------------------------------- | ------------------------- |
| EC2                  | vCPU On-Demand Running Instances                  | 32–96 vCPU (tùy type)    |
| VPC                  | VPCs per region                                   | 5                         |
| VPC                  | Subnets per VPC                                   | 200                       |
| ELB                  | Application Load Balancers per region             | 50                        |
| IAM                  | Roles per account                                 | 1,000                     |
| IAM                  | Managed Policies per account                      | 1,500                     |
| RDS                  | DB Instances per region                           | 40                        |
| S3                   | Buckets per account                               | 100                       |
| CloudFormation       | Stacks per account per region                     | 2,000                     |
| Lambda               | Concurrent executions per region                  | 1,000                     |
| SES                  | Max Send Rate                                     | 1 message/giây (sandbox)  |
| EIP                  | Elastic IP Addresses per region                   | 5                         |

---

### Service Limits vs Service Quotas

```
Trusted Advisor "Service Limits" checks
        = dữ liệu từ AWS Service Quotas console

Để tăng quota:
  1. Cách 1 — Service Quotas Console:
     Service Quotas → Chọn service → Request quota increase
     (Tự động hóa được, có API)

  2. Cách 2 — AWS Support Case:
     Support → Create case → Service limit increase
     (Cần điền form thủ công)

  3. Cách 3 — API:
     service_quotas_client.request_service_quota_increase(
         ServiceCode='ec2',
         QuotaCode='L-1216C47A',
         DesiredValue=200
     )
```

---

## Ma Trận Ưu Tiên Xử Lý

Khi có nhiều issues, ưu tiên theo thứ tự:

```
Ưu Tiên 1 — CRITICAL (Xử lý ngay trong vài giờ):
  ❌ MFA on Root Account
  ❌ Security Groups 0.0.0.0/0 cho port database/SSH
  ❌ S3 Public WRITE permissions
  ❌ Exposed Access Keys

Ưu Tiên 2 — HIGH (Xử lý trong sprint này):
  ❌ RDS không có Multi-AZ (production)
  ❌ RDS backup tắt
  ❌ Service Limits ≥ 100% (đã chạm giới hạn)
  ❌ CloudTrail không bật

Ưu Tiên 3 — MEDIUM (Lên kế hoạch trong tháng):
  ⚠️ Idle EC2/RDS instances (lãng phí chi phí)
  ⚠️ Service Limits ≥ 80% (sắp chạm giới hạn)
  ⚠️ IAM Access Key không rotate
  ⚠️ EBS volumes không có snapshot

Ưu Tiên 4 — LOW (Cải thiện khi có thời gian):
  ⚠️ CloudFront cache hit rate thấp
  ⚠️ Route 53 TTL cao
  ⚠️ RI/Savings Plans optimization
  ⚠️ AZ balance cho ASG
```

---

## Checks Phổ Biến Nhất Trong Phỏng Vấn

### "Trusted Advisor check nào bạn thấy quan trọng nhất?"

**Câu trả lời mẫu theo thứ tự ưu tiên:**

```
Security (Bắt buộc cho mọi account):
1. MFA on Root Account — Không có MFA = rủi ro catastrophic
2. Security Groups 0.0.0.0/0 — Attack surface lớn nhất
3. CloudTrail Logging — Không có audit = không điều tra được

Fault Tolerance (Production readiness):
4. RDS Multi-AZ — Single point of failure cho database
5. RDS Backups — RPO = infinity nếu không có backup

Cost (Tối ưu vận hành):
6. Idle EC2 Instances — Hay xảy ra trong môi trường dev/staging
7. Unassociated EIPs — Dễ bỏ quên, tích lũy theo thời gian
```

### "Làm sao bạn tích hợp Trusted Advisor vào CI/CD pipeline?"

```
1. EventBridge rule → Capture Trusted Advisor check status changes
2. Lambda → Phân loại severity, gửi alert, tạo JIRA ticket tự động
3. Support API → Pull results hàng ngày, push vào dashboard nội bộ
4. Security check violations → Block deployment đến production
   (VD: Nếu S3 bucket policy thay đổi → public → block PR merge)
```

---

## Điều Hướng

| Điều Hướng                       | Link                                                             |
| --------------------------------- | ---------------------------------------------------------------- |
| ← README                          | [README.md](./README.md)                                         |
| → Tiếp: Programmatic Access       | [2-programmatic-access.md](./2-programmatic-access.md)           |
| ↑ Chỉ Mục                         | [INDEX.md](../INDEX.md)                                          |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
