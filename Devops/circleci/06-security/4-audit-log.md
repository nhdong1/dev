# Audit Log — Nhật Ký Kiểm Toán và Compliance

> Audit Log — Nhật Ký Kiểm Toán là tính năng CircleCI ghi lại mọi hoạt động quản trị trong tổ chức: ai đã thay đổi gì, khi nào, từ địa chỉ IP nào. Đây là công cụ thiết yếu cho việc đáp ứng các yêu cầu bảo mật (compliance — tuân thủ) và điều tra sự cố bảo mật.

## 📚 Mục Lục

1. [Audit Log là gì?](#audit-log-là-gì)
2. [Các Sự Kiện Được Ghi Lại](#các-sự-kiện-được-ghi-lại)
3. [Cấu Trúc Một Log Entry](#cấu-trúc-một-log-entry)
4. [Truy Xuất Audit Logs](#truy-xuất-audit-logs)
5. [Export và Tích Hợp SIEM](#export-và-tích-hợp-siem)
6. [Use Cases Thực Tế](#use-cases-thực-tế)
7. [Compliance Requirements](#compliance-requirements)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Audit Log là gì?

**Audit Log** (còn gọi là Audit Trail — Dấu Vết Kiểm Toán) là bản ghi bất biến (immutable — không thể thay đổi sau khi ghi) về các hành động quan trọng xảy ra trong hệ thống.

```
Ví dụ câu hỏi Audit Log có thể trả lời:
  "Ai đã xóa context production-secrets?"
  "Khi nào AWS_SECRET_ACCESS_KEY được cập nhật?"
  "Developer nào đã bật follow cho project X?"
  "Có ai đã tạo OAuth token mới không?"
  "IP nào đã truy cập CircleCI dashboard lúc 3h sáng?"
```

### Tại Sao Audit Log Quan Trọng?

| Lý Do | Chi Tiết |
|-------|---------|
| **Phát hiện bất thường** | Phát hiện thay đổi không mong muốn nhanh chóng |
| **Điều tra sự cố** | Tìm root cause — nguyên nhân gốc rễ sau khi incident xảy ra |
| **Compliance** | Đáp ứng SOC 2, PCI DSS, HIPAA, ISO 27001 |
| **Non-repudiation** — Không thể phủ nhận | Chứng minh ai đã làm gì (trong tranh chấp pháp lý) |
| **Access review** | Định kỳ xem xét ai có quyền truy cập gì |

---

## Các Sự Kiện Được Ghi Lại

CircleCI Audit Log ghi lại các nhóm sự kiện sau:

### Quản Lý Context — Context Management

| Sự Kiện | Mô Tả |
|---------|-------|
| `context.create` | Tạo context mới |
| `context.delete` | Xóa context |
| `context.env_var.create` | Thêm environment variable vào context |
| `context.env_var.delete` | Xóa environment variable khỏi context |
| `context.access.update` | Thay đổi quyền truy cập context |

### Quản Lý Project — Project Management

| Sự Kiện | Mô Tả |
|---------|-------|
| `project.follow` | Bắt đầu theo dõi project |
| `project.unfollow` | Dừng theo dõi project |
| `project.settings.update` | Cập nhật cài đặt project |
| `project.api_token.create` | Tạo API token cho project |
| `project.api_token.delete` | Xóa API token |
| `project.env_var.create` | Thêm environment variable vào project |
| `project.env_var.delete` | Xóa environment variable khỏi project |

### Quản Lý Tổ Chức — Organization Management

| Sự Kiện | Mô Tả |
|---------|-------|
| `organization.user.add` | Thêm thành viên vào tổ chức |
| `organization.user.remove` | Xóa thành viên khỏi tổ chức |
| `organization.settings.update` | Cập nhật cài đặt tổ chức |

### Xác Thực — Authentication

| Sự Kiện | Mô Tả |
|---------|-------|
| `user.login` | Người dùng đăng nhập |
| `user.logout` | Người dùng đăng xuất |
| `user.api_token.create` | Tạo personal API token |
| `user.api_token.delete` | Xóa personal API token |
| `user.authorized_application.create` | Cấp quyền cho OAuth app |
| `user.authorized_application.delete` | Thu hồi quyền OAuth app |

### Self-Hosted Runner — Máy Chạy Tự Quản Lý

| Sự Kiện | Mô Tả |
|---------|-------|
| `runner.resource_class.create` | Tạo resource class mới |
| `runner.resource_class.delete` | Xóa resource class |
| `runner.token.create` | Tạo runner token |
| `runner.token.delete` | Xóa runner token |

---

## Cấu Trúc Một Log Entry

Mỗi Audit Log entry — mục nhật ký là một JSON object với các trường sau:

```json
{
  "id": "b5e5f900-4a7b-11ee-be56-0242ac120002",
  "version": 1,
  "action": "context.env_var.create",
  "success": true,
  "request_ip": "203.0.113.42",
  "occurred_at": "2026-05-18T08:30:00.000Z",
  
  "actor": {
    "id": "actor-uuid-here",
    "login": "john.doe",
    "name": "John Doe",
    "details": {
      "type": "user"
    }
  },
  
  "target": {
    "id": "context-uuid-here",
    "name": "production-secrets",
    "details": {
      "name": "AWS_SECRET_ACCESS_KEY",
      "type": "env_var"
    }
  },
  
  "scope": {
    "id": "org-uuid-here",
    "name": "my-organization",
    "type": "organization"
  }
}
```

### Giải Thích Các Trường

| Trường | Ý Nghĩa |
|--------|---------|
| `id` | UUID duy nhất của log entry |
| `version` | Phiên bản schema (để backward compatibility) |
| `action` | Loại hành động (xem danh sách ở trên) |
| `success` | `true` nếu thành công, `false` nếu thất bại |
| `request_ip` | IP address của người thực hiện hành động |
| `occurred_at` | Thời điểm xảy ra (UTC — Giờ Phối Hợp Quốc Tế) |
| `actor` | Ai đã thực hiện hành động |
| `target` | Đối tượng bị tác động |
| `scope` | Phạm vi (organization, project) |

---

## Truy Xuất Audit Logs

### Qua CircleCI Dashboard — Giao Diện Web

```
Organization Settings → Security → Audit Log
  → Chọn khoảng thời gian
  → Lọc theo action type
  → Export CSV
```

### Qua CircleCI API

```bash
CIRCLECI_TOKEN="your-personal-api-token"
ORG_ID="your-organization-id"

# Lấy audit logs (mặc định 30 ngày gần nhất)
curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs" \
  | jq .

# Lọc theo thời gian
curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs?start_date=2026-05-01T00:00:00Z&end_date=2026-05-18T23:59:59Z" \
  | jq .

# Lọc theo action
curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs?action=context.env_var.create" \
  | jq .
```

### Tham Số Query — Query Parameters

| Tham Số | Mô Tả | Ví Dụ |
|---------|-------|-------|
| `start_date` | Từ thời điểm (ISO 8601) | `2026-05-01T00:00:00Z` |
| `end_date` | Đến thời điểm (ISO 8601) | `2026-05-18T23:59:59Z` |
| `action` | Lọc theo loại sự kiện | `context.env_var.delete` |
| `page_token` | Phân trang (next page token) | `abc123...` |

---

## Export và Tích Hợp SIEM

**SIEM — Security Information and Event Management** — Hệ Thống Quản Lý Thông Tin và Sự Kiện Bảo Mật (Splunk, Elasticsearch, Datadog Security, IBM QRadar).

### Export Tự Động vào S3

```bash
#!/bin/bash
# Script export audit logs vào S3 hàng ngày

CIRCLECI_TOKEN="$CIRCLECI_TOKEN"
ORG_ID="$CIRCLECI_ORG_ID"
S3_BUCKET="company-security-audit-logs"
DATE=$(date -u +%Y-%m-%d)
YESTERDAY=$(date -u -d yesterday +%Y-%m-%dT00:00:00Z)
TODAY=$(date -u +%Y-%m-%dT00:00:00Z)

# Lấy logs của ngày hôm qua
curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs?start_date=${YESTERDAY}&end_date=${TODAY}" \
  | jq . > "/tmp/circleci-audit-${DATE}.json"

# Upload lên S3
aws s3 cp \
  "/tmp/circleci-audit-${DATE}.json" \
  "s3://${S3_BUCKET}/circleci/${DATE}/audit-log.json"

echo "Đã export audit log ngày ${DATE} lên S3"
```

### Tích Hợp với Splunk

```bash
# Splunk HTTP Event Collector — Bộ Thu Thập Sự Kiện HTTP
SPLUNK_URL="https://splunk.company.com:8088/services/collector"
SPLUNK_HEC_TOKEN="your-hec-token"

# Lấy logs và gửi vào Splunk
curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs" \
  | jq -c '.items[]' \
  | while read event; do
      curl -s -X POST \
        -H "Authorization: Splunk $SPLUNK_HEC_TOKEN" \
        -H "Content-Type: application/json" \
        "$SPLUNK_URL" \
        -d "{\"sourcetype\": \"circleci:audit\", \"event\": $event}"
    done
```

### Tích Hợp với Datadog

```bash
# Gửi CircleCI audit logs vào Datadog Logs
DD_API_KEY="your-datadog-api-key"

curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs" \
  | jq -c '.items[]' \
  | while read event; do
      curl -X POST \
        -H "DD-API-KEY: $DD_API_KEY" \
        -H "Content-Type: application/json" \
        "https://http-intake.logs.datadoghq.com/api/v2/logs" \
        -d "[{\"message\": $event, \"service\": \"circleci\", \"tags\": [\"source:circleci\"]}]"
    done
```

### Tích Hợp với Elasticsearch + Kibana — ELK Stack

```python
#!/usr/bin/env python3
"""
Script Python xuất CircleCI Audit Logs vào Elasticsearch
"""
import requests
import json
from datetime import datetime, timedelta
from elasticsearch import Elasticsearch

CIRCLECI_TOKEN = "your-token"
ORG_ID = "your-org-id"
ES_HOST = "https://elasticsearch.company.com:9200"

# Kết nối Elasticsearch
es = Elasticsearch([ES_HOST])

# Lấy audit logs từ CircleCI API
def fetch_audit_logs(start_date, end_date):
    url = f"https://circleci.com/api/v2/organizations/{ORG_ID}/audit-logs"
    params = {
        "start_date": start_date.isoformat() + "Z",
        "end_date": end_date.isoformat() + "Z"
    }
    headers = {"Circle-Token": CIRCLECI_TOKEN}
    
    all_logs = []
    next_token = None
    
    while True:
        if next_token:
            params["page_token"] = next_token
        
        response = requests.get(url, params=params, headers=headers)
        data = response.json()
        all_logs.extend(data.get("items", []))
        
        next_token = data.get("next_page_token")
        if not next_token:
            break
    
    return all_logs

# Index vào Elasticsearch
def index_to_elasticsearch(logs):
    for log in logs:
        es.index(
            index=f"circleci-audit-{datetime.utcnow().strftime('%Y.%m')}",
            id=log["id"],
            document=log
        )
    print(f"Đã index {len(logs)} log entries vào Elasticsearch")

# Chạy hàng ngày
yesterday = datetime.utcnow() - timedelta(days=1)
today = datetime.utcnow()
logs = fetch_audit_logs(yesterday, today)
index_to_elasticsearch(logs)
```

---

## Use Cases Thực Tế

### Use Case 1: Phát Hiện Thay Đổi Bất Thường — Anomaly Detection

```bash
# Tìm tất cả lần xóa environment variables trong 24 giờ qua
curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs?action=context.env_var.delete" \
  | jq '.items[] | {time: .occurred_at, actor: .actor.login, var: .target.details.name, context: .target.name}'

# Kết quả ví dụ:
# {
#   "time": "2026-05-18T03:15:00.000Z",
#   "actor": "unknown-user",
#   "var": "AWS_SECRET_ACCESS_KEY",
#   "context": "production-secrets"
# }
# → Cảnh báo! Biến nhạy cảm bị xóa lúc 3h sáng bởi user lạ!
```

### Use Case 2: Điều Tra Incident — Incident Investigation

```bash
# Kịch bản: Pipeline production bị lỗi sau khi deploy
# Cần tìm xem có ai thay đổi gì trong 2 giờ trước đó không

START="2026-05-18T06:00:00Z"
END="2026-05-18T08:00:00Z"

curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs?start_date=${START}&end_date=${END}" \
  | jq '.items[] | select(.success == true) | {
      time: .occurred_at,
      action: .action,
      actor: .actor.login,
      target: .target.name
    }'

# Ví dụ phát hiện:
# {
#   "time": "2026-05-18T06:45:00.000Z",
#   "action": "context.env_var.create",
#   "actor": "bob.dev",
#   "target": "production-database"
# }
# → Bob đã thêm biến mới vào context production lúc 6:45!
```

### Use Case 3: Offboarding — Kiểm Tra Khi Nhân Viên Rời Công Ty

```bash
# Kiểm tra mọi hành động của một user cụ thể trong 30 ngày qua
DEPARTING_USER="john.doe"

curl -s \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/organizations/${ORG_ID}/audit-logs" \
  | jq --arg user "$DEPARTING_USER" \
    '.items[] | select(.actor.login == $user) | {
      time: .occurred_at,
      action: .action,
      target: .target.name
    }'
```

### Use Case 4: Báo Cáo Compliance — Compliance Report

```python
#!/usr/bin/env python3
"""
Tạo báo cáo SOC 2 về các thay đổi secrets trong 90 ngày
"""
import requests
from datetime import datetime, timedelta
import csv

def generate_compliance_report():
    end_date = datetime.utcnow()
    start_date = end_date - timedelta(days=90)
    
    # Các action liên quan đến secrets changes — thay đổi secrets
    secret_actions = [
        "context.env_var.create",
        "context.env_var.delete",
        "project.env_var.create",
        "project.env_var.delete"
    ]
    
    report_rows = []
    
    for action in secret_actions:
        url = f"https://circleci.com/api/v2/organizations/{ORG_ID}/audit-logs"
        params = {
            "start_date": start_date.isoformat() + "Z",
            "end_date": end_date.isoformat() + "Z",
            "action": action
        }
        
        response = requests.get(
            url, params=params,
            headers={"Circle-Token": CIRCLECI_TOKEN}
        )
        
        for item in response.json().get("items", []):
            report_rows.append({
                "Timestamp": item["occurred_at"],
                "Action": item["action"],
                "Actor": item["actor"]["login"],
                "Target": item["target"]["name"],
                "Success": item["success"],
                "IP Address": item["request_ip"]
            })
    
    # Xuất ra CSV
    with open(f"compliance-report-{end_date.strftime('%Y%m')}.csv", "w") as f:
        writer = csv.DictWriter(f, fieldnames=report_rows[0].keys())
        writer.writeheader()
        writer.writerows(report_rows)
    
    print(f"Đã tạo báo cáo compliance với {len(report_rows)} sự kiện")

generate_compliance_report()
```

---

## Compliance Requirements

### SOC 2 Type II — Kiểm Soát Hệ Thống và Tổ Chức

Audit Log CircleCI hỗ trợ các control — kiểm soát SOC 2:

```
CC6.1 — Logical Access: Ghi lại mọi thay đổi quyền truy cập
  → context.access.update, organization.user.add/remove

CC6.3 — Access Removal: Xác nhận quyền bị thu hồi kịp thời
  → organization.user.remove (timestamp quan trọng)

CC7.2 — Monitoring: Theo dõi bất thường
  → Tất cả action types, đặc biệt failed actions (success: false)

CC8.1 — Change Management: Ghi lại thay đổi cấu hình
  → project.settings.update, context.env_var.create/delete
```

### PCI DSS — Payment Card Industry Data Security Standard

```
Requirement 10.2 — Audit logging:
  10.2.1: Ghi lại individual user access → user.login
  10.2.2: Ghi lại root/admin actions → context changes by org admins
  10.2.4: Ghi lại invalid access attempts → success: false events
  10.2.5: Ghi lại access/changes to secrets → context.env_var.*

Requirement 10.3 — Log entry fields:
  ✅ User ID (actor.id, actor.login)
  ✅ Event type (action)
  ✅ Date and time (occurred_at)
  ✅ Source IP (request_ip)
  ✅ Success/failure (success)
```

### HIPAA — Health Insurance Portability and Accountability Act

```
§164.312(b) — Audit Controls:
  Implement hardware, software, and/or procedural mechanisms
  that record and examine activity in information systems
  that contain or use ePHI
  
  → CircleCI Audit Log đáp ứng yêu cầu ghi lại hoạt động
  → Kết hợp với SIEM để có audit trail hoàn chỉnh
```

---

## Best Practices

### 1. Export Thường Xuyên — Regular Export

```yaml
# CircleCI job tự động export audit logs hàng ngày
version: 2.1

jobs:
  export-audit-logs:
    docker:
      - image: cimg/python:3.11
    steps:
      - run:
          name: Export audit logs lên S3
          command: |
            python3 scripts/export-audit-logs.py \
              --org-id $CIRCLECI_ORG_ID \
              --token $CIRCLECI_TOKEN \
              --s3-bucket $AUDIT_LOG_BUCKET \
              --date $(date -u -d yesterday +%Y-%m-%d)

workflows:
  daily-export:
    triggers:
      - schedule:
          cron: "0 1 * * *"    # 1h sáng UTC mỗi ngày
          filters:
            branches:
              only: main
    jobs:
      - export-audit-logs:
          context: audit-export-creds
```

### 2. Thiết Lập Alert — Cảnh Báo Khi Có Hành Động Nghi Vấn

```python
#!/usr/bin/env python3
"""
Cảnh báo qua Slack khi có hành động nhạy cảm
"""

SENSITIVE_ACTIONS = [
    "context.env_var.delete",      # Xóa secret
    "context.delete",              # Xóa toàn bộ context
    "project.api_token.delete",    # Xóa API token
    "organization.user.remove",    # Xóa thành viên
]

def check_sensitive_actions():
    # Lấy logs 15 phút gần nhất
    end = datetime.utcnow()
    start = end - timedelta(minutes=15)
    
    logs = fetch_audit_logs(start, end)
    
    alerts = [
        log for log in logs
        if log["action"] in SENSITIVE_ACTIONS
    ]
    
    if alerts:
        message = "⚠️ *CircleCI Security Alert*\n"
        for alert in alerts:
            message += f"• `{alert['action']}` by *{alert['actor']['login']}* at {alert['occurred_at']}\n"
            message += f"  Target: {alert['target']['name']}\n"
        
        send_slack_alert(message)

# Chạy mỗi 15 phút qua cron hoặc Lambda
check_sensitive_actions()
```

### 3. Retention Policy — Chính Sách Lưu Trữ

```
CircleCI giữ Audit Logs trong bao lâu:
  Free Plan: 30 ngày
  Performance Plan: 90 ngày
  Scale Plan: 90 ngày (có thể export để lưu lâu hơn)

Khuyến nghị lưu trữ:
  SOC 2: Tối thiểu 1 năm
  PCI DSS: Tối thiểu 1 năm, online 90 ngày
  HIPAA: 6 năm
  
→ Phải tự export và lưu vào S3/archive nếu cần giữ lâu hơn
```

### 4. Least Privilege cho Audit Log Access — Quyền Tối Thiểu

```
Ai được đọc Audit Logs:
  ✅ Security team
  ✅ Compliance team
  ✅ Senior DevOps/SRE
  ❌ Regular developers (không cần thiết)
  
→ Dùng CircleCI Context với Security Groups để hạn chế
   API token dùng để đọc audit logs
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: CircleCI Audit Log ghi lại những gì?**

> **A:** Audit Log ghi lại các hành động quản trị quan trọng bao gồm: tạo/xóa/sửa Context và environment variables, thêm/xóa thành viên tổ chức, tạo/xóa API tokens, thay đổi settings dự án, và các sự kiện xác thực (login/logout). Mỗi entry chứa: ai làm (actor), làm gì (action), lên ai/cái gì (target), khi nào (timestamp), từ đâu (IP address), và kết quả (success/failure).

**Q: Audit Log khác với application logs như thế nào?**

> **A:** Application logs ghi lại hoạt động runtime của ứng dụng (errors, requests, performance). Audit Log ghi lại hành động quản trị của con người và hệ thống với mục đích accountability — trách nhiệm giải trình và compliance. Audit Log thường là immutable — không thể sửa đổi sau khi ghi, và được lưu trữ lâu dài hơn để đáp ứng yêu cầu pháp lý.

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để tích hợp CircleCI Audit Log vào quy trình SOC 2?**

> **A:** CircleCI Audit Log đáp ứng các SOC 2 control CC6.1 (logical access), CC6.3 (access removal), CC7.2 (monitoring), CC8.1 (change management). Để sẵn sàng cho audit: (1) Export logs hàng ngày vào S3 để đảm bảo retention 1 năm, (2) Tích hợp vào SIEM để có real-time alerting, (3) Định kỳ review access logs để kiểm tra principle of least privilege, (4) Document quy trình xử lý khi phát hiện bất thường.

**Q: Nếu team mở Audit Log và thấy biến `DB_PASSWORD` của production bị xóa lúc 2h sáng bởi user lạ, bạn làm gì?**

> **A (ví dụ trả lời):** Quy trình incident response — ứng phó sự cố: (1) Verify ngay — xem actor.login có phải user hợp lệ không, kiểm tra request_ip, (2) Assume worst case — rotate DB_PASSWORD ngay lập tức, (3) Kiểm tra có access nào đến DB trong khoảng thời gian đó không (DB logs), (4) Revoke session/token của user đó nếu tài khoản bị compromise, (5) Review toàn bộ logs trong 48h để xem còn hành động bất thường nào khác, (6) Báo cáo security incident theo quy trình công ty, (7) Post-mortem: tăng cường Security Groups cho production context.

---

## 🔗 Điều Hướng

| ← Trước | Vị Trí | Tiếp → |
|---------|--------|--------|
| [3-ip-ranges.md](./3-ip-ranges.md) | **4-audit-log.md** | [07-integration/](../07-integration/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Độ Khó:** ⭐⭐ Trung Bình
**Thời Gian Đọc:** ~40 phút
