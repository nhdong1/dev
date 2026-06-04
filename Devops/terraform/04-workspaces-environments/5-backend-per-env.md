# Backend Per Environment — Backend Riêng Cho Từng Môi Trường

> Cách ly hoàn toàn state file theo môi trường là nền tảng cho một Terraform setup an toàn. Không bao giờ dùng chung backend giữa dev và production.

---

## 📚 Mục Lục

1. [Tại sao cần backend riêng?](#tại-sao-cần-backend-riêng)
2. [Kiến trúc backend per environment](#kiến-trúc-backend-per-environment)
3. [AWS S3 + DynamoDB backend](#aws-s3--dynamodb-backend)
4. [GCS — Google Cloud Storage — backend](#gcs-google-cloud-storage-backend)
5. [Azure Blob Storage backend](#azure-blob-storage-backend)
6. [Terraform Cloud workspaces](#terraform-cloud-workspaces)
7. [Bootstrapping — Khởi tạo — backend infrastructure](#bootstrapping-khởi-tạo-backend-infrastructure)
8. [Backend migration — Di chuyển backend](#backend-migration-di-chuyển-backend)
9. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Tại sao cần backend riêng?

### Rủi ro khi dùng chung backend

```
Scenario: Tất cả môi trường dùng chung S3 bucket "company-terraform-state"

Developer A: "Tôi sẽ xóa database cũ ở dev"
→ terraform destroy -target=aws_db_instance.main

Thực tế: Developer A đang ở workspace "prod" mà không biết
→ Database production bị xóa!
→ 4 tiếng downtime, data loss
```

### Lợi ích của backend riêng biệt

| Lợi Ích | Giải Thích |
|---------|------------|
| **Cách ly tuyệt đối** | Không thể tình cờ access state prod khi làm việc ở dev |
| **IAM per environment** — Quyền truy cập riêng | Dev team không có quyền đọc/ghi state prod |
| **Compliance** — Tuân thủ | Nhiều chuẩn (PCI-DSS, HIPAA, SOC2) yêu cầu cách ly |
| **Audit trail** — Nhật ký kiểm tra | Rõ ràng ai truy cập state của môi trường nào |
| **Cost tracking** — Theo dõi chi phí | Dễ phân tách chi phí theo môi trường |

---

## Kiến trúc backend per environment

### Cấu trúc S3 buckets khuyến nghị

```
Cùng AWS Account — Tối thiểu:
├── S3: company-terraform-state-dev
│   └── terraform.tfstate
├── S3: company-terraform-state-staging
│   └── terraform.tfstate
└── S3: company-terraform-state-prod
    └── terraform.tfstate

Khác AWS Account — Tốt nhất:
├── Account Dev (ID: 111222333444)
│   └── S3: terraform-state-111222333444
├── Account Staging (ID: 555666777888)
│   └── S3: terraform-state-555666777888
└── Account Prod (ID: 999000111222)
    └── S3: terraform-state-999000111222
```

### Naming convention — Quy tắc đặt tên

```
Quy tắc: {company}-terraform-state-{environment}
Ví dụ:   mycompany-terraform-state-dev
         mycompany-terraform-state-staging
         mycompany-terraform-state-prod

Hoặc dùng account ID (unique — duy nhất trên toàn AWS):
         terraform-state-{account_id}
Ví dụ:   terraform-state-123456789012
```

---

## AWS S3 + DynamoDB backend

### Cấu hình cho từng môi trường

**`environments/dev/backend.tf`:**
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-dev"
    key            = "services/webapp/terraform.tfstate"
    region         = "us-east-1"
    
    # DynamoDB — Cơ Sở Dữ Liệu NoSQL — cho state locking
    dynamodb_table = "terraform-state-lock-dev"
    
    encrypt        = true   # Mã hóa state file
    
    # Profile với quyền hạn chế cho dev
    profile        = "dev"
  }
}
```

**`environments/prod/backend.tf`:**
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"   # Bucket khác!
    key            = "services/webapp/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-prod"         # DynamoDB table khác!
    encrypt        = true
    
    # Assume role trong production account — cần MFA thường
    role_arn       = "arn:aws:iam::111222333444:role/TerraformStateRole"
  }
}
```

### Backend với partial configuration — Cấu hình một phần

Thay vì hardcode trong backend.tf, có thể truyền qua CLI:

**`backend.tf` (chỉ khai báo backend type):**
```hcl
terraform {
  backend "s3" {}   # Không có config — sẽ truyền qua CLI hoặc file
}
```

**Truyền config khi init:**
```bash
# Dev
terraform init \
  -backend-config="bucket=mycompany-terraform-state-dev" \
  -backend-config="key=services/webapp/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-state-lock-dev"

# Prod
terraform init \
  -backend-config="bucket=mycompany-terraform-state-prod" \
  -backend-config="key=services/webapp/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-state-lock-prod"
```

**Hoặc dùng backend config file:**
```bash
# environments/dev/backend.hcl
bucket         = "mycompany-terraform-state-dev"
key            = "services/webapp/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock-dev"

# Chạy
terraform init -backend-config="../../backend-configs/dev.hcl"
```

### Cấu hình IAM policies — Chính sách quyền truy cập

**Policy cho Terraform (dev):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3StateAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::mycompany-terraform-state-dev/*"
    },
    {
      "Sid": "S3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::mycompany-terraform-state-dev"
    },
    {
      "Sid": "DynamoDBLocking",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/terraform-state-lock-dev"
    }
  ]
}
```

**Quan trọng:** Dev team **KHÔNG có** policy tương tự cho production bucket.

---

## GCS — Google Cloud Storage — backend

**`environments/dev/backend.tf`:**
```hcl
terraform {
  backend "gcs" {
    bucket  = "mycompany-terraform-state-dev"
    prefix  = "services/webapp"       # Thay cho "key" của S3
  }
}
```

**`environments/prod/backend.tf`:**
```hcl
terraform {
  backend "gcs" {
    bucket  = "mycompany-terraform-state-prod"   # GCS bucket khác!
    prefix  = "services/webapp"
  }
}
```

### Tạo GCS bucket cho backend

```bash
# Tạo bucket với versioning — Lịch sử phiên bản — và encryption — Mã hóa
gcloud storage buckets create gs://mycompany-terraform-state-prod \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --versioning \
  --default-encryption-key=projects/my-project/locations/us-central1/keyRings/terraform-keyring/cryptoKeys/terraform-key

# Bật object versioning để có thể rollback state
gcloud storage buckets update gs://mycompany-terraform-state-prod \
  --versioning

# Cấu hình IAM — Chỉ cho phép Terraform service account
gcloud storage buckets add-iam-policy-binding gs://mycompany-terraform-state-prod \
  --member="serviceAccount:terraform-prod@myproject.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

---

## Azure Blob Storage backend

**`environments/dev/backend.tf`:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-dev-rg"
    storage_account_name = "tfstatedev001"           # Phải unique toàn cầu
    container_name       = "tfstate"
    key                  = "services/webapp/terraform.tfstate"
  }
}
```

**`environments/prod/backend.tf`:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-prod-rg"
    storage_account_name = "tfstateprod001"    # Storage account khác!
    container_name       = "tfstate"
    key                  = "services/webapp/terraform.tfstate"
    
    # Managed Identity — Danh Tính Được Quản Lý — cho production (không cần secret)
    use_azuread_auth     = true
    subscription_id      = "prod-subscription-id"
  }
}
```

---

## Terraform Cloud workspaces

Terraform Cloud — Đám Mây Terraform — có khái niệm "workspace" khác với Terraform CLI workspaces. Mỗi Terraform Cloud workspace tương đương với một môi trường độc lập hoàn toàn.

**Cấu trúc:**
```
Terraform Cloud Organization: mycompany
├── Workspace: myapp-dev
│   ├── State: riêng biệt
│   ├── Variables: riêng biệt
│   └── Run history: riêng biệt
├── Workspace: myapp-staging
│   └── ...
└── Workspace: myapp-prod
    ├── State: riêng biệt
    ├── Variables: riêng biệt (prod secrets)
    └── Approvals: manual required — Bắt buộc phê duyệt thủ công
```

**`environments/dev/backend.tf`:**
```hcl
terraform {
  backend "remote" {
    organization = "mycompany"
    
    workspaces {
      name = "myapp-dev"   # Tên workspace cụ thể
    }
  }
}
```

**`environments/prod/backend.tf`:**
```hcl
terraform {
  backend "remote" {
    organization = "mycompany"
    
    workspaces {
      name = "myapp-prod"  # Workspace khác!
    }
  }
}
```

---

## Bootstrapping — Khởi tạo — backend infrastructure

**Vấn đề con gà và quả trứng:**
```
Terraform cần S3 bucket để lưu state.
Nhưng để tạo S3 bucket cũng cần Terraform.
→ Phải tạo bucket bằng cách khác trước.
```

### Cách 1: Tạo bucket bằng CLI (Đơn giản)

```bash
# Tạo S3 bucket cho state
aws s3api create-bucket \
  --bucket mycompany-terraform-state-prod \
  --region us-east-1

# Bật versioning
aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state-prod \
  --versioning-configuration Status=Enabled

# Bật server-side encryption — Mã hóa phía server
aws s3api put-bucket-encryption \
  --bucket mycompany-terraform-state-prod \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms"
      }
    }]
  }'

# Block public access — Chặn truy cập công khai
aws s3api put-public-access-block \
  --bucket mycompany-terraform-state-prod \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Tạo DynamoDB table cho state locking
aws dynamodb create-table \
  --table-name terraform-state-lock-prod \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### Cách 2: Bootstrap module — Module khởi tạo

Tạo một Terraform config đặc biệt chỉ dùng local state để tạo backend infrastructure:

```hcl
# bootstrap/main.tf
# Dùng local backend vì chưa có S3 bucket
terraform {
  backend "local" {
    path = "bootstrap.tfstate"   # Commit file này lên git để theo dõi
  }
}

# Tạo S3 bucket cho state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state-prod"

  lifecycle {
    prevent_destroy = true   # Không bao giờ bị xóa bởi Terraform
  }
  
  tags = {
    Purpose = "Terraform State Backend"
    Environment = "prod"
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Tạo DynamoDB table cho locking
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock-prod"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

---

## Backend migration — Di chuyển backend

Khi cần thay đổi backend (ví dụ từ local sang S3, hoặc từ S3 sang Terraform Cloud):

```bash
# Bước 1: Thay đổi backend config trong backend.tf

# Bước 2: Chạy terraform init để migrate
terraform init -migrate-state

# Terraform sẽ hỏi:
# "Do you want to copy existing state to the new backend?"
# → Yes để migrate state sang backend mới

# Bước 3: Verify — Kiểm tra — state đã migrate thành công
terraform state list

# Bước 4: Xóa state cũ (sau khi verify xong)
# Với local backend: xóa file terraform.tfstate cũ
# Với S3: Không cần xóa, có thể để làm backup
```

### Trường hợp đặc biệt: Đổi từ workspace sang directory separation

```bash
# Bước 1: Backup state hiện tại
terraform workspace select prod
terraform state pull > backup-prod.tfstate

# Bước 2: Tạo thư mục environments/prod/
mkdir -p environments/prod

# Bước 3: Config backend mới trong environments/prod/backend.tf
# (Xem cấu hình ở trên)

# Bước 4: Init với backend mới
cd environments/prod
terraform init

# Bước 5: Push state backup vào backend mới
terraform state push ../../backup-prod.tfstate

# Bước 6: Verify
terraform plan   # Phải thấy "No changes"
```

---

## Câu hỏi phỏng vấn

### Q1: Tại sao cần backend riêng cho từng môi trường?

**Trả lời:**

Ba lý do chính:

1. **Cách ly bảo mật**: Nếu dùng chung S3 bucket, developer có quyền đọc state dev cũng có thể vô tình (hoặc cố ý) đọc state prod — có thể chứa endpoints, credentials, sensitive outputs.

2. **Tránh tai nạn**: Backend riêng + thư mục riêng = không thể apply nhầm vào prod. Với workspace chung, rủi ro cao hơn vì developer có thể quên switch workspace.

3. **Compliance**: Các tiêu chuẩn như PCI-DSS, SOC2 yêu cầu cách ly môi trường prod khỏi non-prod về mặt access control.

Trong thực tế, tôi thêm `allowed_account_ids` trong provider config để Terraform từ chối thực thi nếu đang ở sai AWS account — một lớp bảo vệ nữa.

---

### Q2: Làm sao tạo S3 backend mà không bị chicken-and-egg problem — Vấn đề con gà và quả trứng?

**Trả lời:**

Hai cách phổ biến:

1. **Tạo bằng CLI trước**: Dùng `aws s3api create-bucket`, `aws dynamodb create-table` để tạo infrastructure cho backend, sau đó mới dùng Terraform để quản lý mọi thứ khác.

2. **Bootstrap module với local state**: Tạo một Terraform config nhỏ dùng `backend "local"` để tạo S3 bucket + DynamoDB. File `bootstrap.tfstate` này commit lên git. Sau khi có bucket, các config khác dùng S3 backend bình thường.

Điều quan trọng: Thêm `lifecycle { prevent_destroy = true }` vào bucket và DynamoDB table để tránh bị xóa nhầm.

---

### Q3: Mô tả quá trình migrate state từ local sang S3

**Trả lời:**

```bash
# 1. Thêm backend "s3" vào main.tf/backend.tf
# 2. terraform init -migrate-state
# 3. Terraform hỏi có muốn copy state không → Yes
# 4. terraform state list để verify
# 5. Xóa terraform.tfstate local
```

Điểm quan trọng: Đây là zero-downtime operation — Không có thời gian ngừng hoạt động — vì Terraform chỉ copy state, không tạo hoặc xóa tài nguyên.

---

## 💡 Best Practices — Thực Hành Tốt Nhất

1. **Một bucket riêng cho mỗi môi trường** — không dùng prefix để phân biệt
2. **Bật versioning** trên state bucket — có thể rollback khi state bị corrupt
3. **Bật MFA Delete** — Xóa cần xác thực MFA — trên production state bucket
4. **`prevent_destroy = true`** cho bucket và DynamoDB resources
5. **Encrypt state** với KMS — Key Management Service — key riêng cho prod
6. **Block public access** hoàn toàn cho state bucket
7. **Audit logging** cho S3 bucket — dùng CloudTrail để theo dõi ai access state
8. **Cross-region replication** cho production state bucket — disaster recovery

---

## 📋 Checklist Backend Setup Per Environment

```
Dev Backend:
- [ ] S3 bucket tạo xong với versioning
- [ ] DynamoDB table cho locking
- [ ] Encryption bật
- [ ] Public access blocked
- [ ] IAM policy: dev team có quyền đọc/ghi

Staging Backend:
- [ ] S3 bucket riêng (không dùng chung với dev)
- [ ] DynamoDB table riêng
- [ ] IAM policy: staging deployment role có quyền, dev team không có

Production Backend:
- [ ] S3 bucket trong separate account (nếu có)
- [ ] KMS key riêng cho encryption
- [ ] MFA Delete bật
- [ ] CloudTrail logging bật
- [ ] Cross-region replication
- [ ] IAM policy: chỉ CI/CD role có quyền, không ai khác
- [ ] DynamoDB table với on-demand billing
```

---

**Hoàn thành:** Topic 04-workspaces-environments đã đủ nội dung.  
**Quay lại:** [README.md](./README.md) — Tổng quan phần này  
**Chủ đề tiếp theo:** [05-security/](../05-security/) — Bảo mật Terraform
