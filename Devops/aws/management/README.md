# AWS Management & Governance Services — Lộ Trình Học Tập

> Hướng dẫn toàn diện về **AWS Management and Governance** — bộ dịch vụ quản trị, giám sát, kiểm toán và tự động hóa vận hành hạ tầng đám mây, từ CloudWatch (Giám Sát Đám Mây), CloudTrail (Kiểm Toán), AWS Config (Tuân Thủ Cấu Hình) đến Organizations (Quản Lý Đa Tài Khoản) và Systems Manager (Quản Lý Hệ Thống).

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Các Năng Lực Cốt Lõi](#các-năng-lực-cốt-lõi)
3. [Tổng Quan Các Dịch Vụ](#tổng-quan-các-dịch-vụ)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)
6. [Bước Tiếp Theo](#bước-tiếp-theo)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] CloudWatch — Metrics (Chỉ Số), Logs (Nhật Ký), Alarms (Cảnh Báo)
- [ ] CloudTrail — Audit Logging (Ghi Nhật Ký Kiểm Toán), Event History (Lịch Sử Sự Kiện)
- [ ] AWS Config — Resource Inventory (Kiểm Kê Tài Nguyên), Compliance Rules (Quy Tắc Tuân Thủ)
- [ ] AWS Trusted Advisor — Best Practice Checks (Kiểm Tra Best Practice)

### **Giai Đoạn 2: Kỹ Năng Vận Hành (Tuần 3–5)**

- [ ] AWS Systems Manager (SSM) — Patch Manager, Session Manager, Parameter Store
- [ ] AWS CloudFormation — Infrastructure as Code (Hạ Tầng Dưới Dạng Mã), Stacks, StackSets
- [ ] AWS Organizations — Multi-Account Strategy (Chiến Lược Đa Tài Khoản), SCPs
- [ ] AWS Health Dashboard — Service Events, Personal Health Dashboard

### **Giai Đoạn 3: Quản Trị Nâng Cao (Tuần 6–8)**

- [ ] AWS Control Tower — Landing Zone (Vùng Hạ Cánh), Guardrails (Rào Chắn Quản Trị)
- [ ] AWS Service Catalog — Approved Portfolios (Danh Mục Được Phê Duyệt)
- [ ] AWS License Manager — License Tracking (Theo Dõi Giấy Phép)
- [ ] AWS Audit Manager — Evidence Collection (Thu Thập Bằng Chứng Kiểm Toán)

### **Giai Đoạn 4: Chuyên Sâu (Tuần 9+)**

- [ ] Observability Strategy (Chiến Lược Quan Sát) — CloudWatch + X-Ray + OpenTelemetry
- [ ] Compliance Automation (Tự Động Hóa Tuân Thủ) — Config Rules + Remediation
- [ ] Multi-Account Governance (Quản Trị Đa Tài Khoản) — Organizations + Control Tower
- [ ] Cost Governance (Quản Trị Chi Phí) — Budgets, Cost Anomaly Detection

---

## 🏢 Các Năng Lực Cốt Lõi

| Năng Lực                                        | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| ----------------------------------------------- | ---------- | --------- | ---------- |
| **CloudWatch Monitoring & Alerting**            | ⭐⭐⭐      | 2 tuần    | -          |
| **CloudTrail Audit & Compliance**               | ⭐⭐⭐      | 1 tuần    | -          |
| **AWS Config Resource Compliance**              | ⭐⭐⭐      | 1 tuần    | -          |
| **Systems Manager Operations**                  | ⭐⭐⭐      | 2 tuần    | -          |
| **CloudFormation Infrastructure as Code**       | ⭐⭐⭐      | 2 tuần    | -          |
| **Organizations & Multi-Account Management**    | ⭐⭐⭐      | 1 tuần    | -          |
| **Control Tower & Landing Zone**                | ⭐⭐        | 1 tuần    | -          |
| **Cost Governance & Budget Controls**           | ⭐⭐        | 1 tuần    | -          |
| **Audit Manager & Compliance Reporting**        | ⭐⭐        | 1 tuần    | -          |
| **Service Catalog & Approved Resources**        | ⭐          | 1 tuần    | -          |

---

## 🗂️ Tổng Quan Các Dịch Vụ

### 📁 **1. CloudWatch — Giám Sát & Quan Sát** (`01-cloudwatch/`)

**CloudWatch** là hệ thống quan sát (observability) trung tâm của AWS, thu thập dữ liệu từ hầu hết mọi dịch vụ.

**Các thành phần chính:**
- **Metrics** (Chỉ Số) — Dữ liệu dạng số theo thời gian: CPU, RAM, Request Count
- **Logs** (Nhật Ký) — Thu thập, lọc và phân tích log từ ứng dụng, hệ thống
- **Alarms** (Cảnh Báo) — Kích hoạt SNS, Auto Scaling, Lambda khi vượt ngưỡng
- **Dashboards** (Bảng Điều Khiển) — Trực quan hóa dữ liệu đa chiều
- **Container Insights** — Giám sát ECS, EKS, Kubernetes
- **Application Insights** — Phát hiện sự cố ứng dụng tự động

**Khi nào dùng:**
> Giám sát tất cả tài nguyên AWS, thiết lập cảnh báo, phân tích log tập trung, xây dựng dashboard vận hành.

---

### 📁 **2. CloudTrail — Kiểm Toán & Ghi Nhật Ký API** (`02-cloudtrail/`)

**CloudTrail** (Dấu Vết Đám Mây) ghi lại mọi lời gọi API (API Call) trên AWS — ai làm gì, lúc nào, từ đâu.

**Các thành phần chính:**
- **Event History** (Lịch Sử Sự Kiện) — 90 ngày lịch sử gần nhất miễn phí
- **Trails** (Dấu Vết) — Ghi liên tục vào S3 + CloudWatch Logs
- **Management Events** (Sự Kiện Quản Lý) — CreateInstance, DeleteBucket, AttachRole...
- **Data Events** (Sự Kiện Dữ Liệu) — S3 GetObject, Lambda Invoke (phải bật thêm)
- **Insights** — Phát hiện hoạt động bất thường (unusual API activity)
- **Organization Trail** — Ghi nhật ký cho toàn bộ multi-account organization

**Khi nào dùng:**
> Kiểm toán bảo mật, điều tra sự cố, tuân thủ PCI-DSS/HIPAA/SOC2, phát hiện thay đổi trái phép.

---

### 📁 **3. AWS Config — Tuân Thủ Cấu Hình Tài Nguyên** (`03-aws-config/`)

**AWS Config** theo dõi cấu hình (configuration) của tài nguyên AWS theo thời gian và đánh giá tuân thủ (compliance) theo rules.

**Các thành phần chính:**
- **Configuration Recorder** (Bộ Ghi Cấu Hình) — Ghi lại mọi thay đổi cấu hình
- **Configuration Snapshot** (Ảnh Chụp Cấu Hình) — Trạng thái toàn bộ tài nguyên tại một thời điểm
- **Config Rules** (Quy Tắc Config) — AWS Managed + Custom Lambda rules
- **Conformance Packs** (Gói Tuân Thủ) — Bộ rules đóng gói theo framework (CIS, PCI)
- **Remediation Actions** (Hành Động Khắc Phục) — Tự động sửa lỗi với SSM Automation
- **Aggregator** — Tổng hợp compliance từ nhiều account/region

**Khi nào dùng:**
> Kiểm tra cấu hình sai (misconfiguration), theo dõi drift, chứng minh tuân thủ, tự động remediation.

---

### 📁 **4. AWS Systems Manager (SSM) — Quản Lý Hệ Thống Vận Hành** (`04-systems-manager/`)

**SSM — AWS Systems Manager** là bộ công cụ vận hành (operational toolkit) toàn diện cho EC2, on-premises và multi-cloud.

**Các thành phần chính:**
- **Session Manager** — SSH không cần port 22, không cần key pair, ghi log đầy đủ
- **Patch Manager** — Vá lỗi (patching) tự động theo lịch, maintenance windows
- **Parameter Store** — Lưu trữ cấu hình và secret an toàn (không thay thế Secrets Manager)
- **Run Command** — Thực thi lệnh hàng loạt trên fleet EC2 không cần SSH
- **Inventory** — Thu thập thông tin phần mềm, cấu hình, ứng dụng đang chạy
- **Automation** — Runbook tự động hóa vận hành phức tạp (SSM Documents)
- **OpsCenter** — Tập hợp vấn đề vận hành (operational issues) vào một nơi
- **Distributor** — Phân phối phần mềm/agent tùy chỉnh

**Khi nào dùng:**
> Quản lý fleet EC2, patching tự động, truy cập instance không cần bastion host, quản lý secrets đơn giản.

---

### 📁 **5. AWS CloudFormation — Hạ Tầng Dưới Dạng Mã** (`05-cloudformation/`)

**CloudFormation — IaC (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã)** native của AWS, triển khai và quản lý tài nguyên qua template YAML/JSON.

**Các thành phần chính:**
- **Templates** (Mẫu) — File YAML/JSON mô tả tài nguyên mong muốn
- **Stacks** (Ngăn Xếp) — Tập hợp tài nguyên được quản lý cùng nhau
- **StackSets** — Triển khai stack đồng thời trên nhiều account/region
- **Change Sets** (Bộ Thay Đổi) — Xem trước thay đổi trước khi áp dụng
- **Drift Detection** (Phát Hiện Trôi Dạt) — Phát hiện thay đổi thủ công ngoài IaC
- **CloudFormation Registry** — Dùng resource types bên thứ ba
- **CDK — Cloud Development Kit** — Viết IaC bằng Python/TypeScript/Java thay vì YAML

**Khi nào dùng:**
> Tạo môi trường reproducible (có thể tái tạo), quản lý vòng đời tài nguyên, cross-account deployment.

---

### 📁 **6. AWS Organizations — Quản Lý Đa Tài Khoản** (`06-organizations/`)

**AWS Organizations** cho phép quản lý tập trung nhiều tài khoản AWS, áp dụng chính sách và hợp nhất billing.

**Các thành phần chính:**
- **Management Account** (Tài Khoản Quản Lý) — Root account điều hành toàn bộ
- **Member Accounts** (Tài Khoản Thành Viên) — Môi trường dev/staging/prod riêng biệt
- **OU — Organizational Units** (Đơn Vị Tổ Chức) — Nhóm account theo phòng ban/môi trường
- **SCP — Service Control Policies** (Chính Sách Kiểm Soát Dịch Vụ) — Giới hạn quyền toàn tài khoản
- **Consolidated Billing** (Hóa Đơn Hợp Nhất) — Tổng hợp chi phí, tận dụng volume discount
- **Delegated Administrator** — Ủy quyền quản lý dịch vụ cụ thể cho member account

**Khi nào dùng:**
> Doanh nghiệp nhiều team/môi trường, tách biệt security boundary, tối ưu chi phí qua consolidated billing.

---

### 📁 **7. AWS Control Tower — Landing Zone & Quản Trị** (`07-control-tower/`)

**Control Tower** (Tháp Kiểm Soát) tự động hóa việc thiết lập môi trường đa tài khoản an toàn theo AWS best practice.

**Các thành phần chính:**
- **Landing Zone** (Vùng Hạ Cánh) — Môi trường đa tài khoản được cấu hình sẵn, bảo mật
- **Guardrails** (Rào Chắn) — Chính sách quản trị bắt buộc (preventive) hoặc phát hiện (detective)
  - *Preventive Guardrails* — SCP chặn hành động vi phạm
  - *Detective Guardrails* — Config rules phát hiện vi phạm
- **Account Factory** (Xưởng Tài Khoản) — Tạo tài khoản mới tự động theo template chuẩn
- **Dashboard** — Trạng thái compliance của toàn bộ landing zone
- **Customizations for Control Tower (CfCT)** — Mở rộng với CloudFormation/SCPs tùy chỉnh

**Khi nào dùng:**
> Doanh nghiệp mới bắt đầu multi-account, cần landing zone chuẩn nhanh, quản trị tập trung.

---

### 📁 **8. AWS Trusted Advisor — Cố Vấn Best Practice** (`08-trusted-advisor/`)

**Trusted Advisor** (Cố Vấn Đáng Tin) phân tích môi trường AWS và đưa ra khuyến nghị cải thiện theo 5 lĩnh vực.

**5 lĩnh vực kiểm tra:**
- **Cost Optimization** (Tối Ưu Chi Phí) — Idle instances, unused EIPs, underutilized resources
- **Performance** (Hiệu Suất) — High-utilization instances, CloudFront headers
- **Security** (Bảo Mật) — MFA, public S3 buckets, security groups quá rộng
- **Fault Tolerance** (Khả Năng Chịu Lỗi) — Multi-AZ RDS, EBS snapshots, Route 53 health
- **Service Limits** (Giới Hạn Dịch Vụ) — Cảnh báo gần đạt quota

**Lưu ý:** Business/Enterprise Support mở khóa toàn bộ checks; Basic/Developer chỉ có ~7 checks cơ bản.

---

### 📁 **9. AWS Health Dashboard — Sức Khỏe Dịch Vụ** (`09-health-dashboard/`)

**AWS Health Dashboard** cung cấp thông tin về sự kiện ảnh hưởng đến dịch vụ và tài nguyên của bạn.

**Hai chế độ:**
- **Service Health Dashboard** — Trạng thái toàn cầu tất cả dịch vụ AWS công khai
- **Personal Health Dashboard (PHD)** — Sự kiện ảnh hưởng cụ thể đến account của bạn
  - Planned maintenance (bảo trì có lịch)
  - Account-specific issues (sự cố tài nguyên cụ thể)
  - Notifications qua EventBridge, SNS, Slack

---

### 📁 **10. Cost Management & Governance — Quản Trị Chi Phí** (`10-cost-governance/`)

**Bộ công cụ quản trị chi phí AWS:**

- **AWS Budgets** (Ngân Sách) — Đặt ngưỡng chi phí, nhận cảnh báo, hành động tự động
- **Cost Explorer** (Khám Phá Chi Phí) — Phân tích và dự báo chi phí theo service/tag/account
- **Cost Anomaly Detection** (Phát Hiện Chi Phí Bất Thường) — ML phát hiện chi tiêu đột biến
- **Savings Plans** (Kế Hoạch Tiết Kiệm) — Cam kết sử dụng để giảm chi phí compute
- **Resource Tagging Strategy** (Chiến Lược Gắn Nhãn) — Tag policy, cost allocation

---

### 📁 **11. Phỏng Vấn & Tình Huống Thực Tế** (`11-interview-prep/`)

- Top 20 câu hỏi phỏng vấn AWS Management & Governance
- Tình huống thiết kế hệ thống (System Design Scenarios)
- Câu chuyện sự cố theo phương pháp STAR
- So sánh dịch vụ (CloudTrail vs Config vs CloudWatch)
- Bài tập thực hành và case study

---

## 📊 Ma Trận Kỹ Năng

### Người Mới Bắt Đầu (0–1 năm)

- [ ] Hiểu CloudWatch Metrics và Alarms cơ bản
- [ ] Biết CloudTrail là gì và tại sao cần bật
- [ ] Hiểu khái niệm IaC với CloudFormation
- [ ] Biết Trusted Advisor kiểm tra những gì
- [ ] Hiểu tại sao cần nhiều AWS account

### Trung Cấp (1–3 năm)

- [ ] Thiết kế hệ thống monitoring với CloudWatch Dashboards + Alarms
- [ ] Cấu hình CloudTrail ghi log đa region và multi-account
- [ ] Viết Config Rules tùy chỉnh với Lambda
- [ ] Sử dụng SSM Session Manager thay bastion host
- [ ] Triển khai CloudFormation StackSets đa account
- [ ] Thiết kế OU structure và SCP cho Organizations

### Nâng Cao (3–5+ năm)

- [ ] Thiết kế observability platform (CloudWatch + X-Ray + OpenTelemetry)
- [ ] Xây dựng compliance-as-code pipeline (Config + Remediation + EventBridge)
- [ ] Thiết kế landing zone với Control Tower + Account Factory
- [ ] Kiến trúc cost governance đa account với Budgets + Anomaly Detection
- [ ] Tự động hóa incident response với SSM Automation + EventBridge
- [ ] Tích hợp SIEM với CloudTrail + Security Hub + GuardDuty

---

## 🎓 So Sánh Dịch Vụ Nhanh

### CloudTrail vs CloudWatch Logs vs AWS Config

| Tiêu Chí               | CloudTrail                      | CloudWatch Logs                | AWS Config                        |
| ---------------------- | ------------------------------- | ------------------------------ | --------------------------------- |
| **Mục đích**           | Ghi nhật ký API call            | Thu thập & phân tích log       | Theo dõi cấu hình tài nguyên      |
| **Câu hỏi trả lời**    | "Ai đã làm gì?"                 | "Ứng dụng đang hoạt động thế nào?" | "Cấu hình có đúng không?"     |
| **Dữ liệu**            | API events từ AWS Control Plane | Log tùy chỉnh từ ứng dụng/OS  | Resource configuration changes    |
| **Lưu trữ**            | S3 bucket                       | Log Groups (CloudWatch)        | S3 bucket                         |
| **Compliance use case**| Kiểm toán ai thay đổi gì        | Debug lỗi ứng dụng             | Xác nhận cấu hình đúng policy     |

### CloudFormation vs CDK vs Terraform

| Tiêu Chí                | CloudFormation         | AWS CDK                      | Terraform                     |
| ----------------------- | ---------------------- | ---------------------------- | ----------------------------- |
| **Ngôn ngữ**            | YAML / JSON            | Python, TypeScript, Java...  | HCL (HashiCorp Config Lang)   |
| **Native AWS**          | ✅ Có                  | ✅ Có (compile ra CFN)       | ❌ Multi-cloud                |
| **State management**    | AWS quản lý            | AWS quản lý                  | Tự quản lý (state file)       |
| **Learning curve**      | Thấp–Trung             | Cao (cần biết lập trình)     | Trung                         |
| **Multi-cloud**         | ❌ Chỉ AWS             | ❌ Chủ yếu AWS               | ✅ Có                         |
| **Best for**            | AWS native, đơn giản   | Developer-centric IaC        | Multi-cloud, ecosystem rộng   |

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                         | Thư Mục                                                             | Ưu Tiên       |
| ------------------------------- | ------------------------------------------------------------------- | ------------- |
| Bắt đầu từ đây                  | [INDEX.md](./INDEX.md)                                              | Đầu tiên      |
| Giám sát & cảnh báo             | [01-cloudwatch/README.md](./01-cloudwatch/README.md)                | Thiết yếu     |
| Kiểm toán & bảo mật             | [02-cloudtrail/README.md](./02-cloudtrail/README.md)                | Thiết yếu     |
| Tuân thủ cấu hình               | [03-aws-config/README.md](./03-aws-config/README.md)                | Thiết yếu     |
| Vận hành hệ thống               | [04-systems-manager/README.md](./04-systems-manager/README.md)      | Quan trọng    |
| Infrastructure as Code          | [05-cloudformation/README.md](./05-cloudformation/README.md)        | Quan trọng    |
| Đa tài khoản                    | [06-organizations/README.md](./06-organizations/README.md)          | Quan trọng    |
| Phỏng vấn                       | [11-interview-prep/](./11-interview-prep/)                          | Trước phỏng vấn |

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Phổ Biến Theo Chủ Đề

#### Monitoring & Observability (Giám Sát & Quan Sát)

- [ ] CloudWatch Metrics vs CloudWatch Logs — khác nhau như thế nào?
- [ ] Thiết kế hệ thống cảnh báo cho production application
- [ ] CloudWatch Alarm states: OK / ALARM / INSUFFICIENT_DATA
- [ ] Giải thích CloudWatch Container Insights

#### Audit & Compliance (Kiểm Toán & Tuân Thủ)

- [ ] CloudTrail vs AWS Config — khi nào dùng cái nào?
- [ ] Làm thế nào phát hiện ai đã xóa S3 bucket?
- [ ] Config Rules vs Security Hub — sự khác biệt?
- [ ] Thiết kế audit logging cho PCI-DSS compliance

#### Infrastructure as Code

- [ ] CloudFormation Stack vs StackSet — khác nhau gì?
- [ ] Change Set trong CloudFormation là gì?
- [ ] Xử lý CloudFormation drift như thế nào?
- [ ] CDK vs CloudFormation — khi nào dùng CDK?

#### Multi-Account Strategy (Chiến Lược Đa Tài Khoản)

- [ ] Tại sao nên dùng nhiều AWS account?
- [ ] SCP — Service Control Policy hoạt động thế nào?
- [ ] Control Tower vs Organizations — khác biệt?
- [ ] Thiết kế OU structure cho doanh nghiệp 50 team

#### Incident & Operations (Sự Cố & Vận Hành)

- [ ] Dùng SSM Session Manager thay SSH như thế nào?
- [ ] SSM Parameter Store vs Secrets Manager — khi nào dùng cái nào?
- [ ] Thiết kế automated remediation với AWS Config + SSM Automation
- [ ] Phân biệt SSM Run Command vs SSM Automation

Xem `11-interview-prep/` để tra cứu đầy đủ Q&A.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc nhận vai trò mới, kiểm tra:

- [ ] Có thể giải thích sự khác nhau giữa CloudTrail, CloudWatch và Config không?
- [ ] Biết thiết kế monitoring strategy cho production system không?
- [ ] Hiểu SCPs trong AWS Organizations hoạt động thế nào không?
- [ ] Biết cấu hình SSM Session Manager không cần bastion host không?
- [ ] Có thể viết CloudFormation template cơ bản không?
- [ ] Hiểu tại sao nên dùng multi-account strategy không?
- [ ] Biết Control Tower Landing Zone gồm những gì không?
- [ ] Có thể thiết kế cost governance cho môi trường đa account không?
- [ ] Hiểu các loại Guardrails trong Control Tower không?
- [ ] Biết Config Remediation Actions hoạt động thế nào không?

---

## 🚀 Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Xem INDEX.md để nắm toàn bộ cấu trúc
├─ 3️⃣  Bắt đầu với 01-cloudwatch/ (nền tảng monitoring)
├─ 4️⃣  Tiếp tục 02-cloudtrail/ (audit thiết yếu)
├─ 5️⃣  Học 03-aws-config/ (compliance automation)
├─ 6️⃣  Thực hành với 04-systems-manager/ (vận hành thực tế)
├─ 7️⃣  Nắm IaC qua 05-cloudformation/
└─ 8️⃣  Chuẩn bị phỏng vấn tại 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
