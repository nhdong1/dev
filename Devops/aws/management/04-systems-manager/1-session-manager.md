# SSM Session Manager — Truy Cập Instance Không Cần SSH

> **Session Manager** là tính năng của AWS Systems Manager cho phép mở phiên làm việc tương tác (interactive session) vào EC2 instances và on-premises servers qua trình duyệt web hoặc AWS CLI — không cần port 22, không cần key pair, không cần bastion host.

---

## 📚 Mục Lục

1. [Vì Sao Session Manager?](#vì-sao-session-manager)
2. [Kiến Trúc Hoạt Động](#kiến-trúc-hoạt-động)
3. [Thiết Lập Session Manager](#thiết-lập-session-manager)
4. [Session Logging & Audit](#session-logging--audit)
5. [Port Forwarding](#port-forwarding)
6. [Kiểm Soát Truy Cập Với IAM](#kiểm-soát-truy-cập-với-iam)
7. [VPC Endpoints Cho Private Subnet](#vpc-endpoints-cho-private-subnet)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vì Sao Session Manager?

### Hạn Chế Của SSH Truyền Thống

```
Mô hình SSH cũ:
┌─────────────────────────────────────────────────────────┐
│ Internet ──SSH (Port 22)──▶ Bastion Host ──SSH──▶ EC2  │
│                                                         │
│ Rủi ro bảo mật:                                         │
│  • Security Group phải mở port 22 ra internet           │
│  • SSH key pair: nếu bị đánh cắp → toàn bộ fleet lộ    │
│  • Bastion Host = Single point of failure & attack      │
│  • Không có audit log đầy đủ theo chuẩn kiểm toán       │
│  • Quản lý key rotation thủ công tốn kém                │
└─────────────────────────────────────────────────────────┘
```

### Mô Hình Session Manager

```
Mô hình Session Manager:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  User ──HTTPS──▶ AWS Console / CLI                      │
│                       │                                 │
│                       ▼                                 │
│              SSM Service Endpoint                       │
│                       │  (WebSocket qua HTTPS)          │
│                       ▼                                 │
│              SSM Agent trên EC2                         │
│                                                         │
│ Bảo mật:                                                │
│  ✅ Không mở port 22   ✅ Không cần key pair             │
│  ✅ IAM kiểm soát      ✅ Audit log tự động              │
│  ✅ Mã hóa in-transit  ✅ Hỗ trợ private subnet          │
└─────────────────────────────────────────────────────────┘
```

---

## Kiến Trúc Hoạt Động

### Luồng Kết Nối Chi Tiết

```
Người Dùng                 AWS Control Plane            EC2 Instance
     │                            │                          │
     │──1. Gọi StartSession API──▶│                          │
     │                            │──2. Tạo WebSocket────────▶│
     │                            │      channel              │
     │◀─3. Trả về session URL─────│                          │
     │                            │                          │
     │──4. Kết nối WebSocket─────────────────────────────────▶│
     │                            │                          │
     │◀──5. Tương tác terminal────────────────────────────────│
     │                            │                          │
     │──6. Đóng session──────────────────────────────────────▶│
     │                            │──7. Ghi audit log────────▶S3/CW
```

### Các SSM Endpoints Cần Có

| Endpoint | Mục Đích |
|----------|---------|
| `ssm.REGION.amazonaws.com` | SSM API calls |
| `ssmmessages.REGION.amazonaws.com` | Session WebSocket messages |
| `ec2messages.REGION.amazonaws.com` | EC2 message delivery |

---

## Thiết Lập Session Manager

### Bước 1: IAM Instance Profile

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMCorePermissions",
      "Effect": "Allow",
      "Action": [
        "ssm:UpdateInstanceInformation",
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel",
        "ec2messages:AcknowledgeMessage",
        "ec2messages:DeleteMessage",
        "ec2messages:FailMessage",
        "ec2messages:GetEndpoint",
        "ec2messages:GetMessages",
        "ec2messages:SendReply"
      ],
      "Resource": "*"
    }
  ]
}
```

> Cách đơn giản hơn: Gắn AWS Managed Policy `AmazonSSMManagedInstanceCore` vào Instance Profile.

### Bước 2: Gắn IAM Role Cho EC2

```bash
# Tạo instance profile
aws iam create-instance-profile \
  --instance-profile-name SSMInstanceProfile

# Gắn role vào profile
aws iam add-role-to-instance-profile \
  --instance-profile-name SSMInstanceProfile \
  --role-name EC2SSMRole

# Gắn profile vào EC2 instance đang chạy
aws ec2 associate-iam-instance-profile \
  --instance-id i-1234567890abcdef0 \
  --iam-instance-profile Name=SSMInstanceProfile
```

### Bước 3: Xác Nhận Instance Đã Đăng Ký SSM

```bash
# Kiểm tra danh sách managed instances
aws ssm describe-instance-information \
  --filters "Key=PingStatus,Values=Online" \
  --query 'InstanceInformationList[*].[InstanceId,PingStatus,PlatformType]' \
  --output table
```

### Bước 4: Bắt Đầu Session Qua CLI

```bash
# Cài AWS Session Manager Plugin trước
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

# Mở session terminal
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --region ap-southeast-1
```

### Bước 5: Bắt Đầu Session Qua Console

```
AWS Console → Systems Manager → Session Manager → Start Session
→ Chọn Instance → Start Session
```

---

## Session Logging & Audit

### Tại Sao Phải Bật Session Logging?

Session logging đảm bảo **traceability** (khả năng truy vết) — biết chính xác ai làm gì trong session nào. Đây là yêu cầu bắt buộc trong các môi trường tuân thủ PCI-DSS, HIPAA, SOC2.

### Cấu Hình Session Logging

```json
// Session Manager Preferences (cấu hình trong SSM Console)
{
  "schemaVersion": "1.0",
  "description": "Session Manager Preferences",
  "sessionType": "Standard_Stream",
  "inputs": {
    "s3BucketName": "my-session-logs-bucket",
    "s3KeyPrefix": "session-logs/",
    "s3EncryptionEnabled": true,
    "cloudWatchLogGroupName": "/aws/ssm/sessions",
    "cloudWatchEncryptionEnabled": true,
    "cloudWatchStreamingEnabled": true,
    "kmsKeyId": "arn:aws:kms:ap-southeast-1:123456789:key/abcd1234",
    "runAsEnabled": false,
    "runAsDefaultUser": ""
  }
}
```

### Ví Dụ S3 Bucket Policy Cho Session Logs

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMSessionManagerLogs",
      "Effect": "Allow",
      "Principal": {
        "Service": "ssm.amazonaws.com"
      },
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::my-session-logs-bucket/session-logs/*"
    }
  ]
}
```

### Nội Dung Log Session

```
# Ví dụ log session trong S3
Session ID: user-abc123-0abc123def456789
Start Date: 2026-05-17T10:30:00Z
End Date: 2026-05-17T10:45:23Z
Owner: arn:aws:iam::123456789:user/devops-engineer
Target: i-1234567890abcdef0
Region: ap-southeast-1

# Command history được record đầy đủ:
$ sudo systemctl status nginx
$ cat /var/log/nginx/error.log
$ sudo tail -f /var/log/app/application.log
```

### Tra Cứu Session History

```bash
# Xem danh sách sessions gần đây
aws ssm describe-sessions \
  --state History \
  --filters "key=Target,value=i-1234567890abcdef0" \
  --query 'Sessions[*].[SessionId,StartDate,Status,Target,Owner]' \
  --output table
```

---

## Port Forwarding

### Port Forwarding Là Gì?

**Port Forwarding** (Chuyển Tiếp Cổng) cho phép tạo đường hầm (tunnel) bảo mật từ máy local đến tài nguyên trong VPC private — không cần expose port ra internet.

### Trường Hợp Dùng Phổ Biến

```
Kịch bản 1: Truy cập RDS trong private subnet
┌──────────────────────────────────────────────────────────┐
│  Local Machine :5432 ──tunnel──▶ EC2 ──▶ RDS :5432      │
│                                                          │
│  aws ssm start-session \                                 │
│    --target i-1234567890abcdef0 \                        │
│    --document-name AWS-StartPortForwardingSession \      │
│    --parameters '{"portNumber":["5432"],                 │
│                   "localPortNumber":["5432"]}'           │
└──────────────────────────────────────────────────────────┘

Kịch bản 2: Truy cập web app nội bộ
┌──────────────────────────────────────────────────────────┐
│  Local Browser :8080 ──tunnel──▶ EC2 App :80             │
│                                                          │
│  aws ssm start-session \                                 │
│    --target i-1234567890abcdef0 \                        │
│    --document-name AWS-StartPortForwardingSession \      │
│    --parameters '{"portNumber":["80"],                   │
│                   "localPortNumber":["8080"]}'           │
└──────────────────────────────────────────────────────────┘
```

### Port Forwarding Đến Remote Host (SSM Plugin v1.1.23+)

```bash
# Tunnel đến RDS không qua EC2 làm trung gian
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["mydb.cluster-ro-abc123.ap-southeast-1.rds.amazonaws.com"],
    "portNumber": ["5432"],
    "localPortNumber": ["5432"]
  }'

# Sau đó kết nối bình thường từ local:
psql -h localhost -p 5432 -U admin -d mydb
```

---

## Kiểm Soát Truy Cập Với IAM

### Phân Quyền Chi Tiết (Fine-Grained Access Control)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSessionOnTaggedInstances",
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:instance/*"
      ],
      "Condition": {
        "StringEquals": {
          "ssm:resourceTag/Environment": "Production",
          "ssm:resourceTag/Team": "DevOps"
        }
      }
    },
    {
      "Sid": "DenySessionToSensitiveInstances",
      "Effect": "Deny",
      "Action": "ssm:StartSession",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ssm:resourceTag/Sensitivity": "HighSecurity"
        }
      }
    },
    {
      "Sid": "AllowSessionManagerCoreActions",
      "Effect": "Allow",
      "Action": [
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus",
        "ssm:DescribeInstanceProperties",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowTerminateOwnSessions",
      "Effect": "Allow",
      "Action": "ssm:TerminateSession",
      "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
    }
  ]
}
```

### Chạy Session Với User Cụ Thể (Run As)

```json
// Cấu hình trong Session Preferences
{
  "runAsEnabled": true,
  "runAsDefaultUser": "ssm-user"
}
```

```bash
# IAM user có thể override run-as user qua tag
# Gắn tag vào IAM user: SSMSessionRunAs = ec2-user
```

---

## VPC Endpoints Cho Private Subnet

### Khi Nào Cần VPC Endpoint?

```
Instance trong private subnet:
┌──────────────────────────────────────────────────────────┐
│  Private Subnet                                          │
│  ┌──────────┐                                            │
│  │  EC2     │                                            │
│  │  Instance│                                            │
│  └────┬─────┘                                            │
│       │                                                  │
│       │ Cần kết nối đến SSM endpoints                    │
│       │                                                  │
│  Tùy chọn 1: NAT Gateway → Internet → SSM endpoints      │
│  Tùy chọn 2: VPC Endpoints (PrivateLink) → Không qua IGW │
└──────────────────────────────────────────────────────────┘
```

### Tạo VPC Endpoints Cần Thiết

```bash
# 3 Interface Endpoints bắt buộc cho SSM Session Manager
ENDPOINTS=(
  "com.amazonaws.ap-southeast-1.ssm"
  "com.amazonaws.ap-southeast-1.ssmmessages"
  "com.amazonaws.ap-southeast-1.ec2messages"
)

for SERVICE in "${ENDPOINTS[@]}"; do
  aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0abc123def456789 \
    --service-name $SERVICE \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-0abc123 subnet-0def456 \
    --security-group-ids sg-0abc123 \
    --private-dns-enabled
done

# Thêm endpoint cho S3 (nếu dùng S3 session logging)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc123def456789 \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0abc123
```

### Security Group Cho VPC Endpoints

```bash
# Security Group cho Interface Endpoints
# Inbound: Port 443 từ EC2 instances trong VPC
aws ec2 authorize-security-group-ingress \
  --group-id sg-endpoint-0abc123 \
  --protocol tcp \
  --port 443 \
  --source-group sg-ec2-instances
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Giải thích Session Manager hoạt động như thế nào — tại sao không cần port 22?**

> Session Manager dùng SSM Agent trên instance để thiết lập kết nối WebSocket outbound đến SSM service endpoint qua HTTPS (port 443). Vì kết nối được khởi tạo từ bên trong instance ra ngoài (outbound), không có traffic inbound cần thiết — do đó không cần mở port 22 hay bất kỳ inbound port nào. Quyền truy cập được kiểm soát hoàn toàn bởi IAM.

**Q: Session Manager có thể thay thế hoàn toàn SSH không?**

> Hầu hết trường hợp có. Session Manager cho terminal access đầy đủ, port forwarding, và SSH tunneling. Ngoại lệ: Một số tác vụ yêu cầu SFTP, SCP để transfer file lớn — SSH vẫn tiện hơn. Tuy nhiên, ngay cả file transfer có thể giải quyết bằng S3, SSM Run Command, hoặc Session Manager với port forwarding.

**Q: Làm thế nào đảm bảo toàn bộ session đều được log?**

> Bật Session Preferences (Ưu Tiên Phiên) ở cấp SSM: bật CloudWatch Logs và S3 logging. Tạo SCP (Service Control Policy) trong Organizations ngăn người dùng tắt logging. Dùng Config Rule kiểm tra session preferences có bật logging không. Lưu S3 bucket logs với Object Lock để không ai xóa được.

### Nâng Cao

**Q: Thiết kế giải pháp zero-trust access cho EC2 fleet 1000 instances ở private subnet?**

> **(1) VPC Endpoints:** Tạo Interface Endpoints cho ssm, ssmmessages, ec2messages, và s3 trong mỗi VPC — loại bỏ hoàn toàn NAT Gateway dependency cho SSM traffic.
>
> **(2) IAM Conditions:** Dùng `ssm:resourceTag` conditions để chỉ cho phép session dựa trên tag môi trường và team. Dùng `aws:SourceIP` hoặc `aws:VpcSourceIp` để giới hạn IP source.
>
> **(3) Session Logging:** Bật logging bắt buộc vào S3 encrypted (KMS) + CloudWatch. Bật Object Lock trên S3 bucket với Compliance mode để đảm bảo log không bị xóa.
>
> **(4) MFA:** Yêu cầu MFA trong IAM policy trước khi StartSession: `"Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "true"}}`.
>
> **(5) Monitoring:** Dùng CloudTrail ghi lại `ssm:StartSession`, `ssm:TerminateSession`. Tạo CloudWatch alarm khi có session từ IAM user lạ hoặc ngoài giờ làm việc.

**Q: Port forwarding qua SSM có phù hợp production không?**

> Phù hợp cho maintenance và debugging, nhưng không dùng cho traffic production thường xuyên. Lý do: SSM Session Manager có latency cao hơn kết nối trực tiếp, và không phù hợp cho high-throughput database connections. Tuy nhiên, cho các tác vụ như DBA query ad-hoc, kiểm tra Redis, hoặc truy cập internal API — hoàn toàn phù hợp và an toàn hơn nhiều so với mở port ra internet.

---

**Liên Quan:** [Patch Manager](./2-patch-manager.md) | [Parameter Store](./3-parameter-store.md) | [Run Command](./4-run-command-automation.md)
