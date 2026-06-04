# AWS Audit Manager — Thu Thập Bằng Chứng Tuân Thủ Tự Động

> AWS Audit Manager — Trình Quản Lý Kiểm Toán — giúp tự động hóa quá trình thu thập bằng chứng (evidence collection) để chuẩn bị cho các cuộc kiểm toán tuân thủ, giảm thiểu công việc thủ công từ hàng tuần xuống còn vài giờ.

---

## 🎯 Audit Manager Là Gì?

AWS Audit Manager là dịch vụ liên tục thu thập bằng chứng từ các dịch vụ AWS của bạn và ánh xạ (map) chúng vào các control (kiểm soát) của framework tuân thủ như **PCI-DSS**, **SOC2**, **HIPAA**, **CIS Benchmarks**, **ISO 27001**.

### Vấn Đề Audit Manager Giải Quyết

```
Kiểm Toán Thủ Công (Truyền Thống):
├── Chụp màn hình console → tốn 2-3 ngày/audit
├── Xuất CSV từ nhiều dịch vụ → dễ sai sót
├── Gửi email hỏi team → mất nhiều lần qua lại
├── Tổng hợp bằng tay → không nhất quán
└── Kết quả: 2-4 tuần chuẩn bị cho mỗi audit

Với Audit Manager:
├── Tự động thu thập từ Config, CloudTrail, Security Hub
├── Ánh xạ bằng chứng vào đúng control của framework
├── Lưu trữ tập trung, có thể xem lại bất kỳ lúc nào
├── Xuất báo cáo một click
└── Kết quả: 1-2 ngày review + vài giờ xuất báo cáo
```

---

## 🏗️ Kiến Trúc Audit Manager

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Audit Manager                         │
│                                                             │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Framework  │    │  Assessment  │    │   Evidence    │  │
│  │ (PCI-DSS /  │───▶│ (Assessment  │───▶│   Folder      │  │
│  │  HIPAA /    │    │  cho môi     │    │  (Bằng Chứng  │  │
│  │  SOC2 /     │    │  trường cụ   │    │   theo Control│  │
│  │  Custom)    │    │  thể)        │    │   ID)         │  │
│  └─────────────┘    └──────────────┘    └───────────────┘  │
│                             │                               │
│              ┌──────────────┼──────────────┐                │
│              ▼              ▼              ▼                │
│       ┌────────────┐ ┌────────────┐ ┌────────────┐         │
│       │ AWS Config │ │CloudTrail  │ │Security Hub│         │
│       │ (Resource  │ │(API Logs)  │ │(Findings)  │         │
│       │  Config)   │ │            │ │            │         │
│       └────────────┘ └────────────┘ └────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## 📦 Các Khái Niệm Cốt Lõi

### Framework (Khung Tuân Thủ)

Bộ sưu tập các control được tổ chức theo tiêu chuẩn tuân thủ.

```
Loại Framework:
├── Prebuilt Frameworks (Khung Có Sẵn):
│   ├── PCI-DSS v3.2.1 / v4.0
│   ├── HIPAA
│   ├── SOC 2 — Security, Availability, Confidentiality
│   ├── ISO 27001
│   ├── CIS Benchmark v1.4 / v3.0
│   ├── AWS Foundational Security Best Practices
│   ├── FedRAMP Moderate
│   └── NIST Cybersecurity Framework
│
└── Custom Frameworks (Khung Tùy Chỉnh):
    ├── Kết hợp controls từ nhiều framework
    ├── Thêm controls nội bộ của tổ chức
    └── Tái sử dụng controls đã xây dựng
```

### Control (Kiểm Soát)

Một yêu cầu cụ thể trong framework. Mỗi control có thể được hỗ trợ bởi nhiều loại bằng chứng.

```yaml
# Ví dụ Control trong PCI-DSS
Control ID: PCI-DSS-2.2
Control Name: "System configuration standards"
Control Description: "Phát triển tiêu chuẩn cấu hình cho tất cả loại hệ thống"

Evidence Sources:
  - AWS Config Rules:
      - ec2-instance-no-public-ip
      - restricted-ssh
      - s3-bucket-public-read-prohibited
  - CloudTrail Events:
      - CreateSecurityGroup
      - AuthorizeSecurityGroupIngress
  - Security Hub Findings:
      - ec2-instance-detailed-monitoring-enabled
```

### Assessment (Đánh Giá)

Một lần chạy đánh giá tuân thủ cho một môi trường AWS cụ thể (tài khoản + regions + framework).

```
Assessment Configuration:
├── Framework: PCI-DSS v4.0
├── AWS Accounts: production-account (123456789012)
├── AWS Regions: us-east-1, us-west-2, ap-southeast-1
├── Evidence Collection: Automated (tự động)
└── Assessment Duration: 90 ngày (thường theo chu kỳ audit)
```

### Evidence (Bằng Chứng)

Dữ liệu thực tế được thu thập để chứng minh tuân thủ hoặc không tuân thủ.

```
Loại Evidence:
├── Compliance Check Evidence:
│   └── Kết quả kiểm tra Config Rule (COMPLIANT/NON_COMPLIANT)
│
├── AWS API Call Evidence:
│   └── CloudTrail events liên quan đến control
│
├── Configuration Data:
│   └── Snapshot cấu hình tài nguyên tại thời điểm đánh giá
│
└── Manual Evidence (Bằng Chứng Thủ Công):
    └── File upload (chứng chỉ, biên bản họp, quyết định...)
```

---

## ⚙️ Thiết Lập Audit Manager

### Bước 1: Kích Hoạt Dịch Vụ

```bash
# Bật Audit Manager
aws auditmanager register-account \
  --kms-key arn:aws:kms:us-east-1:123456789012:key/your-kms-key-id

# Kiểm tra trạng thái
aws auditmanager get-account-status
# Output: { "status": "ACTIVE" }

# Bật cho delegated admin (tài khoản được ủy quyền trong Organizations)
aws auditmanager register-organization-admin-account \
  --admin-account-id 555666777888
```

### Bước 2: Tạo Assessment Từ Prebuilt Framework

```bash
# Lấy danh sách framework có sẵn
aws auditmanager list-assessment-frameworks \
  --framework-type Standard \
  --query 'frameworkMetadataList[*].{Name:name,Id:id}' \
  --output table

# Tạo assessment PCI-DSS
aws auditmanager create-assessment \
  --name "Production-PCI-DSS-2024-Q4" \
  --description "Đánh giá PCI-DSS Quý 4 năm 2024 cho môi trường production" \
  --assessment-reports-destination '{
    "destinationType": "S3",
    "destination": "s3://my-audit-reports-bucket/pci-dss/"
  }' \
  --scope '{
    "awsAccounts": [{"id": "123456789012", "name": "production"}],
    "awsServices": [
      {"serviceName": "ec2"},
      {"serviceName": "rds"},
      {"serviceName": "s3"},
      {"serviceName": "iam"},
      {"serviceName": "vpc"}
    ]
  }' \
  --roles '[{
    "roleType": "PROCESS_OWNER",
    "roleArn": "arn:aws:iam::123456789012:role/AuditManagerRole"
  }]' \
  --framework-id "FRAMEWORK_ID_HERE"
```

### Bước 3: Xem Evidence Thu Thập Được

```bash
# Lấy assessment ID
ASSESSMENT_ID="your-assessment-id"

# Lấy danh sách control sets
aws auditmanager get-assessment \
  --assessment-id $ASSESSMENT_ID \
  --query 'assessment.framework.controlSets[*].{Id:id,Name:name}'

# Xem evidence cho một control cụ thể
aws auditmanager get-evidence-folders-by-assessment-control \
  --assessment-id $ASSESSMENT_ID \
  --control-set-id "CONTROL_SET_ID" \
  --control-id "CONTROL_ID"

# Xuất evidence ra file
aws auditmanager create-assessment-report \
  --name "Q4-2024-PCI-DSS-Report" \
  --description "Báo cáo PCI-DSS Q4 2024" \
  --assessment-id $ASSESSMENT_ID
```

---

## 🔧 Custom Framework (Khung Tuân Thủ Tùy Chỉnh)

### Tạo Custom Control

```bash
aws auditmanager create-control \
  --name "INTERNAL-SEC-001: MFA Required for Console Access" \
  --description "Tất cả IAM users phải bật MFA khi truy cập AWS Console" \
  --control-mapping-sources '[
    {
      "sourceName": "Config-MFA-Check",
      "sourceDescription": "Kiểm tra MFA bật cho tất cả IAM users",
      "sourceSetUpOption": "System_Controls_Mapping",
      "sourceType": "AWS_Config",
      "sourceKeyword": {
        "keywordInputType": "SELECT_FROM_LIST",
        "keywordValue": "mfa-enabled-for-iam-console-access"
      }
    },
    {
      "sourceName": "CloudTrail-Console-Login",
      "sourceDescription": "API calls liên quan đến console login",
      "sourceSetUpOption": "System_Controls_Mapping",
      "sourceType": "AWS_CLOUDTRAIL",
      "sourceKeyword": {
        "keywordInputType": "INPUT_TEXT",
        "keywordValue": "ConsoleLogin"
      }
    }
  ]' \
  --action-plan-title "Bật MFA ngay lập tức" \
  --action-plan-instructions "1. Vào IAM Console\n2. Chọn user cần bật MFA\n3. Security credentials → Assign MFA device\n4. Verify hoạt động"
```

### Tạo Custom Framework

```bash
aws auditmanager create-assessment-framework \
  --name "INTERNAL-SECURITY-FRAMEWORK-2024" \
  --description "Framework bảo mật nội bộ kết hợp CIS + PCI-DSS + yêu cầu riêng" \
  --compliance-type "Custom" \
  --control-sets '[
    {
      "name": "Identity & Access Management",
      "controls": [
        {"id": "CONTROL_ID_MFA"},
        {"id": "CONTROL_ID_LEAST_PRIVILEGE"},
        {"id": "CONTROL_ID_PASSWORD_POLICY"}
      ]
    },
    {
      "name": "Data Protection",
      "controls": [
        {"id": "CONTROL_ID_S3_ENCRYPTION"},
        {"id": "CONTROL_ID_RDS_ENCRYPTION"},
        {"id": "CONTROL_ID_KMS_ROTATION"}
      ]
    }
  ]'
```

---

## 📊 Quy Trình Thu Thập Bằng Chứng Tự Động

### Cách Audit Manager Thu Thập Evidence

```
Nguồn 1 — AWS Config:
├── Kiểm tra Config Rules → COMPLIANT / NON_COMPLIANT
├── Config history → cấu hình tài nguyên theo thời gian
└── Kết quả được lưu dưới dạng compliance_check evidence

Nguồn 2 — AWS CloudTrail:
├── Lọc events theo keyword (ví dụ: "PutBucketPolicy", "CreateKey")
├── Ghi lại ai thực hiện, khi nào, từ đâu
└── Lưu dưới dạng aws_api_call evidence

Nguồn 3 — AWS Security Hub:
├── Lấy findings theo severity
├── Ánh xạ findings vào control tương ứng
└── Lưu dưới dạng security_hub evidence

Nguồn 4 — Manual Upload:
├── File scan pentest, chứng chỉ ISO, biên bản họp
├── Policy documents, SLA agreements
└── Lưu trong evidence folder của control
```

### Vòng Đời Evidence

```
Thu Thập → Xem Xét → Phê Duyệt → Báo Cáo

Automated Collection:
  ├── Mỗi 24 giờ: Audit Manager chạy thu thập tự động
  ├── Theo sự kiện: CloudTrail events được capture real-time
  └── Liên tục: Config Rules updates → evidence ngay lập tức

Review Process:
  ├── Control Owner (Người Sở Hữu Control) xem evidence
  ├── Đánh giá: Reviewed / Insufficient Evidence
  ├── Comment: Giải thích nếu non-compliant hoặc exception
  └── Delegate: Chuyển cho người khác review nếu cần

Report Generation:
  ├── Generate assessment report → PDF/CSV
  ├── Upload vào S3 bucket audit
  └── Chia sẻ với auditor bên ngoài
```

---

## 🔐 IAM Permissions Cho Audit Manager

### Role Cho Audit Manager Service

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AuditManagerDataCollection",
      "Effect": "Allow",
      "Action": [
        "config:GetComplianceDetailsByConfigRule",
        "config:DescribeConfigRules",
        "cloudtrail:LookupEvents",
        "securityhub:GetFindings",
        "iam:ListUsers",
        "iam:ListRoles",
        "s3:ListBuckets",
        "s3:GetBucketPolicy",
        "s3:GetBucketEncryption",
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups",
        "rds:DescribeDBInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AuditManagerEvidenceStorage",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-audit-reports-bucket/*"
    },
    {
      "Sid": "KMSForEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/audit-kms-key"
    }
  ]
}
```

### Role Cho Control Owner (Người Xem Xét Bằng Chứng)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "auditmanager:GetAssessment",
        "auditmanager:ListAssessments",
        "auditmanager:GetEvidence",
        "auditmanager:ListEvidenceForEvidenceFolder",
        "auditmanager:BatchImportEvidenceToAssessmentControl",
        "auditmanager:UpdateAssessmentControl",
        "auditmanager:UpdateAssessmentControlSetStatus"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 💰 Chi Phí Audit Manager

```
Mô Hình Tính Phí:
├── Data assessments: $1.25 per resource-assessment per month
│   └── Ví dụ: 100 EC2 instances × 3 controls = 300 resource-assessments
│   └── Chi phí: 300 × $1.25 = $375/tháng
│
├── Assessments: Không tính phí thêm (chỉ tính resource-assessments)
│
└── Evidence storage trong S3: Tính phí S3 chuẩn

Tối Ưu Chi Phí:
├── Chỉ bao gồm services thực sự trong scope audit
├── Giới hạn AWS regions trong assessment scope
├── Dùng lifecycle policy để xóa evidence cũ (>7 năm)
└── Tắt assessment sau khi audit xong, bật lại kỳ sau
```

---

## 📌 Best Practices

### 1. Chuẩn Bị Trước Audit

```
6 Tuần Trước Audit:
├── Bật assessment với framework phù hợp
├── Assign control owners cho từng nhóm controls
├── Xem evidence đã thu thập, xác định gap
└── Upload manual evidence (chứng chỉ, policy docs)

2 Tuần Trước Audit:
├── Review tất cả controls, comment giải thích exceptions
├── Chạy lại Config Rules để cập nhật trạng thái
├── Tổng hợp non-compliant items và kế hoạch khắc phục
└── Generate assessment report và review

1 Tuần Trước Audit:
├── Final review với leadership
├── Prepare QA sessions
└── Package evidence folders theo yêu cầu auditor
```

### 2. Tích Hợp Với CI/CD Pipeline

```python
import boto3

def check_compliance_before_deploy(environment: str, service: str) -> bool:
    """Kiểm tra compliance của service trước khi deploy lên production"""
    audit_manager = boto3.client('auditmanager')
    config = boto3.client('config')
    
    # Lấy danh sách Config Rules liên quan đến service
    rules = config.describe_config_rules(
        Filters={'TagKey': 'Service', 'TagValue': service}
    )
    
    # Kiểm tra tất cả rules COMPLIANT
    for rule in rules['ConfigRules']:
        compliance = config.get_compliance_details_by_config_rule(
            ConfigRuleName=rule['ConfigRuleName']
        )
        
        for result in compliance['EvaluationResults']:
            if result['ComplianceType'] == 'NON_COMPLIANT':
                print(f"[BLOCK] {rule['ConfigRuleName']} NON_COMPLIANT")
                return False
    
    return True

# Sử dụng trong CI/CD
if not check_compliance_before_deploy('production', 'payment-service'):
    raise SystemExit("Deploy bị chặn: vi phạm compliance policy")
```

### 3. Tổ Chức Evidence Hiệu Quả

```
Evidence Organization:
├── Đặt tên assessment theo convention: {Env}-{Framework}-{Year}-{Quarter}
│   └── Ví dụ: Prod-PCI-DSS-2024-Q4
│
├── Assign control owners theo trách nhiệm:
│   ├── IAM controls → Identity Team
│   ├── Network controls → Infrastructure Team
│   ├── Encryption controls → Security Team
│   └── Logging controls → DevOps Team
│
└── Manual evidence checklist trước audit:
    ├── Penetration test report (báo cáo kiểm thử xâm nhập)
    ├── Risk assessment document (tài liệu đánh giá rủi ro)
    ├── Security training records (hồ sơ đào tạo bảo mật)
    ├── Incident response plan (kế hoạch phản hồi sự cố)
    └── Vendor security assessments (đánh giá bảo mật nhà cung cấp)
```

---

## 🔍 Xử Lý Sự Cố Phổ Biến

### Evidence Không Được Thu Thập

```
Triệu Chứng: Control shows "No Evidence" sau 24 giờ

Nguyên Nhân Phổ Biến:
1. AWS Config chưa bật ở region/account trong scope
2. CloudTrail chưa bật hoặc log vào bucket khác
3. IAM role của Audit Manager thiếu permissions
4. Config Rule chưa chạy (cần trigger evaluation)

Khắc Phục:
# Kiểm tra Config status
aws configservice describe-configuration-recorders

# Trigger Config evaluation thủ công
aws configservice start-config-rules-evaluation \
  --config-rule-names "s3-bucket-public-read-prohibited"

# Kiểm tra Audit Manager role permissions
aws auditmanager validate-assessment-report-integrity \
  --s3-relative-path "path/to/report"
```

### Non-Compliant Findings Hợp Lệ (Accepted Risk)

```
Tình Huống: Một số non-compliant items là rủi ro được chấp nhận (accepted risk)

Xử Lý Trong Audit Manager:
1. Mở evidence của control vi phạm
2. Click "Add comment" → giải thích lý do exception
3. Cập nhật status → "Reviewed" với ghi chú "Accepted Risk"
4. Upload tài liệu phê duyệt exception (risk acceptance form)

Ví Dụ Comment:
"EC2 instance i-1234abcd có public IP vì đây là bastion host.
Rủi ro được chấp nhận theo Risk Acceptance #RA-2024-042 
đã được CISO phê duyệt ngày 2024-10-01. 
IP được giới hạn theo Security Group chỉ cho phép 
10.0.0.0/8 (mạng VPN công ty)."
```

---

## 💡 Câu Hỏi Phỏng Vấn

**Q: Audit Manager có thể thay thế hoàn toàn công việc của auditor bên ngoài không?**
> Không. Audit Manager tự động hóa việc thu thập bằng chứng (evidence collection) và tổ chức tài liệu, nhưng không thay thế được phán đoán của auditor. QSA (Qualified Security Assessor) vẫn cần review context, phỏng vấn nhân viên, và đưa ra ý kiến chuyên môn. Audit Manager giúp auditor làm việc nhanh hơn, không thay thế họ.

**Q: Làm thế nào Audit Manager xử lý môi trường multi-account?**
> Bật Audit Manager trên management account hoặc delegated administrator account. Sử dụng Organizations integration để assessment tự động include tất cả member accounts trong scope. Evidence được thu thập từ tất cả accounts và tập trung vào một assessment duy nhất.

**Q: Khác biệt giữa Control Owner và Assessment Owner?**
> Assessment Owner chịu trách nhiệm toàn bộ assessment (thường là CISO hoặc Compliance Manager). Control Owner chịu trách nhiệm review và đánh giá evidence cho một nhóm controls cụ thể (thường là team lead kỹ thuật của từng domain).

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [README.md](README.md) | **1-audit-manager.md** | [2-conformance-packs.md](2-conformance-packs.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
