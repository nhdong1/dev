# Build Artifacts — Lưu Trữ và Quản Lý Kết Quả Build

> **Build Artifact** (kết quả build) là bất kỳ file nào được sinh ra trong quá trình build mà bạn muốn giữ lại — file JAR, WAR, binary, Docker image, báo cáo test, package ZIP... Jenkins cung cấp ba cơ chế chính: **Archive** (lưu trữ dài hạn), **Stash/Unstash** (truyền tạm thời giữa các stage), và **Fingerprint** (theo dõi nguồn gốc artifact).

---

## Mục Lục

1. [Tổng Quan Artifact Management](#tổng-quan-artifact-management)
2. [Archive Artifacts — Lưu Trữ Kết Quả Build](#archive-artifacts--lưu-trữ-kết-quả-build)
3. [Stash và Unstash — Truyền File Giữa Các Stage](#stash-và-unstash--truyền-file-giữa-các-stage)
4. [Fingerprint — Theo Dõi Nguồn Gốc Artifact](#fingerprint--theo-dõi-nguồn-gốc-artifact)
5. [So Sánh Archive vs Stash](#so-sánh-archive-vs-stash)
6. [Quản Lý Dung Lượng Artifact](#quản-lý-dung-lượng-artifact)
7. [Tích Hợp Artifact Repository Bên Ngoài](#tích-hợp-artifact-repository-bên-ngoài)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Artifact Management

### Tại Sao Cần Quản Lý Artifact

```
Build Pipeline
    │
    ├── stage('Build')  → sinh ra: target/app.jar
    │                              target/app.war
    │
    ├── stage('Test')   → sinh ra: test-results/
    │                              coverage/
    │
    └── stage('Package') → sinh ra: myapp-1.2.3.zip
                                    Dockerfile.built

Vấn đề cần giải quyết:
1. Truyền file JAR từ stage Build sang stage Test (cùng pipeline, khác agent)
2. Lưu file JAR để tải về sau khi build xong
3. Biết file JAR này được build từ commit nào, pipeline nào
4. Tự động xóa artifact cũ để tiết kiệm ổ đĩa
```

### Ba Cơ Chế Chính

| Cơ Chế | Mục Đích | Thời Gian Sống | Phạm Vi |
|---------|----------|----------------|---------|
| **Archive** | Lưu trữ lâu dài, tải về được | Cấu hình theo Build Discard Policy | Tất cả build, có thể download |
| **Stash** | Truyền file tạm thời | Chỉ trong pipeline hiện tại | Giữa các stage trong cùng pipeline |
| **Fingerprint** | Theo dõi nguồn gốc | Mãi mãi (metadata) | Xuyên nhiều job |

---

## Archive Artifacts — Lưu Trữ Kết Quả Build

**Archive Artifacts** lưu file vào Jenkins Master và cho phép tải về từ giao diện web sau khi build hoàn thành.

### Cú Pháp Cơ Bản

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Archive') {
            steps {
                // Lưu tất cả file JAR trong thư mục target/
                archiveArtifacts artifacts: 'target/*.jar'

                // Lưu nhiều pattern cùng lúc
                archiveArtifacts artifacts: 'target/*.jar, target/*.war'

                // Lưu đệ quy (recursive) trong tất cả thư mục con
                archiveArtifacts artifacts: '**/target/*.jar'
            }
        }
    }
}
```

### Tùy Chọn Nâng Cao

```groovy
archiveArtifacts(
    artifacts: 'target/*.jar, reports/**/*.xml',

    // Không fail pipeline nếu không tìm thấy file khớp pattern
    allowEmptyArchive: true,

    // Fingerprint artifact (tạo hash MD5 để theo dõi)
    fingerprint: true,

    // Không lưu artifact từ các build cũ hơn khi dùng pattern **
    onlyIfSuccessful: true,

    // Loại trừ pattern cụ thể
    excludes: 'target/*-sources.jar, target/*-javadoc.jar'
)
```

### Đặt archiveArtifacts trong post Block (Khuyến Nghị)

```groovy
pipeline {
    agent any

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
            post {
                always {
                    // Luôn lưu test report dù pass hay fail
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }
    }

    post {
        success {
            // Chỉ lưu JAR khi build thành công
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
        always {
            // Luôn lưu log và báo cáo để debug
            archiveArtifacts(
                artifacts: 'logs/**, target/surefire-reports/**',
                allowEmptyArchive: true
            )
        }
    }
}
```

### Truy Cập Artifact từ UI

```
Jenkins Job → Build #42 → Artifacts
├── target/
│   ├── myapp-1.2.3.jar        [download]
│   └── myapp-1.2.3-tests.jar  [download]
└── logs/
    └── build.log              [download]
```

### Truy Cập Artifact qua API

```bash
# Tải artifact từ build mới nhất
curl -O "http://jenkins.example.com/job/my-job/lastSuccessfulBuild/artifact/target/myapp.jar" \
     --user "user:api-token"

# Tải artifact từ build cụ thể (build #42)
curl -O "http://jenkins.example.com/job/my-job/42/artifact/target/myapp.jar" \
     --user "user:api-token"

# Tải tất cả artifacts dưới dạng ZIP
curl -O "http://jenkins.example.com/job/my-job/lastBuild/artifact/*zip*/archive.zip" \
     --user "user:api-token"
```

---

## Stash và Unstash — Truyền File Giữa Các Stage

**Stash** (cất tạm) lưu file vào bộ nhớ tạm trên Jenkins Master. **Unstash** (lấy ra) khôi phục file đó ở bất kỳ stage nào, kể cả trên agent khác. Artifact bị xóa khi pipeline kết thúc.

### Cú Pháp Cơ Bản

```groovy
pipeline {
    agent none  // Không có agent mặc định — mỗi stage tự chọn

    stages {
        stage('Build') {
            agent { label 'linux-builder' }  // Agent có Maven, Java

            steps {
                sh 'mvn clean package -DskipTests'

                // Cất file JAR vào stash với tên "app-jar"
                stash(
                    name: 'app-jar',
                    includes: 'target/*.jar'
                )
            }
        }

        stage('Test on Linux') {
            agent { label 'linux-tester' }  // Agent khác

            steps {
                // Lấy JAR từ stash — file được chuyển sang agent này
                unstash 'app-jar'

                sh 'java -jar target/myapp.jar --self-test'
                sh 'mvn test -pl integration-tests'
            }
        }

        stage('Test on Windows') {
            agent { label 'windows-tester' }  // Agent Windows!

            steps {
                // Cùng JAR, chạy trên Windows
                unstash 'app-jar'

                bat 'java -jar target\\myapp.jar --self-test'
            }
        }
    }
}
```

### Tùy Chọn Stash

```groovy
stash(
    name: 'build-output',          // Tên định danh (bắt buộc)
    includes: 'target/**/*.jar',   // File cần cất (Ant glob pattern)
    excludes: 'target/**/*-test*.jar, target/**/*-sources.jar',  // File loại trừ
    allowEmpty: true               // Không báo lỗi nếu không có file khớp
)
```

### Truyền Nhiều Stash Trong Pipeline Phức Tạp

```groovy
pipeline {
    agent none

    stages {
        stage('Compile') {
            agent { docker { image 'maven:3.9-eclipse-temurin-17' } }
            steps {
                sh 'mvn compile'
                stash name: 'compiled-classes', includes: 'target/classes/**'
            }
        }

        stage('Package') {
            agent { docker { image 'maven:3.9-eclipse-temurin-17' } }
            steps {
                unstash 'compiled-classes'
                sh 'mvn package -DskipTests'
                stash name: 'distribution', includes: 'target/myapp-*.jar, target/myapp-*.war'
            }
        }

        stage('Integration Test') {
            agent { label 'integration-server' }
            steps {
                unstash 'distribution'
                sh './run-integration-tests.sh'
                stash name: 'test-reports', includes: 'test-results/**'
            }
        }

        stage('Publish Results') {
            agent { label 'master' }
            steps {
                unstash 'test-reports'
                unstash 'distribution'

                // Publish test results
                junit 'test-results/**/*.xml'

                // Archive final artifacts
                archiveArtifacts artifacts: 'target/myapp-*.jar', fingerprint: true
            }
        }
    }
}
```

### Giới Hạn Stash Cần Biết

```
⚠️ Giới hạn quan trọng của Stash:

1. Kích thước: Mặc định tối đa 100MB mỗi stash
   → Thay đổi: Manage Jenkins → System → Max stash size

2. Phạm vi: Chỉ trong cùng một pipeline run
   → Không dùng được giữa hai pipeline khác nhau

3. Lưu trữ: Trên Jenkins Master (JENKINS_HOME/stashes)
   → Nhiều stash lớn → tốn RAM và ổ đĩa Master

4. Thay thế cho file lớn: Dùng External Storage
   → S3, Artifactory, Nexus tốt hơn cho file >100MB
```

---

## Fingerprint — Theo Dõi Nguồn Gốc Artifact

**Fingerprint** (dấu vân tay) là hash MD5 của artifact, dùng để theo dõi file đó được tạo bởi build nào và được dùng ở job nào.

### Cách Hoạt Động

```
Build Job A → tạo myapp.jar → tính MD5: a1b2c3d4...
                             → lưu vào Jenkins: "a1b2c3d4 = Job A, Build #15"

Deploy Job B → nhận myapp.jar → tính MD5: a1b2c3d4...
                               → Jenkins biết: "File này từ Job A #15"

Sau này:
  Artifact myapp.jar → Jenkins hiển thị:
    "Produced by: Job A, Build #15"
    "Used by: Deploy Job B, Build #3 và #7"
```

### Kích Hoạt Fingerprinting

**Khi archive:**

```groovy
// fingerprint: true → tự động tính và lưu hash
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
```

**Khi copy artifact từ job khác:**

```groovy
stage('Deploy') {
    steps {
        // Copy artifact từ upstream job và fingerprint
        copyArtifacts(
            projectName: 'build-job',
            filter: 'target/*.jar',
            fingerprintArtifacts: true,    // Theo dõi artifact này
            selector: lastSuccessful()      // Lấy từ build thành công gần nhất
        )
    }
}
```

**Fingerprint file thủ công:**

```groovy
stage('Record') {
    steps {
        // Ghi nhận fingerprint mà không archive
        fingerprint 'target/*.jar'
    }
}
```

### Xem Fingerprint trong Jenkins UI

```
1. Job → Build #15 → "Fingerprints" trong sidebar
   → Hiển thị hash MD5 của từng artifact

2. Manage Jenkins → Fingerprints
   → Tìm kiếm theo hash để biết artifact đến từ đâu

3. Job → Build → "Used in:" / "Produced by:"
   → Liên kết giữa các job sử dụng cùng artifact
```

### Ứng Dụng Thực Tế của Fingerprint

```
Câu hỏi: "Version 1.2.3 đang chạy trên production có vấn đề.
          Nó được build từ commit nào? Đã được test ở job nào?"

Trả lời qua Fingerprint:
  myapp-1.2.3.jar
    → Produced by: build-job #47 (commit: abc123def)
    → Used by: integration-test-job #23 ✅
    → Used by: deploy-staging #8 ✅
    → Used by: deploy-production #5 ✅
```

---

## So Sánh Archive vs Stash

| Tiêu Chí | Archive Artifacts | Stash/Unstash |
|----------|-------------------|---------------|
| **Mục đích** | Lưu trữ lâu dài, tải về được | Truyền file tạm thời trong pipeline |
| **Thời gian sống** | Theo Build Discard Policy | Chỉ trong pipeline hiện tại |
| **Phạm vi** | Tất cả người dùng Jenkins có thể tải | Chỉ trong cùng pipeline run |
| **Fingerprint** | Hỗ trợ | Không hỗ trợ |
| **UI Download** | Có (trang Build → Artifacts) | Không |
| **Giới hạn kích thước** | Theo dung lượng ổ đĩa | Mặc định 100MB mỗi stash |
| **Khi nào dùng** | JAR cuối, báo cáo, release binary | Test report tạm, compiled classes giữa stages |

### Quyết Định Dùng Cái Nào

```
File này cần tồn tại sau khi pipeline kết thúc?
├── CÓ → Archive Artifacts
└── KHÔNG → Stash

File này cần dùng trên agent khác trong cùng pipeline?
├── CÓ → Stash + Unstash
└── KHÔNG → Không cần cả hai, dùng workspace cục bộ

File này cần theo dõi nguồn gốc qua nhiều job?
└── Dùng Fingerprint (kết hợp với Archive hoặc riêng)
```

---

## Quản Lý Dung Lượng Artifact

Artifact tích lũy theo thời gian và chiếm nhiều ổ đĩa. Jenkins cung cấp **Build Discard Policy** (chính sách loại bỏ build cũ) để tự động dọn dẹp.

### Cấu Hình trong Jenkinsfile

```groovy
pipeline {
    agent any

    options {
        // Giữ tối đa 10 build gần nhất
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Build') {
            steps { sh 'make build' }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'dist/**'
        }
    }
}
```

### Các Tùy Chọn Build Discard

```groovy
options {
    buildDiscarder(logRotator(
        // Số build tối đa giữ lại (cả log lẫn artifact)
        numToKeepStr: '10',

        // Số ngày giữ build log (0 = không xóa theo thời gian)
        daysToKeepStr: '30',

        // Số build có artifact tối đa giữ lại
        // (giữ artifact ít hơn log — artifact tốn nhiều ổ đĩa hơn)
        artifactNumToKeepStr: '3',

        // Số ngày giữ artifact
        artifactDaysToKeepStr: '7'
    ))
}
```

### Workspace Cleanup — Dọn Dẹp Workspace

Plugin **Workspace Cleanup** xóa workspace sau mỗi build để tiết kiệm ổ đĩa trên agent:

```groovy
pipeline {
    agent any

    options {
        // Xóa workspace trước khi build bắt đầu
        skipDefaultCheckout(true)
    }

    stages {
        stage('Checkout') {
            steps {
                // Xóa workspace sạch trước checkout
                cleanWs()
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }

    post {
        always {
            // Archive trước, rồi mới cleanup
            archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true

            // Xóa workspace sau khi build xong (kể cả fail)
            cleanWs(
                cleanWhenAborted: true,
                cleanWhenFailure: true,
                cleanWhenNotBuilt: false,
                cleanWhenSuccess: true,
                cleanWhenUnstable: true,
                deleteDirs: true
            )
        }
    }
}
```

---

## Tích Hợp Artifact Repository Bên Ngoài

Với các project lớn, nên dùng **Artifact Repository** (kho lưu trữ artifact) chuyên dụng thay vì lưu trực tiếp trên Jenkins:

### Nexus Repository Manager

```groovy
stage('Publish to Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-credentials',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
        )]) {
            sh """
                mvn deploy \
                  -DaltDeploymentRepository=nexus::default::http://nexus.example.com/repository/maven-releases/ \
                  -Dusername=${NEXUS_USER} \
                  -Dpassword=${NEXUS_PASS}
            """
        }
    }
}
```

### JFrog Artifactory

```groovy
stage('Publish to Artifactory') {
    steps {
        rtUpload(
            serverId: 'artifactory-server',    // Khai báo trong Jenkins Global Config
            spec: '''{
                "files": [{
                    "pattern": "target/*.jar",
                    "target": "libs-release-local/com/example/myapp/${params.VERSION}/"
                }]
            }''',
            buildName: env.JOB_NAME,
            buildNumber: env.BUILD_NUMBER
        )

        // Publish build info lên Artifactory
        rtPublishBuildInfo(serverId: 'artifactory-server')
    }
}
```

### Amazon S3

```groovy
stage('Publish to S3') {
    steps {
        withAWS(credentials: 'aws-credentials', region: 'ap-southeast-1') {
            s3Upload(
                bucket: 'my-artifacts-bucket',
                path: "releases/${params.VERSION}/",
                includePathPattern: 'target/*.jar',
                workingDir: '.'
            )
        }
    }
}

// Hoặc dùng AWS CLI
stage('Upload to S3') {
    steps {
        withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-credentials'
        ]]) {
            sh """
                aws s3 cp target/myapp.jar \
                  s3://my-artifacts-bucket/releases/${params.VERSION}/myapp.jar \
                  --region ap-southeast-1
            """
        }
    }
}
```

---

## Best Practices

### 1. Luôn Archive Artifact Quan Trọng Trong Post Block

```groovy
post {
    always {
        // Test report: luôn archive dù pass hay fail (để debug)
        junit allowEmptyResults: true, testResults: 'target/surefire-reports/**/*.xml'
        archiveArtifacts allowEmptyArchive: true, artifacts: 'logs/**'
    }
    success {
        // Build artifact: chỉ archive khi success
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
}
```

### 2. Đặt Tên Artifact Rõ Ràng Kèm Version

```groovy
stage('Package') {
    steps {
        script {
            def version = sh(script: 'git describe --tags --always', returnStdout: true).trim()
            sh "mv target/myapp.jar target/myapp-${version}.jar"
        }
        archiveArtifacts artifacts: "target/myapp-*.jar", fingerprint: true
    }
}
```

### 3. Dùng Fingerprint Cho Audit Trail (Nhật Ký Kiểm Toán)

```groovy
// Build job
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

// Deploy job — copy và fingerprint để Jenkins biết liên kết
copyArtifacts(
    projectName: 'build-job',
    selector: specific(params.BUILD_NUMBER),
    fingerprintArtifacts: true
)
```

### 4. Giới Hạn Thời Gian Sống của Stash

```groovy
// Stash chỉ trong cùng pipeline — đặt timeout để tránh treo
timeout(time: 30, unit: 'MINUTES') {
    unstash 'build-output'
}
```

### 5. Monitor (Theo Dõi) Dung Lượng Artifact

```bash
# Xem dung lượng thư mục artifact trên Jenkins Master
du -sh $JENKINS_HOME/jobs/*/builds/*/archive/

# Tìm job dùng nhiều dung lượng nhất
du -s $JENKINS_HOME/jobs/*/builds/ | sort -n | tail -20
```

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Archive Artifacts và Stash trong Jenkins?**

> **Archive Artifacts** lưu file vào Jenkins Master vĩnh viễn (theo Build Discard Policy), cho phép người dùng tải về qua UI hoặc API, và hỗ trợ Fingerprint để theo dõi nguồn gốc. **Stash** lưu file tạm thời trong RAM/disk của Master, chỉ tồn tại trong một pipeline run, không tải về được, dùng để truyền file giữa các stage hoặc giữa các agent trong cùng pipeline. Quy tắc đơn giản: dùng Stash cho trung gian trong pipeline, Archive cho kết quả cuối cùng.

**Q: Fingerprint trong Jenkins là gì và tại sao cần dùng?**

> Fingerprint là hash MD5 của artifact, cho phép Jenkins theo dõi "vòng đời" của file: được tạo bởi job nào, build nào, và được tiêu thụ bởi job nào. Cần dùng khi: (1) cần audit trail — truy xuất file đang chạy production về commit và build cụ thể; (2) cần biết impact — phiên bản library nào đang được dùng ở bao nhiêu job; (3) compliance — hệ thống tài chính/y tế cần chứng minh artifact không bị thay đổi giữa test và production.

**Q: Khi nào nên dùng external artifact repository (Nexus, Artifactory, S3) thay vì lưu trực tiếp trên Jenkins?**

> Nên dùng external repository khi: **(1) Nhiều team cần truy cập artifact** — Jenkins artifact chỉ tải được qua UI/API Jenkins; **(2) Cần versioning và metadata phong phú** — Nexus/Artifactory hỗ trợ semantic versioning, checksum, dependency graph; **(3) Lưu lượng lớn** — Jenkins không được thiết kế làm artifact store chính; **(4) Retention policy phức tạp** — giữ release builds mãi, giữ snapshot 30 ngày; **(5) Integration với build tools** — Maven, Gradle có thể resolve dependency trực tiếp từ Nexus/Artifactory. Jenkins lưu artifact là giải pháp đơn giản cho team nhỏ; enterprise dùng Nexus/Artifactory.

**Q: Làm thế nào để đảm bảo artifact lưu trữ không chiếm hết ổ đĩa Jenkins?**

> Ba lớp kiểm soát: **(1) Build Discard Policy** — đặt `buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '3'))` trong mỗi Jenkinsfile, giữ artifact ít hơn log vì artifact tốn nhiều ổ đĩa hơn; **(2) Workspace Cleanup Plugin** — gọi `cleanWs()` sau mỗi build để xóa workspace trên agent; **(3) External Storage** — với artifact lớn hoặc cần retention dài, lưu lên S3/Nexus thay vì Jenkins, để Jenkins chỉ lưu pointer/reference. Monitoring: định kỳ kiểm tra `du -sh $JENKINS_HOME/jobs/` để phát hiện job dùng nhiều ổ đĩa.

---

**Liên Kết Liên Quan:**
- [3-build-parameters.md](3-build-parameters.md) — Tham số hóa build
- [README.md](README.md) — Tổng quan Build Triggers
- `09-monitoring-maintenance/disk-management.md` — Quản lý ổ đĩa Jenkins

**Cập Nhật:** 2026-05-10
