# Kubernetes Agents — Agent Động Trên Kubernetes

> **Kubernetes Agent** (tác nhân Kubernetes) là cơ chế Jenkins tự động tạo Pod (đơn vị chạy container nhỏ nhất trên Kubernetes) để thực thi build, và xóa Pod đó ngay sau khi hoàn thành. Đây là phương pháp **cloud-native** (bản địa đám mây) tiên tiến nhất, cho phép Jenkins scale (mở rộng) tự động theo nhu cầu thực tế.

---

## Mục Lục

1. [Tại Sao Dùng Kubernetes Agent](#tại-sao-dùng-kubernetes-agent)
2. [Cài Đặt Kubernetes Plugin](#cài-đặt-kubernetes-plugin)
3. [Pod Template — Khuôn Mẫu Pod](#pod-template--khuôn-mẫu-pod)
4. [Jenkinsfile Với Kubernetes Agent](#jenkinsfile-với-kubernetes-agent)
5. [Sidecar Containers — Container Phụ Trợ](#sidecar-containers--container-phụ-trợ)
6. [JCasC — Jenkins Configuration as Code](#jcasc--jenkins-configuration-as-code)
7. [Persistent Volume và Cache](#persistent-volume-và-cache)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Dùng Kubernetes Agent

### So Sánh Static Agent vs Kubernetes Agent

```
┌─────────────────────────────────────────────────────────────┐
│ Static Agent                                                  │
│                                                               │
│  Trả tiền 24/7 cho 10 agent VM                               │
│  Dù ban đêm không có build                                    │
│  Dù cuối tuần 0 developer làm việc                          │
│                                                               │
│  Chi phí: cố định, lãng phí khi idle (nhàn rỗi)             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Kubernetes Agent                                              │
│                                                               │
│  Buổi sáng: 50 developer push code → 50 Pod được tạo        │
│  Buổi tối: không có build → 0 Pod, 0 chi phí agent          │
│  Sau release: 200 build cùng lúc → 200 Pod tạm thời         │
│                                                               │
│  Chi phí: theo nhu cầu thực tế (pay-per-use)                │
└─────────────────────────────────────────────────────────────┘
```

### Ưu Điểm Kubernetes Agent

- **Tự động scale** — tạo Pod khi cần, xóa ngay khi xong; không giới hạn số lượng đồng thời
- **Cô lập hoàn toàn** — mỗi build có Pod riêng, namespace riêng (tùy cấu hình)
- **Tận dụng tài nguyên K8s** — tích hợp với node scheduling, resource quota, limit range
- **Môi trường sạch** — Pod bị xóa sau build, không có "dirt" (rác) từ build trước
- **Declarative** — cấu hình dưới dạng YAML, lưu trong Git, dễ kiểm soát phiên bản

---

## Cài Đặt Kubernetes Plugin

### Bước 1 — Cài Plugin

```
Manage Jenkins → Plugins → Available Plugins
→ Search: "Kubernetes"
→ Install: "Kubernetes" (kubernetes plugin chính thức của Jenkins)
→ Install without restart
```

### Bước 2 — Tạo Service Account Trên K8s

Jenkins cần một Service Account (tài khoản dịch vụ) trên Kubernetes để tạo/xóa Pod.

```yaml
# jenkins-sa.yaml — Service Account và RBAC cho Jenkins
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins-agents

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-agent-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-agent-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-agent-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
```

```bash
kubectl apply -f jenkins-sa.yaml
```

### Bước 3 — Cấu Hình Kubernetes Cloud Trên Jenkins

```
Manage Jenkins → Clouds → Add a new cloud → Kubernetes
→ Kubernetes URL:        https://kubernetes.default.svc   (nếu Jenkins chạy trong K8s)
                         hoặc https://<k8s-api-server>:6443
→ Kubernetes Namespace:  jenkins-agents
→ Jenkins URL:           http://jenkins.jenkins.svc.cluster.local:8080
→ Jenkins tunnel:        jenkins.jenkins.svc.cluster.local:50000

→ Credentials: (Service Account token hoặc kubeconfig)
→ Test Connection → "Connected to Kubernetes v1.28..."
→ Save
```

> **Nếu Jenkins chạy trong K8s:** Kubernetes URL để trống — plugin tự phát hiện qua `KUBERNETES_SERVICE_HOST`.

---

## Pod Template — Khuôn Mẫu Pod

**Pod Template** (khuôn mẫu Pod) định nghĩa cấu hình của Pod sẽ được tạo: image nào, tài nguyên bao nhiêu, volume nào được mount, biến môi trường...

### Pod Template Đơn Giản

```yaml
# Pod Template dạng YAML — có thể đặt trong Jenkinsfile
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp                          # Container JNLP (bắt buộc) — agent kết nối vào Controller
      image: jenkins/inbound-agent:latest
      resources:
        requests:
          memory: "256Mi"
          cpu:    "100m"
        limits:
          memory: "512Mi"
          cpu:    "500m"

    - name: maven                         # Container chạy build Maven
      image: maven:3.9-eclipse-temurin-17
      command: ["sleep"]
      args:    ["infinity"]               # Giữ container chạy để nhận lệnh từ Jenkins
      resources:
        requests:
          memory: "512Mi"
          cpu:    "250m"
        limits:
          memory: "2Gi"
          cpu:    "1"
```

### Dùng Pod Template Trong Jenkinsfile

```groovy
pipeline {
    agent {
        kubernetes {
            // Định nghĩa Pod Template inline (trực tiếp trong Jenkinsfile)
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: jnlp
                      image: jenkins/inbound-agent:latest
                      resources:
                        requests:
                          memory: "256Mi"
                          cpu: "100m"
                    - name: maven
                      image: maven:3.9-eclipse-temurin-17
                      command: [sleep]
                      args: [infinity]
                      resources:
                        requests:
                          memory: "1Gi"
                          cpu: "500m"
                        limits:
                          memory: "2Gi"
                          cpu: "1"
            '''
            defaultContainer 'maven'      // Container mặc định khi chạy steps
        }
    }

    stages {
        stage('Build') {
            steps {
                // Chạy trong container 'maven' (defaultContainer)
                sh 'mvn clean package -DskipTests'
            }
        }
    }
}
```

### Chỉ Định Container Cụ Thể

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: jnlp
                      image: jenkins/inbound-agent:latest
                    - name: maven
                      image: maven:3.9-eclipse-temurin-17
                      command: [sleep]
                      args: [infinity]
                    - name: node
                      image: node:20-alpine
                      command: [sleep]
                      args: [infinity]
                    - name: kubectl
                      image: bitnami/kubectl:latest
                      command: [sleep]
                      args: [infinity]
            '''
        }
    }

    stages {
        stage('Build Java') {
            steps {
                container('maven') {     // Chỉ định rõ container
                    sh 'mvn clean package -DskipTests'
                    stash includes: 'target/*.jar', name: 'jar'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                container('node') {      // Chuyển sang container node
                    sh 'npm ci && npm run build'
                    stash includes: 'dist/**', name: 'frontend'
                }
            }
        }

        stage('Deploy') {
            steps {
                container('kubectl') {   // Dùng kubectl để deploy
                    sh 'kubectl apply -f k8s/'
                    sh 'kubectl rollout status deployment/myapp'
                }
            }
        }
    }
}
```

---

## Sidecar Containers — Container Phụ Trợ

**Sidecar** (container chạy cạnh container chính) là pattern (mẫu thiết kế) dùng để cung cấp dịch vụ cho build — ví dụ: database, cache, hoặc proxy.

### Ví Dụ: Integration Test Với PostgreSQL

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: jnlp
                      image: jenkins/inbound-agent:latest

                    - name: test-runner
                      image: maven:3.9-eclipse-temurin-17
                      command: [sleep]
                      args: [infinity]
                      env:
                        # Trỏ đến PostgreSQL sidecar (localhost vì cùng Pod)
                        - name: DB_HOST
                          value: localhost
                        - name: DB_PORT
                          value: "5432"
                        - name: DB_NAME
                          value: testdb
                        - name: DB_USER
                          value: testuser
                        - name: DB_PASS
                          value: testpass

                    # Sidecar: PostgreSQL chỉ tồn tại trong thời gian build
                    - name: postgres
                      image: postgres:15-alpine
                      env:
                        - name: POSTGRES_DB
                          value: testdb
                        - name: POSTGRES_USER
                          value: testuser
                        - name: POSTGRES_PASSWORD
                          value: testpass
                      resources:
                        requests:
                          memory: "256Mi"
                          cpu: "100m"
                        limits:
                          memory: "512Mi"
                          cpu: "500m"
            '''
        }
    }

    stages {
        stage('Integration Tests') {
            steps {
                container('test-runner') {
                    sh '''
                        # Chờ PostgreSQL sẵn sàng
                        until pg_isready -h localhost -p 5432; do sleep 1; done
                        # Chạy integration test
                        mvn verify -Pintegration-tests
                    '''
                }
            }
        }
    }
}
```

### Ví Dụ: Build Với Docker-in-Docker Sidecar

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: jnlp
                      image: jenkins/inbound-agent:latest

                    - name: builder
                      image: docker:24-cli
                      command: [sleep]
                      args: [infinity]
                      env:
                        # Trỏ đến Docker daemon sidecar
                        - name: DOCKER_HOST
                          value: tcp://localhost:2375

                    # Sidecar: Docker daemon (DinD — Docker-in-Docker)
                    - name: dind
                      image: docker:24-dind
                      securityContext:
                        privileged: true          # Bắt buộc cho DinD
                      env:
                        - name: DOCKER_TLS_CERTDIR
                          value: ""               # Tắt TLS cho đơn giản (chỉ trong K8s nội bộ)
            '''
        }
    }

    stages {
        stage('Build Image') {
            steps {
                container('builder') {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                    sh 'docker push myregistry/myapp:${BUILD_NUMBER}'
                }
            }
        }
    }
}
```

---

## JCasC — Jenkins Configuration as Code

**JCasC** (Jenkins Configuration as Code — Cấu Hình Jenkins Dưới Dạng Code) cho phép định nghĩa toàn bộ cấu hình Jenkins (bao gồm Kubernetes Cloud và Pod Template) trong file YAML — lưu được trong Git, tái sử dụng được, không cần click UI.

### Cài Đặt JCasC Plugin

```
Manage Jenkins → Plugins → Available Plugins
→ Search: "Configuration as Code"
→ Install without restart
```

### File casc.yaml — Cấu Hình Kubernetes Cloud

```yaml
# /var/jenkins_home/casc.yaml
jenkins:
  clouds:
    - kubernetes:
        name:                  "kubernetes"
        serverUrl:             "https://kubernetes.default.svc"
        namespace:             "jenkins-agents"
        jenkinsUrl:            "http://jenkins.jenkins.svc.cluster.local:8080"
        jenkinsTunnel:         "jenkins.jenkins.svc.cluster.local:50000"
        connectTimeout:        5
        readTimeout:           15
        containerCapStr:       "100"    # Tối đa 100 Pod đồng thời
        maxRequestsPerHostStr: "32"

        templates:
          # Pod Template 1: Build Java Maven
          - name:      "maven-agent"
            label:     "maven k8s"
            namespace: "jenkins-agents"
            nodeUsageMode: NORMAL
            containers:
              - name:    "jnlp"
                image:   "jenkins/inbound-agent:latest"
                alwaysPullImage: false
                resourceRequestCpu:    "100m"
                resourceRequestMemory: "256Mi"
                resourceLimitCpu:      "500m"
                resourceLimitMemory:   "512Mi"

              - name:    "maven"
                image:   "maven:3.9-eclipse-temurin-17"
                command: "sleep"
                args:    "infinity"
                resourceRequestCpu:    "500m"
                resourceRequestMemory: "1Gi"
                resourceLimitCpu:      "2"
                resourceLimitMemory:   "4Gi"

            volumes:
              - persistentVolumeClaim:
                  claimName: "maven-cache-pvc"    # Cache Maven dependencies
                  mountPath: "/root/.m2"
                  readOnly:  false

          # Pod Template 2: Build NodeJS
          - name:      "node-agent"
            label:     "node k8s"
            namespace: "jenkins-agents"
            containers:
              - name:    "jnlp"
                image:   "jenkins/inbound-agent:latest"
                resourceRequestCpu:    "100m"
                resourceRequestMemory: "256Mi"

              - name:    "node"
                image:   "node:20-alpine"
                command: "sleep"
                args:    "infinity"
                resourceRequestCpu:    "250m"
                resourceRequestMemory: "512Mi"
                resourceLimitCpu:      "1"
                resourceLimitMemory:   "2Gi"

            volumes:
              - persistentVolumeClaim:
                  claimName: "npm-cache-pvc"       # Cache npm packages
                  mountPath: "/root/.npm"
                  readOnly:  false
```

### Load JCasC Khi Khởi Động Jenkins

```yaml
# docker-compose.yaml hoặc Kubernetes Deployment cho Jenkins
environment:
  - CASC_JENKINS_CONFIG=/var/jenkins_home/casc.yaml
```

Hoặc qua Manage Jenkins:

```
Manage Jenkins → Configuration as Code
→ Path or URL: /var/jenkins_home/casc.yaml
→ Reload Configuration from Disk
```

---

## Persistent Volume và Cache

Build trong K8s agent thường cần cache (Maven, npm, pip...) để không download lại mỗi lần.

### Tạo PersistentVolumeClaim (PVC — Yêu Cầu Lưu Trữ Bền Vững)

```yaml
# maven-cache-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-cache-pvc
  namespace: jenkins-agents
spec:
  accessModes:
    - ReadWriteMany    # Nhiều Pod có thể đọc/ghi đồng thời
  storageClassName: nfs-storage    # Tùy thuộc vào cluster
  resources:
    requests:
      storage: 20Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: npm-cache-pvc
  namespace: jenkins-agents
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f maven-cache-pvc.yaml
```

### Dùng PVC Trong Pod Template

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: jnlp
                      image: jenkins/inbound-agent:latest
                    - name: maven
                      image: maven:3.9-eclipse-temurin-17
                      command: [sleep]
                      args: [infinity]
                      volumeMounts:
                        - name: maven-cache
                          mountPath: /root/.m2      # Maven local repository
                  volumes:
                    - name: maven-cache
                      persistentVolumeClaim:
                        claimName: maven-cache-pvc  # PVC đã tạo ở trên
            '''
            defaultContainer 'maven'
        }
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'  // Dependencies được cache trong PVC
            }
        }
    }
}
```

---

## Resource Management — Quản Lý Tài Nguyên

### Request vs Limit

| Khái Niệm | Ý Nghĩa | Ảnh Hưởng |
|-----------|---------|-----------|
| **Request** (yêu cầu) | Tài nguyên tối thiểu Pod cần | K8s dùng để lên lịch Pod trên Node |
| **Limit** (giới hạn) | Tài nguyên tối đa Pod được dùng | Vượt quá limit: CPU bị throttle, RAM bị kill |

```yaml
resources:
  requests:
    memory: "512Mi"    # Pod cần ít nhất 512MB RAM để được schedule
    cpu:    "250m"     # 0.25 CPU core
  limits:
    memory: "2Gi"      # Tối đa 2GB RAM; vượt quá → OOMKilled (Out Of Memory)
    cpu:    "1"        # Tối đa 1 CPU core; vượt quá → throttled (bị giới hạn tốc độ)
```

### Namespace Resource Quota (Hạn Ngạch Tài Nguyên)

```yaml
# Giới hạn tổng tài nguyên trong namespace jenkins-agents
apiVersion: v1
kind: ResourceQuota
metadata:
  name: jenkins-agents-quota
  namespace: jenkins-agents
spec:
  hard:
    pods:               "50"       # Tối đa 50 Pod đồng thời
    requests.cpu:       "20"       # Tổng CPU request tối đa 20 cores
    requests.memory:    "40Gi"     # Tổng RAM request tối đa 40GB
    limits.cpu:         "40"
    limits.memory:      "80Gi"
```

---

## Câu Hỏi Phỏng Vấn

**Q: Kubernetes Agent hoạt động như thế nào? Khác gì Docker Agent?**

> Kubernetes Agent dùng Kubernetes Plugin để tạo một Pod trên K8s cluster khi có build request. Pod chứa ít nhất container `jnlp` (kết nối với Controller qua JNLP) và các container build tùy chỉnh. Sau khi build xong, Pod bị xóa. Khác với Docker Agent (chạy container trên một máy host cố định), K8s Agent tận dụng scheduler của Kubernetes để tự động chọn Node phù hợp, scale không giới hạn, và tích hợp với resource quota, namespace isolation của K8s.

**Q: Pod Template là gì? Cách định nghĩa và dùng trong Jenkinsfile?**

> Pod Template định nghĩa "blueprint" (bản thiết kế) của Pod sẽ được tạo: image cho từng container, resource request/limit, volume, biến môi trường. Có thể định nghĩa tập trung trong Kubernetes Cloud config (Manage Jenkins → Clouds) hoặc trực tiếp trong Jenkinsfile bằng `agent { kubernetes { yaml '...' } }`. Dùng `container('tên-container')` trong steps để chỉ định container nào thực thi lệnh.

**Q: JCasC là gì và tại sao quan trọng trong môi trường production?**

> JCasC (Jenkins Configuration as Code) là plugin cho phép toàn bộ cấu hình Jenkins (bao gồm Cloud config, Pod Template, Security settings) được định nghĩa trong file YAML. Quan trọng vì: **(1) GitOps** — cấu hình lưu trong Git, có lịch sử thay đổi, review được qua PR; **(2) Disaster Recovery** — khi Jenkins mất, restore chỉ cần apply lại file YAML; **(3) Consistency** — môi trường dev/staging/production giống hệt nhau; **(4) Không cần click UI** — phù hợp với Infrastructure as Code (IaC — Hạ Tầng Dưới Dạng Code).

**Q: Làm thế nào để tối ưu thời gian khởi động Kubernetes Agent?**

> Bốn chiến lược: **(1) Image pre-pull** — cấu hình `imagePullPolicy: IfNotPresent` thay vì `Always` để không pull image mỗi lần; **(2) Small image** — dùng Alpine hoặc Distroless image thay vì full OS; **(3) Cache dependencies** — dùng PVC mount vào `/root/.m2` hoặc `/root/.npm`; **(4) Warm pool** — cấu hình `containerCap` và giữ một số Pod sẵn sàng trước. Mục tiêu: đưa thời gian từ 60–90 giây xuống còn 15–30 giây.

---

**Liên Kết Liên Quan:**
- [1-agent-configuration.md](1-agent-configuration.md) — SSH và JNLP Static Agent
- [2-docker-agents.md](2-docker-agents.md) — Docker Agent (tiền thân của K8s Agent)
- [4-node-management.md](4-node-management.md) — Quản lý Node và lifecycle
- [README.md](README.md) — Tổng quan Distributed Builds

**Cập Nhật:** 2026-05-11
