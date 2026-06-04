# Kế Hoạch Học 90 Ngày GitHub Actions

> Lộ trình học có cấu trúc theo tuần — từ zero đến sẵn sàng phỏng vấn. Mỗi tuần có mục tiêu rõ ràng, tài liệu cần đọc, bài tập thực hành và checklist kiểm tra.

---

## 📋 Tổng Quan 90 Ngày

```
Tháng 1 (Ngày 1–30):    Nền Tảng & CI Pipeline
Tháng 2 (Ngày 31–60):   CD, Security & Advanced Topics
Tháng 3 (Ngày 61–90):   Thực Hành, Mock Interviews & Portfolio
```

**Cam kết thời gian:**
- **Minimum** (Tối Thiểu): 1 giờ/ngày = 90 giờ
- **Recommended** (Đề Nghị): 1.5 giờ/ngày = 135 giờ
- **Intensive** (Chuyên Sâu): 2+ giờ/ngày = 180+ giờ

**Nguyên tắc học:**
1. Đọc lý thuyết → thực hành ngay trong ngày
2. Không chuyển sang tuần mới nếu chưa hoàn thành 80% checklist
3. Ghi chú câu hỏi phát sinh → trả lời vào cuối tuần
4. Commit code thực hành lên GitHub hàng ngày

---

## 🗓️ Tháng 1: Nền Tảng & CI Pipeline (Ngày 1–30)

### Tuần 1 (Ngày 1–7): Kiến Trúc & Nền Tảng

**Mục Tiêu:** Hiểu kiến trúc GitHub Actions, viết được workflow đầu tiên.

**Tài Liệu Cần Đọc:**
- [ ] `01-fundamentals/README.md` — Kiến trúc tổng quan
- [ ] `01-fundamentals/1-workflow-syntax.md` — YAML syntax đầy đủ
- [ ] `01-fundamentals/2-events-triggers.md` — Events và triggers

**Thực Hành Mỗi Ngày:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Setup repository thực hành, làm Bài Tập 1 (Hello World) | 1 giờ |
| Thứ 3 | Thử các trigger khác nhau: workflow_dispatch, schedule | 1 giờ |
| Thứ 4 | Viết workflow in ra tất cả github.* contexts | 45 phút |
| Thứ 5 | Đọc về runners, thử tạo job chạy trên macOS | 1 giờ |
| Thứ 6 | Đọc contexts và expressions, thực hành `if` conditions | 1 giờ |
| Thứ 7 | Ôn tập, ghi chú, thử nghiệm tự do | 1 giờ |
| CN | Nghỉ ngơi | — |

**Checklist Cuối Tuần 1:**
- [ ] Giải thích được kiến trúc: Event → Workflow → Job → Step → Action
- [ ] Viết workflow với nhiều jobs, có `needs` dependency
- [ ] Dùng được `if`, `continue-on-error`, `always()`
- [ ] Hiểu sự khác nhau giữa `on: push` và `on: pull_request`
- [ ] Đọc được workflow logs và hiểu từng step

---

### Tuần 2 (Ngày 8–14): Runners & Environment Variables

**Mục Tiêu:** Nắm vững runners, environment variables, và secrets cơ bản.

**Tài Liệu Cần Đọc:**
- [ ] `01-fundamentals/3-runners.md` — GitHub-hosted vs self-hosted
- [ ] `01-fundamentals/4-contexts-expressions.md` — Contexts và expressions
- [ ] `01-fundamentals/5-environment-variables.md` — Env vars

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Tạo workflow chạy trên Windows + Linux cùng lúc | 1 giờ |
| Thứ 3 | Thực hành `$GITHUB_ENV`, `$GITHUB_OUTPUT`, `$GITHUB_STEP_SUMMARY` | 1 giờ |
| Thứ 4 | Tạo repository secret, dùng trong workflow | 45 phút |
| Thứ 5 | Tạo environment variable ở workflow, job, step level | 1 giờ |
| Thứ 6 | Viết workflow với expressions phức tạp | 1 giờ |
| Thứ 7 | Tổng hợp, làm lại các bài khó | 1 giờ |

**Checklist Cuối Tuần 2:**
- [ ] Giải thích được khi nào dùng GitHub-hosted vs self-hosted runner
- [ ] Dùng được `$GITHUB_ENV` để set environment variable cho steps sau
- [ ] Dùng được `$GITHUB_OUTPUT` để pass output giữa steps
- [ ] Hiểu rõ secret scopes: organization, repository, environment
- [ ] Viết expression `${{ ... }}` thành thạo

---

### Tuần 3 (Ngày 15–21): CI Pipeline Hoàn Chỉnh

**Mục Tiêu:** Xây dựng được CI pipeline production-ready.

**Tài Liệu Cần Đọc:**
- [ ] `02-ci-pipeline/README.md` — CI concepts
- [ ] `02-ci-pipeline/1-checkout-setup.md` — Checkout và setup actions
- [ ] `02-ci-pipeline/2-testing-strategies.md` — Testing trong CI
- [ ] `02-ci-pipeline/3-linting-quality.md` — Code quality gates

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Làm Bài Tập 2 (CI Pipeline Node.js) hoàn chỉnh | 1 giờ |
| Thứ 3 | Làm Bài Tập 3 (Matrix Strategy) | 45 phút |
| Thứ 4 | Thêm code coverage report vào CI pipeline | 1 giờ |
| Thứ 5 | Setup SonarQube/CodeClimate (hoặc đọc tài liệu) | 1 giờ |
| Thứ 6 | Đọc `02-ci-pipeline/4-build-artifacts.md`, thực hành | 1 giờ |
| Thứ 7 | Xây dựng CI pipeline cho dự án thực tế của bạn | 1.5 giờ |

**Checklist Cuối Tuần 3:**
- [ ] Viết CI pipeline từ đầu (checkout → lint → test → build) không cần tham khảo
- [ ] Test với matrix strategy trên 3 phiên bản Node.js
- [ ] Upload test coverage artifacts
- [ ] CI pipeline chạy < 5 phút
- [ ] Biết cách interpret (Giải Thích) test coverage reports

---

### Tuần 4 (Ngày 22–28): Caching & Performance

**Mục Tiêu:** Tối ưu CI pipeline, giảm thời gian chạy.

**Tài Liệu Cần Đọc:**
- [ ] `07-caching-performance/1-caching-dependencies.md`
- [ ] `07-caching-performance/2-cache-key-strategies.md`
- [ ] `07-caching-performance/3-artifacts.md`
- [ ] `07-caching-performance/4-performance-optimization.md`

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Làm Bài Tập 4 (Cache Dependencies) | 45 phút |
| Thứ 3 | Làm Bài Tập 5 (Artifacts) | 45 phút |
| Thứ 4 | So sánh thời gian chạy có và không có cache | 1 giờ |
| Thứ 5 | Implement path filtering để giảm unnecessary runs | 1 giờ |
| Thứ 6 | Đọc billing và cost optimization | 45 phút |
| Thứ 7 | Tối ưu CI pipeline của tuần 3 — target giảm 30% | 1 giờ |

**Checklist Cuối Tuần 4:**
- [ ] Cấu hình cache với correct key pattern (hashFiles)
- [ ] Hiểu restore-keys fallback strategy
- [ ] Upload/download artifacts giữa jobs
- [ ] Thêm path filtering vào existing workflow
- [ ] Đo được cache hit rate (>70% sau lần chạy đầu)

---

**Tổng Kết Tháng 1:**
- [ ] Đã hoàn thành Bài Tập 1–6
- [ ] Có CI pipeline hoàn chỉnh cho một project thực tế
- [ ] Tự tin giải thích kiến trúc GitHub Actions
- [ ] Nắm vững caching, artifacts, matrix strategy

---

## 🗓️ Tháng 2: CD, Security & Advanced Topics (Ngày 31–60)

### Tuần 5 (Ngày 29–35): CD Pipeline & Environments

**Mục Tiêu:** Xây dựng CD pipeline với staging và production environments.

**Tài Liệu Cần Đọc:**
- [ ] `03-cd-deployments/README.md`
- [ ] `03-cd-deployments/1-environments.md` — Environments và protection rules
- [ ] `03-cd-deployments/2-deployment-strategies.md` — Rolling, Blue/Green, Canary

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Setup staging + production environments trên GitHub | 30 phút |
| Thứ 3 | Làm Bài Tập 7 (Environments) | 1 giờ |
| Thứ 4 | Thêm manual approval cho production | 45 phút |
| Thứ 5 | Thiết kế Blue/Green deployment pipeline (vẽ sơ đồ) | 1 giờ |
| Thứ 6 | Implement Canary deployment concept (simulate) | 1 giờ |
| Thứ 7 | Kết hợp CI + CD thành full pipeline | 1.5 giờ |

**Checklist Cuối Tuần 5:**
- [ ] Tạo được staging và production environments với protection rules
- [ ] CD pipeline deploy staging tự động, production cần approval
- [ ] Giải thích được trade-offs giữa Rolling, Blue/Green, Canary
- [ ] Biết cách rollback deployment

---

### Tuần 6 (Ngày 36–42): Docker & Container Deployments

**Mục Tiêu:** Build, push và deploy Docker images trong CI/CD.

**Tài Liệu Cần Đọc:**
- [ ] `03-cd-deployments/3-docker-deployments.md`
- [ ] `03-cd-deployments/4-kubernetes-deployments.md` — Đọc nhanh

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Làm Bài Tập 9 (Docker Build & Push) | 1 giờ |
| Thứ 3 | Thêm Docker layer caching vào workflow | 45 phút |
| Thứ 4 | Setup multi-stage build (Xây Dựng Đa Giai Đoạn) | 1 giờ |
| Thứ 5 | Image scanning với Trivy | 45 phút |
| Thứ 6 | Deploy container lên staging (ECS/Cloud Run/Render) | 1 giờ |
| Thứ 7 | Review và document pipeline | 1 giờ |

---

### Tuần 7 (Ngày 43–49): Secrets & OIDC

**Mục Tiêu:** Master secret management và implement OIDC authentication.

**Tài Liệu Cần Đọc:**
- [ ] `04-secrets-variables/1-secrets-management.md`
- [ ] `04-secrets-variables/2-variables.md`
- [ ] `04-secrets-variables/3-oidc.md` — ĐỌC KỸ NHẤT

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Tạo secrets ở các levels: org, repo, environment | 45 phút |
| Thứ 3 | Làm Bài Tập 12 (OIDC với AWS) — nếu có AWS account | 1.5 giờ |
| Thứ 4 | Nếu không có AWS: đọc và viết giải thích OIDC | 1 giờ |
| Thứ 5 | Chuẩn bị trả lời câu hỏi 13 (OIDC) trong Interview Guide | 1 giờ |
| Thứ 6 | HashiCorp Vault integration concepts (đọc tài liệu) | 45 phút |
| Thứ 7 | Practice: giải thích OIDC flow bằng lời với người khác | 30 phút |

**Checklist Cuối Tuần 7:**
- [ ] Giải thích được tại sao OIDC tốt hơn long-lived credentials
- [ ] Biết cấu hình IAM Trust Policy cho GitHub Actions OIDC
- [ ] Phân biệt được repository secrets vs environment secrets
- [ ] Trả lời câu hỏi 13 trong 2 phút không cần nhìn notes

---

### Tuần 8 (Ngày 50–56): Reusable Workflows & Custom Actions

**Mục Tiêu:** Thiết kế reusable components cho CI/CD platform.

**Tài Liệu Cần Đọc:**
- [ ] `05-reusable/README.md`
- [ ] `05-reusable/1-reusable-workflows.md`
- [ ] `05-reusable/2-composite-actions.md`

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Làm Bài Tập 8 (Reusable Workflow) | 1 giờ |
| Thứ 3 | Làm Bài Tập 13 (Composite Action) | 1 giờ |
| Thứ 4 | Refactor CI pipeline thành reusable workflow | 1 giờ |
| Thứ 5 | Học `workflow_call` inputs/outputs/secrets patterns | 1 giờ |
| Thứ 6 | Đọc `05-reusable/3-javascript-actions.md` | 45 phút |
| Thứ 7 | Build simple JavaScript action (Hello World with @actions/core) | 1.5 giờ |

**Checklist Cuối Tuần 8:**
- [ ] Tạo được reusable workflow với inputs, outputs, secrets
- [ ] Tạo composite action với multiple steps
- [ ] Giải thích khi nào dùng reusable workflow vs composite action
- [ ] Hiểu `secrets: inherit` pattern

---

### Tuần 9 (Ngày 57–63): Security Hardening

**Mục Tiêu:** Implement security best practices toàn diện.

**Tài Liệu Cần Đọc:**
- [ ] `08-security/README.md`
- [ ] `08-security/1-permissions.md`
- [ ] `08-security/3-supply-chain.md`
- [ ] `08-security/6-security-hardening.md`

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Làm Bài Tập 10 (Security Scanning) | 1 giờ |
| Thứ 3 | Audit và pin tất cả actions trong existing workflows bởi SHA | 1 giờ |
| Thứ 4 | Setup Dependabot cho GitHub Actions | 30 phút |
| Thứ 5 | Review và harden permissions trong tất cả workflows | 1 giờ |
| Thứ 6 | Implement dependency review action | 45 phút |
| Thứ 7 | Security audit toàn bộ workflows đã viết | 1 giờ |

**Checklist Cuối Tuần 9:**
- [ ] Tất cả actions được pin bởi SHA đầy đủ
- [ ] Mọi job có minimal permissions block
- [ ] Dependabot setup cho weekly updates
- [ ] Giải thích được supply chain attack và cách phòng ngừa

---

**Tổng Kết Tháng 2:**
- [ ] Đã hoàn thành Bài Tập 7–13
- [ ] Full CI/CD pipeline với Docker + environments
- [ ] OIDC hoặc hiểu sâu OIDC concepts
- [ ] Reusable workflow library cơ bản
- [ ] Security hardened workflows

---

## 🗓️ Tháng 3: Thực Hành, Mock Interviews & Portfolio (Ngày 61–90)

### Tuần 10 (Ngày 64–70): Advanced Topics & Monitoring

**Mục Tiêu:** Nắm advanced topics cần thiết cho senior positions.

**Tài Liệu Cần Đọc:**
- [ ] `06-matrix-concurrency/2-concurrency.md` — Concurrency groups
- [ ] `09-self-hosted-runners/README.md` — Tổng quan
- [ ] `10-monitoring-debugging/1-debug-logging.md`
- [ ] `10-monitoring-debugging/3-metrics-observability.md`

**Thực Hành:**

| Ngày | Bài Tập | Thời Gian |
|---|---|---|
| Thứ 2 | Implement concurrency groups trong CD pipeline | 45 phút |
| Thứ 3 | Debug một workflow lỗi bằng ACTIONS_STEP_DEBUG | 1 giờ |
| Thứ 4 | Làm Bài Tập 11 (Slack Notifications) | 1 giờ |
| Thứ 5 | Đọc về ARC (Actions Runner Controller) | 1 giờ |
| Thứ 6 | Làm Bài Tập 14 (Release Automation) | 1.5 giờ |
| Thứ 7 | Ôn tập tất cả advanced concepts | 1 giờ |

---

### Tuần 11 (Ngày 71–77): Portfolio Project

**Mục Tiêu:** Xây dựng portfolio project CI/CD hoàn chỉnh để showcase trong phỏng vấn.

**Portfolio Project Requirements:**

Chọn một trong các options:

**Option A — Personal Project (Dự Án Cá Nhân):**
- Tìm một project của bạn trên GitHub (hoặc tạo mới)
- Implement full CI/CD pipeline
- Document rõ ràng trong README

**Option B — Sample App:**
- Clone một open-source Node.js/Python app đơn giản
- Build CI/CD pipeline từ đầu cho nó

**Checklist Portfolio:**
- [ ] CI pipeline: lint + test + coverage + build
- [ ] CD pipeline: staging (tự động) + production (manual approval)
- [ ] Docker: build + push lên GHCR
- [ ] Security: pin actions, minimal permissions, dependency review
- [ ] Reusable: ít nhất 1 reusable workflow hoặc composite action
- [ ] Notifications: Slack hoặc GitHub Issue khi deploy
- [ ] README giải thích kiến trúc pipeline

---

### Tuần 12 (Ngày 78–84): Chuẩn Bị Phỏng Vấn

**Mục Tiêu:** Đọc Interview Guide, chuẩn bị STAR stories, luyện tập.

**Ngày 78–79: Interview Guide**
- [ ] Đọc kỹ `1-INTERVIEW_GUIDE.md`
- [ ] Tự trả lời từng câu hỏi mà không nhìn đáp án
- [ ] Ghi chú những câu trả lời chưa tốt

**Ngày 80–81: STAR Stories**
- [ ] Đọc `2-star-stories.md`
- [ ] Tùy chỉnh ít nhất 5 câu chuyện STAR từ kinh nghiệm thực tế
- [ ] Luyện kể từng câu trong 3–4 phút

**Ngày 82–83: System Design**
- [ ] Đọc `3-system-design-scenarios.md`
- [ ] Vẽ sơ đồ cho 3 kịch bản trên giấy
- [ ] Luyện explain với timer 20 phút/kịch bản

**Ngày 84: Tổng hợp điểm yếu**
- [ ] List 5 topics còn yếu nhất
- [ ] Lên kế hoạch ôn lại trong tuần 13

---

### Tuần 13 (Ngày 85–90): Mock Interviews & Final Prep

**Ngày 85–86: Self Mock Interview**

Ngồi một mình, đặt timer, trả lời to từng câu hỏi:

```
Mock Interview Session (1 giờ):
1. Technical questions (30 phút):
   - 5 câu từ Interview Guide (random chọn)
   - Không nhìn đáp án
   - Ghi âm hoặc ghi video

2. System design (20 phút):
   - Chọn 1 kịch bản từ Kịch Bản 3 (System Design)
   - Vẽ và giải thích

3. Behavioral (10 phút):
   - 2 câu STAR
```

**Ngày 87–88: Mock Interview Với Người Khác**

Nhờ bạn bè, đồng nghiệp, hoặc tham gia cộng đồng DevOps practice:
- Người hỏi dùng câu hỏi từ Interview Guide
- Feedback sau mỗi câu
- Focus vào: rõ ràng, có ví dụ thực tế, không dùng jargon không cần thiết

**Ngày 89: Nghiên Cứu Công Ty**

Trước phỏng vấn với một công ty cụ thể:
- [ ] Tìm hiểu tech stack của họ (AWS/GCP/Azure? Kubernetes?)
- [ ] Xem job description kỹ — highlight keywords
- [ ] Chuẩn bị câu hỏi ngược (hỏi về pipeline hiện tại, pain points)
- [ ] Adjust STAR stories để relevant với company

**Ngày 90: Nghỉ Ngơi & Chuẩn Bị Tâm Lý**

- [ ] Xem lại portfolio project
- [ ] Đọc lại 5 điểm quan trọng nhất
- [ ] Ngủ đủ giấc
- [ ] Không học thêm gì mới — ôn lại những gì đã biết

---

## 📊 Milestone Tracking (Theo Dõi Cột Mốc)

### Cột Mốc 1: Cuối Tuần 2 (Ngày 14)
Bạn có thể:
- [ ] Giải thích GitHub Actions architecture không cần notes
- [ ] Viết basic workflow YAML từ đầu

### Cột Mốc 2: Cuối Tuần 4 (Ngày 28)
Bạn có thể:
- [ ] Build CI pipeline hoàn chỉnh cho Node.js app
- [ ] Tối ưu pipeline với cache — hit rate >70%

### Cột Mốc 3: Cuối Tuần 7 (Ngày 49)
Bạn có thể:
- [ ] Thiết kế và build CD pipeline với environments
- [ ] Giải thích OIDC flow và cấu hình IAM

### Cột Mốc 4: Cuối Tuần 9 (Ngày 63)
Bạn có thể:
- [ ] Viết reusable workflows
- [ ] Security audit và harden pipeline

### Cột Mốc 5: Cuối Ngày 90
Bạn có thể:
- [ ] Trả lời tự tin 20 câu phỏng vấn
- [ ] Kể 5 câu chuyện STAR với metrics thực tế
- [ ] Thiết kế CI/CD cho whiteboard question trong 20 phút

---

## 📅 Daily Schedule Template (Mẫu Lịch Hàng Ngày)

```
20:00 – 20:10  Review notes ngày hôm qua (10 phút)
20:10 – 20:45  Đọc tài liệu theo kế hoạch (35 phút)
20:45 – 21:30  Thực hành — viết code, tạo workflows (45 phút)
21:30 – 21:45  Ghi chú câu hỏi, tổng kết học được gì (15 phút)
```

**Tips duy trì động lực:**
- Commit code thực hành lên GitHub public — tạo streak
- Tham gia GitHub Actions discussions trên Reddit r/devops
- Theo dõi GitHub changelog — Actions ra tính năng mới liên tục
- Chia sẻ progress với study buddy hoặc cộng đồng

---

## 🎯 Self-Assessment Rubric (Thang Đánh Giá Bản Thân)

Dùng thang điểm 1–5 để tự đánh giá sau mỗi topic:

```
1 — Không biết / Chưa học
2 — Đọc qua nhưng chưa làm được
3 — Có thể làm với tài liệu tham khảo
4 — Có thể làm không cần tham khảo
5 — Có thể giải thích cho người khác
```

| Topic | Tuần 4 | Tuần 8 | Tuần 12 | Mục Tiêu |
|---|---|---|---|---|
| Workflow YAML Syntax | | | | 5 |
| CI Pipeline Design | | | | 5 |
| CD + Environments | | | | 5 |
| Secrets & OIDC | | | | 4 |
| Reusable Workflows | | | | 4 |
| Docker Integration | | | | 4 |
| Security Hardening | | | | 4 |
| Matrix Strategy | | | | 4 |
| Caching | | | | 5 |
| Debugging | | | | 4 |
| Self-hosted Runners | | | | 3 |
| System Design | | | | 4 |

**Target trước phỏng vấn:** Không có topic nào dưới 3 (Mid-level) hoặc 4 (Senior level).

---

## ⚡ Kế Hoạch Rút Gọn Nếu Ít Thời Gian

### 30 Ngày (2 giờ/ngày)

```
Tuần 1:  01-fundamentals + 02-ci-pipeline (Bài tập 1–3)
Tuần 2:  03-cd-deployments + 04-secrets (Bài tập 7, OIDC)
Tuần 3:  05-reusable + 08-security (Bài tập 8, 10)
Tuần 4:  Interview Guide + STAR stories + 2 mock interviews
```

### 14 Ngày (3 giờ/ngày) — Intensive Sprint

```
Ngày 1–3:   Fundamentals + CI pipeline (Bài tập 1–5)
Ngày 4–6:   CD + Secrets + OIDC (Bài tập 7, 12)
Ngày 7–9:   Reusable + Security (Bài tập 8, 10, 13)
Ngày 10–11: Interview Guide — đọc và luyện tập
Ngày 12–13: STAR stories + System design scenarios
Ngày 14:    Mock interview + nghỉ ngơi
```

### 7 Ngày — Emergency Prep

```
Ngày 1:  Fundamentals — chỉ đọc README files
Ngày 2:  CI Pipeline — làm Bài tập 2
Ngày 3:  CD + OIDC — đọc và hiểu khái niệm
Ngày 4:  Interview Guide câu 1–10
Ngày 5:  Interview Guide câu 11–20
Ngày 6:  STAR stories — chuẩn bị 3 câu
Ngày 7:  Self mock interview + nghỉ ngơi
```

---

## 📚 Tài Nguyên Bổ Sung

### Đọc Hàng Tuần
- GitHub Blog — Engineering section
- GitHub Actions Changelog

### Công Cụ Hỗ Trợ
- **act** — Test workflows locally trước khi push
- **actionlint** — Lint workflow YAML để phát hiện lỗi sớm
- **GitHub CLI (`gh`)** — Quản lý workflows từ terminal

### Cộng Đồng
- r/devops — thảo luận CI/CD thực tế
- CNCF Slack #github-actions — kênh chuyên biệt

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
