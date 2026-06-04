# 1 — Cấu Trúc Thư Mục Shared Library

> Một Shared Library (thư viện dùng chung) trong Jenkins có cấu trúc thư mục quy ước rõ ràng. Hiểu đúng vai trò của từng thư mục là bước đầu tiên để xây dựng thư viện đúng cách và bảo trì lâu dài.

---

## Mục Tiêu

- Nắm cấu trúc thư mục chuẩn: `vars/`, `src/`, `resources/`
- Hiểu vai trò và quy tắc đặt tên cho từng thư mục
- Tạo và đăng ký Shared Library vào Jenkins
- Biết cách nạp (load) library trong Jenkinsfile

---

## Cấu Trúc Thư Mục Chuẩn

```
jenkins-shared-library/          ← Root của Git repository
│
├── vars/                        ← Global Variables (step tùy chỉnh)
│   ├── buildDockerImage.groovy  ← Tạo step buildDockerImage()
│   ├── deployToKubernetes.groovy
│   ├── runSonarQube.groovy
│   ├── sendSlackNotification.groovy
│   └── buildDockerImage.txt     ← (tùy chọn) tài liệu cho step
│
├── src/                         ← Groovy Classes (lớp Groovy)
│   └── com/
│       └── company/
│           └── jenkins/
│               ├── DockerHelper.groovy
│               ├── KubernetesHelper.groovy
│               └── Utils.groovy
│
├── resources/                   ← Static Resources (tài nguyên tĩnh)
│   └── com/
│       └── company/
│           └── jenkins/
│               ├── deploy-template.yaml
│               └── health-check.sh
│
├── test/                        ← Unit tests (kiểm thử đơn vị)
│   └── groovy/
│       └── com/company/jenkins/
│           ├── BuildDockerImageTest.groovy
│           └── KubernetesHelperTest.groovy
│
├── Jenkinsfile                  ← Pipeline test cho chính thư viện
├── build.gradle                 ← Build script cho JenkinsPipelineUnit
└── README.md                    ← Tài liệu sử dụng thư viện
```

---

## Chi Tiết Từng Thư Mục

### `vars/` — Global Variables (Biến Toàn Cục)

**Mục đích:** Định nghĩa các step (bước) tùy chỉnh có thể gọi trực tiếp trong bất kỳ Jenkinsfile nào.

**Quy tắc quan trọng:**
- Tên file = tên step khi gọi (camelCase — kiểu lạc đà)
- Mỗi file cần có hàm `call(...)` — đây là hàm được gọi khi dùng step
- File `.txt` cùng tên sẽ được hiển thị trong mục **Global Variables Reference** của Jenkins

**Ví dụ file trong `vars/`:**

```groovy
// vars/buildDockerImage.groovy
def call(Map config = [:]) {
    // config.imageName  — tên image
    // config.tag        — tag version
    // config.registry   — địa chỉ registry
    def imageName = config.imageName ?: env.JOB_NAME.toLowerCase()
    def tag       = config.tag       ?: env.BUILD_NUMBER
    def registry  = config.registry  ?: 'docker.io'

    echo "Building Docker image: ${registry}/${imageName}:${tag}"
    sh "docker build -t ${registry}/${imageName}:${tag} ."
    sh "docker push ${registry}/${imageName}:${tag}"
}
```

**File tài liệu tương ứng:**

```text
// vars/buildDockerImage.txt
Build và push Docker image lên registry.

Parameters:
  imageName  (String, optional) — Tên image. Mặc định: tên job
  tag        (String, optional) — Tag version. Mặc định: BUILD_NUMBER
  registry   (String, optional) — Địa chỉ registry. Mặc định: docker.io

Example:
  buildDockerImage(
      imageName: 'my-service',
      tag:       env.GIT_COMMIT[0..7],
      registry:  'registry.company.com'
  )
```

---

### `src/` — Groovy Classes (Lớp Groovy)

**Mục đích:** Tổ chức logic phức tạp theo kiểu hướng đối tượng (OOP — Object-Oriented Programming).

**Quy tắc quan trọng:**
- Cấu trúc package (gói) theo quy ước Java: `src/com/company/jenkins/ClassName.groovy`
- Dòng đầu file phải khai báo `package com.company.jenkins`
- Class phải `implement Serializable` để Jenkins có thể lưu trạng thái pipeline (checkpoint)
- Không dùng trực tiếp `sh`, `echo` — phải truyền đối tượng `steps` vào

**Ví dụ Groovy class:**

```groovy
// src/com/company/jenkins/DockerHelper.groovy
package com.company.jenkins

class DockerHelper implements Serializable {

    private def steps      // Tham chiếu đến pipeline steps
    private String registry

    DockerHelper(def steps, String registry = 'docker.io') {
        this.steps    = steps
        this.registry = registry
    }

    /**
     * Build Docker image từ Dockerfile trong thư mục hiện tại.
     * @param imageName  Tên image (không kèm registry)
     * @param tag        Tag của image
     */
    void buildImage(String imageName, String tag) {
        def fullName = "${registry}/${imageName}:${tag}"
        steps.echo "Building: ${fullName}"
        steps.sh   "docker build -t ${fullName} ."
    }

    /**
     * Push image lên registry sau khi build.
     */
    void pushImage(String imageName, String tag) {
        def fullName = "${registry}/${imageName}:${tag}"
        steps.sh "docker push ${fullName}"
    }

    /**
     * Build và push trong một lần gọi.
     */
    void buildAndPush(String imageName, String tag) {
        buildImage(imageName, tag)
        pushImage(imageName, tag)
    }
}
```

**Cách dùng class từ Jenkinsfile:**

```groovy
@Library('company-jenkins-lib') _
import com.company.jenkins.DockerHelper

pipeline {
    agent any
    stages {
        stage('Build & Push') {
            steps {
                script {
                    def docker = new DockerHelper(this, 'registry.company.com')
                    docker.buildAndPush('my-service', env.BUILD_NUMBER)
                }
            }
        }
    }
}
```

---

### `resources/` — Static Resources (Tài Nguyên Tĩnh)

**Mục đích:** Lưu các file không phải code Groovy: YAML template, shell script, config, JSON.

**Quy tắc quan trọng:**
- Cấu trúc thư mục tùy ý, nhưng nên theo package để tránh xung đột tên
- Truy cập bằng hàm `libraryResource('path/to/file')` — trả về nội dung file dạng String

**Ví dụ sử dụng resources:**

```groovy
// resources/com/company/jenkins/k8s-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: __APP_NAME__
  namespace: __NAMESPACE__
spec:
  replicas: __REPLICAS__
  selector:
    matchLabels:
      app: __APP_NAME__
  template:
    spec:
      containers:
        - name: __APP_NAME__
          image: __IMAGE__
```

```groovy
// vars/deployToKubernetes.groovy — dùng template từ resources
def call(Map config) {
    // Nạp template YAML từ resources/
    def template = libraryResource('com/company/jenkins/k8s-deploy.yaml')

    // Thay thế placeholder bằng giá trị thực
    def manifest = template
        .replace('__APP_NAME__',  config.appName)
        .replace('__NAMESPACE__', config.namespace ?: 'default')
        .replace('__REPLICAS__',  (config.replicas ?: 1).toString())
        .replace('__IMAGE__',     config.image)

    writeFile file: 'deploy.yaml', text: manifest
    sh 'kubectl apply -f deploy.yaml'
}
```

---

## Tạo Repository Shared Library

### Bước 1: Khởi tạo Git repository

```bash
# Tạo thư mục và khởi tạo Git
mkdir company-jenkins-lib
cd company-jenkins-lib
git init

# Tạo cấu trúc thư mục
mkdir -p vars src/com/company/jenkins resources/com/company/jenkins test/groovy

# Tạo file đầu tiên
touch vars/helloWorld.groovy
touch README.md
```

### Bước 2: Viết step đầu tiên để kiểm tra

```groovy
// vars/helloWorld.groovy
def call(String name = 'World') {
    echo "Hello, ${name}! This is your first Shared Library step."
}
```

### Bước 3: Đẩy lên Git remote

```bash
git add .
git commit -m "feat: initial shared library structure"
git remote add origin git@github.com:company/company-jenkins-lib.git
git push -u origin main
```

---

## Đăng Ký Library Vào Jenkins

### Cách 1: Global Library qua giao diện web

**Đường dẫn:** `Manage Jenkins → System → Global Pipeline Libraries`

| Trường | Giá Trị Ví Dụ | Mô Tả |
|--------|---------------|-------|
| Name | `company-jenkins-lib` | Tên dùng trong `@Library(...)` |
| Default version | `main` | Branch/tag mặc định nếu không chỉ định |
| Allow default version to be overridden | ✅ | Cho phép Jenkinsfile chỉ định version khác |
| Include @Library changes in job's changelog | ✅ | Hiện thay đổi library trong build log |
| Retrieval method | Modern SCM | Dùng Git plugin hiện đại |
| Source Code Management | Git | Chọn Git |
| Project Repository | `git@github.com:company/jenkins-lib.git` | URL repository |
| Credentials | `github-ssh-key` | Credentials để clone repo |

### Cách 2: Global Library qua JCasC (Jenkins Configuration as Code — Cấu Hình Jenkins Dưới Dạng Code)

```yaml
# jenkins.yaml — dùng với JCasC plugin
unclassified:
  globalLibraries:
    libraries:
      - name: "company-jenkins-lib"
        defaultVersion: "main"
        allowVersionOverride: true
        includeInChangesets: true
        retriever:
          modernSCM:
            scm:
              git:
                remote: "git@github.com:company/jenkins-lib.git"
                credentialsId: "github-ssh-key"
```

### Cách 3: Folder-level Library (Thư Viện Cấp Thư Mục)

Vào Folder → **Configure** → **Pipeline Libraries** → Thêm library (tương tự Global Library).

Phù hợp khi mỗi team có thư viện riêng, độc lập với nhau.

---

## Nạp Library Trong Jenkinsfile

### Cú pháp `@Library`

```groovy
// Dùng default version (main)
@Library('company-jenkins-lib') _

// Dùng version cố định — khuyến nghị cho production
@Library('company-jenkins-lib@v2.1.0') _

// Dùng branch cụ thể — dành cho phát triển/thử nghiệm
@Library('company-jenkins-lib@feature/new-docker-step') _

// Nạp nhiều library cùng lúc
@Library(['company-jenkins-lib@v2.1.0', 'infra-lib@v1.0.0']) _
```

Dấu `_` sau `@Library(...)` là bắt buộc khi không import class cụ thể — nó "tiêu thụ" annotation để Groovy không hiểu nhầm cú pháp.

### Cú pháp `library()` step (linh hoạt hơn)

```groovy
// Trong Scripted Pipeline hoặc script block — nạp theo điều kiện
pipeline {
    agent any
    stages {
        stage('Setup') {
            steps {
                script {
                    // Nạp library động, có thể dùng biến
                    def libVersion = params.LIB_VERSION ?: 'main'
                    def lib = library("company-jenkins-lib@${libVersion}")
                }
            }
        }
    }
}
```

---

## Implicit Loading (Nạp Ngầm Định)

Khi bật **Load implicitly** trong cấu hình Global Library, mọi pipeline tự động có thể dùng các step trong `vars/` mà không cần `@Library(...)`.

**Ưu điểm:** Tiện lợi, không cần khai báo trong từng Jenkinsfile.
**Nhược điểm:** Khó biết pipeline đang dùng library nào, dễ xung đột tên nếu có nhiều library.

**Khuyến nghị:** Tắt implicit loading, luôn khai báo `@Library` tường minh kèm version cố định.

---

## So Sánh `vars/` vs `src/`

| Tiêu Chí | `vars/` | `src/` |
|----------|---------|--------|
| Mục đích | Pipeline step tùy chỉnh | Logic phức tạp, tái sử dụng nội bộ |
| Cú pháp gọi | `myStep(args)` trực tiếp | `new ClassName(this).method()` |
| Khả năng CPS | Có (mặc định) | Không (phải dùng `@NonCPS`) |
| Tổ chức code | Một file = một step | Package Java, nhiều class |
| Độ phức tạp | Đơn giản | Phức tạp hơn |
| Test | Khó test riêng lẻ | Dễ unit test hơn |
| Dùng khi | Step đơn giản, gọi trực tiếp | Xử lý phức tạp, cần OOP |

**Thực tế:** Hầu hết các step đặt trong `vars/`, logic phức tạp thì tách vào `src/` và `vars/` gọi vào.

---

## Ví Dụ Thực Tế: Cấu Trúc Library Cho Enterprise

```
company-jenkins-lib/
├── vars/
│   ├── buildMaven.groovy          ← Build Java với Maven
│   ├── buildGradle.groovy         ← Build Java với Gradle
│   ├── buildDockerImage.groovy    ← Build Docker image
│   ├── pushDockerImage.groovy     ← Push lên registry
│   ├── deployHelmChart.groovy     ← Deploy qua Helm lên K8s
│   ├── runSonarQube.groovy        ← Quét chất lượng mã
│   ├── sendSlackNotification.groovy ← Gửi thông báo Slack
│   └── ciPipeline.groovy          ← Pipeline chuẩn tích hợp tất cả
│
├── src/com/company/jenkins/
│   ├── DockerHelper.groovy        ← Xử lý Docker operations
│   ├── KubernetesHelper.groovy    ← Xử lý K8s operations
│   ├── GitHelper.groovy           ← Xử lý Git operations
│   ├── NotificationHelper.groovy  ← Xử lý gửi thông báo
│   └── PipelineConfig.groovy      ← Đọc và validate config
│
├── resources/com/company/jenkins/
│   ├── helm-values-template.yaml  ← Template Helm values
│   ├── sonar-project.properties   ← Cấu hình SonarQube mặc định
│   └── scripts/
│       ├── health-check.sh        ← Script kiểm tra health
│       └── cleanup-images.sh      ← Script dọn dẹp Docker images
│
└── test/groovy/com/company/jenkins/
    ├── BuildMavenTest.groovy
    ├── DockerHelperTest.groovy
    └── KubernetesHelperTest.groovy
```

---

## Lỗi Thường Gặp Khi Tạo Library

### Lỗi 1: Sai cấu trúc thư mục

```
# Sai — không có hàm call()
// vars/myStep.groovy
void myFunction() { ... }   ← Jenkins không nhận ra đây là step

# Đúng
// vars/myStep.groovy
def call(Map config) { ... }  ← Phải có hàm call()
```

### Lỗi 2: Class trong src/ không implement Serializable

```groovy
// Sai — Jenkins không thể serialize (lưu trạng thái) class này
class MyHelper {
    def steps
}

// Đúng
class MyHelper implements Serializable {
    def steps
}
```

### Lỗi 3: Dùng this.steps không đúng trong class

```groovy
// Sai — gọi sh trực tiếp từ class (không có context pipeline)
class DockerHelper implements Serializable {
    void build(String name) {
        sh "docker build -t ${name} ."  // ← Lỗi: không biết sh() là gì
    }
}

// Đúng — truyền steps vào constructor
class DockerHelper implements Serializable {
    def steps
    DockerHelper(steps) { this.steps = steps }

    void build(String name) {
        steps.sh "docker build -t ${name} ."  // ← Đúng
    }
}
```

---

## Checklist Tạo Shared Library Mới

```
□ Tạo Git repository riêng cho thư viện
□ Tạo đúng cấu trúc: vars/, src/, resources/, test/
□ Viết README.md với hướng dẫn sử dụng
□ Step trong vars/ có hàm call() đúng cú pháp
□ Class trong src/ implement Serializable
□ Đăng ký library vào Jenkins (Global hoặc Folder level)
□ Cấu hình Credentials để clone repo thư viện
□ Test thư viện bằng pipeline thử nghiệm
□ Tag version đầu tiên (ví dụ: v1.0.0)
□ Thông báo cho team về cách sử dụng
```

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
