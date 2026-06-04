# Docker Agents — Agent Trong Docker Container

> **Docker Agent** cho phép Jenkins chạy build bên trong Docker container (vùng chứa Docker) tạm thời. Container được tạo khi build bắt đầu và tự động bị xóa sau khi hoàn thành — đảm bảo môi trường build sạch sẽ, tái lập được (reproducible), và không có side-effect (tác dụng phụ) giữa các build.

---

## Mục Lục

1. [Tại Sao Dùng Docker Agent](#tại-sao-dùng-docker-agent)
2. [Yêu Cầu Cài Đặt](#yêu-cầu-cài-đặt)
3. [Cú Pháp Khai Báo Docker Agent](#cú-pháp-khai-báo-docker-agent)
4. [Dockerfile Agent — Agent Từ Dockerfile Tùy Chỉnh](#dockerfile-agent--agent-từ-dockerfile-tùy-chỉnh)
5. [Registry Authentication — Xác Thực Registry](#registry-authentication--xác-thực-registry)
6. [Docker-in-Docker vs Docker Socket](#docker-in-docker-vs-docker-socket)
7. [Multi-Stage Pipeline với Docker Agent](#multi-stage-pipeline-với-docker-agent)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Dùng Docker Agent

### Vấn Đề Với Static Agent

```
# Static Agent — Vấn đề điển hình:

Agent Node duy nhất
    ├── Project A cần Java 11
    ├── Project B cần Java 17
    ├── Project C cần Node.js 18
    └── Project D cần Python 3.9

→ Xung đột phiên bản (version conflict)
→ "Works on my machine" syndrome
→ Phụ thuộc vào cấu hình thủ công của agent
```

### Docker Agent Giải Quyết Thế Nào

```
# Docker Agent — Môi trường độc lập:

Build Project A → Container: openjdk:11     ← Java 11 riêng
Build Project B → Container: openjdk:17     ← Java 17 riêng
Build Project C → Container: node:18        ← Node.js 18 riêng
Build Project D → Container: python:3.9     ← Python 3.9 riêng

→ Không xung đột
→ Môi trường giống hệt nhau mỗi lần build
→ Không cần cấu hình thủ công agent
```

### Luồng Hoạt Động Docker Agent

```
Jenkins Pipeline bắt đầu
        │
        ▼
Jenkins gọi Docker daemon: "Tạo container từ image X"
        │
        ▼
Docker pull image (nếu chưa có local)
        │
        ▼
Container khởi động, Jenkins agent.jar được inject vào
        │
        ▼
Build chạy bên trong container
        │
        ▼
Build hoàn thành → Container bị xóa tự động
```

---

## Yêu Cầu Cài Đặt

### Cài Docker Pipeline Plugin

```
Manage Jenkins → Plugins → Available Plugins
→ Search: "Docker Pipeline"
→ Install without restart
```

### Jenkins Controller Cần Truy Cập Docker Daemon

**Cách 1: Docker socket mount (trên máy chạy Jenkins)**

```bash
# Cấp quyền user jenkins truy cập Docker socket
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**Cách 2: Jenkins Agent có cài Docker**

```bash
# Trên Agent node, cài Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker jenkins
```

**Cách 3: Docker daemon từ xa (Docker TCP)**

```
Manage Jenkins → Clouds → Add a new cloud → Docker
→ Docker Host URI: tcp://docker-host:2376
→ Credentials: (TLS certificate nếu cần)
```

---

## Cú Pháp Khai Báo Docker Agent

### Dùng Image Sẵn Có Từ Docker Hub

```groovy
pipeline {
    // Cả pipeline chạy trong container node:18-alpine
    agent {
        docker {
            image 'node:18-alpine'
            // Tùy chọn thêm:
            args  '-v /tmp:/tmp'           // Mount volume
            label 'docker'                 // Chọn agent node có Docker
        }
    }

    stages {
        stage('Install') {
            steps {
                sh 'node --version'
                sh 'npm ci'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
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

### Truyền Biến Môi Trường và Volume

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '''
                -v $HOME/.m2:/root/.m2
                -e MAVEN_OPTS="-Xmx1024m"
                --memory=2g
                --cpus=2
            '''
        }
    }

    stages {
        stage('Build') {
            steps {
                // Maven cache được mount từ host → không cần download lại mỗi lần
                sh 'mvn clean package -DskipTests'
            }
        }
    }
}
```

> **Lưu ý:** Mount thư mục Maven cache (`~/.m2`) từ host vào container giúp tái sử dụng dependency đã tải, giảm thời gian build đáng kể.

### Agent Khác Nhau Cho Từng Stage

```groovy
pipeline {
    agent none  // Không có agent mặc định

    stages {
        stage('Test — Java') {
            agent { docker { image 'openjdk:17-slim' } }
            steps {
                sh 'java -version'
                sh './gradlew test'
            }
        }

        stage('Lint — NodeJS') {
            agent { docker { image 'node:20-alpine' } }
            steps {
                sh 'npm ci && npm run lint'
            }
        }

        stage('Security Scan') {
            agent { docker { image 'owasp/dependency-check:latest' } }
            steps {
                sh 'dependency-check.sh --project myapp --scan .'
            }
        }

        stage('Build Docker Image') {
            // Stage này cần Docker-in-Docker hoặc Docker socket
            agent { label 'docker' }
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
    }
}
```

---

## Dockerfile Agent — Agent Từ Dockerfile Tùy Chỉnh

Khi không có image sẵn phù hợp, bạn có thể định nghĩa môi trường build bằng một Dockerfile tùy chỉnh ngay trong repository.

### Cấu Trúc Thư Mục

```
project-root/
├── Jenkinsfile
├── ci/
│   └── Dockerfile.build     ← Dockerfile cho build environment
├── src/
└── ...
```

### Dockerfile Build Environment

```dockerfile
# ci/Dockerfile.build
FROM ubuntu:22.04

# Cài đặt các dependency cần thiết cho build
RUN apt-get update && apt-get install -y \
    openjdk-17-jdk \
    maven \
    nodejs \
    npm \
    git \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Cấu hình Maven
ENV MAVEN_OPTS="-Xmx1024m -XX:+UseG1GC"

# Tạo user không có quyền root để chạy build (best practice bảo mật)
RUN useradd -m -u 1000 build
USER build
WORKDIR /home/build
```

### Jenkinsfile Dùng Dockerfile Agent

```groovy
pipeline {
    agent {
        dockerfile {
            filename 'ci/Dockerfile.build'    // Đường dẫn Dockerfile
            dir      '.'                       // Thư mục build context
            label    'docker'                  // Agent cần cài Docker
            // Build args cho Dockerfile
            additionalBuildArgs '--build-arg VERSION=1.0'
            // Tùy chọn cho container
            args '-v $HOME/.m2:/home/build/.m2'
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

### Cache Docker Image Build

```groovy
pipeline {
    agent {
        dockerfile {
            filename 'ci/Dockerfile.build'
            label    'docker'
            // Pull từ registry để dùng như cache layer
            additionalBuildArgs '--cache-from myregistry/build-env:latest'
        }
    }
    // ...
}
```

---

## Registry Authentication — Xác Thực Registry

Khi dùng image từ private registry (registry riêng tư) như Docker Hub private repo, AWS ECR (Elastic Container Registry), hoặc GitLab Registry, cần xác thực.

### Lưu Registry Credentials Vào Jenkins

```
Manage Jenkins → Credentials → System → Global credentials
→ Add Credentials
   Kind:        Username with password
   ID:          docker-hub-credentials
   Description: Docker Hub login
   Username:    <docker-hub-username>
   Password:    <docker-hub-password-or-access-token>
→ OK
```

### Dùng Credentials Trong Pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'myregistry.example.com/private/build-env:latest'
            registryUrl        'https://myregistry.example.com'
            registryCredentialsId 'registry-credentials'  // ID trong Jenkins Credentials
            label              'docker'
        }
    }

    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
}
```

### AWS ECR — Elastic Container Registry

```groovy
pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '123456789012'
        AWS_REGION     = 'ap-southeast-1'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME     = 'myapp'
    }

    stages {
        stage('Build & Push') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-credentials']]) {
                    script {
                        // Login vào ECR trước khi pull/push
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        """

                        // Build và push image
                        sh """
                            docker build -t ${ECR_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} .
                            docker push ${ECR_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }
    }
}
```

---

## Docker-in-Docker vs Docker Socket

Khi Pipeline cần **build Docker image** (không chỉ chạy trong Docker), có hai cách tiếp cận với trade-off khác nhau.

### Cách 1: Docker Socket Mount (Khuyến Nghị)

Mount file socket Docker từ host vào container — container giao tiếp trực tiếp với Docker daemon của host.

```groovy
pipeline {
    agent {
        docker {
            image 'docker:24-cli'  // Image chứa Docker CLI
            // Mount Docker socket từ host vào container
            args  '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {
        stage('Build Image') {
            steps {
                // Container này có thể chạy docker commands
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker push myregistry/myapp:${BUILD_NUMBER}'
            }
        }
    }
}
```

**Ưu điểm:** Đơn giản, hiệu suất tốt, image được cache lại trên host.

**Nhược điểm:** Container có quyền root trên Docker daemon của host — **rủi ro bảo mật** nếu Jenkinsfile không đáng tin cậy.

### Cách 2: DinD — Docker-in-Docker (Cô Lập Hoàn Toàn)

Chạy Docker daemon bên trong container, hoàn toàn tách biệt với host.

```groovy
pipeline {
    agent {
        docker {
            image  'docker:24-dind'    // Image có Docker daemon
            args   '--privileged'      // Cần quyền privileged để chạy daemon
        }
    }

    stages {
        stage('Build Image') {
            steps {
                sh 'dockerd &'         // Khởi động Docker daemon trong container
                sh 'sleep 3'           // Chờ daemon sẵn sàng
                sh 'docker build -t myapp .'
            }
        }
    }
}
```

**Ưu điểm:** Cô lập hoàn toàn, an toàn hơn về bảo mật.

**Nhược điểm:** Cần `--privileged`, không cache image giữa các build, chậm hơn.

### Cách 3: Kaniko — Build Image Không Cần Docker Daemon

**Kaniko** là công cụ build Docker image mà không cần Docker daemon — phù hợp nhất cho môi trường Kubernetes.

```groovy
pipeline {
    agent {
        docker {
            image 'gcr.io/kaniko-project/executor:latest'
            args  '--entrypoint=""'  // Override entrypoint để chạy lệnh tùy chỉnh
        }
    }

    stages {
        stage('Build & Push') {
            steps {
                withCredentials([file(credentialsId: 'kaniko-secret',
                                      variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        /kaniko/executor \
                          --context . \
                          --dockerfile Dockerfile \
                          --destination gcr.io/myproject/myapp:${BUILD_NUMBER} \
                          --cache=true
                    '''
                }
            }
        }
    }
}
```

### So Sánh Ba Cách

| Tiêu Chí | Docker Socket | Docker-in-Docker | Kaniko |
|----------|--------------|------------------|--------|
| **Bảo mật** | Thấp (quyền root host) | Trung bình (privileged) | Cao (không cần daemon) |
| **Hiệu suất** | Cao (cache host) | Thấp (không cache) | Trung bình |
| **Độ phức tạp** | Thấp | Cao | Trung bình |
| **Phù hợp cho** | Dev, team nhỏ | Môi trường cô lập | Kubernetes production |

---

## Multi-Stage Pipeline với Docker Agent

### Pipeline Hoàn Chỉnh CI/CD Với Docker Agent

```groovy
pipeline {
    agent none  // Không có agent mặc định — mỗi stage tự chọn

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME      = 'myapp'
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }

    stages {
        // Stage 1: Chạy Unit Test trong môi trường Java
        stage('Unit Tests') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args  '-v $HOME/.m2:/root/.m2'  // Cache Maven deps
                    label 'docker'
                }
            }
            steps {
                sh 'mvn test'
                // Lưu kết quả test để xem sau
                junit 'target/surefire-reports/*.xml'
            }
        }

        // Stage 2: Build JAR artifact
        stage('Build JAR') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args  '-v $HOME/.m2:/root/.m2'
                    label 'docker'
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
                // Lưu artifact để dùng ở stage sau
                stash includes: 'target/*.jar', name: 'app-jar'
            }
        }

        // Stage 3: Phân tích bảo mật (Security Scan)
        stage('Security Scan') {
            agent {
                docker {
                    image 'owasp/dependency-check:latest'
                    args  '-v odc-data:/usr/share/dependency-check/data'
                    label 'docker'
                }
            }
            steps {
                unstash 'app-jar'
                sh '''
                    /usr/share/dependency-check/bin/dependency-check.sh \
                      --project myapp \
                      --scan target/*.jar \
                      --format JSON \
                      --out dependency-check-report.json
                '''
                archiveArtifacts 'dependency-check-report.json'
            }
        }

        // Stage 4: Build Docker Image (cần Docker socket)
        stage('Build Docker Image') {
            agent { label 'docker' }  // Agent có Docker cài sẵn
            steps {
                unstash 'app-jar'
                sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        // Stage 5: Push lên Registry
        stage('Push Image') {
            agent { label 'docker' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId:  'registry-credentials',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh """
                        echo \$REGISTRY_PASS | docker login ${DOCKER_REGISTRY} \
                          -u \$REGISTRY_USER --password-stdin
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                                   ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            // Dọn dẹp image cũ trên agent sau mỗi build
            node('docker') {
                sh "docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} || true"
            }
        }
    }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Docker Agent khác gì Static Agent? Khi nào nên dùng Docker Agent?**

> Static Agent là máy vật lý/VM luôn chạy, môi trường cố định và có thể bị "nhiễm" bởi các build trước. Docker Agent tạo container mới mỗi build — môi trường hoàn toàn sạch, tái lập được, và tự xóa sau khi build xong. Dùng Docker Agent khi: cần nhiều môi trường khác nhau (Java 11, 17, Node 18...), muốn đảm bảo build reproducible, hoặc không muốn quản lý nhiều Static Agent riêng lẻ.

**Q: Docker-in-Docker và Docker Socket Mount khác nhau như thế nào? Cái nào an toàn hơn?**

> Docker Socket Mount cho container dùng chung Docker daemon của host qua `/var/run/docker.sock` — đơn giản, hiệu suất cao, nhưng container có quyền root trên host daemon, là rủi ro bảo mật. Docker-in-Docker chạy daemon riêng trong container (cần `--privileged`) — cô lập tốt hơn nhưng chậm hơn, không cache được image. Kaniko an toàn nhất vì không cần daemon, phù hợp cho Kubernetes production.

**Q: Làm thế nào để tăng tốc Docker Agent build bằng caching?**

> Ba chiến lược: **(1) Mount Maven/npm cache từ host** (`-v $HOME/.m2:/root/.m2`) để không download lại dependency; **(2) Pull image với `--cache-from`** khi dùng Dockerfile agent, tận dụng layer cache từ registry; **(3) Dùng Docker socket mount** thay vì DinD để tận dụng image cache sẵn có trên host agent. Với Kubernetes, dùng PersistentVolumeClaim (PVC — Yêu Cầu Lưu Trữ Bền Vững) để lưu cache giữa các build.

**Q: Cách xác thực với private registry trong Jenkins Pipeline?**

> Lưu credentials (username/password hoặc token) vào Jenkins Credentials Store. Trong Pipeline khai báo Docker Agent: dùng `registryUrl` và `registryCredentialsId`. Với AWS ECR: dùng `withCredentials` để lấy AWS credentials rồi chạy `aws ecr get-login-password` để lấy temporary token. Không bao giờ hardcode (mã hóa cứng) password trong Jenkinsfile.

---

**Liên Kết Liên Quan:**
- [1-agent-configuration.md](1-agent-configuration.md) — SSH và JNLP Static Agent
- [3-kubernetes-agents.md](3-kubernetes-agents.md) — Kubernetes Pod Agent (bước tiến hóa từ Docker Agent)
- [README.md](README.md) — Tổng quan Distributed Builds

**Cập Nhật:** 2026-05-11
