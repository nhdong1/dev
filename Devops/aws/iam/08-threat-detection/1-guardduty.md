# GuardDuty — Phát Hiện Mối Đe Dọa Tự Động

> GuardDuty là dịch vụ phát hiện mối đe dọa (threat detection) liên tục, dựa trên Machine Learning (Học Máy) và threat intelligence feeds (nguồn thông tin tình báo về mối đe dọa). Hoạt động không cần agent, không ảnh hưởng hiệu năng workload.

## 📚 Mục Lục

1. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
2. [Nguồn Dữ Liệu](#nguồn-dữ-liệu)
3. [Finding Types — Phân Loại Phát Hiện](#finding-types)
4. [Severity Levels — Mức Độ Nghiêm Trọng](#severity-levels)
5. [Suppression Rules — Quy Tắc Loại Trừ](#suppression-rules)
6. [Multi-Account Setup](#multi-account-setup)
7. [Tự Động Phản Hồi](#tự-động-phản-hồi)
8. [Pricing — Chi Phí](#pricing)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Tổng Quan

```
AWS Infrastructure
    ├── CloudTrail (Management Events + Data Events)
    ├── VPC Flow Logs (Nhật Ký Luồng Mạng)
    ├── DNS Query Logs (Nhật Ký Truy Vấn DNS)
    ├── S3 Data Events (Sự Kiện Dữ Liệu S3)
    ├── EKS Audit Logs (Nhật Ký Kiểm Toán Kubernetes)
    ├── RDS/Aurora Login Events (Sự Kiện Đăng Nhập Cơ Sở Dữ Liệu)
    └── Lambda Network Activity (Hoạt Động Mạng Lambda)
            │
            ▼
    ┌─────────────────────────────┐
    │       GuardDuty Engine      │
    │  ┌──────────────────────┐   │
    │  │ ML Anomaly Detection  │   │  ← Phát hiện hành vi bất thường
    │  │ Threat Intelligence   │   │  ← So khớp với danh sách IP/domain độc hại
    │  │ Rule-based Detection  │   │  ← Quy tắc cố định cho các tấn công đã biết
    │  └──────────────────────┘   │
    └─────────────────────────────┘
            │
            ▼
    Findings → Security Hub → EventBridge → Lambda/SNS/SQS
```

### Nguyên Tắc Hoạt Động

GuardDuty **không cài agent** lên EC2 hay tài nguyên. Nó phân tích dữ liệu log thông qua API của AWS, do đó:
- Không làm chậm workload
- Không cần thay đổi cấu hình hiện có
- Tự động mở rộng khi tài nguyên tăng

---

## Nguồn Dữ Liệu

### Luôn Bật (Foundational Data Sources)

| Nguồn | Mô Tả | Loại Mối Đe Dọa Phát Hiện |
|---|---|---|
| **CloudTrail Management Events** | API calls tạo/xóa/sửa tài nguyên | Unauthorized API calls, account takeover (chiếm tài khoản) |
| **VPC Flow Logs** | Metadata traffic vào/ra instances | Port scanning, C2 communication (giao tiếp Command & Control) |
| **Route 53 DNS Logs** | Domain lookups từ VPC resources | Malware, cryptomining domains, DNS tunneling (đường hầm DNS) |

### Protection Plans (Kế Hoạch Bảo Vệ Tùy Chọn)

| Protection Plan | Nguồn Dữ Liệu Bổ Sung | Use Case |
|---|---|---|
| **S3 Protection** | S3 Data Events | Phát hiện exfiltration (rò rỉ dữ liệu), ransomware trên S3 |
| **EKS Protection** | EKS Audit Logs, EKS Runtime | Privilege escalation (leo thang đặc quyền) trong Kubernetes |
| **Malware Protection** | EBS volume snapshots | Phát hiện malware trên EC2 và ECS |
| **RDS Protection** | RDS/Aurora login activity | Credential stuffing (nhồi nhét thông tin đăng nhập), brute force |
| **Lambda Protection** | Lambda network activity | Cryptomining, data exfiltration từ Lambda |

---

## Finding Types — Phân Loại Phát Hiện

GuardDuty tổ chức findings theo định dạng: **`ThreatPurpose:ResourceType/ThreatFamilyName`**

### Threat Purpose (Mục Đích Tấn Công)

| Prefix | Ý Nghĩa |
|---|---|
| `Backdoor` | Cố gắng tạo backdoor (cửa hậu) vào hệ thống |
| `Behavior` | Hành vi bất thường so với baseline (đường cơ sở) |
| `CryptoCurrency` | Hoạt động đào tiền mã hóa (cryptomining) |
| `DefenseEvasion` | Cố gắng né tránh phát hiện (xóa logs, tắt monitoring) |
| `Discovery` | Trinh sát môi trường (liệt kê tài nguyên, users) |
| `Exfiltration` | Rò rỉ dữ liệu ra ngoài |
| `Impact` | Hành động gây thiệt hại (xóa dữ liệu, thay đổi cấu hình) |
| `InitialAccess` | Điểm xâm nhập ban đầu |
| `PenTest` | Hoạt động từ công cụ pentest (kiểm thử xâm nhập) |
| `Persistence` | Cố gắng duy trì quyền truy cập |
| `Policy` | Vi phạm best practices (thực hành tốt nhất) |
| `PrivilegeEscalation` | Leo thang đặc quyền |
| `Recon` | Reconnaissance (trinh sát) |
| `Stealth` | Ẩn hoạt động độc hại |
| `Trojan` | Phần mềm độc hại loại trojan |
| `UnauthorizedAccess` | Truy cập trái phép |

### Ví Dụ Finding Types Quan Trọng

#### Nhóm IAM/Credential Compromise (Xâm Phạm Thông Tin Đăng Nhập)

```
UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B
    ↳ Đăng nhập Console từ IP/địa điểm bất thường

UnauthorizedAccess:IAMUser/MaliciousIPCaller
    ↳ API call từ IP trong danh sách đen (threat intelligence list)

Persistence:IAMUser/UserPermissions
    ↳ IAM user tạo thêm users, access keys, hoặc policies mới — dấu hiệu persistence

PrivilegeEscalation:IAMUser/AdministrativePermissions
    ↳ Cố gắng gán quyền admin cho chính mình

Stealth:IAMUser/CloudTrailLoggingDisabled
    ↳ Tắt CloudTrail — thường là bước đầu sau khi xâm nhập

Impact:IAMUser/AnomalousBehavior
    ↳ Hành vi API bất thường — ML phát hiện sai lệch so với baseline
```

#### Nhóm EC2/Instance Compromise (Xâm Phạm EC2)

```
CryptoCurrency:EC2/BitcoinTool.B!DNS
    ↳ EC2 kết nối đến domain liên quan đến cryptomining qua DNS

Backdoor:EC2/C&CActivity.B!DNS
    ↳ EC2 giao tiếp với C2 server (Command & Control) đã biết

Trojan:EC2/BlackholeTraffic
    ↳ EC2 nhận traffic từ domain trong "blackhole" threat list

Behavior:EC2/NetworkPortUnusual
    ↳ EC2 lắng nghe trên port bất thường so với lịch sử

UnauthorizedAccess:EC2/TorIPCaller
    ↳ Truy cập SSH/RDP từ Tor exit node

Recon:EC2/PortProbeUnprotectedPort
    ↳ Port scanning nhắm vào port không được bảo vệ (0.0.0.0/0)
```

#### Nhóm S3 (Khi bật S3 Protection)

```
Exfiltration:S3/ObjectRead.Unusual
    ↳ Download lượng lớn S3 objects bất thường

UnauthorizedAccess:S3/MaliciousIPCaller
    ↳ Truy cập S3 từ IP độc hại trong threat list

Policy:S3/BucketBlockPublicAccessDisabled
    ↳ Tắt Block Public Access trên S3 bucket

Stealth:S3/ServerAccessLoggingDisabled
    ↳ Tắt server access logging trên S3 bucket
```

#### Nhóm EKS (Kubernetes)

```
PrivilegeEscalation:Kubernetes/PrivilegedContainer
    ↳ Container chạy với privileged mode (chế độ đặc quyền)

Persistence:Kubernetes/ContainerWithSensitiveMount
    ↳ Container mount /etc hoặc /root từ host

Discovery:Kubernetes/MaliciousIPCaller
    ↳ Kubernetes API calls từ IP độc hại

Impact:Kubernetes/SuccessfulAnonymousAccess
    ↳ Truy cập ẩn danh thành công vào Kubernetes API
```

---

## Severity Levels — Mức Độ Nghiêm Trọng

| Mức Độ | Score | Màu | Ý Nghĩa | Hành Động |
|---|---|---|---|---|
| **Critical** | 9.0–10.0 | 🔴 Đỏ đậm | Xâm phạm đang xảy ra, thiệt hại ngay lập tức | Phản hồi ngay (<15 phút) |
| **High** | 7.0–8.9 | 🔴 Đỏ | Dấu hiệu rõ ràng của compromise hoặc tấn công | Điều tra trong 1 giờ |
| **Medium** | 4.0–6.9 | 🟡 Vàng | Hành vi đáng ngờ, có thể là false positive | Điều tra trong 24 giờ |
| **Low** | 1.0–3.9 | 🟢 Xanh | Thông tin bổ sung, rủi ro thấp | Review trong tuần |

### Nguyên Tắc Ưu Tiên

```
High + EC2 instance đang chạy production → NGAY LẬP TỨC isolate
High + IAM user/role bị ảnh hưởng → Revoke credentials ngay
Medium + xảy ra nhiều lần (volume) → Điều tra ngay
Medium + isolated test account → Có thể defer
Low → Review batch cuối ngày hoặc cuối tuần
```

---

## Suppression Rules — Quy Tắc Loại Trừ

**Suppression Rules** (Quy Tắc Loại Trừ) cho phép tự động archive (lưu trữ) các findings đã biết là false positive (dương tính giả) hoặc không liên quan.

### Khi Nào Dùng Suppression

- Vulnerability scanner của bạn gây ra findings `Recon:EC2/PortProbeUnprotectedPort`
- Pentest engagement tạo ra `UnauthorizedAccess:IAMUser/MaliciousIPCaller` từ IP đã biết
- Bastion host hợp lệ kết nối từ IP office cố định
- NAT Gateway tạo ra traffic hợp lệ bị nhận dạng nhầm

### Cú Pháp Suppression Rule

```json
{
  "FindingCriteria": {
    "Criterion": {
      "type": {
        "Equals": ["Recon:EC2/PortProbeUnprotectedPort"]
      },
      "resource.instanceDetails.tags.key": {
        "Equals": ["Environment"]
      },
      "resource.instanceDetails.tags.value": {
        "Equals": ["security-scanner"]
      }
    }
  },
  "Name": "SuppressSecurityScannerFindings",
  "Action": "ARCHIVE"
}
```

```bash
# Tạo suppression rule qua CLI
aws guardduty create-filter \
  --detector-id <DETECTOR_ID> \
  --name "SuppressKnownScannerIPs" \
  --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "type": {"Equals": ["Recon:EC2/PortProbeUnprotectedPort"]},
      "service.action.networkConnectionAction.remoteIpDetails.ipAddressV4": {
        "Equals": ["203.0.113.10", "203.0.113.11"]
      }
    }
  }'
```

### Trusted IP Lists vs Threat IP Lists

```bash
# Trusted IP List — IP nội bộ, không sinh findings
aws guardduty create-ip-set \
  --detector-id <DETECTOR_ID> \
  --name "InternalNetworks" \
  --format TXT \
  --location s3://my-guardduty-lists/trusted-ips.txt \
  --activate

# Threat IP List — Thêm IP bổ sung vào danh sách nguy hiểm
aws guardduty create-threat-intel-set \
  --detector-id <DETECTOR_ID> \
  --name "CustomThreatFeeds" \
  --format TXT \
  --location s3://my-guardduty-lists/custom-threats.txt \
  --activate
```

---

## Multi-Account Setup — Thiết Lập Đa Tài Khoản

### Mô Hình Delegated Administrator (Quản Trị Ủy Quyền)

```
Management Account (Tài Khoản Quản Lý)
    └── Ủy quyền GuardDuty cho Security Account

Security Account (Tài Khoản Bảo Mật) — Delegated Admin
    ├── Nhận tất cả findings từ member accounts
    ├── Bật/tắt GuardDuty cho member accounts
    ├── Cấu hình suppression rules tập trung
    └── Export findings sang S3 / Security Hub

Member Accounts (Tài Khoản Thành Viên)
    ├── Dev Account — findings tự động gửi lên Admin
    ├── Staging Account — findings tự động gửi lên Admin
    └── Production Account — findings tự động gửi lên Admin
```

```bash
# Bước 1: Từ Management Account — ủy quyền Security Account
aws guardduty enable-organization-admin-account \
  --admin-account-id 111122223333

# Bước 2: Từ Security Account — bật auto-enable cho members mới
aws guardduty update-organization-configuration \
  --detector-id <DETECTOR_ID> \
  --auto-enable NEW \
  --features '[
    {"Name": "S3_DATA_EVENTS", "AutoEnable": "NEW"},
    {"Name": "EKS_AUDIT_LOGS", "AutoEnable": "NEW"},
    {"Name": "RDS_LOGIN_EVENTS", "AutoEnable": "NEW"}
  ]'

# Bước 3: Thêm member accounts hiện có
aws guardduty create-members \
  --detector-id <DETECTOR_ID> \
  --account-details '[
    {"AccountId": "444455556666", "Email": "dev@company.com"},
    {"AccountId": "777788889999", "Email": "prod@company.com"}
  ]'
```

---

## Tự Động Phản Hồi — Automated Response

### EventBridge Integration

```json
// EventBridge rule để bắt GuardDuty findings
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7] }]
  }
}
```

### Lambda Auto-Remediation (Tự Động Khắc Phục)

```python
import boto3
import json

def lambda_handler(event, context):
    finding = event['detail']
    severity = finding['severity']
    finding_type = finding['type']
    
    # Xử lý EC2 bị compromise
    if finding_type.startswith('UnauthorizedAccess:EC2') and severity >= 7:
        instance_id = finding['resource']['instanceDetails']['instanceId']
        isolate_ec2_instance(instance_id)
        create_forensic_snapshot(instance_id)
        notify_security_team(finding)
    
    # Xử lý IAM credential bị compromise
    elif 'IAMUser' in finding_type and severity >= 7:
        username = finding['resource']['accessKeyDetails']['userName']
        disable_iam_access_keys(username)
        notify_security_team(finding)

def isolate_ec2_instance(instance_id):
    ec2 = boto3.client('ec2')
    # Gán isolation security group (chặn tất cả traffic)
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=['sg-isolation-0123456789']  # SG không có inbound/outbound rules
    )

def disable_iam_access_keys(username):
    iam = boto3.client('iam')
    paginator = iam.get_paginator('list_access_keys')
    for page in paginator.paginate(UserName=username):
        for key in page['AccessKeyMetadata']:
            iam.update_access_key(
                UserName=username,
                AccessKeyId=key['AccessKeyId'],
                Status='Inactive'
            )
```

### Luồng Phản Hồi Đầy Đủ

```
GuardDuty Finding (High Severity)
    │
    ▼
EventBridge Rule (bắt finding)
    │
    ├── SNS Topic → Email/SMS cảnh báo team
    │
    ├── SQS Queue → Ticket system (Jira, PagerDuty)
    │
    └── Lambda Function
            ├── Isolate EC2 (thay Security Group)
            ├── Revoke IAM credentials
            ├── Create EBS snapshot (forensic evidence)
            ├── Tag resource: {"SecurityStatus": "Compromised"}
            └── Create Systems Manager OpsItem (ticket vận hành)
```

---

## Pricing — Chi Phí

| Nguồn Dữ Liệu | Đơn Giá |
|---|---|
| CloudTrail Management Events | $4.00 / 1M events |
| CloudTrail S3 Data Events | $0.80 / 1M events |
| VPC Flow Logs & DNS Logs | $1.00 / GB |
| S3 Protection | $0.20–$0.80 / GB |
| EKS Audit Logs | $0.60 / vCPU-hour |
| Malware Protection | Theo vCPU của instance |
| RDS Protection | Theo instance |

### Tối Ưu Chi Phí

```bash
# Xem chi phí ước tính trước khi bật
aws guardduty get-usage-statistics \
  --detector-id <DETECTOR_ID> \
  --usage-statistic-type SUM_BY_DATA_SOURCE \
  --usage-criteria '{"DataSources": ["CLOUD_TRAIL","DNS_LOGS","FLOW_LOGS","S3_LOGS"]}'

# Dùng 30-day free trial để ước tính
# GuardDuty cung cấp 30 ngày miễn phí cho mỗi account mới
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: GuardDuty có cần cài CloudTrail không?**
A: GuardDuty sử dụng CloudTrail events nhưng **không yêu cầu** bạn phải tạo Trail trước. GuardDuty tự phân tích events qua API riêng của nó. Tuy nhiên, bật CloudTrail vẫn cần thiết để lưu trữ và audit lâu dài.

**Q: Tắt GuardDuty có xóa findings không?**
A: Findings được lưu 90 ngày. Tắt GuardDuty (disable detector) không xóa ngay — findings vẫn truy vấn được trong 90 ngày. Xóa detector mới xóa hoàn toàn.

**Q: GuardDuty khác IDS (Intrusion Detection System — Hệ Thống Phát Hiện Xâm Nhập) truyền thống như thế nào?**
A: IDS truyền thống phân tích packet-level (nội dung gói tin), cần agent, dễ bị bypass. GuardDuty phân tích metadata (CloudTrail API calls, VPC Flow Logs metadata), không cần agent, dùng ML để phát hiện anomaly (bất thường) — khó bypass hơn vì tấn công để lại dấu vết trong metadata dù có mã hóa.

**Q: EC2 bị GuardDuty báo cryptomining. Làm gì?**
A:
1. **Isolate** — Thay Security Group thành isolation SG (chặn outbound)
2. **Snapshot** — Tạo EBS snapshot để forensic
3. **Investigate** — Dùng Detective để phân tích timeline, xem EC2 kết nối đến domain nào
4. **Remediate** — Launch instance mới từ AMI sạch
5. **Root cause** — Xem CloudTrail để tìm cách attacker xâm nhập (IAM key lộ? SSH brute force?)
6. **Document** — Viết incident report

**Q: Suppression Rules ảnh hưởng đến việc tính phí không?**
A: Không. Suppression rules chỉ archive findings sau khi đã tạo. GuardDuty vẫn phân tích dữ liệu và tính phí như bình thường. Suppression chỉ giúp giảm alert noise (nhiễu cảnh báo).

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
