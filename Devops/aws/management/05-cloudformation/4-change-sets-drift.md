# Change Sets & Drift Detection — Kiểm Soát Thay Đổi & Phát Hiện Trôi Dạt

> **Change Set** (Bộ Thay Đổi) cho phép xem trước chính xác những gì sẽ xảy ra khi bạn update stack trước khi thực sự áp dụng — tương tự `terraform plan`. **Drift Detection** (Phát Hiện Trôi Dạt) phát hiện khi tài nguyên bị thay đổi thủ công bên ngoài CloudFormation, gây ra sự khác biệt giữa trạng thái thực tế và trạng thái được định nghĩa trong template.

---

## 📚 Mục Lục

1. [Change Sets — Xem Trước Thay Đổi](#change-sets--xem-trước-thay-đổi)
2. [Drift Detection — Phát Hiện Trôi Dạt](#drift-detection--phát-hiện-trôi-dạt)
3. [Import Resources — Nhập Tài Nguyên Hiện Có](#import-resources--nhập-tài-nguyên-hiện-có)
4. [Refactoring — Đổi Logical ID](#refactoring--đổi-logical-id)
5. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Change Sets — Xem Trước Thay Đổi

### Tại Sao Cần Change Set?

```
Vấn đề:
UpdateStack trực tiếp → Không biết resource nào bị Replace (thay thế hoàn toàn)
→ EC2 instance bị terminate → Downtime bất ngờ
→ RDS instance bị xóa và tạo lại → Mất dữ liệu

Giải pháp: Change Set
1. Tạo Change Set → Xem preview thay đổi
2. Xác nhận không có Replacement nguy hiểm
3. Execute Change Set → Áp dụng thay đổi an toàn
```

### Luồng Làm Việc Change Set

```
Template V2                Change Set                 Stack
(mới)         →  Create  →  (preview)    →  Execute  →  (updated)
                              │
                              │ Xem:
                              ├── Resource: MyEC2
                              │   Action: Modify
                              │   Replacement: False ✅
                              │
                              ├── Resource: MyRDS
                              │   Action: Modify
                              │   Replacement: True ⚠️ (NGUY HIỂM)
                              │
                              └── Resource: MyNewBucket
                                  Action: Add ✅
```

### Tạo và Quản Lý Change Set

```bash
# 1. Tạo Change Set
aws cloudformation create-change-set \
  --stack-name prod-application-stack \
  --change-set-name add-elasticache-v2 \
  --template-body file://template-v2.yaml \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=CacheNodeType,ParameterValue=cache.t3.micro \
  --description "Thêm ElastiCache cluster cho session caching" \
  --capabilities CAPABILITY_IAM

# 2. Chờ Change Set sẵn sàng
aws cloudformation wait change-set-create-complete \
  --stack-name prod-application-stack \
  --change-set-name add-elasticache-v2

# 3. Xem chi tiết Change Set
aws cloudformation describe-change-set \
  --stack-name prod-application-stack \
  --change-set-name add-elasticache-v2

# 4a. Execute nếu OK
aws cloudformation execute-change-set \
  --stack-name prod-application-stack \
  --change-set-name add-elasticache-v2

# 4b. Delete nếu không muốn áp dụng
aws cloudformation delete-change-set \
  --stack-name prod-application-stack \
  --change-set-name add-elasticache-v2

# Liệt kê tất cả change sets của stack
aws cloudformation list-change-sets \
  --stack-name prod-application-stack
```

### Đọc Kết Quả Change Set

```json
{
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Modify",                    // Add | Modify | Remove | Import | Dynamic
        "LogicalResourceId": "WebServerASG",
        "PhysicalResourceId": "prod-web-asg",
        "ResourceType": "AWS::AutoScaling::AutoScalingGroup",
        "Replacement": "False",                // True | False | Conditional
        "Scope": ["Properties"],               // Phần nào của resource thay đổi
        "Details": [
          {
            "Target": {
              "Attribute": "Properties",
              "Name": "MinSize",               // Property nào thay đổi
              "RequiresRecreation": "Never"    // Never | Conditionally | Always
            },
            "Evaluation": "Static",            // Static | Dynamic
            "ChangeSource": "DirectModification"
          }
        ]
      }
    },
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Add",
        "LogicalResourceId": "ElastiCacheCluster",
        "ResourceType": "AWS::ElastiCache::CacheCluster",
        "Replacement": "N/A"
      }
    }
  ]
}
```

### Giải Mã Replacement Values

| Replacement | Ý Nghĩa | Mức Độ Nguy Hiểm |
|------------|---------|-----------------|
| `False` | Resource được update tại chỗ | ✅ An toàn |
| `True` | Resource bị xóa và tạo mới hoàn toàn | ⚠️ Nguy hiểm — mất data, đổi ID |
| `Conditional` | Phụ thuộc vào giá trị runtime — không biết chắc | ⚠️ Cần kiểm tra kỹ |

### Change Set Cho Stack Mới (CREATE_PENDING)

```bash
# Tạo Change Set cho stack chưa tồn tại
aws cloudformation create-change-set \
  --stack-name new-stack \
  --change-set-name initial-create \
  --change-set-type CREATE \           # Mặc định là UPDATE
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=prod

# Xem preview toàn bộ tài nguyên sẽ được tạo
aws cloudformation describe-change-set \
  --stack-name new-stack \
  --change-set-name initial-create

# Execute để tạo stack
aws cloudformation execute-change-set \
  --stack-name new-stack \
  --change-set-name initial-create
```

---

## Drift Detection — Phát Hiện Trôi Dạt

### Khái Niệm Drift

**Drift** (Trôi Dạt) là sự khác biệt giữa cấu hình **thực tế** của tài nguyên và cấu hình **mong muốn** được định nghĩa trong CloudFormation template.

```
Template (expected):          Actual (thực tế):
Security Group:               Security Group:
  Inbound: 443 from 0.0.0.0/0   Inbound: 443 from 0.0.0.0/0
                                 Inbound: 22 from 0.0.0.0/0  ← ai đó thêm thủ công
→ DRIFTED!
```

### Trigger Drift Detection

```bash
# Phát hiện drift cho toàn bộ stack
aws cloudformation detect-stack-drift \
  --stack-name prod-application-stack

# Lưu Drift Detection ID
DRIFT_ID=$(aws cloudformation detect-stack-drift \
  --stack-name prod-application-stack \
  --query 'StackDriftDetectionId' \
  --output text)

# Chờ drift detection hoàn thành (có thể mất vài phút)
aws cloudformation wait stack-drift-detection-complete \
  --stack-drift-detection-id $DRIFT_ID

# Xem kết quả tổng quan
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id $DRIFT_ID
```

### Xem Chi Tiết Drift

```bash
# Xem tất cả resource drift của stack
aws cloudformation describe-stack-resource-drifts \
  --stack-name prod-application-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED

# Xem drift của một resource cụ thể
aws cloudformation detect-stack-resource-drift \
  --stack-name prod-application-stack \
  --logical-resource-id WebServerSecurityGroup
```

### Trạng Thái Drift

| Trạng Thái Stack | Ý Nghĩa |
|-----------------|---------|
| `DRIFTED` | Ít nhất một resource bị drift |
| `IN_SYNC` | Tất cả resources khớp với template |
| `NOT_CHECKED` | Drift detection chưa chạy |

| Trạng Thái Resource | Ý Nghĩa |
|--------------------|---------|
| `MODIFIED` | Resource tồn tại nhưng cấu hình đã thay đổi |
| `DELETED` | Resource trong template nhưng không còn tồn tại thực tế |
| `IN_SYNC` | Khớp hoàn toàn với template |
| `NOT_CHECKED` | Resource type không hỗ trợ drift detection |

### Kết Quả Drift Chi Tiết

```json
{
  "StackResourceDrifts": [
    {
      "LogicalResourceId": "WebServerSecurityGroup",
      "ResourceType": "AWS::EC2::SecurityGroup",
      "PhysicalResourceId": "sg-0abc123def",
      "StackResourceDriftStatus": "MODIFIED",
      "ExpectedProperties": "{\"GroupDescription\":\"Web Server SG\",\"SecurityGroupIngress\":[{\"IpProtocol\":\"tcp\",\"FromPort\":443,\"ToPort\":443,\"CidrIp\":\"0.0.0.0/0\"}]}",
      "ActualProperties": "{\"GroupDescription\":\"Web Server SG\",\"SecurityGroupIngress\":[{\"IpProtocol\":\"tcp\",\"FromPort\":443,\"ToPort\":443,\"CidrIp\":\"0.0.0.0/0\"},{\"IpProtocol\":\"tcp\",\"FromPort\":22,\"ToPort\":22,\"CidrIp\":\"0.0.0.0/0\"}]}",
      "PropertyDifferences": [
        {
          "PropertyPath": "/SecurityGroupIngress/1",
          "ExpectedValue": null,
          "ActualValue": "{\"IpProtocol\":\"tcp\",\"FromPort\":22,\"ToPort\":22,\"CidrIp\":\"0.0.0.0/0\"}",
          "DifferenceType": "ADD"    // ADD | REMOVE | NOT_EQUAL
        }
      ]
    }
  ]
}
```

### Xử Lý Drift

```
Sau khi phát hiện Drift có 3 lựa chọn:

1. Remediate — Khắc phục (khuyến nghị):
   Update template để reflect thay đổi thủ công (nếu thay đổi hợp lệ)
   → Chạy `update-stack` để sync lại

2. Revert — Hoàn tác:
   Update stack với template gốc (force overwrite thay đổi thủ công)
   → Thay đổi thủ công sẽ bị xóa

3. Accept — Chấp nhận (không khuyến nghị):
   Bỏ qua drift — rủi ro cao vì mất sync giữa template và thực tế

Automation:
EventBridge → detect drift hàng ngày → Lambda → notify team → ticket
```

### Resource Types Hỗ Trợ Drift Detection

> Không phải mọi resource type đều hỗ trợ drift detection. Kiểm tra danh sách tại AWS Documentation.

**Hỗ trợ đầy đủ:**
- EC2: Instances, SecurityGroups, VPCs, Subnets, Route Tables
- IAM: Roles, Policies, Groups
- S3: Buckets (một số properties)
- RDS: DBInstances, DBClusters
- Lambda: Functions, EventSourceMappings
- CloudWatch: Alarms, LogGroups

**Không hỗ trợ hoặc hỗ trợ giới hạn:**
- Custom Resources
- Một số resource properties phức tạp

---

## Import Resources — Nhập Tài Nguyên Hiện Có

**Resource Import** cho phép đưa tài nguyên AWS **đã tồn tại** (tạo ngoài CloudFormation) vào quản lý bởi một stack.

### Khi Nào Dùng Resource Import?

```
Use Cases:
1. Team tạo thủ công tài nguyên trên Console → muốn quản lý bằng IaC
2. Migrate từ Terraform/ARM sang CloudFormation
3. Tách stack lớn thành stack nhỏ hơn
4. Recover stack sau sự cố (DELETE_FAILED, orphaned resources)
```

### Điều Kiện Để Import

- Resource type phải hỗ trợ import (xem danh sách CloudFormation docs)
- Resource chưa thuộc về bất kỳ CloudFormation stack nào
- Cần identifier của resource (physical resource ID)

### Quy Trình Import

```bash
# Bước 1: Chuẩn bị template với resource mới
# Template phải có resource với Logical ID mới
# Template KHÔNG ĐƯỢC thay đổi resources đang quản lý

# Bước 2: Tạo file resource identifiers
cat > resources-to-import.json << 'EOF'
[
  {
    "ResourceType": "AWS::S3::Bucket",
    "LogicalResourceId": "ExistingProductionBucket",
    "ResourceIdentifier": {
      "BucketName": "my-existing-production-bucket"
    }
  },
  {
    "ResourceType": "AWS::DynamoDB::Table",
    "LogicalResourceId": "ExistingUserTable",
    "ResourceIdentifier": {
      "TableName": "users-production"
    }
  }
]
EOF

# Bước 3: Tạo Change Set kiểu IMPORT
aws cloudformation create-change-set \
  --stack-name prod-application-stack \
  --change-set-name import-existing-resources \
  --change-set-type IMPORT \
  --template-body file://template-with-imports.yaml \
  --resources-to-import file://resources-to-import.json

# Bước 4: Review Change Set (phải thấy Action = "Import")
aws cloudformation describe-change-set \
  --stack-name prod-application-stack \
  --change-set-name import-existing-resources

# Bước 5: Execute
aws cloudformation execute-change-set \
  --stack-name prod-application-stack \
  --change-set-name import-existing-resources
```

### Template Khi Import

```yaml
# template-with-imports.yaml
Resources:
  # Các resources HIỆN CÓ trong stack — KHÔNG thay đổi
  ExistingEC2:
    Type: AWS::EC2::Instance
    Properties:
      # ... giữ nguyên

  # Resource MỚI — sẽ được IMPORT từ bên ngoài
  ExistingProductionBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain          # Quan trọng: Retain để không xóa bucket
    Properties:
      BucketName: my-existing-production-bucket  # Phải match tên bucket thực
      # Chỉ cần properties mà CloudFormation cần để quản lý
      # Không cần khai báo mọi property — drift detection sẽ phát hiện sau

  ExistingUserTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain
    Properties:
      TableName: users-production
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
```

### Resource Identifiers Thường Gặp

| Resource Type | Identifier Key |
|--------------|----------------|
| `AWS::S3::Bucket` | `BucketName` |
| `AWS::DynamoDB::Table` | `TableName` |
| `AWS::EC2::Instance` | `InstanceId` |
| `AWS::EC2::SecurityGroup` | `GroupId` |
| `AWS::IAM::Role` | `RoleName` |
| `AWS::RDS::DBInstance` | `DBInstanceIdentifier` |
| `AWS::Lambda::Function` | `FunctionName` |
| `AWS::SQS::Queue` | `QueueUrl` |
| `AWS::SNS::Topic` | `TopicArn` |

---

## Refactoring — Đổi Logical ID

Đôi khi cần đổi tên Logical ID của resource trong template (refactor) mà không muốn xóa và tạo lại resource.

### Vấn Đề

```yaml
# Template cũ — Logical ID không rõ nghĩa
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: prod-app-assets

# Template mới — Muốn đổi thành tên rõ nghĩa hơn
Resources:
  ProductionAssetsBucket:    # Đổi Logical ID
    Type: AWS::S3::Bucket
    Properties:
      BucketName: prod-app-assets

# VẤN ĐỀ: Nếu update trực tiếp:
# - CloudFormation thấy: Delete "MyBucket", Create "ProductionAssetsBucket"
# → Bucket bị XÓA và tạo lại → Mất dữ liệu!
```

### Giải Pháp: Import với Logical ID Mới

```bash
# Bước 1: Xóa resource khỏi stack (retain resource)
# Option A: Thêm DeletionPolicy: Retain vào resource cũ, rồi xóa khỏi template
# Option B: Dùng retain-resources khi delete

# Bước 2: Import với Logical ID mới
cat > refactor-import.json << 'EOF'
[
  {
    "ResourceType": "AWS::S3::Bucket",
    "LogicalResourceId": "ProductionAssetsBucket",
    "ResourceIdentifier": {
      "BucketName": "prod-app-assets"
    }
  }
]
EOF

aws cloudformation create-change-set \
  --stack-name prod-stack \
  --change-set-name refactor-bucket-id \
  --change-set-type IMPORT \
  --template-body file://template-with-new-id.yaml \
  --resources-to-import file://refactor-import.json
```

---

## Câu Hỏi Phỏng Vấn

**Q: Change Set và trực tiếp update-stack khác nhau gì? Khi nào nên dùng Change Set?**
> `update-stack` áp dụng ngay lập tức mà không cho xem trước. **Change Set** tạo ra preview để bạn review trước khi execute — đặc biệt quan trọng để phát hiện `Replacement: True` (tài nguyên sẽ bị xóa và tạo lại). Nên dùng Change Set trong mọi môi trường production và khi template thay đổi phức tạp. CI/CD pipeline tốt luôn dùng Change Set + manual approval step trước khi execute.

**Q: Drift Detection có tự động khắc phục drift không?**
> Không — Drift Detection chỉ **phát hiện** và báo cáo sự khác biệt, không tự sửa. Để khắc phục drift, bạn phải chủ động: hoặc cập nhật template để phản ánh thay đổi thực tế (nếu thay đổi hợp lệ), hoặc chạy `update-stack` với template gốc để overwrite thay đổi thủ công.

**Q: Không phải mọi resource type đều hỗ trợ Drift Detection — ảnh hưởng thế nào đến chiến lược quản trị?**
> Resource không hỗ trợ drift detection hiển thị status `NOT_CHECKED`. Điều này nghĩa là bạn cần bổ sung cơ chế khác để giám sát chúng — ví dụ: AWS Config Rules để phát hiện thay đổi cấu hình. Chiến lược tốt là kết hợp CloudFormation Drift Detection (cho resources hỗ trợ) + AWS Config (cho tất cả resources).

**Q: Sau khi Import resource vào stack, resource có bị thay đổi không?**
> Bản thân tài nguyên không bị thay đổi ngay khi import. Tuy nhiên, khi bạn **update stack sau đó**, CloudFormation sẽ bắt đầu quản lý resource đó theo template — nếu cấu hình thực tế khác template, update sẽ điều chỉnh resource để match template. Đó là lý do sau khi import nên ngay lập tức chạy Drift Detection để xác nhận template đã phản ánh đúng cấu hình thực tế.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
