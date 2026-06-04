# 1 — Secrets Management — Quản Lý Bí Mật Trong Terraform

> Secrets trong Terraform: nếu nó xuất hiện trong state file ở dạng plaintext — văn bản thường — thì coi như bị lộ.

---

## 🎯 Vấn Đề Cần Giải Quyết

### Anti-pattern — Cách Làm Sai — Phổ Biến

```hcl
# ❌ TUYỆT ĐỐI KHÔNG LÀM NÀY
resource "aws_db_instance" "main" {
  username = "admin"
  password = "MyPassword123!"   # Hardcode trong code → lộ lên git
}

variable "db_password" {
  default = "MyPassword123!"    # Lộ trong state file dạng plaintext
}
```

**Hậu quả:**
- Password lộ trong git history — không xóa được hoàn toàn
- Password lộ trong `terraform.tfstate` dạng plaintext
- Password lộ trong CI/CD logs khi dùng `-var`

---

## 🏗️ Ba Chiến Lược Quản Lý Secrets

### Chiến Lược 1: Đọc Từ Secret Manager Qua Data Source

Đây là cách được khuyến nghị nhất — Terraform không tự lưu trữ secret mà chỉ đọc runtime.

#### AWS SSM Parameter Store — Kho Tham Số AWS

```hcl
# Lưu secret trong SSM trước (bằng CLI hoặc console)
# aws ssm put-parameter --name "/prod/db/password" \
#   --value "MySecretPassword" --type SecureString

# Đọc trong Terraform
data "aws_ssm_parameter" "db_password" {
  name            = "/prod/db/password"
  with_decryption = true
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = data.aws_ssm_parameter.db_password.value
  # Giá trị vẫn có thể lộ trong state file!
  # Cần kết hợp sensitive variable để giảm thiểu
}
```

#### AWS Secrets Manager — Trình Quản Lý Bí Mật AWS

```hcl
# Đọc secret từ Secrets Manager
data "aws_secretsmanager_secret" "db_creds" {
  name = "prod/myapp/db-credentials"
}

data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = data.aws_secretsmanager_secret.db_creds.id
}

locals {
  db_creds = jsondecode(
    data.aws_secretsmanager_secret_version.db_creds.secret_string
  )
}

resource "aws_db_instance" "main" {
  username = local.db_creds["username"]
  password = local.db_creds["password"]
}
```

#### HashiCorp Vault — Kho Lưu Bí Mật Của HashiCorp

```hcl
# Cấu hình Vault provider
provider "vault" {
  address = "https://vault.company.internal:8200"
  # Xác thực qua AWS IAM auth method hoặc token
}

# Đọc secret từ KV secrets engine
data "vault_kv_secret_v2" "db_password" {
  mount = "secret"
  name  = "prod/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_kv_secret_v2.db_password.data["username"]
  password = data.vault_kv_secret_v2.db_password.data["password"]
}
```

**Ưu điểm của Vault:**
- Dynamic secrets — bí mật động — tự tạo và tự hết hạn
- Fine-grained audit log — nhật ký chi tiết ai đọc secret nào
- Lease management — quản lý thời hạn — tự động rotate

---

### Chiến Lược 2: SOPS — Secrets OPerationS — Mã Hoá File Secrets

SOPS cho phép mã hoá file `.tfvars` hoặc YAML và commit lên git an toàn.

#### Cài Đặt và Sử Dụng SOPS

```bash
# Cài đặt SOPS
brew install sops              # macOS
apt-get install sops           # Ubuntu/Debian

# Tạo file secrets.yaml
cat > secrets.yaml << EOF
db_password: MySecretPassword
api_key: sk-abc123
EOF

# Mã hoá với AWS KMS
sops --kms arn:aws:kms:us-east-1:123456789:key/my-key-id \
     --encrypt secrets.yaml > secrets.enc.yaml

# Commit file đã mã hoá lên git
git add secrets.enc.yaml
git commit -m "add encrypted secrets"

# Giải mã khi dùng (cần quyền KMS)
sops --decrypt secrets.enc.yaml > secrets.yaml
```

#### Tích Hợp SOPS Với Terraform

```bash
# Trong CI/CD pipeline
sops --decrypt secrets.enc.yaml > /tmp/secrets.yaml

# Dùng như tfvars
terraform apply -var-file=/tmp/secrets.yaml

# Cleanup ngay sau khi dùng
rm /tmp/secrets.yaml
```

#### .sops.yaml — Cấu Hình SOPS

```yaml
# .sops.yaml — đặt ở root của repo
creation_rules:
  # Dùng AWS KMS cho môi trường production
  - path_regex: ".*prod.*"
    kms: "arn:aws:kms:us-east-1:123456789:key/prod-key-id"

  # Dùng age encryption cho development
  - path_regex: ".*dev.*"
    age: "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p"

  # Mặc định
  - kms: "arn:aws:kms:us-east-1:123456789:key/default-key-id"
    age: "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p"
```

---

### Chiến Lược 3: Inject Qua Environment Variables — Biến Môi Trường

```hcl
# variables.tf
variable "db_password" {
  type      = string
  sensitive = true
  # Không có default
}

# Inject qua môi trường — không cần truyền trực tiếp
# export TF_VAR_db_password="$(aws ssm get-parameter \
#   --name /prod/db/password --with-decryption \
#   --query Parameter.Value --output text)"
# terraform apply
```

---

## ⚠️ State File Và Secrets — Vấn Đề Chưa Giải Quyết Hoàn Toàn

Kể cả khi dùng secret manager, **Terraform vẫn lưu giá trị vào state file** khi resource được tạo.

```json
// terraform.tfstate — plaintext JSON
{
  "resources": [{
    "type": "aws_db_instance",
    "instances": [{
      "attributes": {
        "password": "MySecretPassword",  // ← Vẫn lộ ở đây!
        "username": "admin"
      }
    }]
  }]
}
```

### Giải Pháp Giảm Thiểu Rủi Ro State File

```hcl
# 1. Bật mã hoá state backend
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true          # Mã hoá với KMS
    kms_key_id = "arn:aws:kms:us-east-1:123456789:key/state-key"

    # State locking với DynamoDB
    dynamodb_table = "terraform-state-lock"
  }
}

# 2. Kiểm soát quyền đọc state file
# Chỉ IAM role được phép mới đọc được S3 bucket chứa state
```

```bash
# 3. Giới hạn ai có thể đọc state
aws s3api put-bucket-policy --bucket my-terraform-state \
  --policy file://state-bucket-policy.json
```

```json
// state-bucket-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": [
        "arn:aws:iam::123456789:role/terraform-ci-role",
        "arn:aws:iam::123456789:role/platform-engineer-role"
      ]
    },
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-terraform-state/*"
  }]
}
```

---

## 🔄 Rotation — Xoay Vòng Secrets

### Tự Động Rotate Với AWS Secrets Manager

```hcl
resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn

  rotation_rules {
    automatically_after_days = 30  # Rotate mỗi 30 ngày
  }
}
```

### Terragrunt + External Secret Manager Pattern

```hcl
# terragrunt.hcl
inputs = {
  # Đọc secret lúc runtime, không lưu trong code
  db_password = run_cmd("aws", "ssm", "get-parameter",
    "--name", "/prod/db/password",
    "--with-decryption",
    "--query", "Parameter.Value",
    "--output", "text"
  )
}
```

---

## 📊 So Sánh Các Giải Pháp

| Giải Pháp              | Ưu Điểm                          | Nhược Điểm                        | Phù Hợp Khi           |
| ---------------------- | -------------------------------- | --------------------------------- | --------------------- |
| AWS SSM Parameter Store | Đơn giản, tích hợp tốt AWS      | Chỉ dùng cho AWS                  | Dự án chỉ dùng AWS    |
| AWS Secrets Manager    | Rotation tự động, rotation hook  | Tốn phí ($0.40/secret/tháng)      | Cần auto-rotation     |
| HashiCorp Vault        | Multi-cloud, dynamic secrets     | Phức tạp, cần self-host hoặc HCP  | Enterprise, multi-cloud |
| SOPS                   | Secrets trong git an toàn        | Phải giải mã trước khi dùng       | GitOps workflow       |
| Env Variables          | Đơn giản                        | Dễ lộ trong logs, shell history   | Local dev, đơn giản   |

---

## 🚫 Những Điều Tuyệt Đối Không Làm

```hcl
# ❌ Hardcode trong code
password = "MyPassword123!"

# ❌ Để trong .tfvars rồi commit lên git
# secrets.tfvars:
# db_password = "MyPassword123!"

# ❌ Dùng -var trực tiếp trong CI/CD logs có thể nhìn thấy
# terraform apply -var="db_password=MyPassword123!"

# ❌ Echo secret ra console/log
output "db_password" {
  value = var.db_password
  # sensitive = true  ← Quên đánh dấu này
}
```

---

## ✅ Checklist Secrets Management

- [ ] Không có secret nào hardcode trong `.tf` files
- [ ] Không có secret trong `.tfvars` được commit lên git
- [ ] Biến nhạy cảm có `sensitive = true`
- [ ] Output nhạy cảm có `sensitive = true`
- [ ] State backend có bật encryption
- [ ] Quyền đọc state file bị giới hạn chặt chẽ
- [ ] Secrets được lưu trong secret manager (Vault/SSM/Secrets Manager)
- [ ] Rotation được thiết lập tự động
- [ ] `.gitignore` có `*.tfstate`, `*.tfstate.*`, `.terraform/`, `secrets.tfvars`

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Terraform state file có lưu secrets không?**
> Có, kể cả khi bạn đọc secret từ Vault hay SSM, Terraform vẫn lưu giá trị vào state file dạng plaintext trong một số trường hợp (vd: `aws_db_instance.password`). Giải pháp là mã hoá state file ở backend (S3 + KMS) và giới hạn quyền đọc state.

**Q: Sự khác nhau giữa SSM Parameter Store và Secrets Manager?**
> SSM Parameter Store: rẻ hơn, đơn giản, không có auto-rotation built-in. Secrets Manager: đắt hơn, có Lambda rotation hook, tracking chi tiết hơn. Dùng Secrets Manager khi cần auto-rotation (database passwords, API keys).

**Q: SOPS dùng để làm gì trong Terraform workflow?**
> SOPS cho phép mã hoá file secrets và commit lên git an toàn. CI/CD pipeline giải mã lúc chạy bằng KMS key. Phù hợp khi muốn lưu cấu hình secrets cùng code trong GitOps workflow.
