# IaC — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã

> IaC là nền tảng của DevOps hiện đại: biến việc quản lý hạ tầng từ thủ công, không nhất quán thành tự động, có thể lặp lại và kiểm thử được.

---

## 📚 Mục Lục

1. [IaC Là Gì](#1-iac-là-gì)
2. [Tại Sao Cần IaC](#2-tại-sao-cần-iac)
3. [Declarative vs Imperative](#3-declarative-vs-imperative)
4. [So Sánh Các Công Cụ IaC](#4-so-sánh-các-công-cụ-iac)
5. [Vị Trí Của Terraform](#5-vị-trí-của-terraform)
6. [Khi Nào Không Dùng Terraform](#6-khi-nào-không-dùng-terraform)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. IaC Là Gì

**IaC — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã** là phương pháp quản lý và cung cấp hạ tầng (server, network, database, storage) thông qua các file cấu hình có thể đọc được, thay vì qua giao diện tương tác hoặc lệnh thủ công.

### Trước IaC: Quản Lý Thủ Công

```
Timeline điển hình khi không có IaC:

Dev: "Tôi cần một database mới cho tính năng này"
     ↓
Ops: [Login vào AWS Console]
     [Click qua 15 màn hình cấu hình RDS]
     [Đặt tên, chọn instance type, cấu hình security group]
     ↓
     3-5 ngày sau...
     ↓
Dev: "Database này khác với staging, tại sao?"
Ops: "À, tôi nhớ không chính xác các setting của staging..."
```

**Vấn đề:**
- **Snowflake servers — Máy chủ bông tuyết:** Mỗi server khác nhau một chút vì được tạo thủ công
- **Configuration drift — Lệch cấu hình:** Staging và production dần dần khác nhau
- **Tribal knowledge — Kiến thức bộ lạc:** Chỉ một người biết cách tạo môi trường
- **Không thể reproduce — Tái tạo lại:** Disaster recovery — Khôi phục thảm họa — mất nhiều ngày
- **Không có audit trail — Dấu vết kiểm tra:** Không biết ai thay đổi gì, khi nào

### Với IaC

```hcl
# database.tf — Mô tả database bằng code
resource "aws_db_instance" "main" {
  identifier        = "myapp-${var.environment}-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = var.db_instance_class
  allocated_storage = var.db_storage_gb

  db_name  = "myapp"
  username = "admin"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  deletion_protection     = var.environment == "prod" ? true : false

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

**Kết quả:**
- Staging và production **giống hệt nhau** về cấu hình (chỉ khác values)
- Bất kỳ ai trong team đều có thể tạo môi trường mới
- Git history — Lịch sử git — cho biết ai thay đổi gì, khi nào, tại sao
- Tạo môi trường mới trong phút, không phải ngày

---

## 2. Tại Sao Cần IaC

### 2.1 Reproducibility — Khả Năng Tái Tạo

```bash
# Tạo môi trường dev
terraform workspace new dev
terraform apply -var-file=dev.tfvars

# Tạo môi trường staging — GIỐNG HỆT nhưng khác config
terraform workspace new staging
terraform apply -var-file=staging.tfvars

# Tạo môi trường production
terraform workspace new prod
terraform apply -var-file=prod.tfvars
```

Tất cả 3 môi trường được tạo từ cùng một codebase — code gốc, đảm bảo nhất quán.

### 2.2 Version Control — Kiểm Soát Phiên Bản

```bash
# Git history của infrastructure
git log --oneline Devops/terraform/
# a3f9c12 feat: tăng instance type RDS lên db.t3.medium
# b8e2d45 fix: sửa security group rule cho port 5432
# c1a7890 feat: thêm read replica cho database production
# d4f3210 chore: update AWS provider lên 5.x

# Review thay đổi trước khi merge
git diff main feature/add-redis-cache
```

### 2.3 Automation — Tự Động Hóa

```yaml
# CI/CD pipeline — Đường ống tích hợp & triển khai liên tục
# Tự động chạy terraform plan khi có PR — Pull Request
name: Terraform Plan
on: [pull_request]
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Plan
        run: |
          terraform init
          terraform plan -out=tfplan
      - name: Comment Plan Output
        # Post plan output as PR comment
        ...
```

### 2.4 Documentation — Tài Liệu Tự Tạo

Code **chính là** tài liệu. Thay vì tài liệu lỗi thời:

```hcl
# Code mô tả chính xác hạ tầng hiện tại
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  # 2 vCPU, 4GB RAM — đủ cho ~1000 concurrent users
  # Tăng lên t3.large nếu CPU > 70% sustained
}
```

### 2.5 Cost Savings — Tiết Kiệm Chi Phí

```bash
# Xóa môi trường dev cuối ngày, tạo lại sáng hôm sau
# Tiết kiệm 65% chi phí (8 giờ/ngày vs 24 giờ/ngày)

# Buổi tối (CI/CD tự động)
terraform destroy -var-file=dev.tfvars -auto-approve

# Sáng hôm sau
terraform apply -var-file=dev.tfvars -auto-approve
```

---

## 3. Declarative vs Imperative

### Imperative — Mệnh Lệnh (Ansible, Shell Script)

```bash
# Script tạo 3 EC2 instances — Mệnh lệnh từng bước
#!/bin/bash
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro

# Vấn đề: Chạy lần 2 → tạo thêm 3 instances nữa!
# Script không biết hiện trạng — current state
```

### Declarative — Khai Báo (Terraform, CloudFormation)

```hcl
# Terraform — Khai báo trạng thái mong muốn
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-xxx"
  instance_type = "t3.micro"
}

# Chạy lần 2 → Terraform kiểm tra: đã có 3 instances rồi
# Kết quả: "No changes. Infrastructure is up-to-date."

# Muốn 5 instances? Đổi count = 5, chạy apply
# Terraform tự tính: cần tạo thêm 2
```

### So Sánh

| Khía Cạnh | Imperative — Mệnh Lệnh | Declarative — Khai Báo |
|-----------|----------------------|----------------------|
| Tư duy | "Làm thế nào" | "Muốn gì" |
| Idempotency — Bất biến | Phải tự xử lý | Tự động |
| Trạng thái hiện tại | Không biết | Luôn theo dõi |
| Thay đổi hạ tầng | Viết thêm script | Sửa config, re-apply |
| Ví dụ | Ansible, Shell, Python boto3 | Terraform, Pulumi, CloudFormation |

**Idempotency — Bất biến (tính lũy đẳng):** Chạy nhiều lần với cùng input → cùng kết quả. Là tính chất cốt lõi của IaC declarative.

---

## 4. So Sánh Các Công Cụ IaC

### Terraform vs CloudFormation — AWS Native

| Tiêu Chí | Terraform | CloudFormation |
|----------|-----------|----------------|
| **Multi-cloud** | ✅ Hỗ trợ AWS, GCP, Azure, và 3000+ provider | ❌ Chỉ AWS |
| **Ngôn ngữ** | HCL — dễ đọc, ngắn gọn | JSON/YAML — verbose — dài dòng |
| **State** | File riêng, cần quản lý | AWS quản lý (CloudFormation stack) |
| **Community** | Cộng đồng lớn, nhiều module | Nhỏ hơn |
| **Tốc độ** | Nhanh hơn | Chậm hơn (AWS API overhead) |
| **Phù hợp** | Đa cloud, team DevOps | AWS-only shop |

**Ví dụ cùng một task:**

```yaml
# CloudFormation — AWS native (YAML)
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-bucket
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Environment
          Value: prod
```

```hcl
# Terraform (HCL) — ngắn gọn hơn
resource "aws_s3_bucket" "main" {
  bucket = "my-bucket"
  tags   = { Environment = "prod" }
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration { status = "Enabled" }
}
```

### Terraform vs Pulumi

| Tiêu Chí | Terraform | Pulumi |
|----------|-----------|--------|
| **Ngôn ngữ** | HCL (domain-specific — chuyên dụng) | Python, TypeScript, Go, C# |
| **Phù hợp** | Ops-focused teams | Dev-focused teams |
| **Ecosystem** | Mature — Trưởng thành | Đang phát triển |
| **Testing** | Terratest (Go) | Unittest trong ngôn ngữ yêu thích |
| **Learning curve** | Thấp hơn | Cao hơn (phải biết Python/TS) |

```python
# Pulumi — Python
import pulumi
import pulumi_aws as aws

bucket = aws.s3.Bucket("my-bucket",
    versioning=aws.s3.BucketVersioningArgs(enabled=True),
    tags={"Environment": "prod"})
```

### Terraform vs Ansible

> **Lưu ý quan trọng:** Đây là hai công cụ **bổ sung cho nhau**, không phải thay thế.

| Khía Cạnh | Terraform | Ansible |
|-----------|-----------|---------|
| **Mục đích chính** | Provisioning — Cấp phát hạ tầng | Configuration Management — Quản lý cấu hình |
| **Mô hình** | Declarative | Thường Imperative (với idempotency) |
| **State** | Có state file | Stateless — Không có state |
| **Phù hợp** | Tạo EC2, VPC, RDS | Cài phần mềm lên server đã tạo |
| **Agent** | Agentless | Agentless |

**Workflow kết hợp phổ biến:**

```
Terraform → Tạo EC2 instance
     ↓
Ansible → Cài Nginx, cấu hình app, deploy code
     ↓
Terraform → Cập nhật Load Balancer để route traffic
```

### Terraform vs CDK — Cloud Development Kit

| Tiêu Chí | Terraform | AWS CDK |
|----------|-----------|---------|
| **Ngôn ngữ** | HCL | TypeScript, Python, Java, C# |
| **Output** | Terraform plan | CloudFormation template |
| **Multi-cloud** | ✅ | ❌ (chỉ AWS) |
| **Abstraction** | Low-level (gần với API) | High-level (construct library) |

---

## 5. Vị Trí Của Terraform

### Tại Sao Terraform Thống Trị Thị Trường

```
Stack khảo sát DevOps (2024):
  Terraform:        ~80% thị phần IaC
  CloudFormation:   ~15%
  Pulumi:           ~5%
  Ansible (IaC):    ~20% (overlap với Terraform)
```

**Lý do:**

1. **Multi-cloud by design:** Cùng workflow cho AWS, GCP, Azure
2. **Provider ecosystem — Hệ sinh thái Provider:** 3000+ providers (kể cả GitHub, Datadog, PagerDuty)
3. **Terraform Registry — Kho module:** Hàng nghìn module được viết sẵn
4. **Community — Cộng đồng:** Lớn nhất trong không gian IaC
5. **HCL readability — Khả năng đọc HCL:** Dễ học hơn YAML phức tạp của CloudFormation

### Terraform Trong Hệ Sinh Thái DevOps

```
Developer
    │ git push
    ▼
GitHub/GitLab (PR review)
    │ merge
    ▼
CI/CD Pipeline (GitHub Actions / GitLab CI)
    │ terraform plan  →  Review output
    │ terraform apply →  Deploy
    ▼
Cloud Infrastructure (AWS / GCP / Azure)
    │
    ├── Networking (VPC, Subnet, Security Group)
    ├── Compute (EC2, ECS, EKS)
    ├── Database (RDS, ElastiCache)
    └── Storage (S3, EFS)
```

---

## 6. Khi Nào Không Dùng Terraform

### Terraform Không Phù Hợp Cho

1. **Application deployment — Triển khai ứng dụng:** Dùng Helm, ArgoCD, hoặc CI/CD tools
   ```
   ❌ Terraform deploy Docker container mỗi lần code thay đổi
   ✅ Terraform tạo EKS cluster, Kubernetes deploy app
   ```

2. **Configuration management — Quản lý cấu hình server:** Dùng Ansible, Chef
   ```
   ❌ Terraform cài Nginx config, manage files trên server
   ✅ Ansible cài và cấu hình Nginx sau khi Terraform tạo server
   ```

3. **Ad-hoc tasks — Tác vụ tức thời:** Dùng AWS CLI, scripts
   ```
   ❌ Terraform để restart một service
   ✅ AWS CLI / Ansible cho tác vụ one-time
   ```

4. **Thay đổi rất thường xuyên:** Terraform không phù hợp cho resources thay đổi hàng giờ

### Anti-patterns — Kiểu Dùng Sai

```hcl
# ❌ Anti-pattern: Dùng null_resource + local-exec để chạy script
resource "null_resource" "install_app" {
  provisioner "local-exec" {
    command = "ansible-playbook install.yml"
  }
}
# Vấn đề: Không declarative, khó reproduce, không idempotent

# ✅ Thay thế: Tách ra — Terraform tạo infra, Ansible quản lý config
```

---

## 7. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: IaC — Infrastructure as Code — là gì? Lợi ích chính?**

> IaC là phương pháp quản lý hạ tầng bằng các file code có thể version-control được. Lợi ích chính:
> 1. **Reproducibility — Tái tạo:** Tạo môi trường giống hệt nhau mọi lần
> 2. **Automation — Tự động hóa:** Loại bỏ click thủ công
> 3. **Auditability — Khả năng kiểm tra:** Git history theo dõi mọi thay đổi
> 4. **Collaboration — Cộng tác:** PR review cho infrastructure changes
> 5. **Disaster recovery — Khôi phục thảm họa:** Rebuild toàn bộ hạ tầng trong phút

**Q: Terraform khác CloudFormation như thế nào?**

> Terraform là multi-cloud — đa cloud — và dùng HCL dễ đọc hơn. CloudFormation là AWS native và managed state. Tôi chọn Terraform khi cần hỗ trợ nhiều cloud hoặc muốn ecosystem lớn hơn; chọn CloudFormation khi cần deep AWS integration — tích hợp sâu với AWS — không muốn quản lý state file.

**Q: Declarative vs Imperative IaC — khi nào dùng cái nào?**

> Declarative — khai báo (Terraform, CloudFormation) tốt cho provisioning — cấp phát hạ tầng vì idempotent và tự theo dõi state. Imperative — mệnh lệnh (Ansible, scripts) tốt hơn cho configuration management — quản lý cấu hình — nơi cần logic phức tạp hoặc thứ tự thực thi rõ ràng. Trong thực tế, chúng tôi dùng cả hai: Terraform provision infra, Ansible configure servers.

### Câu Hỏi Nâng Cao

**Q: Tại sao Terraform được ưa chuộng hơn Pulumi dù Pulumi cho phép dùng ngôn ngữ lập trình?**

> Terraform's HCL — HashiCorp Configuration Language — có learning curve — đường cong học tập — thấp hơn, và tập trung vào infrastructure logic, không phải programming logic. Với Ops team, HCL dễ tiếp cận hơn TypeScript hay Python. Ngoài ra, Terraform có ecosystem — hệ sinh thái — trưởng thành hơn với hàng nghìn modules đã được kiểm chứng. Tuy nhiên, Pulumi phù hợp hơn cho team developer-heavy cần unit test bằng ngôn ngữ quen thuộc.

---

## 📋 Checklist Tự Đánh Giá

- [ ] Có thể giải thích IaC trong 2 phút không cần notes
- [ ] Phân biệt được declarative vs imperative với ví dụ cụ thể
- [ ] Biết khi nào chọn Terraform vs CloudFormation vs Ansible
- [ ] Hiểu tại sao không commit `terraform.tfstate` lên git
- [ ] Có thể mô tả một workflow IaC hoàn chỉnh từ code đến cloud

---

## 🔗 Liên Kết

- ← [README phần Fundamentals](./README.md)
- → [2. HCL Syntax](./2-hcl-syntax.md) — Bước tiếp theo
- Tham khảo: [Terraform Documentation](https://developer.hashicorp.com/terraform/intro)

---

**Cập Nhật Lần Cuối:** 2026-05-12  
**Trạng Thái:** ✅ Hoàn thành
