# ⚙️ GitHub Actions — Lộ Trình Kiến Thức CI/CD

> Hướng dẫn toàn diện về GitHub Actions — nền tảng CI/CD (Continuous Integration / Continuous Delivery — Tích Hợp Liên Tục / Phân Phối Liên Tục) tích hợp sẵn trong GitHub, từ cơ bản đến vận hành nâng cao.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Khái niệm CI/CD và vị trí của GitHub Actions trong hệ sinh thái DevOps
- [ ] Cấu trúc file workflow (`.github/workflows/*.yml`)
- [ ] Events (Sự kiện kích hoạt) — `push`, `pull_request`, `schedule`, `workflow_dispatch`
- [ ] Jobs (Công việc) và Steps (Bước thực thi)
- [ ] Runners (Máy chạy) — GitHub-hosted và self-hosted

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)**

- [ ] Secrets (Bí mật) và Variables (Biến) — quản lý thông tin nhạy cảm
- [ ] Artifacts (Tệp đầu ra) và Cache (Bộ đệm) — tối ưu thời gian build
- [ ] Matrix Strategy (Chiến lược Ma trận) — chạy song song nhiều môi trường
- [ ] Reusable Workflows (Workflow Tái Sử Dụng) và Composite Actions
- [ ] Environments (Môi trường) và Deployment Protection Rules (Quy Tắc Bảo Vệ Triển Khai)

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7–10)**

- [ ] Custom Actions — JavaScript, Docker, Composite
- [ ] Security Hardening (Tăng Cường Bảo Mật) — OIDC, least-privilege, dependency review
- [ ] Self-hosted Runners — cài đặt, bảo mật, auto-scaling
- [ ] Monitoring & Observability (Giám Sát & Quan Sát) workflow
- [ ] Cost Optimization (Tối Ưu Chi Phí) — phút sử dụng, concurrency limits

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] GitHub Actions cho Kubernetes (Triển Khai Container)
- [ ] Tích hợp với cloud providers (AWS, GCP, Azure)
- [ ] Advanced patterns — GitOps, trunk-based development
- [ ] Enterprise governance và compliance workflows

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực | Mức Độ Ưu Tiên | Thời Gian | Trạng Thái |
|---|---|---|---|
| **Workflow Fundamentals** (Nền Tảng Workflow) | ⭐⭐⭐ | 1 tuần | - |
| **CI Pipeline** (Đường Ống Tích Hợp Liên Tục) | ⭐⭐⭐ | 2 tuần | - |
| **CD & Deployments** (Phân Phối & Triển Khai) | ⭐⭐⭐ | 2 tuần | - |
| **Secrets & Security** (Bí Mật & Bảo Mật) | ⭐⭐⭐ | 1 tuần | - |
| **Reusable Workflows** (Workflow Tái Sử Dụng) | ⭐⭐⭐ | 1 tuần | - |
| **Custom Actions** (Action Tùy Chỉnh) | ⭐⭐ | 2 tuần | - |
| **Self-hosted Runners** (Máy Chạy Tự Quản Lý) | ⭐⭐ | 1 tuần | - |
| **Monitoring & Debugging** (Giám Sát & Gỡ Lỗi) | ⭐⭐ | 1 tuần | - |
| **Cost & Performance** (Chi Phí & Hiệu Năng) | ⭐⭐ | 1 tuần | - |
| **Enterprise & Compliance** (Doanh Nghiệp & Tuân Thủ) | ⭐ | 2 tuần | - |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. Nền Tảng** (`01-fundamentals/`)

- Kiến trúc GitHub Actions — Events, Workflows, Jobs, Steps, Actions
- YAML syntax (Cú Pháp YAML) và cấu trúc workflow
- Runners — GitHub-hosted (Ubuntu, Windows, macOS) và self-hosted
- Contexts (Ngữ Cảnh) và Expressions (Biểu Thức) — `${{ github.* }}`, `${{ env.* }}`
- Default environment variables (Biến Môi Trường Mặc Định)

### 📁 **2. CI Pipeline** (`02-ci-pipeline/`)

- **Trigger Events** — push, pull_request, schedule, workflow_dispatch, workflow_call
- Checkout và setup actions — `actions/checkout`, `actions/setup-node`
- Testing (Kiểm Thử) — unit tests, integration tests, code coverage
- Linting (Phân Tích Mã Tĩnh) và code quality gates
- Build artifacts (Tệp Đầu Ra Build) — packaging, versioning
- Status checks (Kiểm Tra Trạng Thái) bảo vệ nhánh

### 📁 **3. CD & Deployments** (`03-cd-deployments/`)

- Environments (Môi Trường) — staging, production, protection rules
- Deployment strategies (Chiến Lược Triển Khai) — rolling, blue/green, canary
- GitHub Releases và semantic versioning (Đánh Số Phiên Bản Ngữ Nghĩa)
- Container deployments — Docker build, push lên registry
- Deploy lên Kubernetes bằng `kubectl` / Helm
- Cloud deployments — AWS ECS/EKS, GCP GKE, Azure AKS

### 📁 **4. Secrets & Variables** (`04-secrets-variables/`)

- Secrets (Bí Mật) — organization, repository, environment scope
- Variables (Biến) — cấp độ tổ chức, repo, môi trường
- OIDC — OpenID Connect — Xác Thực Không Cần Long-lived Credentials
- Secret rotation (Xoay Vòng Bí Mật) và best practices
- Tích hợp HashiCorp Vault và AWS Secrets Manager

### 📁 **5. Reusable Workflows & Actions** (`05-reusable/`)

- Reusable Workflows (Workflow Tái Sử Dụng) — `workflow_call`
- Composite Actions (Action Kết Hợp) — đóng gói nhiều steps
- JavaScript Actions — Node.js, `@actions/core`, `@actions/github`
- Docker Container Actions — môi trường tùy chỉnh
- Marketplace Actions — đánh giá, pin version, fork khi cần

### 📁 **6. Matrix & Concurrency** (`06-matrix-concurrency/`)

- Matrix Strategy (Chiến Lược Ma Trận) — test đa phiên bản, đa OS
- `include` / `exclude` trong matrix
- Concurrency (Đồng Thời) — `concurrency` groups, cancel-in-progress
- Fan-out và fan-in patterns (Mẫu Phân Tán & Tập Hợp)

### 📁 **7. Caching & Performance** (`07-caching-performance/`)

- `actions/cache` — cache dependencies (npm, pip, Maven, Gradle)
- Cache key strategies (Chiến Lược Khóa Cache)
- Artifacts (Tệp Đầu Ra) — upload, download, retention policy
- Tối ưu hóa thời gian workflow
- Billing (Tính Phí) — phút sử dụng, storage, tính toán chi phí

### 📁 **8. Security** (`08-security/`)

- Least-privilege permissions (Quyền Tối Thiểu) — `permissions` block
- GITHUB_TOKEN (Token Tự Động) — scope và giới hạn
- OIDC với AWS / GCP / Azure — không cần lưu long-lived secrets
- Dependency review (Đánh Giá Phụ Thuộc) và Dependabot
- Code scanning — CodeQL, third-party SAST tools
- Supply chain security (Bảo Mật Chuỗi Cung Ứng) — pin actions by SHA

### 📁 **9. Self-hosted Runners** (`09-self-hosted-runners/`)

- Cài đặt và đăng ký runner
- Runner groups (Nhóm Runner) và labels
- Auto-scaling runners — ARC (Actions Runner Controller — Bộ Điều Khiển Runner)
- Bảo mật self-hosted runners — network isolation, ephemeral runners
- Maintenance (Bảo Trì) và monitoring runners

### 📁 **10. Monitoring & Debugging** (`10-monitoring-debugging/`)

- Workflow logs (Nhật Ký Workflow) và debug logging
- `ACTIONS_STEP_DEBUG` và `ACTIONS_RUNNER_DEBUG`
- Notifications (Thông Báo) — Slack, email, GitHub Issues
- Metrics (Chỉ Số) — thời gian chạy, tỉ lệ thất bại
- Audit logs (Nhật Ký Kiểm Toán) cho enterprise

### 📁 **11. Phỏng Vấn & Thực Hành** (`11-interview-prep/`)

- Top 20 câu hỏi GitHub Actions phỏng vấn
- System design scenarios (Kịch Bản Thiết Kế Hệ Thống)
- Bài tập thực hành — xây dựng CI/CD pipeline từ đầu
- Các lỗi phổ biến và cách debug

---

## 🎓 Theo Nền Tảng / Cloud

### **GitHub Actions + AWS**

```
Strengths: OIDC native, ECR, ECS, EKS, Lambda deployments
Ideal for: AWS-centric teams
Covered in: 03-cd-deployments, 04-secrets-variables, 08-security
```

### **GitHub Actions + GCP**

```
Strengths: Workload Identity Federation, GKE, Cloud Run
Ideal for: GCP-centric teams
Covered in: 03-cd-deployments, 08-security
```

### **GitHub Actions + Azure**

```
Strengths: Azure Login action, AKS, Azure Container Registry
Ideal for: Microsoft / .NET ecosystems
Covered in: 03-cd-deployments, 08-security
```

### **GitHub Actions + Kubernetes**

```
Strengths: kubectl, Helm, Kustomize, ARC (Actions Runner Controller)
Ideal for: Container-native teams
Covered in: 03-cd-deployments, 09-self-hosted-runners
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề | Thư Mục | Độ Ưu Tiên |
|---|---|---|
| Bắt đầu từ đâu | [Roadmap](./ROADMAP.md) | Đọc trước |
| Câu hỏi phỏng vấn | [11-interview-prep](./11-interview-prep/) | Trước phỏng vấn |
| CI Pipeline mẫu | [02-ci-pipeline](./02-ci-pipeline/) | Thiết yếu |
| Bảo mật OIDC | [08-security/oidc.md](./08-security/oidc.md) | Thiết yếu |
| Reusable Workflows | [05-reusable](./05-reusable/) | Nâng cao |

---

## 📊 Ma Trận Kỹ Năng

### Mức Cơ Bản (0–1 năm kinh nghiệm)

- [ ] Hiểu cấu trúc workflow YAML
- [ ] Biết các trigger events phổ biến
- [ ] Tạo CI pipeline đơn giản (checkout → test → build)
- [ ] Sử dụng GitHub-hosted runners
- [ ] Dùng secrets cơ bản trong workflow

### Mức Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Thiết kế CD pipeline với environments và approvals
- [ ] Viết reusable workflows và composite actions
- [ ] Triển khai matrix strategy cho đa môi trường
- [ ] Tích hợp OIDC thay thế long-lived credentials
- [ ] Cấu hình cache dependencies hiệu quả
- [ ] Tối ưu workflow cost và thời gian chạy

### Mức Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Xây dựng custom JavaScript / Docker actions
- [ ] Vận hành self-hosted runners với auto-scaling (ARC)
- [ ] Thiết kế GitOps pipeline end-to-end
- [ ] Triển khai enterprise governance và compliance workflows
- [ ] Security hardening toàn diện — supply chain, SAST, DAST
- [ ] Capacity planning và cost optimization ở quy mô lớn

---

## 🚀 Bắt Đầu Như Thế Nào

### Bước 1: Thiết Lập Mục Tiêu Học

```
Chọn hướng đi:
- Generalist DevOps Engineer (tất cả nền tảng)
- Specialist (chuyên sâu AWS / GCP / Azure)
- Platform Engineer (tự xây dựng internal CI/CD platform)
```

### Bước 2: Dựng Môi Trường Thực Hành

```bash
# Tạo repository thực hành trên GitHub
# Thêm file workflow đầu tiên
mkdir -p .github/workflows
cat > .github/workflows/hello.yml << 'EOF'
name: Hello World
on: push
jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello, GitHub Actions!"
EOF
```

### Bước 3: Học + Thực Hành

```
1. Đọc một module (30 phút)
2. Tạo workflow mẫu tương ứng (30 phút)
3. Chạy thử và quan sát logs (15–30 phút)
4. Review checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị theo phương pháp STAR:
- Situation  (Bối Cảnh)
- Task       (Nhiệm Vụ)
- Action     (Hành Động Đã Thực Hiện)
- Result     (Kết Quả Đạt Được)
```

---

## 📖 Tài Liệu Tham Khảo

### Đọc Bắt Buộc

- **GitHub Actions Official Docs** — docs.github.com/en/actions
- **"Automating Workflows with GitHub Actions"** — O'Reilly
- **GitHub Actions Security Hardening Guide** — docs.github.com

### Tài Liệu Chính Thức

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
- [Reusable Workflows Guide](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

### Bài Viết & Blog Quan Trọng

- GitHub Blog — Engineering category
- Actions Changelog (github.com/github/roadmap)
- DevOps Weekly Newsletter

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Thường Gặp Theo Danh Mục

#### CI/CD Fundamentals (Nền Tảng CI/CD)

- [ ] Sự khác nhau giữa `on: push` và `on: pull_request`?
- [ ] Giải thích vòng đời của một workflow run
- [ ] Cách chia sẻ data giữa các jobs?
- [ ] `needs` keyword dùng để làm gì?

#### Security (Bảo Mật)

- [ ] OIDC là gì và tại sao tốt hơn long-lived credentials?
- [ ] Phân biệt `GITHUB_TOKEN` và Personal Access Token
- [ ] Cách hạn chế quyền của workflow với `permissions` block
- [ ] Supply chain attack trong GitHub Actions là gì?

#### Performance & Cost (Hiệu Năng & Chi Phí)

- [ ] Cache hoạt động như thế nào trong GitHub Actions?
- [ ] Matrix strategy giúp gì cho parallel testing?
- [ ] Cách giảm billing minutes?
- [ ] Concurrency groups giải quyết vấn đề gì?

#### Architecture (Kiến Trúc)

- [ ] Khi nào dùng reusable workflow vs composite action?
- [ ] Cách thiết kế multi-environment deployment pipeline
- [ ] Self-hosted runner phù hợp với trường hợp nào?
- [ ] GitOps là gì và implement bằng GitHub Actions như thế nào?

Xem `11-interview-prep/` để có bộ Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc nhận vai trò mới, kiểm tra:

- [ ] Có thể giải thích sự khác nhau giữa workflow, job, step, action
- [ ] Có thể viết CI pipeline từ đầu không cần tham khảo
- [ ] Có thể thiết kế CD pipeline với staging và production environments
- [ ] Hiểu và áp dụng được OIDC cho cloud deployments
- [ ] Có thể viết reusable workflow với inputs/outputs
- [ ] Biết cách debug workflow bị lỗi từ logs
- [ ] Có thể tính toán và tối ưu chi phí GitHub Actions
- [ ] Nắm được supply chain security best practices
- [ ] Có thể setup self-hosted runner cho nhu cầu đặc thù
- [ ] Có thể thiết kế GitOps pipeline end-to-end

---

## 📞 Hỗ Trợ & Tài Nguyên

### Học Tập

- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [GitHub Skills (Interactive Labs)](https://skills.github.com/)
- [Act — Chạy GitHub Actions Locally](https://github.com/nektos/act)

### Công Cụ

- **act** — Chạy workflows locally để test nhanh
- **actionlint** — Linter cho file workflow YAML
- **GitHub CLI (gh)** — Quản lý workflows từ terminal
- **Grafana + Prometheus** — Giám sát self-hosted runners

### Cộng Đồng

- GitHub Community Forum
- r/devops (Reddit)
- DevOps StackExchange
- CNCF Slack (kênh #github-actions)

---

## 📋 Cách Sử Dụng Tài Liệu Này

### Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Làm bài tập thực hành cho mỗi module
4. Xây dựng một portfolio project hoàn chỉnh

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep](./11-interview-prep/)
2. Học sâu nền tảng cloud bạn đang ứng tuyển (AWS/GCP/Azure)
3. Chuẩn bị 2–3 câu chuyện incident STAR
4. Luyện tập giải thích khái niệm rõ ràng, không dùng buzzwords

### Áp Dụng Công Việc

1. Tham khảo [08-security](./08-security/) cho bảo mật pipeline
2. Dùng [05-reusable](./05-reusable/) để chuẩn hóa workflows
3. Xem [07-caching-performance](./07-caching-performance/) để tối ưu
4. Kiểm tra [02-ci-pipeline](./02-ci-pipeline/) cho best practices

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học phù hợp (Cơ Bản / Trung Cấp / Nâng Cao)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Tạo repository thực hành trên GitHub
├─ 5️⃣  Hoàn thành bài tập cho từng chủ đề
├─ 6️⃣  Xây dựng portfolio project CI/CD hoàn chỉnh
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
