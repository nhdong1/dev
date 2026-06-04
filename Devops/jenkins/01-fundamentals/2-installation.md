# Cài Đặt Jenkins: Standalone, Docker, Kubernetes

## Tổng Quan

Jenkins có thể được cài đặt theo nhiều cách tùy vào môi trường và nhu cầu. Bài này hướng dẫn 3 phương thức phổ biến nhất theo thứ tự từ đơn giản đến phức tạp.

| Phương Thức | Khi Nào Dùng | Độ Phức Tạp |
|------------|-------------|-------------|
| **Standalone** (WAR file / apt / yum) | Lab cá nhân, VM đơn | ⭐ |
| **Docker** | Development, demo nhanh | ⭐⭐ |
| **Kubernetes** (Helm) | Production, scalable | ⭐⭐⭐ |

---

## 1. Cài Đặt Standalone (Trực Tiếp Trên Máy Chủ)

### Yêu Cầu Hệ Thống

| Thành Phần | Tối Thiểu | Khuyến Nghị (Production) |
|-----------|----------|--------------------------|
| **Java** | JDK 11 | JDK 17 LTS |
| **RAM** | 256 MB | 4 GB+ |
| **CPU** | 1 core | 4+ cores |
| **Ổ đĩa** | 10 GB | 50 GB+ (SSD) |

> Jenkins yêu cầu Java — đây là dependency duy nhất bắt buộc.

### Cài Trên Ubuntu/Debian

```bash
# Bước 1: Cài Java 17 (JDK — Java Development Kit)
sudo apt update
sudo apt install -y openjdk-17-jdk

# Kiểm tra phiên bản Java
java -version
# Output: openjdk version "17.0.x" ...

# Bước 2: Thêm Jenkins repository và GPG key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  "https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Bước 3: Cài Jenkins
sudo apt update
sudo apt install -y jenkins

# Bước 4: Khởi động Jenkins service
sudo systemctl enable jenkins    # Tự khởi động khi reboot
sudo systemctl start jenkins
sudo systemctl status jenkins    # Kiểm tra trạng thái

# Bước 5: Lấy mật khẩu admin ban đầu
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Cài Trên CentOS/RHEL/Amazon Linux

```bash
# Bước 1: Cài Java 17
sudo yum install -y java-17-openjdk-devel

# Bước 2: Thêm Jenkins repository
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Bước 3: Cài Jenkins
sudo yum install -y jenkins

# Bước 4: Khởi động
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Lấy mật khẩu ban đầu
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Cài Bằng WAR File (Java Web Archive — Gói Ứng Dụng Web Java)

WAR file là phương thức linh hoạt nhất — chạy được trên bất kỳ hệ điều hành nào có Java.

```bash
# Tải Jenkins WAR (chọn LTS — Long-Term Support — Hỗ Trợ Dài Hạn)
wget https://get.jenkins.io/war-stable/latest/jenkins.war

# Chạy Jenkins trên port 8080
java -jar jenkins.war --httpPort=8080

# Chạy trên port khác (ví dụ 9090)
java -jar jenkins.war --httpPort=9090

# Chỉ định thư mục JENKINS_HOME khác
export JENKINS_HOME=/opt/jenkins-data
java -jar jenkins.war --httpPort=8080
```

### Truy Cập Web UI Lần Đầu

```
1. Mở trình duyệt: http://localhost:8080
2. Nhập initialAdminPassword
3. Chọn "Install suggested plugins" (cài plugin được đề xuất)
4. Tạo tài khoản admin đầu tiên
5. Cấu hình Jenkins URL (đặt đúng địa chỉ public nếu cần)
```

### Quản Lý Jenkins Service

```bash
# Khởi động / dừng / khởi động lại
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins

# Xem log realtime
sudo journalctl -u jenkins -f

# Kiểm tra port đang lắng nghe
sudo ss -tlnp | grep 8080

# Cấu hình JVM cho Jenkins service
sudo nano /etc/default/jenkins   # Ubuntu/Debian
# hoặc
sudo nano /etc/sysconfig/jenkins  # CentOS/RHEL
# Tìm JAVA_ARGS và thêm: -Xmx2g -Xms512m
```

---

## 2. Cài Đặt Với Docker

### Yêu Cầu

- Docker Engine đã cài và đang chạy
- Docker Compose (tùy chọn, khuyến nghị)

### Chạy Jenkins Nhanh (Dùng Để Thử Nghiệm)

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk17
```

Giải thích các tham số:
- `-p 8080:8080` — Port giao diện web (host:container)
- `-p 50000:50000` — Port JNLP Agent (agent kết nối vào Controller qua port này)
- `-v jenkins_home:/var/jenkins_home` — Volume lưu dữ liệu Jenkins, không mất khi container restart

### Docker Compose (Khuyến Nghị Cho Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins-controller
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock  # Cho phép Jenkins dùng Docker trên host
    environment:
      - JAVA_OPTS=-Xmx2g -Xms512m -Dhudson.footerURL=http://localhost:8080
    user: root  # Cần để truy cập docker.sock (xem lưu ý bảo mật bên dưới)

  jenkins-agent:
    image: jenkins/inbound-agent:latest
    container_name: jenkins-agent-01
    restart: unless-stopped
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_AGENT_NAME=agent-01
      - JENKINS_SECRET=<agent-secret-từ-controller>
      - JENKINS_AGENT_WORKDIR=/home/jenkins/agent
    depends_on:
      - jenkins

volumes:
  jenkins_home:
    driver: local
```

```bash
# Khởi động
docker compose up -d

# Xem log
docker compose logs -f jenkins

# Lấy mật khẩu ban đầu
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Dừng
docker compose down
```

> **Lưu ý bảo mật:** Mount `/var/run/docker.sock` cho phép Jenkins container tạo container Docker mới — đây là kỹ thuật "Docker outside of Docker" (DooD). Chạy với `user: root` để có quyền đọc socket. Trong production, nên dùng Docker-in-Docker (DinD) hoặc Kaniko để build image an toàn hơn.

### Dockerfile Tùy Chỉnh (Thêm Tool Vào Image)

```dockerfile
# custom-jenkins.Dockerfile
FROM jenkins/jenkins:lts-jdk17

# Cài plugin trước khi khởi động (không cần qua UI)
RUN jenkins-plugin-cli --plugins \
    git:latest \
    workflow-aggregator:latest \
    blueocean:latest \
    kubernetes:latest \
    docker-workflow:latest

# Chuyển sang root để cài thêm tool hệ thống
USER root
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    kubectl \
    && rm -rf /var/lib/apt/lists/*

# Cài Helm (Kubernetes package manager)
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Quay về user jenkins
USER jenkins
```

```bash
# Build image tùy chỉnh
docker build -f custom-jenkins.Dockerfile -t my-jenkins:latest .

# Chạy image tùy chỉnh
docker run -d --name jenkins -p 8080:8080 my-jenkins:latest
```

---

## 3. Cài Đặt Trên Kubernetes Với Helm

### Yêu Cầu

- Kubernetes cluster đang hoạt động (minikube, k3s, EKS, GKE, AKS...)
- Helm 3 đã cài
- kubectl đã cấu hình kết nối vào cluster

### Cài Helm Chart Chính Thức

```bash
# Bước 1: Thêm Helm repository của Jenkins
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Bước 2: Tạo namespace riêng
kubectl create namespace jenkins

# Bước 3: Xem giá trị mặc định để tùy chỉnh
helm show values jenkins/jenkins > jenkins-values.yaml
```

### Tùy Chỉnh values.yaml

```yaml
# jenkins-values.yaml — Các cài đặt quan trọng cần điều chỉnh

controller:
  # Phiên bản Jenkins
  tag: "2.440.3-jdk17"

  # Số lượng Executor trên Controller (nên đặt 0)
  numExecutors: 0

  # Plugin cài sẵn
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
    - job-dsl:latest

  # Tài nguyên cho Controller Pod
  resources:
    requests:
      cpu: "500m"       # 0.5 CPU core
      memory: "2Gi"     # 2 GB RAM
    limits:
      cpu: "2000m"      # 2 CPU cores
      memory: "4Gi"     # 4 GB RAM

  # JVM options
  javaOpts: "-Xmx3g -Xms1g"

  # Persistent Volume (lưu trữ bền vững) cho JENKINS_HOME
  persistence:
    enabled: true
    size: 50Gi
    storageClass: "standard"    # Thay bằng storage class của cluster

  # Service type để truy cập Jenkins UI
  serviceType: ClusterIP   # Dùng với Ingress

  # Ingress (bộ định tuyến HTTP) để expose Jenkins ra ngoài
  ingress:
    enabled: true
    ingressClassName: nginx
    hostName: jenkins.example.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod

  # Admin credentials
  adminUser: admin
  adminPassword: "change-me-secure-password"

agent:
  # Pod Template cho dynamic agents
  enabled: true
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
```

### Triển Khai Lên Kubernetes

```bash
# Bước 4: Cài Jenkins với values tùy chỉnh
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --values jenkins-values.yaml \
  --wait    # Đợi đến khi Pod sẵn sàng

# Kiểm tra trạng thái
kubectl get pods -n jenkins
kubectl get svc -n jenkins
kubectl get pvc -n jenkins

# Xem log Controller
kubectl logs -n jenkins -l app.kubernetes.io/name=jenkins -f

# Lấy mật khẩu admin (nếu không đặt trong values.yaml)
kubectl exec -n jenkins -it \
  $(kubectl get pod -n jenkins -l app.kubernetes.io/name=jenkins -o name) \
  -- cat /var/jenkins_home/secrets/initialAdminPassword

# Truy cập qua port-forward (không cần Ingress)
kubectl port-forward svc/jenkins -n jenkins 8080:8080
# Mở http://localhost:8080
```

### Nâng Cấp Jenkins Helm Chart

```bash
# Cập nhật repo
helm repo update

# Xem phiên bản mới nhất
helm search repo jenkins/jenkins --versions | head -5

# Nâng cấp (upgrade)
helm upgrade jenkins jenkins/jenkins \
  --namespace jenkins \
  --values jenkins-values.yaml \
  --set controller.tag=2.452.3-jdk17

# Rollback nếu có lỗi
helm rollback jenkins 1 --namespace jenkins
```

---

## 4. Cấu Hình Sau Cài Đặt (Post-Install)

Sau khi cài xong, thực hiện các bước này trước khi đưa vào sử dụng:

### Kiểm Tra Danh Sách Plugin Cần Thiết

```
Pipeline: Aggregator         — Core pipeline plugin
Git                          — Tích hợp Git
Credentials Binding          — Inject credentials vào build
Blue Ocean                   — Giao diện hiện đại
Matrix Authorization Strategy — Phân quyền chi tiết
Role-based Authorization     — Phân quyền theo vai trò
Kubernetes                   — Dynamic agent trên K8s
Docker Pipeline              — Build và push Docker image
Slack Notification           — Thông báo Slack
```

### Cấu Hình Jenkins URL

Quan trọng khi dùng Webhook: Jenkins cần URL chính xác để nhận callback.

```
Manage Jenkins → System → Jenkins URL
→ Nhập: https://jenkins.example.com (URL public)
```

### Tắt Agent-to-Master Security Bypass (Bảo Mật)

```
Manage Jenkins → Security → Agent → Controller Security
→ Bật "Enable Agent → Controller Access Control"
```

---

## 5. So Sánh Ba Phương Thức Cài Đặt

| Tiêu Chí | Standalone | Docker | Kubernetes |
|---------|-----------|--------|------------|
| **Dễ cài** | ✅ Đơn giản | ✅ Nhanh | ⚠️ Phức tạp hơn |
| **Isolation** | ❌ Chia sẻ OS | ✅ Container riêng | ✅ Pod riêng |
| **Scalability** | ❌ Khó scale | ⚠️ Giới hạn | ✅ Auto-scale |
| **HA** (High Availability) | ❌ Không có | ⚠️ Cần cấu hình thêm | ✅ K8s tự xử lý |
| **Backup** | Manual | Volume backup | PVC snapshot |
| **Phù hợp** | Lab, học | Dev, demo | Production |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao nên dùng Kubernetes để chạy Jenkins trong production?**

A: Kubernetes mang lại dynamic agent — agent được tạo khi có build và xóa ngay sau khi xong, không lãng phí tài nguyên khi idle. Controller chạy trong Pod được K8s tự động restart khi crash (self-healing). PVC đảm bảo dữ liệu JENKINS_HOME an toàn khi Pod được reschedule. Kết hợp Helm và JCasC (Jenkins Configuration as Code), toàn bộ cấu hình Jenkins được quản lý dưới dạng code, dễ reproducible và audit.

**Q: Port 50000 trong Jenkins dùng để làm gì?**

A: Port 50000 là cổng JNLP (Java Network Launch Protocol) — Agent dùng port này để kết nối vào Controller. Khi Agent khởi động, nó mở kết nối TCP đến `controller:50000` để nhận lệnh và báo cáo kết quả build. Nếu dùng SSH Agent thay JNLP, port này không cần thiết. Trong Kubernetes, port này vẫn cần nếu dùng inbound agent.
