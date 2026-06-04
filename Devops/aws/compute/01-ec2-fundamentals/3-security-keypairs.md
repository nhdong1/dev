# Security Groups, Key Pairs & IAM Instance Profiles

> Bộ ba bảo mật cốt lõi của EC2: **Security Groups** (tường lửa ảo), **Key Pairs** (xác thực SSH/RDP), và **IAM Instance Profiles** (quyền truy cập AWS services).

## 📚 Mục Lục

1. [Security Groups — Nhóm Bảo Mật](#security-groups--nhóm-bảo-mật)
2. [NACL — Network Access Control List — Danh Sách Kiểm Soát Truy Cập Mạng](#nacl--network-access-control-list)
3. [Security Group vs NACL](#security-group-vs-nacl)
4. [Key Pairs — Cặp Khóa SSH](#key-pairs--cặp-khóa-ssh)
5. [IAM Roles & Instance Profiles](#iam-roles--instance-profiles)
6. [Best Practices Bảo Mật EC2](#best-practices-bảo-mật-ec2)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Security Groups — Nhóm Bảo Mật

### Security Group Là Gì?

**Security Group** là tường lửa ảo (virtual firewall) cấp instance, kiểm soát traffic vào (inbound) và ra (outbound) của EC2 instance.

```
Internet
   │
   ▼
[Security Group — lớp lọc traffic]
   │                    │
   ├── Inbound Rules    ├── Outbound Rules
   │   (traffic vào)    │   (traffic ra)
   │                    │
   ▼                    ▼
[EC2 Instance]
```

### Đặc Điểm Quan Trọng

```
✅ Stateful    — Response traffic tự động được phép
                 (không cần outbound rule cho response)

✅ Allow-only  — Chỉ có Allow rules, không có Deny rules
                 (muốn block → dùng NACL)

✅ Multiple    — Một instance có thể có nhiều Security Groups
                 (rules được union lại)

✅ Dynamic     — Thay đổi rule áp dụng ngay lập tức (không restart)

✅ VPC-bound   — Security Group gắn với VPC, không thể dùng cross-VPC
```

### Cấu Trúc Rule

```
Inbound Rule:
  Type     | Protocol | Port Range | Source
  ---------|----------|------------|------------------
  SSH      | TCP      | 22         | 0.0.0.0/0 (không nên!)
  HTTP     | TCP      | 80         | 0.0.0.0/0
  HTTPS    | TCP      | 443        | 0.0.0.0/0
  Custom   | TCP      | 8080       | sg-0abc123 (SG khác)
  MySQL    | TCP      | 3306       | 10.0.0.0/8 (VPC CIDR)

Outbound Rule:
  Type     | Protocol | Port Range | Destination
  ---------|----------|------------|------------------
  All      | All      | All        | 0.0.0.0/0 (mặc định)
```

### Stateful — Tính Có Trạng Thái

```
Request (client → server):
  Client:53280 → Server:443
  [Security Group kiểm tra inbound rule 443 → ALLOW]

Response (server → client):
  Server:443 → Client:53280
  [Security Group TỰ ĐỘNG ALLOW — stateful!]
  (Không cần outbound rule cho port 53280)
```

### Security Group Referencing — Tham Chiếu Giữa SGs

Thay vì dùng IP ranges, có thể dùng Security Group ID làm source/destination:

```
Ví dụ: Web tier → App tier → DB tier

[WebSG]  → inbound rule của [AppSG] cho phép traffic từ [WebSG]
[AppSG]  → inbound rule của [DBSG]  cho phép traffic từ [AppSG]

Lợi ích:
  - Tự động update khi instance IP thay đổi
  - Không cần maintain IP ranges thủ công
  - Bảo mật hơn (chỉ source từ đúng SG mới vào được)
```

```bash
# Tạo Security Group
aws ec2 create-security-group \
  --group-name "app-server-sg" \
  --description "Security group for application servers" \
  --vpc-id vpc-0abc123

# Thêm inbound rule từ Security Group khác
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 8080 \
  --source-group sg-0def456  # traffic chỉ từ SG này

# Thêm inbound HTTPS từ internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Xem rules
aws ec2 describe-security-groups --group-ids sg-0abc123
```

### Default Security Group

Mỗi VPC có một **default Security Group**:
- Cho phép **tất cả inbound** từ cùng default SG
- Cho phép **tất cả outbound**
- Không nên dùng cho production (quá rộng)

---

## NACL — Network Access Control List

### NACL Là Gì?

**NACL — Network Access Control List — Danh Sách Kiểm Soát Truy Cập Mạng** là tường lửa ở cấp subnet (không phải instance).

```
Internet
   │
   ▼
[NACL — cấp Subnet]
   │
   ▼
[Security Group — cấp Instance]
   │
   ▼
[EC2 Instance]
```

### Đặc Điểm NACL

```
✅ Stateless  — Phải định nghĩa cả inbound VÀ outbound rules
✅ Allow/Deny — Có cả Allow và Deny rules (khác SG)
✅ Numbered   — Rules được đánh số, thực thi theo thứ tự từ thấp đến cao
✅ Subnet     — Áp dụng cho toàn bộ subnet
✅ Default    — Default NACL cho phép all traffic in/out
```

### NACL Rules Example

```
Inbound Rules:
  Rule # | Type  | Protocol | Port  | Source      | Allow/Deny
  -------|-------|----------|-------|-------------|----------
  100    | HTTPS | TCP      | 443   | 0.0.0.0/0   | ALLOW
  110    | HTTP  | TCP      | 80    | 0.0.0.0/0   | ALLOW
  120    | SSH   | TCP      | 22    | 10.0.0.0/8  | ALLOW
  200    | All   | All      | All   | 1.2.3.4/32  | DENY  ← Block IP
  *      | All   | All      | All   | 0.0.0.0/0   | DENY  ← Default deny

Outbound Rules (stateless — phải explicit):
  Rule # | Type  | Protocol | Port        | Dest        | Allow/Deny
  -------|-------|----------|-------------|-------------|----------
  100    | All   | All      | All         | 0.0.0.0/0   | ALLOW
  *      | All   | All      | All         | 0.0.0.0/0   | DENY
```

> **Ephemeral Ports (Cổng Tạm Thời):** Khi client kết nối đến server, server reply về ephemeral port (1024-65535) của client. Vì NACL stateless, phải có outbound rule cho range 1024-65535 để cho phép response.

---

## Security Group vs NACL

| Tiêu Chí           | Security Group              | NACL                         |
| ------------------ | --------------------------- | ---------------------------- |
| Cấp độ            | Instance                    | Subnet                       |
| Stateful/Stateless | Stateful                    | Stateless                    |
| Rules              | Allow only                  | Allow và Deny                |
| Thứ tự rules       | Tất cả rules được xét       | Theo số thứ tự               |
| Mặc định           | Deny all inbound, Allow all outbound | Allow all (default NACL) |
| Dùng khi           | Bảo vệ từng instance        | Block IP/range ở subnet level|

### Khi Nào Dùng NACL?

NACL hữu ích khi cần **block toàn bộ subnet** hoặc **block IP cụ thể** mà không thể làm bằng Security Group (SG không có Deny rule):

```
Use case điển hình:
  1. Block IP tấn công DDoS nhanh chóng (add deny rule vào NACL)
  2. Thêm lớp bảo vệ trước Security Group (defense in depth)
  3. Compliance requirement cần network-level access control
```

---

## Key Pairs — Cặp Khóa SSH

### Key Pair Là Gì?

**Key Pair** là cặp khóa mật mã asymmetric (public/private key) dùng để xác thực SSH (Linux) hoặc decrypt password (Windows RDP).

```
AWS lưu:    Public Key (trong authorized_keys của instance)
Bạn giữ:   Private Key (.pem file) — TUYỆT ĐỐI BÍ MẬT
```

### Tạo và Quản Lý Key Pair

```bash
# Tạo key pair mới (AWS tạo, bạn download private key)
aws ec2 create-key-pair \
  --key-name "production-key" \
  --key-type rsa \
  --key-format pem \
  --query 'KeyMaterial' \
  --output text > production-key.pem

chmod 400 production-key.pem  # QUAN TRỌNG: chỉ owner read

# Hoặc import public key tự tạo (khuyến nghị cho production)
ssh-keygen -t ed25519 -f ~/.ssh/aws-production -C "production key 2026"
aws ec2 import-key-pair \
  --key-name "production-key" \
  --public-key-material fileb://~/.ssh/aws-production.pub

# Liệt kê key pairs
aws ec2 describe-key-pairs

# Xóa key pair (chỉ xóa khỏi AWS, không xóa private key đã download)
aws ec2 delete-key-pair --key-name "old-key"
```

### SSH vào EC2

```bash
# Linux/macOS
ssh -i production-key.pem ec2-user@<public-ip>
# (Amazon Linux 2: ec2-user, Ubuntu: ubuntu, RHEL: ec2-user hoặc root)

# Troubleshooting SSH fail:
# 1. Kiểm tra Security Group: inbound TCP 22 từ IP của bạn
# 2. Kiểm tra file permission: chmod 400 key.pem
# 3. Kiểm tra đúng username (ec2-user cho AL2, ubuntu cho Ubuntu)
# 4. Kiểm tra instance đang running và status check passed
# 5. Kiểm tra public IP/DNS đúng không

# Windows RDP
aws ec2 get-password-data \
  --instance-id i-0abc123 \
  --priv-launch-key production-key.pem
# Sau đó dùng Remote Desktop Connection với password vừa lấy
```

### Key Pair Best Practices

```
1. ĐỪNG dùng chung một key pair cho nhiều môi trường
   ✅ production-key, staging-key, dev-key (riêng biệt)

2. Dùng ed25519 thay vì RSA (bảo mật hơn, key ngắn hơn)

3. Import public key thay vì để AWS tạo (bạn kiểm soát private key)

4. Backup private key an toàn (password manager, HSM)

5. Xóa instance access key thường xuyên, tạo key mới định kỳ

6. Ưu tiên dùng SSM Session Manager thay vì SSH trực tiếp
   (không cần key pair, không cần mở port 22)
```

### AWS Systems Manager Session Manager — Thay Thế SSH

**SSM Session Manager** (Trình Quản Lý Phiên SSM) cho phép kết nối vào EC2 mà **không cần port 22 và key pair**:

```bash
# Kết nối qua SSM (cần SSM Agent + IAM role đúng)
aws ssm start-session --target i-0abc123

# Port forwarding qua SSM (không cần SSH tunnel)
aws ssm start-session \
  --target i-0abc123 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3306"],"localPortNumber":["13306"]}'
```

---

## IAM Roles & Instance Profiles

### Tại Sao Cần IAM Role Cho EC2?

Nếu ứng dụng trên EC2 cần gọi AWS API (S3, DynamoDB, SQS...), nó cần credentials. Có 2 cách:

```
❌ SAI: Hard-code AWS Access Key + Secret Key vào code/config
  - Key bị lộ khi code leak lên GitHub
  - Không rotate được dễ dàng
  - Không audit được ai dùng key đó

✅ ĐÚNG: Dùng IAM Role gắn vào EC2 via Instance Profile
  - Credentials tự động rotate mỗi vài giờ
  - Không cần hard-code bất cứ thứ gì
  - Audit qua CloudTrail
  - Least privilege — chỉ cấp quyền cần thiết
```

### Instance Profile Là Gì?

```
IAM Role      → Định nghĩa permissions (S3:GetObject, DynamoDB:Query...)
Instance Profile → Container chứa IAM Role, gắn vào EC2

Một Instance Profile chứa đúng một IAM Role.
Một EC2 instance chỉ có thể gắn một Instance Profile.
```

```
EC2 Instance
  │
  └── Instance Profile (aws:ec2:instanceprofile/MyAppProfile)
        │
        └── IAM Role (arn:aws:iam::123456789:role/MyAppRole)
              │
              ├── Inline Policy: {"s3:GetObject" on "my-bucket/*"}
              └── Managed Policy: "AmazonDynamoDBReadOnlyAccess"
```

### Tạo và Gắn IAM Role

```bash
# 1. Tạo trust policy (cho phép EC2 assume role)
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# 2. Tạo IAM Role
aws iam create-role \
  --role-name "MyAppEC2Role" \
  --assume-role-policy-document file://trust-policy.json

# 3. Gắn permissions vào role
aws iam attach-role-policy \
  --role-name "MyAppEC2Role" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# 4. Tạo Instance Profile
aws iam create-instance-profile \
  --instance-profile-name "MyAppInstanceProfile"

# 5. Gắn role vào instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name "MyAppInstanceProfile" \
  --role-name "MyAppEC2Role"

# 6. Gắn instance profile vào instance đang chạy
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc123 \
  --iam-instance-profile Name="MyAppInstanceProfile"
```

### Credentials Rotation Tự Động

AWS SDK tự động lấy credentials từ **IMDS — Instance Metadata Service** (dịch vụ siêu dữ liệu):

```bash
# Xem credentials hiện tại của instance (từ bên trong instance)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# → MyAppEC2Role

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyAppEC2Role
# → {"AccessKeyId": "...", "SecretAccessKey": "...", "Token": "...", "Expiration": "..."}
# Credentials này tự động rotate, ứng dụng không cần làm gì
```

### Nguyên Tắc Least Privilege — Quyền Tối Thiểu

```
Sai: Gắn AdministratorAccess vào EC2
✅  Đúng: Chỉ cấp đúng quyền cần thiết

Ví dụ app cần đọc S3 và ghi DynamoDB:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-specific-bucket",
        "arn:aws:s3:::my-specific-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem", "dynamodb:UpdateItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/my-table"
    }
  ]
}
```

---

## Best Practices Bảo Mật EC2

### Security Group Hardening

```
1. Không bao giờ mở port 22 (SSH) hoặc 3389 (RDP) ra 0.0.0.0/0
   → Dùng VPN, Bastion Host, hoặc SSM Session Manager

2. Rule theo nguyên tắc minimum necessary:
   - Web server: 80, 443 từ 0.0.0.0/0
   - App server: Chỉ từ Web SG
   - DB: Chỉ từ App SG

3. Thường xuyên review và xóa unused Security Groups

4. Dùng AWS Config rule để detect overly permissive SGs:
   "restricted-ssh", "restricted-common-ports"

5. Log flow với VPC Flow Logs để phát hiện bất thường
```

### IAM Hardening

```
1. Không bao giờ để Access Keys trên EC2 (file ~/.aws/credentials)
   → Luôn dùng IAM Role + Instance Profile

2. Review role permissions định kỳ với IAM Access Analyzer

3. Dùng AWS Managed Policies khi phù hợp, không tự viết
   nếu AWS đã có policy đúng yêu cầu

4. Enable MFA (Multi-Factor Authentication) cho console access

5. Rotate Key Pairs định kỳ hoặc khi nhân viên nghỉ việc
```

### Bastion Host Pattern — Máy Chủ Nhảy

```
Internet → Bastion Host (Public Subnet) → Private EC2 Instances
           (chỉ Bastion có port 22 mở từ IP trust)

Bastion Host thay thế hiện đại:
  → SSM Session Manager (không cần Bastion, không cần port 22)
  → EC2 Instance Connect (web-based SSH, tạm thời)
```

---

## Câu Hỏi Phỏng Vấn

**Q: Security Group stateful nghĩa là gì? Cho ví dụ thực tế.**
> Stateful có nghĩa là khi một request được cho phép vào (inbound), response của nó tự động được phép ra (outbound) mà không cần outbound rule. Ví dụ: Client kết nối từ port 54321 đến server port 443. Security Group chỉ cần inbound rule cho port 443 — response về port 54321 tự động được phép. Khác với NACL (stateless): phải có cả inbound 443 và outbound ephemeral ports 1024-65535.

**Q: Tại sao không nên để Access Key AWS trực tiếp trong EC2?**
> Khi key bị lộ (GitHub push nhầm, log file, vulnerability), attacker có toàn quyền truy cập với permissions của key đó. Key cứng trong code rất khó rotate. IAM Role + Instance Profile an toàn hơn vì: credentials tự rotate mỗi giờ, không lộ static key trong code, có thể revoke ngay bằng cách xóa role, audit được qua CloudTrail.

**Q: Làm sao xâm nhập EC2 mà không cần SSH key?**
> Có nhiều cách hợp lệ: (1) **SSM Session Manager** — không cần port 22, không cần key pair, chỉ cần IAM permissions và SSM Agent chạy. (2) **EC2 Instance Connect** — inject tạm thời public key qua AWS Console. (3) **AWS Console Serial Console** — truy cập qua serial port. Trong production, SSM Session Manager là best practice.

**Q: Một EC2 instance có thể gắn bao nhiêu Security Groups?**
> Mỗi instance có thể gắn tối đa 5 Security Groups mặc định (có thể tăng lên 16 theo request). Rules từ tất cả SGs được union lại — nếu bất kỳ SG nào allow traffic, traffic được phép. Không có precedence, không có Deny trong SG.

---

## Liên Kết

- [← AMI & Storage](./2-ami-storage.md)
- [Tiếp theo: User Data & Metadata →](./4-user-data-metadata.md)
- [AWS Security Groups Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

---

**Cập Nhật:** 2026-05-14 | **Trạng Thái:** ✅ Hoàn thành
