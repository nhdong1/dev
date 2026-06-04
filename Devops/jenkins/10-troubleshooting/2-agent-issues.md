# Agent Issues — Xử Lý Sự Cố Kết Nối Agent

> Agent (tác nhân) là thành phần thực thi build trong mô hình Master/Agent của Jenkins. Tài liệu này hướng dẫn chẩn đoán và khắc phục các sự cố thường gặp: agent offline, JNLP timeout, SSH key lỗi, và Docker agent.

## Mục Lục

1. [Kiến Trúc Agent Và Điểm Lỗi](#kiến-trúc-agent-và-điểm-lỗi)
2. [Agent Offline — Mất Kết Nối](#agent-offline--mất-kết-nối)
3. [JNLP Agent — Lỗi Kết Nối](#jnlp-agent--lỗi-kết-nối)
4. [SSH Agent — Lỗi Xác Thực](#ssh-agent--lỗi-xác-thực)
5. [Docker Agent — Lỗi Container](#docker-agent--lỗi-container)
6. [Kubernetes Agent — Lỗi Pod](#kubernetes-agent--lỗi-pod)
7. [Agent Crash Giữa Build](#agent-crash-giữa-build)
8. [Quản Lý Agent Tự Động](#quản-lý-agent-tự-động)

---

## Kiến Trúc Agent Và Điểm Lỗi

```
Jenkins Controller (Master)
         │
         │ ← Điểm lỗi 1: Network/Firewall
         │
    ┌────┴────┐
    │         │
 JNLP      SSH
 Agent     Agent
    │         │
    └────┬────┘
         │
    ┌────┴────────────┐
    │                 │
 Docker Agent    Kubernetes Agent (K8s Pod)
    │
 Container
```

### Các Loại Kết Nối Agent

| Loại | Cổng Mặc Định | Ai Kết Nối Đến Ai |
|------|---------------|-------------------|
| **JNLP** (Java Network Launch Protocol) | 50000 (TCP) | Agent → Controller |
| **SSH** | 22 (TCP) | Controller → Agent |
| **Inbound Agent** (thế hệ mới của JNLP) | 50000 (TCP) | Agent → Controller |
| **WebSocket Agent** | 443/80 (HTTPS/HTTP) | Agent → Controller |

---

## Agent Offline — Mất Kết Nối

### Triệu Chứng

```
Waiting for next available executor on [agent-name]
[agent-name] is offline
Build stayed in queue: Waiting for next available executor
```

### Chẩn Đoán Nhanh

**Bước 1:** Kiểm tra trạng thái agent trong Jenkins UI:
```
Manage Jenkins → Nodes → [agent-name] → Log
```

**Bước 2:** Đọc log của agent — tìm nguyên nhân disconnect:

| Thông báo trong Log | Nguyên Nhân |
|--------------------|-------------|
| `Connection was broken` | Mạng không ổn định |
| `Remote host closed connection` | Agent process bị kill |
| `java.io.IOException: Remote call failed` | Agent JVM crash |
| `hudson.remoting.ChannelClosedException` | Channel (kênh giao tiếp) bị đóng đột ngột |
| `No route to host` | Firewall hoặc routing lỗi |

**Bước 3:** SSH vào máy agent và kiểm tra:

```bash
# Kiểm tra process Jenkins agent còn chạy không
ps aux | grep jenkins-agent

# Kiểm tra kết nối mạng đến controller
nc -zv jenkins-controller.company.com 50000
# Kết quả mong muốn: "Connection to jenkins-controller.company.com 50000 port [tcp] succeeded"

# Kiểm tra log hệ thống
journalctl -u jenkins-agent --since "1 hour ago"
```

### Khởi Động Lại Agent Tự Động

```bash
# Tạo systemd service để tự restart agent khi crash
# File: /etc/systemd/system/jenkins-agent.service

[Unit]
Description=Jenkins Agent
After=network.target

[Service]
User=jenkins
WorkingDirectory=/opt/jenkins-agent
# Command khởi động agent — thay URL và secret của bạn
ExecStart=/usr/bin/java -jar /opt/jenkins-agent/agent.jar \
    -url http://jenkins-controller:8080 \
    -secret @/opt/jenkins-agent/secret.txt \
    -name my-agent \
    -workDir /opt/jenkins-agent/workspace
# Tự restart sau 5 giây nếu crash
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
# Kích hoạt service
systemctl enable jenkins-agent
systemctl start jenkins-agent
systemctl status jenkins-agent
```

---

## JNLP Agent — Lỗi Kết Nối

### Lỗi: Port 50000 Bị Block

**Triệu chứng:**
```
java.net.ConnectException: Connection refused
    at java.net.PlainSocketImpl.socketConnect(Native Method)
    connecting to Jenkins controller on port 50000
```

**Kiểm tra firewall:**

```bash
# Trên agent — kiểm tra có kết nối được đến port 50000 không
telnet jenkins-controller.company.com 50000
# Nếu không kết nối được: firewall đang block port này

# Trên controller — kiểm tra Jenkins đang listen trên port 50000
ss -tlnp | grep 50000
# hoặc
netstat -tlnp | grep 50000
```

**Giải pháp:**

```bash
# Mở port 50000 trong firewall (iptables)
iptables -A INPUT -p tcp --dport 50000 -j ACCEPT

# Hoặc dùng firewalld
firewall-cmd --permanent --add-port=50000/tcp
firewall-cmd --reload
```

**Hoặc chuyển sang WebSocket** — không cần mở port riêng:

```
Manage Jenkins → Configure Global Security
→ Agents → TCP port for inbound agents: WebSocket
```

```bash
# Khởi động agent với WebSocket
java -jar agent.jar \
    -url http://jenkins-controller:8080 \
    -secret <secret> \
    -name my-agent \
    -webSocket \
    -workDir /opt/jenkins-agent
```

### Lỗi: Secret Sai

**Triệu chứng:**
```
SEVERE: http://jenkins-controller:8080/ provided secret does not match
```

**Lấy lại secret đúng:**

```
Jenkins → Manage Nodes → [agent-name] → Configure
→ Phần "Connect agent" → Copy command đầy đủ
```

Hoặc qua Script Console:
```groovy
// Manage Jenkins → Script Console
def agent = Jenkins.instance.getNode("my-agent")
println jenkins.slaves.JNLPLauncher.getSecret(agent)
```

---

## SSH Agent — Lỗi Xác Thực

### Lỗi: Host Key Verification Failed

**Triệu chứng:**
```
Host key verification failed.
ERROR: Failed to connect to Jenkins agent
```

**Nguyên nhân:** Controller chưa biết SSH fingerprint (dấu vân tay) của agent.

**Giải pháp:**

```bash
# Trên Jenkins controller — thêm agent vào known_hosts (danh sách host đã biết)
ssh-keyscan -H agent-hostname >> ~/.ssh/known_hosts
# Hoặc
ssh-keyscan -H agent-ip >> ~/.ssh/known_hosts
```

**Hoặc cấu hình trong Jenkins UI:**
```
Manage Nodes → [agent] → Configure
→ Launch method: Launch agents via SSH
→ Host Key Verification Strategy:
   - "Non verifying" (không khuyến khích cho production)
   - "Known hosts file" (an toàn hơn, dùng ~/.ssh/known_hosts)
   - "Manually trusted key" (nhập fingerprint thủ công)
```

### Lỗi: Permission Denied (publickey)

**Triệu chứng:**
```
Permission denied (publickey,password).
ssh: connect to host agent-node port 22: Connection refused
```

**Chẩn đoán:**

```bash
# Trên controller — test SSH thủ công
ssh -i /var/jenkins_home/.ssh/id_rsa -v jenkins@agent-node
# -v: verbose mode để xem quá trình xác thực

# Kiểm tra authorized_keys trên agent
cat /home/jenkins/.ssh/authorized_keys
# Phải chứa public key của Jenkins controller
```

**Giải pháp:**

```bash
# Trên agent — thêm public key của controller
# 1. Lấy public key từ controller
ssh jenkins@jenkins-controller 'cat ~/.ssh/id_rsa.pub'

# 2. Thêm vào authorized_keys của agent
echo "ssh-rsa AAAA... jenkins@controller" >> /home/jenkins/.ssh/authorized_keys

# 3. Đảm bảo permissions đúng — SSH rất strict về permissions
chmod 700 /home/jenkins/.ssh
chmod 600 /home/jenkins/.ssh/authorized_keys
chown -R jenkins:jenkins /home/jenkins/.ssh
```

---

## Docker Agent — Lỗi Container

### Lỗi: Cannot Connect to Docker Daemon

**Triệu chứng:**
```
ERROR: error during connect: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

**Nguyên nhân:** Jenkins process không có quyền truy cập Docker socket (ổ cắm Docker).

**Giải pháp:**

```bash
# Thêm user jenkins vào group docker
usermod -aG docker jenkins

# Restart Jenkins để áp dụng
systemctl restart jenkins

# Kiểm tra
sudo -u jenkins docker ps
```

```groovy
// Hoặc mount socket khi chạy Jenkins trong Docker
// docker-compose.yml
services:
  jenkins:
    image: jenkins/jenkins:lts
    volumes:
      # Mount Docker socket vào container
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - "docker"  # Thêm Jenkins user vào group docker
```

### Lỗi: Image Không Pull Được

**Triệu chứng:**
```
ERROR: Failed to pull image "myregistry.company.com/myimage:latest"
unauthorized: authentication required
```

**Giải pháp:**

```groovy
// Cấu hình registry credentials trong pipeline
pipeline {
    agent {
        docker {
            image 'myregistry.company.com/myimage:latest'
            registryUrl 'https://myregistry.company.com'
            registryCredentialsId 'registry-credentials'  // ID trong Jenkins Credentials
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'echo "Running in private registry image"'
            }
        }
    }
}
```

### Container Bị Kill Do Hết Resource

**Triệu chứng:**
```
Error response from daemon: OOM killed
# hoặc
The command '/bin/sh -c mvn package' returned a non-zero code: 137
```

Exit code 137 = container bị kill bởi OOM Killer (OS kernel kill process vì hết RAM).

**Giải pháp:**

```groovy
// Tăng memory limit cho Docker container
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            // Giới hạn tài nguyên container
            args '--memory=4g --memory-swap=4g --cpus=2'
        }
    }
}
```

---

## Kubernetes Agent — Lỗi Pod

### Lỗi: Pod Không Start

**Triệu chứng:**
```
ERROR: Kubernetes pod [jenkins-agent-xxx] failed to start within 600 seconds
```

**Chẩn đoán:**

```bash
# Xem tất cả pods trong namespace jenkins
kubectl get pods -n jenkins

# Xem chi tiết pod bị lỗi (xem Events ở cuối output)
kubectl describe pod jenkins-agent-xxx -n jenkins

# Xem log của pod
kubectl logs jenkins-agent-xxx -n jenkins

# Xem log init container (nếu có)
kubectl logs jenkins-agent-xxx -n jenkins -c jnlp
```

**Các lỗi pod phổ biến:**

| Error | Nguyên Nhân | Giải Pháp |
|-------|-------------|-----------|
| `ImagePullBackOff` | Không pull được image | Kiểm tra image name, registry credentials |
| `CrashLoopBackOff` | Container crash liên tục | Xem logs, kiểm tra config |
| `Pending` mãi | Không có node đủ tài nguyên | Tăng cluster capacity |
| `OOMKilled` | Hết RAM | Tăng memory limit trong pod template |

### Cấu Hình Pod Template Đúng Cách

```groovy
// Jenkinsfile với Kubernetes pod template
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1"
  restartPolicy: Never
'''
            defaultContainer 'maven'
        }
    }
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

## Agent Crash Giữa Build

### Xử Lý Build Bị Abort Do Agent Crash

**Triệu chứng:**
```
java.io.IOException: Agent went offline while the task was in progress
ERROR: Agent disconnected during build
```

**Chiến lược phục hồi:**

```groovy
// Cấu hình retry tự động khi agent crash
pipeline {
    options {
        // Retry toàn bộ pipeline tối đa 2 lần
        retry(2)
    }
    stages {
        stage('Flaky Stage') {
            options {
                // Retry stage này độc lập
                retry(3)
            }
            steps {
                sh 'run-unstable-script.sh'
            }
        }
    }
}
```

### Stash/Unstash — Bảo Vệ Artifacts Khi Agent Thay Đổi

```groovy
// Stash (cất giữ) artifacts trên Controller để dùng ở agent khác
pipeline {
    stages {
        stage('Build on Agent A') {
            agent { label 'build-agent' }
            steps {
                sh 'mvn package'
                // Stash — lưu file vào Controller, không phụ thuộc agent
                stash name: 'build-artifacts', includes: 'target/*.jar'
            }
        }
        stage('Deploy on Agent B') {
            agent { label 'deploy-agent' }
            steps {
                // Unstash — lấy file từ Controller về agent mới
                unstash 'build-artifacts'
                sh 'deploy.sh target/app.jar'
            }
        }
    }
}
```

---

## Quản Lý Agent Tự Động

### Script Tự Động Bring Agent Online

```groovy
// Manage Jenkins → Script Console
// Bật lại tất cả offline agents
Jenkins.instance.nodes.each { node ->
    def computer = node.toComputer()
    if (computer.offline) {
        println "Bringing online: ${node.name}"
        computer.connect(true)  // true = force reconnect
    }
}
```

### Monitoring Agent Health (Sức Khỏe Agent)

```groovy
// Script kiểm tra sức khỏe tất cả agents
Jenkins.instance.nodes.each { node ->
    def computer = node.toComputer()
    def status = computer.offline ? "OFFLINE" : "ONLINE"
    def executors = computer.countBusy()
    def totalExecutors = computer.numExecutors

    println "${node.name}: ${status} | Executors: ${executors}/${totalExecutors}"

    if (computer.offline) {
        println "  Offline cause: ${computer.offlineCause}"
    }
}
```

---

**Xem Tiếp:** [3-performance-issues.md](3-performance-issues.md) — xử lý vấn đề hiệu suất Jenkins
