# 3 — Best Practices: Versioning, Testing và Tài Liệu Hóa

> Một Shared Library tốt không chỉ hoạt động đúng — mà còn phải **dễ bảo trì**, **an toàn khi nâng cấp**, **được kiểm thử** và **được tài liệu hóa** đầy đủ. Bài này tập trung vào các thực hành vận hành thực tế giúp Shared Library tồn tại lâu dài trong môi trường production (môi trường thực).

---

## Mục Tiêu

- Thiết kế chiến lược versioning (quản lý phiên bản) an toàn cho library
- Kiểm thử library với `JenkinsPipelineUnit` (framework kiểm thử pipeline) mà không cần Jenkins thật
- Thiết lập CI/CD (tích hợp và triển khai liên tục) cho chính library
- Viết tài liệu rõ ràng và có thể tự động sinh ra
- Xây dựng governance model (mô hình quản trị) cho thư viện dùng chung toàn tổ chức

---

## Phần 1: Versioning — Quản Lý Phiên Bản

### Tại Sao Versioning Quan Trọng?

Nếu không có versioning, một thay đổi trong thư viện có thể phá vỡ **hàng trăm pipeline cùng lúc**:

```
Không có versioning:
  Library (branch: main) ──► Project-A Jenkinsfile (dùng main)
                         ──► Project-B Jenkinsfile (dùng main)
                         ──► Project-C Jenkinsfile (dùng main)
  
  Ai đó merge code lỗi vào main
  → Toàn bộ 3 project đều bị hỏng ngay lập tức!

Có versioning:
  Library v1.0.0 ──► Project-A Jenkinsfile (@v1.0.0) ← Ổn định
  Library v1.1.0 ──► Project-B Jenkinsfile (@v1.1.0) ← Ổn định
  Library main   ──► Project-X (đang phát triển)    ← Có thể lỗi
```

### Chiến Lược Versioning: Semantic Versioning (SemVer — Phiên Bản Ngữ Nghĩa)

Áp dụng **SemVer** (định dạng `MAJOR.MINOR.PATCH`) cho Shared Library:

| Phần | Ý Nghĩa | Ví Dụ |
|------|---------|-------|
| `MAJOR` | Thay đổi breaking (phá vỡ tương thích ngược) | `buildDockerImage(name, tag)` → `buildDockerImage(Map config)` |
| `MINOR` | Thêm tính năng mới, tương thích ngược | Thêm tham số mới có default value |
| `PATCH` | Sửa lỗi, không thay đổi interface | Fix bug trong logic |

```bash
# Tạo tag version trên Git
git tag -a v2.1.0 -m "feat: thêm hỗ trợ Helm 3 cho deployToKubernetes"
git push origin v2.1.0

# Trong Jenkinsfile — dùng version cố định
@Library('company-jenkins-lib@v2.1.0') _
```

### Quy Trình Release (Phát Hành) Library

```
Phát triển tính năng mới
          │
          ▼
    feature/my-feature ──► Pull Request ──► main
          │                    │
          │              Code Review
          │              CI Tests pass
          │
          ▼
    Tạo Release PR: main → release/v2.1.0
          │
          ▼
    CHANGELOG.md + version bump
          │
          ▼
    Tag: git tag v2.1.0 && git push origin v2.1.0
          │
          ▼
    Thông báo cho team: "Library v2.1.0 đã sẵn sàng"
          │
          ▼
    Các project migrate dần từ v2.0.x → v2.1.0
```

### Branch Strategy (Chiến Lược Nhánh) Cho Library

```
main (nhánh chính)
│  ← Luôn ổn định, đã được test
│  ← Không push trực tiếp, chỉ merge qua PR
│
├── feature/add-helm-support    ← Phát triển tính năng
├── feature/improve-slack-step  ← Phát triển tính năng
├── fix/docker-push-retry       ← Sửa lỗi
│
└── Các tag: v1.0.0, v1.1.0, v1.2.0, v2.0.0, v2.1.0
```

### CHANGELOG.md — Nhật Ký Thay Đổi

```markdown
# Changelog

## [v2.1.0] — 2026-05-11

### Added (Thêm Mới)
- `deployHelmChart`: hỗ trợ Helm 3 với `helm upgrade --install`
- `buildDockerImage`: thêm tham số `buildArgs` cho `--build-arg`

### Changed (Thay Đổi)
- `sendSlackNotification`: cải thiện format message, thêm emoji theo trạng thái

### Fixed (Sửa Lỗi)
- `deployToKubernetes`: fix lỗi timeout khi cluster chậm (tăng từ 60s lên 120s)

## [v2.0.0] — 2026-04-01

### Breaking Changes (Thay Đổi Không Tương Thích Ngược)
- `buildDockerImage`: đổi signature từ `(name, tag)` sang `(Map config)`
  - Migration: `buildDockerImage('app', '1.0')` → `buildDockerImage(imageName:'app', tag:'1.0')`

### Added
- `standardCIPipeline`: pipeline skeleton cho toàn tổ chức
- `withDockerRegistry`: custom block để đăng nhập registry
```

---

## Phần 2: Testing Với JenkinsPipelineUnit

### JenkinsPipelineUnit Là Gì?

**JenkinsPipelineUnit** là một framework (khung làm việc) Groovy cho phép kiểm thử Jenkins pipeline và Shared Library **mà không cần Jenkins server thật**. Nó mock (giả lập) tất cả các Jenkins step (`sh`, `echo`, `withCredentials`, v.v.) và cho phép viết unit test chuẩn.

**Lợi ích:**
- Phát hiện lỗi sớm, trước khi merge vào main
- CI/CD cho chính library — test tự động trên mỗi PR
- Tài liệu hóa hành vi mong đợi qua test cases
- Chạy nhanh (vài giây) so với test trên Jenkins thật (vài phút)

### Cài Đặt Môi Trường

**Yêu cầu:** Java 11+, Groovy 2.5+, Gradle hoặc Maven.

```groovy
// build.gradle — file build cho library
plugins {
    id 'groovy'
}

repositories {
    mavenCentral()
}

dependencies {
    // JenkinsPipelineUnit — framework kiểm thử
    testImplementation 'com.lesfurets:jenkins-pipeline-unit:1.21'

    // Spock Framework — DSL (Domain Specific Language) viết test đẹp hơn (tùy chọn)
    testImplementation 'org.spockframework:spock-core:2.3-groovy-3.0'

    // Groovy
    implementation 'org.codehaus.groovy:groovy-all:3.0.17'
}

test {
    useJUnitPlatform()
}

sourceSets {
    main {
        groovy { srcDirs = ['src', 'vars'] }
        resources { srcDirs = ['resources'] }
    }
    test {
        groovy { srcDirs = ['test/groovy'] }
    }
}
```

### Viết Test Cho Global Variable

```groovy
// test/groovy/BuildDockerImageTest.groovy
import com.lesfurets.jenkins.unit.BasePipelineTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import static org.junit.jupiter.api.Assertions.*

class BuildDockerImageTest extends BasePipelineTest {

    @BeforeEach
    void setUp() {
        // Cấu hình helper — chỉ định thư mục chứa vars/
        helper.registerSharedLibrary(
            library().name('company-jenkins-lib')
                     .defaultVersion('main')
                     .allowOverride(true)
                     .targetPath(new File('.').absolutePath)
                     .retriever(localSource('.'))
                     .build()
        )
        setUp()

        // Cài đặt mock cho các step Jenkins
        binding.setVariable('env', [
            JOB_NAME:     'test-job',
            BUILD_NUMBER: '42',
            WORKSPACE:    '/workspace'
        ])
    }

    @Test
    void 'buildDockerImage_withAllParams_shouldBuildAndPush'() {
        // Arrange (Chuẩn bị)
        def script = loadScript('vars/buildDockerImage.groovy')

        // Act (Thực thi)
        script.call(
            imageName: 'my-service',
            tag:       '1.0.0',
            registry:  'registry.company.com'
        )

        // Assert (Kiểm tra)
        // Kiểm tra lệnh docker build đã được gọi đúng
        assertThat(
            helper.callStack
                  .findAll { it.methodName == 'sh' }
                  .collect { it.args[0] as String },
            hasItem(containsString('docker build -t registry.company.com/my-service:1.0.0'))
        )

        // Kiểm tra lệnh docker push đã được gọi
        assertThat(
            helper.callStack
                  .findAll { it.methodName == 'sh' }
                  .collect { it.args[0] as String },
            hasItem(containsString('docker push registry.company.com/my-service:1.0.0'))
        )
    }

    @Test
    void 'buildDockerImage_withNoParams_shouldUseDefaults'() {
        def script = loadScript('vars/buildDockerImage.groovy')

        script.call([:])   // Gọi với Map rỗng — toàn bộ dùng default

        // Kiểm tra tên job được dùng làm imageName mặc định
        def shCalls = helper.callStack
                            .findAll { it.methodName == 'sh' }
                            .collect { it.args[0] as String }

        assertTrue(shCalls.any { it.contains('test-job') })
        assertTrue(shCalls.any { it.contains('42') })  // BUILD_NUMBER
    }

    @Test
    void 'buildDockerImage_shouldReturnFullImageName'() {
        def script = loadScript('vars/buildDockerImage.groovy')

        def result = script.call(
            imageName: 'api-gateway',
            tag:       '2.0',
            registry:  'ecr.aws/mycompany'
        )

        assertEquals('ecr.aws/mycompany/api-gateway:2.0', result)
    }
}
```

### Viết Test Cho Groovy Class Trong `src/`

```groovy
// test/groovy/DockerHelperTest.groovy
import com.company.jenkins.DockerHelper
import com.lesfurets.jenkins.unit.BasePipelineTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import static org.junit.jupiter.api.Assertions.*

class DockerHelperTest extends BasePipelineTest {

    DockerHelper helper
    def capturedCommands = []

    @BeforeEach
    void setUp() {
        super.setUp()

        // Mock đối tượng steps
        def mockSteps = [
            sh:   { String cmd -> capturedCommands << cmd },
            echo: { String msg -> println "[MOCK ECHO] ${msg}" }
        ] as Object

        helper = new DockerHelper(mockSteps, 'registry.test.com')
    }

    @Test
    void 'buildImage_shouldRunDockerBuildCommand'() {
        helper.buildImage('my-app', '1.0.0')

        assertTrue(
            capturedCommands.any { it.contains('docker build -t registry.test.com/my-app:1.0.0 .') }
        )
    }

    @Test
    void 'pushImage_shouldRunDockerPushCommand'() {
        helper.pushImage('my-app', '1.0.0')

        assertTrue(
            capturedCommands.any { it.contains('docker push registry.test.com/my-app:1.0.0') }
        )
    }

    @Test
    void 'buildAndPush_shouldRunBothCommands'() {
        helper.buildAndPush('my-app', '1.0.0')

        assertEquals(2, capturedCommands.size())
        assertTrue(capturedCommands[0].contains('docker build'))
        assertTrue(capturedCommands[1].contains('docker push'))
    }
}
```

### Chạy Test

```bash
# Chạy toàn bộ test suite
./gradlew test

# Chạy một test class cụ thể
./gradlew test --tests "BuildDockerImageTest"

# Xem report (báo cáo) sau khi chạy
open build/reports/tests/test/index.html
```

---

## Phần 3: CI/CD Cho Chính Library

Library cần có pipeline test riêng để tự động kiểm tra mỗi khi có thay đổi.

### Jenkinsfile Cho Library

```groovy
// Jenkinsfile trong root của jenkins-shared-library repository
pipeline {
    agent {
        docker {
            image 'openjdk:17-jdk-slim'
            args  '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Tests') {
            steps {
                sh './gradlew test'
            }
            post {
                always {
                    junit 'build/test-results/test/*.xml'
                    publishHTML([
                        reportDir:   'build/reports/tests/test',
                        reportFiles: 'index.html',
                        reportName:  'Test Report'
                    ])
                }
            }
        }

        stage('Code Quality') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh './gradlew sonarqube'
                }
            }
        }

        stage('Validate Syntax') {
            steps {
                // Kiểm tra cú pháp Groovy của tất cả file trong vars/ và src/
                sh '''
                    find vars src -name "*.groovy" -exec groovy -c {} \\;
                    echo "Tất cả file Groovy đều hợp lệ cú pháp"
                '''
            }
        }

        stage('Tag Release') {
            when {
                branch 'main'
                expression { currentBuild.changeSets.size() > 0 }
            }
            steps {
                script {
                    def version = readFile('VERSION').trim()
                    sh "git tag -a v${version} -m 'Release v${version}'"
                    sh "git push origin v${version}"
                }
            }
        }
    }

    post {
        failure {
            slackSend(
                channel: '#jenkins-lib-alerts',
                color:   'danger',
                message: "Library CI thất bại: ${env.JOB_NAME}#${env.BUILD_NUMBER}"
            )
        }
    }
}
```

---

## Phần 4: Tài Liệu Hóa Library

### File `vars/*.txt` — Tài Liệu Tích Hợp Jenkins

Jenkins tự động hiển thị nội dung file `.txt` trong phần **Pipeline Syntax → Global Variables Reference**. Mỗi step nên có file `.txt` tương ứng:

```text
# vars/buildDockerImage.txt
Build và push Docker image lên container registry.

## Parameters (Tham Số)

  imageName  (String, optional)
             Tên image không kèm registry và tag.
             Mặc định: tên Jenkins job (chữ thường).

  tag        (String, optional)
             Tag của image (ví dụ: v1.2.3, git-commit-sha).
             Mặc định: BUILD_NUMBER.

  registry   (String, optional)
             Địa chỉ container registry.
             Mặc định: 'docker.io'.

  dockerfile (String, optional)
             Đường dẫn đến Dockerfile.
             Mặc định: 'Dockerfile' (thư mục hiện tại).

  context    (String, optional)
             Docker build context (ngữ cảnh build).
             Mặc định: '.' (thư mục hiện tại).

## Returns (Giá Trị Trả Về)

  String — Tên image đầy đủ bao gồm registry, tên và tag.
  Ví dụ: 'registry.company.com/my-service:42'

## Examples (Ví Dụ)

  // Dùng tất cả tham số
  def imageRef = buildDockerImage(
      imageName:  'my-service',
      tag:        env.GIT_COMMIT[0..7],
      registry:   'registry.company.com',
      dockerfile: 'docker/Dockerfile.prod',
      context:    '.'
  )

  // Dùng tham số tối thiểu
  buildDockerImage(imageName: 'my-service')

  // Toàn bộ dùng mặc định
  buildDockerImage()

## Notes (Lưu Ý)

  - Yêu cầu Docker đã được cài đặt trên agent
  - Cần đăng nhập registry trước (dùng withDockerRegistry block)
  - Image được push ngay sau khi build thành công
```

### README.md Của Library — Tài Liệu Cho Người Dùng

```markdown
# company-jenkins-lib — Jenkins Shared Library

Thư viện Jenkins dùng chung cho toàn bộ tổ chức Company.

## Cài Đặt

Thêm vào đầu Jenkinsfile:

```groovy
@Library('company-jenkins-lib@v2.1.0') _
```

## Danh Sách Step (API Reference)

| Step | Mô Tả | Ví Dụ |
|------|-------|-------|
| `buildDockerImage(Map)` | Build và push Docker image | `buildDockerImage(imageName: 'app')` |
| `deployToKubernetes(Map)` | Deploy lên K8s qua kubectl | `deployToKubernetes(appName: 'app')` |
| `runSonarQube(Map)` | Quét chất lượng mã | `runSonarQube(projectKey: 'my-app')` |
| `sendSlackNotification(Map)` | Gửi thông báo Slack | `sendSlackNotification(channel: '#dev')` |
| `standardCIPipeline(Map, Closure)` | Pipeline CI/CD chuẩn | Xem bên dưới |

## Ví Dụ Nhanh

### Dùng standardCIPipeline (Khuyến Nghị)

```groovy
@Library('company-jenkins-lib@v2.1.0') _

standardCIPipeline(
    appName:      'my-service',
    buildTool:    'maven',     // 'maven', 'gradle', 'npm'
    registry:     'registry.company.com',
    slackChannel: '#team-alerts'
)
```

### Dùng step riêng lẻ

```groovy
@Library('company-jenkins-lib@v2.1.0') _

pipeline {
    agent any
    stages {
        stage('Docker Build') {
            steps {
                buildDockerImage(imageName: 'my-service', tag: env.BUILD_NUMBER)
            }
        }
    }
}
```

## Phiên Bản

Luôn dùng version cố định trong Jenkinsfile production:
- Ổn định: `@v2.1.0`
- Phát triển: `@main` (không dùng production)

## Đóng Góp (Contributing)

1. Fork repo, tạo feature branch
2. Viết code + unit test
3. Chạy `./gradlew test` — toàn bộ test phải pass
4. Tạo Pull Request vào `main`
5. Sau khi merge, tạo tag release nếu cần
```

---

## Phần 5: Governance Model — Mô Hình Quản Trị

### Ai Được Phép Thay Đổi Library?

```
Người dùng (Tất cả team)
    │
    ├── Đọc tài liệu
    ├── Dùng step đã có
    └── Tạo Issue (vấn đề) → báo cáo bug hoặc yêu cầu tính năng

Platform Team (Nhóm nền tảng) / Library Maintainers (Người bảo trì)
    │
    ├── Review và merge Pull Request
    ├── Quyết định breaking changes
    ├── Thực hiện release (tag version)
    └── Hỗ trợ migration (di chuyển phiên bản)

Contribution từ các team (tùy chính sách)
    │
    ├── Tạo PR với step mới
    ├── Bắt buộc: unit test + tài liệu
    └── Platform Team review và merge
```

### Quy Tắc Cho Library

```markdown
## Library Guidelines (Hướng Dẫn Thư Viện)

### Yêu Cầu Bắt Buộc Khi Thêm Step Mới
- [ ] Unit test coverage (độ phủ kiểm thử) ≥ 80%
- [ ] File .txt tài liệu trong vars/
- [ ] Cập nhật CHANGELOG.md
- [ ] Tham số đều có default value hợp lý
- [ ] Không breaking change trong minor/patch release

### Quy Ước Đặt Tên
- Step: camelCase, động từ trước danh từ: `buildDockerImage`, `deployToK8s`
- Class: PascalCase: `DockerHelper`, `KubernetesHelper`
- Hằng số: UPPER_SNAKE_CASE: `DEFAULT_TIMEOUT`

### Xử Lý Lỗi
- Luôn ném exception rõ ràng với message có ý nghĩa
- Không nuốt exception im lặng
- Đặt currentBuild.result = 'FAILURE' trước khi ném error()
```

---

## Phần 6: Migration Khi Có Breaking Changes

Khi phải thay đổi interface của một step, cần có kế hoạch migration (di chuyển) rõ ràng:

### Phương Pháp: Deprecation Period (Giai Đoạn Lỗi Thời)

```groovy
// vars/buildDockerImage.groovy — v2.0.0

def call(Map config) {
    // Xử lý cú pháp cũ (backward compat trong thời gian deprecation)
    _warnIfOldSignature(config)
    // Logic mới...
}

// Hỗ trợ cú pháp cũ: buildDockerImage('imageName', 'tag')
def call(String imageName, String tag) {
    echo "⚠️ DEPRECATED: buildDockerImage(name, tag) sẽ bị xóa trong v3.0.0."
    echo "   Hãy chuyển sang: buildDockerImage(imageName: '${imageName}', tag: '${tag}')"

    // Chuyển tiếp sang cú pháp mới
    call(imageName: imageName, tag: tag)
}

@NonCPS
private void _warnIfOldSignature(Map config) {
    // Kiểm tra nếu có key cũ không còn được hỗ trợ
    if (config.containsKey('name')) {
        throw new IllegalArgumentException(
            "Tham số 'name' đã đổi thành 'imageName' từ v2.0.0. " +
            "Xem CHANGELOG.md để biết cách migrate."
        )
    }
}
```

### Thông Báo Migration Cho Team

```
Kênh Slack #jenkins-lib-updates:

📢 jenkins-shared-lib v2.0.0 — Breaking Change

buildDockerImage đã thay đổi interface:

Cũ (v1.x):   buildDockerImage('my-app', '1.0.0')
Mới (v2.x):  buildDockerImage(imageName: 'my-app', tag: '1.0.0')

Timeline:
• v2.0.0 (hôm nay): Cả hai cú pháp đều hoạt động, cú pháp cũ in warning
• v2.1.0 (30 ngày): Cú pháp cũ in warning rõ hơn + hướng dẫn tự động
• v3.0.0 (90 ngày): Cú pháp cũ BỊ XÓA — phải migrate trước ngày X

Cần hỗ trợ migrate? Liên hệ @platform-team hoặc mở Issue.
```

---

## Tóm Tắt Best Practices

### Versioning
- ✅ Dùng SemVer: `MAJOR.MINOR.PATCH`
- ✅ Pinned version trong Jenkinsfile production: `@v2.1.0`
- ✅ Duy trì `CHANGELOG.md` rõ ràng
- ✅ Tag mọi release, không dùng branch `main` cho production
- ❌ Không dùng `@main` hoặc `@latest` trong pipeline production

### Testing
- ✅ Test mọi step quan trọng với JenkinsPipelineUnit
- ✅ CI tự động cho library — test trên mỗi PR
- ✅ Test cả happy path (luồng thành công) lẫn error case (trường hợp lỗi)
- ✅ Mock credentials — không dùng secret thật trong test
- ❌ Không merge khi test chưa pass

### Tài Liệu Hóa
- ✅ File `.txt` cho mỗi step trong `vars/`
- ✅ README với ví dụ thực tế
- ✅ Inline comment (ghi chú trong code) cho logic không rõ ràng
- ✅ Cập nhật tài liệu song song với thay đổi code
- ❌ Không để tài liệu lỗi thời (outdated docs tệ hơn không có docs)

### Thiết Kế API
- ✅ Dùng `Map config` với default values rõ ràng
- ✅ Tên step mô tả hành động: `buildDockerImage`, `deployToKubernetes`
- ✅ Return giá trị hữu ích khi có thể
- ✅ Throw exception với message có ý nghĩa
- ❌ Không hardcode logic đặc thù dự án trong library chung
- ❌ Không nuốt exception im lặng

---

## Câu Hỏi Phỏng Vấn

1. **Tại sao phải versioning Shared Library? Hậu quả nếu không làm?**
   → Không versioning dẫn đến library thay đổi phá vỡ tất cả pipeline đang dùng. Mọi breaking change trở thành incident production.

2. **JenkinsPipelineUnit là gì? Lợi ích so với test trên Jenkins thật?**
   → Framework mock Jenkins step để test Groovy unit. Chạy nhanh hơn hàng chục lần, không cần Jenkins server, tích hợp được vào CI của chính library.

3. **Khi có breaking change trong library, bạn xử lý như thế nào?**
   → Deprecation period: giữ cú pháp cũ với warning, đưa ra timeline migration rõ ràng, thông báo cho toàn team, xóa cú pháp cũ sau đủ thời gian.

4. **Làm sao đảm bảo quality (chất lượng) của Library trước khi merge vào main?**
   → PR review bắt buộc, unit test phải pass, CI pipeline tự động, code coverage threshold (ngưỡng độ phủ), tài liệu phải được cập nhật.

5. **Ai nên "sở hữu" Shared Library trong một tổ chức lớn?**
   → Platform Team hoặc DevOps Guild — nhóm có trách nhiệm rõ ràng về CI/CD infrastructure. Các team khác có thể đóng góp qua PR nhưng Platform Team là reviewer cuối.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
