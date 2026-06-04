# 🛡️ Zero Trust Architecture — Kiến Trúc Không Tin Tưởng Mặc Định Trên AWS

> **Zero Trust** là triết lý bảo mật dựa trên nguyên tắc **"Never trust, always verify"** — không bao giờ tin tưởng mặc định bất kỳ request nào, kể cả từ bên trong mạng nội bộ. Mọi yêu cầu truy cập đều phải được xác thực danh tính, kiểm tra thiết bị, và cấp quyền tối thiểu — liên tục, không phải chỉ một lần khi đăng nhập.

---

## 📚 Mục Lục

1. [Nguyên Tắc Nền Tảng Zero Trust](#1-nguyên-tắc-nền-tảng-zero-trust)
2. [7 Tenets Của NIST Zero Trust Architecture](#2-7-tenets-của-nist-zero-trust-architecture)
3. [AWS Services Trong Zero Trust](#3-aws-services-trong-zero-trust)
4. [AWS Verified Access — Truy Cập Không Cần VPN](#4-aws-verified-access)
5. [Network Microsegmentation](#5-network-microsegmentation)
6. [Identity-Centric Security](#6-identity-centric-security)
7. [Device Trust](#7-device-trust)
8. [Continuous Monitoring và Re-verification](#8-continuous-monitoring-và-re-verification)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Kiến Trúc Tham Chiếu](#10-kiến-trúc-tham-chiếu)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Nguyên Tắc Nền Tảng Zero Trust

### Mô Hình Bảo Mật Cũ vs Zero Trust

```
MÔ HÌNH CŨ — Castle & Moat (Lâu Đài và Hào):
┌─────────────────────────────────────────────────────┐
│                   Trusted Zone                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Server A │  │ Server B │  │ Server C │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│   ↑ Trong mạng = tin tưởng, không cần xác thực lại  │
└──────────────────────────────────────────────────────┘
          ↑ Firewall ↑
[Internet — Không tin tưởng]

Vấn đề: Attacker vào được bên trong → lateral movement tự do


MÔ HÌNH ZERO TRUST:
[Internet] ──► [Identity Verification]
[Internal] ──► [Device Check        ] ──► [Decision Engine] ──► [Resource]
[Partner]  ──► [Context Evaluation  ]       (Allow/Deny)

Mọi request đều bị xử lý như nhau — không có vùng "an toàn" mặc định
```

### 3 Nguyên Tắc Cốt Lõi

| Nguyên Tắc | Mô Tả | AWS Implementation |
|---|---|---|
| **Verify explicitly** (Xác minh rõ ràng) | Xác thực và ủy quyền mọi request dựa trên tất cả dữ liệu available | IAM, Cognito, Verified Access |
| **Use least privilege** (Đặc quyền tối thiểu) | Cấp quyền tối thiểu cần thiết, dùng JIT và time-limited access | IAM Conditions, SCP, Permission Boundary |
| **Assume breach** (Giả định đã bị xâm phạm) | Thiết kế hệ thống như thể attacker đã ở trong; minimize blast radius | VPC segmentation, GuardDuty, logging |

---

## 2. 7 Tenets Của NIST Zero Trust Architecture

NIST SP 800-207 định nghĩa 7 nguyên tắc nền tảng của Zero Trust:

```
Tenet 1: All data sources and computing services are resources
         (Mọi nguồn dữ liệu và dịch vụ đều là tài nguyên cần bảo vệ)
         AWS: S3, RDS, ECS, Lambda, APIs đều là resources cần policy

Tenet 2: All communication is secured regardless of network location
         (Mọi giao tiếp đều phải được mã hóa dù ở mạng nào)
         AWS: TLS in-transit, VPC PrivateLink, mTLS với App Mesh

Tenet 3: Access to resources is granted on a per-session basis
         (Cấp quyền theo từng phiên, không phải vĩnh viễn)
         AWS: STS temporary credentials, IAM session policies

Tenet 4: Access is determined by dynamic policy
         (Quyền được quyết định bởi policy động, xét nhiều yếu tố)
         AWS: IAM Conditions, Verified Access policies

Tenet 5: Monitor and measure integrity/security posture of all assets
         (Giám sát trạng thái bảo mật mọi tài sản liên tục)
         AWS: Security Hub, GuardDuty, Inspector, Config

Tenet 6: Authentication and authorization are dynamic and enforced
         (Xác thực và ủy quyền là động và được thi hành liên tục)
         AWS: Verified Access continuous re-auth, short-lived tokens

Tenet 7: Collect as much data as possible to improve security posture
         (Thu thập dữ liệu tối đa để cải thiện bảo mật)
         AWS: CloudTrail, VPC Flow Logs, CloudWatch Logs
```

---

## 3. AWS Services Trong Zero Trust

### Kiến Trúc Tổng Quan Các Services

```
┌────────────────────────────────────────────────────────────────────┐
│                      ZERO TRUST LAYERS                              │
│                                                                      │
│  IDENTITY LAYER (Lớp Định Danh)                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐   │
│  │ IAM + STS   │  │ Identity    │  │ Cognito (App users)      │   │
│  │ (workforce) │  │ Center (SSO)│  │                          │   │
│  └─────────────┘  └─────────────┘  └──────────────────────────┘   │
│                                                                      │
│  DEVICE LAYER (Lớp Thiết Bị)                                        │
│  ┌─────────────────────┐  ┌─────────────────────────────────────┐  │
│  │ Systems Manager     │  │ AWS IoT Device Defender             │  │
│  │ (EC2 posture check) │  │ (IoT device trust)                  │  │
│  └─────────────────────┘  └─────────────────────────────────────┘  │
│                                                                      │
│  NETWORK LAYER (Lớp Mạng)                                           │
│  ┌──────────────┐  ┌────────────┐  ┌──────────┐  ┌────────────┐   │
│  │ Verified     │  │ VPC Lattice│  │ CloudFront│  │ WAF        │   │
│  │ Access       │  │ (micro-    │  │           │  │            │   │
│  │ (app access) │  │ segment)   │  └──────────┘  └────────────┘   │
│  └──────────────┘  └────────────┘                                   │
│                                                                      │
│  DETECTION LAYER (Lớp Phát Hiện)                                    │
│  ┌──────────────┐  ┌────────────┐  ┌────────────────────────────┐  │
│  │ GuardDuty    │  │ Security   │  │ CloudTrail + Athena        │  │
│  │ (anomaly)    │  │ Hub        │  │ (audit trail)              │  │
│  └──────────────┘  └────────────┘  └────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### Bảng Services và Vai Trò Zero Trust

| AWS Service | Vai Trò Trong Zero Trust | Zero Trust Tenet |
|---|---|---|
| **IAM + STS** | Identity verification, least privilege, temp credentials | 1, 3, 4 |
| **IAM Identity Center** | Centralized SSO, workforce identity | 1, 6 |
| **AWS Verified Access** | Per-request application access without VPN | 4, 6 |
| **VPC Lattice** | Service-to-service microsegmentation | 1, 2, 4 |
| **CloudFront + WAF** | Edge security, request inspection | 2, 4 |
| **GuardDuty** | Threat detection, anomaly identification | 5, 7 |
| **Security Hub** | Centralized security posture | 5, 7 |
| **AWS Inspector** | Vulnerability assessment | 5 |
| **AWS Config** | Compliance monitoring, drift detection | 5, 7 |
| **Systems Manager** | Device posture, patch compliance | 5 |
| **CloudTrail** | Audit logging mọi API call | 7 |
| **VPC Flow Logs** | Network traffic visibility | 7 |

---

## 4. AWS Verified Access

AWS Verified Access (truy cập đã xác minh) cho phép cấp quyền truy cập ứng dụng nội bộ **mà không cần VPN** — mỗi request được xác minh dựa trên identity và device posture.

### So Sánh VPN Truyền Thống vs Verified Access

```
VPN TRUYỀN THỐNG:
User ──VPN connect──► [Trusted Network] ──► Mọi ứng dụng nội bộ
                           ↑
                     Vào được VPN = vào được tất cả
                     (Blast radius: rất lớn)

AWS VERIFIED ACCESS:
User ──request──► [Verified Access] ──► Kiểm tra identity + device
                       │
                       ├── Identity: IAM Identity Center, Okta, Azure AD
                       ├── Device: Jamf, CrowdStrike, Jamf posture
                       └── Context: IP, time, location
                       │
                       ▼ Per-request decision
                  ──► [App A] hoặc [App B] (chỉ app được phép)
                     (Blast radius: nhỏ — chỉ app cụ thể)
```

### Cấu Trúc Verified Access

```
Verified Access Trust Provider (Nguồn tin cậy):
├── Identity Trust Provider: IAM Identity Center, OIDC (Okta, Azure AD)
└── Device Trust Provider: Jamf, CrowdStrike, JumpCloud, Qualys

Verified Access Group (Nhóm):
└── Policy: Cedar policy language

Verified Access Endpoint (Điểm cuối ứng dụng):
├── Type: HTTP / load balancer / network interface
└── Gắn với 1 Verified Access Group
```

### Cấu Hình Verified Access Qua CLI

```bash
# Tạo Verified Access Trust Provider (dùng IAM Identity Center)
aws ec2 create-verified-access-trust-provider \
  --trust-provider-type user \
  --user-trust-provider-type iam-identity-center \
  --policy-reference-name idc \
  --description "IAM Identity Center trust provider"

# Tạo Verified Access Instance
aws ec2 create-verified-access-instance \
  --description "Production Zero Trust access"

# Gắn trust provider vào instance
aws ec2 attach-verified-access-trust-provider \
  --verified-access-instance-id vai-1234567890abcdef0 \
  --verified-access-trust-provider-id vatp-1234567890abcdef0

# Tạo Verified Access Group với policy
aws ec2 create-verified-access-group \
  --verified-access-instance-id vai-1234567890abcdef0 \
  --policy-document '{
    "cedar": "permit(principal, action, resource)\nwhen {\n  context.idc.groups.contains(\"DevTeam\") &&\n  context.idc.email.endsWith(\"@company.com\")\n};"
  }' \
  --description "Dev team access group"

# Tạo endpoint cho internal app
aws ec2 create-verified-access-endpoint \
  --verified-access-group-id vagr-1234567890abcdef0 \
  --endpoint-type load-balancer \
  --attachment-type vpc \
  --protocol https \
  --domain-certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/xxx \
  --endpoint-domain-prefix "internal-app" \
  --load-balancer-options '{
    "LoadBalancerArn": "arn:aws:elasticloadbalancing:...",
    "Port": 443,
    "Protocol": "https",
    "SubnetIds": ["subnet-xxx"]
  }'
```

### Verified Access Policy (Cedar Language)

```cedar
// Cho phép truy cập nếu user thuộc group DevTeam VÀ email công ty
permit(principal, action, resource)
when {
  context.idc.groups.contains("DevTeam") &&
  context.idc.email.endsWith("@company.com")
};

// Cho phép truy cập nếu device compliant VÀ không roaming
permit(principal, action, resource)
when {
  context.idc.email.endsWith("@company.com") &&
  context.jamf.is_compliant == true &&
  context.jamf.is_roaming == false
};

// Deny nếu user bị suspend
forbid(principal, action, resource)
when {
  context.idc.status == "suspended"
};
```

---

## 5. Network Microsegmentation

### Security Groups Như Microsegmentation Layer

```
TRUYỀN THỐNG — Flat Network:
[Web] → [App] → [DB]  (không có kiểm soát lateral movement)

MICROSEGMENTATION với Security Groups:
[Web-SG] → (chỉ port 443) → [App-SG] → (chỉ port 5432) → [DB-SG]
    ↑                             ↑                              ↑
    Chỉ nhận từ ALB          Chỉ nhận từ Web-SG            Chỉ nhận từ App-SG
```

**Security Group Rules Best Practice:**

```bash
# Tạo tiered security groups
# Web tier - nhận từ ALB
aws ec2 create-security-group \
  --group-name "web-sg" \
  --description "Web tier - accepts from ALB only" \
  --vpc-id vpc-xxx

aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp \
  --port 443 \
  --source-group sg-alb  # Chỉ từ ALB SG, không từ 0.0.0.0/0

# App tier - nhận từ Web tier
aws ec2 create-security-group \
  --group-name "app-sg" \
  --description "App tier - accepts from web tier only"

aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 8080 \
  --source-group sg-web  # Chỉ từ Web SG

# DB tier - nhận từ App tier only
aws ec2 authorize-security-group-ingress \
  --group-id sg-db \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app  # Chỉ từ App SG
```

### VPC Lattice — Service-to-Service Zero Trust

VPC Lattice là managed service cho phép kiểm soát traffic giữa services với **identity-based policies** — không phụ thuộc vào IP/CIDR.

```
KHÔNG CÓ VPC LATTICE:
Service A (VPC-1) → peering → Service B (VPC-2)
  → Phải quản lý CIDR, route tables, security groups phức tạp
  → Không biết "ai" đang gọi — chỉ biết IP nguồn

VỚI VPC LATTICE:
Service A → [VPC Lattice Service Network] → Service B
  → Policy: "Chỉ role với tag team=payments được gọi /api/payments"
  → Identity-aware, không phụ thuộc IP
  → Works cross-VPC, cross-account tự động
```

**VPC Lattice Auth Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "vpc-lattice-svcs:Invoke",
      "Resource": "arn:aws:vpc-lattice:us-east-1:123456789012:service/svc-xxx/",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/team": "payments",
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

```bash
# Tạo VPC Lattice service network
aws vpc-lattice create-service-network \
  --name "production-network" \
  --auth-type "AWS_IAM"

# Tạo service
aws vpc-lattice create-service \
  --name "payments-api" \
  --auth-type "AWS_IAM"

# Gắn auth policy
aws vpc-lattice put-auth-policy \
  --resource-identifier svc-xxx \
  --policy file://lattice-auth-policy.json
```

---

## 6. Identity-Centric Security

### Từ Network-Centric Sang Identity-Centric

```
NETWORK-CENTRIC (Cũ):                IDENTITY-CENTRIC (Zero Trust):
"IP 10.0.1.5 được phép"             "Role payments-processor được phép"
"Subnet 10.0.1.0/24 trusted"        "User alice@company.com authenticated"
                                      "Device compliant with MDM"
                                      "MFA completed 5 min ago"
```

### IAM Conditions Cho Identity-Centric Controls

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::sensitive-data/*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "3600"
        },
        "StringEquals": {
          "aws:PrincipalTag/clearance": "confidential"
        },
        "IpAddress": {
          "aws:SourceIp": ["10.0.0.0/8", "172.16.0.0/12"]
        }
      }
    }
  ]
}
```

### JIT Access — Just-In-Time Access (Truy Cập Đúng Lúc)

```
JIT Access Pattern:
1. Developer request access qua ticketing system (ServiceNow, Jira)
2. Automation tạo temporary IAM role với specific permissions
3. Role chỉ tồn tại trong X giờ (ví dụ: 4 giờ)
4. CloudTrail log mọi action trong window đó
5. Role tự động hết hạn / bị revoke sau khi ticket close

vs STANDING PRIVILEGES:
  → Developer có access mọi lúc
  → Nếu credential bị lộ: bị lợi dụng bất cứ lúc nào
```

```bash
# Script tạo JIT access (chạy từ automation khi ticket approved)
#!/bin/bash
DEVELOPER=$1
DURATION_HOURS=$2
RESOURCE_ARN=$3

# Tạo role tạm thời với expiry
aws iam create-role \
  --role-name "jit-${DEVELOPER}-$(date +%s)" \
  --assume-role-policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": {\"AWS\": \"arn:aws:iam::123456789012:user/${DEVELOPER}\"},
      \"Action\": \"sts:AssumeRole\",
      \"Condition\": {
        \"DateLessThan\": {
          \"aws:CurrentTime\": \"$(date -u -d "+${DURATION_HOURS} hours" +%Y-%m-%dT%H:%M:%SZ)\"
        }
      }
    }]
  }"

# Gắn policy với resource cụ thể
# ... (schedule EventBridge rule để xóa role sau DURATION_HOURS giờ)
```

---

## 7. Device Trust

### Systems Manager — Kiểm Tra Trạng Thái EC2

```bash
# Kiểm tra patch compliance của tất cả instances
aws ssm describe-instance-patch-states \
  --instance-ids $(aws ec2 describe-instances \
    --query 'Reservations[].Instances[].InstanceId' \
    --output text) \
  --query 'InstancePatchStates[?InstalledPendingRebootCount>`0`]'

# Tag instance không compliant
aws ec2 create-tags \
  --resources i-compromised \
  --tags Key=patch-compliant,Value=false

# Policy từ chối truy cập nếu instance không compliant
# (Dùng trong IAM policy với condition check tag)
```

### Device Posture Với Verified Access + Jamf/CrowdStrike

```
Flow kiểm tra device:

1. User mở browser → truy cập app.company.com
2. Verified Access intercept request
3. Check identity: IAM Identity Center → authenticated ✅
4. Check device:
   ├── Jamf: MDM enrolled ✅, OS up-to-date ✅, disk encrypted ✅
   └── CrowdStrike: No active threats ✅
5. Policy evaluation:
   ├── User in group "DevTeam" ✅
   ├── Device compliant ✅
   └── Company email ✅
6. → ALLOW → Forward request đến app

Nếu device không compliant:
→ DENY → Redirect đến trang "Hãy cập nhật OS trước khi truy cập"
```

---

## 8. Continuous Monitoring và Re-verification

### GuardDuty Cho Continuous Threat Detection

```
GuardDuty liên tục phân tích:
├── CloudTrail events    → "alice đang call GetObject 10,000 lần/phút" → ANOMALY
├── VPC Flow Logs       → "EC2 đang connect ra IP ở nước ngoài lạ" → SUSPICIOUS
├── DNS Logs            → "Lambda gọi domain known-malware-c2.com" → MALWARE
└── EKS Audit Logs      → "Pod đang privilege escalate" → COMPROMISE

Khi phát hiện:
→ GuardDuty finding → Security Hub → EventBridge → Lambda remediation
→ Tự động revoke credential, isolate instance, block IP
```

### Re-verification Triggers (Kích Hoạt Xác Minh Lại)

```
Events cần trigger re-verification:

1. Sau khoảng thời gian nhất định:
   └── STS token expire → bắt buộc assume role lại → re-auth

2. Khi phát hiện anomaly:
   └── GuardDuty alert → revoke current session → yêu cầu MFA lại

3. Khi context thay đổi:
   └── IP address thay đổi → step-up auth (xác thực tăng cường)
   └── Thời điểm bất thường → re-verify

4. Khi quyền được mở rộng (privilege escalation attempt):
   └── Audit log → alert → human approval required
```

### CloudTrail + Athena Cho Forensics

```sql
-- Truy vấn Athena tìm lateral movement
SELECT
  useridentity.arn,
  COUNT(DISTINCT sourceipaddress) AS distinct_ips,
  COUNT(DISTINCT eventname) AS distinct_actions,
  MIN(eventtime) AS first_seen,
  MAX(eventtime) AS last_seen
FROM cloudtrail_logs
WHERE
  eventtime BETWEEN '2026-05-15' AND '2026-05-16'
  AND errorcode IS NULL
GROUP BY useridentity.arn
HAVING COUNT(DISTINCT sourceipaddress) > 3  -- Cùng 1 user từ nhiều IP
ORDER BY distinct_actions DESC
LIMIT 50;
```

---

## 9. Implementation Roadmap

### Từ Traditional → Zero Trust (Lộ Trình Chuyển Đổi)

```
PHASE 1 — FOUNDATION (Tháng 1-3): "Know your identities"
├── Inventory tất cả IAM users/roles/service accounts
├── Enable MFA cho mọi human user
├── Enable CloudTrail, GuardDuty, Security Hub
├── Phân loại tài nguyên (data classification)
└── Xây dựng baseline: ai truy cập gì

PHASE 2 — STRENGTHEN IDENTITY (Tháng 4-6): "Identity-first"
├── Migrate sang IAM Identity Center (Single Sign-On)
├── Eliminate long-lived access keys (thay bằng roles/OIDC)
├── Implement ABAC cho team/env isolation
├── Enable MFA step-up cho sensitive actions
└── Audit và revoke unused permissions

PHASE 3 — SEGMENT NETWORK (Tháng 7-9): "Micro-perimeters"
├── Implement Security Group microsegmentation
├── Migrate private app access sang Verified Access (thay VPN)
├── Deploy VPC Lattice cho service-to-service
├── Enable VPC Flow Logs và anomaly detection
└── Private endpoints cho mọi AWS service (không qua internet)

PHASE 4 — AUTOMATE (Tháng 10-12): "Continuous verification"
├── Auto-remediation cho common findings (GuardDuty → Lambda)
├── JIT access workflow cho privileged access
├── Device posture checks qua Verified Access
├── Continuous compliance monitoring (Config + Security Hub)
└── Incident response runbooks hoàn chỉnh

PHASE 5 — OPTIMIZE (Năm 2+): "Mature Zero Trust"
├── Machine learning anomaly detection
├── User behavior analytics (UEBA)
├── Risk-based adaptive access
└── Pen testing và Red team exercises
```

### Metrics Đo Lường Tiến Độ

| Metric | Baseline | Target Phase 2 | Target Phase 4 |
|---|---|---|---|
| % users có MFA | ~50% | 100% | 100% |
| % access keys > 90 days | ~40% | < 5% | 0% |
| % resources có tags | ~30% | > 80% | > 95% |
| % ứng dụng qua VPN | 100% | 70% | < 10% |
| MTTR khi detect threat | 4 giờ | 30 phút | < 5 phút |
| % automated remediation | 0% | 30% | > 70% |

---

## 10. Kiến Trúc Tham Chiếu

### Zero Trust Reference Architecture Trên AWS

```
INTERNET / PARTNER / REMOTE EMPLOYEE
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        EDGE LAYER                                    │
│  ┌─────────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ CloudFront  │     │    WAF       │     │  Route53 Resolver  │   │
│  │ (CDN + TLS) │────►│ (L7 inspect) │     │  DNS Firewall      │   │
│  └─────────────┘     └──────────────┘     └────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     ACCESS LAYER                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              AWS Verified Access                              │   │
│  │  ┌───────────────────┐  ┌──────────────────────────────┐    │   │
│  │  │ Identity Provider │  │    Device Trust Provider     │    │   │
│  │  │ IAM Identity Ctr  │  │    (Jamf/CrowdStrike)        │    │   │
│  │  └───────────────────┘  └──────────────────────────────┘    │   │
│  │           │                          │                        │   │
│  │           └──────────┬───────────────┘                       │   │
│  │                      ▼                                        │   │
│  │            Cedar Policy Evaluation                            │   │
│  │         (Allow / Deny per request)                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
          │ (Only allowed requests pass)
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                                  │
│                                                                      │
│  ┌─────────────┐         ┌─────────────┐         ┌──────────────┐  │
│  │   App A     │         │   App B     │         │   App C      │  │
│  │  (EC2/ECS)  │         │  (Lambda)   │         │  (EKS)       │  │
│  └──────┬──────┘         └──────┬──────┘         └──────┬───────┘  │
│         │                       │                         │          │
│         └───────────────────────┴─────────────────────────┘          │
│                                 │                                    │
│                    ┌────────────▼───────────┐                       │
│                    │     VPC Lattice         │                       │
│                    │ (Service-to-Service     │                       │
│                    │  Identity-based authz)  │                       │
│                    └────────────┬───────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     DATA LAYER                                       │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │   S3     │  │   RDS    │  │ DynamoDB │  │   Secrets Manager  │  │
│  │ (bucket  │  │ (VPC     │  │          │  │   KMS              │  │
│  │  policy) │  │  only)   │  │          │  │                    │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘  │
│                                                                      │
│  ALL ACCESS VIA VPC ENDPOINTS (không qua internet)                  │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼ Logs từ mọi layer
┌─────────────────────────────────────────────────────────────────────┐
│                  MONITORING LAYER                                    │
│  CloudTrail │ VPC Flow Logs │ GuardDuty │ Security Hub │ Inspector  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q1: Zero Trust là gì và tại sao nó quan trọng hơn mô hình perimeter truyền thống?**

> Zero Trust dựa trên nguyên tắc "Never trust, always verify" — không tin tưởng mặc định bất kỳ request nào dù đến từ mạng nội bộ hay bên ngoài. Quan trọng hơn vì: (1) 80% data breach có liên quan đến compromised credentials — kẻ tấn công đã ở "trong mạng"; (2) Mô hình làm việc từ xa (remote work) đã phá vỡ khái niệm "mạng nội bộ"; (3) Cloud workloads không có perimeter rõ ràng.

**Q2: AWS Verified Access giải quyết vấn đề gì so với VPN truyền thống?**

> VPN truyền thống cấp quyền truy cập toàn bộ "trusted network" khi user connect thành công — blast radius rất lớn. Verified Access cấp quyền truy cập theo từng ứng dụng cụ thể, xác minh theo từng request dựa trên identity + device posture. Không cần VPN client, không có "network-level access".

**Q3: VPC Lattice giúp gì cho Zero Trust trong microservices?**

> VPC Lattice cho phép kiểm soát service-to-service traffic dựa trên identity (IAM role/tag), không dựa vào IP/CIDR. Điều này thực hiện Zero Trust tenet "verify identity, not network location" cho internal service mesh. Policy có thể kiểm soát: role nào được gọi endpoint nào, với HTTP method nào.

**Q4: Sự khác biệt giữa Zero Trust network và Zero Trust workload?**

> Zero Trust network kiểm soát "traffic đi qua đâu" — microsegmentation, Security Groups, VPC Lattice. Zero Trust workload kiểm soát "identity nào được làm gì" — IAM roles, service accounts, mTLS certificates. Zero Trust thực sự cần cả hai lớp: dù traffic đi qua được network layer, workload layer vẫn re-verify identity.

**Q5: Làm thế nào để implement JIT (Just-In-Time) access trên AWS?**

> JIT access trên AWS: (1) Không có standing privileges — developer không có quyền mặc định; (2) Khi cần, request qua ticketing system → workflow tạo temporary IAM role với specific permissions và expiry time; (3) CloudTrail log mọi action; (4) EventBridge rule tự động thu hồi role sau khi hết thời gian hoặc ticket close. AWS IAM Identity Center cũng hỗ trợ time-bound permission assignment.

**Q6: Làm thế nào bạn đo lường "độ trưởng thành" Zero Trust của một tổ chức?**

> Dựa trên CISA Zero Trust Maturity Model (5 pillars): Identity (% MFA, % passwordless), Device (% MDM enrolled, % compliant), Network (% microsegmented, % Verified Access), Application (% API auth-required), Data (% classified, % encrypted). Có thể dùng metrics: % long-lived credentials eliminated, % lateral movement attempts blocked, MTTR cho threat detection.

---

## 12. Key Takeaways

> **Zero Trust không phải sản phẩm** — đó là triết lý thiết kế và tập hợp các thực hành. AWS cung cấp building blocks (IAM, Verified Access, VPC Lattice, GuardDuty), nhưng bạn phải kết hợp chúng theo nguyên tắc Zero Trust.

> **"Never trust, always verify"** có nghĩa là mỗi request phải kèm proof of identity — không phải chỉ "đã login một lần là xong". STS temporary credentials, session policies, và MFA step-up là các công cụ thực thi điều này.

> **AWS Verified Access** là bước tiến lớn nhất cho Zero Trust trên AWS — thay thế VPN cho ứng dụng internal, với per-request identity + device verification.

> **VPC Lattice** đưa Zero Trust xuống tầng service-to-service — microservices không còn phụ thuộc IP/CIDR, mà xác thực lẫn nhau qua IAM identity.

> **Zero Trust journey là marathon, không phải sprint** — bắt đầu với identity (MFA, eliminate long-lived keys), sau đó segment network, sau đó automate verification. Cố gắng làm tất cả cùng lúc sẽ thất bại.
