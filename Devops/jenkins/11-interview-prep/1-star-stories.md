# STAR Stories — Câu Chuyện CI/CD Incidents Thực Tế

> Bộ câu chuyện STAR (Situation — Task — Action — Result) về các sự cố CI/CD thường gặp trong môi trường production. Đọc, chọn câu phù hợp với kinh nghiệm bản thân, và điều chỉnh chi tiết để phù hợp với dự án thực tế của bạn.

## Mục Lục

1. [Cách Sử Dụng Tài Liệu Này](#cách-sử-dụng-tài-liệu-này)
2. [Story 1: Pipeline Bị Break Sau Plugin Update](#story-1-pipeline-bị-break-sau-plugin-update)
3. [Story 2: Build Time Tăng Gấp 3 Lần — Tối Ưu CI/CD](#story-2-build-time-tăng-gấp-3-lần--tối-ưu-cicd)
4. [Story 3: Secret Rò Rỉ Trong Build Log](#story-3-secret-rò-rỉ-trong-build-log)
5. [Story 4: Jenkins Hết Disk Giữa Đêm](#story-4-jenkins-hết-disk-giữa-đêm)
6. [Story 5: Migration Jenkins Sang Kubernetes Agent](#story-5-migration-jenkins-sang-kubernetes-agent)
7. [Story 6: Deploy Nhầm Environment — Production Incident](#story-6-deploy-nhầm-environment--production-incident)
8. [Story 7: Shared Library Breaking Change Ảnh Hưởng 20 Teams](#story-7-shared-library-breaking-change-ảnh-hưởng-20-teams)
9. [Câu Hỏi Behavioral Thường Gặp](#câu-hỏi-behavioral-thường-gặp)

---

## Cách Sử Dụng Tài Liệu Này

### Nguyên Tắc STAR

```
S — Situation (Bối Cảnh):
    Mô tả môi trường, scale, team size, tech stack.
    Đủ context để interviewer hiểu mức độ phức tạp.
    Tránh dài dòng — 2-3 câu là đủ.

T — Task (Nhiệm Vụ):
    Bạn chịu trách nhiệm gì? Deadline? Constraint?
    Dùng "Tôi" chứ không phải "Chúng tôi".
    Nêu rõ bạn là người chủ động hay được giao.

A — Action (Hành Động):
    PHẦN QUAN TRỌNG NHẤT — chiếm 60% thời gian kể.
    Nêu cụ thể: dùng công cụ gì, lệnh gì, quyết định gì.
    Giải thích tại sao chọn hướng đó (trade-off).
    Nêu obstacles bạn gặp và vượt qua thế nào.

R — Result (Kết Quả):
    Kết quả đo được: số liệu, %, thời gian, tỷ lệ lỗi.
    Kết quả về business: tiết kiệm bao nhiêu, giảm downtime.
    Bài học rút ra (optional nhưng gây ấn tượng tốt).
```

### Cách Điều Chỉnh

1. Đọc story, hiểu cốt lõi vấn đề
2. Thay số liệu (team size, thời gian, tỷ lệ) bằng con số thực tế của bạn
3. Thay tech stack nếu khác (Maven → Gradle, Docker Hub → ECR, v.v.)
4. Bỏ chi tiết không phù hợp, thêm chi tiết đặc thù của project bạn
5. Luyện nói to — đừng chỉ đọc

---

## Story 1: Pipeline Bị Break Sau Plugin Update

**Câu hỏi thường gặp:**
- "Kể về một lần bạn xử lý sự cố CI/CD nghiêm trọng"
- "Kể về một lần mọi thứ bị break và bạn phải sửa gấp"

---

**S — Situation:**

Tôi làm DevOps Engineer tại công ty fintech với 15 microservices trên Kubernetes. Jenkins phục vụ khoảng 80 developer với ~200 build/ngày. Một buổi sáng, toàn bộ pipeline bắt đầu fail sau khi Jenkins tự động cập nhật các plugin qua cấu hình "check for updates automatically".

**T — Task:**

Tôi được giao nhiệm vụ tìm nguyên nhân và restore CI/CD trong vòng 2 giờ — đang gần deadline sprint, team không thể merge code và deploy.

**A — Action:**

Bước đầu, tôi kiểm tra Console Output của các build fail và thấy lỗi chung: `java.lang.NoSuchMethodError: hudson.model.Run.getArtifacts()`. Đây là dấu hiệu của plugin incompatibility (xung đột phiên bản plugin).

Tôi vào Manage Jenkins → Manage Plugins → Installed và lọc theo "Recently updated" — phát hiện Pipeline plugin và Pipeline: Groovy plugin vừa được update lên version mới sáng nay.

Tôi tạo test pipeline đơn giản nhất có thể để xác nhận vấn đề:
```groovy
pipeline {
    agent any
    stages {
        stage('Test') { steps { echo 'hello' } }
    }
}
```
Build fail với cùng lỗi → xác nhận vấn đề ở core Pipeline plugin, không phải code của team.

Kiểm tra Jenkins changelog và plugin changelog → phát hiện Pipeline: Groovy 2.x có breaking change với Pipeline plugin 2.x cũ. Chúng bị update lên incompatible versions.

Quyết định: Downgrade (hạ phiên bản) Pipeline: Groovy plugin về version trước. Tôi tải file `.hpi` (Jenkins Plugin Package) từ archives.jenkins.io, upload thủ công qua Plugin Manager → Advanced → Upload Plugin. Restart Jenkins.

Sau restart, tôi verify bằng test pipeline — thành công. Thông báo cho team qua Slack và mở lại CI.

Sau khi resolved, tôi:
1. Tắt auto-update plugin trong Jenkins System config
2. Tạo policy: mọi plugin update phải test trên Jenkins staging trước
3. Thiết lập Jenkins staging instance (bản sao của production) để test updates

**R — Result:**

- Downtime CI/CD: 90 phút (trong mức cho phép 2 giờ)
- Zero deploy delay cho sprint deadline
- Policy mới ngăn chặn được 2 plugin incidents tương tự trong 6 tháng tiếp theo
- **Bài học:** Không bao giờ để Jenkins tự động update plugin trong production. Luôn có staging Jenkins để test trước.

---

## Story 2: Build Time Tăng Gấp 3 Lần — Tối Ưu CI/CD

**Câu hỏi thường gặp:**
- "Kể về một lần bạn tối ưu hiệu suất hệ thống"
- "Bạn đã cải thiện CI/CD pipeline như thế nào?"

---

**S — Situation:**

Tại công ty e-commerce, tôi phụ trách Jenkins CI/CD cho một monorepo Java với 8 modules. Sau 6 tháng, build time tăng từ 12 phút lên 38 phút/build. Với 50-60 build/ngày, developer phải chờ rất lâu để nhận feedback từ CI.

**T — Task:**

Tôi tự nhận nhiệm vụ phân tích bottleneck (điểm nghẽn) và đưa build time xuống dưới 15 phút mà không tăng thêm chi phí infrastructure đáng kể.

**A — Action:**

**Phân tích:**
Tôi thu thập data từ 50 build gần nhất và phân tích stage duration:
```
Stage breakdown (trung bình):
  Checkout:          2 phút (Maven repo download cả dependency)
  Build:             8 phút
  Unit Test:        12 phút (chạy tuần tự)
  Integration Test:  8 phút
  SonarQube:         5 phút
  Docker Build:      3 phút
  TOTAL:            38 phút
```

**Vấn đề xác định được:**

1. **Dependency download mỗi build:** Mỗi build download 300MB Maven dependencies từ internet do cache không được dùng lại giữa các build (Docker agent fresh container mỗi lần)

2. **Unit Test chạy tuần tự:** 8 modules test tuần tự, không song song

3. **Docker build không dùng layer cache:** Mỗi lần COPY pom.xml và COPY src/ cùng layer, invalidate cache khi code thay đổi

**Giải pháp triển khai:**

**Fix 1 — Maven Dependency Cache với Persistent Volume:**
```yaml
# Kubernetes Pod Template
volumes:
- name: maven-cache
  persistentVolumeClaim:
    claimName: maven-repository-cache
containers:
- name: maven
  volumeMounts:
  - name: maven-cache
    mountPath: /root/.m2/repository
```

**Fix 2 — Parallel Test Stages:**
```groovy
stage('Test') {
    parallel {
        stage('Module API') { steps { sh 'mvn test -pl api' } }
        stage('Module Service') { steps { sh 'mvn test -pl service' } }
        stage('Module Repository') { steps { sh 'mvn test -pl repository' } }
        // ... 5 modules còn lại
    }
}
```

**Fix 3 — Dockerfile tối ưu layer cache:**
```dockerfile
# ✅ Copy pom.xml trước, download dependencies (cached layer)
COPY pom.xml .
RUN mvn dependency:go-offline

# Sau đó copy source (layer này invalidate khi code thay đổi)
COPY src/ src/
RUN mvn package -DskipTests
```

**R — Result:**

```
Sau tối ưu:
  Checkout:          0.5 phút (cache hit, chỉ download thay đổi)
  Build:             7   phút
  Test (parallel):   4   phút (8 modules song song, bottleneck ~4 phút)
  Integration Test:  8   phút (không thay đổi)
  SonarQube:         1   phút (scan diff, không toàn bộ)
  Docker Build:      1   phút (layer cache hit thường xuyên)
  TOTAL:            ~12  phút
```

- Build time: giảm từ 38 phút → 12 phút (giảm **68%**)
- Developer feedback loop nhanh hơn ~26 phút/build
- Chi phí infrastructure tăng nhẹ (PVC cho Maven cache, ~$20/tháng)
- Team hài lòng — không còn phàn nàn về CI chậm

---

## Story 3: Secret Rò Rỉ Trong Build Log

**Câu hỏi thường gặp:**
- "Kể về một sự cố bảo mật bạn đã xử lý"
- "Bạn đảm bảo secrets không bị lộ trong CI/CD như thế nào?"

---

**S — Situation:**

Trong team startup 20 người, tôi phát hiện trong lịch sử build log Jenkins có API token của AWS được in ra — developer đã hardcode `aws configure` với access key trong Jenkinsfile để "thử nhanh" và commit lên repository.

**T — Task:**

Tôi cần: (1) revoke credential bị lộ ngay lập tức, (2) xóa sạch khỏi log và git history, (3) thiết lập hệ thống ngăn tái diễn.

**A — Action:**

**Immediate Response (phản ứng ngay lập tức — trong 30 phút đầu):**

1. Revoke AWS Access Key bị lộ qua AWS Console → IAM → Delete Access Key
2. Kiểm tra AWS CloudTrail (nhật ký hoạt động AWS) trong 24h qua — xem key đã bị dùng chưa
3. Xóa Console Output trong Jenkins UI (không xóa được hoàn toàn nếu đã được index)
4. Force-rotate (buộc xoay vòng) tất cả credentials AWS của team

**Fix Git History:**
```bash
# Xóa file chứa credential khỏi toàn bộ git history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch Jenkinsfile' \
  --prune-empty --tag-name-filter cat -- --all

git push origin --force --all
```

**Thiết lập hệ thống phòng ngừa:**

1. **Cấu hình Jenkins Credentials Store** cho tất cả secrets:
```groovy
// ❌ Cách cũ (developer đã làm)
sh 'aws configure set aws_access_key_id AKIAXXXXXXXX'

// ✅ Cách đúng — dùng Jenkins Credentials
withCredentials([usernamePassword(
    credentialsId: 'aws-credentials',
    usernameVariable: 'AWS_ACCESS_KEY_ID',
    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
)]) {
    sh 'aws s3 cp target/*.jar s3://my-bucket/'
}
```

2. **Cài đặt `git-secrets` hook** (công cụ ngăn commit chứa secret):
```bash
git secrets --install
git secrets --register-aws
# Tự động scan mỗi commit, block nếu phát hiện AWS pattern
```

3. **Tích hợp Trivy secret scanning** vào pipeline:
```groovy
stage('Secret Scan') {
    steps {
        sh 'trivy fs --scanners secret --exit-code 1 .'
    }
}
```

4. **Training 30 phút** cho team về Credentials best practices

**R — Result:**

- Không có evidence key bị dùng trái phép (CloudTrail clean)
- Git history đã được sanitized (làm sạch)
- Zero secret leak incidents trong 12 tháng tiếp theo
- Secret scanning được tích hợp vào onboarding checklist (danh sách hướng dẫn nhân viên mới)
- **Bài học:** Fail-fast với secret scanning phải là first-class citizen trong pipeline, không phải afterthought.

---

## Story 4: Jenkins Hết Disk Giữa Đêm

**Câu hỏi thường gặp:**
- "Kể về một incident production bạn đã xử lý"
- "Bạn xử lý on-call như thế nào?"

---

**S — Situation:**

3 giờ sáng, tôi nhận alert từ PagerDuty: Jenkins master server báo disk usage 98%. Toàn bộ build đang fail với lỗi `No space left on device`. Hệ thống phục vụ 5 team với release sáng hôm sau.

**T — Task:**

On-call engineer — tôi cần restore Jenkins hoạt động bình thường trước 7 giờ sáng để không ảnh hưởng business.

**A — Action:**

**Triage nhanh (5 phút đầu):**
```bash
# SSH vào Jenkins server
df -h
# /var/jenkins_home: 99% → 498GB / 500GB

du -sh /var/jenkins_home/* | sort -rh | head -20
# /var/jenkins_home/jobs: 420GB (!!)
# /var/jenkins_home/workspace: 60GB
```

**Giải phóng disk ngay:**
```bash
# Dọn workspace của build đã hoàn thành
find /var/jenkins_home/workspace -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;

# Xóa build log cũ (giữ 30 build gần nhất cho mỗi job)
# Thực hiện qua Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 groovy = << 'EOF'
Jenkins.instance.getAllItems(Job.class).each { job ->
    def builds = job.getBuilds()
    if (builds.size() > 30) {
        builds[30..-1].each { build ->
            build.delete()
        }
    }
}
EOF
```

Sau 20 phút: disk xuống còn 65% → Jenkins hoạt động trở lại.

**Root cause analysis (phân tích nguyên nhân gốc):**

- Không có Build Discard Policy trên 12/20 jobs
- Artifact được archive nhưng không bao giờ xóa
- Workspace không được cleanup sau build

**Fix lâu dài:**

1. **Global default Build Discard Policy:**

Vào Manage Jenkins → System → Global Build Discard Policy:
```
Keep builds: 30 days or 50 builds (whichever comes first)
Keep artifacts: 10 builds
```

2. **Thêm vào Shared Library template:**
```groovy
// Mọi pipeline đều có
options {
    buildDiscarder(logRotator(
        numToKeepStr: '30',
        daysToKeepStr: '14',
        artifactNumToKeepStr: '5'
    ))
    disableConcurrentBuilds()
}
post {
    always { cleanWs() }
}
```

3. **Alert sớm hơn:** Thêm monitoring alert ở 70% và 85% disk usage thay vì chỉ 95%

4. **Tăng disk từ 500GB lên 1TB** cho headroom (khoảng trống dự phòng)

**R — Result:**

- Restore trong 45 phút — trước deadline 7 giờ sáng
- Release sáng hôm sau không bị ảnh hưởng
- Không xảy ra disk incident nào trong 18 tháng sau
- Alert system phát hiện sớm hơn, chủ động xử lý thay vì reactive

---

## Story 5: Migration Jenkins Sang Kubernetes Agent

**Câu hỏi thường gặp:**
- "Kể về một dự án migration/modernization bạn đã thực hiện"
- "Làm thế nào bạn cải thiện khả năng scale của CI/CD?"

---

**S — Situation:**

Công ty có Jenkins với 8 static agent (VM cố định) — 4 Linux, 4 Windows. Mỗi agent có 4 Executor. Thường xuyên xảy ra tình trạng: build queue dài 20-30 phút vào giờ cao điểm (9–11 giờ sáng), nhưng agent rảnh hoàn toàn vào ban đêm và cuối tuần.

**T — Task:**

Tôi được giao migrate Linux agents sang Kubernetes Dynamic Agents để giải quyết vấn đề scale, giảm chi phí infrastructure, và không làm gián đoạn ~150 build/ngày.

**A — Action:**

**Giai đoạn 1 — Chuẩn bị (2 tuần):**

Kiểm kê (inventory) tất cả jobs trên Linux agents:
```bash
# Liệt kê job dùng Linux agent
grep -r "label.*linux" /var/jenkins_home/jobs/*/config.xml | cut -d: -f1
```

Phân loại job:
- 80% job: standard Java build — migrate được
- 15% job: cần Docker socket (Docker-in-Docker) — cần special config
- 5% job: cần tool đặc biệt (hardware key, legacy lib) — **không migrate**

**Giai đoạn 2 — Setup Kubernetes Agent:**

```groovy
// Shared Library: vars/k8sAgent.groovy
def call(Map config = [:], Closure body) {
    def defaultConfig = [
        cpuRequest: '500m',
        cpuLimit: '2',
        memoryRequest: '512Mi',
        memoryLimit: '2Gi',
        image: 'maven:3.9-eclipse-temurin-17'
    ]
    def mergedConfig = defaultConfig + config

    podTemplate(
        containers: [
            containerTemplate(
                name: 'maven',
                image: mergedConfig.image,
                resourceRequestCpu: mergedConfig.cpuRequest,
                resourceLimitCpu: mergedConfig.cpuLimit,
                resourceRequestMemory: mergedConfig.memoryRequest,
                resourceLimitMemory: mergedConfig.memoryLimit,
                command: 'sleep', args: '99d'
            )
        ]
    ) {
        node(POD_LABEL) {
            body()
        }
    }
}
```

**Giai đoạn 3 — Migrate song song (4 tuần):**

Chiến lược: Thêm label `k8s` vào job mới, giữ `linux` label cho job cũ. Dần dần migrate từng batch.

```groovy
// Trước migration
agent { label 'linux' }

// Sau migration
agent {
    kubernetes {
        yaml """..."""
    }
}
```

**Giai đoạn 4 — Validation và Cutover:**

Chạy song song 2 tuần: cùng job chạy trên cả VM và K8s, so sánh kết quả. Sau 2 tuần, tất cả consistent → shutdown VM agents.

**R — Result:**

- Build queue wait time: giảm từ 25 phút → 2 phút vào giờ cao điểm (dynamic scaling)
- Chi phí EC2 instances: giảm 45% (Spot instances, không trả khi không có build)
- Environment isolation (cô lập môi trường): mỗi build có fresh container, không có "works on my agent" issue
- Migration zero-downtime (không có downtime): team tiếp tục build bình thường trong suốt quá trình
- **Bài học:** Migrate theo batch nhỏ và validate song song trước khi tắt hệ thống cũ.

---

## Story 6: Deploy Nhầm Environment — Production Incident

**Câu hỏi thường gặp:**
- "Kể về sai lầm bạn đã mắc phải và bạn học được gì"
- "Kể về một incident và cách bạn xử lý"

---

**S — Situation:**

Tôi là người duy nhất vận hành CI/CD cho startup 8 người. Pipeline có 2 environment: staging và production, dùng chung Jenkinsfile với tham số `ENVIRONMENT`. Một ngày, trong lúc vội vã deploy fix hotfix (bản vá khẩn), tôi đã chọn nhầm `production` thay vì `staging` cho version chưa được test đầy đủ.

**T — Task:**

Nhận ra ngay sau khi deploy — tôi cần rollback production trong vòng vài phút và ngăn điều này xảy ra lần nữa.

**A — Action:**

**Immediate Rollback (khôi phục ngay):**

```bash
# Rollback Helm deployment về version trước
helm rollback myapp 0  # 0 = version trước version hiện tại
# Verify
kubectl rollout status deployment/myapp
```

Thời gian rollback: 3 phút. Error rate trở về 0%.

**Post-mortem (phân tích sau sự cố):**

Viết post-mortem document với root causes:
1. Không có confirmation step trước khi deploy production
2. UI Jenkins cho phép chọn environment từ dropdown — quá dễ nhầm
3. Không có diff review trước khi deploy

**Giải pháp cải thiện:**

1. **Tách biệt pipeline cho production** — không dùng chung parameter:
```groovy
// Jenkinsfile-staging
pipeline {
    agent any
    stages { /* ... */ }
    post { always { echo "STAGING deploy" } }
}

// Jenkinsfile-production
pipeline {
    agent any
    stages {
        stage('Pre-deploy Check') {
            steps {
                // Kiểm tra staging đã pass trong 24h gần nhất
                sh './scripts/verify-staging-green.sh'
            }
        }
        stage('Approval') {
            steps {
                input(
                    message: "DEPLOY TO PRODUCTION — Are you sure?",
                    submitter: 'senior-dev,cto'
                )
            }
        }
        stage('Deploy') {
            steps {
                sh 'helm upgrade --install myapp ./chart --set env=production'
            }
        }
    }
}
```

2. **Cấu hình job color coding** và naming convention rõ ràng:
   - `myapp-staging-deploy` (UI màu xanh lá)
   - `myapp-PRODUCTION-deploy` (UI màu đỏ, lock icon)

3. **Restrict production job** — chỉ senior dev mới có quyền trigger

**R — Result:**

- Production downtime: ~3 phút (rollback nhanh nhờ Helm)
- Impact: <0.1% request bị lỗi trong 3 phút
- Không có khách hàng report issue
- Zero production deploy incidents trong 1 năm sau
- **Bài học:** Gates (cổng phê duyệt) và separate pipelines quan trọng hơn convenience. Bất kỳ thứ gì liên quan production cần ít nhất một confirmation step.

---

## Story 7: Shared Library Breaking Change Ảnh Hưởng 20 Teams

**Câu hỏi thường gặp:**
- "Kể về một lần bạn phải phối hợp với nhiều team để giải quyết vấn đề"
- "Bạn quản lý Shared Libraries như thế nào để tránh breaking changes?"

---

**S — Situation:**

Tôi là Platform Engineer, quản lý Jenkins Shared Library dùng chung cho 20 team (120 Jenkinsfile). Tôi refactor (cải tổ) function `buildDockerImage()` để thêm tính năng multi-platform build — nhưng thay đổi signature (chữ ký hàm) không backwards-compatible. Sau khi merge lên `main` branch của library (không có version tag), tất cả pipeline trên 20 teams đều fail.

**T — Task:**

Tôi cần: restore 120 pipelines ngay lập tức, không break thêm, và thiết lập versioning strategy (chiến lược quản lý phiên bản) để ngăn tái diễn.

**A — Action:**

**Immediate Fix:**

Revert commit ngay lập tức:
```bash
git revert HEAD --no-edit
git push origin main
```

Sau 5 phút, tất cả pipeline tự phục hồi (vì @Library('shared-lib') không pin version → luôn dùng main HEAD).

**Root Cause:**

- Shared Library không dùng version tag
- Không có deprecation process (quy trình báo trước khi xóa tính năng)
- Không có changelog thông báo cho 20 teams

**Thiết kế Versioning Strategy:**

**Semantic versioning (quản lý phiên bản ngữ nghĩa) cho Shared Library:**

```
v1.0.0 — Major version: breaking changes
v1.1.0 — Minor version: tính năng mới, backwards-compatible
v1.0.1 — Patch version: bug fixes
```

```groovy
// Teams pin vào Major version — tự nhận Minor/Patch updates
@Library('shared-lib@v1') _

// Teams pin vào exact version — không tự update
@Library('shared-lib@v1.2.3') _
```

**Deprecation process mới:**

```groovy
// vars/buildDockerImage.groovy
def call(String imageName, String tag) {
    // Deprecated — sẽ bị xóa ở v2.0.0
    echo "WARNING: buildDockerImage(name, tag) is deprecated. Use buildDockerImage(config map) instead."
    buildDockerImageNew(imageName: imageName, tag: tag)
}

def call(Map config) {
    // New API
    // ...
}
```

**Communication plan (kế hoạch thông báo):**

1. Announce deprecation 4 tuần trước khi remove
2. Tạo CHANGELOG.md trong shared library repo
3. Weekly Slack update cho #platform-engineering channel
4. Migration guide với examples cho từng breaking change

**R — Result:**

- Outage duration (thời gian mất hoạt động): 8 phút
- Zero impact nếu teams đã pin version (insentive để teams pin sau incident này)
- 18/20 teams migrate sang versioned library trong 2 tuần
- 0 breaking change incidents trong 12 tháng sau
- **Bài học:** Shared Library phải có versioning từ ngày 1. Backwards compatibility là trách nhiệm của platform team, không phải consumer team.

---

## Câu Hỏi Behavioral Thường Gặp

### Bảng Câu Hỏi — STAR Story Phù Hợp

| Câu Hỏi Phỏng Vấn                                             | Story Phù Hợp                    |
| ------------------------------------------------------------- | --------------------------------- |
| "Kể về incident CI/CD nghiêm trọng nhất bạn từng xử lý"      | Story 1, 4, hoặc 6               |
| "Kể về một lần bạn tối ưu hiệu suất hệ thống"                | Story 2                           |
| "Làm thế nào bạn xử lý sự cố bảo mật?"                       | Story 3                           |
| "Bạn xử lý on-call giữa đêm như thế nào?"                    | Story 4                           |
| "Kể về một dự án migration/modernization"                     | Story 5                           |
| "Kể về sai lầm bạn đã mắc và bạn học được gì"                | Story 6                           |
| "Làm thế nào bạn phối hợp với nhiều team?"                    | Story 7                           |
| "Bạn đảm bảo backward compatibility như thế nào?"             | Story 7                           |
| "Bạn tiếp cận scale vấn đề như thế nào?"                     | Story 2 hoặc Story 5             |

### Câu Hỏi Follow-up Thường Gặp

Sau khi kể STAR story, interviewer hay hỏi thêm:

```
"Nếu làm lại, bạn sẽ làm gì khác?"
→ Nêu 1-2 điều cụ thể, thể hiện growth mindset

"Tại sao bạn chọn giải pháp đó thay vì [alternative]?"
→ Giải thích trade-off một cách rõ ràng

"Bạn đã đo lường kết quả như thế nào?"
→ Luôn có số liệu cụ thể chuẩn bị sẵn

"Team phản ứng như thế nào?"
→ Đề cập đến communication, collaboration, bài học chia sẻ
```

---

**Xem tiếp:** [2-system-design-scenarios.md](2-system-design-scenarios.md) — Bài toán thiết kế hệ thống CI/CD pipeline
