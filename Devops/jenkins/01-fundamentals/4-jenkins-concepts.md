# Khái Niệm Jenkins: Job, Build, Workspace, Artifact, View

## Tổng Quan

Trước khi viết pipeline hay vận hành Jenkins, cần nắm vững 5 khái niệm nền tảng: **Job**, **Build**, **Workspace**, **Artifact**, và **View**. Đây là "từ vựng" cơ bản khi làm việc với Jenkins hàng ngày.

---

## 1. Job (Công Việc) — Đơn Vị Cấu Hình

### Job là gì?

**Job** (hay Project) là đơn vị cấu hình cơ bản trong Jenkins — định nghĩa "làm gì" và "làm như thế nào". Job lưu trữ:
- Cấu hình source code (Git URL, branch)
- Build trigger (webhook, cron, manual)
- Build steps (sh, mvn, gradle...)
- Post-build actions (gửi email, archive artifact...)

### Các Loại Job

#### Freestyle Project (Dự Án Tự Do)

Job cổ điển nhất, cấu hình hoàn toàn qua UI (giao diện đồ họa). Không dùng Jenkinsfile.

```
✅ Ưu điểm: Dễ dùng, không cần biết Groovy
❌ Nhược điểm: Cấu hình không lưu trong SCM, khó review, khó tái tạo
⚠️  Phù hợp: Job đơn giản, team mới bắt đầu với Jenkins
```

#### Pipeline (Luồng Quy Trình)

Job dùng **Jenkinsfile** — file cấu hình pipeline lưu trong repository. Đây là loại job khuyến nghị hiện nay.

```
✅ Ưu điểm: Pipeline as Code, version-controlled, dễ review và audit
✅ Hỗ trợ Declarative và Scripted syntax
⚠️  Phù hợp: Hầu hết các use case hiện đại
```

#### Multibranch Pipeline (Pipeline Đa Nhánh)

Jenkins tự động phát hiện các nhánh (branch) và Pull Request trong repository, tạo Pipeline riêng cho từng nhánh.

```
✅ Ưu điểm: Tự động hóa hoàn toàn — push nhánh mới → có pipeline ngay
✅ Mỗi nhánh có build history riêng
✅ Hỗ trợ PR validation (kiểm tra Pull Request trước khi merge)
⚠️  Phù hợp: Team dùng Git Flow hoặc GitHub Flow
```

#### Organization Folder (Thư Mục Tổ Chức)

Quét toàn bộ GitHub Organization hoặc Bitbucket Project, tự động tạo Multibranch Pipeline cho mỗi repository có Jenkinsfile.

```
✅ Ưu điểm: Quản lý pipeline cho hàng chục repo một lúc
⚠️  Phù hợp: Tổ chức có nhiều repository
```

#### Folder (Thư Mục)

Không phải job thực sự — dùng để nhóm các job liên quan theo team, project, hoặc môi trường.

```
jenkins/
├── team-backend/         ← Folder
│   ├── api-service       ← Pipeline job
│   └── worker-service    ← Pipeline job
└── team-frontend/        ← Folder
    └── web-app           ← Pipeline job
```

### Tạo Job — Pipeline Job Cơ Bản

```
New Item
  ├── Nhập tên: my-app-pipeline
  ├── Chọn: Pipeline
  └── OK

Cấu hình:
  General:
    Description: Pipeline CI/CD cho my-app
    ☑ Discard old builds
      Max # of builds to keep: 20

  Pipeline:
    Definition: Pipeline script from SCM   ← Lấy Jenkinsfile từ repository
    SCM: Git
    Repository URL: https://github.com/org/my-app.git
    Credentials: github-creds
    Branch: */main
    Script Path: Jenkinsfile    ← Tên file, thường là Jenkinsfile ở root
```

### Job Configuration Được Lưu Ở Đâu?

```
$JENKINS_HOME/jobs/<tên-job>/config.xml
```

Ví dụ nội dung `config.xml` (rút gọn):
```xml
<?xml version='1.1' encoding='UTF-8'?>
<flow-definition plugin="workflow-job">
  <description>Pipeline CI/CD cho my-app</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <jenkins.model.BuildDiscarderProperty>
      <strategy class="LogRotator">
        <numToKeepStr>20</numToKeepStr>
      </strategy>
    </jenkins.model.BuildDiscarderProperty>
  </properties>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsScmFlowDefinition">
    <scm class="hudson.plugins.git.GitSCM">
      <userRemoteConfigs>
        <hudson.plugins.git.UserRemoteConfig>
          <url>https://github.com/org/my-app.git</url>
        </hudson.plugins.git.UserRemoteConfig>
      </userRemoteConfigs>
    </scm>
    <scriptPath>Jenkinsfile</scriptPath>
  </definition>
</flow-definition>
```

---

## 2. Build (Lần Thực Thi) — Một Lần Chạy Job

### Build là gì?

**Build** là một lần thực thi của một Job. Mỗi build có:
- **Build number** (số thứ tự): #1, #2, #3... tăng dần, không đặt lại
- **Build status** (trạng thái): SUCCESS, FAILURE, UNSTABLE, ABORTED, NOT_BUILT
- **Build log** (nhật ký): toàn bộ output của tất cả bước thực thi
- **Build duration** (thời gian chạy)
- **Build parameters** (tham số đầu vào, nếu job dùng tham số)

### Build Status — Trạng Thái Build

| Status | Màu | Ý Nghĩa |
|--------|-----|---------|
| **SUCCESS** | Xanh lá | Tất cả stage chạy thành công |
| **FAILURE** | Đỏ | Ít nhất một stage thất bại (exit code ≠ 0) |
| **UNSTABLE** | Vàng | Build chạy xong nhưng có cảnh báo (test thất bại, threshold vượt ngưỡng) |
| **ABORTED** | Xám | Build bị dừng thủ công hoặc do timeout |
| **NOT_BUILT** | Xám nhạt | Stage không chạy vì điều kiện `when` không thỏa |

### Build Lifecycle (Vòng Đời Build)

```
QUEUED (đang chờ trong queue)
    ↓ Executor rảnh
IN_PROGRESS (đang chạy)
    ↓ Hoàn thành
SUCCESS / FAILURE / UNSTABLE / ABORTED
```

### Build Number và Build URL

```
Build #45 của job "my-app":
  URL:    http://jenkins.example.com/job/my-app/45/
  Log:    http://jenkins.example.com/job/my-app/45/console
  API:    http://jenkins.example.com/job/my-app/45/api/json
```

Trong Jenkinsfile, `BUILD_NUMBER` là biến môi trường mặc định:
```groovy
sh "docker build -t my-app:${BUILD_NUMBER} ."
sh "docker push my-app:${BUILD_NUMBER}"
```

### Build Parameters (Tham Số Build)

Job có thể nhận tham số đầu vào khi trigger:

```groovy
pipeline {
    parameters {
        string(name: 'APP_VERSION', defaultValue: '1.0.0', description: 'Phiên bản ứng dụng')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Môi trường deploy')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Có chạy test không?')
    }
    stages {
        stage('Deploy') {
            steps {
                echo "Deploy version ${params.APP_VERSION} to ${params.DEPLOY_ENV}"
                script {
                    if (params.RUN_TESTS) {
                        sh 'mvn test'
                    }
                }
            }
        }
    }
}
```

### Build History (Lịch Sử Build) và Build Discard Policy

**Build History** lưu log và metadata của mỗi build. Không giới hạn → ổ đĩa đầy theo thời gian.

**Build Discard Policy** (chính sách xóa build cũ):

```groovy
// Trong Jenkinsfile
options {
    buildDiscarder(logRotator(
        numToKeepStr: '20',         // Giữ tối đa 20 build gần nhất
        daysToKeepStr: '30',        // Giữ build trong 30 ngày
        artifactNumToKeepStr: '5',  // Giữ artifact của 5 build gần nhất
        artifactDaysToKeepStr: '7'  // Giữ artifact trong 7 ngày
    ))
}
```

---

## 3. Workspace (Không Gian Làm Việc)

### Workspace là gì?

**Workspace** là thư mục trên Agent nơi Jenkins:
- Checkout (sao chép) source code từ Git
- Tạo các file trung gian trong quá trình build (compiled classes, test reports...)
- Tạo output artifact (jar, war, docker image...)

### Đường Dẫn Workspace Mặc Định

```
$JENKINS_HOME/workspace/<tên-job>/            # Build trên Controller
/var/jenkins/workspace/<tên-job>/             # Build trên Agent (tùy cấu hình)
/home/jenkins/agent/workspace/<tên-job>/      # Agent Docker thường dùng đường dẫn này
```

### Custom Workspace (Workspace Tùy Chỉnh)

```groovy
pipeline {
    agent {
        node {
            label 'linux'
            customWorkspace '/opt/build/my-app'   // Đặt workspace cố định
        }
    }
    // ...
}
```

### Workspace Trong Pipeline Đa Agent

Workspace **không được chia sẻ** giữa các Agent:

```groovy
pipeline {
    stages {
        stage('Build') {
            agent { label 'builder' }
            steps {
                sh 'mvn package'
                // Workspace nằm trên agent "builder"
                stash name: 'build-result', includes: 'target/*.jar'
            }
        }
        stage('Deploy') {
            agent { label 'deployer' }
            steps {
                // Agent "deployer" không có file từ stage Build
                unstash 'build-result'    // Lấy file từ Controller qua stash
                sh 'java -jar target/*.jar'
            }
        }
    }
}
```

### Dọn Dẹp Workspace

```groovy
// Xóa workspace sau khi build (dùng Workspace Cleanup Plugin)
post {
    always {
        cleanWs()
    }
}

// Xóa trước khi checkout
options {
    skipDefaultCheckout(true)
}
stages {
    stage('Checkout') {
        steps {
            cleanWs()
            checkout scm
        }
    }
}
```

---

## 4. Artifact (Kết Quả Build)

### Artifact là gì?

**Artifact** (kết quả build) là file được tạo ra từ quá trình build và cần được lưu lại để:
- Deploy lên server
- Chạy test ở stage sau
- Audit (kiểm tra lại) phiên bản đã release
- Chia sẻ giữa các job liên quan

Ví dụ artifact phổ biến:
- `.jar`, `.war`, `.ear` — Java applications
- Docker image (thường push lên registry thay vì archive trực tiếp)
- `.zip`, `.tar.gz` — bundle ứng dụng
- Test reports (JUnit XML, HTML coverage report)
- Terraform plan files

### Archive Artifact — Lưu Artifact Trong Jenkins

```groovy
post {
    success {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        archiveArtifacts artifacts: 'build/reports/**/*.html'
    }
}
```

Artifact được lưu tại:
```
$JENKINS_HOME/jobs/<tên-job>/builds/<số-build>/archive/
```

Truy cập qua URL:
```
http://jenkins.example.com/job/my-app/45/artifact/target/my-app-1.0.jar
```

### Fingerprint — Theo Dõi Artifact Giữa Các Job

**Fingerprint** (dấu vân tay) là MD5 hash của artifact, dùng để theo dõi một artifact được dùng bởi bao nhiêu job và build nào.

```groovy
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
```

Dùng artifact từ job khác:
```groovy
copyArtifacts(
    projectName: 'my-app/main',        // Lấy artifact từ job này
    filter: 'target/*.jar',
    fingerprintArtifacts: true,
    selector: lastSuccessfulBuild()    // Lấy từ build thành công gần nhất
)
```

### Stash/Unstash — Chuyển File Trong Pipeline

Khác với Archive, **Stash** (lưu tạm) chỉ tồn tại trong phạm vi một lần chạy pipeline:

```groovy
// Stage 1: Build và stash
stage('Build') {
    steps {
        sh 'mvn package'
        stash name: 'app-jar', includes: 'target/*.jar'
    }
}

// Stage 2: Test dùng file đã stash
stage('Integration Test') {
    agent { label 'test-agent' }
    steps {
        unstash 'app-jar'
        sh 'java -jar target/app.jar --test-mode'
    }
}
```

**So sánh Archive vs Stash:**

| | Archive | Stash |
|--|---------|-------|
| **Lưu lâu** | Vĩnh viễn (theo discard policy) | Chỉ trong một build |
| **Truy cập** | Qua UI và API | Chỉ trong pipeline đang chạy |
| **Mục đích** | Release artifact, audit | Chuyển file giữa agent trong pipeline |
| **Kích thước** | Lớn (jar, zip, reports) | Nhỏ đến trung bình |

---

## 5. View (Giao Diện Xem) — Tổ Chức Dashboard

### View là gì?

**View** (giao diện xem / bảng điều khiển) là cách nhóm và hiển thị các Job trên Dashboard Jenkins. Mặc định Jenkins có một view "All" hiển thị toàn bộ job.

### Các Loại View

#### List View (Dạng Danh Sách)

View mặc định — hiển thị job theo dạng bảng với các cột: tên, trạng thái, kết quả build cuối.

```
Tạo View:
  Dashboard → + (thêm view mới)
  Name: Backend Team
  Type: List View
  
  Cấu hình:
    Add Jobs: Chọn thủ công hoặc dùng regex
    Job Filters: Use a regular expression: my-app.*
```

#### My View (View Cá Nhân)

Tự động hiển thị các job mà người dùng có quyền truy cập — mỗi developer chỉ thấy job của team mình.

#### Nested Views (View Lồng Nhau)

Cần plugin **Nested Views** — tổ chức view theo cây thư mục.

#### Blue Ocean (Giao Diện Hiện Đại)

Không phải View truyền thống — Blue Ocean là plugin cung cấp giao diện pipeline visualization (trực quan hóa pipeline) hiện đại hơn.

Truy cập: `http://jenkins.example.com/blue`

---

## 6. Built-in Environment Variables (Biến Môi Trường Tích Hợp)

Jenkins tự động cung cấp các biến môi trường cho mỗi build:

| Biến | Ý Nghĩa | Ví Dụ Giá Trị |
|------|---------|---------------|
| `BUILD_NUMBER` | Số thứ tự build | `45` |
| `BUILD_ID` | ID duy nhất của build | `2024-01-15_10-30-00` |
| `BUILD_URL` | URL đầy đủ của build | `http://jenkins/job/my-app/45/` |
| `JOB_NAME` | Tên job | `my-app` |
| `JOB_BASE_NAME` | Tên job không có folder | `my-app` |
| `WORKSPACE` | Đường dẫn workspace trên Agent | `/var/jenkins/workspace/my-app` |
| `GIT_COMMIT` | Hash commit đang build | `abc123def456...` |
| `GIT_BRANCH` | Nhánh đang build | `origin/main` |
| `NODE_NAME` | Tên agent đang chạy | `linux-agent-01` |
| `JENKINS_URL` | URL của Jenkins Controller | `http://jenkins.example.com/` |

```groovy
// Sử dụng trong Jenkinsfile
steps {
    echo "Building ${JOB_NAME} #${BUILD_NUMBER}"
    echo "Git commit: ${GIT_COMMIT[0..7]}"     // 8 ký tự đầu của commit hash
    sh "docker tag my-app my-app:${BUILD_NUMBER}-${GIT_COMMIT[0..7]}"
}
```

Xem toàn bộ danh sách tại: `http://jenkins.example.com/env-vars.html`

---

## 7. Mối Quan Hệ Giữa Các Khái Niệm

```
JOB (Cấu hình — config.xml)
  │
  ├── Khi trigger (webhook/cron/manual)
  │
  ▼
BUILD #N (Lần thực thi — thư mục builds/N/)
  │
  ├── Checkout source code
  │         ↓
  │     WORKSPACE (Thư mục trên Agent)
  │         │  ← Tất cả bước build chạy tại đây
  │         ↓
  ├── Tạo kết quả build
  │         ↓
  │     ARTIFACT (Lưu vào archive/ hoặc push registry)
  │
  └── Build Log, Test Reports (lưu trong thư mục builds/N/)

VIEW (Dashboard — nhóm nhiều Job để xem nhanh)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác nhau giữa Freestyle Project và Pipeline Job là gì?**

A: Freestyle Project cấu hình hoàn toàn qua UI — dễ dùng nhưng cấu hình không được version-controlled trong SCM, khó review khi team lớn, không hỗ trợ các tính năng pipeline nâng cao như parallel stages hay conditional logic. Pipeline Job dùng Jenkinsfile lưu trong repository — cấu hình là code, được review qua pull request, có history đầy đủ, hỗ trợ toàn bộ tính năng Jenkins Pipeline. Trong môi trường production hiện đại, Pipeline Job (đặc biệt là Multibranch Pipeline) là tiêu chuẩn.

**Q: UNSTABLE build status có nghĩa là gì?**

A: UNSTABLE (không ổn định) là trạng thái trung gian — build chạy hoàn chỉnh nhưng kết quả không hoàn toàn tốt. Thường xảy ra khi: test pass nhưng test coverage thấp hơn ngưỡng quy định, có warning từ static analysis tool (SonarQube, Checkstyle), hoặc pipeline tự gọi `currentBuild.result = 'UNSTABLE'`. Khác với FAILURE — UNSTABLE không dừng pipeline ngay mà vẫn tiếp tục các stage tiếp theo (trừ khi có điều kiện `when` kiểm tra trạng thái). Dùng khi muốn cảnh báo mà không chặn deployment.

**Q: Khi nào dùng Archive Artifact và khi nào dùng Stash?**

A: Dùng **Stash** khi cần chuyển file giữa các stage hoặc agent trong cùng một lần chạy pipeline — ví dụ build JAR ở stage Build, chuyển sang stage Deploy trên agent khác. File stash tự động xóa sau khi pipeline kết thúc. Dùng **Archive Artifact** khi cần lưu file lâu dài để download sau, dùng cho deployment thực sự, hoặc tham chiếu từ job khác (với copyArtifacts). Archive artifact tồn tại theo Build Discard Policy, có thể truy cập qua UI và API sau nhiều ngày.
