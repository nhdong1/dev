# Plugin Mở Rộng Pipeline

> Nhóm plugin chuyên biệt giúp pipeline Jenkins trở nên mạnh mẽ hơn — từ thao tác file YAML/JSON, quản lý artifact, đến cải thiện trực quan hóa và giám sát build.

## Mục Lục

1. [Pipeline Utility Steps Plugin](#pipeline-utility-steps-plugin)
2. [Blue Ocean Plugin — Chi Tiết](#blue-ocean-plugin--chi-tiết)
3. [Pipeline Stage View Plugin](#pipeline-stage-view-plugin)
4. [Build Monitor Plugin](#build-monitor-plugin)
5. [Workspace Cleanup Plugin](#workspace-cleanup-plugin)
6. [Copy Artifact Plugin](#copy-artifact-plugin)
7. [Parameterized Trigger Plugin](#parameterized-trigger-plugin)
8. [So Sánh Các Plugin Visualize](#so-sánh-các-plugin-visualize)

---

## Pipeline Utility Steps Plugin

**ID:** `pipeline-utility-steps`
**Giới Thiệu:** Cung cấp hàng chục step (bước) tiện ích trong Jenkinsfile — đọc/ghi file, thao tác YAML/JSON/Properties, nén file, tìm kiếm file.

### Đọc và Ghi File Cấu Hình

**readYaml / writeYaml** — Đọc và ghi file YAML:

```groovy
// Đọc file YAML
def config = readYaml file: 'deploy-config.yaml'
echo "Deploy to: ${config.environment.name}"
echo "Replicas: ${config.deployment.replicas}"

// Ghi file YAML
def newConfig = [
    version: "1.2.3",
    deployedAt: new Date().toString()
]
writeYaml file: 'deploy-result.yaml', data: newConfig
```

**readJSON / writeJSON** — Đọc và ghi file JSON:

```groovy
// Đọc package.json để lấy version
def pkg = readJSON file: 'package.json'
echo "App version: ${pkg.version}"

// Ghi file JSON
writeJSON file: 'build-info.json', json: [
    buildNumber: env.BUILD_NUMBER,
    gitCommit: env.GIT_COMMIT,
    timestamp: System.currentTimeMillis()
], pretty: 4
```

**readProperties** — Đọc file `.properties` kiểu Java:

```groovy
// app.properties: APP_VERSION=1.2.3 \n DB_HOST=localhost
def props = readProperties file: 'app.properties'
echo "Version: ${props.APP_VERSION}"
echo "DB: ${props.DB_HOST}"
```

---

### Tìm Kiếm và Thao Tác File

**findFiles** — Tìm file theo pattern (mẫu glob):

```groovy
// Tìm tất cả file test result
def testResults = findFiles(glob: '**/TEST-*.xml')
echo "Tìm thấy ${testResults.length} file test result"

// Tìm artifact sau build
def jars = findFiles(glob: 'target/*.jar')
if (jars.length == 0) {
    error("Không tìm thấy JAR file sau build!")
}
echo "JAR: ${jars[0].name} (${jars[0].length} bytes)"
```

**zip / unzip** — Nén và giải nén:

```groovy
// Nén thư mục dist thành artifact
zip zipFile: 'app-dist.zip', dir: 'dist/'

// Giải nén file config
unzip zipFile: 'config-bundle.zip', dir: 'config/'
```

**tar** — Nén tar.gz:

```groovy
tar file: 'artifacts.tar.gz', dir: 'build/', compress: true
```

---

### Tính Checksum và Xác Minh

```groovy
// Tính SHA256 của artifact
def sha = sha256 'target/app.jar'
echo "SHA256: ${sha}"

// Lưu vào file để CI artifact verification
writeFile file: 'app.jar.sha256', text: sha
```

---

### Ví Dụ Thực Tế: Pipeline Dùng Pipeline Utility Steps

```groovy
pipeline {
    agent any

    stages {
        stage('Read Config') {
            steps {
                script {
                    // Đọc version từ package.json
                    def pkg = readJSON file: 'package.json'
                    env.APP_VERSION = pkg.version
                    env.APP_NAME = pkg.name

                    // Đọc config deploy theo environment
                    def deployConfig = readYaml file: "deploy/${params.ENV}.yaml"
                    env.DEPLOY_NAMESPACE = deployConfig.kubernetes.namespace
                    env.REPLICAS = deployConfig.kubernetes.replicas.toString()
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Package') {
            steps {
                // Tìm và đánh dấu artifact
                script {
                    def distFiles = findFiles(glob: 'dist/**/*')
                    echo "Đóng gói ${distFiles.length} files"
                }
                zip zipFile: "${env.APP_NAME}-${env.APP_VERSION}.zip", dir: 'dist/'
            }
        }

        stage('Write Build Info') {
            steps {
                writeJSON file: 'build-info.json', json: [
                    name: env.APP_NAME,
                    version: env.APP_VERSION,
                    buildNumber: env.BUILD_NUMBER,
                    gitBranch: env.GIT_BRANCH,
                    gitCommit: env.GIT_COMMIT[0..7]
                ], pretty: 2
                archiveArtifacts 'build-info.json'
            }
        }
    }
}
```

---

## Blue Ocean Plugin — Chi Tiết

**ID:** `blueocean`
**URL truy cập:** `http://jenkins:8080/blue`

### Thành Phần Blue Ocean

Blue Ocean là một tập hợp plugin, không phải một plugin đơn:

| Plugin Con                     | Chức Năng                                    |
| ------------------------------ | -------------------------------------------- |
| blueocean-core-js              | JavaScript core và REST API                  |
| blueocean-pipeline-api-impl    | API cho Pipeline visualization               |
| blueocean-git-pipeline         | Tạo pipeline từ Git repository               |
| blueocean-github-pipeline      | Tạo pipeline từ GitHub                       |
| blueocean-gitlab-pipeline      | Tạo pipeline từ GitLab                       |
| blueocean-rest                 | REST API endpoints                           |

### Visualize Pipeline

Blue Ocean vẽ pipeline thành **flowchart** — thấy rõ:
- Stage nào đang chạy (xanh nhấp nháy)
- Stage nào đã thành công (xanh)
- Stage nào thất bại (đỏ)
- Thời gian mỗi stage

```
[Checkout] → [Build] → [Test (parallel)] → [Deploy]
               ✅          ✅  Unit Test        ✅
                           ❌  Integration Test
```

### Pipeline Editor (Trình Soạn Thảo Pipeline)

Blue Ocean có visual editor tạo Declarative Pipeline qua drag-and-drop — tốt cho người mới chưa quen cú pháp Jenkinsfile.

```
Blue Ocean → [Tên Job] → Edit → Thêm stage → Thêm step → Save
```

Kết quả lưu trực tiếp vào Jenkinsfile trong repository.

### Log Per Step

Thay vì một console log khổng lồ, Blue Ocean tách log theo từng step:

```
▶ Stage: Build
  ▶ sh 'mvn clean package'        [expand để xem]
  ▶ sh 'mvn surefire-report:report'  [expand để xem]
▶ Stage: Test
  ▶ sh 'mvn test'                 [expand để xem]   ← ❌ lỗi ở đây
```

---

## Pipeline Stage View Plugin

**ID:** `pipeline-stage-view`

### Stage View — Lịch Sử Build Theo Stage

Giao diện bảng lưới trong Jenkins Classic — không cần cài Blue Ocean:

```
             Build #5  Build #6  Build #7  Build #8
Checkout        ✅        ✅        ✅        ✅
Build           ✅        ✅        ✅       ❌  12s
Test            ✅        ✅        ✅        -
Deploy          ✅       ❌  5s     ✅        -

            12s       8s        15s       3s
```

- Màu: xanh = thành công, đỏ = thất bại, xám = bị bỏ qua
- Số dưới: thời gian build mỗi lần
- Click vào ô đỏ → xem log của stage đó ngay lập tức

### Cấu Hình Hiển Thị

```groovy
options {
    // Giới hạn số build hiển thị trong Stage View
    buildDiscarder(logRotator(numToKeepStr: '10'))
}
```

---

## Build Monitor Plugin

**ID:** `build-monitor-plugin`

Dashboard (bảng điều khiển) tổng quan nhiều job cùng lúc — thường dùng để hiển thị trên màn hình TV tại văn phòng.

```
┌─────────────────────────────────────────────────┐
│  BUILD MONITOR                                  │
├──────────────┬──────────────┬───────────────────┤
│  frontend    │  backend     │  mobile           │
│   ✅ #127    │  ❌ #89      │   ✅ #45          │
│  2 min ago  │  Just now    │  5 min ago        │
└──────────────┴──────────────┴───────────────────┘
```

**Cấu hình:**
```
New View → Build Monitor View → Chọn jobs muốn theo dõi
```

---

## Workspace Cleanup Plugin

**ID:** `ws-cleanup`

Tự động dọn dẹp (cleanup) workspace sau hoặc trước build — tránh tích tụ file cũ gây hết đĩa.

```groovy
pipeline {
    agent any
    options {
        // Cleanup trước khi bắt đầu build
        skipDefaultCheckout(false)
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
    post {
        always {
            // Dọn dẹp sau build dù thành công hay thất bại
            cleanWs(
                cleanWhenSuccess: true,
                cleanWhenFailure: false,    // Giữ lại để debug khi lỗi
                cleanWhenAborted: true,
                deleteDirs: true,
                patterns: [
                    [pattern: 'target/**', type: 'INCLUDE'],
                    [pattern: '**/.git/**', type: 'EXCLUDE']  // Giữ .git
                ]
            )
        }
    }
}
```

---

## Copy Artifact Plugin

**ID:** `copyartifact`

Copy artifact (kết quả build) từ một job khác sang job hiện tại — hữu ích khi tách pipeline thành nhiều job.

```groovy
// Trong job "deploy-production"
stage('Get Artifact') {
    steps {
        // Copy artifact từ job "build-backend" lần build gần nhất thành công
        copyArtifacts(
            projectName: 'build-backend',
            filter: 'target/app-*.jar',
            selector: lastSuccessful(),
            target: 'artifacts/'
        )
    }
}
```

```groovy
// Tùy chọn selector (bộ chọn build)
selector: lastSuccessful()              // Build thành công gần nhất
selector: specific('42')               // Build số 42
selector: latestSavedBuild()           // Build được "keep forever"
selector: upstream(fallbackToLastSuccessful: true)  // Build upstream tương ứng
```

**Lưu ý:** Job nguồn cần có cấu hình cho phép copy artifact:
```
Job Configuration → Post-build Actions → Archive artifacts: target/*.jar
→ Advanced → Allow the following projects to copy this artifact: deploy-*
```

---

## Parameterized Trigger Plugin

**ID:** `parameterized-trigger`

Kích hoạt job khác với tham số — tạo pipeline đa giai đoạn (multi-stage pipeline) bằng cách liên kết các job.

```groovy
// Trigger job deploy sau khi build thành công
stage('Trigger Deploy') {
    steps {
        build job: 'deploy-to-staging',
              parameters: [
                  string(name: 'APP_VERSION', value: env.APP_VERSION),
                  string(name: 'DOCKER_IMAGE', value: env.DOCKER_IMAGE),
                  booleanParam(name: 'RUN_SMOKE_TESTS', value: true)
              ],
              wait: true,          // Chờ job kia chạy xong
              propagate: true      // Nếu job kia fail → job này cũng fail
    }
}
```

---

## So Sánh Các Plugin Visualize

| Tính Năng                         | Blue Ocean          | Stage View         | Build Monitor      |
| --------------------------------- | ------------------- | ------------------ | ------------------ |
| **Giao diện**                    | Hiện đại, React SPA | Classic Jenkins    | Dashboard TV       |
| **Visualize pipeline**           | Flowchart chi tiết  | Bảng lưới lịch sử  | Trạng thái nhanh   |
| **Log per step**                 | ✅ Có               | ❌ Không           | ❌ Không           |
| **Pipeline editor (trực quan)**  | ✅ Có               | ❌ Không           | ❌ Không           |
| **Phù hợp cho**                  | Dev xem build cá nhân | Theo dõi lịch sử | Màn hình team      |
| **Trạng thái phát triển**        | Không tích cực      | Tích cực           | Tích cực           |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Pipeline Utility Steps Plugin giải quyết vấn đề gì mà Groovy thuần không làm được?**
> Groovy thuần có thể đọc file, nhưng Pipeline Utility Steps cung cấp các step được tối ưu cho sandbox Pipeline, xử lý path tương đối so với workspace, và tích hợp tốt hơn với Jenkins pipeline lifecycle. Ngoài ra, `readYaml`/`readJSON` trả về object native của Groovy thay vì phải parse thủ công.

**Q: Khi nào nên dùng Copy Artifact Plugin thay vì Stash/Unstash?**
> `Stash/Unstash` chỉ dùng trong cùng một Pipeline run, trong một Jenkinsfile. Copy Artifact dùng để lấy artifact từ **job khác** hoặc **build run khác** — ví dụ: job deploy cần lấy JAR từ job build của lần chạy cụ thể.

**Q: Tại sao Workspace Cleanup quan trọng trong môi trường production?**
> Mỗi build có thể tạo ra hàng trăm MB file tạm, dependency, artifact. Trên agent có disk nhỏ, không cleanup dẫn đến "No space left on device" (hết ổ đĩa) — toàn bộ pipeline trên node đó sẽ fail. Cleanup `post { always { cleanWs() } }` là best practice bắt buộc.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
