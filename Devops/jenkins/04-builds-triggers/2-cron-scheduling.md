# Cron Scheduling và Poll SCM — Lên Lịch Build Định Kỳ

> **Cron Scheduling** (lên lịch theo cron) và **Poll SCM** (kiểm tra định kỳ Source Control Management — Hệ Thống Quản Lý Mã Nguồn) là hai cơ chế cho phép Jenkins tự động chạy build theo thời gian. Không giống Webhook, Jenkins tự chủ động thực hiện — không phụ thuộc vào hệ thống bên ngoài gửi thông báo.

---

## Mục Lục

1. [Cú Pháp Cron của Jenkins](#cú-pháp-cron-của-jenkins)
2. [H Symbol — Ký Hiệu Hash Thông Minh](#h-symbol--ký-hiệu-hash-thông-minh)
3. [Poll SCM — Kiểm Tra Thay Đổi Định Kỳ](#poll-scm--kiểm-tra-thay-đổi-định-kỳ)
4. [Cấu Hình trong Jenkinsfile](#cấu-hình-trong-jenkinsfile)
5. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
6. [So Sánh và Lựa Chọn](#so-sánh-và-lựa-chọn)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cú Pháp Cron của Jenkins

Jenkins sử dụng cú pháp cron tương tự Unix, với **5 trường** cách nhau bằng khoảng trắng:

```
MINUTE  HOUR  DOM  MONTH  DOW
  │       │     │     │     │
  │       │     │     │     └── Day of Week (0–7, 0 và 7 đều là Chủ Nhật)
  │       │     │     └──────── Month (1–12)
  │       │     └────────────── Day of Month (1–31)
  │       └──────────────────── Hour (0–23)
  └──────────────────────────── Minute (0–59)
```

### Ký Hiệu Đặc Biệt

| Ký Hiệu | Ý Nghĩa | Ví Dụ |
|---------|---------|-------|
| `*` | Mọi giá trị | `* * * * *` — mỗi phút |
| `H` | Hash — giá trị ngẫu nhiên nhất quán (xem bên dưới) | `H * * * *` |
| `,` | Liệt kê nhiều giá trị | `0,30 * * * *` — phút 0 và 30 |
| `-` | Phạm vi giá trị | `0 9-17 * * *` — từ 9h đến 17h |
| `/` | Bước nhảy | `*/15 * * * *` — mỗi 15 phút |

### Ví Dụ Cron Phổ Biến

```
# Mỗi 15 phút
H/15 * * * *

# Mỗi giờ
H * * * *

# Hàng ngày lúc 2 giờ sáng
H 2 * * *

# Hàng ngày lúc 8 giờ sáng, Thứ Hai đến Thứ Sáu
H 8 * * 1-5

# Mỗi Chủ Nhật lúc 3 giờ sáng
H 3 * * 0

# Đầu mỗi tháng (ngày 1) lúc nửa đêm
H 0 1 * *

# Mỗi 2 giờ từ 8h đến 20h, ngày làm việc
H 8-20/2 * * 1-5
```

---

## H Symbol — Ký Hiệu Hash Thông Minh

Đây là tính năng **độc đáo của Jenkins** không có trong cron Unix chuẩn.

### Vấn Đề Khi Không Dùng H

Giả sử bạn có 50 job cùng cấu hình `0 * * * *` (chạy đầu mỗi giờ):

```
Thời điểm 00:00:00 → 50 jobs cùng bắt đầu → Build Queue quá tải
Thời điểm 01:00:00 → 50 jobs cùng bắt đầu → Build Queue quá tải
...
```

Kết quả: Jenkins trở nên chậm và không phản hồi ở những thời điểm đó.

### Cách H Giải Quyết

`H` — **Hash Symbol** — Jenkins tính giá trị dựa trên **tên job** (một hàm hash nhất quán):

```
Job "backend-service"     → H tính ra → phút 23 → chạy lúc xx:23
Job "frontend-app"        → H tính ra → phút 47 → chạy lúc xx:47
Job "notification-worker" → H tính ra → phút 11 → chạy lúc xx:11
```

**Kết quả:** 50 jobs được phân tán đều trong giờ — không còn spike tải.

### Cú Pháp H chi tiết

```
H         → Bất kỳ giá trị nào trong phạm vi hợp lệ của trường
H(0,30)   → Một giá trị ngẫu nhiên nhất quán trong [0, 30]
H/15      → Mỗi 15 đơn vị, bắt đầu từ offset ngẫu nhiên

Ví dụ:
H * * * *         → Mỗi giờ tại phút ngẫu nhiên (0–59)
H/15 * * * *      → Mỗi 15 phút, offset ngẫu nhiên
H(0,30) * * * *   → Một lần mỗi giờ, trong khoảng phút 0–30
H H * * *         → Một lần mỗi ngày, giờ và phút ngẫu nhiên
H H(0,7) * * *    → Một lần mỗi ngày, trong khoảng 0–7 giờ sáng
```

### So Sánh H vs * cho Cùng Tần Suất

```
Mục tiêu: Chạy mỗi 15 phút

❌ Không tốt: */15 * * * *
   → Mọi job đều chạy tại: 00, 15, 30, 45 phút
   → Tất cả cùng spike

✅ Tốt hơn: H/15 * * * *
   → Job A: 03, 18, 33, 48 phút
   → Job B: 07, 22, 37, 52 phút
   → Job C: 11, 26, 41, 56 phút
   → Tải phân tán đều
```

> **Quy tắc vàng:** Luôn dùng `H` thay vì giá trị cố định `0` hoặc `*` khi chỉ quan tâm đến *tần suất*, không cần *thời điểm chính xác*.

---

## Poll SCM — Kiểm Tra Thay Đổi Định Kỳ

**Poll SCM** (kiểm tra định kỳ hệ thống quản lý mã nguồn): Jenkins định kỳ kiểm tra repository Git xem có commit mới không. Nếu có → kích hoạt build. Nếu không → bỏ qua.

### Cơ Chế Hoạt Động

```
Jenkins (theo lịch)          Git Repository
      │                            │
      │── GET /refs/heads/main ───►│
      │◄── "commit: abc123" ───────│
      │                            │
      │  [So sánh với lần poll trước]
      │  abc123 ≠ def456 (lần trước)
      │  → Có thay đổi mới!
      │
      │  [Kích hoạt build]
      ▼
  Build Job
```

### Cấu Hình Poll SCM

**Trong Declarative Pipeline:**

```groovy
pipeline {
    agent any

    triggers {
        // Kiểm tra repository mỗi 5 phút
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
}
```

**Trong Freestyle Job:**

```
Job Configuration → Build Triggers
→ ✅ Poll SCM
   Schedule: H/5 * * * *
```

### Xem Lịch Sử Polling

```
Job → Polling Log (trong sidebar bên trái)
→ Hiển thị từng lần poll: thời điểm, kết quả, commit được phát hiện
```

Ví dụ output trong Polling Log:

```
Started on 10 May 2026 14:05:03
Checking connectivity to remote repository...
Fetching changes from 1 remote Git repository
Fetching upstream changes from https://github.com/user/repo.git
  > git fetch --tags --force --progress origin
Seen branch in repository origin/main
Seen 1 remote branch
  > git ls-remote -h https://github.com/user/repo.git HEAD refs/heads/*

Done. Took 2.3 sec
Changes found
```

### Poll SCM vs Webhook: Quyết Định Khi Nào

```
Jenkins accessible từ internet?
├── CÓ → Dùng Webhook (phản hồi tức thì, tải thấp)
└── KHÔNG (Jenkins sau firewall)
    ├── VPN tunnel khả dụng? → Cân nhắc webhook qua VPN
    └── Không → Dùng Poll SCM với tần suất phù hợp
        ├── Build nhanh, cần phản hồi ngay: H/5 * * * *
        └── Build chậm, không cấp bách: H/15 * * * *
```

---

## Cấu Hình trong Jenkinsfile

### Kết Hợp Cron và Poll SCM

```groovy
pipeline {
    agent any

    triggers {
        // Cron: chạy mỗi đêm lúc 2 giờ sáng (dù không có commit mới)
        cron('H 2 * * *')

        // Poll SCM: kiểm tra mỗi 10 phút trong giờ làm việc
        pollSCM('H/10 8-18 * * 1-5')
    }

    stages {
        stage('Detect Trigger Type') {
            steps {
                script {
                    // Phân biệt build do cron hay do poll SCM
                    def cause = currentBuild.getBuildCauses('hudson.triggers.TimerTrigger$TimerTriggerCause')
                    if (cause) {
                        echo "Build kích hoạt bởi: Cron Schedule (lịch định kỳ)"
                    } else {
                        echo "Build kích hoạt bởi: Poll SCM (phát hiện thay đổi)"
                    }
                }
            }
        }

        stage('Nightly Full Test') {
            when {
                // Chỉ chạy full test suite vào ban đêm (build từ cron)
                triggeredBy 'TimerTrigger'
            }
            steps {
                sh 'mvn test -Psuite=full'
            }
        }

        stage('Quick Test') {
            when {
                // Chạy quick test khi phát hiện thay đổi code
                not { triggeredBy 'TimerTrigger' }
            }
            steps {
                sh 'mvn test -Psuite=smoke'
            }
        }
    }
}
```

### Tắt Build Định Kỳ Trên Nhánh PR

```groovy
pipeline {
    agent any

    triggers {
        // Chỉ kích hoạt trên nhánh chính, không poll trên feature branch
        pollSCM(env.BRANCH_NAME == 'main' ? 'H/10 * * * *' : '')
    }

    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
}
```

### Cron cho Scheduled Reports (Báo Cáo Theo Lịch)

```groovy
pipeline {
    agent any

    // Chạy mỗi thứ Hai 7 giờ sáng để tạo báo cáo tuần
    triggers {
        cron('H 7 * * 1')
    }

    stages {
        stage('Generate Report') {
            steps {
                sh './scripts/generate-weekly-report.sh'
            }
        }

        stage('Send Email') {
            steps {
                emailext(
                    subject: "Weekly Build Report — ${new Date().format('yyyy-MM-dd')}",
                    body: readFile('report.html'),
                    mimeType: 'text/html',
                    to: 'team@example.com'
                )
            }
        }
    }
}
```

---

## Ví Dụ Thực Tế

### Ví Dụ 1 — CI Pipeline với Fallback Poll

Dùng Webhook là chính, Poll SCM là dự phòng phát hiện thay đổi bị bỏ lỡ:

```groovy
pipeline {
    agent any

    triggers {
        // Webhook xử lý hầu hết các trường hợp
        githubPush()

        // Poll mỗi giờ để không bỏ lỡ sự kiện webhook bị fail
        pollSCM('H * * * *')
    }

    stages {
        stage('Build & Test') {
            steps {
                sh 'make ci'
            }
        }
    }
}
```

### Ví Dụ 2 — Nightly Build (Build Ban Đêm)

```groovy
pipeline {
    agent {
        docker { image 'maven:3.9-eclipse-temurin-17' }
    }

    // Chạy mỗi đêm lúc 1–3 giờ sáng (H phân tán các job)
    triggers {
        cron('H 1-3 * * *')
    }

    stages {
        stage('Full Integration Test') {
            steps {
                sh 'mvn verify -Pintegration-tests'
            }
        }

        stage('Security Scan') {
            steps {
                sh 'mvn dependency-check:check'
            }
        }

        stage('Code Coverage Report') {
            steps {
                sh 'mvn jacoco:report'
                publishHTML(target: [
                    reportDir: 'target/site/jacoco',
                    reportFiles: 'index.html',
                    reportName: 'Code Coverage'
                ])
            }
        }
    }

    post {
        failure {
            emailext(
                subject: "Nightly Build FAILED — ${currentBuild.fullDisplayName}",
                body: "Chi tiết: ${env.BUILD_URL}",
                to: 'team@example.com'
            )
        }
    }
}
```

### Ví Dụ 3 — Release Build Theo Lịch

```groovy
pipeline {
    agent any

    // Release mỗi thứ Sáu lúc 4 giờ chiều
    triggers {
        cron('H 16 * * 5')
    }

    environment {
        RELEASE_VERSION = sh(
            script: "git describe --tags --abbrev=0 2>/dev/null || echo 'v0.0.0'",
            returnStdout: true
        ).trim()
    }

    stages {
        stage('Build Release') {
            steps {
                sh "make build VERSION=${RELEASE_VERSION}"
            }
        }

        stage('Create Docker Image') {
            steps {
                sh "docker build -t myapp:${RELEASE_VERSION} ."
                sh "docker push myapp:${RELEASE_VERSION}"
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh "./deploy.sh staging ${RELEASE_VERSION}"
            }
        }
    }
}
```

---

## So Sánh và Lựa Chọn

### Bảng So Sánh Đầy Đủ

| Tiêu Chí | Webhook | Poll SCM | Cron |
|----------|---------|----------|------|
| **Kích hoạt** | Sự kiện từ bên ngoài | Jenkins chủ động kiểm tra | Theo lịch cố định |
| **Độ trễ** | 1–3 giây | Theo chu kỳ poll | Đúng lịch |
| **Điều kiện** | Phụ thuộc vào thay đổi code | Phụ thuộc vào thay đổi code | Không điều kiện |
| **Tải mạng** | Rất thấp | Trung bình (liên tục poll) | Thấp |
| **Yêu cầu** | Jenkins accessible từ GitHub/GitLab | Chỉ cần Jenkins đọc được repo | Không yêu cầu gì thêm |
| **Trường hợp dùng** | CI/CD mọi push/PR | Jenkins sau firewall | Nightly build, báo cáo |

### Quyết Định Sử Dụng

```
Câu hỏi 1: Bạn cần build MỌI thay đổi code?
├── CÓ → Webhook (hoặc Poll SCM nếu firewall)
└── KHÔNG → Tiếp tục câu hỏi 2

Câu hỏi 2: Bạn cần build theo THỜI GIAN CỐ ĐỊNH?
├── CÓ → Cron (ví dụ: nightly build, weekly report)
└── KHÔNG → Không cần trigger tự động

Câu hỏi 3: Jenkins có thể nhận HTTP từ GitHub?
├── CÓ → Webhook
└── KHÔNG → Poll SCM

Câu hỏi 4: Cần cả hai (CI + nightly)?
└── Kết hợp: githubPush() + cron('H 2 * * *')
```

---

## Câu Hỏi Phỏng Vấn

**Q: H trong cron Jenkins là gì? Tại sao nên dùng H thay vì số cụ thể?**

> `H` là Hash Symbol — Jenkins tính giá trị dựa trên tên job thông qua hàm hash, tạo ra offset ngẫu nhiên nhưng nhất quán cho mỗi job. Lý do dùng `H`: khi nhiều job cùng cấu hình `0 * * * *`, tất cả chạy đúng phút 0 mỗi giờ, tạo ra spike tải đột ngột cho Jenkins và hệ thống. `H * * * *` phân tán chúng ra các phút khác nhau trong giờ, giữ tải ổn định. `H/15` đảm bảo mỗi 15 phút một lần nhưng không phải tất cả job đều chạy cùng lúc.

**Q: Poll SCM hoạt động như thế nào và có hiệu quả không?**

> Poll SCM định kỳ gửi request đến Git repository để so sánh HEAD commit hiện tại với lần poll trước. Nếu khác → có thay đổi → trigger build. Không hiệu quả bằng Webhook vì: liên tục tạo API request dù không có thay đổi, gây tải lên Git server; độ trễ bằng với khoảng cách giữa hai lần poll (poll 5 phút một lần = trễ tối đa 5 phút). Tuy nhiên vẫn hữu ích khi Jenkins nằm sau firewall không thể nhận Webhook.

**Q: Làm sao phân biệt build được kích hoạt bởi Cron vs bởi người dùng thủ công trong Jenkinsfile?**

> Dùng `currentBuild.getBuildCauses()` để lấy danh sách nguyên nhân kích hoạt. Cron triggers có class `hudson.triggers.TimerTrigger$TimerTriggerCause`, manual triggers có `hudson.model.Cause$UserIdCause`. Trong Declarative Pipeline, dùng điều kiện `when { triggeredBy 'TimerTrigger' }` để chạy stage khác nhau tùy theo nguyên nhân kích hoạt.

**Q: Khi nào nên kết hợp cả Webhook và Poll SCM cho cùng một job?**

> Kết hợp cả hai tạo ra cơ chế dự phòng: Webhook xử lý 99% trường hợp tức thì; Poll SCM định kỳ (mỗi giờ hoặc vài giờ) đảm bảo không bỏ lỡ sự kiện nếu Webhook bị fail tạm thời (network issue, GitHub outage). Đây là kiến trúc defense-in-depth — chấp nhận chi phí polling thấp để đổi lấy độ tin cậy cao hơn.

---

**Liên Kết Liên Quan:**
- [1-webhooks.md](1-webhooks.md) — Webhook: phương pháp ưu tiên hàng đầu
- [3-build-parameters.md](3-build-parameters.md) — Tham số hóa build thủ công

**Cập Nhật:** 2026-05-10
