# Notifications — Thông Báo Kết Quả Build Jenkins

> Notification (thông báo) là cầu nối giữa kết quả pipeline và team — giúp mọi người biết build thành công hay thất bại mà không cần xem trực tiếp Jenkins. Thông báo tốt giúp team phản ứng nhanh với sự cố và giữ vòng phản hồi (feedback loop) ngắn.

---

## Mục Lục

1. [Tổng Quan Chiến Lược Thông Báo](#1-tổng-quan-chiến-lược-thông-báo)
2. [Slack Integration](#2-slack-integration)
3. [Email (Mailer) Integration](#3-email-mailer-integration)
4. [Microsoft Teams Integration](#4-microsoft-teams-integration)
5. [Telegram Bot Integration](#5-telegram-bot-integration)
6. [Post Block — Kiểm Soát Thời Điểm Thông Báo](#6-post-block---kiểm-soát-thời-điểm-thông-báo)
7. [Thông Báo Nâng Cao](#7-thông-báo-nâng-cao)
8. [Best Practices](#8-best-practices)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Chiến Lược Thông Báo

### 1.1 Khi Nào Nên Thông Báo?

```
Chiến lược thông báo hiệu quả:

✅ Thông báo KHI NÀO:
   - Build FAIL      → Thông báo ngay lập tức cho người commit
   - Build FIX       → Thông báo khi build xanh trở lại sau khi đỏ
   - Deploy PROD     → Thông báo toàn team sau khi deploy production
   - Build UNSTABLE  → Cảnh báo về test failures

❌ KHÔNG thông báo:
   - Mỗi build thành công trên feature branch (gây spam)
   - Build bị bỏ qua (Aborted)
```

### 1.2 Nội Dung Thông Báo Cần Có

```
Thông báo build cần chứa:
  📦 Tên job / pipeline
  🔢 Build number
  🌿 Branch name / PR number
  👤 Người commit (triggered by)
  ⏱  Thời gian thực thi
  ✅/❌ Trạng thái (SUCCESS/FAILURE/UNSTABLE)
  🔗 Link tới build log (để debug)
  📝 Commit message cuối cùng
```

---

## 2. Slack Integration

### 2.1 Cài Plugin và Cấu Hình

```
Plugin Manager → Cài: "Slack Notification Plugin"
```

**Bước 1: Tạo Slack App và lấy Webhook URL**

```
Slack → Your workspace → Apps → Search "Jenkins CI"
→ Add to Slack → Chọn channel → Copy Webhook URL
   (dạng: https://hooks.slack.com/services/T.../B.../...)
```

Hoặc tạo Incoming Webhook riêng:

```
Slack API (api.slack.com) → Your Apps → Create App
→ Incoming Webhooks → Activate → Add New Webhook to Workspace
→ Chọn channel → Copy Webhook URL
```

**Bước 2: Lưu Webhook URL vào Jenkins Credentials**

```
Credentials → Add:
  Kind: Secret text
  Secret: <Slack Webhook URL>
  ID: slack-webhook-url
```

**Bước 3: Cấu hình Slack trong Jenkins**

```
Manage Jenkins → Configure System → Slack
  Workspace: mycompany
  Credential: slack-webhook-url (hoặc Slack App token)
  Default channel: #ci-cd-alerts
→ Test Connection → Send test message
```

### 2.2 Jenkinsfile — Thông Báo Slack Cơ Bản

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#ci-cd-alerts',
                color: 'good',   // xanh lá
                message: "✅ BUILD SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                         "Branch: ${env.BRANCH_NAME}\n" +
                         "Duration: ${currentBuild.durationString}\n" +
                         "Link: ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#ci-cd-alerts',
                color: 'danger',  // đỏ
                message: "❌ BUILD FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                         "Branch: ${env.BRANCH_NAME}\n" +
                         "Triggered by: ${currentBuild.getBuildCauses()[0]?.userId ?: 'automated'}\n" +
                         "Link: ${env.BUILD_URL}console"
            )
        }
        unstable {
            slackSend(
                channel: '#ci-cd-alerts',
                color: 'warning',  // vàng
                message: "⚠️ BUILD UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                         "Test failures detected. Link: ${env.BUILD_URL}testReport"
            )
        }
    }
}
```

### 2.3 Slack Blocks — Thông Báo Đẹp Hơn (Rich Formatting)

```groovy
def sendSlackNotification(String status) {
    def color = status == 'SUCCESS' ? '#36a64f' : (status == 'FAILURE' ? '#e01e5a' : '#ECB22E')
    def emoji  = status == 'SUCCESS' ? '✅' : (status == 'FAILURE' ? '❌' : '⚠️')

    def commitAuthor = sh(script: 'git log -1 --format="%an"', returnStdout: true).trim()
    def commitMsg    = sh(script: 'git log -1 --format="%s"', returnStdout: true).trim()

    slackSend(
        channel: '#ci-cd-alerts',
        color: color,
        blocks: """[
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": "${emoji} ${status}: ${env.JOB_NAME}"
                }
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": "*Build:*\\n#${env.BUILD_NUMBER}"},
                    {"type": "mrkdwn", "text": "*Branch:*\\n${env.BRANCH_NAME}"},
                    {"type": "mrkdwn", "text": "*Author:*\\n${commitAuthor}"},
                    {"type": "mrkdwn", "text": "*Duration:*\\n${currentBuild.durationString}"}
                ]
            },
            {
                "type": "section",
                "text": {"type": "mrkdwn", "text": "*Commit:* ${commitMsg}"}
            },
            {
                "type": "actions",
                "elements": [
                    {
                        "type": "button",
                        "text": {"type": "plain_text", "text": "View Build"},
                        "url": "${env.BUILD_URL}"
                    }
                ]
            }
        ]"""
    )
}

pipeline {
    agent any
    stages { /* ... */ }
    post {
        success  { script { sendSlackNotification('SUCCESS') } }
        failure  { script { sendSlackNotification('FAILURE') } }
        unstable { script { sendSlackNotification('UNSTABLE') } }
    }
}
```

---

## 3. Email (Mailer) Integration

### 3.1 Plugin Và Cấu Hình SMTP

```
Plugin Manager → Cài:
  ✅ "Email Extension Plugin" (Extended E-mail Notification — khuyến nghị)
  ✅ "Mailer Plugin" (cơ bản)
```

**Cấu hình SMTP server:**

```
Manage Jenkins → Configure System → Extended E-mail Notification
  SMTP server:    smtp.gmail.com
  SMTP port:      587
  Use SMTP Auth:  ✅
  User Name:      jenkins-notifications@mycompany.com
  Password:       <App Password>  (không dùng mật khẩu Google thường)
  Use TLS:        ✅
  Default Recipients: devops-team@mycompany.com
```

### 3.2 Jenkinsfile — Email Cơ Bản

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
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
    }

    post {
        failure {
            mail(
                to: "${env.CHANGE_AUTHOR_EMAIL ?: 'devops-team@mycompany.com'}",
                subject: "❌ Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Build ${env.BUILD_NUMBER} failed for job: ${env.JOB_NAME}

Branch: ${env.BRANCH_NAME}
Build URL: ${env.BUILD_URL}
Console Output: ${env.BUILD_URL}console

Please investigate and fix the issue.

-- Jenkins CI
                """.stripIndent()
            )
        }
        fixed {
            // 'fixed' — chỉ trigger khi build xanh trở lại sau khi đỏ
            mail(
                to: 'devops-team@mycompany.com',
                subject: "✅ Jenkins Build Fixed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build is back to green. ${env.BUILD_URL}"
            )
        }
    }
}
```

### 3.3 Email Với HTML Template (Mẫu Email HTML)

```groovy
post {
    always {
        emailext(
            to: 'devops-team@mycompany.com',
            subject: "${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            // Dùng template HTML có sẵn từ Email Extension Plugin
            body: '''${SCRIPT, template="managed:groovy-html.template"}''',
            // Đính kèm test report và console log
            attachLog: true,
            attachmentsPattern: 'target/surefire-reports/*.xml',
            mimeType: 'text/html',
            // Gửi cho người commit gần nhất
            recipientProviders: [
                [$class: 'DevelopersRecipientProvider'],     // người commit trong build
                [$class: 'RequesterRecipientProvider'],      // người trigger build
                [$class: 'FailingTestSuspectsRecipientProvider']  // người gây test fail
            ]
        )
    }
}
```

---

## 4. Microsoft Teams Integration

### 4.1 Cài Plugin

```
Plugin Manager → Cài: "Office 365 Connector" hoặc "Microsoft Teams Notification"
```

### 4.2 Tạo Incoming Webhook Trong Teams

```
Microsoft Teams → Channel → ... → Connectors
→ Incoming Webhook → Configure
→ Nhập tên: "Jenkins CI"
→ Upload icon (tùy chọn)
→ Create → Copy Webhook URL
```

### 4.3 Lưu Credentials và Cấu Hình

```
Credentials → Add:
  Kind: Secret text
  Secret: <Teams Webhook URL>
  ID: teams-webhook-url
```

### 4.4 Jenkinsfile — Teams Notification

```groovy
pipeline {
    agent any

    stages {
        stage('Deploy Production') {
            steps {
                sh './deploy.sh production'
            }
        }
    }

    post {
        success {
            office365ConnectorSend(
                webhookUrl: credentials('teams-webhook-url'),
                color: '#36a64f',
                message: "✅ **Deployment Successful**\n\n" +
                         "**Job:** ${env.JOB_NAME}\n" +
                         "**Build:** #${env.BUILD_NUMBER}\n" +
                         "**Branch:** ${env.BRANCH_NAME}\n" +
                         "[View Build](${env.BUILD_URL})"
            )
        }
        failure {
            office365ConnectorSend(
                webhookUrl: credentials('teams-webhook-url'),
                color: '#e01e5a',
                status: 'FAILED',
                message: "❌ **Build Failed** — ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                factDefinitions: [
                    [name: 'Branch',   template: env.BRANCH_NAME],
                    [name: 'Duration', template: currentBuild.durationString],
                    [name: 'Log',      template: env.BUILD_URL + 'console']
                ]
            )
        }
    }
}
```

### 4.5 Teams Via HTTP POST (Không Dùng Plugin)

```groovy
def sendTeamsNotification(String status, String message) {
    def color   = status == 'SUCCESS' ? '00FF00' : 'FF0000'
    def payload = """
    {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": "${color}",
        "summary": "Jenkins Build ${status}",
        "sections": [{
            "activityTitle": "Jenkins Build ${status}",
            "activitySubtitle": "${env.JOB_NAME} #${env.BUILD_NUMBER}",
            "facts": [
                {"name": "Branch", "value": "${env.BRANCH_NAME}"},
                {"name": "Status", "value": "${status}"},
                {"name": "Duration", "value": "${currentBuild.durationString}"}
            ]
        }],
        "potentialAction": [{
            "@type": "OpenUri",
            "name": "View Build",
            "targets": [{"os": "default", "uri": "${env.BUILD_URL}"}]
        }]
    }
    """

    withCredentials([string(credentialsId: 'teams-webhook-url', variable: 'WEBHOOK')]) {
        sh """
            curl -s -X POST \
                -H 'Content-Type: application/json' \
                -d '${payload}' \
                "${WEBHOOK}"
        """
    }
}
```

---

## 5. Telegram Bot Integration

### 5.1 Tạo Telegram Bot

```
Telegram:
1. Tìm @BotFather
2. /newbot → đặt tên → đặt username (phải kết thúc bằng 'bot')
3. Nhận Bot Token: 1234567890:AAHdqTcvCH1vGWJxfSeofSh0riPQRC3M4M4
4. Thêm bot vào group/channel muốn nhận thông báo
5. Lấy Chat ID:
   GET https://api.telegram.org/bot<TOKEN>/getUpdates
   → chat.id trong response JSON
```

### 5.2 Lưu Credentials

```
Credentials:
  ID: telegram-bot-token   → Secret text: <bot token>
  ID: telegram-chat-id     → Secret text: <chat id>
```

### 5.3 Jenkinsfile — Telegram Notification

```groovy
def sendTelegramNotification(String message) {
    withCredentials([
        string(credentialsId: 'telegram-bot-token', variable: 'BOT_TOKEN'),
        string(credentialsId: 'telegram-chat-id',   variable: 'CHAT_ID')
    ]) {
        sh """
            curl -s -X POST \
                "https://api.telegram.org/bot\${BOT_TOKEN}/sendMessage" \
                -H "Content-Type: application/json" \
                -d '{
                    "chat_id": "\${CHAT_ID}",
                    "text": "${message.replaceAll('"', '\\"')}",
                    "parse_mode": "Markdown",
                    "disable_web_page_preview": true
                }'
        """
    }
}

pipeline {
    agent any

    stages {
        stage('Build') { steps { sh 'mvn package' } }
        stage('Deploy') { steps { sh './deploy.sh' } }
    }

    post {
        success {
            script {
                sendTelegramNotification(
                    "✅ *Build Thành Công*\n" +
                    "Job: `${env.JOB_NAME}`\n" +
                    "Build: `#${env.BUILD_NUMBER}`\n" +
                    "Branch: `${env.BRANCH_NAME}`\n" +
                    "[Xem Log](${env.BUILD_URL})"
                )
            }
        }
        failure {
            script {
                sendTelegramNotification(
                    "❌ *Build Thất Bại*\n" +
                    "Job: `${env.JOB_NAME}`\n" +
                    "Build: `#${env.BUILD_NUMBER}`\n" +
                    "Branch: `${env.BRANCH_NAME}`\n" +
                    "[Xem Console](${env.BUILD_URL}console)"
                )
            }
        }
    }
}
```

---

## 6. Post Block — Kiểm Soát Thời Điểm Thông Báo

### 6.1 Các Điều Kiện Post

```groovy
post {
    always {
        // Luôn chạy — dọn dẹp, archive artifacts
    }
    success {
        // Chỉ khi build SUCCESS
    }
    failure {
        // Chỉ khi build FAILURE
    }
    unstable {
        // Chỉ khi build UNSTABLE (test failures, nhưng không fail hẳn)
    }
    aborted {
        // Chỉ khi build bị hủy thủ công
    }
    changed {
        // Khi kết quả thay đổi so với build trước (SUCCESS→FAILURE hoặc FAILURE→SUCCESS)
    }
    fixed {
        // Khi build trở lại SUCCESS sau khi FAILURE
        // Tương đương: changed + success
    }
    regression {
        // Khi build trở thành FAILURE sau khi SUCCESS
        // Tương đương: changed + failure
    }
    cleanup {
        // Chạy SAU CÙNG, sau tất cả các điều kiện khác
    }
}
```

### 6.2 Chiến Lược Thông Báo Thực Tế

```groovy
post {
    // Thông báo ngay khi có sự cố
    failure {
        slackSend(channel: '#ci-alerts', color: 'danger',
                  message: "❌ FAIL: ${env.JOB_NAME}#${env.BUILD_NUMBER} — ${env.BUILD_URL}")
    }

    // Thông báo khi "fix" — build xanh trở lại
    fixed {
        slackSend(channel: '#ci-alerts', color: 'good',
                  message: "✅ FIXED: ${env.JOB_NAME} đã xanh trở lại! ${env.BUILD_URL}")
    }

    // Production deploy — thông báo toàn team
    success {
        script {
            if (env.BRANCH_NAME == 'main') {
                slackSend(channel: '#deployments', color: 'good',
                          message: "🚀 DEPLOYED TO PRODUCTION: ${env.JOB_NAME}#${env.BUILD_NUMBER}")
                mail(to: 'team@mycompany.com',
                     subject: "Production Deploy: ${env.JOB_NAME}#${env.BUILD_NUMBER}",
                     body: "Deployment successful. Build: ${env.BUILD_URL}")
            }
        }
    }

    // Luôn dọn dẹp workspace
    cleanup {
        deleteDir()
    }
}
```

---

## 7. Thông Báo Nâng Cao

### 7.1 Thông Báo Theo Stage (Per-Stage Notification)

```groovy
pipeline {
    agent any

    stages {
        stage('Deploy Staging') {
            steps {
                script {
                    slackSend(channel: '#deployments', color: '#3498db',
                              message: "🔄 Deploying to STAGING...")
                    sh './deploy.sh staging'
                    slackSend(channel: '#deployments', color: 'good',
                              message: "✅ Staging deploy complete")
                }
            }
        }

        stage('Deploy Production') {
            steps {
                script {
                    // Thông báo trước khi chờ approval
                    slackSend(channel: '#deployments', color: '#f39c12',
                              message: "⏳ Waiting for Production approval — ${env.BUILD_URL}input")
                    input message: 'Deploy to Production?', ok: 'Deploy'
                    slackSend(channel: '#deployments', color: '#3498db',
                              message: "🚀 Deploying to PRODUCTION...")
                    sh './deploy.sh production'
                }
            }
        }
    }
}
```

### 7.2 Thông Báo Kèm Thông Tin Test

```groovy
post {
    always {
        script {
            // Lấy kết quả test từ JUnit report
            def testResults = currentBuild.testResultAction
            if (testResults) {
                def passed  = testResults.passCount
                def failed  = testResults.failCount
                def skipped = testResults.skipCount
                def total   = passed + failed + skipped

                slackSend(
                    channel: '#ci-alerts',
                    color: failed > 0 ? 'danger' : 'good',
                    message: "${failed > 0 ? '❌' : '✅'} Test Results: " +
                             "${passed}/${total} passed, ${failed} failed, ${skipped} skipped\n" +
                             "${env.BUILD_URL}testReport"
                )
            }
        }
    }
}
```

---

## 8. Best Practices

### Chống Spam Thông Báo

```groovy
// Chỉ thông báo khi build trên nhánh quan trọng
post {
    failure {
        script {
            // Không thông báo cho feature branch nhỏ
            if (env.BRANCH_NAME ==~ /^(main|develop|release\/.*)$/) {
                slackSend(channel: '#ci-alerts', color: 'danger',
                          message: "❌ FAIL: ${env.JOB_NAME}#${env.BUILD_NUMBER}")
            }
        }
    }
}
```

### Rate Limiting (Giới Hạn Tần Suất Gửi)

```groovy
// Tránh gửi nhiều thông báo liên tiếp cho cùng một sự cố
// Dùng 'changed' thay vì 'failure' để chỉ thông báo khi trạng thái thay đổi
post {
    regression {
        // Chỉ gửi khi BUILD MỚI chuyển sang trạng thái FAIL (không gửi lại nếu đã fail)
        slackSend(channel: '#ci-alerts', color: 'danger',
                  message: "🔴 Build turned RED: ${env.JOB_NAME}")
    }
    fixed {
        // Chỉ gửi khi build xanh trở lại
        slackSend(channel: '#ci-alerts', color: 'good',
                  message: "🟢 Build turned GREEN: ${env.JOB_NAME}")
    }
}
```

### Thông Tin Cần Thiết Trong Thông Báo

```groovy
// Luôn bao gồm thông tin đủ để debug mà không cần mở Jenkins
def buildInfo = """
Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Branch: ${env.BRANCH_NAME ?: 'N/A'}
Commit: ${env.GIT_COMMIT?.take(8) ?: 'N/A'}
Author: ${sh(script: 'git log -1 --format="%an"', returnStdout: true).trim()}
Duration: ${currentBuild.durationString}
URL: ${env.BUILD_URL}
"""
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Khi nào nên dùng `failure` và khi nào dùng `regression` trong post block?**

> `failure` chạy mỗi khi build fail — kể cả khi build trước đó cũng fail, dẫn đến spam thông báo. `regression` chỉ chạy khi build CHUYỂN từ thành công sang thất bại — tức là lần fail đầu tiên. Trong thực tế, dùng `regression` để cảnh báo sự cố mới và `fixed` để thông báo sự cố đã giải quyết là chiến lược tốt nhất — giảm noise và tăng signal.

**Q: Làm thế nào để gửi thông báo đến người commit gây ra lỗi?**

> Dùng `CHANGE_AUTHOR_EMAIL` environment variable (với Multibranch Pipeline) hoặc `DevelopersRecipientProvider` trong Email Extension Plugin. Với Git, có thể lấy email người commit bằng `git log -1 --format="%ae"`. Lưu ý: cần đảm bảo email trong Git config khớp với email trong hệ thống thông báo.

**Q: Slack webhook URL có nên lưu thẳng trong Jenkinsfile không?**

> Không bao giờ. Webhook URL về bản chất là secret — ai có URL đó đều có thể gửi message vào Slack channel của bạn. Lưu vào Jenkins Credentials Store loại "Secret text" và dùng `withCredentials` hoặc `credentials()` để inject. Không commit vào Git dưới bất kỳ hình thức nào.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
**Chủ Đề Tiếp Theo:** [5-sonarqube-integration.md](5-sonarqube-integration.md)
