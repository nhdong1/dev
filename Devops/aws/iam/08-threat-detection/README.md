# 08 — Threat Detection & Incident Response (Phát Hiện Mối Đe Dọa & Phản Hồi Sự Cố)

> Hệ sinh thái phát hiện mối đe dọa tự động trên AWS: từ giám sát liên tục đến phản hồi sự cố có cấu trúc.

## 📚 Mục Lục Module

| File | Chủ Đề | Mô Tả |
|---|---|---|
| [1-guardduty.md](1-guardduty.md) | GuardDuty | Phát hiện mối đe dọa dựa trên ML, finding types, suppression |
| [2-security-hub.md](2-security-hub.md) | Security Hub | Tổng hợp findings, standards, custom actions |
| [3-inspector.md](3-inspector.md) | Inspector v2 | Quét lỗ hổng CVE trên EC2, ECR, Lambda |
| [4-macie.md](4-macie.md) | Macie | Phân loại dữ liệu S3, phát hiện PII/PHI |
| [5-detective.md](5-detective.md) | Detective | Điều tra sự cố bằng đồ thị quan hệ (graph-based) |
| [6-incident-response.md](6-incident-response.md) | Incident Response | Runbook xử lý sự cố bảo mật AWS chuẩn |

---

## 🎯 Tổng Quan Hệ Sinh Thái

### Vị Trí Trong Defense-in-Depth (Phòng Thủ Theo Chiều Sâu)

```
Lớp 1 — Prevention (Ngăn Chặn)
    IAM policies, SCPs, Permission Boundaries, WAF

Lớp 2 — Detection (Phát Hiện)          ← Module này tập trung ở đây
    GuardDuty, Inspector, Macie, Security Hub

Lớp 3 — Investigation (Điều Tra)        ← Module này tập trung ở đây
    Detective, CloudTrail, CloudWatch Logs Insights

Lớp 4 — Response (Phản Hồi)            ← Module này tập trung ở đây
    Incident Response Runbook, Lambda automation, Systems Manager
```

### Luồng Thông Tin Findings

```
Data Sources (Nguồn Dữ Liệu)
    ├── CloudTrail — API calls
    ├── VPC Flow Logs — network traffic
    ├── DNS logs — domain resolution
    ├── S3 data events — object access
    └── EKS audit logs — Kubernetes activity

        ▼
Detection Layer (Tầng Phát Hiện)
    ├── GuardDuty — threat detection (ML)
    ├── Inspector — vulnerability scanning
    └── Macie — sensitive data discovery

        ▼
Aggregation Layer (Tầng Tổng Hợp)
    └── Security Hub — normalize, correlate, prioritize findings

        ▼
Investigation Layer (Tầng Điều Tra)
    └── Detective — graph-based root cause analysis

        ▼
Response Layer (Tầng Phản Hồi)
    ├── EventBridge — event routing
    ├── Lambda — automated remediation
    └── Systems Manager — runbook execution
```

---

## 🔍 Tóm Tắt Từng Dịch Vụ

### GuardDuty — Threat Intelligence

- **Phương pháp:** Machine Learning + threat feeds + anomaly detection (phát hiện bất thường)
- **Nguồn dữ liệu:** CloudTrail, VPC Flow Logs, DNS logs, S3 data events, EKS audit logs, RDS login events, Lambda network activity
- **Không cần agent:** Hoạt động hoàn toàn dựa trên API, không cài phần mềm trên EC2
- **Output:** Findings phân loại theo severity (mức độ nghiêm trọng): Low / Medium / High / Critical
- **Bật trong:** 1 click, có thể bật toàn Organizations từ delegated administrator

### Security Hub — Centralized Findings (Tổng Hợp Findings Tập Trung)

- **Phương pháp:** Tổng hợp findings từ GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager...
- **Standards (Tiêu Chuẩn):** CIS AWS Foundations, AWS Foundational Security Best Practices (FSBP), PCI-DSS, NIST 800-53
- **Output:** Security Score (Điểm Bảo Mật) theo từng control, aggregated findings dashboard
- **Custom Actions:** Gửi findings đến EventBridge để kích hoạt automation

### Inspector v2 — Vulnerability Management (Quản Lý Lỗ Hổng)

- **Phương pháp:** Agent-based scan (EC2) + API-based scan (ECR, Lambda)
- **CVE database:** National Vulnerability Database (NVD) + AWS threat intelligence
- **Output:** Findings theo severity với CVSS score, package version, remediation path
- **Tích hợp:** Tự động kích hoạt khi EC2 khởi động, ECR image được push, Lambda function được deploy

### Macie — Data Security Posture (Tư Thế Bảo Mật Dữ Liệu)

- **Phương pháp:** ML-based classification + managed data identifiers (bộ nhận dạng dữ liệu)
- **Phát hiện:** PII (Personally Identifiable Information — Thông Tin Cá Nhân Có Thể Nhận Dạng), PHI (Protected Health Information — Thông Tin Sức Khỏe Được Bảo Vệ), financial data, credentials
- **Scope:** Chỉ dành cho S3 (không scan EBS, RDS, DynamoDB)
- **Output:** Data findings + S3 bucket security posture inventory

### Detective — Security Investigation (Điều Tra Bảo Mật)

- **Phương pháp:** Graph database — xây dựng đồ thị quan hệ từ CloudTrail, VPC Flow Logs, GuardDuty findings
- **Thời gian lưu trữ:** 12 tháng dữ liệu hành vi baseline
- **Use case:** Root cause analysis (Phân Tích Nguyên Nhân Gốc Rễ), lateral movement detection (phát hiện di chuyển ngang), blast radius analysis (phân tích phạm vi ảnh hưởng)
- **Tích hợp:** Một click từ GuardDuty finding sang Detective investigation

---

## ⚡ Bật Nhanh Toàn Bộ Ecosystem

```bash
# Bật GuardDuty
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES

# Bật Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Bật Inspector v2
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA

# Bật Macie
aws macie2 enable-macie

# Bật Detective (cần GuardDuty bật trước ít nhất 48 giờ)
aws detective create-graph

# Kích hoạt từ Organizations (delegated admin — quản trị ủy quyền)
aws organizations enable-aws-service-access \
  --service-principal guardduty.amazonaws.com
```

---

## 📊 So Sánh Nhanh

| Tiêu Chí | GuardDuty | Inspector | Macie | Detective |
|---|---|---|---|---|
| **Mục đích** | Threat detection | Vulnerability scan | Data discovery | Investigation |
| **Nguồn dữ liệu** | Logs (CloudTrail, VPC, DNS) | EC2 agent, ECR API | S3 metadata & content | GuardDuty + CloudTrail |
| **Agent cần** | Không | Có (EC2) | Không | Không |
| **Output chính** | Threat findings | CVE findings | Data findings | Behavior graph |
| **Tích hợp Security Hub** | ✅ | ✅ | ✅ | ❌ (investigation tool) |
| **Giá** | Per GB analyzed | Per instance-month | Per GB scanned | Per GB ingested |

---

## 🔗 Liên Kết Với Các Module Khác

| Module | Mối Liên Hệ |
|---|---|
| [07-monitoring-auditing/](../07-monitoring-auditing/) | CloudTrail cung cấp dữ liệu cho GuardDuty và Detective |
| [06-network-security/](../06-network-security/) | VPC Flow Logs là nguồn dữ liệu quan trọng của GuardDuty |
| [03-organizations/](../03-organizations/) | Bật threat detection tập trung qua Organizations |
| [10-advanced/4-security-automation.md](../10-advanced/4-security-automation.md) | Tự động hóa phản hồi findings với EventBridge + Lambda |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

1. GuardDuty phát hiện mối đe dọa bằng cách nào? Nguồn dữ liệu là gì?
2. Sự khác biệt giữa GuardDuty và Inspector?
3. Khi GuardDuty báo EC2 bị compromised (xâm phạm), quy trình xử lý là gì?
4. Security Hub aggregates (tổng hợp) findings từ những dịch vụ nào?
5. Macie phát hiện những loại dữ liệu nhạy cảm nào?
6. Detective dùng để làm gì khác với GuardDuty?
7. Làm thế nào bật threat detection cho toàn bộ Organizations?

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
