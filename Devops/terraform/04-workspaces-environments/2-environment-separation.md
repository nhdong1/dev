# Environment Separation — Phân Tách Môi Trường

> Chiến lược phân tách môi trường là nền tảng cho việc vận hành Terraform an toàn trong team. Làm đúng, bạn ngủ ngon mỗi đêm. Làm sai, bạn có thể deploy nhầm vào production.

---

## 📚 Mục Lục

1. [Tại sao cần phân tách môi trường?](#tại-sao-cần-phân-tách-môi-trường)
2. [Các chiến lược phân tách](#các-chiến-lược-phân-tách)
3. [Directory Separation — Phân Tách Thư Mục](#directory-separation-phân-tách-thư-mục)
4. [Account Separation — Phân Tách Tài Khoản Cloud](#account-separation-phân-tách-tài-khoản-cloud)
5. [Cấu trúc thư mục thực tế](#cấu-trúc-thư-mục-thực-tế)
6. [Chiến lược promotion — Thăng hạng code](#chiến-lược-promotion-thăng-hạng-code)
7. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Tại sao cần phân tách môi trường?

### Vấn đề thực tế

```
Tình huống: Team dùng chung một Terraform state cho tất cả môi trường.

Hậu quả:
- Developer test thay đổi ở dev → vô tình ảnh hưởng prod resources
- Không thể biết ai thay đổi gì ở môi trường nào
- Database prod bị xóa vì nghĩ đang ở dev
```

### Mục tiêu của phân tách môi trường

1. **Cách ly rủi ro** — Risk Isolation: Lỗi ở dev không ảnh hưởng prod
2. **Kiểm soát thay đổi** — Change Control: Thay đổi phải qua dev → staging → prod
3. **Cách ly dữ liệu** — Data Isolation: Data prod không bị truy cập từ dev
4. **Kiểm soát chi phí** — Cost Control: Dễ theo dõi chi phí từng môi trường
5. **Compliance** — Tuân thủ: Yêu cầu pháp lý thường yêu cầu cách ly môi trường

---

## Các chiến lược phân tách

### Chiến lược 1: Workspaces (Ít khuyến nghị)

```
Cùng account → Cùng backend → Workspace khác nhau
```

Xem [1-workspaces.md](./1-workspaces.md) để hiểu chi tiết và giới hạn.

---

### Chiến lược 2: Directory Separation — Phân Tách Thư Mục (Phổ biến)

```
Cùng hoặc khác account → Backend riêng → Thư mục riêng
```

**Cấu trúc:**
```
infra/
├── modules/          ← Code dùng chung
└── environments/
    ├── dev/          ← cd vào đây để làm việc với dev
    ├── staging/      ← cd vào đây để làm việc với staging
    └── prod/         ← cd vào đây để làm việc với prod
```

---

### Chiến lược 3: Account Separation — Phân Tách Tài Khoản (Bảo mật nhất)

```
Account dev     → Backend trong account dev
Account staging → Backend trong account staging  
Account prod    → Backend trong account prod
```

**Cấu trúc AWS Organizations — Tổ Chức AWS:**
```
AWS Organization Root
├── Management Account        ← Chỉ billing, không deploy
├── Development OU            ← Organizational Unit — Đơn Vị Tổ Chức
│   └── dev-account
├── Staging OU
│   └── staging-account
└── Production OU
    └── prod-account
```

---

## Directory Separation — Phân Tách Thư Mục

### Cấu trúc chi tiết

```
infra/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
└── environments/
    ├── dev/
    │   ├── main.tf              ← Gọi modules với config dev
    │   ├── backend.tf           ← Backend config cho dev
    │   ├── variables.tf         ← Khai báo variables
    │   ├── outputs.tf
    │   └── terraform.tfvars     ← Giá trị cụ thể cho dev
    │
    ├── staging/
    │   ├── main.tf              ← Cùng structure, khác giá trị
    │   ├── backend.tf           ← Backend riêng cho staging
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── terraform.tfvars
    │
    └── prod/
        ├── main.tf
        ├── backend.tf           ← Backend riêng cho prod
        ├── variables.tf
        ├── outputs.tf
        └── terraform.tfvars     ← KHÔNG commit file này nếu có secrets!
```

---

### Ví dụ code

**`environments/dev/main.tf`:**
```hcl
# Gọi module networking với cấu hình dev
module "networking" {
  source = "../../modules/networking"

  environment      = "dev"
  vpc_cidr         = "10.0.0.0/16"
  azs              = ["us-east-1a", "us-east-1b"]
  private_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets   = ["10.0.101.0/24", "10.0.102.0/24"]
}

module "compute" {
  source = "../../modules/compute"

  environment   = "dev"
  instance_type = "t3.micro"    # Nhỏ hơn để tiết kiệm chi phí
  min_size      = 1
  max_size      = 2
  vpc_id        = module.networking.vpc_id
  subnet_ids    = module.networking.private_subnet_ids
}

module "database" {
  source = "../../modules/database"

  environment      = "dev"
  instance_class   = "db.t3.small"
  multi_az         = false       # Không cần Multi-AZ ở dev
  deletion_protection = false    # Cho phép xóa ở dev
  subnet_ids       = module.networking.private_subnet_ids
}
```

**`environments/prod/main.tf`:**
```hcl
# Cùng modules, khác cấu hình
module "networking" {
  source = "../../modules/networking"

  environment      = "prod"
  vpc_cidr         = "10.2.0.0/16"       # CIDR — Classless Inter-Domain Routing — khác
  azs              = ["us-east-1a", "us-east-1b", "us-east-1c"]  # 3 AZ cho HA — High Availability
  private_subnets  = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
  public_subnets   = ["10.2.101.0/24", "10.2.102.0/24", "10.2.103.0/24"]
}

module "compute" {
  source = "../../modules/compute"

  environment   = "prod"
  instance_type = "t3.large"    # Lớn hơn cho production workload
  min_size      = 3             # Tối thiểu 3 instances cho HA
  max_size      = 10
  vpc_id        = module.networking.vpc_id
  subnet_ids    = module.networking.private_subnet_ids
}

module "database" {
  source = "../../modules/database"

  environment         = "prod"
  instance_class      = "db.r5.large"
  multi_az            = true       # Multi-AZ bắt buộc cho production
  deletion_protection = true       # Bảo vệ không bị xóa nhầm
  subnet_ids          = module.networking.private_subnet_ids
}
```

**`environments/dev/backend.tf`:**
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-dev"
    key            = "environments/dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-dev"
    encrypt        = true
  }
}
```

**`environments/prod/backend.tf`:**
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"    # Bucket khác hoàn toàn!
    key            = "environments/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-prod"
    encrypt        = true
    
    # Dùng role riêng cho prod với permissions hạn chế hơn
    role_arn       = "arn:aws:iam::PROD_ACCOUNT_ID:role/terraform-backend-role"
  }
}
```

---

## Account Separation — Phân Tách Tài Khoản Cloud

### Tại sao cần account riêng?

Directory separation vẫn có thể dùng cùng một AWS account. Rủi ro:
- Developers có thể vô tình truy cập S3 bucket của prod
- Security group cho dev có thể expose prod network
- Billing không rõ ràng theo môi trường

**Account separation** — mỗi môi trường ở AWS account hoàn toàn riêng biệt — là giải pháp bảo mật nhất.

### Cấu hình với AWS Organizations

**Provider configuration — Cấu hình Provider:**
```hcl
# environments/prod/providers.tf
provider "aws" {
  region = "us-east-1"
  
  # Assume role — Đóng vai — trong production account
  assume_role {
    role_arn     = "arn:aws:iam::111222333444:role/TerraformDeployRole"
    session_name = "terraform-prod-deploy"
  }
  
  # Đảm bảo chắc chắn đang ở đúng account
  allowed_account_ids = ["111222333444"]   # Prod account ID
}
```

```hcl
# environments/dev/providers.tf
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::999888777666:role/TerraformDeployRole"
    session_name = "terraform-dev-deploy"
  }

  allowed_account_ids = ["999888777666"]   # Dev account ID — khác hoàn toàn
}
```

### Quy trình làm việc với Account Separation

```bash
# Cấu hình AWS profiles cho từng account
# ~/.aws/config

[profile dev]
role_arn = arn:aws:iam::999888777666:role/TerraformDeployRole
source_profile = management

[profile prod]
role_arn = arn:aws:iam::111222333444:role/TerraformDeployRole
source_profile = management
mfa_serial = arn:aws:iam::MGMT_ACCOUNT:mfa/username   # MFA — Multi-Factor Authentication — bắt buộc cho prod!

# Làm việc với dev
cd environments/dev
AWS_PROFILE=dev terraform apply

# Làm việc với prod (cần MFA)
cd environments/prod
AWS_PROFILE=prod terraform plan   # Phải nhập MFA code
```

---

## Cấu trúc thư mục thực tế

### Cấu trúc cho startup — công ty khởi nghiệp nhỏ

```
infra/
├── modules/
│   ├── vpc/
│   ├── eks/         ← EKS — Elastic Kubernetes Service — Kubernetes trên AWS
│   └── rds/         ← RDS — Relational Database Service — Cơ Sở Dữ Liệu Quan Hệ
└── environments/
    ├── dev/
    └── prod/
```

### Cấu trúc cho công ty trung bình

```
infra/
├── modules/
│   ├── networking/
│   ├── security/
│   ├── compute/
│   └── data/
└── environments/
    ├── dev/
    ├── staging/
    ├── qa/          ← QA — Quality Assurance — Kiểm Tra Chất Lượng
    └── prod/
```

### Cấu trúc enterprise — doanh nghiệp lớn

```
infra/
├── modules/
│   ├── base-networking/
│   ├── security-baseline/
│   └── application/
├── accounts/
│   ├── management/       ← AWS Organizations management
│   ├── shared-services/  ← Dịch vụ dùng chung (monitoring, logging)
│   ├── dev/
│   │   ├── us-east-1/
│   │   └── eu-west-1/
│   ├── staging/
│   └── prod/
│       ├── us-east-1/
│       └── eu-west-1/
└── global/               ← Route53, IAM, CloudFront (global resources — tài nguyên toàn cầu)
```

---

## Chiến lược promotion — Thăng hạng code

**Promotion** — Thăng Hạng — là quá trình đưa thay đổi từ môi trường thấp lên cao hơn.

### Quy trình chuẩn

```
1. Developer thay đổi code ở feature branch
2. PR (Pull Request — Yêu Cầu Tích Hợp Code) → review → merge vào main
3. CI/CD tự động apply vào dev
4. QA test ở dev
5. Tạo tag/release → tự động apply vào staging
6. PO (Product Owner — Chủ Sản Phẩm) approve → manual apply vào prod
```

### Ví dụ GitHub Actions workflow — Quy Trình Tự Động

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
    paths: ['infra/**']

jobs:
  deploy-dev:
    name: Deploy to Dev
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Terraform Apply Dev
        working-directory: infra/environments/dev
        run: |
          terraform init
          terraform apply -auto-approve
        env:
          AWS_PROFILE: dev

  deploy-staging:
    name: Deploy to Staging
    needs: deploy-dev           # Chỉ chạy sau khi dev thành công
    runs-on: ubuntu-latest
    steps:
      - name: Terraform Apply Staging
        working-directory: infra/environments/staging
        run: terraform apply -auto-approve
        env:
          AWS_PROFILE: staging

  deploy-prod:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production     # Yêu cầu manual approval — Phê duyệt thủ công
    steps:
      - name: Terraform Apply Prod
        working-directory: infra/environments/prod
        run: terraform apply -auto-approve
        env:
          AWS_PROFILE: prod
```

---

## Câu hỏi phỏng vấn

### Q1: Mô tả chiến lược quản lý đa môi trường của bạn

**Trả lời mẫu (STAR — Situation Task Action Result):**

> "Tại công ty trước, chúng tôi có 3 môi trường: dev, staging, prod. Ban đầu dùng workspaces nhưng gặp vấn đề khi developer apply nhầm vào staging.
>
> Tôi đề xuất chuyển sang directory separation với backend riêng biệt cho từng môi trường. Mỗi môi trường có:
> - Thư mục riêng trong `environments/`
> - S3 bucket và DynamoDB table riêng cho state locking
> - AWS role với permissions phù hợp
> - CI/CD pipeline với manual approval cho production
>
> Kết quả: Không còn incident nào do apply nhầm môi trường, và audit trail — Nhật Ký Kiểm Tra — rõ ràng hơn."

---

### Q2: Khi nào nên dùng separate accounts vs separate directories?

**Trả lời:**

| Tình Huống | Khuyến Nghị |
|------------|-------------|
| Startup nhỏ, team < 5 người | Separate directories, cùng account |
| Có yêu cầu compliance (PCI-DSS, HIPAA, SOC2) | Separate accounts bắt buộc |
| Team > 10 người, nhiều môi trường | Separate accounts |
| Chi phí là ưu tiên hàng đầu | Separate directories |
| Cần cách ly network hoàn toàn | Separate accounts + VPC |

PCI-DSS — Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Thẻ Thanh Toán.
HIPAA — Health Insurance Portability and Accountability Act — Đạo Luật Bảo Mật Thông Tin Y Tế.

---

### Q3: Làm sao đảm bảo cấu hình nhất quán giữa các môi trường?

**Trả lời:**

```hcl
# 1. Dùng cùng module version cho tất cả môi trường
module "networking" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"   # Cùng version ở tất cả môi trường!
}

# 2. Dùng shared variables file
# environments/shared.tfvars
# (import vào tất cả môi trường)

# 3. Dùng Terragrunt để giảm lặp code
# (xem 4-terragrunt-intro.md)

# 4. Automated testing — Kiểm thử tự động — để phát hiện drift
# Chạy terraform plan định kỳ để phát hiện cấu hình lệch
```

---

## 💡 Best Practices — Thực Hành Tốt Nhất

1. **Bắt đầu với directory separation** — dễ hiểu, ít rủi ro nhất
2. **Backend riêng cho mỗi môi trường** — không bao giờ dùng chung state bucket
3. **Principle of Least Privilege** — Nguyên Tắc Đặc Quyền Tối Thiểu — cho từng môi trường
4. **Production phải có manual approval** — không bao giờ auto-apply vào prod
5. **Đặt `deletion_protection = true`** cho database production
6. **Dùng `allowed_account_ids`** để đảm bảo đang apply vào đúng AWS account
7. **Thêm environment tag vào tất cả tài nguyên** để dễ quản lý chi phí

---

**Tiếp theo:** [3-tfvars-management.md](./3-tfvars-management.md) — Quản lý `.tfvars` files
