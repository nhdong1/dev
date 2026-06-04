# Plugin Thiết Yếu Cho Jenkins

> Danh sách plugin bắt buộc phải cài khi dựng Jenkins từ đầu — đây là nền tảng cho mọi CI/CD pipeline thực tế.

## Mục Lục

1. [Nhóm Pipeline và Build](#nhóm-pipeline-và-build)
2. [Nhóm SCM — Kết Nối Kho Mã Nguồn](#nhóm-scm--kết-nối-kho-mã-nguồn)
3. [Nhóm Credentials và Bảo Mật](#nhóm-credentials-và-bảo-mật)
4. [Nhóm Giao Diện và Trải Nghiệm](#nhóm-giao-diện-và-trải-nghiệm)
5. [Nhóm Thông Báo](#nhóm-thông-báo)
6. [Checklist Cài Đặt Ban Đầu](#checklist-cài-đặt-ban-đầu)

---

## Nhóm Pipeline và Build

### Pipeline Plugin

**ID:** `workflow-aggregator`
**Phụ Thuộc Bao Gồm:** workflow-job, workflow-cps, workflow-basic-steps, pipeline-stage-view

Plugin lõi để chạy Declarative Pipeline (pipeline khai báo) và Scripted Pipeline (pipeline kịch bản). **Bắt buộc với mọi cài đặt Jenkins hiện đại.**

Kích hoạt:
- Cú pháp `pipeline { }` và `node { }`
- Stage (giai đoạn) và parallel execution (thực thi song song)
- `withCredentials`, `withEnv` blocks

```groovy
// Ví dụ cơ bản khi Pipeline Plugin được cài
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

---

### Pipeline: Stage View Plugin

**ID:** `pipeline-stage-view`

Hiển thị lịch sử các stage theo dạng bảng lưới — mỗi build là một cột, mỗi stage là một hàng. Giao diện classic Jenkins trước Blue Ocean.

```
Build #1  Build #2  Build #3
 ✅ Checkout  ✅  ✅
 ✅ Build     ✅  ❌
 ✅ Test      ✅  -
 ✅ Deploy    ✅  -
```

---

### Build Timeout Plugin

**ID:** `build-timeout`

Tự động hủy build nếu chạy quá lâu — tránh treo agent và tốn tài nguyên.

```groovy
options {
    timeout(time: 30, unit: 'MINUTES')
}
```

---

### AnsiColor Plugin

**ID:** `ansicolor`

Hiển thị màu ANSI trong console log — giúp đọc log dễ hơn với màu đỏ (lỗi), xanh (thành công).

```groovy
options {
    ansiColor('xterm')
}
steps {
    sh 'echo "\033[32mBuild thành công!\033[0m"'
}
```

---

### Timestamper Plugin

**ID:** `timestamper`

Thêm timestamp (dấu thời gian) vào mỗi dòng console log.

```groovy
options {
    timestamps()
}
```

Output:
```
10:23:45 [Build] Running maven...
10:24:12 [Build] BUILD SUCCESS
```

---

## Nhóm SCM — Kết Nối Kho Mã Nguồn

### Git Plugin

**ID:** `git`
**Phụ Thuộc:** git-client, scm-api

Plugin quan trọng nhất để checkout (tải xuống) code từ Git repository. **Bắt buộc cho mọi pipeline dùng Git.**

Tính năng:
- Checkout từ GitHub, GitLab, Bitbucket, Gitea
- Hỗ trợ SSH key và HTTPS credentials
- Shallow clone (clone nông — chỉ lấy commit gần nhất)
- Checkout theo tag, branch, commit hash

```groovy
// Dùng trong Declarative Pipeline
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-ssh-key',
                    url: 'git@github.com:org/repo.git'
            }
        }
    }
}
```

```groovy
// Dùng checkout step đầy đủ hơn
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    extensions: [
        [$class: 'CloneOption', shallow: true, depth: 1]
    ],
    userRemoteConfigs: [[
        credentialsId: 'github-ssh-key',
        url: 'git@github.com:org/repo.git'
    ]]
])
```

---

### GitHub Plugin

**ID:** `github`

Tích hợp sâu hơn với GitHub — không chỉ checkout mà còn:
- Nhận GitHub Webhook (hook sự kiện) để trigger build tức thì
- Cập nhật build status (trạng thái build) lên GitHub commit
- Hiển thị link Jenkins build trên GitHub PR

**Khác với Git Plugin:** Git Plugin chỉ checkout code; GitHub Plugin tương tác với GitHub API.

```groovy
// Sau khi cài, thêm vào Jenkinsfile
post {
    success {
        githubNotify status: 'SUCCESS', description: 'Build passed'
    }
    failure {
        githubNotify status: 'FAILURE', description: 'Build failed'
    }
}
```

---

### GitLab Plugin

**ID:** `gitlab-plugin`

Tương tự GitHub Plugin nhưng dành cho GitLab — tích hợp GitLab webhook, cập nhật merge request status.

---

## Nhóm Credentials và Bảo Mật

### Credentials Plugin

**ID:** `credentials`

Cho phép lưu trữ secret (bí mật) một cách an toàn, mã hóa tại rest (lúc lưu) trong Jenkins:

| Loại Credential         | Dùng Cho                                     |
| ----------------------- | -------------------------------------------- |
| Secret text             | API token, password đơn giản                 |
| Username with password  | Docker registry, npm registry login          |
| SSH Username with key   | Git SSH access, server login                 |
| Certificate             | Client certificate (chứng chỉ khách hàng)   |
| Secret file             | Kubeconfig, service account key              |

```
Manage Jenkins → Credentials → System → Global credentials → Add credentials
```

---

### Credentials Binding Plugin

**ID:** `credentials-binding`

Cho phép dùng credential trong Jenkinsfile thông qua `withCredentials` block — secret không bao giờ xuất hiện trong log.

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'docker-hub',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    )
]) {
    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
}
```

```groovy
// Dùng SSH key
withCredentials([sshUserPrivateKey(
    credentialsId: 'server-ssh-key',
    keyFileVariable: 'SSH_KEY'
)]) {
    sh 'ssh -i $SSH_KEY user@server.com "deploy.sh"'
}
```

**Bảo mật:** Jenkins tự động che giấu (mask) giá trị secret trong log — nếu vô tình in ra sẽ hiện `****`.

---

### SSH Build Agents Plugin

**ID:** `ssh-slaves`

Cho phép Jenkins Master kết nối với Agent (tác nhân build) qua SSH.

```
Manage Jenkins → Nodes → New Node → Permanent Agent
→ Launch method: Launch agents via SSH
→ Host: 192.168.1.100
→ Credentials: [SSH key credential]
```

---

## Nhóm Giao Diện và Trải Nghiệm

### Blue Ocean Plugin

**ID:** `blueocean`

Giao diện người dùng hiện đại cho Jenkins — thay thế giao diện classic (cổ điển).

**Điểm nổi bật:**
- Visualize pipeline (trực quan hóa pipeline) dưới dạng flowchart (sơ đồ luồng)
- Dễ tạo pipeline từ GitHub/GitLab qua wizard (trình hướng dẫn)
- Log per-step — không cần cuộn qua toàn bộ console
- Hiển thị rõ stage nào thất bại, tại sao

```
Truy cập: http://jenkins-server:8080/blue
```

**Lưu ý 2024–2025:** Blue Ocean không còn được phát triển tích cực bởi CloudBees. Jenkins community đang xây dựng giao diện mới. Vẫn hoạt động tốt nhưng không có tính năng mới.

---

### Job DSL Plugin

**ID:** `job-dsl`

Tạo và quản lý Jenkins Jobs (công việc) bằng Groovy DSL (Ngôn Ngữ Đặc Thù Miền Groovy) — Pipeline as Code cho cấu hình job.

```groovy
// Ví dụ tạo Freestyle job bằng Job DSL
job('my-freestyle-job') {
    description('Build và test ứng dụng')
    scm {
        git('https://github.com/org/repo.git', 'main')
    }
    triggers {
        cron('H/15 * * * *')
    }
    steps {
        shell('mvn clean test')
    }
}
```

---

### Configuration as Code (JCasC) Plugin

**ID:** `configuration-as-code`
**Viết tắt:** JCasC — Jenkins Configuration as Code — Cấu Hình Jenkins Như Code

Cho phép toàn bộ cấu hình Jenkins nằm trong file YAML — idempotent (bất biến), có thể version control.

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins Production Server"
  numExecutors: 0
  agentProtocols:
    - "JNLP4-connect"
  securityRealm:
    ldap:
      configurations:
        - server: "ldap://ldap.company.com"
          rootDN: "dc=company,dc=com"
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
```

---

## Nhóm Thông Báo

### Mailer Plugin

**ID:** `mailer`

Gửi email khi build thất bại hoặc build phục hồi (trở lại từ thất bại → thành công).

```groovy
post {
    failure {
        mail to: 'team@company.com',
             subject: "Build Thất Bại: ${currentBuild.fullDisplayName}",
             body: "Build ${env.BUILD_URL} đã thất bại."
    }
}
```

---

### Email Extension Plugin

**ID:** `email-ext`

Phiên bản nâng cao của Mailer — tùy chỉnh template email HTML, trigger linh hoạt hơn.

```groovy
emailext (
    subject: "[${currentBuild.result}] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
    body: '''${SCRIPT, template="groovy-html.template"}''',
    to: "${params.NOTIFY_EMAIL}",
    attachLog: true
)
```

---

## Checklist Cài Đặt Ban Đầu

Khi dựng Jenkins mới, cài theo thứ tự này:

```
Lớp 1: Nền Tảng (Bắt Buộc)
├── workflow-aggregator       (Pipeline)
├── git                       (Git checkout)
├── credentials               (Lưu secret)
└── credentials-binding       (Dùng secret trong pipeline)

Lớp 2: SCM và Trigger
├── github                    (Nếu dùng GitHub)
├── gitlab-plugin             (Nếu dùng GitLab)
└── ssh-slaves                (Nếu dùng SSH Agent)

Lớp 3: Trải Nghiệm
├── pipeline-stage-view       (Xem stage history)
├── blueocean                 (Giao diện hiện đại)
├── timestamper               (Timestamp trong log)
└── ansicolor                 (Màu trong log)

Lớp 4: Thông Báo
├── mailer                    (Email cơ bản)
└── email-ext                 (Email nâng cao)

Lớp 5: Vận Hành
├── configuration-as-code     (JCasC — quản lý config)
└── build-timeout             (Tránh build treo)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa Git Plugin và GitHub Plugin?**
> Git Plugin cung cấp khả năng checkout code từ bất kỳ Git repository (local, remote). GitHub Plugin bổ sung tích hợp API GitHub: nhận webhook, cập nhật commit status, kết nối GitHub Organizations.

**Q: Tại sao nên dùng Credentials Binding thay vì environment variable thông thường?**
> Credentials Binding tự động mask (che giấu) giá trị trong log, và secret không bao giờ được lưu dưới dạng plain text trong Jenkinsfile. Environment variable thông thường dễ bị lộ qua `sh 'env'` hoặc log.

**Q: JCasC Plugin giải quyết vấn đề gì?**
> Không có JCasC, cấu hình Jenkins chỉ tồn tại dưới dạng XML trong `JENKINS_HOME` — khó version control, khó reproduce (tái tạo) khi disaster recovery. JCasC cho phép toàn bộ config nằm trong YAML, commit vào git, tái áp dụng bất cứ lúc nào.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
