# Top 25 Câu Hỏi Phỏng Vấn AWS Security

> Tổng hợp 25 câu hỏi xuất hiện thường xuyên nhất trong phỏng vấn AWS Security Engineer, Cloud Architect, và DevSecOps — kèm đáp án chi tiết và tip trình bày.

---

## 📑 Mục Lục

- [Nhóm 1: IAM & Identity (Q1–Q8)](#nhóm-1-iam--identity)
- [Nhóm 2: Encryption & Secrets (Q9–Q13)](#nhóm-2-encryption--secrets)
- [Nhóm 3: Threat Detection & Monitoring (Q14–Q18)](#nhóm-3-threat-detection--monitoring)
- [Nhóm 4: Architecture & Multi-Account (Q19–Q22)](#nhóm-4-architecture--multi-account)
- [Nhóm 5: Compliance & Advanced (Q23–Q25)](#nhóm-5-compliance--advanced)

---

## Nhóm 1: IAM & Identity

### Q1. Giải thích thứ tự đánh giá IAM policy. Khi nào "Access Denied" xảy ra?

**Đáp án:**

AWS đánh giá quyền theo thứ tự sau (tất cả áp dụng cùng lúc, explicit deny thắng tất cả):

```
1. Explicit Deny (Từ Chối Tường Minh)
   → Nếu có bất kỳ policy nào từ chối tường minh → DENY ngay lập tức

2. SCP (Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ)
   → Nếu SCP không cho phép action → DENY

3. Resource-based Policy (Chính Sách Tài Nguyên)
   → Nếu có → ALLOW (trừ phi bị explicit deny)

4. Permission Boundary (Ranh Giới Quyền Hạn)
   → Nếu action không nằm trong boundary → DENY

5. Identity-based Policy (Chính Sách Định Danh)
   → Nếu không có allow → DENY

6. Session Policy (Chính Sách Phiên — khi dùng AssumeRole với policy)
   → Giới hạn thêm quyền trong session
```

**Nguyên tắc cốt lõi:**
- Mặc định: **Implicit Deny** (Từ Chối Ngầm Định) — không có allow = denied
- **Explicit Deny** thắng mọi Allow dù từ nguồn nào

**Ví dụ thực tế:**
```json
// SCP trong Organizations — chặn xóa CloudTrail
{
  "Effect": "Deny",
  "Action": "cloudtrail:DeleteTrail",
  "Resource": "*"
}
// → Dù IAM user có AdministratorAccess, vẫn không xóa được CloudTrail
```

**Tip phỏng vấn:** Vẽ sơ đồ luồng khi giải thích — phỏng vấn viên đánh giá cao tư duy có hệ thống.

---

### Q2. Phân biệt IAM Role và IAM User. Khi nào dùng Role thay vì User?

**Đáp án:**

| Tiêu Chí | IAM User (Người Dùng IAM) | IAM Role (Vai Trò IAM) |
|---|---|---|
| Định danh | Lâu dài, gắn với 1 người/ứng dụng cụ thể | Tạm thời, bất kỳ entity nào có thể assume |
| Credentials | Access key tĩnh (rủi ro bị lộ) | Temporary credentials (tự hết hạn) |
| Use case | Admin cá nhân, programmatic access cũ | EC2, Lambda, cross-account, federation |
| Best practice | Dùng tối thiểu, ưu tiên Role | **Preferred method** cho mọi workload |

**Dùng Role khi:**
- EC2 instance cần gọi S3, DynamoDB → **Instance Profile** (Hồ Sơ Instance)
- Lambda cần truy cập DynamoDB → **Execution Role** (Role Thực Thi)
- Cross-account access → **AssumeRole** (Giả Định Vai Trò)
- CI/CD pipeline → **OIDC federation** (không cần key tĩnh)
- SSO với Azure AD/Okta → **SAML federation**

**Dùng User khi:**
- Người dùng cần đăng nhập Console thủ công (tạm thời — nên migrate sang IAM Identity Center)
- Legacy application không hỗ trợ role

**Tip phỏng vấn:** "Best practice hiện tại là không tạo access key cho IAM User trong production — dùng IAM Role với temporary credentials hoặc OIDC federation cho CI/CD."

---

### Q3. Trust Policy và Permission Policy khác nhau như thế nào?

**Đáp án:**

**Trust Policy** (Chính Sách Tin Tưởng):
- Đính kèm vào **Role**, định nghĩa **ai được phép assume role** (Principal — Chủ Thể)
- Trả lời câu hỏi: "Ai được quyền trở thành role này?"

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "lambda.amazonaws.com"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "aws:SourceAccount": "123456789012"
      }
    }
  }]
}
```

**Permission Policy** (Chính Sách Quyền Hạn):
- Đính kèm vào **User/Group/Role**, định nghĩa **được làm gì với tài nguyên nào**
- Trả lời câu hỏi: "Được phép thực hiện hành động gì?"

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Quy trình AssumeRole đầy đủ:**
1. Entity gọi `sts:AssumeRole`
2. AWS kiểm tra Trust Policy của role: Entity có được phép assume không?
3. AWS kiểm tra Permission Policy của entity: Có quyền gọi `sts:AssumeRole` không?
4. Nếu cả hai pass → trả về Temporary Security Credentials (Thông Tin Xác Thực Tạm Thời)

---

### Q4. SCPs và Permission Boundaries khác nhau như thế nào? Khi nào dùng cái nào?

**Đáp án:**

| Tiêu Chí | SCPs (Service Control Policies) | Permission Boundaries (Ranh Giới Quyền) |
|---|---|---|
| Áp dụng cho | Account hoặc OU trong Organizations | IAM User hoặc Role cụ thể |
| Điều khiển bởi | Management Account (tài khoản quản lý) | Admin của account đó |
| Scope | Toàn bộ account (kể cả root user) | Một identity cụ thể |
| Mục đích | Guardrail (Lan Can) đa tài khoản | Delegate admin an toàn |
| Override được không | Không (Management Account quyết định) | Admin có thể thay đổi |

**Dùng SCP khi:**
- Ngăn chặn toàn bộ account dùng một region (ví dụ: chỉ cho phép ap-southeast-1)
- Bảo vệ CloudTrail, GuardDuty không bị tắt bởi bất kỳ ai
- Enforce encryption bắt buộc cho mọi S3 bucket trong org

**Dùng Permission Boundary khi:**
- Cho phép developer tự tạo IAM Role nhưng không vượt qua giới hạn nhất định
- Delegate admin quyền tạo user nhưng không thể leo thang đặc quyền (privilege escalation)

**Ví dụ kết hợp:**
```
Management Account → SCP: "Chỉ cho phép us-east-1 và us-west-2"
    └── Dev Account → Permission Boundary: "Role tự tạo không được có AdministratorAccess"
        └── Developer → tự tạo Lambda roles trong boundary đó
```

---

### Q5. Cross-account role hoạt động như thế nào? Giải thích quy trình đầy đủ.

**Đáp án:**

**Kịch bản:** Account A (123456789012) muốn truy cập S3 trong Account B (987654321098)

**Bước 1: Account B tạo Role với Trust Policy**
```json
// Role: arn:aws:iam::987654321098:role/CrossAccountS3Reader
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:role/AppRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id-12345"
      }
    }
  }]
}
```

**Bước 2: Account B gắn Permission Policy vào Role**
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::account-b-bucket/*"
  }]
}
```

**Bước 3: Account A gọi AssumeRole**
```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::987654321098:role/CrossAccountS3Reader" \
  --role-session-name "MySession" \
  --external-id "unique-external-id-12345"
```

**Bước 4: STS trả về Temporary Credentials**
```json
{
  "Credentials": {
    "AccessKeyId": "ASIA...",
    "SecretAccessKey": "...",
    "SessionToken": "...",
    "Expiration": "2026-05-16T10:00:00Z"
  }
}
```

**ExternalId** (ID Bên Ngoài): Bảo vệ chống **Confused Deputy Attack** (Tấn Công Phó Nhầm Lẫn) — ngăn bên thứ ba giả mạo lời gọi AssumeRole.

---

### Q6. Cách bảo mật root account AWS?

**Đáp án — Checklist bắt buộc:**

1. **Bật MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố) cho root** — dùng hardware token (YubiKey) hoặc virtual MFA app
2. **Không tạo access key cho root** — nếu có, xóa ngay lập tức
3. **Không dùng root cho công việc hàng ngày** — tạo IAM user/role riêng với quyền tối thiểu
4. **Đặt billing alerts** — cảnh báo sớm chi phí bất thường (có thể do compromise)
5. **Khóa alternate contacts** — cập nhật thông tin liên lạc bảo mật, billing, operations
6. **Sử dụng Organizations** — quản lý từ Management Account, giới hạn root action bằng SCP
7. **Monitor root activity** — CloudTrail alert khi root account được sử dụng

```json
// CloudWatch alarm cho root login
{
  "MetricName": "RootAccountUsage",
  "FilterPattern": "{ $.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\" }"
}
```

---

### Q7. Giải thích ABAC (Attribute-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Thuộc Tính). Ưu điểm so với RBAC?

**Đáp án:**

**RBAC** (Role-Based Access Control — Kiểm Soát Truy Cập Theo Vai Trò): Gán quyền dựa trên vai trò cố định.

**ABAC**: Gán quyền dựa trên **tags** (nhãn) của cả principal và resource.

**Ví dụ ABAC:**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      // Principal tag phải match Resource tag
      "aws:PrincipalTag/Department": "${s3:ExistingObjectTag/Department}",
      "aws:PrincipalTag/Project": "${s3:ExistingObjectTag/Project}"
    }
  }
}
```

**Ưu điểm ABAC so với RBAC:**

| Tiêu Chí | RBAC | ABAC |
|---|---|---|
| Số policy cần tạo | Tăng theo số role | **1 policy dùng cho mọi team** |
| Thêm resource mới | Phải cập nhật policy | Chỉ cần tag đúng → tự động được phép |
| Scale | Kém với nhiều team/project | **Tốt ở scale lớn** |
| Độ phức tạp | Đơn giản | Phức tạp hơn (phải quản lý tags) |

**Khi dùng ABAC:** Tổ chức lớn, nhiều team, nhiều project với resource tăng liên tục.

---

### Q8. IAM Access Analyzer (Bộ Phân Tích Truy Cập IAM) là gì? Giải quyết vấn đề gì?

**Đáp án:**

IAM Access Analyzer phát hiện các resource được chia sẻ ra ngoài **zone of trust** (vùng tin tưởng — thường là account hoặc organization của bạn).

**Resource được kiểm tra:**
- S3 buckets, S3 Access Points
- KMS keys
- IAM roles (trust policy cho phép external principal)
- Lambda functions và layers
- SQS queues
- Secrets Manager secrets

**Finding types:**
- **External Access** (Truy Cập Bên Ngoài): Resource accessible từ bên ngoài organization
- **Unused Access** (Truy Cập Không Dùng): IAM user/role có quyền không dùng trong 90 ngày

**Ví dụ tích hợp vào CI/CD:**
```bash
# Kiểm tra trước khi deploy Terraform
aws accessanalyzer validate-policy \
  --policy-document file://new-policy.json \
  --policy-type IDENTITY_POLICY
```

**Tip phỏng vấn:** Access Analyzer dùng **formal verification** (xác minh hình thức), không chỉ pattern matching — có thể phát hiện logic phức tạp hơn regex.

---

## Nhóm 2: Encryption & Secrets

### Q9. Giải thích Envelope Encryption (Mã Hóa Phong Bì). Tại sao cần thiết?

**Đáp án:**

**Vấn đề:** KMS có giới hạn — chỉ mã hóa được dữ liệu ≤ 4 KB trực tiếp. File lớn hơn cần giải pháp khác.

**Envelope Encryption** giải quyết bằng 2 lớp khóa:

```
CMK (Customer Master Key — Khóa Chính)  [lưu trong KMS, không bao giờ rời KMS]
  └── DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu)  [tạo mỗi lần mã hóa]
        └── Plaintext Data (Dữ Liệu Thô) → Encrypted Data (Dữ Liệu Đã Mã Hóa)
```

**Quy trình mã hóa:**
1. Gọi KMS `GenerateDataKey` → nhận `PlaintextDEK` + `EncryptedDEK`
2. Dùng `PlaintextDEK` mã hóa data → `EncryptedData`
3. **Xóa** `PlaintextDEK` khỏi memory ngay lập tức
4. Lưu `EncryptedData` + `EncryptedDEK` cùng nhau

**Quy trình giải mã:**
1. Gọi KMS `Decrypt(EncryptedDEK)` → nhận `PlaintextDEK`
2. Dùng `PlaintextDEK` giải mã `EncryptedData`
3. Xóa `PlaintextDEK` khỏi memory

**Lợi ích:**
- Mã hóa data bất kể kích thước
- DEK riêng cho mỗi object → compromise 1 DEK không ảnh hưởng toàn bộ
- CMK không bao giờ rời KMS HSM (Hardware Security Module — Module Bảo Mật Phần Cứng)
- **Performance**: mã hóa/giải mã diễn ra locally, chỉ gọi KMS để xử lý DEK

---

### Q10. Khi nào dùng CloudHSM thay vì KMS?

**Đáp án:**

| Tiêu Chí | AWS KMS | AWS CloudHSM |
|---|---|---|
| Quản lý HSM | AWS (shared) | Bạn (dedicated) |
| FIPS 140-2 Level | Level 2 | **Level 3** |
| Key control | AWS có thể truy cập | **Chỉ bạn** |
| Performance | Standard | **Cao hơn** cho bulk encryption |
| Chi phí | $1/key/tháng | **$2/giờ/cluster** |
| Compliance | Hầu hết | PCI-HSM, eIDAS, banking regulations |

**Chọn CloudHSM khi:**
- Compliance yêu cầu FIPS 140-2 Level 3 (banking, HSM mandate)
- **Custom cryptographic algorithms** (thuật toán mã hóa tùy chỉnh) — KMS chỉ hỗ trợ AES, RSA, ECC
- Oracle TDE (Transparent Data Encryption) — yêu cầu HSM dedicated
- Regulatory requirement: HSM keys không được nằm trên infrastructure của third party
- High-volume TLS offloading (giảm tải TLS lưu lượng cao)

**Vẫn dùng KMS khi:**
- Tích hợp native với S3, EBS, RDS, Lambda
- FIPS 140-2 Level 2 đủ yêu cầu
- Team nhỏ, không muốn quản lý HSM cluster

---

### Q11. Secrets Manager vs Parameter Store — chọn cái nào?

**Đáp án:**

| Tiêu Chí | Secrets Manager | Parameter Store |
|---|---|---|
| Chi phí | $0.40/secret + $0.05/10k API calls | **Miễn phí** (Standard), $0.05/10k (Advanced) |
| Auto-rotation | ✅ Built-in (RDS, Redshift, DocumentDB, custom) | ❌ Cần Lambda riêng |
| Cross-account | ✅ Native | ✅ (Advanced tier) |
| Versioning | ✅ Automatic | ✅ Manual |
| Max size | 64 KB | 4 KB (Standard), 8 KB (Advanced) |
| Replication | ✅ Multi-region | ❌ |

**Chọn Secrets Manager khi:**
- Database credentials cần rotate tự động (RDS, Redshift)
- API keys của third-party service
- Credentials cần replica sang nhiều region
- Chi phí không phải ưu tiên hàng đầu

**Chọn Parameter Store khi:**
- Application config không cần rotation (feature flags, endpoints)
- Môi trường nhiều parameter, chi phí cần tối ưu
- SecureString đơn giản với KMS encryption đủ dùng
- Hierarchy cần linh hoạt (`/app/prod/database/url`)

**Mô hình kết hợp:**
```
Secrets Manager: DB_PASSWORD, API_SECRET, THIRD_PARTY_KEY
Parameter Store: DB_HOST, DB_PORT, FEATURE_FLAG_PAYMENT, LOG_LEVEL
```

---

### Q12. Giải thích Key Policy (Chính Sách Khóa) KMS. Khác IAM policy thế nào?

**Đáp án:**

**Key Policy** là resource-based policy (chính sách dựa trên tài nguyên), gắn trực tiếp vào KMS key.

**Khác biệt quan trọng:**
- IAM policy cho phép user/role dùng KMS **chỉ khi Key Policy cũng cho phép**
- Key Policy phải explicitly give the account access (khác S3 bucket policy)

**Ví dụ Key Policy:**
```json
{
  "Statement": [
    {
      // 1. Root statement — bắt buộc phải có
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      // 2. Cho phép admin quản lý key
      "Sid": "Allow key administrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/KMSAdminRole"
      },
      "Action": ["kms:Create*", "kms:Describe*", "kms:Enable*", "kms:Put*", "kms:ScheduleKeyDeletion"],
      "Resource": "*"
    },
    {
      // 3. Cho phép service encrypt/decrypt
      "Sid": "Allow use of the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/AppRole"
      },
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*"
    }
  ]
}
```

**Nếu thiếu root statement** → bị khóa vĩnh viễn khỏi key → phải liên hệ AWS Support.

---

### Q13. Cách implement database credential rotation không downtime?

**Đáp án — Single User vs Multi-User Rotation:**

**Single User Rotation** (Xoay Vòng Một Người Dùng):
```
Lambda → tạo password mới → update DB → update secret
Vấn đề: Khoảng thời gian ngắn khi password đang đổi có thể gây lỗi
```

**Multi-User Rotation** (Xoay Vòng Đa Người Dùng — Zero Downtime):
```
Duy trì 2 user: user_a (active), user_b (standby)

Bước 1: Cập nhật user_b với password mới
Bước 2: Update secret trỏ sang user_b credentials
Bước 3: user_b trở thành active, user_a thành standby
Bước 4: Lần rotate tiếp theo: đổi user_a, switch lại
```

**Implementation với Secrets Manager:**
```python
# Lambda rotation function
def rotate_secret(event, context):
    step = event['Step']
    secret_id = event['SecretId']
    token = event['ClientRequestToken']
    
    if step == 'createSecret':
        # Tạo password mới, lưu vào AWSPENDING version
        create_new_password(secret_id, token)
    elif step == 'setSecret':
        # Apply password mới vào database
        set_password_in_db(secret_id, token)
    elif step == 'testSecret':
        # Verify password mới hoạt động
        test_connection(secret_id, token)
    elif step == 'finishSecret':
        # Promote AWSPENDING → AWSCURRENT
        finalize_rotation(secret_id, token)
```

---

## Nhóm 3: Threat Detection & Monitoring

### Q14. GuardDuty phát hiện mối đe dọa bằng cách nào? Liệt kê các finding types chính.

**Đáp án:**

**Nguồn dữ liệu GuardDuty phân tích:**
- **VPC Flow Logs** (Nhật Ký Luồng VPC): phát hiện traffic bất thường, port scanning
- **CloudTrail Logs**: phát hiện API calls bất thường
- **DNS Logs**: phát hiện communication với C2 (Command & Control) server
- **EKS Audit Logs**: threat detection trong Kubernetes
- **S3 Data Events**: phát hiện data exfiltration (rò rỉ dữ liệu)
- **Lambda Network Activity**: phát hiện Lambda bị dùng cho crypto mining

**Finding types chính:**

| Category | Finding | Ý Nghĩa |
|---|---|---|
| **Backdoor** | `Backdoor:EC2/DenialOfService.Dns` | EC2 đang tạo DNS-based DDoS |
| **CryptoCurrency** | `CryptoCurrency:EC2/BitcoinTool.B` | Instance đang mine crypto |
| **Trojan** | `Trojan:EC2/BlackholeTraffic` | Traffic tới known C2 IPs |
| **UnauthorizedAccess** | `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` | Console login từ tor exit node |
| **Recon** | `Recon:EC2/PortProbeUnprotectedPort` | Port scanning vào instance |
| **Exfiltration** | `Exfiltration:S3/ObjectRead.Unusual` | Download S3 data bất thường |
| **Persistence** | `Persistence:IAMUser/NetworkPermissions` | Tạo security group rule mới |

**Severity levels:** Low (1.0–3.9) | Medium (4.0–6.9) | High (7.0–8.9) | Critical (9.0–10.0)

**Response quy trình:**
```
High/Critical Finding → EventBridge → Lambda → [Isolate EC2 | Block IP | Revoke credentials]
```

---

### Q15. Thiết kế SIEM (Security Information and Event Management) trên AWS.

**Đáp án — Kiến trúc phổ biến:**

```
Data Sources (Nguồn Dữ Liệu):
├── CloudTrail → S3 bucket (encrypted với KMS)
├── VPC Flow Logs → CloudWatch Logs
├── GuardDuty findings → EventBridge
├── Security Hub findings → EventBridge
└── Application logs → CloudWatch Logs

Ingestion Layer (Lớp Thu Thập):
├── Kinesis Data Firehose → real-time streaming
└── S3 → Batch ingestion

Processing & Storage (Xử Lý & Lưu Trữ):
├── CloudTrail Lake → SQL queries trực tiếp
├── OpenSearch Service → full-text search, dashboards
└── S3 + Athena → ad-hoc analysis, long-term retention

Visualization & Alerting (Trực Quan Hóa & Cảnh Báo):
├── OpenSearch Dashboards → security dashboard
├── CloudWatch Dashboards → operational metrics
├── SNS/PagerDuty → on-call notifications
└── Slack/Teams → team notifications
```

**Athena query ví dụ — tìm root API calls:**
```sql
SELECT eventTime, eventName, sourceIPAddress, userAgent
FROM cloudtrail_logs
WHERE userIdentity.type = 'Root'
  AND eventTime > current_timestamp - interval '24' hour
ORDER BY eventTime DESC;
```

---

### Q16. CloudTrail vs AWS Config — phân biệt mục đích sử dụng.

**Đáp án:**

| Tiêu Chí | CloudTrail | AWS Config |
|---|---|---|
| Ghi gì? | **Ai làm gì, khi nào** (API calls) | **Tài nguyên trông như thế nào** (configuration state) |
| Focus | Accountability (Trách Nhiệm Giải Trình) | Compliance (Tuân Thủ Cấu Hình) |
| Câu hỏi trả lời | "Ai xóa S3 bucket lúc 3am?" | "S3 bucket này có bật versioning không?" |
| Thời gian | Realtime event streaming | Periodic snapshot + change tracking |
| Query | CloudTrail Lake (SQL), Athena | Config Rules, Timeline, Aggregator |

**Khi dùng CloudTrail:**
- Forensics: điều tra xem ai thực hiện hành động gì
- Security audit: track privileged actions
- Compliance: chứng minh ai đã access data nhạy cảm

**Khi dùng AWS Config:**
- Phát hiện misconfiguration (cấu hình sai): S3 bucket public, SG mở port 22 ra internet
- Drift detection (phát hiện sai lệch): resource thay đổi so với trạng thái mong muốn
- Compliance reporting: tỷ lệ % resource tuân thủ CIS Benchmark

---

### Q17. Quy trình xử lý khi GuardDuty phát hiện EC2 instance bị compromise?

**Đáp án — Incident Response Playbook:**

```
BƯỚC 1: Containment (Ngăn Chặn Lây Lan) — trong vòng 15 phút
├── Isolate EC2: thay Security Group bằng quarantine-sg (chặn mọi traffic)
├── Revoke temporary credentials của instance profile
├── Snapshot EBS volume trước khi thay đổi gì
└── Ghi lại metadata: instance ID, IP, time, finding details

BƯỚC 2: Eradication (Loại Bỏ Nguy Cơ)
├── Thu thập VPC Flow Logs, CloudTrail logs về instance
├── Phân tích với Detective để thấy full graph
├── Xác định: initial access vector, lateral movement, exfiltration
└── Tìm IOCs (Indicators of Compromise — Dấu Hiệu Xâm Phạm): C2 IPs, malicious processes

BƯỚC 3: Recovery (Khôi Phục)
├── Terminate compromised instance (KHÔNG stop — dữ liệu volatile mất)
├── Deploy instance mới từ known-good AMI
├── Rotate tất cả credentials đã dùng trên instance đó
└── Patch vulnerability gây ra compromise

BƯỚC 4: Lessons Learned (Bài Học)
├── Post-mortem trong 48 giờ
├── Update runbook nếu cần
└── Implement prevention controls (SCPs, IMDSv2 enforcement)
```

**Automation với EventBridge + Lambda:**
```python
def handle_guardduty_finding(event, context):
    finding = event['detail']
    if finding['severity'] >= 7.0:  # High severity
        instance_id = finding['resource']['instanceDetails']['instanceId']
        # Isolate immediately
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=['sg-quarantine-id']
        )
        # Alert on-call
        sns.publish(TopicArn=ONCALL_TOPIC, Message=json.dumps(finding))
```

---

### Q18. Giải thích Amazon Macie và khi nào cần dùng.

**Đáp án:**

**Macie** là dịch vụ bảo mật data dùng ML để phát hiện **PII** (Personally Identifiable Information — Thông Tin Nhận Dạng Cá Nhân) và dữ liệu nhạy cảm trong S3.

**Các loại dữ liệu phát hiện được:**
- PII: tên, địa chỉ, số điện thoại, email, SSN (Social Security Number — Số An Sinh Xã Hội)
- PHI (Protected Health Information — Thông Tin Y Tế Được Bảo Vệ): theo HIPAA
- Financial data: số thẻ tín dụng, tài khoản ngân hàng, ABA routing numbers
- Credentials: AWS access keys, private keys, passwords trong file

**Cách hoạt động:**
1. Scan S3 buckets theo schedule hoặc on-demand
2. Phân loại data bằng ML models và pattern matching
3. Tạo findings với severity và số lượng records bị ảnh hưởng
4. Gửi findings sang Security Hub và EventBridge

**Khi dùng Macie:**
- Healthcare: tìm PHI trong S3 logs trước khi chia sẻ với vendor
- Fintech: audit trước khi move sang môi trường analytics
- GDPR compliance: định vị data của EU citizens
- Data lake governance: phát hiện data nhạy cảm bị đưa vào sai bucket

---

## Nhóm 4: Architecture & Multi-Account

### Q19. Thiết kế kiến trúc multi-account AWS an toàn cho doanh nghiệp 1000 người.

**Đáp án — AWS Landing Zone Architecture:**

```
Root Management Account (Tài Khoản Quản Lý Gốc)
│  └── SCPs global: deny ap-east-1, enforce MFA, protect CloudTrail
│
├── Security OU (Đơn Vị Tổ Chức Bảo Mật)
│   ├── Log Archive Account — tất cả CloudTrail, Config logs
│   ├── Security Tooling Account — GuardDuty master, Security Hub master
│   └── Audit Account — Audit Manager, compliance tools
│
├── Infrastructure OU
│   ├── Network Account — Transit Gateway, Shared VPCs
│   ├── Shared Services Account — AD, DNS, CI/CD tools
│   └── Backup Account — AWS Backup vaults
│
├── Workloads OU
│   ├── Production OU
│   │   ├── Prod-App-Account-1
│   │   └── Prod-App-Account-2
│   ├── Staging OU
│   │   └── Staging-Account
│   └── Development OU
│       ├── Dev-Team-A-Account
│       └── Dev-Team-B-Account
│
└── Sandbox OU (Tài Khoản Thử Nghiệm — isolated hoàn toàn)
    └── Individual Dev Sandboxes
```

**IAM Identity Center (SSO) flow:**
```
Engineer → SSO Portal → Assume Role (permission set) → Target Account
```

**Network architecture:**
- Production: Strict NACLs, private subnets only, VPC Endpoints cho AWS services
- Development: Flexible nhưng không route sang Production

---

### Q20. Zero Trust Architecture (Kiến Trúc Không Tin Tưởng Mặc Định) trên AWS — implement như thế nào?

**Đáp án — 7 nguyên tắc Zero Trust:**

1. **Verify explicitly** (Xác Minh Tường Minh): Luôn xác thực, không tin dựa trên network location

2. **Least privilege access** (Quyền Tối Thiểu): IAM conditions restrict theo time, IP, MFA status
```json
{
  "Condition": {
    "Bool": {"aws:MultiFactorAuthPresent": "true"},
    "DateLessThan": {"aws:CurrentTime": "2026-12-31T23:59:59Z"},
    "IpAddress": {"aws:SourceIp": ["10.0.0.0/8"]}
  }
}
```

3. **Assume breach** (Giả Định Bị Xâm Phạm): Segment mọi thứ, monitor lateral movement

**AWS services cho Zero Trust:**

| Layer | Service | Mục Đích |
|---|---|---|
| Identity | IAM Identity Center + Conditions | Verify identity + context |
| Device | AWS Verified Access | Kiểm tra device health trước khi allow |
| Network | VPC Endpoints, PrivateLink | Loại bỏ public internet |
| Workload | IMDSv2, IRSA cho EKS | Secure credential access |
| Data | KMS encryption + S3 Object Lock | Encrypt + immutability |
| Visibility | CloudTrail + GuardDuty + Detective | Monitor mọi action |
| Automation | EventBridge + Lambda | Auto-remediation khi phát hiện vi phạm |

---

### Q21. Giải thích Data Perimeter (Vành Đai Dữ Liệu) trên AWS.

**Đáp án:**

Data Perimeter là tập hợp các control đảm bảo:
1. **Chỉ trusted identities** mới truy cập được data
2. **Chỉ trusted resources** mới chứa data của tổ chức
3. **Chỉ từ trusted networks** mới có thể kết nối

**3 loại policy tạo nên Data Perimeter:**

**1. Resource Control Policies (RCPs — Chính Sách Kiểm Soát Tài Nguyên)** — mới trong Organizations:
```json
// Áp dụng cho S3 buckets trong org — chặn access từ ngoài org
{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {
    "StringNotEqualsIfExists": {
      "aws:PrincipalOrgID": "o-yourorgid"
    },
    "BoolIfExists": {
      "aws:PrincipalIsAWSService": "false"
    }
  }
}
```

**2. SCPs** — giới hạn identities trong org:
```json
// Chặn S3 access từ outside org (cho identities)
{
  "Condition": {
    "StringNotEquals": {
      "s3:ResourceAccount": "${aws:PrincipalAccount}"
    }
  }
}
```

**3. VPC Endpoint Policies** (Chính Sách Điểm Cuối VPC) — giới hạn network path:
```json
{
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalOrgID": "o-yourorgid"
    }
  }
}
```

---

### Q22. Thiết kế CI/CD pipeline an toàn trên AWS (không dùng long-lived credentials).

**Đáp án:**

**Vấn đề với cách cũ:** Lưu AWS access key trong GitHub Secrets → rủi ro lộ key.

**Giải pháp: OIDC Federation** (Liên Kết Định Danh OIDC):

```
GitHub Actions → GitHub OIDC Provider → sts:AssumeRoleWithWebIdentity → Temporary Credentials
```

**Setup:**

```hcl
# Terraform: Tạo OIDC Provider cho GitHub
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# IAM Role cho GitHub Actions
resource "aws_iam_role" "github_actions_deploy" {
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # Chỉ cho phép repository cụ thể và branch main
          "token.actions.githubusercontent.com:sub" = "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }]
  })
}
```

**GitHub Actions workflow:**
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
      aws-region: us-east-1
```

---

## Nhóm 5: Compliance & Advanced

### Q23. Giải thích AWS Shared Responsibility Model trong bối cảnh phỏng vấn Security.

**Đáp án:**

**Model phân chia:**

```
AWS chịu trách nhiệm:          Bạn chịu trách nhiệm:
"Security OF the Cloud"         "Security IN the Cloud"
─────────────────────          ──────────────────────────
Physical infrastructure        IAM (users, roles, policies)
Hypervisor                     Encryption (key management)
Global network                 Network configuration (SGs, NACLs)
Hardware (servers, storage)    Data classification & protection
AWS managed services runtime   OS patching (EC2)
                               Application security
```

**Thay đổi theo service model:**

| Service | AWS quản lý thêm | Bạn vẫn quản lý |
|---|---|---|
| EC2 (IaaS) | Hardware, hypervisor | OS, middleware, app, data |
| RDS (PaaS) | OS, DB engine, patching | DB config, encryption, access control |
| S3 (Object Storage) | Infrastructure, durability | Bucket policy, encryption, versioning |
| Lambda (FaaS) | Runtime, scaling | Code security, IAM role, env vars |

**Ví dụ thực tế hay dùng trong phỏng vấn:**
- RDS: AWS vá DB engine, nhưng bạn phải enable encryption at rest
- S3: AWS đảm bảo 11 9s durability, nhưng bạn phải cấu hình bucket policy đúng (S3 public buckets là lỗi của bạn)
- EC2: AWS đảm bảo hypervisor isolation, nhưng bạn phải vá OS và cấu hình Security Groups

---

### Q24. PCI-DSS (Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán) trên AWS — thiết kế kiến trúc tuân thủ.

**Đáp án — Kiến trúc CDE (Cardholder Data Environment — Môi Trường Dữ Liệu Chủ Thẻ):**

```
Internet
    │
[WAF + Shield Advanced]        ← PCI Req 6: Bảo vệ ứng dụng web
    │
[ALB — Application Load Balancer]
    │
[Private Subnet — Application Tier]
    │ ← Security Groups: chỉ accept từ ALB security group
[EC2/ECS — Application]
    │
[Private Subnet — Database Tier]
    │ ← Security Groups: chỉ accept từ App tier
[RDS PostgreSQL]               ← Encryption at rest với KMS CMK
```

**PCI-DSS Requirements → AWS Controls:**

| PCI Req | AWS Control |
|---|---|
| Req 2: No default passwords | AWS Secrets Manager rotation, Systems Manager Patch Manager |
| Req 3: Protect stored cardholder data | KMS encryption, Macie để phát hiện PAN (Primary Account Number) |
| Req 4: Encrypt transmission | ACM TLS 1.2+, Network Firewall cho TLS inspection |
| Req 7: Restrict access | IAM least privilege, SCPs |
| Req 8: Identify & authenticate | IAM Identity Center + MFA, CloudTrail |
| Req 10: Track access | CloudTrail, VPC Flow Logs, CloudWatch Logs Insights |
| Req 11: Test security | Inspector, GuardDuty, AWS Config Rules |

---

### Q25. Làm thế nào debug "Access Denied" khi policy trông có vẻ đúng?

**Đáp án — Checklist Debug IAM (7 bước):**

**Bước 1: Đọc error message đầy đủ**
```
AccessDenied: User: arn:aws:iam::123456789012:user/alice 
is not authorized to perform: s3:GetObject 
on resource: arn:aws:s3:::my-bucket/secret.txt
with an explicit deny
```
→ "explicit deny" = có policy nào đang DENY tường minh, không chỉ thiếu allow.

**Bước 2: Xác định principal**
- IAM User? IAM Role? Federated user? Service?
- Dùng: `aws sts get-caller-identity`

**Bước 3: Dùng IAM Policy Simulator**
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/secret.txt
```

**Bước 4: Kiểm tra từng lớp**
```
[ ] SCP có block action không? (nếu dùng Organizations)
[ ] Permission Boundary có include action không?
[ ] Identity policy có allow action không?
[ ] Resource policy có deny hoặc không include principal không?
[ ] Bucket ACL (nếu là S3)?
[ ] VPC Endpoint Policy (nếu access qua VPC endpoint)?
[ ] Session policy (nếu AssumeRole với policy)?
```

**Bước 5: Kiểm tra conditions**
```json
// Điều kiện có thể fail ngầm:
"Condition": {
  "Bool": {"aws:MultiFactorAuthPresent": "true"}  // → User chưa MFA sẽ bị deny
}
```

**Bước 6: Xem CloudTrail**
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=alice \
  --start-time 2026-05-16T00:00:00Z \
  --query 'Events[?EventName==`GetObject`]'
```

**Bước 7: Dùng Access Analyzer khi suspect unintended deny**
```bash
aws accessanalyzer validate-policy \
  --policy-document file://suspect-policy.json \
  --policy-type IDENTITY_POLICY
```

---

## 📋 Quick Reference — Bảng Tóm Tắt

| Concept | Giải Thích Ngắn |
|---|---|
| Implicit Deny | Không có allow = denied mặc định |
| Explicit Deny | Bất kỳ deny nào thắng mọi allow |
| SCP | Giới hạn account/OU, Management Account kiểm soát |
| Permission Boundary | Giới hạn IAM identity, không tự cấp quyền |
| Trust Policy | Ai được assume role |
| Permission Policy | Được làm gì với resource nào |
| Envelope Encryption | CMK mã hóa DEK, DEK mã hóa data |
| AssumeRole | Lấy temporary credentials để assume một role |
| ExternalId | Bảo vệ chống Confused Deputy Attack |
| ABAC | Quyền dựa trên tags, scale tốt hơn RBAC |

---

**Cập Nhật Lần Cuối:** 2026-05-16
