# So Sánh Chi Tiết Dịch Vụ AWS Management & Governance

> Tập trung vào các cặp/nhóm dịch vụ dễ nhầm nhất trong phỏng vấn. Mỗi so sánh có bảng, câu hỏi quyết định (decision guide) và ví dụ thực tế.

---

## 1. CloudTrail vs AWS Config vs CloudWatch Logs

Đây là bộ ba hay bị nhầm nhất. Nhớ theo công thức: **"Ai làm gì" / "Cấu hình đúng không" / "Ứng dụng chạy thế nào"**.

### Bảng So Sánh

| Tiêu Chí                   | CloudTrail                                | AWS Config                                   | CloudWatch Logs                              |
| -------------------------- | ----------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| **Câu hỏi trả lời**        | Ai đã làm gì, lúc nào, từ đâu?           | Cấu hình tài nguyên có đúng policy không?   | Ứng dụng/hệ thống đang hoạt động thế nào?   |
| **Nguồn dữ liệu**          | AWS Control Plane API calls               | Thay đổi cấu hình tài nguyên AWS            | Log từ ứng dụng, OS, Lambda, VPC Flow...    |
| **Dạng dữ liệu**           | Structured JSON events (API calls)        | Configuration items (JSON resource state)   | Unstructured / semi-structured log lines    |
| **Thời gian lưu trữ**      | S3 (không giới hạn), Event History 90 ngày | S3 (không giới hạn)                        | Log Groups — cấu hình retention 1–3653 ngày |
| **Real-time**              | Gần real-time (vài phút delay)            | Periodic hoặc config change trigger         | Real-time                                    |
| **Dùng để làm gì**         | Audit, điều tra sự cố, pháp lý            | Compliance check, drift detection           | Debug, monitoring, alerting ứng dụng        |
| **Query tool**             | Athena (S3), CloudTrail Lake              | Config Timeline, Aggregator                 | CloudWatch Logs Insights                     |
| **Tự động hóa**            | EventBridge → Lambda khi có sự kiện       | Remediation Actions (SSM Automation)        | Metric Filters → Alarms → SNS/Lambda        |
| **Chi phí đáng chú ý**    | Data Events tính phí; Insights tính phí  | Mỗi config item ghi lại tính phí           | Tính phí theo GB ingest + storage           |

### Decision Guide — Dùng Cái Nào?

```
Câu hỏi: "Ai đã xóa S3 bucket lúc 3 giờ sáng?"
→ CloudTrail (API call history)

Câu hỏi: "S3 bucket nào đang bật public access?"
→ AWS Config (resource state + rules)

Câu hỏi: "Lambda function có lỗi timeout không?"
→ CloudWatch Logs (application logs)

Câu hỏi: "Khi nào security group thay đổi inbound rule?"
→ AWS Config (configuration timeline)
  HOẶC CloudTrail (nếu muốn biết ai thay đổi)
  → Thường kết hợp CẢ HAI
```

### Ví Dụ Thực Tế

**Scenario:** Phát hiện tài nguyên bị thay đổi trái phép.

1. **AWS Config** phát hiện security group thay đổi (detective guardrail)
2. **CloudTrail** xác định ai đã gọi `AuthorizeSecurityGroupIngress`
3. **CloudWatch Logs** xác nhận application log không bị ảnh hưởng
4. **EventBridge** trigger Lambda tự động revert thay đổi

---

## 2. SSM Parameter Store vs AWS Secrets Manager

### Bảng So Sánh

| Tiêu Chí                       | SSM Parameter Store                      | AWS Secrets Manager                          |
| ------------------------------ | ---------------------------------------- | -------------------------------------------- |
| **Mục đích chính**             | Config, metadata, simple secrets         | Secrets có lifecycle management đầy đủ       |
| **Auto-rotation**              | Không có native — phải tự viết Lambda   | Có, built-in cho RDS, Redshift, DocumentDB   |
| **Chi phí**                    | Standard: Miễn phí; Advanced: tính phí  | $0.40/secret/tháng + $0.05/10K API calls    |
| **Kích thước tối đa**          | Standard: 4KB; Advanced: 8KB            | 64KB                                         |
| **Versioning**                 | Có (Label-based)                         | Có (Version ID + staging labels)             |
| **Cross-account access**       | Hạn chế (cần Resource Policy)           | Hỗ trợ tốt qua Resource Policy              |
| **Encryption**                 | AWS KMS (SecureString tier)             | AWS KMS (luôn mã hóa)                        |
| **Hierarchy / Path**           | /app/prod/db-password (path-based)      | Tên phẳng hoặc có prefix tùy chỉnh          |
| **Parameter types**            | String, StringList, SecureString        | Chỉ có secret value (JSON hoặc plaintext)   |
| **Tích hợp ECS/EKS**          | Inject vào container environment vars   | Inject vào container environment vars        |
| **Audit trail**                | CloudTrail ghi mọi GetParameter         | CloudTrail ghi mọi GetSecretValue            |

### Decision Guide — Dùng Cái Nào?

```
→ Database password cần rotate tự động mỗi 30 ngày
  → Secrets Manager (built-in RDS rotation)

→ Config file của ứng dụng: DB_HOST, APP_ENV, FEATURE_FLAGS
  → Parameter Store (miễn phí, phân cấp path)

→ API key của bên thứ ba không cần rotation
  → Parameter Store SecureString (tiết kiệm chi phí)

→ RDS credential chia sẻ với nhiều account
  → Secrets Manager (cross-account Resource Policy)

→ Lưu trữ hàng trăm config variables nhỏ
  → Parameter Store (free tier, path hierarchy)
```

### Ví Dụ Cấu Trúc Parameter Store

```
/myapp/
  production/
    database/
      host          → db.prod.us-east-1.rds.amazonaws.com
      port          → 5432
      name          → appdb
      password      → (SecureString, KMS encrypted)
    redis/
      endpoint      → cache.prod.us-east-1.cache.amazonaws.com
    feature-flags/
      new-ui        → true
      dark-mode     → false
```

---

## 3. AWS Organizations vs AWS Control Tower

### Bảng So Sánh

| Tiêu Chí                      | AWS Organizations                        | AWS Control Tower                            |
| ----------------------------- | ---------------------------------------- | -------------------------------------------- |
| **Là gì**                     | Dịch vụ core quản lý đa tài khoản       | Orchestration layer trên Organizations       |
| **Tạo account**               | API thủ công hoặc script                 | Account Factory — wizard + tự động hóa      |
| **SCPs**                      | Tự tạo và gắn thủ công                  | Guardrails áp SCPs (preventive) tự động      |
| **Config Rules**              | Phải tự thiết lập mỗi account           | Detective Guardrails deploy Config Rules tự  |
| **CloudTrail**                | Phải tự tạo Organization Trail          | Tự động bật Organization Trail              |
| **Log centralization**        | Tự cấu hình S3 bucket policy            | Log Archive account được tạo sẵn            |
| **Audit account**             | Tự tạo                                  | Audit account được tạo tự động              |
| **Customization**             | Tự do hoàn toàn                          | Dùng CfCT — Customizations for Control Tower |
| **Guardrails mandatory**      | Không áp dụng                            | Một số guardrail bắt buộc, không tắt được   |
| **Chi phí**                   | Miễn phí (chỉ trả cho resources)        | Miễn phí (chỉ trả cho underlying services)  |
| **Phù hợp với**               | Mọi quy mô, cần kiểm soát tùy chỉnh    | Doanh nghiệp cần landing zone nhanh, chuẩn  |

### Decision Guide — Dùng Cái Nào?

```
→ Startup nhỏ, 2–5 account, team nhỏ
  → Organizations thuần (đủ dùng, không overhead)

→ Enterprise mới bắt đầu cloud, cần landing zone trong 1–2 tuần
  → Control Tower (provisioning tự động, guardrails sẵn)

→ Đã có Organizations từ trước, muốn thêm governance
  → Có thể enroll vào Control Tower hoặc tự build

→ Cần tùy chỉnh sâu SCP và account structure khác chuẩn AWS
  → Organizations thuần (Control Tower opinionated)

→ Cần Account Factory for Terraform (AFT) — Vending Machine tài khoản
  → Control Tower + AFT
```

### Quan Hệ Giữa Hai Dịch Vụ

```
AWS Organizations (nền tảng)
├── Management Account
├── OU: Security
│   ├── Log Archive Account     ← Control Tower tạo tự động
│   └── Audit Account           ← Control Tower tạo tự động
├── OU: Infrastructure
│   └── Shared Services Account
└── OU: Workloads
    ├── OU: Production
    │   └── Prod Account
    └── OU: Development
        └── Dev Account

Control Tower (lớp automation trên)
├── Guardrails (áp SCPs + Config Rules lên OU)
├── Account Factory (tạo account mới theo template)
└── Dashboard (compliance overview)
```

---

## 4. CloudFormation vs CDK vs Terraform

### Bảng So Sánh

| Tiêu Chí                  | CloudFormation              | AWS CDK                          | Terraform                         |
| ------------------------- | --------------------------- | -------------------------------- | --------------------------------- |
| **Ngôn ngữ**              | YAML / JSON (declarative)   | TypeScript, Python, Java, Go...  | HCL — HashiCorp Configuration Language |
| **Native AWS**            | ✅ 100% native              | ✅ Compile ra CloudFormation     | ❌ Multi-cloud, provider riêng    |
| **State management**      | AWS quản lý tự động         | AWS quản lý (qua CFN)           | Tự quản lý state file             |
| **Multi-cloud**           | ❌ Chỉ AWS                  | ❌ Chủ yếu AWS                   | ✅ AWS, GCP, Azure, on-prem       |
| **Learning curve**        | Thấp (YAML quen thuộc)     | Cao (cần biết lập trình)         | Trung bình (HCL dễ học)           |
| **Abstraction**           | Low-level (mỗi resource)   | High-level (Constructs library)  | Trung bình (modules, providers)   |
| **Type safety**           | Không có                    | Có (TypeScript strict)           | Partial (variable types)          |
| **Testing**               | cfn-lint, taskcat           | Jest unit tests, cdk assertions  | Terratest                         |
| **Rollback**              | Automatic trên failure      | Automatic (qua CFN)              | Thủ công hoặc cần script          |
| **Drift detection**       | ✅ Built-in                 | ✅ Built-in (qua CFN)            | terraform plan detect drift        |
| **Ecosystem**             | AWS native constructs       | CDK Constructs Library (L1/L2/L3)| Terraform Registry (10K+ modules) |
| **Best for**              | AWS native, đơn giản, chuẩn | Developer-centric, reuse cao    | Multi-cloud, team đã dùng TF      |

### CDK Constructs — 3 Cấp Độ

```
L1 (Level 1) — CfnResource: 1:1 với CloudFormation resource
  → CfnBucket, CfnFunction, CfnRole
  → Kiểm soát tối đa, verbose nhất

L2 (Level 2) — Resource: Higher-level với defaults thông minh
  → Bucket, Function, Role
  → Phổ biến nhất, balance giữa control và convenience

L3 (Level 3) — Pattern: Giải pháp multi-resource hoàn chỉnh
  → ApplicationLoadBalancedFargateService, LambdaRestApi
  → Nhanh nhất để build, ít control nhất
```

### Decision Guide — Dùng Cái Nào?

```
→ Chỉ dùng AWS, team không biết lập trình nhiều
  → CloudFormation (YAML quen thuộc, AWS hỗ trợ 100%)

→ Team developer mạnh, muốn reuse logic, tạo internal framework
  → CDK (TypeScript/Python, unit testable, constructs library)

→ Multi-cloud (AWS + GCP + Azure), hoặc quản lý Kubernetes, DNS, GitHub
  → Terraform (provider ecosystem rộng nhất)

→ Đang dùng CloudFormation, muốn migrate sang CDK
  → CDK (compile ra CFN — có thể import existing stacks)

→ StackSets đa account/region bắt buộc
  → CloudFormation StackSets hoặc CDK Pipeline + CDK
```

---

## 5. CloudWatch Alarm vs EventBridge vs SNS

Ba dịch vụ này hay bị nhầm khi thiết kế notification/automation pipeline.

### Bảng So Sánh

| Tiêu Chí                | CloudWatch Alarm                  | Amazon EventBridge                      | Amazon SNS                            |
| ----------------------- | --------------------------------- | --------------------------------------- | ------------------------------------- |
| **Mục đích**            | Kích hoạt dựa trên metric threshold | Route events đến targets                | Pub/Sub messaging                     |
| **Trigger từ**         | CloudWatch Metrics                 | AWS events, custom events, Schedule     | Application code, AWS services        |
| **Target/Action**       | SNS, Auto Scaling, EC2 actions    | Lambda, SQS, SNS, Step Functions...     | Lambda, SQS, HTTP, Email, SMS         |
| **Filter logic**        | Threshold + period + datapoints   | Event pattern matching (JSON)           | Không filter — fan-out tất cả        |
| **Latency**             | ~1 phút (evaluation period)       | Near real-time (<1 giây)               | Near real-time                        |
| **State machine**       | OK / ALARM / INSUFFICIENT_DATA    | Không có state                         | Không có state                        |
| **Use case**            | CPU > 80% → scale up               | EC2 state change → notify Slack         | Alert broadcast đến nhiều subscribers |

### Khi Nào Kết Hợp?

```
Pattern phổ biến:
CloudWatch Metrics → CloudWatch Alarm → SNS Topic → Lambda (xử lý)
                                      → Email/SMS (alert team)
                                      → PagerDuty/OpsGenie (on-call)

Pattern EventBridge:
AWS Config Rule violation → EventBridge → Lambda (auto-remediate)
                                        → SNS → Email (notify)

Pattern tối ưu cho phỏng vấn:
"Alarm khi CPU > 80% trong 3 phút liên tiếp"
→ CloudWatch Alarm: metric=CPUUtilization, threshold=80, period=60s, evaluationPeriods=3
→ Action: SNS topic → Lambda scale up + notify Slack
```

---

## 6. Trusted Advisor vs AWS Config vs Security Hub

Ba dịch vụ kiểm tra/đánh giá môi trường AWS hay bị nhầm vai trò.

### Bảng So Sánh

| Tiêu Chí                  | Trusted Advisor                      | AWS Config                              | Security Hub                             |
| ------------------------- | ------------------------------------ | --------------------------------------- | ---------------------------------------- |
| **Mục đích**              | Best practice recommendations        | Resource configuration tracking         | Centralized security findings            |
| **Scope**                 | Account-level checks                 | Resource-level configuration state      | Aggregated findings từ nhiều services    |
| **Real-time**             | Theo lịch (refresh vài giờ/lần)     | Near real-time on config change         | Real-time                                |
| **Nguồn data**            | AWS phân tích account                | Tự thu thập configuration items         | GuardDuty, Config, Inspector, Macie...  |
| **Remediation**           | Không auto-remediate                 | Auto-remediate qua SSM Automation       | Không auto-remediate (chỉ aggregate)    |
| **Custom rules**          | Không                                | Có — Custom Lambda rules                | Custom Actions, Security Standard rules  |
| **Cost category**         | ✅ Có (idle resources, rightsizing)  | ❌ Không                                | ❌ Không                                 |
| **Support tier**          | Basic: ~7 checks; Business+: full   | Tất cả tiers                            | Tất cả tiers                             |
| **Compliance standard**   | AWS best practices                   | CIS, PCI-DSS (Conformance Packs)        | CIS, PCI-DSS, FSBP, NIST built-in       |

### Decision Guide

```
→ Muốn biết EC2 nào đang idle, S3 nào không dùng để tiết kiệm chi phí
  → Trusted Advisor (Cost Optimization)

→ Muốn biết Security Group nào đang mở port 0.0.0.0/0
  → AWS Config Rule: restricted-ssh + restricted-common-ports
  → HOẶC Security Hub (FSBP checks bao gồm cả điều này)

→ Muốn có dashboard tổng hợp tất cả security findings từ GuardDuty, Inspector, Config
  → Security Hub (aggregator)

→ Muốn auto-revert security group vi phạm
  → AWS Config + SSM Automation Remediation

→ Muốn biết RDS không bật Multi-AZ
  → Trusted Advisor (Fault Tolerance) HOẶC Config Rule: rds-multi-az-support
```

---

## 7. SSM Run Command vs SSM Automation vs AWS Lambda

| Tiêu Chí                  | SSM Run Command                      | SSM Automation                           | AWS Lambda                              |
| ------------------------- | ------------------------------------ | ---------------------------------------- | --------------------------------------- |
| **Mục đích**              | Thực thi command trên EC2 instances  | Runbook phức tạp, multi-step             | Serverless function cho event-driven    |
| **Target**                | EC2, on-premises servers             | AWS resources (không chỉ EC2)           | Event trigger, HTTP, Schedule           |
| **Agent cần thiết**       | SSM Agent trên instance              | SSM Agent + AWS SDK                      | Không cần agent                         |
| **Multi-step**            | Không (mỗi command riêng)           | ✅ Có (Automation Document = workflow)  | ✅ Có (code tự định nghĩa)             |
| **Approval gate**         | Không                                | ✅ Có (human approval step)             | Tự code                                 |
| **Cross-account**         | Hạn chế                              | ✅ Có (delegated permissions)           | ✅ Có (cross-account invoke)            |
| **Use case**              | Restart service, run script batch    | Patching workflow, AMI bake, DR         | API handler, event processing           |

---

## 8. AWS Budgets vs Cost Explorer vs Cost Anomaly Detection

| Tiêu Chí                  | AWS Budgets                          | Cost Explorer                            | Cost Anomaly Detection                  |
| ------------------------- | ------------------------------------ | ---------------------------------------- | --------------------------------------- |
| **Mục đích**              | Đặt ngưỡng và nhận cảnh báo         | Phân tích và dự báo chi phí lịch sử     | Phát hiện chi tiêu bất thường bằng ML  |
| **Proactive/Reactive**    | Proactive (cảnh báo khi gần ngưỡng) | Reactive (phân tích sau khi xảy ra)     | Proactive (ML phát hiện sớm)            |
| **ML/AI**                 | Không                                | Có (forecasting)                         | ✅ Có (anomaly detection)               |
| **Actions**               | ✅ Tự động: deny IAM, apply SCP      | Không có actions                         | SNS notification                        |
| **Granularity**           | Account, service, tag, RI           | Service, account, region, tag, usage    | Service, account, cost category         |
| **Lookback**              | Real-time so với budget             | Lịch sử 13 tháng                         | Học pattern 2–3 tuần                   |

### Khi Nào Dùng Cái Nào?

```
→ Không để vượt $1,000/tháng cho môi trường dev
  → AWS Budgets (đặt threshold + action block IAM)

→ Phân tích tại sao bill tháng này tăng so với tháng trước
  → Cost Explorer (filter by service, tag, region)

→ EC2 instance bị bỏ quên chạy suốt tuần và không ai biết
  → Cost Anomaly Detection (ML alert khi chi tiêu đột biến)

→ Muốn dự báo chi phí 3 tháng tới
  → Cost Explorer (Forecasting)
```

---

## 9. CloudTrail Trails vs CloudTrail Lake

| Tiêu Chí                  | CloudTrail Trails                    | CloudTrail Lake                          |
| ------------------------- | ------------------------------------ | ---------------------------------------- |
| **Ra mắt**                | Service gốc (2013)                   | 2022 — thế hệ mới                       |
| **Lưu trữ**               | S3 bucket (JSON files)               | Managed data lake, không cần S3         |
| **Query**                 | Athena (SQL trên S3) — cần thiết lập | SQL trực tiếp trong CloudTrail console  |
| **Retention**             | Tùy thuộc S3 lifecycle              | 7 năm tối đa                             |
| **Cost model**            | S3 storage + Athena query           | Event ingestion + query                 |
| **Data sources**          | AWS services                         | AWS services + activity events từ Apps  |
| **Setup**                 | Trail → S3 → (optional Athena)       | Event data store — đơn giản hơn         |
| **Phù hợp với**           | Long-term archive, Splunk integration| Quick query, analyst-friendly           |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
