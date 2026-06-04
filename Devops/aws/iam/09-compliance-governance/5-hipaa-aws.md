# HIPAA trên AWS — Tuân Thủ Dữ Liệu Y Tế

> HIPAA (Health Insurance Portability and Accountability Act — Đạo Luật Về Tính Khả Chuyển Và Trách Nhiệm Bảo Hiểm Y Tế) là luật liên bang Hoa Kỳ bảo vệ PHI (Protected Health Information — Thông Tin Sức Khỏe Được Bảo Vệ). Tài liệu này hướng dẫn cách xây dựng hệ thống y tế tuân thủ HIPAA trên AWS.

---

## 🎯 Hiểu HIPAA Trong Bối Cảnh AWS

### Ai Cần Tuân Thủ HIPAA?

```
Covered Entities (Tổ Chức Được Bảo Vệ — Bắt Buộc):
├── Healthcare Providers (Nhà Cung Cấp Dịch Vụ Y Tế):
│   └── Bệnh viện, phòng khám, dược sĩ
├── Health Plans (Kế Hoạch Sức Khỏe):
│   └── Công ty bảo hiểm y tế, Medicare, Medicaid
└── Healthcare Clearinghouses (Trung Tâm Thanh Toán Y Tế):
    └── Xử lý yêu cầu thanh toán y tế

Business Associates (Đối Tác Kinh Doanh — Bắt Buộc Ký BAA):
├── Cloud providers xử lý PHI (bao gồm AWS)
├── Công ty phần mềm y tế (EHR vendors)
├── IT service providers có access PHI
└── Billing companies, data analytics providers

PHI Là Gì (Protected Health Information — Thông Tin Sức Khỏe Được Bảo Vệ):
├── 18 Identifiers (Yếu Tố Nhận Dạng) bao gồm:
│   ├── Tên, địa chỉ, ngày sinh
│   ├── Số điện thoại, email, SSN
│   ├── Hồ sơ bệnh án, số bảo hiểm
│   └── Ảnh chụp, số IP, số thiết bị
└── Kết hợp với: thông tin sức khỏe, điều trị, thanh toán y tế
```

### BAA (Business Associate Agreement — Thỏa Thuận Đối Tác Kinh Doanh)

```
Bước Đầu Tiên PHẢI Làm Trước Khi Dùng AWS Cho PHI:
1. Vào AWS Artifact → Agreements
2. Chấp nhận AWS BAA (HIPAA Business Associate Addendum)
3. AWS BAA áp dụng cho toàn bộ account sau khi ký

Lưu Ý Quan Trọng:
├── BAA chỉ cover các dịch vụ AWS HIPAA Eligible
├── Không phải mọi dịch vụ AWS đều HIPAA eligible
├── Cần kiểm tra danh sách tại: aws.amazon.com/compliance/hipaa-eligible-services-reference
└── PHI chỉ được xử lý trong HIPAA Eligible Services
```

### AWS HIPAA Eligible Services (Dịch Vụ Đủ Điều Kiện HIPAA)

```
Compute:
├── EC2, ECS, EKS, Lambda, Fargate
├── Elastic Beanstalk

Storage:
├── S3, EBS, EFS, Glacier
├── Storage Gateway

Database:
├── RDS (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB)
├── Aurora, DynamoDB, ElastiCache, Redshift
├── Neptune, DocumentDB

Networking:
├── VPC, Direct Connect, Route 53
├── CloudFront, ELB/ALB/NLB

Security:
├── IAM, KMS, CloudHSM, Secrets Manager
├── CloudTrail, Config, GuardDuty, Security Hub
├── Shield Advanced, WAF, Macie

Analytics:
├── Athena, EMR, Kinesis, Glue
├── QuickSight, SageMaker

AI/ML:
├── Comprehend Medical, Transcribe Medical, HealthLake
└── Rekognition (với điều kiện)

KHÔNG HIPAA Eligible (KHÔNG được dùng với PHI):
├── Alexa for Business
├── Amazon WorkMail (email)
└── Một số AI services (kiểm tra danh sách mới nhất)
```

---

## 🏗️ Kiến Trúc HIPAA Trên AWS

```
Người Dùng (Bác Sĩ, Y Tá)
    │ HTTPS + MFA
    ▼
┌─────────────────────────────────────────────────────────────┐
│                   AWS CloudFront + WAF                       │
│           (TLS 1.2+, không cache PHI)                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────┐
│                Private VPC (không có internet gateway)       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Application Layer (Private Subnet)                   │   │
│  │  ├── EHR Application (ECS/Fargate)                   │   │
│  │  ├── Medical Imaging Service (EC2)                   │   │
│  │  └── API Gateway (VPC Interface Endpoint)            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Data Layer (Isolated Private Subnet)                 │   │
│  │  ├── Aurora (PHI records) — mã hóa KMS               │   │
│  │  ├── S3 (Medical images, documents) — mã hóa KMS     │   │
│  │  ├── ElastiCache (Session data) — no PHI cached      │   │
│  │  └── Secrets Manager (DB credentials)                │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Audit & Monitoring (Security Subnet)                 │   │
│  │  ├── CloudTrail Lake (PHI access logs >= 7 years)    │   │
│  │  ├── Macie (PII/PHI discovery in S3)                 │   │
│  │  ├── GuardDuty (Threat detection)                    │   │
│  │  └── Config (Compliance monitoring)                  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔒 Ba Safeguards (Biện Pháp Bảo Vệ) HIPAA

### Administrative Safeguards (Biện Pháp Hành Chính)

```
AWS Services Hỗ Trợ:

1. Security Officer Designation:
   └── IAM role SecurityOfficer với quyền read-all, write-none
   └── CloudWatch dashboard cho Security Officer

2. Workforce Training (Đào Tạo Nhân Viên):
   └── Không phải AWS service, nhưng document trong Audit Manager

3. Access Management:
   └── IAM Identity Center — provisioning/deprovisioning tập trung
   └── Access reviews với IAM Access Analyzer
   └── Automatic deactivation: Lambda function + HR system integration

4. Contingency Planning (Kế Hoạch Dự Phòng):
   └── RDS Multi-AZ: automatic failover < 60 giây
   └── S3 Cross-Region Replication: RPO gần 0
   └── Aurora Global Database: RPO < 1 giây, RTO < 1 phút
   └── Backup: AWS Backup với vault lock (không thể xóa)

5. Audit Controls:
   └── CloudTrail: mọi API call
   └── CloudTrail Insights: phát hiện bất thường
   └── Audit Manager: thu thập evidence tự động
```

### Physical Safeguards (Biện Pháp Vật Lý)

```
AWS Chịu Trách Nhiệm Hoàn Toàn:
├── Physical access controls đến data centers
├── Workstation physical security (cho AWS infrastructure)
├── Media disposal (shredding/degaussing drives)
└── Bằng chứng: AWS SOC 2 Report, AWS PCI-DSS AOC từ Artifact

Bạn Chịu Trách Nhiệm:
├── Workstation của nhân viên y tế (máy tính, tablet)
├── Mobile devices truy cập PHI
└── Physical office security
```

### Technical Safeguards (Biện Pháp Kỹ Thuật)

```
Đây là phần AWS hỗ trợ nhiều nhất:

Access Control:
├── IAM với Least Privilege
├── MFA bắt buộc (HIPAA strongly recommends)
├── Automatic session timeout (ALB session duration)
└── Emergency access procedure (break-glass accounts)

Audit Controls:
├── CloudTrail: ghi mọi truy cập
├── S3 Access Logging: ai đọc PHI nào, khi nào
├── RDS audit logs: queries vào PHI database
└── Retention: tối thiểu 6 năm (HIPAA), khuyến nghị 7 năm

Integrity (Toàn Vẹn Dữ Liệu):
├── S3 Object Integrity (SHA-256 checksum)
├── CloudTrail log file validation
├── KMS key rotation: không ảnh hưởng đến dữ liệu đã mã hóa
└── RDS automated backups với point-in-time recovery

Transmission Security (Bảo Mật Truyền Tải):
├── TLS 1.2+ bắt buộc trên tất cả endpoints
├── VPC Endpoints: PHI không đi qua internet
├── Direct Connect: kết nối dedicated đến on-premises
└── PrivateLink: API calls không expose qua internet
```

---

## 🔐 Mã Hóa PHI Trên AWS

### S3 — Lưu Trữ PHI Tài Liệu

```bash
# Tạo S3 bucket cho PHI với tất cả security controls
BUCKET="hipaa-phi-medical-records"

# Tạo bucket
aws s3api create-bucket \
  --bucket $BUCKET \
  --region us-east-1

# Bật versioning — không thể xóa version cũ
aws s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# Bật default encryption với CMK
aws s3api put-bucket-encryption \
  --bucket $BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/hipaa-phi-kms"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Chặn tất cả public access
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Bật access logging
aws s3api put-bucket-logging \
  --bucket $BUCKET \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "hipaa-access-logs",
      "TargetPrefix": "s3-phi-access/"
    }
  }'

# Lifecycle policy — giữ PHI >= 6 năm (HIPAA), chuyển sang Glacier sau 1 năm
aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "PHI-Retention-HIPAA",
      "Status": "Enabled",
      "Transitions": [
        {"Days": 365, "StorageClass": "GLACIER"},
        {"Days": 2190, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }]
  }'

# Bucket policy — chỉ allow HTTPS, chỉ allow specific IAM roles
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyInsecureConnections",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": ["arn:aws:s3:::hipaa-phi-medical-records/*"],
        "Condition": {"Bool": {"aws:SecureTransport": "false"}}
      },
      {
        "Sid": "AllowOnlyAuthorizedRoles",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::123456789012:role/EHRApplicationRole",
            "arn:aws:iam::123456789012:role/MedicalStaffReadRole"
          ]
        },
        "Action": ["s3:GetObject", "s3:PutObject"],
        "Resource": "arn:aws:s3:::hipaa-phi-medical-records/*"
      }
    ]
  }'
```

### RDS Aurora — Database PHI

```bash
# Tạo Aurora cluster với HIPAA controls
aws rds create-db-cluster \
  --db-cluster-identifier "hipaa-ehr-cluster" \
  --engine aurora-postgresql \
  --engine-version "15.4" \
  --master-username "admin" \
  --manage-master-user-password \
  --master-user-secret-kms-key-id "arn:aws:kms:us-east-1:123456789012:key/hipaa-rds-kms" \
  --storage-encrypted \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/hipaa-rds-kms" \
  --vpc-security-group-ids "sg-rds-hipaa" \
  --db-subnet-group-name "hipaa-db-subnet-group" \
  --backup-retention-period 35 \
  --enable-cloudwatch-logs-exports '["postgresql"]' \
  --deletion-protection \
  --enable-iam-database-authentication \
  --no-publicly-accessible

# Bật Enhanced Monitoring và Performance Insights
aws rds modify-db-cluster \
  --db-cluster-identifier "hipaa-ehr-cluster" \
  --enable-performance-insights \
  --performance-insights-kms-key-id "arn:aws:kms:us-east-1:123456789012:key/hipaa-rds-kms" \
  --performance-insights-retention-period 731
```

### KMS — Quản Lý Khóa Mã Hóa PHI

```bash
# Tạo CMK riêng cho PHI với policy nghiêm ngặt
aws kms create-key \
  --description "HIPAA PHI Encryption Key — Medical Records" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "Enable account owner",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "Allow EHR application to use key",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789012:role/EHRApplicationRole"},
        "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "kms:ViaService": [
              "s3.us-east-1.amazonaws.com",
              "rds.us-east-1.amazonaws.com",
              "secretsmanager.us-east-1.amazonaws.com"
            ]
          }
        }
      },
      {
        "Sid": "Deny direct key access without service context",
        "Effect": "Deny",
        "Principal": "*",
        "Action": ["kms:Decrypt", "kms:Encrypt"],
        "Resource": "*",
        "Condition": {
          "StringNotLike": {
            "kms:ViaService": "*.amazonaws.com"
          }
        }
      }
    ]
  }'

# Bật automatic key rotation — bắt buộc cho HIPAA
aws kms enable-key-rotation \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/hipaa-phi-kms"
```

---

## 🔍 Macie — Phát Hiện PHI Trong S3

```bash
# Bật Macie
aws macie2 enable-macie

# Tạo classification job để scan PHI định kỳ
aws macie2 create-classification-job \
  --name "PHI-Weekly-Scan" \
  --job-type SCHEDULED \
  --schedule-frequency WEEKLY \
  --s3-job-definition '{
    "bucketDefinitions": [
      {
        "accountId": "123456789012",
        "buckets": ["hipaa-phi-medical-records", "clinical-documents"]
      }
    ]
  }' \
  --managed-data-identifier-selector ALL \
  --custom-data-identifier-ids ["custom-ssn-identifier", "custom-mrn-identifier"]

# Tạo custom identifier cho Medical Record Number (MRN)
aws macie2 create-custom-data-identifier \
  --name "Medical-Record-Number-MRN" \
  --description "Phát hiện Medical Record Numbers theo format nội bộ" \
  --regex "MRN[0-9]{8}" \
  --keywords '["medical record", "patient ID", "MRN"]' \
  --maximum-match-distance 50

# EventBridge rule khi Macie phát hiện PHI bất ngờ
aws events put-rule \
  --name "PHI-Unexpected-Discovery" \
  --event-pattern '{
    "source": ["aws.macie"],
    "detail-type": ["Macie Finding"],
    "detail": {
      "type": ["SensitiveData:S3Object/Personal"],
      "severity": {"score": [{"numeric": [">=", 70]}]}
    }
  }'
```

---

## 📊 Audit Trail Cho HIPAA Compliance

### HIPAA Audit Log Requirements

```
HIPAA yêu cầu log tất cả:
├── Ai truy cập PHI (who accessed)
├── PHI nào được truy cập (what data)
├── Khi nào (when)
├── Từ đâu (from where)
└── Làm gì với PHI (what action: read/write/delete)

Retention: Tối thiểu 6 năm (khuyến nghị 7 năm)
```

```bash
# S3 server access logging — ai đọc PHI file nào
aws s3api put-bucket-logging \
  --bucket hipaa-phi-medical-records \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "hipaa-audit-logs-7years",
      "TargetPrefix": "phi-access-logs/"
    }
  }'

# RDS audit logging (PostgreSQL)
# Trong PostgreSQL parameter group:
# log_connections = 1
# log_disconnections = 1
# log_duration = 1
# log_statement = 'all'  (hoặc 'ddl' để tối ưu)
# pgaudit.log = 'read,write,ddl'

# CloudTrail Lake với retention 7 năm
aws cloudtrail create-event-data-store \
  --name "hipaa-phi-audit-store" \
  --retention-period 2555 \
  --termination-protection-enabled \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/hipaa-phi-kms" \
  --advanced-event-selectors '[
    {
      "Name": "Log all data events for PHI buckets",
      "FieldSelectors": [
        {"Field": "eventCategory", "Equals": ["Data"]},
        {"Field": "resources.ARN", "StartsWith": ["arn:aws:s3:::hipaa-phi"]}
      ]
    }
  ]'
```

### Athena Queries Phân Tích PHI Access

```sql
-- Ai đã truy cập PHI records trong 24 giờ qua?
SELECT
  userIdentity.principalId AS who,
  userIdentity.arn AS role_arn,
  sourceIPAddress AS from_ip,
  eventName AS action,
  requestParameters AS what_phi,
  eventTime AS when,
  awsRegion AS region
FROM cloudtrail_logs
WHERE
  eventTime > DATE_ADD('hour', -24, NOW())
  AND (
    (eventSource = 's3.amazonaws.com'
     AND requestParameters LIKE '%hipaa-phi%')
    OR
    (eventSource = 'rds.amazonaws.com'
     AND requestParameters LIKE '%ehr%')
  )
ORDER BY eventTime DESC;

-- Phát hiện bất thường: truy cập PHI ngoài giờ hành chính
SELECT
  userIdentity.arn,
  eventName,
  eventTime,
  sourceIPAddress
FROM cloudtrail_logs
WHERE
  eventSource = 's3.amazonaws.com'
  AND requestParameters LIKE '%hipaa-phi%'
  AND (
    EXTRACT(HOUR FROM CAST(eventTime AS TIMESTAMP)) < 7
    OR EXTRACT(HOUR FROM CAST(eventTime AS TIMESTAMP)) > 20
    OR EXTRACT(DOW FROM CAST(eventTime AS TIMESTAMP)) IN (0, 6)
  )
ORDER BY eventTime DESC;
```

---

## 🚨 Incident Response Cho HIPAA Breach

### HIPAA Breach Notification Rule (Quy Tắc Thông Báo Vi Phạm HIPAA)

```
Khi Nào Phải Thông Báo:
├── Vi phạm dữ liệu ảnh hưởng >= 500 người → Thông báo HHS và media trong 60 ngày
├── Vi phạm < 500 người → Ghi nhận, thông báo HHS cuối năm
└── Tất cả vi phạm → Thông báo người bị ảnh hưởng trong 60 ngày

4 Yếu Tố Xác Định "Breach" (Vi Phạm):
├── Nature and extent of PHI (loại và mức độ PHI)
├── Who accessed PHI (ai truy cập)
├── Was PHI actually acquired/viewed (có thực sự bị lấy/xem không)
└── Risk of harm to the individual (rủi ro tổn hại)
```

### Runbook HIPAA Breach Response

```python
import boto3
import json
from datetime import datetime

class HIPAABreachResponder:
    """Quy trình phản hồi vi phạm HIPAA tự động"""
    
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.guardduty = boto3.client('guardduty')
        self.sns = boto3.client('sns')
        self.iam = boto3.client('iam')
    
    def respond_to_breach(self, finding: dict) -> None:
        """Phản hồi khi phát hiện vi phạm PHI tiềm năng"""
        
        breach_id = f"BREACH-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}"
        
        # Bước 1: Thu thập bằng chứng ngay lập tức
        evidence = self._collect_evidence(finding)
        
        # Bước 2: Cô lập nếu confirmed breach
        if self._is_confirmed_breach(finding):
            self._isolate_compromised_resource(finding)
        
        # Bước 3: Thông báo Security Officer trong 1 giờ
        self._notify_security_team(breach_id, finding, evidence)
        
        # Bước 4: Lưu incident vào S3 (immutable)
        self._create_incident_record(breach_id, finding, evidence)
        
        # Bước 5: Bắt đầu 60-day countdown cho HHS notification
        self._schedule_regulatory_reminder(breach_id)
    
    def _collect_evidence(self, finding: dict) -> dict:
        """Thu thập bằng chứng từ CloudTrail"""
        ct = boto3.client('cloudtrail')
        
        # Lấy tất cả events liên quan đến resource bị ảnh hưởng
        events = ct.lookup_events(
            LookupAttributes=[{
                'AttributeKey': 'ResourceName',
                'AttributeValue': finding.get('resource_id', '')
            }],
            StartTime=finding.get('start_time'),
            EndTime=finding.get('end_time')
        )
        
        return {
            'cloudtrail_events': events['Events'],
            'finding_details': finding,
            'collection_time': datetime.utcnow().isoformat()
        }
    
    def _isolate_compromised_resource(self, finding: dict) -> None:
        """Cô lập resource bị xâm phạm"""
        resource_type = finding.get('resource_type')
        
        if resource_type == 'EC2':
            ec2 = boto3.client('ec2')
            # Thay Security Group bằng quarantine SG (no ingress/egress)
            ec2.modify_instance_attribute(
                InstanceId=finding['resource_id'],
                Groups=['sg-quarantine-no-traffic']
            )
        elif resource_type == 'IAMUser':
            # Disable compromised IAM user
            self.iam.update_user(
                UserName=finding['resource_id'],
                # Không thể disable user trực tiếp, cần attach deny policy
            )
            self.iam.attach_user_policy(
                UserName=finding['resource_id'],
                PolicyArn='arn:aws:iam::123456789012:policy/DenyAll'
            )
    
    def _notify_security_team(self, breach_id: str, finding: dict, evidence: dict) -> None:
        """Thông báo cho Security Officer và Privacy Officer"""
        message = {
            'breach_id': breach_id,
            'severity': 'CRITICAL',
            'title': 'Potential HIPAA PHI Breach Detected',
            'finding_type': finding.get('type'),
            'affected_resource': finding.get('resource_id'),
            'estimated_affected_patients': finding.get('patient_count', 'Unknown'),
            'immediate_actions_taken': ['evidence collected', 'resource isolated'],
            'hipaa_deadline': '60 days for HHS notification if confirmed breach',
            'next_steps': [
                '1. Legal team review within 4 hours',
                '2. Forensic investigation within 24 hours',
                '3. Determine if breach notification required',
                '4. Contact HHS if >= 500 affected'
            ]
        }
        
        self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:hipaa-breach-alerts',
            Subject=f'[CRITICAL] HIPAA Breach Alert: {breach_id}',
            Message=json.dumps(message, indent=2)
        )
```

---

## ✅ HIPAA Controls Checklist

```
Administrative:
☐ BAA đã ký với AWS (AWS Artifact → Agreements)
☐ HIPAA Security Officer được chỉ định
☐ Workforce training records (đào tạo nhân viên hàng năm)
☐ Disaster Recovery Plan (kế hoạch khôi phục thảm họa)
☐ Incident Response Plan bao gồm breach notification procedure
☐ Business Continuity Plan (kế hoạch duy trì hoạt động)

Technical — Access Control:
☐ MFA bắt buộc cho tất cả user truy cập PHI
☐ IAM Identity Center quản lý tập trung
☐ Automatic session timeout (< 30 phút idle)
☐ Emergency access (break-glass) account documented
☐ Access reviews hàng quý (IAM Access Analyzer)

Technical — Audit Controls:
☐ CloudTrail bật, multi-region, toàn organization
☐ S3 access logging cho PHI buckets
☐ RDS audit logging bật
☐ Log retention >= 6 năm (khuyến nghị 7 năm)
☐ Log immutability: MFA delete, Object Lock, Vault Lock

Technical — Integrity:
☐ S3 Object Integrity checksum
☐ CloudTrail log file validation
☐ Database backups và point-in-time recovery

Technical — Encryption:
☐ PHI at rest: KMS CMK với auto-rotation
☐ PHI in transit: TLS 1.2+ minimum
☐ S3: SSL-only bucket policy
☐ RDS: force_ssl, storage encrypted
☐ EBS: encrypted volumes cho instances xử lý PHI
☐ Secrets Manager: database credentials không hardcoded

Technical — Transmission:
☐ VPC Endpoints cho PHI services (không qua internet)
☐ PrivateLink cho API calls nội bộ
☐ Direct Connect cho kết nối on-premises (nếu có)
☐ WAF và CloudFront cho public-facing services
```

---

## 💡 Câu Hỏi Phỏng Vấn

**Q: PHI và PII khác nhau như thế nào? Khi nào cần HIPAA?**
> PII (Personally Identifiable Information — Thông Tin Nhận Dạng Cá Nhân) là thông tin nhận dạng một người (tên, email, SSN). PHI là PII kết hợp với thông tin sức khỏe (bệnh tật, điều trị, thanh toán y tế). HIPAA áp dụng khi bạn xử lý PHI với tư cách là Covered Entity hoặc Business Associate. GDPR áp dụng cho PII của người dùng EU. Một hệ thống y tế có thể cần cả hai.

**Q: Bước đầu tiên cần làm để deploy ứng dụng healthcare trên AWS là gì?**
> Ký BAA (Business Associate Agreement) với AWS trước tiên — đây là yêu cầu pháp lý bắt buộc. Vào AWS Artifact → Agreements → AWS Artifact Nondisclosure Agreement → chấp nhận HIPAA BAA. Chỉ sau khi BAA được ký, PHI mới được phép xử lý trên AWS. Ngoài ra cần xác nhận các dịch vụ sử dụng nằm trong danh sách HIPAA Eligible Services.

**Q: HIPAA có yêu cầu mã hóa bắt buộc không?**
> HIPAA không mandate (bắt buộc) thuật toán mã hóa cụ thể, nhưng mã hóa là "addressable" implementation specification — nghĩa là bạn phải triển khai nếu "reasonable and appropriate", hoặc phải documented lý do tại sao không. Trong thực tế, không mã hóa PHI gần như không thể justify, và FTC/HHS luôn expect mã hóa. Dùng AES-256 với KMS là tiêu chuẩn được chấp nhận rộng rãi.

**Q: Nếu nhân viên vô tình gửi PHI vào S3 bucket không được mã hóa, phải làm gì?**
> (1) Ngay lập tức: Enable encryption và move PHI sang bucket đúng chuẩn. (2) Xóa PHI khỏi bucket không an toàn (với S3 Object versioning: delete all versions). (3) Kiểm tra access logs: ai đã access bucket đó sau khi PHI bị upload. (4) Phân tích nguy cơ breach: bucket có public access không? Ai khác có thể đã thấy? (5) Nếu xác định là breach: thực hiện HIPAA breach notification procedure. (6) Corrective action: thêm S3 deny policy ngăn upload không mã hóa, dùng Config rule cảnh báo.

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [4-pci-dss-aws.md](4-pci-dss-aws.md) | **5-hipaa-aws.md** | [../10-advanced/README.md](../10-advanced/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
