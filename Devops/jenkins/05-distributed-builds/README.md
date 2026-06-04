# 05 — Distributed Builds: Build Phân Tán

> Tổng quan về kiến trúc Distributed Builds (build phân tán) trong Jenkins — cách phân phối workload cho nhiều Agent (tác nhân) để tăng tốc độ và khả năng mở rộng của hệ thống CI/CD.

---

## Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc Controller/Agent](#kiến-trúc-controlleragent)
3. [Các Loại Agent](#các-loại-agent)
4. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)
5. [So Sánh Nhanh](#so-sánh-nhanh)
6. [Lộ Trình Học](#lộ-trình-học)

---

## Tổng Quan

**Distributed Builds** (build phân tán) là cơ chế Jenkins phân phối công việc build ra nhiều máy (Node) thay vì chạy tất cả trên một máy duy nhất. Đây là nền tảng để:

- **Mở rộng quy mô** — chạy hàng trăm build song song mà không tắc nghẽn
- **Cô lập môi trường** — mỗi build chạy trong môi trường riêng, không ảnh hưởng nhau
- **Tối ưu tài nguyên** — dùng agent phù hợp cho từng loại job (Linux/Windows, GPU, ARM)
- **Tăng độ tin cậy** — khi một agent lỗi, các agent khác tiếp tục hoạt động

```
Jenkins Controller (Bộ Điều Phối)
         │
         ├──── Agent Node 1 (Linux x86_64)     ← Java builds
         ├──── Agent Node 2 (Linux ARM)         ← Embedded builds
         ├──── Agent Node 3 (Windows)           ← .NET builds
         ├──── Docker Agent (tạm thời)          ← Container builds
         └──── Kubernetes Pod Agent (động)      ← Cloud-native builds
```

---

## Kiến Trúc Controller/Agent

> **Lưu ý:** Jenkins đổi tên từ "Master/Slave" sang **"Controller/Agent"** từ phiên bản 2.307+ để phù hợp với thuật ngữ toàn diện hơn (inclusive language).

### Controller — Bộ Điều Phối Trung Tâm

**Controller** (bộ điều phối) chịu trách nhiệm:

| Nhiệm Vụ | Mô Tả |
|----------|-------|
| **Lên lịch build** | Phân bổ job cho agent phù hợp dựa trên Node Label |
| **Lưu cấu hình** | Quản lý tất cả job config, plugin, credentials trong `JENKINS_HOME` |
| **Giao diện Web UI** | Cung cấp dashboard cho người dùng |
| **Điều phối Pipeline** | Thực thi logic điều phối trong Jenkinsfile |

> **Best Practice:** Controller **không nên** chạy build trực tiếp trong môi trường production. Tắt Executor trên Controller (`Manage Jenkins → Nodes → Built-In Node → Executors = 0`).

### Agent — Máy Thực Thi Build

**Agent** (tác nhân) là máy hoặc container nhận lệnh từ Controller và thực thi các build step:

```
Controller gửi lệnh → Agent nhận và thực thi → Kết quả gửi về Controller
```

Mỗi Agent có:
- **Executor** (bộ thực thi) — số lượng build có thể chạy đồng thời trên agent đó
- **Workspace** (không gian làm việc) — thư mục chứa source code khi build
- **Node Label** (nhãn node) — thẻ để Controller chọn đúng agent cho từng job

### Giao Tiếp Controller — Agent

```
┌──────────────────────────────────────────────────────────┐
│                    Jenkins Controller                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │ Build    │  │ Job      │  │ Plugin Manager       │   │
│  │ Queue    │  │ Config   │  │ (Quản lý Plugin)     │   │
│  └────┬─────┘  └──────────┘  └──────────────────────┘   │
└───────┼──────────────────────────────────────────────────┘
        │
        │ Kết nối qua JNLP (port 50000) hoặc SSH (port 22)
        │
   ┌────┴──────────────────────────────┐
   │                                   │
   ▼                                   ▼
┌──────────────┐              ┌──────────────┐
│  Agent 1     │              │  Agent 2     │
│  (SSH)       │              │  (JNLP)      │
│  Executor: 4 │              │  Executor: 2 │
└──────────────┘              └──────────────┘
```

---

## Các Loại Agent

### 1. Static Agent — Agent Cố Định

Agent được cài đặt và cấu hình thủ công trên một máy vật lý hoặc VM (Virtual Machine — Máy Ảo). Luôn sẵn sàng nhận build.

- **Kết nối JNLP** (Java Network Launch Protocol — Giao Thức Khởi Chạy Mạng Java): Agent chủ động kết nối vào Controller
- **Kết nối SSH** (Secure Shell — Giao Thức Shell Bảo Mật): Controller chủ động SSH vào agent

**File:** [1-agent-configuration.md](1-agent-configuration.md)

---

### 2. Docker Agent — Agent Trong Container

Build chạy bên trong Docker container (vùng chứa Docker) tạm thời. Container được tạo khi build bắt đầu và xóa sau khi build hoàn thành.

```
Build bắt đầu → Docker Pull Image → Chạy Container → Build → Xóa Container
```

**Ưu điểm:** Môi trường build sạch sẽ, tái sử dụng Docker image, không cần cài đặt thủ công.

**File:** [2-docker-agents.md](2-docker-agents.md)

---

### 3. Kubernetes Agent — Agent Động Trên K8s

Kubernetes (K8s — Hệ Thống Điều Phối Container) tự động tạo Pod (đơn vị chạy container trên K8s) để làm agent khi có build, và xóa Pod sau khi build xong.

```
Build request → K8s tạo Pod → Agent khởi động → Build → Pod bị xóa
```

**Ưu điểm:** Scale (mở rộng) tự động, tận dụng tài nguyên K8s cluster hiện có, cô lập hoàn toàn.

**File:** [3-kubernetes-agents.md](3-kubernetes-agents.md)

---

### 4. Node Management — Quản Lý Node

Quản lý vòng đời (lifecycle) của tất cả node: thêm/xóa, giám sát trạng thái, cấu hình Executor, xử lý node offline.

**File:** [4-node-management.md](4-node-management.md)

---

## So Sánh Nhanh

| Tiêu Chí | Static Agent (Cố Định) | Docker Agent | Kubernetes Agent |
|----------|----------------------|--------------|-----------------|
| **Thời gian khởi động** | Ngay lập tức | 10–60 giây (pull image) | 30–90 giây (tạo Pod) |
| **Cô lập môi trường** | Thấp (dùng chung OS) | Cao (container riêng) | Rất cao (Pod riêng) |
| **Quản lý tài nguyên** | Thủ công | Semi-tự động | Tự động hoàn toàn |
| **Chi phí duy trì** | Cao (quản lý VM) | Trung bình | Thấp (K8s quản lý) |
| **Phù hợp cho** | Build đặc thù (GPU, legacy) | CI hiện đại, team nhỏ-vừa | Large scale, cloud-native |
| **Yêu cầu hạ tầng** | VM / bare metal | Docker daemon | Kubernetes cluster |

> **Khuyến nghị:** Với hệ thống mới, ưu tiên **Kubernetes Agent** nếu có cluster K8s. Dùng **Docker Agent** nếu không có K8s. Chỉ dùng **Static Agent** cho các trường hợp đặc thù mà Docker/K8s không đáp ứng được (driver phần cứng, license cố định, legacy OS).

---

## Lộ Trình Học

```
Bước 1: Nắm kiến trúc Controller/Agent ──► README.md (file này)
        ↓
Bước 2: Cấu hình Agent cơ bản (SSH/JNLP) ─► 1-agent-configuration.md
        ↓
Bước 3: Docker Agent (build trong container) ► 2-docker-agents.md
        ↓
Bước 4: Kubernetes Agent (dynamic scaling) ──► 3-kubernetes-agents.md
        ↓
Bước 5: Quản lý Node toàn diện ─────────────► 4-node-management.md
```

### Checklist Năng Lực

#### Beginner
- [ ] Giải thích được sự khác biệt giữa Controller và Agent
- [ ] Kết nối một Static Agent qua SSH hoặc JNLP
- [ ] Gán Node Label và dùng `agent { label '...' }` trong Jenkinsfile
- [ ] Xem trạng thái Agent trên trang `/computer` của Jenkins

#### Intermediate
- [ ] Cấu hình Docker Agent với `agent { docker { image '...' } }`
- [ ] Viết Pipeline dùng nhiều Agent khác nhau cho các stage
- [ ] Triển khai Kubernetes Plugin với Pod Template cơ bản
- [ ] Xử lý Agent offline và xem nguyên nhân kết nối thất bại

#### Advanced
- [ ] Thiết kế Pod Template tùy chỉnh với nhiều container (sidecar)
- [ ] Cấu hình JCasC (Jenkins Configuration as Code — Cấu Hình Jenkins Dưới Dạng Code) cho Agent
- [ ] Tối ưu số lượng Executor và resource request/limit trên K8s
- [ ] Implement Cloud Agent tự scale theo nhu cầu

---

## Câu Hỏi Phỏng Vấn Liên Quan

- **Tại sao nên đặt Executor = 0 trên Controller trong production?**
- **JNLP Agent và SSH Agent khác nhau ở chỗ nào? Khi nào dùng cái nào?**
- **Kubernetes Agent có ưu điểm gì so với Static Agent?**
- **Cách cấu hình để build Java và build NodeJS dùng agent khác nhau?**
- **Khi Agent bị offline giữa chừng, build xử lý thế nào?**

---

**Thời Gian Học:** 5–6 giờ
**Độ Khó:** ⭐⭐ Trung Bình
**Cập Nhật:** 2026-05-11
