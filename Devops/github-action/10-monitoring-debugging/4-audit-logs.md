# 4. Audit Logs — Nhật Ký Kiểm Toán GitHub Actions

> Audit logs (nhật ký kiểm toán) ghi lại toàn bộ hoạt động trong tổ chức — ai làm gì, khi nào, ở đâu — phục vụ compliance (tuân thủ quy định), bảo mật, và điều tra sự cố.

---

## 📋 Mục Lục

1. [Audit Logs Là Gì và Tại Sao Quan Trọng](#1-audit-logs-là-gì-và-tại-sao-quan-trọng)
2. [Truy Cập Audit Logs Trên GitHub](#2-truy-cập-audit-logs-trên-github)
3. [Audit Events Liên Quan GitHub Actions](#3-audit-events-liên-quan-github-actions)
4. [GitHub API Cho Audit Logs](#4-github-api-cho-audit-logs)
5. [Export và Streaming Audit Logs](#5-export-và-streaming-audit-logs)
6. [Tích Hợp SIEM](#6-tích-hợp-siem)
7. [Compliance Frameworks](#7-compliance-frameworks)
8. [Điều Tra Sự Cố Qua Audit Logs](#8-điều-tra-sự-cố-qua-audit-logs)
9. [Best Practices](#9-best-practices)

---

## 1. Audit Logs Là Gì và Tại Sao Quan Trọng

### Định Nghĩa

**Audit log** (nhật ký kiểm toán) là bản ghi bất biến (immutable record) ghi lại mọi hành động quan trọng trong hệ thống:

```
Ai (Actor)  + Làm gì (Action)  + Với cái gì (Resource)  + Khi nào (Timestamp)  + Từ đâu (IP/Location)
```

### Tại Sao Cần Cho GitHub Actions

| Trường Hợp | Audit Log Giúp Gì |
|---|---|
| Secrets bị lộ | Ai đã truy cập/chỉnh sửa secret? Khi nào? |
| Workflow bị sửa | Ai thêm/xóa bước trong workflow? |
| Runner không mong muốn | Runner lạ nào đã chạy job quan trọng? |
| Compliance audit | Bằng chứng ai deploy lên production, có approval không? |
| Insider threat | Phát hiện hành vi bất thường của thành viên team |

### Phân Biệt Audit Logs vs Workflow Logs

| | Audit Logs | Workflow Logs |
|---|---|---|
| Nội dung | Hành động người dùng (quản lý) | Output của steps (kỹ thuật) |
| Ai tạo ra | GitHub platform | Runner |
| Retention | Tối đa 7 năm (Enterprise) | Tối đa 400 ngày |
| Truy cập | Owner/Admin tổ chức | Ai có quyền read repo |
| Format | JSON structured | Plain text |

---

## 2. Truy Cập Audit Logs Trên GitHub

### Qua GitHub UI

```
Organization → Settings → Audit log

Hoặc URL trực tiếp:
https://github.com/organizations/YOUR-ORG/settings/audit-log
```

### Tính Năng Tìm Kiếm

```
# Lọc theo action
action:workflows.run_started

# Lọc theo actor (người thực hiện)
actor:john.doe

# Lọc theo thời gian
created:>2026-01-01

# Lọc theo IP
actor_ip:192.168.1.1

# Kết hợp
actor:john.doe action:workflows.completed created:>2026-01-01
```

### Yêu Cầu Quyền Truy Cập

| Loại Tài Khoản | Phạm Vi Audit Log |
|---|---|
| Repo Owner | Repository audit log (giới hạn) |
| Org Owner | Toàn bộ organization audit log |
| Enterprise Admin | Toàn bộ enterprise — tất cả orgs |
| GitHub Enterprise Server | Self-hosted — full control |

---

## 3. Audit Events Liên Quan GitHub Actions

### Nhóm Workflow Events (Sự Kiện Workflow)

| Event | Mô Tả |
|---|---|
| `workflows.run_started` | Workflow run bắt đầu |
| `workflows.completed` | Workflow run hoàn thành (success/failure) |
| `workflows.approved_workflow_job` | Job được phê duyệt thủ công |
| `workflows.cancelled_workflow_run` | Workflow run bị hủy |
| `workflows.deleted_workflow_run` | Workflow run bị xóa |

### Nhóm Secrets Events (Sự Kiện Bí Mật)

| Event | Mô Tả |
|---|---|
| `org.add_actions_secret` | Thêm secret mới vào org |
| `org.remove_actions_secret` | Xóa secret khỏi org |
| `org.update_actions_secret` | Cập nhật secret |
| `repo.create_actions_secret` | Tạo secret trong repo |
| `repo.remove_actions_secret` | Xóa secret trong repo |

### Nhóm Runner Events (Sự Kiện Runner)

| Event | Mô Tả |
|---|---|
| `org.runner_group_created` | Tạo runner group mới |
| `org.runner_group_runners_added` | Thêm runner vào group |
| `org.self_hosted_runner_online` | Self-hosted runner online |
| `org.self_hosted_runner_offline` | Self-hosted runner offline |
| `repo.register_self_hosted_runner` | Đăng ký self-hosted runner |
| `repo.remove_self_hosted_runner` | Gỡ self-hosted runner |

### Nhóm Permissions Events (Sự Kiện Quyền Hạn)

| Event | Mô Tả |
|---|---|
| `org.actions_permission_updated` | Thay đổi quyền Actions của org |
| `repo.actions_enabled` | Bật GitHub Actions cho repo |
| `repo.actions_disabled` | Tắt GitHub Actions cho repo |
| `org.allow_actions_organization` | Cho phép actions từ org khác |

---

## 4. GitHub API Cho Audit Logs

### Lấy Audit Log Qua REST API

```bash
# Org audit log (yêu cầu Owner token)
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/YOUR-ORG/audit-log?per_page=100&phrase=action:workflows"

# Lọc theo thời gian
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/YOUR-ORG/audit-log?per_page=100&after=2026-01-01T00:00:00Z"

# Lấy qua GraphQL API (Enterprise)
curl -X POST -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ organization(login: \"YOUR-ORG\") { auditLog(first: 100, query: \"action:workflows\") { nodes { ... on WorkflowRunAuditEntry { action actor createdAt } } } } }"
  }' \
  "https://api.github.com/graphql"
```

### Script Phân Tích Audit Logs

```python
#!/usr/bin/env python3
"""Phân tích audit logs để tìm hoạt động bất thường."""

import requests
from datetime import datetime, timedelta
from collections import defaultdict

GITHUB_TOKEN = "ghp_..."
ORG = "myorg"

headers = {
    "Authorization": f"Bearer {GITHUB_TOKEN}",
    "Accept": "application/vnd.github.v3+json"
}

def get_audit_logs(phrase="", days=7):
    """Lấy audit logs trong N ngày gần đây."""
    logs = []
    url = f"https://api.github.com/orgs/{ORG}/audit-log"
    params = {"per_page": 100, "phrase": phrase}
    
    while url:
        resp = requests.get(url, headers=headers, params=params)
        data = resp.json()
        
        if not isinstance(data, list):
            break
        
        # Lọc theo thời gian
        cutoff = datetime.utcnow() - timedelta(days=days)
        for entry in data:
            ts = entry.get("@timestamp", 0) / 1000  # milliseconds → seconds
            if datetime.utcfromtimestamp(ts) > cutoff:
                logs.append(entry)
            else:
                return logs  # đã quá ngưỡng thời gian, dừng
        
        # Pagination
        link = resp.headers.get("Link", "")
        url = None
        if 'rel="next"' in link:
            for part in link.split(","):
                if 'rel="next"' in part:
                    url = part.split(";")[0].strip()[1:-1]
    
    return logs

def analyze_secrets_access(logs):
    """Phân tích thay đổi secrets — cảnh báo hoạt động bất thường."""
    secret_ops = [l for l in logs if "secret" in l.get("action", "").lower()]
    
    by_actor = defaultdict(list)
    for op in secret_ops:
        by_actor[op.get("actor")].append(op)
    
    print("=== Thay Đổi Secrets ===")
    for actor, ops in by_actor.items():
        print(f"  {actor}: {len(ops)} operations")
        for op in ops:
            ts = datetime.utcfromtimestamp(op.get("@timestamp", 0) / 1000)
            print(f"    [{ts}] {op.get('action')} — {op.get('repo', {})}")

def detect_unusual_runners(logs):
    """Phát hiện self-hosted runners lạ được đăng ký."""
    runner_events = [l for l in logs if "runner" in l.get("action", "").lower()]
    
    print("\n=== Runner Events ===")
    for event in runner_events:
        ts = datetime.utcfromtimestamp(event.get("@timestamp", 0) / 1000)
        print(f"  [{ts}] {event.get('action')} by {event.get('actor')}")

logs = get_audit_logs(phrase="", days=7)
analyze_secrets_access(logs)
detect_unusual_runners(logs)
```

---

## 5. Export và Streaming Audit Logs

### Export Thủ Công (GitHub UI)

```
Audit Log → Export → JSON hoặc CSV
Tối đa 100,000 entries mỗi export
```

### Audit Log Streaming (Enterprise Feature)

GitHub Enterprise Cloud hỗ trợ stream audit logs real-time đến:
- Amazon S3
- Azure Blob Storage
- Google Cloud Storage
- Datadog
- Splunk

```
Organization Settings → Audit log → Log streaming → Configure

Cấu hình sẽ push mọi audit event đến destination ngay lập tức
```

### Tự Động Export Qua Workflow

```yaml
name: Export Audit Logs Daily

on:
  schedule:
    - cron: '0 1 * * *'   # 1:00 AM mỗi ngày (UTC)

jobs:
  export-audit-logs:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4

      - name: Fetch yesterday's audit logs
        run: |
          YESTERDAY=$(date -d 'yesterday' -u +%Y-%m-%d)
          
          curl -s -H "Authorization: Bearer ${{ secrets.ORG_ADMIN_TOKEN }}" \
            "https://api.github.com/orgs/${{ github.repository_owner }}/audit-log?per_page=100&phrase=created:${YESTERDAY}" \
            > "audit-logs/audit-${YESTERDAY}.json"
          
          echo "Exported $(jq length audit-logs/audit-${YESTERDAY}.json) entries"

      - name: Commit and push
        run: |
          git config --global user.email "ci@company.com"
          git config --global user.name "CI Bot"
          git add audit-logs/
          git commit -m "chore: audit logs $(date -d 'yesterday' -u +%Y-%m-%d)" || echo "No changes"
          git push
```

---

## 6. Tích Hợp SIEM

**SIEM** — Security Information and Event Management (Hệ Thống Quản Lý Thông Tin và Sự Kiện Bảo Mật)

### Gửi Audit Logs Lên Splunk

```yaml
- name: Forward to Splunk HEC (HTTP Event Collector)
  run: |
    # Lấy audit logs
    LOGS=$(curl -s -H "Authorization: Bearer ${{ secrets.ORG_ADMIN_TOKEN }}" \
      "https://api.github.com/orgs/${{ github.repository_owner }}/audit-log?per_page=100")
    
    # Gửi từng entry đến Splunk
    echo "$LOGS" | jq -c '.[]' | while read -r entry; do
      curl -X POST "${{ secrets.SPLUNK_HEC_URL }}/services/collector/event" \
        -H "Authorization: Splunk ${{ secrets.SPLUNK_HEC_TOKEN }}" \
        -H "Content-Type: application/json" \
        -d "{\"event\": $entry, \"sourcetype\": \"github:audit\", \"index\": \"github_actions\"}"
    done
```

### Gửi Audit Logs Lên Elastic Stack (ELK)

```yaml
- name: Forward to Elasticsearch
  run: |
    LOGS=$(curl -s -H "Authorization: Bearer ${{ secrets.ORG_ADMIN_TOKEN }}" \
      "https://api.github.com/orgs/${{ github.repository_owner }}/audit-log?per_page=100")
    
    echo "$LOGS" | jq -c '.[]' | while read -r entry; do
      TIMESTAMP=$(echo "$entry" | jq -r '."@timestamp"')
      curl -X POST "${{ secrets.ELASTICSEARCH_URL }}/github-audit-$(date +%Y.%m)/_doc" \
        -H "Authorization: Basic ${{ secrets.ELASTIC_BASIC_AUTH }}" \
        -H "Content-Type: application/json" \
        -d "$entry"
    done
```

### Gửi Lên AWS CloudWatch Logs

```yaml
- name: Send to CloudWatch Logs
  run: |
    LOGS=$(curl -s -H "Authorization: Bearer ${{ secrets.ORG_ADMIN_TOKEN }}" \
      "https://api.github.com/orgs/${{ github.repository_owner }}/audit-log?per_page=100")
    
    # AWS CLI gửi logs
    aws logs create-log-group --log-group-name "/github-actions/audit" 2>/dev/null || true
    aws logs create-log-stream \
      --log-group-name "/github-actions/audit" \
      --log-stream-name "$(date +%Y-%m-%d)" 2>/dev/null || true
    
    # Chuẩn bị log events
    EVENTS=$(echo "$LOGS" | jq '[.[] | {"timestamp": (."@timestamp"), "message": (. | tostring)}]')
    
    aws logs put-log-events \
      --log-group-name "/github-actions/audit" \
      --log-stream-name "$(date +%Y-%m-%d)" \
      --log-events "$EVENTS"
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: us-east-1
```

---

## 7. Compliance Frameworks

### SOC 2 Type II (Service Organization Control)

SOC 2 yêu cầu bằng chứng kiểm soát truy cập và thay đổi:

```yaml
# Workflow thu thập bằng chứng cho SOC 2
name: SOC2 Evidence Collection

on:
  schedule:
    - cron: '0 0 1 * *'   # Ngày đầu mỗi tháng

jobs:
  collect-evidence:
    runs-on: ubuntu-latest
    steps:
      - name: Collect deployment approvals
        run: |
          # Lấy tất cả deployments được approve trong tháng qua
          curl -s -H "Authorization: Bearer ${{ secrets.ORG_ADMIN_TOKEN }}" \
            "https://api.github.com/orgs/${{ github.repository_owner }}/audit-log?phrase=action:deployments.create+action:workflows.approved" \
            > evidence/deployments-$(date +%Y-%m).json

      - name: Collect secrets changes
        run: |
          # Bằng chứng ai thay đổi secrets (change management)
          curl -s -H "Authorization: Bearer ${{ secrets.ORG_ADMIN_TOKEN }}" \
            "https://api.github.com/orgs/${{ github.repository_owner }}/audit-log?phrase=action:org.add_actions_secret+action:org.update_actions_secret" \
            > evidence/secrets-changes-$(date +%Y-%m).json

      - name: Upload to compliance storage
        run: |
          aws s3 sync evidence/ s3://compliance-evidence/github-actions/$(date +%Y-%m)/
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: us-east-1
```

### PCI DSS (Payment Card Industry Data Security Standard)

```
PCI DSS Requirement 10: Track and monitor all access to network resources

GitHub Actions áp dụng:
- Log mọi deploy đến môi trường xử lý thanh toán
- Audit trail đầy đủ với timestamp và actor
- Giữ logs tối thiểu 12 tháng, 3 tháng online
- Alert khi có thay đổi bất thường
```

### HIPAA (Health Insurance Portability and Accountability Act)

```
HIPAA Technical Safeguards — GitHub Actions:
- Kiểm soát truy cập: Ai có quyền deploy lên môi trường có PHI (Protected Health Information)?
- Audit controls: Mọi workflow chạy phải được log với actor
- Integrity: Code không bị thay đổi giữa test và deploy (sử dụng artifacts + SHA)
- Transmission security: OIDC thay vì long-lived credentials
```

---

## 8. Điều Tra Sự Cố Qua Audit Logs

### Kịch Bản: Secret Bị Lộ

```bash
# Câu hỏi cần trả lời:
# 1. Secret nào bị ảnh hưởng?
# 2. Ai đã tạo/chỉnh sửa secret gần đây?
# 3. Workflows nào có thể đã dùng secret này?

# Bước 1: Tìm thay đổi secret
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/MYORG/audit-log?phrase=action:repo.create_actions_secret+repo:MYREPO"

# Bước 2: Tìm tất cả runs đã dùng secret này
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/MYORG/MYREPO/actions/runs?per_page=100"

# Bước 3: Timeline của sự cố
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/MYORG/audit-log?phrase=actor:suspicious-user&created:>2026-01-01"
```

### Kịch Bản: Workflow Bị Sửa Trái Phép

```bash
# Tìm ai đã thay đổi file workflow
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/MYORG/MYREPO/commits?path=.github/workflows/deploy.yml&per_page=20"

# Xem diff của commit đáng ngờ
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/MYORG/MYREPO/commits/COMMIT_SHA"
```

### Kịch Bản: Runner Lạ Chạy Jobs Production

```bash
# Tìm tất cả runner registration gần đây
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/MYORG/audit-log?phrase=action:repo.register_self_hosted_runner"

# Kiểm tra runner hiện có
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/MYORG/actions/runners"
```

### Template Báo Cáo Sự Cố

```markdown
## Incident Report — GitHub Actions Security Event

**Thời Gian Phát Hiện:** 2026-05-12 14:30 UTC
**Mức Độ Nghiêm Trọng:** High / Medium / Low
**Trạng Thái:** Đang Điều Tra / Đã Xử Lý

### Tóm Tắt Sự Cố
[Mô tả ngắn gọn sự cố]

### Timeline
| Thời Gian | Sự Kiện | Nguồn |
|---|---|---|
| 14:00 | Thay đổi secret bất thường | Audit log |
| 14:15 | Workflow chạy với secret mới | Workflow log |
| 14:30 | Alert được trigger | Monitoring |

### Root Cause (Nguyên Nhân Gốc Rễ)
[Phân tích nguyên nhân]

### Impact (Tác Động)
[Những gì bị ảnh hưởng]

### Actions Taken (Biện Pháp Đã Thực Hiện)
- [ ] Rotate bị secret
- [ ] Thu hồi quyền truy cập
- [ ] Review tất cả runs từ thời điểm sự cố
- [ ] Thông báo stakeholders

### Prevention (Phòng Ngừa)
[Cải tiến để không tái diễn]
```

---

## 9. Best Practices

### Retention Policy (Chính Sách Lưu Giữ)

| Loại | Thời Gian Tối Thiểu | Ghi Chú |
|---|---|---|
| Workflow logs | 90 ngày | Có thể lên 400 ngày |
| Audit logs (GitHub) | 7 năm (Enterprise) | 180 ngày (Free/Team) |
| SIEM/external logs | Theo yêu cầu compliance | SOC2: 1 năm; PCI: 1 năm online + 2 năm offline |
| Exported evidence | 3–7 năm | Tùy framework |

### Access Control Cho Audit Logs

```
Nguyên tắc Least Privilege (Quyền Tối Thiểu):
  - Chỉ Security team và Compliance team được đọc audit logs
  - Không ai được xóa audit logs
  - Export phải được log lại (ai export, khi nào)
  - API token dùng cho audit log phải rotate định kỳ
```

### Automated Alerting Rules (Quy Tắc Cảnh Báo Tự Động)

```yaml
# Những sự kiện cần alert ngay lập tức:
HIGH_PRIORITY_EVENTS:
  - org.remove_actions_secret      # Xóa secret tổ chức
  - repo.remove_actions_secret     # Xóa secret repo
  - org.actions_permission_updated # Thay đổi quyền Actions
  - repo.register_self_hosted_runner # Runner mới lạ
  - workflows.approved_workflow_job # Ai approve deployment
```

### Kiểm Tra Định Kỳ

```
Hàng tuần:
  □ Review thay đổi secrets trong 7 ngày
  □ Kiểm tra runner registrations mới
  □ Xem tỉ lệ workflow failures bất thường

Hàng tháng:
  □ Review quyền truy cập repository (member review)
  □ Audit runner groups và permissions
  □ Export logs cho compliance archive

Hàng quý:
  □ Full compliance audit review
  □ Cập nhật alerting rules dựa trên threat landscape mới
  □ Test disaster recovery từ audit log evidence
```

---

## 🔗 Liên Kết Liên Quan

- [1-debug-logging.md](./1-debug-logging.md) — Workflow logs chi tiết kỹ thuật
- [2-workflow-notifications.md](./2-workflow-notifications.md) — Alert khi phát hiện sự kiện bất thường
- [3-metrics-observability.md](./3-metrics-observability.md) — Metrics bổ sung cho audit trail
- [../08-security/6-security-hardening.md](../08-security/6-security-hardening.md) — Tăng cường bảo mật toàn diện

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
