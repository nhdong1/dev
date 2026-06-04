# AWS Firewall Manager — Quản Lý Tường Lửa Tập Trung Đa Tài Khoản

> AWS Firewall Manager — Trình Quản Lý Tường Lửa — cho phép quản trị viên bảo mật (security administrator) áp dụng và duy trì chính sách WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web), Security Groups (Nhóm Bảo Mật), Shield Advanced, Network Firewall, và Route 53 Resolver DNS Firewall đồng nhất trên toàn bộ tổ chức AWS.

---

## 🎯 Tại Sao Cần Firewall Manager?

### Thách Thức Trong Môi Trường Multi-Account

```
Không Có Firewall Manager:
├── 50 accounts × 5 regions = 250 nơi cần cấu hình WAF
├── Developer vô tình mở port 22/3389 ra internet
├── Security Group rules drift theo thời gian — khó phát hiện
├── WAF rules không đồng nhất giữa các accounts
├── Phải SSH vào từng account để kiểm tra → tốn hàng giờ
└── Không có cách nào phát hiện non-compliance real-time

Với Firewall Manager:
├── Một chính sách → tự động apply cho 250+ environments
├── Non-compliant resources bị phát hiện ngay lập tức
├── Auto-remediation: tự động fix hoặc block deploy
├── Dashboard tập trung xem trạng thái toàn bộ org
└── Accounts mới tự động được áp dụng policy khi join org
```

---

## 🏗️ Kiến Trúc Firewall Manager

```
┌─────────────────────────────────────────────────────────────┐
│              AWS Organizations Management Account            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Firewall Manager Administrator Account     │    │
│  │   (thường là Security Hub account)                  │    │
│  │                                                     │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │    │
│  │  │ WAF Policy   │  │  SG Policy   │  │ Shield   │  │    │
│  │  │              │  │              │  │ Policy   │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                   AWS Organizations                          │
└───────────────────────────┼──────────────────────────────────┘
                            │ Tự Động Deploy
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Prod Acct   │  │  Dev Acct    │  │  New Account │
  │              │  │              │  │ (tự động bao │
  │  WAF WebACL  │  │  WAF WebACL  │  │  gồm khi join│
  │  SG Policy   │  │  SG Policy   │  │  org)        │
  └──────────────┘  └──────────────┘  └──────────────┘
```

---

## 🔑 Yêu Cầu Tiên Quyết

```
1. AWS Organizations phải được bật
2. AWS Config phải được bật ở tất cả member accounts và regions
3. Firewall Manager Administrator Account phải được chỉ định
4. Với WAF policies: bật AWS WAF và cấu hình AWS Config WAF rules
5. Với Shield policies: đăng ký Shield Advanced

Chỉ Định Administrator Account:
```

```bash
# Chỉ định Security account làm Firewall Manager admin
# (Chạy từ Organizations management account)
aws fms associate-admin-account \
  --admin-account "555666777888"

# Verify
aws fms get-admin-account
# Output: { "AdminAccount": "555666777888", "RoleStatus": "READY" }
```

---

## 📋 Loại Policy Trong Firewall Manager

### 1. WAF Policy (Chính Sách WAF)

Áp dụng Web ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập) cho CloudFront distributions, ALBs (Application Load Balancers), API Gateways trên toàn org.

```bash
# Tạo WAF Policy
aws fms put-policy \
  --policy '{
    "PolicyName": "Org-WAF-Common-Rules",
    "SecurityServicePolicyData": {
      "Type": "WAFV2",
      "ManagedServiceData": "{
        \"type\": \"WAFV2\",
        \"preProcessRuleGroups\": [
          {
            \"ruleGroupArn\": null,
            \"overrideAction\": {\"type\": \"NONE\"},
            \"managedRuleGroupIdentifier\": {
              \"vendorName\": \"AWS\",
              \"name\": \"AWSManagedRulesCommonRuleSet\"
            },
            \"ruleGroupType\": \"ManagedRuleGroup\",
            \"excludeRules\": []
          },
          {
            \"ruleGroupArn\": null,
            \"overrideAction\": {\"type\": \"NONE\"},
            \"managedRuleGroupIdentifier\": {
              \"vendorName\": \"AWS\",
              \"name\": \"AWSManagedRulesKnownBadInputsRuleSet\"
            },
            \"ruleGroupType\": \"ManagedRuleGroup\"
          },
          {
            \"ruleGroupArn\": null,
            \"overrideAction\": {\"type\": \"NONE\"},
            \"managedRuleGroupIdentifier\": {
              \"vendorName\": \"AWS\",
              \"name\": \"AWSManagedRulesSQLiRuleSet\"
            },
            \"ruleGroupType\": \"ManagedRuleGroup\"
          }
        ],
        \"postProcessRuleGroups\": [],
        \"defaultAction\": {\"type\": \"ALLOW\"},
        \"overrideCustomerWebACLAssociation\": false
      }"
    },
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "ResourceTags": [],
    "ExcludeResourceTags": false,
    "RemediationEnabled": true,
    "IncludeMap": {
      "ORG_UNIT": ["ou-xxxx-yyyyyyy"]
    }
  }'
```

### 2. Security Group Policy (Chính Sách Nhóm Bảo Mật)

Kiểm soát Security Group rules trên EC2 instances và ENIs (Elastic Network Interface — Giao Diện Mạng Linh Hoạt).

#### Security Group Usage Audit Policy (Kiểm Toán Sử Dụng Security Group)

```bash
aws fms put-policy \
  --policy '{
    "PolicyName": "Block-Overly-Permissive-SGs",
    "SecurityServicePolicyData": {
      "Type": "SECURITY_GROUPS_USAGE_AUDIT",
      "ManagedServiceData": "{
        \"type\": \"SECURITY_GROUPS_USAGE_AUDIT\",
        \"deleteUnusedSecurityGroups\": false,
        \"coalesceRedundantSecurityGroups\": false,
        \"optionalDelayForUnusedInMinutes\": 1440
      }"
    },
    "ResourceType": "AWS::EC2::SecurityGroup",
    "RemediationEnabled": false,
    "IncludeMap": {
      "ACCOUNT": ["123456789012", "234567890123"]
    }
  }'
```

#### Security Group Content Audit Policy (Kiểm Toán Nội Dung Security Group)

```bash
# Phát hiện và tùy chọn chặn các Security Group rules nguy hiểm
aws fms put-policy \
  --policy '{
    "PolicyName": "No-Public-SSH-RDP",
    "SecurityServicePolicyData": {
      "Type": "SECURITY_GROUPS_CONTENT_AUDIT",
      "ManagedServiceData": "{
        \"type\": \"SECURITY_GROUPS_CONTENT_AUDIT\",
        \"securityGroupAction\": {\"type\": \"ALLOW\"},
        \"securityGroups\": [
          {
            \"id\": \"sg-baseline-template-id\"
          }
        ]
      }"
    },
    "ResourceType": "AWS::EC2::Instance",
    "RemediationEnabled": true,
    "IncludeMap": {}
  }'
```

### 3. Shield Advanced Policy (Chính Sách Shield Nâng Cao)

Bật Shield Advanced DDoS protection cho tài nguyên quan trọng.

```bash
aws fms put-policy \
  --policy '{
    "PolicyName": "Shield-Production-Resources",
    "SecurityServicePolicyData": {
      "Type": "SHIELD_ADVANCED"
    },
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "ResourceTags": [
      {"Key": "Environment", "Value": "production"}
    ],
    "ExcludeResourceTags": false,
    "RemediationEnabled": true,
    "IncludeMap": {
      "ORG_UNIT": ["ou-prod-yyyyyyy"]
    }
  }'
```

### 4. Network Firewall Policy (Chính Sách Tường Lửa Mạng)

Deploy AWS Network Firewall vào các VPC theo pattern chuẩn.

```bash
aws fms put-policy \
  --policy '{
    "PolicyName": "VPC-Network-Firewall-Standard",
    "SecurityServicePolicyData": {
      "Type": "NETWORK_FIREWALL",
      "ManagedServiceData": "{
        \"type\": \"NETWORK_FIREWALL\",
        \"networkFirewallStatelessRuleGroupReferences\": [],
        \"networkFirewallStatelessDefaultActions\": [\"aws:forward_to_sfe\"],
        \"networkFirewallStatelessFragmentDefaultActions\": [\"aws:forward_to_sfe\"],
        \"networkFirewallStatefulRuleGroupReferences\": [
          {
            \"resourceARN\": \"arn:aws:network-firewall:us-east-1:555666777888:stateful-rulegroup/block-malicious-domains\"
          }
        ],
        \"networkFirewallOrchestrationConfig\": {
          \"singleFirewallEndpointPerVPC\": false,
          \"allowedIPV4CidrList\": []
        }
      }"
    },
    "ResourceType": "AWS::EC2::VPC",
    "RemediationEnabled": true
  }'
```

### 5. DNS Firewall Policy (Chính Sách Tường Lửa DNS)

Chặn DNS queries đến các domain độc hại trên toàn org.

```bash
aws fms put-policy \
  --policy '{
    "PolicyName": "Block-Malicious-DNS",
    "SecurityServicePolicyData": {
      "Type": "DNS_FIREWALL",
      "ManagedServiceData": "{
        \"type\": \"DNS_FIREWALL\",
        \"preProcessRuleGroups\": [
          {
            \"ruleGroupId\": \"rslvr-frg-malware-domains\",
            \"priority\": 1
          }
        ],
        \"postProcessRuleGroups\": []
      }"
    },
    "ResourceType": "AWS::EC2::VPC",
    "RemediationEnabled": true
  }'
```

---

## 📊 Quản Lý và Monitoring

### Xem Compliance Status

```bash
# Xem trạng thái compliance của tất cả policies
aws fms list-policies \
  --query 'PolicyList[*].{Name:PolicyName, Type:SecurityServiceType, Id:PolicyId}'

# Xem compliance cho một policy cụ thể
POLICY_ID="your-policy-id"
aws fms get-compliance-detail \
  --policy-id $POLICY_ID \
  --member-account "123456789012"

# Xem tất cả violations
aws fms list-compliance-status \
  --policy-id $POLICY_ID \
  --query 'PolicyComplianceStatusList[*].{Account:MemberAccount, Status:EvaluationResults}'
```

### Dashboard Compliance Tổng Hợp

```bash
# Lấy violations cho tất cả accounts trong một policy
aws fms list-violation-details \
  --policy-id $POLICY_ID \
  --member-account "123456789012" \
  --resource-id "sg-1234abcd" \
  --resource-type "AWS::EC2::SecurityGroup"
```

### Kích Hoạt Thông Báo

```bash
# Tạo SNS topic cho Firewall Manager violations
aws sns create-topic --name "firewall-manager-violations"

# Tạo EventBridge rule bắt Firewall Manager events
aws events put-rule \
  --name "FMSPolicyViolation" \
  --event-pattern '{
    "source": ["aws.fms"],
    "detail-type": ["FMS Policy State Change"]
  }'

# Lambda function xử lý violation và gửi Slack notification
aws events put-targets \
  --rule "FMSPolicyViolation" \
  --targets '[{
    "Id": "SlackNotifier",
    "Arn": "arn:aws:lambda:us-east-1:555666777888:function:fms-violation-notifier"
  }]'
```

---

## ⚙️ Cấu Hình Nâng Cao

### Include vs Exclude Map (Bao Gồm vs Loại Trừ)

```json
// Áp dụng cho tất cả accounts TRỪ sandbox
{
  "ExcludeMap": {
    "ACCOUNT": ["sandbox-account-id-999"]
  }
}

// Áp dụng CHỈ cho Production OU
{
  "IncludeMap": {
    "ORG_UNIT": ["ou-prod-yyyyyyy"]
  }
}

// Áp dụng cho resources có tag Environment=production
{
  "ResourceTags": [
    {"Key": "Environment", "Value": "production"}
  ],
  "ExcludeResourceTags": false
}

// Áp dụng cho tất cả TRỪ resources có tag ExcludeFromFMS=true
{
  "ResourceTags": [
    {"Key": "ExcludeFromFMS", "Value": "true"}
  ],
  "ExcludeResourceTags": true
}
```

### Hierarchical Policies (Chính Sách Phân Tầng)

```
Policy Layers:
├── Layer 1 — Org-wide Baseline (Nền Tảng Toàn Tổ Chức):
│   ├── Block all public SSH/RDP
│   ├── Mandatory WAF for all ALBs
│   └── Shield Advanced for production resources
│
├── Layer 2 — OU-level (Cấp Đơn Vị Tổ Chức):
│   ├── Production OU: Stricter WAF rules, no exceptions
│   └── Development OU: Relaxed rules, allow testing
│
└── Layer 3 — Account-level (Cấp Tài Khoản):
    └── Specific overrides approved by Security team
```

---

## 🔐 IAM Roles Cần Thiết

### AWSServiceRoleForFMS (Tự Động Tạo)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "wafv2:*",
        "shield:*",
        "ec2:DescribeSecurityGroups",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress",
        "network-firewall:*",
        "route53resolver:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Role Cho Security Admin

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "fms:*",
        "organizations:DescribeOrganization",
        "organizations:ListAccounts",
        "wafv2:ListWebACLs",
        "wafv2:GetWebACL",
        "shield:DescribeProtection",
        "ec2:DescribeSecurityGroups",
        "config:DescribeConfigRules"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 💰 Chi Phí

```
Mô Hình Tính Phí:
├── Per policy per region per account:
│   ├── WAF Policy: $100/policy/region/month
│   ├── Security Group Policy: $100/policy/region/month
│   ├── Shield Advanced Policy: Bao gồm trong Shield ($3,000/month)
│   └── Network Firewall Policy: $100/policy/region/month
│
├── WAF Web ACL: $5/WebACL/month (tạo bởi FMS)
│
└── WAF Rule evaluations: $0.60 per 1M requests

Ví Dụ Chi Phí Thực Tế:
├── 3 WAF policies × 2 regions × 20 accounts = $12,000/month policies
│   Nhưng FMS share một WebACL → chỉ tính $5/WebACL/account
├── Thực tế: 3 policies × 2 regions + 20 WebACLs × $5 = $700/month
└── + WAF request evaluations theo lưu lượng thực tế
```

---

## 🔍 Xử Lý Sự Cố Phổ Biến

### Policy Không Được Apply Vào Account Mới

```
Triệu Chứng: Account mới join org nhưng không có WAF policy

Nguyên Nhân:
1. AWS Config chưa bật ở account mới
2. Service-linked role của FMS chưa được tạo
3. Account nằm trong ExcludeMap

Khắc Phục:
1. Bật AWS Config trong account mới
2. Đảm bảo IAM role AWSServiceRoleForFMS tồn tại:
   aws iam get-role --role-name AWSServiceRoleForFMS
3. Kiểm tra policy scope không exclude account
4. Trigger manual remediation:
   aws fms put-notification-channel \
     --sns-topic-arn "arn:aws:sns:..." \
     --sns-role-name "FMSNotificationRole"
```

### Auto-Remediation Không Hoạt Động

```
Triệu Chứng: Policy báo NON_COMPLIANT nhưng không tự fix

Kiểm Tra:
1. RemediationEnabled: true đã set trong policy
2. Service role có đủ permissions
3. Config recorder đang chạy (FMS dùng Config để detect violations)

Debug:
# Xem violation details
aws fms list-violation-details \
  --policy-id $POLICY_ID \
  --member-account $ACCOUNT_ID \
  --resource-id $RESOURCE_ID \
  --resource-type "AWS::EC2::SecurityGroup"

# Xem FMS events trong CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes '[{
    "AttributeKey": "EventSource",
    "AttributeValue": "fms.amazonaws.com"
  }]' \
  --start-time $(date -d '1 hour ago' --iso-8601=seconds)
```

---

## 📌 Best Practices

### 1. Bắt Đầu Với Audit Mode (Chế Độ Kiểm Toán)

```
Giai Đoạn 1 — Audit (Không Auto-Remediate):
├── Set RemediationEnabled: false
├── Thu thập dữ liệu violations trong 2-4 tuần
├── Phân tích và hiểu pattern vi phạm
└── Lên kế hoạch fix trước khi bật auto-remediation

Giai Đoạn 2 — Gradual Enforcement (Thực Thi Dần Dần):
├── Bật auto-remediation cho Development account trước
├── Theo dõi ảnh hưởng, điều chỉnh exceptions nếu cần
├── Mở rộng sang Staging sau 2 tuần ổn định
└── Cuối cùng bật cho Production

Giai Đoạn 3 — Full Enforcement (Thực Thi Đầy Đủ):
├── Tất cả accounts được cover
├── New accounts tự động được protect
└── Regular review và update policies
```

### 2. Quản Lý Exceptions

```bash
# Tag resources cần exception
aws ec2 create-tags \
  --resources "sg-1234abcd" \
  --tags Key=FMSException,Value=approved-by-security-team-2024-10-15

# Cấu hình policy exclude resources có tag này
aws fms put-policy \
  --policy '{
    ...
    "ResourceTags": [{"Key": "FMSException", "Value": "approved*"}],
    "ExcludeResourceTags": true,
    ...
  }'
```

### 3. Tích Hợp Với ITSM (IT Service Management — Quản Lý Dịch Vụ CNTT)

```python
import boto3
import json
import requests

def create_jira_ticket_for_fms_violation(event, context):
    """Tạo Jira ticket khi Firewall Manager phát hiện violation"""
    
    violation_details = event['detail']
    account_id = event['account']
    
    # Lấy thông tin chi tiết violation
    fms = boto3.client('fms', region_name='us-east-1')
    
    ticket_data = {
        "fields": {
            "project": {"key": "SEC"},
            "summary": f"FMS Violation: {violation_details['policyName']} in {account_id}",
            "description": f"""
**Firewall Manager Policy Violation**

- **Policy:** {violation_details['policyName']}
- **Account:** {account_id}
- **Resource:** {violation_details.get('resourceId', 'N/A')}
- **Violation Type:** {violation_details.get('violationType', 'N/A')}
- **Time:** {event['time']}

**Required Action:** Review và fix within 24 giờ cho Critical, 72 giờ cho High.
            """,
            "issuetype": {"name": "Security Finding"},
            "priority": {"name": "High"},
            "labels": ["fms-violation", "auto-created"]
        }
    }
    
    response = requests.post(
        "https://your-company.atlassian.net/rest/api/2/issue",
        json=ticket_data,
        auth=("user@company.com", "api-token")
    )
    
    return response.json()
```

---

## 💡 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Firewall Manager WAF Policy và WAF WebACL thông thường?**
> WAF WebACL thông thường bạn tạo trong từng account riêng lẻ, cần manage thủ công ở mỗi nơi. Firewall Manager WAF Policy tạo và quản lý WebACL đồng nhất trên nhiều accounts/regions từ một nơi duy nhất, tự động áp dụng cho accounts mới, và báo cáo compliance tập trung. FMS là lớp quản lý ở trên WAF.

**Q: Firewall Manager có thể thay thế Security Groups không?**
> Không hoàn toàn. FMS Security Group policies không thay thế Security Groups — chúng kiểm soát và audit các Security Groups đã tồn tại. FMS có thể: (1) Phát hiện Security Groups vi phạm quy tắc (VD: allow 0.0.0.0/0 port 22), (2) Tùy chọn auto-remediate bằng cách xóa rule vi phạm, (3) Phát hiện Security Groups không được dùng. Nhưng Security Groups vẫn là cơ chế firewall thực sự.

**Q: Có thể dùng Firewall Manager mà không cần AWS Organizations không?**
> Không. Firewall Manager yêu cầu AWS Organizations bắt buộc vì nó dùng Organizations để tự động phát hiện và áp dụng policies cho member accounts. Không có Organizations, không có Firewall Manager. Đây là một trong những lý do chính để migrate sang Organizations structure.

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [2-conformance-packs.md](2-conformance-packs.md) | **3-firewall-manager.md** | [4-pci-dss-aws.md](4-pci-dss-aws.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
