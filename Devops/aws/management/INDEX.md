# AWS Management & Governance — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về **AWS Management and Governance Services** — giám sát, kiểm toán, tuân thủ, vận hành và quản trị hạ tầng đám mây quy mô doanh nghiệp.

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/management/
├── README.md                                            [BẮT ĐẦU TỪ ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                             Chỉ mục đầy đủ (file này)
│
├── 01-cloudwatch/
│   ├── README.md                                        ✅ Tổng quan CloudWatch & observability strategy
│   ├── 1-metrics-namespaces.md                          ✅ Metrics, Namespaces, Dimensions, Statistics
│   ├── 2-alarms-composite.md                            ✅ Alarms, Composite Alarms, SNS actions
│   ├── 3-logs-insights.md                               ✅ Log Groups, Log Streams, Insights queries
│   ├── 4-dashboards-widgets.md                          ✅ Dashboards, Widgets, Cross-account view
│   ├── 5-container-application-insights.md              ✅ Container Insights, Application Insights
│   └── 6-cloudwatch-agent.md                            ✅ CW Agent cài trên EC2, custom metrics
│
├── 02-cloudtrail/
│   ├── README.md                                        ✅ Tổng quan CloudTrail & audit logging
│   ├── 1-event-types.md                                 ✅ Management Events, Data Events, Insights Events
│   ├── 2-trails-configuration.md                        ✅ Single-region vs multi-region, S3 + CW Logs
│   ├── 3-organization-trail.md                          ✅ Centralized logging cho multi-account
│   ├── 4-cloudtrail-insights.md                         ✅ Anomaly detection trong API activity
│   └── 5-forensics-investigation.md                     ✅ Điều tra sự cố, phân tích event history
│
├── 03-aws-config/
│   ├── README.md                                        ✅ Tổng quan AWS Config & compliance automation
│   ├── 1-configuration-recorder.md                      ✅ Recorder setup, resource types, S3 delivery
│   ├── 2-config-rules.md                                ✅ Managed rules, Custom Lambda rules, scope
│   ├── 3-conformance-packs.md                           ✅ CIS, PCI-DSS, NIST conformance packs
│   ├── 4-remediation.md                                 ✅ Automatic remediation với SSM Automation
│   └── 5-aggregator-multiregion.md                      ✅ Aggregator, multi-account compliance view
│
├── 04-systems-manager/
│   ├── README.md                                        ✅ Tổng quan SSM & operational toolkit
│   ├── 1-session-manager.md                             ✅ SSH-less access, audit logs, port forwarding
│   ├── 2-patch-manager.md                               ✅ Patch baselines, Patch groups, maintenance windows
│   ├── 3-parameter-store.md                             ✅ Standard vs Advanced, SecureString, versioning
│   ├── 4-run-command-automation.md                      ✅ Run Command, SSM Documents, Automation runbooks
│   ├── 5-inventory-compliance.md                        ✅ Software inventory, config compliance
│   └── 6-distributor-opscenter.md                       ✅ Package distribution, OpsItems, OpsCenter
│
├── 05-cloudformation/
│   ├── README.md                                        ✅ Tổng quan CloudFormation & IaC trên AWS
│   ├── 1-template-anatomy.md                            ✅ Parameters, Resources, Outputs, Mappings, Conditions
│   ├── 2-stacks-lifecycle.md                            ✅ Create/Update/Delete stack, rollback, events
│   ├── 3-stacksets-multiregion.md                       ✅ StackSets với Organizations, deployment targets
│   ├── 4-change-sets-drift.md                           ✅ Change Sets, Drift Detection, import resources
│   ├── 5-custom-resources.md                            ✅ Custom Resources với Lambda, resource providers
│   └── 6-cdk-comparison.md                              ✅ CDK vs CloudFormation, khi nào dùng CDK
│
├── 06-organizations/
│   ├── README.md                                        ✅ Tổng quan AWS Organizations & multi-account strategy
│   ├── 1-account-structure.md                           ✅ Management account, Member accounts, OU hierarchy
│   ├── 2-scp-policies.md                                ✅ SCP deny list vs allow list, inheritance
│   ├── 3-consolidated-billing.md                        ✅ Billing consolidation, volume discounts, RI sharing
│   ├── 4-delegated-admin.md                             ✅ Delegated administrator pattern, trusted access
│   └── 5-multi-account-patterns.md                      ✅ Account vending, account types, OU design patterns
│
├── 07-control-tower/
│   ├── README.md                                        ✅ Tổng quan Control Tower & landing zone
│   ├── 1-landing-zone-setup.md                          ✅ Landing Zone components, Log Archive, Audit accounts
│   ├── 2-guardrails.md                                  ✅ Preventive vs Detective guardrails, mandatory vs elective
│   ├── 3-account-factory.md                             ✅ Account Factory, Account Factory for Terraform (AFT)
│   └── 4-customizations-cfct.md                         ✅ Customizations for Control Tower (CfCT)
│
├── 08-trusted-advisor/
│   ├── README.md                                        ✅ Tổng quan Trusted Advisor & best practice checks
│   ├── 1-check-categories.md                            ✅ 5 categories: Cost, Performance, Security, FT, Limits
│   └── 2-programmatic-access.md                         ✅ Support API, EventBridge integration, automation
│
├── 09-health-dashboard/
│   ├── README.md                                        ✅ Tổng quan Health Dashboard & service events
│   ├── 1-personal-health.md                             ✅ Personal Health Dashboard, event types, notifications
│   └── 2-eventbridge-integration.md                     ✅ Tự động hóa response với EventBridge + Lambda
│
├── 10-cost-governance/
│   ├── README.md                                        ✅ Tổng quan cost governance & FinOps trên AWS
│   ├── 1-budgets-alerts.md                              ✅ AWS Budgets, budget types, alert actions
│   ├── 2-cost-explorer.md                               ✅ Cost Explorer, filtering, forecasting, rightsizing
│   ├── 3-anomaly-detection.md                           ✅ Cost Anomaly Detection, ML models, monitors
│   ├── 4-tagging-strategy.md                            ✅ Tag policies, cost allocation tags, enforcement
│   └── 5-savings-plans-ri.md                            ✅ Savings Plans vs Reserved Instances, commitment strategies
│
└── 11-interview-prep/
    ├── README.md                                        ✅ Hướng dẫn ôn tập phỏng vấn & lộ trình 2 tuần
    ├── INTERVIEW_GUIDE.md                               ✅ Top 30 câu hỏi AWS Management & Governance
    ├── service-comparison.md                            ✅ So sánh chi tiết các dịch vụ dễ nhầm lẫn (9 cặp)
    ├── system-design-scenarios.md                       ✅ 6 tình huống thiết kế hệ thống thực tế
    ├── star-stories.md                                  ✅ 5 mẫu câu chuyện sự cố theo STAR + template
    └── 90-day-study-plan.md                             ✅ Kế hoạch học 90 ngày có cấu trúc theo tuần
```

---

## ✅ Trạng Thái Tạo Nội Dung

| Chủ Đề                                    | File                                        | Trạng Thái | Chất Lượng    |
| ----------------------------------------- | ------------------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**                  | README.md                                   | ✅         | Toàn diện     |
| **Chỉ Mục Đầy Đủ**                        | INDEX.md (file này)                         | ✅         | Toàn diện     |
| **CloudWatch — Giám Sát**                 | 01-cloudwatch/ (6 files)                    | ✅ Hoàn thành | Toàn diện  |
| **CloudTrail — Kiểm Toán**               | 02-cloudtrail/ (6 files)                    | ✅ Hoàn thành | Toàn diện  |
| **AWS Config — Tuân Thủ**                 | 03-aws-config/ (6 files)                    | ✅ Hoàn thành | Toàn diện  |
| **Systems Manager — Vận Hành**            | 04-systems-manager/ (7 files)               | ✅ Hoàn thành | Toàn diện  |
| **CloudFormation — IaC**                  | 05-cloudformation/ (7 files)                | ✅ Hoàn thành | Toàn diện  |
| **Organizations — Đa Tài Khoản**          | 06-organizations/ (6 files)                 | ✅ Hoàn thành | Toàn diện  |
| **Control Tower — Landing Zone**          | 07-control-tower/ (5 files)                 | ✅ Hoàn thành | Toàn diện   |
| **Trusted Advisor — Best Practice**       | 08-trusted-advisor/ (3 files)               | ✅ Hoàn thành | Toàn diện  |
| **Health Dashboard — Sức Khỏe Dịch Vụ** | 09-health-dashboard/ (2 files)              | ✅ Hoàn thành | -             |
| **Cost Governance — Quản Trị Chi Phí**   | 10-cost-governance/ (6 files)               | ✅ Hoàn thành | Toàn diện   |
| **Phỏng Vấn & Tình Huống**               | 11-interview-prep/ (6 files)                | ✅ Hoàn thành | Toàn diện  |

---

## 🎯 Thứ Tự Ưu Tiên Tạo Nội Dung

### Ưu Tiên Cao (Kỹ năng cốt lõi — hỏi nhiều nhất trong phỏng vấn)

- [x] `01-cloudwatch/README.md` — Monitoring strategy, Metrics, Alarms, Logs Insights
- [x] `02-cloudtrail/README.md` — Audit logging, Event types, Organization Trail
- [x] `03-aws-config/README.md` — Config Rules, Conformance Packs, Remediation
- [x] `04-systems-manager/README.md` — Session Manager, Patch Manager, Parameter Store
- [x] `05-cloudformation/README.md` — Template anatomy, Stacks, StackSets, CDK so sánh

### Ưu Tiên Trung Bình (Kỹ năng nâng cao — doanh nghiệp lớn)

- [x] `06-organizations/README.md` — OU design, SCPs, multi-account patterns
- [x] `07-control-tower/README.md` — Landing Zone, Guardrails, Account Factory
- [x] `10-cost-governance/README.md` — Budgets, Cost Explorer, Tagging strategy
- [x] `11-interview-prep/INTERVIEW_GUIDE.md` — Top 30 câu hỏi phỏng vấn

### Ưu Tiên Thấp Hơn (Tham khảo & chuyên sâu)

- [x] `08-trusted-advisor/` — Best practice checks tự động
- [x] `09-health-dashboard/` — Service health & EventBridge automation
- [x] `11-interview-prep/service-comparison.md` — Bảng so sánh dịch vụ
- [x] `11-interview-prep/system-design-scenarios.md` — Tình huống thiết kế

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Dành Cho Tự Học

```
1. Đọc README.md để nắm tổng quan và lộ trình
2. Chọn giai đoạn phù hợp (Người mới / Trung cấp / Nâng cao)
3. Học tuần tự: CloudWatch → CloudTrail → Config → SSM → CloudFormation
4. Thực hành trên AWS Free Tier hoặc lab environment
5. Xây dựng portfolio project tích hợp nhiều dịch vụ
```

### Dành Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md trước
2. Nắm vững 3 dịch vụ hay nhầm lẫn: CloudTrail vs Config vs CloudWatch
3. Luyện giải thích multi-account strategy và SCPs
4. Chuẩn bị 2–3 câu chuyện sự cố theo phương pháp STAR
5. Nắm IaC: CloudFormation vs CDK vs Terraform khi nào dùng cái nào
```

### Dành Cho Vai Trò DevOps/Cloud Engineer

```
Dùng làm tài liệu tham khảo:
- Thiết lập monitoring: Đọc 01-cloudwatch/
- Audit & compliance: Đọc 02-cloudtrail/ + 03-aws-config/
- Vận hành fleet EC2: Đọc 04-systems-manager/
- Triển khai IaC: Đọc 05-cloudformation/
- Thiết kế multi-account: Đọc 06-organizations/ + 07-control-tower/
```

### Dành Cho Thiết Kế Hệ Thống

```
1. Bắt đầu với compliance requirements (PCI-DSS, HIPAA, SOC2?)
2. Thiết kế account structure (Organizations + OU)
3. Xác định guardrails cần thiết (Control Tower)
4. Thiết kế observability (CloudWatch + CloudTrail + Config)
5. Lên kế hoạch IaC (CloudFormation/CDK + StackSets)
6. Xây dựng cost governance (Budgets + Anomaly Detection + Tagging)
```

---

## 📊 Ước Tính Thời Gian Học

| Phần                                    | Thời Gian    | Độ Khó     | Ưu Tiên |
| --------------------------------------- | ------------ | ---------- | ------- |
| CloudWatch Monitoring                   | 6–8 giờ      | ⭐⭐        | Bắt buộc |
| CloudTrail Audit                        | 4–6 giờ      | ⭐⭐        | Bắt buộc |
| AWS Config Compliance                   | 6–8 giờ      | ⭐⭐        | Bắt buộc |
| Systems Manager Operations              | 8–10 giờ     | ⭐⭐⭐      | Bắt buộc |
| CloudFormation & CDK                    | 10–15 giờ    | ⭐⭐⭐      | Bắt buộc |
| AWS Organizations & SCPs                | 6–8 giờ      | ⭐⭐        | Nên có   |
| Control Tower Landing Zone              | 4–6 giờ      | ⭐⭐⭐      | Nên có   |
| Cost Governance                         | 4–6 giờ      | ⭐⭐        | Nên có   |
| Trusted Advisor & Health Dashboard      | 2–3 giờ      | ⭐          | Tốt hơn |
| Phỏng Vấn & Tình Huống                  | 8–10 giờ     | ⭐⭐⭐      | Bắt buộc |

**Tổng: 60–80 giờ để có kiến thức AWS Management & Governance toàn diện**

---

## 📈 Tracker Tiến Độ Học Tập

Sao chép và tự theo dõi tiến độ:

```markdown
## AWS Management & Governance — Tiến Độ

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)
- [ ] CloudWatch Metrics & Namespaces
- [ ] CloudWatch Alarms & Composite Alarms
- [ ] CloudWatch Logs & Logs Insights
- [ ] CloudTrail bật và cấu hình Trail
- [ ] CloudTrail Event History điều tra sự cố
- [ ] AWS Config Recorder setup
- [ ] AWS Config Rules (Managed + Custom)
- [ ] Trusted Advisor 5 categories

### Giai Đoạn 2: Vận Hành (Tuần 3–5)
- [ ] SSM Session Manager thay SSH
- [ ] SSM Patch Manager lên lịch patching
- [ ] SSM Parameter Store vs Secrets Manager
- [ ] SSM Run Command bulk execution
- [ ] CloudFormation template cơ bản
- [ ] CloudFormation Stack lifecycle
- [ ] CloudFormation Change Sets
- [ ] CloudFormation StackSets đa account

### Giai Đoạn 3: Quản Trị (Tuần 6–8)
- [ ] AWS Organizations OU structure
- [ ] SCPs — allow list vs deny list
- [ ] Consolidated Billing & volume discounts
- [ ] Control Tower Landing Zone
- [ ] Guardrails — Preventive vs Detective
- [ ] Account Factory tạo account tự động
- [ ] AWS Budgets & alerts
- [ ] Cost Anomaly Detection

### Giai Đoạn 4: Chuyên Sâu (Tuần 9+)
- [ ] Config Conformance Packs (CIS/PCI)
- [ ] Config + SSM Remediation tự động
- [ ] CloudWatch Dashboards cross-account
- [ ] Organization Trail centralized logging
- [ ] CDK — Cloud Development Kit
- [ ] FinOps: Tagging policy + Cost allocation
- [ ] Security Hub integration với CloudTrail + Config
- [ ] SIEM integration patterns
```

---

## 🎓 Các Mức Kỹ Năng Được Hỗ Trợ

### Người Mới Bắt Đầu (0–1 năm kinh nghiệm)

- [ ] Hiểu CloudWatch Metrics và Alarms
- [ ] Biết bật và đọc CloudTrail logs
- [ ] Hiểu khái niệm IaC với CloudFormation
- [ ] Biết Trusted Advisor kiểm tra những gì
- [ ] Hiểu lý do cần nhiều AWS account

**Thời gian để thành thạo:** 1–2 tháng

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Thiết kế observability stack hoàn chỉnh
- [ ] Cấu hình Organization Trail multi-account
- [ ] Viết Config Rules và Remediation
- [ ] Triển khai CloudFormation StackSets
- [ ] Thiết kế OU hierarchy và SCPs

**Thời gian để nâng cao:** 2–3 tháng

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Kiến trúc compliance-as-code platform
- [ ] Thiết kế landing zone từ đầu
- [ ] Tích hợp SIEM với AWS audit services
- [ ] Xây dựng automated remediation pipeline
- [ ] FinOps: Governance đa account phức tạp

**Thời gian để tinh thông:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                               | Vị Trí                                                                    |
| ------------------------------------- | ------------------------------------------------------------------------- |
| Tổng quan lộ trình                    | [README.md](README.md)                                                    |
| Setup monitoring & alerting           | [01-cloudwatch/README.md](01-cloudwatch/README.md)                        |
| Điều tra ai xóa tài nguyên            | [02-cloudtrail/README.md](02-cloudtrail/README.md)                        |
| Kiểm tra compliance cấu hình          | [03-aws-config/README.md](03-aws-config/README.md)                        |
| Truy cập EC2 không cần SSH            | [04-systems-manager/README.md](04-systems-manager/README.md)              |
| Triển khai hạ tầng bằng code          | [05-cloudformation/README.md](05-cloudformation/README.md)                |
| Quản lý nhiều AWS account             | [06-organizations/README.md](06-organizations/README.md)                  |
| Tạo landing zone chuẩn                | [07-control-tower/README.md](07-control-tower/README.md)                  |
| Kiểm soát chi phí tự động             | [10-cost-governance/README.md](10-cost-governance/README.md)              |
| Chuẩn bị phỏng vấn                    | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md) |

---

## 💡 Mẹo Học Hiệu Quả

1. **Học qua thực hành:** AWS Free Tier đủ để thử CloudWatch, CloudTrail, Config cơ bản
2. **Hiểu "Tại sao" trước "Cái gì":** Biết lý do cần CloudTrail giúp nhớ lâu hơn
3. **So sánh dịch vụ tương tự:** CloudTrail vs Config vs CloudWatch — hay bị nhầm nhất
4. **Xây dựng mental model:** Vẽ sơ đồ luồng dữ liệu từ tài nguyên → audit service
5. **Thực hành IaC:** Viết CloudFormation template tay ít nhất 1 lần trước khi dùng CDK
6. **Hiểu multi-account từ sớm:** Đây là yêu cầu chuẩn của mọi môi trường production
7. **Cập nhật thường xuyên:** AWS ra tính năng mới hàng tuần — theo dõi AWS What's New
8. **Tạo lab environment:** Dùng Organization với 2–3 account free để thực hành SCPs

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống (living document). Chào đón đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm ví dụ thực tế từ kinh nghiệm
- [ ] Bổ sung section cho dịch vụ chưa có
- [ ] Giải thích rõ hơn khái niệm phức tạp
- [ ] Thêm bài tập thực hành (hands-on labs)

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 2.0 (11-interview-prep hoàn thành — Knowledge Base đầy đủ)
**Trạng Thái:** ✅ README.md & INDEX.md hoàn thành | ✅ 01-cloudwatch/ (7 files) hoàn thành | ✅ 02-cloudtrail/ (6 files) hoàn thành | ✅ 03-aws-config/ (6 files) hoàn thành | ✅ 04-systems-manager/ (7 files) hoàn thành | ✅ 05-cloudformation/ (7 files) hoàn thành | ✅ 06-organizations/ (6 files) hoàn thành | ✅ 07-control-tower/ (5 files) hoàn thành | ✅ 08-trusted-advisor/ (3 files) hoàn thành | ✅ 09-health-dashboard/ (3 files) hoàn thành | ✅ 10-cost-governance/ (6 files) hoàn thành | ✅ 11-interview-prep/ (6 files) hoàn thành
