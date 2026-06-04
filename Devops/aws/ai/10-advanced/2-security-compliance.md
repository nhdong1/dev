# Bảo Mật & Tuân Thủ AWS AI/ML — Security & Compliance

> Hướng dẫn toàn diện về bảo mật ML workload trên AWS: từ VPC isolation (Cô Lập Mạng), IAM least-privilege (Quyền Tối Thiểu), encryption (Mã Hóa) đến audit logging (Nhật Ký Kiểm Toán) và compliance frameworks (Khung Tuân Thủ)

## 📋 Mục Lục

1. [Mô Hình Bảo Mật Chia Sẻ](#mô-hình-bảo-mật-chia-sẻ)
2. [IAM cho SageMaker](#iam-cho-sagemaker)
3. [Network Security — Bảo Mật Mạng](#network-security--bảo-mật-mạng)
4. [Encryption — Mã Hóa Dữ Liệu](#encryption--mã-hóa-dữ-liệu)
5. [VPC Configuration cho SageMaker](#vpc-configuration-cho-sagemaker)
6. [Private Endpoints — Điểm Cuối Riêng](#private-endpoints--điểm-cuối-riêng)
7. [Data Privacy & PII Detection](#data-privacy--pii-detection)
8. [Audit Logging — Nhật Ký Kiểm Toán](#audit-logging--nhật-ký-kiểm-toán)
9. [Compliance Frameworks](#compliance-frameworks)
10. [Checklist Bảo Mật](#checklist-bảo-mật)

---

## Mô Hình Bảo Mật Chia Sẻ

**Shared Responsibility Model** (Mô Hình Trách Nhiệm Chia Sẻ) — AWS và khách hàng cùng chịu trách nhiệm bảo mật:

```
AWS chịu trách nhiệm ("Security OF the Cloud"):
├── Physical security của data centers
├── Network infrastructure (backbone AWS)
├── Hypervisor và host OS bên dưới instances
├── Managed service software (SageMaker control plane)
└── Hardware (GPU, CPU, storage)

Khách hàng chịu trách nhiệm ("Security IN the Cloud"):
├── IAM policies và user management
├── Network configuration (VPC, Security Groups, NACLs)
├── Encryption of data (at rest và in transit)
├── Training data và model artifacts
├── Application-level security (API keys trong code, v.v.)
└── Compliance và governance
```

**Cho AI/ML workloads, rủi ro đặc thù:**
- **Data poisoning** (Đầu Độc Dữ Liệu): Kẻ tấn công inject dữ liệu độc hại vào training set
- **Model theft** (Đánh Cắp Mô Hình): Lấy cắp model artifacts có IP value cao
- **Inference attacks** (Tấn Công Suy Luận): Truy vấn model để suy ra training data (membership inference)
- **PII leakage** (Rò Rỉ Dữ Liệu Cá Nhân): Training data chứa PII bị lộ qua model outputs

---

## IAM cho SageMaker

### Nguyên Tắc Least Privilege — Quyền Tối Thiểu

Mỗi SageMaker component cần một **IAM Role** (Vai Trò IAM) riêng với chỉ những quyền cần thiết tối thiểu.

```
SageMaker IAM Architecture:
├── SageMaker Execution Role        Quyền cho training/inference containers
├── SageMaker Studio Role           Quyền cho data scientists dùng Studio
├── Pipeline Execution Role         Quyền cho SageMaker Pipelines chạy steps
├── Model Registry Approval Role    Quyền approve/reject model versions
└── Monitoring Role                 Quyền cho Model Monitor đọc/ghi logs
```

### SageMaker Execution Role — Vai Trò Thực Thi

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadTrainingData",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-training-bucket",
        "arn:aws:s3:::my-training-bucket/data/*"
      ]
    },
    {
      "Sid": "S3WriteModelArtifacts",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-model-bucket/models/*"
    },
    {
      "Sid": "ECRReadImages",
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KMSDecryptForTraining",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789:key/my-training-key"
    }
  ]
}
```

### Trust Policy — Chính Sách Tin Tưởng

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "sagemaker.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Phân Tách Vai Trò Theo Môi Trường

```python
import boto3

# Development environment
dev_role = "arn:aws:iam::123456789:role/sagemaker-dev-execution-role"

# Production environment — quyền bị giới hạn hơn
prod_role = "arn:aws:iam::123456789:role/sagemaker-prod-execution-role"

# Training với dev role
estimator = Estimator(
    role=dev_role if environment == "dev" else prod_role,
    # ...
)
```

### Service Control Policies (SCP) — Chính Sách Kiểm Soát Dịch Vụ

SCP áp dụng ở cấp AWS Organization, không thể bị override bởi IAM policies trong account con:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireEncryptionForSageMaker",
      "Effect": "Deny",
      "Action": "sagemaker:CreateTrainingJob",
      "Resource": "*",
      "Condition": {
        "Null": {
          "sagemaker:VolumeKmsKey": "true"
        }
      }
    },
    {
      "Sid": "RequireVPCForSageMaker",
      "Effect": "Deny",
      "Action": "sagemaker:CreateTrainingJob",
      "Resource": "*",
      "Condition": {
        "Null": {
          "sagemaker:VpcSubnets": "true"
        }
      }
    }
  ]
}
```

---

## Network Security — Bảo Mật Mạng

### Defense in Depth — Bảo Mật Theo Chiều Sâu

```
Internet
    │
    ▼
[AWS WAF — Web Application Firewall]        Lọc request độc hại
    │
    ▼
[API Gateway / ALB]                          Load balancer công khai
    │
    ▼
[VPC — Virtual Private Cloud]               Mạng ảo riêng
    │
    ├── [Public Subnet] → NAT Gateway       Subnet công khai
    │
    └── [Private Subnet]                    Subnet riêng
            │
            ├── SageMaker Training Instances
            ├── SageMaker Inference Endpoints
            └── SageMaker Studio Domain
```

### Security Groups — Nhóm Bảo Mật

**Security Groups** hoạt động như firewall cấp instance (stateful — có trạng thái):

```python
import boto3

ec2 = boto3.client("ec2")

# Tạo Security Group cho SageMaker Training
response = ec2.create_security_group(
    GroupName="sagemaker-training-sg",
    Description="Security Group cho SageMaker Training Jobs",
    VpcId="vpc-12345678",
)
sg_id = response["GroupId"]

# Quy tắc Inbound (đầu vào): Chỉ cho phép traffic nội bộ VPC
ec2.authorize_security_group_ingress(
    GroupId=sg_id,
    IpPermissions=[
        {
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "UserIdGroupPairs": [{"GroupId": sg_id}],  # Chỉ từ cùng SG
        },
    ],
)

# Quy tắc Outbound (đầu ra): Chỉ cho phép đến VPC endpoints và S3
ec2.authorize_security_group_egress(
    GroupId=sg_id,
    IpPermissions=[
        {
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "PrefixListIds": [{"PrefixListId": "pl-12345678"}],  # S3 prefix list
        },
    ],
)
```

---

## Encryption — Mã Hóa Dữ Liệu

### Encryption at Rest — Mã Hóa Khi Lưu Trữ

**AWS KMS** (Key Management Service — Dịch Vụ Quản Lý Khóa) quản lý các khóa mã hóa:

```python
import boto3

kms = boto3.client("kms")

# Tạo Customer Managed Key (CMK — Khóa Do Khách Hàng Quản Lý)
response = kms.create_key(
    Description="KMS key cho SageMaker ML workloads",
    KeyUsage="ENCRYPT_DECRYPT",
    Tags=[
        {"TagKey": "Environment", "TagValue": "production"},
        {"TagKey": "Service", "TagValue": "sagemaker"},
    ],
)
kms_key_id = response["KeyMetadata"]["KeyId"]
kms_key_arn = response["KeyMetadata"]["Arn"]

# Sử dụng KMS key trong SageMaker Training Job
from sagemaker.estimator import Estimator

estimator = Estimator(
    image_uri=training_image_uri,
    role=role,
    instance_count=1,
    instance_type="ml.p3.2xlarge",

    # Mã hóa EBS volume (ổ đĩa) của training instance
    volume_kms_key=kms_key_arn,

    # Mã hóa output (model artifacts) trên S3
    output_kms_key=kms_key_arn,
)
```

**Mã hóa S3 bucket chứa training data:**

```python
s3 = boto3.client("s3")

# Bật Server-Side Encryption với KMS key cho S3 bucket
s3.put_bucket_encryption(
    Bucket="my-training-data-bucket",
    ServerSideEncryptionConfiguration={
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "aws:kms",
                    "KMSMasterKeyID": kms_key_arn,
                },
                "BucketKeyEnabled": True,  # Giảm chi phí KMS API calls
            }
        ]
    },
)

# Bắt buộc mã hóa cho mọi object upload (deny unencrypted uploads)
policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnencryptedObjectUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::my-training-data-bucket/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            },
        }
    ],
}
s3.put_bucket_policy(Bucket="my-training-data-bucket", Policy=json.dumps(policy))
```

### Encryption in Transit — Mã Hóa Khi Truyền

- Tất cả AWS API calls sử dụng **TLS 1.2+** (Transport Layer Security — Bảo Mật Lớp Vận Chuyển)
- **Inter-container traffic encryption** (Mã Hóa Giao Tiếp Giữa Các Container): Bắt buộc khi dùng Distributed Training

```python
estimator = Estimator(
    # ...
    encrypt_inter_container_traffic=True,  # Mã hóa giao tiếp giữa nodes
)
```

---

## VPC Configuration cho SageMaker

### Cấu Hình Training Job trong VPC

```python
import sagemaker

estimator = Estimator(
    image_uri=training_image_uri,
    role=role,
    instance_count=1,
    instance_type="ml.p3.2xlarge",

    # VPC configuration
    subnets=["subnet-private-1a", "subnet-private-1b"],  # Private subnets
    security_group_ids=["sg-sagemaker-training"],
    volume_kms_key=kms_key_arn,
    output_kms_key=kms_key_arn,
)
```

**Lưu ý:** Khi training trong VPC private subnet (không có internet access):
- Container **không thể** pull từ Docker Hub hoặc PyPI trực tiếp
- Phải dùng **VPC Endpoints** để truy cập S3, ECR và các AWS services
- Hoặc dùng **NAT Gateway** (cho phép outbound internet, block inbound)

### SageMaker Studio trong VPC

```python
sm = boto3.client("sagemaker")

# Tạo SageMaker Domain trong VPC (không truy cập internet trực tiếp)
sm.create_domain(
    DomainName="my-secure-studio",
    AuthMode="SSO",
    DefaultUserSettings={
        "ExecutionRole": execution_role_arn,
        "SecurityGroups": ["sg-studio"],
    },
    SubnetIds=["subnet-private-1a", "subnet-private-1b"],
    VpcId="vpc-12345678",
    AppNetworkAccessType="VpcOnly",  # QUAN TRỌNG: Chỉ cho phép traffic qua VPC
)
```

---

## Private Endpoints — Điểm Cuối Riêng

### VPC Endpoints (Interface Endpoints) — Điểm Cuối VPC

**AWS PrivateLink** cho phép các resources trong VPC giao tiếp với AWS services mà không cần đi qua internet public:

```python
ec2 = boto3.client("ec2")

# Tạo VPC Endpoint cho S3 (Gateway type — không tính phí)
ec2.create_vpc_endpoint(
    VpcId="vpc-12345678",
    ServiceName="com.amazonaws.us-east-1.s3",
    VpcEndpointType="Gateway",
    RouteTableIds=["rtb-private-1a", "rtb-private-1b"],
)

# Tạo VPC Endpoint cho SageMaker API (Interface type — tính phí ~$7.20/tháng)
ec2.create_vpc_endpoint(
    VpcId="vpc-12345678",
    ServiceName="com.amazonaws.us-east-1.sagemaker.api",
    VpcEndpointType="Interface",
    SubnetIds=["subnet-private-1a"],
    SecurityGroupIds=["sg-vpc-endpoint"],
    PrivateDnsEnabled=True,  # Tự động resolve DNS đến private IP
)

# Tạo VPC Endpoint cho ECR (cần cho pull Docker images)
ec2.create_vpc_endpoint(
    VpcId="vpc-12345678",
    ServiceName="com.amazonaws.us-east-1.ecr.api",
    VpcEndpointType="Interface",
    SubnetIds=["subnet-private-1a"],
    SecurityGroupIds=["sg-vpc-endpoint"],
    PrivateDnsEnabled=True,
)
```

**Danh sách VPC Endpoints cần thiết cho SageMaker trong VPC:**

| Service | Endpoint Type | Bắt Buộc |
|---------|--------------|----------|
| S3 | Gateway | ✅ Bắt buộc |
| SageMaker API | Interface | ✅ Bắt buộc |
| SageMaker Runtime | Interface | ✅ Bắt buộc |
| ECR API | Interface | ✅ (nếu dùng custom container) |
| ECR DKR | Interface | ✅ (nếu dùng custom container) |
| CloudWatch Logs | Interface | Khuyến nghị |
| KMS | Interface | Khuyến nghị (nếu dùng CMK) |
| STS | Interface | Khuyến nghị |

---

## Data Privacy & PII Detection

### Amazon Comprehend PII Detection

**PII** (Personally Identifiable Information — Thông Tin Cá Nhân Nhận Dạng Được): tên, địa chỉ, số CMND, số điện thoại, email, v.v.

```python
import boto3

comprehend = boto3.client("comprehend")

# Detect PII trong text (để kiểm tra training data)
text = "Khách hàng Nguyễn Văn A, số điện thoại 0912345678, email: nguyen.a@example.com"

response = comprehend.detect_pii_entities(
    Text=text,
    LanguageCode="vi",  # Tiếng Việt
)

for entity in response["Entities"]:
    print(f"Type: {entity['Type']}, Score: {entity['Score']:.2f}")
    print(f"  Text: {text[entity['BeginOffset']:entity['EndOffset']]}")

# Output:
# Type: PERSON, Score: 0.99
#   Text: Nguyễn Văn A
# Type: PHONE, Score: 0.98
#   Text: 0912345678
# Type: EMAIL, Score: 0.99
#   Text: nguyen.a@example.com
```

### Redaction — Ẩn/Xóa PII

```python
# Batch PII Redaction cho training data lớn
response = comprehend.start_pii_entities_detection_job(
    InputDataConfig={
        "S3Uri": "s3://my-bucket/raw-training-data/",
        "InputFormat": "ONE_DOC_PER_LINE",
    },
    OutputDataConfig={
        "S3Uri": "s3://my-bucket/pii-redacted-data/",
    },
    Mode="ONLY_REDACTION",       # Chế độ chỉ xóa PII
    RedactionConfig={
        "PiiEntityTypes": ["ALL"],   # Xóa tất cả loại PII
        "MaskMode": "REPLACE_WITH_PII_ENTITY_TYPE",  # Thay bằng [PERSON], [EMAIL], v.v.
    },
    DataAccessRoleArn=role_arn,
    LanguageCode="vi",
    JobName="pii-redaction-training-data",
)
```

### Amazon Macie — Phát Hiện PII trong S3

**Amazon Macie** (Dịch Vụ Phát Hiện Dữ Liệu Nhạy Cảm) tự động scan S3 buckets để tìm PII và dữ liệu nhạy cảm:

```python
macie = boto3.client("macie2")

# Tạo classification job để scan S3 bucket chứa training data
response = macie.create_classification_job(
    jobType="ONE_TIME",
    name="scan-training-data-for-pii",
    s3JobDefinition={
        "bucketDefinitions": [
            {
                "accountId": "123456789012",
                "buckets": ["my-training-data-bucket"],
            }
        ]
    },
)
```

---

## Audit Logging — Nhật Ký Kiểm Toán

### AWS CloudTrail cho SageMaker

**CloudTrail** (Đường Mòn Đám Mây) ghi lại tất cả API calls:

```python
cloudtrail = boto3.client("cloudtrail")

# Tạo trail để log SageMaker API calls
cloudtrail.create_trail(
    Name="sagemaker-audit-trail",
    S3BucketName="my-cloudtrail-bucket",
    IncludeGlobalServiceEvents=True,
    IsMultiRegionTrail=True,          # Log tất cả regions
    EnableLogFileValidation=True,     # Đảm bảo log không bị sửa đổi
    KMSKeyId=kms_key_arn,             # Mã hóa log files
)

# Bật logging
cloudtrail.start_logging(Name="sagemaker-audit-trail")
```

**CloudTrail log SageMaker events như:**
- `CreateTrainingJob`, `DeleteEndpoint`, `CreateModel`
- `InvokeEndpoint` (nếu bật data events)
- `DescribeTrainingJob` (ai đang xem thông tin job)

### Amazon CloudWatch Logs

**CloudWatch Logs** lưu trữ application logs từ training containers và inference containers:

```python
import boto3

logs = boto3.client("logs")

# Xem log của training job
log_group = "/aws/sagemaker/TrainingJobs"
log_stream = "my-training-job/algo-1-1234567890"

response = logs.get_log_events(
    logGroupName=log_group,
    logStreamName=log_stream,
    limit=100,
    startFromHead=True,
)

for event in response["events"]:
    print(event["message"])
```

### AWS Config — Kiểm Soát Cấu Hình Tài Nguyên

**AWS Config** (Cấu Hình AWS) liên tục đánh giá cấu hình tài nguyên theo rules:

```python
config = boto3.client("config")

# Rule: SageMaker Notebook Instance không có public internet access
config.put_config_rule(
    ConfigRule={
        "ConfigRuleName": "sagemaker-notebook-no-direct-internet-access",
        "Source": {
            "Owner": "AWS",
            "SourceIdentifier": "SAGEMAKER_NOTEBOOK_NO_DIRECT_INTERNET_ACCESS",
        },
        "Scope": {
            "ComplianceResourceTypes": ["AWS::SageMaker::NotebookInstance"],
        },
    }
)
```

**AWS Managed Config Rules cho SageMaker:**

| Rule | Kiểm Tra |
|------|---------|
| `SAGEMAKER_ENDPOINT_CONFIGURATION_KMS_KEY_CONFIGURED` | Endpoint config có KMS key không |
| `SAGEMAKER_NOTEBOOK_INSTANCE_INSIDE_VPC` | Notebook instance có trong VPC không |
| `SAGEMAKER_NOTEBOOK_NO_DIRECT_INTERNET_ACCESS` | Notebook không có direct internet access |

---

## Compliance Frameworks

### HIPAA — Quy Định Bảo Vệ Dữ Liệu Y Tế (Healthcare)

**HIPAA** (Health Insurance Portability and Accountability Act — Đạo Luật Về Tính Di Động Và Trách Nhiệm Bảo Hiểm Y Tế) áp dụng cho Protected Health Information (PHI — Thông Tin Y Tế Được Bảo Vệ).

AWS SageMaker là HIPAA-eligible service. Để đạt HIPAA compliance:

```
HIPAA Requirements cho SageMaker:
├── Ký Business Associate Agreement (BAA) với AWS
├── Encrypt PHI at rest (KMS CMK bắt buộc)
├── Encrypt PHI in transit (TLS, encrypt inter-container traffic)
├── Audit logs đầy đủ (CloudTrail + CloudWatch Logs)
├── Access controls (IAM + MFA bắt buộc)
├── Training data không chứa PHI trừ khi đã bảo mật đúng cách
└── Dùng Comprehend Medical để detect/redact PHI trước khi training
```

### GDPR — Quy Định Bảo Vệ Dữ Liệu Châu Âu

**GDPR** (General Data Protection Regulation — Quy Định Chung Về Bảo Vệ Dữ Liệu) của EU/EEA:

```
GDPR Requirements cho ML:
├── Data minimization: Chỉ thu thập data thực sự cần cho model
├── Purpose limitation: Training data chỉ dùng cho mục đích đã khai báo
├── Right to erasure: Có thể xóa dữ liệu cá nhân khỏi training set
├── Data residency: Lưu data trong EU region (eu-west-1, eu-central-1)
└── DPA (Data Processing Agreement): Ký với AWS cho EU workloads
```

### SOC 2 — Kiểm Soát Bảo Mật Dịch Vụ

**SOC 2** (System and Organization Controls — Kiểm Soát Hệ Thống Và Tổ Chức) Type II: AWS đã được audit và có SOC 2 report. Bạn có thể download từ AWS Artifact.

---

## Checklist Bảo Mật

### IAM & Access Control

- [ ] **SageMaker Execution Role** chỉ có quyền tối thiểu cần thiết (least privilege)
- [ ] **Phân tách role** giữa dev, staging và production environments
- [ ] **Bật MFA** (Multi-Factor Authentication — Xác Thực Đa Nhân Tố) cho tất cả IAM users có quyền SageMaker
- [ ] **Không dùng root account** cho SageMaker operations
- [ ] **Audit IAM policies** định kỳ với AWS IAM Access Analyzer

### Network Security

- [ ] **SageMaker Training Jobs** chạy trong VPC private subnet
- [ ] **SageMaker Studio Domain** cấu hình `AppNetworkAccessType: VpcOnly`
- [ ] **VPC Endpoints** đã được tạo cho S3, SageMaker API, ECR
- [ ] **Security Groups** chỉ cho phép traffic cần thiết
- [ ] **Flow Logs** (Nhật Ký Luồng Mạng) bật cho VPC để monitor network traffic

### Encryption

- [ ] **EBS volumes** của training instances được mã hóa với KMS CMK
- [ ] **S3 buckets** chứa training data và model artifacts bật SSE-KMS
- [ ] **Endpoint configuration** có KMS key cho inference data
- [ ] **Inter-container traffic encryption** bật khi dùng Distributed Training
- [ ] **Notebook instances** bật encryption at rest

### Data Privacy

- [ ] **Scan training data** với Amazon Macie trước khi dùng
- [ ] **Redact PII** với Amazon Comprehend trước khi đưa vào training
- [ ] **Training data** không chứa sensitive data không cần thiết
- [ ] **Model artifacts** được lưu trong bucket riêng với restricted access

### Audit & Monitoring

- [ ] **CloudTrail** đang log tất cả SageMaker API calls
- [ ] **CloudWatch Logs** thu thập logs từ training và inference containers
- [ ] **AWS Config rules** được bật cho SageMaker compliance
- [ ] **GuardDuty** (Dịch Vụ Phát Hiện Mối Đe Dọa) đang monitor cho suspicious activity
- [ ] **Log retention policy** đã được thiết lập (ví dụ: giữ 1 năm cho compliance)

---

## 📌 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Làm thế nào để ngăn SageMaker Training Job truy cập internet?**
> A: Cấu hình Training Job trong VPC private subnet không có route ra internet. Tạo VPC Endpoints cho S3 (Gateway), SageMaker API, và ECR (Interface). Dùng Security Groups để restrict outbound traffic chỉ đến các AWS service endpoints. Không cần NAT Gateway nếu chỉ cần truy cập AWS services.

**Q: Làm thế nào để đảm bảo training data không chứa PII?**
> A: Dùng Amazon Macie để tự động scan S3 buckets. Dùng Amazon Comprehend `detect_pii_entities` để detect PII trong text data. Chạy `start_pii_entities_detection_job` với `Mode=ONLY_REDACTION` để tự động redact trước khi training. Với data y tế, dùng Comprehend Medical để detect PHI cụ thể.

**Q: Khi nào cần Customer Managed Key (CMK) thay vì AWS managed key?**
> A: CMK cần khi: (1) Cần audit log chi tiết của key usage; (2) Cần khả năng rotate key thủ công; (3) Compliance yêu cầu kiểm soát key (HIPAA, PCI DSS, FedRAMP); (4) Cần chia sẻ encrypted data cross-account với kiểm soát key chặt chẽ.

**Q: Mô tả kiến trúc bảo mật end-to-end cho ML workload production?**
> A: (1) Data landing zone: S3 với SSE-KMS + bucket policy deny unencrypted upload + Macie scan; (2) Training: VPC private subnet + VPC Endpoints + Security Groups + CMK cho EBS/output + Managed Spot Training với checkpoint mã hóa; (3) Inference: Private endpoint trong VPC hoặc public với WAF + API Gateway + KMS; (4) Monitoring: CloudTrail + CloudWatch Logs + Config rules + GuardDuty; (5) IAM: Separate roles per component + least privilege + MFA.

---

**Liên Kết Liên Quan:**
- [SageMaker Training Jobs](../02-sagemaker/2-sagemaker-training.md)
- [MLOps: Model Monitor](../09-mlops/3-model-monitor.md)
- [Responsible AI](./3-responsible-ai.md)

**Cập Nhật Lần Cuối:** 2026-06-03
