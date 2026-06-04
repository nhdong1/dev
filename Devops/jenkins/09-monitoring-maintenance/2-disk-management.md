# 2 — Disk Management: Quản Lý Ổ Đĩa Jenkins

> Ổ đĩa đầy là một trong những nguyên nhân phổ biến nhất khiến Jenkins bị gián đoạn trong production (môi trường thực). Mỗi lần build tích lũy log, artifact (kết quả build) và workspace (không gian làm việc) — nếu không được dọn dẹp định kỳ, dung lượng sẽ tăng không kiểm soát được. Bài này cung cấp chiến lược toàn diện để kiểm soát ổ đĩa một cách tự động.

---

## Mục Tiêu

- Hiểu cấu trúc `JENKINS_HOME` và phần nào chiếm nhiều dung lượng nhất
- Cấu hình Build Discard Policy (chính sách xóa build cũ) ở cấp job và cấp hệ thống
- Dùng Workspace Cleanup Plugin (plugin dọn dẹp workspace) đúng cách
- Thiết lập Log Rotation (luân chuyển log) để giảm dung lượng log
- Phát hiện và xử lý disk usage (sử dụng ổ đĩa) bất thường

---

## Phần 1: Cấu Trúc JENKINS_HOME Và Phân Bổ Dung Lượng

```
JENKINS_HOME/ (thường là /var/lib/jenkins hoặc /var/jenkins_home)
│
├── jobs/                            ← 60–80% dung lượng
│   └── my-pipeline/
│       ├── config.xml               ← Cấu hình job (nhỏ, vài KB)
│       └── builds/
│           ├── 1/                   ← Build #1 (có thể GB nếu có artifact lớn)
│           │   ├── log              ← Console log (log màn hình)
│           │   ├── archive/         ← Build artifacts đã archive
│           │   └── build.xml        ← Metadata của build
│           ├── 2/
│           └── ...
│
├── workspace/                       ← 20–40% dung lượng
│   └── my-pipeline/                 ← Source code clone + build output
│       ├── src/
│       ├── target/                  ← Maven: compiled classes, JAR — rất lớn
│       └── node_modules/            ← Node.js: có thể hàng GB
│
├── plugins/                         ← 1–2% (ổn định sau khi cài)
├── users/                           ← Nhỏ
├── secrets/                         ← Nhỏ (chứa credentials)
└── logs/                            ← Jenkins master log
```

### Kiểm Tra Dung Lượng Hiện Tại

```bash
# Xem tổng dung lượng JENKINS_HOME
du -sh /var/jenkins_home/

# Xem dung lượng từng thư mục con (sắp xếp từ lớn đến nhỏ)
du -sh /var/jenkins_home/*/ | sort -rh | head -20

# Xem top 10 job chiếm nhiều ổ đĩa nhất
du -sh /var/jenkins_home/jobs/*/ | sort -rh | head -10

# Xem top 10 workspace chiếm nhiều ổ đĩa nhất
du -sh /var/jenkins_home/workspace/*/ | sort -rh | head -10

# Kiểm tra ổ đĩa còn trống bao nhiêu
df -h /var/jenkins_home/
```

---

## Phần 2: Build Discard Policy — Chính Sách Xóa Build Cũ

### Cấu Hình Trong Declarative Pipeline

```groovy
// Jenkinsfile — khai báo Build Discard Policy trong pipeline
pipeline {
    agent any

    options {
        // Discard old builds (xóa build cũ) — hai tiêu chí có thể kết hợp
        buildDiscarder(logRotator(
            // Giữ tối đa N build gần nhất (tính theo số lượng)
            numToKeepStr: '10',

            // Giữ build trong vòng N ngày
            daysToKeepStr: '30',

            // Giữ tối đa N artifact (độc lập với số build được giữ)
            artifactNumToKeepStr: '3',

            // Giữ artifact trong vòng N ngày
            artifactDaysToKeepStr: '7'
        ))
    }

    stages {
        stage('Build') { steps { sh 'mvn package' } }
        stage('Archive') {
            steps {
                // archive artifacts — chỉ 3 bản gần nhất được giữ (theo artifactNumToKeepStr)
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }
}
```

### Cấu Hình Qua Giao Diện Jenkins UI

1. Vào **Job → Configure → General**
2. Tick vào **Discard old builds**
3. Điền:
   - **Days to keep builds:** Số ngày (ví dụ: 30)
   - **Max # of builds to keep:** Số build tối đa (ví dụ: 10)
   - Mở **Advanced** để cấu hình riêng cho artifacts

### Cấu Hình Global Default (Mặc Định Toàn Hệ Thống)

Jenkins có thể đặt Build Discard Policy mặc định cho tất cả job qua **Manage Jenkins → System → Global Build Discard Policy**. Tuy nhiên, cài đặt trong Jenkinsfile sẽ ghi đè cài đặt global.

```groovy
// Ví dụ thiết lập theo môi trường — production giữ ít hơn
options {
    buildDiscarder(logRotator(
        numToKeepStr:         env.BRANCH_NAME == 'main' ? '20' : '5',
        artifactNumToKeepStr: env.BRANCH_NAME == 'main' ? '5'  : '1'
    ))
}
```

### Chiến Lược Khuyến Nghị Theo Loại Pipeline

| Loại Pipeline | `numToKeepStr` | `daysToKeepStr` | `artifactNumToKeepStr` | Lý Do |
|---------------|----------------|-----------------|------------------------|-------|
| Feature branch | 5 | 7 | 1 | Branch tồn tại ngắn, không cần giữ nhiều |
| Main/develop | 20 | 30 | 5 | Cần traceability (truy vết) dài hơn |
| Release pipeline | 50 | 90 | 10 | Cần audit trail (vết kiểm toán) |
| Nightly build | 7 | 14 | 3 | Build mỗi đêm — không cần giữ lâu |

---

## Phần 3: Workspace Cleanup Plugin

### Cài Đặt

**Manage Jenkins → Plugin Manager → Available** → tìm `Workspace Cleanup`

### Dọn Workspace Sau Mỗi Build

```groovy
// Jenkinsfile — dọn workspace sau khi build hoàn thành
pipeline {
    agent any

    options {
        // Dọn workspace trước khi bắt đầu build mới
        // (thay vì sau khi kết thúc — tùy trường hợp dùng cách nào)
        skipDefaultCheckout(false)
    }

    stages {
        stage('Build') {
            steps {
                checkout scm
                sh 'mvn package'
            }
        }
    }

    post {
        // always: dọn dẹp dù build thành công hay thất bại
        always {
            cleanWs(
                // Xóa cả các file bị đánh dấu là không nên xóa
                deleteDirs: true,

                // Pattern (mẫu) — không xóa các file này
                patterns: [
                    [pattern: '.gitignore', type: 'EXCLUDE'],
                    [pattern: 'reports/**', type: 'EXCLUDE']  // Giữ lại reports
                ],

                // notFailBuild: không coi failure khi cleanWs là failure build
                notFailBuild: true
            )
        }
    }
}
```

### Chỉ Dọn File Lớn, Không Dọn Toàn Bộ

```groovy
post {
    always {
        // Chỉ xóa target/ (Maven build output) — giữ source code để debug
        sh 'rm -rf target/ node_modules/ .gradle/'
        
        // Hoặc dùng cleanWs với pattern
        cleanWs(
            patterns: [
                [pattern: 'target/**',       type: 'INCLUDE'],
                [pattern: 'node_modules/**', type: 'INCLUDE'],
                [pattern: '.gradle/**',      type: 'INCLUDE']
            ]
        )
    }
}
```

### Dọn Workspace Theo Lịch (Định Kỳ)

```groovy
// Pipeline chuyên dụng chỉ để dọn dẹp workspace — chạy theo cron
pipeline {
    agent any
    
    triggers {
        // Chạy mỗi Chủ Nhật lúc 2 giờ sáng
        cron('H 2 * * 0')
    }
    
    stages {
        stage('Cleanup All Workspaces') {
            steps {
                script {
                    // Lấy danh sách tất cả node và dọn workspace trên từng node
                    Jenkins.instance.nodes.each { node ->
                        if (node.getChannel() != null) {
                            node.getWorkspaceFor(
                                Jenkins.instance.getItem('my-pipeline')
                            )?.deleteContents()
                            echo "Đã dọn workspace trên node: ${node.displayName}"
                        }
                    }
                }
            }
        }
    }
}
```

---

## Phần 4: Log Rotation — Kiểm Soát Dung Lượng Log

### Cấu Hình Log Rotation Cho Jenkins Master Log

```xml
<!-- $JENKINS_HOME/log/log.properties — giới hạn kích thước log Jenkins master -->
handlers=java.util.logging.FileHandler
java.util.logging.FileHandler.pattern=/var/log/jenkins/jenkins.log
java.util.logging.FileHandler.limit=50000000   <!-- 50 MB mỗi file -->
java.util.logging.FileHandler.count=5           <!-- Giữ 5 file xoay vòng -->
java.util.logging.FileHandler.formatter=java.util.logging.SimpleFormatter
```

### Dùng Logrotate (Linux) Cho Jenkins Log

```bash
# /etc/logrotate.d/jenkins — cấu hình logrotate cho Jenkins
/var/log/jenkins/*.log {
    daily                    # Xoay vòng hàng ngày
    rotate 14                # Giữ 14 file log cũ
    compress                 # Nén file cũ bằng gzip
    delaycompress            # Giữ file hôm qua không nén (để dễ debug)
    missingok                # Không lỗi nếu file không tồn tại
    notifempty               # Không xoay vòng nếu file rỗng
    sharedscripts
    postrotate
        # Gửi tín hiệu cho Jenkins reload log nếu cần
        /bin/kill -HUP $(cat /var/run/jenkins/jenkins.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

---

## Phần 5: Disk Usage Plugin — Theo Dõi Sử Dụng Ổ Đĩa

### Cài Đặt

**Plugin Manager → Available** → tìm `Disk Usage`

### Xem Báo Cáo

Sau khi cài, truy cập **Manage Jenkins → Disk Usage** để xem:

- Tổng dung lượng JENKINS_HOME
- Dung lượng từng job (builds + workspace tách biệt)
- Job nào chiếm nhiều nhất

### Tích Hợp Với Monitoring

```groovy
// Pipeline kiểm tra disk usage và cảnh báo qua Slack
pipeline {
    agent any
    triggers { cron('H */6 * * *') }  // Chạy mỗi 6 giờ

    stages {
        stage('Disk Check') {
            steps {
                script {
                    // Kiểm tra phần trăm ổ đĩa còn trống
                    def diskUsage = sh(
                        script: "df /var/jenkins_home | tail -1 | awk '{print \$5}' | tr -d '%'",
                        returnStdout: true
                    ).trim().toInteger()

                    echo "Disk usage hiện tại: ${diskUsage}%"

                    if (diskUsage > 85) {
                        // Gửi cảnh báo Slack
                        slackSend(
                            channel: '#jenkins-alerts',
                            color:   'danger',
                            message: "⚠️ Jenkins disk usage đang ở mức ${diskUsage}%! Cần dọn dẹp ngay."
                        )
                        error("Disk usage quá cao: ${diskUsage}%")
                    } else if (diskUsage > 70) {
                        slackSend(
                            channel: '#jenkins-alerts',
                            color:   'warning',
                            message: "Jenkins disk usage ở mức ${diskUsage}% — cần chú ý."
                        )
                    }
                }
            }
        }
    }
}
```

---

## Phần 6: Chiến Lược Tổng Thể Và Thực Hành Tốt

### Phân Loại Pipeline Và Chính Sách Tương Ứng

```
Loại 1: CI Pipeline (tích hợp liên tục) — chạy nhiều lần mỗi ngày
  buildDiscarder: numToKeepStr='10', daysToKeepStr='7'
  Workspace: cleanWs() sau mỗi build
  Artifact: artifactNumToKeepStr='2'

Loại 2: Release Pipeline — chạy ít, artifact quan trọng
  buildDiscarder: numToKeepStr='50', daysToKeepStr='90'
  Workspace: cleanWs() sau mỗi build
  Artifact: artifactNumToKeepStr='20', artifactDaysToKeepStr='90'

Loại 3: Scheduled Job — chạy định kỳ (nightly, weekly)
  buildDiscarder: numToKeepStr='7'
  Workspace: cleanWs() sau mỗi build
  Artifact: artifactNumToKeepStr='7'
```

### Tự Động Hóa Dọn Dẹp Định Kỳ

```bash
# Thêm vào crontab trên Jenkins server
# Dọn workspace cũ (không được dùng trong 7 ngày) mỗi Chủ Nhật
0 3 * * 0  find /var/jenkins_home/workspace -maxdepth 1 -mtime +7 -type d -exec rm -rf {} \;

# Nén log Jenkins cũ hơn 3 ngày
0 4 * * *  find /var/jenkins_home/jobs -name "log" -mtime +3 -exec gzip {} \;

# Xóa các temp file Jenkins (tfile*) cũ hơn 1 ngày
0 5 * * *  find /var/jenkins_home -name "tfile*" -mtime +1 -delete
```

### Danh Sách Kiểm Tra Disk Management

```markdown
## Thiết Lập Cơ Bản
[ ] Cài Workspace Cleanup Plugin
[ ] Cài Disk Usage Plugin
[ ] Cấu hình buildDiscarder cho tất cả pipeline chính
[ ] Thêm cleanWs() vào post { always {} } của các pipeline CI

## Monitoring
[ ] Cảnh báo khi disk > 70% (warning) và > 85% (critical)
[ ] Dashboard Grafana theo dõi disk usage theo thời gian
[ ] Weekly report về job nào chiếm nhiều nhất

## Định Kỳ
[ ] Hàng tuần: review top 10 job chiếm nhiều dung lượng nhất
[ ] Hàng tháng: dọn workspace không dùng đến
[ ] Hàng quý: review và cập nhật Build Discard Policy
```

---

## Xử Lý Khẩn Cấp Khi Ổ Đĩa Đầy

```bash
# Bước 1 — Xác định thủ phạm
df -h /var/jenkins_home
du -sh /var/jenkins_home/jobs/*/builds/ | sort -rh | head -10
du -sh /var/jenkins_home/workspace/*/ | sort -rh | head -10

# Bước 2 — Dọn workspace ngay lập tức (an toàn)
find /var/jenkins_home/workspace -maxdepth 2 -name "target" -type d -exec rm -rf {} +
find /var/jenkins_home/workspace -maxdepth 2 -name "node_modules" -type d -exec rm -rf {} +

# Bước 3 — Xóa console log của các build cũ (giữ lại build.xml và artifacts)
find /var/jenkins_home/jobs -name "log" -mtime +30 -delete

# Bước 4 — Trigger Jenkins garbage collection (dọn rác) qua CLI
java -jar jenkins-cli.jar -s http://localhost:8080 groovy = <<EOF
Jenkins.instance.allItems.each { job ->
  if (job instanceof Job) {
    job.logRotate()
    println "Đã rotate: ${job.fullName}"
  }
}
EOF

# Bước 5 — Kiểm tra kết quả
df -h /var/jenkins_home
```

---

## Câu Hỏi Phỏng Vấn

1. **Ổ đĩa Jenkins server đầy lúc 3 giờ sáng khiến build thất bại. Bạn xử lý thế nào?**
   → Ngay lập tức: xóa workspace (an toàn, rebuild được), sau đó console log cũ. Giải pháp dài hạn: cấu hình buildDiscarder và cảnh báo khi disk > 70%.

2. **Sự khác biệt giữa Build Discard Policy theo số lượng và theo số ngày?**
   → Theo số lượng (`numToKeepStr='10'`): luôn giữ đúng 10 build gần nhất, phù hợp cho pipeline chạy không đều. Theo số ngày (`daysToKeepStr='30'`): xóa build cũ hơn 30 ngày, phù hợp cho compliance/audit. Nên dùng cả hai kết hợp.

3. **cleanWs() nên đặt ở đâu trong pipeline? Trước hay sau build?**
   → Thường đặt trong `post { always {} }` để dọn sau khi build. Nếu cần workspace sạch trước khi build (tránh ô nhiễm từ lần trước), có thể thêm `cleanWs()` ở đầu hoặc dùng `options { skipDefaultCheckout(true) }` + checkout sạch.

4. **Làm thế nào để artifact của release build được giữ lâu hơn artifact của feature branch?**
   → Dùng conditional trong `buildDiscarder`:
   ```groovy
   buildDiscarder(logRotator(
       artifactNumToKeepStr: env.BRANCH_NAME == 'main' ? '20' : '2'
   ))
   ```

5. **Workspace của một job đang bị lock bởi một build — làm thế nào để xóa?**
   → Đợi build kết thúc (hoặc abort nó), sau đó mới xóa workspace. Không nên xóa workspace khi build đang chạy vì có thể gây lỗi không đoán trước được.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
