# Jenkins — Lộ Trình Học và Vận Hành CI/CD

> Hướng dẫn toàn diện về Jenkins — từ kiến trúc nền tảng đến vận hành pipeline production, bao gồm tất cả kỹ năng cốt lõi dành cho kỹ sư DevOps và Backend.

## Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Kiến trúc Jenkins: Master/Agent — mô hình điều phối và thực thi build
- [ ] Job (công việc) và Build (lần thực thi) — các khái niệm cơ bản
- [ ] Pipeline cơ bản: Declarative Pipeline (pipeline khai báo) và Scripted Pipeline (pipeline kịch bản)
- [ ] Jenkinsfile — lưu cấu hình pipeline trong Source Control
- [ ] Plugin (tiện ích mở rộng) thiết yếu và quản lý plugin

### **Giai Đoạn 2: Kỹ Năng Vận Hành Cốt Lõi (Tuần 3–6)**

- [ ] Distributed Builds (build phân tán): Agent, Node, Executor (bộ thực thi)
- [ ] Build Triggers (kích hoạt build): Webhook, Cron, Poll SCM
- [ ] Credentials (thông tin xác thực): Secret, SSH Key, API Token
- [ ] Integration (tích hợp): Git, Docker, Kubernetes, Slack
- [ ] Security (bảo mật): Authentication (xác thực), Authorization (phân quyền), Role Strategy

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7–10)**

- [ ] Shared Libraries (thư viện dùng chung) — tái sử dụng code pipeline
- [ ] Multibranch Pipeline (pipeline đa nhánh) và PR builds
- [ ] Kubernetes Agents (agent trên Kubernetes) — dynamic build workers
- [ ] Monitoring (giám sát): Jenkins metrics, Prometheus exporter
- [ ] Backup và Restore — bảo vệ cấu hình Jenkins

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] GitOps với Jenkins — tích hợp ArgoCD, FluxCD
- [ ] Pipeline as Code (pipeline dưới dạng code) — best practices nâng cao
- [ ] Tối ưu hiệu suất: JVM tuning, GC (Garbage Collection) settings
- [ ] High Availability (tính sẵn sàng cao) cho Jenkins
- [ ] Migration (di chuyển) và nâng cấp Jenkins LTS

---

## Năng Lực Cốt Lõi

| Năng Lực                                | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| --------------------------------------- | ---------- | --------- | ---------- |
| **Kiến Trúc và Khái Niệm Cơ Bản**     | ⭐⭐⭐      | 1 tuần    | -          |
| **Pipeline: Declarative & Scripted**   | ⭐⭐⭐      | 2 tuần    | -          |
| **Distributed Builds (Build Phân Tán)**| ⭐⭐⭐      | 1 tuần    | -          |
| **Security & Credentials (Bảo Mật)**  | ⭐⭐⭐      | 1 tuần    | -          |
| **Integration với Git & Docker**       | ⭐⭐⭐      | 1 tuần    | -          |
| **Shared Libraries (Thư Viện Chung)** | ⭐⭐⭐      | 1 tuần    | -          |
| **Monitoring & Maintenance (Vận Hành)**| ⭐⭐⭐      | 1 tuần    | -          |
| **Troubleshooting (Xử Lý Sự Cố)**     | ⭐⭐⭐      | 1 tuần    | -          |
| **Kubernetes Agents**                  | ⭐⭐        | 1 tuần    | -          |
| **Plugin Management (Quản Lý Plugin)** | ⭐⭐        | 1 tuần    | -          |

---

## Tổng Quan Chủ Đề

### **1. Kiến Trúc Jenkins** (`01-fundamentals/`)

- Master/Agent model (mô hình chủ-tác nhân): phân chia điều phối và thực thi
- Executor (bộ thực thi) — số lượng build chạy song song trên mỗi node
- Build Queue (hàng đợi build) — quản lý thứ tự và ưu tiên job
- Workspace (không gian làm việc) — thư mục chứa source code khi build
- Jenkins Home Directory — cấu trúc thư mục lưu config, job, plugin

### **2. Pipeline: Declarative & Scripted** (`02-pipeline/`)

- **Declarative Pipeline** — cú pháp cấu trúc rõ ràng, khuyến nghị cho người mới
  - `pipeline`, `agent`, `stages`, `steps`, `post` directives
- **Scripted Pipeline** — Groovy DSL (Domain-Specific Language), linh hoạt hơn
  - `node`, `stage`, `sh`, `bat`, `try-catch` blocks
- **Jenkinsfile** — pipeline dưới dạng file lưu trong repository
- **Multibranch Pipeline** — tự động phát hiện nhánh và tạo pipeline tương ứng
- **Parallel Stages** (stage song song) — tăng tốc build bằng chạy song song

### **3. Plugin: Quản Lý và Sử Dụng** (`03-plugins/`)

- Plugin Manager (quản lý plugin): cài đặt, cập nhật, gỡ bỏ an toàn
- Plugin thiết yếu: Git, Pipeline, Blue Ocean, Credentials, Mailer
- Plugin tích hợp: Docker Pipeline, Kubernetes, Slack Notification, SonarQube Scanner
- Dependency management (quản lý phụ thuộc) — tránh xung đột phiên bản

### **4. Build Triggers và Quản Lý Build** (`04-builds-triggers/`)

- **Webhook** (hook sự kiện): GitHub/GitLab webhook kích hoạt build tức thì
- **Poll SCM** (kiểm tra định kỳ SCM): Jenkins tự kiểm tra thay đổi theo lịch
- **Cron Schedule** (lịch cron): `H/15 * * * *` — cú pháp và các trường hợp dùng
- **Build Parameters** (tham số build): String, Choice, Boolean, File
- **Build Artifacts** (kết quả build): Archive, Stash/Unstash, Fingerprint

### **5. Distributed Builds: Build Phân Tán** (`05-distributed-builds/`)

- **Agent Node** (nút tác nhân): kết nối qua JNLP (Java Network Launch Protocol) hoặc SSH
- **Node Labels** (nhãn node): gán nhãn để chọn agent phù hợp cho từng job
- **Docker Agents** — chạy build trong Docker container tạm thời
- **Kubernetes Agents** (agent động trên K8s): Pod Template, JCasC (Jenkins Configuration as Code)
- **Executors** (bộ thực thi): cấu hình số lượng, quản lý tài nguyên

### **6. Security: Bảo Mật Jenkins** (`06-security/`)

- **Authentication** (xác thực): Jenkins DB, LDAP, Active Directory, OAuth2
- **Authorization** (phân quyền): Matrix-based Security, Role Strategy Plugin
- **Credentials** (thông tin xác thực): Secret text, Username/Password, SSH Key, Certificate
- **CSP — Content Security Policy**: cấu hình HTTP headers bảo vệ giao diện
- Security hardening: tắt CLI qua HTTP, vô hiệu hóa Groovy sandbox bypass

### **7. Integration: Tích Hợp Công Cụ** (`07-integration/`)

- **Git Integration**: GitHub, GitLab, Bitbucket — webhook, SSH deploy key
- **Docker Integration**: build image, push lên registry, Docker-in-Docker vs Docker socket
- **Kubernetes Deployment**: kubectl, Helm deploy từ Jenkins pipeline
- **Notifications** (thông báo): Slack, Email, Microsoft Teams, Telegram
- **SonarQube Integration**: Quality Gate (cổng chất lượng), code coverage

### **8. Shared Libraries: Thư Viện Dùng Chung** (`08-shared-libraries/`)

- Cấu trúc thư viện: `vars/` (biến toàn cục), `src/` (Groovy classes), `resources/`
- **Global Variables** (biến toàn cục): định nghĩa trong `vars/`, gọi như function
- Versioning (quản lý phiên bản): tag, branch cho thư viện
- Testing shared libraries với `JenkinsPipelineUnit`
- Best practices: đặt tên rõ ràng, tài liệu hóa, kiểm thử trước khi dùng production

### **9. Monitoring & Maintenance: Giám Sát và Bảo Trì** (`09-monitoring-maintenance/`)

- **Jenkins Metrics**: prometheus plugin, số lượng build, thời gian build, queue length
- **Disk Management** (quản lý ổ đĩa): Build Discard Policy, Workspace Cleanup Plugin
- **Backup & Restore**: ThinBackup Plugin, backup `JENKINS_HOME`, job configs
- **Upgrade Guide** (hướng dẫn nâng cấp): LTS (Long-Term Support) vs Weekly release, kiểm tra plugin compatibility

### **10. Troubleshooting: Xử Lý Sự Cố** (`10-troubleshooting/`)

- **Build Failures** (lỗi build): phân tích log, exit code, environment issues
- **Agent Connectivity** (kết nối agent): JNLP timeout, SSH key, firewall
- **Performance Issues** (hiệu suất): JVM heap, GC pressure, thread dump
- **Plugin Conflicts** (xung đột plugin): classloader hell, dependency version mismatch
- **Production Checklist** (danh sách kiểm tra): trước khi triển khai Jenkins lên môi trường thực

### **11. Interview Prep: Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 20 câu hỏi phỏng vấn Jenkins thường gặp
- Câu chuyện STAR về CI/CD incidents
- Bài toán thiết kế hệ thống CI/CD pipeline

---

## Ma Trận Kỹ Năng

### Beginner (0–1 năm kinh nghiệm)

- [ ] Hiểu được sự khác biệt giữa Declarative và Scripted Pipeline
- [ ] Tạo Jenkinsfile cơ bản với các stage: Build, Test, Deploy
- [ ] Cài đặt và quản lý plugin thông qua Plugin Manager
- [ ] Cấu hình Git integration (tích hợp với kho mã nguồn)
- [ ] Dùng Credentials để lưu Secret an toàn

### Intermediate (1–3 năm kinh nghiệm)

- [ ] Thiết kế Multibranch Pipeline với PR validation
- [ ] Cấu hình Docker Agent và Kubernetes Agent
- [ ] Viết và tái sử dụng Shared Libraries
- [ ] Thiết lập Webhook trigger từ GitHub/GitLab
- [ ] Cấu hình Role-based Access Control (RBAC — Kiểm Soát Truy Cập Dựa Trên Vai Trò)
- [ ] Tích hợp SonarQube Quality Gate vào pipeline

### Advanced (3–5+ năm kinh nghiệm)

- [ ] Tối ưu hoá hiệu suất Jenkins: JVM tuning, executor management
- [ ] Thiết kế Jenkins High Availability (tính sẵn sàng cao)
- [ ] Xây dựng Shared Library ecosystem cho toàn tổ chức
- [ ] Migration Jenkins sang Kubernetes-native setup
- [ ] Incident command và post-mortem (phân tích sau sự cố) cho CI/CD failures
- [ ] Implement Jenkins Configuration as Code (JCasC)

---

## Chuẩn Bị Phỏng Vấn

### Câu Hỏi Theo Chủ Đề

#### Kiến Trúc và Pipeline

- [ ] Giải thích sự khác biệt giữa Declarative Pipeline và Scripted Pipeline
- [ ] Master/Agent model hoạt động như thế nào?
- [ ] Khi nào nên dùng Parallel Stages (stage song song)?
- [ ] Jenkinsfile nên đặt ở đâu và quản lý như thế nào?

#### Distributed Builds và Agents

- [ ] Cách cấu hình Dynamic Agent trên Kubernetes
- [ ] Sự khác biệt giữa JNLP Agent và SSH Agent
- [ ] Làm thế nào để đảm bảo build isolation (cô lập build)?

#### Security (Bảo Mật)

- [ ] Cách quản lý Credentials an toàn trong Jenkins Pipeline
- [ ] Giải thích Matrix-based Security vs Role Strategy
- [ ] Những best practices bảo mật Jenkins trong môi trường production

#### Troubleshooting (Xử Lý Sự Cố Thực Tế)

- [ ] Kể về một lần bạn debug một pipeline phức tạp (STAR format)
- [ ] Xử lý Agent mất kết nối giữa chừng build như thế nào?
- [ ] Khi Jenkins chạy chậm dần theo thời gian, bạn xử lý thế nào?

Xem `11-interview-prep/INTERVIEW_GUIDE.md` để có bộ câu hỏi đầy đủ với gợi ý trả lời.

---

## Hướng Dẫn Sử Dụng Tài Liệu Này

### Cho Người Tự Học

1. Bắt đầu từ [Lộ Trình Học](#lộ-trình-học)
2. Học tuần tự từng giai đoạn
3. Thực hành mỗi chủ đề trên môi trường lab
4. Xây dựng pipeline thực tế cho một dự án cá nhân

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào `11-interview-prep/INTERVIEW_GUIDE.md`
2. Ôn kỹ Pipeline (Declarative, Shared Libraries)
3. Chuẩn bị 2–3 câu chuyện STAR về CI/CD incidents
4. Luyện giải thích kiến trúc bằng lời nói rõ ràng

### Cho Công Việc Hàng Ngày

- Sự cố build: xem `10-troubleshooting/`
- Cần tích hợp mới: xem `07-integration/`
- Cần chia sẻ code pipeline: xem `08-shared-libraries/`
- Cần cấp quyền cho team member: xem `06-security/authorization.md`

---

## Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ một lần
├─ 2️⃣  Chọn lộ trình (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Dựng môi trường lab với Docker (Jenkins + agent)
├─ 5️⃣  Viết Jenkinsfile đầu tiên cho một project thực tế
├─ 6️⃣  Thực hành từng chủ đề theo thứ tự
└─ 7️⃣  Ôn phỏng vấn bằng 11-interview-prep/
```

---

## Tài Liệu Tham Khảo

### Đọc Thêm

- **"Jenkins: The Definitive Guide"** by John Ferguson Smart — nền tảng vững chắc
- **"Continuous Delivery"** by Jez Humble & David Farley — nguyên lý CI/CD
- **"The Phoenix Project"** — tư duy DevOps và CI/CD trong tổ chức

### Tài Liệu Chính Thức

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Jenkins Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)
- [Jenkins Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)

### Blog và Cộng Đồng

- Jenkins Community Blog
- CloudBees DevOps Resource Center
- r/devops (Reddit)
- DevOps StackExchange

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
**Maintainer:** Backend Interview Prep
