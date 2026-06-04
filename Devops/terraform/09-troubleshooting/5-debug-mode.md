# TF_LOG=DEBUG — Chế Độ Gỡ Lỗi Chi Tiết

> Khi gặp lỗi khó hiểu trong Terraform, `TF_LOG` — biến môi trường bật chế độ ghi nhật ký — là công cụ đầu tiên cần dùng. Bài này hướng dẫn cách dùng hiệu quả.

---

## Các Cấp Độ Log

| Cấp Độ    | Mô Tả                                                              | Khi Nào Dùng                                 |
| --------- | ------------------------------------------------------------------ | -------------------------------------------- |
| `ERROR`   | Chỉ lỗi nghiêm trọng                                               | Tìm lỗi cụ thể nhanh                        |
| `WARN`    | Cảnh báo + errors                                                  | Xem cảnh báo có thể bỏ qua                  |
| `INFO`    | Thông tin chung về quá trình                                       | Theo dõi tiến trình apply                    |
| `DEBUG`   | Chi tiết đầy đủ bao gồm API calls                                  | Debug lỗi provider, authentication           |
| `TRACE`   | Cực kỳ chi tiết — mọi thứ                                          | Debug provider internals — hiếm khi cần     |
| `JSON`    | Output định dạng JSON (Terraform 1.1+)                             | Parse log bằng script/jq                     |

---

## Cách Bật TF\_LOG

### Bật Log Cơ Bản

```bash
# Cách 1: Export trước khi chạy lệnh
export TF_LOG=DEBUG
terraform apply

# Cách 2: Inline — Bật trực tiếp trong lệnh
TF_LOG=DEBUG terraform plan

# Cách 3: Ghi log ra file
export TF_LOG=DEBUG
export TF_LOG_PATH=/tmp/terraform-debug.log
terraform apply
# Log sẽ ghi vào file thay vì stderr

# Tắt log (sau khi debug xong)
unset TF_LOG
unset TF_LOG_PATH
```

### Log Riêng Cho Provider

```bash
# Bật log riêng cho provider (không hiển thị log core của Terraform)
export TF_LOG_PROVIDER=DEBUG
terraform apply

# Bật cả hai
export TF_LOG=INFO            # Core Terraform ở mức INFO
export TF_LOG_PROVIDER=DEBUG  # Provider ở mức DEBUG
terraform apply
```

### Log Định Dạng JSON

```bash
# Terraform 1.1+: Log dạng JSON để dễ parse
export TF_LOG=JSON
terraform apply 2>&1 | jq 'select(.@level == "error")'

# Lọc theo level
export TF_LOG=JSON
export TF_LOG_PATH=/tmp/tf-json.log
terraform apply
cat /tmp/tf-json.log | jq 'select(.@level == "error" or .@level == "warn")'
```

---

## Phân Tích Log DEBUG

### Cấu Trúc Log DEBUG

```
2026-05-12T10:30:01.123Z [INFO]  Terraform version: 1.7.0
2026-05-12T10:30:01.234Z [DEBUG] provider: starting provider: path=.terraform/providers/.../terraform-provider-aws_v5.31.0
2026-05-12T10:30:02.345Z [DEBUG] provider.terraform-provider-aws: 2026/05/12 10:30:02 [DEBUG] AWS Request:
  PUT https://s3.amazonaws.com/my-bucket
  User-Agent: HashiCorp Terraform/1.7.0 (+https://www.terraform.io) Terraform Plugin SDK/2.10.1 terraform-provider-aws/5.31.0
  Content-Type: application/xml
  ...
2026-05-12T10:30:02.456Z [DEBUG] provider.terraform-provider-aws: 2026/05/12 10:30:02 [DEBUG] AWS Response:
  HTTP/1.1 200 OK
  ...
```

### Tìm Thông Tin Quan Trọng Trong Log

```bash
# Tìm HTTP requests thực sự được gửi đi
grep -E "HTTP Request|PUT|GET|POST|DELETE" /tmp/terraform-debug.log

# Tìm HTTP responses và status codes
grep -E "HTTP Response|HTTP/1\.1" /tmp/terraform-debug.log

# Tìm errors cụ thể
grep -i "error\|failed\|denied\|unauthorized" /tmp/terraform-debug.log

# Tìm resource đang được xử lý
grep "aws_instance.web\|aws_s3_bucket.data" /tmp/terraform-debug.log

# Xem thứ tự các API calls
grep "AWS Request" /tmp/terraform-debug.log | grep -o "https://[^ ]*"

# Tìm authentication-related log
grep -i "credential\|auth\|sts\|assume.role" /tmp/terraform-debug.log
```

---

## Các Kỹ Thuật Debug Nâng Cao

### Debug Kế Hoạch Apply — Plan Debug

```bash
# Save plan ở dạng binary và JSON
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# Phân tích plan JSON để hiểu thay đổi
cat tfplan.json | jq '.resource_changes[] | {address: .address, action: .change.actions}'

# Xem thay đổi của resource cụ thể
cat tfplan.json | jq '.resource_changes[] | select(.address == "aws_instance.web")'

# Đếm số lượng thay đổi theo loại
cat tfplan.json | jq '[.resource_changes[].change.actions[]] | group_by(.) | map({action: .[0], count: length})'
```

### Debug State

```bash
# Xem state đầy đủ ở dạng JSON
terraform state pull | jq .

# Xem resource cụ thể trong state
terraform state show aws_instance.web

# So sánh state với thực tế
terraform refresh    # Cập nhật state từ cloud
terraform plan       # Xem sự khác biệt sau refresh

# Xem lineage và serial của state (để phát hiện conflict)
terraform state pull | jq '{lineage: .lineage, serial: .serial, version: .version}'
```

### Debug Provider Configuration

```bash
# Validate cấu hình provider
terraform validate

# Xem provider cụ thể được dùng
terraform providers

# Xem provider schema đầy đủ
terraform providers schema -json | jq '.provider_schemas["registry.terraform.io/hashicorp/aws"]'

# Lock file — file khoá phiên bản — của provider
cat .terraform.lock.hcl
```

---

## Xử Lý Các Tình Huống Debug Cụ Thể

### Tình Huống 1: "Error: No valid credential sources"

```bash
# Bật DEBUG để xem Terraform tìm credential ở đâu
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "credential\|provider\|auth"

# Kết quả mẫu cho thấy thứ tự tìm credential:
# 1. Environment variables (AWS_ACCESS_KEY_ID, etc.)
# 2. Shared credentials file (~/.aws/credentials)
# 3. AWS config file (~/.aws/config)
# 4. IAM role từ EC2 instance metadata
# 5. ECS container credentials
```

### Tình Huống 2: Resource Không Tạo Được

```bash
# Debug xem API call nào thất bại
TF_LOG=DEBUG TF_LOG_PATH=/tmp/create-debug.log terraform apply 2>&1

# Tìm API call bị lỗi
grep -A5 "AWS Response" /tmp/create-debug.log | grep -B2 "4[0-9][0-9]\|5[0-9][0-9]"

# Xem error body đầy đủ
grep -A20 "error\|Error" /tmp/create-debug.log | head -100
```

### Tình Huống 3: Apply Lâu Không Xong

```bash
# Xem Terraform đang làm gì
TF_LOG=INFO terraform apply 2>&1 | grep -E "Still creating|Still modifying"

# Kết quả mẫu:
# aws_eks_cluster.main: Still creating... [10m elapsed]
# aws_rds_cluster.main: Still creating... [5m elapsed]

# Kiểm tra trực tiếp trên cloud
aws eks describe-cluster --name production --query 'cluster.status'
aws rds describe-db-clusters --db-cluster-identifier production
```

### Tình Huống 4: State Lock Không Giải Phóng

```bash
# Xem lock info
terraform force-unlock --help

# Tìm lock ID trong log hoặc DynamoDB
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "lock\|LockID"

# Xem lock trực tiếp trong DynamoDB
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "bucket/path/terraform.tfstate"}}'

# Force unlock (cẩn thận — chỉ dùng khi chắc chắn không có apply nào đang chạy)
terraform force-unlock <LOCK_ID>
```

---

## Công Cụ Bổ Sung Cho Debug

### terraformrc — File Cấu Hình Terraform

```hcl
# ~/.terraformrc hoặc %APPDATA%/terraform.rc (Windows)
provider_installation {
  # Debug: Dùng local provider thay vì registry
  dev_overrides {
    "hashicorp/aws" = "/path/to/local/provider/build"
  }
}

# Disable checkpoint — Tắt kiểm tra phiên bản mới
disable_checkpoint = true

# Plugin cache directory — Thư mục cache plugin
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

### Checklist Debug Nhanh

```bash
# 1-Minute Debug Checklist:
terraform validate                    # Syntax OK?
terraform fmt -check                  # Format OK?
terraform plan 2>&1 | tail -50        # Error message rõ ràng?
TF_LOG=ERROR terraform plan 2>&1     # Chỉ xem errors?

# 5-Minute Deep Debug:
TF_LOG=DEBUG TF_LOG_PATH=/tmp/debug.log terraform plan
grep -i "error\|denied\|fail" /tmp/debug.log | head -30
terraform state pull | jq 'keys'      # State structure OK?
terraform providers                   # Provider versions OK?
```

---

## Performance Debug — Debug Hiệu Năng

```bash
# Đo thời gian từng operation
time terraform plan
time terraform apply

# Profile apply để tìm resource chậm
TF_LOG=INFO terraform apply 2>&1 | grep -E "Creating|Created|Still creating" | \
  awk '{print $1, $NF}' | sort

# Xem resource nào tốn nhiều thời gian nhất
TF_LOG=INFO TF_LOG_PATH=/tmp/perf.log terraform apply
grep "Still creating" /tmp/perf.log | sort -t[ -k2 -V
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khi Terraform báo lỗi không rõ nguyên nhân, bạn debug thế nào?**

A: Tôi tiếp cận theo thứ tự từ đơn giản đến phức tạp:
1. Đọc error message kỹ — thường có đủ thông tin
2. Chạy `terraform validate` để kiểm tra syntax
3. Bật `TF_LOG=ERROR` để thấy lỗi không bị ẩn
4. Nếu vẫn chưa rõ, bật `TF_LOG=DEBUG` và ghi ra file
5. Grep log file để tìm HTTP response code và error body từ API
6. Kiểm tra state với `terraform state pull | jq .`

---

## Tóm Tắt

```
TF_LOG Levels:
├── ERROR  → Chỉ errors
├── WARN   → Warnings + errors
├── INFO   → Tiến trình + warnings + errors
├── DEBUG  → API calls + full details ← Thường dùng nhất
└── TRACE  → Mọi thứ (rất nhiều output)

Workflow Debug:
1. TF_LOG=DEBUG TF_LOG_PATH=/tmp/tf.log terraform apply
2. grep -i "error\|denied" /tmp/tf.log
3. Tìm HTTP response code không mong muốn
4. Fix theo nguyên nhân gốc

Tools bổ sung:
├── terraform show -json tfplan.binary | jq
├── terraform state pull | jq
└── terraform providers schema -json
```

---

**Xem Thêm:**
- [`3-provider-errors.md`](./3-provider-errors.md) — Xử lý lỗi provider cụ thể
- [`1-state-corruption.md`](./1-state-corruption.md) — Debug state corruption
