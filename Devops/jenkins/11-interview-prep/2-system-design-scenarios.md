# System Design — Thiết Kế Hệ Thống CI/CD Pipeline

> Bộ bài toán thiết kế hệ thống CI/CD thực tế, với phân tích yêu cầu, kiến trúc đề xuất, trade-off và các câu hỏi follow-up thường gặp trong phỏng vấn kỹ sư DevOps senior.

## Mục Lục

1. [Cách Tiếp Cận System Design CI/CD](#cách-tiếp-cận-system-design-cicd)
2. [Scenario 1: CI/CD Pipeline cho Monolith](#scenario-1-cicd-pipeline-cho-monolith)
3. [Scenario 2: CI/CD cho Microservices trên Kubernetes](#scenario-2-cicd-cho-microservices-trên-kubernetes)
4. [Scenario 3: Jenkins High Availability và Disaster Recovery](#scenario-3-jenkins-high-availability-và-disaster-recovery)
5. [Scenario 4: Multi-tenant Jenkins cho Enterprise](#scenario-4-multi-tenant-jenkins-cho-enterprise)
6. [Scenario 5: Pipeline cho ML Model Deployment](#scenario-5-pipeline-cho-ml-model-deployment)
7. [Câu Hỏi Follow-up Thường Gặp](#câu-hỏi-follow-up-thường-gặp)

---

## Cách Tiếp Cận System Design CI/CD

### Framework 5 Bước

```
1. CLARIFY (làm rõ yêu cầu) — 3-5 phút
   - Scale: bao nhiêu developer, bao nhiêu build/ngày?
   - Tech stack: ngôn ngữ, container, Kubernetes?
   - Requirements: nonfunctional — availability, security, compliance?
   - Constraints: on-premise hay cloud? budget?

2. HIGH-LEVEL DESIGN (thiết kế tổng thể) — 5 phút
   - Vẽ sơ đồ kiến trúc tổng thể
   - Xác định các thành phần chính
   - Mô tả luồng CI/CD từ commit đến production

3. COMPONENT DEEP-DIVE (đi sâu từng thành phần) — 10 phút
   - Pipeline stages và logic
   - Agent/infrastructure design
   - Security model
   - Integration với external systems

4. TRADE-OFFS (đánh đổi) — 3 phút
   - Tại sao chọn giải pháp này?
   - Ưu/nhược điểm so với alternatives
   - Gì có thể sai và cách mitigation

5. PRODUCTION CONCERNS (vấn đề production) — 3 phút
   - Monitoring và alerting
   - Failure scenarios và recovery
   - Scalability path
   - Migration plan nếu đang có hệ thống cũ
```

### Bảng Yêu Cầu Cần Làm Rõ

| Nhóm            | Câu Hỏi Cần Hỏi                                                    |
| --------------- | ------------------------------------------------------------------ |
| Scale           | Bao nhiêu developer? Bao nhiêu repositories? Build/ngày?           |
| Environments    | Bao nhiêu môi trường (dev/staging/prod)? Deploy strategy?          |
| Tech Stack      | Ngôn ngữ? Container? Kubernetes? Cloud provider?                   |
| Security        | Compliance (SOC2, PCI-DSS)? Secret management? Network policy?     |
| Availability    | SLA cho CI/CD? Chấp nhận downtime bao nhiêu?                      |
| Team            | Một team vận hành Jenkins hay self-service cho từng team?          |

---

## Scenario 1: CI/CD Pipeline cho Monolith

### Đề Bài

> "Công ty bạn có một ứng dụng Java monolith, 20 developer, deploy lên 3 server vật lý (bare-metal). Thiết kế CI/CD pipeline end-to-end."

### Clarify

```
Scale: 20 developer, 1 repository, ~30-50 build/ngày
Stack: Java 17, Maven, Docker, 3 bare-metal servers (dev/staging/prod)
Requirements: deploy không có downtime, rollback nhanh
Security: không có compliance đặc biệt, credential phải an toàn
```

### Kiến Trúc Tổng Thể

```
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ Webhook (HTTP POST)
    ▼
Jenkins Master
    │
    │ phân phối build
    ▼
Jenkins Agent (VM)
    │
    ├── Stage: Checkout
    ├── Stage: Build (mvn package)
    ├── Stage: Unit Test
    ├── Stage: Code Quality (SonarQube)
    ├── Stage: Docker Build & Push → Docker Registry
    ├── Stage: Deploy Staging → bare-metal staging server
    ├── Stage: Integration Test
    ├── Stage: Approval Gate (chỉ khi main branch)
    └── Stage: Deploy Production → bare-metal prod server (Blue-Green)
```

### Jenkinsfile Chi Tiết

```groovy
pipeline {
    agent { label 'build-server' }

    environment {
        APP_NAME    = 'myapp'
        REGISTRY    = 'registry.company.com'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        SONAR_URL   = 'http://sonarqube:9000'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests -B'
            }
            post {
                success {
                    stash name: 'build-artifact', includes: 'target/*.jar'
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test -B'
                        junit 'target/surefire-reports/*.xml'
                    }
                }
                stage('Code Quality') {
                    steps {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                                mvn sonar:sonar \
                                    -Dsonar.host.url=${SONAR_URL} \
                                    -Dsonar.login=${SONAR_TOKEN} \
                                    -B
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                unstash 'build-artifact'
                script {
                    def fullTag = "${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        def image = docker.build(fullTag)
                        image.push()
                        image.push('latest')
                    }
                    env.DOCKER_IMAGE = fullTag
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'staging-server',
                        transfers: [
                            sshTransfer(
                                execCommand: """
                                    docker pull ${env.DOCKER_IMAGE} &&
                                    docker stop ${APP_NAME} || true &&
                                    docker rm ${APP_NAME} || true &&
                                    docker run -d --name ${APP_NAME} \
                                        -p 8080:8080 \
                                        --restart unless-stopped \
                                        ${env.DOCKER_IMAGE}
                                """
                            )
                        ]
                    )
                ])
            }
        }

        stage('Integration Tests') {
            steps {
                sh 'mvn verify -Pintegration -Dapp.url=http://staging-server:8080 -B'
            }
        }

        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                // Approval gate (cổng phê duyệt)
                timeout(time: 2, unit: 'HOURS') {
                    input(
                        message: "Deploy ${IMAGE_TAG} to Production?",
                        ok: 'Deploy',
                        submitter: 'release-team'
                    )
                }

                // Blue-Green deployment (triển khai Blue-Green)
                script {
                    def currentColor = sh(
                        script: 'ssh prod-server "cat /etc/app/current-color"',
                        returnStdout: true
                    ).trim()
                    def newColor = currentColor == 'blue' ? 'green' : 'blue'
                    def newPort = newColor == 'blue' ? '8081' : '8082'

                    sh """
                        ssh prod-server "
                            docker pull ${env.DOCKER_IMAGE} &&
                            docker stop ${APP_NAME}-${newColor} || true &&
                            docker rm ${APP_NAME}-${newColor} || true &&
                            docker run -d --name ${APP_NAME}-${newColor} \
                                -p ${newPort}:8080 \
                                ${env.DOCKER_IMAGE} &&
                            # Smoke test trên instance mới
                            sleep 10 &&
                            curl -f http://localhost:${newPort}/health &&
                            # Switch traffic (chuyển traffic)
                            echo ${newColor} > /etc/app/current-color &&
                            nginx -s reload
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: ":white_check_mark: Deploy ${env.APP_NAME}:${IMAGE_TAG} succeeded — ${env.GIT_COMMIT_SHORT}"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: ":x: Build ${env.BUILD_NUMBER} FAILED — ${env.JOB_NAME}"
            )
        }
        always {
            cleanWs()
        }
    }
}
```

### Trade-offs và Câu Hỏi Follow-up

**Q: Tại sao dùng Blue-Green deployment?**
> Blue-Green cho phép rollback instant bằng cách switch nginx về color cũ. Với bare-metal, đây là cách đơn giản nhất để có zero-downtime deploy. Alternative là Rolling deployment — phức tạp hơn trên bare-metal.

**Q: Điều gì xảy ra nếu SonarQube Quality Gate fail?**
> Pipeline abort ngay tại stage Quality Gate, không deploy lên bất kỳ environment nào. Developer nhận Slack notification và phải fix code quality trước khi re-push.

---

## Scenario 2: CI/CD cho Microservices trên Kubernetes

### Đề Bài

> "Thiết kế CI/CD pipeline cho hệ thống 30 microservices, 100 developer, deploy lên Kubernetes. Yêu cầu: mỗi service deploy độc lập, không ảnh hưởng nhau."

### Clarify

```
Scale: 100 developer, 30 repos, ~200-300 build/ngày
Stack: Go/Java/Python mixed, Docker, Kubernetes (EKS), Helm
Environments: dev/staging/prod (3 clusters riêng biệt)
Security: Secrets trong AWS Secrets Manager
Deployment: GitOps với ArgoCD
Availability: CI/CD SLA 99.5%, deploy không downtime
```

### Kiến Trúc Tổng Thể

```
Developer Team
    │
    │ git push (feature branch)
    ▼
GitHub Repository (per service)
    │
    │ Webhook
    ▼
Jenkins Master (Kubernetes)
    │
    │ Kubernetes Agent (dynamic Pod)
    ▼
Jenkins Pipeline
    ├── CI Phase ────────────────────────────────────────────────────┐
    │   ├── Checkout + version generation                            │
    │   ├── Build (language-specific: Maven/Go build/pip install)   │
    │   ├── Unit Test + Coverage                                     │
    │   ├── Container Image Build (Kaniko — không cần Docker daemon) │
    │   ├── Image Push → ECR (Elastic Container Registry)           │
    │   ├── Security Scan (Trivy image scan)                        │
    │   └── SonarQube Quality Gate                                  │
    │                                                                │
    └── CD Phase ─────────────────────────────────────────────────── ▼
        ├── Update Helm values.yaml (tag mới)
        ├── Git commit + push → GitOps repo
        └── ArgoCD tự detect change → deploy lên K8s cluster
```

### Shared Library cho Multi-language

```groovy
// vars/microservicePipeline.groovy — Shared Library dùng chung cho 30 services

def call(Map config) {
    def defaults = [
        language: 'java',           // java | go | python
        testCommand: '',
        dockerfilePath: 'Dockerfile',
        helmChartPath: './chart',
        ecrRepo: '',
        gitOpsRepo: 'git@github.com:company/gitops-configs.git',
        deployTimeout: 10           // phút
    ]
    config = defaults + config

    pipeline {
        agent {
            kubernetes {
                yaml buildPodTemplate(config.language)
            }
        }

        options {
            buildDiscarder(logRotator(numToKeepStr: '20'))
            timeout(time: 30, unit: 'MINUTES')
        }

        environment {
            IMAGE_TAG     = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"
            ECR_REGISTRY  = "123456789.dkr.ecr.ap-southeast-1.amazonaws.com"
            FULL_IMAGE    = "${ECR_REGISTRY}/${config.ecrRepo}:${IMAGE_TAG}"
        }

        stages {
            stage('Build & Test') {
                steps {
                    container(config.language) {
                        script {
                            if (config.language == 'java') {
                                sh 'mvn clean package -B'
                                sh 'mvn test -B'
                            } else if (config.language == 'go') {
                                sh 'go build ./...'
                                sh 'go test ./... -v'
                            } else if (config.language == 'python') {
                                sh 'pip install -r requirements.txt'
                                sh 'pytest tests/ -v --cov=.'
                            }
                        }
                    }
                }
            }

            stage('Container Build') {
                steps {
                    container('kaniko') {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                         credentialsId: 'aws-ecr-credentials']]) {
                            sh """
                                /kaniko/executor \
                                    --context=${env.WORKSPACE} \
                                    --dockerfile=${config.dockerfilePath} \
                                    --destination=${FULL_IMAGE} \
                                    --cache=true \
                                    --cache-repo=${ECR_REGISTRY}/cache
                            """
                        }
                    }
                }
            }

            stage('Security Scan') {
                steps {
                    container('trivy') {
                        sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${FULL_IMAGE}"
                    }
                }
            }

            stage('GitOps Update') {
                when {
                    anyOf {
                        branch 'main'
                        branch 'develop'
                    }
                }
                steps {
                    container('git') {
                        withCredentials([sshUserPrivateKey(credentialsId: 'gitops-ssh-key',
                                                           keyFileVariable: 'SSH_KEY')]) {
                            script {
                                def env_name = env.BRANCH_NAME == 'main' ? 'production' : 'staging'
                                sh """
                                    GIT_SSH_COMMAND='ssh -i ${SSH_KEY}' \
                                    git clone ${config.gitOpsRepo} gitops-configs

                                    # Cập nhật image tag trong Helm values
                                    sed -i 's|tag:.*|tag: "${IMAGE_TAG}"|' \
                                        gitops-configs/${config.ecrRepo}/${env_name}/values.yaml

                                    cd gitops-configs
                                    git config user.email "ci@company.com"
                                    git config user.name "Jenkins CI"
                                    git add .
                                    git commit -m "ci: update ${config.ecrRepo} to ${IMAGE_TAG} [${env_name}]"
                                    GIT_SSH_COMMAND='ssh -i ${SSH_KEY}' git push
                                """
                            }
                        }
                    }
                }
            }
        }

        post {
            always { cleanWs() }
            failure {
                slackSend(
                    channel: '#ci-failures',
                    color: 'danger',
                    message: ":x: ${config.ecrRepo} build ${IMAGE_TAG} failed"
                )
            }
        }
    }
}

// Tạo Pod template theo ngôn ngữ
def buildPodTemplate(String language) {
    def containers = [
        """
        - name: ${language}
          image: ${getBaseImage(language)}
          command: ["sleep"]
          args: ["99d"]
        """,
        """
        - name: kaniko
          image: gcr.io/kaniko-project/executor:latest
          command: ["sleep"]
          args: ["99d"]
        """,
        """
        - name: trivy
          image: aquasec/trivy:latest
          command: ["sleep"]
          args: ["99d"]
        """,
        """
        - name: git
          image: alpine/git:latest
          command: ["sleep"]
          args: ["99d"]
        """
    ]

    return """
apiVersion: v1
kind: Pod
spec:
  containers:
${containers.join('')}
"""
}

def getBaseImage(String language) {
    def images = [
        java:   'maven:3.9-eclipse-temurin-17',
        go:     'golang:1.22-alpine',
        python: 'python:3.12-slim'
    ]
    return images[language] ?: 'ubuntu:22.04'
}
```

**Cách dùng trong từng service — Jenkinsfile chỉ 5 dòng:**

```groovy
@Library('platform-shared-lib@v3') _

microservicePipeline(
    language: 'java',
    ecrRepo: 'payment-service',
    gitOpsRepo: 'git@github.com:company/gitops-configs.git'
)
```

### GitOps Flow với ArgoCD

```
Jenkins Pipeline (CI)                ArgoCD (CD)
        │                                │
        │ 1. Build & push image          │
        │ 2. Update gitops repo ─────────┤
        │                                │ 3. Detect change trong 3 phút
        │                                │ 4. Sync với Kubernetes cluster
        │                                │ 5. Health check sau deploy
        │                                │ 6. Rollback tự động nếu fail
```

### Trade-offs

**Q: Tại sao dùng GitOps (ArgoCD) thay vì deploy trực tiếp từ Jenkins?**
> GitOps tách biệt CI (build, test) và CD (deploy). Lợi ích:
> - Audit trail hoàn chỉnh trong Git — ai thay đổi gì, khi nào
> - Rollback bằng `git revert` — không cần Jenkins
> - Self-healing — nếu ai kubectl delete nhầm, ArgoCD tự restore
> - Pipeline Jenkins không cần Kubernetes access — giảm attack surface

**Q: Tại sao dùng Kaniko thay vì Docker?**
> Kaniko build Docker image trong container mà không cần Docker daemon, không cần privileged mode. An toàn hơn DinD và socket mounting trong môi trường Kubernetes.

---

## Scenario 3: Jenkins High Availability và Disaster Recovery

### Đề Bài

> "Thiết kế Jenkins HA cho công ty fintech với SLA 99.9% cho CI/CD. Nếu Jenkins master fail, phải restore trong 5 phút."

### Clarify

```
SLA: 99.9% uptime (downtime tối đa 8.7 giờ/năm, ~43 phút/tháng)
RTO (Recovery Time Objective — mục tiêu thời gian khôi phục): 5 phút
RPO (Recovery Point Objective — mục tiêu điểm khôi phục): 30 phút (mất tối đa 30 phút data)
Scale: 200 developer, 500 build/ngày
Infrastructure: AWS EKS
Compliance: SOC2
```

### Kiến Trúc HA

```
Route 53 (DNS failover)
       │
       ▼
Application Load Balancer
       │
       ▼
Jenkins Master (Kubernetes Deployment — 1 replica)
       │
       ├── Persistent Volume (EBS gp3) ← JENKINS_HOME
       │         │
       │         └── EBS Snapshot mỗi 30 phút (tự động)
       │
       ├── Config lưu trong Git (JCasC)
       │         │
       │         └── Jenkins tự load lại khi restart
       │
       └── Jenkins Agents (Dynamic — Kubernetes Pods)
                 │
                 └── Không bị ảnh hưởng khi Master restart
```

**Kubernetes Deployment cho Jenkins:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  strategy:
    type: Recreate   # Xóa pod cũ trước khi tạo pod mới (tránh 2 Jenkins cùng mount EBS)
  selector:
    matchLabels:
      app: jenkins
  template:
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:2.440.3-lts-jdk17
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 30
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        - name: jcasc-config
          mountPath: /var/jenkins_home/casc_configs
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-home-pvc
      - name: jcasc-config
        configMap:
          name: jenkins-casc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home-pvc
  namespace: jenkins
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3-encrypted   # EBS gp3 với encryption
  resources:
    requests:
      storage: 200Gi
```

### Backup và Recovery

**Automated EBS Snapshot (ảnh chụp EBS tự động):**

```yaml
# AWS Data Lifecycle Manager Policy
retention:
  interval: 30        # minutes
  count: 48           # giữ 48 snapshot = 24 giờ gần nhất
  tags:
    - key: Name
      value: jenkins-home-backup
```

**Recovery Runbook (sổ tay phục hồi):**

```bash
# Kịch bản: Jenkins Pod không khởi động được, EBS bị corrupt

# Bước 1: Xác định snapshot tốt nhất (trong 30 phút gần nhất)
aws ec2 describe-snapshots \
    --filters "Name=tag:Name,Values=jenkins-home-backup" \
    --query 'Snapshots | sort_by(@, &StartTime) | [-5:]'

# Bước 2: Tạo EBS volume mới từ snapshot
aws ec2 create-volume \
    --snapshot-id snap-0abc123 \
    --availability-zone ap-southeast-1a \
    --volume-type gp3 \
    --encrypted

# Bước 3: Xóa PVC cũ, tạo PV + PVC trỏ vào volume mới
kubectl delete pvc jenkins-home-pvc -n jenkins
kubectl apply -f jenkins-pv-restore.yaml

# Bước 4: Deploy lại Jenkins
kubectl rollout restart deployment/jenkins -n jenkins
kubectl rollout status deployment/jenkins -n jenkins

# Bước 5: Verify
curl -f http://jenkins.internal/login
```

**Mục tiêu RTO đạt được:**

```
EKS detect pod fail:        ~30 giây
Tạo pod mới:                ~60 giây
Pull image (cached):        ~30 giây
Jenkins startup:            ~120 giây
Health check pass:          ~30 giây
TOTAL:                      ~4.5 phút ✅ (dưới RTO 5 phút)
```

### Chaos Testing (Kiểm Tra Độ Bền)

Định kỳ hàng tháng, chạy chaos drill (bài kiểm tra hỗn loạn):

```bash
# Kill Jenkins pod đột ngột, đo thời gian recovery
kubectl delete pod -l app=jenkins -n jenkins --grace-period=0

# Bắt đầu đếm giờ, đợi recovery tự động
time kubectl wait --for=condition=ready pod -l app=jenkins -n jenkins --timeout=600s
```

### Trade-offs

**Q: Tại sao không dùng multi-master (nhiều master) cho HA?**
> Jenkins không được thiết kế để chạy multi-master. Jenkins Master sử dụng local filesystem (`JENKINS_HOME`) để lưu trạng thái — không thể share giữa nhiều instance mà không có distributed lock mechanism. Commercial solution như CloudBees CI có Operation Center nhưng phức tạp và tốn tiền.

**Q: 4.5 phút recovery — nếu build đang chạy trên agent thì sao?**
> Khi Master fail, Agent mất kết nối và build abort. Sau khi Master recover, Agent tự reconnect. Build cần chạy lại từ đầu (Jenkins mặc định) hoặc có thể dùng [Pipeline Durability Settings](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/#restart-from-a-stage) để resume từ stage đã chạy.

---

## Scenario 4: Multi-tenant Jenkins cho Enterprise

### Đề Bài

> "Công ty 500 developer, 10 product teams, cần CI/CD tập trung nhưng mỗi team phải độc lập, không thấy job của team khác."

### Kiến Trúc Multi-tenant

```
Jenkins Master
├── Folder: /team-backend/
│   ├── job: api-service
│   ├── job: auth-service
│   └── Jenkins Agent: team-backend-agents (namespace k8s riêng)
│
├── Folder: /team-frontend/
│   ├── job: web-app
│   ├── job: admin-portal
│   └── Jenkins Agent: team-frontend-agents
│
├── Folder: /team-data/
│   ├── job: etl-pipeline
│   └── Jenkins Agent: team-data-agents (GPU nodes)
│
└── Folder: /platform/
    ├── job: shared-library-test
    └── Jenkins Agent: platform-agents
```

### Role-based Access Control

```yaml
# Jenkins Configuration as Code (JCasC)
jenkins:
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "platform-admin"
            permissions:
              - "Overall/Administer"
            assignments:
              - "platform-team"

          - name: "authenticated"
            permissions:
              - "Overall/Read"
            assignments:
              - "authenticated"  # tất cả user đăng nhập

        items:
          - name: "team-backend-role"
            pattern: "team-backend/.*"   # chỉ trong folder team-backend
            permissions:
              - "Job/Build"
              - "Job/Cancel"
              - "Job/Read"
              - "Job/Workspace"
            assignments:
              - "team-backend-developers"

          - name: "team-backend-lead"
            pattern: "team-backend/.*"
            permissions:
              - "Job/Configure"
              - "Job/Create"
              - "Job/Delete"
              - "Job/Build"
              - "Job/Read"
            assignments:
              - "team-backend-leads"
```

### Kubernetes Namespace Isolation (Cô Lập Namespace)

```yaml
# Kubernetes Pod Template — mỗi team dùng namespace riêng
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins-team-backend
---
# NetworkPolicy — chỉ Jenkins Master trong namespace jenkins được kết nối
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jenkins-agent-policy
  namespace: jenkins-team-backend
spec:
  podSelector:
    matchLabels:
      jenkins/agent: "true"
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: jenkins   # chỉ từ namespace jenkins (Master)
```

### Trade-offs

**Q: Multi-tenant Jenkins Master vs nhiều Jenkins Master riêng cho từng team?**

| Tiêu Chí            | Chung 1 Master         | Riêng mỗi team              |
| ------------------- | ---------------------- | ----------------------------|
| Chi phí             | ✅ Thấp               | ❌ Cao (nhiều instances)    |
| Isolation           | ❌ Share tài nguyên   | ✅ Hoàn toàn độc lập       |
| Team autonomy       | ❌ Cần platform team  | ✅ Tự quản lý              |
| Operational burden  | ✅ Một chỗ quản lý    | ❌ N instances quản lý     |
| Plugin compatibility| ❌ Conflict giữa teams | ✅ Không xung đột         |

**Khuyến nghị:** < 5 teams → chung 1 Master. ≥ 10 teams → xem xét tách, hoặc dùng CloudBees Operations Center.

---

## Scenario 5: Pipeline cho ML Model Deployment

### Đề Bài

> "Thiết kế CI/CD pipeline cho ML model (machine learning model — mô hình học máy), bao gồm training, validation, và deploy lên production."

### Kiến Trúc ML Pipeline

```
Data Scientist push code
        │
        ▼
Jenkins CI Pipeline
        │
        ├── Stage: Code Lint & Unit Test (unit test code training)
        │
        ├── Stage: Data Validation
        │   └── Kiểm tra data schema, missing values, distribution shift
        │
        ├── Stage: Model Training (GPU agent — chạy nhiều giờ)
        │   └── Kết quả: model artifact + metrics (accuracy, F1, AUC)
        │
        ├── Stage: Model Validation Gate
        │   └── So sánh metrics với production model hiện tại
        │   └── Nếu model mới tốt hơn threshold → tiếp tục
        │   └── Nếu tệ hơn → fail pipeline, alert team
        │
        ├── Stage: Model Registry Push
        │   └── Push lên MLflow Model Registry với version tag
        │
        └── Stage: Canary Deploy (triển khai canary — thử nghiệm nhỏ)
            ├── Deploy model mới nhận 5% traffic
            ├── Monitor metrics (latency, accuracy, error rate) trong 1 giờ
            └── Auto-promote (tự động thăng cấp) hoặc Auto-rollback
```

**Jenkinsfile cho ML Pipeline:**

```groovy
pipeline {
    agent none  // agent được chỉ định per-stage

    stages {
        stage('Code Quality') {
            agent { label 'cpu-agent' }
            steps {
                sh 'pip install -r requirements-dev.txt'
                sh 'flake8 src/'
                sh 'pytest tests/unit/ -v'
            }
        }

        stage('Data Validation') {
            agent { label 'cpu-agent' }
            steps {
                withCredentials([string(credentialsId: 'data-lake-token', variable: 'DATA_TOKEN')]) {
                    sh 'python scripts/validate_data.py --threshold 0.95'
                }
            }
        }

        stage('Model Training') {
            agent { label 'gpu-agent' }  // Agent có GPU
            options {
                timeout(time: 4, unit: 'HOURS')  // Training có thể mất giờ
            }
            steps {
                sh '''
                    python train.py \
                        --epochs 100 \
                        --output-dir ./models \
                        --mlflow-uri http://mlflow:5000
                '''
                // Lưu metrics để compare
                stash name: 'model-metrics', includes: 'metrics.json,models/**'
            }
        }

        stage('Model Validation') {
            agent { label 'cpu-agent' }
            steps {
                unstash 'model-metrics'
                script {
                    def result = sh(
                        script: 'python scripts/compare_models.py --new metrics.json --threshold 0.02',
                        returnStatus: true
                    )
                    if (result != 0) {
                        error("New model does not meet quality threshold — aborting deploy")
                    }
                }
            }
        }

        stage('Deploy Canary') {
            when { branch 'main' }
            agent { label 'cpu-agent' }
            steps {
                // Deploy 5% traffic lên model mới
                sh 'python scripts/canary_deploy.py --traffic-pct 5 --model-version ${BUILD_NUMBER}'

                // Monitor 1 giờ
                timeout(time: 1, unit: 'HOURS') {
                    sh 'python scripts/monitor_canary.py --duration 3600 --alert-threshold 0.05'
                }

                // Nếu qua — promote 100%
                sh 'python scripts/promote_model.py --model-version ${BUILD_NUMBER}'
            }
        }
    }
}
```

### Trade-offs

**Q: Tại sao không dùng dedicated ML platform như Kubeflow hay SageMaker Pipelines?**
> Jenkins phù hợp khi team đã quen Jenkins và muốn CI/CD ML trong cùng ecosystem với service deployment. Kubeflow Pipelines hay SageMaker Pipelines phù hợp hơn khi ML là core business với nhiều data scientists — có tính năng experiment tracking, model lineage tốt hơn. Không có câu trả lời duy nhất đúng — phụ thuộc vào team skill và scale.

---

## Câu Hỏi Follow-up Thường Gặp

### Sau Scenario 1 (Monolith)

```
Q: Nếu deployment fail giữa chừng, rollback như thế nào?
→ Blue-Green: switch nginx về color cũ (1 lệnh, <30 giây)
   Cần có health check sau deploy để tự động detect fail

Q: Làm sao test database migration trong pipeline?
→ Dùng test database riêng cho CI, chạy migration với --dry-run trước
   Rollback strategy: Flyway/Liquibase undo migration

Q: Nếu muốn feature flag thay vì Blue-Green?
→ Tích hợp LaunchDarkly hoặc Unleash — deploy code nhưng feature tắt
   Enable dần cho từng % user, không cần infrastructure phức tạp
```

### Sau Scenario 2 (Microservices)

```
Q: 30 services cùng update — deploy theo thứ tự nào?
→ Dependency graph: service A phụ thuộc B → deploy B trước A
   ArgoCD sync wave: annotation argocd.argoproj.io/sync-wave

Q: Rollback khi một service break toàn bộ system?
→ GitOps: git revert commit trong gitops repo → ArgoCD tự rollback
   Feature flag cho breaking API changes

Q: Service mesh (lưới dịch vụ) ảnh hưởng gì đến pipeline?
→ Istio/Linkerd cho canary deployment ở network level
   Pipeline chỉ cần update VirtualService weight thay vì manage pods trực tiếp
```

### Câu Hỏi So Sánh Thường Gặp

```
Q: Jenkins vs GitLab CI/CD — khi nào chọn GitLab?
→ GitLab CI: codebase đã trên GitLab, muốn tích hợp sâu (MR approval, security scanning built-in)
   Jenkins: infrastructure phức tạp, cần customization cao, nhiều tool khác ngoài GitLab

Q: Jenkins vs Tekton (Kubernetes-native CI/CD)?
→ Tekton: cloud-native, mọi thứ là Kubernetes CRD, phù hợp platform team muốn Kubernetes-first
   Jenkins: ecosystem plugin phong phú, team đã quen Jenkins, nhiều integrations có sẵn

Q: Có nên dùng Jenkins cho 2025 project mới không?
→ Honest answer: Với project mới từ đầu trên cloud, GitHub Actions hoặc GitLab CI/CD thường dễ setup hơn.
   Jenkins vẫn rất giá trị khi: on-premise, complex pipeline, cần Shared Library ecosystem,
   hoặc tổ chức đã có Jenkins expertise và infrastructure.
```

---

**Xem lại:** [INTERVIEW_GUIDE.md](INTERVIEW_GUIDE.md) — Top 20 câu hỏi phỏng vấn với gợi ý trả lời chi tiết
