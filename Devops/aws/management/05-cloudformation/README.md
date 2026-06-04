# AWS CloudFormation — Infrastructure as Code (Hạ Tầng Dưới Dạng Mã)

> **CloudFormation** là dịch vụ IaC (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã) native của AWS, cho phép mô tả và triển khai toàn bộ hạ tầng AWS bằng file template YAML hoặc JSON. Thay vì click-ops thủ công trên Console, mọi tài nguyên được khai báo dưới dạng code, có thể version-control, review, và tái sử dụng.

---

## 📚 Mục Lục

1. [Tại Sao Dùng CloudFormation?](#tại-sao-dùng-cloudformation)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#các-khái-niệm-cốt-lõi)
4. [Luồng Làm Việc Điển Hình](#luồng-làm-việc-điển-hình)
5. [Các File Trong Module Này](#các-file-trong-module-này)
6. [So Sánh Nhanh: CloudFormation vs CDK vs Terraform](#so-sánh-nhanh)
7. [Câu Hỏi Phỏng Vấn Phổ Biến](#câu-hỏi-phỏng-vấn-phổ-biến)

---

## Tại Sao Dùng CloudFormation?

### Vấn Đề Khi Không Dùng IaC

```
Môi trường được tạo thủ công → Không ai biết chính xác cấu hình → 
Không thể tái tạo → Môi trường dev/prod khác nhau → 
Khó debug sự cố → Không thể audit ai thay đổi gì → 
"Works on my machine" ở cấp độ hạ tầng
```

### Lợi Ích CloudFormation Mang Lại

| Lợi Ích | Mô Tả |
|---------|-------|
| **Reproducibility** (Khả Năng Tái Tạo) | Tạo lại môi trường giống hệt bất cứ lúc nào |
| **Version Control** (Kiểm Soát Phiên Bản) | Template trong Git → history, review, rollback |
| **Drift Detection** (Phát Hiện Trôi Dạt) | Phát hiện thay đổi thủ công ngoài template |
| **Dependency Management** (Quản Lý Phụ Thuộc) | CloudFormation tự tính thứ tự tạo tài nguyên |
| **Rollback Tự Động** | Khi update lỗi, tự động quay về trạng thái ổn định |
| **Cross-Account Deployment** | StackSets triển khai đồng thời hàng trăm account |

---

## Kiến Trúc Tổng Quan

```
Developer / CI-CD Pipeline
         │
         │ CloudFormation Template (YAML/JSON)
         ▼
┌─────────────────────────────────────────┐
│         CloudFormation Service          │
│                                         │
│  Template → Validate → Parse → Plan     │
│       │                                 │
│       ▼                                 │
│  Change Set (tùy chọn — xem trước)      │
│       │                                 │
│       ▼                                 │
│  Stack Execution                        │
│  - Tạo tài nguyên theo dependency order │
│  - Ghi Stack Events (log)               │
│  - Rollback nếu lỗi                     │
└─────────────────────────────────────────┘
         │
         ▼
   AWS Resources
   (EC2, VPC, RDS, Lambda, IAM...)
```

### Các Thành Phần Chính

```
Template (File YAML/JSON)
├── AWSTemplateFormatVersion    ← Phiên bản template (tùy chọn)
├── Description                 ← Mô tả stack
├── Parameters                  ← Tham số đầu vào khi deploy
├── Mappings                    ← Bảng tra cứu (env → AMI ID...)
├── Conditions                  ← Logic có/không tạo resource
├── Resources                   ← [BẮT BUỘC] Danh sách tài nguyên
├── Outputs                     ← Giá trị xuất ra (VPC ID, ARN...)
└── Metadata                    ← Cấu hình giao diện Console (Groups...)

Stack (Tập Hợp Tài Nguyên)
├── Created từ Template
├── Có Stack Name duy nhất trong region
├── Quản lý toàn bộ vòng đời tài nguyên (create/update/delete)
└── Stack Events ghi lại mọi thao tác

StackSet (Triển Khai Đa Account)
├── Deploy cùng một template → nhiều account + region
├── Tích hợp với AWS Organizations
└── Quản lý centralized
```

---

## Các Khái Niệm Cốt Lõi

### Template vs Stack vs StackSet

| Khái Niệm | Mô Tả | Ví Dụ |
|-----------|-------|-------|
| **Template** | File mô tả tài nguyên mong muốn | `vpc-template.yaml` |
| **Stack** | Tập tài nguyên được tạo từ template | `prod-vpc-stack` |
| **StackSet** | Một template deploy tới nhiều account/region | Deploy VPC tới 50 accounts |

### Resource Lifecycle (Vòng Đời Tài Nguyên)

```
CREATE: Template → CloudFormation tạo tài nguyên mới
UPDATE: Template thay đổi → CloudFormation tính delta → Áp dụng thay đổi
DELETE: Xóa stack → Tất cả tài nguyên bị xóa (trừ DeletionPolicy: Retain)
```

### Intrinsic Functions (Hàm Tích Hợp) — Thường Gặp

```yaml
!Ref            # Tham chiếu tài nguyên/parameter
!GetAtt         # Lấy attribute của tài nguyên
!Sub            # Nội suy chuỗi: "arn:aws:s3:::${BucketName}"
!Join           # Nối chuỗi: !Join ["-", [prod, vpc, 01]]
!Select         # Chọn phần tử từ list
!If             # Điều kiện: !If [IsProd, m5.large, t3.micro]
!ImportValue    # Import Output từ stack khác (cross-stack reference)
!FindInMap      # Tra cứu giá trị trong Mappings
```

### Pseudo Parameters (Tham Số Giả) — AWS Tự Cung Cấp

```yaml
AWS::AccountId      # ID tài khoản AWS hiện tại
AWS::Region         # Region đang deploy (vd: ap-southeast-1)
AWS::StackName      # Tên stack hiện tại
AWS::StackId        # ARN đầy đủ của stack
AWS::NoValue        # Dùng để "không set" một property có điều kiện
```

---

## Luồng Làm Việc Điển Hình

### Tạo Mới Stack

```bash
# 1. Validate template trước khi deploy
aws cloudformation validate-template \
  --template-body file://template.yaml

# 2. Tạo stack
aws cloudformation create-stack \
  --stack-name my-production-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=prod \
  --capabilities CAPABILITY_IAM

# 3. Theo dõi tiến trình
aws cloudformation describe-stack-events \
  --stack-name my-production-stack
```

### Update Stack Với Change Set (Khuyến Nghị)

```bash
# 1. Tạo Change Set — xem trước thay đổi
aws cloudformation create-change-set \
  --stack-name my-production-stack \
  --change-set-name update-add-rds \
  --template-body file://template-v2.yaml

# 2. Xem Change Set
aws cloudformation describe-change-set \
  --stack-name my-production-stack \
  --change-set-name update-add-rds

# 3. Áp dụng nếu OK
aws cloudformation execute-change-set \
  --stack-name my-production-stack \
  --change-set-name update-add-rds
```

---

## Các File Trong Module Này

| File | Nội Dung |
|------|----------|
| [1-template-anatomy.md](./1-template-anatomy.md) | Cấu trúc template: Parameters, Resources, Outputs, Mappings, Conditions |
| [2-stacks-lifecycle.md](./2-stacks-lifecycle.md) | Vòng đời stack: Create, Update, Delete, Rollback, Stack Events |
| [3-stacksets-multiregion.md](./3-stacksets-multiregion.md) | StackSets với Organizations — triển khai đa account/region |
| [4-change-sets-drift.md](./4-change-sets-drift.md) | Change Sets, Drift Detection, Import Resources |
| [5-custom-resources.md](./5-custom-resources.md) | Custom Resources với Lambda, resource providers bên thứ ba |
| [6-cdk-comparison.md](./6-cdk-comparison.md) | CDK vs CloudFormation — khi nào dùng công cụ nào |

---

## So Sánh Nhanh

### CloudFormation vs CDK vs Terraform

| Tiêu Chí | CloudFormation | AWS CDK | Terraform |
|----------|---------------|---------|-----------|
| **Ngôn ngữ** | YAML / JSON | Python, TypeScript, Java, Go... | HCL (HashiCorp Config Language) |
| **Native AWS** | ✅ 100% | ✅ Compile ra CFN | ❌ Multi-cloud via provider |
| **State management** | AWS quản lý tự động | AWS quản lý tự động | File state tự quản lý |
| **Abstraction level** | Thấp (1:1 với API) | Cao (Constructs tái sử dụng) | Trung bình |
| **Learning curve** | Trung bình | Cao (cần lập trình) | Trung bình |
| **Multi-cloud** | ❌ Chỉ AWS | ❌ Chủ yếu AWS | ✅ Azure, GCP, AWS... |
| **Ecosystem** | AWS chính thức | AWS chính thức | Cộng đồng rộng lớn |
| **Debugging** | Stack Events | Stack Events + TypeScript errors | Terraform plan |
| **Khi nào dùng** | AWS thuần, không muốn lập trình | Đội dev mạnh, reuse nhiều | Multi-cloud hoặc team đã quen |

> Chi tiết xem [6-cdk-comparison.md](./6-cdk-comparison.md)

---

## Câu Hỏi Phỏng Vấn Phổ Biến

### Cơ Bản

**Q: CloudFormation Stack và StackSet khác nhau thế nào?**
> **Stack** quản lý tài nguyên trong một account + region. **StackSet** triển khai cùng một template tới nhiều account và region đồng thời — thường dùng với AWS Organizations để rollout chính sách/cơ sở hạ tầng chuẩn hoá.

**Q: CAPABILITY_IAM là gì và khi nào cần?**
> Khi template tạo hoặc sửa IAM resources (Role, Policy, User), CloudFormation yêu cầu bạn xác nhận tường minh rằng bạn hiểu rủi ro bằng cờ `--capabilities CAPABILITY_IAM` (hoặc `CAPABILITY_NAMED_IAM` nếu đặt tên IAM resource tùy chỉnh).

**Q: Rollback hoạt động thế nào khi stack update thất bại?**
> CloudFormation tự động rollback về trạng thái trước khi update (trạng thái cuối cùng ổn định). Nếu rollback cũng thất bại, stack rơi vào trạng thái `UPDATE_ROLLBACK_FAILED` — cần can thiệp thủ công (skip resources hoặc liên hệ AWS Support).

### Nâng Cao

**Q: Cross-Stack Reference (Tham Chiếu Chéo Giữa Stack) là gì?**
> Stack A export Output (dùng `Export: Name: ...`), Stack B import bằng `!ImportValue`. Tạo dependency giữa các stack — không thể xóa Stack A khi Stack B còn đang import.

**Q: Drift Detection phát hiện được những gì?**
> Phát hiện sự khác biệt giữa cấu hình thực tế của tài nguyên và cấu hình được định nghĩa trong template. Ví dụ: ai đó vào Console sửa Security Group thủ công → Drift Detection báo `MODIFIED`. Không phải mọi resource type đều hỗ trợ Drift Detection.

**Q: Custom Resource dùng khi nào?**
> Khi CloudFormation không hỗ trợ native một tài nguyên hoặc API call cụ thể. Ví dụ: gọi API bên thứ ba khi deploy, seed database với dữ liệu ban đầu, tạo certificate trên CA nội bộ. Implement bằng Lambda function, CloudFormation gọi Lambda với event Create/Update/Delete.

---

## 🔗 Điều Hướng

| Trước | Module Này | Tiếp Theo |
|-------|-----------|-----------|
| [04-systems-manager/](../04-systems-manager/README.md) | **05-cloudformation/** | [06-organizations/](../06-organizations/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
