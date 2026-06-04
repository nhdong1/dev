# 3. Metrics & Observability — Chỉ Số và Quan Sát Workflow

> Đo lường và theo dõi hiệu suất (performance), tỉ lệ thành công (success rate), và xu hướng của GitHub Actions workflows — từ công cụ built-in đến tích hợp Datadog, Grafana, và Prometheus.

---

## 📋 Mục Lục

1. [Metrics Cơ Bản Trong GitHub](#1-metrics-cơ-bản-trong-github)
2. [GitHub API Cho Metrics](#2-github-api-cho-metrics)
3. [Tích Hợp Datadog](#3-tích-hợp-datadog)
4. [Grafana và Prometheus](#4-grafana-và-prometheus)
5. [Custom Metrics — Chỉ Số Tùy Chỉnh](#5-custom-metrics--chỉ-số-tùy-chỉnh)
6. [Job Summary — Báo Cáo Tóm Tắt](#6-job-summary--báo-cáo-tóm-tắt)
7. [SLI / SLO Cho CI/CD Pipeline](#7-sli--slo-cho-cicd-pipeline)
8. [Best Practices](#8-best-practices)

---

## 1. Metrics Cơ Bản Trong GitHub

### Insights Tab

Mỗi repository có tab **Actions** với thông tin:

```
Actions → (chọn workflow) → hiển thị:
  - Tổng số runs trong 30 ngày
  - Tỉ lệ thành công / thất bại
  - Thời gian chạy trung bình
  - Danh sách runs gần đây
```

### GitHub Billing Usage

```
Organization Settings → Billing → Actions:
  - Phút sử dụng theo tháng (per-OS breakdown)
  - Storage cho artifacts và cache
  - So sánh với giới hạn plan
```

### GitHub REST API — Lấy Metrics Cơ Bản

```bash
# Lấy danh sách workflow runs
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/actions/runs?per_page=100&branch=main"

# Lấy thông tin một run cụ thể (bao gồm thời gian chạy)
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/actions/runs/RUN_ID"

# Lấy billing usage
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/actions/billing"
```

---

## 2. GitHub API Cho Metrics

### Script Thu Thập Metrics

```python
#!/usr/bin/env python3
"""Thu thập metrics từ GitHub Actions API."""

import requests
import json
from datetime import datetime, timedelta
from statistics import mean, median

GITHUB_TOKEN = "ghp_..."  # Personal Access Token
OWNER = "myorg"
REPO = "myrepo"
WORKFLOW_ID = "ci.yml"

headers = {
    "Authorization": f"Bearer {GITHUB_TOKEN}",
    "Accept": "application/vnd.github.v3+json"
}

def get_workflow_runs(days=30):
    """Lấy tất cả runs trong N ngày gần đây."""
    since = (datetime.utcnow() - timedelta(days=days)).isoformat() + "Z"
    runs = []
    page = 1
    
    while True:
        url = f"https://api.github.com/repos/{OWNER}/{REPO}/actions/workflows/{WORKFLOW_ID}/runs"
        params = {"per_page": 100, "page": page, "created": f">{since}"}
        resp = requests.get(url, headers=headers, params=params)
        data = resp.json()
        
        if not data.get("workflow_runs"):
            break
        runs.extend(data["workflow_runs"])
        
        if len(data["workflow_runs"]) < 100:
            break
        page += 1
    
    return runs

def calculate_metrics(runs):
    """Tính toán metrics từ danh sách runs."""
    if not runs:
        return {}
    
    total = len(runs)
    successful = sum(1 for r in runs if r["conclusion"] == "success")
    failed = sum(1 for r in runs if r["conclusion"] == "failure")
    
    # Thời gian chạy tính bằng giây
    durations = []
    for run in runs:
        if run["status"] == "completed" and run["created_at"] and run["updated_at"]:
            start = datetime.fromisoformat(run["created_at"].replace("Z", "+00:00"))
            end = datetime.fromisoformat(run["updated_at"].replace("Z", "+00:00"))
            durations.append((end - start).total_seconds())
    
    return {
        "total_runs": total,
        "success_rate_pct": round(successful / total * 100, 1) if total > 0 else 0,
        "failure_rate_pct": round(failed / total * 100, 1) if total > 0 else 0,
        "avg_duration_sec": round(mean(durations), 1) if durations else 0,
        "median_duration_sec": round(median(durations), 1) if durations else 0,
        "p95_duration_sec": round(sorted(durations)[int(len(durations) * 0.95)], 1) if durations else 0,
    }

runs = get_workflow_runs(days=30)
metrics = calculate_metrics(runs)
print(json.dumps(metrics, indent=2))
```

### Kết Quả Mẫu

```json
{
  "total_runs": 248,
  "success_rate_pct": 94.8,
  "failure_rate_pct": 5.2,
  "avg_duration_sec": 187.4,
  "median_duration_sec": 162.0,
  "p95_duration_sec": 341.0
}
```

---

## 3. Tích Hợp Datadog

### Gửi Custom Metrics Lên Datadog

```yaml
name: CI Pipeline với Datadog Metrics

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
        run: |
          START=$(date +%s%3N)
          npm test
          END=$(date +%s%3N)
          echo "duration_ms=$((END - START))" >> $GITHUB_OUTPUT

      - name: Send metrics to Datadog
        if: always()
        run: |
          STATUS=0
          if [ "${{ job.status }}" != "success" ]; then STATUS=1; fi
          
          # Gửi metric tỉ lệ lỗi
          curl -X POST "https://api.datadoghq.com/api/v2/series" \
            -H "Content-Type: application/json" \
            -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
            -d "{
              \"series\": [
                {
                  \"metric\": \"github_actions.build.failure\",
                  \"type\": 1,
                  \"points\": [{\"timestamp\": $(date +%s), \"value\": $STATUS}],
                  \"tags\": [
                    \"repo:${{ github.repository }}\",
                    \"branch:${{ github.ref_name }}\",
                    \"workflow:${{ github.workflow }}\"
                  ]
                },
                {
                  \"metric\": \"github_actions.build.duration_ms\",
                  \"type\": 3,
                  \"points\": [{\"timestamp\": $(date +%s), \"value\": ${{ steps.tests.outputs.duration_ms }}}],
                  \"tags\": [
                    \"repo:${{ github.repository }}\",
                    \"branch:${{ github.ref_name }}\"
                  ]
                }
              ]
            }"
```

### Datadog Dashboard Setup

```json
// Ví dụ widget cho Datadog dashboard
{
  "title": "GitHub Actions — Build Success Rate",
  "type": "timeseries",
  "requests": [{
    "q": "100 - avg:github_actions.build.failure{*} * 100",
    "display_type": "line"
  }]
}
```

---

## 4. Grafana và Prometheus

### Push Metrics Lên Pushgateway

```yaml
- name: Push metrics to Prometheus Pushgateway
  if: always()
  run: |
    STATUS=1
    if [ "${{ job.status }}" == "success" ]; then STATUS=0; fi
    
    DURATION=${{ steps.build.outputs.duration_sec || 0 }}
    REPO=$(echo "${{ github.repository }}" | tr '/' '_')
    
    # Pushgateway — Cổng Đẩy Metrics Cho Prometheus
    cat <<EOF | curl --data-binary @- "${{ secrets.PUSHGATEWAY_URL }}/metrics/job/github_actions/instance/${REPO}"
    # TYPE github_actions_build_failure gauge
    github_actions_build_failure{repo="${REPO}",branch="${{ github.ref_name }}",workflow="${{ github.workflow }}"} ${STATUS}
    # TYPE github_actions_build_duration_seconds gauge
    github_actions_build_duration_seconds{repo="${REPO}",branch="${{ github.ref_name }}"} ${DURATION}
    EOF
```

### Grafana Dashboard JSON (Mẫu)

```json
{
  "panels": [
    {
      "title": "Build Success Rate (%)",
      "type": "stat",
      "targets": [{
        "expr": "100 - (sum(github_actions_build_failure) / count(github_actions_build_failure)) * 100"
      }]
    },
    {
      "title": "Build Duration (p95)",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(github_actions_build_duration_seconds_bucket[1h]))"
      }]
    }
  ]
}
```

---

## 5. Custom Metrics — Chỉ Số Tùy Chỉnh

### Đo Thời Gian Từng Phase

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Time — start
        id: start
        run: echo "ts=$(date +%s)" >> $GITHUB_OUTPUT

      - name: Install dependencies
        id: install
        run: |
          START=$(date +%s)
          npm ci
          END=$(date +%s)
          echo "duration=$((END - START))" >> $GITHUB_OUTPUT

      - name: Run tests
        id: test
        run: |
          START=$(date +%s)
          npm test -- --coverage
          END=$(date +%s)
          echo "duration=$((END - START))" >> $GITHUB_OUTPUT
          # Lấy coverage từ output
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          echo "coverage=$COVERAGE" >> $GITHUB_OUTPUT

      - name: Build
        id: build
        run: |
          START=$(date +%s)
          npm run build
          END=$(date +%s)
          echo "duration=$((END - START))" >> $GITHUB_OUTPUT

      - name: Report metrics
        if: always()
        run: |
          TOTAL=$(($(date +%s) - ${{ steps.start.outputs.ts }}))
          echo "## Build Metrics" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Phase | Duration |" >> $GITHUB_STEP_SUMMARY
          echo "|---|---|" >> $GITHUB_STEP_SUMMARY
          echo "| Install | ${{ steps.install.outputs.duration }}s |" >> $GITHUB_STEP_SUMMARY
          echo "| Test | ${{ steps.test.outputs.duration }}s |" >> $GITHUB_STEP_SUMMARY
          echo "| Build | ${{ steps.build.outputs.duration }}s |" >> $GITHUB_STEP_SUMMARY
          echo "| **Total** | **${TOTAL}s** |" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Test Coverage:** ${{ steps.test.outputs.coverage }}%" >> $GITHUB_STEP_SUMMARY
```

### Theo Dõi Kích Thước Bundle

```yaml
- name: Check bundle size
  id: bundle
  run: |
    npm run build
    SIZE=$(du -sb dist/ | cut -f1)
    SIZE_KB=$((SIZE / 1024))
    echo "size_kb=$SIZE_KB" >> $GITHUB_OUTPUT
    
    # So sánh với ngưỡng
    MAX_SIZE_KB=500
    if [ $SIZE_KB -gt $MAX_SIZE_KB ]; then
      echo "::warning::Bundle size ${SIZE_KB}KB exceeds limit ${MAX_SIZE_KB}KB"
    fi
    
    echo "Bundle size: ${SIZE_KB}KB" >> $GITHUB_STEP_SUMMARY
```

---

## 6. Job Summary — Báo Cáo Tóm Tắt

`GITHUB_STEP_SUMMARY` — file đặc biệt để viết Markdown hiển thị trên trang workflow run.

### Summary Toàn Diện

```yaml
- name: Generate full report
  if: always()
  run: |
    cat >> $GITHUB_STEP_SUMMARY << 'EOF'
    # 📊 CI Pipeline Report

    ## Test Results

    | Suite | Tests | Passed | Failed | Coverage |
    |---|---|---|---|---|
    | Unit | 248 | 245 | 3 | 87.4% |
    | Integration | 42 | 42 | 0 | — |

    ## Build Info

    - **Version:** `${{ steps.version.outputs.tag }}`
    - **Duration:** `${{ steps.timer.outputs.total }}s`
    - **Image Size:** `${{ steps.docker.outputs.size }}`

    ## Quality Gates (Cổng Chất Lượng)

    | Check | Status |
    |---|---|
    | Unit Tests | ✅ Passed |
    | Coverage ≥ 80% | ✅ 87.4% |
    | Lint | ✅ No issues |
    | Security Scan | ⚠️ 1 medium issue |

    EOF
```

---

## 7. SLI / SLO Cho CI/CD Pipeline

### Định Nghĩa

- **SLI** — Service Level Indicator (Chỉ Số Mức Dịch Vụ) — số liệu đo lường thực tế
- **SLO** — Service Level Objective (Mục Tiêu Mức Dịch Vụ) — ngưỡng cam kết
- **Error Budget** — Ngân Sách Lỗi — lượng lỗi được phép trong kỳ SLO

### SLO Tiêu Chuẩn Cho CI/CD

| SLI | SLO Khuyến Nghị | Cách Đo |
|---|---|---|
| Build success rate | ≥ 95% trong 7 ngày | Số runs thành công / tổng runs |
| P95 build duration | ≤ 10 phút | Percentile 95 thời gian chạy |
| Deployment frequency | ≥ 1 deploy/ngày | Số deploys thành công |
| MTTR (Mean Time to Recovery — Thời Gian Trung Bình Phục Hồi) | ≤ 30 phút | Thời gian từ fail đến fix |

### Workflow Kiểm Tra SLO Hàng Ngày

```yaml
name: SLO Check

on:
  schedule:
    - cron: '0 8 * * *'   # 8:00 AM mỗi ngày

jobs:
  check-slo:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch last 7 days metrics
        id: metrics
        run: |
          # Gọi GitHub API để lấy runs 7 ngày qua
          RUNS=$(curl -s -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            "https://api.github.com/repos/${{ github.repository }}/actions/workflows/ci.yml/runs?per_page=100&created=>$(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%SZ)")
          
          TOTAL=$(echo $RUNS | jq '.total_count')
          SUCCESS=$(echo $RUNS | jq '[.workflow_runs[] | select(.conclusion == "success")] | length')
          RATE=$(echo "scale=1; $SUCCESS * 100 / $TOTAL" | bc)
          
          echo "success_rate=$RATE" >> $GITHUB_OUTPUT
          echo "total=$TOTAL" >> $GITHUB_OUTPUT

      - name: Alert if SLO breach
        if: ${{ steps.metrics.outputs.success_rate < 95 }}
        run: |
          echo "::error::SLO BREACH: Build success rate ${{ steps.metrics.outputs.success_rate }}% < 95% SLO"
          # Gửi Slack alert...
```

---

## 8. Best Practices

### Dashboard Tối Thiểu Cần Có

```
✅ 4 Golden Signals cho CI/CD Pipeline:

1. Latency   — Thời gian trung bình và P95 của build
2. Traffic   — Số workflows chạy mỗi ngày/tuần
3. Errors    — Tỉ lệ fail và top lỗi thường gặp
4. Saturation — Queue time của runners (đặc biệt self-hosted)
```

### Chiến Lược Thu Thập Metrics

```
Không nên:
  ❌ Thu thập mọi thứ — gây noise và chi phí cao
  ❌ Chỉ monitor khi có sự cố — quá muộn
  ❌ Metrics mà không có alerting

Nên:
  ✅ Tập trung vào metrics business quan trọng (deploy success, build time)
  ✅ Thiết lập baseline trước khi alert
  ✅ Review metrics hàng tuần để tìm xu hướng
  ✅ Liên kết metrics với alerts và runbooks (tài liệu xử lý sự cố)
```

### Alerting Thresholds (Ngưỡng Cảnh Báo) Gợi Ý

| Metric | Warning | Critical |
|---|---|---|
| Build success rate (7d) | < 97% | < 95% |
| P95 build duration | > 15 phút | > 20 phút |
| Queue time (self-hosted) | > 5 phút | > 15 phút |
| Failed deploys (24h) | ≥ 2 | ≥ 3 |

---

## 🔗 Liên Kết Liên Quan

- [1-debug-logging.md](./1-debug-logging.md) — Debug khi metrics xấu
- [2-workflow-notifications.md](./2-workflow-notifications.md) — Alerting khi SLO breach
- [4-audit-logs.md](./4-audit-logs.md) — Metrics về thao tác người dùng

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
