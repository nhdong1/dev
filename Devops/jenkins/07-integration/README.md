# 07 — Integration: Tích Hợp Jenkins với Hệ Sinh Thái DevOps

> Jenkins không chỉ là công cụ CI — nó là trung tâm điều phối (orchestration hub) kết nối toàn bộ toolchain DevOps: từ quản lý mã nguồn, đóng gói container, triển khai lên Kubernetes, đến kiểm tra chất lượng mã và thông báo kết quả.

---

## Mục Tiêu Học

Sau khi hoàn thành chủ đề này, bạn sẽ có thể:

- Cấu hình Jenkins tích hợp với GitHub, GitLab, Bitbucket qua Webhook và SSH Key
- Build và push Docker image từ Jenkins pipeline một cách an toàn
- Triển khai ứng dụng lên Kubernetes bằng `kubectl` và Helm từ pipeline
- Gửi thông báo build tới Slack, Email, Microsoft Teams, Telegram
- Tích hợp SonarQube để kiểm tra chất lượng mã và áp dụng Quality Gate

---

## Danh Sách File

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-git-integration.md](1-git-integration.md) | GitHub, GitLab, Bitbucket — Webhook, SSH Key, Personal Access Token | ⭐⭐ |
| [2-docker-integration.md](2-docker-integration.md) | Build image, push registry, Docker-in-Docker vs Docker socket | ⭐⭐ |
| [3-kubernetes-deployment.md](3-kubernetes-deployment.md) | kubectl, Helm deploy từ Jenkins pipeline | ⭐⭐⭐ |
| [4-notifications.md](4-notifications.md) | Slack, Email (Mailer), Microsoft Teams, Telegram | ⭐⭐ |
| [5-sonarqube-integration.md](5-sonarqube-integration.md) | SonarQube Scanner, Quality Gate, code coverage | ⭐⭐⭐ |

---

## Luồng Tích Hợp End-to-End (Minh Họa)

```
Developer push code
        │
        ▼
  Git Repository ──── Webhook ────► Jenkins Master
  (GitHub/GitLab)                        │
                                         ▼
                                   Pipeline Trigger
                                         │
                       ┌──────────────────────────────────┐
                       │          Pipeline Stages          │
                       │                                   │
                       │  1. Checkout   ◄── Git clone      │
                       │  2. Build      ◄── Maven/Gradle   │
                       │  3. Test       ◄── JUnit/pytest   │
                       │  4. SonarQube  ◄── Quality Gate   │
                       │  5. Docker     ◄── Build & Push   │
                       │  6. Deploy     ◄── kubectl/Helm   │
                       │  7. Notify     ◄── Slack/Email    │
                       └──────────────────────────────────┘
```

---

## Tổng Quan Công Nghệ Tích Hợp

### Git Platform (Nền Tảng Quản Lý Mã Nguồn)

| Nền Tảng | Webhook | Auth | SCM Plugin |
|----------|---------|------|-----------|
| GitHub | ✅ GitHub App / Webhook | PAT, SSH Key, GitHub App | `github` plugin |
| GitLab | ✅ GitLab Webhook | PAT, Deploy Key | `gitlab-plugin` |
| Bitbucket | ✅ Bitbucket Webhook | App Password, SSH | `bitbucket` plugin |

### Container và Deployment (Triển Khai Container)

| Công Cụ | Mục Đích | Plugin |
|---------|----------|--------|
| Docker | Build và push image | `docker-pipeline` |
| Docker Hub | Public registry | Credentials |
| AWS ECR | Private registry trên AWS | `amazon-ecr` |
| Kubernetes | Orchestration (điều phối container) | `kubernetes-cli` |
| Helm | Package manager cho K8s | `helm` CLI trực tiếp |

### Quality và Notification (Chất Lượng và Thông Báo)

| Công Cụ | Mục Đích | Plugin |
|---------|----------|--------|
| SonarQube | Static Analysis (phân tích tĩnh mã nguồn) | `sonarqube` |
| Slack | Chat notification | `slack` |
| Email | SMTP notification | `mailer` |
| Microsoft Teams | Enterprise notification | `office-365-connector` |
| Telegram | Bot notification | `telegram-notifications` |

---

## Kiến Trúc Credentials (Thông Tin Xác Thực) Cho Integration

Mọi tích hợp đều yêu cầu lưu trữ credentials an toàn trong Jenkins. Sơ đồ dưới đây cho thấy luồng credentials:

```
Jenkins Credentials Store
│
├── git-ssh-key          → SSH Private Key   → GitHub/GitLab/Bitbucket
├── docker-registry-cred → Username/Password → Docker Hub / ECR
├── kubeconfig-prod      → Secret File       → Kubernetes cluster
├── sonarqube-token      → Secret Text       → SonarQube Server
├── slack-webhook-url    → Secret Text       → Slack channel
└── smtp-credentials     → Username/Password → Email server
```

**Nguyên tắc quan trọng:**
- Không bao giờ hardcode (mã hóa cứng) credentials trong Jenkinsfile
- Dùng `withCredentials` block để inject (tiêm) credentials vào môi trường
- Credentials bị mask (che) trong log — không bao giờ hiển thị dưới dạng plain text

---

## Thứ Tự Học Đề Xuất

```
Bắt đầu
   │
   ▼
1-git-integration.md     ← Bắt buộc: kết nối Jenkins với code repository
   │
   ▼
2-docker-integration.md  ← Bắt buộc: build và publish container image
   │
   ▼
3-kubernetes-deployment.md ← Nâng cao: deploy lên K8s cluster
   │
   ▼
4-notifications.md       ← Thực tế: thông báo kết quả build cho team
   │
   ▼
5-sonarqube-integration.md ← Nâng cao: đảm bảo chất lượng mã
```

---

## Điều Kiện Tiên Quyết

Trước khi học chủ đề này, bạn nên nắm vững:

- [x] Viết Declarative Pipeline cơ bản (`02-pipeline/`)
- [x] Quản lý Credentials trong Jenkins (`06-security/credentials.md`)
- [x] Cấu hình Docker Agent (`05-distributed-builds/docker-agents.md`)
- [x] Hiểu khái niệm Plugin và cách cài đặt (`03-plugins/`)

---

## Câu Hỏi Phỏng Vấn Liên Quan

1. Webhook và Poll SCM khác nhau như thế nào? Khi nào dùng cái nào?
2. Docker-in-Docker (DinD) và Docker socket mounting — ưu nhược điểm?
3. Làm thế nào để triển khai lên nhiều Kubernetes cluster (đa cụm) từ một pipeline?
4. Quality Gate trong SonarQube hoạt động ra sao và tại sao nó quan trọng?
5. Cách bảo mật Webhook để chỉ Git server mới có thể kích hoạt build?

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
