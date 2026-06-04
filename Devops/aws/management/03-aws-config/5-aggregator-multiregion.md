# Aggregator & Multi-Account Compliance — Tổng Hợp Tuân Thủ Đa Account/Region

> **AWS Config Aggregator** (Bộ Tổng Hợp Config) thu thập dữ liệu cấu hình và kết quả compliance từ nhiều tài khoản AWS và nhiều region, tổng hợp vào một **Aggregator Account** (Tài Khoản Tổng Hợp) duy nhất — tạo ra bức tranh tuân thủ toàn diện cho toàn bộ tổ chức.

---

## 📚 Mục Lục

1. [Tại Sao Cần Aggregator?](#tại-sao-cần-aggregator)
2. [Kiến Trúc Aggregator](#kiến-trúc-aggregator)
3. [Aggregator vs Organization Trail](#aggregator-vs-organization-trail)
4. [Cấu Hình Aggregator](#cấu-hình-aggregator)
5. [Authorization — Phân Quyền Chia Sẻ Dữ Liệu](#authorization--phân-quyền-chia-sẻ-dữ-liệu)
6. [Query Compliance Tổng Hợp](#query-compliance-tổng-hợp)
7. [Tích Hợp Với Security Hub](#tích-hợp-với-security-hub)
8. [Multi-Account Config Rule Deployment](#multi-account-config-rule-deployment)
9. [Compliance Dashboard](#compliance-dashboard)
10. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Aggregator?

### Vấn Đề Không Có Aggregator

```
Tổ chức có 50 AWS accounts × 5 regions = 250 Config dashboards
  → Không thể xem compliance tổng thể
  → Báo cáo audit mất hàng ngày để thu thập
  → Không biết account nào đang vi phạm
  → CISO/Auditor không có visibility tập trung
```

### Với Aggregator

```
50 accounts × 5 regions → 1 Aggregator dashboard
  → Compliance score tổng hợp toàn tổ chức
  → Phát hiện NON_COMPLIANT resource ở bất kỳ đâu
  → Báo cáo audit tự động
  → Advanced query across tất cả accounts
```

### Aggregator Cung Cấp Gì?

| Tính Năng | Mô Tả |
|----------|-------|
| **Aggregate Compliance Data** | Tổng hợp kết quả Config Rules từ mọi account/region |
| **Aggregate Configuration Data** | Xem configuration items của tất cả tài nguyên |
| **Cross-account Query** | SQL-like query để tìm kiếm trên toàn tổ chức |
| **Compliance Summary** | Tổng số COMPLIANT/NON_COMPLIANT theo rule, account, region |
| **Resource Inventory** | Kiểm kê tất cả tài nguyên trong tổ chức |

---

## Kiến Trúc Aggregator

### Mô Hình Hub-and-Spoke

```
┌──────────────────────────────────────────────────────────────┐
│                     AWS Organization                         │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Aggregator Account (Hub)                   │    │
│  │                                                      │    │
│  │  ┌────────────────────────────────────────────────┐ │    │
│  │  │         AWS Config Aggregator                  │ │    │
│  │  │                                                │ │    │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │ │    │
│  │  │  │Account A │  │Account B │  │Account C │    │ │    │
│  │  │  │us-east-1 │  │ap-se-1   │  │eu-west-1 │    │ │    │
│  │  │  │ap-se-1   │  │us-east-1 │  │us-east-1 │    │ │    │
│  │  │  └──────────┘  └──────────┘  └──────────┘    │ │    │
│  │  └────────────────────────────────────────────────┘ │    │
│  │                                                      │    │
│  │  Dashboard: 3 accounts, 6 regions, unified view      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Account A  │  │  Account B  │  │  Account C  │         │
│  │  (Source)   │  │  (Source)   │  │  (Source)   │         │
│  │  Config ON  │  │  Config ON  │  │  Config ON  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

### Hai Phương Thức Source

#### Phương Thức 1: Tích Hợp AWS Organizations (Khuyến Nghị)

```
Aggregator → Organizations API → Tự động discover tất cả member accounts
  → Không cần authorization từng account
  → Tự động bao gồm account mới
  → Có thể exclude account cụ thể
```

#### Phương Thức 2: Chỉ Định Account Cụ Thể

```
Aggregator → Danh sách account IDs + regions cụ thể
  → Mỗi source account phải cấp authorization
  → Quản lý thủ công khi có account mới
  → Phù hợp khi không dùng Organizations
```

---

## Aggregator vs Organization Trail

| Tiêu Chí | Config Aggregator | Organization Trail (CloudTrail) |
|---------|-----------------|--------------------------------|
| **Dịch vụ** | AWS Config | AWS CloudTrail |
| **Thu thập** | Configuration state + Compliance | API call events |
| **Mục đích** | Compliance visibility tổng hợp | Centralized audit logging |
| **Tạo từ** | Aggregator account | Management account |
| **Source data** | Config trong từng account | CloudTrail trong từng account |
| **Query** | AWS Config Advanced Query | CloudTrail Lake / Athena |
| **Phát hiện** | Misconfiguration | Unauthorized API activity |

> Cả hai đều cần thiết: Aggregator cho compliance overview, Organization Trail cho audit trail.

---

## Cấu Hình Aggregator

### Tạo Aggregator Tích Hợp Organizations

```bash
# Bật trusted access cho Config trong Organizations (từ Management Account)
aws organizations enable-aws-service-access \
  --service-principal config.amazonaws.com

# Ủy quyền Aggregator Account làm delegated administrator (từ Management Account)
aws organizations register-delegated-administrator \
  --account-id 111122223333 \
  --service-principal config.amazonaws.com

# Tạo aggregator (từ Aggregator Account)
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name "organization-wide-aggregator" \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::111122223333:role/ConfigAggregatorRole",
    "AllAwsRegions": true
  }'
```

### Tạo Aggregator Với Account Cụ Thể

```bash
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name "multi-account-aggregator" \
  --account-aggregation-sources '[
    {
      "AccountIds": ["111111111111", "222222222222", "333333333333"],
      "AllAwsRegions": true
    }
  ]'
```

### Tạo Aggregator Qua CloudFormation

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS Config Aggregator cho Organization

Resources:
  # IAM Role cho Aggregator
  AggregatorRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ConfigAggregatorRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: config.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSConfigRoleForOrganizations

  # Config Aggregator
  OrganizationAggregator:
    Type: AWS::Config::ConfigurationAggregator
    Properties:
      ConfigurationAggregatorName: org-aggregator
      OrganizationAggregationSource:
        RoleArn: !GetAtt AggregatorRole.Arn
        AllAwsRegions: true
```

---

## Authorization — Phân Quyền Chia Sẻ Dữ Liệu

### Khi Dùng Organizations (Không Cần Authorization Thủ Công)

Với tích hợp Organizations, Aggregator Account tự động nhận dữ liệu từ tất cả member accounts mà không cần cấu hình gì thêm trên từng account.

### Khi Dùng Individual Accounts (Cần Authorization)

Mỗi **source account** phải cấp authorization cho Aggregator Account:

```bash
# Chạy trên source account (KHÔNG phải aggregator account)
aws configservice put-aggregation-authorization \
  --authorized-account-id 111122223333 \   # ID của Aggregator Account
  --authorized-aws-region ap-southeast-1   # Region của Aggregator
```

### Xem Danh Sách Authorized Aggregators

```bash
# Xem từ source account
aws configservice describe-aggregation-authorizations

# Xem từ aggregator account
aws configservice describe-pending-aggregation-requests
```

---

## Query Compliance Tổng Hợp

### AWS Config Advanced Query

**Advanced Query** cho phép dùng SQL-like syntax để truy vấn dữ liệu cấu hình trên toàn bộ accounts/regions trong aggregator.

### Ví Dụ Queries Thực Tế

#### Tìm Tất Cả S3 Bucket Không Có Mã Hóa

```sql
SELECT
  accountId,
  awsRegion,
  resourceId,
  configuration.serverSideEncryptionConfiguration
FROM
  aws_s3_bucket
WHERE
  configuration.serverSideEncryptionConfiguration IS NULL
```

#### Tìm EC2 Instance Đang Chạy Nhưng Không Có Tag Environment

```sql
SELECT
  accountId,
  awsRegion,
  resourceId,
  resourceName,
  configuration.instanceType,
  tags
FROM
  aws_ec2_instance
WHERE
  configuration.state.name = 'running'
  AND tags['Environment'] IS NULL
ORDER BY
  accountId, awsRegion
```

#### Đếm NON_COMPLIANT Resources Theo Account

```sql
SELECT
  accountId,
  COUNT(*) AS nonCompliantCount
FROM
  aws_config_rule_compliance
WHERE
  complianceType = 'NON_COMPLIANT'
GROUP BY
  accountId
ORDER BY
  nonCompliantCount DESC
```

#### Tìm Security Groups Có Rule Mở SSH

```sql
SELECT
  accountId,
  awsRegion,
  resourceId,
  resourceName,
  configuration.ipPermissions
FROM
  aws_ec2_security_group
WHERE
  configuration.ipPermissions[*].ipRanges[*].cidrIp = '0.0.0.0/0'
  AND configuration.ipPermissions[*].fromPort = 22
```

#### Inventory: Tổng Hợp Tất Cả RDS Instance Theo Account

```sql
SELECT
  accountId,
  awsRegion,
  resourceId,
  configuration.dbInstanceClass,
  configuration.engine,
  configuration.multiAZ,
  configuration.storageEncrypted
FROM
  aws_rds_dbinstance
ORDER BY
  accountId, awsRegion
```

### Chạy Query Qua CLI

```bash
# Chạy query trên aggregator
aws configservice select-aggregate-resource-config \
  --configuration-aggregator-name "org-aggregator" \
  --expression "
    SELECT accountId, awsRegion, resourceId, configuration.instanceType
    FROM aws_ec2_instance
    WHERE configuration.state.name = 'running'
    LIMIT 100
  "
```

### Chạy Query Qua Boto3 (Python)

```python
import boto3

config_client = boto3.client('config')

def query_noncompliant_s3_buckets(aggregator_name: str) -> list:
    """Lấy danh sách S3 buckets NON_COMPLIANT trên toàn organization"""
    results = []
    paginator = config_client.get_paginator('select_aggregate_resource_config')

    query = """
    SELECT
      accountId,
      awsRegion,
      resourceId,
      configuration.serverSideEncryptionConfiguration
    FROM aws_s3_bucket
    WHERE configuration.serverSideEncryptionConfiguration IS NULL
    """

    for page in paginator.paginate(
        ConfigurationAggregatorName=aggregator_name,
        Expression=query
    ):
        for result in page.get('Results', []):
            import json
            results.append(json.loads(result))

    return results


# Sử dụng
buckets = query_noncompliant_s3_buckets('org-aggregator')
print(f"Tìm thấy {len(buckets)} S3 buckets không có mã hóa:")
for bucket in buckets:
    print(f"  Account: {bucket['accountId']} | Region: {bucket['awsRegion']} | Bucket: {bucket['resourceId']}")
```

---

## Tích Hợp Với Security Hub

### Config Aggregator + Security Hub

```
AWS Config (mỗi account) → Findings
         ↓
Security Hub (mỗi account) → Thu thập local findings
         ↓
Security Hub (aggregated) → Tập trung tất cả findings
         ↓
CISO Dashboard
```

### Luồng Dữ Liệu

```
1. Config Rule đánh giá → NON_COMPLIANT
2. Config gửi finding sang Security Hub (tự động nếu đã enable)
3. Security Hub map sang security standards (CIS, FSBP...)
4. Security Hub Aggregator thu thập từ tất cả accounts
5. CISO xem consolidated security posture
```

### Bật Security Hub Để Nhận Config Findings

```bash
# Bật Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Bật AWS Config integration
aws securityhub enable-import-findings-for-product \
  --product-arn "arn:aws:securityhub:::product/aws/config"
```

---

## Multi-Account Config Rule Deployment

### Vấn Đề: Deploy Rules Đến 50 Accounts

Không thể login vào từng account để tạo rules — cần tự động hóa.

### Giải Pháp 1: Organization Conformance Packs

```bash
# Deploy conformance pack cho toàn bộ organization một lần
aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name "org-security-baseline" \
  --template-s3-uri "s3://config-templates/security-baseline.yaml" \
  --delivery-s3-bucket "central-config-results"
```

### Giải Pháp 2: CloudFormation StackSets

```yaml
# stackset-config-rules.yaml — Deploy qua StackSets
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  S3PublicReadRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED
```

```bash
# Tạo StackSet và deploy đến tất cả accounts trong OU
aws cloudformation create-stack-set \
  --stack-set-name config-security-rules \
  --template-body file://stackset-config-rules.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name config-security-rules \
  --deployment-targets OrganizationalUnitIds=["ou-abc-12345678"] \
  --regions ap-southeast-1 us-east-1 eu-west-1
```

### So Sánh Hai Giải Pháp

| Tiêu Chí | Organization Conformance Pack | CloudFormation StackSets |
|---------|------------------------------|--------------------------|
| **Dễ dùng** | Đơn giản hơn | Phức tạp hơn |
| **Tự động thêm account** | ✅ Tự động | ✅ Với auto-deployment |
| **Kết hợp Remediation** | ✅ Trong pack | ❌ Cần template riêng |
| **Compliance Score** | ✅ Built-in | ❌ Phải tự tổng hợp |
| **Version control** | YAML trong S3 | YAML trong S3/Git |
| **Phù hợp** | Config rules + remediation | IaC đa dịch vụ |

---

## Compliance Dashboard

### Xem Compliance Tổng Hợp Qua CLI

```bash
# Xem compliance summary theo từng rule
aws configservice get-aggregate-compliance-details-by-config-rule \
  --configuration-aggregator-name "org-aggregator" \
  --config-rule-name "s3-bucket-public-read-prohibited" \
  --compliance-type NON_COMPLIANT

# Đếm tài nguyên COMPLIANT/NON_COMPLIANT theo account
aws configservice get-aggregate-compliance-details-by-config-rule \
  --configuration-aggregator-name "org-aggregator" \
  --config-rule-name "s3-bucket-public-read-prohibited" \
  --query 'AggregateEvaluationResults[].{Account:AccountId,Region:AwsRegion,Status:ComplianceType}'
```

### Tạo Compliance Report Tự Động

```python
import boto3
import json
from datetime import datetime

def generate_weekly_compliance_report(aggregator_name: str) -> dict:
    """Tạo báo cáo tuân thủ hàng tuần cho toàn organization"""
    config_client = boto3.client('config')
    report = {
        'generatedAt': datetime.utcnow().isoformat(),
        'aggregator': aggregator_name,
        'summary': {},
        'nonCompliantByAccount': {},
        'topViolatedRules': []
    }

    # Lấy tất cả rules
    rules_response = config_client.describe_aggregate_compliance_by_config_rules(
        ConfigurationAggregatorName=aggregator_name
    )

    total_compliant = 0
    total_non_compliant = 0
    rule_violation_counts = {}

    for rule in rules_response.get('AggregateComplianceByConfigRules', []):
        rule_name = rule['ConfigRuleName']
        compliance = rule['Compliance']

        compliant_count = compliance.get('ComplianceContributorCount', {}).get('CappedCount', 0)
        non_compliant_count = compliance.get('NonCompliantCount', {}).get('CappedCount', 0)

        total_compliant += compliant_count
        total_non_compliant += non_compliant_count

        if non_compliant_count > 0:
            rule_violation_counts[rule_name] = non_compliant_count

    report['summary'] = {
        'totalCompliant': total_compliant,
        'totalNonCompliant': total_non_compliant,
        'complianceScore': round(
            total_compliant / (total_compliant + total_non_compliant) * 100, 2
        ) if (total_compliant + total_non_compliant) > 0 else 0
    }

    # Top 10 rules bị vi phạm nhiều nhất
    sorted_violations = sorted(rule_violation_counts.items(), key=lambda x: x[1], reverse=True)
    report['topViolatedRules'] = [
        {'ruleName': name, 'violationCount': count}
        for name, count in sorted_violations[:10]
    ]

    return report


# Sử dụng và lưu báo cáo
report = generate_weekly_compliance_report('org-aggregator')
print(f"Compliance Score: {report['summary']['complianceScore']}%")
print(f"Top violated rule: {report['topViolatedRules'][0] if report['topViolatedRules'] else 'None'}")

# Lưu báo cáo lên S3
s3 = boto3.client('s3')
s3.put_object(
    Bucket='compliance-reports',
    Key=f"weekly/{datetime.utcnow().strftime('%Y-%m-%d')}/report.json",
    Body=json.dumps(report, indent=2)
)
```

### CloudWatch Dashboard Cho Aggregated Compliance

```json
{
  "widgets": [
    {
      "type": "text",
      "properties": {
        "markdown": "# Organization Compliance Dashboard\nCập nhật mỗi 6 giờ từ Config Aggregator"
      }
    },
    {
      "type": "metric",
      "title": "Compliance Score Theo Thời Gian",
      "properties": {
        "metrics": [
          ["ComplianceMetrics", "OrgComplianceScore", {"label": "Org Score %"}]
        ],
        "period": 21600,
        "yAxis": {"left": {"min": 0, "max": 100}}
      }
    },
    {
      "type": "metric",
      "title": "NON_COMPLIANT Resources",
      "properties": {
        "metrics": [
          ["ComplianceMetrics", "NonCompliantResources", {"label": "Total NON_COMPLIANT"}]
        ],
        "period": 21600
      }
    }
  ]
}
```

---

## Thực Hành Tốt Nhất

### 1. Dùng Organizations Integration, Không Dùng Individual Accounts

```
✅ Organizations Integration:
   - Tự động bao gồm account mới
   - Không cần quản lý authorization thủ công
   - Tích hợp sẵn với delegated admin pattern

❌ Individual Account Authorization:
   - Mỗi account mới phải thêm thủ công
   - Dễ bỏ sót account
   - Overhead quản trị cao
```

### 2. Chọn Aggregator Account Hợp Lý

```
Không dùng:
  ❌ Management Account (root account — nên giữ ít quyền nhất có thể)
  ❌ Production account của ứng dụng

Nên dùng:
  ✅ Dedicated Security/Audit account
  ✅ Hoặc Log Archive account (cùng với CloudTrail Organization Trail)
```

### 3. Kết Hợp Với Security Hub Aggregator

```
Một nguồn sự thật (Single Source of Truth):
  Config Aggregator → Compliance của resource configuration
  Security Hub Aggregator → Security findings tổng hợp
  CloudTrail Lake → Centralized audit trail query
```

### 4. Lên Lịch Export Báo Cáo Tự Động

```
EventBridge Scheduler (hàng tuần)
  → Lambda: generate_compliance_report()
  → Lưu PDF/JSON lên S3
  → Email cho CISO, Auditor, Team Lead
```

### 5. Alert Cho Compliance Score Giảm

```
Lambda (chạy hàng ngày) → Tính compliance score
  → CloudWatch PutMetricData
  → CloudWatch Alarm: Nếu score < 85% → SNS → Slack #security
```

---

## Câu Hỏi Phỏng Vấn

**Q: Thiết kế compliance monitoring cho tổ chức có 100 AWS accounts. Bạn sẽ làm gì?**

A: Kiến trúc gồm ba lớp:
1. **Source accounts** — Bật AWS Config với recorder trên tất cả accounts (dùng Organization Conformance Pack để deploy rules đồng loạt)
2. **Aggregation** — Tạo Config Aggregator trong Security account, tích hợp với Organizations để tự động collect từ 100 accounts; bật Security Hub Aggregator song song
3. **Visibility** — CloudWatch Dashboard với custom metrics từ Lambda (chạy hàng ngày tính compliance score), alert khi score < threshold, export báo cáo hàng tuần cho CISO và auditor

**Q: Config Aggregator có cho phép remediation từ xa không?**

A: Không. Config Aggregator chỉ là "read-only view" — nó tổng hợp dữ liệu nhưng không thể kích hoạt remediation trên source accounts. Để remediate, phải cấu hình Remediation Actions trong từng source account (dùng Organization Conformance Packs với remediation bao gồm trong pack). Aggregator chỉ giúp nhìn thấy vấn đề; hành động sửa chữa phải được cấu hình cục bộ.

**Q: Advanced Query trong Config Aggregator khác với Athena + S3 như thế nào?**

A: Advanced Query dùng SQL-like syntax để query **dữ liệu cấu hình hiện tại** (near real-time) trực tiếp trong Config service — không cần infrastructure thêm. Athena + S3 query **lịch sử cấu hình** (historical data) từ Config Snapshots được lưu trong S3 — phù hợp khi cần phân tích trend theo thời gian, tìm cấu hình tại thời điểm cụ thể trong quá khứ, hoặc phân tích dữ liệu khối lượng rất lớn với chi phí thấp hơn.

---

## Tóm Tắt Module 03-aws-config

```
AWS Config — Bức Tranh Toàn Diện

Configuration Recorder    → Ghi lại WHAT changed (cấu hình)
Config Rules              → Đánh giá IS IT RIGHT? (tuân thủ)
Conformance Packs         → Framework-based compliance (CIS/PCI/NIST)
Remediation Actions       → Fix IT automatically (SSM Automation)
Aggregator                → View across ALL accounts (centralized)

Kết hợp với:
  CloudTrail    → WHO changed it? (audit trail)
  CloudWatch    → Is it healthy? (performance)
  Security Hub  → Consolidated security posture (findings)
  EventBridge   → React to compliance events (automation)
```

---

**Đây là file cuối cùng của 03-aws-config/**

**Quay lại:** [README.md](README.md) — Tổng quan AWS Config
**Module tiếp theo:** [04-systems-manager/](../04-systems-manager/README.md) — AWS Systems Manager

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
