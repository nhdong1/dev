# 2 — Remote Backend — Lưu State Trên Dịch Vụ Tập Trung

> Remote backend — Backend từ xa — là nơi Terraform lưu state file trên dịch vụ tập trung thay vì máy cục bộ, cho phép nhiều người và pipeline CI/CD cùng chia sẻ state an toàn.

---

## 📚 Mục Lục

1. [Backend Là Gì?](#1-backend-là-gì)
2. [AWS S3 + DynamoDB — Backend Phổ Biến Nhất](#2-aws-s3--dynamodb--backend-phổ-biến-nhất)
3. [GCS — Google Cloud Storage Backend](#3-gcs--google-cloud-storage-backend)
4. [Azure Blob Storage Backend](#4-azure-blob-storage-backend)
5. [Terraform Cloud Backend](#5-terraform-cloud-backend)
6. [Partial Configuration — Cấu Hình Một Phần](#6-partial-configuration--cấu-hình-một-phần)
7. [terraform_remote_state — Đọc State Từ Project Khác](#7-terraform_remote_state--đọc-state-từ-project-khác)
8. [Migrate Backend — Chuyển Đổi Backend](#8-migrate-backend--chuyển-đổi-backend)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Backend Là Gì?

Backend trong Terraform quyết định **hai việc**:

1. **Lưu state ở đâu** — S3, GCS, Azure Blob, Terraform Cloud, etc.
2. **Chạy operations ở đâu** — Local machine hay remote (chỉ áp dụng với một số backend như Terraform Cloud)

```hcl
terraform {
  backend "s3" {   # ← Chọn loại backend
    # ... cấu hình
  }
}
```

**Các backend được dùng phổ biến nhất:**

| Backend | Tốt Cho | State Locking |
|---------|---------|---------------|
| `s3` (AWS) | AWS-based teams | DynamoDB (cần cấu hình thêm) |
| `gcs` (GCP) | GCP-based teams | Tự động (built-in) |
| `azurerm` (Azure) | Azure-based teams | Blob lease (tự động) |
| `remote` / `cloud` | Terraform Cloud / Enterprise | Tự động |
| `local` | Dev cá nhân, học tập | Không có |

---

## 2. AWS S3 + DynamoDB — Backend Phổ Biến Nhất

### Bước 1: Tạo S3 Bucket và DynamoDB Table

```hcl
# bootstrap/main.tf — Tạo trước bằng Terraform hoặc tay
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state-prod"

  # Ngăn xóa nhầm bucket chứa state
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"  # Bắt buộc — cho phép rollback state
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"  # Mã hoá at rest — khi lưu
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Bước 2: Cấu Hình Backend

```hcl
# main.tf — hoặc versions.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"
    key            = "prod/vpc/terraform.tfstate"   # Đường dẫn trong bucket
    region         = "us-east-1"
    encrypt        = true                            # Mã hoá in transit
    dynamodb_table = "terraform-state-locks"         # State Locking
  }
}
```

### Cấu Trúc Key Được Khuyến Nghị

```
s3://mycompany-terraform-state-prod/
├── global/
│   └── iam/terraform.tfstate
├── prod/
│   ├── vpc/terraform.tfstate
│   ├── eks/terraform.tfstate
│   └── rds/terraform.tfstate
├── staging/
│   ├── vpc/terraform.tfstate
│   └── eks/terraform.tfstate
└── dev/
    └── vpc/terraform.tfstate
```

### IAM Policy — Phân Quyền Cho Terraform

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::mycompany-terraform-state-prod/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::mycompany-terraform-state-prod"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/terraform-state-locks"
    }
  ]
}
```

---

## 3. GCS — Google Cloud Storage Backend

GCS — Google Cloud Storage — có State Locking tích hợp sẵn, không cần Firestore hay Bigtable thêm.

```hcl
terraform {
  backend "gcs" {
    bucket  = "mycompany-terraform-state"
    prefix  = "prod/vpc"   # Thư mục trong bucket
  }
}
```

**Tạo bucket:**

```bash
# Tạo bucket với versioning
gsutil mb -l us-central1 gs://mycompany-terraform-state
gsutil versioning set on gs://mycompany-terraform-state

# Phân quyền cho service account
gsutil iam ch serviceAccount:terraform@myproject.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://mycompany-terraform-state
```

**Lưu ý:** GCS backend tự động handle State Locking — không cần bước cấu hình thêm.

---

## 4. Azure Blob Storage Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "mycompany-terraform-rg"
    storage_account_name = "mycompanytfstate"
    container_name       = "tfstate"
    key                  = "prod.vpc.terraform.tfstate"
  }
}
```

**Tạo bằng Azure CLI:**

```bash
# Tạo resource group
az group create --name mycompany-terraform-rg --location eastus

# Tạo storage account
az storage account create \
  --name mycompanytfstate \
  --resource-group mycompany-terraform-rg \
  --location eastus \
  --sku Standard_LRS \
  --encryption-services blob

# Tạo container
az storage container create \
  --name tfstate \
  --account-name mycompanytfstate
```

**Lưu ý:** Azure Blob Storage dùng **blob lease** — thuê blob — để thực hiện State Locking tự động.

---

## 5. Terraform Cloud Backend

Terraform Cloud cung cấp remote state, locking, và remote execution — chạy plan/apply trên server của HashiCorp thay vì máy local.

```hcl
terraform {
  cloud {
    organization = "mycompany"
    workspaces {
      name = "prod-vpc"
      # Hoặc dùng tags để map nhiều workspace:
      # tags = ["prod", "vpc"]
    }
  }
}
```

**Đăng nhập:**

```bash
terraform login   # Mở browser để lấy token
```

**So sánh Terraform Cloud vs S3:**

| Tính Năng | S3 + DynamoDB | Terraform Cloud |
|-----------|---------------|-----------------|
| Remote state | ✅ | ✅ |
| State locking | ✅ (DynamoDB) | ✅ (tự động) |
| Remote execution | ❌ | ✅ |
| UI xem state | ❌ | ✅ |
| VCS integration | ❌ | ✅ (GitHub, GitLab) |
| Cost | Rất thấp (~$1/tháng) | Miễn phí ≤5 users |
| Tự kiểm soát hoàn toàn | ✅ | ❌ |

---

## 6. Partial Configuration — Cấu Hình Một Phần

Backend block không được dùng variables hay expressions — vì backend khởi tạo trước khi variables được load. Giải pháp là **Partial Configuration** — Cấu Hình Một Phần.

```hcl
# main.tf — Chỉ khai báo loại backend, không điền chi tiết
terraform {
  backend "s3" {}
}
```

**Điền chi tiết qua file:**

```hcl
# backend.hcl (không commit file này nếu có secrets)
bucket         = "mycompany-terraform-state-prod"
key            = "prod/vpc/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-state-locks"
```

```bash
# Khởi tạo với file config
terraform init -backend-config=backend.hcl

# Hoặc truyền từng tham số
terraform init \
  -backend-config="bucket=mycompany-terraform-state-prod" \
  -backend-config="key=prod/vpc/terraform.tfstate" \
  -backend-config="region=us-east-1"
```

**Tại sao cần Partial Configuration?**

```
1. Cho phép dùng cùng code cho nhiều môi trường
   → dev dùng backend-dev.hcl, prod dùng backend-prod.hcl

2. Bảo mật: Không hardcode bucket name, region vào code
   → Thay đổi backend không cần sửa code

3. CI/CD: Truyền qua biến môi trường hoặc secrets manager
```

---

## 7. terraform_remote_state — Đọc State Từ Project Khác

Cho phép một Terraform project đọc output của project khác — pattern phổ biến trong kiến trúc nhiều layer.

```hcl
# Project B đọc VPC ID từ Project A (đã apply)
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state-prod"
    key    = "prod/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Dùng output của Project A
resource "aws_eks_cluster" "main" {
  name     = "prod-eks"
  role_arn = aws_iam_role.eks.arn

  vpc_config {
    subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
  }
}
```

**Lưu ý quan trọng:** Chỉ đọc được `outputs` — đầu ra — của project kia, không đọc được toàn bộ state.

**Kiến trúc nhiều tầng điển hình:**

```
Layer 1: networking/     → outputs: vpc_id, subnet_ids
Layer 2: database/       → đọc Layer 1 → outputs: db_endpoint
Layer 3: application/    → đọc Layer 1 + 2 → deploy app
```

---

## 8. Migrate Backend — Chuyển Đổi Backend

Khi cần chuyển từ local sang remote, hoặc từ backend này sang backend khác:

```bash
# 1. Cập nhật cấu hình backend trong main.tf
# 2. Chạy init với -migrate-state
terraform init -migrate-state

# Terraform sẽ hỏi xác nhận:
# Do you want to copy existing state to the new backend?
# → Nhập "yes"
```

**Ví dụ: Local → S3**

```hcl
# Trước: Không có backend block (dùng local)
# Sau: Thêm backend S3
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

```bash
terraform init -migrate-state
# → Terraform copy terraform.tfstate lên S3 tự động
```

---

## 9. Câu Hỏi Phỏng Vấn

### Q: Tại sao cần remote backend?

**Trả lời mẫu:**
> Remote backend giải quyết ba vấn đề của local state: (1) Collaboration — nhiều người cùng làm việc cần chia sẻ state; (2) State Locking — ngăn hai pipeline apply cùng lúc gây corrupt; (3) Security — state có thể chứa secrets nên cần encryption và IAM controls riêng. Với team, remote backend là bắt buộc từ môi trường staging trở lên.

### Q: Tại sao S3 cần thêm DynamoDB cho state locking?

**Trả lời mẫu:**
> S3 không có cơ chế atomic lock tự nhiên. DynamoDB được dùng để implement distributed lock — khoá phân tán — vì nó hỗ trợ conditional writes và TTL. Khi apply bắt đầu, Terraform ghi một item vào DynamoDB với key là path của state file. Nếu item đã tồn tại (nghĩa là process khác đang apply), Terraform báo lỗi locked và không tiếp tục.

### Q: Dùng terraform_remote_state khi nào?

**Trả lời mẫu:**
> Khi kiến trúc Terraform chia thành nhiều layer độc lập — ví dụ networking, database, application — và layer sau cần dùng output của layer trước. Lưu ý chỉ nên expose những thông tin cần thiết qua outputs, không nên dùng remote_state để couple quá nhiều giữa các project. Một số team thích dùng biến môi trường hoặc SSM Parameter Store thay vì remote_state để giảm coupling.

---

## 🔗 Đọc Tiếp

- [3-state-locking.md](./3-state-locking.md) — State Locking chi tiết và cách xử lý deadlock
- [4-state-commands.md](./4-state-commands.md) — Các lệnh thao tác state

---

**Cập Nhật Lần Cuối:** 2026-05-12
