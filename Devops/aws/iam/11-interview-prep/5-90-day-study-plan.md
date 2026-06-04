# Kế Hoạch Học Tập 90 Ngày — AWS Security

> Lộ trình có cấu trúc để đạt trình độ phỏng vấn AWS Security Engineer trong 90 ngày — từ nền tảng đến thực chiến.

---

## 📑 Mục Lục

1. [Tổng Quan Lộ Trình](#tổng-quan-lộ-trình)
2. [Giai Đoạn 1: Nền Tảng (Ngày 1–30)](#giai-đoạn-1-nền-tảng-ngày-130)
3. [Giai Đoạn 2: Chuyên Sâu (Ngày 31–60)](#giai-đoạn-2-chuyên-sâu-ngày-3160)
4. [Giai Đoạn 3: Thực Chiến (Ngày 61–90)](#giai-đoạn-3-thực-chiến-ngày-6190)
5. [Lab Setup Hướng Dẫn](#lab-setup-hướng-dẫn)
6. [Tài Nguyên Học Tập](#tài-nguyên-học-tập)
7. [Theo Dõi Tiến Độ](#theo-dõi-tiến-độ)

---

## Tổng Quan Lộ Trình

```
GIAI ĐOẠN 1 (Ngày 1–30): Nền Tảng
├── IAM fundamentals + Encryption
├── Setup AWS lab account
└── Hiểu Shared Responsibility Model

GIAI ĐOẠN 2 (Ngày 31–60): Chuyên Sâu
├── Threat detection + Monitoring
├── Network security
├── Multi-account + Compliance
└── Xây dựng portfolio projects

GIAI ĐOẠN 3 (Ngày 61–90): Thực Chiến
├── Mock interviews
├── STAR stories
├── System design practice
└── Certification preparation
```

**Thời gian học mỗi ngày:** 1.5–2 giờ (học viên toàn thời gian: 3–4 giờ)

---

## Giai Đoạn 1: Nền Tảng (Ngày 1–30)

### Tuần 1 (Ngày 1–7): IAM Fundamentals

**Mục tiêu:** Hiểu và viết được IAM policy từ đầu không cần tài liệu.

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 1 | IAM Users, Groups, Roles overview | `01-iam-fundamentals/README.md` | Tạo IAM user, group, gán policy |
| 2 | Policy types: Identity vs Resource vs SCP | `01-iam-fundamentals/2-policy-types.md` | Viết S3 bucket policy từ đầu |
| 3 | IAM Conditions — keys và operators | `01-iam-fundamentals/3-iam-conditions.md` | Policy với IP condition, MFA condition |
| 4 | Permission Boundaries | `01-iam-fundamentals/4-permission-boundaries.md` | Create role bị giới hạn bởi boundary |
| 5 | IAM Best Practices | `01-iam-fundamentals/5-iam-best-practices.md` | Audit IAM configuration |
| 6 | **Review + Lab ngày**: Cross-account role | `01-iam-fundamentals/1-users-groups-roles.md` | AssumeRole giữa 2 accounts |
| 7 | **Quiz + Flashcards**: Ôn lại tuần 1 | - | IAM Policy Simulator walkthrough |

**Quiz Tuần 1:**
- [ ] Giải thích IAM evaluation order từ đầu không nhìn tài liệu
- [ ] Viết policy cho phép EC2 đọc S3 bucket cụ thể
- [ ] Giải thích Explicit Deny vs Implicit Deny
- [ ] Setup cross-account role thành công trong lab

---

### Tuần 2 (Ngày 8–14): Encryption & Key Management

**Mục tiêu:** Hiểu envelope encryption, biết khi nào dùng KMS vs CloudHSM, thiết lập KMS trong lab.

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 8 | KMS key types: CMK, AWS-managed, data keys | `04-encryption-kms/1-kms-key-types.md` | Tạo CMK, mã hóa/giải mã file |
| 9 | Envelope Encryption — DEK lifecycle | `04-encryption-kms/2-envelope-encryption.md` | Encrypt S3 object với KMS |
| 10 | Key Policies vs IAM Policies | `04-encryption-kms/3-key-policies.md` | Viết key policy từ đầu |
| 11 | KMS Grants — temporary delegation | `04-encryption-kms/4-kms-grants.md` | Create grant cho Lambda |
| 12 | CloudHSM — FIPS 140-2 Level 3 | `04-encryption-kms/5-cloudhsm.md` | Đọc architecture (không cần lab — đắt tiền) |
| 13 | S3 Encryption Options | `04-encryption-kms/6-s3-encryption-options.md` | So sánh SSE-S3 vs SSE-KMS |
| 14 | **Review + Project**: Encrypt RDS với CMK | - | Lab: encrypt RDS, verify với CloudTrail |

**Quiz Tuần 2:**
- [ ] Giải thích envelope encryption step-by-step
- [ ] Khi nào dùng CloudHSM thay vì KMS?
- [ ] Viết key policy cho phép Lambda decrypt nhưng không create/delete key

---

### Tuần 3 (Ngày 15–21): Secrets & Identity Federation

**Mục tiêu:** Biết quản lý credentials an toàn, setup SSO cho multi-account.

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 15 | Secrets Manager — store, rotate, retrieve | `05-secrets-certificates/1-secrets-manager.md` | Lưu DB password, test rotation |
| 16 | Parameter Store — hierarchy, SecureString | `05-secrets-certificates/2-parameter-store.md` | Tạo parameter hierarchy `/app/prod/` |
| 17 | Secrets Manager vs Parameter Store | `05-secrets-certificates/3-secrets-vs-parameter.md` | So sánh chi phí + use cases |
| 18 | ACM certificates | `05-secrets-certificates/4-acm-certificates.md` | Request certificate, attach to ALB |
| 19 | IAM Identity Center (SSO) | `02-identity-federation/1-iam-identity-center.md` | Setup SSO với 2 accounts |
| 20 | SAML và OIDC Federation | `02-identity-federation/2-saml-federation.md` | Setup GitHub OIDC cho Actions |
| 21 | **Review + Lab ngày**: Full SSO setup | - | Permission sets, account assignment |

**Quiz Tuần 3:**
- [ ] Khi nào dùng Secrets Manager, khi nào dùng Parameter Store?
- [ ] Tại sao OIDC federation an toàn hơn access key cho CI/CD?
- [ ] Lab: Xoay vòng RDS password tự động với Secrets Manager

---

### Tuần 4 (Ngày 22–30): Organizations & Review Giai Đoạn 1

**Mục tiêu:** Hiểu multi-account design, viết SCP, và ôn tập toàn bộ giai đoạn 1.

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 22 | Organizations — OU hierarchy | `03-organizations/1-organizations-setup.md` | Tạo OU, move account |
| 23 | SCPs — syntax, examples, common mistakes | `03-organizations/2-service-control-policies.md` | Viết SCP chặn region, protect CloudTrail |
| 24 | Control Tower — landing zone, guardrails | `03-organizations/3-control-tower.md` | Nghiên cứu architecture (không cần lab) |
| 25 | Account Vending — tự động tạo account | `03-organizations/4-account-vending.md` | Đọc AFT architecture |
| 26 | **Comprehensive Lab**: Multi-account setup | - | 3 accounts: management, log-archive, workload |
| 27–28 | **Review toàn bộ Giai Đoạn 1** | Tất cả modules 1–3, 4–5 | Practice quiz 30 câu hỏi |
| 29 | **Mock Quiz Giai Đoạn 1**: Viết 5 IAM policies | - | Không dùng tài liệu |
| 30 | **Reflection**: Ghi notes điểm yếu cần ôn thêm | - | Cập nhật tracking checklist |

---

## Giai Đoạn 2: Chuyên Sâu (Ngày 31–60)

### Tuần 5 (Ngày 31–37): Network Security

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 31 | Security Groups — stateful, best practices | `06-network-security/1-security-groups.md` | Thiết kế SG cho 3-tier app |
| 32 | NACLs — stateless, subnet protection | `06-network-security/2-nacls.md` | So sánh SG vs NACL |
| 33 | VPC Endpoints — Gateway vs Interface | `06-network-security/3-vpc-endpoints.md` | Setup S3 Gateway Endpoint |
| 34 | PrivateLink — private service connectivity | `06-network-security/4-privatelink.md` | Endpoint + endpoint service |
| 35 | WAF — rules, managed groups | `06-network-security/5-waf-setup.md` | Block SQLi, rate limiting |
| 36 | Shield DDoS + Network Firewall | `06-network-security/6-shield-ddos.md`, `7-network-firewall.md` | Review architecture |
| 37 | **Lab Day**: Design VPC security architecture | - | Diagram + implementation |

---

### Tuần 6 (Ngày 38–44): Monitoring & Auditing

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 38 | CloudTrail setup, S3 integrity, Lake | `07-monitoring-auditing/1-cloudtrail-setup.md` | Org trail, log file validation |
| 39 | CloudTrail analysis — Athena queries | `07-monitoring-auditing/2-cloudtrail-analysis.md` | Query: root usage, failed logins |
| 40 | AWS Config rules — managed, custom | `07-monitoring-auditing/3-aws-config-rules.md` | Enable rules, auto-remediation |
| 41 | Access Analyzer — external access detection | `07-monitoring-auditing/4-access-analyzer.md` | Analyze S3, role policies |
| 42 | Security Dashboard — CloudWatch | `07-monitoring-auditing/5-security-dashboard.md` | Build security metrics dashboard |
| 43 | **Lab Day**: Build security audit pipeline | - | CloudTrail → Athena → dashboard |
| 44 | **Review**: CloudTrail vs Config distinction | - | 10 scenario questions |

---

### Tuần 7 (Ngày 45–51): Threat Detection

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 45 | GuardDuty — finding types, severity | `08-threat-detection/1-guardduty.md` | Enable, review sample findings |
| 46 | Security Hub — ASFF, standards | `08-threat-detection/2-security-hub.md` | Enable standards, custom insights |
| 47 | Inspector — EC2/ECR/Lambda scanning | `08-threat-detection/3-inspector.md` | Scan EC2, review CVEs |
| 48 | Macie — S3 data classification | `08-threat-detection/4-macie.md` | Scan bucket, find PII |
| 49 | Detective — graph investigation | `08-threat-detection/5-detective.md` | Investigate sample finding |
| 50 | Incident Response Runbook | `08-threat-detection/6-incident-response.md` | Practice EC2 isolation steps |
| 51 | **Lab Day**: End-to-end incident simulation | - | GuardDuty finding → isolate → investigate |

---

### Tuần 8 (Ngày 52–58): Compliance & Advanced Topics

| Ngày | Chủ Đề | Tài Liệu | Lab |
|---|---|---|---|
| 52 | Audit Manager — evidence collection | `09-compliance-governance/1-audit-manager.md` | Setup framework |
| 53 | Conformance Packs — CIS, PCI, HIPAA | `09-compliance-governance/2-conformance-packs.md` | Deploy CIS pack |
| 54 | Firewall Manager — centralized mgmt | `09-compliance-governance/3-firewall-manager.md` | Review architecture |
| 55 | PCI-DSS trên AWS | `09-compliance-governance/4-pci-dss-aws.md` | Map requirements → controls |
| 56 | ABAC — tag-based access | `10-advanced/1-abac.md` | Implement tag-based S3 policy |
| 57 | Zero Trust Architecture | `10-advanced/2-zero-trust.md` | Design diagram |
| 58 | Data Perimeter — RCPs, SCPs | `10-advanced/3-data-perimeter.md` | Review policy examples |

---

### Ngày 59–60: Portfolio Project

**Project: "Secure Startup Infrastructure"**

Xây dựng kiến trúc AWS an toàn cho một startup giả định:

```
Deliverables:
├── Multi-account structure (3 accounts: management, prod, dev)
├── IAM Identity Center với 3 permission sets
├── CloudTrail organization trail + S3 log archive
├── GuardDuty + Security Hub enabled và configured
├── Secrets Manager cho DB credentials với rotation
├── S3 buckets với SSE-KMS, Block Public Access
├── 3 SCPs (region restriction, CloudTrail protection, encryption enforcement)
└── Security dashboard với key metrics

Documentation:
├── Architecture diagram
├── Security decisions và trade-offs
└── Incident response runbook
```

---

## Giai Đoạn 3: Thực Chiến (Ngày 61–90)

### Tuần 9 (Ngày 61–67): Interview Prep — Technical

| Ngày | Activity | Mục Tiêu |
|---|---|---|
| 61 | Đọc `1-top-questions.md` + tự trả lời không nhìn đáp án | Pass rate > 80% |
| 62 | IAM troubleshooting drills — 10 debug scenarios | Debug trong < 5 phút |
| 63 | Write IAM policies từ đầu — 5 complex scenarios | Không tài liệu, chính xác |
| 64 | System design practice: Multi-account architecture | Trình bày 20 phút |
| 65 | System design practice: PCI-DSS fintech | Trình bày 20 phút |
| 66 | Mock Q&A với đồng nghiệp (hoặc tự record) | Feedback từ người khác |
| 67 | Review và fix điểm yếu từ mock | Targeted review |

---

### Tuần 10 (Ngày 68–74): Interview Prep — Behavioral

| Ngày | Activity |
|---|---|
| 68 | Viết đầy đủ 3 STAR stories từ kinh nghiệm thực tế |
| 69 | Luyện kể Story 1 + 2: record, nghe lại, refine |
| 70 | Luyện kể Story 3 + STAR cho behavioral questions |
| 71 | Practice "elevator pitch" về bản thân (2 phút) |
| 72 | Chuẩn bị 10 câu hỏi thông minh để hỏi lại interviewer |
| 73 | Full mock interview với friend (technical + behavioral) |
| 74 | Debrief và cải thiện |

---

### Tuần 11 (Ngày 75–81): Certification Sprint (AWS Security Specialty)

**AWS Certified Security — Specialty (SCS-C02)**

Domain distribution:
```
Domain 1: Threat Detection & Incident Response (14%)
Domain 2: Security Logging & Monitoring (18%)
Domain 3: Infrastructure Security (20%)
Domain 4: Identity & Access Management (16%)
Domain 5: Data Protection (18%)
Domain 6: Management & Security Governance (14%)
```

| Ngày | Activity |
|---|---|
| 75 | Practice exam 1: 65 câu (timed) — identify weak areas |
| 76 | Review incorrect answers, ôn lại các service yếu |
| 77 | Practice exam 2: 65 câu (timed) |
| 78 | Review + deep dive 2 domain yếu nhất |
| 79 | Practice exam 3: 65 câu (timed) |
| 80 | Final review: cheatsheet tất cả services |
| 81 | Rest day — không study |

**Target score:** ≥ 800/1000 trước khi đăng ký thi thật.

---

### Tuần 12 (Ngày 82–90): Final Preparation

| Ngày | Activity |
|---|---|
| 82 | Full mock interview #2 — cải thiện từ round 1 |
| 83 | Review toàn bộ `11-interview-prep/` một lần |
| 84 | Ôn lại STAR stories — version ngắn và version dài |
| 85 | IAM policy sprint: viết 10 policies không tài liệu |
| 86 | System design sprint: 2 bài toán trong 40 phút |
| 87 | Rest + Light review |
| 88 | Chuẩn bị logistics phỏng vấn: môi trường, setup, mental |
| 89 | Final review: top 10 concepts quan trọng nhất |
| 90 | **Interview Day** hoặc certification exam |

---

## Lab Setup Hướng Dẫn

### AWS Account Setup

```bash
# Tạo 2–3 accounts free tier cho lab:
# Account 1: Management / Lab-Main
# Account 2: Workload (để luyện cross-account)
# Account 3: Security tooling (optional)

# Bước 1: Bật CloudTrail ngay từ đầu (miễn phí tier)
aws cloudtrail create-trail \
  --name lab-trail \
  --s3-bucket-name my-cloudtrail-logs-$(aws sts get-caller-identity --query Account --output text) \
  --is-multi-region-trail

aws cloudtrail start-logging --name lab-trail

# Bước 2: Bật GuardDuty (30 ngày trial miễn phí)
aws guardduty create-detector --enable

# Bước 3: Tạo billing alarm (tránh chi phí bất ngờ)
aws cloudwatch put-metric-alarm \
  --alarm-name billing-alert-10usd \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --period 86400 \
  --statistic Maximum \
  --alarm-actions arn:aws:sns:us-east-1:$(aws sts get-caller-identity --query Account --output text):billing-alerts
```

### Chi Phí Lab Ước Tính

| Dịch Vụ | Chi Phí Tháng | Ghi Chú |
|---|---|---|
| CloudTrail | ~$2 | 1 trail, S3 storage |
| GuardDuty | $0 (trial) | 30 ngày miễn phí |
| KMS | ~$1 | 1–2 CMKs |
| S3 | ~$1 | Logs + test data |
| EC2 (lab instances) | ~$5–10 | Tắt khi không dùng |
| RDS (lab) | ~$15–20 | db.t3.micro, tắt khi không dùng |
| Security Hub | ~$3 | 30 ngày trial |
| **Tổng** | **~$25–35/tháng** | Tắt resources khi không dùng |

**Tip tiết kiệm chi phí:**
- Dùng `aws ec2 stop-instances` và `aws rds stop-db-instance` sau mỗi lab session
- Setup AWS Budget alert ($20/tháng)
- Dùng t3.micro (free tier 750 giờ/tháng)

---

## Tài Nguyên Học Tập

### Tài Liệu Chính Thức (Miễn Phí)

| Tài Liệu | Link | Khi Dùng |
|---|---|---|
| IAM User Guide | docs.aws.amazon.com/iam/ | Reference cho policy syntax |
| Security Best Practices Whitepaper | aws.amazon.com/security/ | Big picture understanding |
| Well-Architected Security Pillar | aws.amazon.com/architecture/well-architected/ | Design principles |
| AWS Security Blog | aws.amazon.com/blogs/security/ | Weekly — tính năng mới |

### Khóa Học (Trả Phí)

| Khóa Học | Platform | Thời Gian | Phù Hợp |
|---|---|---|---|
| AWS Security Specialty (Adrian Cantrill) | learn.cantrill.io | 40+ giờ | Chuẩn bị cert |
| AWS Security Path | A Cloud Guru | 30+ giờ | Tổng quát |
| AWS Security Fundamentals (Stephane Maarek) | Udemy | 15 giờ | Nhanh, thực tế |

### Công Cụ Thực Hành

```bash
# Prowler — Security audit AWS
pip install prowler
prowler aws --services iam s3 cloudtrail guardduty

# Parliament — Lint IAM policies
pip install parliament
parliament --string '{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}'

# iamlive — Capture IAM permissions cần thiết
# (chạy cùng code, capture permissions thực tế được dùng)
pip install iamlive
iamlive --set-ini
```

### Practice Exams

| Resource | Câu Hỏi | Chất Lượng |
|---|---|---|
| Tutorials Dojo (Jon Bonso) | 390+ | ⭐⭐⭐⭐⭐ |
| Whizlabs | 300+ | ⭐⭐⭐⭐ |
| AWS Official Practice (Free) | 20 | ⭐⭐⭐ |

---

## Theo Dõi Tiến Độ

### Checklist Giai Đoạn 1 (Ngày 1–30)

```
IAM Fundamentals:
□ Viết được IAM policy với conditions từ đầu
□ Giải thích IAM evaluation order đầy đủ
□ Debug được Access Denied trong 5 phút
□ Setup cross-account role thành công

Encryption:
□ Giải thích envelope encryption step-by-step
□ Biết khi nào dùng KMS vs CloudHSM
□ Encrypt RDS với CMK trong lab
□ Viết key policy đúng cú pháp

Secrets & Federation:
□ Setup Secrets Manager rotation cho RDS
□ Cấu hình Parameter Store hierarchy
□ Setup GitHub OIDC federation
□ Triển khai IAM Identity Center với 2 accounts

Organizations:
□ Viết 3 SCPs khác nhau
□ Tạo OU hierarchy trong lab
□ Hiểu SCP inheritance model
```

### Checklist Giai Đoạn 2 (Ngày 31–60)

```
Network Security:
□ Design VPC security cho 3-tier app
□ Setup WAF với SQL injection rules
□ Cấu hình VPC Endpoint cho S3, DynamoDB

Monitoring:
□ Setup CloudTrail organization trail
□ Viết Athena query phân tích CloudTrail
□ Enable Config với auto-remediation
□ Build CloudWatch security dashboard

Threat Detection:
□ Enable GuardDuty, review finding types
□ Setup Security Hub với CIS benchmark
□ Scan EC2 với Inspector
□ Simulate incident + isolation drill

Compliance:
□ Deploy CIS conformance pack
□ Map PCI-DSS requirements sang AWS controls
□ Implement ABAC policy
```

### Checklist Giai Đoạn 3 (Ngày 61–90)

```
Interview Preparation:
□ Trả lời 25 câu hỏi top không cần tài liệu (>80%)
□ Debug IAM scenario trong < 5 phút
□ System design 2 bài toán với trade-offs
□ Hoàn thành 3 STAR stories, kể trong < 4 phút

Certification:
□ Practice exam ≥ 800/1000 (3 lần liên tiếp)
□ Đã đăng ký thi hoặc interview

Portfolio:
□ Lab project documented trên GitHub
□ Architecture diagram hoàn chỉnh
□ Runbook incident response
```

---

## 💡 Tips Học Hiệu Quả

### Daily Habits (Thói Quen Hàng Ngày)

```
Sáng (30 phút):
└── Đọc AWS Security Blog — 1 bài viết mới
└── Review flashcards của ngày hôm qua

Tối (60–90 phút):
└── Study module mới theo lịch
└── Làm lab thực hành
└── Ghi notes điểm quan trọng

Cuối tuần:
└── Review toàn bộ tuần
└── Làm lab lớn hơn
└── Mock interview hoặc practice exam
```

### Spaced Repetition (Lặp Lại Theo Khoảng Cách)

```
Dùng Anki hoặc Notion để tạo flashcards:
Card mặt trước: "Khi nào dùng CloudHSM thay vì KMS?"
Card mặt sau: "FIPS 140-2 Level 3, custom algorithms, dedicated HSM requirement..."

Review schedule: Ngày 1 → Ngày 3 → Ngày 7 → Ngày 14 → Ngày 30
```

### Community Learning (Học Cộng Đồng)

```
Tham gia:
├── AWS Security Community Slack (awssecuritydigest.com)
├── r/aws subreddit — tag security
├── LinkedIn: follow AWS Security specialists
└── Twitter/X: @awssecurityblog, @JadedHacker
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
