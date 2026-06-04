# 4 — Security Best Practices: Hardening Jenkins cho Production

> Hardening (gia cố bảo mật) Jenkins là tập hợp các kỹ thuật và cấu hình giúp giảm thiểu bề mặt tấn công (attack surface), bảo vệ dữ liệu, và đảm bảo Jenkins không trở thành điểm yếu trong chuỗi CI/CD. Tài liệu này bao gồm CSP header, Script Security, Audit Log, và checklist production đầy đủ.

## Mục Lục

1. [Attack Surface Jenkins](#1-attack-surface-jenkins)
2. [Network Security](#2-network-security)
3. [CSP — Content Security Policy](#3-csp--content-security-policy)
4. [Script Security Plugin](#4-script-security-plugin)
5. [Agent-to-Master Security](#5-agent-to-master-security)
6. [Audit Logging](#6-audit-logging)
7. [Jenkins Update và Patch Management](#7-jenkins-update-và-patch-management)
8. [Secrets Management nâng cao](#8-secrets-management-nâng-cao)
9. [Hardening JVM và OS](#9-hardening-jvm-và-os)
10. [Security Scanning và Compliance](#10-security-scanning-và-compliance)
11. [Incident Response](#11-incident-response)
12. [Production Security Checklist](#12-production-security-checklist)

---

## 1. Attack Surface Jenkins

### Các Vector Tấn Công Phổ Biến

```
Internet / Internal Network
         │
         ▼
┌────────────────────────────────────────────────────────┐
│              JENKINS ATTACK VECTORS                    │
│                                                        │
│  1. Web UI / REST API                                  │
│     → Brute force login                                │
│     → CSRF (Cross-Site Request Forgery)                │
│     → XSS (Cross-Site Scripting)                       │
│     → Exposed /script endpoint                         │
│                                                        │
│  2. Jenkins CLI (qua HTTP / SSH)                       │
│     → Unauthorized CLI access                          │
│     → Java deserialization vulnerabilities             │
│                                                        │
│  3. Agent Connection                                   │
│     → Rogue agent (agent giả mạo) kết nối vào master  │
│     → Agent-to-master bypass                           │
│                                                        │
│  4. Pipeline / Groovy Script                           │
│     → Groovy sandbox escape                            │
│     → Command injection trong sh/bat steps             │
│     → Reading sensitive files qua workspace            │
│                                                        │
│  5. Plugin Vulnerabilities                             │
│     → Outdated plugins với CVE đã biết                 │
│     → Malicious plugin từ untrusted source             │
└────────────────────────────────────────────────────────┘
```

---

## 2. Network Security

### Reverse Proxy với HTTPS

Jenkins **không nên** expose trực tiếp ra internet. Luôn đặt sau reverse proxy (Nginx, Apache, AWS ALB) với HTTPS:

```nginx
# /etc/nginx/conf.d/jenkins.conf
server {
    listen 80;
    server_name jenkins.example.com;
    # Redirect tất cả HTTP sang HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name jenkins.example.com;

    ssl_certificate     /etc/ssl/certs/jenkins.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.key;

    # Chỉ cho TLS 1.2 và 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;
    ssl_prefer_server_ciphers off;

    # HSTS (HTTP Strict Transport Security — Bảo Mật Vận Chuyển Nghiêm Ngặt HTTP)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        # Tăng timeout cho long-running pipeline requests
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}
```

### Jenkins Proxy Configuration

```
Manage Jenkins → System → Jenkins URL:
→ Điền: https://jenkins.example.com  (không phải http://127.0.0.1:8080)
→ Quan trọng: Jenkins dùng URL này để tạo webhook callback URL, OAuth redirect
```

### Firewall Rules (Quy Tắc Tường Lửa)

```
# Chỉ cho phép traffic cần thiết đến Jenkins:

# Port 443  → Từ: Reverse proxy / Load balancer → Jenkins
# Port 8080  → Từ: Reverse proxy (127.0.0.1) → Jenkins
# Port 50000 → Từ: Agent nodes → Jenkins Master (JNLP agent port)
# Port 22    → Từ: SSH agents hoặc bastion host (nếu dùng SSH agent)

# Chặn hoàn toàn:
# - Port 8080 từ internet (truy cập trực tiếp, bỏ qua reverse proxy)
# - Jenkins Script Console từ mọi IP ngoại trừ admin whitelist
```

### Tắt Jenkins CLI qua HTTP

Jenkins CLI qua HTTP/HTTPS từng có nhiều lỗ hổng deserialization. Tắt nếu không cần:

```
Manage Jenkins → Security → CLI
→ Tắt "Enable CLI over Remoting" (CLI qua Remoting protocol)
→ Giữ "Allow CLI to connect with SSH" nếu cần CLI qua SSH (an toàn hơn)
```

---

## 3. CSP — Content Security Policy

**CSP** (Content Security Policy — Chính Sách Bảo Mật Nội Dung) là HTTP header ngăn chặn XSS (Cross-Site Scripting — Tấn Công Kịch Bản Xuyên Trang) bằng cách kiểm soát nguồn tải resource.

### Vấn Đề Với CSP Mặc Định Của Jenkins

Jenkins có CSP header mặc định rất chặt — có thể làm vỡ giao diện của một số plugin (Blue Ocean, Allure Report, HTML Publisher):

```
X-Content-Security-Policy: sandbox; default-src 'none'; ...
```

### Cấu Hình CSP Cân Bằng (Bảo Mật + Tương Thích Plugin)

```groovy
// Script Console — set CSP header tùy chỉnh
System.setProperty(
    "hudson.model.DirectoryBrowserSupport.CSP",
    "default-src 'self'; " +
    "img-src 'self' data:; " +
    "style-src 'self' 'unsafe-inline'; " +
    "script-src 'self' 'unsafe-inline'; " +
    "font-src 'self'"
)
```

**Lưu ý:** Setting này không persist qua restart. Để persist, dùng JVM argument:

```
# /etc/default/jenkins hoặc Dockerfile
JAVA_OPTS="-Dhudson.model.DirectoryBrowserSupport.CSP=\"default-src 'self'; ...\""
```

### CSP cho Jenkins Master UI

Cấu hình CSP header trong Nginx reverse proxy:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;

    # CSP Header
    add_header Content-Security-Policy
        "default-src 'self'; "
        "script-src 'self' 'unsafe-inline' 'unsafe-eval'; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: blob:; "
        "font-src 'self' data:; "
        "connect-src 'self' wss:; "
        "frame-ancestors 'none';"
        always;

    # Chống clickjacking
    add_header X-Frame-Options "DENY" always;

    # Chống MIME sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # Referrer Policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

### Kiểm Tra CSP Headers

```bash
# Kiểm tra headers Jenkins trả về
curl -I https://jenkins.example.com | grep -i "content-security\|x-frame\|x-content"

# Test XSS protection bằng cách inject script vào URL param
curl "https://jenkins.example.com/search?q=<script>alert(1)</script>"
# Kết quả mong đợi: script bị encode, không thực thi
```

---

## 4. Script Security Plugin

**Script Security Plugin** (Plugin Bảo Mật Script) kiểm soát việc thực thi Groovy script trong Jenkins — đặc biệt quan trọng vì Groovy có thể truy cập Java API và hệ thống file.

### Hai Chế Độ Thực Thi Script

```
┌─────────────────────────────────────────────────────────────┐
│                   SCRIPT EXECUTION MODES                    │
│                                                             │
│  SANDBOX MODE (Chế Độ Hộp Cát) — Mặc định cho pipeline     │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Chỉ cho phép "safe" Groovy API                     │    │
│  │ Tự động từ chối lệnh nguy hiểm                     │    │
│  │ Ví dụ bị chặn:                                     │    │
│  │   new File('/etc/passwd').text                      │    │
│  │   Runtime.exec('rm -rf /')                         │    │
│  │   Jenkins.instance.restart()                        │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  UNSANDBOXED — Script Console, Approved Scripts             │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Toàn quyền truy cập Java/Groovy API                │    │
│  │ Chỉ admin được approve                             │    │
│  │ Cẩn thận: có thể đọc files, gọi system commands   │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Script Approval (Phê Duyệt Script)

Khi pipeline dùng Groovy code không có trong whitelist (danh sách trắng), Jenkins yêu cầu admin approve:

```
Manage Jenkins → In-process Script Approval

Pending Scripts:
✅ Approve    staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods ...
✅ Approve    method groovy.lang.GroovyObject getProperty java.lang.String
❌ Deny       staticMethod java.lang.Runtime exec java.lang.String  ← NGUY HIỂM
```

### Các Script Nguy Hiểm Thường Gặp Cần Từ Chối

```groovy
// Tuyệt đối KHÔNG approve các lệnh sau:
Runtime.getRuntime().exec("...")       // Command execution (thực thi lệnh)
new File("/etc/passwd").text           // Đọc file system
System.getenv()                        // Lấy toàn bộ environment variables
Jenkins.instance.doReload()            // Reload Jenkins config
Jenkins.instance.setSecurityRealm(null)// Tắt security
Thread.sleep(Long.MAX_VALUE)           // Denial of Service
```

### Declarative Pipeline Tránh Sandbox Issues

```groovy
// ❌ Scripted Pipeline — dễ gặp sandbox issues khi dùng Groovy API nâng cao
node {
    def files = new File('/workspace').listFiles()  // Bị block bởi sandbox
    // Cần approve staticMethod java.io.File listFiles
}

// ✅ Declarative Pipeline với sh step — an toàn hơn
pipeline {
    agent any
    stages {
        stage('List files') {
            steps {
                // Dùng shell thay vì Groovy trực tiếp
                sh 'ls -la'
                script {
                    // Dùng Pipeline DSL-safe APIs
                    def files = findFiles(glob: '**/*.java')
                    echo "Found ${files.size()} Java files"
                }
            }
        }
    }
}
```

### Shared Libraries và Script Security

```groovy
// Shared Library code trong vars/ chạy trong sandbox
// Shared Library code trong src/ cần approval hoặc @NonCPS annotation

// src/com/example/Utils.groovy
package com.example

class Utils implements Serializable {
    // @NonCPS: method không serialize — thoát sandbox một phần
    @com.cloudbees.groovy.cps.NonCPS
    static List<String> parseJson(String json) {
        def slurper = new groovy.json.JsonSlurper()
        return slurper.parseText(json)
    }
}
```

---

## 5. Agent-to-Master Security

**Agent-to-Master Security** (Bảo Mật Từ Agent Đến Master) ngăn agent độc hại điều khiển Jenkins master — quan trọng khi agent chạy untrusted code (code không tin cậy).

### Mô Hình Đe Dọa

```
Kịch bản tấn công:
1. Kẻ tấn công submit PR chứa mã độc trong Jenkinsfile
2. Jenkins tự động build PR → agent thực thi Jenkinsfile
3. Trong Jenkinsfile có code đọc credentials của master:
   sh "cat ${JENKINS_HOME}/credentials.xml"
4. Kết quả build chứa encrypted credentials
5. Kẻ tấn công dùng master key (nếu có) để decrypt

Hoặc:
3. Code inject vào agent → agent gửi malicious command lên master
4. Master thực thi lệnh với quyền admin
```

### Bật Agent-to-Master Access Control

```
Manage Jenkins → Security
→ ☑ Enable Agent → Master Access Control

Khi bật:
- Agent chỉ được đọc/ghi trong workspace của build đó
- Agent không đọc được files ngoài workspace
- Agent không gửi được command thay đổi Jenkins config
- Agent không truy cập credentials của job khác
```

### Cấu Hình Rules (Nếu Cần Nới Lỏng)

```groovy
// Script Console — xem và sửa agent access rules
import jenkins.security.s4m.*

def defender = Jenkins.instance.getExtensionList(
    RuleFilePathFilter.class
)[0]

// Xem rules hiện tại
println defender.toString()

// Thêm rule cho phép agent đọc file cụ thể (cẩn thận!)
// Không nên mở rộng quá nhiều
```

### Isolated Build Environments

```groovy
// Dùng Docker agent để cô lập hoàn toàn
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-21'
            // Container riêng cho mỗi build
            // Khi build xong: container bị xóa
            // Build code không có quyền truy cập Jenkins home
            args '-v /tmp:/tmp:rw --network=bridge'
            // Không mount Jenkins home vào container
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn package'
                // Code chạy trong container isolated
                // Không thể đọc /var/jenkins_home
            }
        }
    }
}
```

---

## 6. Audit Logging

**Audit Log** (Nhật Ký Kiểm Tra) ghi lại tất cả hành động quan trọng trong Jenkins — ai đăng nhập, ai thay đổi cấu hình, job nào được kích hoạt, credentials nào được truy cập.

### Plugin Cần Thiết

```
Audit Trail Plugin (audit-trail)
```

### Cấu Hình Audit Trail

```
Manage Jenkins → Configure System → Audit Trail

Loggers:
  ☑ Log File Logger:
    Log Location:    /var/log/jenkins/audit.log
    Log File Size:   100 MB
    Log File Count:  10 (rotation — giữ 10 file cuối)

  ☑ Syslog Logger:
    Syslog Server:   syslog.example.com
    Port:            514
    Syslog Facility: USER

URI Patterns (Các endpoint cần ghi log):
  .*/configSubmit.*
  .*/createItem.*
  .*/deleteItem.*
  .*/credential-store/.*
  .*/securityRealm/.*
  .*/safeRestart.*
  .*/quietDown.*
  .*/credentials/.*
```

### Ví Dụ Audit Log Entries

```
2026-05-11 10:23:45 +0700 [alice] POST /jenkins/job/payments-api/build
   → alice kích hoạt build payments-api

2026-05-11 10:45:12 +0700 [admin] POST /jenkins/credentials/store/system/domain/_/credential/stripe-api-key/config.xml
   → admin update credential 'stripe-api-key'

2026-05-11 11:02:33 +0700 [bob] POST /jenkins/job/payments-api/configure
   → bob thay đổi cấu hình job payments-api

2026-05-11 11:15:00 +0700 [unknown] GET /jenkins/script
   → Truy cập Script Console! Cần điều tra ngay
```

### Tích Hợp Với SIEM (Security Information and Event Management)

```yaml
# Gửi audit log đến ELK Stack (Elasticsearch, Logstash, Kibana)
# filebeat.yml trên Jenkins server
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/jenkins/audit.log
      - /var/log/jenkins/jenkins.log
    fields:
      service: jenkins
      environment: production
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.match: after

output.logstash:
  hosts: ["logstash.example.com:5044"]
```

### Alert Quan Trọng Cần Set Up

```yaml
# Cảnh báo cần theo dõi trong SIEM/monitoring:

1. Đăng nhập thất bại nhiều lần:
   Điều kiện: > 5 lần thất bại trong 5 phút từ cùng IP
   Hành động: Block IP, alert security team

2. Truy cập Script Console:
   Điều kiện: Bất kỳ GET/POST nào đến /script
   Hành động: Alert ngay, không ngoại lệ

3. Thay đổi Security Configuration:
   Điều kiện: POST đến /configureSecurity
   Hành động: Alert admin, yêu cầu xác nhận

4. Credential được cập nhật ngoài giờ hành chính:
   Điều kiện: Thay đổi credentials sau 20:00 hoặc cuối tuần
   Hành động: Alert on-call engineer

5. Job mới được tạo bởi user không phải admin:
   Điều kiện: createItem bởi non-admin user
   Hành động: Alert cho review thủ công
```

---

## 7. Jenkins Update và Patch Management

### LTS vs Weekly Release

```
Jenkins Release Train (Lịch Phát Hành):
┌────────────────┬───────────────────────────────────────────┐
│ LTS (Long-Term │ Phát hành mỗi 12 tuần                     │
│ Support —      │ Bản ổn định nhất, khuyến nghị production  │
│ Hỗ Trợ Dài    │ Mỗi 4 tuần có bản patch (2.x.1, 2.x.2...)│
│ Hạn)           │ Security fixes được backport              │
├────────────────┼───────────────────────────────────────────┤
│ Weekly         │ Phát hành hàng tuần                       │
│                │ Có tính năng mới nhất                     │
│                │ Ít ổn định hơn                             │
│                │ Dùng cho development, testing             │
└────────────────┴───────────────────────────────────────────┘
```

### Quy Trình Update An Toàn

```bash
# 1. Kiểm tra release notes và security advisories
# https://www.jenkins.io/security/advisories/

# 2. Test trên staging trước
# Môi trường staging phải mirror production

# 3. Backup trước khi update
systemctl stop jenkins
tar -czf /backup/jenkins-$(date +%Y%m%d).tar.gz /var/lib/jenkins

# 4. Update Jenkins
# Debian/Ubuntu:
apt-get update && apt-get install jenkins=2.x.x

# Hoặc thay jenkins.war:
cp /usr/share/jenkins/jenkins.war /backup/jenkins.war.bak
wget -O /usr/share/jenkins/jenkins.war \
  https://get.jenkins.io/war-stable/2.x.x/jenkins.war

# 5. Restart và verify
systemctl start jenkins
# Kiểm tra: http://jenkins.example.com/login

# 6. Update plugins ngay sau khi update Jenkins
# Manage Jenkins → Plugins → Updates → Update All
```

### Theo Dõi Security Advisories

```bash
# Subscribe Jenkins security advisories
# https://www.jenkins.io/security/advisories/
# Hoặc theo dõi RSS feed: https://www.jenkins.io/security/advisories/rss.xml

# Script kiểm tra plugins có CVE đã biết
curl -s "https://plugins.jenkins.io/api/plugin" | jq '.plugins[] | select(.securityWarnings != null) | .name'
```

### Plugin Security Hygiene (Vệ Sinh Bảo Mật Plugin)

```groovy
// Script Console — liệt kê plugins với security warnings
import jenkins.model.*

Jenkins.instance.pluginManager.plugins.findAll { plugin ->
    plugin.hasUpdate()
}.each { plugin ->
    println "${plugin.shortName} ${plugin.version} → ${plugin.updateInfo?.version}"
}

// Liệt kê plugins không dùng (để xem xét xóa)
Jenkins.instance.pluginManager.plugins.findAll { plugin ->
    !plugin.getDependents() && !plugin.isBundled()
}.each { plugin ->
    println "Potentially unused: ${plugin.shortName}"
}
```

---

## 8. Secrets Management nâng cao

### Tránh Secret Trong Build Log

```groovy
// Pattern để đảm bảo secret không xuất hiện trong log

// ❌ Các trường hợp có thể leak:
stage('Danger Zone') {
    steps {
        withCredentials([string(credentialsId: 'my-secret', variable: 'SECRET')]) {
            sh "set -x; ./deploy.sh"  // set -x in ra tất cả lệnh kể cả secret
            sh "env"                  // In tất cả environment variables
            sh "printenv"             // In tất cả environment variables
            echo "Token: $SECRET"     // In trực tiếp
        }
    }
}

// ✅ Đúng cách:
stage('Safe Deploy') {
    steps {
        withCredentials([string(credentialsId: 'my-secret', variable: 'SECRET')]) {
            sh '''
                # Không dùng set -x khi có secret
                ./deploy.sh
                # Script nhận secret qua environment variable
                # Không echo, không print
            '''
        }
    }
}
```

### Secret Scanning (Quét Secret) Trong Code

```yaml
# Dùng Gitleaks trong pipeline để phát hiện secret bị commit nhầm
pipeline {
    agent any
    stages {
        stage('Secret Scan') {
            steps {
                sh """
                    docker run --rm \
                        -v ${WORKSPACE}:/repo \
                        zricethezav/gitleaks:latest \
                        detect \
                        --source /repo \
                        --report-format json \
                        --report-path /repo/gitleaks-report.json \
                        --exit-code 1
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
                failure {
                    echo "⚠️ Secret detected in code! Build failed."
                    // Gửi alert đến security team
                }
            }
        }
    }
}
```

---

## 9. Hardening JVM và OS

### JVM Security Settings (Cài Đặt Bảo Mật JVM)

```bash
# /etc/default/jenkins hoặc Dockerfile ENV
JENKINS_JAVA_OPTIONS="\
  -Djava.awt.headless=true \
  -Djenkins.install.runSetupWizard=false \
  -Dhudson.model.DirectoryBrowserSupport.CSP=\"default-src 'self';\" \
  -Dhudson.security.csrf.DefaultCrumbIssuer.EXCLUDE_SESSION_ID=true \
  -Djenkins.CLI.disabled=true \
  -Dhudson.remoting.ClassFilter.DEFAULTS_OVERRIDE_LOCATION=/var/lib/jenkins/blocked-classes.txt \
  -XX:+UseG1GC \
  -XX:MaxRAMPercentage=75.0 \
  -Xss512k"
```

### OS-level Hardening

```bash
# 1. Chạy Jenkins với dedicated user (không phải root)
# Jenkins installer tạo user 'jenkins' tự động
# Verify:
id jenkins  # uid=1001(jenkins) gid=1001(jenkins)

# 2. Hạn chế quyền file JENKINS_HOME
chmod 750 /var/lib/jenkins
chmod 600 /var/lib/jenkins/secrets/master.key
chmod 600 /var/lib/jenkins/credentials.xml

# 3. Hạn chế quyền chạy sudo (nếu cần)
# /etc/sudoers.d/jenkins
jenkins ALL=(ALL) NOPASSWD: /usr/bin/docker
# Chỉ cho phép lệnh docker cụ thể, không cho ALL

# 4. Bật SELinux hoặc AppArmor profiles
# Giới hạn syscall Jenkins có thể gọi

# 5. Tắt core dumps
echo "* soft core 0" >> /etc/security/limits.conf
echo "* hard core 0" >> /etc/security/limits.conf
# Core dump có thể chứa secrets từ bộ nhớ Jenkins
```

### Docker Security cho Jenkins Container

```dockerfile
# Dockerfile cho Jenkins production
FROM jenkins/jenkins:2.x-lts-jdk21

USER root

# Cập nhật packages
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*

# Tắt default plugins có thể không cần
# (cài chỉ những plugin cần thiết)

# Chạy với non-root user
USER jenkins

# Read-only filesystem (bật ở deployment level)
# docker run --read-only --tmpfs /tmp --tmpfs /var/jenkins_home/...

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s \
  CMD curl -f http://localhost:8080/login || exit 1
```

```yaml
# Kubernetes — SecurityContext cho Jenkins pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: jenkins
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false  # Jenkins cần ghi vào home
            capabilities:
              drop:
                - ALL
```

---

## 10. Security Scanning và Compliance

### OWASP Dependency Check Trong Pipeline

```groovy
// Quét dependency vulnerabilities (lỗ hổng phụ thuộc)
pipeline {
    agent any
    stages {
        stage('Dependency Security Scan') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan . --format HTML --format JSON',
                    odcInstallation: 'OWASP Dependency-Check'
                )
            }
            post {
                always {
                    dependencyCheckPublisher(
                        pattern: 'dependency-check-report.xml',
                        failedTotalCritical: 1,   // Fail nếu có Critical CVE
                        failedTotalHigh: 5         // Fail nếu có > 5 High CVE
                    )
                }
            }
        }
    }
}
```

### Trivy Container Scanning

```groovy
// Quét Docker image cho CVE
stage('Container Security Scan') {
    steps {
        sh """
            # Pull và scan image trước khi push
            docker build -t myapp:${BUILD_NUMBER} .
            
            trivy image \
                --exit-code 1 \
                --severity HIGH,CRITICAL \
                --format table \
                myapp:${BUILD_NUMBER}
        """
    }
}
```

---

## 11. Incident Response

### Khi Phát Hiện Jenkins Bị Xâm Phạm

```
BƯỚC 1 — ISOLATION (CÔ LẬP): 0–5 phút đầu tiên
  □ Cắt mạng Jenkins khỏi internet/internal network
  □ Tắt tất cả running builds
  □ Revoke tất cả API token và credentials
  □ Thông báo team

BƯỚC 2 — INVESTIGATION (ĐIỀU TRA): 5–60 phút
  □ Thu thập logs: audit.log, jenkins.log, access.log (nginx)
  □ Kiểm tra build history: có builds bất thường không?
  □ Kiểm tra Job configs: có Jenkinsfile bị sửa không?
  □ Kiểm tra Credentials: có cái nào bị truy cập không đúng?
  □ Kiểm tra file system: có file lạ trong JENKINS_HOME không?
  □ Kiểm tra network connections: Jenkins đang kết nối đến đâu?

BƯỚC 3 — CONTAINMENT (NGĂN CHẶN): Song song với điều tra
  □ Rotate toàn bộ credentials Jenkins đang giữ
  □ Revoke SSH keys đang được dùng
  □ Update passwords cho tất cả services Jenkins có quyền truy cập
  □ Alert teams liên quan: nếu Jenkins có thể deploy lên production → alert SRE

BƯỚC 4 — RECOVERY (PHỤC HỒI): Sau khi điều tra xong
  □ Restore Jenkins từ backup sạch (trước thời điểm compromise)
  □ Rebuild Jenkins từ IaC (Infrastructure as Code) nếu có
  □ Import lại credentials (đã được rotate)
  □ Test tất cả pipelines trên staging trước
  □ Gradually restore traffic

BƯỚC 5 — POST-MORTEM (PHÂN TÍCH SAU SỰ CỐ)
  □ Timeline đầy đủ của incident
  □ Root cause analysis (phân tích nguyên nhân gốc)
  □ Impact assessment (đánh giá mức độ ảnh hưởng)
  □ Lessons learned và action items
  □ Cải thiện monitoring và alerting
```

---

## 12. Production Security Checklist

### Checklist Triển Khai Jenkins Lên Production

```
NETWORK & ACCESS
═══════════════
☑ Jenkins chạy sau HTTPS reverse proxy (Nginx/Apache/ALB)
☑ HTTP redirect sang HTTPS
☑ Firewall chặn port 8080 từ internet
☑ Port 50000 (JNLP) chỉ mở cho agent network
☑ Jenkins CLI qua HTTP đã bị tắt

AUTHENTICATION
══════════════
☑ Đã bật "Enable Security"
☑ Tắt "Allow users to sign up"
☑ Đang dùng LDAP/OAuth2/SSO thay vì Jenkins DB (nếu có thể)
☑ Đã cấu hình escape hatch account cho trường hợp khẩn cấp
☑ Admin password đủ mạnh (>16 ký tự, complex)

AUTHORIZATION
═════════════
☑ Không dùng "Anyone can do anything" hoặc "Logged-in users can do anything"
☑ Đang dùng Matrix-based hoặc Role Strategy Plugin
☑ Anonymous user không có quyền gì (kể cả Read)
☑ Developer không có Overall/Administer
☑ Credentials chỉ accessible từ đúng scope cần thiết

CREDENTIALS
═══════════
☑ Không có hardcoded secret trong bất kỳ Jenkinsfile nào
☑ Tất cả secrets dùng Jenkins Credentials Store hoặc Vault
☑ Credentials có ID có ý nghĩa, có description rõ ràng
☑ Rotation schedule đã được thiết lập
☑ master.key đã được backup ở nơi an toàn

SCRIPT SECURITY
═══════════════
☑ Script Security Plugin đã bật (mặc định trong Jenkins 2)
☑ Sandbox mode bật cho pipeline (mặc định)
☑ Script Console chỉ accessible cho admin
☑ Không có unapproved scripts đang pending mà không được review
☑ Shared Libraries từ nguồn tin cậy (trusted branch/tag)

AGENT SECURITY
══════════════
☑ Agent-to-Master Access Control đã bật
☑ Build agents dùng dedicated service account (không phải root)
☑ Docker agents tắt privileged mode trừ khi thực sự cần
☑ Kubernetes agents chạy với non-root security context

AUDIT & MONITORING
══════════════════
☑ Audit Trail Plugin đã cài và cấu hình
☑ Logs gửi đến centralized logging (ELK/Splunk/CloudWatch)
☑ Alerts cho suspicious activities đã thiết lập
☑ Thời gian retention log ít nhất 90 ngày (hoặc theo compliance)

UPDATES
════════
☑ Đang dùng Jenkins LTS (không phải Weekly)
☑ Có quy trình update định kỳ (ít nhất mỗi tháng)
☑ Subscribed Jenkins security advisories
☑ Staging environment để test update trước production
☑ Backup procedure đã test restore gần đây

CSP & HEADERS
═════════════
☑ Content-Security-Policy header đã cấu hình
☑ X-Frame-Options: DENY
☑ X-Content-Type-Options: nosniff
☑ HSTS đã bật (max-age ít nhất 1 năm)
☑ CSRF protection bật (Default Crumb Issuer)

BACKUP & RECOVERY
═════════════════
☑ JENKINS_HOME được backup tự động (ThinBackup hoặc snapshot)
☑ master.key backup tách biệt và secure
☑ Recovery procedure đã được test và document
☑ RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) < 4 giờ
☑ RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) < 24 giờ
```

---

## Tóm Tắt Nhanh

| Câu Hỏi                                                      | Câu Trả Lời                                                             |
| ------------------------------------------------------------- | ----------------------------------------------------------------------- |
| CSP header Jenkins mặc định có vấn đề gì?                   | Có thể break một số plugin UI — cần cân bằng giữa security và compatibility |
| Script Security Plugin làm gì?                               | Kiểm soát Groovy code trong pipeline qua sandbox và approval mechanism   |
| Agent-to-Master Security ngăn gì?                            | Ngăn agent độc hại đọc file system master hoặc thay đổi Jenkins config  |
| Audit Trail Plugin ghi lại gì?                               | Mọi HTTP action quan trọng: login, config change, credential access      |
| Bước đầu tiên khi phát hiện Jenkins bị compromise?           | Isolation — cắt mạng ngay, sau đó điều tra                               |

---

**Đã hoàn thành chủ đề 06-security.** Xem thêm:
- [07-integration/](../07-integration/) — Tích hợp Jenkins với Git, Docker, Kubernetes
- [11-interview-prep/](../11-interview-prep/) — Câu hỏi phỏng vấn về Security
