# Kiến Trúc Jenkins: Master/Agent, Executor, Build Queue, Workspace

## Tổng Quan

Jenkins hoạt động theo mô hình **Controller/Agent** (trước đây gọi là Master/Slave — chủ/nô lệ, đã đổi tên vì lý do ngôn ngữ). Nguyên tắc cốt lõi: **Controller điều phối, Agent thực thi**.

---

## 1. Jenkins Controller (Master) — Bộ Điều Phối Trung Tâm

### Controller là gì?

**Controller** (hay Jenkins Master) là tiến trình Jenkins chính, chịu trách nhiệm:

- Lưu trữ cấu hình job, build history (lịch sử build), plugin, credentials
- Điều phối **Build Queue** — quyết định job nào chạy trên Agent nào
- Phục vụ giao diện web và REST API cho người dùng
- Giao tiếp với các **Agent** để phân phối công việc

### Controller KHÔNG nên làm gì?

> **Quy tắc vàng:** Không chạy build trực tiếp trên Controller.

Lý do:
- Build tiêu tốn CPU và RAM → làm chậm giao diện web và API
- Lỗi trong build script có thể ảnh hưởng đến toàn bộ Jenkins process
- Khó scale (mở rộng) khi số lượng build tăng cao

**Thực hành tốt nhất:** Đặt số lượng Executor trên Controller bằng `0` trong môi trường production, chỉ giữ Agent để thực thi.

### Cấu Trúc Thư Mục JENKINS_HOME

```
$JENKINS_HOME/                   # Thư mục gốc Jenkins (mặc định: /var/jenkins_home)
├── config.xml                   # Cấu hình hệ thống chính
├── plugins/                     # Plugin đã cài đặt
│   ├── git/
│   └── pipeline/
├── jobs/                        # Cấu hình và lịch sử build của mỗi Job
│   ├── my-app/
│   │   ├── config.xml           # Cấu hình Job
│   │   └── builds/
│   │       ├── 1/               # Build #1
│   │       └── 2/               # Build #2
├── workspace/                   # Workspace của build chạy trên Controller (nên tránh)
├── secrets/                     # Credentials được mã hóa
├── users/                       # Tài khoản người dùng Jenkins
├── logs/                        # Log hệ thống Jenkins
└── nodes/                       # Cấu hình Agent được kết nối
```

---

## 2. Agent (Node) — Bộ Thực Thi Build

### Agent là gì?

**Agent** (hay Node) là máy tính hoặc container riêng biệt được kết nối vào Jenkins Controller để chạy build. Mỗi Agent có thể là:

- Máy chủ vật lý (physical server) hoặc VM (Virtual Machine — máy ảo)
- Docker container — được tạo khi cần, xóa sau khi build xong
- Pod trên Kubernetes — dynamic scaling (mở rộng linh hoạt)

### Phương Thức Kết Nối Agent

| Phương Thức | Cơ Chế | Khi Nào Dùng |
|------------|---------|--------------|
| **JNLP** (Java Network Launch Protocol — Giao Thức Khởi Chạy Mạng Java) | Agent mở kết nối ra Controller (outbound) | Agent sau firewall, không nhận inbound connection |
| **SSH** (Secure Shell) | Controller SSH vào Agent (inbound) | Linux/Unix agent có SSH server |
| **Inbound Agent** (Docker) | Tương tự JNLP, dùng trong container | Docker/Kubernetes dynamic agents |
| **Windows Service** | Dịch vụ Windows chạy agent | Agent là máy Windows |

### Node Labels — Nhãn Để Phân Loại Agent

**Node Label** (nhãn node) là chuỗi ký tự gán cho Agent để phân loại khả năng. Pipeline chọn Agent dựa trên label.

```groovy
// Pipeline chỉ chạy trên Agent có label "linux" và "docker"
pipeline {
    agent { label 'linux && docker' }
    stages {
        stage('Build') {
            steps {
                sh 'docker build .'
            }
        }
    }
}
```

Ví dụ các label thường dùng:
- `linux`, `windows`, `macos` — hệ điều hành
- `docker`, `kubectl` — công cụ sẵn có
- `large` — agent có nhiều RAM/CPU hơn
- `prod-deploy` — agent được phép deploy production

---

## 3. Executor — Số Lượng Build Song Song

### Executor là gì?

**Executor** (bộ thực thi) là một "khe" (slot) trên Agent (hoặc Controller) để chạy một build. Số lượng Executor quyết định bao nhiêu build có thể chạy song song trên một node.

```
Agent "build-server-01"
┌─────────────────────────────────┐
│ Executor #1: [Build: my-app #45] │  ← Đang chạy
│ Executor #2: [Build: api #12]    │  ← Đang chạy
│ Executor #3: [IDLE]              │  ← Rảnh, sẵn sàng nhận build tiếp theo
└─────────────────────────────────┘
```

### Cấu Hình Executor

Truy cập: **Manage Jenkins → Nodes → [Tên Node] → Configure → # of executors**

**Quy tắc đặt số lượng Executor:**
- Với CPU-intensive builds (build tốn CPU): `số Executor = số CPU core`
- Với I/O-intensive builds (build tốn ổ đĩa/mạng): có thể đặt `2× số CPU core`
- Controller production: `0` (không chạy build trên Controller)

### Executor trên Controller vs Agent

| | Controller | Agent |
|--|-----------|-------|
| **Mục đích** | Thường đặt = 0, không chạy build | Nơi build thực sự chạy |
| **Ảnh hưởng** | Ảnh hưởng UI nếu chạy build | Cô lập, không ảnh hưởng Controller |
| **Best practice** | Tắt hết Executor | Cấu hình theo tài nguyên máy |

---

## 4. Build Queue — Hàng Đợi Build

### Build Queue là gì?

**Build Queue** (hàng đợi build) là danh sách các build đang chờ được gán cho Executor rảnh. Khi tất cả Executor đều bận, build mới sẽ vào hàng đợi.

```
Build Queue (Hàng Đợi)
┌────────────────────────────────────────────┐
│ #1  my-app/main      (đang chờ 00:02:15)   │ ← Được phân phối tiếp theo
│ #2  api-service #88  (đang chờ 00:00:45)   │
│ #3  frontend #23     (đang chờ 00:00:10)   │
└────────────────────────────────────────────┘
         ↓ Khi có Executor rảnh
┌───────────────────────────────┐
│ Agent linux-01 — Executor #2  │ ← Nhận build #1 từ Queue
└───────────────────────────────┘
```

### Các Yếu Tố Ảnh Hưởng Build Queue

1. **Throttle Concurrent Builds** (giới hạn build song song): Plugin giới hạn số build cùng lúc cho một job hoặc theo category
2. **Node Label**: Build chờ cho đến khi có Agent có đúng label
3. **Quiet Period** (thời gian yên tĩnh): Trì hoãn có chủ ý trước khi build, cho phép gom nhiều commit thành một build
4. **Build Priority** (ưu tiên build): Plugin Priority Sorter cho phép ưu tiên job quan trọng hơn

### Khi Nào Build Queue Trở Nên Vấn Đề?

Dấu hiệu: build chờ > 5 phút thường xuyên → cần thêm Agent hoặc tăng Executor.

---

## 5. Workspace — Không Gian Làm Việc

### Workspace là gì?

**Workspace** (không gian làm việc) là thư mục trên Agent nơi Jenkins checkout (sao chép) source code và thực hiện các bước build. Mỗi job thường có workspace riêng trên mỗi Agent.

```
/var/jenkins/workspace/
├── my-app/              ← Workspace của Job "my-app" trên Agent này
│   ├── src/
│   ├── tests/
│   └── Jenkinsfile
├── api-service/         ← Workspace của Job "api-service"
└── frontend/            ← Workspace của Job "frontend"
```

### Vòng Đời Workspace

```
Trigger Build
     ↓
Checkout Source Code vào Workspace
     ↓
Thực thi các bước build (sh, bat, maven, gradle...)
     ↓
Lưu Artifact (kết quả) nếu có cấu hình
     ↓
Build kết thúc
     ↓
Workspace tồn tại cho build tiếp theo (không tự xóa)
     ↓ (nếu dùng Workspace Cleanup Plugin)
Xóa Workspace sau build
```

### Stash và Unstash — Chuyển File Giữa Các Agent

Khi pipeline chạy trên nhiều Agent khác nhau, workspace không được chia sẻ. Dùng `stash`/`unstash` (lưu tạm / lấy lại) để chuyển file:

```groovy
pipeline {
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps {
                sh 'mvn package -DskipTests'
                stash name: 'app-jar', includes: 'target/*.jar'
            }
        }
        stage('Deploy') {
            agent { label 'prod-deploy' }
            steps {
                unstash 'app-jar'
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
```

### Quản Lý Workspace

- **Workspace Cleanup Plugin**: Tự động xóa workspace trước hoặc sau build
- **Custom Workspace**: Chỉ định đường dẫn workspace tùy chỉnh thay vì mặc định
- **Disk Space**: Workspace tích lũy qua nhiều build → cần giám sát ổ đĩa

---

## 6. Luồng Hoàn Chỉnh: Từ Trigger Đến Build

```
Developer push code lên Git
        ↓
GitHub/GitLab gửi Webhook đến Jenkins Controller
        ↓
Controller nhận webhook → tạo Build Request
        ↓
Controller kiểm tra Build Queue
        ↓
Controller tìm Agent phù hợp (label match, Executor rảnh)
        ↓
Build được giao cho Executor trên Agent
        ↓
Agent checkout source code vào Workspace
        ↓
Agent thực thi các Pipeline Stage: Build → Test → Package → Deploy
        ↓
Kết quả build (SUCCESS/FAILURE) báo về Controller
        ↓
Controller lưu Build Log, Artifact, cập nhật Build History
        ↓
Controller gửi Notification (Slack, Email...)
```

---

## 7. Jenkins Architecture Trong Kubernetes

Khi chạy Jenkins trên Kubernetes, mô hình Controller/Agent được mở rộng với **dynamic agents** (agent động):

```
┌─────────────────────────────────────────────────────┐
│               Kubernetes Cluster                     │
│                                                      │
│  ┌─────────────────────────────┐                    │
│  │   Jenkins Controller Pod    │                    │
│  │   (StatefulSet, 1 replica)  │                    │
│  │   - PVC: /var/jenkins_home  │                    │
│  └──────────────┬──────────────┘                    │
│                 │ Kubernetes API                     │
│    ┌────────────┼────────────┐                      │
│    ▼            ▼            ▼                       │
│  ┌──────┐  ┌──────┐  ┌──────┐                      │
│  │Agent │  │Agent │  │Agent │  ← Tạo khi cần       │
│  │Pod#1 │  │Pod#2 │  │Pod#3 │  ← Xóa sau build     │
│  └──────┘  └──────┘  └──────┘                      │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Lợi ích:** Auto-scaling (tự mở rộng) — không cần duy trì Agent cố định, tiết kiệm tài nguyên.

---

## Tóm Tắt Các Khái Niệm

| Khái Niệm | Định Nghĩa | Nằm Ở Đâu |
|-----------|-----------|-----------|
| **Controller** | Não trung tâm: điều phối, lưu config, phục vụ UI | Máy chủ Jenkins chính |
| **Agent/Node** | Máy thực thi build | Máy riêng biệt / Container / Pod |
| **Executor** | Số lượng build song song trên một node | Thuộc tính của Agent/Controller |
| **Build Queue** | Danh sách build đang chờ Executor rảnh | Quản lý bởi Controller |
| **Workspace** | Thư mục chứa source code khi build | Trên Agent |
| **JENKINS_HOME** | Thư mục gốc lưu mọi dữ liệu Jenkins | Trên Controller |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao số Executor trên Controller nên đặt bằng 0?**

A: Controller cần tài nguyên để phục vụ giao diện web, điều phối build Queue, và quản lý plugin. Nếu build chạy trên Controller, chúng cạnh tranh tài nguyên với các tác vụ điều phối, gây chậm UI và tăng rủi ro — lỗi build script có thể ảnh hưởng đến toàn bộ Jenkins process. Bằng cách tách biệt Controller và Agent, ta đảm bảo tính ổn định và dễ scale.

**Q: Sự khác biệt giữa JNLP Agent và SSH Agent là gì?**

A: JNLP Agent tự mở kết nối ra phía Controller (outbound), phù hợp khi Agent nằm sau firewall không nhận inbound connection. SSH Agent thì ngược lại — Controller SSH vào Agent để khởi động agent process, yêu cầu Agent có SSH server và Controller có thể reach Agent qua mạng. JNLP linh hoạt hơn trong môi trường cloud, còn SSH đơn giản hơn khi quản lý agent truyền thống.

**Q: Workspace có được chia sẻ giữa các Agent không?**

A: Không. Mỗi Agent có workspace riêng biệt trên file system của nó. Để chuyển file giữa các Agent trong cùng pipeline, dùng `stash`/`unstash` — Controller làm trung gian lưu trữ tạm thời. Đây là lý do quan trọng cần hiểu khi thiết kế multi-agent pipeline.
