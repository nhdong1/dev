# Terraform Knowledge Base — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về Terraform — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã

## 📁 Cấu Trúc Thư Mục

```
Devops/terraform/
├── README.md                               [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
│
├── 01-fundamentals/
│   ├── README.md                           Nền tảng Terraform, HCL, vòng đời tài nguyên
│   ├── 1-what-is-iac.md                      IaC là gì, lợi ích, so sánh tools
│   ├── 2-hcl-syntax.md                       HCL — HashiCorp Configuration Language — cú pháp
│   ├── 3-providers-resources.md              Provider & Resource cơ bản
│   ├── 4-variables-outputs.md                Variables, Locals, Outputs
│   └── 5-lifecycle.md                        init, plan, apply, destroy — vòng đời
│
├── 02-state-management/
│   ├── README.md                           Quản lý state, backend, locking
│   ├── 1-state-explained.md                  State file là gì, chứa gì, tại sao quan trọng
│   ├── 2-remote-backend.md                   S3+DynamoDB, GCS, Azure Blob, Terraform Cloud
│   ├── 3-state-locking.md                    State Locking — Khoá trạng thái — và deadlock
│   ├── 4-state-commands.md                   terraform state list/show/mv/rm/pull/push
│   └── 5-state-recovery.md                   Phục hồi khi state bị corrupt — hỏng
│
├── 03-modules/
│   ├── README.md                           Thiết kế module, best practices
│   ├── 1-module-structure.md                 Cấu trúc module chuẩn
│   ├── 2-input-output.md                     Variables, Outputs, Type Constraints
│   ├── 3-module-registry.md                  Terraform Registry — public & private
│   ├── 4-module-versioning.md                Semantic versioning — Đánh số phiên bản
│   └── 5-composition-patterns.md             Flat, Nested, Wrapper module patterns
│
├── 04-workspaces-environments/
│   ├── README.md                           Quản lý đa môi trường
│   ├── 1-workspaces.md                       Terraform Workspaces — giới hạn và dùng khi nào
│   ├── 2-environment-separation.md           dev/staging/prod strategy — chiến lược phân tách
│   ├── 3-tfvars-management.md                .tfvars file per environment
│   ├── 4-terragrunt-intro.md                 Terragrunt — DRY multi-env management
│   └── 5-backend-per-env.md                  Backend riêng cho từng môi trường
│
├── 05-security/
│   ├── README.md                           Bảo mật Terraform toàn diện
│   ├── 1-secrets-management.md               Vault, AWS Secrets Manager, SOPS
│   ├── 2-iam-roles.md                        IAM — Identity Access Management — role tối thiểu
│   ├── 3-sensitive-variables.md              Sensitive vars, output masking
│   ├── 4-static-analysis.md                  tfsec, Checkov, Terrascan
│   └── 5-audit-logging.md                    Nhật ký kiểm tra — CloudTrail, audit logs
│
├── 06-cicd/
│   ├── README.md                           Tích hợp CI/CD với Terraform
│   ├── 1-github-actions.md                   GitHub Actions workflow cho Terraform
│   ├── 2-gitlab-ci.md                        GitLab CI/CD pipeline
│   ├── 3-atlantis.md                         Atlantis — PR-based automation — tự động qua PR
│   ├── 4-terraform-cloud.md                  Terraform Cloud / HCP Terraform
│   └── 5-rollback-strategy.md                Chiến lược rollback khi có sự cố
│
├── 07-testing/
│   ├── README.md                           Kiểm thử hạ tầng Terraform
│   ├── 1-validate-fmt.md                     terraform validate, fmt, lint
│   ├── 2-tflint.md                           TFLint — Kiểm tra lỗi và best practices
│   ├── 3-terratest.md                        Terratest — Unit & Integration testing
│   ├── 4-checkov-opa.md                      Checkov & OPA — Open Policy Agent — compliance
│   └── 5-test-strategy.md                    Chiến lược kiểm thử toàn diện
│
├── 08-monitoring/
│   ├── README.md                           Giám sát & quan sát hạ tầng Terraform
│   ├── 1-drift-detection.md                  Drift Detection — Phát hiện lệch cấu hình
│   ├── 2-infracost.md                        Infracost — Ước tính chi phí trong CI/CD
│   ├── 3-change-audit.md                     Kiểm tra ai thay đổi gì, khi nào
│   ├── 4-resource-tagging.md                 Tagging strategy — Chiến lược gán nhãn
│   └── 5-alerting.md                         Cảnh báo khi có thay đổi ngoài dự kiến
│
├── 09-troubleshooting/
│   ├── README.md                           Xử lý sự cố & incident response
│   ├── 1-state-corruption.md                 State file hỏng — cách phát hiện và phục hồi
│   ├── 2-dependency-issues.md                Dependency Cycle — Vòng phụ thuộc
│   ├── 3-provider-errors.md                  Lỗi provider — timeout, rate limit, auth
│   ├── 4-import-moved.md                     terraform import & moved block
│   ├── 5-debug-mode.md                       TF_LOG=DEBUG — chế độ gỡ lỗi chi tiết
│   └── 6-production-checklist.md             Checklist trước khi apply vào production
│
├── 10-advanced/
│   ├── README.md                           Chủ đề Terraform nâng cao
│   ├── 1-dynamic-blocks.md                   Dynamic Blocks — Khối động
│   ├── 2-meta-arguments.md                   depends_on, lifecycle, provisioner
│   ├── 3-for-each-count.md                   for_each vs count — so sánh và dùng khi nào
│   ├── 4-custom-providers.md                 Viết Custom Provider — Provider tùy chỉnh
│   ├── 5-terraform-cdk.md                    CDK — Cloud Development Kit — Python/TypeScript
│   ├── 6-opentofu.md                         OpenTofu — Nhánh open-source của Terraform
│   └── 7-multi-region-account.md             Kiến trúc đa vùng & đa tài khoản
│
├── 11-interview-prep/
│   ├── README.md                           Tổng quan chuẩn bị phỏng vấn
│   ├── 1-INTERVIEW_GUIDE.md                  Top 20 câu hỏi phỏng vấn Terraform
│   ├── 2-star-stories.md                     Câu chuyện sự cố theo phương pháp STAR
│   ├── 3-system-design-scenarios.md          Thiết kế hệ thống với IaC
│   ├── 4-technical-questions.md              Q&A kỹ thuật tổng hợp
│   └── 5-90-day-study-plan.md                Kế hoạch học 90 ngày có cấu trúc
```

---

## ✅ Những Gì Đã Tạo

| Chủ Đề                                    | File                                   | Trạng Thái | Chất Lượng |
| ----------------------------------------- | -------------------------------------- | ---------- | ---------- |
| **Tổng quan & Lộ trình**                  | README.md                              | ✅         | Toàn diện  |
| **Chỉ mục đầy đủ**                        | INDEX.md                               | ✅         | Toàn diện  |
| **01 Fundamentals — Tổng quan phần**      | 01-fundamentals/README.md              | ✅         | Toàn diện  |
| **01 IaC là gì, lợi ích, so sánh tools** | 01-fundamentals/1-what-is-iac.md       | ✅         | Toàn diện  |
| **01 HCL — Cú pháp đầy đủ**              | 01-fundamentals/2-hcl-syntax.md        | ✅         | Toàn diện  |
| **01 Providers & Resources**              | 01-fundamentals/3-providers-resources.md | ✅       | Toàn diện  |
| **01 Variables, Locals, Outputs**         | 01-fundamentals/4-variables-outputs.md | ✅         | Toàn diện  |
| **01 Vòng đời: init→plan→apply→destroy** | 01-fundamentals/5-lifecycle.md         | ✅         | Toàn diện  |
| **02 State Management — Tổng quan phần** | 02-state-management/README.md          | ✅         | Toàn diện  |
| **02 State file là gì, chứa gì**         | 02-state-management/1-state-explained.md | ✅       | Toàn diện  |
| **02 Remote Backend — S3, GCS, Azure**   | 02-state-management/2-remote-backend.md | ✅        | Toàn diện  |
| **02 State Locking và deadlock**         | 02-state-management/3-state-locking.md | ✅         | Toàn diện  |
| **02 terraform state commands**          | 02-state-management/4-state-commands.md | ✅        | Toàn diện  |
| **02 Phục hồi khi state bị corrupt**    | 02-state-management/5-state-recovery.md | ✅        | Toàn diện  |
| **03 Modules — Tổng quan phần**         | 03-modules/README.md                    | ✅         | Toàn diện  |
| **03 Cấu trúc module chuẩn**            | 03-modules/1-module-structure.md        | ✅         | Toàn diện  |
| **03 Variables, Outputs, Type Constraints** | 03-modules/2-input-output.md        | ✅         | Toàn diện  |
| **03 Terraform Registry public & private** | 03-modules/3-module-registry.md      | ✅         | Toàn diện  |
| **03 Semantic Versioning — Đánh số phiên bản** | 03-modules/4-module-versioning.md | ✅        | Toàn diện  |
| **03 Flat, Nested, Wrapper module patterns** | 03-modules/5-composition-patterns.md | ✅       | Toàn diện  |
| **04 Workspaces & Environments — Tổng quan** | 04-workspaces-environments/README.md  | ✅       | Toàn diện  |
| **04 Terraform Workspaces — giới hạn và khi nào dùng** | 04-workspaces-environments/1-workspaces.md | ✅ | Toàn diện |
| **04 Environment Separation — dev/staging/prod** | 04-workspaces-environments/2-environment-separation.md | ✅ | Toàn diện |
| **04 .tfvars management theo môi trường**   | 04-workspaces-environments/3-tfvars-management.md | ✅   | Toàn diện  |
| **04 Terragrunt — DRY multi-env management** | 04-workspaces-environments/4-terragrunt-intro.md | ✅  | Toàn diện  |
| **04 Backend riêng cho từng môi trường**    | 04-workspaces-environments/5-backend-per-env.md   | ✅   | Toàn diện  |
| **05 Security — Tổng quan phần**            | 05-security/README.md                             | ✅   | Toàn diện  |
| **05 Secrets Management — Vault, SSM, SOPS** | 05-security/1-secrets-management.md              | ✅   | Toàn diện  |
| **05 IAM Roles — Least Privilege**          | 05-security/2-iam-roles.md                        | ✅   | Toàn diện  |
| **05 Sensitive Variables & Output Masking** | 05-security/3-sensitive-variables.md              | ✅   | Toàn diện  |
| **05 Static Analysis — tfsec, Checkov**     | 05-security/4-static-analysis.md                 | ✅   | Toàn diện  |
| **05 Audit Logging — CloudTrail, Config**   | 05-security/5-audit-logging.md                   | ✅   | Toàn diện  |
| **06 CI/CD — Tổng quan phần**               | 06-cicd/README.md                                 | ✅   | Toàn diện  |
| **06 GitHub Actions workflow cho Terraform** | 06-cicd/1-github-actions.md                      | ✅   | Toàn diện  |
| **06 GitLab CI/CD pipeline**                | 06-cicd/2-gitlab-ci.md                            | ✅   | Toàn diện  |
| **06 Atlantis — PR-based automation**       | 06-cicd/3-atlantis.md                             | ✅   | Toàn diện  |
| **06 Terraform Cloud / HCP Terraform**      | 06-cicd/4-terraform-cloud.md                      | ✅   | Toàn diện  |
| **06 Rollback Strategy — Chiến lược khôi phục** | 06-cicd/5-rollback-strategy.md               | ✅   | Toàn diện  |
| **07 Testing — Tổng quan phần**                 | 07-testing/README.md                          | ✅   | Toàn diện  |
| **07 terraform validate & fmt**                 | 07-testing/1-validate-fmt.md                  | ✅   | Toàn diện  |
| **07 TFLint — Linting nâng cao**                | 07-testing/2-tflint.md                        | ✅   | Toàn diện  |
| **07 Terratest — Unit & Integration Testing**   | 07-testing/3-terratest.md                     | ✅   | Toàn diện  |
| **07 Checkov & OPA — Compliance Testing**       | 07-testing/4-checkov-opa.md                   | ✅   | Toàn diện  |
| **07 Chiến lược kiểm thử toàn diện**            | 07-testing/5-test-strategy.md                 | ✅   | Toàn diện  |
| **08 Monitoring — Tổng quan phần**              | 08-monitoring/README.md                       | ✅   | Toàn diện  |
| **08 Drift Detection — Phát Hiện Lệch Cấu Hình** | 08-monitoring/1-drift-detection.md          | ✅   | Toàn diện  |
| **08 Infracost — Ước Tính Chi Phí CI/CD**       | 08-monitoring/2-infracost.md                  | ✅   | Toàn diện  |
| **08 Change Audit — Kiểm Tra Thay Đổi**         | 08-monitoring/3-change-audit.md               | ✅   | Toàn diện  |
| **08 Resource Tagging — Chiến Lược Gán Nhãn**   | 08-monitoring/4-resource-tagging.md           | ✅   | Toàn diện  |
| **08 Alerting — Cảnh Báo Thay Đổi Bất Thường** | 08-monitoring/5-alerting.md                   | ✅   | Toàn diện  |
| **09 Troubleshooting — Tổng quan phần**         | 09-troubleshooting/README.md                  | ✅   | Toàn diện  |
| **09 State Corruption — Phát Hiện và Phục Hồi** | 09-troubleshooting/1-state-corruption.md      | ✅   | Toàn diện  |
| **09 Dependency Cycle — Vòng Phụ Thuộc**        | 09-troubleshooting/2-dependency-issues.md     | ✅   | Toàn diện  |
| **09 Provider Errors — Timeout, Rate Limit**    | 09-troubleshooting/3-provider-errors.md       | ✅   | Toàn diện  |
| **09 terraform import & moved block**           | 09-troubleshooting/4-import-moved.md          | ✅   | Toàn diện  |
| **09 TF_LOG=DEBUG — Chế Độ Gỡ Lỗi**            | 09-troubleshooting/5-debug-mode.md            | ✅   | Toàn diện  |
| **09 Production Checklist — Trước Khi Apply**   | 09-troubleshooting/6-production-checklist.md  | ✅   | Toàn diện  |
| **10 Advanced — Tổng quan phần**                | 10-advanced/README.md                         | ✅   | Toàn diện  |
| **10 Dynamic Blocks — Khối Động**               | 10-advanced/1-dynamic-blocks.md               | ✅   | Toàn diện  |
| **10 Meta-arguments — depends_on, lifecycle**   | 10-advanced/2-meta-arguments.md               | ✅   | Toàn diện  |
| **10 for_each vs count — So Sánh Chuyên Sâu**  | 10-advanced/3-for-each-count.md               | ✅   | Toàn diện  |
| **10 Custom Provider — Provider Tùy Chỉnh**     | 10-advanced/4-custom-providers.md             | ✅   | Toàn diện  |
| **10 CDK for Terraform — Python/TypeScript**    | 10-advanced/5-terraform-cdk.md               | ✅   | Toàn diện  |
| **10 OpenTofu — Nhánh Open-source**             | 10-advanced/6-opentofu.md                    | ✅   | Toàn diện  |
| **10 Kiến Trúc Đa Vùng & Đa Tài Khoản**        | 10-advanced/7-multi-region-account.md        | ✅   | Toàn diện  |
| **11 Interview Prep — Tổng quan phần**          | 11-interview-prep/README.md                  | ✅   | Toàn diện  |
| **11 Top 20 Câu Hỏi Phỏng Vấn Terraform**      | 11-interview-prep/1-INTERVIEW_GUIDE.md       | ✅   | Toàn diện  |
| **11 Câu Chuyện Sự Cố theo Phương Pháp STAR**  | 11-interview-prep/2-star-stories.md          | ✅   | Toàn diện  |
| **11 Thiết Kế Hệ Thống với IaC**               | 11-interview-prep/3-system-design-scenarios.md | ✅ | Toàn diện  |
| **11 Q&A Kỹ Thuật Tổng Hợp**                  | 11-interview-prep/4-technical-questions.md   | ✅   | Toàn diện  |
| **11 Kế Hoạch Học 90 Ngày**                    | 11-interview-prep/5-90-day-study-plan.md     | ✅   | Toàn diện  |

---

## 🎯 Cần Tạo Tiếp (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao (Kỹ năng cốt lõi)

- [x] `01-fundamentals/README.md` — Nền tảng HCL, providers, lifecycle ✅
- [x] `02-state-management/README.md` — Remote state, locking ✅
- [x] `02-state-management/1-state-explained.md` — State file là gì ✅
- [x] `02-state-management/2-remote-backend.md` — S3, GCS, Terraform Cloud ✅
- [x] `02-state-management/3-state-locking.md` — State Locking và deadlock ✅
- [x] `02-state-management/4-state-commands.md` — Các lệnh state ✅
- [x] `02-state-management/5-state-recovery.md` — Phục hồi state corrupt ✅
- [x] `03-modules/README.md` — Module design patterns ✅
- [x] `03-modules/1-module-structure.md` — Cấu trúc module chuẩn ✅
- [x] `03-modules/2-input-output.md` — Variables, Outputs, Type Constraints ✅
- [x] `03-modules/3-module-registry.md` — Terraform Registry public & private ✅
- [x] `03-modules/4-module-versioning.md` — Semantic Versioning ✅
- [x] `03-modules/5-composition-patterns.md` — Flat, Nested, Wrapper patterns ✅
- [x] `11-interview-prep/README.md` — Tổng quan chuẩn bị phỏng vấn ✅
- [x] `11-interview-prep/1-INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn ✅
- [x] `11-interview-prep/2-star-stories.md` — Câu chuyện STAR ✅
- [x] `11-interview-prep/3-system-design-scenarios.md` — System design với IaC ✅
- [x] `11-interview-prep/4-technical-questions.md` — Q&A kỹ thuật tổng hợp ✅
- [x] `11-interview-prep/5-90-day-study-plan.md` — Kế hoạch học 90 ngày ✅
- [ ] `ROADMAP.md` — Kế hoạch học 90 ngày chi tiết

### Ưu Tiên Trung (Kỹ năng vận hành)

- [x] `06-cicd/README.md` — CI/CD integration ✅
- [x] `05-security/README.md` — Bảo mật hạ tầng ✅
- [x] `09-troubleshooting/README.md` — Xử lý sự cố ✅
- [x] `04-workspaces-environments/README.md` — Quản lý môi trường ✅
- [x] `11-interview-prep/2-star-stories.md` — Câu chuyện STAR ✅

### Ưu Tiên Thấp (Tham khảo)

- [x] `07-testing/README.md` — Kiểm thử hạ tầng ✅
- [x] `08-monitoring/README.md` — Giám sát & drift detection ✅
- [x] `10-advanced/README.md` — Chủ đề nâng cao ✅
- [ ] `GLOSSARY.md` — Thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu tham khảo

---

## 🚀 Cách Dùng Knowledge Base Này

### Cho Tự Học

```
1. Bắt đầu với README.md
2. Chọn Lộ Trình (Mới bắt đầu / Trung cấp / Nâng cao)
3. Học từng phần theo thứ tự
4. Làm bài tập thực hành (thiết lập lab thực tế)
5. Xây dựng dự án portfolio
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào State Management (luôn được hỏi)
3. Nắm vững Module Design (luôn được hỏi)
4. Học 06-cicd/ — cách tích hợp Terraform vào team
5. Chuẩn bị câu chuyện sự cố hạ tầng (STAR)
6. Mock interview với đồng nghiệp
```

### Cho Công Việc Thực Tế

```
Dùng làm tài liệu tham khảo:
- Trước deploy: Đọc 09-troubleshooting/production-checklist.md
- Khi có sự cố: Vào 09-troubleshooting/ để chẩn đoán
- Thiết kế module: Theo 03-modules/ best practices
- CI/CD setup: Theo 06-cicd/ hướng dẫn
- Bảo mật: Kiểm tra theo 05-security/ checklist
```

### Cho System Design

```
1. Đọc README.md để hiểu tổng quan
2. Dùng 04-workspaces-environments/ cho multi-env design
3. Dùng 05-security/ cho security architecture
4. Dùng 06-cicd/ cho deployment pipeline design
5. Dùng 10-advanced/ cho multi-region architecture
```

---

## 📊 Ước Tính Thời Gian Học

| Phần                                  | Thời Gian | Độ Khó | Ưu Tiên  |
| ------------------------------------- | --------- | ------ | -------- |
| Nền tảng (Fundamentals)               | 4-6 giờ   | ⭐     | Bắt buộc |
| State Management — Quản lý trạng thái | 6-8 giờ   | ⭐⭐   | Bắt buộc |
| Modules — Mô-đun                      | 8-10 giờ  | ⭐⭐   | Bắt buộc |
| Workspaces & Environments             | 4-6 giờ   | ⭐⭐   | Bắt buộc |
| Security — Bảo mật                    | 6-8 giờ   | ⭐⭐   | Bắt buộc |
| CI/CD Integration                     | 8-10 giờ  | ⭐⭐⭐ | Nên có   |
| Testing — Kiểm thử                    | 6-8 giờ   | ⭐⭐⭐ | Nên có   |
| Monitoring — Giám sát                 | 4-6 giờ   | ⭐⭐   | Nên có   |
| Advanced Topics — Nâng cao            | 15-20 giờ | ⭐⭐⭐ | Tốt hơn  |

**Tổng cộng: 60-90 giờ để có kiến thức Terraform toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Được Hỗ Trợ

### Mới Bắt Đầu (0-1 năm kinh nghiệm)

- [ ] Hiểu IaC là gì và tại sao cần
- [ ] Viết được HCL — HashiCorp Configuration Language — cơ bản
- [ ] Tạo và xóa tài nguyên trên AWS/GCP
- [ ] Hiểu state file và cách không commit lên git
- [ ] Dùng module từ Terraform Registry

**Thời gian để thành thạo:** 2-3 tháng

### Trung Cấp (1-3 năm kinh nghiệm)

- [ ] Thiết kế module tái sử dụng tốt
- [ ] Quản lý remote state với locking
- [ ] Tích hợp Terraform vào CI/CD
- [ ] Quản lý nhiều môi trường hiệu quả
- [ ] Xử lý secrets an toàn

**Thời gian để nâng cấp:** 2-3 tháng để đào sâu

### Nâng Cao (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc multi-account, multi-region
- [ ] Policy as Code — Chính Sách Dưới Dạng Mã
- [ ] Incident command cho hạ tầng
- [ ] Custom provider development
- [ ] Compliance frameworks — Khung tuân thủ

**Thời gian:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu              | Vị Trí                                                                         |
| -------------------- | ------------------------------------------------------------------------------ |
| Tổng quan nhanh      | [README.md](README.md)                                                         |
| Remote backend setup | [02-state-management/remote-backend.md](02-state-management/remote-backend.md) |
| Module design guide  | [03-modules/README.md](03-modules/README.md)                                   |
| CI/CD setup          | [06-cicd/README.md](06-cicd/README.md)                                         |
| Security checklist   | [05-security/README.md](05-security/README.md)                                 |
| Câu hỏi phỏng vấn    | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md)   |

---

## 📈 Theo Dõi Tiến Độ Học

Sao chép và theo dõi tiến độ của bạn:

```markdown
## Hoàn Thành Terraform Knowledge

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)

- [ ] IaC là gì, tại sao dùng Terraform
- [ ] HCL syntax — cú pháp cơ bản
- [ ] Providers & Resources
- [ ] Variables, Locals, Outputs
- [ ] Vòng đời: init, plan, apply, destroy

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)

- [ ] State file — hiểu và quản lý
- [ ] Remote backend với locking
- [ ] Module design & reuse
- [ ] Workspaces và multi-env strategy
- [ ] Secrets management — quản lý bí mật

### Giai Đoạn 3: Vận Hành (Tuần 7-10)

- [ ] CI/CD pipeline với Terraform
- [ ] Drift detection — phát hiện lệch
- [ ] Testing infrastructure — kiểm thử
- [ ] Security scanning — quét bảo mật
- [ ] Incident response cho hạ tầng

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Multi-account architecture
- [ ] Policy as Code với Sentinel/OPA
- [ ] Custom providers
- [ ] Terragrunt mastery
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích IaC và lợi ích không cần nhìn notes
- [ ] Thiết kế cấu trúc Terraform cho dự án thực tế
- [ ] Hiểu trade-off giữa các cách quản lý state
- [ ] Đọc và debug HCL code của người khác
- [ ] Thiết kế module tái sử dụng tốt

### ✅ Năng Lực Vận Hành

- [ ] Thiết lập remote backend an toàn cho team
- [ ] Tích hợp Terraform vào CI/CD pipeline
- [ ] Quản lý nhiều môi trường không bị nhầm lẫn
- [ ] Phát hiện và xử lý drift — lệch cấu hình
- [ ] Áp dụng security best practices — thực hành bảo mật tốt nhất

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời được top 20 câu hỏi Terraform
- [ ] Kể được 2-3 câu chuyện sự cố hạ tầng (STAR)
- [ ] Thiết kế hệ thống với IaC considerations
- [ ] Thảo luận trade-off và constraints tự tin
- [ ] Nắm vững ít nhất 1 cloud provider sâu

---

## 🚀 Bước Tiếp Theo

### Ngay Bây Giờ (Tuần Này)

1. Đọc kỹ README.md
2. Chọn lộ trình học phù hợp với mình
3. Cài Terraform và thiết lập môi trường lab
4. Tạo tài nguyên đầu tiên trên AWS Free Tier / GCP Free Tier

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành `01-fundamentals/`
2. Thiết lập remote backend thực tế
3. Bắt đầu `03-modules/` — viết module đầu tiên
4. Làm lab thực hành cho từng chủ đề

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành tất cả core topics (01-06)
2. Deep dive vào một cloud provider
3. Tích hợp Terraform vào một CI/CD pipeline thực
4. Chuẩn bị 2-3 câu chuyện sự cố

### Dài Hạn (3 Tháng Tới)

1. Thành thạo một cloud provider hoàn toàn
2. Hiểu trade-off giữa các patterns
3. Xây dựng dự án portfolio thực tế
4. Bắt đầu apply vào các vị trí DevOps / Platform Engineer

---

## 💡 Lời Khuyên Từ Thực Tế

1. **Học bằng cách làm:** Đừng chỉ đọc — hãy tạo và destroy tài nguyên thực sự
2. **Commit state vào git là anti-pattern:** Luôn dùng remote backend
3. **`count` vs `for_each`:** Ưu tiên `for_each` để tránh index shifting
4. **Module nhỏ hơn module lớn:** Dễ tái sử dụng và test hơn
5. **Plan trước khi apply:** Luôn review plan output cẩn thận
6. **Đặt tên rõ ràng:** Resource names nên bao gồm environment và purpose
7. **Test state recovery:** Backup state thường xuyên, test restore
8. **Dùng `moved` block:** Khi refactor, không xóa rồi tạo lại tài nguyên

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống. Chào mừng đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm phần cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích tốt hơn cho các concepts phức tạp
- [ ] Hướng dẫn cho cloud provider cụ thể

---

## 📄 Giấy Phép

Knowledge base này mở cho việc học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 2.0
**Trạng Thái:** ✅ README & INDEX hoàn thành | ✅ 01-fundamentals hoàn thành | ✅ 02-state-management hoàn thành | ✅ 03-modules hoàn thành | ✅ 04-workspaces-environments hoàn thành | ✅ 05-security hoàn thành | ✅ 06-cicd hoàn thành | ✅ 07-testing hoàn thành | ✅ 08-monitoring hoàn thành | ✅ 09-troubleshooting hoàn thành | ✅ 10-advanced hoàn thành | ✅ 11-interview-prep hoàn thành
