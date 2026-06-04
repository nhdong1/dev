# IAM Best Practices — Checklist Bảo Mật IAM

> Tổng hợp các thực hành bảo mật IAM theo **AWS Well-Architected Framework** (Khung Kiến Trúc Tốt AWS), CIS AWS Benchmarks (Điểm Chuẩn CIS cho AWS), và kinh nghiệm vận hành thực tế từ sự cố bảo mật lớn.

---

## 1. Root Account Security (Bảo Mật Tài Khoản Gốc)

### Nguyên Tắc

Root account có quyền **tuyệt đối và không thể thu hồi** — không thể bị giới hạn bởi SCPs, Permission Boundaries hay bất kỳ cơ chế IAM nào.

### Checklist

- [ ] **Không dùng root cho thao tác hàng ngày** — tạo IAM admin user ngay sau khi có account mới
- [ ] **Bật MFA cho root account** — dùng hardware MFA key (YubiKey) nếu có thể
- [ ] **Không tạo access key cho root** — nếu đã có, xóa ngay
- [ ] **Lưu root credentials ở nơi an toàn** — password manager dạng offline hoặc vault vật lý
- [ ] **Chỉ dùng root cho các tác vụ đặc biệt:**
  - Thay đổi account settings (email, payment method)
  - Đóng AWS account
  - Khôi phục khi mất quyền truy cập IAM
  - Đăng ký marketplace seller
  - Thay đổi support plan

```bash
# Kiểm tra root account có access key không
aws iam get-account-summary \
  --query 'SummaryMap.AccountAccessKeysPresent'
# Kết quả 0 = an toàn, 1 = nguy hiểm
```

---

## 2. Least Privilege Principle (Nguyên Tắc Đặc Quyền Tối Thiểu)

### Nguyên Tắc

Cấp đúng quyền cần thiết, không hơn không kém. Bắt đầu với quyền tối thiểu và mở rộng dần dần khi có nhu cầu rõ ràng.

### Thực Hành

**Bắt đầu với deny tất cả, thêm dần:**

```json
// Sai: Bắt đầu với AdministratorAccess rồi cắt bớt
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// Đúng: Chỉ cấp đúng những gì cần
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:Query"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
}
```

**Giới hạn Resource cụ thể — không dùng `*` khi không cần:**

```json
// Sai: Truy cập tất cả S3 buckets
"Resource": "arn:aws:s3:::*"

// Đúng: Chỉ bucket của ứng dụng
"Resource": [
  "arn:aws:s3:::my-app-prod-bucket",
  "arn:aws:s3:::my-app-prod-bucket/*"
]
```

**Dùng IAM Access Analyzer để phát hiện unused permissions:**

```bash
# Kích hoạt Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name MyAnalyzer \
  --type ACCOUNT

# Xem recommendations về unused permissions
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/MyAnalyzer
```

### Checklist

- [ ] Không dùng `Action: "*"` trong production policies
- [ ] Không dùng `Resource: "*"` trừ khi service không hỗ trợ resource-level permissions
- [ ] Dùng IAM Access Analyzer để phát hiện quyền chưa dùng
- [ ] Review và giảm quyền định kỳ (ít nhất 6 tháng một lần)
- [ ] Áp dụng IAM conditions để kiểm soát context (region, IP, MFA...)

---

## 3. MFA Enforcement (Bắt Buộc Xác Thực Đa Yếu Tố)

### Checklist

- [ ] Bật MFA cho root account (ưu tiên hardware key)
- [ ] Bắt buộc MFA cho tất cả IAM users có quyền cao
- [ ] Tạo policy buộc user bật MFA trước khi dùng dịch vụ

**Policy buộc tự bật MFA (cho user mới):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSelfManagedMFA",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "BlockAllWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

**Kiểm tra MFA status của tất cả users:**

```bash
aws iam generate-credential-report
aws iam get-credential-report \
  --query 'Content' --output text | base64 --decode \
  | grep -v ",true," | cut -d',' -f1
# Output: users chưa bật MFA
```

---

## 4. Access Key Management (Quản Lý Khóa Truy Cập)

### Nguyên Tắc

Long-term access keys là rủi ro bảo mật nghiêm trọng — tránh dùng khi có thể thay thế bằng IAM Roles.

### Checklist

- [ ] **Không nhúng access key vào code** — dùng environment variables hoặc Secrets Manager
- [ ] **Không commit access key vào Git** — cài pre-commit hook kiểm tra
- [ ] **EC2/Lambda/ECS phải dùng Role** — không dùng access key trong application
- [ ] **Xoay vòng access key ≤90 ngày** — hoặc tự động hóa bằng Lambda
- [ ] **Xóa access key không dùng >90 ngày** — dùng Credential Report để phát hiện
- [ ] **Mỗi user tối đa 2 key** — để hỗ trợ rotation không gián đoạn
- [ ] **Không tạo access key cho root account**

**Kiểm tra key rotation:**

```bash
# Liệt kê tất cả access keys và ngày tạo
aws iam list-users --query 'Users[].UserName' --output text | \
  tr '\t' '\n' | while read user; do
    aws iam list-access-keys --user-name "$user" \
      --query 'AccessKeyMetadata[].{User:UserName,KeyId:AccessKeyId,Created:CreateDate,Status:Status}' \
      --output table
  done
```

**Quy trình rotation không gián đoạn:**

```
1. Tạo access key mới (key thứ 2)
2. Cập nhật tất cả ứng dụng dùng key mới
3. Test kỹ lưỡng
4. Vô hiệu hóa (Inactive) key cũ — chưa xóa
5. Chờ 24-48 giờ theo dõi lỗi
6. Xóa key cũ
```

---

## 5. Role Design (Thiết Kế Role)

### Nguyên Tắc

Ưu tiên role cho mọi workload tự động. Thiết kế role theo chức năng, không theo người dùng.

### Checklist

- [ ] **Mọi EC2/Lambda/ECS phải có dedicated role** — không share role giữa nhiều services
- [ ] **Role name phải mô tả rõ chức năng:** `lambda-order-processor-role` thay vì `my-role`
- [ ] **Dùng `iam:PassRole` với điều kiện service cụ thể:**

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::*:role/lambda-*",
  "Condition": {
    "StringEquals": {
      "iam:PassedToService": "lambda.amazonaws.com"
    }
  }
}
```

- [ ] **Trust Policy phải cụ thể** — không dùng `Principal: "*"` trong trust policy
- [ ] **Đặt `MaxSessionDuration` phù hợp** — mặc định 1 giờ, tối đa 12 giờ cho human access

**Template Role cho Lambda:**

```bash
# Trust policy
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Tạo role
aws iam create-role \
  --role-name lambda-order-processor-role \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for Order Processor Lambda function"

# Thêm basic Lambda logging
aws iam attach-role-policy \
  --role-name lambda-order-processor-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

---

## 6. Policy Management (Quản Lý Policy)

### Checklist

- [ ] **Ưu tiên Customer Managed Policy hơn Inline Policy** — tái sử dụng và quản lý dễ hơn
- [ ] **Đặt tên policy mô tả rõ:** `S3-AppBucket-ReadWrite` thay vì `Policy1`
- [ ] **Thêm `Sid` vào mỗi statement** — dễ debug và audit
- [ ] **Luôn dùng `"Version": "2012-10-17"`** — version cũ `2008-10-17` thiếu tính năng
- [ ] **Test policy với IAM Policy Simulator** trước khi áp dụng production
- [ ] **Kiểm tra policy JSON hợp lệ trước khi deploy:**

```bash
aws iam validate-role-policy \
  --role-name MyRole \
  --policy-document file://policy.json
```

**Policy Simulator — test quyền trước:**

```bash
# Test xem role có quyền s3:GetObject không
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/test.txt
```

---

## 7. Tagging Strategy (Chiến Lược Gắn Tag)

### Checklist

- [ ] **Tag tất cả IAM resources:** users, roles, policies
- [ ] **Tag chuẩn bao gồm:**

```
Owner       : team hoặc person
Project     : tên dự án hoặc service
Environment : dev/staging/production
CostCenter  : trung tâm chi phí
ManagedBy   : terraform/cloudformation/manual
```

- [ ] **Dùng tags để ABAC** (Attribute-Based Access Control):

```json
{
  "Sid": "AllowAccessByProjectTag",
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}"
    }
  }
}
```

- [ ] **SCP bắt buộc tag khi tạo role:**

```json
{
  "Sid": "RequireTagOnCreateRole",
  "Effect": "Deny",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/Owner": "true"
    }
  }
}
```

---

## 8. Auditing & Monitoring (Kiểm Toán & Giám Sát)

### Checklist

- [ ] **Bật CloudTrail** trong tất cả regions, kể cả management account
- [ ] **Bật IAM access logging** — mọi thay đổi IAM đều xuất hiện trong CloudTrail
- [ ] **Thiết lập CloudWatch Alarms cho:**

| Sự Kiện | Mức Độ Ưu Tiên |
|---|---|
| Root account login | CRITICAL |
| Console login không có MFA | HIGH |
| Tạo/xóa IAM user/role | HIGH |
| Thay đổi policy quan trọng | HIGH |
| `sts:AssumeRole` từ IP lạ | MEDIUM |
| Nhiều Access Denied liên tiếp | MEDIUM |

```bash
# Alarm khi root login
aws cloudwatch put-metric-alarm \
  --alarm-name RootLoginAlert \
  --alarm-description "Root account login detected" \
  --metric-name RootAccountLogin \
  --namespace CloudTrailMetrics \
  --statistic Sum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:SecurityAlerts
```

- [ ] **Chạy Credential Report định kỳ** — xem user nào không dùng credentials:

```bash
aws iam generate-credential-report && sleep 5
aws iam get-credential-report \
  --query 'Content' --output text | base64 --decode > credential-report.csv
```

- [ ] **Kích hoạt IAM Access Analyzer** — phát hiện external access

---

## 9. Cross-Account Security (Bảo Mật Liên Tài Khoản)

### Checklist

- [ ] **External ID cho cross-account roles** — chống Confused Deputy attack:

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::PARTNER-ACCOUNT:root"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-secret-id-12345"
    }
  }
}
```

- [ ] **Bắt buộc MFA cho cross-account assume:**

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

- [ ] **Giới hạn `aws:PrincipalOrgID`** để chỉ cho phép trong Org:

```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-xxxxxxxxxx"
  }
}
```

---

## 10. Dọn Dẹp Định Kỳ (IAM Hygiene)

### Checklist Hàng Tháng

- [ ] Xóa IAM user không đăng nhập >90 ngày
- [ ] Xóa access key không dùng >90 ngày
- [ ] Xóa role không có `LastUsed` date trong 90 ngày
- [ ] Xóa inline policy không còn cần thiết
- [ ] Review group membership — remove users không còn trong team

```bash
# Tìm roles không được dùng trong 90 ngày
aws iam get-account-authorization-details \
  --filter Role \
  --query 'RoleDetailList[?RoleLastUsed.LastUsedDate<`2026-02-15`].{Name:RoleName,LastUsed:RoleLastUsed.LastUsedDate}' \
  --output table

# Tìm users không login trong 90 ngày
aws iam get-credential-report \
  --query 'Content' --output text | base64 --decode | \
  awk -F',' 'NR>1 && $5!="N/A" && $5<"2026-02-15" {print $1, $5}'
```

---

## Tóm Tắt: Checklist Nhanh Pre-Deployment

```
IAM Security Checklist — Trước Khi Deploy Production:

Root Account:
  ☐ MFA bật, không có access key
  ☐ Không dùng root cho thao tác hàng ngày

IAM Users:
  ☐ Mỗi người có user riêng
  ☐ MFA bắt buộc cho tất cả
  ☐ Access key rotation ≤90 ngày
  ☐ Không có access key > 90 ngày không dùng

IAM Roles:
  ☐ EC2/Lambda/ECS dùng role, không dùng access key
  ☐ Least privilege — chỉ cấp quyền cần thiết
  ☐ Trust policy cụ thể (không dùng *)
  ☐ Tag đầy đủ (Owner, Project, Environment)

Policies:
  ☐ Không có wildcard Action/Resource không cần thiết
  ☐ Test với IAM Policy Simulator
  ☐ Có Condition kiểm soát context (MFA, IP, Region)

Monitoring:
  ☐ CloudTrail bật tất cả regions
  ☐ Alarm cho root login và IAM changes
  ☐ Access Analyzer kích hoạt
  ☐ Credential Report review định kỳ
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Top 5 IAM mistakes bạn thấy trong thực tế là gì?**

> 1. **Hardcoded access keys** trong code/env vars thay vì dùng Instance Profile
> 2. **Wildcard permissions** (`Action: *`, `Resource: *`) trong production cho tiện
> 3. **Shared IAM users** — nhiều người dùng chung một access key
> 4. **Root account không có MFA** và được dùng thường xuyên
> 5. **Không audit định kỳ** — roles/users zombie tồn tại hàng năm sau khi nhân viên nghỉ

**Q: Làm sao audit IAM hiệu quả trong môi trường lớn (500+ users/roles)?**

> Kết hợp nhiều công cụ: (1) IAM Access Analyzer tự động phát hiện external access và unused permissions; (2) Credential Report hàng tuần để phát hiện stale credentials; (3) CloudTrail + Athena để query who-did-what trong 90 ngày; (4) Prowler hoặc AWS Security Hub với CIS Benchmark để tự động đánh giá IAM configuration; (5) Định kỳ chạy `iam-lint` hoặc `Parliament` trên policy JSON trước khi merge vào IaC.

**Q: Confused Deputy Attack là gì? Cách phòng chống?**

> Confused Deputy là khi attacker lợi dụng third-party service có quyền assume role trong account của bạn để truy cập tài nguyên không được phép. Ví dụ: attacker cung cấp ARN của role bạn cho service tư vấn bên thứ 3 — service đó assume role thay mặt attacker. **Phòng chống:** Thêm `sts:ExternalId` condition vào trust policy — chỉ principal biết External ID mới có thể assume role. External ID là shared secret giữa bạn và third-party, không phải thông tin public.
