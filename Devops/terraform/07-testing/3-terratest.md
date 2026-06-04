# 3 — Terratest — Unit & Integration Testing Thực Tế Trên Cloud

> Terratest là thư viện Go — Golang library — giúp viết automated tests — kiểm thử tự động — cho hạ tầng Terraform bằng cách deploy thực sự lên cloud, kiểm tra kết quả, rồi destroy. Đây là tầng kiểm thử mạnh nhất nhưng cũng tốn kém nhất.

---

## Terratest Là Gì?

Terratest do Gruntwork phát triển, cho phép bạn:

1. **Deploy** hạ tầng thực sự lên AWS/GCP/Azure
2. **Validate** — Xác nhận — kết quả bằng cách gọi API, ping HTTP, kiểm tra port
3. **Destroy** tài nguyên sau khi test hoàn thành

```go
// Ví dụ đơn giản nhất: test một S3 bucket
func TestS3BucketCreation(t *testing.T) {
    // 1. Apply Terraform
    opts := &terraform.Options{
        TerraformDir: "../modules/s3-bucket",
        Vars: map[string]interface{}{
            "bucket_name": "my-test-bucket-12345",
            "environment": "test",
        },
    }

    // Tự động destroy khi test kết thúc (dù pass hay fail)
    defer terraform.Destroy(t, opts)

    // 2. Deploy
    terraform.InitAndApply(t, opts)

    // 3. Lấy output
    bucketName := terraform.Output(t, opts, "bucket_name")

    // 4. Kiểm tra bucket tồn tại và cấu hình đúng
    aws.AssertS3BucketExists(t, "us-east-1", bucketName)
    aws.AssertS3BucketVersioningExists(t, "us-east-1", bucketName)
}
```

---

## Cài Đặt và Thiết Lập

### Yêu Cầu

```bash
# Go 1.21+ cần thiết
go version   # Kiểm tra version

# Cài Go nếu chưa có: https://go.dev/dl/
```

### Khởi Tạo Go Module

```bash
# Trong thư mục test/
cd test/
go mod init github.com/your-org/your-repo/test

# Thêm Terratest dependency
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/gruntwork-io/terratest/modules/aws
go get github.com/gruntwork-io/terratest/modules/http-helper
go get github.com/gruntwork-io/terratest/modules/retry
go get github.com/stretchr/testify/assert
```

### Cấu Trúc Thư Mục Chuẩn

```
terraform-project/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── web-server/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── test/
│   ├── go.mod
│   ├── go.sum
│   ├── vpc_test.go
│   └── web_server_test.go
└── examples/
    └── complete/
        └── main.tf   # Module ví dụ để test
```

---

## Patterns Kiểm Thử Phổ Biến

### Pattern 1: Test Module Đơn Giản

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()   // Chạy song song với test khác để tiết kiệm thời gian

    awsRegion := "us-east-1"

    opts := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "vpc_cidr":    "10.0.0.0/16",
            "environment": "test",
            "region":      awsRegion,
        },
        // Không hiện output khi chạy (bớt noise)
        NoColor: true,
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    // Lấy VPC ID từ output
    vpcID := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcID)

    // Kiểm tra VPC tồn tại trên AWS
    vpc := aws.GetVpcById(t, vpcID, awsRegion)
    assert.Equal(t, "10.0.0.0/16", aws.GetTagValue(vpc.Tags, "CidrBlock"))

    // Kiểm tra subnet tồn tại
    publicSubnetIDs := terraform.OutputList(t, opts, "public_subnet_ids")
    assert.Equal(t, 3, len(publicSubnetIDs))
}
```

### Pattern 2: Test HTTP Endpoint

```go
func TestWebServerModule(t *testing.T) {
    t.Parallel()

    opts := &terraform.Options{
        TerraformDir: "../examples/web-server",
        Vars: map[string]interface{}{
            "instance_type": "t3.micro",
            "environment":   "test",
        },
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    // Lấy URL từ output
    url := terraform.Output(t, opts, "url")

    // Retry — Thử lại — cho đến khi server sẵn sàng (tối đa 5 phút)
    maxRetries := 30
    timeBetweenRetries := 10 * time.Second
    expectedBody := "Hello, World!"

    http_helper.HttpGetWithRetry(
        t,
        url,
        nil,     // TLS config — cấu hình TLS
        200,     // Expected status code — mã trạng thái kỳ vọng
        expectedBody,
        maxRetries,
        timeBetweenRetries,
    )
}
```

### Pattern 3: Test RDS Database

```go
func TestRDSModule(t *testing.T) {
    t.Parallel()

    awsRegion := "us-east-1"

    // Tạo tên ngẫu nhiên để tránh conflict khi chạy song song
    uniqueID := random.UniqueId()
    dbName := fmt.Sprintf("testdb-%s", strings.ToLower(uniqueID))

    opts := &terraform.Options{
        TerraformDir: "../modules/rds",
        Vars: map[string]interface{}{
            "db_name":       dbName,
            "db_password":   "TestPassword123!",
            "instance_class": "db.t3.micro",
            "environment":   "test",
        },
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    // Kiểm tra RDS instance ở trạng thái available
    dbID := terraform.Output(t, opts, "db_instance_id")
    db := aws.GetRdsInstanceById(t, dbID, awsRegion)

    assert.Equal(t, "available", aws.GetRdsInstanceStatus(db))
    assert.Equal(t, true, aws.GetRdsInstanceMultiAZ(db))   // Multi-AZ phải bật
    assert.Equal(t, true, aws.GetRdsInstanceEncrypted(db)) // Encryption phải bật
}
```

### Pattern 4: Idempotency Test — Kiểm Thử Lặp Lại

```go
func TestIdempotency(t *testing.T) {
    // Kiểm tra apply lần 2 không tạo thay đổi (plan rỗng)
    opts := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    // Apply lần 2 — không nên có thay đổi
    exitCode := terraform.PlanExitCode(t, opts)
    // Exit code 0 = không có thay đổi
    // Exit code 2 = có thay đổi
    assert.Equal(t, 0, exitCode, "Second apply should produce no changes")
}
```

---

## Xử Lý Chi Phí và Thời Gian

### Chạy Song Song — Parallel Execution

```go
func TestAll(t *testing.T) {
    // Nhóm test để chạy song song
    t.Run("group1", func(t *testing.T) {
        t.Parallel()
        TestVPCModule(t)
    })

    t.Run("group2", func(t *testing.T) {
        t.Parallel()
        TestWebServerModule(t)
    })
}
```

### Stage-based Testing — Kiểm Thử Theo Giai Đoạn

```go
// Cho phép chạy từng stage riêng, tránh deploy lại khi debug
func TestInfrastructureWithStages(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../examples/complete",
    }

    // Stage 1: Deploy
    defer test_structure.RunTestStage(t, "destroy", func() {
        terraform.Destroy(t, opts)
    })

    test_structure.RunTestStage(t, "deploy", func() {
        terraform.InitAndApply(t, opts)
        // Lưu output để dùng ở stage sau
        test_structure.SaveTerraformOptions(t, "../examples/complete", opts)
    })

    // Stage 2: Validate
    test_structure.RunTestStage(t, "validate", func() {
        opts = test_structure.LoadTerraformOptions(t, "../examples/complete")
        // Chạy validation
        validateInfra(t, opts)
    })
}

// Chạy chỉ stage validate (không deploy lại):
// SKIP_destroy=true SKIP_deploy=true go test -v -run TestInfrastructureWithStages
```

---

## Tích Hợp CI/CD

### GitHub Actions

```yaml
name: Terratest Integration Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths:
      - 'modules/**'

jobs:
  terratest:
    runs-on: ubuntu-latest
    # Chỉ chạy trên nhánh main hoặc PR từ nội bộ (không chạy từ fork)
    if: github.event_name == 'push' || github.event.pull_request.head.repo.full_name == github.repository

    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"
          terraform_wrapper: false   # Quan trọng! Tắt wrapper để Terratest dùng binary trực tiếp

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Run Terratest
        run: |
          cd test/
          go test -v -timeout 30m -run TestVPCModule ./...
        env:
          # Tên ngẫu nhiên để tránh conflict giữa các lần chạy song song
          TF_VAR_suffix: ${{ github.run_id }}
```

### Cleanup Khi CI Thất Bại — Dọn Dẹp Khi CI Thất Bại

```go
// Đảm bảo destroy luôn chạy dù test fail hay pass
func TestWithCleanup(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../modules/rds",
    }

    // defer chạy kể cả khi test panic
    defer func() {
        if r := recover(); r != nil {
            t.Logf("Test panicked: %v, still destroying...", r)
            terraform.Destroy(t, opts)
            panic(r)   // Re-panic sau khi destroy
        }
    }()
    defer terraform.Destroy(t, opts)

    terraform.InitAndApply(t, opts)
    // ... tests
}
```

---

## Chi Phí Thực Tế Và Khi Nào Chạy

| Module Test | Thời Gian Chạy | Chi Phí Ước Tính |
|------------|----------------|-----------------|
| VPC + Subnets | 2-3 phút | < $0.01 |
| EC2 Instance | 3-5 phút | $0.01-0.02 |
| RDS Instance | 10-15 phút | $0.05-0.10 |
| EKS Cluster | 15-25 phút | $0.50-1.00 |
| Full stack | 30-60 phút | $1.00-5.00 |

**Khuyến nghị chiến lược:**
- **Mỗi commit:** Chạy static analysis + Checkov (< 2 phút, $0)
- **Mỗi Pull Request:** Chạy Terratest cho module bị ảnh hưởng
- **Mỗi ngày/tuần:** Chạy full integration test suite — Bộ kiểm thử tích hợp đầy đủ

---

## terraform test — Built-in Framework (Terraform v1.6+)

Từ Terraform v1.6, có framework test tích hợp không cần Go:

```hcl
# tests/vpc_test.tftest.hcl

variables {
  vpc_cidr    = "10.0.0.0/16"
  environment = "test"
}

# Kiểm tra cơ bản với plan (không apply, không tốn tiền)
run "plan_check" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block không đúng"
  }
}

# Apply và kiểm tra thực tế
run "apply_check" {
  command = apply

  assert {
    condition     = length(aws_subnet.public) == 3
    error_message = "Phải có đúng 3 public subnet"
  }
}
```

```bash
# Chạy terraform test
terraform test

# Chỉ chạy file test cụ thể
terraform test -filter=tests/vpc_test.tftest.hcl
```

**So sánh Terratest vs terraform test:**

| Tiêu Chí | Terratest | terraform test |
|---------|-----------|---------------|
| **Ngôn ngữ** | Go | HCL |
| **Độ linh hoạt** | Rất cao (API calls, SSH, HTTP) | Trung bình |
| **Học dễ hơn** | Cần biết Go | Chỉ cần biết HCL |
| **Tốc độ phát triển test** | Chậm hơn | Nhanh hơn |
| **Phù hợp cho** | Test phức tạp, multi-resource | Test module đơn giản |
| **Available since** | 2016 | Terraform v1.6 (2023) |

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao cần Terratest khi đã có `terraform plan`?**

A: `terraform plan` cho biết Terraform *định làm gì*, nhưng không xác nhận kết quả thực tế. Ví dụ: plan nói sẽ tạo security group mở port 443, nhưng chỉ Terratest mới có thể verify rằng HTTPS thực sự hoạt động sau khi deploy, instance đã boot thành công, hoặc database connection pool hoạt động đúng.

**Q: Làm sao tránh chi phí test tích lũy khi CI fail?**

A: Dùng `defer terraform.Destroy(t, opts)` để đảm bảo destroy luôn chạy. Ngoài ra, cài đặt AWS tag budget alerts — Cảnh báo ngân sách theo tag — với tag `Environment=test` và một tầng dọn dẹp định kỳ bằng AWS Config hay script tìm resource cũ hơn N giờ.

**Q: Khi nào dùng `terraform test` thay Terratest?**

A: `terraform test` phù hợp khi team không quen Go, cần test đơn giản theo kiểu "assert output equals X", hoặc chỉ muốn test ở cấp plan mà không deploy thật. Terratest phù hợp khi cần test hành vi thực tế: HTTP response, database connectivity — Khả năng kết nối cơ sở dữ liệu, SSH access.

---

**Tiếp theo:** [4-checkov-opa.md](4-checkov-opa.md) — Checkov & OPA — Open Policy Agent — Compliance Testing
