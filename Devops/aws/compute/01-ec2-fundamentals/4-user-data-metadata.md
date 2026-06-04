# User Data & Instance Metadata — Dữ Liệu Khởi Tạo và Siêu Dữ Liệu

> **User Data** cho phép tự động hóa cấu hình instance khi khởi động lần đầu. **IMDS — Instance Metadata Service — Dịch Vụ Siêu Dữ Liệu** cung cấp thông tin về instance cho ứng dụng đang chạy.

## 📚 Mục Lục

1. [User Data — Dữ Liệu Khởi Tạo](#user-data--dữ-liệu-khởi-tạo)
2. [Cloud-Init — Công Cụ Bootstrap](#cloud-init--công-cụ-bootstrap)
3. [IMDS — Instance Metadata Service](#imds--instance-metadata-service)
4. [IMDSv1 vs IMDSv2](#imdsv1-vs-imdsv2)
5. [Các Metadata Quan Trọng](#các-metadata-quan-trọng)
6. [Instance User Data Patterns](#instance-user-data-patterns)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## User Data — Dữ Liệu Khởi Tạo

### User Data Là Gì?

**User Data** là script (shell script hoặc cloud-init directives) chạy **một lần** khi instance khởi động lần đầu tiên (first boot). Đây là cơ chế bootstrap chính của EC2.

```
Instance Launch
      │
      ▼
   [pending]
      │
      ▼
   [running] ──► cloud-init chạy User Data script
                     │
                     ├── Cài packages
                     ├── Cấu hình services
                     ├── Deploy ứng dụng
                     └── Báo hiệu hoàn thành (cfn-signal / SSM)
```

### Đặc Điểm User Data

```
✅ Chạy một lần (khi first boot) — mặc định
✅ Chạy với quyền root (không cần sudo)
✅ Tối đa 16KB (base64-encoded, ~12KB plain text)
✅ Có thể là shell script (#!/bin/bash) hoặc cloud-init YAML
✅ Log output: /var/log/cloud-init-output.log
⚠️ Timeout không cố định — nếu script lỗi, instance vẫn tiếp tục boot
⚠️ Không nên để secret/password trong User Data (có thể lấy qua IMDS)
```

### User Data Shell Script Cơ Bản

```bash
#!/bin/bash
# User Data script cho Amazon Linux 2023

# Update hệ thống
yum update -y

# Cài đặt packages cần thiết
yum install -y nginx java-21-amazon-corretto awscli

# Cấu hình Nginx
cat > /etc/nginx/conf.d/app.conf << 'NGINX_EOF'
server {
    listen 80;
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
NGINX_EOF

# Bật và start service
systemctl enable nginx
systemctl start nginx

# Lấy config từ SSM Parameter Store (không hard-code credentials)
DB_URL=$(aws ssm get-parameter \
  --name "/myapp/production/db-url" \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text \
  --region us-east-1)

# Export environment variable
echo "DB_URL=${DB_URL}" >> /etc/environment

# Download và chạy ứng dụng từ S3
aws s3 cp s3://my-app-artifacts/app-latest.jar /opt/app/app.jar
systemctl enable myapp
systemctl start myapp

# Báo hiệu CloudFormation stack đã sẵn sàng (nếu dùng CloudFormation)
# /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource MyASG --region us-east-1
```

### User Data cho Ubuntu

```bash
#!/bin/bash
# Ubuntu/Debian
apt-get update -y
apt-get install -y nginx python3 python3-pip

# Ubuntu dùng apt, không phải yum
pip3 install -r /opt/app/requirements.txt
```

### Xem và Cập Nhật User Data

```bash
# Xem User Data của instance (từ bên ngoài)
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 --decode

# Cập nhật User Data (instance phải stopped)
aws ec2 stop-instances --instance-ids i-0abc123
# Đợi instance stopped...

aws ec2 modify-instance-attribute \
  --instance-id i-0abc123 \
  --user-data file://new-userdata.sh

aws ec2 start-instances --instance-ids i-0abc123

# Xem log trong instance
cat /var/log/cloud-init-output.log
```

---

## Cloud-Init — Công Cụ Bootstrap

**Cloud-init** là phần mềm chuẩn để khởi tạo cloud instances, hỗ trợ nhiều định dạng User Data:

### Shell Script (#!/bin/bash)

Đơn giản nhất, dùng khi cần chạy commands.

### Cloud-Config YAML (#cloud-config)

Declarative format, nhiều tính năng hơn shell script:

```yaml
#cloud-config

# Cài packages
packages:
  - nginx
  - python3
  - git

# Tạo files
write_files:
  - path: /etc/nginx/conf.d/app.conf
    content: |
      server {
          listen 80;
          location / {
              proxy_pass http://localhost:8080;
          }
      }
    owner: root:root
    permissions: '0644'

# Chạy commands
runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - cd /opt/app && python3 app.py &

# Tạo users
users:
  - name: deploy
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - "ssh-ed25519 AAAA... deploy@company.com"

# Cài docker
# (dùng apt_sources hoặc yum_repos cho package repo)
```

### MIME Multi-Part (Kết Hợp Nhiều Script)

```
Content-Type: multipart/mixed; boundary="==BOUNDARY=="
MIME-Version: 1.0

--==BOUNDARY==
Content-Type: text/cloud-config; charset="us-ascii"

#cloud-config
packages:
  - nginx

--==BOUNDARY==
Content-Type: text/x-shellscript; charset="us-ascii"

#!/bin/bash
systemctl start nginx

--==BOUNDARY==--
```

---

## IMDS — Instance Metadata Service

### IMDS Là Gì?

**IMDS — Instance Metadata Service — Dịch Vụ Siêu Dữ Liệu** là HTTP endpoint cục bộ tại IP link-local `169.254.169.254` (chỉ truy cập được từ bên trong instance), cung cấp thông tin về instance.

```
Từ bên trong EC2 instance:
curl http://169.254.169.254/latest/meta-data/

Kết quả (danh sách categories):
  ami-id
  hostname
  instance-id
  instance-type
  local-ipv4
  public-ipv4
  placement/availability-zone
  iam/security-credentials/
  ...
```

### Metadata Categories Quan Trọng

```bash
# Instance ID
curl http://169.254.169.254/latest/meta-data/instance-id
# → i-0abc123def456789

# Instance Type
curl http://169.254.169.254/latest/meta-data/instance-type
# → m7g.xlarge

# AZ và Region
curl http://169.254.169.254/latest/meta-data/placement/availability-zone
# → us-east-1a
curl http://169.254.169.254/latest/meta-data/placement/region
# → us-east-1

# Private IP
curl http://169.254.169.254/latest/meta-data/local-ipv4
# → 10.0.1.45

# Public IP (trả về 404 nếu không có public IP)
curl http://169.254.169.254/latest/meta-data/public-ipv4
# → 52.86.100.200

# AMI ID
curl http://169.254.169.254/latest/meta-data/ami-id
# → ami-0abc123

# IAM credentials từ Instance Profile
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# → MyAppEC2Role
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyAppEC2Role
# → {"AccessKeyId": "...", "SecretAccessKey": "...", "Token": "...", "Expiration": "..."}
```

### Instance Identity Document — Tài Liệu Danh Tính

```bash
# Signed document chứa thông tin instance, dùng để verify instance identity
curl http://169.254.169.254/latest/dynamic/instance-identity/document
# → JSON với accountId, instanceId, region, imageId, instanceType...

# Dùng để verify instance thật sự là từ AWS account của bạn
# (ứng dụng 3rd party như Vault, Consul dùng để authen EC2)
```

---

## IMDSv1 vs IMDSv2

### IMDSv1 — Phiên Bản Cũ (Không Bảo Mật)

```bash
# IMDSv1: Gọi trực tiếp, không cần token
curl http://169.254.169.254/latest/meta-data/instance-id
```

**Vấn đề bảo mật IMDSv1:** Nếu ứng dụng có lỗi SSRF — Server-Side Request Forgery (Giả Mạo Request Phía Máy Chủ), attacker có thể dùng lỗ hổng đó để gọi IMDS và lấy IAM credentials:

```
Tình huống tấn công SSRF:
  Attacker gửi: GET /fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
  Ứng dụng lỗi → forward request đến IMDS
  IMDS trả về IAM credentials
  Attacker có toàn bộ credentials!
```

### IMDSv2 — Phiên Bản Mới (An Toàn Hơn)

**IMDSv2** yêu cầu session-oriented flow với token:

```bash
# Bước 1: Lấy token (TTL tối đa 21600 giây = 6 giờ)
TOKEN=$(curl -X PUT \
  "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Bước 2: Dùng token trong mọi request
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

**Tại sao IMDSv2 bảo vệ khỏi SSRF?**

```
SSRF attack với IMDSv2:
  1. Attacker gửi request SSRF để lấy token (PUT method)
     → Nhiều SSRF proxy không hỗ trợ PUT
     → PUT với custom header phức tạp hơn để exploit
  
  2. Ngay cả nếu lấy được token, token có TTL ngắn
  
  3. Header X-aws-ec2-metadata-token không thể set bởi browser
     (ngăn chặn một số attack vector)
```

### Bắt Buộc IMDSv2 (Best Practice)

```bash
# Disable IMDSv1, chỉ cho phép IMDSv2 cho instance đang chạy
aws ec2 modify-instance-metadata-options \
  --instance-id i-0abc123 \
  --http-tokens required \
  --http-endpoint enabled

# Đặt IMDSv2 required khi launch instance mới
aws ec2 run-instances \
  --image-id ami-0abc123 \
  --instance-type m7g.large \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  ...

# Đặt default cho toàn account (mới nhất)
aws ec2 modify-instance-metadata-defaults \
  --http-tokens required \
  --region us-east-1
```

### Kiểm Tra IMDSv2 Compliance

```bash
# Tìm instances đang dùng IMDSv1 (chưa require token)
aws ec2 describe-instances \
  --filters "Name=metadata-options.http-tokens,Values=optional" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# CloudWatch metric để monitor IMDS usage
# MetadataNoToken — số lần IMDS được gọi không có token (IMDSv1)
```

---

## Các Metadata Quan Trọng

### Metadata Categories Đầy Đủ

```
/latest/meta-data/
├── ami-id                          AMI được dùng để tạo instance
├── instance-id                     ID của instance
├── instance-type                   Loại instance (m7g.xlarge...)
├── hostname                        Private DNS hostname
├── local-hostname                  Private DNS (ip-10-0-1-45.ec2.internal)
├── local-ipv4                      Private IP
├── public-hostname                 Public DNS (nếu có)
├── public-ipv4                     Public IP (nếu có)
├── mac                             MAC address của ENI chính
├── network/
│   └── interfaces/macs/<mac>/
│       ├── vpc-id                  VPC ID
│       ├── subnet-id               Subnet ID
│       └── security-group-ids      Security group IDs
├── placement/
│   ├── availability-zone           us-east-1a
│   ├── region                      us-east-1
│   └── instance-life-cycle         on-demand | spot | scheduled
├── iam/
│   ├── info                        IAM info
│   └── security-credentials/
│       └── <role-name>             Temporary credentials
├── spot/
│   ├── termination-time            Khi instance sắp bị terminate (Spot)
│   └── instance-action             {"action":"terminate","time":"..."}
└── tags/instance/                  Tags của instance (IMDSv2 required)
    ├── Name
    ├── Environment
    └── ...
```

### Spot Instance Termination Notice — Thông Báo Xóa Spot

```bash
# Script chạy trên Spot Instance để detect termination notice
# AWS gửi notice 2 phút trước khi terminate

while true; do
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/spot/termination-time)
  
  if [[ "$HTTP_CODE" == "200" ]]; then
    TERMINATION_TIME=$(curl -s \
      -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/spot/termination-time)
    
    echo "SPOT TERMINATION NOTICE at $TERMINATION_TIME"
    # Graceful shutdown: drain connections, save state, notify ALB
    # Gửi cảnh báo vào SNS hoặc CloudWatch
    break
  fi
  
  sleep 5
done
```

### Instance Tags qua IMDS (IMDSv2 Only)

```bash
# Lấy tên tag Environment từ IMDS (không cần AWS CLI)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

ENVIRONMENT=$(curl -s \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/tags/instance/Environment)

echo "Running in: $ENVIRONMENT"
```

> **Lưu ý:** Phải bật tính năng "Allow tags in instance metadata" khi launch hoặc qua `modify-instance-metadata-options`.

---

## Instance User Data Patterns

### Pattern 1: Download Config từ S3

```bash
#!/bin/bash
REGION=$(curl -s -H "X-aws-ec2-metadata-token: $(curl -s -X PUT \
  'http://169.254.169.254/latest/api/token' \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')" \
  http://169.254.169.254/latest/meta-data/placement/region)

ENVIRONMENT=$(curl -s -H "X-aws-ec2-metadata-token: ..." \
  http://169.254.169.254/latest/meta-data/tags/instance/Environment)

# Download config theo environment
aws s3 cp \
  s3://my-configs/${ENVIRONMENT}/app.properties \
  /etc/myapp/app.properties \
  --region $REGION

systemctl start myapp
```

### Pattern 2: Register vào Service Discovery

```bash
#!/bin/bash
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Đăng ký vào Route 53 hoặc Consul
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"${INSTANCE_ID}.internal.myapp.com\",
        \"Type\": \"A\",
        \"TTL\": 60,
        \"ResourceRecords\": [{\"Value\": \"${PRIVATE_IP}\"}]
      }
    }]
  }"
```

### Pattern 3: Config từ SSM Parameter Store

```bash
#!/bin/bash
REGION="us-east-1"

# Lấy DB connection string (SecureString — encrypted)
DB_URL=$(aws ssm get-parameter \
  --name "/myapp/production/database-url" \
  --with-decryption \
  --region $REGION \
  --query 'Parameter.Value' \
  --output text)

# Ghi vào environment file
cat > /etc/myapp/environment << EOF
DATABASE_URL=${DB_URL}
APP_ENV=production
EOF

chmod 600 /etc/myapp/environment
```

---

## Câu Hỏi Phỏng Vấn

**Q: User Data có chạy mỗi lần restart instance không?**
> Không, mặc định User Data chỉ chạy một lần khi first boot. Nếu muốn chạy mỗi lần boot, phải cấu hình cloud-init hoặc thêm script vào `/var/lib/cloud/scripts/per-boot/`. Tuy nhiên, cách này ít dùng — thường dùng systemd service hoặc cron thay thế.

**Q: Giải thích tấn công SSRF lên IMDS và IMDSv2 ngăn chặn thế nào?**
> SSRF là lỗ hổng cho phép attacker khiến server gửi request đến URL tùy ý. Nếu ứng dụng có SSRF và chạy trên EC2 với IMDSv1, attacker có thể yêu cầu server fetch `http://169.254.169.254/...` để lấy IAM credentials. IMDSv2 ngăn chặn bằng cách yêu cầu PUT request với custom header để lấy session token trước — hầu hết SSRF proxy không hỗ trợ PUT method với custom headers. Ngoài ra, token có TTL giới hạn.

**Q: User Data có thể truy cập AWS services không? Làm thế nào?**
> Có, nhưng cần IAM Role gắn vào instance (Instance Profile). Khi User Data script chạy, AWS SDK và CLI tự động lấy credentials từ IMDS endpoint. Không cần và không nên hard-code AWS credentials trong User Data — vì User Data có thể đọc được bởi bất kỳ user nào trong instance.

**Q: Tại sao IP 169.254.169.254 là link-local?**
> Link-local address (169.254.0.0/16) là IP chỉ có trong local network segment, không route được qua internet hoặc router. AWS dùng IP này cho IMDS vì chỉ instance trong host đó mới truy cập được — không thể gọi từ ngoài internet. Mỗi EC2 instance đều có route này tự động trong routing table của OS.

---

## Liên Kết

- [← Security Groups & Key Pairs](./3-security-keypairs.md)
- [Tiếp theo: Placement Groups →](./5-placement-groups.md)
- [AWS User Data Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [IMDS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)

---

**Cập Nhật:** 2026-05-14 | **Trạng Thái:** ✅ Hoàn thành
