# Webhooks — Kích Hoạt Build Qua Sự Kiện

> **Webhook** (móc sự kiện) là cơ chế HTTP callback: khi có sự kiện xảy ra trên GitHub/GitLab (push code, tạo pull request, merge branch...), nền tảng đó gửi ngay một HTTP POST request đến Jenkins để kích hoạt build. Đây là cách nhanh nhất và hiệu quả nhất để tự động hóa CI/CD.

---

## Mục Lục

1. [Kiến Trúc Webhook](#kiến-trúc-webhook)
2. [GitHub Webhook](#github-webhook)
3. [GitLab Webhook](#gitlab-webhook)
4. [Bảo Mật Webhook](#bảo-mật-webhook)
5. [Xử Lý Sự Kiện Nâng Cao](#xử-lý-sự-kiện-nâng-cao)
6. [Troubleshooting Webhook](#troubleshooting-webhook)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Webhook

### Luồng Hoạt Động

```
┌─────────────┐     Push/PR/Tag     ┌──────────────┐
│  Developer  │────────────────────►│  GitHub/     │
│             │                     │  GitLab      │
└─────────────┘                     └──────┬───────┘
                                           │ HTTP POST
                                           │ /github-webhook/
                                           ▼
                                    ┌──────────────┐
                                    │   Jenkins    │
                                    │  Controller  │
                                    └──────┬───────┘
                                           │ Trigger
                                           ▼
                                    ┌──────────────┐
                                    │  Build Job   │
                                    │  (Pipeline)  │
                                    └──────────────┘
```

### So Sánh Webhook vs Poll SCM

| Tiêu Chí | Webhook | Poll SCM |
|----------|---------|----------|
| **Độ trễ** | ~1–3 giây | 1–15 phút (theo lịch) |
| **Tài nguyên** | Rất thấp (chỉ tốn khi có sự kiện) | Trung bình (liên tục kiểm tra) |
| **Độ tin cậy** | Phụ thuộc vào kết nối mạng | Jenkins tự chủ động |
| **Yêu cầu mạng** | Jenkins phải accessible từ internet | Không cần |
| **Khuyến nghị** | ✅ Ưu tiên hàng đầu | ⚠️ Dùng khi không thể webhook |

> **Nguyên tắc:** Luôn dùng Webhook nếu Jenkins của bạn có thể nhận HTTP từ nền tảng Git. Poll SCM chỉ là phương án dự phòng khi Jenkins nằm sau firewall nghiêm ngặt.

---

## GitHub Webhook

### Cài Đặt GitHub Webhook

#### Bước 1 — Cài Plugin trên Jenkins

Đảm bảo plugin **GitHub Integration** đã được cài:

```
Manage Jenkins → Plugins → Available Plugins
→ Search: "GitHub Integration Plugin"
→ Install without restart
```

#### Bước 2 — Thêm GitHub Server trên Jenkins

```
Manage Jenkins → System → GitHub
→ Add GitHub Server
   Name:       github.com
   API URL:    https://api.github.com
   Credentials: (GitHub Personal Access Token)
```

#### Bước 3 — Tạo Webhook trên GitHub Repository

```
GitHub Repository → Settings → Webhooks → Add webhook

Payload URL:    http://your-jenkins.example.com/github-webhook/
Content type:   application/json
Secret:         <your-secret-token>           ← Quan trọng!

Events to trigger:
  ✅ Just the push event                      ← CI đơn giản
  - hoặc -
  ✅ Let me select individual events
     ✅ Pushes
     ✅ Pull requests
     ✅ Create (tags/branches)
```

> **Lưu ý:** URL Jenkins phải kết thúc bằng `/github-webhook/` (có dấu `/` cuối).

#### Bước 4 — Cấu Hình Jenkins Job

**Declarative Pipeline:**

```groovy
pipeline {
    agent any

    triggers {
        // Kích hoạt khi GitHub gửi webhook
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
    }
}
```

**Freestyle Job (Công việc truyền thống):**

```
Job Configuration → Build Triggers
→ ✅ GitHub hook trigger for GITScm polling
```

---

### Cấu Hình Theo Loại Sự Kiện

#### Trigger Chỉ Cho Tag (Phát Hành Release)

```groovy
pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {
        stage('Check Tag') {
            when {
                // Chỉ chạy khi là tag release (v1.0.0, v2.3.1...)
                tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
            }
            steps {
                sh './scripts/build-release.sh'
            }
        }
    }
}
```

#### Trigger Theo Nhánh Cụ Thể

```groovy
pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {
        stage('Production Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './scripts/deploy-production.sh'
            }
        }

        stage('Staging Deploy') {
            when {
                branch 'develop'
            }
            steps {
                sh './scripts/deploy-staging.sh'
            }
        }
    }
}
```

---

## GitLab Webhook

### Cài Đặt GitLab Webhook

#### Bước 1 — Cài Plugin GitLab

```
Manage Jenkins → Plugins → Available Plugins
→ Search: "GitLab Plugin"
→ Install without restart
```

#### Bước 2 — Tạo API Token trên GitLab

```
GitLab → User Settings → Access Tokens
→ Name: jenkins-webhook
→ Scopes: ✅ api
→ Create personal access token
→ Lưu token vào Jenkins Credentials
```

#### Bước 3 — Cấu Hình Jenkins

```
Manage Jenkins → System → GitLab
→ Connection name: gitlab.example.com
→ GitLab host URL: https://gitlab.example.com
→ Credentials: (GitLab API Token vừa tạo)
→ Test Connection → Should return "Success"
```

#### Bước 4 — Tạo Webhook trên GitLab Repository

```
GitLab Project → Settings → Webhooks → Add new webhook

URL:            http://your-jenkins.example.com/project/YOUR-JOB-NAME
Secret token:   <your-secret-token>

Trigger:
  ✅ Push events
  ✅ Merge request events
  ✅ Tag push events

SSL verification: ✅ Enable SSL verification (nếu Jenkins dùng HTTPS)
```

#### Bước 5 — Cấu Hình Jenkinsfile cho GitLab

```groovy
pipeline {
    agent any

    triggers {
        // Trigger từ GitLab webhook
        gitlab(
            triggerOnPush: true,
            triggerOnMergeRequest: true,
            triggerOpenMergeRequestOnPush: 'never',
            branchFilterType: 'All'
        )
    }

    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }

        stage('Test') {
            steps {
                sh 'make test'
            }
        }
    }

    post {
        success {
            // Cập nhật trạng thái build lên GitLab Merge Request
            updateGitlabCommitStatus name: 'jenkins', state: 'success'
        }
        failure {
            updateGitlabCommitStatus name: 'jenkins', state: 'failed'
        }
    }
}
```

---

## Bảo Mật Webhook

Webhook endpoint Jenkins (`/github-webhook/`) là HTTP endpoint công khai — bất kỳ ai cũng có thể gửi POST request giả mạo nếu không được bảo vệ.

### Secret Token — Xác Minh Chữ Ký

**Cơ chế hoạt động:**

```
1. Bạn đặt Secret Token trên cả GitHub và Jenkins
2. GitHub ký HTTP request bằng HMAC-SHA256 với secret đó
3. Jenkins kiểm tra chữ ký trong header X-Hub-Signature-256
4. Nếu chữ ký khớp → request hợp lệ; không khớp → từ chối
```

**Cấu hình trên Jenkins:**

```
Job Configuration → Build Triggers
→ GitHub hook trigger for GITScm polling
→ Advanced → Secret: <your-secret-token>
```

Hoặc lưu secret trong Jenkins Credentials:

```groovy
pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        // Không cần đặt secret trong code — Jenkins Plugin tự xử lý
        WEBHOOK_SECRET = credentials('github-webhook-secret')
    }
}
```

### Các Biện Pháp Bảo Mật Khác

#### 1 — IP Allowlist (Danh Sách IP Cho Phép)

GitHub công bố danh sách IP gửi webhook:

```bash
# Lấy danh sách IP GitHub webhook (Meta API)
curl https://api.github.com/meta | jq '.hooks'

# Ví dụ output:
# ["192.30.252.0/22", "185.199.108.0/22", "140.82.112.0/20"]
```

Cấu hình firewall/nginx chỉ chấp nhận từ các dải IP này:

```nginx
# nginx.conf — chỉ cho phép GitHub IP gửi webhook
location /github-webhook/ {
    allow 192.30.252.0/22;
    allow 185.199.108.0/22;
    allow 140.82.112.0/20;
    deny all;

    proxy_pass http://jenkins:8080;
}
```

#### 2 — HTTPS Bắt Buộc

Không bao giờ dùng HTTP thuần cho webhook endpoint trong production:

```nginx
server {
    listen 80;
    server_name jenkins.example.com;
    return 301 https://$host$request_uri;  # Chuyển hướng HTTP → HTTPS
}

server {
    listen 443 ssl;
    ssl_certificate     /etc/letsencrypt/live/jenkins.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.example.com/privkey.pem;

    location /github-webhook/ {
        proxy_pass http://jenkins:8080;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 3 — Rate Limiting (Giới Hạn Tần Suất)

Ngăn tấn công DoS (Denial of Service — Từ Chối Dịch Vụ) bằng giới hạn số request:

```nginx
# Giới hạn 10 request/phút từ mỗi IP
limit_req_zone $binary_remote_addr zone=webhook:10m rate=10r/m;

server {
    location /github-webhook/ {
        limit_req zone=webhook burst=20 nodelay;
        proxy_pass http://jenkins:8080;
    }
}
```

---

## Xử Lý Sự Kiện Nâng Cao

### Generic Webhook Trigger — Webhook Tùy Chỉnh

Plugin **Generic Webhook Trigger** cho phép nhận webhook từ bất kỳ hệ thống nào (không chỉ GitHub/GitLab):

```groovy
pipeline {
    agent any

    triggers {
        GenericTrigger(
            // Trích xuất giá trị từ JSON payload
            genericVariables: [
                [key: 'BRANCH_NAME', value: '$.ref'],
                [key: 'COMMIT_MESSAGE', value: '$.commits[0].message'],
                [key: 'PUSHER_NAME', value: '$.pusher.name']
            ],

            // Token xác thực (thêm vào URL: ?token=MY_SECRET_TOKEN)
            token: 'MY_SECRET_TOKEN',

            // Chỉ trigger khi branch là main hoặc develop
            regexpFilterText: '$BRANCH_NAME',
            regexpFilterExpression: 'refs/heads/(main|develop)'
        )
    }

    stages {
        stage('Info') {
            steps {
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Commit: ${env.COMMIT_MESSAGE}"
                echo "Pusher: ${env.PUSHER_NAME}"
            }
        }
    }
}
```

**URL gọi webhook:**

```bash
curl -X POST \
  "http://jenkins.example.com/generic-webhook-trigger/invoke?token=MY_SECRET_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ref": "refs/heads/main", "commits": [{"message": "Fix bug"}], "pusher": {"name": "alice"}}'
```

### Multibranch Pipeline với Webhook

Multibranch Pipeline — Pipeline Đa Nhánh — tự động phát hiện tất cả nhánh trong repository và tạo pipeline riêng cho từng nhánh:

```groovy
// Jenkinsfile tại gốc repository
// Tự động áp dụng cho TẤT CẢ nhánh

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy to Production') {
            // Chỉ deploy khi là nhánh main
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh production'
            }
        }
    }
}
```

**Cấu hình Multibranch Job nhận webhook:**

```
Multibranch Pipeline Job → Configuration
→ Scan Multibranch Pipeline Triggers
→ ✅ Periodically if not otherwise run: 1 day   ← Fallback scan
→ Build Configuration:
   Mode: by Jenkinsfile
```

GitHub/GitLab webhook gửi tới `/multibranch-webhook-trigger/invoke` sẽ kích hoạt scan và build tự động.

---

## Troubleshooting Webhook

### Kiểm Tra Webhook Delivery (Lịch Sử Gửi)

**Trên GitHub:**

```
GitHub Repository → Settings → Webhooks
→ Click vào webhook → Recent Deliveries

Mỗi lần gửi hiển thị:
- Status code (200 = thành công, 4xx/5xx = lỗi)
- Request headers và body
- Response từ Jenkins
→ "Redeliver" để gửi lại payload cũ
```

**Trên GitLab:**

```
Project → Settings → Webhooks → Edit
→ Recent events → View details
```

### Các Lỗi Thường Gặp

#### Lỗi 1: HTTP 302 hoặc Jenkins không nhận được webhook

**Nguyên nhân:** Jenkins đang redirect HTTP sang HTTPS nhưng GitHub không follow redirect.

**Khắc phục:**

```bash
# Kiểm tra Jenkins có chạy trên HTTPS không
curl -v http://jenkins.example.com/github-webhook/

# Nếu thấy "301 Moved Permanently", cấu hình GitHub webhook dùng HTTPS ngay
# Payload URL: https://jenkins.example.com/github-webhook/
```

#### Lỗi 2: HTTP 403 — CSRF Protection Blocked (Bảo Vệ CSRF Chặn)

**Nguyên nhân:** Jenkins CSRF protection (bảo vệ Cross-Site Request Forgery) chặn webhook.

**Khắc phục:**

```
Manage Jenkins → Security → CSRF Protection
→ ✅ Enable proxy compatibility
```

Hoặc thêm webhook URL vào exclusion list qua Groovy:

```groovy
// Trong Jenkins Script Console (Manage Jenkins → Script Console)
import jenkins.model.Jenkins
import hudson.security.csrf.DefaultCrumbIssuer

def jenkins = Jenkins.instance
jenkins.setCrumbIssuer(new DefaultCrumbIssuer(true))
jenkins.save()
```

#### Lỗi 3: Build Không Trigger Dù Webhook Gửi Thành Công (HTTP 200)

**Kiểm tra step-by-step:**

```bash
# 1. Xem Jenkins System Log
Manage Jenkins → System Log → All Jenkins Logs
# Tìm dòng: "Received POST for ..."

# 2. Kiểm tra Git plugin có nhận sự kiện không
Manage Jenkins → System Log → Add new log recorder
   Name: GitHub Hook Log
   Logger: com.cloudbees.jenkins.GitHubPushTrigger → Level: FINE

# 3. Kiểm tra job có cấu hình đúng Git URL không
# URL trong job phải KHỚP CHÍNH XÁC với URL repository trên GitHub
# Sai: https://github.com/user/repo
# Đúng: https://github.com/user/repo.git  ← hoặc ngược lại, tùy cấu hình
```

#### Lỗi 4: Signature Mismatch — Chữ Ký Không Khớp

```bash
# Kiểm tra header X-Hub-Signature-256 trong Jenkins log
# GitHub gửi: X-Hub-Signature-256: sha256=<hash>

# Nguyên nhân thường gặp:
# - Secret token trên GitHub và Jenkins khác nhau
# - Secret token có khoảng trắng thừa khi copy-paste
# - Jenkins lưu secret sai encoding

# Kiểm tra: tạo lại secret token và cập nhật cả hai bên
```

### Công Cụ Debug Webhook

```bash
# Dùng ngrok để expose Jenkins local ra internet (cho môi trường dev)
ngrok http 8080

# Ngrok cung cấp URL public: https://abc123.ngrok.io
# Dùng URL này làm Payload URL trong GitHub webhook
# Truy cập http://localhost:4040 để xem tất cả request webhook

# Hoặc dùng Webhook.site để kiểm tra payload trước
# → Trỏ GitHub webhook → webhook.site để xem raw payload
# → Sau đó trỏ lại Jenkins
```

---

## Câu Hỏi Phỏng Vấn

**Q: Webhook và Poll SCM khác nhau như thế nào? Khi nào dùng cái nào?**

> Webhook là cơ chế **push** — GitHub/GitLab chủ động gửi thông báo đến Jenkins khi có sự kiện. Poll SCM là cơ chế **pull** — Jenkins định kỳ hỏi repository "có thay đổi mới không?". Webhook phản hồi trong 1–3 giây và không tốn tài nguyên polling. Poll SCM tạo độ trễ và tải API không cần thiết. Dùng Webhook khi Jenkins có thể nhận HTTP từ internet; dùng Poll SCM khi Jenkins nằm sau firewall nghiêm ngặt không thể mở endpoint.

**Q: Làm thế nào để bảo mật webhook endpoint khỏi bị giả mạo?**

> Ba lớp bảo mật: **(1) Secret Token** — GitHub/GitLab ký request bằng HMAC-SHA256, Jenkins xác minh chữ ký trước khi xử lý; **(2) IP Allowlist** — chỉ chấp nhận request từ dải IP của GitHub/GitLab; **(3) HTTPS** — mã hóa toàn bộ payload trong transit. Kết hợp cả ba tạo ra bảo mật nhiều lớp.

**Q: Webhook gửi HTTP 200 nhưng build không chạy — bạn debug như thế nào?**

> Kiểm tra tuần tự: **(1)** Xem Jenkins System Log tìm dòng "Received POST" để xác nhận Jenkins nhận được event; **(2)** Kiểm tra Git URL trong job có khớp chính xác với URL repository trên GitHub không; **(3)** Xác nhận job có kích hoạt "GitHub hook trigger for GITScm polling"; **(4)** Bật GitHub Hook Log để xem chi tiết processing. Lỗi thường gặp nhất là URL không khớp (có/không có `.git` ở cuối).

**Q: Generic Webhook Trigger là gì và khi nào cần dùng?**

> Generic Webhook Trigger là plugin cho phép nhận webhook từ *bất kỳ* hệ thống nào — Jira, Trello, hệ thống nội bộ — không chỉ GitHub/GitLab. Nó trích xuất dữ liệu từ JSON/XML payload bằng JSONPath/XPath và đặt vào biến môi trường. Dùng khi cần trigger Jenkins từ hệ thống không có plugin tích hợp sẵn.

---

**Liên Kết Liên Quan:**
- [2-cron-scheduling.md](2-cron-scheduling.md) — Lịch Cron và Poll SCM (phương án thay thế webhook)
- [README.md](README.md) — Tổng quan Build Triggers

**Cập Nhật:** 2026-05-10
