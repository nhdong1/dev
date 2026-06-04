# CloudTrail Forensics & Investigation — Điều Tra Sự Cố Và Phân Tích Event History

> **Digital forensics** (Pháp Chứng Kỹ Thuật Số) với CloudTrail là nghệ thuật tái hiện lại chính xác những gì đã xảy ra trên AWS — ai làm gì, theo thứ tự nào, hậu quả ra sao — từ dấu vết không thể giả mạo được lưu trong audit logs. Đây là kỹ năng thiết yếu cho Cloud Security Engineer và DevOps Engineer.

---

## 📚 Mục Lục

1. [Nguyên Tắc Forensics Trên AWS](#nguyên-tắc-forensics-trên-aws)
2. [Phương Pháp Điều Tra — Investigation Framework](#phương-pháp-điều-tra--investigation-framework)
3. [Công Cụ Điều Tra](#công-cụ-điều-tra)
4. [Tình Huống Thực Tế — Playbooks](#tình-huống-thực-tế--playbooks)
5. [Query Patterns Quan Trọng](#query-patterns-quan-trọng)
6. [Bảo Tồn Bằng Chứng — Evidence Preservation](#bảo-tồn-bằng-chứng--evidence-preservation)
7. [Reporting & Timeline Reconstruction](#reporting--timeline-reconstruction)
8. [Chuẩn Bị Trước Khi Sự Cố Xảy Ra](#chuẩn-bị-trước-khi-sự-cố-xảy-ra)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Nguyên Tắc Forensics Trên AWS

### Tính Không Thể Phủ Nhận Của CloudTrail

CloudTrail logs là bằng chứng **non-repudiable** (không thể phủ nhận) vì:
- Được ghi bởi AWS infrastructure — entity bị điều tra không thể kiểm soát
- Log File Validation tạo cryptographic proof (bằng chứng mã hóa)
- IAM principal, timestamp, IP address đều được ghi chính xác

### Giới Hạn Cần Biết

```
CloudTrail GHI LẠI:
  ✅ Mọi API call đến AWS endpoints
  ✅ Console actions (dùng API ngầm)
  ✅ SDK/CLI calls
  ✅ Service-to-service calls
  ✅ Cross-account actions

CloudTrail KHÔNG GHI:
  ❌ Traffic bên trong EC2 (SSH session content)
  ❌ Database queries trực tiếp (SELECT, INSERT...)
  ❌ Nội dung file trong S3 (chỉ ghi metadata của GetObject)
  ❌ In-memory operations
  ❌ Network packets (cần VPC Flow Logs)
```

### Tam Giác Điều Tra

```
     CloudTrail
     (Who did what?)
          │
          │
          ▼
CloudWatch Logs ──── VPC Flow Logs
(What happened    (Network activity)
 in the app?)
```

---

## Phương Pháp Điều Tra — Investigation Framework

### Quy Trình PICERL

```
P - Preparation (Chuẩn Bị)
    → Đảm bảo Trail đang chạy, logs được lưu đúng
    → Có Athena table setup sẵn
    → Có quyền query và xuất logs

I - Identification (Nhận Diện)
    → Xác định event/resource bị ảnh hưởng
    → Xác định khoảng thời gian sự cố

C - Containment (Ngăn Chặn)
    → Cô lập credentials bị compromise
    → Ngăn sự cố lan rộng thêm

E - Eradication (Loại Bỏ)
    → Xóa access của kẻ tấn công
    → Khắc phục misconfiguration

R - Recovery (Phục Hồi)
    → Restore resources bị xóa/thay đổi
    → Verify hệ thống sạch

L - Lessons Learned (Bài Học)
    → Viết post-mortem
    → Cải thiện security posture
```

---

## Công Cụ Điều Tra

### 1. Event History — Điều Tra Nhanh (UI)

**Phù hợp:** Tra cứu ad-hoc, 90 ngày gần nhất, không cần setup.

```
CloudTrail → Event History
  Filter by:
    - Event name: DeleteBucket, RunInstances...
    - User name: john.doe
    - Resource type: AWS::S3::Bucket
    - Resource name: my-bucket-name
    - Time range: [from] → [to]
    - AWS access key: AKIAIOSFODNN7EXAMPLE
```

**Giới hạn:** Chỉ Management Events, tối đa 90 ngày, không có advanced filtering.

### 2. Amazon Athena — Phân Tích Quy Mô Lớn

**Phù hợp:** Query phức tạp, dữ liệu nhiều tháng, cross-account, lọc theo nhiều điều kiện.

**Setup Athena Table:**

```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        invokedBy: STRING,
        accessKeyId: STRING,
        userName: STRING,
        sessionContext: STRUCT<
            sessionIssuer: STRUCT<
                type: STRING,
                principalId: STRING,
                arn: STRING,
                accountId: STRING,
                userName: STRING
            >,
            attributes: STRUCT<
                mfaAuthenticated: STRING,
                creationDate: STRING
            >
        >
    >,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    awsRegion STRING,
    sourceIPAddress STRING,
    userAgent STRING,
    requestParameters STRING,
    responseElements STRING,
    errorCode STRING,
    errorMessage STRING,
    requestID STRING,
    eventID STRING,
    eventType STRING,
    recipientAccountId STRING,
    readOnly STRING,
    resources ARRAY<STRUCT<
        arn: STRING,
        accountId: STRING,
        type: STRING
    >>
)
COMMENT 'CloudTrail table for log analysis'
PARTITIONED BY (
    account_id STRING,
    region STRING,
    year STRING,
    month STRING,
    day STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-bucket/AWSLogs/'
TBLPROPERTIES ('has_encrypted_data'='true');
```

### 3. CloudWatch Logs Insights — Real-time Query

**Phù hợp:** Query nhanh trong CloudWatch Logs, real-time alerting.

```
# Truy vấn cơ bản
fields eventTime, userIdentity.arn, eventName, sourceIPAddress
| filter eventSource = "s3.amazonaws.com"
| sort eventTime desc
| limit 50
```

### 4. Amazon OpenSearch Service

**Phù hợp:** SIEM (Security Information and Event Management), full-text search, dashboards.

Kiến trúc pipeline:
```
CloudTrail → S3 → Lambda → OpenSearch → Kibana Dashboard
                                      → Alerts
```

---

## Tình Huống Thực Tế — Playbooks

### Playbook 1: "Ai Đã Xóa S3 Bucket?"

**Tình huống:** 3 giờ sáng, monitoring alert — backup bucket không còn tồn tại.

**Bước 1: Tìm event xóa bucket**

```sql
-- Athena query
SELECT
    eventTime,
    userIdentity.arn AS actor,
    userIdentity.type AS actor_type,
    userIdentity.accountId AS account,
    awsRegion,
    sourceIPAddress,
    userAgent,
    requestParameters
FROM cloudtrail_logs
WHERE eventName = 'DeleteBucket'
  AND year = '2026' AND month = '05' AND day = '17'
ORDER BY eventTime DESC;
```

**Bước 2: Tìm hoạt động trước đó của actor**

```sql
-- Xem actor đã làm gì trong 1 giờ trước khi xóa bucket
SELECT
    eventTime,
    eventName,
    eventSource,
    sourceIPAddress
FROM cloudtrail_logs
WHERE userIdentity.arn = 'arn:aws:iam::123456789012:user/john.doe'
  AND eventTime BETWEEN '2026-05-17T02:00:00Z' AND '2026-05-17T03:00:00Z'
ORDER BY eventTime;
```

**Bước 3: Kiểm tra nếu actor là assumed role**

```sql
-- Nếu actor là AssumedRole, tìm ai đã assume role đó
SELECT
    eventTime,
    userIdentity.arn AS requester,
    requestParameters,
    sourceIPAddress
FROM cloudtrail_logs
WHERE eventName = 'AssumeRole'
  AND requestParameters LIKE '%RoleToAssume%'
  AND eventTime > '2026-05-17T01:00:00Z'
ORDER BY eventTime;
```

**Kết quả điều tra:**
```
Tìm ra: IAM user "john.doe" đăng nhập từ IP 203.0.113.45 (IP lạ, không phải VPN công ty)
        lúc 02:47 UTC, không có MFA, xóa bucket lúc 02:51 UTC.
        
Action: 
  - Revoke access key của john.doe ngay lập tức
  - Force password reset
  - Restore bucket từ S3 Replication hoặc backup
  - Báo cáo incident
```

---

### Playbook 2: "EC2 Instances Bị Tạo Bởi Ai?"

**Tình huống:** Hóa đơn AWS tháng này tăng đột biến $50,000 — phát hiện hàng trăm EC2 instance lạ.

**Bước 1: Liệt kê tất cả RunInstances gần đây**

```sql
SELECT
    eventTime,
    userIdentity.arn AS creator,
    awsRegion,
    json_extract_scalar(requestParameters, '$.instanceType') AS instance_type,
    json_extract_scalar(responseElements, '$.instancesSet.items[0].instanceId') AS instance_id,
    sourceIPAddress
FROM cloudtrail_logs
WHERE eventName = 'RunInstances'
  AND year = '2026' AND month = '05'
ORDER BY eventTime DESC
LIMIT 200;
```

**Bước 2: Group theo actor để tìm thủ phạm**

```sql
SELECT
    userIdentity.arn AS creator,
    COUNT(*) AS instances_created,
    MIN(eventTime) AS first_seen,
    MAX(eventTime) AS last_seen,
    ARRAY_AGG(DISTINCT awsRegion) AS regions,
    ARRAY_AGG(DISTINCT sourceIPAddress) AS source_ips
FROM cloudtrail_logs
WHERE eventName = 'RunInstances'
  AND year = '2026' AND month = '05'
GROUP BY userIdentity.arn
ORDER BY instances_created DESC;
```

**Bước 3: Lấy danh sách instance IDs để terminate**

```sql
SELECT
    json_extract_scalar(responseElements, '$.instancesSet.items[0].instanceId') AS instance_id,
    awsRegion,
    eventTime
FROM cloudtrail_logs
WHERE eventName = 'RunInstances'
  AND userIdentity.arn = 'arn:aws:iam::123456789012:user/attacker'
  AND year = '2026' AND month = '05';
```

**Terminate hàng loạt qua CLI:**

```bash
# Lấy instance IDs từ Athena output và terminate
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags]' \
  --output text | grep -v legitimate-tag

# Terminate batch
aws ec2 terminate-instances \
  --instance-ids i-0abc123 i-0def456 i-0ghi789
```

---

### Playbook 3: "Credential Bị Lộ — Blast Radius Analysis"

**Tình huống:** Security team phát hiện AWS Access Key bị post lên GitHub public repo.

**Bước 1: Xác định access key**

```
Key ID: AKIAIOSFODNN7EXAMPLE (từ GitHub secret scanning alert)
```

**Bước 2: Tìm tất cả actions thực hiện bởi key này**

```sql
-- CloudTrail lưu access key trong userIdentity.accessKeyId
SELECT
    eventTime,
    eventName,
    eventSource,
    awsRegion,
    sourceIPAddress,
    userAgent,
    errorCode,
    requestParameters
FROM cloudtrail_logs
WHERE userIdentity.accessKeyId = 'AKIAIOSFODNN7EXAMPLE'
ORDER BY eventTime;
```

**Bước 3: Phân tích blast radius**

```sql
-- Tổng hợp theo loại action
SELECT
    eventSource,
    eventName,
    COUNT(*) AS count,
    MIN(eventTime) AS first_seen,
    MAX(eventTime) AS last_seen,
    errorCode
FROM cloudtrail_logs
WHERE userIdentity.accessKeyId = 'AKIAIOSFODNN7EXAMPLE'
GROUP BY eventSource, eventName, errorCode
ORDER BY count DESC;
```

**Bước 4: Kiểm tra resources bị tạo (để cleanup)**

```sql
-- EC2 instances tạo bởi key này
SELECT
    json_extract_scalar(responseElements, '$.instancesSet.items[0].instanceId') AS instance_id,
    eventTime,
    awsRegion
FROM cloudtrail_logs
WHERE userIdentity.accessKeyId = 'AKIAIOSFODNN7EXAMPLE'
  AND eventName = 'RunInstances'
  AND errorCode IS NULL;

-- IAM users/roles được tạo (attacker có thể tạo backdoor)
SELECT
    eventTime, eventName, requestParameters
FROM cloudtrail_logs
WHERE userIdentity.accessKeyId = 'AKIAIOSFODNN7EXAMPLE'
  AND eventSource = 'iam.amazonaws.com'
  AND eventName IN ('CreateUser', 'CreateRole', 'CreateAccessKey',
                    'AttachUserPolicy', 'AttachRolePolicy',
                    'PutUserPolicy', 'PutRolePolicy')
  AND errorCode IS NULL;
```

**Immediate Actions:**

```bash
# 1. Revoke access key NGAY LẬP TỨC
aws iam update-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive

# 2. Liệt kê và xóa IAM resources do attacker tạo
aws iam list-users --query 'Users[?CreateDate>`2026-05-17`]'

# 3. Revoke tất cả STS sessions liên quan
# (Xóa user policy, detach managed policies, hoặc đổi user password)
```

---

### Playbook 4: "Root Account Đã Được Dùng"

**Tình huống:** Nhận alert từ CloudWatch Alarm — root account vừa đăng nhập.

**Bước 1: Lấy chi tiết root login**

```sql
SELECT
    eventTime,
    userIdentity.type,
    eventName,
    sourceIPAddress,
    userAgent,
    additionalEventData,
    responseElements
FROM cloudtrail_logs
WHERE userIdentity.type = 'Root'
  AND year = '2026' AND month = '05' AND day = '17'
ORDER BY eventTime;
```

**Bước 2: Xem root account đã làm gì**

```sql
SELECT
    eventTime,
    eventName,
    eventSource,
    sourceIPAddress,
    requestParameters,
    errorCode
FROM cloudtrail_logs
WHERE userIdentity.type = 'Root'
  AND eventTime > '2026-05-17T00:00:00Z'
ORDER BY eventTime;
```

**Câu hỏi cần trả lời:**
- Có MFA được dùng không? (`additionalEventData.MFAUsed`)
- IP có phải IP của admin team không?
- Action thực hiện có hợp lý không? (Một số tasks buộc phải dùng root)
- Có tạo IAM user mới hoặc thay đổi quyền không?

---

## Query Patterns Quan Trọng

### Phát Hiện Tấn Công IAM

```sql
-- 1. Tìm credential enumeration (attacker liệt kê IAM resources)
SELECT
    eventTime,
    userIdentity.arn,
    eventName,
    sourceIPAddress,
    COUNT(*) OVER (PARTITION BY userIdentity.arn
                  ORDER BY eventTime
                  ROWS BETWEEN 10 PRECEDING AND CURRENT ROW) AS rolling_count
FROM cloudtrail_logs
WHERE eventName IN ('ListUsers', 'ListRoles', 'ListGroups',
                    'ListAttachedUserPolicies', 'GetUser',
                    'DescribeInstances', 'ListBuckets')
  AND year = '2026' AND month = '05'
ORDER BY eventTime;

-- 2. Tìm privilege escalation attempts
SELECT eventTime, userIdentity.arn, eventName, requestParameters, errorCode
FROM cloudtrail_logs
WHERE eventName IN (
    'AttachUserPolicy', 'AttachRolePolicy',
    'PutUserPolicy', 'PutRolePolicy',
    'CreatePolicyVersion', 'SetDefaultPolicyVersion',
    'PassRole', 'CreateLoginProfile',
    'UpdateLoginProfile'
)
ORDER BY eventTime DESC;
```

### Phát Hiện Data Exfiltration (Rò Rỉ Dữ Liệu)

```sql
-- Tìm GetObject lượng lớn từ IP lạ (cần Data Events bật)
SELECT
    sourceIPAddress,
    userIdentity.arn,
    COUNT(*) AS objects_downloaded,
    MIN(eventTime) AS start_time,
    MAX(eventTime) AS end_time
FROM cloudtrail_logs
WHERE eventName = 'GetObject'
  AND year = '2026' AND month = '05'
GROUP BY sourceIPAddress, userIdentity.arn
HAVING COUNT(*) > 1000
ORDER BY objects_downloaded DESC;
```

### Phát Hiện Security Group Thay Đổi

```sql
-- Tìm mọi thay đổi Security Group
SELECT
    eventTime,
    userIdentity.arn,
    eventName,
    requestParameters,
    sourceIPAddress,
    awsRegion
FROM cloudtrail_logs
WHERE eventName IN (
    'AuthorizeSecurityGroupIngress',
    'AuthorizeSecurityGroupEgress',
    'RevokeSecurityGroupIngress',
    'RevokeSecurityGroupEgress',
    'CreateSecurityGroup',
    'DeleteSecurityGroup'
)
ORDER BY eventTime DESC;
```

### Phát Hiện CloudTrail Tampering

```sql
-- Ai cố gắng tắt hoặc xóa CloudTrail?
SELECT
    eventTime,
    userIdentity.arn,
    eventName,
    awsRegion,
    sourceIPAddress,
    errorCode
FROM cloudtrail_logs
WHERE eventName IN (
    'StopLogging',
    'DeleteTrail',
    'UpdateTrail',
    'PutEventSelectors'
)
ORDER BY eventTime DESC;
```

---

## Bảo Tồn Bằng Chứng — Evidence Preservation

Khi xảy ra sự cố nghiêm trọng, cần bảo tồn evidence trước khi thực hiện bất kỳ thay đổi nào:

### Checklist Evidence Preservation

```
1. KHÔNG XÓA resources bị compromise ngay
   → EC2: Tạo snapshot trước khi terminate
   → S3: Bật Object Lock / Versioning nếu chưa có

2. Export CloudTrail logs liên quan ra nơi an toàn
   → Download từ S3 vào máy local hoặc offline storage
   → Tính hash SHA-256 của từng file để prove integrity

3. Chụp lại state hiện tại của tài nguyên bị ảnh hưởng
   → aws ec2 describe-instances
   → aws iam get-user / list-attached-user-policies
   → aws s3api get-bucket-policy

4. Ghi lại timeline điều tra
   → Mỗi bước điều tra, ai làm, lúc nào, kết quả gì

5. Lưu network artifacts (nếu cần)
   → VPC Flow Logs
   → Load Balancer access logs
   → WAF logs
```

### Tạo Evidence Package

```bash
#!/bin/bash
# Thu thập evidence cho một incident
INCIDENT_ID="INC-2026-001"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
START_TIME="2026-05-17T00:00:00Z"
END_TIME="2026-05-17T06:00:00Z"

mkdir -p evidence/${INCIDENT_ID}

# 1. Download CloudTrail logs
aws s3 sync \
  s3://my-cloudtrail-bucket/AWSLogs/${ACCOUNT_ID}/CloudTrail/ \
  evidence/${INCIDENT_ID}/cloudtrail/ \
  --exclude "*" \
  --include "*/2026/05/17/*"

# 2. Tạo hash manifest
find evidence/${INCIDENT_ID} -type f -exec sha256sum {} \; > \
  evidence/${INCIDENT_ID}/MANIFEST.sha256

# 3. Snapshot EC2 instances bị nghi ngờ
INSTANCE_IDS=("i-0abc123" "i-0def456")
for INSTANCE_ID in "${INSTANCE_IDS[@]}"; do
  aws ec2 create-snapshot \
    --volume-id $(aws ec2 describe-instances \
      --instance-ids ${INSTANCE_ID} \
      --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId' \
      --output text) \
    --description "Evidence snapshot - ${INCIDENT_ID} - ${INSTANCE_ID}" \
    --tag-specifications "ResourceType=snapshot,Tags=[{Key=IncidentId,Value=${INCIDENT_ID}}]"
done

echo "Evidence collection complete for ${INCIDENT_ID}"
```

---

## Reporting & Timeline Reconstruction

### Mẫu Báo Cáo Sự Cố

```markdown
# Báo Cáo Sự Cố Bảo Mật — INC-2026-001

## Tóm Tắt
- **Thời gian phát hiện:** 2026-05-17T03:15:00Z
- **Thời gian sự cố bắt đầu:** 2026-05-17T02:47:00Z (ước tính)
- **Mức độ:** Nghiêm trọng (S1)
- **Tác động:** Xóa 3 S3 buckets sản xuất, mất ~50GB data

## Timeline

| Thời Gian (UTC)     | Event                                              | Nguồn         |
| ------------------- | -------------------------------------------------- | ------------- |
| 2026-05-16T18:30:00 | Access key AKIA... commit lên GitHub public repo   | GitHub logs   |
| 2026-05-17T02:47:00 | ConsoleLogin từ IP 203.0.113.45 (không có MFA)     | CloudTrail    |
| 2026-05-17T02:49:00 | ListBuckets — liệt kê tất cả S3 buckets            | CloudTrail    |
| 2026-05-17T02:51:13 | DeleteBucket: backup-prod-2026                     | CloudTrail    |
| 2026-05-17T02:51:45 | DeleteBucket: logs-archive-2026                    | CloudTrail    |
| 2026-05-17T02:52:07 | DeleteBucket: customer-data-exports                | CloudTrail    |
| 2026-05-17T03:00:00 | CloudWatch Alarm kích hoạt — S3 access failed      | CloudWatch    |
| 2026-05-17T03:15:00 | Security team nhận alert                           | PagerDuty     |
| 2026-05-17T03:20:00 | Revoke access key                                  | Remediation   |

## Root Cause (Nguyên Nhân Gốc Rễ)
AWS access key bị hardcode trong source code và commit lên repository public.

## Bài Học
1. Implement pre-commit hook để phát hiện credentials
2. Bật GitHub secret scanning alerts
3. Require MFA cho tất cả IAM users
4. Bật S3 Object Lock cho buckets quan trọng
```

---

## Chuẩn Bị Trước Khi Sự Cố Xảy Ra

### Security Runbook Pre-built Queries

Chuẩn bị sẵn các query thường dùng để không mất thời gian viết khi sự cố xảy ra:

```bash
# Save vào file ~/cloudtrail-runbook.sh
CLOUDTRAIL_DB="cloudtrail_db"
CLOUDTRAIL_TABLE="cloudtrail_logs"

# Function: Tìm actions bởi một user/role
find_actor_actions() {
    local actor_arn=$1
    local hours_back=${2:-24}
    aws athena start-query-execution \
        --query-string "SELECT eventTime, eventName, awsRegion, sourceIPAddress
                        FROM ${CLOUDTRAIL_TABLE}
                        WHERE userIdentity.arn = '${actor_arn}'
                          AND eventTime > DATE_ADD('hour', -${hours_back}, NOW())
                        ORDER BY eventTime;" \
        --query-execution-context Database=${CLOUDTRAIL_DB}
}

# Function: Tìm actions với một access key
find_key_actions() {
    local key_id=$1
    aws athena start-query-execution \
        --query-string "SELECT eventTime, eventName, eventSource, awsRegion, sourceIPAddress, errorCode
                        FROM ${CLOUDTRAIL_TABLE}
                        WHERE userIdentity.accessKeyId = '${key_id}'
                        ORDER BY eventTime;" \
        --query-execution-context Database=${CLOUDTRAIL_DB}
}
```

### Checklist Chuẩn Bị Forensics

```
☑ CloudTrail Multi-Region Trail đang chạy với Log File Validation
☑ Logs gửi vào S3 + CloudWatch Logs
☑ Athena table được tạo và test sẵn
☑ S3 bucket có MFA Delete + Object Lock (COMPLIANCE mode)
☑ CloudWatch Alarms cho các event nguy hiểm (root login, trail disabled...)
☑ Runbook điều tra được viết và test trước
☑ Team biết ai có quyền query CloudTrail, ai có quyền revoke credentials
☑ Contact list: Security team, Legal, Compliance
☑ Log retention policy đáp ứng regulatory requirements (1 năm cho PCI-DSS)
☑ Athena query results được lưu vào S3 bucket riêng (không overwrite)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Ai đó xóa CloudTrail Trail của bạn — bạn còn có thể điều tra được không?

**Vẫn có thể**, nhờ:

1. **Event History (90 ngày):** Vẫn hiển thị Management Events gần nhất, kể cả event `DeleteTrail`.
2. **S3 logs đã tồn tại:** Logs ghi trước khi Trail bị xóa vẫn nằm trong S3.
3. **S3 Access Logs:** Nếu bật S3 server access logging cho bucket, sẽ ghi lại ai download/xóa files.
4. **CloudWatch Alarm:** Nếu đã tạo Alarm cho `DeleteTrail` event, team nhận alert ngay.

**Quan trọng hơn:** Nếu dùng SCP để deny `DeleteTrail` cho tất cả member accounts, kẻ tấn công không thể xóa Trail ngay cả khi có quyền admin trong account.

---

### Q2: Tại sao cần giữ CloudTrail logs ít nhất 1 năm?

**Yêu cầu regulatory (pháp lý):**
- **PCI-DSS:** Yêu cầu giữ audit log 1 năm (3 tháng phải accessible ngay, 9 tháng còn lại lưu trữ)
- **HIPAA:** 6 năm cho health data audit logs
- **SOC 2:** 1 năm cho audit evidence
- **ISO 27001:** 1 năm tối thiểu

**Lý do thực tế:** Một số tấn công kiểu APT (Advanced Persistent Threat — Mối Đe Dọa Dai Dẳng Nâng Cao) "nằm vùng" trong hệ thống nhiều tháng trước khi có hành động rõ ràng. Phân tích forensics cần dữ liệu dài hạn để tái hiện đầy đủ.

---

### Q3: Làm thế nào tái hiện chuỗi sự kiện cross-account?

**Tình huống:** User trong Account A assume role trong Account B và thực hiện action.

**Bước 1:** Trong Account B, tìm event với `userIdentity.type = AssumedRole`:
```sql
SELECT userIdentity.arn, userIdentity.sessionContext.sessionIssuer.arn, eventTime, eventName
FROM cloudtrail_logs_account_b
WHERE eventName = 'DeleteObject';
```

**Bước 2:** `sessionContext.sessionIssuer.arn` cho biết role ở Account B. Tìm ai đã AssumeRole:
```sql
SELECT userIdentity.arn AS requester, eventTime, requestParameters, sourceIPAddress
FROM cloudtrail_logs_account_a
WHERE eventName = 'AssumeRole'
  AND requestParameters LIKE '%role-in-account-b%';
```

**Kết quả:** `john.doe` trong Account A assume `CrossAccountRole` lúc 02:45, sau đó thực hiện `DeleteObject` trong Account B lúc 02:47. Chain hoàn chỉnh.

---

### Q4: Làm thế nào biết action được thực hiện từ trong VPN công ty hay từ ngoài?

**Cách 1 — Kiểm tra sourceIPAddress:**
```sql
SELECT sourceIPAddress, COUNT(*) as count
FROM cloudtrail_logs
WHERE userIdentity.arn = 'arn:aws:iam::123456789012:user/john.doe'
GROUP BY sourceIPAddress
ORDER BY count DESC;
```

So sánh với danh sách IP VPN công ty (ví dụ: `10.0.0.0/8`, `203.0.113.0/24`).

**Cách 2 — userAgent:**
- `console.amazonaws.com` = AWS Management Console
- `aws-cli/2.x.x` = CLI tool
- Custom SDK user agents
- Nếu thấy user agent lạ (Python script, Golang binary...) = dấu hiệu automation của kẻ tấn công

**Cách 3 — Enrichment với IP geolocation:**
Dùng Lambda để enrich CloudTrail events với IP geolocation data từ MaxMind GeoIP database — nếu IP từ quốc gia không hoạt động, raise alert.

---

### Q5: CloudTrail có đủ để điều tra toàn bộ sự cố không?

**Không hoàn toàn.** CloudTrail là nền tảng nhưng cần kết hợp:

| Loại Dữ Liệu Cần          | Nguồn                             |
| -------------------------- | ---------------------------------- |
| API calls                  | CloudTrail                        |
| Network traffic            | VPC Flow Logs                     |
| DNS queries                | Route53 Resolver Logs             |
| OS-level activity          | CloudWatch Agent Logs             |
| Application logs           | CloudWatch Logs / S3              |
| Database queries           | RDS slow query log / audit log    |
| Container activity         | ECS/EKS Container Insights        |
| Web application traffic    | ALB Access Logs / WAF Logs        |

**Security investigation** tốt nhất cần correlate (tương quan) tất cả nguồn này — đó là lý do doanh nghiệp lớn dùng **SIEM** (Security Information and Event Management) như Splunk, IBM QRadar, hoặc Amazon Security Lake.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [4-cloudtrail-insights.md](./4-cloudtrail-insights.md) | [README.md](./README.md)
