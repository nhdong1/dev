# 1 — Monitoring: Giám Sát Jenkins với Prometheus và Grafana

> Không thể vận hành tốt những gì bạn không thể đo lường. Bài này hướng dẫn thiết lập **observability stack** (bộ công cụ quan sát) đầy đủ cho Jenkins: từ thu thập metrics (chỉ số), xây dựng dashboard (bảng điều khiển), đến alerting (cảnh báo) khi có sự cố.

---

## Mục Tiêu

- Cài đặt và cấu hình Prometheus Plugin để xuất Jenkins metrics
- Viết PromQL (Prometheus Query Language — ngôn ngữ truy vấn Prometheus) truy vấn các metrics quan trọng
- Xây dựng Grafana dashboard hiển thị sức khỏe Jenkins theo thời gian thực
- Thiết lập AlertManager (trình quản lý cảnh báo) gửi thông báo qua Slack và email
- Phân biệt các loại metrics và biết khi nào cần cảnh báo

---

## Phần 1: Prometheus Plugin — Xuất Jenkins Metrics

### Cài Đặt

1. **Manage Jenkins → Plugin Manager → Available** → tìm `Prometheus metrics`
2. Cài đặt và khởi động lại Jenkins
3. Sau khi cài, metrics có sẵn tại: `http://<jenkins-host>:8080/prometheus/`

### Kiểm Tra Endpoint

```bash
# Xem raw metrics từ Jenkins
curl -s http://admin:token@jenkins:8080/prometheus/ | head -50

# Kết quả mẫu:
# HELP jenkins_queue_size_value Jenkins queue size
# TYPE jenkins_queue_size_value gauge
jenkins_queue_size_value{} 3.0

# HELP jenkins_builds_duration_milliseconds Jenkins build duration in milliseconds
# TYPE jenkins_builds_duration_milliseconds summary
jenkins_builds_duration_milliseconds{jenkins_job="my-pipeline",quantile="0.5"} 45000.0
```

### Cấu Hình Prometheus Scrape (Thu Thập Metrics)

```yaml
# prometheus.yml — cấu hình Prometheus scrape Jenkins
scrape_configs:
  - job_name: 'jenkins'
    # scrape_interval: tần suất thu thập metrics
    scrape_interval: 15s
    # scrape_timeout: thời gian tối đa cho mỗi lần thu thập
    scrape_timeout: 10s
    
    metrics_path: '/prometheus/'
    
    # Thông tin xác thực nếu Jenkins bật security
    basic_auth:
      username: 'prometheus'
      password: 'your-api-token'
    
    static_configs:
      - targets: ['jenkins-master:8080']
        labels:
          environment: 'production'    # Gán nhãn môi trường
          team: 'platform'             # Gán nhãn team
```

### Kiểm Tra Prometheus Nhận Được Metrics

```bash
# Truy vấn Prometheus API để xác nhận target hoạt động
curl 'http://prometheus:9090/api/v1/targets' | jq '.data.activeTargets[] | select(.labels.job=="jenkins") | {health, lastScrape}'

# Kết quả mong muốn:
# {
#   "health": "up",
#   "lastScrape": "2026-05-11T10:00:00.123Z"
# }
```

---

## Phần 2: Metrics Quan Trọng Cần Theo Dõi

### Nhóm 1: Queue Metrics (Metrics Hàng Đợi Build)

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|--------|---------|-----------------|
| `jenkins_queue_size_value` | Số build đang chờ trong hàng đợi | > 20 build |
| `jenkins_queue_buildable_value` | Build có thể chạy ngay (đủ executor) | > 10 build |
| `jenkins_queue_stuck_value` | Build bị kẹt không thể phân công agent | > 0 (bất kỳ) |
| `jenkins_queue_waiting_value` | Build đang chờ do quiet period hoặc dependency | Tham khảo |

```promql
# PromQL — tổng số build đang chờ trong 5 phút qua
avg_over_time(jenkins_queue_size_value[5m])

# PromQL — build bị kẹt (không có agent phù hợp)
jenkins_queue_stuck_value > 0
```

### Nhóm 2: Executor Metrics (Metrics Bộ Thực Thi)

| Metric | Ý Nghĩa | Công Thức |
|--------|---------|-----------|
| `jenkins_executor_count_value` | Tổng số executor trên tất cả node | - |
| `jenkins_executor_in_use_value` | Số executor đang chạy build | - |
| `jenkins_executor_free_value` | Số executor còn rảnh | - |

```promql
# PromQL — tỷ lệ sử dụng executor (executor utilization rate)
# Nếu > 90% trong thời gian dài → cần thêm agent
jenkins_executor_in_use_value / jenkins_executor_count_value * 100

# PromQL — executor theo label node (ví dụ: chỉ xem docker-agent)
jenkins_executor_in_use_value{node_label="docker"}
```

### Nhóm 3: Build Metrics (Metrics Build)

| Metric | Ý Nghĩa |
|--------|---------|
| `jenkins_builds_duration_milliseconds` | Thời gian thực thi build (phân vị) |
| `jenkins_builds_success_build_count` | Số build thành công |
| `jenkins_builds_failed_build_count` | Số build thất bại |
| `jenkins_builds_unstable_build_count` | Số build không ổn định (test failures nhưng không lỗi nghiêm trọng) |

```promql
# PromQL — tỷ lệ build thất bại theo job (build failure rate)
rate(jenkins_builds_failed_build_count{jenkins_job="my-pipeline"}[1h])
/
rate(
  (jenkins_builds_success_build_count + jenkins_builds_failed_build_count)
  {jenkins_job="my-pipeline"}[1h]
) * 100

# PromQL — thời gian build trung vị (p50) theo từng pipeline
jenkins_builds_duration_milliseconds{quantile="0.5", jenkins_job="api-service"} / 1000 / 60
# Đơn vị: phút
```

### Nhóm 4: Node/Agent Metrics (Metrics Node và Agent)

| Metric | Ý Nghĩa | Cảnh Báo |
|--------|---------|----------|
| `jenkins_node_count_value` | Tổng số node (bao gồm master) | - |
| `jenkins_node_online_value` | Số node đang online | Giảm bất ngờ |
| `jenkins_node_offline_value` | Số node đang offline | > 0 |

```promql
# PromQL — node nào đang offline
jenkins_node_online_value == 0

# PromQL — tỷ lệ node khả dụng
jenkins_node_online_value / jenkins_node_count_value * 100
```

### Nhóm 5: JVM Metrics (Metrics Java Virtual Machine)

```promql
# PromQL — JVM heap memory usage (lượng bộ nhớ heap đang dùng)
# Nếu > 85% liên tục → nguy cơ OutOfMemoryError
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# PromQL — GC pause duration (thời gian dừng do Garbage Collection)
# Nếu G1GC pause > 500ms thường xuyên → cần tuning JVM
rate(jvm_gc_pause_seconds_sum[5m]) / rate(jvm_gc_pause_seconds_count[5m])
```

---

## Phần 3: Grafana Dashboard — Xây Dựng Bảng Điều Khiển

### Cấu Hình Datasource (Nguồn Dữ Liệu)

```json
// Grafana datasource configuration — Prometheus
{
  "name": "Prometheus-Jenkins",
  "type": "prometheus",
  "url": "http://prometheus:9090",
  "access": "proxy",
  "isDefault": true
}
```

### Dashboard JSON — Panel Quan Trọng

Dưới đây là các panel (bảng hiển thị) cần thiết trong Jenkins dashboard:

#### Panel 1: Build Queue Length (Độ Dài Hàng Đợi)

```json
{
  "title": "Build Queue Length (Độ Dài Hàng Đợi)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "jenkins_queue_size_value",
      "legendFormat": "Tổng hàng đợi"
    },
    {
      "expr": "jenkins_queue_stuck_value",
      "legendFormat": "Bị kẹt (stuck)"
    }
  ],
  "thresholds": {
    "steps": [
      { "color": "green", "value": null },
      { "color": "yellow", "value": 10 },
      { "color": "red", "value": 20 }
    ]
  }
}
```

#### Panel 2: Executor Utilization Rate (Tỷ Lệ Sử Dụng Executor)

```json
{
  "title": "Executor Utilization % (Tỷ Lệ Sử Dụng)",
  "type": "gauge",
  "targets": [
    {
      "expr": "jenkins_executor_in_use_value / jenkins_executor_count_value * 100",
      "legendFormat": "Đang dùng %"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "min": 0,
      "max": 100,
      "unit": "percent",
      "thresholds": {
        "steps": [
          { "color": "green",  "value": null },
          { "color": "yellow", "value": 70 },
          { "color": "red",    "value": 90 }
        ]
      }
    }
  }
}
```

#### Panel 3: Build Success Rate (Tỷ Lệ Build Thành Công)

```json
{
  "title": "Build Success Rate % (Tỷ Lệ Thành Công)",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(jenkins_builds_success_build_count) / (sum(jenkins_builds_success_build_count) + sum(jenkins_builds_failed_build_count)) * 100",
      "legendFormat": "Success Rate"
    }
  ]
}
```

### Import Dashboard Có Sẵn

Cộng đồng đã có dashboard Jenkins sẵn trên Grafana.com. Có thể import bằng cách:

1. Grafana → **Dashboards → Import**
2. Nhập Dashboard ID: `9964` (Jenkins: Performance and Health Overview)
3. Chọn datasource Prometheus vừa cấu hình
4. Tùy chỉnh thêm theo nhu cầu

---

## Phần 4: AlertManager — Cảnh Báo Tự Động

### Cấu Hình Alert Rules (Quy Tắc Cảnh Báo) Trong Prometheus

```yaml
# jenkins-alerts.yml — File quy tắc cảnh báo
groups:
  - name: jenkins_alerts
    rules:

      # Cảnh báo khi hàng đợi build quá dài
      - alert: JenkinsBuildQueueHigh
        expr: jenkins_queue_size_value > 20
        for: 5m            # Kéo dài hơn 5 phút mới cảnh báo
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Hàng đợi Jenkins quá dài"
          description: |
            Có {{ $value }} build đang chờ trong hàng đợi.
            Kiểm tra executor availability (tính khả dụng của executor) hoặc xem có node offline không.

      # Cảnh báo khi build bị kẹt (không có agent phù hợp)
      - alert: JenkinsBuildStuck
        expr: jenkins_queue_stuck_value > 0
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Build Jenkins bị kẹt — không có agent phù hợp"
          description: |
            {{ $value }} build đang bị kẹt và không thể được phân công agent.
            Nguyên nhân thường gặp: tất cả node offline, label node không khớp,
            hoặc node bị suspend (tạm dừng).

      # Cảnh báo khi executor utilization (tỷ lệ sử dụng executor) quá cao
      - alert: JenkinsExecutorUtilizationHigh
        expr: |
          (jenkins_executor_in_use_value / jenkins_executor_count_value * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Executor Jenkins sắp hết"
          description: |
            Tỷ lệ sử dụng executor đạt {{ printf "%.1f" $value }}%.
            Cân nhắc thêm agent node để tránh hàng đợi dài.

      # Cảnh báo khi node offline bất ngờ
      - alert: JenkinsNodeOffline
        expr: jenkins_node_offline_value > 0
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Jenkins agent node đang offline"
          description: "{{ $value }} node đang offline. Kiểm tra kết nối agent."

      # Cảnh báo khi JVM heap usage (dùng bộ nhớ heap) cao
      - alert: JenkinsJVMHeapHigh
        expr: |
          jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100 > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "JVM heap Jenkins sắp đầy"
          description: |
            JVM heap đang dùng {{ printf "%.1f" $value }}%.
            Nguy cơ OutOfMemoryError. Cần tăng -Xmx hoặc tối ưu JVM.

      # Cảnh báo khi tỷ lệ build thất bại cao
      - alert: JenkinsHighFailureRate
        expr: |
          (
            rate(jenkins_builds_failed_build_count[15m])
            /
            rate((jenkins_builds_success_build_count + jenkins_builds_failed_build_count)[15m])
          ) * 100 > 30
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Tỷ lệ build thất bại cao"
          description: |
            Tỷ lệ thất bại đang là {{ printf "%.1f" $value }}% trong 15 phút qua.
```

### Cấu Hình AlertManager Gửi Slack

```yaml
# alertmanager.yml
global:
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'team']
  group_wait: 30s        # Đợi 30s để gom nhóm alert
  group_interval: 5m     # Gửi lại mỗi 5 phút nếu còn active
  repeat_interval: 4h    # Nhắc lại mỗi 4 giờ nếu chưa resolve

  receiver: 'slack-platform-team'

  routes:
    # Alert critical → kênh #incidents
    - match:
        severity: critical
      receiver: 'slack-incidents'
      continue: true

    # Alert warning → kênh #jenkins-alerts
    - match:
        severity: warning
      receiver: 'slack-platform-team'

receivers:
  - name: 'slack-platform-team'
    slack_configs:
      - channel: '#jenkins-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: |
          *Trạng thái:* {{ .Status | toUpper }}
          {{ range .Alerts }}
          *Mô tả:* {{ .Annotations.description }}
          *Thời gian:* {{ .StartsAt.Format "2006-01-02 15:04:05" }}
          {{ end }}
        color: |
          {{ if eq .Status "firing" }}danger{{ else }}good{{ end }}

  - name: 'slack-incidents'
    slack_configs:
      - channel: '#incidents'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        color: 'danger'
```

---

## Phần 5: Monitoring Checklist — Danh Sách Kiểm Tra Giám Sát

### Thiết Lập Ban Đầu (One-time Setup)

```markdown
[ ] Cài đặt Prometheus Plugin trong Jenkins
[ ] Cấu hình Prometheus scrape Jenkins endpoint
[ ] Thêm Jenkins datasource vào Grafana
[ ] Import hoặc tạo Jenkins dashboard trong Grafana
[ ] Thiết lập alert rules trong Prometheus
[ ] Cấu hình AlertManager gửi Slack/email
[ ] Test alert bằng cách tạo tình huống giả lập
[ ] Tài liệu hóa runbook (sổ tay xử lý) cho từng alert
```

### Kiểm Tra Hàng Ngày

```markdown
[ ] Xem tổng quan dashboard — build success rate hôm nay
[ ] Kiểm tra queue length — có kẹt không?
[ ] Kiểm tra node health — tất cả agent online?
[ ] Kiểm tra disk usage của JENKINS_HOME
[ ] Review alert history — có alert bị bỏ sót không?
```

### Phân Tích Hàng Tuần

```markdown
[ ] Build duration trend — build nào đang chậm dần?
[ ] Executor utilization trend — cần thêm agent không?
[ ] Top slow jobs (pipeline chạy chậm nhất)
[ ] Top failing jobs (pipeline thất bại nhiều nhất)
[ ] JVM heap trend — có memory leak không?
```

---

## Phần 6: Tích Hợp Với Monitoring Stack Hiện Có

### Nếu Dùng Datadog

```groovy
// Jenkinsfile — gửi custom metric lên Datadog
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    sh 'mvn package'
                    def duration = System.currentTimeMillis() - startTime

                    // Gửi custom metric (chỉ số tùy chỉnh) lên Datadog
                    sh """
                        curl -X POST "https://api.datadoghq.com/api/v1/series" \\
                          -H "DD-API-KEY: ${DD_API_KEY}" \\
                          -d '{
                            "series": [{
                              "metric": "jenkins.build.duration",
                              "points": [[${System.currentTimeMillis() / 1000}, ${duration}]],
                              "tags": ["job:${JOB_NAME}", "env:production"]
                            }]
                          }'
                    """
                }
            }
        }
    }
}
```

### Nếu Dùng CloudWatch (AWS)

```groovy
// Gửi metrics Jenkins lên AWS CloudWatch
withAWS(credentials: 'aws-prod', region: 'ap-southeast-1') {
    sh """
        aws cloudwatch put-metric-data \\
          --namespace "Jenkins/BuildMetrics" \\
          --metric-data \\
            MetricName=BuildDuration,Value=${buildDurationSeconds},Unit=Seconds \\
            Dimensions=Name=JobName,Value=${env.JOB_NAME}
    """
}
```

---

## Tóm Tắt Các Metrics Quan Trọng

| Metric | Ngưỡng Cảnh Báo | Hành Động Khi Vượt Ngưỡng |
|--------|----------------|--------------------------|
| Queue size | > 20 build | Thêm agent, kiểm tra node offline |
| Stuck build | > 0 | Kiểm tra label, node offline |
| Executor utilization | > 90% trong 10 phút | Scale out agent |
| Node offline | > 0 | Restart agent, kiểm tra kết nối |
| JVM heap | > 85% | Tăng `-Xmx`, phân tích heap dump |
| Build failure rate | > 30% trong 15 phút | Kiểm tra logs, rollback nếu cần |

---

## Câu Hỏi Phỏng Vấn

1. **Bạn theo dõi Jenkins bằng gì? Metrics nào quan trọng nhất?**
   → Prometheus + Grafana. Metrics quan trọng nhất: queue length, executor utilization, node health, JVM heap. Queue length phản ánh ngay lập tức bottleneck của hệ thống.

2. **Khi nào thì cần thêm agent? Dựa vào metric nào để quyết định?**
   → Khi executor utilization > 80% liên tục kết hợp với queue length tăng dần theo thời gian. Không dựa vào cảm giác — dựa vào dữ liệu trend ít nhất 1 tuần.

3. **Alert fatigue (mệt mỏi vì cảnh báo) là gì? Cách phòng tránh?**
   → Alert quá nhiều khiến team bỏ qua mọi cảnh báo. Phòng tránh: chỉ alert những gì cần hành động ngay, đặt ngưỡng hợp lý, dùng `for: 5m` để tránh flapping (cảnh báo bật tắt liên tục), và thường xuyên review alert rules.

4. **Bạn đo lường SLA (Service Level Agreement — thỏa thuận mức dịch vụ) cho Jenkins pipeline như thế nào?**
   → Theo dõi build success rate theo pipeline, build duration percentile (p50, p95), và mean time between failures (thời gian trung bình giữa các lần thất bại). Dashboard hàng tuần review với team.

5. **Nếu Jenkins master đột nhiên chậm lại, bạn điều tra thế nào?**
   → Kiểm tra JVM heap và GC pause metrics trước, sau đó queue size, rồi thread dump nếu cần. Thường là memory pressure hoặc build nào đó chiếm quá nhiều tài nguyên.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
