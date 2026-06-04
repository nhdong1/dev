# IAM Roles & Instance Profiles — Quản Lý Quyền Truy Cập Cho AWS Compute

> IAM (Identity and Access Management — Quản Lý Danh Tính và Quyền Truy Cập) là nền tảng bảo mật của AWS. Instance Profiles (Hồ Sơ Máy Chủ) cho phép EC2 instances và các compute services tương tác an toàn với AWS APIs mà không cần hard-code credentials (thông tin xác thực nhúng trong mã).

---

## 📚 Mục Lục

1. [Kiến Trúc IAM Tổng Quan](#kiến-trúc-iam-tổng-quan)
2. [IAM Roles cho Compute Services](#iam-roles-cho-compute-services)
3. [EC2 Instance Profiles](#ec2-instance-profiles)
4. [IAM Policies — Chính Sách Quyền](#iam-policies--chính-sách-quyền)
5. [Least Privilege — Nguyên Tắc Đặc Quyền Tối Thiểu](#least-privilege--nguyên-tắc-đặc-quyền-tối-thiểu)
6. [Thực Hành Best Practices](#thực-hành-best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc IAM Tổng Quan

### Các Thành Phần Cốt Lõi

```
IAM (Identity and Access Management)
├── Principal (Chủ Thể) — ai đang thực hiện hành động?
│   ├── IAM Users (Người Dùng IAM) — con người
│   ├── IAM Roles (Vai Trò IAM) — services, apps, cross-account
│   └── AWS Services — EC2, Lambda, ECS…
│
├── Action (Hành Động) — làm gì?
│   ├── s3:GetObject, ec2:DescribeInstances…
│   └── Định nghĩa trong IAM Policies
│
├── Resource (Tài Nguyên) — trên cái gì?
│   └── ARN (Amazon Resource Name — Tên Tài Nguyên Amazon)
│       ví dụ: arn:aws:s3:::my-bucket/my-key
│
└── Condition (Điều Kiện) — trong hoàn cảnh nào?
    ├── IP address, MFA, time of day…
    └── Giúp fine-tune permissions
```

### Authentication vs Authorization

```
Authentication (Xác Thực) — "Bạn là ai?"
├── IAM Users: username + password + MFA
├── IAM Roles: AssumeRole (giả định vai trò) + STS token
└── EC2 Instance: Instance Profile → lấy credentials từ IMDS

Authorization (Phân Quyền) — "Bạn được làm gì?"
└── IAM Policies đánh giá theo thứ tự:
    1. Explicit DENY (từ chối rõ ràng) → DENY ngay lập tức
    2. Explicit ALLOW (cho phép rõ ràng) → ALLOW
    3. Implicit DENY (từ chối ngầm định) → DENY (mặc định)
```

---

## IAM Roles cho Compute Services

### Tại Sao Dùng Roles Thay Vì Access Keys?

```
❌ VẤN ĐỀ với Access Keys (Khóa Truy Cập):
   • Long-term credentials (thông tin xác thực dài hạn) — không tự hết hạn
   • Thường bị hard-code trong code hoặc config files
   • Dễ bị lộ qua Git, logs, hoặc process listing
   • Khó rotate (xoay vòng) khi có nhiều services dùng chung

✅ LỢI ÍCH của IAM Roles:
   • Temporary credentials (thông tin xác thực tạm thời) — tự động rotate mỗi 1 giờ
   • AWS tự quản lý — không cần lưu trữ
   • Tuân theo principle of least privilege dễ hơn
   • Audit qua CloudTrail dễ theo dõi hơn
```

### Cách IAM Roles Hoạt Động — STS (Security Token Service)

```
┌────────────────────────────────────────────────────────────┐
│                    IAM Role Flow                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  EC2 Instance         IMDS v2              STS             │
│  ─────────────        ──────────           ──────────      │
│       │                   │                   │            │
│       │  Request creds     │                   │            │
│       │──────────────────►│                   │            │
│       │                   │  AssumeRole        │            │
│       │                   │──────────────────►│            │
│       │                   │                   │ Validate   │
│       │                   │  Temp Credentials  │ Role       │
│       │                   │◄──────────────────│            │
│       │  Return creds      │                   │            │
│       │◄──────────────────│                   │            │
│       │                   │                   │            │
│  App sử dụng temp credentials (~1 giờ TTL)    │            │
│  AWS SDK tự động refresh trước khi hết hạn   │            │
│                                                            │
└────────────────────────────────────────────────────────────┘

IMDS = Instance Metadata Service — Dịch Vụ Siêu Dữ Liệu Máy Chủ
STS  = Security Token Service — Dịch Vụ Token Bảo Mật
TTL  = Time To Live — Thời Gian Sống
```

### Roles Cho Từng Compute Service

#### EC2 — Instance Profile (Hồ Sơ Máy Chủ)

```bash
# Tạo IAM Role cho EC2
aws iam create-role \
  --role-name EC2-S3-ReadRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

# Gắn policy vào role
aws iam attach-role-policy \
  --role-name EC2-S3-ReadRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Tạo Instance Profile và thêm Role vào
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-ReadProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadProfile \
  --role-name EC2-S3-ReadRole

# Gắn Instance Profile vào EC2 khi launch
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --iam-instance-profile Name=EC2-S3-ReadProfile \
  --key-name my-key-pair
```

#### Lambda — Execution Role (Vai Trò Thực Thi)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
# Policy cho Lambda Execution Role cơ bản
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
    // Thêm permissions tùy theo nhu cầu của function
  ]
}
```

#### ECS — Task Role (Vai Trò Task) vs Task Execution Role

```
Task Role (Vai Trò Task):
└── Permissions mà APPLICATION trong container cần
    ví dụ: đọc S3, ghi DynamoDB, gọi API khác

Task Execution Role (Vai Trò Thực Thi Task):
└── Permissions mà ECS AGENT cần để setup container
    ví dụ: pull image từ ECR, lấy secrets từ Secrets Manager
    ví dụ: ghi logs vào CloudWatch

         ┌─────────────────────────────────────┐
         │          ECS Task                    │
         │                                      │
         │  ┌──────────────┐  Task Role         │
         │  │  Application │ ─────────────────► S3, DynamoDB
         │  │  Container   │                    │
         │  └──────────────┘                    │
         │                                      │
         │  ECS Agent ─── Task Execution Role ► ECR, CloudWatch
         └─────────────────────────────────────┘
```

#### EKS — IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho Service Accounts)

```bash
# IRSA cho phép từng Kubernetes Pod có IAM permissions riêng
# Không cần dùng node-level IAM role (quá nhiều permissions)

# 1. Tạo IAM OIDC Provider cho EKS cluster
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# 2. Tạo IAM Role với trust relationship cho Service Account
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace default \
  --name my-service-account \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# 3. Pod tự động nhận credentials qua projected volume
# Không cần thay đổi application code
```

---

## EC2 Instance Profiles

### Instance Profile vs IAM Role

```
IAM Role (Vai Trò IAM):
└── Container trừu tượng chứa permissions
    → Có thể gắn vào nhiều loại Principal khác nhau

Instance Profile (Hồ Sơ Máy Chủ):
└── Container đặc biệt cho IAM Role khi dùng với EC2
    → Một Profile chỉ chứa một Role
    → Được truyền vào EC2 khi launch
    → AWS Console tự động tạo Profile cùng tên Role

Quan hệ:
Role ←── gắn vào ──► Instance Profile ←── gắn vào ──► EC2 Instance
```

### Thay Đổi Instance Profile Khi EC2 Đang Chạy

```bash
# Xem instance profile hiện tại
aws ec2 describe-instances \
  --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[].Instances[].IamInstanceProfile'

# Gắn instance profile mới (nếu chưa có)
aws ec2 associate-iam-instance-profile \
  --instance-id i-1234567890abcdef0 \
  --iam-instance-profile Name=NewInstanceProfile

# Thay thế instance profile đang có
ASSOCIATION_ID=$(aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=i-1234567890abcdef0 \
  --query 'IamInstanceProfileAssociations[0].AssociationId' \
  --output text)

aws ec2 replace-iam-instance-profile-association \
  --iam-instance-profile Name=NewInstanceProfile \
  --association-id $ASSOCIATION_ID
```

### Lấy Credentials Từ IMDS (Instance Metadata Service — Dịch Vụ Siêu Dữ Liệu)

```bash
# IMDS v2 (khuyến nghị) — yêu cầu session token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Lấy tên của Instance Profile
PROFILE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/)

# Lấy credentials tạm thời
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/$PROFILE

# Response:
# {
#   "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
#   "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG...",
#   "Token": "AQoDYXdzEJr...",   ← Session token
#   "Expiration": "2026-05-15T18:00:00Z"
# }
```

### Enforce IMDS v2 — Bắt Buộc Dùng Phiên Bản 2

```bash
# Enforce IMDSv2 khi launch instance mới
aws ec2 run-instances \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=1"

# Enforce IMDSv2 trên instance đang chạy
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \
  --http-put-response-hop-limit 1

# HttpPutResponseHopLimit=1: chặn container bên trong VM
# truy cập IMDS của host (quan trọng khi chạy containers)
```

---

## IAM Policies — Chính Sách Quyền

### Cấu Trúc JSON Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadBucket",            // Statement ID (tùy chọn)
      "Effect": "Allow",                      // Allow hoặc Deny
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ],
      "Condition": {                          // Điều kiện thêm (tùy chọn)
        "StringEquals": {
          "aws:RequestedRegion": "ap-southeast-1"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

### Các Loại Policy

```
AWS Managed Policies (Chính Sách AWS Quản Lý):
├── Ví dụ: AmazonS3ReadOnlyAccess, AmazonEC2FullAccess
├── AWS tạo và cập nhật khi có service mới
└── Không thể chỉnh sửa — dùng khi cần nhanh

Customer Managed Policies (Chính Sách Tự Tạo):
├── Bạn tạo, quản lý, versioning
├── Có thể reuse cho nhiều roles
└── Khuyến nghị cho production

Inline Policies (Chính Sách Nội Tuyến):
├── Gắn trực tiếp vào một User/Role/Group cụ thể
├── Xóa cùng khi xóa entity
└── Dùng khi cần policy unique cho entity đó

Resource-Based Policies (Chính Sách Dựa Trên Tài Nguyên):
├── Gắn vào resource (S3 bucket, KMS key…)
├── Định nghĩa ai được phép access resource
└── Cho phép cross-account access không cần assume role
```

### Policy Evaluation Logic — Logic Đánh Giá

```
Khi có request đến AWS:

1. Collect tất cả applicable policies:
   - SCPs (Service Control Policies) từ AWS Organizations
   - Permission Boundaries (Ranh Giới Quyền)
   - IAM Identity Policies (Chính Sách Danh Tính)
   - Resource-Based Policies

2. Đánh giá theo thứ tự:
   Step 1: Có Explicit DENY nào không? → DENY (dừng)
   Step 2: Có SCP DENY không? → DENY (dừng)
   Step 3: Permission Boundary có ALLOW không?
   Step 4: Có ALLOW trong identity policy không?
   Step 5: Có Resource-Based Policy ALLOW không?
   Step 6: Không tìm thấy ALLOW → DENY (implicit deny)

Kết luận: Default là DENY — phải explicitly ALLOW
```

---

## Least Privilege — Nguyên Tắc Đặc Quyền Tối Thiểu

### Định Nghĩa

> **Least Privilege (Đặc Quyền Tối Thiểu):** Mỗi entity chỉ được có đúng các permissions cần thiết để hoàn thành công việc của mình — không hơn, không kém.

### Tại Sao Quan Trọng?

```
Blast Radius (Phạm Vi Thiệt Hại) khi bị compromise:

❌ Overprivileged Role (Role Quá Nhiều Quyền):
   EC2 với AdministratorAccess
   → Attacker có thể: tạo IAM users, xóa dữ liệu, launch instances...
   → Blast radius: TOÀN BỘ AWS account

✅ Least Privilege Role:
   EC2 chỉ có s3:GetObject vào specific bucket
   → Attacker chỉ có thể: đọc files trong một bucket
   → Blast radius: RẤT NHỎ
```

### Quy Trình Áp Dụng Least Privilege

```
Bước 1: Bắt đầu với permissions rộng (để ứng dụng chạy được)
   → Dùng AWS Managed Policy rộng ban đầu

Bước 2: Dùng IAM Access Analyzer để phân tích
   → Xem permissions nào thực sự được dùng

Bước 3: Thu hẹp permissions dựa trên access report
   → Tạo Customer Managed Policy với chỉ permissions cần thiết

Bước 4: Monitor và tiếp tục cải thiện
   → Dùng CloudTrail để phát hiện unused permissions
```

### Công Cụ Hỗ Trợ Least Privilege

#### IAM Access Analyzer — Phân Tích Quyền Truy Cập

```bash
# Tạo Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name my-analyzer \
  --type ACCOUNT

# Xem findings (phát hiện)
aws accessanalyzer list-findings \
  --analyzer-name my-analyzer

# Lấy báo cáo: permissions nào đã được dùng trong 90 ngày qua
aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::123456789012:role/MyRole

# Xem chi tiết report
aws iam get-service-last-accessed-details \
  --job-id <job-id-từ-lệnh-trên>
```

#### IAM Policy Simulator — Mô Phỏng Chính Sách

```bash
# Kiểm tra policy trước khi apply
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
  --action-names s3:GetObject s3:DeleteObject \
  --resource-arns arn:aws:s3:::my-bucket/*
```

### Permission Boundaries — Ranh Giới Quyền

```
Permission Boundary (Ranh Giới Quyền):
└── Giới hạn MAXIMUM permissions mà một entity có thể có
    → Entity vẫn cần có explicit ALLOW trong identity policy
    → Dùng khi cần delegate role creation cho teams khác

Ví dụ: Dev team được tạo roles nhưng không vượt quá boundary

    Permission Boundary        Identity Policy
    ┌──────────────────┐      ┌──────────────────┐
    │ S3: Full Access  │  ∩   │ S3: Full Access  │  = S3: Full Access
    │ EC2: Describe    │      │ EC2: Full Access  │  = EC2: Describe only
    │                  │      │ RDS: Full Access  │  = DENIED (không trong boundary)
    └──────────────────┘      └──────────────────┘
```

```json
// Ví dụ Permission Boundary: chỉ cho phép dùng S3 và SSM
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ssm:*",
        "logs:*",
        "cloudwatch:*"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Thực Hành Best Practices

### 1. Không Bao Giờ Hard-Code Credentials

```python
# ❌ SAI — tuyệt đối không làm thế này
import boto3
session = boto3.Session(
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',
    aws_secret_access_key='wJalrXUtnFEMI...'
)

# ✅ ĐÚNG — SDK tự lấy credentials từ role
import boto3
client = boto3.client('s3')  # Tự dùng credentials từ IAM Role
```

### 2. Nguyên Tắc Phân Tách — Separation of Duties

```
Môi Trường Dev:
├── Dev role: Tạo/sửa/xóa resources trong dev account
└── Không có quyền vào prod account

Môi Trường Prod:
├── App role: Chỉ đọc/ghi vào specific resources
├── Deployment role: Chỉ update ECS/Lambda/EC2 (không xóa)
└── Admin role: Full access, yêu cầu MFA, limited users
```

### 3. Cross-Account Access — Truy Cập Đa Tài Khoản

```bash
# Tài khoản B muốn cho phép Tài khoản A truy cập
# Trong Tài khoản B: Tạo role với trust relationship

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT-A-ID:role/DevRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id-12345"  // Prevent confused deputy attack
      }
    }
  }]
}

# Trong Tài khoản A: Assume role để access Tài khoản B
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT-B-ID:role/CrossAccountRole \
  --role-session-name MySession \
  --external-id unique-external-id-12345
```

### 4. Tag-Based Access Control — Kiểm Soát Dựa Trên Thẻ

```json
// Cho phép EC2 chỉ tương tác với resources cùng project tag
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "s3:ResourceTag/Project": "${aws:PrincipalTag/Project}"
      }
    }
  }]
}
```

### 5. Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ

```json
// SCP ngăn chặn việc tắt CloudTrail (audit trail bảo mật)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyDisablingCloudTrail",
    "Effect": "Deny",
    "Action": [
      "cloudtrail:DeleteTrail",
      "cloudtrail:StopLogging",
      "cloudtrail:UpdateTrail"
    ],
    "Resource": "*"
  }]
}
```

### 6. Rotate Credentials Định Kỳ — Xoay Vòng Thông Tin Xác Thực

```bash
# Kiểm tra access keys nào chưa được rotate trong 90 ngày
aws iam generate-credential-report
aws iam get-credential-report \
  --output text \
  --query 'Content' \
  | base64 -d

# Tạo access key mới, cập nhật apps, xóa key cũ
aws iam create-access-key --user-name my-user
# ... cập nhật apps dùng key mới ...
aws iam delete-access-key \
  --user-name my-user \
  --access-key-id AKIAIOSFODNN7EXAMPLE
```

---

## Monitoring IAM — Giám Sát Quyền Truy Cập

### CloudTrail cho IAM Events

```bash
# Tìm tất cả AssumeRole calls trong 1 giờ qua
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time $(date -d '-1 hour' --iso-8601=seconds)

# Tìm ai đã tạo/xóa IAM resources
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateRole

# Lọc bằng CloudWatch Logs Insights
fields @timestamp, userIdentity.arn, eventName, sourceIPAddress
| filter eventSource = "iam.amazonaws.com"
| filter eventName in ["CreateRole", "AttachRolePolicy", "DeleteRole"]
| sort @timestamp desc
| limit 50
```

### GuardDuty IAM Findings

```
GuardDuty phát hiện:
├── IAMUser:InitialAccessTacticsDetected — đăng nhập từ IP lạ
├── UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B
│   → Đăng nhập từ IP không thông thường
├── Recon:IAMUser/MaliciousIPCaller
│   → API calls từ IP được biết là ác ý
└── PrivilegeEscalation:IAMUser/AdministrativePermissions
    → Cố gắng leo thang đặc quyền
```

---

## Câu Hỏi Phỏng Vấn

### Q1: EC2 Instance Profile là gì và tại sao dùng thay vì access keys?

**Trả lời:** Instance Profile là container chứa một IAM Role cho EC2 instances. Khi EC2 có Instance Profile, AWS tự động cung cấp temporary credentials (thông tin xác thực tạm thời) qua IMDS (Instance Metadata Service — Dịch Vụ Siêu Dữ Liệu Máy Chủ). Credentials này tự động rotate mỗi 1 giờ, nên không cần quản lý thủ công. So với access keys (khóa truy cập tĩnh), Instance Profile an toàn hơn nhiều vì: không có long-term credentials để bị đánh cắp, tuân theo least privilege dễ hơn, và mọi API calls đều được audit trong CloudTrail với identity cụ thể.

### Q2: Giải thích IAM Policy evaluation logic (logic đánh giá chính sách)?

**Trả lời:** AWS đánh giá theo thứ tự ưu tiên: đầu tiên kiểm tra explicit DENY — nếu có bất kỳ policy nào DENY thì từ chối ngay. Tiếp theo kiểm tra explicit ALLOW — nếu có ALLOW và không có DENY nào, thì cho phép. Cuối cùng nếu không tìm thấy ALLOW, AWS áp dụng implicit DENY (từ chối ngầm định). Thứ tự áp dụng: SCPs (Service Control Policies) → Permission Boundaries → Identity Policies → Resource-Based Policies. Điều này có nghĩa là một Explicit DENY luôn thắng mọi ALLOW.

### Q3: Least Privilege là gì và cách implement?

**Trả lời:** Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu) nghĩa là mỗi entity chỉ được phép thực hiện đúng những actions cần thiết cho công việc của nó. Cách implement: bắt đầu với permissions rộng để ứng dụng hoạt động, dùng IAM Access Analyzer để xem permissions nào thực sự được sử dụng trong 90 ngày qua, sau đó tạo Customer Managed Policy với chỉ những permissions đó. Tiếp tục monitor qua CloudTrail và Access Analyzer để phát hiện permissions không dùng đến.

### Q4: IRSA (IAM Roles for Service Accounts) trong EKS hoạt động thế nào?

**Trả lời:** IRSA cho phép từng Kubernetes Pod có IAM permissions riêng thay vì dùng permissions của toàn bộ EC2 node. Cách hoạt động: EKS cluster được tích hợp với OIDC Provider (OpenID Connect — Nhà Cung Cấp Xác Thực Mở). Khi tạo IAM Role, định nghĩa trust relationship cho phép OIDC Provider của cluster đó assume role. Khi gắn Role vào Kubernetes Service Account, Pods dùng Service Account đó sẽ tự động nhận IAM credentials qua projected volume, không cần node-level permissions. Lợi ích: giảm blast radius, tuân theo least privilege ở cấp Pod, dễ audit.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Tổng Quan Security | [README.md](./README.md) |
| → SSM Systems Manager | [2-systems-manager.md](./2-systems-manager.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
