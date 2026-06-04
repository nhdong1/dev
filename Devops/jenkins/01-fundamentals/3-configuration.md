# Cấu Hình Jenkins: System Config, JVM Settings, Global Tool

## Tổng Quan

Sau khi cài đặt, Jenkins cần được cấu hình trước khi đưa vào sử dụng. Bài này hướng dẫn các cài đặt hệ thống quan trọng, tối ưu JVM (Java Virtual Machine — Máy Ảo Java), và cấu hình Global Tool để pipeline có thể dùng Maven, Gradle, Node.js, v.v.

---

## 1. Manage Jenkins — Trung Tâm Cấu Hình

Mọi cấu hình Jenkins nằm tại: **Manage Jenkins** (thanh bên trái khi đăng nhập admin)

```
Manage Jenkins
├── System              ← Cấu hình hệ thống (Jenkins URL, email, số Executor)
├── Tools               ← Cấu hình công cụ build toàn cục (Maven, JDK, Node.js)
├── Plugins             ← Quản lý Plugin
├── Nodes               ← Quản lý Agent và Controller executors
├── Credentials         ← Quản lý thông tin xác thực
├── Security            ← Authentication và Authorization
├── Script Console      ← Chạy Groovy script (dùng khi debug hoặc admin khẩn cấp)
└── System Log          ← Log hệ thống Jenkins
```

---

## 2. System Configuration (Cấu Hình Hệ Thống)

Truy cập: **Manage Jenkins → System**

### Jenkins URL (Địa Chỉ Jenkins)

```
Jenkins URL: https://jenkins.example.com/
```

**Tại sao quan trọng:**
- Webhook callback từ GitHub/GitLab cần URL chính xác
- Email notification chứa link đến build — cần URL đúng để click vào được
- JNLP Agent dùng URL này để kết nối

**Sai lầm phổ biến:** Để mặc định `http://localhost:8080` trong production → webhook không hoạt động, email link không truy cập được từ bên ngoài.

### System Admin Email (Email Quản Trị)

```
System Admin e-mail address: jenkins-noreply@example.com
```

Địa chỉ này dùng làm địa chỉ "From" khi Jenkins gửi email thông báo.

### Global Properties (Thuộc Tính Toàn Cục)

**Environment Variables** (Biến Môi Trường): Khai báo biến môi trường áp dụng cho tất cả build.

```
☑ Environment variables
  DOCKER_REGISTRY  = registry.example.com
  APP_ENV          = production
  MAVEN_OPTS       = -Xmx1g
```

Các biến này có thể dùng trong Jenkinsfile:

```groovy
steps {
    sh "docker push ${DOCKER_REGISTRY}/my-app:${BUILD_NUMBER}"
}
```

### Quiet Period (Thời Gian Yên Tĩnh)

```
Quiet period: 5 (giây)
```

Khi nhận trigger (webhook, poll SCM), Jenkins đợi thêm N giây trước khi bắt đầu build. Dùng để gom nhiều commit push liên tiếp thành một build duy nhất, tránh build thừa.

### SCM Checkout Retry Count (Số Lần Thử Lại Khi Checkout SCM Thất Bại)

```
SCM checkout retry count: 2
```

Số lần Jenkins thử lại checkout source code nếu gặp lỗi tạm thời (network timeout, Git server quá tải).

### Number of Executors (Số Executor Trên Controller)

```
# of executors: 0
```

**Production:** Luôn đặt = 0. Build không nên chạy trên Controller.  
**Lab/Development:** Có thể đặt 2-4 để tiện dùng mà không cần cấu hình Agent riêng.

---

## 3. JVM Settings (Cài Đặt Máy Ảo Java)

Jenkins chạy trên JVM — cấu hình JVM ảnh hưởng trực tiếp đến hiệu suất và sự ổn định.

### Vị Trí Cấu Hình JVM

**Ubuntu/Debian (apt):**
```bash
sudo nano /etc/default/jenkins
```

```bash
# /etc/default/jenkins
JAVA_ARGS="-Xmx4g -Xms1g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

**CentOS/RHEL (yum):**
```bash
sudo nano /etc/sysconfig/jenkins
```

```bash
# /etc/sysconfig/jenkins
JENKINS_JAVA_OPTIONS="-Xmx4g -Xms1g -XX:+UseG1GC"
```

**Docker:**
```yaml
# docker-compose.yml
environment:
  - JAVA_OPTS=-Xmx4g -Xms1g -XX:+UseG1GC
```

**Kubernetes Helm:**
```yaml
# values.yaml
controller:
  javaOpts: "-Xmx3g -Xms1g -XX:+UseG1GC"
```

### Giải Thích Các JVM Options Quan Trọng

| Option | Ý Nghĩa | Giá Trị Khuyến Nghị |
|--------|---------|---------------------|
| `-Xmx4g` | Maximum Heap Memory (bộ nhớ heap tối đa) | 50–75% RAM máy chủ |
| `-Xms1g` | Initial Heap Memory (bộ nhớ heap ban đầu) | 25% của Xmx |
| `-XX:+UseG1GC` | Bật G1 Garbage Collector (GC — Bộ Thu Gom Rác) | Khuyến nghị cho Jenkins |
| `-XX:MaxGCPauseMillis=200` | GC pause tối đa 200ms | Giảm lag giao diện |
| `-Djava.awt.headless=true` | Không dùng màn hình đồ họa | Bắt buộc trên server |
| `-Dhudson.model.DirectoryBrowserSupport.CSP=` | Tắt CSP (chú ý: bảo mật) | Chỉ dùng khi cần xem HTML report |

### Cấu Hình JVM Cho Jenkins Theo Kích Thước

**Small (< 50 jobs, 2-5 agent):**
```
-Xmx2g -Xms512m -XX:+UseG1GC
```

**Medium (50–200 jobs, 5–20 agent):**
```
-Xmx4g -Xms1g -XX:+UseG1GC -XX:MaxGCPauseMillis=200
```

**Large (> 200 jobs, 20+ agent):**
```
-Xmx8g -Xms2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:G1HeapRegionSize=16m
```

### Kiểm Tra JVM Health (Sức Khỏe JVM)

Xem thông tin JVM đang chạy:
```
Manage Jenkins → System Information → "Memory Usage" section
```

Hoặc dùng REST API:
```bash
curl http://localhost:8080/computer/api/json?pretty=true
```

---

## 4. Global Tool Configuration (Cấu Hình Công Cụ Toàn Cục)

Truy cập: **Manage Jenkins → Tools**

Global Tool cho phép pipeline dùng các build tool (Maven, Gradle, JDK, Node.js, Git...) mà không cần cài trước trên Agent — Jenkins tự tải về và cài.

### Cấu Hình JDK (Java Development Kit)

```
JDK installations
  ☑ Install automatically
    Name:    JDK-17
    Version: 17 (latest)

  ☑ Install automatically
    Name:    JDK-11
    Version: 11 (latest)
```

Dùng trong Jenkinsfile:
```groovy
tools {
    jdk 'JDK-17'    // Khớp với Name đã cấu hình
}
```

### Cấu Hình Maven (Apache Maven — Công Cụ Build Java)

```
Maven installations
  ☑ Install automatically
    Name:    Maven-3.9
    Version: 3.9.6
```

Dùng trong Jenkinsfile:
```groovy
pipeline {
    agent any
    tools {
        maven 'Maven-3.9'   // Tên phải khớp chính xác
        jdk   'JDK-17'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
    }
}
```

### Cấu Hình Gradle (Gradle Build Tool)

```
Gradle installations
  ☑ Install automatically
    Name:    Gradle-8.5
    Version: Gradle 8.5
```

### Cấu Hình Node.js (Cần Plugin NodeJS)

Cài plugin **NodeJS** trước, sau đó cấu hình:

```
NodeJS installations
  ☑ Install automatically
    Name:            NodeJS-20-LTS
    Version:         NodeJS 20.11.0
    Global packages: yarn@1.22.19   ← Cài npm package toàn cục
```

Dùng trong Jenkinsfile:
```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-20-LTS'
    }
    stages {
        stage('Install') {
            steps {
                sh 'npm install'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }
}
```

### Cấu Hình Git

```
Git installations
  Name:          Default
  Path to Git executable: /usr/bin/git   ← Đường dẫn đến binary git trên Agent
```

Thông thường Git đã có sẵn trên Agent, Jenkins tự detect. Chỉ cần chỉ định khi git nằm ở đường dẫn không chuẩn.

---

## 5. Credentials Configuration (Cấu Hình Thông Tin Xác Thực)

Truy cập: **Manage Jenkins → Credentials**

Credentials trong Jenkins được mã hóa và không hiển thị dạng plain text sau khi lưu.

### Phân Loại Scope (Phạm Vi)

| Scope | Phạm Vi | Khi Nào Dùng |
|-------|--------|-------------|
| **System** | Chỉ Jenkins nội bộ dùng | Kết nối email server, agent SSH key |
| **Global** | Tất cả job/pipeline đều dùng được | Secret phổ biến: Docker registry, GitHub token |
| **Folder/Project** | Chỉ job trong folder/project đó | Secrets riêng cho từng team/project |

### Các Loại Credential

```
Secret text          — API token, webhook secret
Username/Password    — Docker registry, Nexus, database
SSH Username + Key   — Kết nối Git qua SSH, SSH vào server
Certificate (.p12)   — Apple signing, mTLS
Secret file          — kubeconfig, service account JSON
```

### Tạo Credential Qua UI

```
Credentials → System → Global credentials → Add Credentials
  Kind:       Secret text
  Scope:      Global
  Secret:     ghp_xxxxxxxxxxxx   (GitHub Personal Access Token)
  ID:         github-token        ← ID này dùng trong Jenkinsfile
  Description: GitHub PAT for webhook and API calls
```

Dùng trong Jenkinsfile:
```groovy
withCredentials([
    string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')
]) {
    sh 'curl -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user'
}
```

---

## 6. Security Configuration (Cấu Hình Bảo Mật)

Truy cập: **Manage Jenkins → Security**

### Authentication (Xác Thực) Cơ Bản

```
Security Realm: Jenkins' own user database
  ☑ Allow users to sign up    ← Tắt trong production!
```

**Production khuyến nghị:**
```
Security Realm: LDAP  (hoặc Active Directory, OAuth2 Proxy)
```

### Authorization (Phân Quyền) Cơ Bản

```
Authorization: Matrix-based security
  Authenticated Users: Read
  admin:               Administer (toàn quyền)
```

### Cross Site Request Forgery Protection (Bảo Vệ CSRF)

```
☑ Enable proxy compatibility   ← Bật nếu Jenkins đứng sau reverse proxy (nginx, HAProxy)
☑ CSRF Protection              ← Luôn bật trong production
```

### Agent ↔ Controller Security

```
☑ Enable Agent → Controller Access Control
```

Tắt tùy chọn này rất nguy hiểm: Agent compromised (bị tấn công) có thể đọc secrets trên Controller.

---

## 7. Jenkins Configuration as Code — JCasC

**JCasC** (Jenkins Configuration as Code — Cấu Hình Jenkins Dưới Dạng Code) cho phép lưu toàn bộ cấu hình Jenkins trong file YAML, thay vì click qua UI. Đây là best practice hiện đại.

### Cài Plugin

```
Cài plugin: "Configuration as Code"
```

### Ví Dụ File casc.yaml

```yaml
# casc.yaml — Cấu hình Jenkins hoàn chỉnh dưới dạng code

jenkins:
  systemMessage: "Jenkins — Managed by JCasC"
  numExecutors: 0         # Controller không chạy build
  mode: NORMAL

  # Agent security
  remotingSecurity:
    enabled: true

  # Global credentials
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          password: "${JENKINS_ADMIN_PASSWORD}"  # Từ environment variable

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: admin
            permissions:
              - Overall/Administer
            assignments:
              - admin
          - name: developer
            permissions:
              - Overall/Read
              - Job/Read
              - Job/Build
            assignments:
              - developer-group

tool:
  git:
    installations:
      - name: Default
        home: /usr/bin/git

  maven:
    installations:
      - name: Maven-3.9
        properties:
          - installSource:
              installers:
                - maven:
                    id: 3.9.6

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: docker-registry-creds
              username: "${DOCKER_USERNAME}"
              password: "${DOCKER_PASSWORD}"
              description: "Docker Registry Credentials"

unclassified:
  location:
    url: "https://jenkins.example.com/"
    adminAddress: "jenkins-admin@example.com"

  mailer:
    smtpHost: smtp.example.com
    smtpPort: 587
    useSsl: false
    charset: UTF-8
```

### Load JCasC Khi Khởi Động

```bash
# Chỉ định đường dẫn file cấu hình qua biến môi trường
export CASC_JENKINS_CONFIG=/path/to/casc.yaml
java -jar jenkins.war

# Hoặc trong Docker
docker run -d \
  -e CASC_JENKINS_CONFIG=/var/jenkins_home/casc.yaml \
  -v $(pwd)/casc.yaml:/var/jenkins_home/casc.yaml \
  jenkins/jenkins:lts-jdk17
```

**Ưu điểm JCasC:**
- Cấu hình được version-controlled (quản lý phiên bản) trong Git
- Dễ reproducible (tái tạo) môi trường Jenkins giống hệt nhau
- Audit trail (nhật ký kiểm tra) rõ ràng — ai thay đổi gì, khi nào
- Phù hợp với GitOps workflow

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Nên đặt Xmx cho Jenkins là bao nhiêu?**

A: Thông thường 50–75% RAM của máy chủ dành cho Jenkins process, phần còn lại cho OS và agent (nếu chạy cùng máy). Ví dụ máy 8GB RAM: đặt `-Xmx4g -Xms1g`. Nếu Jenkins chạy riêng trong container Kubernetes, giới hạn memory của Pod là 8GB thì đặt `-Xmx6g` để còn buffer cho JVM overhead. Quan trọng: luôn dùng G1GC thay PermGen/ParallelGC cũ để tránh Full GC pause làm đơ giao diện.

**Q: JCasC mang lại lợi ích gì so với cấu hình thủ công qua UI?**

A: Ba lợi ích lớn nhất: (1) Infrastructure as Code — cấu hình Jenkins được commit vào Git, có audit trail và có thể review như code thông thường; (2) Disaster Recovery nhanh — nếu Jenkins mất dữ liệu, chỉ cần chạy lại với file YAML là khôi phục đúng trạng thái ban đầu; (3) Multi-environment consistency — dev/staging/production dùng cùng file YAML, chỉ khác environment variable, đảm bảo môi trường nhất quán. Đây là yêu cầu ngày càng phổ biến trong team DevOps chuyên nghiệp.
