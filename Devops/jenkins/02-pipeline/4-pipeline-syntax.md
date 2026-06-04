# Pipeline Syntax — Tham Chiếu Cú Pháp Đầy Đủ

> Tài liệu tham chiếu toàn bộ cú pháp Jenkins Pipeline: directive, step tích hợp sẵn, biến môi trường, và các lệnh hay dùng. Dùng file này như một cheatsheet khi viết hoặc debug Jenkinsfile.

## Mục Lục

1. [Declarative Pipeline — Cấu Trúc Đầy Đủ](#declarative-pipeline--cấu-trúc-đầy-đủ)
2. [Directive Cấp Pipeline](#directive-cấp-pipeline)
3. [Directive Cấp Stage](#directive-cấp-stage)
4. [Built-in Steps — Bước Tích Hợp Sẵn](#built-in-steps--bước-tích-hợp-sẵn)
5. [Biến Môi Trường Tự Động](#biến-môi-trường-tự-động)
6. [Điều Kiện when — Tham Chiếu Đầy Đủ](#điều-kiện-when--tham-chiếu-đầy-đủ)
7. [Các Loại Credentials Step](#các-loại-credentials-step)
8. [Script Block — Sử Dụng Groovy](#script-block--sử-dụng-groovy)
9. [Pipeline Snippet Generator](#pipeline-snippet-generator)
10. [Groovy String Cheatsheet](#groovy-string-cheatsheet)

---

## Declarative Pipeline — Cấu Trúc Đầy Đủ

Sơ đồ tất cả directive có thể dùng trong Declarative Pipeline:

```
pipeline {
    agent { ... }                     ← Bắt buộc (hoặc agent none)
    │
    ├── options { ... }               ← Tùy chọn
    ├── environment { ... }           ← Biến môi trường
    ├── parameters { ... }            ← Tham số đầu vào
    ├── tools { ... }                 ← Công cụ build
    ├── triggers { ... }              ← Trigger tự động
    │
    ├── stages {                      ← Bắt buộc
    │   ├── stage('Name') {
    │   │   ├── agent { ... }         ← Override agent cấp pipeline
    │   │   ├── when { ... }          ← Điều kiện chạy
    │   │   ├── options { ... }       ← Override options cấp pipeline
    │   │   ├── environment { ... }   ← Biến môi trường cục bộ
    │   │   ├── tools { ... }         ← Override tools
    │   │   ├── input { ... }         ← Chờ xác nhận
    │   │   ├── steps { ... }         ← Các bước thực thi
    │   │   └── post { ... }          ← Xử lý sau stage
    │   │
    │   └── stage('Parallel') {
    │       └── parallel {
    │           ├── stage('Branch A') { ... }
    │           └── stage('Branch B') { ... }
    │       }
    │   }
    │
    └── post {                        ← Xử lý sau toàn pipeline
        ├── always  { ... }
        ├── success { ... }
        ├── failure { ... }
        ├── unstable { ... }
        ├── changed { ... }
        ├── aborted { ... }
        └── cleanup { ... }
    }
}
```

---

## Directive Cấp Pipeline

### `agent` — Toàn Bộ Các Tùy Chọn

```groovy
// Bất kỳ agent nào
agent any

// Không agent mặc định (chỉ định riêng cho từng stage)
agent none

// Theo nhãn
agent { label 'linux' }
agent { label 'linux && docker' }     // Phải có CẢ HAI nhãn
agent { label 'linux || mac' }        // Có nhãn HOẶC

// Docker container
agent {
    docker {
        image           'maven:3.8-jdk-17'
        label           'docker-agent'        // Chạy trên agent có nhãn này
        args            '-v /tmp:/tmp --cpus=2'
        registryUrl     'https://registry.example.com'
        registryCredentialsId 'registry-creds'
        reuseNode       true    // Tái dùng workspace hiện tại
        alwaysPull      true    // Luôn pull image mới nhất
    }
}

// Từ Dockerfile
agent {
    dockerfile {
        filename       'Dockerfile.ci'
        dir            'ci/'
        label          'linux'
        additionalBuildArgs '--build-arg JAVA_VERSION=17'
        args           '-v /tmp:/tmp'
    }
}

// Kubernetes Pod
agent {
    kubernetes {
        label         'k8s-agent'     // Label của pod (override tự động tạo)
        defaultContainer 'jnlp'
        yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8-jdk-17
    command: [sleep]
    args:    ["9999"]
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
      limits:   { cpu: "2",    memory: "4Gi" }
  - name: docker
    image: docker:23-dind
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
```

### `options` — Tất Cả Tùy Chọn

```groovy
options {
    // Xóa build cũ
    buildDiscarder(logRotator(
        numToKeepStr:          '30',     // Giữ 30 build
        daysToKeepStr:         '90',     // Hoặc giữ trong 90 ngày
        artifactNumToKeepStr:  '5',      // Giữ artifact 5 build
        artifactDaysToKeepStr: '30'
    ))

    // Timeout toàn pipeline
    timeout(time: 60, unit: 'MINUTES')    // MINUTES, HOURS, SECONDS, DAYS

    // Retry khi thất bại (tổng số lần chạy)
    retry(3)

    // Không chạy song song
    disableConcurrentBuilds()
    disableConcurrentBuilds(abortPrevious: true)    // Hủy build trước khi có build mới

    // Bỏ checkout mặc định
    skipDefaultCheckout()

    // Bỏ giai đoạn tổng quan (stage overview) khi không có stage nào fail
    skipStagesAfterUnstable()

    // Thêm timestamp vào log
    timestamps()

    // Màu ANSI trong console log (cần AnsiColor plugin)
    ansiColor('xterm')

    // Không cho phép resume sau khi Jenkins restart
    disableRestart()

    // Giữ build artifact dù build bị xóa
    preserveStashes()
    preserveStashes(buildCount: 5)

    // Lock resource (cần Lockable Resources plugin)
    lock(resource: 'production-deploy-lock')

    // Checkmark build thành công dù có stage unstable
    quietPeriod(10)    // Chờ 10 giây trước khi bắt đầu (để gộp nhiều commit)

    // Dùng Declarative Checkout (tương thích hơn)
    checkoutToSubdirectory('source')

    // Thêm properties cho job
    rateLimitBuilds(throttle: [count: 1, durationName: 'minute'])
}
```

### `environment` — Khai Báo Biến

```groovy
environment {
    // Biến thường
    APP_NAME = 'my-app'
    VERSION  = '1.0.0'

    // Tham chiếu biến khác
    IMAGE_TAG = "${APP_NAME}:${VERSION}"

    // Lấy từ Credentials (secret text)
    API_TOKEN = credentials('my-api-token')

    // Lấy từ Credentials (username/password)
    // Tự động tạo: DOCKER_CREDS_USR và DOCKER_CREDS_PSW
    DOCKER_CREDS = credentials('docker-registry-creds')

    // Lấy từ file credentials
    // Tự động tạo biến chứa đường dẫn đến file tạm
    KUBECONFIG = credentials('kubeconfig-secret-file')

    // Lấy output shell
    GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
}
```

### `parameters` — Tất Cả Loại Tham Số

```groovy
parameters {
    string(
        name:         'DEPLOY_ENV',
        defaultValue: 'staging',
        trim:         true,            // Tự động trim whitespace
        description:  'Môi trường deploy'
    )

    text(
        name:         'CHANGELOG',
        defaultValue: '',
        description:  'Nội dung changelog (nhiều dòng)'
    )

    choice(
        name:    'REGION',
        choices: ['us-east-1', 'eu-west-1', 'ap-southeast-1'],
        description: 'AWS Region'
    )

    booleanParam(
        name:         'SKIP_TESTS',
        defaultValue: false,
        description:  'Bỏ qua test'
    )

    file(
        name:        'CONFIG_OVERRIDE',
        description: 'Upload file cấu hình tùy chỉnh'
    )

    password(
        name:         'TEMP_SECRET',
        defaultValue: '',
        description:  'Mật khẩu tạm (không lưu vào history)'
    )

    // Active Choices (cần Active Choices plugin)
    // Tạo lựa chọn động dựa trên giá trị tham số khác
}
```

Truy cập tham số: `params.DEPLOY_ENV`, `params.SKIP_TESTS`

### `tools` — Công Cụ Được Hỗ Trợ

```groovy
tools {
    // Tên phải khớp với Global Tool Configuration
    maven  'Maven 3.9'
    jdk    'OpenJDK 21'
    gradle 'Gradle 8'
    nodejs 'Node.js 20 LTS'
    go     'Go 1.22'
    git    'Git 2.44'
    // Custom tool (cần Custom Tools plugin)
    custom 'my-custom-tool'
}
```

### `triggers` — Kích Hoạt Tự Động

```groovy
triggers {
    // Cron — chạy theo lịch
    // Cú pháp: MINUTE HOUR DOM MONTH DOW
    cron('H 2 * * 1-5')           // Mỗi ngày trong tuần lúc 2:xx sáng
    cron('@daily')                 // Mỗi ngày
    cron('@weekly')                // Mỗi tuần
    cron('H/15 * * * *')          // Mỗi 15 phút (H = hash, phân tán tải)

    // Poll SCM — tự kiểm tra thay đổi
    pollSCM('H/5 * * * *')        // Kiểm tra mỗi 5 phút

    // Upstream trigger — chạy sau khi job khác hoàn thành
    upstream(
        upstreamProjects: 'other-job,another-job',
        threshold:        hudson.model.Result.SUCCESS
    )

    // GitLab trigger (cần GitLab plugin)
    gitlab(
        triggerOnPush:                true,
        triggerOnMergeRequest:        true,
        branchFilterType:             'All',
        secretToken:                  'abc123'
    )

    // GitHub trigger (thông qua webhook)
    // Không cần cấu hình trigger — webhook gọi trực tiếp
}
```

---

## Directive Cấp Stage

### `when` — Điều Kiện Đầy Đủ

Xem [mục riêng bên dưới](#điều-kiện-when--tham-chiếu-đầy-đủ).

### `input` — Tạm Dừng Chờ Xác Nhận

```groovy
input {
    message    'Deploy lên Production?'
    id         'production-approval'          // ID để tham chiếu programmatically
    ok         'Phê Duyệt Deploy'
    submitter  'alice,ops-lead,bob'           // User hoặc group được phép xác nhận
    submitterParameter 'APPROVER'             // Lưu tên người xác nhận vào biến
    parameters {                              // Thu thập thêm thông tin khi approve
        choice(
            name:    'DEPLOY_STRATEGY',
            choices: ['rolling', 'blue-green', 'canary'],
            description: 'Chiến lược deploy'
        )
        string(
            name:        'ROLLBACK_VERSION',
            description: 'Phiên bản rollback nếu cần'
        )
    }
}
```

---

## Built-in Steps — Bước Tích Hợp Sẵn

### Shell và Lệnh Hệ Thống

```groovy
// sh — Linux/macOS
sh 'command'
sh '''
    line 1
    line 2
'''
sh(script: 'command', returnStdout: true)    // Trả về output
sh(script: 'command', returnStatus: true)    // Trả về exit code

// bat — Windows batch
bat 'command'
bat(script: 'command', returnStdout: true)

// powershell — Windows PowerShell
powershell 'Get-Process'
powershell(script: 'Write-Output "hello"', returnStdout: true)
```

### File Operations (Thao Tác File)

```groovy
// Đọc file
def content    = readFile 'path/to/file.txt'
def yamlData   = readYaml  file: 'config.yml'
def jsonData   = readJSON  file: 'package.json'
def props      = readProperties file: 'build.properties'
def csvContent = readCSV  file: 'data.csv'

// Ghi file
writeFile  file: 'output.txt', text: 'content here', encoding: 'UTF-8'
writeYaml  file: 'output.yml', data: [key: 'value']
writeJSON  file: 'output.json', json: [name: 'app', version: '1.0']

// Kiểm tra file tồn tại
def exists = fileExists 'config.yml'

// Thao tác thư mục
dir('subdirectory') {
    // Mọi lệnh trong block này chạy trong thư mục 'subdirectory'
    sh 'ls'
}

// Xóa thư mục
deleteDir()    // Xóa thư mục hiện tại
```

### Source Control (Quản Lý Mã Nguồn)

```groovy
// Checkout source code (dùng cấu hình của job)
checkout scm

// Checkout tùy chỉnh
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    userRemoteConfigs: [[
        url:           'https://github.com/org/repo.git',
        credentialsId: 'github-creds'
    ]],
    extensions: [
        [$class: 'CloneOption',
         shallow: true, depth: 1],          // Shallow clone
        [$class: 'SubmoduleOption',
         recursiveSubmodules: true]          // Submodules
    ]
])

// Lấy thông tin git
def commitSHA   = sh(script: 'git rev-parse HEAD',       returnStdout: true).trim()
def shortSHA    = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
def authorEmail = sh(script: 'git log -1 --format="%ae"', returnStdout: true).trim()
def commitMsg   = sh(script: 'git log -1 --format="%s"',  returnStdout: true).trim()
```

### Artifact Management (Quản Lý Artifact)

```groovy
// Archive — lưu artifact cho build này
archiveArtifacts artifacts:   'target/*.jar, dist/**',
                 fingerprint:  true,         // Tạo fingerprint để tracking
                 allowEmptyArchive: true,    // Không fail nếu không có file
                 onlyIfSuccessful: true      // Chỉ archive khi build success

// Stash / Unstash — chia sẻ file giữa các stage/agent
stash name:     'compiled',
      includes: 'target/*.jar,Dockerfile',
      excludes: 'target/*-sources.jar'

unstash 'compiled'    // Khôi phục stash trên agent hiện tại

// Copy artifact từ job khác (cần Copy Artifact plugin)
copyArtifacts projectName:  'upstream-job',
              filter:        '*.jar',
              selector:      lastSuccessful(),
              target:        'libs/'

// Publish JUnit test results
junit testResults:      'target/surefire-reports/**/*.xml',
      allowEmptyResults: true,
      healthScaleFactor: 2.0    // Scaling factor cho coverage
```

### Thông Báo (Notification)

```groovy
// Email (cần Mailer plugin)
mail to:      'team@example.com, devops@example.com',
     subject:  "Build ${currentBuild.result}: ${env.JOB_NAME}",
     body:     "Build URL: ${env.BUILD_URL}",
     mimeType: 'text/html'

// Slack (cần Slack Notification plugin)
slackSend channel:   '#ci-cd',
          color:     'good',        // good, warning, danger
          message:   "Build thành công: ${env.JOB_NAME}"

slackSend channel:  '#ci-cd',
          color:    '#FF0000',      // Hoặc hex color
          blocks:   [              // Block Kit message
              [type: 'section',
               text: [type: 'mrkdwn', text: "*Build thất bại!*\n${env.BUILD_URL}"]]
          ]
```

### Pipeline Control (Điều Khiển Pipeline)

```groovy
// Dừng pipeline và đánh dấu fail
error 'Phát hiện lỗi nghiêm trọng!'

// Dừng pipeline sạch (không phải failure)
script {
    currentBuild.result = 'ABORTED'
    error 'Build bị hủy theo yêu cầu'
}

// Retry — thử lại n lần
retry(3) {
    sh './flaky-command.sh'
}

// Timeout — hết thời gian → fail
timeout(time: 5, unit: 'MINUTES') {
    sh './slow-command.sh'
}

// Chờ đến khi điều kiện đúng
waitUntil {
    def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" http://localhost/health', returnStdout: true).trim()
    return status == '200'
}

// Sleep — ngủ
sleep time: 30, unit: 'SECONDS'    // SECONDS, MINUTES, HOURS

// Echo — in ra log
echo "Build number: ${env.BUILD_NUMBER}"

// Milestone — kiểm soát thứ tự build
milestone(label: 'after-tests', ordinal: 1)    // Build cũ hơn bị hủy khi đến đây

// Lock — khóa resource để tránh race condition
lock('deploy-production') {
    sh './deploy.sh'
}
```

### Workspace Management (Quản Lý Workspace)

```groovy
// Dọn dẹp workspace (cần Workspace Cleanup plugin)
cleanWs()
cleanWs(
    cleanWhenAborted:     true,
    cleanWhenFailure:     false,    // Giữ workspace khi fail (để debug)
    cleanWhenNotBuilt:    true,
    cleanWhenSuccess:     true,
    cleanWhenUnstable:    true,
    deleteDirs:           true,
    disableDeferredWipeout: false,
    patterns: [[pattern: '**/.git', type: 'EXCLUDE']]
)

// Lấy đường dẫn workspace
echo "Workspace: ${env.WORKSPACE}"

// Chạy trong thư mục con
dir('subdir') {
    sh 'pwd'    // Sẽ in: /path/to/workspace/subdir
}

// Tạo thư mục tạm (xóa sau block)
withTempDir {
    sh 'cp -r src/ .'
    sh 'make build'
}
```

---

## Biến Môi Trường Tự Động

Jenkins tự động inject các biến môi trường sau vào mọi build:

### Biến Jenkins Chuẩn

| Biến                       | Mô Tả                                                  | Ví Dụ                                          |
| -------------------------- | ------------------------------------------------------ | ---------------------------------------------- |
| `BUILD_NUMBER`             | Số thứ tự build                                        | `42`                                           |
| `BUILD_ID`                 | ID của build (thường = BUILD_NUMBER)                   | `42`                                           |
| `BUILD_DISPLAY_NAME`       | Tên hiển thị trên UI                                   | `#42`                                          |
| `BUILD_TAG`                | Chuỗi định danh duy nhất cho build                    | `jenkins-my-job-42`                            |
| `BUILD_URL`                | URL đầy đủ của build                                   | `https://jenkins.example.com/job/my-job/42/`  |
| `JOB_NAME`                 | Tên job                                                | `my-pipeline`                                  |
| `JOB_BASE_NAME`            | Tên job không có đường dẫn folder                     | `my-pipeline`                                  |
| `JOB_URL`                  | URL của job                                            | `https://jenkins.example.com/job/my-pipeline/`|
| `JENKINS_URL`              | URL gốc của Jenkins                                    | `https://jenkins.example.com/`                 |
| `JENKINS_HOME`             | Thư mục home của Jenkins                               | `/var/jenkins_home`                            |
| `WORKSPACE`                | Đường dẫn workspace trên agent                         | `/var/lib/jenkins/workspace/my-pipeline`       |
| `NODE_NAME`                | Tên của agent đang chạy                                | `linux-agent-01`                               |
| `NODE_LABELS`              | Danh sách nhãn của agent                               | `linux docker maven`                           |
| `EXECUTOR_NUMBER`          | Số thứ tự executor trên agent                          | `0`                                            |

### Biến Git (Khi Dùng Git SCM)

| Biến                   | Mô Tả                                 | Ví Dụ                                     |
| ---------------------- | ------------------------------------- | ----------------------------------------- |
| `GIT_COMMIT`           | SHA commit đầy đủ                     | `abc123def456...`                         |
| `GIT_PREVIOUS_COMMIT`  | SHA commit của build trước            | `xyz789...`                               |
| `GIT_BRANCH`           | Tên nhánh (dạng origin/main)          | `origin/main`                             |
| `GIT_LOCAL_BRANCH`     | Tên nhánh local                       | `main`                                    |
| `GIT_URL`              | URL remote                            | `https://github.com/org/repo.git`         |
| `GIT_URL_1`            | URL remote đầu tiên (nếu nhiều remote)| `https://github.com/org/repo.git`         |

### Biến Multibranch Pipeline

| Biến                  | Mô Tả                                              | Ví Dụ                    |
| --------------------- | -------------------------------------------------- | ------------------------ |
| `BRANCH_NAME`         | Tên nhánh hiện tại                                 | `feature/my-feature`     |
| `BRANCH_IS_PRIMARY`   | Có phải nhánh chính không                          | `true` hoặc `false`      |
| `CHANGE_ID`           | ID của Pull Request (nếu là PR build)              | `42`                     |
| `CHANGE_URL`          | URL của Pull Request                               | `https://github.com/...` |
| `CHANGE_TITLE`        | Tiêu đề Pull Request                               | `Add feature X`          |
| `CHANGE_AUTHOR`       | Username của tác giả PR                            | `alice`                  |
| `CHANGE_AUTHOR_EMAIL` | Email của tác giả PR                               | `alice@example.com`      |
| `CHANGE_TARGET`       | Nhánh đích của PR (thường là main)                 | `main`                   |

---

## Điều Kiện when — Tham Chiếu Đầy Đủ

### Điều Kiện Đơn Lẻ

```groovy
// Theo tên nhánh (hỗ trợ wildcard)
when { branch 'main' }
when { branch 'release/*' }
when { branch pattern: '^release/\\d+\\.\\d+$', comparator: 'REGEXP' }

// Theo tag git
when { tag 'v*' }
when { tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP' }

// Khi là Pull Request
when { changeRequest() }
when { changeRequest target: 'main' }     // PR target vào main
when { changeRequest author: 'alice' }    // PR của alice
when { changeRequest branch: 'feature/*' }

// Biến môi trường
when { environment name: 'DEPLOY_ENV', value: 'production' }

// Biểu thức Groovy
when {
    expression {
        return params.RUN_DEPLOY && env.BRANCH_NAME == 'main'
    }
}

// File thay đổi (cần built-in changeset support)
when { changeset 'src/**/*.java' }
when { changeset glob: 'pom.xml' }

// Build trigger (cần trigger detection)
when { triggeredBy 'TimerTrigger' }          // Chạy bởi cron schedule
when { triggeredBy 'UserIdCause' }           // Chạy thủ công bởi user
when { triggeredBy cause: 'UserIdCause', detail: 'alice' }

// Upstream — được trigger bởi job khác
when { upstream(upstreamProjects: 'other-job', threshold: hudson.model.Result.SUCCESS) }

// Không bao giờ chạy (dùng để tạm thời disable stage)
when { not { expression { return true } } }
```

### Kết Hợp Điều Kiện

```groovy
// allOf — TẤT CẢ điều kiện phải đúng (AND)
when {
    allOf {
        branch 'main'
        environment name: 'DEPLOY_READY', value: 'true'
        not { changeRequest() }    // Không phải PR
    }
}

// anyOf — ÍT NHẤT MỘT điều kiện đúng (OR)
when {
    anyOf {
        branch 'main'
        branch 'release/*'
        tag 'v*'
    }
}

// not — PHỦ ĐỊNH
when {
    not {
        anyOf {
            branch 'develop'
            changeRequest()
        }
    }
}
```

### `beforeAgent` và `beforeInput`

```groovy
stage('Deploy') {
    when {
        beforeAgent true    // Đánh giá when TRƯỚC khi allocate agent
                            // Tiết kiệm tài nguyên khi điều kiện không thỏa
        branch 'main'
    }
    agent { label 'expensive-deploy-agent' }
    steps { sh './deploy.sh' }
}

stage('Approve') {
    when {
        beforeInput true    // Đánh giá when TRƯỚC khi hiện input prompt
        branch 'main'
    }
    input { message 'Deploy?' }
    steps { sh './deploy.sh' }
}
```

---

## Các Loại Credentials Step

### `withCredentials` — Sử Dụng Credentials Trong Steps

```groovy
withCredentials([
    // Secret text
    string(credentialsId: 'api-token', variable: 'API_TOKEN'),

    // Username và Password
    usernamePassword(
        credentialsId:   'db-creds',
        usernameVariable: 'DB_USER',
        passwordVariable: 'DB_PASS'
    ),

    // SSH Private Key
    sshUserPrivateKey(
        credentialsId:   'deploy-key',
        keyFileVariable: 'SSH_KEY_FILE',
        passphraseVariable: 'SSH_PASSPHRASE',
        usernameVariable: 'SSH_USER'
    ),

    // File (certificate, kubeconfig, ...)
    file(credentialsId: 'ssl-cert',       variable: 'CERT_FILE'),
    file(credentialsId: 'kubeconfig',     variable: 'KUBECONFIG'),

    // Certificate (PEM)
    certificate(
        credentialsId:   'p12-cert',
        keystoreVariable: 'KEYSTORE',
        passwordVariable: 'KEYSTORE_PASS'
    ),

    // Docker Registry (cần Docker plugin)
    dockerRegistryEndpoint(
        credentialsId:   'registry-creds',
        registryAddress: 'https://registry.example.com'
    )
]) {
    // Credentials khả dụng trong block này
    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com'
    sh "psql -U $DB_USER -h db.example.com -d mydb"
    sh "ssh -i $SSH_KEY_FILE $SSH_USER@server.example.com 'sudo systemctl restart myapp'"
    sh 'kubectl apply -f k8s/'    // KUBECONFIG đã được set tự động
}
```

### `withEnv` — Ghi Đè Biến Môi Trường Tạm Thời

```groovy
withEnv(['JAVA_HOME=/usr/lib/jvm/java-17', 'PATH+JAVA=${JAVA_HOME}/bin']) {
    sh 'java -version'
}

// Ghi đè nhiều biến
withEnv([
    "APP_ENV=production",
    "DB_HOST=db.production.example.com",
    "LOG_LEVEL=warn"
]) {
    sh './start-app.sh'
}
```

---

## Script Block — Sử Dụng Groovy

Trong Declarative Pipeline, dùng `script { }` để chạy Groovy code:

```groovy
steps {
    script {
        // Groovy code đầy đủ
        def version = sh(script: 'cat VERSION', returnStdout: true).trim()
        def parts   = version.tokenize('.')
        def major   = parts[0].toInteger()
        def minor   = parts[1].toInteger()

        if (major >= 2) {
            echo "Major version: ${major} — tính năng mới"
        }

        // Thao tác với biến môi trường
        env.NEW_VERSION = "${major}.${minor + 1}.0"
        env.IMAGE_TAG   = "myapp:${env.NEW_VERSION}"

        // Groovy collection operations
        def services = ['auth', 'payment', 'notification']
        def filtered = services.findAll { it != 'notification' }
        def tags     = filtered.collect { "${it}:${version}" }
        echo "Images to build: ${tags.join(', ')}"

        // Map operations
        def config = [env: 'production', region: 'us-east-1']
        config.each { k, v -> echo "${k} = ${v}" }

        // Regular expressions
        def semver = ~/\d+\.\d+\.\d+/
        if (version =~ semver) {
            echo "Version hợp lệ: ${version}"
        }
    }
}
```

---

## Pipeline Snippet Generator

Jenkins cung cấp công cụ tạo cú pháp pipeline tự động tại:

```
https://your-jenkins.example.com/pipeline-syntax/
```

- **Snippet Generator**: Chọn step, điền tham số → nhận cú pháp Groovy
- **Declarative Directive Generator**: Tương tự nhưng cho directive (agent, options, ...)
- **Global Variable Reference**: Tham chiếu tất cả biến global có sẵn

### Cách Dùng Snippet Generator

1. Truy cập `https://jenkins.example.com/pipeline-syntax/`
2. Chọn step trong dropdown (ví dụ: `withCredentials`)
3. Điền các tham số
4. Bấm **"Generate Pipeline Script"**
5. Copy đoạn code tạo ra vào Jenkinsfile

---

## Groovy String Cheatsheet

Hay bị nhầm lẫn giữa single quotes và double quotes trong Groovy:

```groovy
// Double quotes "" — String interpolation (nội suy biến)
def name = "World"
echo "Hello, ${name}!"          // → Hello, World!
sh  "echo Hello, ${name}"       // Shell nhận: echo Hello, World

// Single quotes '' — String literal (không nội suy)
echo 'Hello, ${name}!'          // → Hello, ${name}!  (in nguyên)
sh  'echo Hello, $USER'         // Shell nhận: echo Hello, $USER → Shell tự mở rộng $USER

// Triple quotes — Multiline
def script = """
    echo "Name: ${name}"        // Nội suy biến Groovy
    echo "User: $USER"          // Shell mở rộng
"""
sh script

def literal = '''
    echo 'Hello'                // Không nội suy
    echo $HOME                  // Shell mở rộng $HOME
'''
sh literal

// GString — Chú ý khi dùng với sh
def url = "https://example.com"
sh "curl ${url}"                // ✅ Groovy nội suy URL trước, truyền vào shell

// Khi cần dùng $ của shell bên trong double quotes
sh "echo \$HOME"                // ✅ Escape $ để shell tự mở rộng
sh 'echo $HOME'                 // ✅ Hoặc dùng single quotes
```

### Các Phương Thức String Hay Dùng

```groovy
def s = "  Hello, World!  "

s.trim()                    // "Hello, World!"
s.toLowerCase()             // "  hello, world!  "
s.toUpperCase()             // "  HELLO, WORLD!  "
s.replace('World', 'Groovy') // "  Hello, Groovy!  "
s.contains('World')         // true
s.startsWith('Hello')       // false (có khoảng trắng đầu)
s.trim().startsWith('Hello') // true
s.split(', ')               // ["  Hello", "World!  "]
"1.2.3".tokenize('.')       // ["1", "2", "3"] — tiện hơn split
"123".toInteger()           // 123
123.toString()              // "123"

// String matching
"v1.2.3" ==~ /v\d+\.\d+\.\d+/      // true — regex match toàn bộ
"build v1.2.3 done" =~ /v\d+/       // Matcher object — match một phần
("build v1.2.3 done" =~ /v(\d+\.\d+\.\d+)/)[0][1]  // "1.2.3"
```

---

**Tiếp Theo:** Đọc [5-multibranch-pipeline.md](5-multibranch-pipeline.md) để hiểu cách tổ chức pipeline cho dự án nhiều nhánh và Pull Request.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
