# CloudFormation Stacks — Vòng Đời & Quản Lý

> **Stack** (Ngăn Xếp) là đơn vị triển khai cơ bản của CloudFormation — một tập hợp tài nguyên AWS được tạo, cập nhật và xóa cùng nhau như một đơn vị thống nhất. Mọi thao tác trên stack đều được ghi lại qua **Stack Events** và có cơ chế **Rollback** tự động khi lỗi.

---

## 📚 Mục Lục

1. [Trạng Thái Stack](#trạng-thái-stack)
2. [Tạo Stack — CREATE](#tạo-stack--create)
3. [Cập Nhật Stack — UPDATE](#cập-nhật-stack--update)
4. [Xóa Stack — DELETE](#xóa-stack--delete)
5. [Stack Events — Ghi Nhật Ký Thao Tác](#stack-events--ghi-nhật-ký-thao-tác)
6. [Rollback — Cơ Chế Khôi Phục](#rollback--cơ-chế-khôi-phục)
7. [Stack Policies — Bảo Vệ Tài Nguyên](#stack-policies--bảo-vệ-tài-nguyên)
8. [Nested Stacks — Stack Lồng Nhau](#nested-stacks--stack-lồng-nhau)
9. [Termination Protection](#termination-protection)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Trạng Thái Stack

### Bảng Trạng Thái Đầy Đủ

| Trạng Thái | Ý Nghĩa | Hành Động Có Thể Tiếp Theo |
|-----------|---------|--------------------------|
| `CREATE_IN_PROGRESS` | Đang tạo tài nguyên | Chờ hoàn thành |
| `CREATE_COMPLETE` | Tạo thành công | Update / Delete |
| `CREATE_FAILED` | Tạo thất bại | Xóa stack (rollback tự động trước đó) |
| `UPDATE_IN_PROGRESS` | Đang cập nhật | Chờ hoàn thành |
| `UPDATE_COMPLETE` | Cập nhật thành công | Update / Delete |
| `UPDATE_ROLLBACK_IN_PROGRESS` | Đang rollback sau update lỗi | Chờ |
| `UPDATE_ROLLBACK_COMPLETE` | Rollback thành công | Update lại / Delete |
| `UPDATE_ROLLBACK_FAILED` | **Rollback thất bại** | Can thiệp thủ công (xem bên dưới) |
| `DELETE_IN_PROGRESS` | Đang xóa | Chờ |
| `DELETE_COMPLETE` | Xóa thành công | — |
| `DELETE_FAILED` | Xóa thất bại | Retry / Can thiệp thủ công |
| `ROLLBACK_IN_PROGRESS` | Rollback CREATE lỗi | Chờ |
| `ROLLBACK_COMPLETE` | Rollback CREATE xong | Chỉ có thể Delete |
| `ROLLBACK_FAILED` | Rollback CREATE lỗi | Can thiệp thủ công |
| `REVIEW_IN_PROGRESS` | Đang chờ duyệt Change Set | Approve / Discard |
| `IMPORT_IN_PROGRESS` | Đang import tài nguyên | Chờ |
| `IMPORT_COMPLETE` | Import thành công | Update / Delete |
| `IMPORT_ROLLBACK_IN_PROGRESS` | Rollback import lỗi | Chờ |

### Vòng Đời Trạng Thái Điển Hình

```
                    ┌───────────────────────────────────────┐
                    │            STACK LIFECYCLE             │
                    └───────────────────────────────────────┘

   CREATE:  [Không tồn tại] → CREATE_IN_PROGRESS → CREATE_COMPLETE
                                       │
                                       ▼ (khi lỗi)
                             ROLLBACK_IN_PROGRESS → ROLLBACK_COMPLETE
                                                         │
                                                         ▼
                                                    (chỉ xóa được)

   UPDATE:  CREATE_COMPLETE → UPDATE_IN_PROGRESS → UPDATE_COMPLETE
                                       │
                                       ▼ (khi lỗi)
                          UPDATE_ROLLBACK_IN_PROGRESS → UPDATE_ROLLBACK_COMPLETE

   DELETE:  UPDATE_COMPLETE → DELETE_IN_PROGRESS → DELETE_COMPLETE
```

---

## Tạo Stack — CREATE

### Luồng Xử Lý Khi Tạo

```
1. Validate Template
   - Kiểm tra cú pháp YAML/JSON
   - Kiểm tra resource types hợp lệ
   - Kiểm tra required properties

2. Resolve Dependencies
   - Xây dựng dependency graph từ Ref/GetAtt/DependsOn
   - Tính thứ tự tạo tài nguyên

3. Create Resources (Theo Dependency Order)
   - Tạo song song các resource không phụ thuộc nhau
   - Đợi resource phụ thuộc hoàn thành trước

4. Ghi Stack Events
   - Mỗi resource: CREATE_IN_PROGRESS → CREATE_COMPLETE/FAILED

5. Nếu bất kỳ resource nào FAILED → Trigger Rollback
```

### Lệnh CLI Tạo Stack

```bash
# Tạo stack cơ bản
aws cloudformation create-stack \
  --stack-name prod-vpc-stack \
  --template-body file://vpc-template.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=VpcCidr,ParameterValue=10.0.0.0/16 \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --on-failure DO_NOTHING \    # Không xóa stack khi lỗi (để debug)
  --tags \
    Key=Project,Value=MyApp \
    Key=Owner,Value=platform-team

# Chờ đến khi stack hoàn thành (blocking)
aws cloudformation wait stack-create-complete \
  --stack-name prod-vpc-stack

# Kiểm tra kết quả
aws cloudformation describe-stacks \
  --stack-name prod-vpc-stack \
  --query 'Stacks[0].StackStatus'
```

### on-failure Options

| Giá Trị | Hành Vi Khi Tạo Thất Bại |
|---------|--------------------------|
| `ROLLBACK` | (Mặc định) Xóa tất cả tài nguyên đã tạo |
| `DO_NOTHING` | Giữ nguyên — để debug xem tài nguyên nào lỗi |
| `DELETE` | Xóa luôn cả stack (không giữ lại gì) |

---

## Cập Nhật Stack — UPDATE

### Các Loại Thay Đổi Khi Update

CloudFormation phân loại thay đổi theo **Update Type** của từng property:

| Update Type | Ý Nghĩa | Ví Dụ |
|------------|---------|-------|
| **No Interruption** (Không Gián Đoạn) | Cập nhật tại chỗ, service không downtime | Thêm tag vào EC2 |
| **Some Interruption** (Gián Đoạn Ít) | Tài nguyên restart tạm thời | Đổi instance type EC2 |
| **Replacement** (Thay Thế) | Tạo resource mới, xóa resource cũ | Đổi AMI ID của EC2 |

> **Nguy hiểm với Replacement:** Khi resource bị replace, ID của nó thay đổi. Các resource khác reference resource này bằng Ref/GetAtt sẽ được update theo — có thể gây downtime.

### Luồng Update Không Dùng Change Set

```bash
# Cập nhật trực tiếp (không xem trước)
aws cloudformation update-stack \
  --stack-name prod-vpc-stack \
  --template-body file://vpc-template-v2.yaml \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=VpcCidr,ParameterValue=10.1.0.0/16

# Chờ update hoàn thành
aws cloudformation wait stack-update-complete \
  --stack-name prod-vpc-stack
```

### Chỉ Cập Nhật Parameters (Không Đổi Template)

```bash
# Dùng lại template cũ, chỉ thay đổi parameter
aws cloudformation update-stack \
  --stack-name prod-vpc-stack \
  --use-previous-template \              # Giữ nguyên template hiện tại
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=DesiredCapacity,ParameterValue=5   # Chỉ đổi cái này
```

### Update Behavior — Thứ Tự Xử Lý

```
1. Phân tích sự khác biệt giữa template cũ và mới
2. Tính toán resources cần Create / Modify / Delete
3. Thực hiện thay đổi:
   a. Tạo mới tài nguyên mới (CREATE)
   b. Sửa tài nguyên thay đổi (MODIFY)
   c. Xóa tài nguyên bị loại bỏ (DELETE) — chạy cuối cùng
4. Nếu lỗi → UPDATE_ROLLBACK_IN_PROGRESS → Khôi phục trạng thái cũ
```

---

## Xóa Stack — DELETE

### Thứ Tự Xóa Tài Nguyên

CloudFormation xóa tài nguyên theo thứ tự ngược với thứ tự tạo (LIFO — Last In First Out) — tức là resource phụ thuộc được xóa trước resource nó phụ thuộc vào.

```bash
# Xóa stack
aws cloudformation delete-stack \
  --stack-name prod-vpc-stack

# Xóa có retain một số resource
aws cloudformation delete-stack \
  --stack-name prod-vpc-stack \
  --retain-resources LogBucket DatabaseInstance   # Không xóa các resource này

# Chờ xóa hoàn thành
aws cloudformation wait stack-delete-complete \
  --stack-name prod-vpc-stack
```

### Tại Sao DELETE_FAILED?

```
Lý do phổ biến:
1. S3 Bucket không rỗng → không thể xóa → cần empty bucket trước
2. Security Group đang được EC2/RDS sử dụng → phải xóa resource đó trước
3. IAM Role đang được attach vào EC2 instance profile → cần detach trước
4. DeletionPolicy: Retain → resource không bị xóa nhưng stack vẫn clean up đúng
5. Resource đã bị xóa thủ công bên ngoài CloudFormation (không đồng bộ)
```

### Xử Lý DELETE_FAILED

```bash
# Option 1: Xóa với skip resources lỗi
aws cloudformation delete-stack \
  --stack-name prod-vpc-stack \
  --retain-resources ProblematicS3Bucket

# Option 2: Fix vấn đề rồi retry
# - Empty S3 bucket thủ công
# - Sau đó delete stack bình thường

# Option 3: Force delete (cẩn thận — mất data)
# Thường phải làm qua Console hoặc API với option RetainResources
```

---

## Stack Events — Ghi Nhật Ký Thao Tác

**Stack Events** ghi lại mọi thao tác trên stack — cực kỳ hữu ích để debug lỗi.

### Xem Stack Events

```bash
# Xem tất cả events
aws cloudformation describe-stack-events \
  --stack-name prod-vpc-stack

# Lọc events lỗi
aws cloudformation describe-stack-events \
  --stack-name prod-vpc-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED` || ResourceStatus==`UPDATE_FAILED`]'

# Xem events theo thời gian (mới nhất trước)
aws cloudformation describe-stack-events \
  --stack-name prod-vpc-stack \
  --query 'StackEvents[*].{Time:Timestamp, Resource:LogicalResourceId, Status:ResourceStatus, Reason:ResourceStatusReason}' \
  --output table
```

### Cấu Trúc Stack Event

```json
{
  "StackId": "arn:aws:cloudformation:...",
  "EventId": "vpc-abc123-CREATE_IN_PROGRESS-...",
  "StackName": "prod-vpc-stack",
  "LogicalResourceId": "PublicSubnet",        // Tên logic trong template
  "PhysicalResourceId": "subnet-0abc123def",  // ID thực của resource
  "ResourceType": "AWS::EC2::Subnet",
  "Timestamp": "2026-05-17T10:30:00.000Z",
  "ResourceStatus": "CREATE_FAILED",
  "ResourceStatusReason": "The subnet '10.0.1.0/24' conflicts with another subnet in the VPC."
}
```

### Debug Workflow Với Stack Events

```
1. Stack Status = CREATE_FAILED / UPDATE_FAILED
2. Chạy describe-stack-events
3. Tìm events có ResourceStatus = CREATE_FAILED
4. Đọc ResourceStatusReason → tìm nguyên nhân
5. Fix template → re-deploy

Ví dụ lỗi hay gặp:
- "Access Denied" → IAM Role thiếu quyền
- "Limit exceeded" → Đã đạt quota service limit
- "Resource already exists" → Tài nguyên tên đó đã tồn tại
- "Invalid parameter" → Cấu hình sai trong Properties
```

---

## Rollback — Cơ Chế Khôi Phục

### Rollback Tự Động

```
CREATE lỗi:
Template → CREATE_IN_PROGRESS → (lỗi) → ROLLBACK_IN_PROGRESS
  - Xóa tất cả resource đã tạo thành công
  - Stack về ROLLBACK_COMPLETE (chỉ có thể Delete)

UPDATE lỗi:
UPDATE_IN_PROGRESS → (lỗi) → UPDATE_ROLLBACK_IN_PROGRESS
  - Khôi phục tất cả resource về trạng thái trước update
  - Stack về UPDATE_ROLLBACK_COMPLETE (có thể Update lại)
```

### Disable Rollback — Cho Mục Đích Debug

```bash
# Tạo stack không rollback khi lỗi (giữ resource để debug)
aws cloudformation create-stack \
  --stack-name debug-stack \
  --template-body file://template.yaml \
  --disable-rollback                     # Flag này

# Hoặc dùng on-failure
aws cloudformation create-stack \
  --on-failure DO_NOTHING
```

### UPDATE_ROLLBACK_FAILED — Trạng Thái Nguy Hiểm

Đây là trạng thái phức tạp nhất: Update thất bại, nhưng quá trình rollback cũng thất bại.

```
Nguyên nhân phổ biến:
1. Resource bên ngoài thay đổi trong lúc rollback
2. Service limit đã đạt trong lúc rollback
3. Resource đã bị xóa thủ công bên ngoài

Giải pháp:
1. Dùng Console → "Continue Update Rollback" với Skip Resources
2. CLI:
   aws cloudformation continue-update-rollback \
     --stack-name my-stack \
     --resources-to-skip ProblematicResource

3. Nếu không được → liên hệ AWS Support
```

---

## Stack Policies — Bảo Vệ Tài Nguyên

**Stack Policy** (Chính Sách Stack) ngăn chặn update không mong muốn lên các tài nguyên quan trọng trong stack.

### Cú Pháp Stack Policy

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "Update:Replace",
      "Principal": "*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    }
  ]
}
```

### Áp Dụng Stack Policy

```bash
# Áp dụng khi tạo stack
aws cloudformation create-stack \
  --stack-name prod-stack \
  --template-body file://template.yaml \
  --stack-policy-body file://stack-policy.json

# Cập nhật stack policy
aws cloudformation set-stack-policy \
  --stack-name prod-stack \
  --stack-policy-body file://stack-policy.json

# Override stack policy tạm thời trong lần update này
aws cloudformation update-stack \
  --stack-name prod-stack \
  --template-body file://template-v2.yaml \
  --stack-policy-during-update-body file://override-policy.json
```

### Các Action Trong Stack Policy

| Action | Mô Tả |
|--------|-------|
| `Update:Modify` | Sửa resource (No/Some interruption) |
| `Update:Replace` | Tạo mới resource (Replacement) |
| `Update:Delete` | Xóa resource khỏi stack |
| `Update:*` | Tất cả loại update |

---

## Nested Stacks — Stack Lồng Nhau

**Nested Stack** (Stack Lồng Nhau) cho phép chia nhỏ template lớn thành các template con có thể tái sử dụng.

```
Root Stack
├── Network Stack (nested)
│   ├── VPC
│   ├── Subnets
│   └── Route Tables
├── Security Stack (nested)
│   ├── Security Groups
│   └── NACLs
└── Application Stack (nested)
    ├── EC2 Instances
    └── Load Balancer
```

### Khai Báo Nested Stack

```yaml
# Root template
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack          # Resource type đặc biệt
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/network-template.yaml
      Parameters:
        Environment: !Ref Environment
        VpcCidr: "10.0.0.0/16"
      TimeoutInMinutes: 30

  ApplicationStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack                   # Cần Network trước
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/app-template.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VPCId  # Lấy Output từ nested stack
        SubnetId: !GetAtt NetworkStack.Outputs.PublicSubnetId
```

### Nested Stack vs Cross-Stack Reference

| Tiêu Chí | Nested Stack | Cross-Stack Reference |
|---------|-------------|----------------------|
| **Vòng đời** | Parent quản lý con | Độc lập nhau |
| **Chia sẻ dữ liệu** | GetAtt trên Outputs | ImportValue |
| **Xóa** | Xóa parent → xóa tất cả | Xóa độc lập |
| **Reuse** | Mỗi parent deploy riêng | Một stack, nhiều consumer |
| **Phù hợp khi** | Template quá lớn, cần modular | Teams khác nhau quản lý stack |

---

## Termination Protection

**Termination Protection** (Bảo Vệ Xóa Stack) ngăn stack bị xóa ngẫu nhiên.

```bash
# Bật Termination Protection
aws cloudformation update-termination-protection \
  --stack-name prod-vpc-stack \
  --enable-termination-protection

# Tắt Termination Protection (cần làm trước khi xóa)
aws cloudformation update-termination-protection \
  --stack-name prod-vpc-stack \
  --no-enable-termination-protection

# Kiểm tra trạng thái
aws cloudformation describe-stacks \
  --stack-name prod-vpc-stack \
  --query 'Stacks[0].EnableTerminationProtection'
```

> **Best Practice:** Luôn bật Termination Protection cho stack production. Bật ngay khi tạo stack hoặc sau `CREATE_COMPLETE`.

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa `ROLLBACK_COMPLETE` và `UPDATE_ROLLBACK_COMPLETE` là gì?**
> `ROLLBACK_COMPLETE` xảy ra sau khi **CREATE** thất bại và rollback hoàn tất — stack ở trạng thái này chỉ có thể bị **xóa**, không thể update. `UPDATE_ROLLBACK_COMPLETE` xảy ra sau khi **UPDATE** thất bại và rollback hoàn tất — stack vẫn ở trạng thái ổn định trước update, có thể **tiếp tục update** bình thường.

**Q: Làm thế nào xử lý `UPDATE_ROLLBACK_FAILED`?**
> Dùng `continue-update-rollback` với `--resources-to-skip` để bỏ qua resource đang gây rollback thất bại. CloudFormation sẽ tiếp tục rollback các resource còn lại. Resource bị skip sẽ ở trạng thái không xác định — cần xử lý thủ công sau đó.

**Q: Khi nào nên dùng Nested Stack vs Cross-Stack Reference?**
> Dùng **Nested Stack** khi các template thuộc về một đơn vị triển khai duy nhất (deploy/xóa cùng nhau) và template quá lớn để quản lý. Dùng **Cross-Stack Reference** khi các team khác nhau quản lý stack riêng biệt và cần chia sẻ giá trị như VPC ID, subnet ID — vì vòng đời của chúng độc lập.

**Q: Stack Policy khác IAM Policy thế nào trong bối cảnh CloudFormation?**
> **IAM Policy** kiểm soát ai (user/role) có thể thực hiện thao tác nào trên CloudFormation. **Stack Policy** kiểm soát resource nào trong stack được phép update/replace — ngay cả admin có đủ IAM quyền vẫn bị Stack Policy chặn nếu cố update resource được bảo vệ (trừ khi override bằng `--stack-policy-during-update-body`).

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
