# 2. Workflow Notifications — Thông Báo Khi Workflow Chạy

> Tích hợp thông báo (notifications) tự động qua Slack, email, Microsoft Teams, và GitHub Issues — để team biết ngay khi CI/CD có sự cố.

---

## 📋 Mục Lục

1. [Tổng Quan Chiến Lược Thông Báo](#1-tổng-quan-chiến-lược-thông-báo)
2. [Slack Notifications](#2-slack-notifications)
3. [Microsoft Teams Notifications](#3-microsoft-teams-notifications)
4. [Email Notifications](#4-email-notifications)
5. [GitHub Issues / Pull Request Comments](#5-github-issues--pull-request-comments)
6. [PagerDuty và On-Call Alerts](#6-pagerduty-và-on-call-alerts)
7. [Custom Webhooks](#7-custom-webhooks)
8. [Template Thông Báo Thực Tế](#8-template-thông-báo-thực-tế)
9. [Best Practices](#9-best-practices)

---

## 1. Tổng Quan Chiến Lược Thông Báo

### Nguyên Tắc Chọn Kênh Thông Báo

| Kênh | Dùng Khi | Ưu Điểm | Nhược Điểm |
|---|---|---|---|
| Slack | Thông báo team tức thì | Real-time, thread reply | Cần workspace setup |
| Email | Báo cáo định kỳ, compliance | Tự động, ghi lại | Dễ bị bỏ qua |
| GitHub Issues | Bug tracking từ CI | Gắn liền với code | Tạo noise nếu lạm dụng |
| Teams | Môi trường Microsoft | Tích hợp sâu với Office | Cấu hình phức tạp |
| PagerDuty | Production alerts | On-call rotation | Tốn kém, over-engineering nếu không cần |

### Khi Nào Nên Gửi Thông Báo

```
✅ Gửi thông báo:
  - Workflow fail trên nhánh main/master/release
  - Deployment production thành công hoặc thất bại
  - Security scan phát hiện lỗ hổng nghiêm trọng
  - SLA breach (vi phạm thỏa thuận mức độ dịch vụ)

❌ Không nên gửi thông báo:
  - Mỗi lần workflow chạy thành công (quá nhiều noise)
  - PR feature branch fail (chỉ notify tác giả PR)
  - Scheduled jobs ổn định (chỉ alert khi fail)
```

---

## 2. Slack Notifications

### Cách 1 — Dùng action/slack-send (Khuyến Nghị)

```yaml
name: CI với Slack Notifications

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        id: tests
        run: npm test

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v2
        with:
          channel-id: '#deployments'
          slack-message: |
            ❌ *Build Failed* on `${{ github.repository }}`
            Branch: `${{ github.ref_name }}`
            Commit: ${{ github.sha }}
            Actor: ${{ github.actor }}
            <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v2
        with:
          channel-id: '#deployments'
          slack-message: "✅ Build passed on `${{ github.ref_name }}` by ${{ github.actor }}"
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Cách 2 — Dùng Incoming Webhook (Đơn Giản Hơn)

```yaml
- name: Notify Slack via webhook
  if: failure()
  run: |
    curl -X POST \
      -H 'Content-type: application/json' \
      --data '{
        "text": "❌ Workflow failed!",
        "attachments": [
          {
            "color": "danger",
            "fields": [
              {"title": "Repository", "value": "${{ github.repository }}", "short": true},
              {"title": "Branch", "value": "${{ github.ref_name }}", "short": true},
              {"title": "Commit", "value": "${{ github.sha }}", "short": true},
              {"title": "Actor", "value": "${{ github.actor }}", "short": true}
            ],
            "actions": [
              {"type": "button", "text": "View Run", "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}
            ]
          }
        ]
      }' \
      ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Slack Block Kit — Thông Báo Đẹp Hơn

```yaml
- name: Send rich Slack notification
  if: always()
  uses: slackapi/slack-github-action@v2
  with:
    channel-id: ${{ secrets.SLACK_CHANNEL_ID }}
    payload: |
      {
        "blocks": [
          {
            "type": "header",
            "text": {
              "type": "plain_text",
              "text": "${{ job.status == 'success' && '✅' || '❌' }} Deployment ${{ job.status }}"
            }
          },
          {
            "type": "section",
            "fields": [
              {"type": "mrkdwn", "text": "*Repository:*\n${{ github.repository }}"},
              {"type": "mrkdwn", "text": "*Branch:*\n${{ github.ref_name }}"},
              {"type": "mrkdwn", "text": "*Triggered by:*\n${{ github.actor }}"},
              {"type": "mrkdwn", "text": "*Run ID:*\n${{ github.run_id }}"}
            ]
          },
          {
            "type": "actions",
            "elements": [
              {
                "type": "button",
                "text": {"type": "plain_text", "text": "View Details"},
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            ]
          }
        ]
      }
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Cấu Hình Slack Bot Token

```
1. Truy cập https://api.slack.com/apps
2. Create New App → From scratch
3. Đặt tên, chọn workspace
4. OAuth & Permissions → Bot Token Scopes:
   - chat:write
   - chat:write.public (nếu muốn gửi vào public channels)
5. Install to Workspace → Copy Bot User OAuth Token
6. Thêm vào GitHub Secrets: SLACK_BOT_TOKEN
```

---

## 3. Microsoft Teams Notifications

```yaml
- name: Notify Microsoft Teams
  if: failure()
  run: |
    curl -H 'Content-Type: application/json' \
      -d '{
        "@type": "MessageCard",
        "@context": "https://schema.org/extensions",
        "themeColor": "FF0000",
        "summary": "GitHub Actions Build Failed",
        "sections": [{
          "activityTitle": "❌ Build Failed",
          "activitySubtitle": "${{ github.repository }} — ${{ github.ref_name }}",
          "facts": [
            {"name": "Repository", "value": "${{ github.repository }}"},
            {"name": "Branch", "value": "${{ github.ref_name }}"},
            {"name": "Commit", "value": "${{ github.sha }}"},
            {"name": "Triggered by", "value": "${{ github.actor }}"}
          ]
        }],
        "potentialAction": [{
          "@type": "OpenUri",
          "name": "View Run",
          "targets": [{"os": "default", "uri": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}]
        }]
      }' \
      ${{ secrets.TEAMS_WEBHOOK_URL }}
```

---

## 4. Email Notifications

### GitHub Built-in Email (Không Cần Cấu Hình)

GitHub tự động gửi email khi:
- Workflow fail trên nhánh bạn đang watch
- Workflow được tắt tự động do liên tục fail

Cài đặt tại: **Profile → Settings → Notifications → GitHub Actions**

### Email Tùy Chỉnh Qua SendGrid

```yaml
- name: Send email notification
  if: failure()
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.sendgrid.net
    server_port: 587
    username: apikey
    password: ${{ secrets.SENDGRID_API_KEY }}
    subject: "❌ Build Failed: ${{ github.repository }} (${{ github.ref_name }})"
    to: devops-team@company.com
    from: ci-bot@company.com
    html_body: |
      <h2>Build Failed</h2>
      <table>
        <tr><td><b>Repository</b></td><td>${{ github.repository }}</td></tr>
        <tr><td><b>Branch</b></td><td>${{ github.ref_name }}</td></tr>
        <tr><td><b>Commit</b></td><td>${{ github.sha }}</td></tr>
        <tr><td><b>Actor</b></td><td>${{ github.actor }}</td></tr>
      </table>
      <p><a href="${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}">View Run Details</a></p>
```

---

## 5. GitHub Issues / Pull Request Comments

### Tự Động Tạo Issue Khi Production Fail

```yaml
name: Production Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        id: deploy
        run: ./scripts/deploy.sh

      - name: Create GitHub Issue on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const issue = await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🚨 Production deployment failed — ${context.sha.substring(0, 7)}`,
              body: `## Production Deployment Failed
              
              **Branch:** ${context.ref}
              **Commit:** ${context.sha}
              **Actor:** ${context.actor}
              **Run:** [View Details](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})
              
              ## Next Steps
              - [ ] Investigate the failure
              - [ ] Rollback if needed
              - [ ] Fix the issue
              - [ ] Re-deploy
              `,
              labels: ['bug', 'production', 'ci-failure'],
              assignees: [context.actor]
            });
            
            console.log(`Created issue #${issue.data.number}: ${issue.data.html_url}`);
```

### Comment Trên Pull Request

```yaml
- name: Comment PR with test results
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const coverage = '${{ steps.test.outputs.coverage }}';
      const status = '${{ job.status }}';
      const emoji = status === 'success' ? '✅' : '❌';
      
      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `## ${emoji} CI Results
        
        | Check | Status |
        |---|---|
        | Tests | ${status === 'success' ? '✅ Passed' : '❌ Failed'} |
        | Coverage | ${coverage}% |
        
        [View Full Run](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})
        `
      });
```

---

## 6. PagerDuty và On-Call Alerts

```yaml
- name: Trigger PagerDuty alert
  if: failure()
  run: |
    curl -X POST \
      -H 'Content-Type: application/json' \
      -d '{
        "routing_key": "${{ secrets.PAGERDUTY_ROUTING_KEY }}",
        "event_action": "trigger",
        "payload": {
          "summary": "GitHub Actions deployment failed: ${{ github.repository }}",
          "severity": "critical",
          "source": "github-actions",
          "component": "${{ github.repository }}",
          "group": "CI/CD",
          "custom_details": {
            "branch": "${{ github.ref_name }}",
            "commit": "${{ github.sha }}",
            "actor": "${{ github.actor }}",
            "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
        }
      }' \
      https://events.pagerduty.com/v2/enqueue
```

---

## 7. Custom Webhooks

### Gửi Đến Hệ Thống Nội Bộ

```yaml
- name: Notify internal system
  if: always()
  run: |
    curl -X POST \
      -H 'Content-Type: application/json' \
      -H "Authorization: Bearer ${{ secrets.INTERNAL_API_TOKEN }}" \
      -d '{
        "event": "workflow_completed",
        "status": "${{ job.status }}",
        "repository": "${{ github.repository }}",
        "branch": "${{ github.ref_name }}",
        "commit_sha": "${{ github.sha }}",
        "actor": "${{ github.actor }}",
        "run_id": "${{ github.run_id }}",
        "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
        "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"
      }' \
      https://internal.company.com/api/ci-events
```

---

## 8. Template Thông Báo Thực Tế

### Pattern Tổng Hợp (Notify Nhiều Kênh)

```yaml
name: Deployment Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        id: deploy
        run: ./deploy.sh

  notify:
    needs: deploy
    runs-on: ubuntu-latest
    if: always()   # luôn chạy dù deploy thành công hay thất bại
    steps:
      - name: Set notification vars
        id: vars
        run: |
          if [ "${{ needs.deploy.result }}" == "success" ]; then
            echo "emoji=✅" >> $GITHUB_OUTPUT
            echo "color=good" >> $GITHUB_OUTPUT
            echo "status=SUCCESS" >> $GITHUB_OUTPUT
          else
            echo "emoji=❌" >> $GITHUB_OUTPUT
            echo "color=danger" >> $GITHUB_OUTPUT
            echo "status=FAILED" >> $GITHUB_OUTPUT
          fi

      - name: Notify Slack
        uses: slackapi/slack-github-action@v2
        with:
          channel-id: '#deployments'
          slack-message: "${{ steps.vars.outputs.emoji }} *Deploy ${{ steps.vars.outputs.status }}* — `${{ github.repository }}` on `${{ github.ref_name }}`"
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

      - name: Create issue on failure
        if: needs.deploy.result == 'failure'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: '🚨 Production deploy failed',
              body: `Deploy failed for commit ${context.sha}\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['bug', 'production']
            });
```

---

## 9. Best Practices

### Giảm Notification Fatigue (Mệt Mỏi Vì Thông Báo Quá Nhiều)

```yaml
# Chỉ thông báo khi trạng thái thay đổi (fail → pass hoặc pass → fail)
# Cần lưu trạng thái trước đó

- name: Check previous status
  id: prev
  run: |
    PREV=$(gh run list --branch main --limit 2 --json conclusion --jq '.[1].conclusion' 2>/dev/null || echo "success")
    echo "previous=$PREV" >> $GITHUB_OUTPUT
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

- name: Notify only on status change
  if: |
    (job.status == 'failure' && steps.prev.outputs.previous == 'success') ||
    (job.status == 'success' && steps.prev.outputs.previous == 'failure')
  run: echo "Notify team about status change"
```

### Cấu Trúc Secrets Cần Có

```
Repository Secrets (Bí Mật Cần Cấu Hình):
  SLACK_BOT_TOKEN        — OAuth token của Slack Bot
  SLACK_WEBHOOK_URL      — URL webhook (nếu dùng Incoming Webhooks)
  SLACK_CHANNEL_ID       — ID kênh Slack
  TEAMS_WEBHOOK_URL      — URL webhook Microsoft Teams
  SENDGRID_API_KEY       — API key SendGrid để gửi email
  PAGERDUTY_ROUTING_KEY  — Routing key PagerDuty
```

### Quy Tắc Vàng

1. **Alert on main, không phải feature branches** — tránh spam
2. **Luôn kèm link đến run** — giúp tìm logs nhanh
3. **Context đủ trong message** — repo, branch, actor, commit SHA
4. **Phân loại mức độ** — info cho success, warning cho slow runs, critical cho production fail
5. **Test notification workflow riêng** — dùng `workflow_dispatch` để test mà không ảnh hưởng CI

---

## 🔗 Liên Kết Liên Quan

- [1-debug-logging.md](./1-debug-logging.md) — Debug khi nhận thông báo lỗi
- [3-metrics-observability.md](./3-metrics-observability.md) — Dashboard theo dõi xu hướng
- [4-audit-logs.md](./4-audit-logs.md) — Ghi lại lịch sử thao tác

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
