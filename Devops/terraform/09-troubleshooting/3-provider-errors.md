# Provider Errors — Lỗi Provider — Timeout, Rate Limit, Authentication

> Provider — Nhà cung cấp — là cầu nối giữa Terraform và API của cloud/service. Lỗi provider là những lỗi hàng ngày mà mọi kỹ sư Terraform đều gặp.

---

## Phân Loại Lỗi Provider

| Loại Lỗi                              | HTTP Status | Nguyên Nhân                              |
| ------------------------------------- | ----------- | ---------------------------------------- |
| Authentication Error — Lỗi xác thực  | 401, 403    | Credential sai, hết hạn, thiếu permission |
| Rate Limit — Giới hạn tốc độ         | 429         | Gọi API quá nhiều lần trong thời gian ngắn |
| Timeout — Hết thời gian chờ          | 504, N/A    | Resource tạo quá lâu, mạng chậm         |
| Resource Not Found — Không tìm thấy  | 404         | Resource bị xóa ngoài Terraform          |
| Conflict — Xung đột                  | 409         | Resource đang được thay đổi bởi process khác |
| Service Unavailable — Dịch vụ không có | 503       | Cloud provider có sự cố tạm thời         |

---

## Authentication Errors — Lỗi Xác Thực

### AWS Authentication

**Lỗi điển hình:**

```
Error: configuring Terraform AWS Provider: no valid credential sources found
Error: error configuring S3 Backend: no valid credential sources found
Error: AccessDenied: User is not authorized to perform: ec2:DescribeInstances
```

**Nguyên nhân và fix:**

```bash
# Kiểm tra credential hiện tại
aws sts get-caller-identity

# Lỗi 1: Không có credential nào được cấu hình
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"

# Lỗi 2: Profile sai
export AWS_PROFILE=production
# Kiểm tra profile có tồn tại không
cat ~/.aws/credentials | grep "\[production\]"

# Lỗi 3: Assume Role — Mượn vai trò — hết hạn (STS token 1-12h)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789:role/TerraformRole \
  --role-session-name terraform-session

# Lỗi 4: Thiếu permission
# Kiểm tra policy được gán cho user/role
aws iam list-attached-user-policies --user-name terraform-user
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123:user/terraform-user \
  --action-names ec2:CreateInstance
```

**Cấu hình provider đúng cách:**

```hcl
# Cách 1: Dùng IAM Role (khuyến nghị cho CI/CD)
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789:role/TerraformRole"
    session_name = "terraform-${var.environment}"
    duration     = "2h"
  }
}

# Cách 2: Dùng profile cho development local
provider "aws" {
  region  = "us-east-1"
  profile = "dev-account"
}

# Cách 3: Environment variables (cho CI/CD)
# Không hardcode credential trong code, dùng env vars
provider "aws" {
  region = "us-east-1"
  # AWS_ACCESS_KEY_ID và AWS_SECRET_ACCESS_KEY tự động được đọc từ env
}
```

### GCP Authentication — Xác Thực Google Cloud

**Lỗi điển hình:**

```
Error: google: could not find default credentials
Error: Error 403: Required 'compute.instances.insert' permission
```

```bash
# Fix: Xác thực Application Default Credentials — ADC
gcloud auth application-default login

# Hoặc dùng Service Account Key — Key tài khoản dịch vụ
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"

# Kiểm tra credentials hiện tại
gcloud auth list
```

---

## Rate Limit Errors — Lỗi Giới Hạn Tốc Độ

### Hiểu Rate Limiting

Mỗi cloud provider có giới hạn số lần gọi API mỗi giây/phút:

| Provider | Giới Hạn Điển Hình                                    |
| -------- | ----------------------------------------------------- |
| AWS      | EC2: 100 req/s; API Gateway: 500 req/s mỗi account    |
| GCP      | Compute: 1000 req/100s; IAM: 600 req/min              |
| Azure    | ARM: 1200 req/5min mỗi subscription                   |

**Lỗi điển hình:**

```
Error: Error creating Instance: googleapi: Error 429: Quota exceeded
Error: ThrottlingException: Rate exceeded
Error: TooManyRequests: exceeded request limit
```

### Cấu Hình Retry — Thử Lại Tự Động

```hcl
# Cấu hình retry cho AWS provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  # Retry configuration
  max_retries = 10  # Thử lại tối đa 10 lần
  # AWS provider tự động dùng exponential backoff — tăng thời gian chờ theo cấp số nhân
}
```

```hcl
# Cấu hình retry cho Google provider
provider "google" {
  project = "my-project"
  region  = "us-central1"

  request_timeout = "120s"
  request_reason  = "terraform-apply"
}
```

### Xử Lý Rate Limit Bằng Parallelism — Độ Song Song

```bash
# Giảm số resource tạo đồng thời (mặc định là 10)
terraform apply -parallelism=3

# Với môi trường có nhiều resource hoặc strict rate limit
terraform apply -parallelism=1   # Serial — tuần tự — hoàn toàn

# Kiểm tra parallelism phù hợp
terraform plan -parallelism=5
```

### Tách Apply Theo Batch

```bash
# Nếu apply 200+ resource cùng lúc gây rate limit
# Tách thành nhiều lần apply nhỏ hơn

# Apply một module cụ thể
terraform apply -target=module.network
terraform apply -target=module.security
terraform apply -target=module.compute

# Chờ giữa các lần apply nếu cần
sleep 30 && terraform apply -target=module.database
```

---

## Timeout Errors — Lỗi Hết Thời Gian Chờ

### Nguyên Nhân Timeout

- Resource tạo lâu hơn dự kiến (RDS, EKS cluster mất 15-30 phút)
- Mạng chậm giữa Terraform runner và cloud API
- Service đang có vấn đề tạm thời

**Lỗi điển hình:**

```
Error: timeout while waiting for state to become 'available'
Error: context deadline exceeded
Error: Error waiting for DB Instance to be available: timeout after 30m0s
```

### Tùy Chỉnh Timeout

```hcl
resource "aws_db_instance" "main" {
  identifier     = "production-db"
  instance_class = "db.r5.xlarge"
  engine         = "postgres"

  # Tăng timeout cho resource tạo lâu
  timeouts {
    create = "60m"   # Mặc định thường là 40m
    update = "80m"
    delete = "60m"
  }
}

resource "aws_eks_cluster" "main" {
  name     = "production"
  role_arn = aws_iam_role.eks.arn

  timeouts {
    create = "30m"
    delete = "30m"
  }

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}
```

### Xử Lý Timeout Trong CI/CD

```yaml
# GitHub Actions: Tăng timeout cho job
jobs:
  terraform:
    runs-on: ubuntu-latest
    timeout-minutes: 60   # Job timeout — mặc định 6h, đặt explicit

    steps:
      - name: Terraform Apply
        run: terraform apply -auto-approve
        timeout-minutes: 45   # Step timeout riêng
```

---

## Provider Version Conflicts — Xung Đột Phiên Bản Provider

**Lỗi điển hình:**

```
Error: Failed to query available provider packages
Error: Inconsistent dependency lock file
Error: The following providers do not have any checksums
```

**Fix:**

```bash
# Khóa phiên bản provider cụ thể
# Trong versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.31.0"  # Pin phiên bản chính xác — không dùng ~>
    }
  }
}

# Xóa lock file cũ và khởi tạo lại
rm .terraform.lock.hcl
terraform init

# Upgrade provider lên phiên bản mới hơn
terraform init -upgrade

# Nếu có lỗi checksum — tổng kiểm tra
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64
```

---

## "Resource Already Exists" Error — Lỗi Resource Đã Tồn Tại

**Lỗi điển hình:**

```
Error: Error creating S3 bucket: BucketAlreadyExists
Error: Error creating Security Group: InvalidGroup.Duplicate
Error: error creating IAM Role (my-role): EntityAlreadyExists
```

**Nguyên nhân:** Resource đã được tạo trước đó (bằng tay hoặc apply trước) nhưng không có trong state.

**Fix:** Dùng `terraform import` để đưa resource vào quản lý.

```bash
# Import resource vào state
terraform import aws_s3_bucket.data my-existing-bucket
terraform import aws_iam_role.lambda arn:aws:iam::123:role/my-role
terraform import aws_security_group.web sg-0abc123def456

# Sau khi import, chạy plan để verify
terraform plan
# Plan nên hiển thị "No changes" hoặc chỉ những thay đổi nhỏ về tags
```

---

## Debug Provider Issues — Debug Lỗi Provider

```bash
# Bật log chi tiết của provider
export TF_LOG=DEBUG
export TF_LOG_PROVIDER=DEBUG   # Log riêng của provider
export TF_LOG_PATH=/tmp/tf-provider-debug.log

terraform apply 2>&1 | tee /tmp/tf-apply.log

# Tìm HTTP request thực sự được gửi đi
grep "HTTP Request" /tmp/tf-provider-debug.log
grep "HTTP Response" /tmp/tf-provider-debug.log

# Xem error response từ API
grep "ERROR" /tmp/tf-provider-debug.log
grep "RequestError" /tmp/tf-provider-debug.log
```

---

## Provider Troubleshooting Checklist

```
Khi gặp lỗi provider, kiểm tra theo thứ tự:

1. Authentication
   □ Credential còn hợp lệ không? (Kiểm tra expiry)
   □ Đúng AWS account / GCP project / Azure subscription?
   □ Có đủ permission không?

2. Rate Limit
   □ Apply bao nhiêu resource cùng lúc?
   □ Có thể giảm -parallelism không?
   □ Có cần -target từng phần không?

3. Timeout
   □ Resource có cần thời gian tạo lâu không?
   □ Có thể tăng timeout trong resource block?

4. Version Conflict
   □ .terraform.lock.hcl có bị thay đổi không?
   □ Cần chạy terraform init -upgrade?

5. Resource Already Exists
   □ Resource có tồn tại bên ngoài Terraform?
   □ Cần terraform import không?
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khi apply Terraform bị rate limit, bạn xử lý thế nào?**

A: Trước tiên xác nhận đây là rate limit qua error message (429, ThrottlingException). Sau đó:
1. Giảm `-parallelism` (ví dụ từ 10 xuống 3)
2. Dùng `-target` để apply từng phần nhỏ hơn
3. Cấu hình `max_retries` trong provider block
4. Nếu cần thiết, thêm `sleep` giữa các `-target` apply

Dài hạn: Review xem có thể tách state thành nhiều workspace để phân tán API calls không.

---

## Tóm Tắt

```
Provider Errors:
├── Authentication (401/403)
│   → Kiểm tra credential, permission, assume-role token hết hạn
├── Rate Limit (429)
│   → Giảm parallelism, dùng -target, bật max_retries
├── Timeout (504)
│   → Tăng timeout block trong resource definition
├── Already Exists (409)
│   → terraform import để đưa vào state
└── Version Conflict
    → rm .terraform.lock.hcl && terraform init -upgrade
```

---

**Xem Thêm:**
- [`5-debug-mode.md`](./5-debug-mode.md) — TF\_LOG để debug chi tiết
- [`4-import-moved.md`](./4-import-moved.md) — terraform import khi resource đã tồn tại
