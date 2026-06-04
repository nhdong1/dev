# GitHub Actions Knowledge Base — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về GitHub Actions — CI/CD (Continuous Integration / Continuous Delivery — Tích Hợp Liên Tục / Phân Phối Liên Tục)

## 📁 Cấu Trúc Thư Mục

```
Devops/github-action/
├── README.md                               [BẮT ĐẦU TẠI ĐÂY] Lộ trình học & tổng quan
│
├── 01-fundamentals/
│   ├── README.md                           Kiến trúc GitHub Actions, khái niệm cốt lõi
│   ├── 1-workflow-syntax.md                Cú pháp YAML workflow đầy đủ
│   ├── 2-events-triggers.md                Tất cả events — push, PR, schedule, dispatch
│   ├── 3-runners.md                        GitHub-hosted vs self-hosted runners
│   ├── 4-contexts-expressions.md           ${{ github.* }}, ${{ env.* }}, ${{ secrets.* }}
│   └── 5-environment-variables.md          Biến môi trường mặc định và tùy chỉnh
│
├── 02-ci-pipeline/
│   ├── README.md                           CI pipeline, triggers, status checks
│   ├── 1-checkout-setup.md                 actions/checkout, setup-node, setup-python
│   ├── 2-testing-strategies.md             Unit test, integration test, code coverage
│   ├── 3-linting-quality.md                ESLint, Prettier, SonarQube, code quality gates
│   ├── 4-build-artifacts.md                Đóng gói, versioning, upload artifacts
│   └── 5-branch-protection.md              Status checks, required reviews, merge rules
│
├── 03-cd-deployments/
│   ├── README.md                           CD pipeline, environments, deployment strategies
│   ├── 1-environments.md                   Staging, production, protection rules, approvals
│   ├── 2-deployment-strategies.md          Rolling, Blue/Green, Canary deployments
│   ├── 3-docker-deployments.md             Build image, push to registry, deploy container
│   ├── 4-kubernetes-deployments.md         kubectl, Helm, Kustomize, GitOps
│   ├── 5-aws-deployments.md                ECS, EKS, Lambda, S3, CloudFront
│   ├── 6-gcp-deployments.md                GKE, Cloud Run, Artifact Registry
│   └── 7-azure-deployments.md              AKS, Azure Container Registry, App Service
│
├── 04-secrets-variables/
│   ├── README.md                           Quản lý bí mật và biến
│   ├── 1-secrets-management.md             Repository, organization, environment secrets
│   ├── 2-variables.md                      Variables scope — org, repo, environment
│   ├── 3-oidc.md                           OIDC — OpenID Connect, không cần long-lived creds
│   ├── 4-vault-integration.md              Tích hợp HashiCorp Vault
│   └── 5-aws-secrets-manager.md            Tích hợp AWS Secrets Manager
│
├── 05-reusable/
│   ├── README.md                           Reusable workflows và actions
│   ├── 1-reusable-workflows.md             workflow_call, inputs, outputs, secrets
│   ├── 2-composite-actions.md              Composite actions — đóng gói nhiều steps
│   ├── 3-javascript-actions.md             Node.js actions, @actions/core, @actions/github
│   ├── 4-docker-actions.md                 Docker Container actions
│   └── 5-marketplace-guide.md              Chọn, đánh giá và pin actions từ Marketplace
│
├── 06-matrix-concurrency/
│   ├── README.md                           Matrix strategy và concurrency control
│   ├── 1-matrix-strategy.md                Test đa phiên bản, đa OS, include/exclude
│   ├── 2-concurrency.md                    Concurrency groups, cancel-in-progress
│   └── 3-fan-out-fan-in.md                 Fan-out & fan-in patterns — song song và tập hợp
│
├── 07-caching-performance/
│   ├── README.md                           Cache, artifacts, tối ưu chi phí
│   ├── 1-caching-dependencies.md           actions/cache cho npm, pip, Maven, Gradle, Go
│   ├── 2-cache-key-strategies.md           Cache key patterns, restore keys
│   ├── 3-artifacts.md                      Upload/download artifacts, retention policy
│   ├── 4-performance-optimization.md       Tối ưu thời gian chạy workflow
│   └── 5-billing-cost.md                   Billing minutes, storage, tính toán và tiết kiệm chi phí
│
├── 08-security/
│   ├── README.md                           Bảo mật toàn diện cho GitHub Actions
│   ├── 1-permissions.md                    permissions block, GITHUB_TOKEN scopes
│   ├── 2-oidc-cloud-auth.md                OIDC với AWS/GCP/Azure — không cần secrets
│   ├── 3-supply-chain.md                   Pin actions by SHA, Dependabot, dependency review
│   ├── 4-code-scanning.md                  CodeQL, SAST, DAST tích hợp trong CI
│   ├── 5-secret-scanning.md                Phát hiện secrets bị lộ trong code
│   └── 6-security-hardening.md             Checklist bảo mật toàn diện cho enterprise
│
├── 09-self-hosted-runners/
│   ├── README.md                           Self-hosted runners — cài đặt và vận hành
│   ├── 1-setup-registration.md             Cài đặt, đăng ký runner, labels, groups
│   ├── 2-arc-autoscaling.md                ARC — Actions Runner Controller, auto-scaling trên K8s
│   ├── 3-security-isolation.md             Network isolation, ephemeral runners, hardening
│   └── 4-maintenance-monitoring.md         Bảo trì, giám sát, upgrade runners
│
├── 10-monitoring-debugging/
│   ├── README.md                           Giám sát và debug workflows
│   ├── 1-debug-logging.md                  ACTIONS_STEP_DEBUG, ACTIONS_RUNNER_DEBUG
│   ├── 2-workflow-notifications.md         Slack, email, GitHub Issues notifications
│   ├── 3-metrics-observability.md          Thời gian chạy, tỉ lệ lỗi, custom metrics
│   └── 4-audit-logs.md                     Audit logs cho enterprise, compliance
│
├── 11-interview-prep/
│   ├── README.md                           Tổng quan phỏng vấn GitHub Actions
│   ├── 1-INTERVIEW_GUIDE.md                Top 20 câu hỏi phỏng vấn, tips trả lời
│   ├── 2-star-stories.md                   Mẫu câu chuyện STAR cho incident/projects
│   ├── 3-system-design-scenarios.md        Kịch bản thiết kế pipeline CI/CD
│   ├── 4-hands-on-exercises.md             Bài tập thực hành có đáp án
│   └── 5-90-day-study-plan.md              Kế hoạch học 90 ngày có cấu trúc

```

---

## ✅ Trạng Thái Các File Đã Tạo

| Chủ Đề | File | Trạng Thái | Chất Lượng |
|---|---|---|---|
| **Tổng Quan & Lộ Trình** | README.md | ✅ | Toàn diện |
| **Chỉ Mục** | INDEX.md | ✅ | Đầy đủ |
| **Nền Tảng — Tổng Quan** | 01-fundamentals/README.md | ✅ | Toàn diện |
| **Nền Tảng — YAML Syntax** | 01-fundamentals/1-workflow-syntax.md | ✅ | Toàn diện |
| **Nền Tảng — Events** | 01-fundamentals/2-events-triggers.md | ✅ | Toàn diện |
| **Nền Tảng — Runners** | 01-fundamentals/3-runners.md | ✅ | Toàn diện |
| **Nền Tảng — Contexts** | 01-fundamentals/4-contexts-expressions.md | ✅ | Toàn diện |
| **Nền Tảng — Env Vars** | 01-fundamentals/5-environment-variables.md | ✅ | Toàn diện |
| **CI Pipeline — Tổng Quan** | 02-ci-pipeline/README.md | ✅ | Toàn diện |
| **CI Pipeline — Checkout & Setup** | 02-ci-pipeline/1-checkout-setup.md | ✅ | Toàn diện |
| **CI Pipeline — Testing** | 02-ci-pipeline/2-testing-strategies.md | ✅ | Toàn diện |
| **CI Pipeline — Linting** | 02-ci-pipeline/3-linting-quality.md | ✅ | Toàn diện |
| **CI Pipeline — Build & Artifacts** | 02-ci-pipeline/4-build-artifacts.md | ✅ | Toàn diện |
| **CI Pipeline — Branch Protection** | 02-ci-pipeline/5-branch-protection.md | ✅ | Toàn diện |
| **CD & Deployments — Tổng Quan** | 03-cd-deployments/README.md | ✅ | Toàn diện |
| **CD & Deployments — Environments** | 03-cd-deployments/1-environments.md | ✅ | Toàn diện |
| **CD & Deployments — Strategies** | 03-cd-deployments/2-deployment-strategies.md | ✅ | Toàn diện |
| **CD & Deployments — Docker** | 03-cd-deployments/3-docker-deployments.md | ✅ | Toàn diện |
| **CD & Deployments — Kubernetes** | 03-cd-deployments/4-kubernetes-deployments.md | ✅ | Toàn diện |
| **CD & Deployments — AWS** | 03-cd-deployments/5-aws-deployments.md | ✅ | Toàn diện |
| **CD & Deployments — GCP** | 03-cd-deployments/6-gcp-deployments.md | ✅ | Toàn diện |
| **CD & Deployments — Azure** | 03-cd-deployments/7-azure-deployments.md | ✅ | Toàn diện |
| **Secrets — Tổng Quan** | 04-secrets-variables/README.md | ✅ | Toàn diện |
| **Secrets — Secrets Management** | 04-secrets-variables/1-secrets-management.md | ✅ | Toàn diện |
| **Secrets — Variables** | 04-secrets-variables/2-variables.md | ✅ | Toàn diện |
| **Secrets — OIDC** | 04-secrets-variables/3-oidc.md | ✅ | Toàn diện |
| **Secrets — HashiCorp Vault** | 04-secrets-variables/4-vault-integration.md | ✅ | Toàn diện |
| **Secrets — AWS Secrets Manager** | 04-secrets-variables/5-aws-secrets-manager.md | ✅ | Toàn diện |
| **Reusable — Tổng Quan** | 05-reusable/README.md | ✅ | Toàn diện |
| **Reusable — Reusable Workflows** | 05-reusable/1-reusable-workflows.md | ✅ | Toàn diện |
| **Reusable — Composite Actions** | 05-reusable/2-composite-actions.md | ✅ | Toàn diện |
| **Reusable — JavaScript Actions** | 05-reusable/3-javascript-actions.md | ✅ | Toàn diện |
| **Reusable — Docker Actions** | 05-reusable/4-docker-actions.md | ✅ | Toàn diện |
| **Reusable — Marketplace Guide** | 05-reusable/5-marketplace-guide.md | ✅ | Toàn diện |
| **Matrix & Concurrency — Tổng Quan** | 06-matrix-concurrency/README.md | ✅ | Toàn diện |
| **Matrix & Concurrency — Matrix Strategy** | 06-matrix-concurrency/1-matrix-strategy.md | ✅ | Toàn diện |
| **Matrix & Concurrency — Concurrency** | 06-matrix-concurrency/2-concurrency.md | ✅ | Toàn diện |
| **Matrix & Concurrency — Fan-out & Fan-in** | 06-matrix-concurrency/3-fan-out-fan-in.md | ✅ | Toàn diện |
| **Caching — Tổng Quan** | 07-caching-performance/README.md | ✅ | Toàn diện |
| **Caching — Dependencies** | 07-caching-performance/1-caching-dependencies.md | ✅ | Toàn diện |
| **Caching — Cache Key Strategies** | 07-caching-performance/2-cache-key-strategies.md | ✅ | Toàn diện |
| **Caching — Artifacts** | 07-caching-performance/3-artifacts.md | ✅ | Toàn diện |
| **Caching — Performance Optimization** | 07-caching-performance/4-performance-optimization.md | ✅ | Toàn diện |
| **Caching — Billing & Cost** | 07-caching-performance/5-billing-cost.md | ✅ | Toàn diện |
| **Security — Tổng Quan** | 08-security/README.md | ✅ | Toàn diện |
| **Security — Permissions** | 08-security/1-permissions.md | ✅ | Toàn diện |
| **Security — OIDC Cloud Auth** | 08-security/2-oidc-cloud-auth.md | ✅ | Toàn diện |
| **Security — Supply Chain** | 08-security/3-supply-chain.md | ✅ | Toàn diện |
| **Security — Code Scanning** | 08-security/4-code-scanning.md | ✅ | Toàn diện |
| **Security — Secret Scanning** | 08-security/5-secret-scanning.md | ✅ | Toàn diện |
| **Security — Hardening** | 08-security/6-security-hardening.md | ✅ | Toàn diện |
| **Self-hosted Runners — Tổng Quan** | 09-self-hosted-runners/README.md | ✅ | Toàn diện |
| **Self-hosted Runners — Setup & Registration** | 09-self-hosted-runners/1-setup-registration.md | ✅ | Toàn diện |
| **Self-hosted Runners — ARC & Auto-scaling** | 09-self-hosted-runners/2-arc-autoscaling.md | ✅ | Toàn diện |
| **Self-hosted Runners — Security & Isolation** | 09-self-hosted-runners/3-security-isolation.md | ✅ | Toàn diện |
| **Self-hosted Runners — Maintenance & Monitoring** | 09-self-hosted-runners/4-maintenance-monitoring.md | ✅ | Toàn diện |
| **Monitoring & Debugging — Tổng Quan** | 10-monitoring-debugging/README.md | ✅ | Toàn diện |
| **Monitoring & Debugging — Debug Logging** | 10-monitoring-debugging/1-debug-logging.md | ✅ | Toàn diện |
| **Monitoring & Debugging — Notifications** | 10-monitoring-debugging/2-workflow-notifications.md | ✅ | Toàn diện |
| **Monitoring & Debugging — Metrics & Observability** | 10-monitoring-debugging/3-metrics-observability.md | ✅ | Toàn diện |
| **Monitoring & Debugging — Audit Logs** | 10-monitoring-debugging/4-audit-logs.md | ✅ | Toàn diện |
| **Interview Prep — Tổng Quan** | 11-interview-prep/README.md | ✅ | Toàn diện |
| **Interview Prep — Top 20 Câu Hỏi** | 11-interview-prep/1-INTERVIEW_GUIDE.md | ✅ | Toàn diện |
| **Interview Prep — STAR Stories** | 11-interview-prep/2-star-stories.md | ✅ | Toàn diện |
| **Interview Prep — System Design** | 11-interview-prep/3-system-design-scenarios.md | ✅ | Toàn diện |
| **Interview Prep — Hands-on Exercises** | 11-interview-prep/4-hands-on-exercises.md | ✅ | Toàn diện |
| **Interview Prep — 90-Day Study Plan** | 11-interview-prep/5-90-day-study-plan.md | ✅ | Toàn diện |

---

## 🎯 Ưu Tiên Tạo Nội Dung (Theo Thứ Tự)

### Ưu Tiên Cao (Kỹ Năng Cốt Lõi)

- [x] `01-fundamentals/` — Kiến trúc, syntax, events, runners, contexts, env vars ✅
- [x] `02-ci-pipeline/` — CI pipeline hoàn chỉnh (6 files) ✅
- [x] `03-cd-deployments/` — CD pipeline hoàn chỉnh (8 files) ✅
- [x] `04-secrets-variables/` — Secrets, Variables, OIDC, Vault, AWS Secrets Manager (6 files) ✅
- [x] `11-interview-prep/` — Phỏng vấn (6 files) ✅

### Ưu Tiên Trung Bình (Kỹ Năng Nâng Cao)

- [x] `05-reusable/` — Reusable Workflows, Composite Actions, JS Actions, Docker Actions, Marketplace Guide (6 files) ✅
- [x] `08-security/` — Bảo mật pipeline toàn diện (7 files) ✅
- [x] `09-self-hosted-runners/` — Setup, ARC auto-scaling, Security, Maintenance (5 files) ✅
- [x] `07-caching-performance/` — Cache, Artifacts, Performance, Billing (6 files) ✅
- [ ] `ROADMAP.md` — Lộ trình 90 ngày

### Ưu Tiên Thấp (Tài Liệu Tham Khảo)

- [ ] `GLOSSARY.md` — Thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu học
- [ ] `CHECKLIST.md` — Checklist pre-deploy
- [x] `10-monitoring-debugging/` — Debug và observability ✅
- [x] `06-matrix-concurrency/` — Matrix Strategy, Concurrency, Fan-out/Fan-in (4 files) ✅

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Tự Học

```
1. Bắt đầu với README.md
2. Chọn Lộ Trình Học (Cơ Bản / Trung Cấp / Nâng Cao)
3. Đi qua từng section theo thứ tự
4. Làm bài tập thực hành (tạo repository GitHub)
5. Xây dựng portfolio project CI/CD hoàn chỉnh
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào nền tảng cloud của công ty ứng tuyển
3. Học sâu 02-ci-pipeline/ và 03-cd-deployments/ (luôn hỏi)
4. Học OIDC trong 04-secrets-variables/ (hot topic năm 2025-2026)
5. Chuẩn bị câu chuyện STAR từ dự án thực tế
6. Luyện tập với người khác (mock interview)
```

### Áp Dụng Công Việc

```
Dùng làm tài liệu tham khảo:
- Trước khi setup pipeline mới: Đọc 02-ci-pipeline/ + 03-cd-deployments/
- Khi có security concern: Đến 08-security/
- Khi workflow chậm: Đến 07-caching-performance/
- Khi cần tái sử dụng: Đến 05-reusable/
- Khi debug lỗi: Đến 10-monitoring-debugging/
```

### Thiết Kế Hệ Thống

```
1. Đọc 03-cd-deployments/README.md để nắm deployment strategies
2. Dùng 04-secrets-variables/oidc.md cho cloud authentication
3. Theo 08-security/ cho enterprise security requirements
4. Theo 05-reusable/ để thiết kế internal CI/CD platform
5. Dùng 09-self-hosted-runners/ nếu cần on-premise runners
```

---

## 📊 Ước Tính Thời Gian Học

| Section | Thời Gian | Độ Khó | Ưu Tiên |
|---|---|---|---|
| Fundamentals (Nền Tảng) | 4–6 giờ | ⭐ | Bắt Buộc |
| CI Pipeline | 6–8 giờ | ⭐⭐ | Bắt Buộc |
| CD & Deployments | 8–10 giờ | ⭐⭐ | Bắt Buộc |
| Secrets & Security | 6–8 giờ | ⭐⭐⭐ | Bắt Buộc |
| Reusable Workflows | 4–6 giờ | ⭐⭐ | Nên Có |
| Matrix & Concurrency | 3–4 giờ | ⭐⭐ | Nên Có |
| Caching & Performance | 3–4 giờ | ⭐⭐ | Nên Có |
| Self-hosted Runners | 4–6 giờ | ⭐⭐⭐ | Nên Có |
| Advanced Topics | 10–15 giờ | ⭐⭐⭐ | Tốt Hơn |

**Tổng: 50–70 giờ để có kiến thức GitHub Actions toàn diện**

---

## 🎓 Các Cấp Độ Được Hỗ Trợ

### Cơ Bản (0–1 năm kinh nghiệm)

- [ ] Hiểu workflow YAML syntax
- [ ] Biết tạo CI pipeline đơn giản
- [ ] Dùng được GitHub Secrets
- [ ] Sử dụng actions từ Marketplace
- [ ] Đọc hiểu workflow logs

**Thời gian đạt được:** 1–2 tháng

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Thiết kế multi-environment CD pipeline
- [ ] Viết reusable workflows
- [ ] Tích hợp OIDC với cloud
- [ ] Tối ưu cache và performance
- [ ] Debug workflow phức tạp

**Thời gian đạt được:** 2–3 tháng để nâng cấp

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Xây dựng custom actions
- [ ] Vận hành self-hosted runners quy mô lớn
- [ ] Thiết kế enterprise CI/CD platform
- [ ] Security hardening và compliance
- [ ] Cost optimization ở quy mô lớn

**Thời gian đạt được:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu | Vị Trí |
|---|---|
| Tổng quan nhanh | [README.md](README.md) |
| CI pipeline mẫu | [02-ci-pipeline/README.md](02-ci-pipeline/README.md) |
| Deploy lên production | [03-cd-deployments/README.md](03-cd-deployments/README.md) |
| Cấu hình OIDC | [04-secrets-variables/oidc.md](04-secrets-variables/oidc.md) |
| Tái sử dụng workflow | [05-reusable/reusable-workflows.md](05-reusable/reusable-workflows.md) |
| Bảo mật pipeline | [08-security/security-hardening.md](08-security/security-hardening.md) |
| Setup self-hosted runner | [09-self-hosted-runners/README.md](09-self-hosted-runners/README.md) |
| Câu hỏi phỏng vấn | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Độ Học

Sao chép phần này và theo dõi tiến độ cá nhân:

```markdown
## Tiến Độ Học GitHub Actions

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)
- [ ] Kiến trúc GitHub Actions
- [ ] YAML workflow syntax
- [ ] Trigger events
- [ ] Runners và environments
- [ ] Contexts và expressions

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)
- [ ] CI pipeline đầy đủ
- [ ] CD với environments
- [ ] Secrets và OIDC
- [ ] Cache và artifacts
- [ ] Reusable workflows

### Giai Đoạn 3: Nâng Cao (Tuần 7–10)
- [ ] Custom actions
- [ ] Security hardening
- [ ] Self-hosted runners
- [ ] Matrix strategy
- [ ] Cost optimization

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)
- [ ] GitOps pipeline
- [ ] Enterprise governance
- [ ] ARC auto-scaling
- [ ] System design scenarios
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn phải có khả năng:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích kiến trúc GitHub Actions không cần nhìn tài liệu
- [ ] Viết CI pipeline hoàn chỉnh từ đầu (checkout → test → build → artifact)
- [ ] Thiết kế CD pipeline với staging → production approval gates
- [ ] Đọc hiểu và debug workflow logs

### ✅ Năng Lực Vận Hành

- [ ] Thiết lập OIDC authentication với AWS/GCP/Azure
- [ ] Viết reusable workflow được dùng lại nhiều repos
- [ ] Tối ưu workflow cost giảm ≥30% thời gian chạy
- [ ] Cài đặt và vận hành self-hosted runner
- [ ] Implement supply chain security (pin by SHA, dependency review)

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 20 câu hỏi GitHub Actions
- [ ] Kể được 2–3 câu chuyện STAR về CI/CD pipeline
- [ ] Thiết kế pipeline CI/CD cho system design interview
- [ ] Thảo luận trade-offs và constraints
- [ ] Nắm sâu nền tảng cloud mà công ty đang dùng

---

## 🚀 Bước Tiếp Theo

### Ngay Tuần Này

1. Đọc README.md kỹ từ đầu đến cuối
2. Chọn lộ trình học phù hợp với level hiện tại
3. Tạo repository GitHub để thực hành
4. Viết workflow "Hello World" đầu tiên

### 2 Tuần Tới

1. Hoàn thành `01-fundamentals/`
2. Xây dựng CI pipeline thực tế cho một project
3. Bắt đầu `03-cd-deployments/`
4. Làm bài tập hands-on cho mỗi chủ đề

### 1 Tháng Tới

1. Hoàn thành tất cả core topics (01–05)
2. Tích hợp OIDC với một cloud provider
3. Chuẩn bị 2–3 câu chuyện STAR
4. Thực hành mock interview với người khác

### 3 Tháng Tới

1. Master một cloud deployment pattern hoàn toàn
2. Xây dựng internal CI/CD template library
3. Đóng góp custom action lên Marketplace
4. Bắt đầu ứng tuyển hoặc nhận thêm DevOps responsibilities

---

## 💡 Mẹo Thực Tế

1. **Học bằng cách làm:** Đừng chỉ đọc — tạo workflows thật, gây lỗi, sửa lỗi
2. **Dùng `act` để test locally:** Tiết kiệm GitHub Actions minutes khi phát triển
3. **Pin actions theo SHA:** `actions/checkout@v4` có thể bị thay đổi, SHA thì không
4. **Đọc workflow logs kỹ:** 90% vấn đề có thể debug qua logs mà không cần thêm step
5. **Thiết kế workflow như code:** Tái sử dụng, đặt tên rõ ràng, viết comments khi cần
6. **Bảo mật ngay từ đầu:** OIDC và least-privilege không phải optional
7. **Theo dõi billing:** Kiểm tra GitHub Actions usage hàng tuần để tránh surprise bills
8. **Đọc changelog:** GitHub Actions ra tính năng mới liên tục — theo dõi để cập nhật

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Chào đón mọi đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm sections cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Workflow templates cho các use cases phổ biến

---

## 📄 Giấy Phép

Knowledge base này mở để học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 2.0
**Trạng Thái:** ✅ README + INDEX + 01-fundamentals + 02-ci-pipeline + 03-cd-deployments + 04-secrets-variables + 05-reusable + 06-matrix-concurrency + 07-caching-performance + 08-security + 09-self-hosted-runners + 10-monitoring-debugging + 11-interview-prep Hoàn Thành
