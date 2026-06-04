# Build Parameters — Tham Số Hóa Build Jenkins

> **Build Parameters** (tham số build) cho phép người dùng hoặc hệ thống truyền thông tin vào job khi kích hoạt build. Thay vì job luôn làm một việc cố định, bạn có thể điều chỉnh hành vi mỗi lần chạy — chọn môi trường deploy, phiên bản ứng dụng, hay bật tắt tính năng — mà không cần sửa Jenkinsfile.

---

## Mục Lục

1. [Tổng Quan Build Parameters](#tổng-quan-build-parameters)
2. [Các Loại Parameter](#các-loại-parameter)
3. [Khai Báo Parameters trong Jenkinsfile](#khai-báo-parameters-trong-jenkinsfile)
4. [Sử Dụng Parameters Trong Pipeline](#sử-dụng-parameters-trong-pipeline)
5. [Parameters Nâng Cao](#parameters-nâng-cao)
6. [Parameterized Trigger — Kích Hoạt Job Con Với Tham Số](#parameterized-trigger--kích-hoạt-job-con-với-tham-số)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Build Parameters

### Khi Nào Cần Dùng Parameters

```
✅ Dùng parameters khi:
   - Deploy lên môi trường khác nhau (dev/staging/production)
   - Chọn phiên bản ứng dụng hoặc Docker tag cụ thể
   - Bật/tắt tính năng kiểm thử (skip integration test, run smoke only)
   - Truyền thông tin từ upstream job sang downstream job
   - Trigger build thủ công với cấu hình tùy chọn

❌ Không nên dùng parameters khi:
   - Cố định mọi thứ trong code — dùng environment variable hoặc config file
   - Parameters quá nhiều → Jenkinsfile trở nên phức tạp, khó maintain
```

### Luồng Hoạt Động

```
Người dùng / API / Upstream Job
           │
           │  Truyền parameters
           ▼
  Jenkins "Build with Parameters"
           │
           │  env.PARAM_NAME = "value"
           ▼
      Jenkinsfile sử dụng
      ${params.PARAM_NAME}
```

---

## Các Loại Parameter

### 1. String Parameter — Tham Số Chuỗi

Nhập chuỗi văn bản tự do.

```groovy
parameters {
    string(
        name: 'VERSION',
        defaultValue: 'latest',
        description: 'Docker image tag hoặc version để deploy (ví dụ: 1.2.3, latest, v2.0.0-rc1)'
    )
}
```

**UI hiển thị:** Text input field

**Sử dụng:**

```groovy
sh "docker pull myapp:${params.VERSION}"
sh "kubectl set image deployment/myapp myapp=myapp:${params.VERSION}"
```

---

### 2. Choice Parameter — Tham Số Lựa Chọn

Dropdown list với các lựa chọn định sẵn. Ngăn người dùng nhập giá trị không hợp lệ.

```groovy
parameters {
    choice(
        name: 'ENVIRONMENT',
        choices: ['dev', 'staging', 'production'],
        description: 'Môi trường triển khai ứng dụng'
    )
}
```

**UI hiển thị:** Select dropdown

**Lưu ý:** Giá trị đầu tiên trong list là **giá trị mặc định** (default value).

**Sử dụng:**

```groovy
stage('Deploy') {
    steps {
        sh "./deploy.sh ${params.ENVIRONMENT}"
    }
}

// Hoặc dùng switch/case
stage('Deploy') {
    steps {
        script {
            switch(params.ENVIRONMENT) {
                case 'production':
                    sh './scripts/deploy-prod.sh'
                    break
                case 'staging':
                    sh './scripts/deploy-staging.sh'
                    break
                default:
                    sh './scripts/deploy-dev.sh'
            }
        }
    }
}
```

---

### 3. Boolean Parameter — Tham Số Đúng/Sai

Checkbox — `true` hoặc `false`.

```groovy
parameters {
    booleanParam(
        name: 'SKIP_TESTS',
        defaultValue: false,
        description: 'Bỏ qua bước chạy unit test (chỉ dùng khi deploy khẩn cấp)'
    )
    booleanParam(
        name: 'DRY_RUN',
        defaultValue: true,
        description: 'Chạy thử — hiển thị lệnh sẽ thực thi mà không thực sự thay đổi gì'
    )
}
```

**UI hiển thị:** Checkbox

**Sử dụng:**

```groovy
stage('Test') {
    when {
        // Chỉ chạy khi SKIP_TESTS là false
        expression { return !params.SKIP_TESTS }
    }
    steps {
        sh 'mvn test'
    }
}

stage('Deploy') {
    steps {
        script {
            if (params.DRY_RUN) {
                echo "[DRY RUN] Sẽ thực thi: ./deploy.sh ${params.ENVIRONMENT}"
                echo "[DRY RUN] Không có thay đổi thực sự nào được thực hiện."
            } else {
                sh "./deploy.sh ${params.ENVIRONMENT}"
            }
        }
    }
}
```

---

### 4. Password Parameter — Tham Số Mật Khẩu

Giống String nhưng **giá trị bị che** (masked) trong UI và log.

```groovy
parameters {
    password(
        name: 'DEPLOY_TOKEN',
        defaultValue: '',
        description: 'Token xác thực để deploy (sẽ được che trong log)'
    )
}
```

**Quan trọng:** Không nên dùng Password Parameter cho secret production — hãy dùng Jenkins Credentials Store thay thế. Password Parameter chỉ phù hợp cho thao tác ad-hoc (tạm thời) hoặc demo.

**Sử dụng:**

```groovy
withCredentials([string(credentialsId: 'deploy-token', variable: 'DEPLOY_TOKEN')]) {
    // Cách đúng: dùng Credentials Store
    sh "curl -H 'Authorization: Bearer ${DEPLOY_TOKEN}' https://api.example.com/deploy"
}
```

---

### 5. File Parameter — Tham Số File

Cho phép người dùng **upload file** khi trigger build thủ công.

```groovy
parameters {
    file(
        name: 'CONFIG_FILE',
        description: 'File cấu hình JSON để ghi đè mặc định (config.json)'
    )
}
```

**Sử dụng:**

```groovy
stage('Deploy with Custom Config') {
    steps {
        // File được đặt trong workspace với tên tham số
        sh "cp ${params.CONFIG_FILE} ./config/app-config.json"
        sh "./deploy.sh"
    }
}
```

**Hạn chế:** Chỉ hoạt động với build thủ công qua UI — không dùng được với API trigger hay Webhook trigger.

---

### 6. Text Parameter — Tham Số Văn Bản Nhiều Dòng

Textarea cho phép nhập nội dung nhiều dòng.

```groovy
parameters {
    text(
        name: 'RELEASE_NOTES',
        defaultValue: '',
        description: 'Ghi chú phát hành (release notes) cho phiên bản này'
    )
}
```

**Sử dụng:**

```groovy
stage('Tag Release') {
    steps {
        sh """
            git tag -a v${params.VERSION} -m "${params.RELEASE_NOTES}"
            git push origin v${params.VERSION}
        """
    }
}
```

---

## Khai Báo Parameters trong Jenkinsfile

### Declarative Pipeline

```groovy
pipeline {
    agent any

    parameters {
        string(
            name: 'VERSION',
            defaultValue: 'latest',
            description: 'Version để deploy'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'production'],
            description: 'Môi trường triển khai'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Bỏ qua unit test'
        )
        booleanParam(
            name: 'NOTIFY_SLACK',
            defaultValue: true,
            description: 'Gửi thông báo lên Slack sau deploy'
        )
    }

    stages {
        stage('Build') {
            steps {
                echo "Building version: ${params.VERSION}"
                sh "make build VERSION=${params.VERSION}"
            }
        }

        stage('Test') {
            when {
                expression { !params.SKIP_TESTS }
            }
            steps {
                sh 'make test'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying to: ${params.ENVIRONMENT}"
                sh "./deploy.sh ${params.ENVIRONMENT} ${params.VERSION}"
            }
        }
    }

    post {
        success {
            script {
                if (params.NOTIFY_SLACK) {
                    slackSend(
                        channel: '#deployments',
                        message: "✅ Deploy ${params.VERSION} → ${params.ENVIRONMENT} thành công!"
                    )
                }
            }
        }
    }
}
```

### Scripted Pipeline

```groovy
// Khai báo parameters trong properties() — chạy một lần để đăng ký
properties([
    parameters([
        string(name: 'VERSION', defaultValue: 'latest', description: 'Version'),
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Môi trường'),
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests')
    ])
])

node {
    stage('Build') {
        echo "Version: ${params.VERSION}"
        sh "make build VERSION=${params.VERSION}"
    }

    if (!params.SKIP_TESTS) {
        stage('Test') {
            sh 'make test'
        }
    }

    stage('Deploy') {
        sh "./deploy.sh ${params.ENVIRONMENT} ${params.VERSION}"
    }
}
```

> **Lưu ý quan trọng:** Lần đầu chạy Declarative Pipeline có `parameters {}` block, Jenkins sẽ **không hiển thị UI nhập tham số** ngay lập tức — cần chạy một lần để Jenkins đăng ký tham số, sau đó lần chạy thứ hai mới có "Build with Parameters" trong menu.

---

## Sử Dụng Parameters Trong Pipeline

### Truy Cập Giá Trị Parameter

```groovy
// Dùng params.PARAM_NAME (khuyến nghị)
echo "Environment: ${params.ENVIRONMENT}"

// Dùng như env variable
echo "Environment: ${env.ENVIRONMENT}"

// Trong shell script (bash)
sh "echo Deploying to ${params.ENVIRONMENT}"

// Trong shell script với biến (tránh injection)
sh '''
    ENV="$ENVIRONMENT"
    echo "Deploying to $ENV"
'''
```

### Validate (Kiểm Tra Hợp Lệ) Parameters

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'VERSION', defaultValue: '', description: 'Phiên bản cần deploy')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Môi trường')
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    // Kiểm tra version không được để trống
                    if (!params.VERSION?.trim()) {
                        error("❌ Parameter VERSION không được để trống!")
                    }

                    // Kiểm tra version đúng định dạng semantic versioning
                    if (!params.VERSION.matches(/^\d+\.\d+\.\d+$/)) {
                        error("❌ VERSION phải theo định dạng x.y.z (ví dụ: 1.2.3)")
                    }

                    // Bảo vệ production: yêu cầu xác nhận thủ công
                    if (params.ENVIRONMENT == 'production') {
                        def confirm = input(
                            message: "⚠️ Bạn đang deploy lên PRODUCTION. Xác nhận?",
                            parameters: [
                                booleanParam(name: 'CONFIRMED', defaultValue: false, description: 'Tôi xác nhận')
                            ]
                        )
                        if (!confirm) {
                            error("Deploy production bị hủy.")
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh "./deploy.sh ${params.ENVIRONMENT} ${params.VERSION}"
            }
        }
    }
}
```

---

## Parameters Nâng Cao

### Active Choices Parameter — Tham Số Động

Plugin **Active Choices** (tham số động) cho phép danh sách lựa chọn thay đổi dựa trên giá trị tham số khác:

```groovy
// Cài plugin: "Active Choices Plugin" trước khi dùng
properties([
    parameters([
        // Tham số 1: Chọn môi trường
        [$class: 'ChoiceParameter',
            choiceType: 'PT_SINGLE_SELECT',
            name: 'ENVIRONMENT',
            script: [$class: 'GroovyScript',
                script: [classpath: [], sandbox: false,
                    script: "return ['dev', 'staging', 'production']"
                ]
            ],
            description: 'Môi trường deploy'
        ],

        // Tham số 2: Danh sách service tự động cập nhật theo ENVIRONMENT
        [$class: 'CascadeChoiceParameter',
            choiceType: 'PT_CHECKBOX',
            name: 'SERVICES',
            referencedParameters: 'ENVIRONMENT',
            script: [$class: 'GroovyScript',
                script: [classpath: [], sandbox: false,
                    script: '''
                        if (ENVIRONMENT == "production") {
                            return ["api-service:selected", "auth-service:selected", "notification-service"]
                        } else {
                            return ["api-service:selected", "auth-service", "notification-service", "debug-tools:selected"]
                        }
                    '''
                ]
            ],
            description: 'Services cần deploy'
        ]
    ])
])
```

### Build với Parameters qua API

Trigger build với parameters qua REST API:

```bash
# POST với form data
curl -X POST \
  "http://jenkins.example.com/job/my-job/buildWithParameters" \
  --user "admin:api-token" \
  --data "ENVIRONMENT=staging&VERSION=1.2.3&SKIP_TESTS=false"

# POST với JSON body (Generic Webhook Trigger)
curl -X POST \
  "http://jenkins.example.com/generic-webhook-trigger/invoke?token=MY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"environment": "staging", "version": "1.2.3"}'
```

---

## Parameterized Trigger — Kích Hoạt Job Con Với Tham Số

Truyền parameters từ pipeline hiện tại sang job khác:

### Dùng build() Step

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'VERSION', defaultValue: 'latest', description: 'Version')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Môi trường')
    }

    stages {
        stage('Build Image') {
            steps {
                sh "docker build -t myapp:${params.VERSION} ."
                sh "docker push myapp:${params.VERSION}"
            }
        }

        stage('Trigger Deploy Job') {
            steps {
                // Kích hoạt job deploy riêng với parameters từ job này
                build(
                    job: 'deploy-pipeline',
                    parameters: [
                        string(name: 'IMAGE_TAG', value: params.VERSION),
                        string(name: 'TARGET_ENV', value: params.ENVIRONMENT),
                        booleanParam(name: 'NOTIFY_SLACK', value: true)
                    ],
                    wait: true,           // Chờ deploy job hoàn thành
                    propagate: true       // Nếu deploy job fail → job này cũng fail
                )
            }
        }
    }
}
```

### Kích Hoạt Song Song Nhiều Môi Trường

```groovy
stage('Deploy to All Environments') {
    parallel {
        stage('Deploy Dev') {
            steps {
                build(
                    job: 'deploy-pipeline',
                    parameters: [
                        string(name: 'ENVIRONMENT', value: 'dev'),
                        string(name: 'VERSION', value: params.VERSION)
                    ],
                    wait: false    // Không chờ — chạy song song
                )
            }
        }
        stage('Deploy Staging') {
            steps {
                build(
                    job: 'deploy-pipeline',
                    parameters: [
                        string(name: 'ENVIRONMENT', value: 'staging'),
                        string(name: 'VERSION', value: params.VERSION)
                    ],
                    wait: false
                )
            }
        }
    }
}
```

---

## Best Practices

### 1. Đặt Tên Rõ Ràng và Mô Tả Đầy Đủ

```groovy
// ❌ Xấu — tên mơ hồ, không mô tả
parameters {
    string(name: 'v', defaultValue: '', description: '')
    booleanParam(name: 'flag', defaultValue: false, description: '')
}

// ✅ Tốt — tên rõ ràng, mô tả hữu ích
parameters {
    string(
        name: 'APP_VERSION',
        defaultValue: 'latest',
        description: 'Docker image tag (ví dụ: 1.2.3, latest, v2.0.0-rc1). Xem docker.hub.example.com/myapp'
    )
    booleanParam(
        name: 'SKIP_INTEGRATION_TESTS',
        defaultValue: false,
        description: 'Bỏ qua integration tests. Chỉ dùng khi hotfix khẩn cấp và đã test thủ công.'
    )
}
```

### 2. Giá Trị Mặc Định An Toàn

```groovy
parameters {
    // Mặc định là môi trường ít nguy hiểm nhất
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'])
    //                                      ^^^  ← Mặc định là dev, không phải production

    // Mặc định bảo thủ (conservative default) — thà chậm hơn là sai
    booleanParam(name: 'SKIP_TESTS', defaultValue: false)  // Mặc định: CHẠY test

    // Mặc định dry run — xem trước khi thực hiện
    booleanParam(name: 'DRY_RUN', defaultValue: true)     // Mặc định: chỉ xem
}
```

### 3. Bảo Vệ Production Bằng Input Step

```groovy
stage('Production Gate') {
    when {
        expression { params.ENVIRONMENT == 'production' }
    }
    steps {
        // Dừng pipeline, yêu cầu người có quyền xác nhận thủ công
        input(
            message: "⚠️ Triển khai phiên bản ${params.VERSION} lên PRODUCTION?",
            ok: 'Xác nhận Deploy',
            submitter: 'admin,ops-team',    // Chỉ những user này mới có thể xác nhận
            submitterParameter: 'APPROVED_BY'
        )
    }
}
```

### 4. Tránh Command Injection (Tiêm Lệnh Độc Hại)

```groovy
// ❌ Nguy hiểm — người dùng có thể nhập: "; rm -rf /"
sh "echo ${params.USER_INPUT}"

// ✅ An toàn — dùng biến môi trường qua env, không nội suy trực tiếp
sh 'echo "$USER_INPUT"'

// ✅ An toàn — validate trước
script {
    if (!params.VERSION.matches(/^[\w\.\-]+$/)) {
        error("VERSION chứa ký tự không hợp lệ!")
    }
}
sh "docker pull myapp:${params.VERSION}"
```

### 5. Không Dùng Parameters Cho Secret

```groovy
// ❌ Sai — secret hiển thị trong build history
parameters {
    password(name: 'DB_PASSWORD', defaultValue: '', description: 'Mật khẩu database')
}

// ✅ Đúng — dùng Jenkins Credentials Store
stage('Deploy') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'database-credentials',
                usernameVariable: 'DB_USER',
                passwordVariable: 'DB_PASSWORD'
            )
        ]) {
            sh './deploy.sh'
        }
    }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Các loại Build Parameter nào Jenkins hỗ trợ và khi nào dùng từng loại?**

> Jenkins hỗ trợ: **String** — văn bản tự do (version number, branch name); **Choice** — dropdown cố định (môi trường deploy, region); **Boolean** — checkbox true/false (skip test, dry run, enable feature); **Password** — text bị che (nhưng nên dùng Credentials Store thay thế); **File** — upload file khi trigger thủ công; **Text** — textarea nhiều dòng (release notes). Nguyên tắc: dùng Choice khi có tập giá trị hữu hạn — ngăn lỗi nhập sai; dùng String khi giá trị không biết trước; dùng Boolean cho cờ bật/tắt.

**Q: Làm sao bảo vệ môi trường production khỏi bị deploy nhầm qua Parameters?**

> Nhiều lớp bảo vệ: **(1) Choice Parameter** — giữ production ở cuối list hoặc loại hẳn khỏi các pipeline CI thông thường; **(2) `input` step** — dừng pipeline, yêu cầu người dùng có quyền xác nhận thủ công với `submitter: 'ops-team'`; **(3) `when` condition** — kiểm tra môi trường trước mỗi bước nguy hiểm; **(4) Separate job** — tạo job deploy-production riêng với quyền truy cập hạn chế, không phải cùng job với dev/staging.

**Q: Parameters được truy cập như thế nào trong pipeline và có khác với environment variable không?**

> Trong Declarative Pipeline, parameters truy cập qua `params.PARAM_NAME` (object `params` được Jenkins inject). Khác environment variable ở chỗ: `params` chỉ chứa giá trị người dùng truyền vào khi trigger; `env` chứa tất cả biến môi trường bao gồm built-in Jenkins variables (`BUILD_NUMBER`, `JOB_NAME`...) và giá trị từ `environment {}` block. Parameters cũng tự động được đặt vào `env`, nên `env.PARAM_NAME` cũng hoạt động, nhưng `params.PARAM_NAME` rõ ràng hơn và nên ưu tiên dùng.

**Q: Tại sao không nên dùng Password Parameter cho secret production?**

> Password Parameter che giá trị trong UI nhưng vẫn lưu trong build history (không được mã hóa), hiển thị trong Build Parameters log, và truyền qua URL/form data có thể bị log ở proxy/load balancer. Jenkins Credentials Store lưu secret được mã hóa, tích hợp với audit log, hỗ trợ rotation, và che giá trị trong console log với `****`. Password Parameter chỉ phù hợp cho demo hoặc thao tác one-off không nhạy cảm.

---

**Liên Kết Liên Quan:**
- [1-webhooks.md](1-webhooks.md) — Kích hoạt build tự động từ Git events
- [4-build-artifacts.md](4-build-artifacts.md) — Lưu trữ và chia sẻ kết quả build
- `06-security/credentials.md` — Quản lý secret an toàn

**Cập Nhật:** 2026-05-10
