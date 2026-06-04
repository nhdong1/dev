# Thông Báo Pipeline — Slack, Email, Webhook

> Khi pipeline **fail** hoặc **deploy production** xong, team cần biết ngay qua **Slack**, **email**, hoặc **webhook** tích hợp PagerDuty, Microsoft Teams, Discord.

## 📚 Mục Lục

1. [Slack Orb](#slack-orb)
2. [Thông Báo Theo Trạng Thái Job](#thông-báo-theo-trạng-thái-job)
3. [Webhooks Tùy Chỉnh](#webhooks-tùy-chỉnh)
4. [Email](#email)
5. [Thiết Kế Thông Báo Hữu Ích](#thiết-kế-thông-báo-hữu-ích)
6. [Best Practices](#best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Slack Orb

**Orb:** `circleci/slack@4.x`

```yaml
version: 2.1

orbs:
  slack: circleci/slack@4.12.0

jobs:
  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Deploy
          command: ./scripts/deploy.sh
      - slack/notify:
          event: pass
          channel: C01234567
          template: success_tagged_deploy
      - slack/notify:
          event: fail
          channel: C01234567
          mentions: "@channel"
          template: basic_fail_1
```

**Slack OAuth token** lưu trong Context (ví dụ `SLACK_ACCESS_TOKEN`), job khai báo `context: slack-notifications`.

### Template tùy chỉnh

```yaml
  custom_success:
    jobs:
      - my-job:
          post-steps:
            - slack/notify:
                custom: |
                  {
                    "blocks": [
                      {
                        "type": "section",
                        "text": {
                          "type": "mrkdwn",
                          "text": "✅ *${CIRCLE_PROJECT_REPONAME}* deploy `<< parameters.tag >>` thành công\n<${CIRCLE_BUILD_URL}|Xem build>"
                        }
                      }
                    ]
                  }
```

Biến built-in: `CIRCLE_BUILD_URL`, `CIRCLE_BRANCH`, `CIRCLE_USERNAME`, `CIRCLE_SHA1`.

---

## Thông Báo Theo Trạng Thái Job

CircleCI không có hook global “mọi pipeline” trong config đơn giản; pattern phổ biến:

| Pattern | Mô Tả |
|---------|--------|
| **post-step** trên job quan trọng | `slack/notify` sau deploy |
| **Job riêng `notify-failure`** | Chạy khi job trước fail (`when: on_fail` trong workflow — dùng job wrapper) |
| **Slack app CircleCI** | Tích hợp org-level trong Project Settings (ít linh hoạt hơn orb) |

Workflow với job thông báo lỗi (ý tưởng):

```yaml
workflows:
  ci:
    jobs:
      - test
      - build:
          requires: [test]
      - deploy:
          requires: [build]
      - notify-on-fail:
          requires:
            - deploy
          filters:
            branches:
              only: main
          # Thực tế: dùng slack orb event: fail trong cùng job deploy
          # hoặc API Slack từ step when step trước fail
```

Thực hành tốt nhất: đặt `slack/notify` **fail** trong **mọi** job deploy và test quan trọng.

---

## Webhooks Tùy Chỉnh

```yaml
      - run:
          name: Gửi webhook
          command: |
            curl -X POST "$DEPLOY_WEBHOOK_URL" \
              -H "Content-Type: application/json" \
              -d "{
                \"project\": \"${CIRCLE_PROJECT_REPONAME}\",
                \"branch\": \"${CIRCLE_BRANCH}\",
                \"sha\": \"${CIRCLE_SHA1}\",
                \"status\": \"success\",
                \"build_url\": \"${CIRCLE_BUILD_URL}\"
              }"
```

Dùng cho **PagerDuty Events API**, **Teams Incoming Webhook**, hoặc internal dashboard.

Ký HMAC nếu endpoint yêu cầu xác thực (secret trong Context).

---

## Email

CircleCI gửi email mặc định theo **Project Settings → Notifications** (ai follow project). Trong config, email tùy chỉnh thường qua:

- Webhook tới service gửi mail (SendGrid, SES)
- Hoặc orb partner

```yaml
      - run:
          when: on_fail
          command: |
            aws ses send-email \
              --from alerts@company.com \
              --destination ToAddresses=team@company.com \
              --message "Subject={Data='Pipeline Failed'},Body={Text={Data='Build ${CIRCLE_BUILD_NUM}'}}"
```

(`when: on_fail` áp dụng cho step trong job.)

---

## Thiết Kế Thông Báo Hữu Ích

Nội dung nên có:

1. **Tên project** và **nhánh**
2. **Commit SHA** ngắn + link GitHub/GitLab
3. **Link trực tiếp** `CIRCLE_BUILD_URL`
4. **Job/step fail** (từ log hoặc message cố định)
5. Chỉ `@channel` khi production / main fail — tránh noise trên nhánh feature

```
❌ Pipeline FAILED
Project: payment-api | Branch: main
Commit: a1b2c3d by @dev
Job: deploy-production (step: kubectl apply)
→ https://app.circleci.com/pipelines/...
```

---

## Best Practices

1. Channel Slack riêng: `#ci-staging`, `#ci-production`
2. Không gửi secret trong payload webhook
3. **Mute** notification cho scheduled pipeline không quan trọng (filter workflow)
4. Kết hợp **Insights** — khi flaky test, thông báo nên ghi “có thể flaky” nếu detect được
5. Test orb Slack trên nhánh dev trước khi `@channel` production

---

## Câu Hỏi Phỏng Vấn

**Slack orb vs Slack integration trong UI?**  
Orb: kiểm soát trong code, versioned, khác nhau theo job; UI: cấu hình nhanh, ít chi tiết theo workflow.

**Làm sao giảm alert fatigue?**  
Chỉ notify fail trên main/prod; gom nhiều fail liên tiếp; dùng thread thay ping mới mỗi commit.

**Webhook bảo mật thế nào?**  
HTTPS, secret header, rotate URL khi lộ, không log full URL có token.
