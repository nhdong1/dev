# Docker Integration — Build Image và Push Registry Từ Jenkins

> Docker Integration cho phép Jenkins tự động build Docker image, chạy test trong container, và push image lên registry (kho lưu trữ image) như một phần của CI/CD pipeline.

---

## Mục Lục

1. [Tổng Quan Kiến Trúc](#1-tổng-quan-kiến-trúc)
2. [Cài Plugin và Cấu Hình](#2-cài-plugin-và-cấu-hình)
3. [Docker-in-Docker vs Docker Socket](#3-docker-in-docker-vs-docker-socket)
4. [Build và Push Image](#4-build-và-push-image)
5. [Multi-stage Build](#5-multi-stage-build)
6. [Registry Đa Dạng](#6-registry-đa-dạng)
7. [Image Scanning (Quét Bảo Mật Image)](#7-image-scanning-quét-bảo-mật-image)
8. [Best Practices](#8-best-practices)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Kiến Trúc

```
Jenkins Pipeline Stage: Docker Build & Push
──────────────────────────────────────────

Source Code (Dockerfile + App)
        │
        ▼
  Docker Build
  (docker build -t myapp:v1.0 .)
        │
        ▼
  Docker Push
  (docker push registry/myapp:v1.0)
        │
        ├──► Docker Hub        (public/private registry)
        ├──► AWS ECR           (Elastic Container Registry)
        ├──► Google GCR        (Google Container Registry)
        ├──► Azure ACR         (Azure Container Registry)
        └──► Harbor            (self-hosted registry)
```

---

## 2. Cài Plugin và Cấu Hình

### 2.1 Plugin Cần Thiết

```
Manage Jenkins → Plugin Manager → Available plugins
→ Cài các plugin sau:
   ✅ Docker Pipeline        — DSL steps: docker.build(), docker.withRegistry()
   ✅ Docker plugin          — cấu hình Docker daemon cho Jenkins
   ✅ CloudBees Docker Build and Publish (tùy chọn)
```

### 2.2 Cấu Hình Docker Daemon

**Option 1: Docker cài trực tiếp trên Jenkins controller**

```bash
# Trên máy chủ Jenkins (Ubuntu/Debian)
sudo usermod -aG docker jenkins   # thêm user jenkins vào nhóm docker
sudo systemctl restart jenkins    # khởi động lại Jenkins để nhận group mới
```

**Option 2: Jenkins trong Docker Container (Docker-in-Docker)**

```yaml
# docker-compose.yml — Jenkins với Docker socket mount
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    volumes:
      - jenkins_home:/var/jenkins_home
      # Mount Docker socket từ host vào container Jenkins
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8080:8080"
      - "50000:50000"
    user: root  # cần root để truy cập docker.sock
```

---

## 3. Docker-in-Docker vs Docker Socket

Đây là một trong những câu hỏi phỏng vấn phổ biến nhất về Docker + Jenkins.

### 3.1 Docker Socket Mounting (Docker Socket — Gắn Kết Socket)

```
Jenkins Container
  │
  │  /var/run/docker.sock (mount từ host)
  ▼
Host Docker Daemon  ──── tạo container anh em (sibling containers)
  │
  ├─ Container A (build step)
  └─ Container B (test step)
```

**Ưu điểm:**
- Đơn giản, hiệu suất cao (dùng Docker daemon của host)
- Containers được tạo là sibling containers — nằm cùng level với Jenkins container
- Không cần privileged mode (chế độ đặc quyền)

**Nhược điểm:**
- Bảo mật kém hơn — Jenkins container có quyền điều khiển toàn bộ Docker daemon của host
- Nếu pipeline chạy `docker rm -f $(docker ps -aq)` — xóa hết container trên host!
- Không phù hợp với môi trường multi-tenant (nhiều team dùng chung)

### 3.2 Docker-in-Docker — DinD (Docker Bên Trong Docker)

```
Jenkins Container
  │
  │  Docker CLI → kết nối tới
  ▼
DinD Container (docker:dind — chạy Docker daemon riêng)
  │  (cần --privileged flag)
  ├─ Container A (build step)
  └─ Container B (test step)
```

**Ưu điểm:**
- Cô lập hoàn toàn — mỗi Jenkins agent có Docker daemon riêng
- Phù hợp với Kubernetes agent (Kubernetes dùng DinD trong Pod)
- Không ảnh hưởng Docker daemon của host

**Nhược điểm:**
- Cần `--privileged` mode — vẫn có rủi ro bảo mật
- Hiệu suất thấp hơn (overlay on overlay filesystem)
- Phức tạp hơn trong cấu hình

### 3.3 Kaniko (Thay Thế Không Cần Docker Daemon)

```
Kaniko — build Docker image mà không cần Docker daemon
  ✅ Không cần privileged mode
  ✅ An toàn hơn trong Kubernetes
  ✅ Được Google khuyến nghị
  
Cách dùng trong Kubernetes Pod:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--context=git://github.com/myorg/myrepo"
        - "--destination=myregistry/myapp:latest"
```

**So sánh tổng hợp:**

| Phương Pháp | Bảo Mật | Hiệu Suất | Độ Phức Tạp | Phù Hợp |
|-------------|---------|-----------|------------|---------|
| Docker Socket | ⚠️ Thấp | ⭐⭐⭐ Cao | ⭐ Đơn giản | VM-based Jenkins |
| DinD | ⚠️ Trung | ⭐⭐ Trung | ⭐⭐ Trung | Docker Compose |
| Kaniko | ✅ Cao | ⭐⭐ Trung | ⭐⭐⭐ Phức tạp | Kubernetes |

---

## 4. Build và Push Image

### 4.1 Sử Dụng Docker Pipeline DSL (Ngôn Ngữ Đặc Thù Miền)

```groovy
pipeline {
    agent any

    environment {
        // Tên image và tag — dùng BUILD_NUMBER để tạo tag duy nhất
        IMAGE_NAME = 'myorg/myapp'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
        // Credentials ID lưu trong Jenkins Credentials Store
        REGISTRY_CRED = credentials('docker-hub-credentials')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // docker.build() — build image từ Dockerfile trong thư mục hiện tại
                    // Trả về đối tượng dockerImage có thể dùng ở các bước sau
                    dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Test Image') {
            steps {
                script {
                    // Chạy container từ image vừa build để test
                    dockerImage.inside {
                        sh 'python -m pytest tests/ -v'
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // docker.withRegistry() — đăng nhập registry, push, rồi tự logout
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        // Push với tag version
                        dockerImage.push("${IMAGE_TAG}")
                        // Push thêm tag 'latest'
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                // Xóa image local để giải phóng dung lượng đĩa
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }
    }

    post {
        failure {
            // Đảm bảo cleanup ngay cả khi pipeline thất bại
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
```

### 4.2 Sử Dụng Shell Commands (Lệnh Shell Trực Tiếp)

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'myorg/myapp'
        IMAGE_TAG  = "${env.GIT_COMMIT[0..7]}"  // 8 ký tự đầu của commit hash
    }

    stages {
        stage('Build') {
            steps {
                sh """
                    docker build \
                        --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                        --build-arg VCS_REF=${env.GIT_COMMIT} \
                        --build-arg VERSION=${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -f docker/Dockerfile \
                        .
                """
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker logout
                    '''
                }
            }
        }
    }
}
```

### 4.3 Build Argument và Label (Đối Số Build Và Nhãn)

```dockerfile
# Dockerfile — nhận build arguments từ Jenkins
FROM python:3.11-slim

ARG BUILD_DATE
ARG VCS_REF
ARG VERSION

# OCI Image Labels — metadata chuẩn cho Docker image
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.source="https://github.com/myorg/myapp"

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

## 5. Multi-stage Build

### 5.1 Multi-stage Dockerfile (Dockerfile Đa Giai Đoạn)

Multi-stage build giúp tạo image nhỏ hơn bằng cách tách giai đoạn build và runtime:

```dockerfile
# Stage 1: Builder — chứa đầy đủ build tools
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
# Download dependencies trước để tận dụng Docker layer cache
RUN mvn dependency:go-offline -B
COPY src/ src/
RUN mvn package -DskipTests -B

# Stage 2: Runtime — image nhỏ gọn chỉ có runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Chỉ copy artifact từ builder stage — không copy source code hay build tools
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 5.2 Pipeline Cho Multi-stage Build

```groovy
pipeline {
    agent {
        docker {
            // Dùng Docker agent — không cần Docker cài trên controller
            image 'docker:24-dind'
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {
        stage('Build Multi-stage Image') {
            steps {
                sh """
                    docker build \
                        --target builder \
                        --cache-from ${IMAGE_NAME}:builder-cache \
                        -t ${IMAGE_NAME}:builder-cache \
                        .
                    docker build \
                        --cache-from ${IMAGE_NAME}:builder-cache \
                        --cache-from ${IMAGE_NAME}:latest \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        .
                """
            }
        }
    }
}
```

---

## 6. Registry Đa Dạng

### 6.1 AWS ECR (Elastic Container Registry — Kho Lưu Trữ Container AWS)

```groovy
pipeline {
    agent any

    environment {
        AWS_REGION      = 'ap-southeast-1'
        AWS_ACCOUNT_ID  = '123456789012'
        ECR_REGISTRY    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO        = 'myapp'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }

        stage('Build & Push to ECR') {
            steps {
                sh """
                    docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} .
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }
    }
}
```

### 6.2 Google GCR (Google Container Registry — Kho Lưu Trữ Container Google)

```groovy
stage('Push to GCR') {
    steps {
        withCredentials([file(credentialsId: 'gcp-service-account-key', variable: 'GCP_KEY')]) {
            sh """
                cat ${GCP_KEY} | docker login \
                    -u _json_key \
                    --password-stdin \
                    https://gcr.io
                docker push gcr.io/${GCP_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
            """
        }
    }
}
```

### 6.3 Harbor (Self-hosted Registry — Registry Tự Quản Lý)

```groovy
docker.withRegistry('https://harbor.mycompany.com', 'harbor-credentials') {
    dockerImage.push("${IMAGE_TAG}")
    dockerImage.push('latest')
}
```

---

## 7. Image Scanning (Quét Bảo Mật Image)

### 7.1 Trivy — Quét Lỗ Hổng Bảo Mật

```groovy
stage('Security Scan') {
    steps {
        // Trivy — công cụ quét CVE (Common Vulnerabilities and Exposures — Lỗ Hổng Bảo Mật Phổ Biến)
        sh """
            docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy:latest image \
                --exit-code 1 \
                --severity HIGH,CRITICAL \
                --no-progress \
                ${IMAGE_NAME}:${IMAGE_TAG}
        """
    }
}
```

### 7.2 Tích Hợp Kết Quả Scan Vào Pipeline

```groovy
stage('Security Scan') {
    steps {
        script {
            def scanResult = sh(
                script: """
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v ${WORKSPACE}:/output \
                        aquasec/trivy:latest image \
                        --format json \
                        --output /output/trivy-report.json \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """,
                returnStatus: true
            )

            // Archive report để xem sau
            archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true

            if (scanResult != 0) {
                currentBuild.result = 'UNSTABLE'
                echo "WARNING: Security vulnerabilities found — check trivy-report.json"
            }
        }
    }
}
```

---

## 8. Best Practices

### Tagging Strategy (Chiến Lược Đặt Tag)

```bash
# Không bao giờ dùng mỗi 'latest' — không truy vết được
❌  myapp:latest

# Dùng kết hợp để truy vết và tiện deploy
✅  myapp:1.2.3              → Semantic version (phiên bản ngữ nghĩa)
✅  myapp:git-abc1234f       → Git commit hash (mã commit)
✅  myapp:build-42           → Jenkins build number (số build)
✅  myapp:2026-05-11         → Date-based tag (tag theo ngày)
```

### Build Cache (Bộ Nhớ Đệm Build)

```groovy
// Tận dụng BuildKit (bộ xây dựng nâng cao của Docker) để cache tốt hơn
environment {
    DOCKER_BUILDKIT = '1'
}

sh """
    docker build \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        --cache-from ${IMAGE_NAME}:latest \
        -t ${IMAGE_NAME}:${IMAGE_TAG} \
        .
"""
```

### Image Cleanup (Dọn Dẹp Image)

```groovy
post {
    always {
        // Dọn dẹp image sau khi build xong để tránh đầy đĩa
        sh "docker image prune -f --filter 'dangling=true'"
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
    }
}
```

### Không Lưu Secret Trong Image

```dockerfile
# ❌ SAI — secret bị lưu trong layer history của image
RUN echo "DB_PASSWORD=secret" >> /etc/environment

# ✅ ĐÚNG — truyền secret qua environment variable lúc runtime
# Trong docker run:
#   docker run -e DB_PASSWORD=$DB_PASSWORD myapp:latest

# ✅ ĐÚNG — dùng Docker BuildKit secret (bí mật BuildKit)
RUN --mount=type=secret,id=db_password \
    DB_PASSWORD=$(cat /run/secrets/db_password) && \
    python setup.py
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Docker-in-Docker và Docker socket mounting khác nhau như thế nào?**

> Docker socket mounting (`-v /var/run/docker.sock:/var/run/docker.sock`) cho phép Jenkins container điều khiển Docker daemon của host trực tiếp, tạo sibling containers. Đơn giản và hiệu suất cao nhưng kém bảo mật vì Jenkins có thể làm bất cứ điều gì trên host. Docker-in-Docker (DinD) chạy Docker daemon riêng trong container, cô lập hoàn toàn nhưng cần `--privileged` mode và có overhead về hiệu suất. Trong Kubernetes, Kaniko là lựa chọn tốt nhất vì không cần Docker daemon.

**Q: Làm thế nào để lưu Docker credentials an toàn trong Jenkins?**

> Lưu credentials trong Jenkins Credentials Store loại "Username with password", không bao giờ hardcode trong Jenkinsfile. Dùng `docker.withRegistry()` hoặc `withCredentials()` để inject credentials vào environment variables — Jenkins tự động mask chúng trong logs. Với cloud registries như ECR, dùng IAM Role thay vì access key.

**Q: Cách tối ưu thời gian Docker build trong CI?**

> Có ba chiến lược chính: (1) Layer caching — sắp xếp Dockerfile để các layer ít thay đổi (như `pip install`) đặt trước, layer thường thay đổi (như `COPY . .`) đặt sau; (2) `--cache-from` — pull image cũ từ registry làm cache source trước khi build; (3) BuildKit với `DOCKER_BUILDKIT=1` — parallel build stages và mount cache thông minh hơn.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
**Chủ Đề Tiếp Theo:** [3-kubernetes-deployment.md](3-kubernetes-deployment.md)
