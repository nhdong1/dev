# 2 — IAM Roles — Phân Quyền Tối Thiểu Cho Terraform

> IAM — Identity and Access Management — Quản Lý Danh Tính và Truy Cập. Quy tắc vàng: **chỉ cấp đúng những gì cần thiết, không hơn.**

---

## 🎯 Nguyên Tắc Least Privilege — Đặc Quyền Tối Thiểu

Least Privilege nghĩa là một entity (user, role, service) chỉ được cấp đúng những quyền cần thiết để thực hiện công việc của nó, không có thêm bất kỳ quyền nào khác.

**Tại sao quan trọng với Terraform?**

Terraform CI/CD runner thường chạy với quyền rất rộng. Nếu bị xâm phạm:
```
Attacker → Terraform Role → Tạo IAM admin user → Chiếm toàn bộ AWS account
```

---

## 🏗️ Kiến Trúc IAM Cho Terraform

### Mô Hình Phân Tầng Cho Nhiều Môi Trường

```
┌──────────────────────────────────────────────────────────┐
│                   GitHub Actions / GitLab CI              │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │          OIDC — OpenID Connect — Federation         │ │
│  │   (Không cần lưu credentials dài hạn)               │ │
│  └──────────────────────┬──────────────────────────────┘ │
└─────────────────────────┼────────────────────────────────┘
                          │ AssumeRole
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │  Dev Role   │ │Staging Role │ │  Prod Role  │
   │ (rộng hơn) │ │  (trung)    │ │ (chặt nhất)│
   └─────────────┘ └─────────────┘ └─────────────┘
```

---

## 🔐 Thiết Lập OIDC Authentication — Xác Thực Không Cần Secret

OIDC — OpenID Connect — cho phép GitHub Actions xác thực với AWS mà không cần lưu AWS credentials.

### Bước 1: Tạo OIDC Identity Provider Trong AWS

```hcl
# iam-oidc.tf
resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = [
    "sts.amazonaws.com"
  ]

  # Thumbprint của GitHub OIDC certificate
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]

  tags = {
    Name        = "github-actions-oidc"
    ManagedBy   = "terraform"
  }
}
```

### Bước 2: Tạo IAM Role Với Trust Policy

```hcl
# terraform-ci-role.tf

# Trust policy — Chính sách tin tưởng
data "aws_iam_policy_document" "terraform_ci_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github_actions.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    # Chỉ cho phép repo cụ thể, branch cụ thể
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      # Chỉ cho phép từ repo myorg/myrepo, branch main
      values   = ["repo:myorg/myrepo:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "terraform_ci" {
  name               = "terraform-ci-prod-role"
  assume_role_policy = data.aws_iam_policy_document.terraform_ci_trust.json

  # Timeout ngắn — session tồn tại tối đa 1 giờ
  max_session_duration = 3600

  tags = {
    Environment = "production"
    Purpose     = "terraform-ci-cd"
    ManagedBy   = "terraform"
  }
}
```

### Bước 3: Tạo Policy Với Least Privilege

```hcl
# Ví dụ: Policy cho team quản lý VPC và EC2
data "aws_iam_policy_document" "terraform_ci_permissions" {
  # Quyền VPC — Virtual Private Cloud
  statement {
    sid    = "VPCManagement"
    effect = "Allow"
    actions = [
      "ec2:CreateVpc",
      "ec2:DeleteVpc",
      "ec2:DescribeVpcs",
      "ec2:ModifyVpcAttribute",
      "ec2:CreateSubnet",
      "ec2:DeleteSubnet",
      "ec2:DescribeSubnets",
      "ec2:CreateInternetGateway",
      "ec2:DeleteInternetGateway",
      "ec2:AttachInternetGateway",
      "ec2:DetachInternetGateway",
      "ec2:CreateRouteTable",
      "ec2:DeleteRouteTable",
      "ec2:CreateRoute",
      "ec2:AssociateRouteTable"
    ]
    resources = ["*"]

    # Giới hạn theo region — điều kiện bổ sung
    condition {
      test     = "StringEquals"
      variable = "aws:RequestedRegion"
      values   = ["us-east-1", "ap-southeast-1"]
    }
  }

  # Quyền S3 chỉ cho bucket cụ thể
  statement {
    sid    = "S3StateBucket"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket"
    ]
    resources = [
      "arn:aws:s3:::my-terraform-state-bucket",
      "arn:aws:s3:::my-terraform-state-bucket/*"
    ]
  }

  # DynamoDB cho state locking
  statement {
    sid    = "DynamoDBStateLock"
    effect = "Allow"
    actions = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:DeleteItem"
    ]
    resources = [
      "arn:aws:dynamodb:us-east-1:123456789:table/terraform-state-lock"
    ]
  }

  # Từ chối tường minh — Explicit Deny — nguy hiểm nhất
  statement {
    sid    = "DenyDangerousActions"
    effect = "Deny"
    actions = [
      "iam:CreateUser",
      "iam:AttachUserPolicy",
      "iam:CreateAccessKey",
      "organizations:*",
      "account:*"
    ]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "terraform_ci" {
  name        = "terraform-ci-prod-policy"
  description = "Minimal permissions for Terraform CI/CD in production"
  policy      = data.aws_iam_policy_document.terraform_ci_permissions.json
}

resource "aws_iam_role_policy_attachment" "terraform_ci" {
  role       = aws_iam_role.terraform_ci.name
  policy_arn = aws_iam_policy.terraform_ci.arn
}
```

---

## 🔍 Kỹ Thuật Tìm Ra Quyền Tối Thiểu Cần Thiết

### Phương Pháp 1: IAM Access Analyzer — Công Cụ Phân Tích Truy Cập

```bash
# Bật Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name terraform-ci-analyzer \
  --type ACCOUNT

# Sau khi chạy Terraform một thời gian, xem quyền thực sự được dùng
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:accessanalyzer:us-east-1:123456789:analyzer/terraform-ci-analyzer
```

### Phương Pháp 2: CloudTrail + IAM Policy Generator

```bash
# 1. Chạy Terraform với overprivileged role trước
# 2. Lọc CloudTrail events trong 7 ngày gần nhất
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=terraform-ci-role \
  --start-time $(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[*].{Action:CloudTrailEvent}' | \
  jq -r '.[].Action' | \
  jq -r '.eventName' | sort -u

# 3. Dùng danh sách này để tạo policy tối thiểu
```

### Phương Pháp 3: Dùng iamlive — Công Cụ Theo Dõi API Calls

```bash
# Cài iamlive
brew install iann0036/iamlive/iamlive  # macOS

# Bật proxy mode để capture API calls
iamlive --set-ini --output-file policy.json

# Chạy Terraform trong môi trường có proxy này
HTTP_PROXY=http://127.0.0.1:10080 \
HTTPS_PROXY=http://127.0.0.1:10080 \
terraform apply -auto-approve

# Xem policy được generate
cat policy.json
```

---

## 🌍 IAM Role Theo Môi Trường

### Dev Environment — Môi Trường Phát Triển

```hcl
# dev-role.tf — Rộng hơn để developer dễ iterate
data "aws_iam_policy_document" "terraform_dev_permissions" {
  statement {
    effect = "Allow"
    actions = [
      "ec2:*",
      "rds:*",
      "s3:*",
      "iam:GetRole",          # Đọc được nhưng không tạo/sửa role
      "iam:GetPolicy",
      "iam:ListRoles"
    ]
    resources = ["*"]

    # Chỉ trong dev account
    condition {
      test     = "StringEquals"
      variable = "aws:RequestedRegion"
      values   = ["us-east-1"]
    }
  }

  # Vẫn deny tạo IAM users/keys ngay cả ở dev
  statement {
    effect = "Deny"
    actions = [
      "iam:CreateUser",
      "iam:CreateAccessKey",
      "iam:AttachUserPolicy"
    ]
    resources = ["*"]
  }
}
```

### Production Environment — Môi Trường Sản Xuất

```hcl
# prod-role.tf — Chặt chẽ, chỉ đủ để apply Terraform
data "aws_iam_policy_document" "terraform_prod_permissions" {
  # Chỉ các actions cần thiết, không dùng wildcard (*) ở actions
  statement {
    sid    = "EC2ReadOnly"
    effect = "Allow"
    actions = [
      "ec2:Describe*"   # Chỉ đọc — wildcard chỉ cho Describe
    ]
    resources = ["*"]
  }

  statement {
    sid    = "EC2Specific"
    effect = "Allow"
    actions = [
      "ec2:RunInstances",
      "ec2:TerminateInstances",
      "ec2:ModifyInstanceAttribute"
    ]
    # Chỉ instance trong region cụ thể
    resources = [
      "arn:aws:ec2:us-east-1:123456789:instance/*"
    ]

    # Phải có tag Environment=production
    condition {
      test     = "StringEquals"
      variable = "aws:ResourceTag/Environment"
      values   = ["production"]
    }
  }

  # Explicit deny mọi delete action trong prod — yêu cầu manual confirmation
  statement {
    sid    = "DenyDeleteWithoutApproval"
    effect = "Deny"
    actions = [
      "rds:DeleteDBInstance",
      "dynamodb:DeleteTable",
      "s3:DeleteBucket"
    ]
    resources = ["*"]

    # Trừ khi có specific condition tag được set
    condition {
      test     = "StringNotEquals"
      variable = "aws:RequestTag/ApprovedForDeletion"
      values   = ["true"]
    }
  }
}
```

---

## 📋 Permission Boundary — Ranh Giới Quyền

Permission Boundary giới hạn tối đa quyền mà một role có thể có, kể cả khi có policy khác cố tình mở rộng.

```hcl
# Tạo permission boundary
resource "aws_iam_policy" "terraform_boundary" {
  name        = "TerraformPermissionBoundary"
  description = "Maximum permissions any Terraform-created role can have"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:*",
          "s3:*",
          "rds:*",
          "cloudwatch:*",
          "logs:*"
        ]
        Resource = "*"
      },
      # Tuyệt đối không được phép kể cả với boundary
      {
        Effect = "Deny"
        Action = [
          "iam:*",
          "organizations:*",
          "account:*",
          "billing:*"
        ]
        Resource = "*"
      }
    ]
  })
}

# Áp dụng boundary khi tạo role trong Terraform
resource "aws_iam_role" "app_role" {
  name                 = "my-app-role"
  assume_role_policy   = data.aws_iam_policy_document.app_trust.json
  permissions_boundary = aws_iam_policy.terraform_boundary.arn
}
```

---

## 🔄 GitHub Actions Workflow Dùng OIDC

```yaml
# .github/workflows/terraform.yml
name: Terraform Apply

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Cần để lấy OIDC token
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Xác thực với AWS qua OIDC — không cần lưu credentials
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/terraform-ci-prod-role
          aws-region: us-east-1
          role-session-name: GitHubActions-${{ github.run_id }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        run: terraform apply tfplan
```

---

## 📊 So Sánh Các Phương Pháp Xác Thực

| Phương Pháp              | Bảo Mật | Độ Phức Tạp | Khuyến Nghị     |
| ------------------------ | ------- | ----------- | --------------- |
| OIDC (GitHub/GitLab)     | ⭐⭐⭐⭐⭐ | Trung bình  | ✅ Dùng cho CI/CD |
| IAM Role (EC2/ECS)       | ⭐⭐⭐⭐⭐ | Thấp        | ✅ Dùng khi chạy trên AWS |
| Long-term Access Keys    | ⭐⭐      | Thấp        | ❌ Tránh dùng   |
| Vault AWS Auth Method    | ⭐⭐⭐⭐⭐ | Cao         | ✅ Khi đã dùng Vault |
| Instance Profile         | ⭐⭐⭐⭐⭐ | Thấp        | ✅ Self-hosted runners |

---

## ✅ Checklist IAM Best Practices

- [ ] Không dùng long-term access keys trong CI/CD
- [ ] Dùng OIDC cho GitHub Actions / GitLab CI
- [ ] Mỗi môi trường có role riêng (dev, staging, prod)
- [ ] Role prod bị giới hạn chặt nhất
- [ ] Không có `"Action": "*"` hay `"Resource": "*"` kết hợp nhau
- [ ] Có explicit deny cho các actions nguy hiểm
- [ ] Permission boundary được áp dụng
- [ ] IAM Access Analyzer đang bật và monitor
- [ ] Credentials review định kỳ (90 ngày)
- [ ] MFA — Multi-Factor Authentication — bật cho human users

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao không nên dùng AdministratorAccess cho Terraform CI/CD?**
> Nếu CI/CD bị xâm phạm, attacker có full control. Dùng Least Privilege: chỉ cấp quyền cho resources mà Terraform code cụ thể đó quản lý. Thêm explicit deny cho các actions đặc biệt nguy hiểm như tạo IAM user, xóa database.

**Q: OIDC authentication hoạt động như thế nào với GitHub Actions?**
> GitHub Actions lấy một JWT token từ GitHub OIDC provider. AWS trust policy được cấu hình để tin tưởng GitHub OIDC provider. CI/CD gọi `sts:AssumeRoleWithWebIdentity` với JWT đó để nhận temporary credentials. Không cần lưu AWS credentials dài hạn ở GitHub Secrets.

**Q: Permission Boundary là gì và khi nào dùng?**
> Permission Boundary là policy đặc biệt định nghĩa quyền tối đa một role có thể có. Dùng khi cần ngăn privilege escalation — leo thang đặc quyền: kể cả khi policy của role bị thay đổi, boundary vẫn giới hạn quyền thực tế.
