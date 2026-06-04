# Git Integration — Tích Hợp Jenkins với GitHub, GitLab, Bitbucket

> Git Integration (tích hợp Git) là bước đầu tiên và quan trọng nhất để Jenkins có thể tự động lấy mã nguồn, phát hiện thay đổi và kích hoạt pipeline đúng thời điểm.

---

## Mục Lục

1. [Tổng Quan Kiến Trúc](#1-tổng-quan-kiến-trúc)
2. [GitHub Integration](#2-github-integration)
3. [GitLab Integration](#3-gitlab-integration)
4. [Bitbucket Integration](#4-bitbucket-integration)
5. [SSH Key Setup](#5-ssh-key-setup)
6. [Webhook Bảo Mật](#6-webhook-bảo-mật)
7. [Multibranch Pipeline với Git](#7-multibranch-pipeline-với-git)
8. [Best Practices](#8-best-practices)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Kiến Trúc

Jenkins có thể nhận thông báo từ Git platform theo hai cơ chế:

```
Cơ Chế 1: Webhook (Push-based — Dựa trên Sự Kiện)
─────────────────────────────────────────────────
Developer push code
        │
        ▼
 Git Platform (GitHub/GitLab/Bitbucket)
        │
        │  HTTP POST /github-webhook/
        ▼
 Jenkins (nhận sự kiện → trigger build ngay lập tức)


Cơ Chế 2: Poll SCM (Pull-based — Kiểm Tra Định Kỳ)
──────────────────────────────────────────────────
Jenkins (theo lịch cron, ví dụ: mỗi 5 phút)
        │
        │  git fetch → kiểm tra có commit mới không?
        ▼
 Git Platform
        │
        │  có thay đổi → trigger build
        ▼
 Jenkins pipeline chạy
```

| Tiêu Chí | Webhook | Poll SCM |
|----------|---------|----------|
| Độ trễ | Gần như tức thì (< 1 giây) | Tùy lịch (1–5 phút) |
| Tải lên server | Thấp (chỉ khi có sự kiện) | Cao (liên tục query) |
| Cấu hình | Phức tạp hơn (cần mở port) | Đơn giản hơn |
| Phù hợp | Production, team đông | Private network, firewall |

---

## 2. GitHub Integration

### 2.1 Cài Plugin

```
Manage Jenkins → Plugin Manager → Available plugins
→ Cài: "GitHub Plugin" và "GitHub Branch Source Plugin"
```

### 2.2 Personal Access Token (PAT — Token Truy Cập Cá Nhân)

**Bước 1: Tạo PAT trên GitHub**

```
GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
→ Generate new token
→ Chọn scopes:
   ✅ repo          (truy cập repository private)
   ✅ admin:repo_hook  (quản lý webhook)
   ✅ read:org      (đọc thông tin tổ chức — nếu dùng GitHub Org)
```

**Bước 2: Lưu PAT vào Jenkins Credentials**

```
Manage Jenkins → Credentials → System → Global credentials
→ Add Credentials:
   Kind: Secret text
   Secret: <dán PAT vào đây>
   ID: github-pat
   Description: GitHub Personal Access Token
```

**Bước 3: Cấu hình GitHub Server trong Jenkins**

```
Manage Jenkins → Configure System → GitHub
→ Add GitHub Server:
   Name: GitHub
   API URL: https://api.github.com
   Credentials: github-pat (chọn từ dropdown)
→ Test connection → "Credentials verified"
```

### 2.3 Cấu Hình Webhook Trên GitHub

```
Repository → Settings → Webhooks → Add webhook
  Payload URL:    http://<jenkins-url>/github-webhook/
  Content type:   application/json
  Secret:         <chuỗi bí mật ngẫu nhiên, lưu lại để dùng ở Jenkins>
  Which events?   ✅ Just the push event
                  ✅ Pull requests (nếu muốn trigger khi có PR)
→ Add webhook
```

**Cấu hình Jenkins nhận webhook:**

```
Job → Configure → Build Triggers
→ ✅ GitHub hook trigger for GITScm polling
```

### 2.4 Jenkinsfile Mẫu — GitHub Integration

```groovy
pipeline {
    agent any

    triggers {
        // githubPush() kết hợp với webhook để trigger tức thì
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                // checkout scm — tự động dùng cấu hình SCM của job
                checkout scm
                sh 'git log --oneline -5'
            }
        }

        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
    }

    post {
        always {
            // Cập nhật trạng thái commit trên GitHub
            step([$class: 'GitHubCommitStatusSetter',
                reposSource: [$class: 'ManuallyEnteredRepositorySource',
                              url: env.GIT_URL],
                statusResultSource: [$class: 'ConditionalStatusResultSource',
                                     results: [[$class: 'AnyBuildResult',
                                                message: 'Build completed',
                                                state: currentBuild.result ?: 'SUCCESS']]]
            ])
        }
    }
}
```

### 2.5 GitHub App (Khuyến Nghị Cho Enterprise)

GitHub App là phương thức xác thực hiện đại hơn PAT, được khuyến nghị cho môi trường tổ chức:

```
Ưu điểm của GitHub App so với PAT:
- Fine-grained permissions (phân quyền chi tiết hơn)
- Không gắn với tài khoản cá nhân (nếu người dùng nghỉ việc, PAT bị thu hồi)
- Rate limit (giới hạn tần suất) cao hơn
- Audit log (nhật ký kiểm toán) rõ ràng hơn

Cài đặt:
1. GitHub → Settings → Developer settings → GitHub Apps → New GitHub App
2. Điền thông tin, set Webhook URL: http://<jenkins>/github-webhook/
3. Permissions: Contents (Read), Pull requests (Read & Write), Commit statuses (Read & Write)
4. Tạo Private Key → tải file .pem
5. Cài plugin "GitHub App" trên Jenkins
6. Thêm Credentials loại "GitHub App"
```

---

## 3. GitLab Integration

### 3.1 Cài Plugin

```
Plugin Manager → Cài: "GitLab Plugin"
```

### 3.2 GitLab Personal Access Token

```
GitLab → User Settings → Access Tokens
→ Name: jenkins-integration
→ Scopes:
   ✅ api         (toàn quyền API)
   ✅ read_repository
→ Create personal access token → Copy token
```

**Lưu vào Jenkins:**

```
Credentials → Add:
  Kind: GitLab API token
  API token: <paste token>
  ID: gitlab-api-token
```

### 3.3 Cấu Hình GitLab Connection

```
Manage Jenkins → Configure System → GitLab
  Connection name: GitLab
  GitLab host URL: https://gitlab.com  (hoặc URL self-hosted)
  Credentials: gitlab-api-token
→ Test Connection → "Success"
```

### 3.4 Webhook Trên GitLab

```
Repository → Settings → Webhooks
  URL:          http://<jenkins-url>/project/<job-name>
  Secret token: <chuỗi bí mật>
  Trigger:
    ✅ Push events
    ✅ Merge request events
    ✅ Tag push events
→ Add webhook
```

### 3.5 Jenkinsfile Mẫu — GitLab Integration

```groovy
pipeline {
    agent any

    triggers {
        gitlab(
            triggerOnPush: true,
            triggerOnMergeRequest: true,
            branchFilterType: 'All'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: env.gitlabSourceBranch ?: '*/main']],
                    userRemoteConfigs: [[
                        url: env.gitlabSourceRepoHttpUrl,
                        credentialsId: 'gitlab-deploy-key'
                    ]]
                ])
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
        }
    }

    post {
        success {
            // Cập nhật Merge Request status trên GitLab
            updateGitlabCommitStatus name: 'jenkins', state: 'success'
        }
        failure {
            updateGitlabCommitStatus name: 'jenkins', state: 'failed'
        }
    }
}
```

---

## 4. Bitbucket Integration

### 4.1 Cài Plugin

```
Plugin Manager → Cài: "Bitbucket Plugin" và "Bitbucket Branch Source Plugin"
```

### 4.2 App Password (Mật Khẩu Ứng Dụng) Cho Bitbucket Cloud

```
Bitbucket → Personal settings → App passwords → Create app password
  Label: jenkins
  Permissions:
    ✅ Repositories: Read, Write
    ✅ Pull requests: Read
    ✅ Webhooks: Read, Write
→ Copy password (chỉ hiển thị một lần)
```

**Lưu vào Jenkins:**

```
Credentials → Add:
  Kind: Username with password
  Username: <bitbucket-username>
  Password: <app-password>
  ID: bitbucket-cred
```

### 4.3 Webhook Trên Bitbucket

```
Repository → Repository settings → Webhooks → Add webhook
  Title:   Jenkins Trigger
  URL:     http://<jenkins-url>/bitbucket-hook/
  Triggers:
    ✅ Repository push
    ✅ Pull Request: Created, Updated, Merged
→ Save
```

---

## 5. SSH Key Setup

### 5.1 Tại Sao Dùng SSH Key?

SSH Key là phương thức xác thực được khuyến nghị cho Git operations vì:
- Không cần nhập password mỗi lần
- An toàn hơn PAT (Private key không bao giờ truyền qua mạng)
- Dễ revoke (thu hồi) mà không ảnh hưởng tài khoản người dùng

### 5.2 Tạo SSH Key Pair (Cặp Khóa SSH)

```bash
# Tạo cặp khóa RSA 4096-bit dành riêng cho Jenkins
ssh-keygen -t ed25519 -C "jenkins@mycompany.com" -f ~/.ssh/jenkins_deploy_key

# Kết quả:
# ~/.ssh/jenkins_deploy_key     → Private key (giữ bí mật, upload lên Jenkins)
# ~/.ssh/jenkins_deploy_key.pub → Public key (upload lên Git platform)
```

### 5.3 Thêm Public Key Vào Git Platform

```
GitHub:
  Repository → Settings → Deploy keys → Add deploy key
  Title: Jenkins CI
  Key: <nội dung file jenkins_deploy_key.pub>
  ✅ Allow write access (chỉ nếu Jenkins cần push — ví dụ: tag release)

GitLab:
  Repository → Settings → Repository → Deploy keys → Add key

Bitbucket:
  Repository → Repository settings → Access keys → Add key
```

### 5.4 Lưu Private Key Vào Jenkins

```
Credentials → Add:
  Kind: SSH Username with private key
  ID: git-ssh-key
  Username: git           (mặc định cho GitHub/GitLab)
  Private Key: Enter directly → paste nội dung private key
  Passphrase: <nếu key có passphrase>
```

### 5.5 Dùng SSH Key Trong Pipeline

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // Dùng SSH key để clone private repository
                git(
                    url: 'git@github.com:myorg/myrepo.git',
                    credentialsId: 'git-ssh-key',
                    branch: 'main'
                )
            }
        }

        stage('Tag Release') {
            steps {
                // Push tag về Git — cần write access
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'git-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        export GIT_SSH_COMMAND="ssh -i $SSH_KEY -o StrictHostKeyChecking=no"
                        git tag -a v${BUILD_NUMBER} -m "Release v${BUILD_NUMBER}"
                        git push origin v${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
}
```

---

## 6. Webhook Bảo Mật

### 6.1 Tại Sao Cần Bảo Mật Webhook?

Nếu không có bảo mật, bất kỳ ai biết URL webhook của Jenkins đều có thể kích hoạt build — nguy cơ:
- Tốn tài nguyên (denial of service)
- Trigger build giả mạo với code độc hại

### 6.2 Webhook Secret Token

**Phía Git Platform (GitHub):**

```
Webhook settings:
  Secret: <chuỗi ngẫu nhiên mạnh, ví dụ: openssl rand -hex 32>
```

**Phía Jenkins — xác minh HMAC signature (chữ ký HMAC):**

GitHub tự động thêm header `X-Hub-Signature-256` vào mỗi request. Jenkins GitHub Plugin tự xác minh signature này.

```groovy
// Trong Jenkins job configuration:
// Build Triggers → GitHub hook trigger for GITScm polling
// Jenkins tự xác thực HMAC-SHA256 signature
```

### 6.3 IP Allowlisting (Danh Sách Trắng IP)

Hạn chế chỉ cho phép IP của Git platform gọi webhook:

```
GitHub IP ranges: https://api.github.com/meta → hooks
GitLab.com:       34.74.90.64/28, 34.74.226.0/24 (thay đổi theo thời gian)

Cấu hình firewall/nginx:
  allow 192.30.252.0/22;   # GitHub IP range
  deny all;
```

### 6.4 Jenkins Reverse Proxy Bảo Mật

```nginx
# nginx config — chỉ cho phép /github-webhook/ từ GitHub IPs
location /github-webhook/ {
    allow 192.30.252.0/22;
    allow 185.199.108.0/22;
    allow 140.82.112.0/20;
    deny all;
    proxy_pass http://jenkins:8080;
}
```

---

## 7. Multibranch Pipeline với Git

### 7.1 Cấu Hình Multibranch Pipeline

```
New Item → Multibranch Pipeline
  Branch Sources → Add source → GitHub / GitLab / Bitbucket
    Credentials: git-ssh-key
    Repository: myorg/myrepo
  Behaviors:
    ✅ Discover branches (phát hiện nhánh)
    ✅ Discover pull requests from origin
    ✅ Filter by name (wildcard):
         Include: main release/* feature/*
         Exclude: wip/* draft/*
  Scan Multibranch Pipeline Triggers:
    Periodically if not otherwise run: 1 hour
```

### 7.2 Jenkinsfile Cho Multibranch

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                // env.BRANCH_NAME — tên nhánh hiện tại
                // env.CHANGE_ID   — PR number nếu là PR build
                echo "Branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy Staging') {
            // Chỉ deploy staging khi build trên nhánh develop
            when {
                branch 'develop'
            }
            steps {
                sh './deploy.sh staging'
            }
        }

        stage('Deploy Production') {
            // Chỉ deploy production khi build trên nhánh main
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh './deploy.sh production'
            }
        }
    }
}
```

---

## 8. Best Practices

### Quản Lý Credentials

```
✅ Dùng Deploy Key riêng biệt cho từng repo — không dùng chung một key
✅ Đặt tên Credentials ID có ý nghĩa: github-myrepo-deploy-key
✅ Rotate (xoay vòng) credentials định kỳ — 3–6 tháng một lần
✅ Dùng GitHub App thay vì PAT cho tổ chức lớn
❌ Không bao giờ hardcode token/password trong Jenkinsfile
❌ Không dùng tài khoản cá nhân cho CI — tạo service account riêng
```

### Webhook Reliability (Độ Tin Cậy Webhook)

```
✅ Cấu hình cả Webhook (primary) và Poll SCM (fallback — dự phòng)
✅ Monitor webhook delivery failures trong Git platform settings
✅ Đặt timeout hợp lý: Jenkins phải phản hồi trong 10 giây
✅ Dùng HTTPS cho webhook URL — không dùng HTTP plain text
```

### Git Checkout Optimization (Tối Ưu Lấy Mã Nguồn)

```groovy
// Shallow clone — chỉ lấy N commit gần nhất, giảm thời gian checkout
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    extensions: [
        // Shallow clone với depth = 1 — nhanh hơn nhiều với repo lớn
        [$class: 'CloneOption', depth: 1, shallow: true, noTags: true],
        // Xóa workspace trước khi checkout — đảm bảo môi trường sạch
        [$class: 'CleanBeforeCheckout'],
        // Timeout checkout sau 10 phút
        [$class: 'CheckoutOption', timeout: 10]
    ],
    userRemoteConfigs: [[
        url: 'git@github.com:myorg/myrepo.git',
        credentialsId: 'git-ssh-key'
    ]]
])
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa Webhook và Poll SCM?**

> Webhook là cơ chế push-based — Git platform chủ động gửi thông báo tới Jenkins ngay khi có sự kiện (push, PR), độ trễ gần như 0. Poll SCM là cơ chế pull-based — Jenkins định kỳ hỏi Git server xem có thay đổi không, tốn tài nguyên và có độ trễ theo chu kỳ. Webhook luôn được ưu tiên cho production vì hiệu quả hơn. Dùng Poll SCM khi Jenkins nằm trong mạng nội bộ và Git server không thể kết nối ra ngoài.

**Q: Tại sao nên dùng Deploy Key thay vì tài khoản người dùng?**

> Deploy Key được gắn trực tiếp với repository, không gắn với tài khoản cá nhân. Nếu nhân viên nghỉ việc và tài khoản bị xóa, mọi pipeline dùng SSH key của họ sẽ bị lỗi. Deploy Key cho phép revoke quyền truy cập mà không ảnh hưởng đến người dùng khác và tuân theo nguyên tắc least privilege (quyền tối thiểu).

**Q: Làm thế nào để bảo mật webhook endpoint của Jenkins?**

> Có ba lớp bảo mật: (1) Webhook Secret — Git platform ký HMAC-SHA256 lên payload, Jenkins xác minh chữ ký; (2) IP Allowlisting — chỉ cho phép IP của Git platform gọi webhook endpoint; (3) HTTPS — mã hóa toàn bộ lưu lượng. Kết hợp cả ba lớp để đảm bảo an toàn.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
**Chủ Đề Tiếp Theo:** [2-docker-integration.md](2-docker-integration.md)
