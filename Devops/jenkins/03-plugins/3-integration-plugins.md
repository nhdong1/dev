# Plugin Tích Hợp Công Cụ DevOps

> Nhóm plugin kết nối Jenkins với hệ sinh thái DevOps: Docker, Kubernetes, Slack, SonarQube, Nexus, và nhiều hơn nữa. Đây là lớp tích hợp giúp Jenkins trở thành trung tâm điều phối của toàn bộ CI/CD pipeline.

## Mục Lục

1. [Docker Integration](#docker-integration)
2. [Kubernetes Plugin](#kubernetes-plugin)
3. [Slack Notification Plugin](#slack-notification-plugin)
4. [SonarQube Scanner Plugin](#sonarqube-scanner-plugin)
5. [Artifact Repository Integration](#artifact-repository-integration)
6. [Notification Plugins Khác](#notification-plugins-khác)
7. [Cloud Provider Plugins](#cloud-provider-plugins)
8. [So Sánh và Lựa Chọn Plugin](#so-sánh-và-lựa-chọn-plugin)

---

## Docker Integration

### Docker Plugin

**ID:** `docker-plugin`

Quản lý Docker container và image từ Jenkins — tạo container làm build agent động (dynamic agent), push image lên registry.

**Cấu hình Docker Cloud (Đám Mây Docker) làm Agent:**

```
Manage Jenkins → Clouds → Add a new cloud → Docker
```

```
Docker Host URI: tcp://docker-host:2376
Docker TLS: [cấu hình certificate nếu dùng TLS]
```

**Cấu hình Docker Agent Template (Mẫu Agent Docker):**

```
Docker Cloud → Docker Agent templates → Add
├── Labels: docker-build
├── Docker Image: maven:3.9-eclipse-temurin-17
├── Remote File System Root: /home/jenkins
└── Instance Capacity: 5
```

```groovy
// Dùng trong Jenkinsfile
pipeline {
    agent {
        label 'docker-build'
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

### Docker Pipeline Plugin

**ID:** `docker-workflow`

Cho phép dùng Docker trực tiếp trong Declarative và Scripted Pipeline — không cần cấu hình cloud.

**Build Docker Image:**

```groovy
stage('Build Image') {
    steps {
        script {
            // Build image từ Dockerfile trong repo
            def image = docker.build("myapp:${env.BUILD_NUMBER}")

            // Push lên Docker Hub hoặc private registry
            docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                image.push()
                image.push('latest')    // Tag thêm 'latest'
            }
        }
    }
}
```

**Chạy Step trong Docker Container:**

```groovy
// Cách 1: agent docker block (toàn bộ pipeline chạy trong container)
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '-v /tmp:/tmp'         // Mount volume
        }
    }
    stages {
        stage('Test') {
            steps {
                sh 'npm ci && npm test'
            }
        }
    }
}
```

```groovy
// Cách 2: docker.image().inside() (chỉ một số stage chạy trong container)
stage('Run Tests in Container') {
    steps {
        script {
            docker.image('python:3.11-slim').inside('-v $HOME/.cache:/root/.cache') {
                sh 'pip install -r requirements.txt && pytest'
            }
        }
    }
}
```

**Multi-stage Build (Xây Dựng Nhiều Giai Đoạn) — Build Agent và App Image:**

```groovy
stage('Build và Push') {
    steps {
        script {
            // Build image dùng Dockerfile.ci (không phải production Dockerfile)
            def builderImage = docker.build("builder-${env.BUILD_NUMBER}",
                "-f Dockerfile.ci .")

            // Chạy tests trong builder image
            builderImage.inside {
                sh 'go test ./...'
            }

            // Build production image nhỏ hơn
            def prodImage = docker.build("myapp:${env.GIT_COMMIT[0..7]}")

            docker.withRegistry('https://registry.company.com', 'registry-creds') {
                prodImage.push()
                if (env.BRANCH_NAME == 'main') {
                    prodImage.push('stable')
                }
            }
        }
    }
}
```

---

### Docker-in-Docker vs Docker Socket

**Vấn đề:** Khi Jenkins Agent chạy trong Docker container, làm sao agent đó có thể build Docker image?

| Phương Pháp                              | Cách Thực Hiện                           | Ưu Điểm                    | Nhược Điểm                              |
| ---------------------------------------- | ---------------------------------------- | -------------------------- | --------------------------------------- |
| **Docker Socket Mount**                  | Mount `/var/run/docker.sock` vào agent   | Đơn giản, hiệu suất cao    | Rủi ro bảo mật (root access)            |
| **DinD — Docker in Docker**              | Chạy Docker daemon trong container       | Cô lập tốt hơn             | Phức tạp, cần privileged container      |
| **Kaniko**                               | Build image không cần Docker daemon      | Bảo mật nhất               | Không hỗ trợ tất cả Dockerfile features |

```groovy
// Dùng Docker Socket — phổ biến nhất
pipeline {
    agent {
        docker {
            image 'docker:24-dind'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
}
```

---

## Kubernetes Plugin

**ID:** `kubernetes`
**Tên Đầy Đủ:** Kubernetes Plugin — cho phép Jenkins dùng Kubernetes (hệ thống điều phối container) làm cloud provider để chạy dynamic agents (tác nhân động) dưới dạng Pod.

### Kiến Trúc Kubernetes Agent

```
Jenkins Master
     │
     │ (tạo Pod khi có job)
     ▼
Kubernetes Cluster
     │
     ├── Pod: jenkins-agent-abc123
     │   ├── container: jnlp      (kết nối với Master qua JNLP)
     │   └── container: build     (container chứa build tool)
     │
     └── Pod: jenkins-agent-def456
         ├── container: jnlp
         └── container: docker    (container có Docker CLI)
```

### Cấu Hình Kubernetes Cloud

```
Manage Jenkins → Clouds → Add a new cloud → Kubernetes
├── Kubernetes URL: https://k8s-api-server:6443
├── Kubernetes Namespace: jenkins
├── Jenkins URL: http://jenkins.jenkins.svc.cluster.local:8080
└── Jenkins tunnel: jenkins-agent.jenkins.svc.cluster.local:50000
```

### Pod Template — Mẫu Pod Agent

```groovy
// Định nghĩa Pod Template trong Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
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
    command: [cat]
    tty: true
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
  - name: docker
    image: docker:24
    command: [cat]
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        stage('Test') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                }
            }
        }
    }
}
```

### Pod Template Qua Giao Diện Jenkins

```
Manage Jenkins → Clouds → [Kubernetes Cloud] → Pod Templates → Add Pod Template
├── Name: maven-agent
├── Namespace: jenkins
├── Labels: maven
├── Containers:
│   ├── Container 1:
│   │   ├── Name: jnlp
│   │   └── Docker image: jenkins/inbound-agent:latest
│   └── Container 2:
│       ├── Name: maven
│       ├── Docker image: maven:3.9-eclipse-temurin-17
│       └── Command: cat (giữ container sống)
└── Resource Limit Memory: 1Gi, CPU: 1000m
```

### JCasC với Kubernetes Plugin

```yaml
# jenkins.yaml (JCasC — Cấu Hình Như Code)
jenkins:
  clouds:
    - kubernetes:
        name: kubernetes
        serverUrl: "https://k8s-api:6443"
        namespace: jenkins
        jenkinsUrl: "http://jenkins:8080"
        jenkinsTunnel: "jenkins-agent:50000"
        templates:
          - name: "maven-agent"
            label: "maven"
            containers:
              - name: jnlp
                image: "jenkins/inbound-agent:latest"
              - name: maven
                image: "maven:3.9-eclipse-temurin-17"
                command: "cat"
                ttyEnabled: true
            yaml: |
              spec:
                serviceAccountName: jenkins-agent
```

---

## Slack Notification Plugin

**ID:** `slack`

Gửi thông báo kết quả build lên Slack channel (kênh Slack).

### Cấu Hình Slack App

1. Tạo Slack App tại `api.slack.com/apps`
2. Bật Incoming Webhooks (hook gửi tin nhắn vào)
3. Thêm webhook URL vào Jenkins Credentials dưới dạng Secret text

```
Manage Jenkins → System → Slack
├── Workspace: company-workspace
├── Credential: [secret với Slack webhook URL]
└── Default channel: #build-notifications
```

### Dùng Trong Jenkinsfile

```groovy
post {
    success {
        slackSend(
            channel: '#deployments',
            color: 'good',          // good=xanh, warning=vàng, danger=đỏ
            message: ":white_check_mark: *${env.JOB_NAME}* #${env.BUILD_NUMBER} thành công\n" +
                     "Branch: `${env.GIT_BRANCH}`\n" +
                     "Commit: `${env.GIT_COMMIT[0..7]}`\n" +
                     "Thời gian: ${currentBuild.durationString}\n" +
                     "<${env.BUILD_URL}|Xem build>"
        )
    }
    failure {
        slackSend(
            channel: '#build-alerts',
            color: 'danger',
            message: ":x: *${env.JOB_NAME}* #${env.BUILD_NUMBER} THẤT BẠI\n" +
                     "Branch: `${env.GIT_BRANCH}`\n" +
                     "<${env.BUILD_URL}console|Xem log lỗi>"
        )
    }
    unstable {
        slackSend(
            channel: '#build-notifications',
            color: 'warning',
            message: ":warning: *${env.JOB_NAME}* #${env.BUILD_NUMBER} không ổn định (test failures)"
        )
    }
}
```

### Block Kit — Thông Báo Định Dạng Phong Phú

```groovy
// Dùng Slack Block Kit (định dạng phong phú hơn)
def blocks = """
[
  {
    "type": "header",
    "text": {"type": "plain_text", "text": "Build ${currentBuild.result}"}
  },
  {
    "type": "section",
    "fields": [
      {"type": "mrkdwn", "text": "*Job:*\\n${env.JOB_NAME}"},
      {"type": "mrkdwn", "text": "*Build:*\\n#${env.BUILD_NUMBER}"},
      {"type": "mrkdwn", "text": "*Branch:*\\n${env.GIT_BRANCH}"},
      {"type": "mrkdwn", "text": "*Duration:*\\n${currentBuild.durationString}"}
    ]
  }
]
"""
slackSend channel: '#deployments', blocks: blocks
```

---

## SonarQube Scanner Plugin

**ID:** `sonar`
**Kết Hợp Với:** SonarQube server (server phân tích code)

### Luồng Tích Hợp SonarQube

```
Code commit → Jenkins Pipeline
                   │
                   ▼
           SonarQube Scanner
           (phân tích static code)
                   │
                   ▼
           SonarQube Server
           (lưu kết quả, tính metrics)
                   │
                   ▼
           Quality Gate (Cổng Chất Lượng)
           ├── PASSED → tiếp tục deploy
           └── FAILED → dừng pipeline, thông báo team
```

### Cấu Hình SonarQube Server

```
Manage Jenkins → System → SonarQube servers → Add SonarQube
├── Name: SonarQube-Production
├── Server URL: http://sonarqube.company.com:9000
└── Server authentication token: [credential loại Secret text]
```

### Cấu Hình SonarQube Scanner Tool

```
Manage Jenkins → Tools → SonarQube Scanner → Add SonarQube Scanner
├── Name: SonarScanner-5.0
└── Install automatically: ✅
```

### Dùng Trong Pipeline

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube-Production') {
            // Maven project
            sh 'mvn sonar:sonar -Dsonar.projectKey=my-app'

            // Gradle project
            sh './gradlew sonarqube -Dsonar.projectKey=my-app'

            // Node.js project (cần sonar-scanner CLI)
            sh '''
                sonar-scanner \
                  -Dsonar.projectKey=my-frontend \
                  -Dsonar.sources=src \
                  -Dsonar.exclusions=**/*.test.js \
                  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
            '''
        }
    }
}

stage('Quality Gate') {
    steps {
        // Chờ SonarQube phân tích xong và lấy kết quả Quality Gate
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
            // abortPipeline: true → fail pipeline nếu Quality Gate FAILED
        }
    }
}
```

### Quality Gate — Cổng Chất Lượng

Quality Gate là tập hợp điều kiện được định nghĩa trong SonarQube:

```
Quality Gate "Default":
├── Coverage (độ phủ test) >= 80%
├── Duplicated Lines (dòng trùng lặp) <= 3%
├── Maintainability Rating (điểm bảo trì) = A
├── Reliability Rating (điểm độ tin cậy) = A
└── Security Rating (điểm bảo mật) = A
```

Nếu bất kỳ điều kiện nào không đạt → Quality Gate FAILED → pipeline dừng.

### SonarQube với Multibranch Pipeline

```groovy
// Cấu hình phân tích theo branch
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube-Production') {
            sh """
                mvn sonar:sonar \
                  -Dsonar.projectKey=my-app \
                  -Dsonar.branch.name=${env.BRANCH_NAME}
            """
        }
    }
}
```

---

## Artifact Repository Integration

### Nexus Artifact Uploader Plugin

**ID:** `nexus-artifact-uploader`

Upload artifact (kết quả build) lên Nexus Repository Manager (trình quản lý kho artifact).

```groovy
stage('Publish to Nexus') {
    steps {
        nexusArtifactUploader(
            nexusVersion: 'nexus3',
            protocol: 'http',
            nexusUrl: 'nexus.company.com:8081',
            groupId: 'com.company',
            version: env.APP_VERSION,
            repository: 'releases',
            credentialsId: 'nexus-credentials',
            artifacts: [
                [
                    artifactId: 'my-app',
                    classifier: '',
                    file: "target/my-app-${env.APP_VERSION}.jar",
                    type: 'jar'
                ],
                [
                    artifactId: 'my-app',
                    classifier: '',
                    file: "target/my-app-${env.APP_VERSION}.pom",
                    type: 'pom'
                ]
            ]
        )
    }
}
```

### Amazon S3 Publisher Plugin

**ID:** `s3`

Upload artifact lên Amazon S3 (dịch vụ lưu trữ đám mây của Amazon).

```groovy
stage('Archive to S3') {
    steps {
        withAWS(credentials: 'aws-credentials', region: 'ap-southeast-1') {
            s3Upload(
                bucket: 'my-artifacts-bucket',
                path: "builds/${env.APP_NAME}/${env.APP_VERSION}/",
                includePathPattern: 'target/*.jar',
                workingDir: '.'
            )
        }
    }
}
```

---

## Notification Plugins Khác

### Microsoft Teams Notification Plugin

**ID:** `office-365-connector`

Gửi thông báo vào Microsoft Teams channel.

```groovy
post {
    failure {
        office365ConnectorSend(
            webhookUrl: 'https://company.webhook.office.com/webhookb2/...',
            status: 'FAILED',
            message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} thất bại",
            color: 'ff0000'
        )
    }
}
```

### Telegram Notification Plugin

**ID:** `telegram-notifications`

```groovy
post {
    always {
        telegramSend(
            message: "${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            chatId: '-1001234567890'
        )
    }
}
```

### Generic Webhook Trigger Plugin

**ID:** `generic-webhook-trigger`

Nhận webhook từ bất kỳ hệ thống nào và trigger pipeline dựa trên payload — không cần plugin riêng cho từng service.

```groovy
// Trigger khi nhận POST request với token
triggers {
    GenericTrigger(
        genericVariables: [
            [key: 'DEPLOY_ENV', value: '$.environment'],
            [key: 'IMAGE_TAG', value: '$.image_tag']
        ],
        token: 'my-secret-trigger-token',
        causeString: 'Deploy triggered for $DEPLOY_ENV',
        regexpFilterText: '$DEPLOY_ENV',
        regexpFilterExpression: 'staging|production'
    )
}
```

```bash
# Gọi từ hệ thống khác
curl -X POST \
  "http://jenkins.company.com/generic-webhook-trigger/invoke?token=my-secret-trigger-token" \
  -H "Content-Type: application/json" \
  -d '{"environment": "staging", "image_tag": "v1.2.3"}'
```

---

## Cloud Provider Plugins

### Amazon EC2 Plugin

**ID:** `ec2`

Tự động tạo EC2 instance (máy chủ ảo trên AWS) làm Jenkins Agent khi cần, xóa đi sau khi build xong.

```
Manage Jenkins → Clouds → Amazon EC2
├── Region: ap-southeast-1
├── AMI: ami-xxxxxxxxx (AMI có cài Java)
├── Instance Type: t3.medium
└── Spot Instance: ✅ (tiết kiệm chi phí)
```

### Azure VM Agents Plugin

**ID:** `azure-vm-agents`

Tương tự EC2 Plugin nhưng cho Azure Virtual Machine.

---

## So Sánh và Lựa Chọn Plugin

### Stack Phổ Biến Theo Công Nghệ

**Stack Java/Spring Boot:**
```
Git Plugin + Maven integration
Docker Pipeline (build image)
SonarQube Scanner (code quality)
Nexus Artifact Uploader (lưu JAR)
Kubernetes Plugin (dynamic agent)
Slack Notification (thông báo)
```

**Stack Node.js/Frontend:**
```
Git Plugin + NodeJS Plugin
Docker Pipeline (build image)
SonarQube Scanner
S3 Publisher (upload static files)
Slack Notification
```

**Stack Python:**
```
Git Plugin
Docker Pipeline
SonarQube Scanner (Python rules)
Kubernetes Plugin
Slack Notification
```

### Tiêu Chí Lựa Chọn Plugin

```
1. Số download hàng tuần trên plugins.jenkins.io (> 10,000 = phổ biến)
2. Ngày release gần nhất (< 6 tháng = đang được bảo trì)
3. Số open issues trên GitHub (không quá nhiều bug)
4. Compatible Jenkins version (phiên bản Jenkins tương thích)
5. Có documentation rõ ràng không
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao Kubernetes Plugin được ưa chuộng hơn SSH Agent trong môi trường cloud native?**
> SSH Agent cần máy chủ luôn chạy, tốn chi phí kể cả lúc không có job. Kubernetes Plugin tạo Pod chỉ khi cần, xóa ngay sau khi build xong — tiết kiệm chi phí, dễ scale (mở rộng), mỗi build chạy trong môi trường sạch hoàn toàn (cô lập hoàn hảo). Ngoài ra, Pod template cho phép định nghĩa resource limit chính xác.

**Q: waitForQualityGate hoạt động như thế nào? Jenkins làm sao biết SonarQube đã phân tích xong?**
> SonarQube Scanner gửi kết quả lên SonarQube server và nhận về một `taskId` (mã tác vụ). `waitForQualityGate` liên tục poll (hỏi định kỳ) SonarQube API với taskId đó để kiểm tra trạng thái. Khi nhận PASSED hoặc FAILED, nó trả về kết quả cho Jenkins. Nếu quá timeout (thường 5 phút) chưa có kết quả → pipeline fail với lý do timeout.

**Q: Sự khác biệt giữa Docker Plugin và Docker Pipeline Plugin?**
> Docker Plugin (`docker-plugin`) cho phép cấu hình Docker host làm **cloud provider** để tạo dynamic agent — quản lý vòng đời container agent. Docker Pipeline Plugin (`docker-workflow`) cung cấp các **step trong Jenkinsfile** như `docker.build()`, `docker.image().inside()` — để thao tác Docker trong quá trình build. Cả hai thường được cài cùng nhau vì phục vụ mục đích khác nhau.

**Q: Khi nào nên dùng Generic Webhook Trigger thay vì GitHub Plugin?**
> GitHub Plugin chỉ nhận webhook từ GitHub. Generic Webhook Trigger nhận từ bất kỳ nguồn nào — Jira, Artifactory, ArgoCD, monitoring tools — với cấu hình linh hoạt để extract (trích xuất) giá trị từ payload JSON/XML. Đặc biệt hữu ích khi muốn trigger pipeline từ các hệ thống không có plugin Jenkins riêng.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
