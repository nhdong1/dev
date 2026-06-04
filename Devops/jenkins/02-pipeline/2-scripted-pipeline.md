# Scripted Pipeline — Pipeline Kịch Bản Groovy

> Scripted Pipeline dùng Groovy DSL (Domain-Specific Language — Ngôn Ngữ Đặc Thù Miền) thuần túy, cho phép viết logic phức tạp không thể thực hiện bằng Declarative Pipeline. Hiểu Scripted Pipeline giúp bạn đọc và bảo trì Jenkinsfile legacy và viết Shared Libraries.

## Mục Lục

1. [Cấu Trúc Cơ Bản](#cấu-trúc-cơ-bản)
2. [node — Chọn Agent Thực Thi](#node--chọn-agent-thực-thi)
3. [stage — Định Nghĩa Giai Đoạn](#stage--định-nghĩa-giai-đoạn)
4. [Xử Lý Lỗi Với try/catch/finally](#xử-lý-lỗi-với-trycatchfinally)
5. [Biến và Kiểu Dữ Liệu Groovy](#biến-và-kiểu-dữ-liệu-groovy)
6. [Điều Kiện và Vòng Lặp](#điều-kiện-và-vòng-lặp)
7. [Closure và Hàm Trong Groovy](#closure-và-hàm-trong-groovy)
8. [Parallel — Chạy Song Song](#parallel--chạy-song-song)
9. [currentBuild — Đối Tượng Build Hiện Tại](#currentbuild--đối-tượng-build-hiện-tại)
10. [Shared Steps — Tái Sử Dụng Code](#shared-steps--tái-sử-dụng-code)
11. [Scripted vs Declarative — Khi Nào Dùng Cái Nào](#scripted-vs-declarative--khi-nào-dùng-cái-nào)
12. [Ví Dụ Thực Tế Hoàn Chỉnh](#ví-dụ-thực-tế-hoàn-chỉnh)

---

## Cấu Trúc Cơ Bản

Scripted Pipeline bọc toàn bộ logic bên trong `node { }` block:

```groovy
node {
    // Tất cả code Groovy chạy tại đây trên agent được chọn

    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh 'mvn clean package'
    }

    stage('Test') {
        sh 'mvn test'
    }
}
```

Không có từ khóa `pipeline`, `stages`, `steps`, `post` như Declarative Pipeline — đây là Groovy script thuần túy với các hàm đặc biệt của Jenkins (`node`, `stage`, `sh`, `checkout`...).

### So Sánh Cấu Trúc

```
Declarative Pipeline          Scripted Pipeline
─────────────────────         ─────────────────────
pipeline {                    node {
  agent any                     // agent được chọn bởi node()
  stages {
    stage('Build') {            stage('Build') {
      steps {                     // không có steps wrapper
        sh 'mvn package'          sh 'mvn package'
      }                         }
    }
  }
  post {                        // Phải tự quản lý với try/finally
    always { ... }
  }
}                             }
```

---

## node — Chọn Agent Thực Thi

`node` phân bổ một executor (bộ thực thi) trên agent và chạy code bên trong block đó.

```groovy
// Chạy trên bất kỳ agent nào
node {
    sh 'mvn package'
}

// Chạy trên agent có nhãn 'linux'
node('linux') {
    sh 'mvn package'
}

// Chạy trên agent có nhãn 'linux && docker'
node('linux && docker') {
    sh 'docker build .'
}

// Lồng node — stage khác nhau chạy trên agent khác nhau
node('build-server') {
    stage('Build') {
        sh 'mvn package'
        stash name: 'jar', includes: 'target/*.jar'
    }
}

node('deploy-server') {
    stage('Deploy') {
        unstash 'jar'
        sh './deploy.sh'
    }
}
```

**Lưu ý:** Mỗi `node { }` block chiếm một executor cho đến khi block kết thúc. Nếu lồng `node` bên trong `node`, pipeline chiếm hai executor cùng lúc.

---

## stage — Định Nghĩa Giai Đoạn

`stage` trong Scripted Pipeline chỉ là một hàm nhận tên và block:

```groovy
node {
    stage('Checkout') {
        // Code của stage này
        checkout scm
    }

    stage('Build') {
        sh 'mvn clean package -DskipTests'
    }

    stage('Test') {
        sh 'mvn test'
        junit 'target/surefire-reports/*.xml'
    }

    // Stage có thể bị bỏ qua với điều kiện
    if (env.BRANCH_NAME == 'main') {
        stage('Deploy') {
            sh './deploy.sh'
        }
    }
}
```

**Khác biệt quan trọng:** Trong Scripted Pipeline, `stage` chỉ ảnh hưởng đến hiển thị trên Jenkins UI — không có cơ chế validation như Declarative Pipeline.

---

## Xử Lý Lỗi Với try/catch/finally

Thay vì dùng `post` như Declarative, Scripted Pipeline dùng `try/catch/finally` của Groovy:

### Cấu Trúc Cơ Bản

```groovy
node {
    try {
        stage('Build') {
            sh 'mvn package'
        }

        stage('Test') {
            sh 'mvn test'
        }

        stage('Deploy') {
            sh './deploy.sh'
        }

        // Chỉ chạy khi toàn bộ pipeline thành công
        currentBuild.result = 'SUCCESS'

    } catch (e) {
        // Chạy khi có exception (lỗi)
        currentBuild.result = 'FAILURE'
        echo "Pipeline thất bại: ${e.getMessage()}"

        // Gửi thông báo khi fail
        mail to: 'team@example.com',
             subject: "Build Failed: ${env.JOB_NAME}",
             body:    "Xem: ${env.BUILD_URL}"

        throw e    // Ném lại exception để Jenkins biết build fail

    } finally {
        // LUÔN chạy, dù thành công hay thất bại
        echo "Dọn dẹp workspace"
        cleanWs()
    }
}
```

### Xử Lý Lỗi Theo Stage

```groovy
node {
    stage('Test') {
        try {
            sh 'mvn test'
        } catch (e) {
            // Đánh dấu UNSTABLE thay vì FAILURE khi test fail
            currentBuild.result = 'UNSTABLE'
            echo "Có test thất bại nhưng pipeline tiếp tục"
        } finally {
            // Luôn publish test results
            junit allowEmptyResults: true,
                  testResults: 'target/surefire-reports/*.xml'
        }
    }

    stage('Deploy') {
        // Vẫn chạy dù test fail (nếu muốn)
        if (currentBuild.result != 'FAILURE') {
            sh './deploy.sh'
        }
    }
}
```

### `error` — Ném Lỗi Tùy Chỉnh

```groovy
node {
    stage('Validate') {
        def version = sh(script: 'cat VERSION', returnStdout: true).trim()
        if (!version.matches(/\d+\.\d+\.\d+/)) {
            error "Version '${version}' không đúng định dạng semver!"
            // Tương đương throw new Exception(...)
        }
    }
}
```

---

## Biến và Kiểu Dữ Liệu Groovy

```groovy
node {
    // Khai báo biến — dùng def (dynamic typing)
    def appName    = 'my-app'
    def version    = '1.0.0'
    def buildNum   = env.BUILD_NUMBER.toInteger()

    // Kiểu rõ ràng
    String registry = 'registry.example.com'
    boolean runTests = true
    int retryCount   = 3

    // String interpolation — dùng double quotes ""
    def imageTag = "${registry}/${appName}:${version}-${buildNum}"
    sh "docker build -t ${imageTag} ."

    // Multiline string
    def script = """
        echo "Building ${appName}"
        mvn clean package
        docker build -t ${imageTag} .
    """
    sh script

    // List (danh sách)
    def environments = ['staging', 'production']
    for (env in environments) {
        echo "Deploy lên: ${env}"
    }

    // Map (từ điển)
    def config = [
        registry: 'registry.example.com',
        namespace: 'production',
        replicas: 3
    ]
    echo "Registry: ${config.registry}"
    echo "Namespace: ${config['namespace']}"
}
```

### Lấy Output Từ Shell

```groovy
node {
    stage('Get Info') {
        // returnStdout: true — trả về output dưới dạng String
        def gitCommit = sh(
            script:       'git rev-parse --short HEAD',
            returnStdout: true
        ).trim()   // .trim() để bỏ dòng trắng cuối

        // returnStatus: true — trả về exit code (0 = thành công)
        def exitCode = sh(
            script:       'test -f config.yml',
            returnStatus: true
        )
        def configExists = (exitCode == 0)

        echo "Commit: ${gitCommit}"
        echo "Config exists: ${configExists}"

        // Kết hợp vào biến môi trường
        env.IMAGE_TAG = "myapp:${gitCommit}"
    }
}
```

---

## Điều Kiện và Vòng Lặp

Scripted Pipeline cho phép dùng toàn bộ cú pháp Groovy/Java:

### Câu Lệnh Điều Kiện

```groovy
node {
    // if/else
    if (env.BRANCH_NAME == 'main') {
        stage('Deploy Production') {
            sh './deploy-prod.sh'
        }
    } else if (env.BRANCH_NAME.startsWith('release/')) {
        stage('Deploy Staging') {
            sh './deploy-staging.sh'
        }
    } else {
        echo "Nhánh ${env.BRANCH_NAME} — bỏ qua deploy"
    }

    // switch/case
    switch (params.DEPLOY_ENV) {
        case 'production':
            sh './deploy-prod.sh'
            break
        case 'staging':
            sh './deploy-staging.sh'
            break
        default:
            echo "Môi trường không xác định: ${params.DEPLOY_ENV}"
    }

    // Toán tử ba ngôi (ternary)
    def namespace = (env.BRANCH_NAME == 'main') ? 'production' : 'staging'
    sh "kubectl apply -f k8s/ -n ${namespace}"

    // Elvis operator (null-safe)
    def tag = params.CUSTOM_TAG ?: env.BUILD_NUMBER
}
```

### Vòng Lặp

```groovy
node {
    // for truyền thống
    for (int i = 0; i < 3; i++) {
        echo "Lần thử ${i + 1}"
    }

    // for-each với List
    def services = ['auth-service', 'payment-service', 'user-service']
    for (service in services) {
        stage("Deploy ${service}") {
            sh "helm upgrade --install ${service} ./helm/${service}"
        }
    }

    // each với Closure
    services.each { service ->
        sh "kubectl rollout status deployment/${service}"
    }

    // collect — transform list
    def imageTags = services.collect { svc ->
        "registry.example.com/${svc}:${env.BUILD_NUMBER}"
    }

    // findAll — lọc list
    def failedServices = services.findAll { svc ->
        sh(script: "kubectl get pod -l app=${svc} | grep -v Running", returnStatus: true) == 0
    }

    // while
    int attempts = 0
    while (attempts < 5) {
        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health', returnStdout: true).trim()
        if (status == '200') {
            echo "Service sẵn sàng!"
            break
        }
        sleep(time: 10, unit: 'SECONDS')
        attempts++
    }
}
```

---

## Closure và Hàm Trong Groovy

### Khai Báo Hàm (Function)

Trong Scripted Pipeline, có thể định nghĩa hàm ở cấp script (bên ngoài `node`):

```groovy
// Hàm helper — định nghĩa NGOÀI node block
def deployToEnvironment(String env, String imageTag) {
    echo "Deploy ${imageTag} lên ${env}"
    withCredentials([string(credentialsId: "${env}-kubeconfig", variable: 'KUBECONFIG')]) {
        sh """
            helm upgrade --install myapp ./helm/myapp \
                --namespace ${env} \
                --set image.tag=${imageTag} \
                --wait
        """
    }
}

def runWithRetry(int maxAttempts, Closure action) {
    int attempt = 1
    while (attempt <= maxAttempts) {
        try {
            action()
            return    // Thành công — thoát
        } catch (e) {
            if (attempt == maxAttempts) throw e
            echo "Thất bại lần ${attempt}/${maxAttempts} — thử lại sau 30s"
            sleep(30)
            attempt++
        }
    }
}

// Sử dụng trong pipeline
node {
    def imageTag = "myapp:${env.BUILD_NUMBER}"

    stage('Build') {
        sh "docker build -t ${imageTag} ."
    }

    stage('Deploy Staging') {
        deployToEnvironment('staging', imageTag)
    }

    stage('Smoke Test') {
        runWithRetry(5) {
            sh 'curl --fail http://staging.example.com/health'
        }
    }

    if (env.BRANCH_NAME == 'main') {
        stage('Deploy Production') {
            deployToEnvironment('production', imageTag)
        }
    }
}
```

### Closure (Hàm Ẩn Danh)

```groovy
// Closure — hàm ẩn danh, gán vào biến
def notifySlack = { String message, String color ->
    slackSend channel: '#deployments', color: color, message: message
}

node {
    try {
        stage('Build') { sh 'mvn package' }
        notifySlack("✅ Build thành công: ${env.JOB_NAME}", 'good')
    } catch (e) {
        notifySlack("❌ Build thất bại: ${env.JOB_NAME}", 'danger')
        throw e
    }
}
```

---

## Parallel — Chạy Song Song

```groovy
node {
    stage('Parallel Tests') {
        // Định nghĩa các nhánh parallel trong Map
        def testBranches = [:]

        testBranches['Unit Tests'] = {
            node('linux') {
                unstash 'source'
                sh 'mvn test -Dtest=UnitTest*'
                junit 'target/surefire-reports/unit/*.xml'
            }
        }

        testBranches['Integration Tests'] = {
            node('linux') {
                unstash 'source'
                sh 'mvn verify -Pintegration'
                junit 'target/surefire-reports/integration/*.xml'
            }
        }

        testBranches['Security Scan'] = {
            node('security-scanner') {
                unstash 'source'
                sh 'mvn dependency-check:check'
            }
        }

        // Chạy song song tất cả nhánh
        parallel testBranches
    }
}
```

### Parallel Động (Dynamic Parallel)

```groovy
node {
    stage('Deploy to All Regions') {
        def regions = ['us-east-1', 'eu-west-1', 'ap-southeast-1']
        def deployBranches = [:]

        // Tạo parallel branch động từ list
        regions.each { region ->
            // Quan trọng: cần gán vào local variable trong closure
            // để tránh bug "variable capture" trong Groovy
            def localRegion = region
            deployBranches[localRegion] = {
                node('deploy-agent') {
                    withEnv(["AWS_REGION=${localRegion}"]) {
                        sh "helm upgrade --install myapp ./helm/myapp --kube-context=${localRegion}"
                    }
                }
            }
        }

        parallel deployBranches
    }
}
```

**Lưu ý quan trọng về Closure Capture:** Khi tạo parallel branch động trong vòng lặp, phải gán biến loop vào biến local trước khi dùng trong closure — vì Groovy closure capture theo reference, không phải by value.

```groovy
// SAI — tất cả closure đều dùng giá trị cuối cùng của 'region'
regions.each { region ->
    branches[region] = { sh "deploy ${region}" }    // Bug! region bị capture by reference
}

// ĐÚNG — gán vào local variable
regions.each { region ->
    def localRegion = region    // ✅ capture giá trị hiện tại
    branches[localRegion] = { sh "deploy ${localRegion}" }
}
```

---

## currentBuild — Đối Tượng Build Hiện Tại

`currentBuild` là đối tượng chứa thông tin và cho phép điều khiển build hiện tại:

```groovy
node {
    // Đọc thông tin build
    echo "Build number: ${currentBuild.number}"
    echo "Build URL:    ${currentBuild.absoluteUrl}"
    echo "Job name:     ${currentBuild.fullProjectName}"
    echo "Duration:     ${currentBuild.durationString}"

    // Kết quả build trước
    def previousBuild = currentBuild.previousBuild
    if (previousBuild != null) {
        echo "Kết quả build trước: ${previousBuild.result}"
    }

    // Đặt kết quả build
    currentBuild.result = 'SUCCESS'    // 'SUCCESS', 'UNSTABLE', 'FAILURE', 'ABORTED'

    // Đặt mô tả build (hiển thị trên Jenkins UI)
    currentBuild.description = "Deploy v${params.VERSION} lên ${params.ENV}"

    // Đặt displayName (tên hiển thị thay cho #123)
    currentBuild.displayName = "#${currentBuild.number} — v${params.VERSION}"

    // Kiểm tra kết quả
    script {
        if (currentBuild.result == 'UNSTABLE') {
            echo "Build UNSTABLE — có test thất bại nhưng pipeline tiếp tục"
        }
    }
}
```

---

## Shared Steps — Tái Sử Dụng Code

Scripted Pipeline cho phép tổ chức code tốt hơn khi kết hợp với Shared Libraries. Ngay cả không dùng Shared Libraries, có thể tái sử dụng code bằng cách định nghĩa hàm trong Jenkinsfile:

```groovy
// ========= Hàm Helper =========

def buildDockerImage(Map config) {
    def imageName = config.registry + '/' + config.name + ':' + config.tag
    sh "docker build -t ${imageName} -f ${config.dockerfile ?: 'Dockerfile'} ."
    return imageName
}

def pushDockerImage(String imageName, String credentialsId) {
    withCredentials([usernamePassword(
        credentialsId: credentialsId,
        usernameVariable: 'USER',
        passwordVariable: 'PASS'
    )]) {
        sh "docker login -u $USER -p $PASS"
        sh "docker push ${imageName}"
        sh "docker rmi ${imageName}"
    }
}

def deployWithHelm(Map config) {
    withCredentials([file(credentialsId: config.kubeconfigId, variable: 'KUBECONFIG')]) {
        sh """
            helm upgrade --install ${config.releaseName} ${config.chartPath} \
                --namespace ${config.namespace} \
                --set image.tag=${config.imageTag} \
                --wait --timeout ${config.timeout ?: '5m'}
        """
    }
}

def sendNotification(String status, String message) {
    def color = (status == 'success') ? 'good' : 'danger'
    def icon  = (status == 'success') ? '✅' : '❌'
    slackSend(
        channel: '#deployments',
        color:   color,
        message: "${icon} ${message}"
    )
}

// ========= Main Pipeline =========

def imageTag = ''

node('build-server') {
    try {
        stage('Checkout') {
            checkout scm
            imageTag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }

        stage('Build Image') {
            def imageName = buildDockerImage([
                registry:   'registry.example.com',
                name:       'payment-service',
                tag:        imageTag,
                dockerfile: 'Dockerfile.prod'
            ])

            pushDockerImage(imageName, 'registry-credentials')
            env.FINAL_IMAGE = imageName
        }

        stage('Deploy') {
            deployWithHelm([
                releaseName:  'payment-service',
                chartPath:    './helm/payment-service',
                namespace:    'production',
                imageTag:     imageTag,
                kubeconfigId: 'prod-kubeconfig',
                timeout:      '10m'
            ])
        }

        sendNotification('success', "payment-service:${imageTag} deployed thành công")
        currentBuild.result = 'SUCCESS'

    } catch (e) {
        sendNotification('failure', "payment-service:${imageTag} deploy thất bại")
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        cleanWs()
    }
}
```

---

## Scripted vs Declarative — Khi Nào Dùng Cái Nào

### Dùng Declarative Pipeline Khi

- Cần pipeline đơn giản đến trung bình độ phức tạp
- Muốn cú pháp rõ ràng, dễ đọc cho cả team
- Cần validation sớm trước khi chạy
- Người mới hoặc team chưa quen Groovy
- Cần các feature như `when`, `post`, `options` không cần logic phức tạp

### Dùng Scripted Pipeline Khi

- Cần logic điều kiện phức tạp không thể biểu diễn bằng `when`
- Xây dựng Shared Libraries (thường viết bằng Scripted style)
- Cần vòng lặp động để tạo stage/parallel dựa trên dữ liệu runtime
- Dùng các pattern Groovy nâng cao (closure, higher-order functions)
- Bảo trì Jenkinsfile legacy (nhiều dự án cũ dùng Scripted)

### Kết Hợp: `script { }` Trong Declarative Pipeline

Giải pháp tốt nhất thường là **Declarative Pipeline + `script { }` block cho logic phức tạp**:

```groovy
pipeline {
    agent any

    stages {
        stage('Determine Version') {
            steps {
                script {
                    // Logic Groovy phức tạp bên trong Declarative Pipeline
                    def tagOutput = sh(script: 'git describe --tags --abbrev=0 2>/dev/null || echo "0.0.0"', returnStdout: true).trim()
                    def parts = tagOutput.tokenize('.')
                    env.MAJOR = parts[0]
                    env.MINOR = parts[1]
                    env.PATCH = (parts[2].toInteger() + 1).toString()
                    env.NEW_VERSION = "${env.MAJOR}.${env.MINOR}.${env.PATCH}"
                }
                echo "Phiên bản tiếp theo: ${env.NEW_VERSION}"
            }
        }

        stage('Dynamic Deploy') {
            steps {
                script {
                    // Tạo parallel branches động trong Declarative Pipeline
                    def services = readJSON file: 'services.json'
                    def deployBranches = [:]
                    services.each { svc ->
                        def localSvc = svc
                        deployBranches[localSvc.name] = {
                            sh "helm upgrade --install ${localSvc.name} ./helm/${localSvc.name}"
                        }
                    }
                    parallel deployBranches
                }
            }
        }
    }
}
```

---

## Ví Dụ Thực Tế Hoàn Chỉnh

Pipeline phức tạp dùng Scripted Pipeline cho hệ thống microservices:

```groovy
// ========= Cấu hình toàn cục =========

def REGISTRY     = 'registry.company.com'
def SERVICES     = ['auth', 'payment', 'notification', 'api-gateway']
def DEPLOY_ORDER = [
    ['auth', 'payment', 'notification'],    // Deploy song song nhóm 1
    ['api-gateway']                         // Deploy nhóm 2 sau khi nhóm 1 hoàn thành
]

// ========= Hàm Helper =========

def buildAndPush(String service, String tag) {
    def image = "${REGISTRY}/${service}:${tag}"
    dir("services/${service}") {
        sh "docker build -t ${image} ."
    }
    withCredentials([usernamePassword(
        credentialsId: 'registry-creds',
        usernameVariable: 'U', passwordVariable: 'P'
    )]) {
        sh "docker login -u $U -p $P ${REGISTRY}"
        sh "docker push ${image}"
        sh "docker rmi ${image}"
    }
    return image
}

def healthCheck(String service, String namespace) {
    def url = "http://${service}.${namespace}.svc.cluster.local/health"
    retry(5) {
        sleep(time: 10, unit: 'SECONDS')
        sh "curl --fail --silent ${url}"
    }
}

// ========= Main Pipeline =========

node('build-master') {
    def commitSHA = ''
    def deployedImages = [:]

    try {
        stage('Checkout') {
            checkout scm
            commitSHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            currentBuild.displayName = "#${env.BUILD_NUMBER} — ${commitSHA}"

            // Detect changed services
            def changedFiles = sh(
                script: 'git diff --name-only HEAD~1 HEAD',
                returnStdout: true
            ).trim().split('\n')

            def changedServices = SERVICES.findAll { svc ->
                changedFiles.any { f -> f.startsWith("services/${svc}/") }
            }

            if (changedServices.isEmpty()) {
                echo "Không có service nào thay đổi — dừng pipeline"
                currentBuild.result = 'NOT_BUILT'
                return
            }

            echo "Services thay đổi: ${changedServices.join(', ')}"
            env.CHANGED_SERVICES = changedServices.join(',')
        }

        stage('Test') {
            def changedServices = env.CHANGED_SERVICES.split(',')
            def testBranches = [:]

            changedServices.each { svc ->
                def localSvc = svc
                testBranches["Test: ${localSvc}"] = {
                    node('test-runner') {
                        checkout scm
                        dir("services/${localSvc}") {
                            try {
                                sh 'mvn test'
                            } finally {
                                junit allowEmptyResults: true,
                                      testResults: 'target/surefire-reports/*.xml'
                            }
                        }
                    }
                }
            }

            parallel testBranches
        }

        stage('Build Images') {
            def changedServices = env.CHANGED_SERVICES.split(',')
            def buildBranches = [:]

            changedServices.each { svc ->
                def localSvc = svc
                buildBranches["Build: ${localSvc}"] = {
                    node('docker-builder') {
                        checkout scm
                        deployedImages[localSvc] = buildAndPush(localSvc, commitSHA)
                    }
                }
            }

            parallel buildBranches
        }

        // Deploy theo thứ tự nhóm
        DEPLOY_ORDER.each { group ->
            def groupName = group.join(', ')
            stage("Deploy: ${groupName}") {
                def deployBranches = [:]

                group.each { svc ->
                    if (deployedImages.containsKey(svc)) {
                        def localSvc = svc
                        deployBranches["Deploy: ${localSvc}"] = {
                            node('deploy-agent') {
                                withCredentials([file(credentialsId: 'prod-kubeconfig', variable: 'KUBECONFIG')]) {
                                    sh """
                                        helm upgrade --install ${localSvc} ./helm/${localSvc} \
                                            --namespace production \
                                            --set image.tag=${commitSHA} \
                                            --wait --timeout 5m
                                    """
                                }
                                healthCheck(localSvc, 'production')
                            }
                        }
                    }
                }

                if (!deployBranches.isEmpty()) {
                    parallel deployBranches
                }
            }
        }

        currentBuild.result = 'SUCCESS'
        slackSend(
            channel: '#deployments',
            color:   'good',
            message: "✅ Deploy ${commitSHA} thành công — Services: ${env.CHANGED_SERVICES}"
        )

    } catch (e) {
        currentBuild.result = 'FAILURE'
        slackSend(
            channel: '#deployments',
            color:   'danger',
            message: "❌ Deploy ${commitSHA} thất bại — ${e.getMessage()}"
        )
        throw e

    } finally {
        cleanWs()
    }
}
```

---

**Tiếp Theo:** Đọc [3-jenkinsfile.md](3-jenkinsfile.md) để hiểu cách quản lý Jenkinsfile trong SCM và các best practices.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
