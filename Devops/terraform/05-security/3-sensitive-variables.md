# 3 — Sensitive Variables — Biến Nhạy Cảm Và Output Masking

> `sensitive = true` không mã hoá — nó chỉ ẩn khỏi terminal output. State file vẫn lưu plaintext.

---

## 🎯 Vấn Đề Với Biến Nhạy Cảm

Khi chạy `terraform plan` hay `terraform apply` mà không có `sensitive = true`:

```
# Terminal output không che giấu:
aws_db_instance.main: Creating...
  password         = "MySuperSecretPassword123!"  ← Lộ trong log
  username         = "admin"
```

**Ai có thể thấy log này?**
- CI/CD pipeline logs (GitHub Actions, GitLab CI)
- Mọi người có quyền xem pipeline logs
- Log aggregation systems — hệ thống tổng hợp log

---

## 🔐 sensitive = true Trong Variables

### Khai Báo Biến Nhạy Cảm

```hcl
# variables.tf

variable "db_password" {
  type        = string
  description = "Mật khẩu database production"
  sensitive   = true        # ← Đánh dấu nhạy cảm
  # Không có default — buộc phải truyền vào
}

variable "api_key" {
  type      = string
  sensitive = true
}

variable "tls_private_key" {
  type      = string
  sensitive = true
}

# Đối tượng chứa nhiều field nhạy cảm
variable "database_credentials" {
  type = object({
    username = string
    password = string  # Cả object sẽ bị sensitive
  })
  sensitive = true
}
```

### Hiệu Ứng Của sensitive = true

```
# Terminal output khi có sensitive = true:
aws_db_instance.main: Creating...
  password         = (sensitive value)  ← Được che giấu
  username         = "admin"

# Khi plan show:
  ~ resource "aws_db_instance" "main" {
      ~ password = (sensitive value)
    }
```

---

## 📤 Sensitive Outputs — Output Nhạy Cảm

### Đánh Dấu Output Là Sensitive

```hcl
# outputs.tf

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true  # Bắt buộc nếu value chứa sensitive data
}

output "db_connection_string" {
  value = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive = true  # Vì chứa password trong string
}

# Nếu input variable là sensitive, output cũng phải là sensitive
output "api_endpoint_with_key" {
  value     = "${aws_api_gateway_stage.main.invoke_url}?key=${var.api_key}"
  sensitive = true
}
```

### Terraform Bắt Buộc Khai Báo sensitive

```
# Terraform sẽ báo lỗi nếu output chứa sensitive value nhưng không đánh dấu:
│ Error: Output refers to sensitive values
│
│   on outputs.tf line 3, in output "db_password":
│    3:   value = aws_db_instance.main.password
│
│ To reduce the risk of accidentally exporting sensitive data that was not
│ intended to be exported, Terraform requires that any root module output
│ containing sensitive data be explicitly marked as sensitive
```

---

## 🔍 Truy Cập Sensitive Output

```bash
# Mặc định — không hiển thị giá trị sensitive
terraform output db_password
# (sensitive value)

# Xem tất cả output kể cả sensitive (cẩn thận!)
terraform output -json
# {
#   "db_password": {
#     "sensitive": true,
#     "type": "string",
#     "value": "MySuperSecretPassword123!"  ← Xuất hiện trong JSON!
#   }
# }

# Chỉ lấy giá trị cụ thể
terraform output -raw db_password
# MySuperSecretPassword123!  ← Xuất hiện khi dùng -raw!
```

**Lưu ý quan trọng:** `terraform output -json` và `terraform output -raw` vẫn hiển thị sensitive values. Cần kiểm soát quyền chạy lệnh này.

---

## 🏗️ Propagation — Lan Truyền Sensitive

Khi dùng sensitive variable trong biểu thức, kết quả tự động trở thành sensitive:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

locals {
  # connection_string tự động là sensitive vì dùng var.db_password
  connection_string = "host=db password=${var.db_password}"
}

output "conn_string" {
  value     = local.connection_string
  sensitive = true  # Bắt buộc — Terraform báo lỗi nếu thiếu
}
```

### Khi Nào Không Cần sensitive = true Trong Output

```hcl
# Nếu chỉ dùng phần không nhạy cảm
output "db_host" {
  value     = aws_db_instance.main.address  # Chỉ hostname, không có password
  sensitive = false                          # Mặc định, không cần ghi
}
```

---

## 🛡️ ephemeral Resources — Tài Nguyên Phù Du (Terraform 1.10+)

Terraform 1.10 giới thiệu `ephemeral` resources — giá trị không được ghi vào state file.

```hcl
# Đọc secret từ Vault mà không lưu vào state
ephemeral "vault_kv_secret" "db_password" {
  mount = "secret"
  name  = "prod/database"
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = ephemeral.vault_kv_secret.db_password.data["password"]
  # Giá trị này KHÔNG được lưu vào state file
}
```

**Tại sao quan trọng?**
- Trước Terraform 1.10: sensitive variable vẫn lưu vào state (chỉ ẩn khỏi terminal)
- Với ephemeral: giá trị hoàn toàn không tồn tại trong state file

---

## 📋 Passing Secrets — Truyền Secrets An Toàn Vào Terraform

### Cách 1: Environment Variables

```bash
# Shell — không xuất hiện trong git, không trong command history
export TF_VAR_db_password="$(aws ssm get-parameter \
  --name /prod/db/password --with-decryption \
  --query Parameter.Value --output text)"

terraform apply
unset TF_VAR_db_password  # Xóa sau khi dùng
```

### Cách 2: Từ Secret Manager Trực Tiếp

```hcl
# Đọc từ AWS SSM, không cần truyền qua variable
data "aws_ssm_parameter" "db_password" {
  name            = "/prod/db/password"
  with_decryption = true
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
  # Lưu ý: vẫn vào state file!
}
```

### Cách 3: -var-file Với File Không Commit

```bash
# Tạo file chứa secrets — KHÔNG commit file này
cat > /tmp/secrets.tfvars << EOF
db_password = "$(aws ssm get-parameter --name /prod/db/password \
  --with-decryption --query Parameter.Value --output text)"
EOF

terraform apply -var-file=/tmp/secrets.tfvars

# Xóa ngay sau khi dùng
rm /tmp/secrets.tfvars
```

---

## 🚨 Những Trường Hợp Sensitive Bị Lộ

### Lộ Trong Logs

```hcl
# Anti-pattern: dùng sensitive value trong provisioner
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "echo ${var.db_password}"  # ❌ Lộ trong terraform apply output
  }
}
```

### Lộ Trong Error Messages

```bash
# Khi resource tạo thất bại, error message có thể chứa request body
# bao gồm cả password nếu API trả về request trong error

# Giải pháp: dùng prevent_destroy lifecycle
resource "aws_db_instance" "main" {
  lifecycle {
    prevent_destroy = true
  }
}
```

### Lộ Khi Debug

```bash
# TF_LOG=DEBUG sẽ log MỌI thứ bao gồm sensitive values
export TF_LOG=DEBUG  # ❌ Không bật trong production CI/CD
export TF_LOG=ERROR  # ✅ Chỉ log lỗi
```

---

## 🔒 Bảo Vệ Sensitive Data Trong State

```hcl
# Backend với mã hoá
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"

    # Mã hoá với KMS
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/my-key"

    # Kiểm soát truy cập qua IAM
    # Chỉ role/user được phép mới đọc được bucket này
  }
}
```

```bash
# Kiểm tra state file có sensitive data không
terraform state pull | jq '.resources[].instances[].attributes | keys'

# Nếu thấy "password", "secret_key", v.v. → cần kiểm tra encryption
```

---

## 📊 Tóm Tắt: Sensitive Flag Làm Được Và Không Làm Được

| Hành Vi                                   | sensitive = true |
| ----------------------------------------- | :------------: |
| Ẩn khỏi `terraform plan` output           | ✅             |
| Ẩn khỏi `terraform apply` output          | ✅             |
| Ẩn khỏi `terraform output` (không flag)   | ✅             |
| Mã hoá trong state file                   | ❌             |
| Ẩn khỏi `terraform output -json`          | ❌             |
| Ẩn khỏi `terraform output -raw`           | ❌             |
| Ngăn ghi vào state file                   | ❌             |
| Ẩn khỏi `TF_LOG=DEBUG` logs               | ❌             |

---

## ✅ Checklist Sensitive Variables

- [ ] Tất cả variables chứa password/token/key có `sensitive = true`
- [ ] Tất cả outputs chứa sensitive data có `sensitive = true`
- [ ] Không có `TF_LOG=DEBUG` trong production CI/CD
- [ ] State backend có bật encryption
- [ ] Không có sensitive value trong provisioner commands
- [ ] Không commit `.tfvars` chứa secrets lên git
- [ ] `.gitignore` có `*.tfstate`, `secrets.tfvars`, `*.auto.tfvars`
- [ ] Cân nhắc dùng ephemeral resources (Terraform 1.10+)

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: sensitive = true có ngăn secret lưu vào state file không?**
> Không. `sensitive = true` chỉ ẩn giá trị khỏi terminal output (plan, apply). State file vẫn chứa giá trị dạng plaintext. Để bảo vệ state, cần mã hoá backend (S3 + KMS) và kiểm soát IAM access. Terraform 1.10+ có `ephemeral` resources để thực sự không lưu vào state.

**Q: Làm sao biết output nào đang chứa sensitive data?**
> Chạy `terraform output -json` và xem field `"sensitive": true`. Kiểm tra state file với `terraform state pull | jq` để tìm fields như `password`, `secret`, `token`. Dùng tfsec/Checkov để scan code phát hiện sensitive outputs chưa được đánh dấu.

**Q: Khi nào nên dùng sensitive = true vs ephemeral resources?**
> `sensitive = true`: khi cần biến/output nhưng muốn ẩn khỏi terminal. `ephemeral` (1.10+): khi không muốn giá trị tồn tại trong state file chút nào — phù hợp cho runtime credentials, temporary tokens. Ưu tiên `ephemeral` khi có thể vì bảo mật hơn.
