# AWS Security, Identity & Compliance — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về Bảo Mật, Định Danh và Tuân Thủ trên AWS

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/iam/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-iam-fundamentals/
│   ├── README.md                               ✅ Nền tảng IAM, khái niệm cốt lõi
│   ├── 1-users-groups-roles.md                 ✅ Users, Groups, Roles chi tiết
│   ├── 2-policy-types.md                       ✅ Identity/Resource/SCPs/Permissions Boundary
│   ├── 3-iam-conditions.md                     ✅ Condition keys, operators, examples
│   ├── 4-permission-boundaries.md              ✅ Giới hạn quyền, use cases
│   └── 5-iam-best-practices.md                 ✅ Checklist bảo mật IAM
│
├── 02-identity-federation/
│   ├── README.md                               ✅ Federation, SSO, Cognito overview
│   ├── 1-iam-identity-center.md                ✅ AWS SSO đa tài khoản
│   ├── 2-saml-federation.md                    ✅ SAML 2.0 với AD/Okta/Azure AD
│   ├── 3-oidc-federation.md                    ✅ OIDC với GitHub Actions, Google
│   ├── 4-cognito-user-pools.md                 ✅ Auth cho ứng dụng web/mobile
│   └── 5-cross-account-roles.md                ✅ Mô hình hub-and-spoke
│
├── 03-organizations/
│   ├── README.md                               ✅ Multi-account strategy, landing zone overview
│   ├── 1-organizations-setup.md                ✅ Tạo OU hierarchy, service integrations
│   ├── 2-service-control-policies.md           ✅ SCPs — cú pháp, ví dụ thực tế, lỗi phổ biến
│   ├── 3-control-tower.md                      ✅ Landing zone tự động, guardrails, AFT
│   └── 4-account-vending.md                    ✅ Tự động tạo tài khoản — AVM, AFT, pipeline
│
├── 04-encryption-kms/
│   ├── README.md                               ✅ Encryption strategy trên AWS
│   ├── 1-kms-key-types.md                      ✅ CMK vs AWS-managed vs data keys
│   ├── 2-envelope-encryption.md                ✅ Mã hóa phong bì, DEK lifecycle
│   ├── 3-key-policies.md                       ✅ Key policy vs IAM policy
│   ├── 4-kms-grants.md                         ✅ Grants — ủy quyền dùng key tạm thời
│   ├── 5-cloudhsm.md                           ✅ CloudHSM — FIPS 140-2 Level 3
│   └── 6-s3-encryption-options.md              ✅ SSE-S3 vs SSE-KMS vs SSE-C vs CSE
│
├── 05-secrets-certificates/
│   ├── README.md                               ✅ Secrets lifecycle management
│   ├── 1-secrets-manager.md                    ✅ Lưu trữ, xoay vòng, truy xuất bí mật
│   ├── 2-parameter-store.md                    ✅ Hierarchy, SecureString, versions
│   ├── 3-secrets-vs-parameter.md               ✅ So sánh và hướng dẫn chọn lựa
│   ├── 4-acm-certificates.md                   ✅ TLS cert lifecycle, auto-renewal
│   └── 5-acm-private-ca.md                     ✅ Internal PKI trên AWS
│
├── 06-network-security/
│   ├── README.md                               ✅ Defense-in-depth network design
│   ├── 1-security-groups.md                    ✅ Stateful firewall, best practices
│   ├── 2-nacls.md                              ✅ Stateless rules, subnet protection
│   ├── 3-vpc-endpoints.md                      ✅ Gateway vs Interface endpoints
│   ├── 4-privatelink.md                        ✅ Private service connectivity
│   ├── 5-waf-setup.md                          ✅ WAF rules, managed rule groups
│   ├── 6-shield-ddos.md                        ✅ DDoS protection tiers
│   └── 7-network-firewall.md                   ✅ Stateful firewall có thể lập trình
│
├── 07-monitoring-auditing/
│   ├── README.md                               ✅ Security observability strategy
│   ├── 1-cloudtrail-setup.md                   ✅ Trail config, S3 integrity, Lake
│   ├── 2-cloudtrail-analysis.md                ✅ Phân tích log, Athena queries
│   ├── 3-aws-config-rules.md                   ✅ Managed rules, custom rules, remediation
│   ├── 4-access-analyzer.md                    ✅ Phát hiện external access
│   └── 5-security-dashboard.md                 ✅ CloudWatch dashboards cho security
│
├── 08-threat-detection/
│   ├── README.md                               ✅ Threat detection ecosystem, so sánh dịch vụ
│   ├── 1-guardduty.md                          ✅ Finding types, severity levels, suppression rules
│   ├── 2-security-hub.md                       ✅ Aggregation, ASFF, standards, custom actions
│   ├── 3-inspector.md                          ✅ EC2/ECR/Lambda vulnerability scanning, CVSS
│   ├── 4-macie.md                              ✅ S3 data classification, PII/PHI detection
│   ├── 5-detective.md                          ✅ Graph-based investigation, behavior graph
│   └── 6-incident-response.md                  ✅ Runbook xử lý sự cố bảo mật AWS
│
├── 09-compliance-governance/
│   ├── README.md                               ✅ Compliance strategy on AWS
│   ├── 1-audit-manager.md                      ✅ Tự động thu thập bằng chứng
│   ├── 2-conformance-packs.md                  ✅ CIS, PCI-DSS, HIPAA conformance packs
│   ├── 3-firewall-manager.md                   ✅ Centralized WAF/SG/Shield management
│   ├── 4-pci-dss-aws.md                        ✅ Payment Card Industry compliance
│   └── 5-hipaa-aws.md                          ✅ Healthcare data compliance
│
├── 10-advanced/
│   ├── README.md                               ✅ Advanced security patterns overview
│   ├── 1-abac.md                               ✅ Attribute-Based Access Control
│   ├── 2-zero-trust.md                         ✅ Zero Trust Architecture trên AWS
│   ├── 3-data-perimeter.md                     ✅ Vành đai dữ liệu, resource policies
│   ├── 4-security-automation.md                ✅ Auto-remediation với EventBridge + Lambda
│   └── 5-supply-chain-security.md              ✅ CodeArtifact, Sigstore, SBOM
│
├── 11-interview-prep/
│   ├── README.md                               ✅ Tổng quan chuẩn bị phỏng vấn
│   ├── 1-top-questions.md                      ✅ Top 25 câu hỏi AWS Security + đáp án chi tiết
│   ├── 2-iam-troubleshooting.md                ✅ Debug Access Denied, policy conflicts, 6 scenarios
│   ├── 3-system-design-security.md             ✅ 5 bài toán thiết kế kiến trúc bảo mật
│   ├── 4-star-stories.md                       ✅ 6 mẫu câu chuyện sự cố STAR + tips
│   └── 5-90-day-study-plan.md                  ✅ Kế hoạch học tập 90 ngày có cấu trúc
│
├── GLOSSARY.md                                 (Cần tạo) Thuật ngữ AWS Security
├── RESOURCES.md                                (Cần tạo) Sách, blog, công cụ, chứng chỉ
└── CHECKLIST.md                                (Cần tạo) Checklist pre-deployment & pre-interview
```

---

## ✅ Trạng Thái Nội Dung

| Chủ Đề | File | Trạng Thái | Mức Độ |
|---|---|---|---|
| **Tổng Quan & Lộ Trình** | README.md | ✅ Hoàn Thành | Toàn Diện |
| **Chỉ Mục Đầy Đủ** | INDEX.md | ✅ Hoàn Thành | Đầy Đủ |
| **IAM Fundamentals** | 01-iam-fundamentals/ | ✅ Hoàn Thành | Toàn Diện |
| **Identity Federation** | 02-identity-federation/ | ✅ Hoàn Thành | - |
| **Organizations & Multi-Account** | 03-organizations/ | ✅ Hoàn Thành | Toàn Diện |
| **Encryption & KMS** | 04-encryption-kms/ | ✅ Hoàn Thành | Toàn Diện |
| **Secrets & Certificates** | 05-secrets-certificates/ | ✅ Hoàn Thành | Toàn Diện |
| **Network Security** | 06-network-security/ | ✅ Hoàn Thành | Toàn Diện |
| **Monitoring & Auditing** | 07-monitoring-auditing/ | ✅ Hoàn Thành | Toàn Diện |
| **Threat Detection** | 08-threat-detection/ | ✅ Hoàn Thành | Toàn Diện |
| **Compliance & Governance** | 09-compliance-governance/ | ✅ Hoàn Thành | Toàn Diện |
| **Advanced Topics** | 10-advanced/ | ✅ Hoàn Thành | Toàn Diện |
| **Interview Prep** | 11-interview-prep/ | ✅ Hoàn Thành | Toàn Diện |

---

## 🎯 Thứ Tự Ưu Tiên Tạo Nội Dung

### Ưu Tiên Cao (Kỹ Năng Cốt Lõi)

- [ ] `01-iam-fundamentals/README.md` — IAM users, roles, policies cơ bản
- [ ] `04-encryption-kms/README.md` — KMS, envelope encryption
- [ ] `07-monitoring-auditing/cloudtrail-setup.md` — audit logging thiết yếu
- [ ] `08-threat-detection/guardduty.md` — phát hiện mối đe dọa tự động
- [ ] `11-interview-prep/top-questions.md` — Top 25 câu hỏi phỏng vấn

### Ưu Tiên Trung Bình (Kỹ Năng Nâng Cao)

- [ ] `02-identity-federation/iam-identity-center.md` — SSO đa tài khoản
- [ ] `03-organizations/service-control-policies.md` — SCPs chi tiết
- [ ] `05-secrets-certificates/secrets-manager.md` — auto-rotation
- [ ] `08-threat-detection/security-hub.md` — tổng hợp findings
- [ ] `09-compliance-governance/audit-manager.md` — evidence collection

### Ưu Tiên Thấp Hơn (Tài Liệu Tham Khảo)

- [ ] `10-advanced/zero-trust.md` — kiến trúc zero trust
- [ ] `10-advanced/data-perimeter.md` — vành đai dữ liệu
- [ ] `06-network-security/waf-setup.md` — WAF chi tiết
- [ ] `GLOSSARY.md` — thuật ngữ
- [ ] `RESOURCES.md` — tài liệu học tập

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Tự Học

```
1. Đọc README.md — hiểu toàn cảnh
2. Chọn lộ trình phù hợp với mục tiêu nghề nghiệp
3. Học từng module theo thứ tự giai đoạn
4. Thực hành trên AWS (dùng account lab riêng)
5. Xây dựng dự án portfolio minh chứng kỹ năng
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/top-questions.md
2. Ôn kỹ 01-iam-fundamentals/ (luôn được hỏi)
3. Ôn 04-encryption-kms/ (câu hỏi kiến trúc phổ biến)
4. Chuẩn bị 2-3 câu chuyện sự cố theo STAR
5. Luyện debug IAM với iam-troubleshooting.md
```

### Kỹ Sư Vận Hành (Operations)

```
Dùng như tài liệu tham khảo:
- Trước deployment: Xem CHECKLIST.md
- Xử lý sự cố: Đến 08-threat-detection/incident-response.md
- Audit: Dùng 07-monitoring-auditing/ và 09-compliance-governance/
- Thiết kế hệ thống: Tham khảo 03-organizations/ cho multi-account
- Rotation: Theo hướng dẫn 05-secrets-certificates/
```

### Thiết Kế Kiến Trúc (Architecture)

```
1. Đọc 03-organizations/ cho multi-account design
2. Dùng 01-iam-fundamentals/ để thiết kế least privilege
3. Theo 06-network-security/ cho defense-in-depth
4. Tham chiếu 04-encryption-kms/ cho data protection
5. Kiểm tra 09-compliance-governance/ cho regulatory requirements
```

---

## 📊 Ước Tính Thời Gian Học

| Module | Thời Gian | Độ Khó | Mức Ưu Tiên |
|---|---|---|---|
| IAM Fundamentals | 6–8 giờ | ⭐⭐ | Bắt Buộc |
| Identity Federation & SSO | 4–6 giờ | ⭐⭐ | Bắt Buộc |
| Organizations & Multi-Account | 4–6 giờ | ⭐⭐⭐ | Bắt Buộc |
| Encryption & KMS | 6–8 giờ | ⭐⭐⭐ | Bắt Buộc |
| Secrets & Certificates | 3–4 giờ | ⭐⭐ | Bắt Buộc |
| Network Security | 5–7 giờ | ⭐⭐ | Bắt Buộc |
| Monitoring & Auditing | 4–6 giờ | ⭐⭐ | Bắt Buộc |
| Threat Detection | 5–7 giờ | ⭐⭐ | Bắt Buộc |
| Compliance & Governance | 6–8 giờ | ⭐⭐⭐ | Nên Học |
| Advanced Topics | 10–15 giờ | ⭐⭐⭐ | Nice-to-Have |

**Tổng cộng: 53–75 giờ cho kiến thức AWS Security toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Được Hỗ Trợ

### Người Mới Bắt Đầu (0–1 năm kinh nghiệm)

- [ ] Users, Groups, Roles, Policies cơ bản
- [ ] MFA và root account security
- [ ] S3 bucket policies và ACLs
- [ ] Security Groups cơ bản
- [ ] Shared Responsibility Model

**Thời gian đạt được:** 1–2 tháng

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] IAM role design cho microservices
- [ ] KMS encryption end-to-end
- [ ] Multi-account với Organizations
- [ ] GuardDuty + Security Hub integration
- [ ] Secrets Manager với auto-rotation

**Thời gian nâng cao:** 2–3 tháng thực hành chuyên sâu

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Zero Trust architecture design
- [ ] Data perimeter enforcement
- [ ] Security automation pipelines
- [ ] Compliance program management
- [ ] Advanced threat hunting

**Thời gian:** Học liên tục — lĩnh vực luôn thay đổi

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu | Vị Trí |
|---|---|
| Tổng quan toàn bộ | [README.md](README.md) |
| IAM cơ bản | [01-iam-fundamentals/README.md](01-iam-fundamentals/README.md) |
| SSO đa tài khoản | [02-identity-federation/1-iam-identity-center.md](02-identity-federation/1-iam-identity-center.md) |
| Multi-account design | [03-organizations/README.md](03-organizations/README.md) |
| Mã hóa & KMS | [04-encryption-kms/README.md](04-encryption-kms/README.md) |
| Quản lý bí mật | [05-secrets-certificates/1-secrets-manager.md](05-secrets-certificates/1-secrets-manager.md) |
| Bảo mật mạng | [06-network-security/README.md](06-network-security/README.md) |
| CloudTrail & Config | [07-monitoring-auditing/README.md](07-monitoring-auditing/README.md) |
| GuardDuty & incidents | [08-threat-detection/6-incident-response.md](08-threat-detection/6-incident-response.md) |
| Compliance | [09-compliance-governance/README.md](09-compliance-governance/README.md) |
| Câu hỏi phỏng vấn | [11-interview-prep/1-top-questions.md](11-interview-prep/1-top-questions.md) |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## Tiến Độ AWS Security

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)
- [ ] IAM Users, Groups, Roles
- [ ] Policy types & JSON syntax
- [ ] Permission Boundaries
- [ ] MFA & account security
- [ ] Shared Responsibility Model

### Giai Đoạn 2: Dịch Vụ Định Danh (Tuần 3–5)
- [ ] IAM Identity Center (SSO)
- [ ] SAML federation với IdP
- [ ] Cross-account role setup
- [ ] Cognito User Pools
- [ ] AWS Organizations + SCPs

### Giai Đoạn 3: Hạ Tầng Bảo Mật (Tuần 6–8)
- [ ] KMS key types & policies
- [ ] Envelope encryption
- [ ] Secrets Manager rotation
- [ ] WAF rule groups
- [ ] VPC security architecture

### Giai Đoạn 4: Giám Sát & Phát Hiện (Tuần 9–11)
- [ ] CloudTrail + Lake setup
- [ ] GuardDuty finding types
- [ ] Security Hub standards
- [ ] AWS Config rules
- [ ] Inspector scanning

### Giai Đoạn 5: Tuân Thủ & Nâng Cao (Tuần 12+)
- [ ] Audit Manager setup
- [ ] Conformance packs
- [ ] Zero Trust design
- [ ] Data perimeter
- [ ] Security automation
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Cơ Bản

- [ ] Thiết kế IAM policy phức tạp không cần tài liệu
- [ ] Giải thích Shared Responsibility Model chi tiết
- [ ] Debug Access Denied errors một cách có hệ thống
- [ ] Thiết kế kiến trúc multi-account an toàn
- [ ] Cấu hình encryption end-to-end cho dữ liệu nhạy cảm

### ✅ Năng Lực Vận Hành

- [ ] Phân tích CloudTrail log để tìm hoạt động đáng ngờ
- [ ] Xử lý GuardDuty findings theo runbook chuẩn
- [ ] Triển khai Secrets Manager rotation không gián đoạn
- [ ] Thiết lập Security Hub với standards tuân thủ
- [ ] Thực hiện security audit với AWS Config

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin 25 câu hỏi AWS Security phổ biến
- [ ] Kể được 2–3 câu chuyện sự cố bảo mật (STAR)
- [ ] Thiết kế hệ thống an toàn trong bài toán system design
- [ ] Thảo luận được trade-offs giữa các giải pháp bảo mật
- [ ] Nắm vững ít nhất một compliance framework

---

## 🚀 Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc toàn bộ README.md
2. Chọn lộ trình phù hợp mục tiêu
3. Bắt đầu `01-iam-fundamentals/` — luôn được hỏi trong phỏng vấn
4. Tạo AWS lab account, bật CloudTrail và GuardDuty

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành 01-iam-fundamentals/
2. Thực hành viết IAM policies từ đầu
3. Bắt đầu 04-encryption-kms/
4. Lab: mã hóa S3 bucket với CMK

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành các chủ đề cốt lõi (01–08)
2. Thiết lập GuardDuty + Security Hub trong lab
3. Chuẩn bị 2–3 câu chuyện STAR từ kinh nghiệm thực tế
4. Mock interview với đồng nghiệp

### Dài Hạn (3 Tháng Tới)

1. Hoàn thành toàn bộ knowledge base
2. Xây dựng portfolio project bảo mật thực tế
3. Đăng ký thi AWS Certified Security — Specialty
4. Đóng góp vào open-source security tools (Prowler, ScoutSuite)

---

## 💡 Mẹo Học Tập Hiệu Quả

1. **Luôn dùng lab account riêng:** Không thực hành trên production — tạo AWS account riêng
2. **Bật CloudTrail ngay từ đầu:** Quan sát mọi hành động của bạn để học từ API calls
3. **Đọc IAM Policy Simulator:** Kiểm thử policy trước khi áp dụng thực tế
4. **Nghiên cứu các breach thực tế:** Capital One breach, Tesla crypto-mining — học từ sự cố thật
5. **Biết tại sao, không chỉ cái gì:** Hiểu nguyên lý, không chỉ học thuộc lòng dịch vụ
6. **Follow AWS Security Blog:** Cập nhật tính năng mới hàng tuần
7. **Luyện tập viết policy JSON:** Đây là kỹ năng phân biệt senior với junior
8. **Document sự cố lab của bạn:** Post-mortem ngay cả trong lab là thói quen tốt

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn thêm nội dung?

Đây là tài liệu sống — đóng góp luôn được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Cập nhật khi AWS ra tính năng mới
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Thêm bài lab thực hành chi tiết

---

## 📄 Giấy Phép

Knowledge base này mở để học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 3.0 (11-interview-prep hoàn thành — Knowledge Base Đầy Đủ)
**Trạng Thái:** ✅ README Hoàn Thành | ✅ INDEX Hoàn Thành | ✅ 01-iam-fundamentals Hoàn Thành | ✅ 02-identity-federation Hoàn Thành | ✅ 03-organizations Hoàn Thành | ✅ 04-encryption-kms Hoàn Thành | ✅ 05-secrets-certificates Hoàn Thành | ✅ 06-network-security Hoàn Thành | ✅ 07-monitoring-auditing Hoàn Thành | ✅ 08-threat-detection Hoàn Thành | ✅ 09-compliance-governance Hoàn Thành | ✅ 10-advanced Hoàn Thành | ✅ 11-interview-prep Hoàn Thành
