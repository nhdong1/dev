# Security Hub — Trung Tâm Bảo Mật Tổng Hợp

> AWS Security Hub là dịch vụ tổng hợp, chuẩn hóa và ưu tiên hóa findings (phát hiện bảo mật) từ nhiều dịch vụ AWS và đối tác bên thứ ba, cho phép xem toàn cảnh tình trạng bảo mật của tổ chức.

## 📚 Mục Lục

1. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
2. [Nguồn Findings Tích Hợp](#nguồn-findings-tích-hợp)
3. [Security Standards — Tiêu Chuẩn Bảo Mật](#security-standards)
4. [Findings & ASFF Format](#findings--asff-format)
5. [Custom Actions — Hành Động Tùy Chỉnh](#custom-actions)
6. [Multi-Account Aggregation](#multi-account-aggregation)
7. [Automation Rules — Quy Tắc Tự Động Hóa](#automation-rules)
8. [Tích Hợp Với EventBridge](#tích-hợp-với-eventbridge)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Tổng Quan

```
Integrated Services (Dịch Vụ Tích Hợp)
    ├── GuardDuty        — threat findings
    ├── Inspector v2     — vulnerability findings
    ├── Macie            — data security findings
    ├── IAM Access Analyzer — unintended access findings
    ├── Firewall Manager — policy violation findings
    ├── Systems Manager  — patch compliance findings
    └── Third-party partners (Aqua, Palo Alto, Splunk...)
            │
            ▼ (chuẩn hóa sang ASFF)
    ┌─────────────────────────────────────────┐
    │             Security Hub                │
    │  ┌─────────────────────────────────┐    │
    │  │    ASFF Normalization           │    │  ← Chuẩn hóa định dạng
    │  │    Security Score Calculation   │    │  ← Tính điểm bảo mật
    │  │    Finding Deduplication        │    │  ← Loại bỏ trùng lặp
    │  │    Cross-region Aggregation     │    │  ← Tổng hợp đa region
    │  └─────────────────────────────────┘    │
    └─────────────────────────────────────────┘
            │
            ▼
    ┌────────────────────────────────┐
    │   Output                       │
    │   ├── Security Hub Console     │
    │   ├── EventBridge → Automation │
    │   ├── Custom Actions → Lambda  │
    │   └── API → SIEM (Splunk, QRadar) │
    └────────────────────────────────┘
```

### ASFF — AWS Security Finding Format

**ASFF** (AWS Security Finding Format — Định Dạng Finding Bảo Mật AWS) là schema JSON chuẩn hóa để biểu diễn mọi finding bất kể nguồn gốc.

```json
{
  "SchemaVersion": "2018-10-08",
  "Id": "arn:aws:securityhub:us-east-1:123456789012:finding/abc-123",
  "ProductArn": "arn:aws:securityhub:us-east-1::product/aws/guardduty",
  "GeneratorId": "arn:aws:guardduty:us-east-1:123456789012:detector/xyz",
  "AwsAccountId": "123456789012",
  "Types": ["TTPs/Initial Access/Exploit Public-Facing Application"],
  "CreatedAt": "2026-05-16T10:30:00Z",
  "UpdatedAt": "2026-05-16T10:35:00Z",
  "Severity": {
    "Label": "HIGH",
    "Normalized": 70
  },
  "Title": "EC2 instance communicating with known C2 server",
  "Description": "Instance i-0123456789abcdef0 is communicating with a known command and control server.",
  "Resources": [
    {
      "Type": "AwsEc2Instance",
      "Id": "arn:aws:ec2:us-east-1:123456789012:instance/i-0123456789abcdef0",
      "Region": "us-east-1",
      "Details": {
        "AwsEc2Instance": {
          "InstanceId": "i-0123456789abcdef0",
          "Type": "t3.medium",
          "ImageId": "ami-0123456789abcdef0"
        }
      }
    }
  ],
  "Compliance": {
    "Status": "FAILED"
  },
  "WorkflowState": "NEW",
  "RecordState": "ACTIVE"
}
```

---

## Nguồn Findings Tích Hợp

### AWS Native Integrations (Tích Hợp AWS Gốc)

| Dịch Vụ | Loại Finding | Bật Tự Động |
|---|---|---|
| **GuardDuty** | Threat detection findings | Khi bật Security Hub |
| **Inspector v2** | CVE vulnerability findings | Khi bật Security Hub |
| **Macie** | Sensitive data findings | Khi bật Security Hub |
| **IAM Access Analyzer** | External access, unused access | Khi bật Security Hub |
| **Firewall Manager** | Policy compliance violations | Khi bật Security Hub |
| **AWS Health** | Service degradation events | Khi bật Security Hub |
| **Systems Manager Patch Manager** | Patch compliance findings | Khi bật Security Hub |
| **Config** | Rule evaluation results | Qua Security Hub controls |

### Third-Party Integrations (Đối Tác Bên Thứ Ba)

Security Hub hỗ trợ 70+ đối tác bao gồm:
- **Splunk, IBM QRadar** — SIEM integration
- **Palo Alto Networks, Check Point** — Firewall findings
- **Aqua Security, Sysdig** — Container security
- **CrowdStrike, Carbon Black** — EDR (Endpoint Detection and Response)
- **Tenable, Qualys** — Vulnerability management

```bash
# Liệt kê tất cả integrations
aws securityhub list-enabled-products-for-import

# Bật tích hợp với một partner
aws securityhub enable-import-findings-for-product \
  --product-subscription-arn arn:aws:securityhub:us-east-1::product/splunk/splunk-enterprise
```

---

## Security Standards — Tiêu Chuẩn Bảo Mật

Security Hub đánh giá môi trường theo các tiêu chuẩn bảo mật và tính **Security Score** (Điểm Bảo Mật) từ 0–100%.

### Các Tiêu Chuẩn Hỗ Trợ

| Tiêu Chuẩn | Mô Tả | Số Controls |
|---|---|---|
| **AWS Foundational Security Best Practices (FSBP)** | Best practices AWS, cập nhật liên tục | 300+ |
| **CIS AWS Foundations Benchmark v1.4** | CIS Center for Internet Security benchmark | 58 |
| **CIS AWS Foundations Benchmark v3.0** | Phiên bản mới nhất | 67 |
| **PCI-DSS v3.2.1** | Payment Card Industry Data Security Standard | 150+ |
| **NIST SP 800-53 Rev 5** | Tiêu chuẩn bảo mật liên bang Mỹ | 200+ |
| **AWS Resource Tagging Standard** | Kiểm tra tagging compliance | 19 |

### Bật Standards

```bash
# Bật tất cả standards mặc định
aws securityhub enable-security-hub \
  --enable-default-standards

# Bật thêm PCI-DSS
aws securityhub batch-enable-standards \
  --standards-subscription-requests '[
    {
      "StandardsArn": "arn:aws:securityhub:us-east-1::standards/pci-dss/v/3.2.1"
    }
  ]'

# Xem tất cả controls của một standard
aws securityhub describe-standards-controls \
  --standards-subscription-arn <ARN>
```

### Security Score — Điểm Bảo Mật

```
Security Score = (Controls Passed / Total Controls Enabled) × 100

Ví dụ:
- Total controls: 300
- Passed: 240
- Failed: 60
- Security Score: 80%

Mục tiêu: ≥ 90% cho môi trường production
```

### Ví Dụ Controls Quan Trọng (FSBP)

```
IAM.1  — IAM policies không nên có quyền admin đầy đủ (*)
IAM.4  — Hardware MFA bắt buộc cho root account
IAM.6  — Hardware MFA bắt buộc cho root account
S3.1   — S3 bucket nên bật Block Public Access
S3.2   — S3 bucket không nên public readable
EC2.2  — VPC default security group không nên có rules
KMS.4  — KMS keys nên bật key rotation
RDS.1  — RDS snapshots không nên public
CloudTrail.1 — CloudTrail phải bật ở tất cả regions
Config.1 — AWS Config phải được bật
```

---

## Custom Actions — Hành Động Tùy Chỉnh

**Custom Actions** cho phép gửi findings được chọn đến EventBridge để kích hoạt quy trình tùy chỉnh.

### Tạo Custom Action

```bash
# Tạo custom action "Isolate EC2"
aws securityhub create-action-target \
  --name "IsolateEC2Instance" \
  --description "Isolate compromised EC2 instance" \
  --id "IsolateEC2"

# Output: ActionTargetArn
# arn:aws:securityhub:us-east-1:123456789012:action/custom/IsolateEC2
```

### EventBridge Rule Cho Custom Action

```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Custom Action"],
  "detail": {
    "actionName": ["IsolateEC2Instance"]
  }
}
```

### Lambda Handler

```python
def lambda_handler(event, context):
    findings = event['detail']['findings']
    
    for finding in findings:
        # Lấy instance ID từ finding
        resources = finding.get('Resources', [])
        for resource in resources:
            if resource['Type'] == 'AwsEc2Instance':
                instance_id = resource['Id'].split('/')[-1]
                
                # Thực hiện isolation
                isolate_instance(instance_id)
                
                # Cập nhật finding workflow
                update_finding_workflow(
                    finding_id=finding['Id'],
                    product_arn=finding['ProductArn'],
                    workflow_status='IN_PROGRESS',
                    note='EC2 isolated. Investigation in progress.'
                )

def update_finding_workflow(finding_id, product_arn, workflow_status, note):
    sh = boto3.client('securityhub')
    sh.batch_update_findings(
        FindingIdentifiers=[{
            'Id': finding_id,
            'ProductArn': product_arn
        }],
        Workflow={'Status': workflow_status},
        Note={
            'Text': note,
            'UpdatedBy': 'AutomatedResponse'
        }
    )
```

---

## Multi-Account Aggregation — Tổng Hợp Đa Tài Khoản

### Aggregation Region (Region Tổng Hợp)

```
Aggregation Region (us-east-1) — Security Account
    ├── us-east-1 findings (local)
    ├── us-west-2 findings (linked)
    ├── eu-west-1 findings (linked)
    └── ap-southeast-1 findings (linked)

Tất cả findings từ member accounts và linked regions
    → Hiển thị trong một dashboard duy nhất
```

```bash
# Bật cross-region aggregation
aws securityhub create-finding-aggregator \
  --region-linking-mode ALL_REGIONS

# Hoặc chỉ một số regions cụ thể
aws securityhub create-finding-aggregator \
  --region-linking-mode SPECIFIED_REGIONS \
  --regions us-east-1 us-west-2 eu-west-1
```

### Organization Integration

```bash
# Từ Management Account: ủy quyền Security Account làm admin
aws securityhub enable-organization-admin-account \
  --admin-account-id 111122223333

# Từ Security Account: bật auto-enable cho members mới
aws securityhub update-organization-configuration \
  --auto-enable \
  --auto-enable-standards DEFAULT
```

---

## Automation Rules — Quy Tắc Tự Động Hóa

**Automation Rules** (Quy Tắc Tự Động Hóa) cho phép tự động cập nhật workflow status, severity, hoặc suppress findings dựa trên điều kiện.

```bash
# Tạo automation rule: tự động suppress Low severity findings từ test accounts
aws securityhub create-automation-rule \
  --rule-name "SuppressTestAccountLowFindings" \
  --rule-order 1 \
  --rule-status ENABLED \
  --criteria '{
    "SeverityLabel": [{"Value": "LOW", "Comparison": "EQUALS"}],
    "AwsAccountId": [
      {"Value": "444455556666", "Comparison": "EQUALS"}
    ]
  }' \
  --actions '[
    {
      "Type": "FINDING_FIELDS_UPDATE",
      "FindingFieldsUpdate": {
        "Workflow": {"Status": "SUPPRESSED"},
        "Note": {
          "Text": "Auto-suppressed: Low severity in test account",
          "UpdatedBy": "AutomationRule"
        }
      }
    }
  ]'
```

### Workflow States (Trạng Thái Quy Trình)

| Status | Ý Nghĩa |
|---|---|
| `NEW` | Finding mới chưa được xem xét |
| `NOTIFIED` | Đã thông báo cho team liên quan |
| `IN_PROGRESS` | Đang điều tra / xử lý |
| `RESOLVED` | Đã giải quyết xong |
| `SUPPRESSED` | Đã loại trừ (false positive hoặc chấp nhận rủi ro) |

---

## Tích Hợp Với EventBridge

### Tất Cả Security Hub Events Đến EventBridge

```json
// Bắt tất cả High/Critical findings mới
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["HIGH", "CRITICAL"]
      },
      "Workflow": {
        "Status": ["NEW"]
      }
    }
  }
}
```

### Insight-Based Notifications (Thông Báo Dựa Trên Insight)

```bash
# Tạo insight: nhóm findings theo account + severity
aws securityhub create-insight \
  --name "HighSeverityByAccount" \
  --filters '{
    "SeverityLabel": [{"Value": "HIGH", "Comparison": "EQUALS"}],
    "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}]
  }' \
  --group-by-attribute "AwsAccountId"
```

---

## Câu Hỏi Phỏng Vấn

**Q: Security Hub khác GuardDuty như thế nào?**
A: GuardDuty là nguồn tạo ra findings (threat detection). Security Hub là nơi tổng hợp findings từ nhiều nguồn (GuardDuty, Inspector, Macie, partners...) và đánh giá compliance theo standards. Bạn cần cả hai: GuardDuty phát hiện, Security Hub tổng hợp và prioritize.

**Q: Làm thế nào để đạt Security Score 90%+?**
A: 
1. Bật tất cả standards phù hợp
2. Xem findings theo severity — fix Critical và High trước
3. Disable controls không applicable (nếu có justification)
4. Dùng Automation Rules để suppress known false positives
5. Track score theo thời gian, set baseline và alert khi giảm

**Q: ASFF là gì và tại sao quan trọng?**
A: ASFF (AWS Security Finding Format) là schema JSON chuẩn hóa. Quan trọng vì nó cho phép mọi dịch vụ (AWS và third-party) gửi findings theo cùng format, giúp Security Hub xử lý, so sánh, và dedup (loại bỏ trùng lặp) findings một cách nhất quán. Cũng là format để gửi custom findings từ code của bạn.

**Q: Custom Actions khác Automation Rules như thế nào?**
A: Custom Actions là thủ công — analyst chọn finding và click action → kích hoạt EventBridge event. Automation Rules là tự động — áp dụng ngay khi finding được import, dựa trên điều kiện định nghĩa trước. Dùng Custom Actions cho investigation workflows, Automation Rules cho suppression và classification tự động.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
