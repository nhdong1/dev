# Bảo Trì & Giám Sát Self-hosted Runners

> Vận hành self-hosted runners không chỉ là cài đặt một lần. File này bao gồm: update runner binary, giám sát trạng thái fleet (đội runner), troubleshooting (xử lý sự cố) khi runner offline, capacity planning (lên kế hoạch tài nguyên), và log tập trung.

---

## 📚 Mục Lục

1. [Cập Nhật Runner Binary](#cập-nhật-runner-binary)
2. [Giám Sát Trạng Thái Runner](#giám-sát-trạng-thái-runner)
3. [Troubleshooting Runner Offline](#troubleshooting-runner-offline)
4. [Capacity Planning](#capacity-planning)
5. [Log Tập Trung Cho Runner Fleet](#log-tập-trung-cho-runner-fleet)
6. [Tự Động Hóa Bảo Trì](#tự-động-hóa-bảo-trì)
7. [Runbook Xử Lý Sự Cố Phổ Biến](#runbook-xử-lý-sự-cố-phổ-biến)
8. [Bài Tập Thực Hành](#bài-tập-thực-hành)

---

## 🔄 Cập Nhật Runner Binary

### Tại Sao Phải Cập Nhật Thường Xuyên?

- GitHub Actions deprecates (loại bỏ hỗ trợ) runner versions cũ sau khoảng 6 tháng
- Runner hết hạn hỗ trợ sẽ từ chối nhận jobs
- Bản vá bảo mật được phát hành thường xuyên
- GitHub thông báo qua email khi runner của bạn sắp out-of-date (hết hạn hỗ trợ)

### Kiểm Tra Phiên Bản Hiện Tại

```bash
# Xem phiên bản runner đang chạy
cat /opt/actions-runner/package.json | grep '"version"'

# Hoặc kiểm tra qua GitHub API
curl -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runners" \
  | jq '.runners[] | {name: .name, version: .version, status: .status}'
```

### Cập Nhật Thủ Công

```bash
#!/bin/bash
# update-runner.sh

set -euo pipefail

RUNNER_DIR="/opt/actions-runner"
RUNNER_SERVICE="actions.runner.*.service"

# Lấy phiên bản mới nhất
LATEST_VERSION=$(curl -s \
  https://api.github.com/repos/actions/runner/releases/latest \
  | jq -r '.tag_name' | sed 's/v//')

# Xem phiên bản hiện tại
CURRENT_VERSION=$(cat "${RUNNER_DIR}/package.json" | jq -r '.version')

echo "Hiện tại: ${CURRENT_VERSION} | Mới nhất: ${LATEST_VERSION}"

if [[ "${CURRENT_VERSION}" == "${LATEST_VERSION}" ]]; then
  echo "Runner đã là phiên bản mới nhất. Không cần cập nhật."
  exit 0
fi

# Dừng service
sudo systemctl stop ${RUNNER_SERVICE}

# Backup config hiện tại
cp "${RUNNER_DIR}/.runner" "${RUNNER_DIR}/.runner.bak"
cp "${RUNNER_DIR}/.credentials" "${RUNNER_DIR}/.credentials.bak"

# Tải phiên bản mới
cd "${RUNNER_DIR}"
curl -sL -o runner-new.tar.gz \
  "https://github.com/actions/runner/releases/download/v${LATEST_VERSION}/actions-runner-linux-x64-${LATEST_VERSION}.tar.gz"

# Xác minh SHA
EXPECTED_SHA=$(curl -s \
  "https://github.com/actions/runner/releases/download/v${LATEST_VERSION}/actions-runner-linux-x64-${LATEST_VERSION}.tar.gz.sha256")
ACTUAL_SHA=$(sha256sum runner-new.tar.gz | awk '{print $1}')

if [[ "${EXPECTED_SHA}" != "${ACTUAL_SHA}" ]]; then
  echo "ERROR: SHA256 không khớp! Hủy cập nhật."
  rm runner-new.tar.gz
  exit 1
fi

# Giải nén (ghi đè binaries, giữ config)
tar xzf runner-new.tar.gz
rm runner-new.tar.gz

# Cài lại service scripts
sudo ./svc.sh install runner

# Khởi động lại
sudo systemctl start ${RUNNER_SERVICE}

echo "Cập nhật thành công lên phiên bản ${LATEST_VERSION}"
```

### Cập Nhật Tự Động Với GitHub Actions Workflow

```yaml
# .github/workflows/update-runners.yml
# Tự động cập nhật runner binary hàng tuần
name: Update Self-hosted Runners

on:
  schedule:
    - cron: '0 2 * * 1'    # Mỗi thứ Hai lúc 2 giờ sáng
  workflow_dispatch:

jobs:
  update:
    # Chạy trên runner KHÁC với runner đang được cập nhật
    runs-on: ubuntu-latest
    steps:
      - name: Trigger runner update via SSH
        run: |
          # Dùng SSH hoặc Systems Manager (AWS SSM) để chạy update script
          aws ssm send-command \
            --targets "Key=tag:role,Values=github-runner" \
            --document-name "AWS-RunShellScript" \
            --parameters 'commands=["/opt/actions-runner/update-runner.sh"]' \
            --region us-east-1
```

### Cập Nhật ARC (Actions Runner Controller) Trên Kubernetes

```bash
# Cập nhật ARC Controller
helm upgrade arc \
  --namespace arc-systems \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <new-version>

# Cập nhật RunnerScaleSet (runner image được tự động cập nhật trong Pod template)
helm upgrade arc-runner-set \
  --namespace arc-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --reuse-values \
  --set template.spec.containers[0].image="ghcr.io/actions/actions-runner:latest"

# Xem version changelog trước khi upgrade
helm show values \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <new-version> | grep -A5 "changelog"
```

---

## 📊 Giám Sát Trạng Thái Runner

### Giám Sát Qua GitHub API

```bash
#!/bin/bash
# check-runner-health.sh — Kiểm tra sức khỏe tất cả runners

GITHUB_PAT="${GITHUB_PAT}"
ORG="${GITHUB_ORG}"

# Lấy danh sách tất cả runners của org
RUNNERS=$(curl -s \
  -H "Authorization: token ${GITHUB_PAT}" \
  "https://api.github.com/orgs/${ORG}/actions/runners?per_page=100")

echo "=== Runner Health Report ==="
echo "Thời gian: $(date)"
echo ""

# Thống kê theo status
TOTAL=$(echo "$RUNNERS" | jq '.total_count')
ONLINE=$(echo "$RUNNERS" | jq '[.runners[] | select(.status=="online")] | length')
OFFLINE=$(echo "$RUNNERS" | jq '[.runners[] | select(.status=="offline")] | length')
BUSY=$(echo "$RUNNERS" | jq '[.runners[] | select(.busy==true)] | length')

echo "Tổng: ${TOTAL} | Online: ${ONLINE} | Offline: ${OFFLINE} | Đang chạy job: ${BUSY}"
echo ""

# Danh sách runners offline
echo "=== Runners OFFLINE (cần kiểm tra) ==="
echo "$RUNNERS" | jq -r '.runners[] | select(.status=="offline") | "\(.name) — Labels: \(.labels | map(.name) | join(","))"'

# Danh sách runners đang bận
echo ""
echo "=== Runners đang chạy job ==="
echo "$RUNNERS" | jq -r '.runners[] | select(.busy==true) | "\(.name) — Đang bận"'
```

### Prometheus Exporter Tùy Chỉnh

```python
#!/usr/bin/env python3
# github_runner_exporter.py
# Xuất metrics runner cho Prometheus

import requests
import time
from prometheus_client import start_http_server, Gauge

GITHUB_TOKEN = os.environ['GITHUB_TOKEN']
ORG = os.environ['GITHUB_ORG']
HEADERS = {
    'Authorization': f'token {GITHUB_TOKEN}',
    'Accept': 'application/vnd.github+json'
}

# Định nghĩa Prometheus metrics
runner_total = Gauge('github_runner_total', 'Tổng số runners', ['org'])
runner_online = Gauge('github_runner_online', 'Số runners online', ['org'])
runner_offline = Gauge('github_runner_offline', 'Số runners offline', ['org'])
runner_busy = Gauge('github_runner_busy', 'Số runners đang chạy job', ['org'])

def collect_metrics():
    url = f'https://api.github.com/orgs/{ORG}/actions/runners?per_page=100'
    resp = requests.get(url, headers=HEADERS)
    data = resp.json()

    runners = data.get('runners', [])
    total = data.get('total_count', 0)
    online = sum(1 for r in runners if r['status'] == 'online')
    offline = total - online
    busy = sum(1 for r in runners if r.get('busy', False))

    runner_total.labels(org=ORG).set(total)
    runner_online.labels(org=ORG).set(online)
    runner_offline.labels(org=ORG).set(offline)
    runner_busy.labels(org=ORG).set(busy)

if __name__ == '__main__':
    start_http_server(8080)
    while True:
        collect_metrics()
        time.sleep(60)    # Cập nhật mỗi phút
```

### Alerting Rules (Quy Tắc Cảnh Báo)

```yaml
# prometheus-alert-rules.yml
groups:
  - name: github-runners
    rules:
      # Cảnh báo khi có runner offline hơn 5 phút
      - alert: GitHubRunnerOffline
        expr: github_runner_offline{org="my-org"} > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "GitHub runner offline"
          description: "Có {{ $value }} runner(s) offline trong 5 phút"

      # Cảnh báo khi tất cả runners đều bận (queue buildup)
      - alert: AllRunnersbusy
        expr: |
          github_runner_busy == github_runner_online
          and
          github_runner_online > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Tất cả runners đang bận"
          description: "Queue có thể đang buildup — cân nhắc thêm runners"

      # Cảnh báo nghiêm trọng khi không có runner nào online
      - alert: NoRunnersOnline
        expr: github_runner_online == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Không có runner nào online"
          description: "Tất cả self-hosted runners đều offline — CI/CD bị ngắt"
```

### Grafana Dashboard

```json
{
  "title": "GitHub Self-hosted Runners",
  "panels": [
    {
      "title": "Runner Status Overview",
      "type": "stat",
      "targets": [
        {"expr": "github_runner_online", "legendFormat": "Online"},
        {"expr": "github_runner_offline", "legendFormat": "Offline"},
        {"expr": "github_runner_busy", "legendFormat": "Đang chạy job"}
      ]
    },
    {
      "title": "Runner Utilization (%) — Tỉ Lệ Sử Dụng",
      "type": "gauge",
      "targets": [
        {
          "expr": "github_runner_busy / github_runner_online * 100",
          "legendFormat": "Utilization"
        }
      ],
      "thresholds": [
        {"color": "green", "value": 0},
        {"color": "yellow", "value": 70},
        {"color": "red", "value": 90}
      ]
    },
    {
      "title": "Runner Count Over Time",
      "type": "timeseries",
      "targets": [
        {"expr": "github_runner_total", "legendFormat": "Total"},
        {"expr": "github_runner_online", "legendFormat": "Online"},
        {"expr": "github_runner_busy", "legendFormat": "Busy"}
      ]
    }
  ]
}
```

---

## 🔧 Troubleshooting Runner Offline

### Quy Trình Xử Lý Khi Runner Offline

```
1. Xác nhận runner offline qua GitHub UI hoặc API
2. SSH vào runner machine
3. Kiểm tra systemd service
4. Xem logs để tìm nguyên nhân
5. Sửa và khởi động lại service
6. Xác nhận runner online lại
```

### Kiểm Tra Systemd Service

```bash
# Trên runner machine
sudo systemctl status actions.runner.*.service

# Xem logs gần nhất
sudo journalctl -u actions.runner.*.service -n 100

# Xem logs theo thời gian
sudo journalctl -u actions.runner.*.service --since "30 minutes ago"

# Follow logs realtime
sudo journalctl -u actions.runner.*.service -f
```

### Nguyên Nhân Phổ Biến Và Cách Xử Lý

#### 1. Credentials Hết Hạn

```bash
# Triệu chứng trong logs:
# "Unauthorized" hoặc "403 Forbidden" hoặc "Token is expired"

# Xử lý: Re-register runner với token mới
sudo systemctl stop actions.runner.*.service

# Lấy token mới từ GitHub
NEW_TOKEN=$(curl -s -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runners/registration-token" \
  | jq -r '.token')

cd /opt/actions-runner

# Xóa đăng ký cũ
./config.sh remove --token "${NEW_TOKEN}"

# Đăng ký lại
./config.sh \
  --url "https://github.com/{owner}/{repo}" \
  --token "${NEW_TOKEN}" \
  --name "$(hostname)" \
  --labels "self-hosted,linux,x64" \
  --unattended

sudo systemctl start actions.runner.*.service
```

#### 2. Mạng Không Kết Nối Được GitHub

```bash
# Kiểm tra kết nối
curl -v https://api.github.com/zen

# Kiểm tra DNS
nslookup api.github.com

# Kiểm tra firewall
sudo iptables -L OUTPUT -n | grep 443

# Nếu dùng egress proxy — kiểm tra proxy hoạt động
curl -x https://proxy.internal:8080 https://api.github.com/zen
```

#### 3. Disk Full (Đĩa Đầy)

```bash
# Kiểm tra disk usage
df -h /opt/actions-runner

# Tìm file chiếm nhiều dung lượng
du -sh /opt/actions-runner/_work/* | sort -rh | head -10

# Xóa workspace cũ (an toàn)
rm -rf /opt/actions-runner/_work/*/

# Xóa Docker cache (nếu runner build Docker images)
docker system prune -af --volumes

# Xóa old log files
sudo journalctl --vacuum-time=7d
```

#### 4. Process Không Terminate (Job Bị Kẹt)

```bash
# Tìm runner process đang chạy
ps aux | grep Runner.Worker

# Xem runner đang chạy job nào
curl -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runners" \
  | jq '.runners[] | select(.busy==true) | {name, id}'

# Force cancel job bị kẹt qua API
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runs/{run_id}/cancel"

# Nếu process không terminate: kill nhẹ trước
sudo kill -SIGTERM <PID>

# Nếu vẫn không xong sau 30 giây
sudo kill -SIGKILL <PID>

sudo systemctl restart actions.runner.*.service
```

#### 5. Runner Version Cũ Bị Từ Chối

```bash
# Triệu chứng:
# "Job was rejected because the runner is not supported..."

# Kiểm tra version
cat /opt/actions-runner/package.json | jq '.version'

# Chạy update script
/opt/actions-runner/update-runner.sh
```

---

## 📐 Capacity Planning

### Thu Thập Dữ Liệu Sử Dụng

```bash
#!/bin/bash
# runner-utilization-report.sh
# Tạo báo cáo sử dụng runners trong 7 ngày

GITHUB_PAT="${GITHUB_PAT}"
OWNER="${GITHUB_OWNER}"
REPO="${GITHUB_REPO}"

# Lấy workflow runs trong 7 ngày
SEVEN_DAYS_AGO=$(date -d "7 days ago" --iso-8601=seconds)

curl -H "Authorization: token ${GITHUB_PAT}" \
  "https://api.github.com/repos/${OWNER}/${REPO}/actions/runs?created=>=${SEVEN_DAYS_AGO}&per_page=100" \
  | jq '
    .workflow_runs
    | group_by(.status)
    | map({
        status: .[0].status,
        count: length,
        avg_duration_seconds: (map(.updated_at | split("T")[0]) | length)
      })
  '
```

### Công Thức Tính Số Runners Cần Thiết

```
Số Runners Tối Thiểu = (Jobs Per Hour × Average Job Duration) / 60 × Buffer Factor

Ví dụ:
- 30 jobs/giờ vào giờ cao điểm
- Mỗi job chạy trung bình 8 phút
- Buffer factor (hệ số đệm): 1.3 (dự phòng 30%)

Số Runners = (30 × 8) / 60 × 1.3 = 4 × 1.3 = ~6 runners

→ Deploy 6 runners (hoặc ARC maxRunners=8 để có thêm headroom)
```

### Phân Tích Queue Wait Time (Thời Gian Chờ Hàng Đợi)

```python
# analyze-queue-time.py
import requests
from datetime import datetime, timedelta

TOKEN = os.environ['GITHUB_TOKEN']
OWNER, REPO = 'myorg', 'myrepo'

def get_workflow_timing():
    url = f'https://api.github.com/repos/{OWNER}/{REPO}/actions/runs'
    params = {'per_page': 100, 'status': 'completed'}
    resp = requests.get(url, headers={'Authorization': f'token {TOKEN}'}, params=params)

    runs = resp.json()['workflow_runs']
    wait_times = []

    for run in runs:
        created = datetime.fromisoformat(run['created_at'].replace('Z', '+00:00'))
        # Lấy jobs để biết khi nào thực sự bắt đầu chạy
        jobs_url = f"https://api.github.com/repos/{OWNER}/{REPO}/actions/runs/{run['id']}/jobs"
        jobs_resp = requests.get(jobs_url, headers={'Authorization': f'token {TOKEN}'})
        jobs = jobs_resp.json().get('jobs', [])

        for job in jobs:
            if job.get('started_at') and job.get('runner_name'):
                started = datetime.fromisoformat(job['started_at'].replace('Z', '+00:00'))
                wait = (started - created).total_seconds()
                wait_times.append({'job': job['name'], 'wait_seconds': wait, 'runner': job['runner_name']})

    if wait_times:
        avg_wait = sum(w['wait_seconds'] for w in wait_times) / len(wait_times)
        max_wait = max(w['wait_seconds'] for w in wait_times)
        print(f"Thời gian chờ trung bình: {avg_wait:.1f}s")
        print(f"Thời gian chờ tối đa: {max_wait:.1f}s")

        if avg_wait > 60:
            print("⚠️  Thời gian chờ > 60 giây — cân nhắc thêm runners")

get_workflow_timing()
```

### Khi Nào Cần Thêm Runners?

| Chỉ Số | Ngưỡng Cảnh Báo | Hành Động |
|---|---|---|
| Queue wait time trung bình | > 2 phút | Thêm 20% runners |
| Runner utilization | > 80% trong giờ cao điểm | Thêm 30% runners |
| Số job bị timeout | > 5% tổng jobs | Kiểm tra resource limits |
| Offline rate | > 10% fleet | Điều tra stability |

---

## 📋 Log Tập Trung Cho Runner Fleet

### Cấu Hình Rsyslog Forward Đến ELK

```bash
# /etc/rsyslog.d/50-github-runner.conf

# Tag logs của runner service
:programname, isequal, "actions.runner" {
  # Thêm metadata
  action(type="mmjsonparse")

  # Forward đến Logstash
  action(
    type="omfwd"
    Target="logstash.internal.corp"
    Port="5514"
    Protocol="tcp"
    Template="RSYSLOG_SyslogProtocol23Format"
  )
}
```

### Logstash Pipeline

```ruby
# /etc/logstash/conf.d/github-runner.conf
input {
  syslog {
    port => 5514
    type => "github-runner"
  }
}

filter {
  if [type] == "github-runner" {
    # Parse runner log format
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:runner_message}"
      }
    }

    # Thêm hostname cho biết runner nào
    mutate {
      add_field => {
        "runner_host" => "%{host}"
        "environment" => "production"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "github-runners-%{+YYYY.MM.dd}"
  }
}
```

### CloudWatch Logs (AWS)

```bash
# Cài CloudWatch Agent
sudo yum install -y amazon-cloudwatch-agent

# Cấu hình
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/opt/actions-runner/_diag/Runner_*.log",
            "log_group_name": "/github-actions/runner-diag",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    },
    "log_stream_name": "default"
  }
}
EOF

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

---

## 🤖 Tự Động Hóa Bảo Trì

### Ansible Playbook Cho Maintenance Định Kỳ

```yaml
# playbooks/runner-maintenance.yml
---
- name: GitHub Runner Maintenance
  hosts: github_runners
  become: true

  tasks:
    - name: Kiểm tra disk space trước bảo trì
      command: df -h /opt/actions-runner
      register: disk_before

    - name: Dừng runner service tạm thời
      systemd:
        name: "actions.runner.{{ github_owner }}.{{ github_repo }}.{{ ansible_hostname }}.service"
        state: stopped
      when: not runner_has_active_job    # Kiểm tra trước khi dừng

    - name: Dọn dẹp workspace cũ
      file:
        path: "/opt/actions-runner/_work"
        state: absent
      notify: recreate_work_dir

    - name: Dọn dẹp Docker (nếu có)
      command: docker system prune -af --volumes
      ignore_errors: true

    - name: Cập nhật OS packages
      apt:
        update_cache: yes
        upgrade: safe
      when: ansible_os_family == "Debian"

    - name: Kiểm tra phiên bản runner
      command: cat /opt/actions-runner/package.json
      register: runner_version_info

    - name: Cập nhật runner nếu cần
      script: /opt/scripts/update-runner.sh
      when: runner_needs_update    # Logic kiểm tra version

    - name: Khởi động lại runner service
      systemd:
        name: "actions.runner.{{ github_owner }}.{{ github_repo }}.{{ ansible_hostname }}.service"
        state: started
        enabled: yes

    - name: Chờ runner online
      uri:
        url: "https://api.github.com/repos/{{ github_owner }}/{{ github_repo }}/actions/runners"
        headers:
          Authorization: "token {{ github_pat }}"
      register: runners_response
      until: >
        runners_response.json.runners |
        selectattr('name', 'equalto', ansible_hostname) |
        selectattr('status', 'equalto', 'online') |
        list | length > 0
      retries: 10
      delay: 15

  handlers:
    - name: recreate_work_dir
      file:
        path: "/opt/actions-runner/_work"
        state: directory
        owner: runner
        group: runner
        mode: '0755'
```

### Maintenance Window Workflow

```yaml
# .github/workflows/runner-maintenance.yml
# Bật maintenance mode: tạm dừng nhận jobs mới
name: Runner Maintenance Window

on:
  workflow_dispatch:
    inputs:
      duration_minutes:
        description: 'Thời gian maintenance (phút)'
        default: '30'
      runner_group:
        description: 'Runner group cần maintenance'
        required: true

jobs:
  maintenance:
    runs-on: ubuntu-latest
    steps:
      - name: Notify team via Slack
        uses: slackapi/slack-github-action@v1.26.0
        with:
          channel-id: 'devops-alerts'
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "text": "🔧 Runner maintenance bắt đầu cho group *${{ inputs.runner_group }}*\nDự kiến: ${{ inputs.duration_minutes }} phút"
            }

      - name: Run maintenance via SSM
        run: |
          aws ssm send-command \
            --targets "Key=tag:RunnerGroup,Values=${{ inputs.runner_group }}" \
            --document-name "AWS-RunShellScript" \
            --parameters 'commands=["/opt/scripts/runner-maintenance.sh"]' \
            --timeout-seconds $(( ${{ inputs.duration_minutes }} * 60 + 300 ))

      - name: Notify completion
        uses: slackapi/slack-github-action@v1.26.0
        with:
          channel-id: 'devops-alerts'
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "text": "✅ Runner maintenance hoàn thành cho group *${{ inputs.runner_group }}*"
            }
```

---

## 📖 Runbook Xử Lý Sự Cố Phổ Biến

### Incident 1: Tất Cả Runners Offline Đột Ngột

```
Triệu chứng: Mọi workflows đang chờ job, không có runner nhận

Bước 1: Xác nhận qua GitHub API
  curl -H "Authorization: token $PAT" \
    "https://api.github.com/orgs/{org}/actions/runners" | jq '.runners[] | .status'

Bước 2: SSH vào một runner — kiểm tra systemd
  sudo systemctl status actions.runner.*.service

Bước 3: Xem logs
  sudo journalctl -u actions.runner.*.service -n 50

Nguyên nhân thường gặp và xử lý:
  A. GitHub API outage (sự cố phía GitHub) → Kiểm tra githubstatus.com, chờ
  B. Network issue (sự cố mạng) → Kiểm tra kết nối HTTPS đến github.com
  C. Credentials expired → Re-register runners
  D. OS update reboot → Đảm bảo service enabled (auto-start sau boot)

Bước 4: Khởi động lại tất cả runners
  ansible github_runners -m systemd -a "name=actions.runner.*.service state=restarted" -b

Bước 5: Xác nhận runners online
  Kiểm tra GitHub Settings → Actions → Runners
```

### Incident 2: Job Bị Kẹt Trong Queue Hàng Giờ

```
Triệu chứng: Jobs có trạng thái "queued" mãi không chạy

Bước 1: Kiểm tra labels trong workflow có match runner không
  grep "runs-on" .github/workflows/*.yml

Bước 2: Kiểm tra runners có labels tương ứng
  curl -H "Authorization: token $PAT" \
    "https://api.github.com/repos/{owner}/{repo}/actions/runners" \
    | jq '.runners[] | {name, labels: .labels[].name}'

Bước 3: Kiểm tra runner group permissions
  Settings → Actions → Runner groups → Verify repo access

Bước 4: Kiểm tra workflow restrictions
  Nếu group có restricted_to_workflows → verify tên workflow file khớp

Xử lý:
  - Nếu label không match: Cập nhật workflow hoặc thêm label cho runner
  - Nếu group không cho phép repo: Thêm repo vào runner group
  - Nếu workflow bị restrict: Thêm workflow vào danh sách allowed
```

### Incident 3: Build Chậm Bất Thường

```
Triệu chứng: Jobs chạy lâu hơn bình thường 2-3 lần

Bước 1: Kiểm tra resource usage trên runner
  top -b -n 1 | head -20
  iostat -x 1 5
  free -h

Bước 2: Kiểm tra network throughput
  iperf3 -c github.com -p 443 2>/dev/null || curl -o /dev/null -s \
    --write-out "%{speed_download}" https://github.com

Bước 3: Kiểm tra nếu có nhiều jobs đang chạy song song trên cùng runner
  ps aux | grep Runner.Worker | wc -l

Nguyên nhân và xử lý:
  A. CPU/Memory cao → Giảm concurrency hoặc nâng specs runner
  B. Disk I/O cao → Chuyển sang SSD, dọn dẹp workspace
  C. Network chậm → Kiểm tra bandwidth, cân nhắc runner gần region hơn
  D. Cache miss → Kiểm tra cache key strategy trong workflow
```

---

## 🧪 Bài Tập Thực Hành

### Bài 1: Thiết Lập Monitoring Cơ Bản

```bash
# Tạo script kiểm tra sức khỏe runner mỗi 5 phút
cat > /opt/scripts/runner-health-check.sh << 'SCRIPT'
#!/bin/bash
RUNNER_SERVICE="actions.runner.*.service"

if ! systemctl is-active --quiet ${RUNNER_SERVICE}; then
  echo "ALERT: Runner service không active!"
  # Gửi alert qua curl đến Slack webhook, PagerDuty, v.v.
  curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-type: application/json' \
    --data "{\"text\":\"🚨 GitHub runner trên $(hostname) offline!\"}"
fi
SCRIPT

chmod +x /opt/scripts/runner-health-check.sh

# Thêm vào crontab
(crontab -l ; echo "*/5 * * * * /opt/scripts/runner-health-check.sh") | crontab -
```

### Bài 2: Viết Runbook Cho Team

```markdown
## Runbook: GitHub Runner Offline

**Người phụ trách:** DevOps Team
**Thời gian giải quyết mục tiêu:** < 15 phút

### Dấu Hiệu Nhận Biết
- Alert từ Grafana/PagerDuty: "GitHubRunnerOffline"
- Developers báo cáo workflows stuck ở "queued"

### Các Bước Xử Lý
1. [ ] Kiểm tra GitHub Status (githubstatus.com)
2. [ ] SSH vào runner: `ssh runner-01.internal`
3. [ ] Kiểm tra service: `sudo systemctl status actions.runner.*`
4. [ ] Xem logs: `sudo journalctl -u actions.runner.* -n 50`
5. [ ] Khởi động lại: `sudo systemctl restart actions.runner.*`
6. [ ] Xác nhận online trong GitHub UI
7. [ ] Ghi lại nguyên nhân vào incident log

### Escalation (Leo Thang)
Nếu không giải quyết được trong 15 phút → @mention #devops-oncall
```

---

## 📋 Maintenance Calendar (Lịch Bảo Trì)

| Tần Suất | Công Việc | Thời Gian Thực Hiện |
|---|---|---|
| **Hàng ngày** | Kiểm tra runners online | Tự động (script) |
| **Hàng ngày** | Dọn dẹp workspace cũ hơn 3 ngày | Tự động (cron) |
| **Hàng tuần** | Kiểm tra phiên bản runner binary | Thứ Hai |
| **Hàng tuần** | Review disk usage | Thứ Hai |
| **Hàng tuần** | OS security patches | Thứ Tư |
| **Hàng tháng** | Review runner groups và permissions | Tuần 1 mỗi tháng |
| **Hàng tháng** | Capacity review — thêm/bớt runners | Tuần 1 mỗi tháng |
| **Hàng quý** | Full security audit | Đầu mỗi quý |
| **Khi cần** | Runner update khi GitHub thông báo | Trong 30 ngày kể từ release |

---

**Tiếp Theo:** Quay về [README.md](./README.md) để xem tổng quan, hoặc đến [08-security](../08-security/) để tìm hiểu bảo mật CI/CD toàn diện.
