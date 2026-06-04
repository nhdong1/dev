# Meta-arguments — Đối Số Siêu Dữ Liệu trong Terraform

> **Meta-arguments** — Đối Số Siêu Dữ Liệu — là các đối số đặc biệt áp dụng được cho mọi loại resource, không phụ thuộc vào provider. Chúng kiểm soát **hành vi của Terraform** chứ không phải cấu hình của tài nguyên cloud. Gồm: `depends_on`, `count`, `for_each`, `provider`, `lifecycle` và `provisioner`.

> Lưu ý: `count` và `for_each` được đề cập chi tiết trong [3-for-each-count.md](3-for-each-count.md). Bài này tập trung vào `depends_on`, `lifecycle`, `provider` và `provisioner`.

---

## 1. `depends_on` — Phụ Thuộc Tường Minh

### Terraform Tự Động Suy Luận Phụ Thuộc

Terraform thông thường tự hiểu thứ tự tạo resource dựa trên reference:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.main.id  # ← Terraform biết phải tạo subnet trước
}
```

### Khi Nào Cần `depends_on`?

Khi có **phụ thuộc ngầm** — phụ thuộc mà Terraform không thể tự suy luận từ code:

```hcl
# Tình huống: IAM policy cần được gán xong trước khi EC2 instance khởi động
# Nhưng EC2 không tham chiếu trực tiếp đến policy attachment

resource "aws_iam_role" "lambda" {
  name = "lambda-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_lambda_function" "api" {
  function_name = "api-handler"
  role          = aws_iam_role.lambda.arn
  # Lambda tham chiếu role nhưng KHÔNG tham chiếu policy_attachment
  # → Terraform có thể tạo Lambda TRƯỚC khi policy được gán xong

  # Giải pháp: dùng depends_on để khai báo tường minh
  depends_on = [
    aws_iam_role_policy_attachment.lambda_vpc
  ]
}
```

### Ví Dụ Thực Tế Khác

```hcl
# S3 bucket cần có bucket policy trước khi CloudFront distribution trỏ vào
resource "aws_s3_bucket" "static" {
  bucket = "my-static-site"
}

resource "aws_s3_bucket_policy" "static" {
  bucket = aws_s3_bucket.static.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.static.arn}/*"
    }]
  })
}

resource "aws_cloudfront_distribution" "cdn" {
  enabled = true

  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = "s3-origin"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }

  # CloudFront không tham chiếu trực tiếp đến bucket_policy
  # nhưng cần policy tồn tại trước
  depends_on = [aws_s3_bucket_policy.static]

  # ... rest of config
}
```

### Lưu Ý Quan Trọng Về `depends_on`

```hcl
# depends_on ở module level — ảnh hưởng đến TẤT CẢ resource trong module
module "database" {
  source = "./modules/rds"

  depends_on = [module.networking]
  # Tất cả resource trong module "database" sẽ chờ module "networking" hoàn thành
}
```

> **Cảnh Báo:** `depends_on` buộc Terraform phải thay thế resource nếu source bị thay đổi vì Terraform không biết resource nào trong source ảnh hưởng đến resource đích. Dùng càng ít càng tốt.

---

## 2. `lifecycle` — Kiểm Soát Vòng Đời Tài Nguyên

`lifecycle` là meta-argument quan trọng nhất để kiểm soát cách Terraform tạo, cập nhật và xóa tài nguyên.

### 2.1 `create_before_destroy` — Tạo Trước Khi Xóa

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
    # Khi cần thay thế instance:
    # 1. Tạo instance MỚI trước
    # 2. Chuyển traffic sang instance mới
    # 3. Xóa instance CŨ
    # → Zero-downtime replacement — Thay thế không gián đoạn dịch vụ
  }
}
```

**Use case điển hình:**
- Auto Scaling Group launch templates
- Load Balancer target groups
- Các tài nguyên yêu cầu zero-downtime deployment

```hcl
# SSL Certificate — phải có trước khi xóa cái cũ
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
    # Quan trọng: ACM certificate mới phải được validate trước khi xóa cái cũ
    # nếu không Load Balancer sẽ mất certificate
  }
}
```

### 2.2 `prevent_destroy` — Bảo Vệ Khỏi Bị Xóa

```hcl
resource "aws_rds_cluster" "production" {
  cluster_identifier = "prod-db-cluster"
  engine             = "aurora-postgresql"
  engine_version     = "14.6"

  lifecycle {
    prevent_destroy = true
    # Terraform sẽ từ chối apply nếu plan có yêu cầu xóa resource này
    # Phải comment hoặc bỏ dòng này trước khi có thể destroy
  }
}

# Output sẽ báo lỗi nếu ai chạy terraform destroy:
# Error: Instance cannot be destroyed
# on main.tf line 1, in resource "aws_rds_cluster" "production":
#   lifecycle {
#     prevent_destroy = true
#   }
```

> **Lưu Ý:** `prevent_destroy = true` chỉ bảo vệ khi chạy `terraform destroy` hoặc plan có destroy action. **Không** bảo vệ khi xóa resource block khỏi code và chạy `terraform apply`.

### 2.3 `ignore_changes` — Bỏ Qua Thay Đổi Cụ Thể

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name        = "app-server"
    Environment = var.environment
    # Tag "LastDeployedBy" được team đặt thủ công ngoài Terraform
  }

  lifecycle {
    ignore_changes = [
      # Terraform sẽ không cố gắng overwrite tag này
      tags["LastDeployedBy"],

      # AMI do CI/CD system cập nhật — Terraform không nên can thiệp
      ami,
    ]
  }
}
```

**Các tình huống phổ biến cần `ignore_changes`:**

```hcl
# 1. Auto Scaling Group — số lượng instance do Auto Scaling tự điều chỉnh
resource "aws_autoscaling_group" "web" {
  desired_capacity = var.initial_capacity
  min_size         = var.min_size
  max_size         = var.max_size

  lifecycle {
    ignore_changes = [
      desired_capacity,  # Auto Scaling tự điều chỉnh, Terraform không nên reset
    ]
  }
}

# 2. EKS Node Group — Kubernetes tự quản lý node count
resource "aws_eks_node_group" "workers" {
  scaling_config {
    desired_size = var.initial_node_count
    min_size     = var.min_nodes
    max_size     = var.max_nodes
  }

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size,
    ]
  }
}

# 3. Database password được rotate — quản lý bởi Secrets Manager
resource "aws_db_instance" "main" {
  identifier     = "main-db"
  engine         = "postgres"
  password       = var.initial_password

  lifecycle {
    ignore_changes = [
      password,  # Sau khi tạo, password được Secrets Manager rotate
    ]
  }
}
```

### 2.4 `replace_triggered_by` — Kích Hoạt Thay Thế Khi Resource Khác Thay Đổi

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = "t3.micro"
}

resource "aws_autoscaling_group" "app" {
  launch_template {
    id      = aws_launch_template.app.id
    version = aws_launch_template.app.latest_version
  }

  lifecycle {
    # Khi launch_template thay đổi → force replace ASG → rolling update
    replace_triggered_by = [
      aws_launch_template.app.latest_version
    ]
  }
}
```

### 2.5 `precondition` và `postcondition` — Điều Kiện Kiểm Tra

```hcl
variable "instance_type" {
  type = string
}

data "aws_ami" "app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["app-*"]
  }

  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI phải là x86_64 architecture. Tìm thấy: ${self.architecture}"
    }
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
      error_message = "Instance type phải là t3.micro, t3.small, hoặc t3.medium. Nhận: ${var.instance_type}"
    }

    postcondition {
      condition     = self.state == "running"
      error_message = "Instance phải ở trạng thái running sau khi tạo."
    }
  }
}
```

---

## 3. `provider` — Chỉ Định Provider Cụ Thể

### Dùng Khi Có Nhiều Instance Của Cùng Provider

```hcl
# Cấu hình hai AWS region
provider "aws" {
  region = "us-east-1"
  alias  = "us_east"
}

provider "aws" {
  region = "ap-southeast-1"
  alias  = "ap_southeast"
}

# Resource mặc định dùng provider không có alias
resource "aws_s3_bucket" "primary" {
  bucket = "primary-us"
  # Dùng provider "aws" mặc định (us-east-1)
}

# Resource chỉ định provider cụ thể
resource "aws_s3_bucket" "replica" {
  bucket = "replica-ap"

  provider = aws.ap_southeast  # Chỉ định dùng provider ở Singapore
}

# Replication configuration
resource "aws_s3_bucket_replication_configuration" "primary_to_replica" {
  bucket = aws_s3_bucket.primary.id
  role   = aws_iam_role.replication.arn

  rule {
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD"
    }
  }
}
```

### Multi-account Pattern — Kiểu Mẫu Đa Tài Khoản

```hcl
# Provider cho account chính (management account)
provider "aws" {
  region = "us-east-1"
  alias  = "management"
}

# Provider assume role vào account con
provider "aws" {
  region = "us-east-1"
  alias  = "workload"

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
  }
}

# Resource trong management account
resource "aws_organizations_account" "workload" {
  provider = aws.management
  name     = "workload-prod"
  email    = "workload-prod@example.com"
}

# Resource trong workload account
resource "aws_vpc" "workload" {
  provider   = aws.workload
  cidr_block = "10.0.0.0/16"
}
```

---

## 4. `provisioner` — Thực Thi Lệnh Sau Khi Tạo Resource

> **Cảnh Báo:** `provisioner` là giải pháp cuối cùng — last resort. HashiCorp khuyến cáo **không dùng** provisioner trong production vì làm mất tính idempotent — bất biến — của Terraform. Dùng User Data, cloud-init, Ansible, hoặc baked AMI thay thế.

### 4.1 `remote-exec` — Chạy Lệnh Trên Remote Host

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deployer.key_name

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file(var.private_key_path)
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y docker",
      "sudo systemctl start docker",
      "sudo systemctl enable docker",
      "sudo docker pull ${var.app_image}:${var.app_version}",
      "sudo docker run -d -p 80:8080 ${var.app_image}:${var.app_version}",
    ]
  }
}
```

### 4.2 `local-exec` — Chạy Lệnh Trên Máy Chạy Terraform

```hcl
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks.arn

  vpc_config {
    subnet_ids = var.subnet_ids
  }

  # Cập nhật kubeconfig sau khi tạo cluster
  provisioner "local-exec" {
    command = <<-EOT
      aws eks update-kubeconfig \
        --name ${self.name} \
        --region ${var.region} \
        --kubeconfig ${path.module}/kubeconfig
    EOT

    environment = {
      AWS_REGION = var.region
    }
  }
}
```

### 4.3 `file` — Copy File Lên Remote Host

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file(var.private_key_path)
    host        = self.public_ip
  }

  # Copy config file lên server
  provisioner "file" {
    source      = "configs/app.conf"
    destination = "/tmp/app.conf"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo mv /tmp/app.conf /etc/app/app.conf",
      "sudo systemctl restart app",
    ]
  }
}
```

### 4.4 `on_failure` — Xử Lý Khi Provisioner Thất Bại

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    on_failure = continue  # Bỏ qua lỗi, tiếp tục (mặc định: fail)

    inline = [
      "sudo systemctl restart optional-service || true",
    ]
  }

  provisioner "remote-exec" {
    on_failure = fail  # Mặc định — dừng lại và mark resource là tainted

    inline = [
      "sudo systemctl start critical-service",
    ]
  }
}
```

### 4.5 `when = destroy` — Chạy Khi Xóa Resource

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  connection {
    type = "ssh"
    user = "ubuntu"
    host = self.public_ip
    private_key = file(var.private_key_path)
  }

  # Chạy cleanup trước khi instance bị xóa
  provisioner "remote-exec" {
    when = destroy

    inline = [
      "sudo systemctl stop app",
      "sudo /opt/app/drain.sh",  # Drain connections gracefully
    ]
  }
}
```

---

## 5. Tổng Hợp So Sánh Meta-arguments

| Meta-argument | Mục Đích | Khi Nào Dùng |
|--------------|---------|-------------|
| `depends_on` | Phụ thuộc tường minh | Phụ thuộc ngầm không tự suy luận được |
| `lifecycle.create_before_destroy` | Zero-downtime replacement | Tài nguyên production cần HA — High Availability |
| `lifecycle.prevent_destroy` | Bảo vệ tài nguyên quan trọng | Database, S3 bucket production |
| `lifecycle.ignore_changes` | Bỏ qua thay đổi ngoài Terraform | ASG desired_count, external rotation |
| `lifecycle.replace_triggered_by` | Force replace khi dependency đổi | Launch template update → ASG replace |
| `lifecycle.precondition` | Validate input trước khi apply | Kiểm tra điều kiện business logic |
| `lifecycle.postcondition` | Validate output sau khi apply | Đảm bảo resource ở trạng thái mong muốn |
| `provider` | Chọn provider instance | Multi-region, multi-account |
| `provisioner` | Chạy lệnh sau khi tạo/xóa | Trường hợp không có giải pháp tốt hơn |

---

## 6. Câu Hỏi Phỏng Vấn Liên Quan

**Q: Khi nào dùng `depends_on` và tại sao nên tránh lạm dụng?**

A: Dùng khi có phụ thuộc ngầm mà Terraform không tự suy luận được — ví dụ IAM policy phải gán xong trước khi Lambda chạy. Tránh lạm dụng vì `depends_on` buộc Terraform coi toàn bộ source là "có thể ảnh hưởng", làm plan kém chính xác và tạo ra thay thế không cần thiết.

**Q: Khác nhau giữa `prevent_destroy = true` và xóa resource block khỏi code?**

A: `prevent_destroy = true` bảo vệ khi plan **có** destroy action. Nếu xóa resource block khỏi code và chạy apply, Terraform vẫn sẽ xóa resource đó — `prevent_destroy` không giúp trong trường hợp này.

**Q: Tại sao `provisioner` được coi là anti-pattern?**

A: Vì provisioner làm mất tính idempotent — bất biến — của Terraform. Nếu provisioner chạy một phần rồi fail, resource sẽ bị mark là "tainted" và cần destroy rồi tạo lại, gây downtime. Các giải pháp thay thế như User Data, cloud-init, baked AMI, hoặc Ansible idempotent hơn.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Bài trước | [1-dynamic-blocks.md](1-dynamic-blocks.md) |
| → Bài tiếp | [3-for-each-count.md](3-for-each-count.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐ Nâng Cao
