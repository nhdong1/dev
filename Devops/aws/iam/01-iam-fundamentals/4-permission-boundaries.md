# Permission Boundaries — Ranh Giới Quyền Hạn

> **Permission Boundaries** (Ranh Giới Quyền Hạn) là tính năng IAM cho phép **ủy quyền an toàn** (safe delegation) — một entity có quyền cao có thể trao cho entity khác khả năng tạo role/user, nhưng giới hạn quyền tối đa mà những role/user đó có thể có.

---

## Khái Niệm Cốt Lõi

### Permission Boundary Là Gì?

Là một managed policy được đặt làm **trần quyền** (permissions ceiling) cho một IAM user hoặc role. Quyền thực tế của entity = **giao điểm** (intersection) giữa:

1. **Identity-based policy** (quyền được cấp trực tiếp)
2. **Permission Boundary** (quyền tối đa được phép)

```
Quyền thực tế = Identity Policy ∩ Permission Boundary

Ví dụ minh họa:
┌─────────────────────────────────────────┐
│         Identity Policy                  │
│  s3:*  ec2:*  iam:*  lambda:*  rds:*   │
│                                          │
│    ┌──────────────────────────┐         │
│    │   Permission Boundary     │         │
│    │  s3:*  ec2:*  lambda:*   │         │
│    │                           │         │
│    │  (Giao điểm = Quyền thực)│         │
│    └──────────────────────────┘         │
│                                          │
│  iam:* và rds:* bị loại ra              │
└─────────────────────────────────────────┘

Quyền thực tế: s3:*  ec2:*  lambda:*
(dù identity policy có iam:* và rds:*, chúng bị boundary chặn)
```

### Boundary Không Cấp Quyền

Permission Boundary **không thể cấp quyền** mà identity policy chưa có. Nó chỉ giới hạn quyền tối đa.

```
Identity Policy: Allow s3:GetObject
Permission Boundary: Allow s3:*

→ Quyền thực tế: s3:GetObject (không phải s3:*)
  Boundary "mở rộng" s3:* nhưng identity chỉ có s3:GetObject
  → Chỉ s3:GetObject được phép
```

---

## Tại Sao Permission Boundaries Quan Trọng?

### Vấn Đề: Privilege Escalation (Leo Thang Đặc Quyền)

```
Kịch bản nguy hiểm (không có boundary):

Developer Alice có quyền:
  iam:CreateRole
  iam:AttachRolePolicy

Alice có thể:
1. Tạo role mới với AdministratorAccess
2. Assign role đó cho bản thân
→ Alice leo thang từ Developer lên Administrator!
```

### Giải Pháp: Permission Boundary Chặn Privilege Escalation

```
Với Permission Boundary:

Quản trị viên thiết lập:
  Mọi role Alice tạo ra PHẢI có Permission Boundary "DeveloperBoundary"
  DeveloperBoundary chỉ cho phép s3:*, lambda:*, dynamodb:*

Alice vẫn có thể:
  iam:CreateRole (để tạo Lambda execution role)

Nhưng role Alice tạo ra:
  Dù attach AdministratorAccess → chỉ có s3:*, lambda:*, dynamodb:*
  (bị boundary chặn)

→ Không thể leo thang đặc quyền!
```

---

## Use Cases Chính

### 1. Delegate IAM Permission Đến Developer

**Scenario:** Cho phép developer tự tạo IAM role cho Lambda function của họ, nhưng không cho tạo role với quyền hơn họ hiện có.

**Step 1: Tạo Permission Boundary cho Lambda roles**

```json
// Policy: LambdaRoleBoundary
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaServices",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

**Step 2: Cấp quyền cho developer, nhưng bắt buộc attach boundary**

```json
// Policy cho Developer (IAM permissions với ràng buộc)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCreateRoleWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:PutRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": 
            "arn:aws:iam::123456789012:policy/LambdaRoleBoundary"
        }
      }
    },
    {
      "Sid": "AllowPassRoleToLambda",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "lambda.amazonaws.com"
        }
      }
    },
    {
      "Sid": "DenyBoundaryModification",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:PutUserPermissionsBoundary",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "*"
    }
  ]
}
```

**Kết quả:** Developer tạo được Lambda role, nhưng role đó không bao giờ có quyền vượt quá `LambdaRoleBoundary`.

### 2. Service Team Isolation (Cô Lập Giữa Các Team)

```
Cấu trúc tổ chức:

Platform Team (quản lý IAM) → Tạo Boundary cho từng service team

TeamA-Boundary: Allow s3:* trên bucket "team-a-*", ec2:* ở VPC "vpc-teamA"
TeamB-Boundary: Allow s3:* trên bucket "team-b-*", ec2:* ở VPC "vpc-teamB"

Team A có thể tự quản lý role của mình,
nhưng không thể vô tình (hoặc cố ý) truy cập tài nguyên Team B
```

### 3. Temporary Elevated Access (Truy Cập Đặc Quyền Tạm Thời)

```
Kịch bản: Cho phép on-call engineer có quyền cao hơn bình thường
trong khung giờ xử lý sự cố, sau đó tự động thu hồi.

1. Tạo "IncidentResponderBoundary" với quyền rộng hơn
2. Lambda function chạy theo schedule tự động:
   - Khi bắt đầu incident: attach boundary rộng hơn
   - Khi hết giờ on-call: revert về boundary hẹp hơn
```

---

## Cú Pháp CLI

### Gắn Boundary Khi Tạo Role

```bash
aws iam create-role \
  --role-name LambdaRole \
  --assume-role-policy-document file://trust-policy.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/LambdaRoleBoundary
```

### Thêm Boundary Vào Role Hiện Có

```bash
aws iam put-role-permissions-boundary \
  --role-name ExistingRole \
  --permissions-boundary arn:aws:iam::123456789012:policy/MyBoundary
```

### Xem Boundary Hiện Tại

```bash
aws iam get-role --role-name MyRole \
  --query 'Role.PermissionsBoundary'
```

### Xóa Boundary

```bash
aws iam delete-role-permissions-boundary \
  --role-name MyRole
```

---

## Interaction Với Các Policy Khác

### Permission Boundary vs SCP

```
SCP → áp dụng cho ACCOUNT (tất cả identities trong account)
Boundary → áp dụng cho INDIVIDUAL user/role

Cả hai đều là "trần quyền" nhưng ở cấp độ khác nhau:

Effective permission = Identity Policy ∩ Boundary ∩ SCP
                     (trong trường hợp có đủ cả ba)
```

### Permission Boundary vs Resource-based Policy

```
Resource-based policy với cross-account:
  Quyền = Identity Policy ∩ Boundary ∩ Resource Policy

Boundary KHÔNG ảnh hưởng đến resource-based policy trong cùng account
→ Nếu S3 bucket policy Allow principal X, nhưng X bị boundary chặn s3:GetObject
→ Request VẪN BỊ TỪ CHỐI (boundary áp dụng cho principal, không cho bucket)
```

### Tổng Hợp Logic

```
1. Explicit Deny ở bất kỳ đâu → TỪ CHỐI

2. Nếu request từ principal có Boundary:
   - Identity Policy Allow AND Boundary Allow → tiếp tục đánh giá
   - Nếu thiếu một trong hai → TỪ CHỐI

3. SCPs (nếu dùng Organizations):
   - SCP phải Allow → tiếp tục
   - Nếu không → TỪ CHỐI

4. Resource-based Policy Allow (trong cùng account) → CHO PHÉP

5. Identity Policy Allow → CHO PHÉP (cuối cùng)
```

---

## Giới Hạn Của Permission Boundaries

| Giới Hạn | Chi Tiết |
|---|---|
| Chỉ áp dụng cho | IAM Users và Roles (không phải Groups) |
| Số boundary/entity | Chỉ 1 boundary per user hoặc role |
| Loại policy | Chỉ managed policy (không dùng inline) |
| Service-linked roles | Không áp dụng được boundary |
| Root user | Không áp dụng được boundary |

---

## Ví Dụ Boundary Thực Tế: DevOps Team

```json
// DevOpsBoundary — Giới hạn DevOps engineers
// Họ có thể làm mọi thứ với EC2, ECS, Lambda
// Nhưng KHÔNG được đụng vào IAM ngoài read và role cho services

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CoreDevOpsServices",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "ecs:*",
        "ecr:*",
        "lambda:*",
        "s3:*",
        "cloudformation:*",
        "logs:*",
        "cloudwatch:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "route53:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "LimitedIAM",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:PassRole"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAMUserAndGroupManagement",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:CreateGroup",
        "iam:DeleteGroup",
        "iam:CreateLoginProfile",
        "iam:CreateAccessKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Permission Boundary cấp quyền không? Tại sao?**

> Không. Boundary chỉ giới hạn quyền tối đa có thể có — nó không tự cấp quyền. Để action được phép, cần CẢ HAI: identity-based policy Allow VÀ boundary Allow. Nếu identity policy không có s3:DeleteObject nhưng boundary cho phép, action vẫn bị từ chối.

**Q: Tại sao cần Permission Boundary khi đã có SCPs?**

> SCPs kiểm soát ở cấp account/OU — giới hạn giống nhau cho tất cả identities trong account. Boundary kiểm soát ở cấp individual role/user — cho phép granular control từng entity. Use case điển hình: dùng SCPs để enforce organizational compliance, dùng Boundaries để an toàn delegate IAM permissions xuống developer teams mà không lo privilege escalation.

**Q: Kịch bản nào boundary giải quyết mà SCP không giải quyết được?**

> **Safe IAM delegation**: Platform team muốn cho Developer team tự quản lý IAM roles cho microservices của họ. SCPs không thể ngăn developer tạo role quá quyền trong cùng account. Boundary giải quyết bằng cách: cho phép `iam:CreateRole` nhưng yêu cầu mọi role được tạo ra phải có boundary cụ thể, đảm bảo developer không thể tạo role với quyền cao hơn chính họ.
