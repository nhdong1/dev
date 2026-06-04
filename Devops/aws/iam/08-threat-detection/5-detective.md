# Amazon Detective — Điều Tra Sự Cố Bảo Mật Bằng Đồ Thị Quan Hệ

> Amazon Detective là dịch vụ phân tích và điều tra bảo mật dùng graph database (cơ sở dữ liệu đồ thị) để tự động xây dựng mô hình hành vi tài nguyên AWS theo thời gian, giúp điều tra nguyên nhân gốc rễ (root cause analysis) và xác định phạm vi ảnh hưởng (blast radius) của sự cố bảo mật.

## 📚 Mục Lục

1. [Tại Sao Cần Detective?](#tại-sao-cần-detective)
2. [Kiến Trúc Graph Model](#kiến-trúc-graph-model)
3. [Nguồn Dữ Liệu](#nguồn-dữ-liệu)
4. [Behavior Graph — Đồ Thị Hành Vi](#behavior-graph)
5. [Investigation Workflow — Quy Trình Điều Tra](#investigation-workflow)
6. [Entity Types — Loại Thực Thể](#entity-types)
7. [Tích Hợp Với GuardDuty & Security Hub](#tích-hợp-với-guardduty--security-hub)
8. [Multi-Account Setup](#multi-account-setup)
9. [Chi Phí](#chi-phí)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Detective?

### Vấn Đề Khi Không Có Detective

```
GuardDuty báo: "EC2 instance i-0abc communicating with known C2 server"
    │
    Câu hỏi cần trả lời:
    ├── Instance này bị compromise lúc nào?
    ├── Attacker vào bằng cách nào? (IAM key? SSH? Container escape?)
    ├── Instance đã kết nối với những IP/domain nào khác?
    ├── Có lateral movement (di chuyển ngang) sang resources khác không?
    ├── Những IAM credentials nào đã được dùng trên instance này?
    └── Blast radius là gì? (Bao nhiêu tài nguyên bị ảnh hưởng?)
    
    Không có Detective:
    → Phải query CloudTrail thủ công (Athena), 
    → Phân tích VPC Flow Logs thủ công,
    → Tự correlate (liên kết) dữ liệu từ nhiều nguồn,
    → Mất nhiều giờ hoặc nhiều ngày.
    
    Có Detective:
    → Click từ GuardDuty finding → mở Detective,
    → Xem timeline, network connections, API calls trực quan,
    → Phân tích xong trong 15-30 phút.
```

---

## Kiến Trúc Graph Model

Detective xây dựng **behavior graph** (đồ thị hành vi) — một graph database lưu trữ quan hệ giữa các entities và sự kiện theo thời gian.

```
Behavior Graph Structure (Cấu Trúc Đồ Thị Hành Vi)

Nodes (Nút)                    Edges (Cạnh)
─────────────────────────────────────────────────────────
AwsAccount ─────────── OWNS ──────────────→ AwsRole
                                            │
AwsAccount ─────────── OWNS ──────────────→ Ec2Instance
                                            │
Ec2Instance ─────────── USES ─────────────→ AwsRole
                                            │
Ec2Instance ─────────── MADE_API_CALL ────→ ApiCall
                                            │
Ec2Instance ─────────── CONNECTED_TO ─────→ IpAddress
                                            │
IpAddress ─────────── FROM_GEO ───────────→ GeoLocation
                                            │
IpAddress ─────────── IN_LIST ────────────→ ThreatIntelFeed
```

### So Sánh Với Phân Tích Truyền Thống

| Khía Cạnh | Thủ Công (Athena + CloudTrail) | Detective |
|---|---|---|
| **Thời gian setup** | Phải tạo Athena tables, write queries | Sẵn sàng ngay sau 24h |
| **Baseline** | Phải tự tính toán | Tự động 12 tháng lịch sử |
| **Correlation** | Phải JOIN nhiều bảng thủ công | Tự động theo graph |
| **Visualization** | Phải tự build với QuickSight | Built-in charts |
| **Context** | Phải tra lookup bên ngoài | Tích hợp sẵn GuardDuty context |
| **Thời gian điều tra** | 4–8 giờ | 15–30 phút |

---

## Nguồn Dữ Liệu

Detective tự động ingestion (nhập dữ liệu) từ:

| Nguồn | Dữ Liệu | Mục Đích |
|---|---|---|
| **CloudTrail** | API calls, user activity | Who did what, when, from where |
| **VPC Flow Logs** | Network traffic metadata | Network connections, data transfer |
| **GuardDuty Findings** | Threat findings | Correlate findings với behavior |
| **EKS Audit Logs** | Kubernetes API calls | Container workload investigation |
| **AWS Organizations** | Account structure | Multi-account context |

**Thời gian lưu trữ:** 12 tháng (fixed, không cấu hình được)

**Yêu cầu tiên quyết:**
- GuardDuty phải được bật **ít nhất 48 giờ** trước khi bật Detective
- VPC Flow Logs phải bật cho các VPCs cần phân tích
- Detective cần 24 giờ để xây dựng behavior graph ban đầu

---

## Behavior Graph — Đồ Thị Hành Vi

### Baseline Building (Xây Dựng Đường Cơ Sở)

Detective học behavior "bình thường" của từng entity trong 2 tuần đầu. Sau đó dùng baseline này để highlight (làm nổi bật) hành vi bất thường.

```
Ví dụ baseline cho IAM Role "ProductionEC2Role":
    ├── Thường gọi: s3:GetObject, dynamodb:Query, cloudwatch:PutMetricData
    ├── Từ IP internal: 10.0.0.0/8
    ├── Giờ hoạt động: 06:00-22:00 UTC
    └── Region: us-east-1, us-west-2

Hành vi bất thường (Detective sẽ highlight):
    ├── Gọi: iam:CreateUser (chưa từng làm)
    ├── Từ IP: 185.220.x.x (Tor exit node)
    ├── Lúc 03:00 UTC
    └── Region: ap-east-1 (chưa từng dùng)
```

### Entity Profile Pages (Trang Hồ Sơ Thực Thể)

Mỗi entity trong Detective có **profile page** với:

1. **Overview Panel** — tóm tắt hoạt động gần đây
2. **Timeline** — sự kiện theo thứ tự thời gian
3. **Network Activity** — IP connections, data volume
4. **API Calls** — phân tích API calls theo tần suất và loại
5. **Related Findings** — GuardDuty findings liên quan
6. **Connected Entities** — các entities khác có liên quan

---

## Investigation Workflow — Quy Trình Điều Tra

### Bước 1: Bắt Đầu Từ GuardDuty Finding

```
GuardDuty Finding: "UnauthorizedAccess:EC2/SSHBruteForce"
    │
    └── Click "Investigate with Detective"
            │
            ▼
    Detective Investigation Dashboard
    ├── Finding details
    ├── Involved resources
    └── "Go to entity profile" links
```

### Bước 2: Phân Tích Entity Profile

```python
# Detective Investigation API — lấy thông tin điều tra
import boto3

detective = boto3.client('detective')

# Lấy danh sách investigations
investigations = detective.list_investigations(
    GraphArn='arn:aws:detective:us-east-1:123456789012:graph:abc123',
    FilterCriteria={
        'Status': {'Value': 'RUNNING'},
        'Severity': {'Value': 'HIGH'}
    }
)

# Tạo investigation mới cho một entity
investigation = detective.start_investigation(
    GraphArn='arn:aws:detective:us-east-1:123456789012:graph:abc123',
    EntityArn='arn:aws:iam::123456789012:user/suspicious-user',
    ScopeStartTime='2026-05-15T00:00:00Z',
    ScopeEndTime='2026-05-16T23:59:59Z'
)

# Lấy indicators (dấu hiệu) của investigation
indicators = detective.get_investigation(
    GraphArn='arn:aws:detective:us-east-1:123456789012:graph:abc123',
    InvestigationId=investigation['InvestigationId']
)
```

### Bước 3: Xác Định Lateral Movement (Di Chuyển Ngang)

```
Câu hỏi: IAM role bị compromised đã truy cập những gì?

Detective Timeline Analysis:
    13:00 UTC — GuardDuty finding: IP độc hại gọi AssumeRole
    13:02 UTC — sts:AssumeRole thành công cho "DeploymentRole"
    13:05 UTC — s3:ListBuckets (tất cả buckets)
    13:07 UTC — s3:GetObject từ bucket "prod-customer-data" (200 objects)
    13:12 UTC — iam:ListUsers, iam:ListRoles
    13:15 UTC — iam:CreateAccessKey cho user "backup-admin"
    13:18 UTC — CloudTrail: PutEventSelectors (thay đổi logging)
    
Blast Radius Analysis:
    ├── Dữ liệu bị exfiltrate: prod-customer-data bucket (ước tính X GB)
    ├── IAM backdoor: access key mới cho backup-admin
    ├── Log tampering: CloudTrail config thay đổi
    └── Thời gian tổng: 18 phút từ initial access đến post-exploitation
```

### Bước 4: Summarize & Document

```bash
# Cập nhật investigation status
aws detective update-investigation-state \
  --graph-arn arn:aws:detective:... \
  --investigation-id abc123 \
  --state ARCHIVE \
  # Options: ACTIVE | ARCHIVED

# Tạo investigation report (export dữ liệu)
# → Dùng Detective console để export findings summary
```

---

## Entity Types — Loại Thực Thể

| Entity Type | Ký Hiệu AWS | Ví Dụ |
|---|---|---|
| **AWS Account** | `AwsAccount` | `123456789012` |
| **IAM Role** | `AwsRole` | `arn:aws:iam::....:role/MyRole` |
| **IAM User** | `AwsUser` | `arn:aws:iam::....:user/john` |
| **EC2 Instance** | `Ec2Instance` | `i-0123456789abcdef0` |
| **IP Address** | `IpAddress` | `203.0.113.100` |
| **Federated User** | `FederatedUser` | SAML federated session |
| **EKS Cluster** | `EksCluster` | `my-prod-cluster` |
| **Container** | `Container` | Container ID trong EKS |
| **Lambda Function** | `LambdaFunction` | `arn:aws:lambda:...` |

### Scope Time (Phạm Vi Thời Gian)

Khi điều tra, có thể chỉ định **Scope Time** — khoảng thời gian cụ thể cần phân tích (tối đa 48 giờ trong khoảng 12 tháng gần nhất).

```
Ví dụ Scope Time cho investigation:
    Start: 2026-05-15T12:00:00Z  (1 giờ trước finding)
    End:   2026-05-15T15:00:00Z  (1 giờ sau finding)
    
    → Detective hiển thị tất cả activity của entity trong khoảng này
    → So sánh với baseline để highlight anomalies
```

---

## Tích Hợp Với GuardDuty & Security Hub

### GuardDuty → Detective (Direct Integration)

```
Trong GuardDuty Console:
Finding → Actions → "Investigate in Amazon Detective"
    │
    ▼
Detective mở investigation với:
    ├── Pre-populated scope time (±1 giờ quanh finding)
    ├── Involved entities đã identified
    └── Related findings từ cùng entities
```

### Security Hub → Detective

```bash
# Security Hub Custom Action để mở Detective
aws securityhub create-action-target \
  --name "InvestigateWithDetective" \
  --description "Open Detective investigation for selected finding" \
  --id "OpenDetective"

# EventBridge rule bắt custom action
# → Lambda gọi Detective API để tạo investigation
# → Trả về investigation URL vào Slack/JIRA
```

---

## Multi-Account Setup — Thiết Lập Đa Tài Khoản

```
Security Account (Administrator Account)
    ├── Owns Behavior Graph
    ├── Ingests data từ all member accounts
    └── Investigators thực hiện analysis từ đây

Member Accounts
    ├── Dev Account → data feed vào behavior graph
    ├── Staging Account → data feed vào behavior graph
    └── Production Account → data feed vào behavior graph
```

```bash
# Từ Security Account: tạo behavior graph
aws detective create-graph \
  --tags '{"Name": "CompanyBehaviorGraph"}'

# Invite member accounts
aws detective create-members \
  --graph-arn arn:aws:detective:us-east-1:111122223333:graph:abc123 \
  --accounts '[
    {"AccountId": "444455556666", "EmailAddress": "dev@company.com"},
    {"AccountId": "777788889999", "EmailAddress": "prod@company.com"}
  ]'

# Từ Organization management account: auto-enable
aws detective enable-organization-admin-account \
  --account-id 111122223333

# Từ Security Account: auto-enable members
aws detective update-organization-configuration \
  --graph-arn arn:aws:detective:... \
  --auto-enable
```

---

## Chi Phí

| Thành Phần | Giá |
|---|---|
| CloudTrail data ingestion | $2.00/GB ingested |
| VPC Flow Logs data ingestion | $1.00/GB ingested |
| GuardDuty Findings ingestion | Miễn phí |

### Lưu Ý Chi Phí

- **30-day free trial** cho account mới
- Chi phí phụ thuộc vào volume log — account có nhiều EC2, VPC, API calls sẽ tốn nhiều hơn
- Dùng Organization setup để chia sẻ một behavior graph thay vì tạo nhiều graphs
- Xem ước tính chi phí trong Detective console trước khi enable

---

## Câu Hỏi Phỏng Vấn

**Q: Detective khác CloudTrail như thế nào?**
A: CloudTrail là hệ thống **ghi nhật ký** — lưu raw API call events. Detective là hệ thống **phân tích** — lấy dữ liệu từ CloudTrail (và VPC Flow Logs, GuardDuty), xây dựng graph database, xác định baseline, và cung cấp interface để điều tra. CloudTrail cho bạn data thô; Detective cho bạn context và correlation.

**Q: Khi nào dùng Detective thay vì query Athena trực tiếp?**
A: Detective tốt hơn khi: (1) cần điều tra nhanh trong incident response, (2) cần correlation tự động giữa nhiều entities, (3) cần so sánh với baseline behavior. Athena tốt hơn khi: (1) cần query rất cụ thể với custom filter phức tạp, (2) cần export large datasets để phân tích offline, (3) cần lưu trữ log hơn 12 tháng.

**Q: Tại sao phải bật GuardDuty trước khi bật Detective?**
A: Detective cần GuardDuty để: (1) sử dụng GuardDuty findings làm điểm bắt đầu điều tra, (2) GuardDuty cung cấp context về threat intelligence. Kỹ thuật là detective cần GuardDuty detector ID để associate với behavior graph. Ngoài ra, 48 giờ là thời gian GuardDuty cần để thiết lập initial baseline trước khi Detective có thể bắt đầu ingestion.

**Q: Blast radius analysis trong Detective là gì?**
A: Blast radius (Phạm Vi Ảnh Hưởng) là tập hợp tất cả tài nguyên bị ảnh hưởng nếu một entity bị compromise. Detective giúp xác định blast radius bằng cách: (1) hiển thị tất cả tài nguyên entity đã truy cập, (2) tìm lateral movement (entity A truy cập entity B, B truy cập C...), (3) hiển thị data volume đã được transfer. Từ đó biết cần isolate/remediate những gì.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
