# Jenkins Knowledge Base — Chỉ Mục Toàn Bộ Tài Liệu

> Chỉ mục tổng thể cho tất cả tài liệu Jenkins, bao gồm trạng thái tạo file, thứ tự học và ước lượng thời gian.

## Cấu Trúc Thư Mục

```
Devops/jenkins/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/
│   ├── README.md                               Tổng quan Jenkins và kiến trúc
│   ├── 1-architecture.md                         Master/Agent, Executor, Build Queue, Workspace
│   ├── 2-installation.md                         Cài đặt Jenkins (Standalone, Docker, Kubernetes)
│   ├── 3-configuration.md                        System config, JVM settings, tool config
│   └── 4-jenkins-concepts.md                     Job, Build, Workspace, Artifact, View
│
├── 02-pipeline/
│   ├── README.md                               Tổng quan Pipeline cơ bản và nâng cao
│   ├── 1-declarative-pipeline.md                 Cú pháp Declarative: agent, stages, post, when
│   ├── 2-scripted-pipeline.md                    Groovy DSL: node, stage, try-catch, closure
│   ├── 3-jenkinsfile.md                          Quản lý Jenkinsfile trong SCM, best practices
│   ├── 4-pipeline-syntax.md                      Tham chiếu cú pháp đầy đủ, directive, option
│   └── 5-multibranch-pipeline.md                 Pipeline đa nhánh, PR builds, branch filtering
│
├── 03-plugins/
│   ├── README.md                               Quản lý và sử dụng Plugin
│   ├── 1-essential-plugins.md                    Danh sách Plugin thiết yếu: Git, Pipeline, Blue Ocean
│   ├── 2-pipeline-plugins.md                     Pipeline Utility Steps, Blue Ocean, Stage View
│   └── 3-integration-plugins.md                  Git, Docker, Kubernetes, Slack, SonarQube
│
├── 04-builds-triggers/
│   ├── README.md                               Kích hoạt build và quản lý build
│   ├── 1-webhooks.md                             GitHub/GitLab Webhook, cấu hình và bảo mật
│   ├── 2-cron-scheduling.md                      Cron syntax, Poll SCM, H symbol (hash cron)
│   ├── 3-build-parameters.md                     String, Choice, Boolean, File parameter
│   └── 4-build-artifacts.md                      Archive, Stash/Unstash, Fingerprint
│
├── 05-distributed-builds/
│   ├── README.md                               Kiến trúc Distributed Builds (build phân tán)
│   ├── 1-agent-configuration.md                  JNLP Agent, SSH Agent, Node label, offline node
│   ├── 2-docker-agents.md                        Docker Plugin, Dockerfile agent, registry auth
│   ├── 3-kubernetes-agents.md                    Kubernetes Plugin, Pod Template, JCasC
│   └── 4-node-management.md                      Quản lý Node, Executor, cloud node lifecycle
│
├── 06-security/
│   ├── README.md                               Tổng quan bảo mật Jenkins
│   ├── 1-authentication.md                       Jenkins DB, LDAP, Active Directory, OAuth2/SSO
│   ├── 2-authorization.md                        Matrix-based Security, Role Strategy Plugin, Project-based
│   ├── 3-credentials.md                          Secret text, Username/Password, SSH Key, Certificate, Vault
│   └── 4-security-best-practices.md             Hardening, CSP header, Script Security, audit log
│
├── 07-integration/
│   ├── README.md                               Tích hợp Jenkins với các công cụ DevOps
│   ├── 1-git-integration.md                      GitHub, GitLab, Bitbucket: webhook, SSH key, token
│   ├── 2-docker-integration.md                   Build image, push registry, Docker-in-Docker vs socket
│   ├── 3-kubernetes-deployment.md                kubectl, Helm deploy từ Jenkins pipeline
│   ├── 4-notifications.md                        Slack, Email (Mailer), Microsoft Teams, Telegram
│   └── 5-sonarqube-integration.md               SonarQube Scanner, Quality Gate, code coverage
│
├── 08-shared-libraries/
│   ├── README.md                               Thư viện dùng chung trong Pipeline
│   ├── 1-library-structure.md                    vars/, src/, resources/ — cấu trúc thư mục chuẩn
│   ├── 2-global-variables.md                     Tạo và dùng Global Variable, call step tùy chỉnh
│   └── 3-best-practices.md                       Versioning, testing với JenkinsPipelineUnit, docs
│
├── 09-monitoring-maintenance/
│   ├── README.md                               Giám sát và bảo trì Jenkins
│   ├── 1-monitoring.md                           Prometheus exporter, Grafana dashboard, queue metrics
│   ├── 2-disk-management.md                      Build Discard Policy, Workspace Cleanup Plugin
│   ├── 3-backup-restore.md                       ThinBackup Plugin, JENKINS_HOME backup, config backup
│   └── 4-upgrade-guide.md                        LTS vs Weekly, plugin compatibility, rollback plan
│
├── 10-troubleshooting/
│   ├── README.md                               Xử lý sự cố Jenkins
│   ├── 1-build-failures.md                       Log analysis, exit code, environment variable issues
│   ├── 2-agent-issues.md                         JNLP timeout, SSH key error, agent offline
│   ├── 3-performance-issues.md                   JVM heap, GC pressure, thread dump, slow UI
│   ├── 4-plugin-conflicts.md                     Classloader hell, dependency version mismatch
│   └── 5-production-checklist.md                Checklist trước khi đưa Jenkins lên production
│
└── 11-interview-prep/
    ├── README.md                               Hướng dẫn chuẩn bị phỏng vấn Jenkins
    ├── INTERVIEW_GUIDE.md                      Top 20 câu hỏi phỏng vấn thường gặp
    ├── 1-star-stories.md                         Câu chuyện STAR về CI/CD incidents
    └── 2-system-design-scenarios.md              Thiết kế hệ thống CI/CD pipeline
```

---

## Trạng Thái Tạo File

| Chủ Đề                                  | Thư Mục / File                       | Trạng Thái | Độ Phủ         |
| --------------------------------------- | ------------------------------------ | ---------- | -------------- |
| **Tổng Quan & Lộ Trình**               | README.md                            | ✅         | Toàn diện      |
| **Chỉ Mục**                            | INDEX.md                             | ✅         | Toàn diện      |
| **Kiến Trúc Nền Tảng**                 | 01-fundamentals/                     | ✅         | Toàn diện      |
| **Pipeline: Declarative & Scripted**   | 02-pipeline/                         | ✅         | Toàn diện      |
| **Quản Lý Plugin**                     | 03-plugins/                          | ✅         | Toàn diện      |
| **Build Triggers và Artifacts**        | 04-builds-triggers/                  | ✅         | Toàn diện      |
| **Distributed Builds**                 | 05-distributed-builds/               | ✅         | Toàn diện      |
| **Security & Credentials**             | 06-security/                         | ✅         | Toàn diện      |
| **Integration với Git, Docker, K8s**   | 07-integration/                      | ✅         | Toàn diện      |
| **Shared Libraries**                   | 08-shared-libraries/                 | ✅         | Toàn diện      |
| **Monitoring & Maintenance**           | 09-monitoring-maintenance/           | ✅         | Toàn diện      |
| **Troubleshooting**                    | 10-troubleshooting/                  | ✅         | Toàn diện      |
| **Interview Prep**                     | 11-interview-prep/                   | ✅         | Toàn diện      |

---

## Thứ Tự Ưu Tiên Tạo Nội Dung

### Ưu Tiên Cao (Kỹ năng cốt lõi — tạo trước)

- [x] `01-fundamentals/` — Kiến trúc và khái niệm nền tảng (4 files)
- [x] `02-pipeline/` — Pipeline Declarative và Scripted (6 files)
- [x] `05-distributed-builds/` — Distributed Builds và Agent (4 files)
- [x] `06-security/` — Security và Credentials (4 files)

### Ưu Tiên Trung Bình (Kỹ năng vận hành)

- [x] `07-integration/` — Tích hợp với Git, Docker, Kubernetes (5 files)
- [x] `08-shared-libraries/` — Shared Libraries nâng cao (3 files)
- [x] `04-builds-triggers/` — Build Triggers và Artifacts (4 files)
- [x] `11-interview-prep/INTERVIEW_GUIDE.md` — Câu hỏi phỏng vấn

### Ưu Tiên Thấp (Tham khảo và vận hành)

- [x] `03-plugins/` — Quản lý Plugin (3 files)
- [x] `09-monitoring-maintenance/` — Giám sát và bảo trì (4 files)
- [x] `10-troubleshooting/` — Xử lý sự cố (5 files)
- [x] `11-interview-prep/` — Câu chuyện STAR, System Design (4 files)

---

## Hướng Dẫn Sử Dụng Knowledge Base

### Cho Người Tự Học

```
1. Đọc README.md để nắm bức tranh tổng thể
2. Chọn lộ trình theo cấp độ (Beginner / Intermediate / Advanced)
3. Đi qua từng thư mục theo thứ tự số
4. Thực hành mỗi chủ đề trên lab Jenkins (Docker Compose hoặc Kubernetes)
5. Xây dựng pipeline thực tế cho một project cá nhân
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Ôn kỹ 02-pipeline/ — hay bị hỏi nhất
3. Nắm chắc 05-distributed-builds/ và 06-security/
4. Chuẩn bị 2–3 câu chuyện STAR về CI/CD
5. Luyện giải thích kiến trúc mà không nhìn tài liệu
```

### Cho Công Việc Hàng Ngày (DevOps Engineer)

```
Dùng như reference:
- Sự cố build:      10-troubleshooting/build-failures.md
- Cấp quyền team:   06-security/authorization.md
- Tích hợp mới:     07-integration/
- Chia sẻ pipeline: 08-shared-libraries/
- Backup Jenkins:   09-monitoring-maintenance/backup-restore.md
- Production prep:  10-troubleshooting/production-checklist.md
```

### Cho System Design

```
1. Đọc 01-fundamentals/ để hiểu kiến trúc Master/Agent
2. Nghiên cứu 05-distributed-builds/ cho scalability
3. Xem 06-security/ cho enterprise security requirements
4. Dùng 07-integration/ để thiết kế luồng CI/CD end-to-end
```

---

## Ước Lượng Thời Gian Học

| Thư Mục                  | Thời Gian    | Độ Khó  | Ưu Tiên |
| ------------------------ | ------------ | ------- | ------- |
| 01-fundamentals          | 3–4 giờ      | ⭐       | Bắt buộc |
| 02-pipeline              | 6–8 giờ      | ⭐⭐     | Bắt buộc |
| 03-plugins               | 2–3 giờ      | ⭐       | Nên học  |
| 04-builds-triggers       | 3–4 giờ      | ⭐⭐     | Bắt buộc |
| 05-distributed-builds    | 5–6 giờ      | ⭐⭐     | Bắt buộc |
| 06-security              | 5–6 giờ      | ⭐⭐     | Bắt buộc |
| 07-integration           | 6–8 giờ      | ⭐⭐     | Nên học  |
| 08-shared-libraries      | 4–5 giờ      | ⭐⭐⭐   | Nên học  |
| 09-monitoring-maintenance| 3–4 giờ      | ⭐⭐     | Nên học  |
| 10-troubleshooting       | 5–6 giờ      | ⭐⭐⭐   | Nên học  |
| 11-interview-prep        | 4–5 giờ      | ⭐⭐     | Bắt buộc |

**Tổng: 46–59 giờ để nắm vững Jenkins từ nền tảng đến nâng cao**

---

## Cấp Độ Kỹ Năng Được Hỗ Trợ

### Beginner (0–1 năm kinh nghiệm)

- [ ] Hiểu mô hình Master/Agent (chủ–tác nhân)
- [ ] Viết Declarative Pipeline cơ bản
- [ ] Cài đặt và cấu hình Plugin
- [ ] Kết nối Git repository với Jenkins
- [ ] Dùng Credentials lưu trữ secret an toàn

**Thời gian để nắm vững:** 1–2 tháng

### Intermediate (1–3 năm kinh nghiệm)

- [ ] Thiết kế Multibranch Pipeline với PR validation
- [ ] Cấu hình Docker Agent và Kubernetes Agent
- [ ] Viết Shared Libraries tái sử dụng
- [ ] Thiết lập Role-based Authorization
- [ ] Tích hợp SonarQube và Notifications

**Thời gian để nắm vững:** 2–3 tháng để nâng cao

### Advanced (3–5+ năm kinh nghiệm)

- [ ] Thiết kế Jenkins HA (High Availability — tính sẵn sàng cao)
- [ ] Xây dựng Shared Library ecosystem cho toàn tổ chức
- [ ] Jenkins Configuration as Code (JCasC)
- [ ] Migrate Jenkins sang Kubernetes-native setup
- [ ] Incident response và CI/CD post-mortem

**Thời gian để nắm vững:** Học liên tục

---

## Điều Hướng Nhanh

| Nhu Cầu                          | Vị Trí                                                         |
| -------------------------------- | -------------------------------------------------------------- |
| Tổng quan & lộ trình             | [README.md](README.md)                                         |
| Kiến trúc Master/Agent           | [01-fundamentals/architecture.md](01-fundamentals/architecture.md) |
| Viết Declarative Pipeline        | [02-pipeline/declarative-pipeline.md](02-pipeline/declarative-pipeline.md) |
| Cấu hình Docker Agent            | [05-distributed-builds/docker-agents.md](05-distributed-builds/docker-agents.md) |
| Quản lý Credentials              | [06-security/credentials.md](06-security/credentials.md)       |
| Tích hợp GitHub Webhook          | [07-integration/git-integration.md](07-integration/git-integration.md) |
| Shared Libraries cơ bản          | [08-shared-libraries/global-variables.md](08-shared-libraries/global-variables.md) |
| Xử lý build thất bại             | [10-troubleshooting/build-failures.md](10-troubleshooting/build-failures.md) |
| Câu hỏi phỏng vấn                | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md) |

---

## Theo Dõi Tiến Độ Học

Sao chép và điền vào khi học:

```markdown
## Tiến Độ Jenkins

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)
- [ ] Kiến trúc Master/Agent
- [ ] Job, Build, Workspace, Executor
- [ ] Declarative Pipeline cơ bản
- [ ] Jenkinsfile trong SCM
- [ ] Plugin thiết yếu

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)
- [ ] Multibranch Pipeline và PR builds
- [ ] Docker Agent cấu hình
- [ ] Credentials và Secret management
- [ ] Git Webhook trigger
- [ ] Role-based Authorization

### Giai Đoạn 3: Nâng Cao (Tuần 7–10)
- [ ] Kubernetes Agent (pod template)
- [ ] Shared Libraries
- [ ] Monitoring với Prometheus
- [ ] Backup và Restore Jenkins
- [ ] Performance tuning

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)
- [ ] Jenkins Configuration as Code (JCasC)
- [ ] High Availability setup
- [ ] Shared Library ecosystem
- [ ] System Design scenarios
- [ ] Mock interviews
```

---

## Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn cần đạt được:

### Năng Lực Nền Tảng

- [ ] Giải thích kiến trúc Master/Agent mà không cần tài liệu
- [ ] Viết Declarative Pipeline từ đầu cho một ứng dụng thực tế
- [ ] Cấu hình Distributed Builds với Docker Agent
- [ ] Thiết lập Authentication và Authorization đúng cách

### Năng Lực Vận Hành

- [ ] Debug build failure một cách có hệ thống (systematic)
- [ ] Thiết kế pipeline tái sử dụng qua Shared Libraries
- [ ] Tích hợp Jenkins với toàn bộ toolchain: Git → Build → Test → Deploy
- [ ] Đảm bảo bảo mật Credentials và không rò rỉ secret

### Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 20 câu hỏi Jenkins
- [ ] Kể được 2–3 câu chuyện STAR về CI/CD
- [ ] Thiết kế pipeline end-to-end cho hệ thống phức tạp
- [ ] Thảo luận được trade-off giữa Jenkins và các công cụ thay thế (GitHub Actions, GitLab CI, CircleCI)

---

## Mẹo Học Hiệu Quả

1. **Thực hành ngay:** Đừng chỉ đọc — dựng Jenkins local với Docker Compose và thử từng khái niệm
2. **Viết Jenkinsfile thật:** Áp dụng cho một project cá nhân, không chỉ ví dụ demo
3. **Gây lỗi chủ động:** Tạo build failure, agent offline để luyện troubleshooting
4. **Hiểu WHY, không chỉ HOW:** Tại sao dùng Shared Libraries? Tại sao Kubernetes Agent tốt hơn SSH Agent?
5. **Theo dõi Jenkins Community:** Blog, plugins.jenkins.io, JIRA changelog
6. **So sánh với tool khác:** Biết Jenkins khác gì GitHub Actions, GitLab CI để nói rõ trade-off

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.6
**Trạng Thái:** ✅ README & INDEX — Đã tạo | ✅ 06-security — Đã hoàn thành | ✅ 07-integration — Đã hoàn thành | ✅ 08-shared-libraries — Đã hoàn thành | ✅ 09-monitoring-maintenance — Đã hoàn thành | ✅ 10-troubleshooting — Đã hoàn thành | ✅ 11-interview-prep — Đã hoàn thành
