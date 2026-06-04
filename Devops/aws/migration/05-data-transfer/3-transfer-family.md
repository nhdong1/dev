# AWS Transfer Family — Managed SFTP/FTPS/FTP/AS2 trên S3 và EFS

> **AWS Transfer Family** là dịch vụ managed (có quản lý) cho phép truyền file qua các giao thức tiêu chuẩn: **SFTP** — SSH File Transfer Protocol (Giao Thức Truyền File Qua SSH), **FTPS** — FTP over SSL/TLS (FTP Có Mã Hóa SSL/TLS), **FTP** — File Transfer Protocol (Giao Thức Truyền File), và **AS2** — Applicability Statement 2 (Chuẩn Trao Đổi File B2B). Dữ liệu upload được lưu thẳng vào **Amazon S3** hoặc **Amazon EFS** mà không cần quản lý server.

## 📚 Mục Lục (Table of Contents)

1. [Transfer Family là gì và khi nào dùng?](#transfer-family-là-gì-và-khi-nào-dùng)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Các Giao Thức Hỗ Trợ](#các-giao-thức-hỗ-trợ)
4. [Identity Providers — Xác Thực Người Dùng](#identity-providers--xác-thực-người-dùng)
5. [Storage Backend — S3 và EFS](#storage-backend--s3-và-efs)
6. [Logical Directory Mapping — Ánh Xạ Thư Mục](#logical-directory-mapping--ánh-xạ-thư-mục)
7. [Security và Compliance](#security-và-compliance)
8. [AS2 — Trao Đổi File B2B](#as2--trao-đổi-file-b2b)
9. [Workflow Automation — Tự Động Hóa Sau Upload](#workflow-automation--tự-động-hóa-sau-upload)
10. [Monitoring và Logging](#monitoring-và-logging)
11. [So Sánh vs Tự Dựng SFTP Server](#so-sánh-vs-tự-dựng-sftp-server)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Transfer Family là gì và khi nào dùng?

### Transfer Family là gì?

```
AWS Transfer Family = Managed SFTP/FTPS/FTP/AS2 server
                      File lưu vào S3 hoặc EFS
                      Không cần quản lý EC2 instance, SSL certs, user accounts

Trước Transfer Family (tự dựng SFTP):
├── Dựng EC2 instance, cài OpenSSH
├── Quản lý user accounts, SSH keys
├── Cấu hình SSL certificate cho FTPS
├── Patch server định kỳ (security updates)
├── Monitor server uptime
└── Scale khi load tăng → thêm server, load balancer

Với Transfer Family:
├── AWS quản lý toàn bộ server infrastructure
├── Chỉ cần cấu hình: users, protocols, storage backend
└── Auto-scale (tự động mở rộng), high availability tích hợp sẵn
```

### Khi nào dùng Transfer Family?

```
Phù hợp:
├── Đối tác B2B (Business-to-Business) gửi file qua SFTP hàng ngày
│   └── Ví dụ: nhà cung cấp gửi file đơn hàng XML
├── Thay thế SFTP server nội bộ tốn kém bảo trì
├── Data ingestion pipeline (đường ống nhập dữ liệu) từ đối tác
├── B2B commerce EDI (Electronic Data Interchange) với AS2
└── Legacy system chỉ hỗ trợ SFTP/FTP

Không phù hợp:
├── Transfer lượng lớn TB dữ liệu → DataSync (chuyên cho bulk transfer)
├── Real-time streaming data → Kinesis / MSK
└── Database migration → DMS
```

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AWS Transfer Family Architecture                    │
├──────────────────────────┬──────────────────────────────────────────────┤
│    External Clients      │              AWS Cloud                        │
│                          │                                               │
│  ┌────────────────┐      │  ┌─────────────────────────────────────────┐ │
│  │ Partner SFTP   │      │  │         Transfer Family Server           │ │
│  │ client         ├──────┼─→│  Endpoint:                               │ │
│  │ (FileZilla,    │      │  │  s-xxxx.server.transfer.region.amazonaws │ │
│  │  WinSCP, etc.) │      │  │  .com                                    │ │
│  └────────────────┘      │  │                                          │ │
│                          │  │  ┌─────────────────────────────────┐    │ │
│  ┌────────────────┐      │  │  │  Identity Provider               │    │ │
│  │ Internal users │      │  │  │  Service-managed / AWS Secrets   │    │ │
│  │ (finance team) ├──────┼─→│  │  Manager / Active Directory /   │    │ │
│  └────────────────┘      │  │  │  Custom Lambda authorizer        │    │ │
│                          │  │  └──────────────┬──────────────────┘    │ │
│                          │  │                 │                         │ │
│                          │  │  ┌──────────────▼──────────────────┐    │ │
│                          │  │  │         Storage Backend          │    │ │
│                          │  │  │   Amazon S3   |   Amazon EFS    │    │ │
│                          │  │  └─────────────────────────────────┘    │ │
│                          │  └─────────────────────────────────────────┘ │
└──────────────────────────┴──────────────────────────────────────────────┘
```

---

## 📡 Các Giao Thức Hỗ Trợ

### 1. SFTP — SSH File Transfer Protocol (Giao Thức Truyền File Qua SSH)

```
Đặc điểm:
├── Mã hóa: SSH (Secure Shell — Giao Thức Bảo Mật Shell) — mã hóa toàn bộ kết nối
├── Port mặc định: 22
├── Authentication: SSH key pair hoặc password
├── Phổ biến nhất trong môi trường Unix/Linux
└── Hầu hết SFTP clients (FileZilla, WinSCP, Cyberduck) đều hỗ trợ

Khi nào dùng SFTP:
├── Đối tác/developer đã có SSH keypair sẵn
├── Linux/Unix environment
└── Yêu cầu mã hóa cao nhất

Cấu hình user với SSH key:
└── Upload public key của user → Transfer Family lưu trong user config
    User kết nối bằng private key tương ứng (không cần mật khẩu)
```

### 2. FTPS — FTP over SSL/TLS (FTP Có Mã Hóa SSL/TLS)

```
Đặc điểm:
├── Mã hóa: TLS (Transport Layer Security — Bảo Mật Tầng Truyền Tải)
├── Port mặc định: 21 (control), 989/990 (data)
├── Authentication: username/password với TLS certificate
└── Hai chế độ:
    ├── Explicit FTPS: Bắt đầu bằng FTP thông thường, upgrade lên TLS
    └── Implicit FTPS: TLS ngay từ đầu kết nối

Khi nào dùng FTPS:
├── Legacy system Windows chỉ hỗ trợ FTPS (không có SSH)
└── Đối tác yêu cầu dùng certificate cụ thể của công ty

Lưu ý: Transfer Family dùng ACM (AWS Certificate Manager) để quản lý TLS cert
```

### 3. FTP — File Transfer Protocol (Giao Thức Truyền File Không Mã Hóa)

```
Đặc điểm:
├── KHÔNG có mã hóa — dữ liệu truyền plaintext (văn bản thuần)
├── Port mặc định: 21
└── Username/password không được mã hóa → chỉ dùng trong mạng nội bộ

Khi nào dùng FTP:
├── Chỉ trong VPC nội bộ (không ra internet)
├── Legacy application không hỗ trợ SFTP hay FTPS
└── Transfer Family FTP endpoint chỉ có thể là VPC-only (không có public internet)

⚠️ Cảnh báo bảo mật:
└── KHÔNG BAO GIỜ expose FTP endpoint ra internet — credentials và data đều không mã hóa
```

### 4. AS2 — Applicability Statement 2 (Chuẩn Trao Đổi File B2B)

```
Đặc điểm:
├── Chuẩn trao đổi file trong B2B commerce (thương mại giữa doanh nghiệp)
├── Mã hóa payload và digital signature (chữ ký số)
├── Non-repudiation (không thể phủ nhận): cả hai bên ký xác nhận receipt
└── Phổ biến trong EDI (Electronic Data Interchange — Trao Đổi Dữ Liệu Điện Tử)

Khi nào dùng AS2:
├── Trao đổi tài liệu thương mại: Purchase Order (đơn mua hàng), Invoice (hóa đơn)
├── Ngành retail (bán lẻ), healthcare, logistics yêu cầu AS2
└── Đối tác yêu cầu dùng AS2 theo hợp đồng

AS2 flow:
├── Sender → encrypt message với public key của receiver
├── Sender → sign message với private key của mình
├── Receiver → decrypt, verify signature
└── Receiver → gửi MDN (Message Disposition Notification — Xác Nhận Nhận Tin) lại sender
```

### So sánh bốn giao thức

| Tiêu Chí | SFTP | FTPS | FTP | AS2 |
| -------- | ---- | ---- | --- | --- |
| **Mã hóa** | SSH | TLS | Không | Có (payload) |
| **Port** | 22 | 21, 989/990 | 21 | 443 (HTTPS) |
| **Authentication** | SSH key / password | Certificate + password | Username/password | X.509 certificate |
| **Phổ biến** | ⭐⭐⭐ | ⭐⭐ | ⭐ (nội bộ) | ⭐⭐ (B2B) |
| **Internet-safe** | ✅ | ✅ | ❌ | ✅ |
| **Non-repudiation** | ❌ | ❌ | ❌ | ✅ |

---

## 👤 Identity Providers — Xác Thực Người Dùng

### 1. Service-managed (Quản lý trong Transfer Family)

```
Đơn giản nhất — AWS lưu user và credentials:
├── Tạo user trực tiếp trong Transfer Family Console
├── Gán SSH public key (cho SFTP) hoặc password
├── Giới hạn: không tích hợp với hệ thống IAM phức tạp
└── Phù hợp: ít users, không có existing identity system

Ví dụ tạo user service-managed:
1. Transfer Family Console → Users → Add user
2. Username: partner-acme
3. Role: IAM role với quyền truy cập S3 bucket
4. Home directory: s3://my-bucket/partners/acme/
5. SSH public key: paste public key của đối tác
```

### 2. AWS Secrets Manager (Lưu Credentials Trong Secrets Manager)

```
Phù hợp khi cần lưu password (không chỉ SSH key):
├── Mỗi user có một secret (bí mật) trong Secrets Manager
├── Secret chứa: password hash (băm mật khẩu), home directory, role ARN
├── Khi user login: Transfer Family gọi Lambda → Lambda lookup Secrets Manager
└── Lợi thế: rotate password (thay mật khẩu định kỳ) tự động

Cấu trúc secret JSON:
{
  "Password": "hashed_password_here",
  "Role": "arn:aws:iam::123456789:role/transfer-user-role",
  "HomeDirectory": "/my-bucket/users/johndoe",
  "PublicKeys": ["ssh-rsa AAAA..."]
}
```

### 3. AWS Directory Service / Active Directory (Tích Hợp AD)

```
Dùng khi công ty đã có Active Directory (Thư Mục Người Dùng Windows):
├── Kết nối Transfer Family với AWS Managed Microsoft AD
│   hoặc AD Connector (kết nối AD on-premises)
├── User login bằng AD credentials (username/password của Windows domain)
├── Không cần tạo lại user trong Transfer Family
└── Phù hợp: enterprise với nhiều user nội bộ đã có trong AD
```

### 4. Custom Identity Provider qua Lambda (Tùy Chỉnh Qua Lambda)

```
Linh hoạt nhất — gọi Lambda function (hàm Lambda) để xác thực:
├── User login → Transfer Family gọi API Gateway → Lambda
├── Lambda kiểm tra credentials với bất kỳ hệ thống nào:
│   ├── Database (MySQL, DynamoDB)
│   ├── LDAP server
│   ├── External API
│   └── Bất kỳ custom logic nào
└── Lambda trả về: allowed/denied + home directory + IAM role

Khi nào dùng:
└── Khi không có AD, và cần tích hợp với hệ thống identity đặc biệt
```

---

## 💾 Storage Backend — S3 và EFS

### Amazon S3 Backend

```
Mapping file path → S3 object key:
├── User home: /my-bucket/users/partner-acme/
├── User upload: report.csv
└── S3 object: s3://my-bucket/users/partner-acme/report.csv

S3 storage classes cho Transfer Family:
├── Mặc định: S3 Standard
├── Có thể dùng S3 Lifecycle rules (quy tắc vòng đời) để tự động archive
│   └── Sau 30 ngày: chuyển sang S3 Standard-IA
│   └── Sau 90 ngày: chuyển sang S3 Glacier
└── Không thể ghi trực tiếp vào Glacier (phải dùng lifecycle)

Lợi thế S3:
├── Unlimited storage (lưu trữ không giới hạn)
├── Tích hợp với S3 Event Notifications, Lambda, SQS, SNS
└── S3 Versioning (lưu nhiều phiên bản file) tự động
```

### Amazon EFS Backend

```
Dùng khi:
├── Ứng dụng đọc file upload từ EFS mount (Linux POSIX file system)
├── Cần file locking (khóa file — tránh nhiều process ghi cùng lúc)
└── Chia sẻ file giữa nhiều EC2 instances qua NFS

EFS path mapping:
├── User home: /efs/users/partner-acme/
└── Giống S3 nhưng là POSIX path thay vì S3 key

Khi nào EFS tốt hơn S3:
├── Application cần POSIX permissions (chmod, chown)
├── Cần read-after-write consistency ngay lập tức
└── Ứng dụng legacy không biết dùng S3 API
```

---

## 📂 Logical Directory Mapping — Ánh Xạ Thư Mục

### Restricted Home Directory (Giới Hạn Thư Mục)

```
Mỗi user chỉ nhìn thấy thư mục của mình, không thấy thư mục của user khác:

User partner-acme thấy:
/
├── uploads/      ← thực ra là s3://my-bucket/partners/acme/uploads/
└── reports/      ← thực ra là s3://my-bucket/partners/acme/reports/

User partner-xyz thấy:
/
├── uploads/      ← thực ra là s3://my-bucket/partners/xyz/uploads/
└── reports/      ← thực ra là s3://my-bucket/partners/xyz/reports/

Cả hai user dùng cùng S3 bucket nhưng không nhìn thấy dữ liệu của nhau.
```

### Logical Directory Mapping (Ánh Xạ Tùy Chỉnh)

```
Cấu trúc mapping:
[
  {
    "Entry": "/uploads",
    "Target": "/my-bucket/partners/acme/uploads"
  },
  {
    "Entry": "/shared",
    "Target": "/shared-bucket/public-reports"
  }
]

User có thể thấy /uploads và /shared
├── /uploads → S3 bucket riêng của partner acme
└── /shared → S3 bucket khác chứa báo cáo public

Lợi thế: Một user có thể truy cập nhiều S3 bucket hoặc EFS khác nhau
         thông qua cùng một Transfer Family server
```

---

## 🔒 Security và Compliance

### IAM Role cho User

```
Mỗi user được gán một IAM Role (Vai Trò IAM) xác định quyền S3:

Ví dụ IAM policy cho partner user:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-bucket/partners/acme/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-bucket",
      "Condition": {
        "StringLike": {"s3:prefix": ["partners/acme/*"]}
      }
    }
  ]
}
```

### Endpoint Types (Loại Endpoint)

```
Public endpoint (Endpoint Công Khai):
├── Hostname: s-xxxx.server.transfer.ap-southeast-1.amazonaws.com
├── Accessible từ internet (đối tác bên ngoài có thể kết nối)
└── Nên dùng Elastic IP để có IP cố định (đối tác whitelist IP)

VPC endpoint (Endpoint Trong VPC — Virtual Private Cloud):
├── Chỉ accessible từ trong VPC
├── Dùng cho FTP (không mã hóa — không an toàn khi ra internet)
└── Internal use case: ứng dụng trong VPC upload file

VPC endpoint với internet-facing:
├── Trong VPC nhưng có Elastic IP → accessible từ internet
└── Cho phép tích hợp với Security Groups và Network ACLs
```

### Compliance Features (Tính Năng Tuân Thủ)

```
CloudTrail logging:
└── Ghi lại mọi API call: CreateUser, DeleteUser, StartFileTransfer...

Transfer Family server logs:
└── Ghi vào CloudWatch Logs: kết nối, đăng nhập, file transfer

S3 server-side encryption:
└── File upload được mã hóa tự động (SSE-S3 hoặc SSE-KMS)

VPC Flow Logs:
└── Theo dõi network traffic vào/ra Transfer Family endpoint
```

---

## 🏭 AS2 — Trao Đổi File B2B

### AS2 Concepts (Khái Niệm AS2)

```
Trading partners (Đối Tác Thương Mại):
├── Mỗi bên có AS2 ID (định danh) và certificate (chứng chỉ X.509)
├── Cần trao đổi public certificates trước khi trading

Message flow (Luồng tin nhắn):
1. Sender chuẩn bị payload (ví dụ: EDI 850 Purchase Order)
2. Encrypt với public key của receiver → chỉ receiver đọc được
3. Sign với private key của sender → receiver verify danh tính sender
4. Gửi qua HTTPS POST đến AS2 URL của receiver
5. Receiver decrypt, verify signature
6. Receiver gửi MDN (Message Disposition Notification):
   ├── Synchronous MDN: trả lời ngay trong cùng HTTPS connection
   └── Asynchronous MDN: gửi HTTP POST đến URL của sender sau đó
```

### Transfer Family AS2 Setup

```
Cấu hình bên AWS (receiver):
├── Tạo Transfer Family server với protocol AS2
├── Tạo Profile cho công ty mình:
│   ├── AS2 ID: MYCOMPANY
│   └── Certificate: upload hoặc dùng ACM cert
├── Tạo Partner Profile cho đối tác:
│   ├── Partner AS2 ID: PARTNERXYZ
│   └── Partner public certificate
└── Tạo Agreement (thỏa thuận) kết nối hai profile:
    ├── Base directory trên S3 nơi lưu file nhận
    └── MDN response mode (sync/async)

Đối tác cấu hình:
├── URL: https://s-xxxx.server.transfer.region.amazonaws.com
├── AS2 ID của bạn: MYCOMPANY
└── Certificate của bạn (public key)
```

---

## ⚙️ Workflow Automation — Tự Động Hóa Sau Upload

### Managed Workflows (Luồng Công Việc Có Quản Lý)

```
Transfer Family Managed Workflows cho phép xử lý file ngay sau upload:

Các bước workflow có thể cấu hình:
├── Copy: sao chép file đến S3 location khác
├── Tag: gắn tag S3 (nhãn) cho file (ví dụ: source=sftp-upload)
├── Delete: xóa file gốc sau khi xử lý
├── Decrypt (PGP): giải mã file mã hóa PGP
└── Custom Lambda step: gọi Lambda function tùy chỉnh

Ví dụ workflow thực tế:
Upload file order.csv.pgp (file đơn hàng mã hóa PGP)
→ Step 1: Decrypt PGP → order.csv
→ Step 2: Tag S3 object: {status: "received", source: "partner-acme"}
→ Step 3: Copy đến s3://processed-bucket/orders/
→ Step 4: Lambda trigger: gọi API để xử lý đơn hàng
→ Step 5: Delete file gốc tại home directory
```

### EventBridge / S3 Event Notifications

```
Ngoài Managed Workflows, có thể dùng:

S3 Event Notifications:
└── S3 gửi event khi có object mới → trigger Lambda, SQS, SNS
    Ví dụ: file mới trong s3://my-bucket/uploads/
    → Lambda function phân tích nội dung file → lưu vào database

EventBridge (Bus Sự Kiện):
└── Transfer Family gửi events về:
    ├── User đăng nhập thành công/thất bại
    ├── File upload/download bắt đầu và kết thúc
    └── Server created/deleted

Dùng EventBridge để:
├── Alert (cảnh báo) khi đăng nhập thất bại nhiều lần
└── Monitor SLA: cảnh báo nếu đối tác chưa gửi file đúng giờ
```

---

## 📊 Monitoring và Logging

### CloudWatch Metrics

```
Metrics Transfer Family:
├── BytesIn: Tổng bytes upload (nhận vào)
├── BytesOut: Tổng bytes download (gửi ra)
├── FilesIn: Số file được upload
├── FilesOut: Số file được download
└── (Theo tag user, protocol, server)
```

### CloudWatch Logs

```
Server logs (ghi vào CloudWatch Log Group):
├── Login success/failure (đăng nhập thành công/thất bại)
├── File upload/download (tên file, kích thước, thời gian)
└── Disconnections (ngắt kết nối)

Log format ví dụ:
[2026-06-03 10:15:32] SFTP user=partner-acme action=UPLOAD
file=/uploads/orders-20260603.csv size=125432 bytes duration=2.3s status=SUCCESS
```

---

## ⚖️ So Sánh vs Tự Dựng SFTP Server

| Tiêu Chí | Transfer Family | Tự dựng SFTP trên EC2 |
| -------- | --------------- | ---------------------- |
| **Setup time** | < 30 phút | Nhiều giờ (cài, cấu hình) |
| **Maintenance** | AWS lo (OS patches, updates) | Tự vá bảo mật định kỳ |
| **High Availability** | ✅ Tích hợp sẵn | Phải tự dựng thêm (Multi-AZ, ELB) |
| **Scaling** | ✅ Auto-scale | Thủ công thêm server |
| **Storage** | S3 / EFS (không giới hạn) | EBS volume (có giới hạn) |
| **User management** | Console / API / AD | Config files, adduser |
| **Monitoring** | CloudWatch tích hợp | Cài thêm Prometheus/Grafana |
| **Chi phí ít user** | Cao hơn (per-hour endpoint) | Thấp hơn (chỉ EC2) |
| **Chi phí nhiều user** | Tương đương hoặc thấp hơn | Cao hơn (nhiều server) |
| **Compliance logging** | ✅ CloudTrail, CloudWatch | Tự cấu hình auditd/syslog |

> **Kết luận:** Transfer Family tốt hơn trong production vì giảm operational overhead. Chỉ tự dựng SFTP khi cần control hoàn toàn hoặc cost rất nhạy cảm với ít user.

---

## 🎓 Câu Hỏi Phỏng Vấn

1. **Transfer Family khác gì so với tự dựng SFTP server trên EC2?**
   - Transfer Family là fully managed: AWS lo infrastructure, high availability, scaling. Không cần patch server, quản lý certificates. File lưu thẳng vào S3/EFS. Tự dựng EC2 linh hoạt hơn nhưng tốn công bảo trì.

2. **Transfer Family hỗ trợ những giao thức nào? Khi nào dùng AS2?**
   - SFTP, FTPS, FTP, AS2. AS2 dùng cho B2B commerce cần non-repudiation (chữ ký số + MDN), phổ biến trong EDI với các ngành retail, healthcare, logistics.

3. **Identity provider của Transfer Family có bao nhiêu loại?**
   - Bốn loại: Service-managed (user lưu trong Transfer Family), Secrets Manager, AWS Directory Service/Active Directory, Custom Lambda (linh hoạt nhất).

4. **File upload qua Transfer Family SFTP được lưu ở đâu?**
   - Lưu thẳng vào S3 bucket hoặc EFS file system được cấu hình làm backend. User SFTP không biết đang dùng S3 — họ chỉ thấy POSIX-like file system.

5. **Managed Workflows trong Transfer Family là gì? Cho ví dụ?**
   - Cho phép tự động xử lý file ngay sau upload: decrypt PGP, tag S3, copy đến location khác, gọi Lambda. Ví dụ: đối tác gửi file mã hóa PGP → workflow tự động decrypt → trigger Lambda xử lý đơn hàng.

6. **FTP endpoint có thể expose ra internet không? Tại sao?**
   - Không nên. FTP không mã hóa credentials và dữ liệu. Transfer Family FTP endpoint chỉ nên là VPC-only (nội bộ VPC). Để ra internet phải dùng SFTP hoặc FTPS.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
