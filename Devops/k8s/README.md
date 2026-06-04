# Kubernetes (K8s) — Lộ Trình Học và Vận Hành

> Hướng dẫn toàn diện về Kubernetes (K8s) — từ các khái niệm nền tảng đến vận hành production, bao gồm tất cả kỹ năng cốt lõi dành cho kỹ sư DevOps và Backend.

## Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Kiến trúc Kubernetes: Control Plane (mặt điều khiển) và Worker Node (nút xử lý)
- [ ] Pod, Deployment, ReplicaSet — đơn vị triển khai cơ bản
- [ ] Service và các kiểu phơi bày mạng (ClusterIP, NodePort, LoadBalancer)
- [ ] Namespace (không gian tên) và phân vùng tài nguyên
- [ ] kubectl — công cụ dòng lệnh quản lý cluster

### **Giai Đoạn 2: Kỹ Năng Vận Hành Cốt Lõi (Tuần 3–6)**

- [ ] ConfigMap và Secret — quản lý cấu hình và bí mật
- [ ] Volume và PersistentVolume (PV) / PersistentVolumeClaim (PVC)
- [ ] Resource Request và Limit — phân bổ và giới hạn tài nguyên
- [ ] Liveness Probe và Readiness Probe — kiểm tra sức khoẻ ứng dụng
- [ ] Rolling Update và Rollback — triển khai không gián đoạn
- [ ] RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò)

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7–10)**

- [ ] Horizontal Pod Autoscaler (HPA — Tự Động Mở Rộng Pod Theo Chiều Ngang)
- [ ] Vertical Pod Autoscaler (VPA — Tự Động Điều Chỉnh Tài Nguyên Pod)
- [ ] Ingress Controller và quản lý traffic HTTP/HTTPS
- [ ] NetworkPolicy (chính sách mạng) — kiểm soát luồng lưu lượng
- [ ] StatefulSet cho workload có trạng thái (database, message queue)
- [ ] DaemonSet và CronJob

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] Helm — quản lý package và template cho Kubernetes
- [ ] Custom Resource Definition (CRD — Định Nghĩa Tài Nguyên Tuỳ Chỉnh) và Operator
- [ ] Service Mesh với Istio hoặc Linkerd
- [ ] GitOps với ArgoCD hoặc Flux
- [ ] Bảo mật cluster nâng cao (Pod Security Admission, OPA Gatekeeper)
- [ ] Multi-cluster và Federation (liên kết nhiều cluster)

---

## Năng Lực Cốt Lõi

| Năng Lực                            | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| ----------------------------------- | ---------- | --------- | ---------- |
| **Kiến Trúc và Khái Niệm Cơ Bản**  | ⭐⭐⭐      | 1 tuần    | -          |
| **Triển Khai và Quản Lý Workload**  | ⭐⭐⭐      | 2 tuần    | -          |
| **Networking và Service**           | ⭐⭐⭐      | 2 tuần    | -          |
| **Bảo Mật và RBAC**                 | ⭐⭐⭐      | 2 tuần    | -          |
| **Lưu Trữ và Volume**               | ⭐⭐⭐      | 1 tuần    | -          |
| **Monitoring và Logging**           | ⭐⭐⭐      | 1 tuần    | -          |
| **Auto Scaling (Tự Động Mở Rộng)** | ⭐⭐⭐      | 1 tuần    | -          |
| **CI/CD và GitOps**                 | ⭐⭐⭐      | 2 tuần    | -          |
| **Helm và Package Management**      | ⭐⭐        | 1 tuần    | -          |
| **Service Mesh**                    | ⭐⭐        | 2 tuần    | -          |

---

## Tổng Quan Chủ Đề

### **1. Kiến Trúc Kubernetes** (`01-architecture/`)

- Control Plane: API Server, etcd, Scheduler, Controller Manager
- Worker Node: kubelet, kube-proxy, Container Runtime (thời gian chạy container)
- Luồng xử lý request từ kubectl đến Pod
- Kubernetes Objects (tài nguyên Kubernetes) và API Groups
- Declarative Model (mô hình khai báo) vs Imperative Model (mô hình mệnh lệnh)

### **2. Workload Management (Quản Lý Tải Công Việc)** (`02-workload/`)

- **Pod** — đơn vị triển khai nhỏ nhất, chứa một hoặc nhiều container
- **Deployment** — quản lý ReplicaSet, hỗ trợ rolling update và rollback
- **StatefulSet** — Deployment có thứ tự và định danh ổn định, dùng cho database
- **DaemonSet** — đảm bảo mỗi node chạy đúng một Pod (ví dụ: log agent)
- **Job và CronJob** — tác vụ một lần hoặc định kỳ
- Init Container và Sidecar Container

### **3. Networking (Mạng)** (`03-networking/`)

- Mô hình mạng Kubernetes: mỗi Pod có IP riêng
- **Service** — ClusterIP, NodePort, LoadBalancer, ExternalName
- **Ingress** và Ingress Controller (NGINX, Traefik, AWS ALB)
- DNS nội bộ cluster với CoreDNS
- **NetworkPolicy** — giới hạn lưu lượng giữa các Pod
- CNI (Container Network Interface — Giao Diện Mạng Container): Calico, Flannel, Cilium

### **4. Storage (Lưu Trữ)** (`04-storage/`)

- Volume, emptyDir, hostPath
- **PersistentVolume (PV)** và **PersistentVolumeClaim (PVC)**
- StorageClass và Dynamic Provisioning (cấp phát động)
- Access Modes: ReadWriteOnce, ReadOnlyMany, ReadWriteMany
- CSI (Container Storage Interface — Giao Diện Lưu Trữ Container)
- Backup và snapshot volume

### **5. Configuration và Secret** (`05-config-secret/`)

- **ConfigMap** — lưu cấu hình dưới dạng key-value hoặc file
- **Secret** — lưu dữ liệu nhạy cảm, mã hoá base64
- Gắn ConfigMap / Secret vào Pod qua biến môi trường hoặc volume
- External Secrets Operator — tích hợp với AWS Secrets Manager, HashiCorp Vault
- Sealed Secrets — mã hoá Secret để lưu trên Git an toàn

### **6. Security (Bảo Mật)** (`06-security/`)

- **RBAC** — Role, ClusterRole, RoleBinding, ClusterRoleBinding
- ServiceAccount (tài khoản dịch vụ) và token
- Pod Security Admission (PSA — Kiểm Soát Bảo Mật Pod): Privileged, Baseline, Restricted
- NetworkPolicy để cô lập lưu lượng
- Image Scanning (quét lỗ hổng image) với Trivy, Snyk
- mTLS (mutual TLS — TLS hai chiều) qua Service Mesh
- Secrets Encryption at Rest (mã hoá bí mật lưu trữ)

### **7. Scaling (Mở Rộng)** (`07-scaling/`)

- **HPA (Horizontal Pod Autoscaler)** — mở rộng số lượng Pod dựa trên CPU/memory/custom metric
- **VPA (Vertical Pod Autoscaler)** — điều chỉnh resource request/limit của Pod
- **Cluster Autoscaler** — tự động thêm/bớt node khi cần
- KEDA (Kubernetes Event-Driven Autoscaling — Tự Động Mở Rộng Dựa Trên Sự Kiện)
- Resource Quota và LimitRange — giới hạn tài nguyên theo Namespace

### **8. Monitoring và Observability (Giám Sát và Khả Năng Quan Sát)** (`08-monitoring/`)

- **Prometheus** — thu thập metric từ cluster và ứng dụng
- **Grafana** — dashboard trực quan hoá metric
- **Loki** — hệ thống tổng hợp log
- **Jaeger / Tempo** — distributed tracing (theo dõi phân tán)
- kube-state-metrics và metrics-server
- Alert Manager — cấu hình cảnh báo và kênh thông báo
- SLO (Service Level Objective — Mục Tiêu Cấp Độ Dịch Vụ) và SLI

### **9. CI/CD và GitOps** (`09-cicd-gitops/`)

- Pipeline CI/CD với GitHub Actions, GitLab CI, Jenkins
- **Helm** — cài đặt, nâng cấp, rollback ứng dụng qua chart
- **ArgoCD** — GitOps continuous delivery cho Kubernetes
- **Flux** — GitOps operator tự đồng bộ cluster với Git repository
- Image registry và tagging strategy (chiến lược đặt tên tag)
- Progressive Delivery: Canary, Blue-Green Deployment

### **10. Troubleshooting (Xử Lý Sự Cố)** (`10-troubleshooting/`)

- Pod không khởi động: CrashLoopBackOff, ImagePullBackOff, Pending
- Node NotReady và taint/toleration
- OOMKilled (Out of Memory — Hết Bộ Nhớ) và resource starvation
- Service không kết nối được — debug DNS và Endpoint
- Ingress không hoạt động — kiểm tra certificate và rule
- Vấn đề lưu trữ: PVC Pending, mount error
- Log và event: `kubectl logs`, `kubectl describe`, `kubectl get events`

### **11. Nền Tảng Cloud và Managed Kubernetes** (`11-cloud-platforms/`)

- **EKS (Elastic Kubernetes Service)** — Kubernetes trên AWS
- **GKE (Google Kubernetes Engine)** — Kubernetes trên GCP
- **AKS (Azure Kubernetes Service)** — Kubernetes trên Azure
- So sánh managed vs self-managed cluster
- IAM (Identity and Access Management — Quản Lý Danh Tính và Truy Cập) tích hợp với cloud provider
- Spot/Preemptible Node — tối ưu chi phí

### **12. Chuẩn Bị Phỏng Vấn** (`12-interview-prep/`)

- Top 30 câu hỏi phỏng vấn Kubernetes
- System Design với Kubernetes
- Câu chuyện sự cố theo phương pháp STAR (Situation — Tình Huống, Task — Nhiệm Vụ, Action — Hành Động, Result — Kết Quả)
- Bài tập thực hành và câu hỏi thực chiến
- Kế hoạch ôn tập 90 ngày

---

## Theo Nền Tảng Cloud

### **Amazon EKS (Elastic Kubernetes Service)**

```
Điểm mạnh: Tích hợp sâu với AWS, IAM Roles for Service Accounts (IRSA),
           Fargate profile (chạy Pod serverless), add-on quản lý
Phù hợp: Hệ thống đang dùng AWS, cần tích hợp RDS, S3, ALB
Tham khảo: 11-cloud-platforms/eks.md
```

### **Google GKE (Google Kubernetes Engine)**

```
Điểm mạnh: Autopilot mode (tự động quản lý node), Workload Identity,
           tích hợp Cloud SQL, GCS, Artifact Registry
Phù hợp: Hệ thống trên GCP, cần managed cluster đơn giản nhất
Tham khảo: 11-cloud-platforms/gke.md
```

### **Azure AKS (Azure Kubernetes Service)**

```
Điểm mạnh: Tích hợp Active Directory, Azure DevOps, managed node pool,
           Windows container support
Phù hợp: Hệ thống enterprise trên Azure, dùng .NET stack
Tham khảo: 11-cloud-platforms/aks.md
```

### **Self-Managed (Tự Quản Lý)**

```
Điểm mạnh: Toàn quyền kiểm soát, linh hoạt, tiết kiệm chi phí
Công cụ: kubeadm, k3s, RKE2, Kubespray
Phù hợp: On-premise, bare-metal, môi trường air-gapped
Tham khảo: 11-cloud-platforms/self-managed.md
```

---

## Liên Kết Nhanh

| Chủ Đề                              | Thư Mục / File                              | Độ Ưu Tiên    |
| ----------------------------------- | ------------------------------------------- | ------------- |
| Bắt đầu từ đây                      | [README.md](./README.md)                    | Đọc đầu tiên  |
| Cấu trúc toàn bộ tài liệu           | [INDEX.md](./INDEX.md)                      | Tổng quan     |
| Kiến trúc K8s                       | [01-architecture/](./01-architecture/)      | Nền tảng      |
| Triển khai workload                 | [02-workload/](./02-workload/)              | Thiết yếu     |
| Networking và Ingress               | [03-networking/](./03-networking/)          | Thiết yếu     |
| Bảo mật và RBAC                     | [06-security/](./06-security/)              | Quan trọng    |
| Scaling và Auto Scaling             | [07-scaling/](./07-scaling/)               | Quan trọng    |
| Monitoring với Prometheus + Grafana | [08-monitoring/](./08-monitoring/)          | Quan trọng    |
| Xử lý sự cố                         | [10-troubleshooting/](./10-troubleshooting/) | Thực chiến    |
| Câu hỏi phỏng vấn                   | [12-interview-prep/](./12-interview-prep/)  | Trước phỏng vấn |

---

## Ma Trận Kỹ Năng

### Người Mới (0–1 năm kinh nghiệm)

- [ ] Hiểu kiến trúc Control Plane và Worker Node
- [ ] Tạo và quản lý Pod, Deployment, Service
- [ ] Dùng kubectl để debug cơ bản
- [ ] Hiểu ConfigMap và Secret
- [ ] Cài đặt ứng dụng đơn giản bằng Helm

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Thiết kế Deployment strategy (Rolling, Canary, Blue-Green)
- [ ] Cấu hình HPA và resource limit phù hợp
- [ ] Thiết lập NetworkPolicy và RBAC cơ bản
- [ ] Vận hành StatefulSet cho database
- [ ] Thiết lập pipeline CI/CD với ArgoCD hoặc Flux
- [ ] Cấu hình Prometheus và Grafana

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Thiết kế multi-cluster và federation
- [ ] Xây dựng Operator và CRD tuỳ chỉnh
- [ ] Tích hợp Service Mesh (Istio / Linkerd)
- [ ] Bảo mật cluster theo tiêu chuẩn CIS Benchmark
- [ ] Capacity planning (lập kế hoạch dung lượng) và cost optimization (tối ưu chi phí)
- [ ] Incident command và post-mortem (phân tích sự cố sau thực tế)

---

## Bắt Đầu Ngay

### Bước 1: Xác Định Mục Tiêu Học Tập

```
Chọn hướng của bạn:
- Kỹ sư Backend cần hiểu K8s để deploy ứng dụng
- DevOps Engineer vận hành và xây dựng platform
- Platform Engineer xây dựng internal developer platform (IDP)
- Site Reliability Engineer (SRE) đảm bảo độ tin cậy hệ thống
```

### Bước 2: Dựng Môi Trường Lab

```bash
# Cài đặt cluster local bằng kind (Kubernetes IN Docker)
kind create cluster --name k8s-lab

# Hoặc dùng minikube
minikube start --cpus=4 --memory=8192

# Cài đặt kubectl
# Sau đó kiểm tra cluster
kubectl cluster-info
kubectl get nodes
```

### Bước 3: Học và Thực Hành

```
1. Đọc lý thuyết một module (30 phút)
2. Thực hành trên cluster lab (30–60 phút)
3. Phá vỡ và sửa lại — học qua lỗi (30 phút)
4. Kiểm tra checklist cuối module (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện theo phương pháp STAR:
- Situation (Tình huống): Bối cảnh, hệ thống, vấn đề
- Task (Nhiệm vụ): Yêu cầu, mục tiêu cần đạt
- Action (Hành động): Những bước bạn đã thực hiện
- Result (Kết quả): Kết quả đo lường được
```

---

## Tài Liệu Tham Khảo

### Sách Quan Trọng

- **"Kubernetes in Action"** — Marko Luksa — Toàn diện nhất về K8s
- **"Kubernetes Patterns"** — Bilgin Ibryam & Roland Huß — Design pattern cho K8s
- **"The Kubernetes Book"** — Nigel Poulton — Dành cho người mới bắt đầu
- **"Production Kubernetes"** — Josh Rosso et al. — Vận hành production thực tế
- **"Designing Distributed Systems"** — Brendan Burns — Nền tảng hệ thống phân tán

### Tài Liệu Chính Thức

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Istio Documentation](https://istio.io/latest/docs/)

### Blog và Nguồn Học Tập

- Kubernetes Blog (kubernetes.io/blog)
- CNCF (Cloud Native Computing Foundation — Tổ Chức Điện Toán Đám Mây Native) Blog
- Learnk8s.io — hướng dẫn thực tế chất lượng cao
- iximiuz.com/en/posts — bài viết sâu về container và K8s

---

## Chuẩn Bị Phỏng Vấn

### Câu Hỏi Theo Chủ Đề

#### Kiến Trúc

- [ ] Mô tả luồng từ khi chạy `kubectl apply` đến khi Pod chạy
- [ ] etcd lưu gì và tại sao quan trọng?
- [ ] Scheduler quyết định đặt Pod lên node nào?

#### Networking

- [ ] ClusterIP, NodePort, LoadBalancer khác nhau thế nào?
- [ ] Ingress Controller hoạt động ra sao?
- [ ] CoreDNS giải quyết tên service như thế nào?

#### Storage

- [ ] Khi nào dùng StatefulSet thay vì Deployment?
- [ ] PV và PVC khác nhau thế nào?
- [ ] StorageClass dynamic provisioning hoạt động ra sao?

#### Bảo Mật

- [ ] Giải thích RBAC trong Kubernetes
- [ ] ServiceAccount dùng để làm gì?
- [ ] Làm thế nào để Pod truy cập AWS S3 an toàn trên EKS?

#### Vận Hành

- [ ] Debug Pod ở trạng thái CrashLoopBackOff như thế nào?
- [ ] HPA scale dựa trên gì? Cấu hình ra sao?
- [ ] Làm thế nào để deploy zero-downtime trên Kubernetes?

#### Sự Cố Thực Tế

- [ ] Kể về một sự cố Kubernetes bạn đã xử lý (phương pháp STAR)
- [ ] Phân tích nguyên nhân gốc rễ (Root Cause Analysis)
- [ ] Các bước phòng ngừa bạn đã triển khai

Xem `12-interview-prep/` để có hướng dẫn Q&A đầy đủ.

---

## Tự Đánh Giá Trước Phỏng Vấn

- [ ] Có thể giải thích kiến trúc K8s không cần nhìn tài liệu
- [ ] Có thể debug Pod bị lỗi chỉ dùng kubectl
- [ ] Có thể thiết kế Deployment có HPA và health check
- [ ] Có thể giải thích RBAC và ServiceAccount
- [ ] Có thể cấu hình Ingress cho nhiều service
- [ ] Có thể thiết lập Prometheus scrape ứng dụng
- [ ] Có thể kể ít nhất 2 câu chuyện sự cố theo STAR
- [ ] Hiểu sự khác nhau giữa EKS, GKE, AKS
- [ ] Có thể triển khai ứng dụng bằng Helm chart
- [ ] Có thể giải thích trade-off giữa Deployment và StatefulSet

---

## Công Cụ Hỗ Trợ

### CLI (Command Line Interface — Giao Diện Dòng Lệnh)

- **kubectl** — công cụ chính quản lý cluster
- **helm** — quản lý Kubernetes package
- **k9s** — giao diện terminal trực quan cho cluster
- **kubectx / kubens** — chuyển đổi nhanh giữa context và namespace
- **stern** — xem log nhiều Pod cùng lúc

### Công Cụ Debug

- **k8slens** — IDE quản lý Kubernetes (có giao diện đồ hoạ)
- **kubectl-debug** — chạy container debug bên cạnh Pod
- **kube-score** — phân tích manifest K8s theo best practice
- **Popeye** — quét cluster tìm vấn đề cấu hình

### Môi Trường Lab

- **kind (Kubernetes IN Docker)** — cluster K8s chạy trong Docker
- **minikube** — cluster K8s single-node trên máy local
- **k3s** — phân phối K8s nhẹ, phù hợp edge và lab
- **Rancher Desktop** — môi trường K8s + container trên desktop

---

**Cập Nhật Lần Cuối:** 2026-05-09
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
