# CircleCI Knowledge Base — Chỉ Mục Đầy Đủ

> Chỉ mục toàn bộ tài liệu CircleCI — CI/CD (Continuous Integration / Continuous Deployment — Tích Hợp Liên Tục / Triển Khai Liên Tục)

## 📁 Cấu Trúc Thư Mục

```
Devops/circleci/
├── README.md                                   [BẮT ĐẦU Ở ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/
│   ├── README.md                               Giới thiệu CI/CD và kiến trúc CircleCI
│   ├── 1-cicd-concepts.md                        CI vs CD (Delivery) vs CD (Deployment)
│   ├── 2-circleci-architecture.md                Pipeline → Workflow → Job → Step
│   ├── 3-executors.md                            Docker, Machine, macOS, Windows executor
│   ├── 4-resource-classes.md                     CPU/RAM resource class và chi phí
│   └── 5-circleci-vs-alternatives.md             So sánh Jenkins, GitHub Actions, GitLab CI
│
├── 02-configuration/
│   ├── README.md                               Hướng dẫn cấu hình .circleci/config.yml
│   ├── 1-yaml-syntax.md                          Cú pháp YAML đầy đủ, anchors, aliases
│   ├── 2-jobs-and-steps.md                       Định nghĩa job, các built-in steps
│   ├── 3-commands.md                             Commands — Lệnh Tái Sử Dụng tùy chỉnh
│   ├── 4-parameters.md                           Pipeline, job, command parameters
│   └── 5-environment-variables.md                Built-in, project-level, org-level vars
│
├── 03-workflows/
│   ├── README.md                               Tổng quan workflow design
│   ├── 1-sequential-workflow.md                  requires, job dependencies
│   ├── 2-parallel-workflow.md                    Fan-out/Fan-in, tăng tốc độ
│   ├── 3-approval-jobs.md                        Manual gate — Cổng Duyệt Thủ Công
│   ├── 4-scheduled-pipelines.md                  Cron triggers — Kích Hoạt Theo Lịch
│   └── 5-branch-tag-filters.md                   Bộ lọc nhánh, tag, regex
│
├── 04-orbs/
│   ├── README.md                               Giới thiệu Orbs — Gói Tích Hợp
│   ├── 1-using-orbs.md                           Import và dùng orb từ registry
│   ├── 2-certified-orbs.md                       aws-cli, docker, node, kubernetes orbs
│   ├── 3-inline-orbs.md                          Inline Orb — Orb Nội Tuyến trong config
│   └── 4-custom-orb-development.md               Tạo, test và publish orb tùy chỉnh
│
├── 05-optimization/
│   ├── README.md                               Chiến lược tối ưu pipeline
│   ├── 1-caching-strategies.md                   save_cache, restore_cache, cache keys
│   ├── 2-workspace.md                            persist_to_workspace, attach_workspace
│   ├── 3-test-splitting.md                       circleci tests split, timing-based
│   ├── 4-parallelism.md                          parallelism key, test distribution
│   └── 5-pipeline-insights.md                    Phân tích bottleneck, success rate
│
├── 06-security/
│   ├── README.md                               Bảo mật pipeline toàn diện
│   ├── 1-contexts.md                             Contexts — Ngữ Cảnh: org-level secrets
│   ├── 2-oidc-integration.md                     OIDC — OpenID Connect với AWS/GCP/Azure
│   ├── 3-ip-ranges.md                            IP Ranges — Dải IP cho firewall whitelist
│   └── 4-audit-log.md                            Audit Log — Nhật Ký Kiểm Toán
│
├── 07-integration/
│   ├── README.md                               Tổng quan tích hợp cloud & tools
│   ├── 1-docker-build-push.md                    Build và push Docker image
│   ├── 2-kubernetes-deploy.md                    Deploy lên EKS, GKE, AKS
│   ├── 3-aws-integration.md                      S3, ECR, ECS, Lambda, CodeDeploy
│   ├── 4-gcp-integration.md                      GCR, Cloud Run, GKE
│   ├── 5-terraform-pipeline.md                   Terraform plan/apply trong CircleCI
│   └── 6-notifications.md                        Slack, email, webhook notifications
│
├── 08-monitoring/
│   ├── README.md                               Giám sát và xử lý sự cố pipeline
│   ├── 1-pipeline-insights.md                    CircleCI Insights dashboard
│   ├── 2-ssh-debugging.md                        Rerun with SSH — Gỡ Lỗi Qua SSH
│   ├── 3-common-errors.md                        OOM, timeout, exit code phổ biến
│   └── 4-flaky-tests.md                          Flaky Tests — Kiểm Thử Không Ổn Định
│
├── 09-advanced/
│   ├── README.md                               Chủ đề nâng cao
│   ├── 1-dynamic-config.md                       Dynamic Config — Cấu Hình Động
│   ├── 2-path-filtering.md                       Path Filtering cho monorepo
│   ├── 3-matrix-jobs.md                          Matrix Jobs — Công Việc Ma Trận
│   ├── 4-self-hosted-runner.md                   Self-Hosted Runner — Máy Chạy Tự Quản Lý
│   └── 5-monorepo-strategy.md                    Chiến lược CI/CD cho monorepo
│
└── 10-interview-prep/
    ├── README.md                               Hướng dẫn chuẩn bị phỏng vấn
    ├── 1-INTERVIEW_GUIDE.md                      Top 20 câu hỏi phỏng vấn & câu trả lời
    ├── 2-star-stories.md                         Câu chuyện STAR về incident CI/CD
    ├── 3-system-design-scenarios.md              Bài toán thiết kế hệ thống CI/CD
    └── 4-tool-comparison.md                      CircleCI vs Jenkins vs GitHub Actions
```

---

## ✅ Đã Tạo — What's Been Created

| Chủ Đề                                | File                                                    | Trạng Thái | Chất Lượng |
| ------------------------------------- | ------------------------------------------------------- | ---------- | ---------- |
| **Tổng Quan & Lộ Trình**              | README.md                                               | ✅         | Toàn Diện  |
| **Chỉ Mục Đầy Đủ**                    | INDEX.md                                                | ✅         | Toàn Diện  |
| **01 — Giới Thiệu Module Fundamentals** | 01-fundamentals/README.md                             | ✅         | Toàn Diện  |
| **01 — Khái Niệm CI/CD**              | 01-fundamentals/1-cicd-concepts.md                      | ✅         | Toàn Diện  |
| **01 — Kiến Trúc CircleCI**           | 01-fundamentals/2-circleci-architecture.md              | ✅         | Toàn Diện  |
| **01 — Executors**                    | 01-fundamentals/3-executors.md                          | ✅         | Toàn Diện  |
| **01 — Resource Classes**             | 01-fundamentals/4-resource-classes.md                   | ✅         | Toàn Diện  |
| **01 — CircleCI vs Alternatives**     | 01-fundamentals/5-circleci-vs-alternatives.md           | ✅         | Toàn Diện  |
| **02 — Giới Thiệu Module Configuration** | 02-configuration/README.md                          | ✅         | Toàn Diện  |
| **02 — Cú Pháp YAML**                 | 02-configuration/1-yaml-syntax.md                       | ✅         | Toàn Diện  |
| **02 — Jobs và Steps**                | 02-configuration/2-jobs-and-steps.md                    | ✅         | Toàn Diện  |
| **02 — Commands**                     | 02-configuration/3-commands.md                          | ✅         | Toàn Diện  |
| **02 — Parameters**                   | 02-configuration/4-parameters.md                        | ✅         | Toàn Diện  |
| **02 — Environment Variables**        | 02-configuration/5-environment-variables.md             | ✅         | Toàn Diện  |
| **03 — Giới Thiệu Module Workflows**  | 03-workflows/README.md                                  | ✅         | Toàn Diện  |
| **03 — Sequential Workflow**          | 03-workflows/1-sequential-workflow.md                   | ✅         | Toàn Diện  |
| **03 — Parallel Workflow**            | 03-workflows/2-parallel-workflow.md                     | ✅         | Toàn Diện  |
| **03 — Approval Jobs**                | 03-workflows/3-approval-jobs.md                         | ✅         | Toàn Diện  |
| **03 — Scheduled Pipelines**          | 03-workflows/4-scheduled-pipelines.md                   | ✅         | Toàn Diện  |
| **03 — Branch & Tag Filters**         | 03-workflows/5-branch-tag-filters.md                    | ✅         | Toàn Diện  |
| **04 — Giới Thiệu Module Orbs**       | 04-orbs/README.md                                       | ✅         | Toàn Diện  |
| **04 — Sử Dụng Orbs**                 | 04-orbs/1-using-orbs.md                                 | ✅         | Toàn Diện  |
| **04 — Certified Orbs**               | 04-orbs/2-certified-orbs.md                             | ✅         | Toàn Diện  |
| **04 — Inline Orbs**                  | 04-orbs/3-inline-orbs.md                                | ✅         | Toàn Diện  |
| **04 — Custom Orb Development**       | 04-orbs/4-custom-orb-development.md                     | ✅         | Toàn Diện  |
| **05 — Giới Thiệu Module Optimization** | 05-optimization/README.md                             | ✅         | Toàn Diện  |
| **05 — Caching Strategies**           | 05-optimization/1-caching-strategies.md                 | ✅         | Toàn Diện  |
| **05 — Workspace**                    | 05-optimization/2-workspace.md                          | ✅         | Toàn Diện  |
| **05 — Test Splitting**               | 05-optimization/3-test-splitting.md                     | ✅         | Toàn Diện  |
| **05 — Parallelism**                  | 05-optimization/4-parallelism.md                        | ✅         | Toàn Diện  |
| **05 — Pipeline Insights**            | 05-optimization/5-pipeline-insights.md                  | ✅         | Toàn Diện  |
| **06 — Giới Thiệu Module Security**   | 06-security/README.md                                   | ✅         | Toàn Diện  |
| **06 — Contexts**                     | 06-security/1-contexts.md                               | ✅         | Toàn Diện  |
| **06 — OIDC Integration**             | 06-security/2-oidc-integration.md                       | ✅         | Toàn Diện  |
| **06 — IP Ranges**                    | 06-security/3-ip-ranges.md                              | ✅         | Toàn Diện  |
| **06 — Audit Log**                    | 06-security/4-audit-log.md                              | ✅         | Toàn Diện  |
| **07 — Giới Thiệu Module Integration** | 07-integration/README.md                               | ✅         | Toàn Diện  |
| **07 — Docker Build & Push**          | 07-integration/1-docker-build-push.md                   | ✅         | Toàn Diện  |
| **07 — Kubernetes Deploy**            | 07-integration/2-kubernetes-deploy.md                   | ✅         | Toàn Diện  |
| **07 — AWS Integration**              | 07-integration/3-aws-integration.md                       | ✅         | Toàn Diện  |
| **07 — GCP Integration**              | 07-integration/4-gcp-integration.md                       | ✅         | Toàn Diện  |
| **07 — Terraform Pipeline**           | 07-integration/5-terraform-pipeline.md                  | ✅         | Toàn Diện  |
| **07 — Notifications**                | 07-integration/6-notifications.md                         | ✅         | Toàn Diện  |
| **08 — Giới Thiệu Module Monitoring** | 08-monitoring/README.md                                   | ✅         | Toàn Diện  |
| **08 — Pipeline Insights (Giám Sát)** | 08-monitoring/1-pipeline-insights.md                      | ✅         | Toàn Diện  |
| **08 — SSH Debugging**                | 08-monitoring/2-ssh-debugging.md                          | ✅         | Toàn Diện  |
| **08 — Common Errors**                | 08-monitoring/3-common-errors.md                          | ✅         | Toàn Diện  |
| **08 — Flaky Tests**                  | 08-monitoring/4-flaky-tests.md                            | ✅         | Toàn Diện  |
| **09 — Giới Thiệu Module Advanced**   | 09-advanced/README.md                                     | ✅         | Toàn Diện  |
| **09 — Dynamic Config**               | 09-advanced/1-dynamic-config.md                           | ✅         | Toàn Diện  |
| **09 — Path Filtering**               | 09-advanced/2-path-filtering.md                           | ✅         | Toàn Diện  |
| **09 — Matrix Jobs**                  | 09-advanced/3-matrix-jobs.md                              | ✅         | Toàn Diện  |
| **09 — Self-Hosted Runner**           | 09-advanced/4-self-hosted-runner.md                       | ✅         | Toàn Diện  |
| **09 — Monorepo Strategy**            | 09-advanced/5-monorepo-strategy.md                        | ✅         | Toàn Diện  |
| **10 — Giới Thiệu Interview Prep**    | 10-interview-prep/README.md                               | ✅         | Toàn Diện  |
| **10 — Top 20 Câu Hỏi Phỏng Vấn**     | 10-interview-prep/1-INTERVIEW_GUIDE.md                    | ✅         | Toàn Diện  |
| **10 — STAR Stories**                 | 10-interview-prep/2-star-stories.md                       | ✅         | Toàn Diện  |
| **10 — System Design Scenarios**      | 10-interview-prep/3-system-design-scenarios.md            | ✅         | Toàn Diện  |
| **10 — Tool Comparison**              | 10-interview-prep/4-tool-comparison.md                    | ✅         | Toàn Diện  |

---

## 🎯 Cần Tạo Tiếp — Still To Create (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao — Kỹ Năng Cốt Lõi

- [x] ~~`01-fundamentals/README.md` — Nền tảng CI/CD và kiến trúc CircleCI~~ ✅
- [x] ~~`02-configuration/README.md` — Cấu hình `.circleci/config.yml` đầy đủ~~ ✅
- [x] ~~`02-configuration/1-yaml-syntax.md` — Cú pháp YAML, anchors, aliases~~ ✅
- [x] ~~`02-configuration/2-jobs-and-steps.md` — Jobs, steps, built-in commands~~ ✅
- [x] ~~`02-configuration/3-commands.md` — Commands tái sử dụng~~ ✅
- [x] ~~`02-configuration/4-parameters.md` — Pipeline, job, command parameters~~ ✅
- [x] ~~`02-configuration/5-environment-variables.md` — Built-in, project-level, context vars~~ ✅
- [x] ~~`03-workflows/README.md` — Thiết kế workflow sequential, parallel, approval~~ ✅
- [x] ~~`03-workflows/1-sequential-workflow.md` — requires, chuỗi phụ thuộc~~ ✅
- [x] ~~`03-workflows/2-parallel-workflow.md` — Fan-out/Fan-in, tăng tốc~~ ✅
- [x] ~~`03-workflows/3-approval-jobs.md` — Manual gate, cổng duyệt thủ công~~ ✅
- [x] ~~`03-workflows/4-scheduled-pipelines.md` — Cron triggers, lập lịch~~ ✅
- [x] ~~`03-workflows/5-branch-tag-filters.md` — Bộ lọc nhánh, tag, regex~~ ✅
- [x] ~~`05-optimization/README.md` — Tổng quan chiến lược tối ưu pipeline~~ ✅
- [x] ~~`05-optimization/1-caching-strategies.md` — Caching chi tiết~~ ✅
- [x] ~~`05-optimization/2-workspace.md` — Workspace giữa các jobs~~ ✅
- [x] ~~`05-optimization/3-test-splitting.md` — Test splitting theo timing/file~~ ✅
- [x] ~~`05-optimization/4-parallelism.md` — Parallelism, resource class~~ ✅
- [x] ~~`05-optimization/5-pipeline-insights.md` — Bottleneck, flaky tests~~ ✅
- [x] ~~`06-security/README.md` — Bảo mật pipeline toàn diện~~ ✅
- [x] ~~`06-security/1-contexts.md` — Contexts và quản lý secrets~~ ✅
- [x] ~~`10-interview-prep/1-INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn~~ ✅

### Ưu Tiên Trung Bình — Kỹ Năng Nâng Cao

- [x] ~~`04-orbs/README.md` — Tổng quan orbs ecosystem~~ ✅
- [x] ~~`04-orbs/1-using-orbs.md` — Import và dùng orb từ registry~~ ✅
- [x] ~~`04-orbs/certified-orbs.md` — Các orb phổ biến: aws-cli, docker, node~~ ✅
- [x] ~~`04-orbs/3-inline-orbs.md` — Inline Orb nội tuyến~~ ✅
- [x] ~~`04-orbs/4-custom-orb-development.md` — Tạo, test và publish custom orb~~ ✅
- [x] ~~`07-integration/README.md` — Tổng quan tích hợp cloud & tools~~ ✅
- [x] ~~`07-integration/1-docker-build-push.md` — CI/CD với Docker~~ ✅
- [x] ~~`07-integration/2-kubernetes-deploy.md` — Deploy EKS, GKE, AKS~~ ✅
- [x] ~~`07-integration/3-aws-integration.md` — Tích hợp AWS toàn diện~~ ✅
- [x] ~~`07-integration/4-gcp-integration.md` — GCR, Cloud Run, GKE~~ ✅
- [x] ~~`07-integration/5-terraform-pipeline.md` — Terraform plan/apply~~ ✅
- [x] ~~`07-integration/6-notifications.md` — Slack, email, webhook~~ ✅
- [x] ~~`05-optimization/test-splitting.md` — Test splitting và parallelism~~ ✅
- [x] ~~`09-advanced/1-dynamic-config.md` — Dynamic Config cho monorepo~~ ✅
- [x] ~~`08-monitoring/README.md` — Giám sát và xử lý sự cố pipeline~~ ✅
- [x] ~~`08-monitoring/1-pipeline-insights.md` — Insights dashboard vận hành~~ ✅
- [x] ~~`08-monitoring/2-ssh-debugging.md` — Debug pipeline qua SSH~~ ✅
- [x] ~~`08-monitoring/3-common-errors.md` — OOM, timeout, exit code~~ ✅
- [x] ~~`08-monitoring/4-flaky-tests.md` — Kiểm thử không ổn định~~ ✅

### Ưu Tiên Thấp — Tham Khảo

- [x] ~~`09-advanced/2-path-filtering.md` — Path Filtering cho monorepo~~ ✅
- [x] ~~`09-advanced/3-matrix-jobs.md` — Matrix jobs cho multi-version testing~~ ✅
- [x] ~~`09-advanced/4-self-hosted-runner.md` — Self-Hosted Runner on-premise~~ ✅
- [x] ~~`09-advanced/5-monorepo-strategy.md` — Chiến lược CI/CD cho monorepo~~ ✅
- [x] ~~`06-security/2-oidc-integration.md` — OIDC với AWS/GCP~~ ✅
- [x] ~~`06-security/3-ip-ranges.md` — IP Ranges cho firewall whitelist~~ ✅
- [x] ~~`06-security/4-audit-log.md` — Audit Log và compliance~~ ✅
- [x] ~~`07-integration/kubernetes-deploy.md` — Deploy lên Kubernetes~~ ✅ (xem `2-kubernetes-deploy.md`)
- [x] ~~`10-interview-prep/README.md` — Tổng quan chuẩn bị phỏng vấn~~ ✅
- [x] ~~`10-interview-prep/2-star-stories.md` — Câu chuyện STAR CI/CD~~ ✅
- [x] ~~`10-interview-prep/3-system-design-scenarios.md` — Bài toán thiết kế~~ ✅
- [x] ~~`10-interview-prep/4-tool-comparison.md` — So sánh CircleCI vs Jenkins vs GHA~~ ✅

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Tự Học — Self-Study

```
1. Bắt đầu với README.md
2. Chọn lộ trình theo mức độ (Beginner/Intermediate/Advanced)
3. Học tuần tự từng module
4. Thực hành song song với việc đọc lý thuyết
5. Xây dựng pipeline thực cho portfolio cá nhân
```

### Chuẩn Bị Phỏng Vấn — Interview Prep

```
1. Đọc 10-interview-prep/1-INTERVIEW_GUIDE.md
2. Ôn lại 02-configuration/ — cấu hình YAML (hay bị hỏi)
3. Nắm vững 05-optimization/ — caching, parallelism
4. Chuẩn bị câu chuyện STAR về pipeline bạn đã xây dựng
5. Thực hành so sánh CircleCI vs Jenkins vs GitHub Actions
```

### Trong Công Việc — On the Job

```
Dùng như tài liệu tham khảo nhanh:
- Cấu hình mới: Xem 02-configuration/
- Pipeline chậm: Xem 05-optimization/
- Lỗi build: Xem 08-monitoring/common-errors.md
- Vấn đề bảo mật: Xem 06-security/
- Tích hợp cloud: Xem 07-integration/
```

### Thiết Kế Hệ Thống — System Design

```
1. Xem 01-fundamentals/circleci-vs-alternatives.md để chọn công cụ
2. Dùng 03-workflows/ để thiết kế luồng CI/CD
3. Xem 09-advanced/monorepo-strategy.md cho monorepo
4. Áp dụng 06-security/ cho compliance requirements
```

---

## 📊 Ước Tính Thời Gian Học

| Module                    | Thời Gian | Độ Khó | Mức Độ Ưu Tiên |
| ------------------------- | --------- | ------ | -------------- |
| Nền Tảng — Fundamentals   | 3–4 giờ   | ⭐     | Bắt Buộc       |
| Cấu Hình — Configuration  | 5–8 giờ   | ⭐⭐   | Bắt Buộc       |
| Workflow Design           | 4–6 giờ   | ⭐⭐   | Bắt Buộc       |
| Orbs                      | 3–4 giờ   | ⭐⭐   | Nên Học        |
| Tối Ưu Hóa — Optimization | 5–7 giờ   | ⭐⭐⭐ | Bắt Buộc       |
| Bảo Mật — Security        | 4–6 giờ   | ⭐⭐   | Bắt Buộc       |
| Tích Hợp — Integration    | 6–10 giờ  | ⭐⭐⭐ | Theo Dự Án     |
| Monitoring & Debug        | 3–4 giờ   | ⭐⭐   | Nên Học        |
| Advanced Topics           | 8–12 giờ  | ⭐⭐⭐ | Tùy Chọn       |
| Interview Prep            | 4–8 giờ   | ⭐⭐   | Trước phỏng vấn |

**Tổng cộng: 45–65 giờ để thành thạo CircleCI (kèm phỏng vấn)**

---

## 🎓 Mức Độ Kỹ Năng Được Hỗ Trợ

### Người Mới — Beginner (0–6 tháng)

- [ ] Hiểu CI/CD và lợi ích cụ thể
- [ ] Chạy pipeline đầu tiên thành công
- [ ] Cấu hình job cơ bản với Docker executor
- [ ] Hiểu Workflow là gì và cách hoạt động
- [ ] Biết dùng environment variables cơ bản

**Thời gian để thành thạo mức này:** 1–2 tháng thực hành

### Trung Cấp — Intermediate (6 tháng–2 năm)

- [ ] Thiết kế workflow phức tạp với requires và filters
- [ ] Cấu hình caching và workspace tối ưu
- [ ] Dùng Contexts để quản lý secrets an toàn
- [ ] Tích hợp orbs phổ biến: aws-cli, docker, slack
- [ ] Song song hóa test để giảm thời gian build
- [ ] Debug pipeline bằng SSH

**Thời gian để thành thạo mức này:** 3–6 tháng thực hành

### Nâng Cao — Advanced (2+ năm)

- [ ] Phát triển và publish Custom Orb
- [ ] Triển khai Dynamic Config cho monorepo lớn
- [ ] Quản lý Self-Hosted Runner cluster
- [ ] Tối ưu chi phí toàn tổ chức
- [ ] Xây dựng CI/CD strategy cho nhiều team
- [ ] OIDC với cloud providers, zero static credentials

**Thời gian để thành thạo mức này:** Học liên tục

---

## 🔗 Điều Hướng Nhanh — Quick Navigation

| Nhu Cầu                 | Vị Trí                                                                       |
| ----------------------- | ---------------------------------------------------------------------------- |
| Tổng quan nhanh         | [README.md](README.md)                                                       |
| Cú pháp YAML config     | [02-configuration/README.md](02-configuration/README.md)                     |
| Thiết kế workflow       | [03-workflows/README.md](03-workflows/README.md)                             |
| Tìm hiểu orbs           | [04-orbs/README.md](04-orbs/README.md)                                       |
| Tối ưu pipeline chậm    | [05-optimization/README.md](05-optimization/README.md)                       |
| Bảo mật secrets         | [06-security/README.md](06-security/README.md)                               |
| Tích hợp AWS/GCP        | [07-integration/README.md](07-integration/README.md)                         |
| Pipeline bị lỗi         | [08-monitoring/README.md](08-monitoring/README.md)                           |
| Dynamic Config/Monorepo | [09-advanced/README.md](09-advanced/README.md)                               |
| Câu hỏi phỏng vấn       | [10-interview-prep/1-INTERVIEW_GUIDE.md](10-interview-prep/1-INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Độ Học — Learning Progress Tracker

Sao chép phần này và đánh dấu theo tiến độ:

```markdown
## Tiến Độ Học CircleCI

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] CI/CD concepts — khái niệm cơ bản
- [ ] Kiến trúc CircleCI
- [ ] Executor: Docker, Machine, macOS
- [ ] Job, Step, Workflow
- [ ] Chạy pipeline đầu tiên thành công

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)

- [ ] YAML config nâng cao
- [ ] Workflow: sequential, parallel, approval
- [ ] Caching với cache keys tối ưu
- [ ] Workspace giữa các jobs
- [ ] Contexts và secrets management
- [ ] Orbs: aws-cli, docker, slack

### Giai Đoạn 3: Vận Hành (Tuần 7–10)

- [ ] Test splitting và parallelism
- [ ] SSH debugging pipeline lỗi
- [ ] Tích hợp AWS/GCP deployment
- [ ] Pipeline Insights và tối ưu
- [ ] Security hardening

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Dynamic Config
- [ ] Path Filtering — monorepo
- [ ] Custom Orb development
- [ ] Self-Hosted Runner
- [ ] OIDC integration
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công — Success Criteria

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Cơ Bản

- [ ] Giải thích CI/CD và vị trí của CircleCI trong quy trình phát triển
- [ ] Viết `.circleci/config.yml` từ đầu không cần tài liệu
- [ ] Thiết kế workflow phù hợp cho bất kỳ dự án nào
- [ ] Giải thích sự khác biệt giữa Job, Workflow, Pipeline

### ✅ Năng Lực Vận Hành

- [ ] Tối ưu pipeline để giảm thời gian build ≥ 30%
- [ ] Bảo mật secrets đúng cách với Contexts và OIDC
- [ ] Debug pipeline lỗi một cách có hệ thống
- [ ] Tích hợp CircleCI với AWS/GCP để deploy tự động
- [ ] Xử lý flaky tests và giảm false failures

### ✅ Sẵn Sàng Phỏng Vấn — Interview Ready

- [ ] Trả lời tự tin top 20 câu hỏi CI/CD phổ biến
- [ ] Kể 2–3 câu chuyện thực tế về pipeline (STAR format)
- [ ] So sánh và chọn CI/CD tool phù hợp với bài toán cụ thể
- [ ] Thiết kế CI/CD architecture cho hệ thống lớn
- [ ] Hiểu chi phí và cách tối ưu credit CircleCI

---

## 💡 Mẹo Học Hiệu Quả — Pro Tips

1. **Học bằng cách làm:** Tạo repo demo và thực hành mỗi concept ngay lập tức
2. **Dùng CircleCI CLI:** Validate config cục bộ trước khi push tiết kiệm rất nhiều thời gian
3. **Đọc Insights thường xuyên:** Pipeline Insights — Thống Kê Pipeline tiết lộ bottleneck ẩn
4. **Cache aggressively — Bộ đệm mạnh:** Caching đúng có thể giảm 50–70% thời gian build
5. **Bảo mật từ đầu:** Dùng Contexts ngay từ project đầu tiên, không dùng project-level vars cho secrets nhạy cảm
6. **Ghi lại incidents:** Mỗi pipeline bị lỗi là cơ hội học và có thêm câu chuyện STAR
7. **Đọc orb source code:** Đọc source của certified orbs giúp học best practices thực tế
8. **Test splitting là game changer:** Áp dụng test splitting khi suite > 5 phút

---

## 📞 Đóng Góp — Contributing

Tìm lỗi hoặc muốn bổ sung nội dung?

Đây là tài liệu sống, luôn được cập nhật:

- [ ] Sửa lỗi nội dung hiện có
- [ ] Thêm ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Bổ sung câu hỏi phỏng vấn mới
- [ ] Cải thiện giải thích các khái niệm phức tạp
- [ ] Thêm use case tích hợp với các công cụ khác

---

**Cập Nhật Lần Cuối:** 2026-05-20
**Phiên Bản:** 1.6
**Trạng Thái:** ✅ Knowledge Base CircleCI HOÀN CHỈNH — README & INDEX | ✅ 01–09 modules | ✅ 10-interview-prep
