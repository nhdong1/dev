# 02 — Pipeline: Declarative & Scripted

> Tổng quan về Jenkins Pipeline — từ cú pháp cơ bản đến pipeline nâng cao nhiều nhánh. Đây là chủ đề được hỏi nhiều nhất trong phỏng vấn Jenkins.

## Mục Lục

1. [Pipeline là gì?](#pipeline-là-gì)
2. [Hai loại Pipeline](#hai-loại-pipeline)
3. [Cấu Trúc File Trong Chủ Đề Này](#cấu-trúc-file-trong-chủ-đề-này)
4. [Lộ Trình Học](#lộ-trình-học)
5. [So Sánh Nhanh](#so-sánh-nhanh)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Pipeline là gì?

**Jenkins Pipeline** là một tập hợp các plugin cho phép triển khai và tích hợp **Continuous Delivery (CD — Phân Phối Liên Tục)** vào Jenkins. Pipeline định nghĩa toàn bộ quá trình build, test, và deploy dưới dạng code — thường lưu trong file `Jenkinsfile` bên trong repository.

### Vì sao Pipeline quan trọng?

| Vấn đề (không dùng Pipeline)         | Giải pháp (dùng Pipeline)                  |
| ------------------------------------- | ------------------------------------------ |
| Cấu hình job chỉ tồn tại trên UI     | Pipeline lưu trong SCM — có version history|
| Không thể review thay đổi CI/CD      | `Jenkinsfile` được review như code thường  |
| Build logic rải rác, khó tái sử dụng | Shared Libraries tập trung hóa logic       |
| Không recover được khi Jenkins restart| Pipeline có thể resume sau khi restart      |

### Pipeline as Code (Pipeline dưới dạng code)

Triết lý cốt lõi của Jenkins Pipeline là **"Pipeline as Code"**: toàn bộ quá trình CI/CD được mô tả trong file code (`Jenkinsfile`), được commit vào repository, được review và quản lý version giống như code ứng dụng.

```
Repository
├── src/                    ← Source code ứng dụng
├── tests/                  ← Test cases
├── Dockerfile              ← Container definition
└── Jenkinsfile             ← CI/CD pipeline definition ← QUAN TRỌNG
```

---

## Hai Loại Pipeline

### 1. Declarative Pipeline (Pipeline Khai Báo) — Khuyến Nghị

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
        }
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        success {
            echo 'Pipeline hoàn thành thành công!'
        }
        failure {
            echo 'Pipeline thất bại!'
        }
    }
}
```

**Đặc điểm:**
- Cú pháp cố định, có cấu trúc rõ ràng
- Dễ đọc, dễ học — khuyến nghị cho hầu hết trường hợp
- Được validate trước khi chạy
- Hỗ trợ `when`, `post`, `options`, `parameters` trực tiếp

### 2. Scripted Pipeline (Pipeline Kịch Bản) — Linh Hoạt Hơn

```groovy
node {
    stage('Build') {
        sh 'mvn clean package'
    }

    stage('Test') {
        try {
            sh 'mvn test'
        } catch (e) {
            currentBuild.result = 'FAILURE'
            throw e
        }
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh './deploy.sh'
        }
    }
}
```

**Đặc điểm:**
- Groovy DSL (Domain-Specific Language — Ngôn Ngữ Đặc Thù Miền) thuần túy
- Linh hoạt hơn — dùng cho logic phức tạp
- Không bị giới hạn bởi cú pháp cố định
- Yêu cầu hiểu Groovy

---

## Cấu Trúc File Trong Chủ Đề Này

```
02-pipeline/
├── README.md                     ← File này — tổng quan
├── 1-declarative-pipeline.md     ← Cú pháp Declarative đầy đủ
├── 2-scripted-pipeline.md        ← Groovy DSL và Scripted Pipeline
├── 3-jenkinsfile.md              ← Quản lý Jenkinsfile trong SCM
├── 4-pipeline-syntax.md          ← Tham chiếu cú pháp và directive
└── 5-multibranch-pipeline.md     ← Multibranch, PR builds, branch filter
```

### Thứ Tự Đọc Đề Xuất

```
1-declarative-pipeline.md   → Học trước — nền tảng quan trọng nhất
2-scripted-pipeline.md      → Hiểu thêm khi cần logic phức tạp
3-jenkinsfile.md            → Best practices quản lý Jenkinsfile
4-pipeline-syntax.md        → Dùng làm tài liệu tham chiếu
5-multibranch-pipeline.md   → Nâng cao — cho dự án thực tế nhiều nhánh
```

---

## Lộ Trình Học

### Beginner (Người Mới)

- [ ] Hiểu khái niệm Stage (giai đoạn), Step (bước), Post (hậu xử lý)
- [ ] Viết Declarative Pipeline với 3 stage cơ bản: Build → Test → Deploy
- [ ] Hiểu `agent any` vs `agent none` vs `agent { label '...' }`
- [ ] Dùng `post { success {} failure {} }` để xử lý kết quả build
- [ ] Lưu Jenkinsfile vào Git repository

### Intermediate (Trung Cấp)

- [ ] Dùng `when` để điều kiện hóa stage (chỉ deploy khi nhánh main)
- [ ] Chạy stage song song với `parallel`
- [ ] Dùng `environment` để khai báo biến môi trường
- [ ] Sử dụng `parameters` để nhận tham số từ người dùng
- [ ] Hiểu vòng đời `stash` / `unstash` để chia sẻ file giữa stage
- [ ] Viết Scripted Pipeline khi cần logic Groovy phức tạp

### Advanced (Nâng Cao)

- [ ] Cấu hình Multibranch Pipeline với branch filtering
- [ ] Thiết lập PR validation pipeline với GitHub/GitLab
- [ ] Viết Shared Libraries để tái sử dụng pipeline code
- [ ] Hiểu `options { skipDefaultCheckout() }` và khi nào cần dùng
- [ ] Debug pipeline với `echo`, `sh 'env'`, `currentBuild` object

---

## So Sánh Nhanh

| Tiêu Chí                | Declarative Pipeline  | Scripted Pipeline     |
| ----------------------- | --------------------- | --------------------- |
| **Cú pháp**             | Cấu trúc cố định      | Groovy DSL tự do      |
| **Độ khó học**          | Thấp                  | Trung bình — cao      |
| **Validation**          | Được validate trước   | Chỉ phát hiện lỗi khi chạy |
| **Linh hoạt**           | Trung bình            | Cao                   |
| **Khuyến nghị**         | Hầu hết trường hợp    | Logic phức tạp        |
| **Hỗ trợ `when`**       | Có sẵn (native)       | Phải tự viết if/else  |
| **Hỗ trợ `post`**       | Có sẵn (native)       | Phải dùng try/finally |
| **Shared Libraries**    | Tương thích tốt       | Tương thích tốt       |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Mức Cơ Bản

**Q: Declarative Pipeline khác Scripted Pipeline ở điểm nào?**

> Declarative Pipeline có cú pháp cố định (`pipeline { }` block), dễ đọc, được validate trước khi chạy, phù hợp cho hầu hết trường hợp. Scripted Pipeline dùng Groovy DSL thuần túy bên trong `node { }` block, linh hoạt hơn nhưng khó học hơn. Trong thực tế, tôi ưu tiên Declarative và chỉ dùng Scripted khi cần logic điều kiện phức tạp mà Declarative không đáp ứng được.

**Q: Pipeline as Code mang lại lợi ích gì?**

> (1) Jenkinsfile được lưu trong SCM nên có lịch sử thay đổi, có thể review và rollback. (2) Developer tự quản lý pipeline của ứng dụng mình mà không cần qua Jenkins admin. (3) Tái sử dụng được qua Shared Libraries. (4) Pipeline resume được sau khi Jenkins controller khởi động lại.

### Mức Trung Bình

**Q: Làm thế nào để chạy hai test suite song song trong pipeline?**

> Dùng `parallel` block trong Declarative Pipeline:
> ```groovy
> stage('Test') {
>     parallel {
>         stage('Unit Tests') { steps { sh 'mvn test -Dtest=UnitTests' } }
>         stage('Integration Tests') { steps { sh 'mvn test -Dtest=IntegrationTests' } }
>     }
> }
> ```

**Q: Khi nào dùng `stash` / `unstash`?**

> Khi pipeline chạy trên nhiều agent khác nhau (agent per stage), file từ stage này không tự động có mặt trên agent của stage khác. `stash` lưu file tạm trên Jenkins controller, `unstash` khôi phục trên agent đích.

### Mức Nâng Cao

**Q: Multibranch Pipeline giải quyết vấn đề gì?**

> Khi dự án có nhiều nhánh (feature, hotfix, main), Multibranch Pipeline tự động phát hiện nhánh mới và tạo job tương ứng. Mỗi nhánh chạy pipeline theo Jenkinsfile của chính nhánh đó — cho phép feature branch có pipeline riêng, khác với pipeline của main.

---

**Tiếp Theo:** Đọc [1-declarative-pipeline.md](1-declarative-pipeline.md) để hiểu cú pháp Declarative Pipeline đầy đủ.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
