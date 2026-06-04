# Top 20 Câu Hỏi Phỏng Vấn Terraform

> Bộ câu hỏi được tổng hợp từ các buổi phỏng vấn DevOps Engineer, Platform Engineer, và SRE — Site Reliability Engineer — tại các công ty công nghệ. Mỗi câu kèm đáp án mẫu, điểm cần nhấn mạnh và lỗi hay mắc.

---

## 🏆 Nhóm 1: State Management — Quản Lý Trạng Thái (Hỏi 95% phỏng vấn)

---

### Câu 1: State file — tệp trạng thái — là gì? Tại sao không nên commit lên Git?

**Đây là câu hỏi sàng lọc cơ bản nhất — nếu không trả lời được, buổi phỏng vấn thường kết thúc sớm.**

#### Đáp Án Mẫu

```
State file (terraform.tfstate) là bản đồ ánh xạ giữa code Terraform
và tài nguyên thực tế trên cloud. Nó lưu:
- Resource ID của mọi tài nguyên đã tạo
- Metadata — Siêu dữ liệu — như ARN, IP, creation time
- Output values — Giá trị đầu ra
- Dependencies — Phụ thuộc giữa resources

Lý do KHÔNG commit lên Git:
1. Chứa secrets: database passwords, private keys, sensitive outputs
   đều lưu dưới dạng plaintext — văn bản thuần túy
2. Race condition — Điều kiện tranh chấp: 2 người pull/push đồng thời
   gây conflict không giải quyết được bằng merge
3. Không có locking: Git không có cơ chế khóa file khi đang apply
4. Audit trail — Lịch sử kiểm tra: Git history không phải nơi track
   thay đổi infrastructure

Giải pháp: Dùng remote backend (S3 + DynamoDB, GCS, Terraform Cloud)
với encryption at rest — mã hóa khi lưu — và state locking tích hợp.
```

#### Điểm Cần Nhấn Mạnh

- Đề cập **security risk** (secrets trong plaintext) — đây là lý do quan trọng nhất
- Giải thích **locking problem** — không chỉ nói "không nên" mà giải thích hậu quả
- Ngay lập tức nêu **giải pháp** thay thế

#### Lỗi Thường Gặp

```
❌ "Vì file này to" — Sai, lý do chính không phải kích thước
❌ "Vì Terraform tự quản lý" — Không đủ, cần giải thích cụ thể
❌ Không đề cập secrets risk — Bỏ qua rủi ro quan trọng nhất
```

---

### Câu 2: Giải thích State Locking — Khoá trạng thái — và điều gì xảy ra khi lock bị stuck?

#### Đáp Án Mẫu

```
State Locking là cơ chế đảm bảo chỉ một tiến trình Terraform được
phép modify state tại một thời điểm.

Cách hoạt động:
- Khi bắt đầu `terraform apply`, Terraform tạo lock record
  (trên DynamoDB với backend S3, hoặc GCS object lock)
- Lock gồm: lock ID, thời gian tạo, operation type, hostname
- Sau khi apply xong, lock được giải phóng
- Nếu có lock tồn tại, Terraform báo lỗi và từ chối chạy

Lock bị stuck (deadlock) xảy ra khi:
- Process bị kill đột ngột (Ctrl+C, timeout CI/CD)
- Network failure — Lỗi mạng — giữa chừng
- Bug trong code Terraform hoặc provider

Xử lý stuck lock:
1. Xác nhận không có ai đang apply thật sự:
   `terraform force-unlock <LOCK_ID>`
   (lấy LOCK_ID từ error message)

2. Với S3 backend: xóa item trong DynamoDB table
   (chỉ làm khi chắc chắn không ai đang apply)

3. Best practice: luôn wrap apply trong try/finally
   để đảm bảo cleanup kể cả khi fail
```

#### Điểm Cần Nhấn Mạnh

- Giải thích **cơ chế kỹ thuật** (không chỉ nói "nó lock lại")
- Nêu **trường hợp thực tế** xảy ra deadlock
- Biết command `force-unlock` và **cảnh báo nguy hiểm** của nó

---

### Câu 3: Remote Backend — Backend từ xa — là gì? So sánh các lựa chọn?

#### Đáp Án Mẫu

```
Remote Backend là nơi lưu state file ở xa thay vì local disk.

Tại sao cần:
- Team nhiều người cần share state
- CI/CD pipeline chạy trên nhiều máy khác nhau
- Audit trail — lịch sử ai thay đổi gì
- State encryption — mã hóa trạng thái

So sánh các backend phổ biến:

┌──────────────────┬──────────────────────────────────────────────┐
│ Backend          │ Đặc điểm                                     │
├──────────────────┼──────────────────────────────────────────────┤
│ S3 + DynamoDB    │ AWS-native, locking qua DynamoDB, encryption │
│                  │ qua KMS — Key Management Service. Chi phí    │
│                  │ rẻ. Cần setup IAM đúng cách.                │
├──────────────────┼──────────────────────────────────────────────┤
│ GCS              │ GCP-native, locking built-in. Đơn giản hơn  │
│                  │ S3 (không cần DynamoDB riêng)               │
├──────────────────┼──────────────────────────────────────────────┤
│ Terraform Cloud  │ Fully managed — Quản lý hoàn toàn. Có UI,  │
│ / HCP Terraform  │ RBAC — Role-Based Access Control, VCS       │
│                  │ integration. Free tier cho team nhỏ.         │
├──────────────────┼──────────────────────────────────────────────┤
│ Azure Blob       │ Azure-native. Locking qua blob lease.        │
└──────────────────┴──────────────────────────────────────────────┘

Khuyến nghị: Dùng backend của cloud provider mà bạn đang dùng chính.
Terraform Cloud khi muốn fully managed và có VCS integration.
```

---

### Câu 4: Làm thế nào để xử lý khi state file bị corrupt — hỏng?

#### Đáp Án Mẫu

```
Khi phát hiện state file corrupt (thường do lỗi partial write):

Bước 1 — Đừng hoảng loạn, đừng apply thêm gì
  terraform state pull > terraform.tfstate.backup
  (lưu bản hiện tại dù hỏng)

Bước 2 — Kiểm tra backup tự động
  - S3 versioning giữ mọi phiên bản cũ
  - Terraform Cloud lưu tất cả state versions
  - Dùng: terraform state pull (từ remote) hoặc
    aws s3api get-object-attributes để xem versions

Bước 3 — Rollback về phiên bản tốt gần nhất
  # Với S3: download version cũ
  aws s3api get-object \
    --bucket my-terraform-state \
    --key terraform.tfstate \
    --version-id <VERSION_ID> \
    terraform.tfstate.good

  # Push version tốt lên lại
  terraform state push terraform.tfstate.good

Bước 4 — Nếu không có backup: rebuild từ reality
  terraform import <resource_type.name> <cloud_resource_id>
  (import từng tài nguyên đang tồn tại trên cloud vào state mới)

Phòng ngừa:
- Bật S3 versioning — KHÔNG BAO GIỜ bỏ qua điều này
- Bật MFA Delete — xóa cần xác thực 2 bước
- Định kỳ backup state ra ngoài
```

---

## 🏆 Nhóm 2: Modules — Mô-đun (Hỏi 90% phỏng vấn)

---

### Câu 5: Module tốt trông như thế nào? Tiêu chí để thiết kế module tốt?

#### Đáp Án Mẫu

```
Module Terraform tốt cần đáp ứng các tiêu chí:

1. Single Responsibility — Đơn trách nhiệm
   Mỗi module làm một việc tốt. Module "networking" tạo VPC.
   Module "compute" tạo EC2. KHÔNG trộn lẫn.

2. Stable Interface — Giao diện ổn định
   Variables và outputs là API của module. Thay đổi phải có
   semantic versioning — đánh số phiên bản ngữ nghĩa.
   Breaking change: major version. New feature: minor. Fix: patch.

3. Sensible Defaults — Giá trị mặc định hợp lý
   variable "enable_deletion_protection" {
     default = true  # An toàn theo mặc định
   }

4. Composable — Có thể ghép lại
   Module không tự tạo dependencies không cần thiết.
   Nhận IDs làm input thay vì tự tạo mọi thứ.

5. Documented — Có tài liệu
   README.md với ví dụ, mô tả inputs/outputs, yêu cầu versions

6. Testable — Có thể kiểm thử
   Có thể deploy isolated để test với Terratest

Ví dụ module structure chuẩn:
modules/vpc/
├── main.tf       # Tài nguyên chính
├── variables.tf  # Input variables
├── outputs.tf    # Outputs
├── versions.tf   # Required providers và Terraform version
└── README.md     # Tài liệu sử dụng
```

---

### Câu 6: `count` vs `for_each` — dùng khi nào? Hậu quả nếu dùng sai?

#### Đáp Án Mẫu

```
count: Dùng khi tạo N tài nguyên giống nhau, không cần key duy nhất
for_each: Dùng khi tạo tài nguyên từ collection có unique key

Vấn đề nghiêm trọng của count:

# Giả sử có list: ["alice", "bob", "charlie"]
resource "aws_iam_user" "users" {
  count = length(var.users)
  name  = var.users[count.index]
}
# State: aws_iam_user.users[0] = alice
#        aws_iam_user.users[1] = bob
#        aws_iam_user.users[2] = charlie

# Nếu xóa "alice" khỏi list: ["bob", "charlie"]
# Terraform thấy:
# [0] alice → bob  (UPDATE)
# [1] bob → charlie (UPDATE)
# [2] charlie → (DELETE)
# Kết quả: destroy bob và charlie rồi tạo lại — RẤT NGUY HIỂM!

Giải pháp với for_each:
resource "aws_iam_user" "users" {
  for_each = toset(var.users)
  name     = each.key
}
# State: aws_iam_user.users["alice"]
#        aws_iam_user.users["bob"]
# Xóa "alice": chỉ destroy alice, bob và charlie nguyên vẹn ✅

Quy tắc thực tế:
- count: Chỉ dùng với boolean (create: 0 hoặc 1)
  count = var.create_bastion ? 1 : 0
- for_each: Dùng cho mọi collection có nhiều hơn 1 phần tử
```

#### Điểm Cần Nhấn Mạnh

- **Index shifting** là vấn đề thực tế gây ra downtime
- Không chỉ nói "for_each tốt hơn" mà giải thích **tại sao**

---

### Câu 7: Khi nào nên dùng Terragrunt? Có nhược điểm không?

#### Đáp Án Mẫu

```
Terragrunt — Wrapper DRY — Don't Repeat Yourself — cho Terraform.

Nên dùng Terragrunt khi:
- Có nhiều environments (dev/staging/prod) với cấu hình tương tự
- Cần manage nhiều AWS accounts (landing zone patterns)
- Backend config bị lặp lại nhiều lần trong code
- Muốn cascade dependencies giữa modules (run-all)

Vấn đề Terragrunt giải quyết:
```hcl
# Không có Terragrunt: lặp 30 lần cho 30 environments
terraform {
  backend "s3" {
    bucket = "my-company-terraform-state"
    key    = "dev/vpc/terraform.tfstate"
    region = "us-east-1"
    dynamodb_table = "terraform-state-lock"
  }
}

# Với Terragrunt: khai báo 1 lần trong root terragrunt.hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "my-company-terraform-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
  }
}
```

Nhược điểm cần thành thật:
- Learning curve — Đường cong học tập — thêm một abstraction layer
- Debugging khó hơn khi gặp lỗi (cần hiểu cả Terraform lẫn Terragrunt)
- Vendor lock-in vào tool của Gruntwork (không phải HashiCorp)
- Community nhỏ hơn Terraform thuần
- Không support tốt với Terraform Cloud/Enterprise

Khi KHÔNG nên dùng: Team nhỏ, ít environments, đã dùng Terraform Cloud
```

---

## 🏆 Nhóm 3: CI/CD & Workflow — Quy Trình (Hỏi 85% phỏng vấn)

---

### Câu 8: Mô tả Terraform workflow — quy trình — trong team nhiều người?

#### Đáp Án Mẫu

```
Workflow chuẩn trong team (Gitflow với PR review):

Developer push code → PR opened
  ↓
CI/CD chạy tự động:
  1. terraform fmt --check   (kiểm tra formatting)
  2. terraform validate      (kiểm tra syntax)
  3. tflint                  (linting nâng cao)
  4. tfsec / Checkov         (security scanning)
  5. terraform plan          (tính toán diff — chênh lệch)
     → Post plan output lên PR comment
  ↓
Team review PR + plan output
  ↓
Approved → Merge to main
  ↓
CI/CD trên main:
  6. terraform plan (lần nữa để đảm bảo up-to-date)
  7. Chờ approval thủ công (cho production)
  8. terraform apply
  ↓
Post-apply checks:
  9. Verify resources created/modified correctly
  10. Update monitoring/alerting nếu cần

Key principles — Nguyên tắc quan trọng:
- KHÔNG ai apply trực tiếp từ laptop lên production
- Plan output phải được review trước khi apply
- State locking ngăn concurrent applies — apply đồng thời
- Mỗi PR = một thay đổi logic hoàn chỉnh

Tools phổ biến:
- Atlantis: PR-based Terraform automation
- Terraform Cloud: Built-in VCS integration
- GitHub Actions / GitLab CI: Tự build workflow
```

---

### Câu 9: Làm sao phát hiện và xử lý drift — lệch cấu hình — trong production?

#### Đáp Án Mẫu

```
Drift — Configuration Drift — xảy ra khi tài nguyên thực tế
trên cloud khác với state Terraform đang lưu.

Nguyên nhân phổ biến:
- Ai đó thay đổi manual qua AWS Console
- Auto-scaling điều chỉnh capacity
- Cloud provider tự update một số properties
- Emergency changes — Thay đổi khẩn cấp — không qua Terraform

Phát hiện drift:
# Cách 1: Chạy plan định kỳ (không apply)
terraform plan -detailed-exitcode
# Exit code 0: no changes
# Exit code 1: error
# Exit code 2: có drift (changes detected)

# Cách 2: terraform plan trả về non-zero → trigger alert
# Tích hợp vào cron job hoặc scheduled pipeline

# Cách 3: Dùng dedicated tools
# - Driftctl (tool chuyên phát hiện drift)
# - AWS Config Rules với Terraform
# - Terraform Cloud có built-in drift detection

Xử lý drift:
Option A — Sync state về reality (dangerous!):
  terraform refresh  # Cập nhật state theo thực tế
  (Cẩn thận: có thể gây destroy ngoài ý muốn khi apply sau)

Option B — Bring reality về code (preferred — ưu tiên):
  1. Hiểu tại sao có drift
  2. Nếu thay đổi manual là đúng: update code rồi plan/apply
  3. Nếu thay đổi manual là sai: revert bằng terraform apply

Option C — Exclude khỏi Terraform:
  lifecycle { ignore_changes = [tags, ami] }
  (Chỉ dùng khi thay đổi không do Terraform quản lý)

Best practice: Mọi thay đổi hạ tầng phải qua Terraform.
Bật AWS Config để alert khi có manual changes.
```

---

### Câu 10: Rollback strategy — Chiến lược rollback — khi terraform apply fail?

#### Đáp Án Mẫu

```
Terraform KHÔNG có built-in rollback. Đây là điểm nhiều người
không biết và gây ra vấn đề trong production.

Tại sao không có rollback:
- Terraform idempotent — Bất biến — nhưng không transactional
- Partial apply để lại state ở trạng thái trung gian
- Cloud resources không phải tất cả đều có rollback API

Các chiến lược thực tế:

1. Immutable Infrastructure — Hạ tầng bất biến:
   - Dùng create_before_destroy lifecycle
   - Tạo resource mới → validate → destroy cái cũ
   - Blue/Green deployment cho EC2/ECS/Lambda

2. State-based rollback:
   # Lưu state trước khi apply
   terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate

   # Nếu cần rollback code (không phải state)
   git revert <commit>
   terraform apply  # Apply về trạng thái cũ

3. Targeted destroy/recreate:
   # Chỉ destroy resource lỗi
   terraform destroy -target=aws_instance.web_new
   terraform apply -target=aws_instance.web_old

4. Manual intervention:
   # Với database: không thể rollback data
   # Cần có application-level rollback riêng

Prevention là tốt nhất:
- terraform plan và review kỹ trước khi apply
- Test trên staging — môi trường kiểm thử — trước
- Có maintenance window — Cửa sổ bảo trì — cho production changes
- Change Management Process — Quy trình quản lý thay đổi
```

---

## 🏆 Nhóm 4: Security — Bảo Mật (Hỏi 75% phỏng vấn)

---

### Câu 11: Quản lý secrets — bí mật — trong Terraform như thế nào?

#### Đáp Án Mẫu

```
Đây là câu hỏi nhạy cảm vì nhiều team làm sai.

KHÔNG BAO GIỜ:
- Hardcode password trong .tf files
- Pass secrets qua terraform.tfvars rồi commit
- Dùng environment variables không encrypted

Các cách đúng:

1. AWS Secrets Manager / SSM Parameter Store:
   data "aws_secretsmanager_secret_version" "db_password" {
     secret_id = "prod/database/password"
   }
   resource "aws_db_instance" "main" {
     password = data.aws_secretsmanager_secret_version.db_password.secret_string
   }
   # Terraform chỉ đọc secret lúc apply, không lưu vào state plain-text
   # (Nhưng vẫn lưu vào state! Đây là giới hạn của Terraform)

2. HashiCorp Vault:
   provider "vault" { address = "https://vault.company.com" }
   data "vault_generic_secret" "db" {
     path = "secret/database"
   }

3. SOPS — Secrets OPerationS — với age/PGP encryption:
   # Encrypt .tfvars file
   sops --encrypt secrets.tfvars > secrets.enc.tfvars
   # Decrypt và pass vào Terraform
   sops --decrypt secrets.enc.tfvars | terraform apply -var-file=/dev/stdin

4. Sensitive variables — Biến nhạy cảm:
   variable "db_password" {
     sensitive = true  # Ẩn khỏi plan/apply output
   }
   # CHÚ Ý: sensitive=true chỉ ẩn khỏi logs, vẫn lưu trong state

Best practice tổng hợp:
- Secrets không bao giờ trong git (kể cả encrypted tfvars nếu key không secure)
- State file phải encrypted at rest
- Rotate secrets — Xoay vòng bí mật — định kỳ
- Audit access to secrets manager
```

---

### Câu 12: IAM role — Vai trò IAM — nào cần cho Terraform CI/CD pipeline?

#### Đáp Án Mẫu

```
Nguyên tắc Least Privilege — Đặc quyền tối thiểu — cho Terraform:

CI/CD cần hai loại permission riêng biệt:

1. Plan Role (read-only + describe):
   - Chạy trong PR pipeline
   - Quyền: Describe, List, Get mọi resources
   - KHÔNG có quyền Create/Modify/Delete
   {
     "Effect": "Allow",
     "Action": [
       "ec2:Describe*",
       "s3:GetObject", "s3:ListBucket",
       "iam:Get*", "iam:List*"
     ]
   }

2. Apply Role (read-write nhưng scoped):
   - Chỉ chạy sau khi được approve
   - Quyền: Create/Modify/Delete các resources cụ thể
   - Restricted by resource tags/paths
   {
     "Effect": "Allow",
     "Resource": "arn:aws:ec2:*:*:instance/*",
     "Condition": {
       "StringEquals": {
         "ec2:ResourceTag/ManagedBy": "terraform",
         "ec2:ResourceTag/Environment": "${var.environment}"
       }
     }
   }

Best practices:
- Dùng OIDC — OpenID Connect — thay vì long-lived access keys
  (GitHub Actions, GitLab CI đều support OIDC với AWS)
- Separate role cho mỗi environment (dev role ≠ prod role)
- Log mọi API calls qua CloudTrail — Nhật ký API
- Regular permission audit — Kiểm tra quyền định kỳ
- Never use AdministratorAccess trong CI/CD
```

---

## 🏆 Nhóm 5: Advanced Scenarios — Kịch Bản Nâng Cao

---

### Câu 13: Giải thích `depends_on` explicit vs implicit dependency?

#### Đáp Án Mẫu

```
Terraform tự động suy ra implicit dependency — phụ thuộc ngầm —
từ references trong code:

# Implicit dependency: security_group_id reference
resource "aws_instance" "web" {
  vpc_security_group_ids = [aws_security_group.web.id]
  # Terraform biết: phải tạo security group trước
}

# Explicit dependency với depends_on: dùng khi side effect
# không được capture qua references

resource "aws_iam_role_policy_attachment" "terraform_policy" {
  role       = aws_iam_role.terraform.name
  policy_arn = aws_iam_policy.terraform.arn
}

resource "aws_instance" "web" {
  # Instance này cần policy đã được attached — gắn kết
  # nhưng không reference trực tiếp role/policy
  depends_on = [aws_iam_role_policy_attachment.terraform_policy]
}

Khi nào dùng depends_on:
- Script/provisioner phụ thuộc vào resource nhưng không reference
- AWS resource cần thời gian propagate (IAM policy, DNS)
- Side effects từ external system

Lưu ý quan trọng:
- depends_on làm graph phức tạp hơn → apply chậm hơn
- Quá nhiều explicit depends_on thường là dấu hiệu module chưa tốt
- Terraform 0.13+ cho phép depends_on trên module level
```

---

### Câu 14: Dynamic blocks — Khối động — dùng khi nào?

#### Đáp Án Mẫu

```
Dynamic blocks giải quyết bài toán: số lượng nested block
không biết trước lúc viết code.

Ví dụ thực tế — Security Group rules:

# Không có dynamic: phải viết cứng từng rule
resource "aws_security_group" "web" {
  ingress {
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
  }
  ingress {
    from_port = 443
    to_port   = 443
    protocol  = "tcp"
  }
  # Nếu cần thêm port, phải sửa code
}

# Với dynamic block: linh hoạt từ variable
variable "ingress_rules" {
  default = [
    { port = 80,  protocol = "tcp" },
    { port = 443, protocol = "tcp" },
    { port = 8080, protocol = "tcp" }
  ]
}

resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port = ingress.value.port
      to_port   = ingress.value.port
      protocol  = ingress.value.protocol
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

Khi nào KHÔNG dùng dynamic blocks:
- Khi số block đã biết và cố định → viết thẳng rõ ràng hơn
- Khi chỉ có 2-3 variations → if/conditional đơn giản hơn
- Dynamic blocks làm code khó đọc hơn → dùng có chừng mực
```

---

### Câu 15: Multi-region architecture — Kiến trúc đa vùng — với Terraform?

#### Đáp Án Mẫu

```
Một trong những câu hỏi phân biệt Senior với Mid-level.

Cách implement:

1. Multiple provider aliases — Bí danh provider:
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"           # DR = Disaster Recovery — Phục Hồi Thảm Họa
  region = "ap-southeast-1"
}

resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "company-data-primary"
}

resource "aws_s3_bucket" "dr" {
  provider = aws.dr
  bucket   = "company-data-dr"
}

2. Modules với provider pass-through:
module "vpc_primary" {
  source    = "./modules/vpc"
  providers = { aws = aws.primary }
}

module "vpc_dr" {
  source    = "./modules/vpc"
  providers = { aws = aws.dr }
}

Thách thức thực tế:
- State management: Nên có state riêng cho mỗi region hay chung?
  → Khuyến nghị: State riêng theo region, dùng data sources để share outputs
- Circular dependencies: Region A cần output của Region B và ngược lại
  → Giải quyết bằng cách split thành multiple apply passes
- Provider version constraints phải nhất quán giữa aliases

Best practice cho large-scale:
- Terragrunt với structure: live/us-east-1/vpc, live/ap-southeast-1/vpc
- Separate state per region per environment
- Cross-region outputs qua SSM Parameter Store hoặc Terraform remote state
```

---

## 🏆 Nhóm 6: Troubleshooting — Xử Lý Sự Cố (Hỏi 70% phỏng vấn)

---

### Câu 16: Làm thế nào khi cần import tài nguyên có sẵn vào Terraform?

#### Đáp Án Mẫu

```
terraform import: Đưa resource đang tồn tại trên cloud vào Terraform management.

Use cases — Trường hợp dùng:
- Legacy resources tạo thủ công trước khi có Terraform
- Sau emergency fix manual trên console
- Adopt resources từ team/account khác

Quy trình chuẩn:

Bước 1: Viết resource block trong code (không có gì trong block)
resource "aws_instance" "web" {
  # Chưa cần điền gì
}

Bước 2: Import
terraform import aws_instance.web i-1234567890abcdef0

Bước 3: Lấy actual state
terraform show -json > current_state.json
# Xem actual properties của resource

Bước 4: Điền vào code cho khớp với actual state
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "web-server"
  }
}

Bước 5: Verify không có drift
terraform plan  # Phải thấy "No changes"

Terraform 1.5+ import block (declarative — khai báo):
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}
# Có thể generate config tự động:
terraform plan -generate-config-out=generated.tf

Lưu ý:
- Import chỉ thêm vào state, không tạo resource mới
- Sau import vẫn cần chạy plan để verify không có unintended changes
- Không phải mọi resource đều importable
```

---

### Câu 17: Dependency Cycle — Vòng phụ thuộc — là gì? Cách giải quyết?

#### Đáp Án Mẫu

```
Dependency cycle xảy ra khi A cần B, B cần C, C cần A.

Ví dụ thực tế:
- Security Group A cho phép traffic từ Security Group B
- Security Group B cho phép traffic từ Security Group A
→ Terraform không biết tạo cái nào trước

Terraform sẽ báo lỗi:
Error: Cycle: aws_security_group.a, aws_security_group.b

Giải pháp:

1. Tách resource rules ra ngoài:
# Thay vì inline ingress rules trong security group,
# dùng aws_security_group_rule riêng biệt
resource "aws_security_group" "a" { }
resource "aws_security_group" "b" { }

# Rules được tạo sau khi cả hai SG đã tồn tại
resource "aws_security_group_rule" "a_from_b" {
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
  type        = "ingress"
  from_port   = 0
  to_port     = 65535
  protocol    = "tcp"
}

2. Dùng data source thay vì direct reference:
# Tạo resource A trước (apply lần 1)
# Lần 2: dùng data source để reference A trong B
data "aws_security_group" "a" {
  id = var.sg_a_id  # ID từ lần apply trước
}

3. Refactor module structure:
# Nếu cycle xảy ra giữa modules, thường là dấu hiệu
# module boundaries chưa được thiết kế tốt
# → Xem xét lại separation of concerns
```

---

### Câu 18: `terraform taint` vs `terraform apply -replace`? Khi nào dùng?

#### Đáp Án Mẫu

```
terraform taint (deprecated từ v0.15.2 — không còn dùng nữa):
  Đánh dấu resource để destroy và recreate lần apply tiếp theo.
  Sửa state trực tiếp → rủi ro khi nhiều người dùng.

terraform apply -replace (recommended — khuyến nghị dùng):
  Thực hiện taint + plan + apply trong một lệnh.
  An toàn hơn vì plan rõ ràng trước khi thực hiện.

terraform apply -replace="aws_instance.web"

Khi nào dùng -replace:
- Resource bị lỗi configuration không thể sửa in-place
  (ví dụ: user data thay đổi, AMI thay đổi)
- Instance bị "poisoned" — nhiễm độc — bởi configuration drift
- Buộc rotate credentials được baked into image
- Resource timeout khi create và muốn thử lại sạch

Khi KHÔNG nên dùng:
- Database instances (sẽ mất data)
- Tài nguyên có dependencies quan trọng
- Khi có cách fix without recreate

Thay thế an toàn hơn:
- Với stateful resources: update in-place nếu provider support
- Với EC2: dùng immutable deployment (ASG rolling update)
- Kiểm tra lifecycle ignore_changes có thể giải quyết không
```

---

## 🏆 Nhóm 7: Architecture & Best Practices — Kiến Trúc & Thực Hành Tốt Nhất

---

### Câu 19: Tổ chức code Terraform cho dự án lớn — nhiều team, nhiều môi trường?

#### Đáp Án Mẫu

```
Đây là câu hỏi phân biệt người đã làm thật sự ở scale lớn.

Structure pattern phổ biến:

Option A — Directory per environment:
infrastructure/
├── modules/           # Shared modules — Module dùng chung
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
└── global/            # IAM roles, Route53 zones (across envs)

Ưu điểm: Đơn giản, dễ hiểu
Nhược điểm: Dễ diverge — khác nhau — giữa envs, code lặp

Option B — Terragrunt với live/modules split:
live/                  # Actual deployments — Triển khai thực tế
├── us-east-1/
│   ├── dev/
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl
│   │   └── eks/
│   │       └── terragrunt.hcl
│   └── prod/
└── ap-southeast-1/
    └── prod/
modules/               # Reusable modules
├── vpc/
└── eks/

Ưu điểm: DRY, easy multi-region, dependencies rõ ràng
Nhược điểm: Phức tạp hơn, cần biết Terragrunt

Option C — Monorepo với component isolation:
- Dùng cho org có nhiều teams
- Each team owns their module namespace
- Shared modules library ở repo riêng
- State isolated per team/service

Nguyên tắc khi scale:
- Nhỏ: Directory per env với variable files
- Vừa: Terragrunt hoặc workspace (cẩn thận với workspace limitations)
- Lớn: Terragrunt với root level terragrunt.hcl
- Very large: Separate repos per domain, shared module registry
```

---

### Câu 20: Làm thế nào đảm bảo compliance — tuân thủ — trong Terraform code?

#### Đáp Án Mẫu

```
Defense in depth — Bảo vệ theo chiều sâu — approach:

Layer 1 — Static Analysis — Phân tích tĩnh (trước khi apply):
- tfsec: Quét security issues trong HCL code
  tfsec . --format json
- Checkov: Compliance against CIS benchmarks, PCI-DSS, HIPAA
  checkov -d . --framework terraform
- tflint: Provider-specific best practices

Layer 2 — Policy as Code — Chính sách dưới dạng mã:
- OPA — Open Policy Agent — với Conftest:
  # policy/terraform.rego
  deny[msg] {
    resource := input.resource.aws_s3_bucket[_]
    not resource.server_side_encryption_configuration
    msg := "S3 bucket must have encryption enabled"
  }

  conftest test plan.json --policy policy/

- Sentinel (Terraform Cloud/Enterprise):
  # Terraform-native policy language
  # Tích hợp sâu vào apply workflow

Layer 3 — Runtime Checks — Kiểm tra sau khi apply:
- AWS Config Rules: Detect non-compliant resources
- AWS Security Hub: Aggregated security findings
- Scheduled terraform plan để phát hiện drift

Layer 4 — Organizational Controls — Kiểm soát tổ chức:
- IAM SCPs — Service Control Policies — ngăn tạo non-compliant resources
- AWS Organizations với guardrails

Quy trình trong CI/CD:
PR opened
  ↓ tfsec + checkov (fail fast — từ chối sớm)
  ↓ terraform plan
  ↓ OPA/Conftest policy check
  ↓ Human review
  ↓ terraform apply
  ↓ Post-apply compliance scan
```

---

## 📋 Tóm Tắt Nhanh Cho Ngày Phỏng Vấn

```
TOP 5 điều LUÔN nhắc đến khi phỏng vấn Terraform:

1. STATE: Remote backend + encryption + versioning + locking
   → S3 + DynamoDB là combo phổ biến nhất trên AWS

2. MODULES: Single responsibility, stable interface, semantic versioning
   → Luôn nêu tiêu chí thiết kế module tốt

3. CI/CD: Plan → Review → Approve → Apply (không bao giờ apply thẳng)
   → OIDC để tránh long-lived credentials

4. SECURITY: Không commit secrets, sensitive=true, vault integration
   → State file vẫn chứa secrets → encryption at rest là bắt buộc

5. OPERATIONS: Drift detection, rollback strategy, STAR stories
   → Kể ít nhất 1 câu chuyện sự cố thực tế
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
