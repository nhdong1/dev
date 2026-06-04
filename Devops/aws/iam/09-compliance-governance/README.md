# 09. Compliance & Governance — Tuân Thủ & Quản Trị Trên AWS

> Chiến lược toàn diện để xây dựng chương trình tuân thủ (compliance program) và quản trị bảo mật (security governance) trên nền tảng AWS — từ thu thập bằng chứng tự động đến kiến trúc đáp ứng PCI-DSS và HIPAA.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Mức Độ |
|---|---|---|
| [1-audit-manager.md](1-audit-manager.md) | AWS Audit Manager — Thu Thập Bằng Chứng Tự Động | ⭐⭐⭐ |
| [2-conformance-packs.md](2-conformance-packs.md) | Config Conformance Packs — Gói Quy Tắc Tuân Thủ | ⭐⭐⭐ |
| [3-firewall-manager.md](3-firewall-manager.md) | AWS Firewall Manager — Quản Lý Tường Lửa Tập Trung | ⭐⭐⭐ |
| [4-pci-dss-aws.md](4-pci-dss-aws.md) | PCI-DSS trên AWS — Tuân Thủ Ngành Thanh Toán | ⭐⭐⭐ |
| [5-hipaa-aws.md](5-hipaa-aws.md) | HIPAA trên AWS — Tuân Thủ Dữ Liệu Y Tế | ⭐⭐⭐ |

---

## 🎯 Tại Sao Compliance & Governance Quan Trọng?

### Rủi Ro Khi Không Tuân Thủ

```
Tài chính:
├── PCI-DSS vi phạm → phạt $5,000–$100,000/tháng
├── HIPAA vi phạm → phạt $100–$50,000/vi phạm, tối đa $1.9M/năm
├── GDPR vi phạm → phạt tới 4% doanh thu toàn cầu hoặc €20M
└── SOC2 thiếu → mất hợp đồng doanh nghiệp

Vận hành:
├── Audit thủ công → mất hàng tuần thu thập bằng chứng
├── Không nhất quán → tài khoản/vùng cấu hình sai
├── Không phát hiện drift → tài nguyên không tuân thủ tồn tại hàng tháng
└── Phản ứng chậm → sự cố leo thang thành breach
```

### Lợi Ích Của Tự Động Hóa Tuân Thủ

```
Tự Động Hóa (Automation):
├── Audit Manager → tự động thu thập 90%+ bằng chứng
├── Conformance Packs → đánh giá hàng nghìn tài nguyên liên tục
├── Firewall Manager → áp dụng chính sách đồng nhất trên mọi tài khoản
└── Security Hub → điểm tuân thủ duy nhất cho toàn bộ tổ chức
```

---

## 🗺️ Kiến Trúc Governance Tổng Thể

```
┌─────────────────────────────────────────────────────────────┐
│                   Management Account                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ Audit       │  │ Firewall     │  │ Security Hub      │  │
│  │ Manager     │  │ Manager      │  │ (Aggregator)      │  │
│  │ (Evidence)  │  │ (WAF/SG/     │  │                   │  │
│  └─────────────┘  │  Shield)     │  └───────────────────┘  │
│                   └──────────────┘                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           AWS Config (Aggregator)                    │    │
│  │     Conformance Packs: CIS / PCI-DSS / HIPAA        │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────┘
                               │ AWS Organizations
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │  Production   │   │  Staging      │   │  Development  │
  │  Account      │   │  Account      │   │  Account      │
  │               │   │               │   │               │
  │ Config Rules  │   │ Config Rules  │   │ Config Rules  │
  │ CloudTrail    │   │ CloudTrail    │   │ CloudTrail    │
  │ GuardDuty     │   │ GuardDuty     │   │ GuardDuty     │
  └───────────────┘   └───────────────┘   └───────────────┘
```

---

## 📋 Các Framework Tuân Thủ Phổ Biến Trên AWS

### PCI-DSS (Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán)

- **Áp dụng cho:** Tổ chức xử lý, lưu trữ, hoặc truyền dữ liệu thẻ tín dụng
- **Phiên bản hiện tại:** PCI-DSS v4.0 (có hiệu lực từ 2024)
- **AWS Responsibility:** Tuân thủ hạ tầng vật lý, hypervisor
- **Customer Responsibility:** Cấu hình dịch vụ, mã hóa, kiểm soát truy cập

```
12 Yêu Cầu PCI-DSS → Dịch Vụ AWS Tương Ứng:
├── Req 1-2: Network controls       → VPC, Security Groups, NACLs, WAF
├── Req 3-4: Data protection        → KMS, ACM, S3 encryption
├── Req 5-6: Vulnerability mgmt     → Inspector, Patch Manager
├── Req 7-8: Access control         → IAM, MFA, Cognito
├── Req 9: Physical security        → AWS manages (Shared Responsibility)
├── Req 10: Logging & monitoring    → CloudTrail, CloudWatch, Config
├── Req 11: Security testing        → Inspector, GuardDuty, Security Hub
└── Req 12: Security policies       → AWS Artifact, Audit Manager
```

### HIPAA (Health Insurance Portability and Accountability Act — Đạo Luật Về Tính Khả Chuyển Và Trách Nhiệm Bảo Hiểm Y Tế)

- **Áp dụng cho:** Covered Entities (Tổ Chức Được Bảo Vệ) và Business Associates (Đối Tác Kinh Doanh)
- **Dữ liệu được bảo vệ:** PHI (Protected Health Information — Thông Tin Sức Khỏe Được Bảo Vệ)
- **AWS HIPAA Eligible Services:** Danh sách dịch vụ có thể dùng với PHI sau khi ký BAA

```
HIPAA Safeguards → Dịch Vụ AWS:
├── Administrative: Audit Manager, IAM Identity Center
├── Physical: AWS manages data centers
└── Technical:
    ├── Access Control → IAM, MFA
    ├── Audit Controls → CloudTrail, Config
    ├── Integrity      → KMS, S3 Object Integrity
    └── Transmission   → TLS/ACM, VPC, PrivateLink
```

### SOC 2 (System and Organization Controls 2 — Kiểm Soát Hệ Thống Và Tổ Chức)

- **Áp dụng cho:** SaaS providers, cloud service companies
- **5 Trust Service Criteria:** Security, Availability, Processing Integrity, Confidentiality, Privacy
- **AWS Artifact:** Tải báo cáo SOC của AWS để bổ sung vào audit của bạn

### CIS Benchmarks (Center for Internet Security — Trung Tâm Bảo Mật Internet)

- **CIS AWS Foundations Benchmark:** Tiêu chuẩn bảo mật cơ bản cho AWS
- **Security Hub:** Tích hợp sẵn CIS Benchmark v1.4 và v3.0
- **Config Conformance Packs:** Gói quy tắc CIS sẵn có

---

## 🔧 Bộ Công Cụ Governance Trên AWS

### Lớp Thu Thập Bằng Chứng (Evidence Layer)

| Công Cụ | Mục Đích | Tự Động Hóa |
|---|---|---|
| **Audit Manager** | Thu thập bằng chứng theo framework | ✅ Cao |
| **AWS Config** | Lịch sử cấu hình tài nguyên | ✅ Liên tục |
| **CloudTrail** | Nhật ký mọi API call | ✅ Tự động |
| **AWS Artifact** | Báo cáo tuân thủ của AWS | Manual Download |

### Lớp Kiểm Soát Chính Sách (Policy Enforcement Layer)

| Công Cụ | Mục Đích | Phạm Vi |
|---|---|---|
| **Firewall Manager** | Quản lý WAF/SG/Shield tập trung | Đa tài khoản |
| **Config Conformance Packs** | Áp dụng bộ quy tắc đóng gói sẵn | Đa tài khoản |
| **SCPs** | Giới hạn quyền ở cấp OU/account | Toàn Organizations |
| **IAM Permission Boundaries** | Giới hạn quyền tối đa của role/user | Per account |

### Lớp Báo Cáo & Tổng Hợp (Reporting Layer)

| Công Cụ | Mục Đích | Output |
|---|---|---|
| **Security Hub** | Điểm tuân thủ tổng hợp | Score 0–100 |
| **Audit Manager** | Báo cáo audit sẵn sàng | PDF/CSV |
| **Config Aggregator** | Tổng hợp compliance đa tài khoản | Dashboard |
| **QuickSight** | Trực quan hóa dữ liệu audit | Dashboard |

---

## 🚀 Lộ Trình Triển Khai Governance

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

```
1. Bật AWS Config ở tất cả regions và tài khoản
2. Thiết lập CloudTrail multi-region trail
3. Kích hoạt Security Hub với CIS Benchmark
4. Bật GuardDuty ở tất cả tài khoản
```

### Giai Đoạn 2: Chuẩn Hóa (Tuần 3–4)

```
1. Deploy Conformance Packs phù hợp framework (CIS/PCI/HIPAA)
2. Triển khai Firewall Manager policies
3. Thiết lập Config Aggregator tại Management Account
4. Cấu hình Security Hub Aggregator
```

### Giai Đoạn 3: Tự Động Hóa (Tuần 5–6)

```
1. Bật Audit Manager với framework phù hợp
2. Thiết lập auto-remediation với Config + Lambda
3. Cấu hình SNS alerts cho non-compliance
4. Tạo dashboard báo cáo với Security Hub
```

### Giai Đoạn 4: Cải Tiến Liên Tục (Tháng 2+)

```
1. Review và cập nhật Conformance Packs theo framework mới
2. Mở rộng auto-remediation sang nhiều loại vi phạm hơn
3. Tích hợp báo cáo vào công cụ GRC (Governance, Risk, Compliance)
4. Đào tạo nhóm về quy trình audit mới
```

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa AWS Artifact và Audit Manager?**
> AWS Artifact cung cấp báo cáo tuân thủ CỦA AWS (AWS tự audit). Audit Manager giúp bạn thu thập bằng chứng cho chương trình tuân thủ CỦA BẠN (audit hệ thống bạn xây dựng trên AWS).

**Q: Làm thế nào để chứng minh tuân thủ PCI-DSS khi hệ thống chạy trên AWS?**
> 1. Tải AWS PCI-DSS Attestation of Compliance từ AWS Artifact → chứng minh hạ tầng AWS đã tuân thủ. 2. Dùng Audit Manager với PCI-DSS framework → thu thập bằng chứng về cấu hình của bạn. 3. Config Conformance Pack PCI-DSS → báo cáo liên tục. 4. Thuê QSA (Qualified Security Assessor — Chuyên Gia Đánh Giá Bảo Mật Đủ Năng Lực) để xác nhận.

**Q: Firewall Manager cần gì để hoạt động?**
> Yêu cầu: 1) Tài khoản là thành viên AWS Organizations. 2) AWS Config phải được bật ở tất cả tài khoản thành viên. 3) Tài khoản quản trị Firewall Manager phải được chỉ định (thường là Security account). 4) Với WAF policies: cần bật AWS WAF.

**Q: Conformance Pack khác Config Rule như thế nào?**
> Config Rule là đơn vị kiểm tra đơn lẻ. Conformance Pack là bộ sưu tập nhiều Config Rules đóng gói thành một template YAML, có thể deploy cùng lúc cho toàn bộ Organizations và ánh xạ sang các control của framework tuân thủ (PCI-DSS, CIS, HIPAA).

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [08-threat-detection/](../08-threat-detection/README.md) | **09-compliance-governance/** | [10-advanced/](../10-advanced/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
