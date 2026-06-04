# 08 — Shared Libraries: Thư Viện Dùng Chung trong Jenkins Pipeline

> Shared Libraries (thư viện dùng chung) là cơ chế cho phép tái sử dụng (reuse) code Groovy giữa nhiều Jenkins pipeline khác nhau. Thay vì copy-paste logic vào từng Jenkinsfile, bạn đặt code vào một repository riêng, rồi mọi pipeline đều có thể gọi chung — giảm trùng lặp, dễ bảo trì và cập nhật tập trung.

---

## Mục Tiêu Học

Sau khi hoàn thành chủ đề này, bạn sẽ có thể:

- Hiểu kiến trúc và cấu trúc thư mục chuẩn của một Shared Library
- Tạo Global Variable (biến toàn cục) và gọi từ bất kỳ Jenkinsfile nào
- Viết Groovy class trong `src/` để tổ chức logic phức tạp
- Quản lý phiên bản thư viện bằng tag và branch (nhánh)
- Kiểm thử Shared Library với `JenkinsPipelineUnit` trước khi đưa vào production (môi trường thực)
- Áp dụng best practices (thực hành tốt nhất) để xây dựng Shared Library ecosystem bền vững

---

## Danh Sách File

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-library-structure.md](1-library-structure.md) | `vars/`, `src/`, `resources/` — cấu trúc thư mục chuẩn, đăng ký thư viện | ⭐⭐ |
| [2-global-variables.md](2-global-variables.md) | Tạo và dùng Global Variable, call step tùy chỉnh, CPS (Continuation Passing Style) | ⭐⭐⭐ |
| [3-best-practices.md](3-best-practices.md) | Versioning (quản lý phiên bản), testing với JenkinsPipelineUnit, tài liệu hóa | ⭐⭐⭐ |

---

## Vấn Đề Shared Libraries Giải Quyết

### Trước Khi Có Shared Libraries

```groovy
// Jenkinsfile của project-A — phải copy toàn bộ logic
pipeline {
    agent any
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = "myrepo/${env.JOB_NAME}:${env.BUILD_NUMBER}"
                    sh "docker build -t ${imageName} ."
                    sh "docker push ${imageName}"
                    // 50 dòng logic khác...
                }
            }
        }
    }
}

// Jenkinsfile của project-B — copy y chang, chỉ đổi tên biến
// → Khi cần sửa logic, phải sửa ở hàng chục Jenkinsfile!
```

### Sau Khi Có Shared Libraries

```groovy
// Jenkinsfile của project-A — gọn, rõ ràng
@Library('company-jenkins-lib@v2.1.0') _

pipeline {
    agent any
    stages {
        stage('Build & Push Image') {
            steps {
                buildAndPushDockerImage(
                    imageName: 'project-a',
                    registry: 'registry.company.com'
                )
            }
        }
    }
}

// Jenkinsfile của project-B — gọi cùng function, logic tập trung một nơi
@Library('company-jenkins-lib@v2.1.0') _
// buildAndPushDockerImage(...) — dùng lại, không cần copy
```

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Master                            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Global Library Configuration             │   │
│  │  Name: company-jenkins-lib                           │   │
│  │  Source: git@github.com:company/jenkins-lib.git      │   │
│  │  Default version: main                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                │
│              Load at pipeline start                         │
│                            │                                │
│  ┌─────────────┐   ┌───────▼──────┐   ┌───────────────┐   │
│  │ Jenkinsfile │──►│ Shared Lib   │   │ Shared Lib    │   │
│  │ project-A   │   │ vars/        │   │ src/          │   │
│  └─────────────┘   │ resources/   │   │ (Groovy class)│   │
│  ┌─────────────┐   └──────────────┘   └───────────────┘   │
│  │ Jenkinsfile │──►│     (cùng thư viện)                  │
│  │ project-B   │                                           │
│  └─────────────┘                                           │
└─────────────────────────────────────────────────────────────┘

         Git Repository: company/jenkins-lib
         ├── vars/
         │   ├── buildAndPushDockerImage.groovy
         │   ├── deployToKubernetes.groovy
         │   └── sendSlackNotification.groovy
         ├── src/
         │   └── com/company/jenkins/
         │       ├── DockerHelper.groovy
         │       └── KubernetesHelper.groovy
         └── resources/
             ├── com/company/jenkins/
             │   └── deploy-template.yaml
             └── scripts/
                 └── health-check.sh
```

---

## Ba Loại Thành Phần Trong Shared Library

### 1. `vars/` — Global Variables (Biến Toàn Cục)

- Mỗi file `.groovy` trong `vars/` tạo ra một **step tùy chỉnh** có thể gọi trực tiếp trong pipeline
- Tên file = tên step (ví dụ: `vars/buildDockerImage.groovy` → `buildDockerImage(...)`)
- Dễ viết, dễ dùng — phù hợp cho **pipeline steps** (bước pipeline)

```groovy
// vars/buildDockerImage.groovy
def call(Map config) {
    sh "docker build -t ${config.imageName}:${config.tag} ."
}

// Trong Jenkinsfile:
buildDockerImage(imageName: 'myapp', tag: env.BUILD_NUMBER)
```

### 2. `src/` — Groovy Classes (Lớp Groovy)

- Tổ chức code theo package (gói) Java chuẩn
- Dùng cho logic phức tạp, cần object-oriented programming (lập trình hướng đối tượng)
- Phải được `@NonCPS` hoặc `Serializable` để hoạt động đúng trong pipeline

```groovy
// src/com/company/jenkins/DockerHelper.groovy
package com.company.jenkins

class DockerHelper implements Serializable {
    def steps
    DockerHelper(steps) { this.steps = steps }

    def buildImage(String name, String tag) {
        steps.sh "docker build -t ${name}:${tag} ."
    }
}
```

### 3. `resources/` — Static Resources (Tài Nguyên Tĩnh)

- Lưu file không phải code: YAML template, shell script, config file
- Truy cập qua `libraryResource('path/to/file')`

```groovy
// Trong pipeline:
def template = libraryResource('com/company/jenkins/deploy-template.yaml')
writeFile file: 'deploy.yaml', text: template
sh 'kubectl apply -f deploy.yaml'
```

---

## Hai Cách Đăng Ký Shared Library

### Cách 1: Global Library (Thư Viện Toàn Cục)

Cấu hình tại **Manage Jenkins → System → Global Pipeline Libraries**:
- Tự động có sẵn cho **mọi pipeline** trong Jenkins
- Có thể `@Library('name') _` hoặc dùng không cần khai báo (nếu bật implicit load)

### Cách 2: Folder-level Library (Thư Viện Cấp Thư Mục)

Cấu hình trong Jenkins Folder (thư mục job):
- Chỉ có sẵn cho pipeline trong folder đó
- Phù hợp cho các team có thư viện riêng

---

## Hai Cách Nạp Library Trong Jenkinsfile

```groovy
// Cách 1: Annotation @Library — nạp ở đầu file
@Library('company-jenkins-lib') _          // dùng default version
@Library('company-jenkins-lib@v2.1.0') _   // pinned version (version cố định)
@Library('company-jenkins-lib@feature/x') _ // dùng branch cụ thể

// Cách 2: Script step — nạp linh hoạt trong runtime
def lib = library('company-jenkins-lib@v2.1.0')
```

---

## Thứ Tự Học Đề Xuất

```
Bắt đầu
   │
   ▼
1-library-structure.md    ← Nắm cấu trúc thư mục và cách đăng ký library
   │
   ▼
2-global-variables.md     ← Viết step đầu tiên trong vars/, hiểu CPS
   │
   ▼
3-best-practices.md       ← Versioning, testing, tài liệu hóa cho production
```

---

## Điều Kiện Tiên Quyết

Trước khi học chủ đề này, bạn nên nắm vững:

- [x] Declarative Pipeline và Scripted Pipeline (`02-pipeline/`)
- [x] Groovy cơ bản: closure (hàm đóng gói), Map, List, class
- [x] Quản lý Credentials trong Jenkins (`06-security/credentials.md`)
- [x] Git workflow: branch, tag, pull request

---

## Câu Hỏi Phỏng Vấn Liên Quan

1. Shared Library là gì? Tại sao cần dùng thay vì copy code vào từng Jenkinsfile?
2. Sự khác biệt giữa `vars/` và `src/` trong Shared Library?
3. CPS (Continuation Passing Style) trong Jenkins pipeline là gì? Tại sao `@NonCPS` quan trọng?
4. Làm thế nào để versioning Shared Library an toàn cho production?
5. Cách test Shared Library mà không cần chạy Jenkins thật?

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
