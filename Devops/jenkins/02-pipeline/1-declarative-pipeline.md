# Declarative Pipeline — Pipeline Khai Báo

> Cú pháp Declarative Pipeline đầy đủ: agent, stages, steps, post, when, environment, options, parameters, tools. Đây là loại pipeline được khuyến nghị cho hầu hết dự án Jenkins.

## Mục Lục

1. [Cấu Trúc Tổng Thể](#cấu-trúc-tổng-thể)
2. [agent — Chỉ Định Nơi Chạy](#agent--chỉ-định-nơi-chạy)
3. [stages và stage — Cấu Trúc Giai Đoạn](#stages-và-stage--cấu-trúc-giai-đoạn)
4. [steps — Các Bước Thực Thi](#steps--các-bước-thực-thi)
5. [post — Xử Lý Sau Build](#post--xử-lý-sau-build)
6. [when — Điều Kiện Chạy Stage](#when--điều-kiện-chạy-stage)
7. [environment — Biến Môi Trường](#environment--biến-môi-trường)
8. [options — Tùy Chọn Pipeline](#options--tùy-chọn-pipeline)
9. [parameters — Tham Số Đầu Vào](#parameters--tham-số-đầu-vào)
10. [tools — Công Cụ Build](#tools--công-cụ-build)
11. [parallel — Chạy Song Song](#parallel--chạy-song-song)
12. [input — Chờ Xác Nhận Thủ Công](#input--chờ-xác-nhận-thủ-công)
13. [Pipeline Hoàn Chỉnh Ví Dụ Thực Tế](#pipeline-hoàn-chỉnh-ví-dụ-thực-tế)
14. [Các Lỗi Phổ Biến](#các-lỗi-phổ-biến)

---

## Cấu Trúc Tổng Thể

Declarative Pipeline luôn bắt đầu bằng khối `pipeline { }` ở cấp cao nhất:

```groovy
pipeline {
    // 1. agent     — chạy trên agent nào
    agent any

    // 2. options   — tùy chọn toàn pipeline
    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    // 3. environment — biến môi trường
    environment {
        APP_NAME = 'my-app'
        VERSION  = '1.0.0'
    }

    // 4. parameters — tham số đầu vào từ người dùng
    parameters {
        string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: 'Môi trường deploy')
    }

    // 5. tools — công cụ build được quản lý bởi Jenkins
    tools {
        maven 'Maven 3.8'
        jdk   'JDK 17'
    }

    // 6. stages — tập hợp các stage (giai đoạn)
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }

    // 7. post — xử lý sau khi pipeline hoàn thành
    post {
        always  { echo 'Pipeline kết thúc' }
        success { echo 'Thành công!' }
        failure { echo 'Thất bại!' }
    }
}
```

**Quy tắc quan trọng:** Các directive như `agent`, `stages`, `post` phải đặt bên trong `pipeline { }`. Không thể đặt logic Groovy tùy ý ở cấp `pipeline` — đây là sự khác biệt lớn so với Scripted Pipeline.

---

## agent — Chỉ Định Nơi Chạy

`agent` chỉ định agent nào (node nào) sẽ thực thi pipeline hoặc stage.

### Các Loại Agent

#### `agent any` — Chạy Trên Bất Kỳ Agent Nào

```groovy
pipeline {
    agent any   // Jenkins chọn agent rảnh bất kỳ
    stages { ... }
}
```

#### `agent none` — Không Có Agent Mặc Định

Dùng khi muốn chỉ định agent riêng cho từng stage:

```groovy
pipeline {
    agent none   // Không có agent mặc định ở cấp pipeline

    stages {
        stage('Build') {
            agent { label 'linux' }   // Stage này dùng agent có nhãn 'linux'
            steps { sh 'mvn package' }
        }
        stage('Deploy') {
            agent { label 'deploy-server' }   // Stage này dùng agent khác
            steps { sh './deploy.sh' }
        }
    }
}
```

**Lưu ý:** Khi dùng `agent none`, mỗi stage bắt buộc phải có `agent` riêng — trừ stage chỉ chứa `parallel`.

#### `agent { label '...' }` — Chọn Agent Theo Nhãn

```groovy
agent {
    label 'linux && docker'    // Agent phải có cả hai nhãn: linux VÀ docker
}

agent {
    label 'linux || mac'       // Agent có nhãn linux HOẶC mac
}
```

#### `agent { docker '...' }` — Chạy Trong Docker Container

```groovy
agent {
    docker {
        image 'maven:3.8-jdk-17'    // Image Docker để build
        args  '-v /tmp:/tmp'        // Tham số truyền vào docker run
        reuseNode true              // Tái dùng workspace của node hiện tại
    }
}
```

#### `agent { dockerfile true }` — Build Từ Dockerfile Trong Repo

```groovy
agent {
    dockerfile {
        filename 'Dockerfile.build'    // Tên Dockerfile (mặc định: 'Dockerfile')
        dir      'docker/'             // Thư mục chứa Dockerfile
        additionalBuildArgs '--build-arg VERSION=1.0'
    }
}
```

#### `agent { kubernetes { ... } }` — Pod Trên Kubernetes

```groovy
agent {
    kubernetes {
        yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8-jdk-17
    command: ['sleep', '9999']
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
"""
        defaultContainer 'maven'
    }
}
```

---

## stages và stage — Cấu Trúc Giai Đoạn

`stages` là directive bắt buộc chứa một hoặc nhiều `stage`. Mỗi `stage` đại diện cho một giai đoạn trong quá trình CI/CD.

```groovy
stages {
    stage('Checkout') {         // Giai đoạn 1: Lấy source code
        steps {
            checkout scm
        }
    }

    stage('Build') {            // Giai đoạn 2: Build ứng dụng
        steps {
            sh 'mvn clean package -DskipTests'
        }
    }

    stage('Test') {             // Giai đoạn 3: Chạy test
        steps {
            sh 'mvn test'
        }
        post {
            always {
                // Publish JUnit test results dù pass hay fail
                junit 'target/surefire-reports/*.xml'
            }
        }
    }

    stage('Deploy') {           // Giai đoạn 4: Deploy
        steps {
            sh './deploy.sh'
        }
    }
}
```

### stage lồng nhau (Nested Stages)

Từ Jenkins 2.235+, có thể lồng `stage` bên trong `stage` để nhóm các bước liên quan:

```groovy
stage('Integration Tests') {
    stages {
        stage('Setup Database') {
            steps { sh './setup-db.sh' }
        }
        stage('Run Tests') {
            steps { sh 'mvn verify -Pintegration' }
        }
        stage('Teardown') {
            steps { sh './teardown-db.sh' }
        }
    }
}
```

---

## steps — Các Bước Thực Thi

`steps` chứa danh sách các lệnh thực thi bên trong một stage.

### Bước Shell

```groovy
steps {
    // sh: chạy lệnh shell trên Linux/macOS
    sh 'echo "Hello World"'
    sh 'mvn clean package'

    // Multiline script
    sh '''
        echo "Bước 1: Clean"
        mvn clean

        echo "Bước 2: Build"
        mvn package
    '''

    // Lấy output của lệnh shell
    script {
        def version = sh(script: 'cat VERSION', returnStdout: true).trim()
        echo "Version: ${version}"
    }

    // bat: chạy lệnh batch trên Windows
    bat 'mvn clean package'
}
```

### Bước File và Workspace

```groovy
steps {
    // Checkout source code từ SCM (Source Control Management)
    checkout scm

    // Đọc file
    script {
        def content = readFile('config.yml')
        echo content
    }

    // Ghi file
    writeFile file: 'output.txt', text: 'Build completed'

    // Copy artifact từ job khác
    copyArtifacts projectName: 'my-other-job', filter: '*.jar'

    // Archive artifact (lưu kết quả build)
    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

    // Publish HTML report
    publishHTML(target: [
        reportDir:   'target/site',
        reportFiles: 'index.html',
        reportName:  'Maven Site'
    ])
}
```

### Bước Credentials (Thông Tin Xác Thực)

```groovy
steps {
    // Dùng username/password
    withCredentials([usernamePassword(
        credentialsId: 'docker-registry-creds',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    )]) {
        sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS registry.example.com'
    }

    // Dùng secret text
    withCredentials([string(credentialsId: 'api-token', variable: 'API_TOKEN')]) {
        sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com/deploy'
    }

    // Dùng SSH private key
    withCredentials([sshUserPrivateKey(
        credentialsId: 'deploy-ssh-key',
        keyFileVariable: 'SSH_KEY'
    )]) {
        sh 'ssh -i $SSH_KEY user@server.example.com "sudo systemctl restart myapp"'
    }
}
```

### Bước Stash / Unstash

```groovy
stage('Build') {
    steps {
        sh 'mvn package'
        // Lưu file tạm để dùng ở stage khác (đặc biệt khi stage khác chạy trên agent khác)
        stash name: 'built-artifacts', includes: 'target/*.jar'
    }
}

stage('Deploy') {
    agent { label 'deploy-server' }
    steps {
        // Khôi phục file đã stash
        unstash 'built-artifacts'
        sh 'java -jar target/myapp.jar'
    }
}
```

---

## post — Xử Lý Sau Build

`post` định nghĩa các hành động thực hiện sau khi pipeline hoặc stage kết thúc, tùy theo kết quả.

```groovy
post {
    // Luôn chạy, bất kể kết quả nào
    always {
        echo 'Pipeline kết thúc — dọn dẹp workspace'
        cleanWs()    // Plugin Workspace Cleanup
    }

    // Chỉ chạy khi thành công
    success {
        echo 'Build thành công!'
        slackSend color: 'good', message: "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} thành công"
    }

    // Chỉ chạy khi thất bại
    failure {
        echo 'Build thất bại!'
        slackSend color: 'danger', message: "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} thất bại"
        // Gửi email thông báo
        mail to: 'team@example.com',
             subject: "Build Failed: ${env.JOB_NAME}",
             body: "Build #${env.BUILD_NUMBER} thất bại. Kiểm tra: ${env.BUILD_URL}"
    }

    // Chạy khi kết quả thay đổi so với build trước
    // (ví dụ: lần trước fail, lần này success)
    changed {
        echo 'Kết quả build thay đổi so với lần trước!'
    }

    // Chạy khi build bị hủy (abort)
    aborted {
        echo 'Build bị hủy!'
    }

    // Chạy khi kết quả là UNSTABLE
    // (thường do test fail nhưng build không crash)
    unstable {
        echo 'Build không ổn định — có test thất bại!'
    }

    // Chỉ chạy khi thành công HOẶC unstable (không bị crash)
    unsuccessful {
        echo 'Build không thành công hoàn toàn!'
    }

    // Chạy khi cleanup — sau tất cả các điều kiện khác
    cleanup {
        echo 'Dọn dẹp cuối cùng'
    }
}
```

**Thứ tự thực thi:** `always` → điều kiện phù hợp (`success`/`failure`/...) → `cleanup`

**`post` ở cấp stage:** Có thể đặt `post` bên trong từng `stage` để xử lý riêng cho stage đó:

```groovy
stage('Test') {
    steps {
        sh 'mvn test'
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'   // Luôn publish test results
        }
        failure {
            archiveArtifacts 'target/failsafe-reports/**'   // Lưu report khi fail
        }
    }
}
```

---

## when — Điều Kiện Chạy Stage

`when` cho phép stage chỉ chạy khi thỏa mãn điều kiện nhất định.

### Các Điều Kiện Phổ Biến

```groovy
// Chỉ chạy trên nhánh 'main'
when {
    branch 'main'
}

// Chỉ chạy khi biến môi trường thỏa mãn
when {
    environment name: 'DEPLOY_ENV', value: 'production'
}

// Biểu thức tùy ý (Groovy)
when {
    expression {
        return params.DEPLOY_ENV == 'production' && env.BRANCH_NAME == 'main'
    }
}

// Chạy khi file thay đổi (cần changeset plugin)
when {
    changeset 'src/**'
}

// Kết hợp nhiều điều kiện với allOf (TẤT CẢ phải đúng)
when {
    allOf {
        branch 'main'
        environment name: 'DEPLOY_READY', value: 'true'
    }
}

// Kết hợp nhiều điều kiện với anyOf (ÍT NHẤT MỘT phải đúng)
when {
    anyOf {
        branch 'main'
        branch 'release/*'
    }
}

// Điều kiện phủ định
when {
    not {
        branch 'develop'
    }
}
```

### Ví Dụ Thực Tế: Deploy Chỉ Khi Ở Main Branch

```groovy
stages {
    stage('Build') {
        steps { sh 'mvn package' }
    }

    stage('Test') {
        steps { sh 'mvn test' }
    }

    stage('Deploy to Staging') {
        when {
            anyOf {
                branch 'main'
                branch 'release/*'
            }
        }
        steps {
            sh './deploy.sh staging'
        }
    }

    stage('Deploy to Production') {
        when {
            allOf {
                branch 'main'
                // Tham số CONFIRM_PROD phải được set thành 'yes'
                expression { return params.CONFIRM_PROD == 'yes' }
            }
        }
        steps {
            sh './deploy.sh production'
        }
    }
}
```

### `beforeAgent` — Tối Ưu Hiệu Suất

Mặc định, Jenkins phân bổ agent trước khi đánh giá điều kiện `when`. Dùng `beforeAgent true` để đánh giá `when` trước — tiết kiệm tài nguyên:

```groovy
stage('Deploy') {
    agent { label 'deploy-server' }
    when {
        beforeAgent true          // Kiểm tra điều kiện TRƯỚC khi allocate agent
        branch 'main'
    }
    steps { sh './deploy.sh' }
}
```

---

## environment — Biến Môi Trường

`environment` khai báo biến môi trường. Có thể đặt ở cấp `pipeline` (toàn cục) hoặc trong `stage` (cục bộ).

```groovy
pipeline {
    environment {
        // Biến thường
        APP_NAME    = 'my-application'
        APP_VERSION = '2.1.0'
        REGISTRY    = 'registry.example.com'

        // Lấy giá trị từ Credentials
        DOCKER_CREDS = credentials('docker-registry-creds')
        // → Tự động tạo: DOCKER_CREDS_USR và DOCKER_CREDS_PSW

        // Lấy từ shell command
        GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Build Docker Image') {
            environment {
                // Biến chỉ tồn tại trong stage này
                IMAGE_TAG = "${REGISTRY}/${APP_NAME}:${APP_VERSION}"
            }
            steps {
                sh "docker build -t ${IMAGE_TAG} ."
                sh "docker push ${IMAGE_TAG}"
            }
        }
    }
}
```

### Biến Môi Trường Tự Động của Jenkins

| Biến                  | Giá Trị                                           |
| --------------------- | ------------------------------------------------- |
| `env.BUILD_NUMBER`    | Số thứ tự build (1, 2, 3...)                      |
| `env.BUILD_URL`       | URL đầy đủ đến trang build                        |
| `env.JOB_NAME`        | Tên job                                           |
| `env.BRANCH_NAME`     | Tên nhánh (chỉ có trong Multibranch Pipeline)     |
| `env.GIT_COMMIT`      | SHA commit đầy đủ                                 |
| `env.GIT_BRANCH`      | Nhánh Git                                         |
| `env.WORKSPACE`       | Đường dẫn thư mục workspace trên agent            |
| `env.NODE_NAME`       | Tên agent đang chạy                               |
| `env.JENKINS_URL`     | URL gốc của Jenkins                               |

---

## options — Tùy Chọn Pipeline

`options` cấu hình hành vi của toàn bộ pipeline.

```groovy
options {
    // Timeout toàn bộ pipeline
    timeout(time: 1, unit: 'HOURS')

    // Retry tự động khi thất bại (tổng cộng chạy 3 lần)
    retry(3)

    // Chỉ giữ lại 10 build gần nhất, 5 artifact gần nhất
    buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))

    // Không chạy nhiều build của cùng job song song
    disableConcurrentBuilds()

    // Bỏ qua checkout mặc định (tự quản lý checkout)
    skipDefaultCheckout()

    // Timestamp trong log
    timestamps()

    // Màu ANSI trong log (cần AnsiColor plugin)
    ansiColor('xterm')

    // Không cho phép chạy lại (replay)
    disableRestart()

    // Chờ tối đa 5 phút trước khi hết hạn lock
    lock(resource: 'deploy-lock', variable: 'LOCKED_RESOURCE')
}
```

### `options` Ở Cấp Stage

Một số option có thể đặt trong stage:

```groovy
stage('Slow Test') {
    options {
        timeout(time: 30, unit: 'MINUTES')   // Timeout riêng cho stage này
        retry(2)                              // Retry riêng cho stage này
    }
    steps {
        sh 'mvn verify -Pintegration-tests'
    }
}
```

---

## parameters — Tham Số Đầu Vào

`parameters` cho phép người dùng nhập tham số trước khi chạy pipeline (qua nút "Build with Parameters" trên UI).

```groovy
parameters {
    // Chuỗi văn bản
    string(
        name:         'DEPLOY_ENV',
        defaultValue: 'staging',
        description:  'Môi trường deploy: staging hoặc production'
    )

    // Lựa chọn từ danh sách
    choice(
        name:    'REGION',
        choices: ['us-east-1', 'eu-west-1', 'ap-southeast-1'],
        description: 'AWS Region để deploy'
    )

    // Boolean (true/false)
    booleanParam(
        name:         'RUN_INTEGRATION_TESTS',
        defaultValue: true,
        description:  'Chạy Integration Tests không?'
    )

    // Upload file
    file(
        name:        'CONFIG_FILE',
        description: 'File cấu hình tùy chỉnh'
    )

    // Chuỗi nhiều dòng
    text(
        name:        'RELEASE_NOTES',
        defaultValue: '',
        description: 'Ghi chú phiên bản release'
    )

    // Password (che khuất giá trị)
    password(
        name:        'DB_PASSWORD',
        defaultValue: '',
        description: 'Mật khẩu database'
    )
}
```

**Sử dụng tham số trong pipeline:**

```groovy
stages {
    stage('Deploy') {
        when {
            expression { return params.DEPLOY_ENV == 'production' }
        }
        steps {
            echo "Deploy lên: ${params.DEPLOY_ENV}"
            echo "Region: ${params.REGION}"

            script {
                if (params.RUN_INTEGRATION_TESTS) {
                    sh 'mvn verify -Pintegration'
                }
            }
        }
    }
}
```

---

## tools — Công Cụ Build

`tools` cấu hình công cụ build được quản lý bởi Jenkins Tool Configuration (Global Tool Configuration).

```groovy
tools {
    // Tên phải khớp với tên đã cấu hình trong Jenkins → Manage Jenkins → Global Tool Configuration
    maven 'Maven 3.8.6'
    jdk   'OpenJDK 17'
    gradle 'Gradle 7.6'
    nodejs 'Node 18 LTS'
    go    'Go 1.21'
}
```

Sau khi khai báo `tools`, Jenkins tự động thêm công cụ vào `PATH` — không cần chỉ đường dẫn tuyệt đối:

```groovy
steps {
    sh 'mvn --version'    // Dùng Maven đã khai báo trong tools
    sh 'java -version'    // Dùng JDK đã khai báo trong tools
}
```

---

## parallel — Chạy Song Song

Giúp tăng tốc pipeline bằng cách chạy nhiều stage cùng lúc.

### Parallel Cơ Bản

```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'mvn test -Dtest=UnitTest*' }
            post { always { junit 'target/surefire-reports/unit/*.xml' } }
        }
        stage('Integration Tests') {
            steps { sh 'mvn verify -Pintegration' }
            post { always { junit 'target/surefire-reports/integration/*.xml' } }
        }
        stage('Security Scan') {
            steps { sh 'mvn dependency-check:check' }
        }
    }
}
```

### Parallel Với Agent Khác Nhau

```groovy
stage('Build Multi-Platform') {
    parallel {
        stage('Build Linux') {
            agent { label 'linux' }
            steps {
                sh 'make build-linux'
                stash name: 'linux-binary', includes: 'dist/linux/*'
            }
        }
        stage('Build Windows') {
            agent { label 'windows' }
            steps {
                bat 'make build-windows'
                stash name: 'windows-binary', includes: 'dist/windows/*'
            }
        }
        stage('Build macOS') {
            agent { label 'macos' }
            steps {
                sh 'make build-macos'
                stash name: 'macos-binary', includes: 'dist/macos/*'
            }
        }
    }
}
```

### `failFast` — Dừng Khi Một Nhánh Thất Bại

```groovy
stage('Parallel Tests') {
    failFast true    // Nếu một stage fail → dừng tất cả stage còn lại

    parallel {
        stage('Unit Tests')       { steps { sh 'mvn test' } }
        stage('Integration Tests') { steps { sh 'mvn verify' } }
        stage('E2E Tests')        { steps { sh 'npm run e2e' } }
    }
}
```

---

## input — Chờ Xác Nhận Thủ Công

`input` tạm dừng pipeline và chờ người dùng xác nhận trước khi tiếp tục — thường dùng trước bước deploy production.

```groovy
stage('Deploy to Production') {
    input {
        message 'Xác nhận deploy lên Production?'
        ok      'Đồng ý Deploy'
        submitter 'alice,bob,ops-team'    // Chỉ các user này được xác nhận
        parameters {
            string(name: 'REASON', defaultValue: '', description: 'Lý do deploy')
        }
    }
    steps {
        echo "Deploy được phê duyệt. Lý do: ${REASON}"
        sh './deploy-production.sh'
    }
}
```

**Lưu ý:** Pipeline đang chờ `input` vẫn chiếm executor slot. Để giải phóng executor khi chờ, dùng `milestone` hoặc thiết kế pipeline không dùng executor khi chờ.

---

## Pipeline Hoàn Chỉnh Ví Dụ Thực Tế

Pipeline Java/Maven hoàn chỉnh từ build đến deploy:

```groovy
pipeline {
    agent none    // Mỗi stage tự chọn agent

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
        timestamps()
    }

    environment {
        APP_NAME    = 'payment-service'
        REGISTRY    = 'registry.company.com'
        IMAGE_TAG   = "${REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}"
        SONAR_TOKEN = credentials('sonarqube-token')
    }

    parameters {
        choice(
            name:    'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: 'Môi trường deploy'
        )
        booleanParam(
            name:         'SKIP_TESTS',
            defaultValue: false,
            description:  'Bỏ qua test (chỉ dùng khi khẩn cấp)'
        )
    }

    stages {
        stage('Checkout') {
            agent any
            steps {
                checkout scm
                script {
                    env.GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "${REGISTRY}/${APP_NAME}:${env.GIT_SHORT}"
                }
                echo "Building commit: ${env.GIT_SHORT}"
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'maven:3.8-eclipse-temurin-17'
                    args  '-v $HOME/.m2:/root/.m2'    // Cache Maven dependencies
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
                stash name: 'app-jar', includes: 'target/*.jar'
            }
        }

        stage('Test & Quality') {
            when {
                not { expression { return params.SKIP_TESTS } }
            }
            parallel {
                stage('Unit Tests') {
                    agent {
                        docker { image 'maven:3.8-eclipse-temurin-17' }
                    }
                    steps {
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('SonarQube Analysis') {
                    agent { label 'sonar-agent' }
                    steps {
                        withSonarQubeEnv('SonarQube Server') {
                            sh 'mvn sonar:sonar'
                        }
                        // Chờ Quality Gate (cổng chất lượng)
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            agent { label 'docker-builder' }
            steps {
                unstash 'app-jar'
                withCredentials([usernamePassword(
                    credentialsId: 'registry-creds',
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh """
                        docker login -u ${REG_USER} -p ${REG_PASS} ${REGISTRY}
                        docker build -t ${IMAGE_TAG} .
                        docker push ${IMAGE_TAG}
                        docker rmi ${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            agent { label 'deploy-agent' }
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            environment {
                KUBECONFIG = credentials('staging-kubeconfig')
            }
            steps {
                sh """
                    helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                        --namespace staging \
                        --set image.tag=${env.GIT_SHORT} \
                        --wait --timeout 5m
                """
            }
        }

        stage('Approval — Production Deploy') {
            when {
                allOf {
                    branch 'main'
                    expression { return params.DEPLOY_ENV == 'production' }
                }
            }
            steps {
                input message: "Deploy ${APP_NAME}:${env.GIT_SHORT} lên Production?",
                      ok:      'Phê Duyệt',
                      submitter: 'ops-leads'
            }
        }

        stage('Deploy to Production') {
            agent { label 'deploy-agent' }
            when {
                allOf {
                    branch 'main'
                    expression { return params.DEPLOY_ENV == 'production' }
                }
            }
            environment {
                KUBECONFIG = credentials('production-kubeconfig')
            }
            steps {
                sh """
                    helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                        --namespace production \
                        --set image.tag=${env.GIT_SHORT} \
                        --wait --timeout 10m
                """
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color:   'good',
                message: "✅ *${APP_NAME}* deploy thành công\n" +
                         "Commit: `${env.GIT_SHORT}` | Build: <${env.BUILD_URL}|#${env.BUILD_NUMBER}>"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color:   'danger',
                message: "❌ *${APP_NAME}* build/deploy thất bại\n" +
                         "Build: <${env.BUILD_URL}|#${env.BUILD_NUMBER}>"
            )
            mail(
                to:      'devops@company.com',
                subject: "Build Failed: ${APP_NAME} #${env.BUILD_NUMBER}",
                body:    "Xem chi tiết tại: ${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()    // Dọn dẹp workspace
        }
    }
}
```

---

## Các Lỗi Phổ Biến

### Lỗi 1: Dùng Groovy Tự Do Ngoài `script { }` Block

```groovy
// SAI — Declarative Pipeline không cho phép Groovy logic ngoài script block
pipeline {
    stages {
        stage('Build') {
            steps {
                def version = '1.0'    // ❌ Lỗi cú pháp
                sh "mvn package -Dversion=${version}"
            }
        }
    }
}

// ĐÚNG — Bọc trong script { }
pipeline {
    stages {
        stage('Build') {
            steps {
                script {
                    def version = '1.0'    // ✅
                    sh "mvn package -Dversion=${version}"
                }
            }
        }
    }
}
```

### Lỗi 2: Dùng `sh` Mà Không Có `script { }` Khi Cần Gán Biến

```groovy
// SAI
steps {
    def output = sh 'git rev-parse HEAD'    // ❌
}

// ĐÚNG
steps {
    script {
        def output = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()    // ✅
        echo "Commit: ${output}"
    }
}
```

### Lỗi 3: Quên `agent` Khi Dùng `agent none` Ở Cấp Pipeline

```groovy
// SAI — stage không có agent khi pipeline dùng agent none
pipeline {
    agent none
    stages {
        stage('Build') {
            steps { sh 'mvn package' }    // ❌ Không có agent → lỗi
        }
    }
}

// ĐÚNG
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }       // ✅ Mỗi stage phải có agent riêng
            steps { sh 'mvn package' }
        }
    }
}
```

### Lỗi 4: Dùng `environment` Với Shell Command Bên Ngoài `script { }`

```groovy
// SAI — sh() trong environment phải dùng cú pháp đúng
environment {
    VERSION = sh 'cat VERSION'    // ❌
}

// ĐÚNG
environment {
    VERSION = sh(script: 'cat VERSION', returnStdout: true).trim()    // ✅
}
```

---

**Tiếp Theo:** Đọc [2-scripted-pipeline.md](2-scripted-pipeline.md) để hiểu Groovy DSL và Scripted Pipeline.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
