# CloudFormation StackSets — Triển Khai Đa Account & Đa Region

> **StackSets** (Bộ Ngăn Xếp) cho phép triển khai một CloudFormation template tới **nhiều AWS account** và **nhiều region** đồng thời. Đây là công cụ thiết yếu cho quản trị đa account (multi-account governance) — đảm bảo hạ tầng chuẩn hoá được rollout nhất quán trên toàn tổ chức.

---

## 📚 Mục Lục

1. [Kiến Trúc StackSets](#kiến-trúc-stacksets)
2. [Hai Mô Hình Quyền](#hai-mô-hình-quyền)
3. [Tạo StackSet](#tạo-stackset)
4. [Deployment Targets — Mục Tiêu Triển Khai](#deployment-targets--mục-tiêu-triển-khai)
5. [Deployment Options — Tuỳ Chọn Triển Khai](#deployment-options--tuỳ-chọn-triển-khai)
6. [Quản Lý Stack Instances](#quản-lý-stack-instances)
7. [StackSets với AWS Organizations](#stacksets-với-aws-organizations)
8. [Auto Deployment — Tự Động Cho Account Mới](#auto-deployment--tự-động-cho-account-mới)
9. [Use Cases Thực Tế](#use-cases-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc StackSets

### Các Khái Niệm Chính

```
StackSet
├── Administrator Account (Tài khoản quản trị)
│   ├── StackSet Definition (Template + cấu hình)
│   └── AWSCloudFormationStackSetAdministrationRole
│
└── Stack Instances (Thực thể trong mỗi account/region)
    ├── Account A / Region 1 → Stack (tài nguyên thực tế)
    ├── Account A / Region 2 → Stack
    ├── Account B / Region 1 → Stack
    └── Account B / Region 2 → Stack
```

**Stack Instance** ≠ **Stack**: Stack Instance là **con trỏ** trong StackSet trỏ đến Stack thực tế trong target account. Một Stack Instance ↔ Một Stack.

### Luồng Xử Lý

```
1. Admin Account gửi StackSet operation (create/update/delete)
2. CloudFormation assume role trong mỗi target account
3. Tạo/cập nhật/xóa Stack trong target account + region
4. Ghi kết quả về StackSet operation status
5. Xử lý song song hoặc tuần tự tùy config
```

---

## Hai Mô Hình Quyền

### 1. Self-Managed Permissions (Quyền Tự Quản Lý)

Cần tạo IAM roles **thủ công** trong cả admin account và target accounts.

```
Admin Account:
  Role: AWSCloudFormationStackSetAdministrationRole
  Trust: cloudformation.amazonaws.com
  Permission: sts:AssumeRole trên StackSetExecutionRole trong target accounts

Target Account(s):
  Role: AWSCloudFormationStackSetExecutionRole
  Trust: Admin Account (chỉ định bằng Account ID)
  Permission: Toàn bộ quyền cần thiết cho template
```

```bash
# Tạo Admin Role (chạy trong admin account)
aws iam create-role \
  --role-name AWSCloudFormationStackSetAdministrationRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "cloudformation.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam put-role-policy \
  --role-name AWSCloudFormationStackSetAdministrationRole \
  --policy-name AssumeRole-AWSCloudFormationStackSetExecutionRole \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/AWSCloudFormationStackSetExecutionRole"
    }]
  }'

# Tạo Execution Role (chạy trong MỖI target account)
aws iam create-role \
  --role-name AWSCloudFormationStackSetExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ADMIN_ACCOUNT_ID:root"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name AWSCloudFormationStackSetExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

### 2. Service-Managed Permissions (Quyền Do Dịch Vụ Quản Lý)

Tích hợp với **AWS Organizations** — CloudFormation tự tạo roles cần thiết, không cần cấu hình IAM thủ công.

```
Yêu cầu:
- Admin Account phải là Management Account hoặc Delegated Administrator
- Organizations trusted access phải được bật cho CloudFormation

Lợi ích:
- Không cần tạo IAM roles thủ công
- Hỗ trợ Auto Deployment (tự động deploy vào account mới)
- Dễ dàng target theo OU (Organizational Unit)
```

```bash
# Bật trusted access cho Organizations
aws organizations enable-aws-service-access \
  --service-principal stacksets.cloudformation.amazonaws.com

# Hoặc set Delegated Administrator
aws organizations register-delegated-administrator \
  --account-id DELEGATED_ADMIN_ACCOUNT_ID \
  --service-principal stacksets.cloudformation.amazonaws.com
```

### So Sánh Hai Mô Hình

| Tiêu Chí | Self-Managed | Service-Managed |
|---------|-------------|-----------------|
| **IAM Roles** | Tạo thủ công | AWS tự tạo |
| **Auto Deployment** | ❌ Không hỗ trợ | ✅ Hỗ trợ |
| **Target theo OU** | ❌ Phải chỉ định account | ✅ Có thể dùng OU ID |
| **Phức tạp setup** | Cao | Thấp |
| **Phù hợp khi** | Không dùng Organizations | Có AWS Organizations |

---

## Tạo StackSet

```bash
# Tạo StackSet với Service-Managed Permissions
aws cloudformation create-stack-set \
  --stack-set-name org-baseline-config \
  --template-body file://baseline-template.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=true \
  --description "CloudTrail và Config cơ sở cho toàn bộ tổ chức" \
  --capabilities CAPABILITY_NAMED_IAM

# Tạo StackSet với Self-Managed Permissions
aws cloudformation create-stack-set \
  --stack-set-name cross-account-monitoring \
  --template-body file://monitoring-template.yaml \
  --permission-model SELF_MANAGED \
  --administration-role-arn arn:aws:iam::ADMIN:role/AWSCloudFormationStackSetAdministrationRole \
  --execution-role-name AWSCloudFormationStackSetExecutionRole \
  --capabilities CAPABILITY_IAM
```

---

## Deployment Targets — Mục Tiêu Triển Khai

### Chỉ Định Theo Account IDs

```bash
aws cloudformation create-stack-instances \
  --stack-set-name org-baseline-config \
  --accounts 111111111111 222222222222 333333333333 \
  --regions ap-southeast-1 us-east-1 eu-west-1 \
  --operation-preferences \
    MaxConcurrentCount=3 \
    FailureToleranceCount=1
```

### Chỉ Định Theo OU (Chỉ Service-Managed)

```bash
aws cloudformation create-stack-instances \
  --stack-set-name org-baseline-config \
  --deployment-targets \
    OrganizationalUnitIds=ou-root-abc123,ou-root-def456 \
  --regions ap-southeast-1 us-east-1 \
  --operation-preferences \
    MaxConcurrentPercentage=25 \
    FailureTolerancePercentage=10
```

### Loại Trừ Account Cụ Thể Khi Target OU

```bash
aws cloudformation create-stack-instances \
  --stack-set-name org-baseline-config \
  --deployment-targets \
    OrganizationalUnitIds=ou-root-abc123 \
    Accounts=999999999999    # Account này bị loại trừ khỏi OU
  --regions ap-southeast-1
# Lưu ý: Accounts trong deployment-targets khi dùng OU = accounts bị exclude
```

---

## Deployment Options — Tuỳ Chọn Triển Khai

**Operation Preferences** kiểm soát cách CloudFormation deploy song song và xử lý lỗi.

```bash
--operation-preferences \
  MaxConcurrentCount=5 \         # Hoặc MaxConcurrentPercentage
  FailureToleranceCount=2 \      # Hoặc FailureTolerancePercentage
  RegionOrder=ap-southeast-1,us-east-1,eu-west-1 \  # Thứ tự region
  RegionConcurrencyType=SEQUENTIAL   # SEQUENTIAL hoặc PARALLEL
```

### Giải Thích Các Tuỳ Chọn

| Option | Ý Nghĩa |
|--------|---------|
| `MaxConcurrentCount` | Số stack instances xử lý đồng thời (absolute number) |
| `MaxConcurrentPercentage` | % tổng số stack instances xử lý đồng thời |
| `FailureToleranceCount` | Số lượng stack failures được phép trước khi dừng operation |
| `FailureTolerancePercentage` | % failures được phép |
| `RegionOrder` | Thứ tự deploy qua các region |
| `RegionConcurrencyType` | `SEQUENTIAL`: từng region một; `PARALLEL`: tất cả regions cùng lúc |

### Chiến Lược Rollout An Toàn

```bash
# Triển khai từng bước: dev trước, prod sau
# Bước 1: Deploy sang dev accounts để test
aws cloudformation create-stack-instances \
  --stack-set-name new-policy-stackset \
  --deployment-targets OrganizationalUnitIds=ou-dev-xxx \
  --regions ap-southeast-1 \
  --operation-preferences MaxConcurrentCount=2,FailureToleranceCount=0

# Bước 2: Sau khi dev OK, deploy staging
aws cloudformation create-stack-instances \
  --stack-set-name new-policy-stackset \
  --deployment-targets OrganizationalUnitIds=ou-staging-yyy \
  --regions ap-southeast-1 \
  --operation-preferences MaxConcurrentCount=5,FailureToleranceCount=1

# Bước 3: Cuối cùng prod — conservative
aws cloudformation create-stack-instances \
  --stack-set-name new-policy-stackset \
  --deployment-targets OrganizationalUnitIds=ou-prod-zzz \
  --regions ap-southeast-1,ap-southeast-2,us-east-1 \
  --operation-preferences \
    MaxConcurrentPercentage=10 \
    FailureToleranceCount=0 \
    RegionOrder=ap-southeast-1,ap-southeast-2,us-east-1 \
    RegionConcurrencyType=SEQUENTIAL
```

---

## Quản Lý Stack Instances

### Cập Nhật StackSet

```bash
# Cập nhật tất cả stack instances
aws cloudformation update-stack-set \
  --stack-set-name org-baseline-config \
  --template-body file://baseline-template-v2.yaml \
  --operation-preferences MaxConcurrentPercentage=20,FailureToleranceCount=5

# Cập nhật chỉ một số accounts/regions
aws cloudformation update-stack-instances \
  --stack-set-name org-baseline-config \
  --accounts 111111111111 \
  --regions ap-southeast-1 \
  --parameter-overrides \
    ParameterKey=LogRetentionDays,ParameterValue=90  # Override riêng cho account này
```

### Xem Trạng Thái Stack Instances

```bash
# Xem tất cả stack instances
aws cloudformation list-stack-instances \
  --stack-set-name org-baseline-config

# Lọc theo account
aws cloudformation list-stack-instances \
  --stack-set-name org-baseline-config \
  --stack-instance-account 111111111111

# Xem trạng thái operation đang chạy
aws cloudformation list-stack-set-operations \
  --stack-set-name org-baseline-config

# Xem kết quả chi tiết của operation
aws cloudformation list-stack-set-operation-results \
  --stack-set-name org-baseline-config \
  --operation-id OPERATION_ID
```

### Xóa Stack Instances

```bash
# Xóa stack instances khỏi một số accounts
aws cloudformation delete-stack-instances \
  --stack-set-name org-baseline-config \
  --accounts 111111111111 \
  --regions ap-southeast-1 \
  --retain-stacks false    # true = giữ stack, false = xóa stack

# Xóa stack instances theo OU
aws cloudformation delete-stack-instances \
  --stack-set-name org-baseline-config \
  --deployment-targets OrganizationalUnitIds=ou-deprecated-xxx \
  --regions ap-southeast-1 \
  --retain-stacks false
```

---

## StackSets với AWS Organizations

### Tích Hợp Toàn Diện

```
Management Account (hoặc Delegated Admin)
├── StackSet: "org-cloudtrail-baseline"
│   └── Targets: Root (toàn bộ Organization)
│       ├── OU: Security → Account: 111, 222
│       ├── OU: Workloads → OU: Dev → Account: 333, 444
│       │                   OU: Prod → Account: 555, 666
│       └── OU: Sandbox → Account: 777 (excluded nếu muốn)
│
└── StackSet: "prod-config-rules"
    └── Targets: OU: Prod only
        └── Account: 555, 666
```

### Delegated Administrator (Quản Trị Ủy Quyền)

```bash
# Management Account ủy quyền cho Security Account
aws organizations register-delegated-administrator \
  --account-id SECURITY_ACCOUNT_ID \
  --service-principal stacksets.cloudformation.amazonaws.com

# Security Account bây giờ có thể tạo/quản lý StackSets
# mà không cần quyền trong Management Account
```

> **Best Practice:** Không chạy workload trong Management Account. Dùng Delegated Administrator pattern để quản lý StackSets từ Security hoặc Operations account.

---

## Auto Deployment — Tự Động Cho Account Mới

Với **Service-Managed Permissions**, khi có account mới gia nhập một OU được target, stack sẽ tự động được tạo.

```bash
# Bật Auto Deployment khi tạo/update StackSet
aws cloudformation create-stack-set \
  --stack-set-name org-baseline-config \
  --permission-model SERVICE_MANAGED \
  --auto-deployment '{
    "Enabled": true,
    "RetainStacksOnAccountRemoval": true
  }'

# Có nghĩa:
# - Account mới vào OU → Stack tự động tạo
# - Account rời OU → Stack vẫn được giữ lại (không bị xóa)
# RetainStacksOnAccountRemoval = false → Account rời OU thì stack bị xóa
```

### Luồng Auto Deployment

```
Account mới tạo (Account Factory / Control Tower)
         │
         ▼
Gia nhập OU "Workloads"
         │
         ▼ (Auto Deployment trigger)
CloudFormation StackSet
         │
         ▼
Tự động tạo stack instances:
- org-cloudtrail-baseline → account mới
- org-config-rules → account mới
- org-security-baseline → account mới
         │
         ▼
Account mới có đầy đủ baseline trong vài phút
```

---

## Use Cases Thực Tế

### 1. Baseline CloudTrail Cho Toàn Tổ Chức

```yaml
# cloudtrail-baseline.yaml
Resources:
  CloudTrail:
    Type: AWS::CloudTrail::Trail
    Properties:
      TrailName: !Sub "${AWS::AccountId}-baseline-trail"
      S3BucketName: !Sub "cloudtrail-logs-${CentralLoggingAccount}"
      IsMultiRegionTrail: true
      IncludeGlobalServiceEvents: true
      IsLogging: true
      EnableLogFileValidation: true
```

```bash
aws cloudformation create-stack-set \
  --stack-set-name org-cloudtrail-baseline \
  --template-body file://cloudtrail-baseline.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=true

aws cloudformation create-stack-instances \
  --stack-set-name org-cloudtrail-baseline \
  --deployment-targets OrganizationalUnitIds=r-root123 \
  --regions ap-southeast-1 us-east-1 eu-west-1 \
  --operation-preferences MaxConcurrentPercentage=25
```

### 2. IAM Password Policy Chuẩn Hoá

```yaml
Resources:
  IAMPasswordPolicy:
    Type: AWS::IAM::AccountPasswordPolicy
    Properties:
      MinimumPasswordLength: 14
      RequireUppercaseCharacters: true
      RequireLowercaseCharacters: true
      RequireNumbers: true
      RequireSymbols: true
      MaxPasswordAge: 90
      PasswordReusePrevention: 24
      HardExpiry: false
      AllowUsersToChangePassword: true
```

### 3. Config Recorder Baseline

```yaml
Resources:
  ConfigRecorder:
    Type: AWS::Config::ConfigurationRecorder
    Properties:
      Name: default
      RoleARN: !GetAtt ConfigRole.Arn
      RecordingGroup:
        AllSupported: true
        IncludeGlobalResourceTypes: true

  ConfigDeliveryChannel:
    Type: AWS::Config::DeliveryChannel
    Properties:
      S3BucketName: !Sub "config-logs-${CentralAccount}-${AWS::Region}"
      ConfigSnapshotDeliveryProperties:
        DeliveryFrequency: TwentyFour_Hours
```

### 4. VPC Flow Logs Baseline

```yaml
Resources:
  FlowLogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "vpc-flowlogs-${AWS::AccountId}-${AWS::Region}"

  VPCFlowLogsDelivery:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceType: VPC
      ResourceId: !Ref ExistingVpcId   # Parameter từ bên ngoài
      TrafficType: ALL
      LogDestinationType: s3
      LogDestination: !GetAtt FlowLogsBucket.Arn
```

---

## Câu Hỏi Phỏng Vấn

**Q: StackSet Stack Instance và Stack khác nhau thế nào?**
> **Stack** là tập tài nguyên AWS thực tế trong một account + region cụ thể. **Stack Instance** là record trong StackSet đại diện cho mối quan hệ giữa StackSet và Stack đó — chứa thông tin như trạng thái, parameter override, account/region. Một Stack Instance → Một Stack. Xóa Stack Instance → Xóa Stack tương ứng (nếu `retain-stacks=false`).

**Q: Self-Managed vs Service-Managed — khi nào chọn cái nào?**
> **Service-Managed** nên là lựa chọn mặc định khi có AWS Organizations vì đơn giản hơn (không cần tạo IAM roles), hỗ trợ target theo OU và Auto Deployment. **Self-Managed** dùng khi không có Organizations, hoặc khi admin account và target accounts không cùng Organization.

**Q: StackSets có rollback tự động không?**
> Có, nhưng ở cấp độ từng stack instance. Nếu một stack instance update thất bại, nó sẽ rollback về trạng thái trước. `FailureToleranceCount/Percentage` quyết định bao nhiêu failures được chấp nhận trước khi dừng toàn bộ operation — nhưng các stack instance đã thành công **không** bị rollback.

**Q: Auto Deployment hoạt động thế nào với Control Tower?**
> Control Tower dùng StackSets internally để deploy baseline vào account mới qua Account Factory. Khi bạn tạo thêm StackSet của riêng mình với Service-Managed và Auto Deployment enabled, nó hoạt động song song — account mới từ Account Factory sẽ trigger cả StackSets của Control Tower lẫn StackSets custom của bạn.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
