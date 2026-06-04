# Multibranch Pipeline — Pipeline Đa Nhánh

> Multibranch Pipeline tự động phát hiện nhánh và Pull Request trong repository, tạo pipeline riêng cho từng nhánh có `Jenkinsfile`. Đây là cơ sở để triển khai chiến lược branch-based CI/CD và PR validation.

## Mục Lục

1. [Multibranch Pipeline là gì?](#multibranch-pipeline-là-gì)
2. [Cấu Hình Multibranch Pipeline](#cấu-hình-multibranch-pipeline)
3. [Branch Filtering — Lọc Nhánh](#branch-filtering--lọc-nhánh)
4. [Pull Request Builds — Build Cho PR](#pull-request-builds--build-cho-pr)
5. [Jenkinsfile Theo Từng Nhánh](#jenkinsfile-theo-từng-nhánh)
6. [Biến Môi Trường Multibranch](#biến-môi-trường-multibranch)
7. [Orphaned Branch Policy — Xử Lý Nhánh Bị Xóa](#orphaned-branch-policy--xử-lý-nhánh-bị-xóa)
8. [GitHub Branch Source — Tích Hợp GitHub](#github-branch-source--tích-hợp-github)
9. [GitLab Branch Source — Tích Hợp GitLab](#gitlab-branch-source--tích-hợp-gitlab)
10. [Organization Folder — Quét Toàn Bộ Tổ Chức](#organization-folder--quét-toàn-bộ-tổ-chức)
11. [Chiến Lược Nhánh Thực Tế](#chiến-lược-nhánh-thực-tế)
12. [Troubleshooting — Xử Lý Sự Cố](#troubleshooting--xử-lý-sự-cố)

---

## Multibranch Pipeline là gì?

**Multibranch Pipeline** là loại Jenkins job đặc biệt tự động quét repository, phát hiện tất cả nhánh và Pull Request có chứa `Jenkinsfile`, rồi tạo sub-job riêng cho mỗi nhánh.

### Pipeline Job Thường vs Multibranch Pipeline

```
Pipeline Job Thường:                    Multibranch Pipeline:
──────────────────────                  ───────────────────────────────
my-app-job                              my-app/
  └── Chạy trên MỘT nhánh cố định        ├── main          (auto-created)
      (phải cấu hình thủ công)            ├── develop       (auto-created)
                                          ├── feature/auth  (auto-created)
                                          ├── feature/pay   (auto-created)
                                          └── PR-42         (auto-created)
```

### Vòng Đời Tự Động

```
1. Developer push code lên nhánh mới
           ↓
2. Webhook (hoặc SCM polling) thông báo Jenkins
           ↓
3. Jenkins quét repository → phát hiện Jenkinsfile trong nhánh mới
           ↓
4. Jenkins tự động tạo sub-job cho nhánh đó
           ↓
5. Build chạy ngay lập tức
           ↓
6. Developer xóa nhánh → Jenkins tự xóa sub-job (theo chính sách)
```

### Lợi Ích

| Lợi Ích                     | Mô Tả                                                              |
| --------------------------- | ------------------------------------------------------------------ |
| **Tự Động Hóa**             | Không cần tạo job thủ công khi có nhánh mới                       |
| **PR Validation**           | Build tự động chạy cho mỗi Pull Request, báo cáo status lên GitHub/GitLab |
| **Branch Isolation**        | Mỗi nhánh có pipeline riêng, config riêng trong Jenkinsfile        |
| **Clean Up Tự Động**        | Job của nhánh bị xóa sẽ tự được dọn sau thời gian cấu hình       |
| **Visibility**              | Blue Ocean UI hiển thị trạng thái mỗi nhánh rõ ràng               |

---

## Cấu Hình Multibranch Pipeline

### Bước 1: Tạo Job

1. Jenkins Dashboard → **New Item**
2. Nhập tên job → Chọn **Multibranch Pipeline** → OK

### Bước 2: Branch Sources (Nguồn Nhánh)

Cấu hình trong phần **Branch Sources**:

#### GitHub

```
Branch Sources → Add source → GitHub
├── Credentials: [Chọn GitHub Personal Access Token]
├── Repository HTTPS URL: https://github.com/org/repo
├── Behaviors:
│   ├── Discover branches: Exclude branches that are also filed as PRs
│   ├── Discover pull requests from origin: Merged with target branch
│   └── Discover pull requests from forks: Merged with target branch
```

**Quyền cần thiết cho GitHub PAT (Personal Access Token):**
- `repo` (full control of private repositories)
- Hoặc `public_repo` cho public repos

#### GitLab

```
Branch Sources → Add source → GitLab
├── Server: [Chọn GitLab server đã cấu hình]
├── Credentials: [GitLab API Token]
├── Owner: org-name
├── Projects: repo-name
└── Behaviors: [Tương tự GitHub]
```

### Bước 3: Build Configuration

```
Build Configuration:
├── Mode: by Jenkinsfile
└── Script Path: Jenkinsfile    (hoặc đường dẫn tùy chỉnh)
```

### Bước 4: Scan Repository Triggers

```
Scan Repository Triggers:
├── Periodically if not otherwise run: [tick] Interval: 1 day
└── (Webhook: được cấu hình tự động nếu dùng GitHub Branch Source plugin)
```

### Bước 5: Orphaned Item Strategy

```
Orphaned Item Strategy:
├── Discard old items: [tick]
├── Days to keep old items: 7
└── Max # of old items to keep: 5
```

---

## Branch Filtering — Lọc Nhánh

Không phải mọi nhánh đều cần pipeline. Cấu hình filter để chỉ build nhánh quan trọng.

### Filter Bằng Regular Expression (Cơ Bản)

Trong **Branch Sources → Behaviors → Filter by name (with regular expression)**:

```
# Chỉ build main, develop, release/*, hotfix/*
^(main|develop|release/.*|hotfix/.*)$

# Build tất cả trừ nhánh bắt đầu bằng wip/
^(?!wip/).*$

# Chỉ build feature branches
^feature/.*$

# Build main và release branches
^(main|master|release-.*)$
```

### Wildcard Strategy (Cách Đơn Giản Hơn)

**Branch Sources → Behaviors → Filter by name (with wildcards)**:

```
Include: main develop release/* hotfix/* feature/*
Exclude: wip/* temp/*
```

### Filter Bằng Pipeline Script (Linh Hoạt Nhất)

Dùng **Filter by name (with script)**:

```groovy
// Groovy script — trả về true nếu nhánh được phép build
if (branchName == 'main') return true
if (branchName == 'develop') return true
if (branchName.startsWith('release/')) return true
if (branchName.startsWith('feature/') && branchName.length() < 50) return true
return false
```

### Property Strategy — Cấu Hình Riêng Cho Nhóm Nhánh

Với **Jenkins 2.x** và plugin **Basic Branch Build Strategies**:

```groovy
// Ví dụ: Chỉ build main nếu có file thay đổi, feature branches build tất cả
// (Cấu hình trong Jenkins UI, không phải trong Jenkinsfile)
```

---

## Pull Request Builds — Build Cho PR

### Cách Hoạt Động

```
Developer tạo PR: feature/auth → main
         ↓
Webhook gửi sự kiện PR mới đến Jenkins
         ↓
Jenkins tạo sub-job: PR-42 (tên tự động)
         ↓
Jenkins merge virtual: feature/auth + main → chạy Jenkinsfile trên kết quả merge
         ↓
Kết quả build được báo cáo lên GitHub/GitLab dưới dạng commit status
         ↓
PR không được merge nếu build fail (nếu đặt Required status checks)
```

### Cấu Hình PR Discovery

Trong **Branch Sources → Behaviors**:

```
Discover pull requests from origin:
├── Merge pull request with target branch revision  ← Khuyến nghị
│   (Kiểm tra code SAU KHI merge với target)
├── The current pull request revision
│   (Kiểm tra code TRƯỚC khi merge — nhanh hơn nhưng ít chính xác hơn)
└── Both the current pull request revision and the merge revision
    (Kiểm tra cả hai — tốn tài nguyên hơn)
```

### Phát Hiện PR Từ Fork

```
Discover pull requests from forks:
├── Trust: Contributors  ← Chỉ build PR từ contributor đã approved
│   (An toàn hơn — tránh PR độc hại từ người lạ chạy code nguy hiểm)
├── Trust: Everyone
│   (Không an toàn cho public repo — ai cũng chạy code được)
└── Trust: Nobody
    (Phải approve thủ công mỗi PR từ fork)
```

**Lưu ý bảo mật:** Không dùng `Trust: Everyone` cho public repository — người dùng độc hại có thể tạo PR để chạy code nguy hiểm trên agent của bạn (đánh cắp credentials, tấn công mạng nội bộ...).

### Kiểm Tra PR Trong Jenkinsfile

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

        // Stage này chỉ chạy cho PR — kiểm tra thêm chất lượng code
        stage('PR Quality Gate') {
            when { changeRequest() }    // Chỉ chạy khi build là PR
            steps {
                // Kiểm tra coverage tăng không giảm
                sh 'mvn jacoco:check'
                // Chạy SonarQube analysis và comment lên PR
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.pullrequest.key=${CHANGE_ID} -Dsonar.pullrequest.branch=${BRANCH_NAME} -Dsonar.pullrequest.base=${CHANGE_TARGET}'
                }
            }
        }

        // Deploy staging chỉ cho nhánh main (sau khi PR được merge)
        stage('Deploy Staging') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }    // Không phải PR
                }
            }
            steps { sh './deploy.sh staging' }
        }

        // Tạo preview environment cho PR (cần infrastructure hỗ trợ)
        stage('Deploy PR Preview') {
            when { changeRequest() }
            steps {
                sh "./deploy-preview.sh pr-${env.CHANGE_ID}"
                // Comment URL lên PR
                script {
                    def previewUrl = "https://pr-${env.CHANGE_ID}.preview.example.com"
                    pullRequest.comment("Preview environment: ${previewUrl}")
                }
            }
        }
    }
}
```

---

## Jenkinsfile Theo Từng Nhánh

Trong Multibranch Pipeline, mỗi nhánh chạy `Jenkinsfile` của chính nhánh đó. Điều này cho phép feature branch có cấu hình pipeline khác với main.

### Pattern 1: Một Jenkinsfile Xử Lý Tất Cả Nhánh

Dùng `when` và `env.BRANCH_NAME` để phân nhánh logic:

```groovy
pipeline {
    agent any

    environment {
        IS_MAIN    = "${env.BRANCH_NAME == 'main'}"
        IS_PR      = "${env.CHANGE_ID != null}"
        DEPLOY_ENV = sh(script: '''
            case "$BRANCH_NAME" in
                main)        echo "production" ;;
                develop)     echo "staging" ;;
                release/*)   echo "uat" ;;
                *)           echo "none" ;;
            esac
        ''', returnStdout: true).trim()
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Code Quality') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    changeRequest()    // PR
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                script {
                    def tag = (env.BRANCH_NAME == 'main') ? 'latest' : env.BRANCH_NAME.replace('/', '-')
                    withCredentials([usernamePassword(credentialsId: 'registry', usernameVariable: 'U', passwordVariable: 'P')]) {
                        sh """
                            docker login -u $U -p $P registry.example.com
                            docker build -t registry.example.com/myapp:${tag} .
                            docker push registry.example.com/myapp:${tag}
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                script {
                    if (env.DEPLOY_ENV != 'none') {
                        sh "./deploy.sh ${env.DEPLOY_ENV}"
                    }
                }
            }
        }

        stage('Production Approval') {
            when { branch 'main' }
            steps {
                input message: 'Deploy lên Production?',
                      ok:      'Xác Nhận',
                      submitter: 'ops-team,tech-leads'
            }
        }

        stage('Deploy Production') {
            when { branch 'main' }
            steps {
                sh './deploy.sh production'
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
            cleanWs()
        }
        success {
            script {
                if (env.CHANGE_ID) {
                    // Gửi status thành công lên PR
                    echo "Build PR #${env.CHANGE_ID} thành công"
                }
            }
        }
    }
}
```

### Pattern 2: Shared Library + Thin Jenkinsfile

Mỗi repo có Jenkinsfile mỏng, logic dùng chung đặt trong Shared Library:

```groovy
// Jenkinsfile — Mỏng và đơn giản
@Library('company-pipeline@v3') _

// Gọi hàm từ Shared Library — tự xử lý branch logic bên trong
javaServicePipeline(
    appName:      'payment-service',
    deployEnvMap: [
        'main':       'production',
        'develop':    'staging',
        'release/*':  'uat'
    ],
    productionApprovers: 'ops-team',
    sonarEnabled: true
)
```

---

## Biến Môi Trường Multibranch

Các biến chỉ có trong Multibranch Pipeline:

```groovy
// Tên nhánh hiện tại
env.BRANCH_NAME         // "feature/my-feature", "main", "PR-42"

// Khi là PR build (CHANGE_* chỉ có giá trị khi build là PR)
env.CHANGE_ID           // "42" (số PR)
env.CHANGE_URL          // "https://github.com/org/repo/pull/42"
env.CHANGE_TITLE        // "Add payment feature"
env.CHANGE_AUTHOR       // "alice"
env.CHANGE_AUTHOR_EMAIL // "alice@example.com"
env.CHANGE_AUTHOR_DISPLAY_NAME  // "Alice Smith"
env.CHANGE_TARGET       // "main" (nhánh đích của PR)
env.CHANGE_BRANCH       // "feature/payment" (nhánh nguồn của PR)
env.CHANGE_FORK         // "" hoặc "fork-owner" (nếu từ fork)

// Kiểm tra có phải PR không
script {
    boolean isPR = (env.CHANGE_ID != null)
    // Hoặc dùng when { changeRequest() }
}
```

---

## Orphaned Branch Policy — Xử Lý Nhánh Bị Xóa

Khi developer xóa nhánh, Jenkins cần biết phải làm gì với sub-job tương ứng.

### Cấu Hình Trong Jenkins UI

```
Orphaned Item Strategy:
├── Discard old items: [tick]
├── Days to keep old items: 7     ← Giữ sub-job thêm 7 ngày sau khi nhánh bị xóa
└── Max # of old items to keep: 5
```

### Cấu Hình Bằng JCasC (Jenkins Configuration as Code)

```yaml
jobs:
  - script: |
      multibranchPipelineJob('my-app') {
        branchSources {
          github {
            id('my-app-source')
            scanCredentialsId('github-token')
            repoOwner('my-org')
            repository('my-app')
          }
        }
        orphanedItemStrategy {
          discardOldItems {
            daysToKeep(7)
            numToKeep(5)
          }
        }
      }
```

### Dọn Dẹp PR Environments Khi Nhánh Bị Xóa

Có thể dùng `post { cleanup { } }` để dọn dẹp PR preview environments:

```groovy
post {
    always {
        script {
            // Khi nhánh là PR → dọn preview environment sau khi build
            if (env.CHANGE_ID && currentBuild.result != 'SUCCESS') {
                sh "./cleanup-preview.sh pr-${env.CHANGE_ID}"
            }
        }
    }
}
```

Hoặc dùng **Multibranch Action Triggers plugin** để trigger cleanup job khi nhánh bị xóa.

---

## GitHub Branch Source — Tích Hợp GitHub

### Cấu Hình Credentials

Tạo GitHub Personal Access Token với quyền:
- `repo` (để đọc private repos, tạo commit statuses)
- `workflow` (nếu cần trigger GitHub Actions)
- `admin:repo_hook` (để tự động tạo webhook)

Thêm vào Jenkins:
**Manage Jenkins → Manage Credentials → Global → Add Credentials**
- Kind: `Secret text`
- Secret: `ghp_xxxxxxxxxxxx` (GitHub PAT)
- ID: `github-pat`

### Cấu Hình GitHub Server

**Manage Jenkins → Configure System → GitHub**:
```
GitHub Servers:
├── Name: github.com
├── API URL: https://api.github.com
└── Credentials: [Chọn PAT đã tạo]
```

### Tính Năng GitHub Checks

Khi dùng GitHub Branch Source plugin, Jenkins tự động:
- Tạo **Commit Status** trên GitHub sau mỗi build
- PR không được merge nếu Jenkins status là **failed** (cấu hình Required Status Checks trên GitHub)

```groovy
// Jenkinsfile có thể ghi đè context của commit status
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                // publishChecks tạo GitHub Check thay vì chỉ Commit Status
                publishChecks name: 'Build', status: 'IN_PROGRESS'
                sh 'mvn package'
                publishChecks name: 'Build', status: 'COMPLETED', conclusion: 'SUCCESS'
            }
        }
    }
}
```

### Required Status Checks Trên GitHub

Cấu hình trên GitHub để bảo vệ nhánh main:

```
Repository Settings → Branches → Branch protection rules → main:
├── Require status checks to pass before merging: [tick]
├── Require branches to be up to date before merging: [tick]
└── Status checks that are required:
    ├── continuous-integration/jenkins/branch
    └── continuous-integration/jenkins/pr-merge
```

---

## GitLab Branch Source — Tích Hợp GitLab

### Cấu Hình GitLab Server

**Manage Jenkins → Configure System → GitLab**:
```
GitLab connections:
├── Connection name: gitlab.company.com
├── Gitlab host URL: https://gitlab.company.com
└── Credentials: [GitLab API Token]
```

### Tạo GitLab API Token

GitLab → Settings → Access Tokens:
- Name: `jenkins-integration`
- Scopes: `api`, `read_repository`

### Discover Merge Requests

GitLab gọi Pull Request là **Merge Request (MR)**:

```
Branch Sources → GitLab:
├── Behaviors:
│   ├── Discover branches: Exclude branches that are also filed as MRs
│   ├── Discover merge requests from origin:
│   │   └── Merged request with target branch revision
│   └── Discover merge requests from forks:
│       └── Trust: Members (Khuyến nghị)
```

### Commit Status Trên GitLab

Jenkins tự động gửi commit status về GitLab. MR hiển thị:
- Pipeline status: ✅ passed / ❌ failed / 🔄 running

Cấu hình MR không được merge khi pipeline fail:
**GitLab → Project → Settings → General → Merge requests → Pipelines must succeed**

---

## Organization Folder — Quét Toàn Bộ Tổ Chức

**Organization Folder** quét toàn bộ GitHub Organization hoặc GitLab Group, tự động tạo Multibranch Pipeline cho mỗi repository có `Jenkinsfile`.

### Tạo GitHub Organization Folder

```
New Item → GitHub Organization
├── Projects → GitHub Organization → Owner: my-org
├── Repository name pattern: .* (tất cả repos)
│   Hoặc: payment-.* (chỉ repos bắt đầu bằng "payment-")
├── Automatic branch project trigger: On push
└── Scan interval: 1 day
```

### Kết Quả

```
Jenkins:
└── my-org/                    ← Organization Folder
    ├── payment-service/       ← Multibranch Pipeline (tự động tạo)
    │   ├── main
    │   ├── develop
    │   └── feature/auth
    ├── user-service/          ← Multibranch Pipeline (tự động tạo)
    │   ├── main
    │   └── hotfix/urgent-fix
    └── notification-service/  ← Multibranch Pipeline (tự động tạo)
        └── main
```

### Bỏ Qua Repository Không Cần CI

Đặt file `.ci-skip` hoặc cấu hình trong `Jenkinsfile`:

```groovy
// Jenkinsfile — Repository không cần CI đầy đủ
// Chỉ chạy linting, không build hay deploy
pipeline {
    agent any
    stages {
        stage('Lint') {
            steps { sh 'pre-commit run --all-files' }
        }
    }
}
```

Hoặc dùng file đặc biệt để báo Jenkins bỏ qua:

```yaml
# .jenkins-ci.yaml (chỉ với Organization Folder có hỗ trợ)
ci-skip: true
```

---

## Chiến Lược Nhánh Thực Tế

### Git Flow với Multibranch Pipeline

```
main          → Deploy production (với approval)
│
develop       → Deploy staging (tự động sau merge)
│
release/1.2.0 → Deploy UAT (tự động sau tạo nhánh)
│
hotfix/fix-bug → Deploy staging để test, sau đó merge vào main
│
feature/*     → Build + test + PR preview (không deploy chính thức)
```

```groovy
// Jenkinsfile cho Git Flow
pipeline {
    agent any

    environment {
        // Xác định môi trường deploy dựa trên tên nhánh
        DEPLOY_ENV = sh(script: '''
            case "$BRANCH_NAME" in
                main)         echo "production" ;;
                develop)      echo "staging"    ;;
                release/*)    echo "uat"        ;;
                hotfix/*)     echo "hotfix"     ;;
                feature/*)    echo "preview"    ;;
                *)            echo "none"       ;;
            esac
        ''', returnStdout: true).trim()
    }

    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean verify'
            }
            post {
                always { junit 'target/surefire-reports/*.xml' }
            }
        }

        stage('SonarQube') {
            when {
                anyOf {
                    branch 'main'; branch 'develop'; changeRequest()
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { return env.DEPLOY_ENV != 'none' }
            }
            steps {
                script {
                    def tag = env.BRANCH_NAME.replace('/', '-')
                    withCredentials([usernamePassword(credentialsId: 'registry', usernameVariable: 'U', passwordVariable: 'P')]) {
                        sh """
                            docker login -u $U -p $P registry.example.com
                            docker build -t registry.example.com/myapp:${tag} .
                            docker push registry.example.com/myapp:${tag}
                        """
                    }
                    env.IMAGE_TAG = tag
                }
            }
        }

        stage('Deploy Preview (PR)') {
            when { changeRequest() }
            steps {
                sh "./deploy-preview.sh pr-${env.CHANGE_ID} ${env.IMAGE_TAG}"
                script {
                    def url = "https://pr-${env.CHANGE_ID}.preview.example.com"
                    echo "Preview: ${url}"
                    // Cần GitHub/GitLab plugin để comment lên PR
                }
            }
        }

        stage('Deploy Non-Production') {
            when {
                allOf {
                    expression { return env.DEPLOY_ENV in ['staging', 'uat', 'hotfix'] }
                    not { changeRequest() }
                }
            }
            steps {
                sh "./deploy.sh ${env.DEPLOY_ENV} ${env.IMAGE_TAG}"
            }
        }

        stage('Production Approval') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                input message:   "Deploy ${env.IMAGE_TAG} lên Production?",
                      ok:        'Phê Duyệt',
                      submitter: 'tech-leads,ops-team'
            }
        }

        stage('Deploy Production') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                withCredentials([file(credentialsId: 'prod-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        helm upgrade --install myapp ./helm/myapp \
                            --namespace production \
                            --set image.tag=${env.IMAGE_TAG} \
                            --atomic --timeout 10m
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.CHANGE_ID) {
                    echo "PR #${env.CHANGE_ID} build thành công — sẵn sàng để review"
                } else if (env.BRANCH_NAME == 'main') {
                    slackSend channel: '#releases', color: 'good',
                              message: "✅ Release ${env.IMAGE_TAG} deploy production thành công"
                }
            }
        }
        failure {
            slackSend channel: '#ci-alerts', color: 'danger',
                      message: "❌ Build thất bại: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
        }
        cleanup {
            // Dọn preview environment khi PR build kết thúc (dù pass hay fail)
            script {
                if (env.CHANGE_ID) {
                    sh "./cleanup-preview.sh pr-${env.CHANGE_ID} || true"
                }
            }
            cleanWs()
        }
    }
}
```

---

## Troubleshooting — Xử Lý Sự Cố

### Vấn Đề 1: Nhánh Mới Không Được Tự Động Phát Hiện

**Triệu chứng:** Developer tạo nhánh mới nhưng Jenkins không tạo job tương ứng.

**Kiểm tra:**
- Webhook có được cấu hình và gọi đến Jenkins không?
- Jenkins URL có accessible từ GitHub/GitLab không?
- Credentials có quyền `admin:repo_hook` không (để tự tạo webhook)?

**Giải pháp tạm thời:** Bấm **"Scan Multibranch Pipeline Now"** trên Jenkins UI.

**Giải pháp lâu dài:**
```
Jenkins → Manage Jenkins → Configure System → GitHub
→ Re-register hooks for all jobs → Bấm "Re-register"
```

### Vấn Đề 2: Jenkinsfile Trong Nhánh Không Được Đọc

**Triệu chứng:** Tạo nhánh mới từ `main` nhưng Jenkins vẫn dùng Jenkinsfile của `main`.

**Nguyên nhân:** Đây là hành vi đúng! Trong Multibranch Pipeline, mỗi nhánh dùng `Jenkinsfile` của chính nhánh đó. Nếu nhánh mới không thay đổi `Jenkinsfile` thì nó giống `main`.

### Vấn Đề 3: PR Build Không Tạo Commit Status Trên GitHub

**Kiểm tra:**
```bash
# Test webhook thủ công
curl -X POST https://jenkins.example.com/github-webhook/ \
     -H "Content-Type: application/json" \
     -d '{"ref": "refs/heads/main", "repository": {"full_name": "org/repo"}}'
```

**Kiểm tra Jenkins log:**
- Manage Jenkins → System Log → All Jenkins Logs → Lọc "GitHub"

### Vấn Đề 4: Build Chạy Trùng Lặp Cho PR

**Nguyên nhân:** Cả `Discover branches` và `Discover pull requests` đều được bật với cùng cấu hình.

**Giải pháp:** Cấu hình `Discover branches` thành **"Exclude branches that are also filed as PRs"** để tránh build trùng.

### Vấn Đề 5: Job Cũ Của Nhánh Đã Xóa Không Bị Dọn

**Giải pháp:** Cấu hình **Orphaned Item Strategy** với `Days to keep old items: 7`.

Hoặc dọn thủ công:
```
Jenkins → My-App Multibranch → Các job nhánh cũ → Delete
```

### Vấn Đề 6: `env.BRANCH_NAME` Có Giá Trị `null` Khi Chạy Pipeline Job Thường

**Nguyên nhân:** `BRANCH_NAME` chỉ có giá trị trong Multibranch Pipeline, không có trong Pipeline Job thường.

**Giải pháp:**
```groovy
// Kiểm tra an toàn
def branchName = env.BRANCH_NAME ?: sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
```

---

**Quay Lại:** Đọc [README.md](README.md) để xem tổng quan chủ đề 02-pipeline.

**Chủ Đề Tiếp Theo:** Xem [03-plugins/README.md](../03-plugins/README.md) để tìm hiểu Plugin Management.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
