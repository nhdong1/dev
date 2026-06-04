# Agent Configuration — Cấu Hình Agent Jenkins

> **Agent** (tác nhân) là máy hoặc tiến trình kết nối với Jenkins Controller để thực thi build. Tài liệu này trình bày hai phương thức kết nối chính: **JNLP** (Java Network Launch Protocol — Giao Thức Khởi Chạy Mạng Java) và **SSH** (Secure Shell — Giao Thức Shell Bảo Mật), cùng cách quản lý Node Label và xử lý offline node.

---

## Mục Lục

1. [Khái Niệm Cơ Bản](#khái-niệm-cơ-bản)
2. [SSH Agent — Kết Nối Qua SSH](#ssh-agent--kết-nối-qua-ssh)
3. [JNLP Agent — Agent Tự Kết Nối Vào](#jnlp-agent--agent-tự-kết-nối-vào)
4. [Node Labels — Nhãn Node](#node-labels--nhãn-node)
5. [Cấu Hình Executor](#cấu-hình-executor)
6. [Offline Node — Xử Lý Node Mất Kết Nối](#offline-node--xử-lý-node-mất-kết-nối)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cơ Bản

### Node vs Agent vs Executor

| Thuật Ngữ | Ý Nghĩa |
|-----------|---------|
| **Node** (nút) | Máy tính (vật lý hoặc ảo) tham gia vào Jenkins cluster |
| **Agent** (tác nhân) | Tiến trình (process) chạy trên Node, nhận và thực thi lệnh từ Controller |
| **Executor** (bộ thực thi) | Một "slot" build trên Agent — số Executor = số build chạy đồng thời |
| **Workspace** (không gian làm việc) | Thư mục trên Agent chứa source code và output của build |
| **Node Label** (nhãn node) | Thẻ gán cho Node để Pipeline chọn đúng agent theo yêu cầu |

### Hai Chiều Kết Nối

```
┌────────────────────────────────────────────────────────────────┐
│  SSH Agent: Controller CHỦ ĐỘNG kết nối vào Agent              │
│                                                                  │
│  Controller ──SSH──► Agent                                       │
│  (port 22 trên Agent phải mở)                                   │
├────────────────────────────────────────────────────────────────┤
│  JNLP Agent: Agent CHỦ ĐỘNG kết nối vào Controller             │
│                                                                  │
│  Agent ──TCP──► Controller (port 50000)                         │
│  (port 50000 trên Controller phải mở)                           │
└────────────────────────────────────────────────────────────────┘
```

---

## SSH Agent — Kết Nối Qua SSH

### Khi Nào Dùng SSH Agent

- Controller có thể kết nối mạng trực tiếp đến Agent
- Agent là Linux/macOS (có SSH daemon)
- Muốn Jenkins quản lý vòng đời agent (tự động khởi động lại)

### Chuẩn Bị Máy Agent

```bash
# 1. Cài Java trên Agent (bắt buộc)
sudo apt-get update && sudo apt-get install -y openjdk-17-jre-headless

# 2. Tạo user jenkins chuyên dụng
sudo useradd -m -s /bin/bash jenkins

# 3. Tạo thư mục làm việc cho agent
sudo mkdir -p /var/jenkins/agent
sudo chown jenkins:jenkins /var/jenkins/agent

# 4. Đảm bảo SSH daemon đang chạy
sudo systemctl status ssh
sudo systemctl enable --now ssh
```

### Tạo SSH Key Pair

```bash
# Chạy trên Controller (hoặc máy admin)
ssh-keygen -t ed25519 -C "jenkins-agent-key" -f ~/.ssh/jenkins_agent_key

# Kết quả tạo ra hai file:
# ~/.ssh/jenkins_agent_key      ← Private key (khóa riêng) — lưu vào Jenkins Credentials
# ~/.ssh/jenkins_agent_key.pub  ← Public key (khóa công khai) — đặt lên Agent
```

### Cài Public Key Lên Agent

```bash
# Copy public key vào authorized_keys của user jenkins trên Agent
ssh-copy-id -i ~/.ssh/jenkins_agent_key.pub jenkins@<agent-ip>

# Hoặc thủ công:
cat ~/.ssh/jenkins_agent_key.pub | \
  ssh jenkins@<agent-ip> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Lưu Private Key vào Jenkins Credentials

```
Manage Jenkins → Credentials → System → Global credentials
→ Add Credentials
   Kind:        SSH Username with private key
   ID:          jenkins-agent-ssh-key
   Description: SSH key cho agent Linux
   Username:    jenkins
   Private Key: ✅ Enter directly → [Paste nội dung file jenkins_agent_key]
→ OK
```

### Thêm SSH Agent Node trên Jenkins

```
Manage Jenkins → Nodes → New Node
→ Node name:  agent-linux-01
→ Type:       ✅ Permanent Agent
→ OK

Cấu hình Node:
   Description:       Linux build agent
   Number of executors: 4
   Remote root directory: /var/jenkins/agent
   Labels:            linux java maven
   Usage:             ✅ Use this node as much as possible
   Launch method:     ✅ Launch agents via SSH

   SSH Host:          192.168.1.100   (hoặc hostname của Agent)
   Credentials:       jenkins-agent-ssh-key (vừa tạo ở trên)
   Host Key Verification Strategy:
      ✅ Manually trusted key Verification Strategy
      (lần đầu kết nối, chấp nhận fingerprint thủ công)

→ Save
→ Launch agent (kết nối thử)
```

### Jenkinsfile Dùng SSH Agent

```groovy
pipeline {
    // Chọn agent có label 'linux'
    agent { label 'linux' }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
}
```

---

## JNLP Agent — Agent Tự Kết Nối Vào

### Khi Nào Dùng JNLP Agent

- Agent nằm sau NAT (Network Address Translation) hoặc firewall — Controller không SSH vào được
- Agent là Windows (không có SSH daemon mặc định)
- Agent chạy trong môi trường cloud hoặc container
- Cần agent khởi động tự động khi reboot

### Mở TCP Port Trên Controller

```
Manage Jenkins → Security
→ Agents section
→ TCP port for inbound agents (Cổng TCP cho agent kết nối vào):
   ✅ Fixed: 50000
→ Save
```

### Cấu Hình JNLP Agent Node

```
Manage Jenkins → Nodes → New Node
→ Node name:  agent-windows-01
→ Type:       ✅ Permanent Agent
→ OK

Cấu hình Node:
   Remote root directory: C:\Jenkins\agent
   Labels:                windows dotnet
   Launch method:         ✅ Launch agent by connecting it to the controller
→ Save
```

Sau khi save, Jenkins hiển thị hướng dẫn kết nối. Truy cập trang Node để lấy lệnh:

```
Manage Jenkins → Nodes → agent-windows-01 → "How to connect this agent"
```

### Chạy Agent Trên Linux (Dòng Lệnh)

```bash
# Download agent.jar từ Controller
curl -sO http://jenkins.example.com/jnlpJars/agent.jar

# Chạy agent (thay thế <secret> và <agent-name> bằng giá trị thực)
java -jar agent.jar \
  -url http://jenkins.example.com/ \
  -secret <agent-secret-token> \
  -name agent-linux-02 \
  -workDir "/var/jenkins/agent"
```

### Chạy Agent Như Systemd Service (Dịch Vụ Hệ Thống)

```ini
# /etc/systemd/system/jenkins-agent.service

[Unit]
Description=Jenkins Agent (JNLP)
After=network.target

[Service]
User=jenkins
WorkingDirectory=/var/jenkins/agent
ExecStart=/usr/bin/java -jar /var/jenkins/agent/agent.jar \
  -url http://jenkins.example.com/ \
  -secret <agent-secret-token> \
  -name agent-linux-02 \
  -workDir "/var/jenkins/agent"
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now jenkins-agent
sudo systemctl status jenkins-agent
```

### Chạy JNLP Agent Trên Windows (Service)

```powershell
# Download agent.jar
Invoke-WebRequest -Uri "http://jenkins.example.com/jnlpJars/agent.jar" `
                  -OutFile "C:\Jenkins\agent.jar"

# Cài đặt như Windows Service dùng WinSW (Windows Service Wrapper)
# Download WinSW: https://github.com/winsw/winsw

# Tạo file jenkins-agent.xml
@"
<service>
  <id>JenkinsAgent</id>
  <name>Jenkins Agent</name>
  <executable>java</executable>
  <arguments>-jar C:\Jenkins\agent.jar -url http://jenkins.example.com/ -secret <token> -name agent-windows-01 -workDir C:\Jenkins\agent</arguments>
  <log mode="roll"/>
</service>
"@ | Out-File -FilePath "C:\Jenkins\jenkins-agent.xml"

# Cài và chạy service
.\winsw.exe install jenkins-agent.xml
.\winsw.exe start jenkins-agent.xml
```

---

## Node Labels — Nhãn Node

**Node Label** (nhãn node) là cơ chế phân loại Agent để Pipeline chọn đúng máy cho từng job. Một Node có thể có nhiều label.

### Đặt Label Cho Node

```
Manage Jenkins → Nodes → <tên node>
→ Labels: linux java maven docker
   (các label cách nhau bằng dấu cách)
→ Save
```

### Chiến Lược Đặt Tên Label

```
# Theo hệ điều hành (operating system)
linux, windows, macos

# Theo kiến trúc CPU (CPU architecture)
x86_64, arm64, arm

# Theo ngôn ngữ / runtime
java, nodejs, python, dotnet, go

# Theo công cụ build
maven, gradle, npm, pip

# Theo môi trường
staging, production, gpu

# Theo team
team-backend, team-frontend, team-data
```

### Dùng Label trong Jenkinsfile

```groovy
// Cách 1: Agent toàn pipeline
pipeline {
    agent { label 'linux && java' }  // Phải có CẢ HAI label

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

```groovy
// Cách 2: Agent khác nhau cho từng stage
pipeline {
    agent none  // Không có agent mặc định

    stages {
        stage('Build Java') {
            agent { label 'java && maven' }
            steps {
                sh 'mvn clean package -DskipTests'
                stash includes: 'target/*.jar', name: 'app-jar'
            }
        }

        stage('Build Docker Image') {
            agent { label 'docker' }
            steps {
                unstash 'app-jar'
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }

        stage('Deploy') {
            agent { label 'linux && production' }
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

### Label Expression — Biểu Thức Label

Jenkins hỗ trợ biểu thức logic để chọn agent:

```groovy
// Toán tử VÀ (&&) — cần cả hai label
agent { label 'linux && docker' }

// Toán tử HOẶC (||) — một trong hai
agent { label 'linux || macos' }

// Toán tử PHỦ ĐỊNH (!) — không có label này
agent { label 'linux && !gpu' }

// Kết hợp phức tạp
agent { label '(linux || macos) && java && !arm64' }
```

---

## Cấu Hình Executor

**Executor** (bộ thực thi) là số lượng build có thể chạy đồng thời trên một Node.

### Nguyên Tắc Chọn Số Executor

```
Executor tối ưu ≈ số CPU cores × (1 + wait_ratio)

Ví dụ:
- Node 4 CPU cores, build chủ yếu là CPU-bound (tính toán nặng):  Executor = 4
- Node 4 CPU cores, build có nhiều I/O wait (chờ mạng, disk):     Executor = 6–8
- Node dùng chạy Docker container builds:                         Executor = 1–2
```

### Cấu Hình Executor

```
Manage Jenkins → Nodes → <tên node>
→ Number of executors (Số lượng executor): 4
→ Save
```

### Tắt Executor Trên Controller

```
Manage Jenkins → Nodes → Built-In Node
→ Number of executors: 0
→ Save
```

> Đây là **best practice** quan trọng trong production: Controller chỉ làm nhiệm vụ điều phối, không chạy build trực tiếp để tránh ảnh hưởng đến hiệu suất và bảo mật.

---

## Offline Node — Xử Lý Node Mất Kết Nối

### Các Trạng Thái Node

| Trạng Thái | Ý Nghĩa |
|-----------|---------|
| **Online** (trực tuyến) | Agent kết nối và sẵn sàng nhận build |
| **Offline** (ngoại tuyến) | Agent mất kết nối, không nhận build mới |
| **Temporarily Offline** (tạm ngoại tuyến) | Người dùng chủ động đưa node offline (bảo trì) |
| **Suspended** (tạm dừng) | Node cloud đã bị thu hồi |

### Nguyên Nhân Node Offline Phổ Biến

```
1. Mạng mất kết nối (network timeout)
2. Agent process bị crash
3. Java process hết bộ nhớ (OutOfMemoryError)
4. SSH key thay đổi hoặc hết hạn
5. Firewall chặn port kết nối
6. Agent máy reboot mà chưa cấu hình auto-start
```

### Xem Log Kết Nối Agent

```
Manage Jenkins → Nodes → <tên node>
→ "Log" — xem log chi tiết quá trình kết nối

# Hoặc xem trực tiếp trên Agent
tail -f /var/jenkins/agent/remoting/logs/remoting.log
```

### Tự Động Kết Nối Lại

**Cho SSH Agent:** Jenkins tự động thử kết nối lại khi mất kết nối.

```
Manage Jenkins → Nodes → <tên node>
→ Availability: ✅ Keep this agent online as much as possible
→ Save
```

**Cho JNLP Agent:** Dùng `Restart=always` trong systemd service (xem phần trên).

### Đưa Node Vào Chế Độ Maintenance (Bảo Trì)

```
Manage Jenkins → Nodes → <tên node>
→ "Mark this node temporarily offline"
→ Nhập lý do: "Maintenance: upgrade Java version"
→ Build đang chạy tiếp tục; build mới không được phân bổ vào node này
```

### Pipeline Xử Lý Khi Agent Offline

```groovy
pipeline {
    agent { label 'linux' }

    options {
        // Thử lại tối đa 3 lần nếu agent bị offline
        retry(3)
        // Timeout toàn pipeline 1 giờ
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        stage('Build') {
            steps {
                retry(2) {
                    // Thử lại bước cụ thể này tối đa 2 lần
                    sh 'make build'
                }
            }
        }
    }
}
```

---

## Tổng Hợp So Sánh SSH vs JNLP

| Tiêu Chí | SSH Agent | JNLP Agent |
|----------|-----------|------------|
| **Ai kết nối trước** | Controller SSH vào Agent | Agent kết nối vào Controller |
| **Port cần mở** | Port 22 trên Agent | Port 50000 trên Controller |
| **Hệ điều hành** | Linux / macOS | Linux, Windows, macOS |
| **Firewall** | Cần Controller → Agent đi được | Cần Agent → Controller đi được |
| **Khởi động lại tự động** | Jenkins tự xử lý | Cần cấu hình systemd / Windows Service |
| **Bảo mật** | SSH key-based (rất an toàn) | Secret token (an toàn nếu dùng HTTPS) |
| **Dùng trong container** | Không phổ biến | Thường dùng (Kubernetes JNLP agent) |
| **Độ phức tạp cài đặt** | Trung bình | Thấp hơn một chút |

---

## Câu Hỏi Phỏng Vấn

**Q: JNLP Agent và SSH Agent khác nhau như thế nào? Khi nào dùng cái nào?**

> SSH Agent: Controller chủ động SSH vào Agent — dùng khi Controller có thể kết nối mạng tới Agent và Agent là Linux/macOS. JNLP Agent: Agent chủ động kết nối vào Controller — dùng khi Agent nằm sau NAT/firewall (Controller không vào được Agent), hoặc Agent là Windows, hoặc Agent chạy trong container/cloud. Kubernetes agent mặc định dùng JNLP vì Pod có thể kết nối ra ngoài nhưng ngoài không SSH vào Pod được.

**Q: Tại sao nên đặt Executor = 0 trên Controller trong production?**

> Controller có nhiệm vụ điều phối: lên lịch build, phục vụ Web UI, quản lý config. Nếu chạy build trực tiếp trên Controller, build nặng sẽ làm chậm toàn bộ hệ thống, UI trở nên ì ạch, và có thể gây bảo mật nguy hiểm nếu Jenkinsfile chạy trực tiếp trên Controller có quyền đọc `JENKINS_HOME` chứa tất cả credentials.

**Q: Node Label được dùng như thế nào trong Pipeline?**

> Node Label là thẻ phân loại Agent. Khi khai báo `agent { label 'linux && java' }`, Jenkins chỉ phân bổ build cho Agent có cả hai label `linux` và `java`. Dùng toán tử `&&` (AND), `||` (OR), `!` (NOT) để viết biểu thức phức tạp. Chiến lược đặt label tốt: theo OS, kiến trúc CPU, runtime, công cụ build, và môi trường.

**Q: Khi một Agent bị offline giữa chừng build, điều gì xảy ra?**

> Build thất bại ngay lập tức với lỗi "Agent went offline while the build was running". Build không tự resume (tiếp tục) vì workspace và trạng thái đang ở trên agent đó. Jenkins đánh dấu build là FAILURE hoặc ABORTED. Để giảm thiểu tác động: dùng `retry()` cho các bước quan trọng, dùng `stash`/`unstash` để lưu trữ artifact trên Controller, và tăng tính ổn định của Agent (systemd auto-restart, monitoring).

---

**Liên Kết Liên Quan:**
- [2-docker-agents.md](2-docker-agents.md) — Agent trong Docker container
- [3-kubernetes-agents.md](3-kubernetes-agents.md) — Kubernetes Pod Agent
- [4-node-management.md](4-node-management.md) — Quản lý Node toàn diện
- [README.md](README.md) — Tổng quan Distributed Builds

**Cập Nhật:** 2026-05-11
