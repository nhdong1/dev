# Build Failures — Phân Tích và Khắc Phục Lỗi Build

> Hướng dẫn có hệ thống để chẩn đoán build failure (lỗi build): từ đọc log, hiểu exit code, xử lý biến môi trường, đến các lỗi script thường gặp.

## Mục Lục

1. [Phương Pháp Đọc Build Log](#phương-pháp-đọc-build-log)
2. [Exit Code — Mã Thoát](#exit-code--mã-thoát)
3. [Lỗi Environment Variable](#lỗi-environment-variable)
4. [Lỗi Workspace và Permissions](#lỗi-workspace-và-permissions)
5. [Lỗi Script Shell Thường Gặp](#lỗi-script-shell-thường-gặp)
6. [Lỗi SCM — Không Clone Được Code](#lỗi-scm--không-clone-được-code)
7. [Lỗi Timeout](#lỗi-timeout)
8. [Kỹ Thuật Debug Nâng Cao](#kỹ-thuật-debug-nâng-cao)
9. [Checklist Xử Lý Build Failure](#checklist-xử-lý-build-failure)

---

## Phương Pháp Đọc Build Log

### Nguyên Tắc: Đọc Từ Cuối Lên

Build log (nhật ký build) thường dài hàng trăm dòng. Lỗi thực sự thường nằm ở **cuối log** — không phải đầu.

```
[Pipeline] End of Pipeline
ERROR: script returned exit code 1       ← dòng này cho biết lệnh nào thất bại
Finished: FAILURE
```

### Tìm Từ Khóa Trong Log

Dùng tính năng **Search** trong Blue Ocean hoặc Classic UI:

| Từ Khóa Tìm Kiếm | Ý Nghĩa |
|------------------|---------|
| `ERROR:` | Lỗi Jenkins pipeline |
| `error:` / `Error:` | Lỗi từ tool (Maven, npm, Docker...) |
| `FAILED` | Test thất bại hoặc build thất bại |
| `Exception` | Java/Groovy exception trong pipeline |
| `exit code` | Mã thoát của lệnh shell |
| `Cannot find` / `No such file` | File hoặc lệnh không tồn tại |
| `Permission denied` | Lỗi phân quyền |
| `Connection refused` | Lỗi kết nối mạng |

### Xem Log Chi Tiết

```groovy
// Bật Pipeline logging chi tiết — thêm vào Jenkinsfile
pipeline {
    options {
        // Giữ tối đa 10 build logs để tránh tốn disk
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage('Debug Info') {
            steps {
                // In tất cả environment variables (biến môi trường)
                sh 'env | sort'
                // In thông tin workspace (không gian làm việc)
                sh 'pwd && ls -la'
            }
        }
    }
}
```

---

## Exit Code — Mã Thoát

### Hiểu Exit Code

Exit code (mã thoát) là số nguyên mà một process trả về khi kết thúc:
- **0** = Thành công (success)
- **Khác 0** = Thất bại (failure)

Jenkins tự động fail build khi lệnh `sh` hoặc `bat` trả về exit code khác 0.

### Các Exit Code Phổ Biến

| Exit Code | Nguyên Nhân Thường Gặp | Cách Xử Lý |
|-----------|------------------------|------------|
| `1` | Lỗi generic, lệnh thất bại | Đọc stderr output phía trên |
| `2` | Lỗi cú pháp (syntax error) | Kiểm tra cú pháp lệnh |
| `126` | Lệnh tồn tại nhưng không có quyền thực thi | `chmod +x script.sh` |
| `127` | Lệnh không tìm thấy (command not found) | Cài tool, kiểm tra PATH |
| `128+N` | Process bị kill bởi signal N | `kill -9` → exit 137 |
| `137` | OOM Kill — bị kill vì hết RAM | Tăng memory limit |
| `143` | SIGTERM — bị dừng gracefully | Timeout hoặc manual stop |

### Ví Dụ Thực Tế

```bash
# Lỗi exit code 127 — lệnh không tồn tại
+ mvn clean package
/bin/sh: mvn: not found
script returned exit code 127

# Giải pháp: cài Maven trong agent, hoặc dùng tool directive
```

```groovy
// Cấu hình tool trong pipeline — tránh "command not found"
pipeline {
    tools {
        // Maven phải được cấu hình trong Manage Jenkins → Tools
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
    }
}
```

### Bắt Exit Code Thủ Công

```groovy
pipeline {
    stages {
        stage('Test') {
            steps {
                script {
                    // returnStatus: true — không fail ngay, lưu exit code vào biến
                    def exitCode = sh(
                        script: 'npm test',
                        returnStatus: true
                    )
                    if (exitCode != 0) {
                        echo "Tests failed with exit code: ${exitCode}"
                        // Có thể archive log trước khi fail
                        archiveArtifacts artifacts: 'test-results/**'
                        error("Tests failed!")
                    }
                }
            }
        }
    }
}
```

---

## Lỗi Environment Variable

### Vấn Đề Phổ Biến

#### 1. Biến Không Tồn Tại (Undefined Variable)

```bash
# Lỗi trong log
/bin/sh: DOCKER_REGISTRY: unbound variable
# hoặc
+ docker push /myimage:latest   ← thiếu tên registry
```

**Nguyên nhân:** Biến môi trường chưa được định nghĩa hoặc tên sai.

```groovy
// Kiểm tra biến trước khi dùng
pipeline {
    stages {
        stage('Validate Env') {
            steps {
                script {
                    // Fail sớm nếu biến bắt buộc chưa có
                    ['DOCKER_REGISTRY', 'IMAGE_TAG', 'DEPLOY_ENV'].each { varName ->
                        if (!env[varName]) {
                            error("Required environment variable '${varName}' is not set")
                        }
                    }
                }
            }
        }
    }
}
```

#### 2. Credentials Không Được Bind Đúng

```bash
# Lỗi khi dùng withCredentials sai
ERROR: No such credential: my-docker-creds
```

**Kiểm tra:**
- Credentials ID có đúng không (phân biệt hoa/thường)
- Credentials đã được tạo trong Jenkins chưa (Manage Jenkins → Credentials)
- Scope của Credentials (Global vs System vs folder-level)

```groovy
// Đúng cách dùng withCredentials — Binding Credentials vào biến môi trường
pipeline {
    stages {
        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',  // phải khớp với ID trong Jenkins
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push myimage:latest
                    '''
                }
            }
        }
    }
}
```

#### 3. Biến Bị Expand Sai Trong Shell

```bash
# Lỗi: Groovy string interpolation trong sh block
sh "echo ${MY_VAR}"   # ← Groovy expand trước, shell không thấy biến
sh 'echo $MY_VAR'     # ← Shell expand, đúng cách
```

**Quy tắc:** Dùng **single quotes** `'...'` khi muốn shell tự xử lý biến. Dùng **double quotes** `"..."` khi cần Groovy expand trước.

---

## Lỗi Workspace và Permissions

### Workspace (Không Gian Làm Việc) Bị Dơ

**Triệu chứng:**
```
ERROR: Directory not empty
error: Your local changes to the following files would be overwritten by checkout
```

**Nguyên nhân:** Build trước để lại file trong workspace, gây xung đột khi checkout (lấy code mới).

```groovy
// Giải pháp 1: Clean workspace trước build
pipeline {
    options {
        // Tự động xóa workspace trước mỗi build
        skipDefaultCheckout(true)
    }
    stages {
        stage('Checkout') {
            steps {
                cleanWs()  // Xóa workspace (yêu cầu Workspace Cleanup Plugin)
                checkout scm
            }
        }
    }
}
```

```groovy
// Giải pháp 2: git clean trong checkout
pipeline {
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [
                        // Xóa các file không được track bởi git
                        [$class: 'CleanBeforeCheckout'],
                        // Xóa cả các file trong .gitignore
                        [$class: 'CleanCheckout', deleteUntrackedNestedRepositories: true]
                    ],
                    userRemoteConfigs: [[url: 'https://github.com/org/repo.git']]
                ])
            }
        }
    }
}
```

### Lỗi Permission Denied

```bash
# Lỗi phổ biến
/bin/sh: ./gradlew: Permission denied
script returned exit code 126
```

**Nguyên nhân:** File `gradlew` (hoặc script khác) không có quyền thực thi sau khi checkout.

```groovy
// Giải pháp: chmod trước khi chạy
steps {
    sh 'chmod +x gradlew'
    sh './gradlew build'
}
```

**Cách tốt hơn:** Đảm bảo quyền thực thi được commit vào git:
```bash
git update-index --chmod=+x gradlew
git commit -m "fix: add execute permission to gradlew"
```

---

## Lỗi Script Shell Thường Gặp

### Pipefail — Lỗi Trong Pipe Bị Bỏ Qua

**Vấn đề:** Mặc định shell chỉ kiểm tra exit code của lệnh **cuối cùng** trong pipe:

```bash
# Lệnh này KHÔNG fail dù grep thất bại
sh 'failing-command | grep something'
# grep thành công (exit 0) → Jenkins không biết failing-command đã lỗi
```

**Giải pháp:** Bật `pipefail`:

```groovy
steps {
    sh '''
        set -euo pipefail
        # -e: exit ngay khi có lỗi
        # -u: treat unset variables as error
        # -o pipefail: fail nếu bất kỳ lệnh nào trong pipe thất bại
        
        mvn clean package | tee build.log
    '''
}
```

### Multi-line Script Xử Lý Lỗi

```groovy
steps {
    sh '''
        set -e  # Dừng ngay khi có lệnh thất bại

        echo "Starting build..."
        mvn clean package

        echo "Running tests..."
        mvn test

        echo "Build complete."
    '''
}
```

### Lệnh Không Tìm Thấy Trong Docker Agent

```bash
# Lỗi
/bin/sh: mvn: not found
```

**Nguyên nhân:** Docker image không có Maven được cài sẵn.

```groovy
// Giải pháp: chỉ định image có sẵn tool
pipeline {
    agent {
        docker {
            // Dùng image Maven chính thức — đã có Java và Maven
            image 'maven:3.9-eclipse-temurin-17'
            // Mount Maven cache để tránh download lại mỗi build
            args '-v /var/jenkins_home/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

---

## Lỗi SCM — Không Clone Được Code

### Lỗi SSH Authentication

```bash
ERROR: Failed to connect to repository
Host key verification failed.
# hoặc
Permission denied (publickey).
```

**Chẩn đoán từng bước:**

```bash
# Trên agent, kiểm tra SSH key
ssh -i /var/jenkins_home/.ssh/id_rsa -T git@github.com
# Kết quả mong muốn: Hi username! You've successfully authenticated...

# Kiểm tra known_hosts (danh sách host đã biết)
cat /var/jenkins_home/.ssh/known_hosts | grep github.com
```

**Giải pháp:**

```groovy
// Dùng SSH Agent Plugin để inject SSH key đúng cách
pipeline {
    stages {
        stage('Checkout') {
            steps {
                sshagent(credentials: ['github-deploy-key']) {
                    sh 'git clone git@github.com:org/repo.git'
                }
            }
        }
    }
}
```

### Lỗi HTTPS Token Hết Hạn

```bash
remote: HTTP Basic: Access denied
fatal: Authentication failed for 'https://github.com/...'
```

**Giải pháp:**
1. Tạo Personal Access Token (PAT — mã token truy cập cá nhân) mới trên GitHub
2. Cập nhật Credentials trong Jenkins (Manage Jenkins → Credentials)
3. Không cần sửa Jenkinsfile — pipeline tự dùng credentials mới

---

## Lỗi Timeout

### Build Timeout

```bash
ERROR: Timeout of 30 minutes exceeded
Finished: ABORTED
```

**Cấu hình timeout hợp lý:**

```groovy
pipeline {
    options {
        // Timeout toàn bộ pipeline
        timeout(time: 60, unit: 'MINUTES')
    }
    stages {
        stage('Integration Tests') {
            options {
                // Timeout riêng cho stage này
                timeout(time: 30, unit: 'MINUTES')
            }
            steps {
                sh 'mvn verify -Pintegration-test'
            }
        }
    }
}
```

### Checkout Timeout Do Repo Quá Lớn

```bash
ERROR: Timeout after 10 minutes
org.eclipse.jgit.api.errors.TransportException: Read timed out
```

**Giải pháp:**

```groovy
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    extensions: [
        // Shallow clone — chỉ lấy commit mới nhất, không lấy toàn bộ history
        [$class: 'CloneOption',
         depth: 1,          // Chỉ lấy 1 commit gần nhất
         shallow: true,
         timeout: 30]       // Timeout 30 phút
    ],
    userRemoteConfigs: [[url: repoUrl, credentialsId: 'git-creds']]
])
```

---

## Kỹ Thuật Debug Nâng Cao

### Replay Build — Chạy Lại Với Pipeline Sửa

Jenkins có tính năng **Replay** cho phép chỉnh sửa Jenkinsfile và chạy lại mà không cần commit:

```
Build #42 → More → Replay → Sửa Jenkinsfile → Run
```

Dùng khi: muốn test fix nhanh trước khi commit vào repo.

### Thêm Debug Steps Tạm Thời

```groovy
stage('Debug') {
    when {
        // Chỉ chạy khi build parameter DEBUG=true
        environment name: 'DEBUG', value: 'true'
    }
    steps {
        sh 'env | sort'           // In tất cả biến môi trường
        sh 'df -h'               // Kiểm tra disk
        sh 'free -m'             // Kiểm tra RAM
        sh 'id && whoami'        // Xem user đang chạy build
        sh 'ls -la workspace/'   // Xem cấu trúc thư mục
    }
}
```

### Script Console — Chạy Groovy Trực Tiếp

```groovy
// Manage Jenkins → Script Console

// Tìm tất cả builds đang chạy
Jenkins.instance.getAllItems(Job.class).each { job ->
    job.builds.each { build ->
        if (build.isBuilding()) {
            println "${job.name} #${build.number} started: ${build.timestampString}"
        }
    }
}

// Xem thông tin chi tiết về một build cụ thể
def build = Jenkins.instance.getItemByFullName('my-job').getBuildByNumber(42)
println "Result: ${build.result}"
println "Duration: ${build.durationString}"
println "Cause: ${build.causes}"
```

---

## Checklist Xử Lý Build Failure

Khi gặp build failure, làm theo thứ tự sau:

```
□ 1. Đọc dòng lỗi cuối cùng trong log (cuộn xuống cuối)
□ 2. Tìm từ khóa: ERROR, Exception, exit code, Permission denied
□ 3. So sánh với build trước đó thành công — có gì thay đổi?
□ 4. Kiểm tra: code thay đổi, agent thay đổi, env thay đổi, plugin cập nhật
□ 5. Tái hiện lỗi: Replay build hoặc trigger build mới
□ 6. Kiểm tra disk space trên agent (df -h)
□ 7. Kiểm tra network connectivity từ agent đến external services
□ 8. Đọc Jenkins System Log nếu lỗi ở tầng infrastructure
□ 9. Hỏi đồng nghiệp hoặc search lỗi cụ thể trên Stack Overflow / Jenkins Issues
□ 10. Ghi lại giải pháp vào runbook (sổ tay vận hành) để dùng lại
```

---

**Xem Tiếp:** [2-agent-issues.md](2-agent-issues.md) — xử lý sự cố kết nối agent
