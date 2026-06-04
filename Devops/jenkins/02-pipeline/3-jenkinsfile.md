# Jenkinsfile — Quản Lý Pipeline Trong SCM

> Jenkinsfile là file text lưu cấu hình pipeline Jenkins bên trong repository. Đây là nền tảng của **Pipeline as Code** — cách tiếp cận hiện đại để quản lý CI/CD. File này trình bày cách tổ chức, quản lý version, và các best practices cho Jenkinsfile.

## Mục Lục

1. [Jenkinsfile là gì?](#jenkinsfile-là-gì)
2. [Vị Trí và Cấu Trúc File](#vị-trí-và-cấu-trúc-file)
3. [Cấu Hình Job Để Dùng Jenkinsfile](#cấu-hình-job-để-dùng-jenkinsfile)
4. [Quản Lý Version Jenkinsfile](#quản-lý-version-jenkinsfile)
5. [Biến Môi Trường Và Secret](#biến-môi-trường-và-secret)
6. [Tổ Chức Jenkinsfile Lớn](#tổ-chức-jenkinsfile-lớn)
7. [Kiểm Tra Cú Pháp Jenkinsfile](#kiểm-tra-cú-pháp-jenkinsfile)
8. [Best Practices](#best-practices)
9. [Anti-Patterns — Những Điều Cần Tránh](#anti-patterns--những-điều-cần-tránh)
10. [Ví Dụ Theo Loại Ứng Dụng](#ví-dụ-theo-loại-ứng-dụng)

---

## Jenkinsfile là gì?

**Jenkinsfile** là file định nghĩa Jenkins Pipeline, được lưu trong repository cùng với source code ứng dụng. Jenkins đọc file này để biết cách build, test, và deploy ứng dụng.

```
my-application/
├── src/
│   └── main/java/...
├── tests/
├── Dockerfile
├── helm/
├── pom.xml
└── Jenkinsfile          ← Pipeline definition ở đây
```

### Lợi Ích Của Jenkinsfile

| Lợi Ích                    | Mô Tả                                                             |
| -------------------------- | ----------------------------------------------------------------- |
| **Version Control**        | Mọi thay đổi pipeline đều có lịch sử git                         |
| **Code Review**            | Pipeline thay đổi phải qua Pull Request — không ai thay đổi UI ngầm |
| **Reproducibility**        | Checkout commit cũ → có pipeline của thời điểm đó                |
| **Branching**              | Mỗi nhánh có pipeline riêng, phù hợp với từng giai đoạn          |
| **Developer Ownership**    | Dev tự quản lý pipeline của ứng dụng mình                        |
| **Disaster Recovery**      | Mất Jenkins → tạo lại job, point đến repo → pipeline hoạt động   |

---

## Vị Trí và Cấu Trúc File

### Vị Trí Mặc Định

Mặc định, Jenkins tìm `Jenkinsfile` ở **thư mục gốc** của repository:

```
repo-root/
└── Jenkinsfile       ← Vị trí mặc định
```

### Tên File Tùy Chỉnh

Có thể đặt tên khác hoặc đặt trong thư mục con — cấu hình trong Jenkins job:

```
repo-root/
├── ci/
│   ├── Jenkinsfile.build       ← Pipeline cho build
│   ├── Jenkinsfile.release     ← Pipeline cho release
│   └── Jenkinsfile.nightly     ← Pipeline chạy đêm
└── Jenkinsfile                 ← Pipeline chính (main CI/CD)
```

### Cấu Trúc File Cơ Bản

```groovy
// Jenkinsfile — Pipeline as Code
// Mọi thay đổi ở đây phải qua code review

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        APP_NAME = 'my-application'
    }

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
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh './ci/deploy.sh'
            }
        }
    }

    post {
        failure {
            mail to: "${env.CHANGE_AUTHOR_EMAIL ?: 'devops@example.com'}",
                 subject: "Build Failed: ${env.JOB_NAME}",
                 body: "${env.BUILD_URL}"
        }
    }
}
```

---

## Cấu Hình Job Để Dùng Jenkinsfile

### Pipeline Job Đơn Lẻ

1. Tạo job mới: **New Item → Pipeline**
2. Trong phần **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/org/repo.git`
   - Credentials: Chọn credentials đã cấu hình
   - Branch: `*/main` hoặc `*/master`
   - Script Path: `Jenkinsfile` (hoặc đường dẫn tùy chỉnh)

### Multibranch Pipeline (Nhiều Nhánh)

Tự động tạo pipeline cho mỗi nhánh có `Jenkinsfile`:

1. Tạo job: **New Item → Multibranch Pipeline**
2. Cấu hình **Branch Sources**: GitHub/GitLab/Bitbucket
3. **Build Configuration**: Script Path = `Jenkinsfile`
4. Jenkins tự động quét repository và tạo job cho mỗi nhánh

---

## Quản Lý Version Jenkinsfile

### Chiến Lược Nhánh

```
main          → Jenkinsfile dùng cho production deploy
│
├── develop   → Jenkinsfile dùng cho staging deploy
│
├── feature/* → Jenkinsfile chỉ build + test, không deploy
│
└── release/* → Jenkinsfile dùng cho release candidate
```

### Kiểm Soát Theo Nhánh Trong Cùng Jenkinsfile

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

        // Feature branches: chỉ build + test — không deploy
        stage('Deploy to Review') {
            when {
                changeRequest()    // Chạy khi đây là Pull Request
            }
            steps {
                sh "./deploy.sh review-${env.CHANGE_ID}"
            }
        }

        // Develop branch: deploy lên dev environment
        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps { sh './deploy.sh dev' }
        }

        // Release branches: deploy lên staging
        stage('Deploy to Staging') {
            when {
                anyOf {
                    branch 'release/*'
                    branch 'main'
                }
            }
            steps { sh './deploy.sh staging' }
        }

        // Chỉ main: deploy production (với approval)
        stage('Production Approval') {
            when { branch 'main' }
            steps {
                input message: 'Deploy lên Production?',
                      ok:      'Phê Duyệt',
                      submitter: 'ops-team'
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            steps { sh './deploy.sh production' }
        }
    }
}
```

---

## Biến Môi Trường Và Secret

### Nguyên Tắc Bảo Mật Cho Jenkinsfile

**KHÔNG BAO GIỜ** lưu secret trực tiếp trong Jenkinsfile:

```groovy
// SAI — Secret để lộ trong git history
environment {
    DB_PASSWORD = 'my-super-secret-password'    // ❌ CỰC KỲ NGUY HIỂM
    API_KEY     = 'sk-abc123xyz'                 // ❌ KHÔNG BAO GIỜ LÀM VẬY
}
```

Luôn dùng Jenkins Credentials:

```groovy
// ĐÚNG — Secret lưu trong Jenkins Credential Store
environment {
    // Credentials ID phải được tạo sẵn trong Jenkins
    DB_PASSWORD   = credentials('database-password')      // Secret text
    DOCKER_CREDS  = credentials('docker-registry')        // Username/Password
    DEPLOY_SSH    = credentials('deploy-ssh-key')         // SSH Private Key
}
```

### Phân Tầng Cấu Hình

```groovy
environment {
    // Cấp 1: Hằng số (constants) — lưu trực tiếp trong Jenkinsfile ✅
    APP_NAME    = 'payment-service'
    BUILD_TOOL  = 'maven'
    JAVA_VER    = '17'

    // Cấp 2: Cấu hình môi trường — lưu trong Jenkins global properties ✅
    // (Manage Jenkins → Configure System → Global properties → Environment variables)
    // REGISTRY = env.COMPANY_REGISTRY  ← đã set ở Jenkins global
    // SONAR_URL = env.COMPANY_SONAR

    // Cấp 3: Secret — luôn dùng Credentials ✅
    REGISTRY_CREDS = credentials('registry-login')
    SONAR_TOKEN    = credentials('sonarqube-token')
    KUBECONFIG_STG = credentials('staging-kubeconfig')
    KUBECONFIG_PRD = credentials('production-kubeconfig')
}
```

### Truyền Cấu Hình Qua File

Thay vì hardcode cấu hình vào Jenkinsfile, đọc từ file cấu hình trong repo:

```groovy
// Đọc cấu hình từ file YAML
stage('Load Config') {
    steps {
        script {
            def config = readYaml file: 'ci/config.yml'
            env.DEPLOY_NAMESPACE  = config.deploy.namespace
            env.REPLICA_COUNT     = config.deploy.replicas.toString()
            env.HEALTH_CHECK_PATH = config.healthcheck.path
        }
    }
}
```

```yaml
# ci/config.yml
deploy:
  namespace: payment-service
  replicas: 3
healthcheck:
  path: /actuator/health
  timeout: 30
```

---

## Tổ Chức Jenkinsfile Lớn

### Chia Nhỏ Bằng Script Ngoài

Với pipeline phức tạp, chuyển logic sang script riêng:

```
my-app/
├── ci/
│   ├── scripts/
│   │   ├── build.sh          ← Script build
│   │   ├── test.sh           ← Script test
│   │   ├── deploy.sh         ← Script deploy
│   │   └── notify.sh         ← Script thông báo
│   └── config.yml            ← Cấu hình CI/CD
└── Jenkinsfile               ← Chỉ chứa flow control
```

```groovy
// Jenkinsfile đơn giản — chỉ định nghĩa flow
pipeline {
    agent any

    stages {
        stage('Build') {
            steps { sh 'ci/scripts/build.sh' }
        }
        stage('Test') {
            steps { sh 'ci/scripts/test.sh' }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh "ci/scripts/deploy.sh ${params.ENV ?: 'staging'}" }
        }
    }

    post {
        always { sh "ci/scripts/notify.sh ${currentBuild.result}" }
    }
}
```

### Dùng Shared Libraries Cho Logic Tái Sử Dụng

```groovy
// Jenkinsfile
@Library('company-pipeline-library@v2.1') _

// Dùng hàm từ Shared Library
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                // mvnBuild là hàm từ Shared Library
                mvnBuild goals: 'clean package', skipTests: false
            }
        }

        stage('Docker') {
            steps {
                // buildAndPushImage là hàm từ Shared Library
                buildAndPushImage registry: env.COMPANY_REGISTRY
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                // helmDeploy là hàm từ Shared Library
                helmDeploy namespace: 'production', timeout: '10m'
            }
        }
    }
}
```

---

## Kiểm Tra Cú Pháp Jenkinsfile

### Dùng Jenkins Linter API

```bash
# Kiểm tra cú pháp Jenkinsfile qua API (cần Jenkins đang chạy)
curl --user admin:API_TOKEN \
     --data-urlencode "jenkinsfile=$(cat Jenkinsfile)" \
     https://jenkins.example.com/pipeline-model-converter/validate

# Kết quả: "Jenkinsfile successfully validated." nếu hợp lệ
```

### Dùng Jenkins CLI

```bash
# Tải Jenkins CLI
wget https://jenkins.example.com/jnlpJars/jenkins-cli.jar

# Kiểm tra cú pháp
java -jar jenkins-cli.jar \
     -s https://jenkins.example.com/ \
     -auth admin:API_TOKEN \
     declarative-linter < Jenkinsfile
```

### Plugin "Pipeline: Declarative Extension Points API"

Khi push Jenkinsfile lên GitHub, nếu dùng GitHub Checks integration, Jenkins tự động kiểm tra cú pháp và báo cáo kết quả trực tiếp trên Pull Request.

### Kiểm Tra Cú Pháp Offline (Không Cần Jenkins)

```bash
# Dùng Jenkins X Pipeline Validator (nếu dùng Jenkins X)
jx pipeline validate

# Hoặc dùng jenkinsfile-runner (Docker image)
docker run --rm \
    -v $(pwd)/Jenkinsfile:/workspace/Jenkinsfile \
    jenkins/jenkinsfile-runner \
    --no-sandbox
```

---

## Best Practices

### 1. Đặt Jenkinsfile Ở Gốc Repository

```
repo-root/
└── Jenkinsfile    ✅ — Dễ tìm, convention phổ biến
```

Ngoại lệ: Monorepo với nhiều service → mỗi service có Jenkinsfile riêng trong thư mục của mình:

```
monorepo/
├── services/
│   ├── auth/
│   │   └── Jenkinsfile    ✅
│   ├── payment/
│   │   └── Jenkinsfile    ✅
│   └── notification/
│       └── Jenkinsfile    ✅
└── Jenkinsfile            ✅ — Pipeline tổng cho toàn monorepo
```

### 2. Giữ Jenkinsfile Ngắn và Đọc Được

```groovy
// Tốt — ngắn, rõ ràng, dễ hiểu
pipeline {
    agent any
    stages {
        stage('Build')  { steps { sh 'make build' } }
        stage('Test')   { steps { sh 'make test' } }
        stage('Deploy') { when { branch 'main' }; steps { sh 'make deploy' } }
    }
}
```

Nếu pipeline dài hơn 150 dòng → xem xét tách logic sang:
- Script files (`ci/scripts/*.sh`)
- Shared Libraries
- File cấu hình YAML

### 3. Luôn Có `timeout` Ở Cấp Pipeline

Tránh pipeline bị treo vô hạn:

```groovy
options {
    timeout(time: 45, unit: 'MINUTES')    // Timeout tổng
}

stage('Integration Tests') {
    options {
        timeout(time: 20, unit: 'MINUTES')    // Timeout riêng cho stage chậm
    }
}
```

### 4. Dùng `buildDiscarder` Để Tiết Kiệm Ổ Đĩa

```groovy
options {
    buildDiscarder(logRotator(
        numToKeepStr:         '30',    // Giữ 30 build gần nhất
        artifactNumToKeepStr: '5'      // Giữ artifact của 5 build gần nhất
    ))
}
```

### 5. Đặt `disableConcurrentBuilds()` Khi Cần

```groovy
options {
    // Tránh deploy race condition khi nhiều commit push cùng lúc
    disableConcurrentBuilds(abortPrevious: true)    // Hủy build cũ khi có build mới
}
```

### 6. Dùng `checkout scm` Thay Vì `git url:`

```groovy
// ĐÚNG — Dùng credential và URL đã cấu hình trong job
checkout scm

// Tránh hardcode nếu có thể
// git url: 'https://github.com/org/repo.git', branch: 'main'
```

### 7. Luôn Publish Kết Quả Test Dù Build Thất Bại

```groovy
stage('Test') {
    steps { sh 'mvn test' }
    post {
        always {
            // Luôn publish, kể cả khi test fail
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
            // Lưu coverage report
            jacoco execPattern: 'target/jacoco.exec'
        }
    }
}
```

### 8. Validate Tham Số Quan Trọng Sớm

```groovy
stage('Validate') {
    steps {
        script {
            // Kiểm tra điều kiện cần thiết trước khi bắt đầu
            if (!params.VERSION.matches(/\d+\.\d+\.\d+/)) {
                error "VERSION phải có định dạng x.y.z — nhận được: '${params.VERSION}'"
            }
            if (params.DEPLOY_ENV == 'production' && env.BRANCH_NAME != 'main') {
                error "Chỉ được deploy production từ nhánh main!"
            }
        }
    }
}
```

### 9. Dùng `cleanWs()` Để Dọn Dẹp

```groovy
post {
    always {
        cleanWs()    // Xóa workspace sau build — tránh ổ đĩa đầy
    }
}
```

### 10. Đặt Build Description Rõ Ràng

```groovy
stage('Setup') {
    steps {
        script {
            def gitSHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            currentBuild.description = "${params.DEPLOY_ENV} | ${gitSHA}"
            currentBuild.displayName = "#${env.BUILD_NUMBER} — ${gitSHA}"
        }
    }
}
```

---

## Anti-Patterns — Những Điều Cần Tránh

### Anti-Pattern 1: Hardcode Thông Tin Môi Trường

```groovy
// SAI ❌ — Cứng nhắc, khó thay đổi
sh 'kubectl --server https://k8s.company.internal --token abc123...'

// ĐÚNG ✅
withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    sh 'kubectl apply -f k8s/'
}
```

### Anti-Pattern 2: Sleep Dài Để Chờ

```groovy
// SAI ❌ — Lãng phí executor, khó debug
sleep(300)    // Ngủ 5 phút chờ deployment
sh 'curl http://app/health'

// ĐÚNG ✅ — Polling có timeout
timeout(time: 5, unit: 'MINUTES') {
    waitUntil {
        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" http://app/health', returnStdout: true).trim()
        return status == '200'
    }
}
```

### Anti-Pattern 3: Bắt Exception Rồi Nuốt

```groovy
// SAI ❌ — Pipeline báo SUCCESS dù thực sự fail
try {
    sh './deploy.sh'
} catch (e) {
    echo "Lỗi: ${e.getMessage()}"
    // Không throw lại → Jenkins không biết pipeline fail
}

// ĐÚNG ✅
try {
    sh './deploy.sh'
} catch (e) {
    echo "Lỗi: ${e.getMessage()}"
    currentBuild.result = 'FAILURE'
    throw e    // Luôn throw lại để Jenkins biết
}
```

### Anti-Pattern 4: Logic Deploy Phức Tạp Trực Tiếp Trong Jenkinsfile

```groovy
// SAI ❌ — Deploy logic nằm trong Jenkinsfile, khó test và maintain
stage('Deploy') {
    steps {
        sh '''
            # 200 dòng bash script deploy phức tạp
            kubectl apply -f k8s/
            kubectl rollout status deployment/myapp
            for i in $(seq 1 10); do
                # ...
            done
        '''
    }
}

// ĐÚNG ✅ — Chuyển logic sang script riêng, Jenkinsfile chỉ gọi
stage('Deploy') {
    steps {
        sh 'ci/scripts/deploy.sh'
    }
}
```

### Anti-Pattern 5: Dùng `sh 'sudo ...'` Trong Pipeline

```groovy
// SAI ❌ — Tiềm ẩn bảo mật, agent nên không cần sudo
sh 'sudo apt-get install -y curl'

// ĐÚNG ✅ — Dùng Docker agent với image đã có sẵn công cụ cần
agent {
    docker { image 'ubuntu:22.04' }
}
steps {
    sh 'apt-get install -y curl'    // Chạy trong container — không cần sudo
}
```

### Anti-Pattern 6: Pipeline Không Có Timeout

```groovy
// SAI ❌ — Nếu lệnh bị treo → pipeline chạy mãi mãi
steps {
    sh './run-long-test.sh'
}

// ĐÚNG ✅
options {
    timeout(time: 30, unit: 'MINUTES')
}
```

---

## Ví Dụ Theo Loại Ứng Dụng

### Java / Maven Application

```groovy
pipeline {
    agent {
        docker { image 'maven:3.9-eclipse-temurin-21' }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 45, unit: 'MINUTES')
    }

    stages {
        stage('Build')  { steps { sh 'mvn clean package -DskipTests' } }
        stage('Test')   {
            steps { sh 'mvn test' }
            post  { always { junit 'target/surefire-reports/*.xml' } }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'helm upgrade --install myapp ./helm/myapp --wait'
                }
            }
        }
    }
}
```

### Node.js Application

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args  '-v $HOME/.npm:/root/.npm'    // Cache npm packages
        }
    }

    stages {
        stage('Install') { steps { sh 'npm ci' } }
        stage('Lint')    { steps { sh 'npm run lint' } }
        stage('Test')    {
            steps { sh 'npm test -- --coverage' }
            post  { always { publishHTML(target: [reportDir: 'coverage', reportFiles: 'index.html', reportName: 'Coverage']) } }
        }
        stage('Build')   { steps { sh 'npm run build' } }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'npm run deploy:prod' }
        }
    }
}
```

### Go Application

```groovy
pipeline {
    agent {
        docker {
            image 'golang:1.22-alpine'
            args  '-v $HOME/go:/go'    // Cache Go modules
        }
    }

    environment {
        GOPATH  = '/go'
        GOPROXY = 'https://proxy.golang.org,direct'
        CGO_ENABLED = '0'
    }

    stages {
        stage('Dependencies') { steps { sh 'go mod download' } }
        stage('Lint')         { steps { sh 'go vet ./...' } }
        stage('Test')         {
            steps { sh 'go test -v -race -coverprofile=coverage.out ./...' }
            post  { always { sh 'go tool cover -html=coverage.out -o coverage.html' } }
        }
        stage('Build')        { steps { sh 'go build -ldflags="-s -w" -o bin/app .' } }
        stage('Docker Image') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh '''
                        docker login -u $U -p $P registry.example.com
                        docker build -t registry.example.com/myapp:${BUILD_NUMBER} .
                        docker push registry.example.com/myapp:${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
}
```

### Python Application

```groovy
pipeline {
    agent {
        docker { image 'python:3.12-slim' }
    }

    environment {
        PIP_CACHE_DIR = '/tmp/pip-cache'
    }

    stages {
        stage('Install')   { steps { sh 'pip install -r requirements.txt -r requirements-dev.txt' } }
        stage('Lint')      { steps { sh 'flake8 src/ && black --check src/' } }
        stage('Type Check') { steps { sh 'mypy src/' } }
        stage('Test')      {
            steps { sh 'pytest tests/ -v --cov=src --cov-report=xml' }
            post  { always { junit 'pytest-results.xml' } }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'python deploy.py' }
        }
    }
}
```

---

**Tiếp Theo:** Đọc [4-pipeline-syntax.md](4-pipeline-syntax.md) để có tài liệu tham chiếu đầy đủ về cú pháp và directive của Jenkins Pipeline.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
