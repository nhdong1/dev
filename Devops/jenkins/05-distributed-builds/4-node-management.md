# Node Management — Quản Lý Node và Executor

> **Node Management** (quản lý node) bao gồm toàn bộ các công việc vận hành liên quan đến Agent Node trong Jenkins: thêm/xóa node, theo dõi trạng thái, cấu hình Executor (bộ thực thi), quản lý vòng đời (lifecycle) của cloud node, và tự động hóa các tác vụ quản trị.

---

## Mục Lục

1. [Tổng Quan Quản Lý Node](#tổng-quan-quản-lý-node)
2. [Giám Sát Trạng Thái Node](#giám-sát-trạng-thái-node)
3. [Cấu Hình Executor và Availability](#cấu-hình-executor-và-availability)
4. [Node Lifecycle — Vòng Đời Node](#node-lifecycle--vòng-đời-node)
5. [Cloud Node Lifecycle — Vòng Đời Node Động](#cloud-node-lifecycle--vòng-đời-node-động)
6. [Groovy Script Quản Lý Node](#groovy-script-quản-lý-node)
7. [Monitoring Node Health](#monitoring-node-health)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Quản Lý Node

### Kiến Trúc Tổng Thể

```
Jenkins Controller
        │
        ├─── Built-In Node (Controller)
        │         └── Executors: 0   ← Tắt trong production
        │
        ├─── Static Nodes (SSH/JNLP)
        │         ├── agent-linux-01  (Labels: linux java, Executors: 4)
        │         ├── agent-linux-02  (Labels: linux docker, Executors: 2)
        │         └── agent-win-01    (Labels: windows dotnet, Executors: 2)
        │
        └─── Cloud Nodes (Dynamic)
                  ├── Kubernetes Cloud
                  │      ├── Pod: maven-agent-abc123 (tạm thời)
                  │      └── Pod: node-agent-xyz789  (tạm thời)
                  └── Docker Cloud
                         └── Container: build-1234    (tạm thời)
```

### Trang Quản Lý Node

Truy cập: `<jenkins-url>/computer/` hoặc `Manage Jenkins → Nodes`

```
Trang /computer/ hiển thị:
┌─────────────────┬──────────┬─────────────┬─────────────┐
│ Node Name        │ Status   │ Executors   │ Labels      │
├─────────────────┼──────────┼─────────────┼─────────────┤
│ Built-In Node   │ ✅ Online │ 0 / 0       │ built-in    │
│ agent-linux-01  │ ✅ Online │ 2 / 4 busy  │ linux java  │
│ agent-linux-02  │ ❌ Offline│ 0 / 2       │ linux docker│
│ agent-win-01    │ ✅ Online │ 1 / 2 busy  │ windows     │
└─────────────────┴──────────┴─────────────┴─────────────┘
```

---

## Giám Sát Trạng Thái Node

### Các Chỉ Số Quan Trọng

| Chỉ Số | Mô Tả | Ngưỡng Cảnh Báo |
|--------|-------|-----------------|
| **Executor Usage** (sử dụng executor) | Tỷ lệ executor đang bận | > 90% liên tục → cần thêm agent |
| **Queue Length** (độ dài hàng đợi) | Số build đang chờ | > 10 build chờ → bottleneck |
| **Agent Uptime** (thời gian hoạt động) | Thời gian agent online liên tục | < 95% → vấn đề kết nối |
| **Build Duration** (thời gian build) | Thời gian trung bình của build | Tăng đột ngột → agent quá tải |
| **Disk Space** (dung lượng ổ đĩa) | Dung lượng còn trống trên agent | < 10GB → cần dọn dẹp |

### Xem Log Agent

```
Manage Jenkins → Nodes → <tên node> → Log

# Log hiển thị:
# - Thời điểm kết nối/ngắt kết nối
# - Lý do ngắt kết nối
# - Java version, OS version của agent
# - Remoting version (phiên bản giao tiếp)
```

### Xem Build History Trên Node

```
<jenkins-url>/computer/<node-name>/builds

# Hiển thị tất cả build đã chạy trên node này
# Lọc theo: job name, thời gian, trạng thái
```

### Jenkins API — Lấy Thông Tin Node

```bash
# Lấy danh sách tất cả node và trạng thái
curl -s "http://jenkins.example.com/computer/api/json?pretty=true" \
     -u admin:api-token | jq '.computer[] | {name: .displayName, offline: .offline}'

# Lấy thông tin chi tiết một node
curl -s "http://jenkins.example.com/computer/agent-linux-01/api/json?pretty=true" \
     -u admin:api-token

# Kiểm tra executor đang bận
curl -s "http://jenkins.example.com/computer/api/json" \
     -u admin:api-token | \
     jq '.computer[] | select(.busyExecutors > 0) | {name: .displayName, busy: .busyExecutors}'
```

---

## Cấu Hình Executor và Availability

### Số Lượng Executor Tối Ưu

```
# Công thức gần đúng:
Executor tối ưu = số CPU cores × hệ số (1.0 đến 2.0)

Hệ số theo loại workload (khối lượng công việc):
  - CPU-bound (tính toán nặng): hệ số 1.0 → executor = số cores
  - I/O-bound (chờ mạng/disk):  hệ số 1.5 → executor = 1.5 × cores
  - Mixed workload (hỗn hợp):   hệ số 1.2–1.5

Ví dụ thực tế:
  - Agent 4 cores, chủ yếu compile Java:  Executor = 4
  - Agent 8 cores, test và deploy:         Executor = 10–12
  - Agent chạy Docker container builds:   Executor = 2 (container tự scale)
```

### Cấu Hình Availability Mode

```
Manage Jenkins → Nodes → <tên node>
→ Availability:

Option 1: "Keep this agent online as much as possible"
   → Agent luôn online, Jenkins tự kết nối lại khi mất kết nối
   → Dùng cho: agent production quan trọng

Option 2: "Bring this agent online according to a schedule"
   → Agent online trong giờ làm việc, offline ngoài giờ
   → Ví dụ: Online 7:00–22:00, Offline các giờ còn lại
   → Dùng cho: tiết kiệm chi phí cloud VM

Option 3: "Take this agent online when in demand, and offline when idle"
   → Online khi có build chờ, offline sau N phút idle (nhàn rỗi)
   → Tham số: In demand delay (trễ bật), Idle delay (trễ tắt)
   → Dùng cho: agent cloud tính phí theo giờ
```

### Schedule-Based Availability

```
Manage Jenkins → Nodes → <tên node>
→ Availability: ✅ Bring this agent online according to a schedule
→ Online: 07 00 * * 1-5      ← Online lúc 7:00 sáng, thứ 2–6
→ Offline: 22 00 * * 1-5     ← Offline lúc 22:00 tối, thứ 2–6
→ Time zone: Asia/Ho_Chi_Minh
```

---

## Node Lifecycle — Vòng Đời Node

### Thêm Node Mới

```
Manage Jenkins → Nodes → New Node

Bước 1: Nhập thông tin cơ bản
   Node name:             agent-linux-03
   Type:                  ✅ Permanent Agent

Bước 2: Cấu hình chi tiết
   Description:           Linux agent for Java builds
   Number of executors:   4
   Remote root directory: /var/jenkins/agent
   Labels:                linux java maven
   Usage:                 Use this node as much as possible
   Launch method:         Launch agents via SSH

   SSH Host:              10.0.0.103
   Credentials:           jenkins-ssh-key
   Host Key Verification: Manually trusted key verification

Bước 3: Save → Launch agent
```

### Đưa Node Vào Maintenance (Bảo Trì)

```
# Cách 1: Qua UI
Manage Jenkins → Nodes → <tên node>
→ "Mark this node temporarily offline"
→ Ghi rõ lý do: "Scheduled maintenance: kernel upgrade - 2026-05-11"

# Build đang chạy: tiếp tục đến khi xong
# Build mới: không được phân bổ vào node này
```

```bash
# Cách 2: Qua Jenkins CLI
java -jar jenkins-cli.jar -s http://jenkins.example.com \
     -auth admin:api-token \
     offline-node agent-linux-01 \
     -m "Maintenance: OS upgrade"

# Bật lại
java -jar jenkins-cli.jar -s http://jenkins.example.com \
     -auth admin:api-token \
     online-node agent-linux-01
```

### Xóa Node

```
# Bước an toàn trước khi xóa:
1. Đưa node offline (mark temporarily offline)
2. Chờ tất cả build đang chạy hoàn thành
3. Kiểm tra không có build nào đang pending trên node
4. Manage Jenkins → Nodes → <tên node> → Delete Agent

# Không xóa node khi đang có build chạy — build sẽ fail
```

### Copy Node (Nhân Bản Cấu Hình)

```
Manage Jenkins → Nodes → New Node
→ Node name: agent-linux-04
→ ✅ Copy Existing Node
→ Copy from: agent-linux-03    ← Nhân bản toàn bộ config
→ Chỉ thay đổi: SSH Host, Description
→ Save
```

---

## Cloud Node Lifecycle — Vòng Đời Node Động

### Docker Cloud Lifecycle

```
Build request vào queue
        │
        ▼
Jenkins Docker Cloud nhận lệnh tạo container
        │
        ▼
docker run <image> <agent-jar> ...   ← Container được tạo
        │
        ▼
Agent kết nối vào Controller qua JNLP
        │
        ▼
Build được phân bổ vào agent
        │
        ▼
Build chạy xong
        │
        ▼
Container bị xóa (docker rm)         ← Tự động dọn dẹp
```

### Kubernetes Pod Lifecycle

```
Build request
        │
        ▼
Kubernetes Plugin gọi K8s API: POST /api/v1/namespaces/jenkins-agents/pods
        │
        ▼
K8s Scheduler (bộ lên lịch K8s) chọn Node phù hợp
        │
        ▼
Kubelet (agent K8s trên Node) pull image, tạo container
        │
        ▼
jnlp container kết nối vào Jenkins Controller
        │
        ▼
Build thực thi trong container chỉ định
        │
        ▼
Build xong → Jenkins gọi K8s API: DELETE /api/v1/.../pods/<pod-name>
        │
        ▼
Pod terminated, tài nguyên được giải phóng
```

### Cấu Hình Timeout Cho Cloud Node

```yaml
# Trong JCasC hoặc Kubernetes Cloud config
kubernetes:
  connectTimeout: 5      # Giây chờ kết nối
  readTimeout:    15     # Giây chờ đọc
  retentionTimeout: 5    # Phút giữ Pod sau khi idle trước khi xóa
  containerCapStr: "100" # Số Pod tối đa đồng thời
```

---

## Groovy Script Quản Lý Node

Jenkins hỗ trợ Groovy Script Console để tự động hóa các tác vụ quản trị node.

> Truy cập: `Manage Jenkins → Script Console`

### Liệt Kê Tất Cả Node Và Trạng Thái

```groovy
// In ra trạng thái tất cả node
Jenkins.instance.nodes.each { node ->
    def computer = node.toComputer()
    println "Node: ${node.name}"
    println "  Labels:     ${node.labelString}"
    println "  Executors:  ${computer?.countBusy()} / ${node.numExecutors}"
    println "  Online:     ${!computer?.isOffline()}"
    println "  Offline?:   ${computer?.isOffline()}"
    println "  Cause:      ${computer?.getOfflineCause()}"
    println ""
}
```

### Đưa Tất Cả Node Offline Trước Maintenance

```groovy
// Đưa TẤT CẢ agent offline (không bao gồm Controller)
// Dùng khi bảo trì toàn hệ thống

import jenkins.model.Jenkins
import hudson.slaves.OfflineCause

def reason = "Emergency maintenance - 2026-05-11 22:00"

Jenkins.instance.nodes.each { node ->
    def computer = node.toComputer()
    if (computer != null && !computer.isOffline()) {
        computer.setTemporarilyOffline(true, new OfflineCause.ByCLI(reason))
        println "Đã offline: ${node.name}"
    }
}

println "Tất cả agent đã được đưa offline."
```

### Bật Lại Tất Cả Node Sau Maintenance

```groovy
Jenkins.instance.nodes.each { node ->
    def computer = node.toComputer()
    if (computer != null && computer.isTemporarilyOffline()) {
        computer.setTemporarilyOffline(false, null)
        println "Đã online: ${node.name}"
    }
}

println "Tất cả agent đã được bật lại."
```

### Xóa Build Queue (Hàng Đợi) Khi Khẩn Cấp

```groovy
// Xóa TẤT CẢ build đang chờ trong queue
// Dùng khi cần dừng khẩn cấp

import jenkins.model.Jenkins

def queue = Jenkins.instance.queue
queue.items.each { item ->
    println "Hủy: ${item.task.name}"
    queue.cancel(item.task)
}

println "Đã xóa ${queue.items.size()} build khỏi queue."
```

### Tìm Build Đang Chạy Trên Một Node

```groovy
// Xem build nào đang chạy trên agent cụ thể
def targetNode = "agent-linux-01"

Jenkins.instance.nodes.find { it.name == targetNode }?.toComputer()?.executors?.each { executor ->
    if (!executor.isIdle()) {
        def build = executor.currentExecutable
        println "Build đang chạy: ${build?.parent?.name} #${build?.number}"
        println "  Bắt đầu:    ${new Date(build?.startTimeInMillis)}"
        println "  Thời lượng: ${(System.currentTimeMillis() - build?.startTimeInMillis) / 1000}s"
    }
}
```

### Dọn Dẹp Workspace Trên Tất Cả Node

```groovy
// Dọn sạch workspace của một job cụ thể trên tất cả node
import hudson.model.Job

def jobName = "my-pipeline-job"
def job = Jenkins.instance.getItemByFullName(jobName)

Jenkins.instance.nodes.each { node ->
    def workspace = node.toComputer()?.getWorkspaceFor(job)
    if (workspace?.exists()) {
        workspace.deleteContents()
        println "Đã dọn workspace trên: ${node.name}"
    }
}

// Dọn cả workspace trên Controller
def builtinWorkspace = Jenkins.instance.toComputer().getWorkspaceFor(job)
builtinWorkspace?.deleteContents()
println "Hoàn thành dọn dẹp workspace."
```

---

## Monitoring Node Health

### Prometheus Metrics Cho Node

Cài **Prometheus Metrics Plugin** để export metrics Jenkins:

```yaml
# Một số metrics quan trọng liên quan đến node:

# Số executor đang bận trên mỗi node
jenkins_executor_count_value{node="agent-linux-01"}
jenkins_executor_in_use_value{node="agent-linux-01"}

# Trạng thái node (0 = offline, 1 = online)
jenkins_node_online_value{node="agent-linux-01"}

# Độ dài queue
jenkins_queue_size_value

# Thời gian build trung bình
jenkins_builds_duration_milliseconds_summary
```

### Grafana Dashboard Cho Node

```json
// Ví dụ panel Grafana — Executor Utilization (Tỷ lệ sử dụng Executor)
{
  "title": "Executor Utilization per Node",
  "type": "bargauge",
  "targets": [
    {
      "expr": "jenkins_executor_in_use_value / jenkins_executor_count_value * 100",
      "legendFormat": "{{node}}"
    }
  ],
  "thresholds": {
    "steps": [
      {"color": "green",  "value": 0},
      {"color": "yellow", "value": 70},
      {"color": "red",    "value": 90}
    ]
  }
}
```

### Alert Rules (Quy Tắc Cảnh Báo)

```yaml
# prometheus-alerts.yaml
groups:
  - name: jenkins-node-alerts
    rules:
      # Cảnh báo khi node offline
      - alert: JenkinsAgentOffline
        expr: jenkins_node_online_value == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Jenkins Agent {{ $labels.node }} offline"
          description: "Agent đã offline hơn 5 phút"

      # Cảnh báo khi queue quá dài
      - alert: JenkinsBuildQueueTooLong
        expr: jenkins_queue_size_value > 20
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Jenkins build queue quá dài: {{ $value }} jobs"
          description: "Cần thêm agent hoặc kiểm tra agent offline"

      # Cảnh báo khi executor sử dụng cao
      - alert: JenkinsHighExecutorUtilization
        expr: >
          (sum(jenkins_executor_in_use_value) / sum(jenkins_executor_count_value)) > 0.9
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Jenkins executor utilization > 90%"
          description: "Cần xem xét thêm agent để tránh bottleneck"
```

---

## Best Practices Tóm Tắt

```
✅ Tắt Executor trên Controller (Executor = 0)
✅ Gán Node Label rõ ràng theo OS, runtime, công cụ
✅ Dùng dedicated user (jenkins) riêng cho agent — không dùng root
✅ Cấu hình auto-reconnect (systemd service cho JNLP)
✅ Giám sát với Prometheus + Grafana
✅ Đặt alert khi node offline hoặc queue quá dài
✅ Dùng JCasC để quản lý cấu hình node dưới dạng code
✅ Dọn workspace định kỳ để tránh đầy ổ đĩa
✅ Ghi rõ lý do khi đưa node vào maintenance
✅ Test kết nối agent trước khi deploy production
```

---

## Câu Hỏi Phỏng Vấn

**Q: Làm thế nào để giám sát sức khỏe của Jenkins Agent trong production?**

> Ba lớp giám sát: **(1) Built-in Jenkins UI** — trang `/computer/` cho trạng thái real-time từng agent; **(2) Prometheus + Grafana** — export metrics qua Prometheus Plugin, xây dashboard theo dõi executor utilization, queue length, node uptime; **(3) Alert** — đặt alert khi node offline > 5 phút hoặc queue > 20 jobs. Ngoài ra, kiểm tra log agent định kỳ để phát hiện lỗi kết nối sớm.

**Q: Khi nào nên tăng số lượng Executor trên một Agent?**

> Khi: **(1)** Build queue thường xuyên có nhiều job chờ trong giờ làm việc; **(2)** Executor utilization > 80–90% liên tục; **(3)** Agent có tài nguyên CPU/RAM còn dư. Không nên tăng khi: workload là CPU-intensive (tính toán nặng) vì quá nhiều build chạy song song sẽ cạnh tranh CPU, làm tất cả chậm hơn. Rule of thumb: executor = CPU cores × 1.0–1.5 tùy workload.

**Q: Làm thế nào để tự động hóa các tác vụ quản trị node?**

> Ba cách: **(1) Groovy Script Console** — chạy script một lần cho tác vụ ad-hoc (offline all nodes, clear queue); **(2) Jenkins Job dùng Groovy** — đặt lịch cron chạy script định kỳ (dọn workspace hàng tuần); **(3) Jenkins CLI** — tích hợp vào shell script hoặc CI pipeline của chính Jenkins cho tác vụ ops. Với Kubernetes Agent, vòng đời được quản lý tự động bởi K8s Plugin — không cần script thủ công.

**Q: Sự khác biệt giữa Static Node và Cloud Node về lifecycle (vòng đời)?**

> Static Node: tồn tại vĩnh viễn, bật/tắt thủ công, tài nguyên luôn được cấp phát dù không có build (lãng phí khi idle). Cloud Node (Docker/K8s): tạo khi có build, xóa ngay sau khi xong — tài nguyên được giải phóng hoàn toàn. Cloud Node phù hợp khi workload không đồng đều theo thời gian (ít build đêm/cuối tuần), giúp tiết kiệm đáng kể chi phí hạ tầng.

---

**Liên Kết Liên Quan:**
- [1-agent-configuration.md](1-agent-configuration.md) — Cấu hình SSH và JNLP Agent
- [2-docker-agents.md](2-docker-agents.md) — Docker Agent và lifecycle
- [3-kubernetes-agents.md](3-kubernetes-agents.md) — Kubernetes Pod Agent
- [README.md](README.md) — Tổng quan Distributed Builds

**Cập Nhật:** 2026-05-11
