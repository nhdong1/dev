# CloudTrail — Management Events, Data Events & Insights Events

> CloudTrail phân loại mọi hoạt động trên AWS thành ba loại event (sự kiện): **Management Events** (Sự Kiện Quản Lý), **Data Events** (Sự Kiện Dữ Liệu) và **Insights Events** (Sự Kiện Nhận Thức). Hiểu sự khác biệt là nền tảng để cấu hình CloudTrail đúng và tối ưu chi phí.

---

## 📚 Mục Lục

1. [Management Events — Sự Kiện Quản Lý](#management-events--sự-kiện-quản-lý)
2. [Data Events — Sự Kiện Dữ Liệu](#data-events--sự-kiện-dữ-liệu)
3. [Insights Events — Sự Kiện Phát Hiện Bất Thường](#insights-events--sự-kiện-phát-hiện-bất-thường)
4. [Cấu Trúc JSON Của Một CloudTrail Event](#cấu-trúc-json-của-một-cloudtrail-event)
5. [So Sánh Ba Loại Event](#so-sánh-ba-loại-event)
6. [Chiến Lược Lựa Chọn Event Type](#chiến-lược-lựa-chọn-event-type)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Management Events — Sự Kiện Quản Lý

**Management Events** ghi lại các thao tác quản lý tài nguyên (resource management operations) trên **control plane** (mặt phẳng điều khiển) — các hành động tạo, sửa, xóa, cấu hình tài nguyên AWS.

### Đặc Điểm

- **Bật mặc định** trong Event History (90 ngày) và trong Trail mới
- **Miễn phí** cho Trail đầu tiên mỗi region
- **Volume thấp** so với Data Events — thường vài nghìn đến vài triệu/tháng
- Ghi lại cả thao tác thành công và thất bại

### Phân Loại: Read vs Write

Management Events chia thành hai nhóm:

| Nhóm              | Mô Tả                                          | Ví Dụ                                             |
| ----------------- | ---------------------------------------------- | ------------------------------------------------- |
| **Write**         | Thay đổi trạng thái tài nguyên                 | `CreateBucket`, `DeleteInstance`, `AttachPolicy`  |
| **Read**          | Lấy thông tin, không thay đổi tài nguyên        | `DescribeInstances`, `GetObject` (nếu là Mgmt API)|

> **Mặc định Trail ghi cả Read và Write.** Trong thực tế, Read events có volume rất lớn (đặc biệt từ monitoring tools gọi `Describe*` liên tục) — có thể lọc bỏ Read để giảm chi phí lưu trữ.

### Ví Dụ Management Events Quan Trọng

```
IAM:
  CreateUser, DeleteUser, CreateRole, AttachUserPolicy
  CreateAccessKey, DeleteAccessKey
  EnableMFADevice, DeactivateMFADevice

EC2:
  RunInstances, TerminateInstances, StopInstances
  AuthorizeSecurityGroupIngress, RevokeSecurityGroupIngress
  CreateKeyPair, DeleteKeyPair, ModifyInstanceAttribute

S3:
  CreateBucket, DeleteBucket, PutBucketPolicy
  PutBucketAcl, PutBucketVersioning, PutBucketEncryption

RDS:
  CreateDBInstance, DeleteDBInstance, ModifyDBInstance
  CreateDBSnapshot, RestoreDBInstanceFromDBSnapshot

CloudFormation:
  CreateStack, UpdateStack, DeleteStack

KMS:
  CreateKey, DisableKey, ScheduleKeyDeletion
  PutKeyPolicy

Organizations:
  CreateAccount, InviteAccountToOrganization
  AttachPolicy (SCP), DetachPolicy
```

### Console Sign-In Events

CloudTrail cũng ghi lại đăng nhập qua AWS Console — quan trọng cho phát hiện truy cập trái phép:

```json
{
  "eventName": "ConsoleLogin",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "john.doe"
  },
  "additionalEventData": {
    "MFAUsed": "No",
    "LoginTo": "https://console.aws.amazon.com/console/home"
  },
  "responseElements": {
    "ConsoleLogin": "Success"
  },
  "sourceIPAddress": "203.0.113.45"
}
```

> **Cảnh báo bảo mật:** `MFAUsed: No` + `sourceIPAddress` lạ = nguy cơ tài khoản bị xâm phạm.

---

## Data Events — Sự Kiện Dữ Liệu

**Data Events** ghi lại các thao tác trên **dữ liệu trong tài nguyên** (data plane operations) — đọc/ghi object S3, invoke Lambda function, đọc/ghi DynamoDB items.

### Đặc Điểm

- **Không bật mặc định** — phải cấu hình trong Trail
- **Tính phí thêm:** $0.10/100k events
- **Volume rất lớn** — production S3 bucket có thể hàng triệu GetObject/ngày
- Phải chọn lọc kỹ để tránh chi phí nổ

### Loại Tài Nguyên Hỗ Trợ Data Events

| Dịch Vụ          | Data Event                                      | Use Case Bật                          |
| ---------------- | ----------------------------------------------- | ------------------------------------- |
| **S3**           | `GetObject`, `PutObject`, `DeleteObject`         | S3 bucket chứa sensitive data         |
| **Lambda**       | `Invoke` (gọi Lambda function)                  | Audit Lambda invocation               |
| **DynamoDB**     | `GetItem`, `PutItem`, `DeleteItem`, `Query`     | Audit database access                 |
| **SNS**          | `Publish` (gửi message)                         | Message audit                         |
| **SQS**          | `SendMessage`, `ReceiveMessage`, `DeleteMessage` | Queue operation audit                 |
| **Cognito**      | `InitiateAuth`, `GetCredentialsForIdentity`     | Auth event audit                      |
| **Secrets Manager** | `GetSecretValue`                             | Audit ai đọc secret                   |
| **SSM**          | `GetParameter`, `GetParameters`                 | Audit đọc Parameter Store             |

### Cấu Hình Data Events — Lọc Cụ Thể

Thay vì bật cho tất cả tài nguyên (tốn kém), hãy lọc:

```json
{
  "DataResources": [
    {
      "Type": "AWS::S3::Object",
      "Values": [
        "arn:aws:s3:::my-sensitive-bucket/",
        "arn:aws:s3:::financial-records-bucket/"
      ]
    },
    {
      "Type": "AWS::Lambda::Function",
      "Values": [
        "arn:aws:lambda:us-east-1:123456789012:function:payment-processor"
      ]
    }
  ]
}
```

### S3 Data Events: Read vs Write

```
Read Events  (phải bật riêng):   GetObject, HeadObject, SelectObjectContent
Write Events (phải bật riêng):   PutObject, DeleteObject, CopyObject, CompleteMultipartUpload

Gợi ý: Với PCI-DSS, HIPAA — bật cả Read và Write cho bucket nhạy cảm
        Với audit thông thường — chỉ bật Write để giảm chi phí
```

---

## Insights Events — Sự Kiện Phát Hiện Bất Thường

**CloudTrail Insights** (Nhận Thức CloudTrail) phân tích tốc độ API write (management events) để phát hiện hoạt động bất thường so với baseline lịch sử.

### Đặc Điểm

- **Không bật mặc định** — phải enable trong Trail
- **Phí:** $0.35/100k write management events được phân tích
- **Phân tích hai loại:** Write API rate spike + Error rate spike (từ 2022)
- **Latency:** Insights events xuất hiện sau 15–30 phút so với thời điểm thực

### Cơ Chế Hoạt Động

```
┌─────────────────────────────────────────────────────────────────┐
│               CLOUDTRAIL INSIGHTS PIPELINE                      │
│                                                                 │
│  Write Management Events                                        │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────┐                                        │
│  │  Baseline Learning  │ ← Phân tích 7 ngày lịch sử API rate   │
│  │  (Học Đường Cơ Sở)  │                                        │
│  └─────────┬───────────┘                                        │
│            │                                                    │
│            ▼                                                    │
│  ┌─────────────────────┐                                        │
│  │  Anomaly Detection  │ ← So sánh rate hiện tại vs baseline    │
│  │ (Phát Hiện Bất      │                                        │
│  │  Thường)            │                                        │
│  └─────────┬───────────┘                                        │
│            │ (Nếu phát hiện bất thường)                         │
│            ▼                                                    │
│  ┌─────────────────────┐                                        │
│  │  Insights Event     │ → S3 + CloudWatch Logs + EventBridge  │
│  └─────────────────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Ví Dụ Insights Event Thực Tế

**Tình huống 1 — Cryptomining sau khi key bị lộ:**
```
Normal baseline:  RunInstances called ~5 times/hour
Insight detected: RunInstances called 200 times in 10 minutes
→ InsightEvent: RunInstances (errorCode: null) — unusual write activity
→ Kẻ tấn công đang spin up EC2 fleet để mine crypto
```

**Tình huống 2 — Privilege escalation:**
```
Normal baseline:  AttachRolePolicy called ~2 times/day
Insight detected: AttachRolePolicy called 50 times in 5 minutes
→ InsightEvent: AttachRolePolicy — unusual write activity
→ Script độc hại đang leo thang quyền hàng loạt
```

**Tình huống 3 — Data deletion:**
```
Normal baseline:  DeleteObject called ~100 times/day
Insight detected: DeleteObject called 10,000 times in 30 minutes
→ InsightEvent: DeleteObject — unusual write activity
→ Ransomware hoặc bug xóa dữ liệu hàng loạt
```

### Cấu Trúc Insights Event

```json
{
  "eventVersion": "1.08",
  "eventType": "AwsCloudTrailInsight",
  "insightDetails": {
    "state": "Start",
    "eventSource": "ec2.amazonaws.com",
    "eventName": "RunInstances",
    "insightType": "ApiCallRateInsight",
    "insightContext": {
      "statistics": {
        "baseline": {
          "average": 5.2
        },
        "insight": {
          "average": 203.7
        },
        "insightDuration": 15,
        "baselineDuration": 10080
      }
    }
  }
}
```

---

## Cấu Trúc JSON Của Một CloudTrail Event

Mỗi CloudTrail event là một JSON object. Hiểu cấu trúc giúp query và điều tra hiệu quả:

```json
{
  "eventVersion": "1.09",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDACKCEVSQ6C2EXAMPLE",
    "arn": "arn:aws:iam::123456789012:user/john.doe",
    "accountId": "123456789012",
    "userName": "john.doe",
    "sessionContext": {
      "sessionIssuer": {},
      "webIdFederationData": {},
      "attributes": {
        "mfaAuthenticated": "true",
        "creationDate": "2026-05-17T03:30:00Z"
      }
    }
  },
  "eventTime": "2026-05-17T03:42:17Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteBucket",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.45",
  "userAgent": "aws-cli/2.15.0 Python/3.12.0",
  "requestParameters": {
    "bucketName": "my-production-data"
  },
  "responseElements": null,
  "requestID": "EXAMPLE123456789",
  "eventID": "12345678-1234-1234-1234-123456789012",
  "readOnly": false,
  "resources": [
    {
      "ARN": "arn:aws:s3:::my-production-data",
      "accountId": "123456789012",
      "type": "AWS::S3::Bucket"
    }
  ],
  "eventType": "AwsApiCall",
  "recipientAccountId": "123456789012",
  "errorCode": null,
  "errorMessage": null
}
```

### Các Trường Quan Trọng Nhất

| Trường                   | Ý Nghĩa                                           | Dùng Để                              |
| ------------------------ | ------------------------------------------------- | ------------------------------------ |
| `userIdentity.arn`       | ARN của principal thực hiện action                | Xác định "Ai?"                       |
| `userIdentity.type`      | `IAMUser`, `AssumedRole`, `AWSService`, `Root`    | Phân loại actor                      |
| `eventTime`              | Thời điểm UTC của event                           | Timeline investigation               |
| `eventName`              | Tên API call (`DeleteBucket`, `RunInstances`...)  | Xác định "Làm gì?"                   |
| `eventSource`            | Dịch vụ nhận request (`s3.amazonaws.com`)         | Filter theo dịch vụ                  |
| `sourceIPAddress`        | IP nguồn (có thể là AWS service endpoint)         | Phát hiện IP lạ                      |
| `requestParameters`      | Tham số của request (bucket name, instance type)  | Chi tiết đối tượng bị tác động       |
| `responseElements`       | Kết quả trả về (instance ID mới tạo...)           | Confirm kết quả                      |
| `errorCode`              | Mã lỗi nếu thất bại (`AccessDenied`, `NoSuchKey`)| Phát hiện tấn công brute-force       |
| `readOnly`               | `true` = Read, `false` = Write                    | Lọc theo loại thao tác               |

### userIdentity.type — Các Loại Actor

```
Root           → Root account (nguy hiểm — không nên dùng)
IAMUser        → IAM user trực tiếp (john.doe)
AssumedRole    → Đang dùng một IAM Role (EC2, Lambda, ECS...)
FederatedUser  → Đăng nhập qua SSO/SAML/OIDC
AWSService     → AWS service tự động thực hiện (ví dụ: autoscaling.amazonaws.com)
AWSAccount     → Cross-account access
```

---

## So Sánh Ba Loại Event

| Tiêu Chí              | Management Events          | Data Events                   | Insights Events                    |
| --------------------- | -------------------------- | ----------------------------- | ---------------------------------- |
| **Mô tả**             | Thao tác trên tài nguyên   | Thao tác trên dữ liệu         | Phát hiện bất thường ML             |
| **Mặc định bật?**     | ✅ Có                      | ❌ Không                      | ❌ Không                           |
| **Phí**               | Free (trail đầu tiên)      | $0.10/100k events             | $0.35/100k write events analyzed   |
| **Volume**            | Thấp–Trung bình            | Rất lớn (triệu/ngày)          | Thấp (chỉ khi có bất thường)       |
| **Latency**           | Near real-time (15 phút)   | Near real-time (15 phút)      | 15–30 phút sau bất thường           |
| **Use case**          | Audit tài nguyên, IAM      | Audit data access, compliance | Threat detection, security alert   |
| **Lưu ở đâu**         | S3 + CW Logs               | S3 + CW Logs                  | S3 + CW Logs + EventBridge         |

---

## Chiến Lược Lựa Chọn Event Type

### Môi Trường Cơ Bản (Startup / Non-critical)

```
✅ Management Events (Write only) — Audit thay đổi tài nguyên
❌ Data Events — Chưa cần, tốn kém
❌ Insights Events — Chưa cần ngay
```

### Môi Trường Production Standard

```
✅ Management Events (Read + Write) — Full audit
✅ Data Events (S3 Write only, selective buckets) — Audit data changes
❌ Data Events Read — Quá nhiều, cân nhắc sau
❌ Insights Events — Cân nhắc nếu có security team
```

### Môi Trường Regulated (PCI-DSS / HIPAA / SOC2)

```
✅ Management Events (Read + Write) — Bắt buộc
✅ Data Events S3 (Read + Write, tất cả bucket nhạy cảm) — Bắt buộc
✅ Data Events Lambda (Invoke) — Audit function calls
✅ Data Events Secrets Manager (GetSecretValue) — Ai đọc secret?
✅ Insights Events — Phát hiện tấn công tự động
```

### Quy Tắc Vàng

```
1. Luôn bật Management Events → Không thể bỏ
2. Data Events: Chỉ bật cho tài nguyên chứa sensitive data
3. Lọc theo ARN cụ thể, không bật "All resources"
4. Bật Insights nếu có security team và budget
5. Bật S3 Data Events Read chỉ khi có regulatory requirement rõ ràng
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao Data Events không bật mặc định?

**Volume và chi phí.** Một production environment có thể có hàng triệu `GetObject` S3 mỗi ngày. Bật mặc định sẽ tạo ra:
- Chi phí lưu trữ khổng lồ (hàng chục GB logs/ngày)
- Chi phí Athena/OpenSearch query cao
- Signal-to-noise ratio (tỷ lệ tín hiệu-nhiễu) thấp — khó tìm event quan trọng

Design principle của AWS: Pay for what you need.

---

### Q2: Làm thế nào phát hiện ai đã đọc một secret trong Secrets Manager?

Bật **Data Events** cho Secrets Manager trong Trail:

```json
{
  "DataResources": [{
    "Type": "AWS::SecretsManager::Secret",
    "Values": ["arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-password"]
  }]
}
```

Sau đó query CloudTrail Event History với `eventName = GetSecretValue`.

---

### Q3: Insights Events có thể dùng để làm gì ngoài alerting?

**Phát hiện sự cố tự động với EventBridge:**

```
Insights Event → CloudWatch Logs → EventBridge Rule → Lambda
                                                        ↓
                                   Tự động: revoke credentials
                                             tạo GuardDuty finding
                                             notify security team
                                             snapshot EC2 để forensics
```

---

### Q4: Khi IAM Role thực hiện action, CloudTrail ghi gì trong userIdentity?

Khi một IAM Role được assume (đảm nhận) và thực hiện action:

```json
{
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROACKCEVSQ6C2:i-0abc123",
    "arn": "arn:aws:sts::123456789012:assumed-role/EC2-Role/i-0abc123",
    "accountId": "123456789012",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROACKCEVSQ6C2",
        "arn": "arn:aws:iam::123456789012:role/EC2-Role",
        "accountId": "123456789012",
        "userName": "EC2-Role"
      }
    }
  }
}
```

`sessionContext.sessionIssuer` cho biết Role gốc — hữu ích khi nhiều EC2 instance dùng cùng một role.

---

### Q5: Làm thế nào filter chỉ lấy failed API calls?

Failed calls (bị từ chối quyền, resource không tồn tại...) có `errorCode` khác null. Trong Athena:

```sql
SELECT
  eventTime,
  userIdentity.arn,
  eventName,
  errorCode,
  errorMessage,
  sourceIPAddress
FROM cloudtrail_logs
WHERE errorCode IS NOT NULL
  AND errorCode = 'AccessDenied'
  AND eventTime > '2026-05-01'
ORDER BY eventTime DESC;
```

Nhiều `AccessDenied` từ cùng một user/IP = dấu hiệu tấn công hoặc misconfiguration.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [README.md](./README.md) | [2-trails-configuration.md](./2-trails-configuration.md)
