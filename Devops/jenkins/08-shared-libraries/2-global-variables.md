# 2 — Global Variables và Custom Steps

> Global Variables (biến toàn cục) trong Jenkins Shared Library là cơ chế chính để tạo ra các **custom step** (bước tùy chỉnh) có thể gọi trực tiếp trong pipeline như `sh`, `echo` hay `docker.build`. Hiểu đúng cách viết, CPS (Continuation Passing Style — Kiểu Truyền Tiếp Tục), và `@NonCPS` là yếu tố quyết định thư viện của bạn hoạt động đúng.

---

## Mục Tiêu

- Viết Global Variable với hàm `call()` đơn giản và nâng cao
- Hiểu CPS (Continuation Passing Style) và tại sao nó quan trọng
- Sử dụng `@NonCPS` đúng cách để tránh lỗi serialization
- Truyền tham số linh hoạt: positional, named Map, Closure (hàm đóng gói)
- Gọi step Jenkins tiêu chuẩn từ bên trong library
- Xây dựng "pipeline skeleton" (khung pipeline) tái sử dụng cho toàn tổ chức

---

## Hàm `call()` — Trái Tim Của Global Variable

Khi Jenkins gặp `buildDockerImage(...)` trong Jenkinsfile, nó tìm file `vars/buildDockerImage.groovy` và gọi hàm `call(...)` trong đó. Đây là quy ước bắt buộc.

### Dạng đơn giản nhất

```groovy
// vars/sayHello.groovy
def call() {
    echo 'Hello from Shared Library!'
}

// Trong Jenkinsfile:
sayHello()   // → In ra: Hello from Shared Library!
```

### Với tham số vị trí (Positional Parameter)

```groovy
// vars/sayHello.groovy
def call(String name) {
    echo "Hello, ${name}!"
}

// Trong Jenkinsfile:
sayHello('Jenkins')   // → Hello, Jenkins!
```

### Với nhiều tham số vị trí

```groovy
// vars/tagDockerImage.groovy
def call(String imageName, String fromTag, String toTag) {
    sh "docker tag ${imageName}:${fromTag} ${imageName}:${toTag}"
    echo "Tagged: ${imageName}:${fromTag} → ${imageName}:${toTag}"
}

// Trong Jenkinsfile:
tagDockerImage('myapp', env.BUILD_NUMBER, 'latest')
```

---

## Named Parameters Bằng Map — Cách Khuyến Nghị

Dùng `Map config` giúp gọi linh hoạt, không phụ thuộc thứ tự tham số, và dễ mở rộng mà không phá vỡ pipeline hiện có.

```groovy
// vars/buildDockerImage.groovy
def call(Map config = [:]) {
    // Lấy giá trị với default (mặc định) bằng toán tử Elvis ?:
    def imageName = config.imageName ?: env.JOB_NAME.toLowerCase()
    def tag       = config.tag       ?: env.BUILD_NUMBER
    def registry  = config.registry  ?: 'docker.io'
    def context   = config.context   ?: '.'
    def dockerfile = config.dockerfile ?: 'Dockerfile'

    def fullName = "${registry}/${imageName}:${tag}"

    echo "=== Building Docker Image ==="
    echo "Image:      ${fullName}"
    echo "Context:    ${context}"
    echo "Dockerfile: ${dockerfile}"

    sh "docker build -f ${dockerfile} -t ${fullName} ${context}"
    sh "docker push ${fullName}"

    return fullName   // Trả về tên image đầy đủ để pipeline dùng tiếp
}
```

**Cách gọi — rất linh hoạt:**

```groovy
// Gọi với tất cả tham số
buildDockerImage(
    imageName:  'my-service',
    tag:        env.GIT_COMMIT[0..7],
    registry:   'registry.company.com',
    dockerfile: 'docker/Dockerfile.prod'
)

// Gọi với tham số tối thiểu — phần còn lại dùng default
buildDockerImage(imageName: 'my-service')

// Gọi không tham số — toàn bộ dùng default
buildDockerImage()

// Lưu giá trị trả về
def imageFullName = buildDockerImage(imageName: 'my-service', tag: '1.2.3')
sh "docker run ${imageFullName}"
```

---

## Groovy Named Arguments — Cú Pháp Đặc Biệt

Groovy cho phép bỏ ngoặc vuông `[:]` khi gọi hàm có tham số Map duy nhất:

```groovy
// Hai cách này tương đương nhau
buildDockerImage([imageName: 'app', tag: '1.0'])
buildDockerImage(imageName: 'app', tag: '1.0')   // Groovy tự gói thành Map
```

Đây là lý do tại sao `config = [:]` (Map rỗng là default) được dùng rộng rãi.

---

## Nhận Closure — Tạo Custom Block

Closure (hàm đóng gói) cho phép tạo custom block giống như `withCredentials {}`, `docker.withRegistry {}`.

```groovy
// vars/withDockerRegistry.groovy
def call(Map config, Closure body) {
    def registry    = config.registry
    def credId      = config.credentialsId

    withCredentials([usernamePassword(
        credentialsId: credId,
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    )]) {
        sh "echo ${DOCKER_PASS} | docker login ${registry} -u ${DOCKER_USER} --password-stdin"
        try {
            body()   // Thực thi block của người dùng
        } finally {
            sh "docker logout ${registry}"
        }
    }
}
```

**Cách dùng — trông như block tích hợp sẵn:**

```groovy
withDockerRegistry(registry: 'registry.company.com', credentialsId: 'docker-creds') {
    sh 'docker build -t registry.company.com/myapp:1.0 .'
    sh 'docker push registry.company.com/myapp:1.0'
}
```

---

## CPS — Continuation Passing Style

### CPS Là Gì?

Jenkins pipeline chạy trên một cơ chế đặc biệt gọi là **CPS (Continuation Passing Style — Kiểu Truyền Tiếp Tục)**. CPS cho phép Jenkins:

- **Tạm dừng và tiếp tục** pipeline sau khi master restart (khởi động lại)
- **Serialize** (lưu trạng thái) tất cả biến cục bộ vào disk
- **Resume** (tiếp tục) từ đúng vị trí sau sự cố

Để làm được điều này, mọi code trong pipeline phải **serializable** (có thể tuần tự hóa). Code trong `vars/` mặc định được Jenkins biên dịch theo cơ chế CPS.

### Vấn Đề Với CPS

Một số cấu trúc Groovy **không tương thích** với CPS và gây lỗi `NotSerializableException`:

```groovy
// vars/problematicStep.groovy

def call() {
    // ❌ Lỗi: Java stream (luồng Java) không serializable
    def result = ['a', 'b', 'c'].stream()
                                 .filter { it != 'b' }
                                 .collect()

    // ❌ Lỗi: Iterator (bộ lặp) không serializable trong một số trường hợp
    def map = [key1: 'val1', key2: 'val2']
    map.each { k, v ->
        // Vòng lặp .each với closure phức tạp có thể gây lỗi
        processEntry(k, v)
    }
}
```

### Giải Pháp: `@NonCPS`

Đánh dấu `@NonCPS` cho hàm không cần serialize — Jenkins sẽ chạy hàm này theo cách thông thường (không CPS):

```groovy
// vars/processData.groovy
import groovy.transform.Field

def call(List<String> items) {
    // Hàm chính vẫn là CPS
    def filtered = filterItems(items)     // Gọi hàm @NonCPS
    echo "Filtered: ${filtered.join(', ')}"
}

@NonCPS
private List<String> filterItems(List<String> items) {
    // Hàm @NonCPS — có thể dùng Java streams, Iterator, v.v.
    return items.stream()
                .filter { it.length() > 2 }
                .sorted()
                .collect()
}
```

**Quy tắc `@NonCPS`:**

| Được phép | Không được phép |
|-----------|-----------------|
| Java streams, Iterator | Gọi pipeline step (`sh`, `echo`, `withCredentials`) |
| Xử lý String phức tạp | Gọi hàm CPS khác |
| Sort, filter, map | Đọc/ghi file (`readFile`, `writeFile`) |
| Regex phức tạp | Dùng `env`, `params`, `currentBuild` |

---

## Các Biến Đặc Biệt Trong Library

Khi chạy trong context pipeline, code trong `vars/` có thể truy cập trực tiếp các đối tượng sau:

```groovy
// vars/showBuildInfo.groovy
def call() {
    // env — biến môi trường của build
    echo "Job Name:    ${env.JOB_NAME}"
    echo "Build No:    ${env.BUILD_NUMBER}"
    echo "Branch:      ${env.GIT_BRANCH}"
    echo "Commit:      ${env.GIT_COMMIT}"
    echo "Workspace:   ${env.WORKSPACE}"

    // currentBuild — thông tin build hiện tại
    echo "Build URL:   ${currentBuild.absoluteUrl}"
    echo "Build Status: ${currentBuild.currentResult}"
    currentBuild.displayName = "#${env.BUILD_NUMBER}-${env.GIT_BRANCH}"

    // params — tham số build (nếu pipeline có parameters)
    if (params.DEPLOY_ENV) {
        echo "Deploy to: ${params.DEPLOY_ENV}"
    }
}
```

---

## Xử Lý Credentials Trong Library

Không bao giờ hardcode (mã hóa cứng) credentials. Nhận `credentialsId` từ người gọi và dùng `withCredentials`:

```groovy
// vars/deployToKubernetes.groovy
def call(Map config) {
    def kubeconfigId = config.kubeconfigCredId ?: 'kubeconfig-prod'
    def namespace    = config.namespace         ?: 'default'
    def manifest     = config.manifestFile      ?: 'deploy.yaml'

    withCredentials([file(credentialsId: kubeconfigId, variable: 'KUBECONFIG')]) {
        sh """
            export KUBECONFIG=${KUBECONFIG}
            kubectl apply -f ${manifest} -n ${namespace}
            kubectl rollout status deployment/${config.appName} -n ${namespace}
        """
    }
}
```

**Cách gọi:**

```groovy
deployToKubernetes(
    appName:          'my-service',
    namespace:        'production',
    kubeconfigCredId: 'kubeconfig-aws-prod',
    manifestFile:     'k8s/deployment.yaml'
)
```

---

## Viết "Pipeline Skeleton" — Khung Pipeline Chuẩn

Đây là pattern (mẫu thiết kế) quan trọng nhất của Shared Library: định nghĩa toàn bộ cấu trúc pipeline chuẩn, để mọi team chỉ cần khai báo tham số.

```groovy
// vars/standardCIPipeline.groovy
def call(Map config, Closure additionalStages = null) {

    // Thiết lập giá trị mặc định
    def buildTool = config.buildTool ?: 'maven'
    def registry  = config.registry  ?: 'registry.company.com'
    def namespace  = config.namespace  ?: 'staging'

    pipeline {
        agent {
            kubernetes {
                yaml libraryResource('com/company/jenkins/pod-template.yaml')
            }
        }

        options {
            timeout(time: 30, unit: 'MINUTES')
            disableConcurrentBuilds()
            buildDiscarder(logRotator(numToKeepStr: '10'))
        }

        environment {
            IMAGE_NAME = "${registry}/${config.appName}"
            IMAGE_TAG  = "${env.BUILD_NUMBER}-${env.GIT_COMMIT[0..6]}"
        }

        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                }
            }

            stage('Build') {
                steps {
                    script {
                        if (buildTool == 'maven') {
                            sh 'mvn clean package -DskipTests'
                        } else if (buildTool == 'gradle') {
                            sh './gradlew build -x test'
                        } else if (buildTool == 'npm') {
                            sh 'npm ci && npm run build'
                        }
                    }
                }
            }

            stage('Test') {
                steps {
                    script {
                        if (buildTool == 'maven') {
                            sh 'mvn test'
                        } else if (buildTool == 'gradle') {
                            sh './gradlew test'
                        } else if (buildTool == 'npm') {
                            sh 'npm test'
                        }
                    }
                }
                post {
                    always {
                        junit '**/target/surefire-reports/*.xml'
                    }
                }
            }

            stage('Docker Build & Push') {
                when {
                    branch pattern: 'main|develop|release/.*', comparator: 'REGEXP'
                }
                steps {
                    buildDockerImage(
                        imageName: config.appName,
                        tag:       env.IMAGE_TAG,
                        registry:  registry
                    )
                }
            }

            stage('Custom Stages') {
                when {
                    expression { additionalStages != null }
                }
                steps {
                    script {
                        additionalStages()   // Gọi custom stages từ Jenkinsfile
                    }
                }
            }

            stage('Deploy to Staging') {
                when {
                    branch 'develop'
                }
                steps {
                    deployToKubernetes(
                        appName:   config.appName,
                        namespace: 'staging',
                        image:     "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                    )
                }
            }
        }

        post {
            success {
                sendSlackNotification(
                    channel: config.slackChannel ?: '#ci-cd',
                    status:  'SUCCESS',
                    message: "${config.appName} build #${env.BUILD_NUMBER} thành công"
                )
            }
            failure {
                sendSlackNotification(
                    channel: config.slackChannel ?: '#ci-cd',
                    status:  'FAILURE',
                    message: "${config.appName} build #${env.BUILD_NUMBER} thất bại"
                )
            }
        }
    }
}
```

**Jenkinsfile của từng project — cực kỳ gọn:**

```groovy
@Library('company-jenkins-lib@v2.1.0') _

standardCIPipeline(
    appName:      'payment-service',
    buildTool:    'maven',
    registry:     'registry.company.com',
    slackChannel: '#team-payment'
) {
    // Custom stages bổ sung (tùy chọn)
    stage('Integration Test') {
        sh 'mvn verify -Pintegration-test'
    }
}
```

---

## Tránh Các Antipattern (Mẫu Xấu)

### Antipattern 1: Hardcode Logic Đặc Thù Dự Án

```groovy
// ❌ Sai — logic quá đặc thù, không tái dùng được
def call() {
    sh 'mvn clean package'               // hardcode Maven
    sh 'docker build -t myapp:latest .'  // hardcode tên app
    sh 'kubectl apply -f k8s/'           // hardcode file
}

// ✅ Đúng — nhận tham số, linh hoạt
def call(Map config) {
    sh "${config.buildCommand}"
    sh "docker build -t ${config.imageName}:${config.tag} ."
    sh "kubectl apply -f ${config.manifestDir}/"
}
```

### Antipattern 2: Nuốt Exception (Bắt Lỗi Không Xử Lý)

```groovy
// ❌ Sai — nuốt exception, pipeline không biết có lỗi
def call() {
    try {
        sh 'docker push myimage:latest'
    } catch (e) {
        echo "Có lỗi nhưng tiếp tục..."   // Nguy hiểm!
    }
}

// ✅ Đúng — ném lại exception hoặc xử lý có chủ đích
def call() {
    try {
        sh 'docker push myimage:latest'
    } catch (e) {
        currentBuild.result = 'FAILURE'
        error "Docker push thất bại: ${e.message}"   // Dừng pipeline
    }
}
```

### Antipattern 3: Dùng `@NonCPS` Quá Rộng

```groovy
// ❌ Sai — @NonCPS toàn bộ hàm chứa pipeline step
@NonCPS
def call() {
    sh 'docker build .'   // Lỗi: không thể gọi pipeline step trong @NonCPS
}

// ✅ Đúng — chỉ @NonCPS hàm helper thuần Groovy
def call() {
    def config = parseConfig()    // Gọi hàm @NonCPS
    sh "docker build ${config}"   // Pipeline step ở hàm CPS chính
}

@NonCPS
private String parseConfig() {
    return '--build-arg ENV=production --no-cache'
}
```

---

## Ví Dụ Thực Tế: Bộ Step Cho CI/CD

### Step 1: runTests.groovy

```groovy
// vars/runTests.groovy
def call(Map config = [:]) {
    def buildTool = config.buildTool ?: 'maven'
    def reportDir = config.reportDir  ?: '**/target/surefire-reports/*.xml'

    stage('Unit Tests') {
        sh buildCommand(buildTool)
        junit reportDir
        publishHTML([
            allowMissing: false,
            reportDir:    'target/site/jacoco',
            reportFiles:  'index.html',
            reportName:   'Code Coverage Report'
        ])
    }
}

@NonCPS
private String buildCommand(String tool) {
    switch(tool) {
        case 'maven':  return 'mvn test'
        case 'gradle': return './gradlew test'
        case 'npm':    return 'npm test -- --ci'
        default:       return "echo 'Unknown build tool: ${tool}'"
    }
}
```

### Step 2: sendSlackNotification.groovy

```groovy
// vars/sendSlackNotification.groovy
def call(Map config) {
    def channel = config.channel ?: '#ci-cd'
    def status  = config.status  ?: currentBuild.currentResult
    def message = config.message ?: buildDefaultMessage(status)

    def color = statusToColor(status)

    slackSend(
        channel:  channel,
        color:    color,
        message:  message
    )
}

@NonCPS
private String statusToColor(String status) {
    switch(status.toUpperCase()) {
        case 'SUCCESS': return 'good'
        case 'FAILURE': return 'danger'
        case 'UNSTABLE': return 'warning'
        default:         return '#439FE0'
    }
}

@NonCPS
private String buildDefaultMessage(String status) {
    return "${env.JOB_NAME} #${env.BUILD_NUMBER} — ${status}"
}
```

---

## Câu Hỏi Phỏng Vấn

1. **CPS là gì trong Jenkins pipeline? Tại sao nó quan trọng?**
   → CPS là cơ chế Jenkins dùng để có thể serialize và resume pipeline sau khi master restart. Mọi code pipeline phải serializable.

2. **Khi nào nên dùng `@NonCPS`? Hạn chế của nó là gì?**
   → Dùng cho hàm helper thuần Groovy không cần serialize (stream, regex phức tạp). Hàm `@NonCPS` không được gọi pipeline step hoặc hàm CPS.

3. **Sự khác biệt giữa truyền tham số vị trí và named Map?**
   → Named Map linh hoạt hơn, không phụ thuộc thứ tự, dễ thêm tham số mới mà không phá vỡ caller cũ.

4. **Pipeline skeleton là gì? Lợi ích khi dùng cho toàn tổ chức?**
   → Định nghĩa toàn bộ cấu trúc pipeline trong library, mỗi project chỉ khai báo tham số. Đảm bảo nhất quán, dễ update chính sách CI/CD tập trung.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
