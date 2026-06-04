# 🏷️ ABAC — Attribute-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Thuộc Tính

> **ABAC** (Attribute-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Thuộc Tính) là mô hình cấp quyền sử dụng **thuộc tính** (tags) của principal, resource và môi trường để đưa ra quyết định — thay vì liệt kê cứng từng ARN hay tên resource trong policy.

---

## 📚 Mục Lục

1. [ABAC vs RBAC — So Sánh Hai Mô Hình](#1-abac-vs-rbac)
2. [IAM Tags Làm Thuộc Tính](#2-iam-tags-làm-thuộc-tính)
3. [Condition Keys Cốt Lõi](#3-condition-keys-cốt-lõi)
4. [Ví Dụ Policy ABAC Hoàn Chỉnh](#4-ví-dụ-policy-abac-hoàn-chỉnh)
5. [Session Tags Với STS AssumeRole](#5-session-tags-với-sts-assumerole)
6. [ABAC Với IAM Identity Center](#6-abac-với-iam-identity-center)
7. [Khi Nào Dùng ABAC vs RBAC](#7-khi-nào-dùng-abac-vs-rbac)
8. [Best Practices](#8-best-practices)
9. [Pitfalls — Bẫy Phổ Biến](#9-pitfalls--bẫy-phổ-biến)
10. [Lab Exercise](#10-lab-exercise)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. ABAC vs RBAC

### RBAC — Role-Based Access Control (Kiểm Soát Truy Cập Dựa Trên Vai Trò)

Trong RBAC, quyền được gắn vào **role**, và role được gắn vào **user**. Muốn thay đổi quyền, phải sửa role hoặc gán role khác.

```
RBAC Structure (Cấu Trúc RBAC):

User Alice ─── assume ──► role-dev-team-a-dev
User Bob   ─── assume ──► role-dev-team-b-dev
User Carol ─── assume ──► role-dev-team-a-prod
User Dave  ─── assume ──► role-dev-team-b-prod

Policy của role-dev-team-a-dev:
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::team-a-dev-bucket"
  ← Hardcode ARN cụ thể
}

Vấn đề: Thêm team C → phải tạo 2 role mới + 2 policy mới
         50 teams × 3 envs = 150 roles!
```

### ABAC — Attribute-Based Access Control (Kiểm Soát Truy Cập Dựa Trên Thuộc Tính)

Trong ABAC, policy so sánh **tag của principal** với **tag của resource**. Một policy duy nhất phục vụ mọi team, mọi môi trường.

```
ABAC Structure (Cấu Trúc ABAC):

User Alice ─── assume ──► role-developer (tag: team=a, env=dev)
User Bob   ─── assume ──► role-developer (tag: team=b, env=dev)
User Carol ─── assume ──► role-developer (tag: team=a, env=prod)

Policy của role-developer (DUY NHẤT):
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/team": "${aws:PrincipalTag/team}",
      "aws:ResourceTag/env":  "${aws:PrincipalTag/env}"
    }
  }
}

Kết quả:
  Alice (team=a, env=dev) → chỉ truy cập bucket có tag team=a, env=dev ✅
  Bob   (team=b, env=dev) → chỉ truy cập bucket có tag team=b, env=dev ✅
  Thêm team C: chỉ cần tạo bucket với tag team=c → tự động được cấp quyền! ✅
```

### Bảng So Sánh ABAC vs RBAC

| Tiêu Chí | RBAC | ABAC |
|---|---|---|
| **Cơ chế** | Quyền gắn vào role/group | Quyền dựa trên so sánh attributes |
| **Số policy** | Nhiều (1 per role/team/env combination) | Ít (1 policy cho nhiều trường hợp) |
| **Scalability** | Kém — số role tăng tuyến tính | Tốt — thêm tag, không thêm policy |
| **Độ phức tạp ban đầu** | Đơn giản | Phức tạp hơn để setup |
| **Kiểm toán (Audit)** | Dễ — nhìn vào role là biết | Khó hơn — phải xem cả tags |
| **Yêu cầu tagging** | Không bắt buộc | Bắt buộc — thiếu tag = mất quyền |
| **Cross-account** | Dễ quản lý | Phức tạp hơn |
| **Phù hợp với** | Môi trường nhỏ, ổn định | Môi trường lớn, nhiều team |

---

## 2. IAM Tags Làm Thuộc Tính

### Tags Trên Principal (Định Danh)

Tags có thể gắn vào:
- **IAM User** — thông qua console hoặc CLI
- **IAM Role** — thông qua role's trust policy hoặc tags
- **Session** — thông qua STS AssumeRole với session tags

```bash
# Gắn tag vào IAM User
aws iam tag-user \
  --user-name alice \
  --tags Key=team,Value=platform Key=env,Value=dev Key=project,Value=core

# Gắn tag vào IAM Role
aws iam tag-role \
  --role-name developer-role \
  --tags Key=team,Value=platform

# Xem tags của user
aws iam list-user-tags --user-name alice
```

### Tags Trên Resources (Tài Nguyên)

```bash
# Tag S3 bucket
aws s3api put-bucket-tagging \
  --bucket my-platform-dev-bucket \
  --tagging 'TagSet=[{Key=team,Value=platform},{Key=env,Value=dev}]'

# Tag EC2 Instance
aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags Key=team,Value=platform Key=env,Value=dev

# Tag Secret trong Secrets Manager
aws secretsmanager tag-resource \
  --secret-id /platform/dev/db-password \
  --tags Key=team,Value=platform Key=env,Value=dev
```

### Tag Naming Convention (Quy Ước Đặt Tên Tag)

```
Khuyến nghị dùng các tag keys nhất quán:

┌────────────────┬───────────────────┬──────────────────────────┐
│ Tag Key        │ Ví dụ Value        │ Mục Đích                 │
├────────────────┼───────────────────┼──────────────────────────┤
│ team           │ platform, data, fe │ Phân biệt team sở hữu   │
│ env            │ dev, staging, prod │ Môi trường               │
│ project        │ payments, auth     │ Project cụ thể           │
│ costcenter     │ cc-1234            │ Phân bổ chi phí          │
│ classification │ public, internal,  │ Mức độ nhạy cảm dữ liệu │
│                │ confidential       │                          │
└────────────────┴───────────────────┴──────────────────────────┘
```

---

## 3. Condition Keys Cốt Lõi

### `aws:PrincipalTag/key` — Tag Của Principal Đang Gửi Request

```
Ý nghĩa: Giá trị của tag có key=<key> gắn vào principal đang gửi request.
Áp dụng cho: IAM User, IAM Role, hoặc Session (qua session tags)

Ví dụ:
  aws:PrincipalTag/team   → "platform" (nếu user/role có tag team=platform)
  aws:PrincipalTag/env    → "dev"
  aws:PrincipalTag/project → "payments"
```

### `aws:ResourceTag/key` — Tag Của Resource Đang Được Truy Cập

```
Ý nghĩa: Giá trị của tag có key=<key> gắn vào resource mà request đang nhắm tới.
Lưu ý: Không phải mọi service đều hỗ trợ condition key này!

Ví dụ:
  aws:ResourceTag/team → "platform" (nếu S3 bucket có tag team=platform)
  aws:ResourceTag/env  → "prod"
```

**Danh sách services hỗ trợ `aws:ResourceTag`:**

| Service | Hỗ Trợ `aws:ResourceTag` | Ghi Chú |
|---|---|---|
| S3 (Object level) | ✅ Có | Dùng `s3:ResourceTag` cho object |
| EC2 | ✅ Có | Instance, Volume, SG, VPC... |
| RDS | ✅ Có | DB Instance, Cluster |
| Lambda | ✅ Có | Function |
| Secrets Manager | ✅ Có | Secret |
| KMS | ✅ Có | Key |
| DynamoDB | ✅ Có | Table |
| S3 (Bucket-level actions) | ⚠️ Một phần | Chỉ một số actions |
| SNS | ✅ Có | Topic |
| SQS | ✅ Có | Queue |

### `aws:RequestTag/key` — Tag Trong Request Đang Tạo Resource

```
Ý nghĩa: Giá trị của tag được gửi kèm trong request tạo/tag resource.
Dùng để: Bắt buộc phải gắn tag khi tạo resource.

Ví dụ use case:
  → Chỉ cho phép tạo EC2 instance nếu kèm tag "team" và "env"
  → Ngăn tạo resource không có tag (untagged resources)
```

### Bảng Tổng Hợp 3 Condition Keys

```
aws:PrincipalTag/X  ←── Từ NGƯỜI gửi request (User/Role/Session)
aws:ResourceTag/X   ←── Từ TÀI NGUYÊN bị truy cập
aws:RequestTag/X    ←── Từ nội dung REQUEST (khi tạo/tag resource)

ABAC pattern cơ bản:
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
       ↑ Tag của resource     ↑ Tag của người gửi request
    }
  }
  → Chỉ cho phép nếu resource.team == principal.team
```

---

## 4. Ví Dụ Policy ABAC Hoàn Chỉnh

### 4.1 Team Isolation — Cô Lập Theo Team

Policy này cho phép developer truy cập **chỉ những resource thuộc team của họ**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessToTeamResources",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::*",
        "arn:aws:s3:::*/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
        }
      }
    },
    {
      "Sid": "AllowEC2ActionsOnTeamInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/team": "${aws:PrincipalTag/team}"
        }
      }
    },
    {
      "Sid": "AllowEC2Describe",
      "Effect": "Allow",
      "Action": "ec2:DescribeInstances",
      "Resource": "*"
    }
  ]
}
```

### 4.2 Environment Isolation — Cô Lập Theo Môi Trường

Chỉ cho phép truy cập resource cùng môi trường:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccessSameEnvSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:*",
      "Condition": {
        "StringEquals": {
          "secretsmanager:ResourceTag/env": "${aws:PrincipalTag/env}"
        }
      }
    },
    {
      "Sid": "DenyProdAccessForNonProd",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": "prod"
        },
        "StringNotEquals": {
          "aws:PrincipalTag/env": "prod"
        }
      }
    }
  ]
}
```

### 4.3 Project-Based Access — Truy Cập Theo Project

Cho phép truy cập dựa trên cả team lẫn project:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDynamoDBAccessByProjectAndTeam",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/*",
      "Condition": {
        "StringEquals": {
          "dynamodb:LeadingKeys": "${aws:PrincipalTag/project}",
          "aws:ResourceTag/team": "${aws:PrincipalTag/team}",
          "aws:ResourceTag/project": "${aws:PrincipalTag/project}"
        }
      }
    }
  ]
}
```

### 4.4 Bắt Buộc Tag Khi Tạo Resource (Tagging Enforcement)

Policy buộc developer phải gắn tag khi tạo EC2:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRunInstancesWithRequiredTags",
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/team": "${aws:PrincipalTag/team}",
          "aws:RequestTag/env": "${aws:PrincipalTag/env}"
        },
        "ForAllValues:StringEquals": {
          "aws:TagKeys": ["team", "env", "project", "owner"]
        }
      }
    },
    {
      "Sid": "AllowRunInstancesForSupportingResources",
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": [
        "arn:aws:ec2:*:*:subnet/*",
        "arn:aws:ec2:*:*:network-interface/*",
        "arn:aws:ec2:*:*:security-group/*",
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:key-pair/*",
        "arn:aws:ec2:*::image/*"
      ]
    },
    {
      "Sid": "AllowCreateTagsOnLaunch",
      "Effect": "Allow",
      "Action": "ec2:CreateTags",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": "RunInstances"
        }
      }
    },
    {
      "Sid": "DenyCreateInstanceWithoutRequiredTags",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "Null": {
          "aws:RequestTag/team": "true",
          "aws:RequestTag/env": "true"
        }
      }
    }
  ]
}
```

---

## 5. Session Tags Với STS AssumeRole

Session tags (tag phiên) cho phép truyền thuộc tính vào temporary credential khi assume role — rất hữu ích khi identity đến từ IdP (Identity Provider) bên ngoài.

### Cách Hoạt Động

```
User login IdP (Okta/AD)
        │
        ▼ SAML assertion hoặc OIDC token
   AWS STS AssumeRoleWithSAML
   hoặc AssumeRoleWithWebIdentity
        │ kèm PrincipalTags từ IdP attribute
        ▼
   Temporary credentials
   với session tags: team=platform, env=dev
        │
        ▼
   IAM Policy kiểm tra aws:PrincipalTag/team
   → Khớp với resource tag → Cho phép truy cập
```

### CLI Example: AssumeRole Với Session Tags

```bash
# Assume role kèm session tags
aws sts assume-role \
  --role-arn "arn:aws:iam::123456789012:role/developer-role" \
  --role-session-name "alice-session" \
  --tags Key=team,Value=platform Key=env,Value=dev Key=project,Value=payments

# Kiểm tra session tags
aws sts get-caller-identity
```

### Trust Policy Để Cho Phép Session Tags

Role cần có trust policy cho phép `sts:TagSession`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/alice"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Condition": {
        "StringLike": {
          "sts:RoleSessionName": "${aws:username}"
        }
      }
    }
  ]
}
```

### Transitive Tag Keys — Tag Kế Thừa

Khi chaining AssumeRole (assume role → rồi assume role khác), có thể truyền session tags tiếp:

```bash
# Assume role lần 2 và giữ lại session tags từ lần 1
aws sts assume-role \
  --role-arn "arn:aws:iam::987654321098:role/cross-account-role" \
  --role-session-name "alice-cross-account" \
  --transitive-tag-keys "team" "env"
  # ← team và env sẽ được giữ lại qua các lần assume role tiếp theo
```

---

## 6. ABAC Với IAM Identity Center

IAM Identity Center (SSO) hỗ trợ ABAC thông qua **attribute mappings** (ánh xạ thuộc tính) từ identity source (nguồn định danh).

### Cấu Hình Attribute Mapping

```
Identity Source (Okta/Azure AD) → IAM Identity Center → Session Tags

Okta User Profile:
  department: "engineering/platform"
  costCenter: "CC-1234"
  jobTitle: "Senior Engineer"
              │
              ▼ (attribute mapping trong IAM Identity Center)
Session Tag:
  team: "platform"      ← từ department
  costcenter: "CC-1234" ← từ costCenter
```

**Trong IAM Identity Center Console:**
1. Settings → Attributes for access control
2. Add attribute mapping:
   - Session attribute key: `team`
   - Maps to user attribute: `${path:enterprise.department}`

### Permission Set Policy Dùng ABAC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ABACAccessViaSSOAttributes",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/team": "${aws:PrincipalTag/team}",
          "aws:ResourceTag/costcenter": "${aws:PrincipalTag/costcenter}"
        }
      }
    }
  ]
}
```

---

## 7. Khi Nào Dùng ABAC vs RBAC

### Dùng ABAC Khi:

```
✅ Tổ chức có nhiều team (> 5) và số team đang tăng
✅ Nhiều môi trường (dev/staging/prod) nhân với nhiều team
✅ Developer self-service: tự tạo resource mà không cần ops
✅ Kết hợp với IAM Identity Center (SSO) với attributes từ IdP
✅ Muốn giảm số lượng IAM policies cần maintain
✅ Resources được tạo tự động (IaC) và luôn được tag đúng
```

### Dùng RBAC Khi:

```
✅ Tổ chức nhỏ, ít thay đổi về team/project
✅ Quyền truy cập cần định nghĩa rõ ràng theo vai trò cụ thể
✅ Cross-account access (ABAC cross-account phức tạp hơn)
✅ Services không hỗ trợ aws:ResourceTag condition key
✅ Audit yêu cầu nhìn thẳng vào role để biết quyền
✅ Đội ngũ chưa có kỷ luật về tagging (ABAC dễ fail nếu quên tag)
```

### Kết Hợp ABAC + RBAC (Best of Both Worlds)

```
Thực tế: Hầu hết tổ chức dùng hybrid approach:

RBAC cho:
  - Base permissions (permissions cơ bản mọi developer đều có)
  - Shared services (monitoring, logging, billing)
  - Cross-account access

ABAC cho:
  - Team-specific resources (S3, EC2, RDS của từng team)
  - Environment isolation
  - Developer self-service resources
```

---

## 8. Best Practices

### 1. Chuẩn Hóa Tag Keys Toàn Tổ Chức

```bash
# Dùng AWS Organizations Tag Policy để enforce tag keys
aws organizations create-policy \
  --name "RequiredTagKeys" \
  --type TAG_POLICY \
  --content '{
    "tags": {
      "team": {
        "tag_key": {
          "@@assign": "team"
        },
        "enforced_for": {
          "@@assign": ["s3:bucket", "ec2:instance", "rds:db"]
        }
      }
    }
  }'
```

### 2. Kết Hợp Permission Boundary Với ABAC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*", "ec2:*", "rds:*"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Validate ABAC Với IAM Policy Simulator

```bash
# Test ABAC policy
aws iam simulate-principal-policy \
  --policy-source-arn "arn:aws:iam::123456789012:role/developer-role" \
  --action-names "s3:GetObject" \
  --resource-arns "arn:aws:s3:::platform-dev-bucket/file.txt" \
  --context-entries '[
    {
      "ContextKeyName": "aws:PrincipalTag/team",
      "ContextKeyValues": ["platform"],
      "ContextKeyType": "string"
    },
    {
      "ContextKeyName": "aws:ResourceTag/team",
      "ContextKeyValues": ["platform"],
      "ContextKeyType": "string"
    }
  ]'
```

### 4. Monitoring ABAC With CloudTrail

```bash
# Tìm các lần truy cập bị từ chối do ABAC mismatch
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject \
  --query 'Events[?ErrorCode==`AccessDenied`]'
```

---

## 9. Pitfalls — Bẫy Phổ Biến

### Pitfall 1: Resource Không Có Tag → Bị Từ Chối

```
Vấn đề:
  Policy: StringEquals aws:ResourceTag/team = ${aws:PrincipalTag/team}
  S3 bucket KHÔNG có tag "team"
  → Condition evaluation: tag key không tồn tại → trả về NULL
  → StringEquals với NULL → FALSE → DENY!

Giải pháp:
  Option A: Dùng điều kiện Null check:
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
    },
    "Null": {
      "aws:ResourceTag/team": "false"  ← Bắt buộc resource phải có tag
    }
  }

  Option B: Tách policy — ABAC chỉ áp dụng cho tagged resources,
            có policy riêng cho shared/untagged resources.
```

### Pitfall 2: Principal Không Có Tag → Mất Quyền

```
Vấn đề:
  User mới chưa được gắn tag "team"
  → aws:PrincipalTag/team = NULL
  → So sánh NULL với resource tag → FALSE → DENY toàn bộ

Giải pháp:
  → Đảm bảo onboarding process luôn gắn tag cho user/role mới
  → Dùng IAM Access Analyzer để detect principals thiếu tag
  → Có fallback: permission nhỏ không cần ABAC cho các thao tác đọc cơ bản
```

### Pitfall 3: Case Sensitivity (Phân Biệt Chữ Hoa Thường)

```
Vấn đề:
  Tag value "Platform" (P viết hoa) vs "platform" (chữ thường)
  → StringEquals phân biệt hoa thường → DENY!

Giải pháp:
  Dùng StringEqualsIgnoreCase thay vì StringEquals:
  "Condition": {
    "StringEqualsIgnoreCase": {
      "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
    }
  }

  Hoặc: Enforce lowercase trong tag policy của Organization.
```

### Pitfall 4: Services Không Hỗ Trợ `aws:ResourceTag`

```
Vấn đề:
  Một số services (CloudWatch Logs, Route53) không hỗ trợ
  aws:ResourceTag condition key → ABAC không hoạt động.

Giải pháp:
  → Kiểm tra IAM documentation cho service cụ thể
  → Dùng ARN pattern matching cho services không hỗ trợ tag
  → Kết hợp RBAC cho những services này
```

### Pitfall 5: Tag Injection Attack (Tấn Công Chèn Tag)

```
Vấn đề:
  Nếu user có quyền iam:TagUser hoặc iam:TagRole với chính họ
  → Họ có thể tự thêm tag "team=admin" → leo thang đặc quyền!

Giải pháp:
  → KHÔNG bao giờ cho user/role tự gán tag vào chính mình
  → Dùng SCPs để prevent self-tagging
  → Chỉ automation (pipeline, IaC) mới được phép tag identities
```

---

## 10. Lab Exercise

### Lab: Team Isolation Với S3 và ABAC

**Mục tiêu:** Tạo môi trường ABAC đơn giản với 2 team, 2 S3 bucket, 1 policy duy nhất.

```bash
# ============================================
# BƯỚC 1: Tạo S3 buckets với tags cho từng team
# ============================================
aws s3 mb s3://abac-lab-team-alpha-$(date +%s)
BUCKET_ALPHA="abac-lab-team-alpha-$(date +%s)"

aws s3 mb s3://abac-lab-team-beta-$(date +%s)
BUCKET_BETA="abac-lab-team-beta-$(date +%s)"

# Tag buckets
aws s3api put-bucket-tagging \
  --bucket $BUCKET_ALPHA \
  --tagging 'TagSet=[{Key=team,Value=alpha},{Key=env,Value=dev}]'

aws s3api put-bucket-tagging \
  --bucket $BUCKET_BETA \
  --tagging 'TagSet=[{Key=team,Value=beta},{Key=env,Value=dev}]'

# ============================================
# BƯỚC 2: Tạo ABAC policy
# ============================================
cat > /tmp/abac-s3-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ByTeamTag",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::*", "arn:aws:s3:::*/*"],
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
        }
      }
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ABACTeamS3Policy \
  --policy-document file:///tmp/abac-s3-policy.json

# ============================================
# BƯỚC 3: Tạo role và gắn policy + tag
# ============================================
# Trust policy cho EC2 hoặc Lambda test
cat > /tmp/trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name abac-alpha-developer \
  --assume-role-policy-document file:///tmp/trust-policy.json

aws iam tag-role \
  --role-name abac-alpha-developer \
  --tags Key=team,Value=alpha

aws iam attach-role-policy \
  --role-name abac-alpha-developer \
  --policy-arn "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/ABACTeamS3Policy"

# ============================================
# BƯỚC 4: Test (chạy với credentials của role)
# ============================================
# Role alpha chỉ truy cập bucket alpha
aws s3 ls s3://$BUCKET_ALPHA    # → Thành công ✅
aws s3 ls s3://$BUCKET_BETA     # → AccessDenied ✅ (đúng như mong đợi)

# ============================================
# BƯỚC 5: Dọn dẹp
# ============================================
aws iam detach-role-policy --role-name abac-alpha-developer \
  --policy-arn "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/ABACTeamS3Policy"
aws iam delete-role --role-name abac-alpha-developer
aws iam delete-policy --policy-arn "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/ABACTeamS3Policy"
aws s3 rb s3://$BUCKET_ALPHA --force
aws s3 rb s3://$BUCKET_BETA --force
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q1: ABAC khác RBAC như thế nào? Khi nào bạn chọn ABAC?**

> RBAC gắn quyền vào role/group cụ thể và liệt kê ARN trong policy. ABAC sử dụng condition keys để so sánh thuộc tính (tags) của principal với tags của resource. Tôi chọn ABAC khi tổ chức có nhiều team × nhiều môi trường — vì ABAC giúp giảm số lượng policy cần quản lý và tự động scale khi thêm team mới.

**Q2: `aws:PrincipalTag`, `aws:ResourceTag`, `aws:RequestTag` khác nhau thế nào?**

> `aws:PrincipalTag/X` là tag gắn vào principal (user/role/session) đang gửi request. `aws:ResourceTag/X` là tag của resource đang bị truy cập. `aws:RequestTag/X` là tag được gửi kèm trong request (dùng để enforce tagging khi tạo resource). ABAC cơ bản so sánh PrincipalTag với ResourceTag.

**Q3: Session tags hoạt động như thế nào và khi nào dùng?**

> Session tags được truyền vào khi gọi `sts:AssumeRole` hoặc `AssumeRoleWithSAML/WebIdentity`. Chúng ghi đè hoặc bổ sung tag cho phiên đó và được nhận diện qua `aws:PrincipalTag`. Session tags hữu ích nhất khi dùng với IAM Identity Center hoặc SAML federation — attributes từ IdP (như department, team trong Okta) được map thành session tags.

**Q4: Điều gì xảy ra nếu resource không có tag và policy dùng ABAC?**

> Nếu resource không có tag key được kiểm tra, condition key trả về NULL. `StringEquals` với NULL sẽ trả về false → request bị DENY. Đây là "pitfall" phổ biến nhất của ABAC. Giải pháp là dùng `Null` condition để bắt buộc resource phải có tag, hoặc có policy riêng cho shared/untagged resources.

**Q5: Tại sao ABAC có thể gây ra leo thang đặc quyền (privilege escalation)?**

> Nếu một principal có quyền tự gắn tag vào chính mình (`iam:TagUser`, `iam:TagRole` cho principal của mình), họ có thể thêm tag như `team=admin` và đột nhiên có quyền truy cập resource của admin. Cần ngăn chặn bằng SCPs hoặc policy deny tự-tag.

**Q6: Làm thế nào để ABAC hoạt động với IAM Identity Center?**

> Trong IAM Identity Center, bật "Attributes for access control", sau đó map attributes từ identity source (Okta, Azure AD) sang session attribute keys. Các attributes này trở thành session tags khi user đăng nhập. Permission Sets có thể dùng `aws:PrincipalTag` để implement ABAC mà không cần nhiều permission set khác nhau.

---

## 12. Key Takeaways

> **ABAC giải quyết "role explosion"** — khi tổ chức có nhiều team × môi trường, số lượng roles trong RBAC tăng theo cấp số nhân. ABAC giữ số lượng policies ổn định dù tổ chức phát triển.

> **3 condition keys cốt lõi:** `aws:PrincipalTag` (người gửi), `aws:ResourceTag` (resource bị truy cập), `aws:RequestTag` (tag trong request tạo mới).

> **Session tags** là cầu nối giữa IdP (Okta, Azure AD) và ABAC — attributes của user trong IdP trở thành session tags, cho phép ABAC hoạt động seamlessly với SSO.

> **Tag discipline là bắt buộc** — ABAC hoàn toàn thất bại nếu resources hoặc principals không có tag. Cần kết hợp với Organizations Tag Policy và tagging enforcement trong IaC pipeline.

> **Không có "winner"** — ABAC và RBAC phục vụ mục đích khác nhau. Hầu hết tổ chức trưởng thành dùng hybrid: RBAC cho base permissions và cross-account, ABAC cho team/env isolation.
