# Jenkins Interview Guide — Top 20 Câu Hỏi Phỏng Vấn

> Bộ 20 câu hỏi phỏng vấn Jenkins thường gặp nhất, kèm gợi ý trả lời chi tiết, ví dụ code thực tế, và các điểm mấu chốt giúp gây ấn tượng với interviewer.

## Mục Lục

- [Nhóm 1: Kiến Trúc và Khái Niệm (Q1–Q4)](#nhóm-1-kiến-trúc-và-khái-niệm)
- [Nhóm 2: Pipeline và Jenkinsfile (Q5–Q9)](#nhóm-2-pipeline-và-jenkinsfile)
- [Nhóm 3: Distributed Builds và Agent (Q10–Q12)](#nhóm-3-distributed-builds-và-agent)
- [Nhóm 4: Security và Credentials (Q13–Q15)](#nhóm-4-security-và-credentials)
- [Nhóm 5: Thực Tế và Troubleshooting (Q16–Q18)](#nhóm-5-thực-tế-và-troubleshooting)
- [Nhóm 6: Nâng Cao và So Sánh (Q19–Q20)](#nhóm-6-nâng-cao-và-so-sánh)

---

## Nhóm 1: Kiến Trúc và Khái Niệm

### Q1. Giải thích kiến trúc Jenkins Master/Agent. Tại sao cần phân tách?

**Mức độ:** Cơ bản — gần như 100% phỏng vấn hỏi.

**Gợi ý trả lời:**

Jenkins hoạt động theo mô hình **Master/Agent** (chủ-tác nhân):

**Jenkins Master (Controller):**
- Lưu trữ toàn bộ cấu hình: job definitions, plugin, user accounts
- Quản lý Build Queue (hàng đợi build) — sắp xếp thứ tự, phân phối
- Điều phối Agent — quyết định agent nào chạy job nào
- Phục vụ Web UI và REST API
- **Không nên chạy build trực tiếp** trên Master (lý do bảo mật và tài nguyên)

**Jenkins Agent (Worker Node):**
- Thực thi build theo lệnh từ Master
- Chứa Workspace (không gian làm việc) — nơi source code được checkout
- Kết nối với Master qua JNLP (Java Network Launch Protocol) hoặc SSH
- Có thể là máy vật lý, VM, Docker container, hoặc Kubernetes Pod

**Tại sao cần phân tách:**

```
Master  ──── điều phối ────►  Agent 1 (Linux, Java build)
              │               Agent 2 (Windows, .NET build)
              └──────────────► Agent 3 (Docker, containerized)
```

- **Bảo mật:** Code người dùng chạy trên Agent, không ảnh hưởng Master
- **Scalability (mở rộng):** Thêm Agent khi workload tăng mà không đụng Master
- **Isolation (cô lập):** Mỗi build có Workspace riêng, không xung đột
- **Đa nền tảng:** Agent Linux build Java, Agent Windows build .NET cùng lúc

**Điểm cộng khi trả lời:** Đề cập rằng từ Jenkins 2.x, "Master" được đổi tên thành "Controller" trong tài liệu chính thức, nhưng thuật ngữ "Master" vẫn được dùng phổ biến.

---

### Q2. Executor là gì? Cấu hình như thế nào cho hợp lý?

**Mức độ:** Cơ bản.

**Gợi ý trả lời:**

**Executor** (bộ thực thi) là một "slot" chạy build trên một Node (nút). Mỗi Node có thể cấu hình nhiều Executor để chạy nhiều build song song.

```
Node "build-server-1" — 4 Executors:
├── Executor 1: Running job "frontend-build"
├── Executor 2: Running job "backend-test"
├── Executor 3: Idle (rảnh)
└── Executor 4: Idle
```

**Nguyên tắc cấu hình Executor:**

| Loại Workload             | Số Executor Khuyến Nghị              |
| ------------------------- | ------------------------------------|
| CPU-intensive (build, compile) | số CPU cores × 1                |
| I/O-intensive (test, deploy)   | số CPU cores × 1.5–2            |
| Mixed                          | Bắt đầu với số CPU, theo dõi rồi tăng |

**Executor trên Master:** Thường đặt là **0** — không cho Master chạy build để bảo vệ tài nguyên và bảo mật.

---

### Q3. Build Queue là gì? Khi nào build bị stuck (mắc kẹt) trong Queue?

**Mức độ:** Cơ bản - Trung bình.

**Gợi ý trả lời:**

**Build Queue** (hàng đợi build) là nơi Jenkins lưu các build đang chờ được giao cho Agent. Build vào Queue khi:
- Tất cả Executor trên tất cả Agent phù hợp đều bận
- Agent bị offline hoặc không có Agent nào match label của job

**Nguyên nhân build bị stuck trong Queue:**

1. **Không có Agent phù hợp:** Job yêu cầu label `linux-docker` nhưng không có Agent nào có label đó
2. **Tất cả Executor bận:** Workload vượt quá capacity
3. **Agent offline:** Agent mất kết nối với Master
4. **Concurrent build limit (giới hạn build song song):** Job được cấu hình chỉ cho phép 1 build chạy cùng lúc

**Cách kiểm tra:** Jenkins UI > Manage Jenkins > Nodes — xem trạng thái từng Node và số Executor rảnh.

---

### Q4. Jenkins Configuration as Code (JCasC) là gì và tại sao quan trọng?

**Mức độ:** Trung bình - Nâng cao.

**Gợi ý trả lời:**

**JCasC — Jenkins Configuration as Code** (cấu hình Jenkins dưới dạng code) là plugin cho phép lưu toàn bộ cấu hình Jenkins vào file YAML, quản lý qua Git.

**Vấn đề JCasC giải quyết:**
- Jenkins truyền thống: cấu hình qua Web UI, dễ bị "configuration drift" (cấu hình lệch giữa môi trường)
- Không có audit trail (lịch sử thay đổi)
- Khó reproduce (tái tạo) khi cần lập Jenkins mới

**Ví dụ file `jenkins.yaml`:**

```yaml
jenkins:
  systemMessage: "Jenkins - Production Environment"
  numExecutors: 0
  agentProtocols:
    - "JNLP4-connect"

  securityRealm:
    ldap:
      configurations:
        - server: "ldap://corp.example.com"
          rootDN: "dc=example,dc=com"
          userSearchBase: "ou=Users"

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            assignments:
              - "jenkins-admins"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              id: "docker-registry"
              username: "ci-bot"
              password: "${DOCKER_REGISTRY_PASSWORD}"
```

**Lợi ích:**
- **GitOps-ready:** Cấu hình review qua Pull Request
- **Idempotent (lũy đẳng):** Apply nhiều lần cho cùng kết quả
- **Disaster Recovery (khôi phục thảm họa):** Restore Jenkins từ file YAML trong vài phút
- **Audit trail:** Toàn bộ thay đổi có lịch sử trong Git

---

## Nhóm 2: Pipeline và Jenkinsfile

### Q5. Declarative Pipeline và Scripted Pipeline khác nhau thế nào? Khi nào dùng cái nào?

**Mức độ:** Cơ bản — hầu như 100% phỏng vấn hỏi.

**Gợi ý trả lời:**

| Tiêu Chí               | Declarative Pipeline               | Scripted Pipeline                    |
| ---------------------- | ---------------------------------- | ------------------------------------ |
| Cú pháp                | Cấu trúc cố định, dễ đọc          | Groovy DSL tự do, linh hoạt          |
| Học dễ                 | Dễ hơn — có sẵn cấu trúc          | Khó hơn — cần biết Groovy           |
| Error handling         | `post { failure {} }` tích hợp sẵn | `try-catch-finally` thủ công        |
| Validation sớm         | Lỗi cú pháp phát hiện trước khi chạy | Lỗi chỉ thấy khi chạy đến dòng đó |
| Tính linh hoạt         | Thấp hơn — có giới hạn cú pháp    | Cao hơn — mọi Groovy đều dùng được  |
| Khuyến nghị            | **Dùng cho hầu hết trường hợp**   | Dùng khi cần logic phức tạp         |

**Declarative Pipeline (pipeline khai báo):**

```groovy
pipeline {
    agent { label 'linux' }

    environment {
        APP_VERSION = '1.0.0'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'mvn test' }
                }
                stage('Integration Tests') {
                    steps { sh 'mvn verify -Pintegration' }
                }
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'helm upgrade --install myapp ./chart'
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', message: "Build ${env.BUILD_NUMBER} succeeded")
        }
        failure {
            slackSend(color: 'danger', message: "Build ${env.BUILD_NUMBER} FAILED")
        }
        always {
            cleanWs()
        }
    }
}
```

**Scripted Pipeline (pipeline kịch bản):**

```groovy
node('linux') {
    def appVersion = '1.0.0'

    try {
        stage('Build') {
            sh 'mvn clean package -DskipTests'
        }

        stage('Test') {
            // Logic điều kiện phức tạp hơn
            def testResults = [:]
            ['unit', 'integration', 'e2e'].each { testType ->
                testResults[testType] = {
                    sh "mvn test -P${testType}"
                }
            }
            parallel testResults
        }

        stage('Deploy') {
            if (env.BRANCH_NAME == 'main') {
                sh 'helm upgrade --install myapp ./chart'
            }
        }
    } catch (Exception e) {
        slackSend(color: 'danger', message: "Build FAILED: ${e.message}")
        throw e
    } finally {
        cleanWs()
    }
}
```

**Khi nào dùng Scripted:**
- Logic điều kiện phức tạp mà Declarative `when` không đáp ứng được
- Dynamic stage generation (tạo stage động từ danh sách)
- Cần tái sử dụng biến Groovy giữa các stage với scope phức tạp

---

### Q6. Shared Libraries là gì? Cấu trúc như thế nào?

**Mức độ:** Trung bình - Nâng cao — hỏi nhiều ở vị trí Senior.

**Gợi ý trả lời:**

**Shared Libraries** (thư viện dùng chung) là Groovy code tái sử dụng, lưu trong Git repository riêng, được nhiều Jenkins Pipeline import và dùng chung.

**Vấn đề cần giải quyết:**
- 50 teams, mỗi team copy-paste cùng logic Docker build → thay đổi 1 chỗ phải sửa 50 Jenkinsfile
- Không có chuẩn hóa quy trình CI/CD toàn tổ chức

**Cấu trúc thư mục chuẩn:**

```
jenkins-shared-library/         ← Git repo riêng
├── vars/                       ← Global Variables (biến toàn cục)
│   ├── dockerBuild.groovy      ← Gọi: dockerBuild.buildAndPush(...)
│   ├── helmDeploy.groovy       ← Gọi: helmDeploy(...)
│   └── notifySlack.groovy      ← Gọi: notifySlack('success')
│
├── src/                        ← Groovy Classes (lớp Groovy)
│   └── com/
│       └── example/
│           └── PipelineHelper.groovy
│
└── resources/                  ← Static files (file tĩnh)
    └── scripts/
        └── setup-env.sh
```

**Ví dụ `vars/dockerBuild.groovy`:**

```groovy
def buildAndPush(String imageName, String tag, String registry = 'docker.io') {
    def fullImageName = "${registry}/${imageName}:${tag}"

    docker.withRegistry("https://${registry}", 'docker-registry-credentials') {
        def image = docker.build(fullImageName)
        image.push()
        image.push('latest')
    }

    echo "Pushed: ${fullImageName}"
    return fullImageName
}
```

**Cách dùng trong Jenkinsfile:**

```groovy
@Library('my-shared-lib@v2.0') _  // import thư viện, version tag v2.0

pipeline {
    agent any
    stages {
        stage('Build & Push') {
            steps {
                script {
                    dockerBuild.buildAndPush('myapp', env.BUILD_NUMBER)
                }
            }
        }
    }
}
```

**Điểm mấu chốt để gây ấn tượng:**
- Shared Libraries nên có **version tag** (thẻ phiên bản) để tránh breaking changes
- Nên có unit test bằng [JenkinsPipelineUnit](https://github.com/jenkinsci/JenkinsPipelineUnit)
- Cấu hình trong Jenkins: Manage Jenkins → System → Global Pipeline Libraries

---

### Q7. Multibranch Pipeline hoạt động như thế nào?

**Mức độ:** Trung bình.

**Gợi ý trả lời:**

**Multibranch Pipeline** (pipeline đa nhánh) tự động phát hiện nhánh Git và tạo Pipeline riêng cho mỗi nhánh có Jenkinsfile.

**Luồng hoạt động:**

```
Git Repository
├── main          → Jenkins tạo job "project/main"
├── develop       → Jenkins tạo job "project/develop"
├── feature/auth  → Jenkins tạo job "project/feature%2Fauth"
└── PR #42        → Jenkins tạo job "project/PR-42"
```

**Lợi ích:**

1. **Tự động hóa:** Tạo nhánh mới → Jenkins tự tạo Pipeline, không cần cấu hình thủ công
2. **PR Validation (kiểm tra PR):** Tự động chạy build và test trên mỗi Pull Request
3. **Branch-specific behavior (hành vi theo nhánh):** Dùng `when { branch 'main' }` để chỉ deploy từ nhánh main

**Ví dụ Jenkinsfile với logic theo nhánh:**

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'mvn package' }
        }
        stage('Test') {
            steps { sh 'mvn test' }
        }
        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh 'helm upgrade --install myapp-staging ./chart --set env=staging'
            }
        }
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh 'helm upgrade --install myapp ./chart --set env=prod'
            }
        }
    }
}
```

**Branch Filtering (lọc nhánh):** Có thể cấu hình chỉ tạo Pipeline cho nhánh phù hợp regex, tránh tạo quá nhiều job.

---

### Q8. `stash` và `unstash` dùng để làm gì?

**Mức độ:** Trung bình.

**Gợi ý trả lời:**

**Stash/Unstash** cho phép truyền file giữa các stage hoặc giữa các Agent trong cùng một Pipeline.

**Vấn đề:**
- Stage "Build" chạy trên Agent A, tạo ra file `.jar`
- Stage "Test" cần file `.jar` đó nhưng có thể chạy trên Agent B (khác workspace)

**Giải pháp với stash:**

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'build-server' }
            steps {
                sh 'mvn package -DskipTests'
                stash name: 'build-artifact', includes: 'target/*.jar'
            }
        }
        stage('Test') {
            agent { label 'test-server' }
            steps {
                unstash 'build-artifact'    // lấy file từ stash
                sh 'mvn test'
            }
        }
        stage('Deploy') {
            agent { label 'deploy-server' }
            steps {
                unstash 'build-artifact'
                sh 'scp target/*.jar deploy-host:/apps/'
            }
        }
    }
}
```

**Stash vs Artifact:**
- **Stash:** Tạm thời, chỉ dùng trong pipeline hiện tại, xóa sau khi pipeline kết thúc
- **Archive Artifact:** Lưu lâu dài, truy cập qua UI, tải về sau nhiều ngày

---

### Q9. `input` step dùng để làm gì? Rủi ro và cách xử lý?

**Mức độ:** Trung bình.

**Gợi ý trả lời:**

**`input` step** tạm dừng Pipeline và chờ người dùng xác nhận thủ công — thường dùng cho approval trước khi deploy production.

```groovy
stage('Deploy Production') {
    steps {
        input(
            message: 'Deploy version ${env.BUILD_NUMBER} to Production?',
            ok: 'Deploy Now',
            submitter: 'devops-team,release-manager',  // chỉ những người này được approve
            parameters: [
                choice(
                    name: 'DEPLOY_REGION',
                    choices: ['us-east-1', 'eu-west-1'],
                    description: 'Target region'
                )
            ]
        )
    }
}
```

**Rủi ro và cách xử lý:**

| Rủi Ro                                  | Cách Xử Lý                                      |
| --------------------------------------- | ----------------------------------------------- |
| Pipeline bị block, chiếm Executor       | Dùng `timeout(time: 24, unit: 'HOURS')` bên ngoài `input` |
| Agent bị lock suốt thời gian chờ       | Đặt `input` trong stage `agent none` riêng     |
| Không biết ai đang chờ approve         | Gửi Slack notification khi đến bước input      |

**Pattern tốt nhất — input không chiếm Agent:**

```groovy
stage('Approval') {
    agent none              // không cần agent khi chờ
    steps {
        timeout(time: 4, unit: 'HOURS') {
            input message: 'Deploy to production?'
        }
    }
}
stage('Deploy') {
    agent { label 'deploy' }
    steps {
        sh 'helm upgrade ...'
    }
}
```

---

## Nhóm 3: Distributed Builds và Agent

### Q10. Sự khác biệt giữa JNLP Agent và SSH Agent?

**Mức độ:** Trung bình.

**Gợi ý trả lời:**

| Tiêu Chí                  | JNLP Agent                              | SSH Agent                              |
| ------------------------- | --------------------------------------- | -------------------------------------- |
| Kết nối                   | Agent **chủ động** kết nối ra Master    | Master **SSH vào** Agent              |
| Firewall-friendly         | ✅ Agent chỉ cần outbound (ra ngoài)   | ❌ Master cần SSH inbound vào Agent   |
| Cấu hình                  | Cài agent.jar, chạy lệnh kết nối       | Chỉ cần SSH key và Java trên Agent    |
| Kubernetes                | ✅ Phù hợp (Pod tự kết nối ra)         | ❌ Phức tạp hơn với Pod              |
| On-premise Agent          | Dùng được                               | ✅ Phổ biến hơn cho server vật lý    |

**JNLP Agent** (Java Network Launch Protocol Agent):

```bash
# Lệnh chạy trên Agent để kết nối Master
java -jar agent.jar \
  -jnlpUrl http://jenkins-master:8080/computer/agent-1/slave-agent.jnlp \
  -secret <secret-token> \
  -workDir /var/jenkins-workspace
```

**SSH Agent:** Cấu hình trong Jenkins UI: Manage Nodes → New Node → Launch method: SSH.

**Khuyến nghị:**
- **Cloud/Kubernetes:** Dùng JNLP (Kubernetes Pod chủ động kết nối ra)
- **On-premise server:** Dùng SSH (đơn giản, ổn định hơn)

---

### Q11. Kubernetes Agent hoạt động như thế nào? Lợi ích so với Static Agent?

**Mức độ:** Trung bình - Nâng cao.

**Gợi ý trả lời:**

**Kubernetes Agent** (agent động trên Kubernetes) sử dụng Kubernetes Plugin để tạo Pod mới cho mỗi build, xóa Pod sau khi build xong.

**Luồng hoạt động:**

```
1. Jenkins nhận job mới
2. Kubernetes Plugin gọi Kubernetes API → tạo Pod mới
3. Pod khởi động JNLP container → kết nối về Jenkins Master
4. Jenkins giao build cho Pod
5. Build chạy trong Pod
6. Build xong → Pod bị xóa tự động
```

**Pod Template (mẫu Pod) — ví dụ Declarative Pipeline:**

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ["sleep"]
    args: ["99d"]
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "2"
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:latest .'
                }
            }
        }
    }
}
```

**Lợi ích so với Static Agent:**

| Tiêu Chí           | Static Agent              | Kubernetes Agent                  |
| ------------------ | ------------------------- | --------------------------------- |
| Chi phí            | Trả tiền 24/7             | Trả tiền khi build (pay-as-you-go)|
| Cấu hình môi trường| Cố định, dễ bị drift      | Fresh container mỗi build        |
| Scale              | Thêm/xóa thủ công         | Tự động scale theo queue          |
| Isolation          | Shared giữa các build      | Hoàn toàn isolated per build     |
| Khởi động          | Nhanh (đã sẵn sàng)        | Chậm hơn (~30s để Pod start)     |

---

### Q12. Docker-in-Docker vs Docker Socket Mounting — khác nhau thế nào?

**Mức độ:** Nâng cao.

**Gợi ý trả lời:**

Khi cần build Docker image từ trong Jenkins Pipeline chạy trên container/Pod:

**Docker-in-Docker — DinD (Docker trong Docker):**

```yaml
# Docker daemon chạy bên trong container
containers:
- name: dind
  image: docker:24-dind
  securityContext:
    privileged: true   # cần quyền privileged — đây là điểm yếu bảo mật
  env:
  - name: DOCKER_TLS_CERTDIR
    value: ""
```

**Docker Socket Mounting (gắn socket Docker):**

```yaml
# Dùng Docker daemon của host node
containers:
- name: docker
  image: docker:24-cli
  volumeMounts:
  - name: docker-sock
    mountPath: /var/run/docker.sock
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
```

**So sánh:**

| Tiêu Chí           | DinD                          | Socket Mounting                    |
| ------------------ | ----------------------------- | ---------------------------------- |
| Isolation          | ✅ Hoàn toàn isolated         | ❌ Dùng chung daemon với host     |
| Bảo mật            | ❌ Cần `privileged: true`     | ❌ Container có quyền như root host|
| Hiệu suất          | Chậm hơn (nested daemon)      | ✅ Nhanh hơn (daemon của host)    |
| Đơn giản           | Phức tạp hơn                  | ✅ Đơn giản hơn                   |

**Khuyến nghị hiện đại:** Dùng **Kaniko** hoặc **Buildah** — build Docker image mà không cần Docker daemon, không cần privileged mode.

---

## Nhóm 4: Security và Credentials

### Q13. Các loại Credentials trong Jenkins và cách dùng an toàn trong Pipeline?

**Mức độ:** Trung bình — gần như 100% phỏng vấn hỏi về security.

**Gợi ý trả lời:**

**Các loại Credentials (thông tin xác thực) trong Jenkins:**

| Loại                  | Dùng Cho                                    |
| --------------------- | ------------------------------------------- |
| Secret text           | API token, password đơn                     |
| Username/Password     | Docker registry, database, SVN              |
| SSH Username/Private Key | Git SSH, server SSH                      |
| Certificate           | Code signing, mutual TLS                    |
| Secret file           | kubeconfig, service account JSON key        |

**Cách dùng an toàn trong Declarative Pipeline:**

```groovy
pipeline {
    agent any
    environment {
        // Bind credentials vào environment variables
        DOCKER_CREDS = credentials('docker-registry')   // tạo DOCKER_CREDS_USR và DOCKER_CREDS_PSW
        SONAR_TOKEN = credentials('sonarqube-token')    // Secret text → biến SONAR_TOKEN
    }
    stages {
        stage('Docker Login') {
            steps {
                sh 'echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
                }
            }
        }
    }
}
```

**Các lỗi bảo mật cần tránh:**

```groovy
// ❌ SAI — in secret ra log
echo "Password is: ${DOCKER_CREDS_PSW}"

// ❌ SAI — hardcode credential
sh 'docker login -u myuser -p mypassword'

// ❌ SAI — lưu credential trong environment variable rồi print env
sh 'env'   // in ra toàn bộ env, lộ secret

// ✅ ĐÚNG — dùng withCredentials với scope nhỏ nhất có thể
withCredentials([...]) {
    sh 'lệnh cần dùng credential này'
}
```

**Jenkins tự động mask (che) giá trị credentials trong log:** Nếu accidently in ra, Jenkins thay bằng `****`.

---

### Q14. Matrix-based Security và Role Strategy Plugin — khác nhau thế nào?

**Mức độ:** Trung bình.

**Gợi ý trả lời:**

**Matrix-based Security (bảo mật dựa trên ma trận):**
- Cấu hình trực tiếp quyền cho từng user hoặc group
- Jenkins tích hợp sẵn, không cần plugin
- Phù hợp: team nhỏ, số lượng user ít

```
Matrix:
            Admin  | Read  | Build | Deploy
alice         ✅    |  ✅   |  ✅   |  ✅
bob           ❌    |  ✅   |  ✅   |  ❌
ci-bot        ❌    |  ✅   |  ✅   |  ❌
```

**Role Strategy Plugin (plugin chiến lược vai trò):**
- Định nghĩa Role (vai trò), gán user vào Role
- Hỗ trợ Project-based role: quyền giới hạn theo từng job/folder
- Phù hợp: tổ chức lớn, nhiều team, nhiều project

```
Roles:
- "developer":     Read, Build jobs trong folder team-name/*
- "team-lead":     Read, Build, Configure jobs trong folder team-name/*
- "platform-eng":  Read, Build, Configure tất cả jobs
- "admin":         Overall/Administer

Assignments:
- alice, bob     → role "developer" trong project "backend/*"
- charlie        → role "team-lead" trong project "backend/*"
- platform-team  → role "platform-eng"
```

**Khuyến nghị:** Với tổ chức > 10 người, dùng Role Strategy Plugin + tích hợp LDAP/AD để quản lý user tập trung.

---

### Q15. Script Security Plugin và Groovy Sandbox là gì?

**Mức độ:** Trung bình - Nâng cao.

**Gợi ý trả lời:**

**Groovy Sandbox** (hộp cát Groovy) giới hạn các API Groovy/Java mà Pipeline script có thể gọi, ngăn code độc hại truy cập file system, network, hoặc thực thi shell.

**Cơ chế hoạt động:**

```
Pipeline Script
      │
      ▼
 Groovy Sandbox ── kiểm tra từng method call
      │
      ├── Approved (đã phê duyệt) → cho phép chạy
      └── Not approved → throw RejectedAccessException
                              │
                              ▼
                   Admin vào ScriptApproval để phê duyệt
```

**Script Security có 2 chế độ:**

1. **Sandbox = ON (mặc định):** Code chạy trong sandbox, method lạ cần admin approve
2. **Sandbox = OFF:** Script chạy với toàn quyền — **CHỈ dùng cho Trusted Administrator**

**ScriptApproval (phê duyệt script):** Khi Sandbox block một method, admin vào *Manage Jenkins > In-Process Script Approval* để approve.

**Lưu ý bảo mật:** Không tắt Sandbox cho pipeline của người dùng thông thường — đây là lỗ hổng bảo mật nghiêm trọng.

---

## Nhóm 5: Thực Tế và Troubleshooting

### Q16. Khi build fail, bạn debug theo quy trình nào?

**Mức độ:** Thực tế — Hay bị hỏi theo dạng behavioral.

**Gợi ý trả lời:**

**Quy trình debug có hệ thống (5 bước):**

**Bước 1 — Đọc log từ dưới lên:**
```
Console Output → cuộn xuống dưới cùng → tìm dòng ERROR đầu tiên
Tìm exit code: "Process exited with code 1" → lỗi ứng dụng
              "Process exited with code 137" → OOM Kill (hết bộ nhớ)
              "Process exited with code 143" → Timeout, bị kill
```

**Bước 2 — Xác định stage fail:**
```
Pipeline View → stage màu đỏ → click vào stage đó → xem log của stage
```

**Bước 3 — Phân loại lỗi:**

| Triệu Chứng                    | Nguyên Nhân Hay Gặp                          |
| ------------------------------ | -------------------------------------------- |
| "No such file or directory"    | Workspace không có file, stash chưa unstash  |
| "Permission denied"            | File không có execute permission, credential thiếu |
| "Cannot connect to Docker"     | Docker daemon không chạy, socket permission  |
| "OutOfMemoryError"             | JVM heap không đủ, process bị OOM Kill       |
| "Agent went offline"           | Network issue, agent crash, idle timeout     |

**Bước 4 — Kiểm tra thay đổi gần đây:**
```
Git log → commit nào gây ra?
Plugin update → plugin mới có break gì không?
Infrastructure change → agent mới, image mới?
```

**Bước 5 — Reproduce và fix:**
```
Replay build với Jenkinsfile sửa nhỏ (không cần commit)
Jenkins UI → Build History → Click build fail → Replay
```

---

### Q17. Agent mất kết nối giữa chừng build — xử lý thế nào?

**Mức độ:** Thực tế.

**Gợi ý trả lời:**

**Triệu chứng:** Build đang chạy, đột nhiên log dừng, pipeline báo `Agent went offline during build`.

**Nguyên nhân hay gặp và cách kiểm tra:**

```
1. Network timeout (timeout mạng):
   - Kiểm tra: xem log agent, ping từ agent về master
   - Fix: tăng TCP keepalive, kiểm tra firewall rules

2. Agent process crash (agent bị crash):
   - Kiểm tra: SSH vào agent, xem process java, system log
   - Fix: kiểm tra OOM Kill (dmesg | grep -i kill), tăng RAM agent

3. Idle timeout (agent tự ngắt kết nối):
   - Kiểm tra: Jenkins > Manage Nodes > Node config > Idle connection timeout
   - Fix: tăng timeout hoặc tắt (set = 0 nếu agent dedicated)

4. Docker container stop (nếu agent là container):
   - Kiểm tra: docker ps -a, docker logs <container>
   - Fix: tăng container resource limits, kiểm tra OOM

5. Spot/Preemptible instance bị thu hồi (cloud):
   - Xử lý: dùng Retry step, checkpointing build
```

**Cách cấu hình retry tự động:**

```groovy
pipeline {
    agent { label 'linux' }
    options {
        retry(3)   // retry toàn bộ pipeline tối đa 3 lần
    }
    stages {
        stage('Build') {
            options {
                retry(2)   // retry stage này riêng 2 lần
            }
            steps {
                sh 'mvn package'
            }
        }
    }
}
```

---

### Q18. Jenkins chạy chậm dần theo thời gian — nguyên nhân và giải pháp?

**Mức độ:** Thực tế - Nâng cao.

**Gợi ý trả lời:**

**Nguyên nhân phổ biến:**

**1. JVM Heap quá nhỏ (bộ nhớ JVM không đủ):**
```bash
# Triệu chứng: GC overhead warning, UI lag
# Kiểm tra: Jenkins UI > Manage Jenkins > System Information > java.vm.free.memory
# Fix: tăng heap size
export JAVA_OPTS="-Xms2g -Xmx8g -XX:+UseG1GC"
```

**2. Build History tích lũy quá nhiều (lịch sử build quá lớn):**
```groovy
// Cấu hình trong Jenkinsfile
options {
    buildDiscarder(logRotator(
        numToKeepStr: '30',         // giữ 30 build gần nhất
        daysToKeepStr: '14',        // hoặc 14 ngày gần nhất
        artifactNumToKeepStr: '5'   // giữ artifact của 5 build
    ))
}
```

**3. Workspace không được dọn dẹp (workspace tích lũy chiếm disk):**
```groovy
post {
    always {
        cleanWs()   // dọn workspace sau mỗi build
    }
}
```

**4. Quá nhiều Plugin chạy đồng thời:**
- Vào Manage Jenkins → Manage Plugins → Installed
- Disable plugin không dùng đến

**5. Pipeline Groovy script tốn bộ nhớ:**
```groovy
// ❌ Tốn bộ nhớ — lưu list lớn trong Groovy variable
def allFiles = sh(returnStdout: true, script: 'find . -name "*.class"').split('\n')

// ✅ Tốt hơn — xử lý trực tiếp trong shell
sh 'find . -name "*.class" | xargs rm -f'
```

**Checklist kiểm tra hiệu suất:**
- [ ] JVM heap ≥ 4GB cho production Jenkins
- [ ] Build Discard Policy đã cấu hình cho tất cả job
- [ ] Workspace Cleanup Plugin được dùng
- [ ] Disk space agent > 20% trống
- [ ] Plugin count < 100 (nhiều hơn → review và disable)

---

## Nhóm 6: Nâng Cao và So Sánh

### Q19. Jenkins vs GitHub Actions — khi nào dùng cái nào?

**Mức độ:** Nâng cao — hay hỏi ở Senior positions.

**Gợi ý trả lời:**

| Tiêu Chí                    | Jenkins                                 | GitHub Actions                         |
| --------------------------- | --------------------------------------- | --------------------------------------- |
| Self-hosted                 | ✅ Fully self-hosted                   | Cả cloud (GitHub-hosted) và self-hosted |
| Cost (chi phí)              | Infrastructure cost + maintenance      | Free cho public repo; tính phút private |
| Setup                       | Phức tạp hơn                           | ✅ Đơn giản — YAML trong repo          |
| Plugin ecosystem            | ✅ 1800+ plugins                       | ✅ GitHub Marketplace (Actions)         |
| Kubernetes integration      | ✅ Kubernetes Plugin, JCasC            | Cần tự setup self-hosted runner        |
| On-premise support          | ✅ Excellent                           | Limited (cần self-hosted runner)        |
| Custom build environment    | ✅ Docker agent linh hoạt             | ✅ Docker container actions             |
| Secret management           | Jenkins Credentials Store              | GitHub Secrets                          |
| Maintenance overhead        | ❌ Cao — cần team quản lý             | ✅ Thấp — GitHub quản lý              |
| Complex pipelines           | ✅ Shared Libraries rất mạnh          | Reusable workflows, composite actions   |

**Khi nào chọn Jenkins:**
- Tổ chức có on-premise infrastructure hoặc private cloud
- Cần build trên nhiều loại OS, hardware đặc biệt
- Đã có team Jenkins với Shared Libraries ecosystem
- Compliance yêu cầu code không đi qua cloud third-party

**Khi nào chọn GitHub Actions:**
- Startup hoặc team nhỏ, muốn setup nhanh
- Codebase đã host trên GitHub
- Không muốn maintain Jenkins infrastructure
- Workload không quá phức tạp

**Câu trả lời cân bằng:** "Không có lựa chọn tốt hơn tuyệt đối — phụ thuộc vào infrastructure hiện tại, quy mô team, và yêu cầu compliance."

---

### Q20. Thiết kế Jenkins High Availability — làm thế nào?

**Mức độ:** Nâng cao — hỏi ở Senior DevOps / Platform Engineer.

**Gợi ý trả lời:**

**Jenkins HA (High Availability — Tính Sẵn Sàng Cao) là thách thức vì Jenkins Master được thiết kế theo kiểu single-node truyền thống.**

**Giải pháp 1: Active-Passive với Shared Storage**

```
Load Balancer
      │
 ┌────┴────┐
 │         │
Jenkins   Jenkins
Primary   Standby  (đang tắt, chỉ bật khi Primary fail)
 │         │
 └────┬────┘
      │
  NFS / EFS / Azure Files
  (shared JENKINS_HOME)
```

- Primary xử lý tất cả traffic
- Standby sync JENKINS_HOME qua shared storage
- Failover: thủ công hoặc tự động qua health check

**Giải pháp 2: Kubernetes Deployment với Persistent Volume**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1   # chỉ 1 replica (Jenkins không hỗ trợ multi-master)
  strategy:
    type: Recreate   # xóa pod cũ trước khi tạo pod mới
  template:
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
```

Kubernetes tự động restart Pod khi fail → downtime ~30-60 giây (RTO — Recovery Time Objective).

**Giải pháp 3: CloudBees CI (Commercial)**

CloudBees cung cấp **Operations Center** (trung tâm vận hành) cho phép nhiều Jenkins Controller hoạt động cùng nhau với HA thực sự.

**Điểm mấu chốt khi trả lời:**

1. Jenkins không hỗ trợ native multi-master clustering
2. Giải pháp phổ biến nhất: Kubernetes + PVC → downtime ngắn khi fail
3. Backup JENKINS_HOME thường xuyên là bắt buộc
4. **RTO vs RPO:** Cần xác định rõ: "downtime tối đa chấp nhận được là bao lâu?" và "mất tối đa bao nhiêu dữ liệu?"

---

## Bảng Tổng Hợp

| Câu | Chủ Đề                     | Mức Độ         | Tần Suất |
| --- | -------------------------- | -------------- | -------- |
| Q1  | Master/Agent Architecture  | Cơ bản         | ⭐⭐⭐⭐⭐ |
| Q2  | Executor Configuration     | Cơ bản         | ⭐⭐⭐    |
| Q3  | Build Queue                | Cơ bản         | ⭐⭐⭐    |
| Q4  | JCasC                      | Nâng cao       | ⭐⭐⭐    |
| Q5  | Declarative vs Scripted    | Cơ bản         | ⭐⭐⭐⭐⭐ |
| Q6  | Shared Libraries           | Nâng cao       | ⭐⭐⭐⭐  |
| Q7  | Multibranch Pipeline       | Trung bình     | ⭐⭐⭐⭐  |
| Q8  | Stash/Unstash              | Trung bình     | ⭐⭐⭐    |
| Q9  | Input Step                 | Trung bình     | ⭐⭐⭐    |
| Q10 | JNLP vs SSH Agent          | Trung bình     | ⭐⭐⭐⭐  |
| Q11 | Kubernetes Agent           | Nâng cao       | ⭐⭐⭐⭐  |
| Q12 | DinD vs Socket Mounting    | Nâng cao       | ⭐⭐⭐    |
| Q13 | Credentials Types & Usage  | Trung bình     | ⭐⭐⭐⭐⭐ |
| Q14 | Matrix-based vs Role Strategy | Trung bình  | ⭐⭐⭐    |
| Q15 | Script Security & Sandbox  | Nâng cao       | ⭐⭐⭐    |
| Q16 | Debug Build Failure        | Thực tế        | ⭐⭐⭐⭐⭐ |
| Q17 | Agent Disconnect           | Thực tế        | ⭐⭐⭐⭐  |
| Q18 | Jenkins Performance        | Nâng cao       | ⭐⭐⭐⭐  |
| Q19 | Jenkins vs GitHub Actions  | So sánh        | ⭐⭐⭐⭐  |
| Q20 | Jenkins High Availability  | Nâng cao       | ⭐⭐⭐    |

---

**Xem tiếp:**
- [1-star-stories.md](1-star-stories.md) — Câu chuyện STAR thực tế
- [2-system-design-scenarios.md](2-system-design-scenarios.md) — Bài toán System Design CI/CD
