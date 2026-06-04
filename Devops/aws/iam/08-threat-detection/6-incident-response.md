# Incident Response — Runbook Xử Lý Sự Cố Bảo Mật AWS

> Hướng dẫn có cấu trúc để phát hiện, điều tra, ngăn chặn, khắc phục và phục hồi sau sự cố bảo mật trên AWS — dựa trên AWS Security Incident Response Guide và NIST SP 800-61 (Computer Security Incident Handling Guide).

## 📚 Mục Lục

1. [Incident Response Framework](#incident-response-framework)
2. [Phân Loại Sự Cố](#phân-loại-sự-cố)
3. [Runbook: IAM Credential Compromise](#runbook-iam-credential-compromise)
4. [Runbook: EC2 Instance Compromise](#runbook-ec2-instance-compromise)
5. [Runbook: S3 Data Breach](#runbook-s3-data-breach)
6. [Runbook: Cryptomining Detection](#runbook-cryptomining-detection)
7. [Runbook: Unauthorized Account Access](#runbook-unauthorized-account-access)
8. [Forensic Preservation — Bảo Toàn Bằng Chứng](#forensic-preservation)
9. [Automated Response Architecture](#automated-response-architecture)
10. [Post-Incident Activities — Hoạt Động Sau Sự Cố](#post-incident-activities)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Incident Response Framework

### 6 Giai Đoạn (NIST SP 800-61)

```
1. PREPARATION (Chuẩn Bị)
   ├── Bật GuardDuty, Security Hub, CloudTrail trước khi incident xảy ra
   ├── Viết runbooks, phân công vai trò
   ├── Chuẩn bị forensic AMI sạch
   └── Test automation scripts định kỳ

2. DETECTION & ANALYSIS (Phát Hiện & Phân Tích)
   ├── GuardDuty/Security Hub báo finding
   ├── Triage — đánh giá mức độ nghiêm trọng
   ├── Mở Detective investigation
   └── Xác định scope (phạm vi ảnh hưởng)

3. CONTAINMENT (Ngăn Chặn)
   ├── Short-term: isolate resource bị ảnh hưởng ngay
   └── Long-term: fix underlying weakness

4. ERADICATION (Loại Bỏ)
   ├── Xóa access của attacker
   ├── Patch vulnerabilities
   └── Xóa malware/backdoors

5. RECOVERY (Phục Hồi)
   ├── Restore từ clean backup hoặc AMI sạch
   ├── Monitor chặt chẽ sau khi restore
   └── Xác nhận hệ thống hoạt động bình thường

6. POST-INCIDENT (Sau Sự Cố)
   ├── Viết incident report
   ├── Lessons learned meeting
   └── Update runbooks, cải thiện controls
```

### Vai Trò Trong Incident Response

| Vai Trò | Trách Nhiệm |
|---|---|
| **Incident Commander** | Điều phối tổng thể, communication, quyết định ưu tiên |
| **Security Analyst** | Điều tra kỹ thuật, phân tích log, Detective investigation |
| **Operations Engineer** | Thực thi containment, recovery, infrastructure changes |
| **Legal/Compliance** | Xác định nghĩa vụ pháp lý, breach notification |
| **Communications** | Thông báo stakeholders nội bộ và ngoài (nếu cần) |

---

## Phân Loại Sự Cố

### Severity Matrix (Ma Trận Mức Độ Nghiêm Trọng)

| Mức Độ | Tiêu Chí | Thời Gian Phản Hồi | Ví Dụ |
|---|---|---|---|
| **P1 — Critical** | Data breach đang xảy ra, production down | < 15 phút | Root account bị xâm phạm, S3 PII bị expose công khai |
| **P2 — High** | Compromise đã xác nhận, nguy cơ lan rộng | < 1 giờ | EC2 giao tiếp C2, IAM credentials bị lộ |
| **P3 — Medium** | Hành vi đáng ngờ, chưa xác nhận | < 4 giờ | Port scanning bất thường, API calls từ IP lạ |
| **P4 — Low** | Violation chính sách không nghiêm trọng | < 24 giờ | S3 bucket thiếu encryption, MFA chưa bật |

---

## Runbook: IAM Credential Compromise — Thông Tin Đăng Nhập IAM Bị Xâm Phạm

### Dấu Hiệu Nhận Biết

- GuardDuty: `UnauthorizedAccess:IAMUser/MaliciousIPCaller`
- GuardDuty: `Persistence:IAMUser/UserPermissions`
- GuardDuty: `Stealth:IAMUser/CloudTrailLoggingDisabled`
- CloudTrail: API calls từ IP/region bất thường
- Macie: AWS_SECRET_ACCESS_KEY tìm thấy trong S3 bucket công khai

### Quy Trình Xử Lý

```bash
# === BƯỚC 1: TRIAGE — Đánh Giá Ban Đầu ===
# Xác định credential nào bị compromise
FINDING_DETAIL=$(aws guardduty get-findings \
  --detector-id <DETECTOR_ID> \
  --finding-ids <FINDING_ID>)

USERNAME=$(echo $FINDING_DETAIL | jq -r '.Findings[0].Resource.AccessKeyDetails.UserName')
ACCESS_KEY=$(echo $FINDING_DETAIL | jq -r '.Findings[0].Resource.AccessKeyDetails.AccessKeyId')

echo "Compromised user: $USERNAME"
echo "Compromised key: $ACCESS_KEY"

# === BƯỚC 2: CONTAINMENT — Ngăn Chặn Ngay ===
# 2a. Disable access key ngay lập tức
aws iam update-access-key \
  --user-name "$USERNAME" \
  --access-key-id "$ACCESS_KEY" \
  --status Inactive

# 2b. Revoke tất cả sessions đang active
aws iam delete-login-profile --user-name "$USERNAME" 2>/dev/null || true

# 2c. Detach tất cả policies (nếu cần isolation hoàn toàn)
# CẢNH BÁO: Kiểm tra kỹ trước khi chạy — có thể break applications
ATTACHED_POLICIES=$(aws iam list-attached-user-policies \
  --user-name "$USERNAME" \
  --query 'AttachedPolicies[].PolicyArn' \
  --output text)

for POLICY_ARN in $ATTACHED_POLICIES; do
  aws iam detach-user-policy \
    --user-name "$USERNAME" \
    --policy-arn "$POLICY_ARN"
done

# 2d. Revoke assume-role sessions (STS tokens) — thêm explicit deny
cat > /tmp/deny-all-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*"
  }]
}
EOF

aws iam put-user-policy \
  --user-name "$USERNAME" \
  --policy-name "EmergencyDenyAll" \
  --policy-document file:///tmp/deny-all-policy.json

# === BƯỚC 3: INVESTIGATION — Điều Tra ===
# 3a. Xem lịch sử hoạt động qua CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue="$USERNAME" \
  --start-time "$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --query 'Events[].{Time:EventTime,Event:EventName,IP:CloudTrailEvent}' \
  --output table

# 3b. Tìm resources được tạo bởi user này
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue="$USERNAME" \
  --query 'Events[?EventName==`CreateUser` || EventName==`CreateAccessKey` || 
           EventName==`CreateRole`].{Time:EventTime,Event:EventName}'

# 3c. Mở Detective investigation (qua console hoặc API)
aws detective start-investigation \
  --graph-arn arn:aws:detective:... \
  --entity-arn "arn:aws:iam::$ACCOUNT_ID:user/$USERNAME" \
  --scope-start-time "$(date -u -d '48 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --scope-end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')"

# === BƯỚC 4: ERADICATION — Loại Bỏ Persistence ===
# 4a. Tìm và xóa backdoor access keys mà attacker tạo
aws iam list-access-keys --user-name "$USERNAME"
# Xóa key mà attacker đã tạo thêm

# 4b. Kiểm tra users mới được tạo trong cùng thời gian
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser \
  --start-time "$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')"

# 4c. Kiểm tra roles mới với trust policy bất thường
aws iam list-roles --query 'Roles[].{Name:RoleName,Created:CreateDate}' \
  --output table

# === BƯỚC 5: RECOVERY ===
# 5a. Tạo access key mới cho user hợp lệ (nếu cần)
aws iam create-access-key --user-name "$USERNAME"

# 5b. Update Secrets Manager với credentials mới
aws secretsmanager rotate-secret --secret-id my-app-credentials

# 5c. Xóa EmergencyDenyAll policy sau khi fix xong
aws iam delete-user-policy \
  --user-name "$USERNAME" \
  --policy-name "EmergencyDenyAll"
```

---

## Runbook: EC2 Instance Compromise — EC2 Bị Xâm Phạm

### Dấu Hiệu Nhận Biết

- GuardDuty: `CryptoCurrency:EC2/BitcoinTool.B!DNS`
- GuardDuty: `Backdoor:EC2/C&CActivity.B!DNS`
- GuardDuty: `UnauthorizedAccess:EC2/TorIPCaller`
- Tăng đột biến CPU/network không giải thích được
- Kết nối outbound đến IP lạ trong VPC Flow Logs

### Quy Trình Xử Lý

```bash
INSTANCE_ID="i-0123456789abcdef0"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"

# === BƯỚC 1: TRIAGE ===
# Lấy thông tin instance
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{
    State:State.Name,
    IP:PublicIpAddress,
    PrivateIP:PrivateIpAddress,
    AMI:ImageId,
    Type:InstanceType,
    SubnetId:SubnetId,
    SecurityGroups:SecurityGroups[].GroupId,
    Tags:Tags
  }'

# === BƯỚC 2: FORENSIC PRESERVATION — Bảo Toàn Bằng Chứng ===
# Bước này PHẢI làm TRƯỚC khi isolate để có evidence

# 2a. Tạo snapshot của tất cả EBS volumes
VOLUMES=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[].Ebs.VolumeId' \
  --output text)

for VOLUME_ID in $VOLUMES; do
  SNAPSHOT_ID=$(aws ec2 create-snapshot \
    --volume-id "$VOLUME_ID" \
    --description "Forensic snapshot - Incident $(date +%Y%m%d-%H%M) - $INSTANCE_ID" \
    --tag-specifications "ResourceType=snapshot,Tags=[
      {Key=Purpose,Value=ForensicEvidence},
      {Key=IncidentDate,Value=$(date +%Y-%m-%d)},
      {Key=OriginalInstance,Value=$INSTANCE_ID}
    ]" \
    --query 'SnapshotId' --output text)
  echo "Created snapshot: $SNAPSHOT_ID for volume $VOLUME_ID"
done

# 2b. Capture memory (nếu SSM Agent còn hoạt động)
aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=[
    "sudo apt-get install -y lime-forensics-dkms || true",
    "sudo insmod /lib/modules/$(uname -r)/lime.ko path=/tmp/memory.lime format=lime",
    "aws s3 cp /tmp/memory.lime s3://forensic-evidence-bucket/$(date +%Y%m%d)/memory-$INSTANCE_ID.lime"
  ]'

# === BƯỚC 3: CONTAINMENT — Isolation ===
# 3a. Tạo isolation security group (không có rules)
ISOLATION_SG=$(aws ec2 create-security-group \
  --group-name "isolation-sg-$(date +%Y%m%d-%H%M)" \
  --description "Isolation SG - Incident $(date +%Y%m%d) - $INSTANCE_ID" \
  --vpc-id <VPC_ID> \
  --query 'GroupId' --output text)

# 3b. Gán isolation SG (thay thế tất cả SGs hiện tại)
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --groups "$ISOLATION_SG"

# 3c. Tag instance với status bị compromised
aws ec2 create-tags \
  --resources "$INSTANCE_ID" \
  --tags \
    Key=SecurityStatus,Value=Compromised \
    Key=IncidentDate,Value="$(date +%Y-%m-%d)" \
    Key=IsolatedBy,Value=IncidentResponse

# 3d. Nếu instance có IAM role — thu hồi quyền của role
INSTANCE_PROFILE=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' \
  --output text)

if [ "$INSTANCE_PROFILE" != "None" ]; then
  echo "WARNING: Instance has IAM profile: $INSTANCE_PROFILE"
  echo "Consider revoking all sessions for this role"
  
  # Thêm inline deny policy vào role
  ROLE_NAME=$(aws iam get-instance-profile \
    --instance-profile-name "$(echo $INSTANCE_PROFILE | cut -d'/' -f2)" \
    --query 'InstanceProfile.Roles[0].RoleName' \
    --output text)
  
  aws iam put-role-policy \
    --role-name "$ROLE_NAME" \
    --policy-name "EmergencyDenyAll" \
    --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'
fi

# === BƯỚC 4: INVESTIGATION ===
# 4a. Phân tích VPC Flow Logs — tìm outbound connections
# Query Athena hoặc CloudWatch Logs Insights
cat > /tmp/flow-log-query.sql << 'EOF'
SELECT 
  srcaddr, dstaddr, dstport, protocol, bytes, packets, action,
  from_unixtime(start) as start_time
FROM vpc_flow_logs
WHERE srcaddr = '<PRIVATE_IP_OF_INSTANCE>'
  AND action = 'ACCEPT'
  AND start >= to_unixtime(current_timestamp - interval '24' hour)
ORDER BY bytes DESC
LIMIT 100;
EOF

# 4b. Tìm processes bất thường (nếu SSM còn truy cập)
aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=[
    "ps aux --sort=-%cpu | head -20",
    "netstat -tulnp",
    "ss -tulnp",
    "crontab -l",
    "ls -la /tmp/ /var/tmp/",
    "last -20",
    "who"
  ]'

# === BƯỚC 5: RECOVERY ===
# 5a. Terminate compromised instance
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"

# 5b. Launch replacement từ clean AMI
aws ec2 run-instances \
  --image-id ami-latest-clean \
  --instance-type t3.medium \
  --security-group-ids sg-original \
  --subnet-id subnet-original \
  --iam-instance-profile Name=OriginalProfile \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=replacement-$(date +%Y%m%d)}]"
```

---

## Runbook: S3 Data Breach — Rò Rỉ Dữ Liệu S3

### Dấu Hiệu Nhận Biết

- Macie: `SensitiveData:S3Object/Personal` trong bucket đột nhiên public
- GuardDuty: `Exfiltration:S3/ObjectRead.Unusual`
- CloudTrail: `GetObject` calls từ IP lạ hoặc với volume bất thường
- Config: Rule `s3-bucket-public-read-prohibited` NONCOMPLIANT

```bash
BUCKET_NAME="compromised-bucket"

# === BƯỚC 1: IMMEDIATE CONTAINMENT ===
# Bật Block Public Access ngay lập tức
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true

# Xóa bucket policy cho phép public access
aws s3api delete-bucket-policy --bucket "$BUCKET_NAME" 2>/dev/null || true

# === BƯỚC 2: ASSESS DATA EXPOSURE ===
# Xem CloudTrail — ai đã đọc gì trong 24 giờ qua
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue="$BUCKET_NAME" \
  --start-time "$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --query 'Events[?EventName==`GetObject`].{
    Time:EventTime,
    User:Username,
    IP:CloudTrailEvent
  }' \
  --output table

# === BƯỚC 3: ASSESS SENSITIVE DATA ===
# Chạy Macie classification job khẩn cấp
aws macie2 create-classification-job \
  --job-type ONE_TIME \
  --name "EmergencyScan-$(date +%Y%m%d)" \
  --s3-job-definition "{
    \"bucketDefinitions\": [{
      \"accountId\": \"$ACCOUNT_ID\",
      \"buckets\": [\"$BUCKET_NAME\"]
    }]
  }" \
  --managed-data-identifier-selector ALL

# === BƯỚC 4: BREACH NOTIFICATION ASSESSMENT ===
# Xác định:
# - Bao nhiêu records bị expose?
# - Loại dữ liệu gì? (PII → GDPR, PHI → HIPAA, card → PCI-DSS)
# - Thời gian expose là bao lâu?
# - Những IP nào đã download?
# → Notify Legal/Compliance team ngay
```

---

## Runbook: Cryptomining Detection — Phát Hiện Đào Tiền Mã Hóa

```bash
# GuardDuty finding: CryptoCurrency:EC2/BitcoinTool.B!DNS

INSTANCE_ID="i-0abc123"

# 1. Kiểm tra CPU usage bất thường
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
  --start-time "$(date -u -d '6 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Average

# 2. Kiểm tra network egress (data gửi ra ngoài)
# VPC Flow Logs query - tổng bytes gửi đi trong 24h

# 3. Isolate và terminate (chi phí thường là mục tiêu chính)
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --groups "sg-isolation"

# 4. Kiểm tra Security Group ban đầu — 
# cryptomining thường vào qua exposed port (SSH 22, RDP 3389, 8080...)
# → Tìm và đóng port bị exploit
```

---

## Runbook: Unauthorized Account Access — Truy Cập Tài Khoản Trái Phép

```bash
# Xử lý khi root account hoặc management account bị nghi ngờ compromise

# === IMMEDIATE ACTIONS (Hành Động Ngay Lập Tức) ===

# 1. Thay đổi root account password ngay (qua console)
# → Bắt buộc làm thủ công, không có API

# 2. Revoke tất cả root account access keys (nếu có)
aws iam delete-access-key \
  --access-key-id <ROOT_ACCESS_KEY_ID>

# 3. Liệt kê và disable tất cả IAM users không quen thuộc
aws iam list-users \
  --query 'Users[].{Name:UserName,Created:CreateDate}' \
  --output table

# 4. Kiểm tra SCPs và xem có bị thay đổi không
aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY

# 5. Kiểm tra CloudTrail — ai đã làm gì trong root account
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=root \
  --start-time "$(date -u -d '7 days ago' '+%Y-%m-%dT%H:%M:%SZ')"

# 6. Liên hệ AWS Support ngay nếu không thể regain access
# → Mở case với "Account Compromise" priority
```

---

## Forensic Preservation — Bảo Toàn Bằng Chứng

### Chain of Custody (Chuỗi Lưu Giữ Bằng Chứng)

Tất cả evidence phải được bảo toàn với:
1. **Integrity** — hash checksum (MD5/SHA256) của snapshot
2. **Immutability** — S3 Object Lock (WORM — Write Once Read Many) cho forensic bucket
3. **Access log** — ai đã truy cập evidence, khi nào

```bash
# Tạo forensic evidence bucket với WORM
aws s3api create-bucket \
  --bucket forensic-evidence-$(date +%Y%m%d) \
  --region us-east-1

aws s3api put-object-lock-configuration \
  --bucket forensic-evidence-$(date +%Y%m%d) \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Years": 7
      }
    }
  }'

# Hash snapshot sau khi tạo (verify integrity)
SNAPSHOT_ID="snap-0123456789abcdef0"
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[0].{ID:SnapshotId,State:State,Encrypted:Encrypted}'
```

---

## Automated Response Architecture

```
GuardDuty Finding (High Severity)
    │
    ▼
EventBridge Rule
    │
    ├── SNS → PagerDuty/Opsgenie alert (cảnh báo on-call)
    │
    ├── SQS → Ticketing system (Jira Service Desk)
    │
    └── Lambda: AutomatedResponder
            │
            ├── EC2 incident → IsolateEC2 state machine
            │       ├── Create EBS snapshots
            │       ├── Replace security groups
            │       ├── Tag as compromised
            │       └── Notify security team
            │
            ├── IAM incident → RevokeCredentials state machine
            │       ├── Disable access keys
            │       ├── Add deny-all inline policy
            │       ├── Create CloudTrail event
            │       └── Notify user's manager
            │
            └── S3 incident → QuarantineBucket state machine
                    ├── Enable block public access
                    ├── Remove bucket policy
                    ├── Enable access logging
                    └── Trigger Macie classification job
```

### Step Functions State Machine

```json
{
  "Comment": "EC2 Incident Response State Machine",
  "StartAt": "TakeForensicSnapshot",
  "States": {
    "TakeForensicSnapshot": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:CreateForensicSnapshot",
      "Next": "IsolateInstance",
      "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}]
    },
    "IsolateInstance": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:IsolateEC2Instance",
      "Next": "NotifyTeam"
    },
    "NotifyTeam": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:...:security-alerts",
        "Message.$": "States.Format('EC2 {} isolated. Investigation required.', $.instanceId)"
      },
      "Next": "WaitForAcknowledgment"
    },
    "WaitForAcknowledgment": {
      "Type": "Wait",
      "Seconds": 3600,
      "Next": "CheckResolution"
    },
    "CheckResolution": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.resolved",
        "BooleanEquals": true,
        "Next": "CleanupAndRecover"
      }],
      "Default": "EscalateIncident"
    },
    "EscalateIncident": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:EscalateToManagement",
      "End": true
    },
    "CleanupAndRecover": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:RecoverInstance",
      "End": true
    }
  }
}
```

---

## Post-Incident Activities — Hoạt Động Sau Sự Cố

### Incident Report Template (Mẫu Báo Cáo Sự Cố)

```markdown
## Incident Report — [INC-2026-001]

**Ngày:** 2026-05-16
**Severity:** P2 — High
**Trạng Thái:** Resolved
**Thời Gian Phản Hồi:** 45 phút (target: < 1 giờ)

### Tóm Tắt
EC2 instance i-0abc123 bị xâm phạm qua exposed SSH port và sử dụng 
để đào Bitcoin, gây tăng chi phí AWS ~$200/ngày.

### Timeline
- 10:15 UTC: GuardDuty phát hiện CryptoCurrency:EC2/BitcoinTool.B!DNS
- 10:18 UTC: PagerDuty alert → on-call engineer
- 10:22 UTC: EBS snapshot tạo xong (forensic evidence)
- 10:25 UTC: Instance isolated (SG thay thế)
- 10:45 UTC: Root cause xác định — SSH port 22 mở cho 0.0.0.0/0
- 11:00 UTC: Replacement instance deployed, service restored

### Root Cause (Nguyên Nhân Gốc Rễ)
Security Group cho phép SSH (port 22) từ 0.0.0.0/0 — vi phạm policy.
Developer đã mở tạm để debug và quên đóng lại.

### Impact (Ảnh Hưởng)
- Thời gian gián đoạn: 0 (isolated, không terminate ngay)
- Chi phí bổ sung ước tính: $47 (8 giờ cryptomining trước khi phát hiện)
- Dữ liệu bị lộ: Không

### Remediation (Khắc Phục)
1. ✅ Instance terminated, replacement deployed
2. ✅ Security Group locked down — không còn 0.0.0.0/0 trên SSH
3. ✅ Thêm Config rule: ec2-security-group-attached-to-eni-periodic
4. ✅ Bật AWS Config remediation tự động cho SSH public

### Lessons Learned (Bài Học Rút Ra)
1. Cần config rule tự động phát hiện và đóng SSH public ngay
2. Thêm SCP: deny mở port 22/3389 cho 0.0.0.0/0
3. Review quarterly — security group audit
```

### Lessons Learned Meeting Agenda

```
1. Timeline review — 5 phút
2. Root cause analysis — 10 phút
3. Was detection timely? — 5 phút
4. Was response effective? — 10 phút
5. What should change? — 15 phút
   - Technical controls
   - Process improvements
   - Training needs
6. Action items với owner và deadline — 10 phút
```

---

## Câu Hỏi Phỏng Vấn

**Q: Nếu GuardDuty báo EC2 bị compromise, làm gì đầu tiên?**
A: Thứ tự đúng:
1. **Bảo toàn bằng chứng** — snapshot EBS ngay TRƯỚC khi isolate (evidence bị mất nếu terminate trước)
2. **Isolate** — thay Security Group thành isolation SG (không terminate vội)
3. **Revoke IAM role** — nếu EC2 có IAM role, thêm deny-all inline policy
4. **Investigate** — dùng Detective, phân tích VPC Flow Logs, CloudTrail
5. **Eradicate** — xóa malware/backdoor, tìm root cause
6. **Recover** — launch replacement từ clean AMI
7. **Document** — viết incident report, lessons learned

**Q: Containment khác Eradication như thế nào?**
A: Containment (Ngăn Chặn) là ngăn thiệt hại lan rộng ngay lập tức — isolate instance, disable credentials, restrict network. Eradication (Loại Bỏ) là xóa hoàn toàn sự hiện diện của attacker — xóa malware, close backdoors, patch vulnerabilities, xóa rogue users. Thứ tự: Containment trước → Investigation → Eradication → Recovery.

**Q: Tại sao cần tạo EBS snapshot trước khi terminate EC2 bị compromise?**
A: Snapshot là forensic evidence (bằng chứng pháp lý kỹ thuật số) — chứa filesystem, processes (memory dump nếu capture kịp), logs, malware samples. Cần thiết cho: (1) phân tích root cause, (2) thủ tục pháp lý nếu có, (3) improve defenses dựa trên TTPs (Tactics, Techniques, Procedures — Chiến Thuật, Kỹ Thuật, Quy Trình) của attacker. Terminate trước = mất evidence vĩnh viễn.

**Q: SLA phản hồi sự cố theo severity nên là bao nhiêu?**
A: Không có con số chuẩn — phụ thuộc vào tổ chức. Nhưng industry common:
- P1 Critical: < 15 phút để acknowledge, < 1 giờ để contain
- P2 High: < 1 giờ để acknowledge, < 4 giờ để contain
- P3 Medium: < 4 giờ để acknowledge, < 24 giờ để investigate
- P4 Low: < 24 giờ để acknowledge, < 1 tuần để remediate

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
