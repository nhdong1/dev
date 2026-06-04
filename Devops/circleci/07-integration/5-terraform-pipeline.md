# Terraform Plan/Apply Trong CircleCI

> **Terraform** — Công Cụ IaC (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã): quản lý tài nguyên cloud bằng file `.tf`. Pipeline chạy **plan** trên mọi thay đổi, **apply** có kiểm soát lên môi trường nhạy cảm.

## 📚 Mục Lục

1. [Luồng CI Cho Terraform](#luồng-ci-cho-terraform)
2. [Cấu Hình Job Cơ Bản](#cấu-hình-job-cơ-bản)
3. [Remote State — Trạng Thái Từ Xa](#remote-state--trạng-thái-từ-xa)
4. [Plan Trên PR, Apply Sau Approval](#plan-trên-pr-apply-sau-approval)
5. [OIDC Thay Cho Static Keys](#oidc-thay-cho-static-keys)
6. [Policy & Security](#policy--security)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Luồng CI Cho Terraform

```
fmt/validate → init → plan (lưu plan file) → [approval] → apply
```

| Bước | Mục Đích |
|------|----------|
| `terraform fmt -check` | Đồng nhất style |
| `terraform validate` | Cú pháp và cấu hình provider |
| `terraform plan` | Xem diff, không thay đổi hạ tầng |
| `terraform apply` | Áp dụng thay đổi (có rủi ro) |

---

## Cấu Hình Job Cơ Bản

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  terraform-plan:
    docker:
      - image: hashicorp/terraform:1.6
    working_directory: ~/project/infra
    steps:
      - checkout
      - aws-cli/setup:
          role-arn: $TF_AWS_ROLE_ARN
          region: ap-southeast-1
      - run:
          name: Terraform Init
          command: terraform init -input=false
      - run:
          name: Terraform Plan
          command: |
            terraform plan -input=false -out=tfplan
      - persist_to_workspace:
          root: .
          paths:
            - infra/tfplan
            - infra/.terraform

  terraform-apply:
    docker:
      - image: hashicorp/terraform:1.6
    working_directory: ~/project/infra
    steps:
      - attach_workspace:
          at: ~/project
      - aws-cli/setup:
          role-arn: $TF_AWS_ROLE_ARN
          region: ap-southeast-1
      - run:
          name: Terraform Apply
          command: terraform apply -input=false -auto-approve tfplan
```

**Lưu ý:** `working_directory` phải trỏ đúng thư mục chứa `.tf` files.

---

## Remote State — Trạng Thái Từ Xa

Không dùng local state trên runner (mất sau job). Backend phổ biến:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**DynamoDB** — bảng lock tránh hai pipeline apply đồng thời (state corruption).

CircleCI job cần quyền S3 read/write và DynamoDB lock trên backend đó.

---

## Plan Trên PR, Apply Sau Approval

```yaml
workflows:
  infrastructure:
    jobs:
      - terraform-plan:
          filters:
            branches:
              ignore: main
      - terraform-plan-main:
          filters:
            branches:
              only: main
      - hold-apply:
          type: approval
          requires:
            - terraform-plan-main
          filters:
            branches:
              only: main
      - terraform-apply:
          requires:
            - hold-apply
          context: terraform-production
```

Comment plan output lên PR: dùng `terraform show -no-color tfplan` và API GitHub (orb hoặc script).

---

## OIDC Thay Cho Static Keys

IAM role trust CircleCI OIDC provider, policy cho phép:

- `s3:GetObject`, `PutObject` trên state bucket
- `dynamodb:GetItem`, `PutItem`, `DeleteItem` trên lock table
- Quyền tạo/sửa resource theo module (VPC, RDS, …) — scope ARN cụ thể

Không lưu `AWS_ACCESS_KEY_ID` dài hạn trong Context cho Terraform.

---

## Policy & Security

| Công Cụ | Vai Trò |
|---------|---------|
| **tfsec** / **checkov** | Scan misconfiguration trước plan |
| **OPA — Open Policy Agent** | Policy as code trên plan JSON |
| **Sentinel** (Terraform Cloud) | Enterprise policy (nếu dùng TFC) |

```yaml
      - run:
          name: tfsec
          command: |
            docker run --rm -v $(pwd):/src aquasec/tfsec /src
```

---

## Best Practices

1. **Pin provider versions** trong `required_providers`
2. Một state file per **blast radius** (không nhét cả org vào một state)
3. Không commit `.tfvars` chứa secrets; dùng CircleCI env hoặc Vault
4. `terraform apply` chỉ từ plan file đã lưu (`-auto-approve tfplan`) — tránh drift giữa plan và apply
5. Destroy production: job riêng + approval kép

---

## Câu Hỏi Phỏng Vấn

**Tại sao cần remote state?**  
Runner ephemeral; local state không chia sẻ giữa job/member team → conflict và mất dữ liệu.

**Plan failed nhưng apply vẫn chạy — tránh thế nào?**  
Workflow `requires` chain; apply job chỉ attach workspace từ plan job thành công.

**Terraform vs CloudFormation trong CI?**  
Cùng IaC; Terraform đa cloud. CircleCI chỉ chạy CLI — pattern plan/apply tương tự.
