# SonarQube Integration — Kiểm Tra Chất Lượng Mã Nguồn Từ Jenkins

> SonarQube Integration cho phép Jenkins tự động phân tích chất lượng mã nguồn (static code analysis — phân tích tĩnh) và áp dụng Quality Gate (cổng chất lượng) — nếu mã không đạt tiêu chuẩn, pipeline sẽ dừng lại và báo lỗi.

---

## Mục Lục

1. [Tổng Quan Kiến Trúc](#1-tổng-quan-kiến-trúc)
2. [Cài Đặt Và Cấu Hình](#2-cài-đặt-và-cấu-hình)
3. [SonarQube Scanner Trong Pipeline](#3-sonarqube-scanner-trong-pipeline)
4. [Quality Gate (Cổng Chất Lượng)](#4-quality-gate-cổng-chất-lượng)
5. [Code Coverage (Độ Phủ Code)](#5-code-coverage-độ-phủ-code)
6. [Branch Analysis và PR Decoration](#6-branch-analysis-và-pr-decoration)
7. [Sonar Properties Cấu Hình](#7-sonar-properties-cấu-hình)
8. [Best Practices](#8-best-practices)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Kiến Trúc

```
SonarQube trong Jenkins Pipeline
──────────────────────────────────

Source Code (Java/Python/JS/...)
     │
     │  SonarQube Scanner phân tích
     ▼
SonarQube Server (lưu trữ và hiển thị kết quả)
     │
     │  Jenkins hỏi: "Quality Gate có pass không?"
     ▼
Quality Gate Result
     │
     ├── PASSED ─► Pipeline tiếp tục (build image, deploy...)
     └── FAILED ─► Pipeline dừng lại, báo lỗi


Các loại vấn đề SonarQube phát hiện:
  🐛 Bug (lỗi)              — code sẽ gây lỗi runtime
  🔒 Vulnerability (lỗ hổng) — rủi ro bảo mật
  💡 Code Smell (mùi code)  — code khó bảo trì
  📋 Duplication (trùng lặp) — % code bị copy-paste
  📊 Coverage (độ phủ)      — % code được test bởi unit test
```

---

## 2. Cài Đặt Và Cấu Hình

### 2.1 Cài Plugin

```
Manage Jenkins → Plugin Manager → Available plugins
→ Cài: "SonarQube Scanner for Jenkins"
```

### 2.2 Tạo SonarQube Token

```
SonarQube Server → My Account → Security → Generate Tokens
  Name: jenkins-scanner
  Type: Global Analysis Token
→ Generate → Copy token (chỉ hiển thị một lần)
```

### 2.3 Lưu Token Vào Jenkins Credentials

```
Manage Jenkins → Credentials → System → Global credentials
→ Add Credentials:
   Kind: Secret text
   Secret: <SonarQube token>
   ID: sonarqube-token
   Description: SonarQube Analysis Token
```

### 2.4 Cấu Hình SonarQube Server Trong Jenkins

```
Manage Jenkins → Configure System → SonarQube servers
→ Add SonarQube:
   Name:            SonarQube
   Server URL:      http://sonarqube.mycompany.com:9000
   Server auth token: sonarqube-token (chọn từ dropdown)
→ Save
```

### 2.5 Cấu Hình SonarQube Scanner Tool

```
Manage Jenkins → Global Tool Configuration → SonarQube Scanner
→ Add SonarQube Scanner:
   Name:             SonarQube Scanner 5.x
   Install automatically: ✅ (Jenkins tự tải về)
   Version:          SonarQube Scanner 5.0.1.3006
```

---

## 3. SonarQube Scanner Trong Pipeline

### 3.1 Declarative Pipeline — Cơ Bản

```groovy
pipeline {
    agent any

    // Khai báo SonarQube environment
    environment {
        SONAR_PROJECT_KEY = 'myapp'
        SONAR_PROJECT_NAME = 'My Application'
    }

    stages {
        stage('Test') {
            steps {
                // Chạy test để tạo coverage report trước khi scan
                sh 'mvn clean verify -Dmaven.test.failure.ignore=true'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // withSonarQubeEnv() — inject các biến môi trường SonarQube tự động
                withSonarQubeEnv('SonarQube') {
                    // Dùng Maven plugin — tích hợp sẵn, không cần cài thêm
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName='${SONAR_PROJECT_NAME}' \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // Chờ SonarQube xử lý xong rồi kiểm tra kết quả
                timeout(time: 5, unit: 'MINUTES') {
                    // waitForQualityGate() — Jenkins poll SonarQube Webhook
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to Quality Gate failure: ${qg.status}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            // Stage này chỉ chạy nếu Quality Gate PASS
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
    }
}
```

### 3.2 SonarQube Scanner — Dự Án Không Dùng Maven

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            // Dùng SonarQube Scanner CLI trực tiếp — cho Python, JS, Go, v.v.
            def scannerHome = tool 'SonarQube Scanner 5.x'
            sh """
                ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=myapp \
                    -Dsonar.projectName='My Application' \
                    -Dsonar.sources=src/ \
                    -Dsonar.tests=tests/ \
                    -Dsonar.python.coverage.reportPaths=coverage.xml \
                    -Dsonar.python.version=3.11 \
                    -Dsonar.sourceEncoding=UTF-8 \
                    -Dsonar.branch.name=${env.BRANCH_NAME}
            """
        }
    }
}
```

### 3.3 JavaScript / TypeScript Project

```groovy
stage('SonarQube Analysis') {
    steps {
        // Chạy test với coverage trước
        sh 'npm test -- --coverage --coverageReporters=lcov'

        withSonarQubeEnv('SonarQube') {
            def scannerHome = tool 'SonarQube Scanner 5.x'
            sh """
                ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=myapp-frontend \
                    -Dsonar.sources=src \
                    -Dsonar.tests=src \
                    -Dsonar.test.inclusions=**/*.test.ts,**/*.spec.ts \
                    -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                    -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info
            """
        }
    }
}
```

### 3.4 Go Project

```groovy
stage('SonarQube Analysis') {
    steps {
        // Tạo coverage report theo định dạng SonarQube chấp nhận
        sh '''
            go test ./... -coverprofile=coverage.out
            go test ./... -json > test-report.json
            gocover-cobertura < coverage.out > coverage.xml
        '''

        withSonarQubeEnv('SonarQube') {
            def scannerHome = tool 'SonarQube Scanner 5.x'
            sh """
                ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=myapp-backend \
                    -Dsonar.sources=. \
                    -Dsonar.exclusions=vendor/**,**/*_test.go \
                    -Dsonar.go.coverage.reportPaths=coverage.out \
                    -Dsonar.go.tests.reportPaths=test-report.json
            """
        }
    }
}
```

---

## 4. Quality Gate (Cổng Chất Lượng)

### 4.1 Cấu Hình SonarQube Webhook

Để `waitForQualityGate()` hoạt động, SonarQube phải gửi webhook về Jenkins khi phân tích xong:

```
SonarQube Server → Administration → Configuration → Webhooks
→ Create:
   Name:   Jenkins
   URL:    http://<jenkins-url>/sonarqube-webhook/
   Secret: <chuỗi bí mật tùy chọn>
→ Save
```

### 4.2 Định Nghĩa Quality Gate

Quality Gate là tập hợp các điều kiện mà mã nguồn phải thỏa mãn:

```
SonarQube → Quality Gates → Create
Điều kiện mẫu cho Quality Gate "Production Ready":

  Metric (Chỉ số)                   Operator  Value
  ──────────────────────────────── ─────────  ─────
  Coverage on New Code              is less than  80%
  Duplicated Lines on New Code      is greater than  3%
  Maintainability Rating on New Code  is worse than  A
  Reliability Rating on New Code    is worse than  A
  Security Rating on New Code       is worse than  A
  Security Hotspots Reviewed        is less than  100%
```

### 4.3 Pipeline Với Quality Gate Chi Tiết

```groovy
stage('Quality Gate Check') {
    steps {
        script {
            timeout(time: 10, unit: 'MINUTES') {
                def qg = waitForQualityGate()

                // Xử lý từng trạng thái Quality Gate
                switch (qg.status) {
                    case 'OK':
                        echo "✅ Quality Gate PASSED"
                        break
                    case 'WARN':
                        // WARN — không fail pipeline nhưng đánh dấu UNSTABLE
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Quality Gate has WARNINGS — check SonarQube dashboard"
                        break
                    case 'ERROR':
                        // Fail pipeline nếu Quality Gate không pass
                        error "❌ Quality Gate FAILED — pipeline aborted. Check: ${env.SONAR_HOST_URL}"
                        break
                    case 'NONE':
                        echo "ℹ️ Quality Gate not configured — skipping"
                        break
                    default:
                        error "Unknown Quality Gate status: ${qg.status}"
                }
            }
        }
    }
}
```

### 4.4 Quality Gate Không Block Pipeline (Chế Độ Cảnh Báo)

```groovy
stage('Quality Gate (Warning Only)') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            script {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    // Chỉ cảnh báo, không block — dùng cho giai đoạn migration
                    unstable("Quality Gate ${qg.status} — code quality issues detected")
                    slackSend(channel: '#code-quality',
                              color: 'warning',
                              message: "⚠️ SonarQube Quality Gate ${qg.status}: ${env.JOB_NAME}#${env.BUILD_NUMBER}\n" +
                                       "Check: ${env.SONAR_HOST_URL}/dashboard?id=${SONAR_PROJECT_KEY}")
                }
            }
        }
    }
}
```

---

## 5. Code Coverage (Độ Phủ Code)

### 5.1 Coverage Với JaCoCo (Java)

JaCoCo — Java Code Coverage library (thư viện đo độ phủ code Java):

```xml
<!-- pom.xml — thêm JaCoCo plugin -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```groovy
stage('Test with Coverage') {
    steps {
        sh 'mvn clean verify'  // JaCoCo tạo report tại target/site/jacoco/
    }
    post {
        always {
            // Publish JaCoCo coverage report lên Jenkins UI
            jacoco(
                execPattern: 'target/*.exec',
                classPattern: 'target/classes',
                sourcePattern: 'src/main/java',
                exclusionPattern: '**/test/**',
                minimumInstructionCoverage: '80',
                minimumBranchCoverage: '70'
            )
        }
    }
}
```

### 5.2 Coverage Với pytest-cov (Python)

```groovy
stage('Test with Coverage') {
    steps {
        sh '''
            pip install pytest pytest-cov
            pytest tests/ \
                --cov=src \
                --cov-report=xml:coverage.xml \
                --cov-report=html:coverage-html \
                --junitxml=test-results.xml
        '''
    }
    post {
        always {
            junit 'test-results.xml'
            publishHTML([
                reportDir: 'coverage-html',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
            ])
        }
    }
}
```

### 5.3 Coverage Với Istanbul/NYC (JavaScript)

```groovy
stage('Test with Coverage') {
    steps {
        sh '''
            npm install
            npm test -- --coverage \
                --coverageReporters=lcov \
                --coverageReporters=cobertura \
                --coverageDirectory=coverage
        '''
    }
}
```

---

## 6. Branch Analysis và PR Decoration

### 6.1 Branch Analysis (Phân Tích Theo Nhánh)

```groovy
// Tự động phát hiện tên nhánh và truyền vào SonarQube
withSonarQubeEnv('SonarQube') {
    sh """
        mvn sonar:sonar \
            -Dsonar.projectKey=myapp \
            -Dsonar.branch.name=${env.BRANCH_NAME}
    """
}
```

### 6.2 Pull Request Decoration (Trang Trí Pull Request)

PR Decoration — SonarQube gắn kết quả phân tích trực tiếp vào PR trên GitHub/GitLab:

```groovy
stage('SonarQube Analysis') {
    when {
        // Chỉ chạy PR analysis khi build từ Pull Request
        changeRequest()
    }
    steps {
        withSonarQubeEnv('SonarQube') {
            sh """
                mvn sonar:sonar \
                    -Dsonar.projectKey=myapp \
                    -Dsonar.pullrequest.key=${env.CHANGE_ID} \
                    -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} \
                    -Dsonar.pullrequest.base=${env.CHANGE_TARGET} \
                    -Dsonar.pullrequest.github.repository=myorg/myapp \
                    -Dsonar.pullrequest.provider=GitHub
            """
        }
    }
}
```

---

## 7. Sonar Properties Cấu Hình

### 7.1 sonar-project.properties

Thay vì truyền tất cả qua command line, tạo file `sonar-project.properties` trong root project:

```properties
# sonar-project.properties
sonar.projectKey=myapp
sonar.projectName=My Application
sonar.projectVersion=1.0

# Source code
sonar.sources=src/main
sonar.tests=src/test
sonar.java.binaries=target/classes

# Coverage
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Exclusions (loại trừ) — không phân tích những file này
sonar.exclusions=\
    **/generated/**,\
    **/test/**,\
    **/*.min.js,\
    **/node_modules/**,\
    **/vendor/**

# Test inclusions
sonar.test.inclusions=**/*Test.java,**/*Spec.java

# Encoding
sonar.sourceEncoding=UTF-8
```

### 7.2 Cấu Hình Qua Maven pom.xml

```xml
<properties>
    <!-- SonarQube properties trong pom.xml -->
    <sonar.projectKey>myapp</sonar.projectKey>
    <sonar.projectName>My Application</sonar.projectName>
    <sonar.coverage.jacoco.xmlReportPaths>
        ${project.build.directory}/site/jacoco/jacoco.xml
    </sonar.coverage.jacoco.xmlReportPaths>
    <sonar.exclusions>**/generated/**,**/*.min.js</sonar.exclusions>
</properties>
```

---

## 8. Best Practices

### Tích Hợp Vào Pipeline Đúng Vị Trí

```groovy
// ✅ Đúng thứ tự: Test trước → SonarQube Scanner → Quality Gate → Build/Deploy
stages:
  1. Checkout
  2. Build
  3. Unit Tests (tạo coverage report)
  4. SonarQube Analysis (dùng coverage report từ bước 3)
  5. Quality Gate  ← đây là "cổng", nếu fail → dừng lại
  6. Docker Build
  7. Deploy

// ❌ Sai: Deploy trước rồi mới kiểm tra Quality Gate
```

### Không Block Pipeline Khi Mới Tích Hợp

```groovy
// Khi lần đầu tích hợp SonarQube vào dự án đã có sẵn
// → Chạy ở chế độ warning trước, sau khi fix issues xong mới enforce

stage('SonarQube (Warning Mode)') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn sonar:sonar -Dsonar.projectKey=myapp'
        }
        timeout(time: 5, unit: 'MINUTES') {
            script {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    // unstable thay vì error — không block, chỉ cảnh báo
                    unstable("Quality Gate ${qg.status}")
                }
            }
        }
    }
}
```

### Phân Tích Chỉ Code Mới (New Code Analysis)

```
SonarQube → Project Settings → New Code Definition
  Chọn: Previous version HOẶC Number of days (30 days)

Lợi ích:
  - Quality Gate chỉ áp dụng cho code MỚI được thêm vào
  - Team không bị blocked bởi technical debt (nợ kỹ thuật) cũ
  - Có thể set tiêu chuẩn cao hơn cho code mới
```

### Sonar Scan Cache (Bộ Nhớ Đệm)

```groovy
// Cache SonarQube scanner data để tăng tốc scan lần sau
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh """
                mvn sonar:sonar \
                    -Dsonar.working.directory=${WORKSPACE}/.sonar \
                    -Dsonar.projectKey=myapp
            """
        }
    }
}

// Lưu cache vào workspace (tránh tải lại dữ liệu scanner)
// Dùng với Jenkins Workspace Stash hoặc shared volume cho agent
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Quality Gate trong SonarQube là gì và tại sao nó quan trọng?**

> Quality Gate là bộ điều kiện (conditions) định nghĩa ngưỡng chấp nhận được cho chất lượng mã — ví dụ: coverage >= 80%, không có critical bug mới, security rating >= A. Nó quan trọng vì đây là cơ chế tự động ngăn chặn (gating) — pipeline sẽ fail và không deploy nếu mã không đạt tiêu chuẩn, thay thế cho việc review thủ công. Nhờ đó, chất lượng code được đảm bảo một cách nhất quán, không phụ thuộc vào con người.

**Q: Sự khác nhau giữa Bug, Vulnerability, và Code Smell trong SonarQube?**

> **Bug** — code sẽ chắc chắn gây ra lỗi runtime hoặc behavior không đúng (ví dụ: null pointer dereference). **Vulnerability** — điểm yếu bảo mật có thể bị khai thác (ví dụ: SQL injection, XSS). **Code Smell** — code đúng về mặt runtime nhưng khó bảo trì, làm tăng technical debt (nợ kỹ thuật) — ví dụ: hàm quá dài, tên biến không rõ ràng, logic lặp lại. Quality Gate thường enforce Bug + Vulnerability cứng rắn hơn Code Smell.

**Q: Tại sao `waitForQualityGate()` cần SonarQube Webhook?**

> `waitForQualityGate()` không polling — nó đợi SonarQube gửi thông báo về Jenkins qua Webhook khi phân tích xong. Nếu không có Webhook, hàm này sẽ chờ mãi (timeout). Webhook cần cấu hình trỏ vào `http://<jenkins-url>/sonarqube-webhook/` trong SonarQube Administration. Cơ chế này hiệu quả hơn polling vì Jenkins không tốn tài nguyên liên tục kiểm tra trạng thái.

**Q: Làm thế nào để áp dụng SonarQube vào project legacy có nhiều issue?**

> Chiến lược hai giai đoạn: (1) Tích hợp ở chế độ warning — dùng `unstable()` thay vì `error()`, không block pipeline, để team thấy được tình trạng; (2) Cấu hình "New Code" mode — Quality Gate chỉ áp dụng cho code mới, không phải toàn bộ codebase. Song song đó, tạo SonarQube issues backlog và dần dần fix technical debt theo sprint. Sau khi code mới đạt chuẩn ổn định, chuyển sang chế độ enforce (dùng `error()`) và bắt đầu tackle legacy issues.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
**Chủ Đề Tiếp Theo:** [../08-shared-libraries/README.md](../08-shared-libraries/README.md)
