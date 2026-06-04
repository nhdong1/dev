# Kế Hoạch Học Kubernetes 90 Ngày

> Lộ trình học có cấu trúc từ người mới đến sẵn sàng phỏng vấn senior-level. Mỗi tuần có mục tiêu rõ ràng, tài nguyên, bài tập, và checkpoint tự đánh giá.

## Tổng Quan

```
Tháng 1 (Ngày 1–30):   Nền tảng vững chắc
Tháng 2 (Ngày 31–60):  Kỹ năng vận hành thực chiến
Tháng 3 (Ngày 61–90):  Nâng cao, tổng hợp, phỏng vấn
```

## Điều Kiện Tiên Quyết

Trước khi bắt đầu, đảm bảo bạn đã:
- [ ] Biết cơ bản về Linux CLI (ls, cd, grep, pipe, sudo)
- [ ] Hiểu container là gì (Docker cơ bản: build, run, push)
- [ ] Biết YAML syntax cơ bản
- [ ] Có tài khoản cloud hoặc máy tính đủ mạnh (≥ 8GB RAM) để chạy cluster local

---

## Tháng 1: Nền Tảng Vững Chắc (Ngày 1–30)

### Tuần 1 (Ngày 1–7): Kiến Trúc và Cài Đặt Môi Trường

**Mục tiêu cuối tuần:**
- Hiểu kiến trúc K8s không cần nhìn tài liệu
- Chạy được cluster local và thao tác kubectl cơ bản

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 1 | Tổng quan K8s, lý do tồn tại, so sánh với Docker Compose | 90 phút | `01-architecture/README.md` |
| 2 | Control Plane: API Server, etcd, Scheduler, Controller Manager | 90 phút | `01-architecture/control-plane.md` |
| 3 | Worker Node: kubelet, kube-proxy, container runtime | 60 phút | `01-architecture/worker-node.md` |
| 4 | Cài kind/minikube, thực hành kubectl basic commands | 120 phút | Lab bên dưới |
| 5 | Kubernetes Objects (tài nguyên K8s): Pod, Deployment, Service, Namespace | 90 phút | `01-architecture/kubernetes-objects.md` |
| 6 | Luồng xử lý từ `kubectl apply` đến Pod chạy | 60 phút | `01-architecture/request-flow.md` |
| 7 | Review + tự đặt câu hỏi + ôn lại điểm yếu | 90 phút | |

**Lab Thực Hành Tuần 1:**
```bash
# Cài môi trường
kind create cluster --name k8s-lab
kubectl cluster-info
kubectl get nodes

# Thực hành cơ bản
kubectl create deployment nginx-test --image=nginx --replicas=3
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash
kubectl delete pod <pod-name>  # Quan sát ReplicaSet tạo Pod mới

# Expose Deployment
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get services
```

**Checkpoint Tuần 1:**
- [ ] Giải thích được vai trò của API Server, etcd, Scheduler (không nhìn tài liệu, trong 3 phút)
- [ ] Chạy được cluster local
- [ ] Tạo và xoá Deployment, Pod, Service bằng kubectl

---

### Tuần 2 (Ngày 8–14): Workload Management

**Mục tiêu cuối tuần:**
- Hiểu và sử dụng được Deployment, StatefulSet, DaemonSet
- Cấu hình health probe đúng cách

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 8 | Pod lifecycle, multi-container Pod, restart policy | 90 phút | `02-workload/pod.md` |
| 9 | Deployment: rolling update, rollback, strategy | 120 phút | `02-workload/deployment.md` |
| 10 | StatefulSet: identity, headless service, VolumeClaimTemplate | 90 phút | `02-workload/statefulset.md` |
| 11 | DaemonSet và Job/CronJob | 60 phút | `02-workload/daemonset.md`, `job-cronjob.md` |
| 12 | Liveness, Readiness, Startup Probe | 90 phút | `02-workload/health-probes.md` |
| 13 | Init Container và Sidecar Container pattern | 60 phút | Bài tập 12 trong hands-on |
| 14 | Review + thực hành rolling update và rollback (Bài tập 5) | 120 phút | `hands-on-scenarios.md` |

**Checkpoint Tuần 2:**
- [ ] Giải thích sự khác nhau giữa Deployment và StatefulSet (với ví dụ cụ thể)
- [ ] Thực hiện rolling update và rollback thành công
- [ ] Cấu hình readinessProbe + livenessProbe cho deployment

---

### Tuần 3 (Ngày 15–21): Networking

**Mục tiêu cuối tuần:**
- Hiểu mô hình mạng K8s từ đầu đến cuối
- Cấu hình được Ingress với TLS

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 15 | Mô hình mạng K8s: Pod network, CNI plugin | 90 phút | `03-networking/README.md` |
| 16 | Service types: ClusterIP, NodePort, LoadBalancer, ExternalName | 90 phút | `03-networking/service-types.md` |
| 17 | Ingress Controller, TLS termination, routing rules | 120 phút | `03-networking/ingress.md` |
| 18 | CoreDNS: service discovery, DNS record format | 60 phút | `03-networking/dns-coredns.md` |
| 19 | NetworkPolicy: ingress/egress rules | 90 phút | `03-networking/network-policy.md` |
| 20 | Thực hành Ingress với TLS (Bài tập 9) | 120 phút | `hands-on-scenarios.md` |
| 21 | Review networking + debug Service không kết nối (Bài tập debug) | 90 phút | |

**Checkpoint Tuần 3:**
- [ ] Giải thích được cách kube-proxy implement ClusterIP (iptables)
- [ ] Cấu hình Ingress với 2 host và TLS
- [ ] Debug Service không có Endpoint

---

### Tuần 4 (Ngày 22–28): Storage và Config Management

**Mục tiêu cuối tuần:**
- Tạo và sử dụng được PV, PVC, StorageClass
- Quản lý ConfigMap và Secret đúng cách

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 22 | Volume types: emptyDir, hostPath, projected | 60 phút | `04-storage/volumes.md` |
| 23 | PersistentVolume, PVC, StorageClass, dynamic provisioning | 120 phút | `04-storage/persistent-volume.md` |
| 24 | Access modes: RWO, ROX, RWX, RWOP | 45 phút | `04-storage/access-modes.md` |
| 25 | ConfigMap: tạo, mount, cập nhật live | 90 phút | `05-config-secret/configmap.md` |
| 26 | Secret: tạo, mount, encryption at rest | 90 phút | `05-config-secret/secret.md` |
| 27 | Thực hành ConfigMap + Secret + StatefulSet (Bài 7+8) | 120 phút | `hands-on-scenarios.md` |
| 28 | Review tháng 1 + Mock phỏng vấn nhỏ (10 câu đầu) | 120 phút | `INTERVIEW_GUIDE.md` |

**Checkpoint Tháng 1:**
- [ ] Deploy 3-tier application đầy đủ (Bài tập 1) thành công
- [ ] Trả lời được câu 1–14 trong `INTERVIEW_GUIDE.md`
- [ ] Hiểu và thực hành được mọi thứ trong Tuần 1–4

---

## Tháng 2: Kỹ Năng Vận Hành Thực Chiến (Ngày 31–60)

### Tuần 5 (Ngày 31–37): Bảo Mật và RBAC

**Mục tiêu cuối tuần:**
- Thiết kế RBAC cho team 5 người
- Hiểu Pod Security và cách hardening container

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 31 | RBAC overview: Role, ClusterRole, Binding | 90 phút | `06-security/rbac.md` |
| 32 | ServiceAccount, token, projected volume | 60 phút | `06-security/service-account.md` |
| 33 | Thực hành RBAC: tạo user với quyền hạn chế (Bài 4) | 120 phút | `hands-on-scenarios.md` |
| 34 | Pod Security Admission: Privileged/Baseline/Restricted | 90 phút | `06-security/pod-security.md` |
| 35 | Image security: scanning với Trivy, signed image | 60 phút | `06-security/image-security.md` |
| 36 | Network security: mTLS overview, Ingress TLS | 60 phút | `06-security/network-security.md` |
| 37 | Secrets encryption at rest + External Secrets | 90 phút | `05-config-secret/external-secrets.md` |

**Checkpoint Tuần 5:**
- [ ] Tạo Role + RoleBinding cho ServiceAccount và verify bằng `kubectl auth can-i`
- [ ] Giải thích IRSA trên EKS (IAM Roles for Service Accounts)
- [ ] Hardening một Deployment: runAsNonRoot, readOnlyRootFilesystem

---

### Tuần 6 (Ngày 38–44): Auto Scaling và Resource Management

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 38 | Resource Request và Limit, QoS class | 90 phút | `07-scaling/resource-management.md` |
| 39 | HPA: cấu hình, metric, behavior | 120 phút | `07-scaling/hpa.md` |
| 40 | Thực hành HPA + load test (Bài 3) | 120 phút | `hands-on-scenarios.md` |
| 41 | VPA: recommendation mode, auto mode | 60 phút | `07-scaling/vpa.md` |
| 42 | Cluster Autoscaler: scale-up, scale-down | 90 phút | `07-scaling/cluster-autoscaler.md` |
| 43 | KEDA: event-driven autoscaling, Kafka/SQS trigger | 90 phút | `07-scaling/keda.md` |
| 44 | ResourceQuota và LimitRange per namespace | 60 phút | `07-scaling/resource-management.md` |

**Checkpoint Tuần 6:**
- [ ] HPA scale đúng khi CPU > threshold trong bài tập
- [ ] Giải thích khi nào dùng HPA vs VPA vs Cluster Autoscaler
- [ ] Tính toán resource request/limit cho một service thực tế

---

### Tuần 7 (Ngày 45–51): Monitoring và Observability

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 45 | Prometheus: scrape config, metric types, PromQL cơ bản | 120 phút | `08-monitoring/prometheus-setup.md` |
| 46 | Grafana: import dashboard, tạo panel, alert rule | 90 phút | `08-monitoring/grafana-dashboards.md` |
| 47 | Loki + Promtail: thu thập log từ Pod | 90 phút | `08-monitoring/loki-logging.md` |
| 48 | kube-state-metrics và metrics-server | 60 phút | `08-monitoring/kube-state-metrics.md` |
| 49 | Distributed tracing với Jaeger: trace context | 60 phút | `08-monitoring/tracing.md` |
| 50 | SLO/SLI/SLA: định nghĩa và cấu hình alert | 90 phút | `08-monitoring/slo-alerting.md` |
| 51 | Thực hành: cài Prometheus + Grafana + xem K8s dashboard | 120 phút | |

**Checkpoint Tuần 7:**
- [ ] Viết PromQL query đếm số Pod restart trong 1 giờ
- [ ] Tạo alert khi Pod restart > 5 lần trong 10 phút
- [ ] Giải thích SLO/SLI bằng ví dụ cụ thể

---

### Tuần 8 (Ngày 52–58): CI/CD và GitOps

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 52 | Helm basics: chart structure, install, upgrade, rollback | 120 phút | `09-cicd-gitops/helm-basics.md` |
| 53 | Helm advanced: template, hook, test, dependency | 90 phút | `09-cicd-gitops/helm-advanced.md` |
| 54 | Thực hành Helm chart (Bài 11) | 120 phút | `hands-on-scenarios.md` |
| 55 | ArgoCD: GitOps concept, ApplicationSet, sync policy | 120 phút | `09-cicd-gitops/argocd.md` |
| 56 | Flux: Kustomization, HelmRelease | 60 phút | `09-cicd-gitops/flux.md` |
| 57 | GitHub Actions deploy lên K8s | 90 phút | `09-cicd-gitops/github-actions-k8s.md` |
| 58 | Canary deployment với Argo Rollouts | 90 phút | `09-cicd-gitops/progressive-delivery.md` |

**Checkpoint Tháng 2:**
- [ ] Trả lời được câu 15–25 trong `INTERVIEW_GUIDE.md`
- [ ] Deploy ứng dụng bằng Helm và ArgoCD thành công
- [ ] Giải thích GitOps workflow và lợi ích so với push-based CD

---

## Tháng 3: Nâng Cao và Chuẩn Bị Phỏng Vấn (Ngày 61–90)

### Tuần 9 (Ngày 61–67): Troubleshooting

**Mục tiêu cuối tuần:**
- Debug được mọi trạng thái Pod lỗi trong < 5 phút
- Xử lý được sự cố networking và storage

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 61 | Pod errors: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled | 120 phút | `10-troubleshooting/pod-errors.md` |
| 62 | Node issues: NotReady, taint, drain, cordon (Bài 10) | 90 phút | `hands-on-scenarios.md` |
| 63 | Networking debug: Service, DNS, Ingress | 90 phút | `10-troubleshooting/networking-debug.md` |
| 64 | Storage issues: PVC Pending, mount error | 60 phút | `10-troubleshooting/storage-issues.md` |
| 65 | Performance: throttling, high latency, resource starvation | 90 phút | `10-troubleshooting/performance-issues.md` |
| 66 | Thực hành debug ngẫu nhiên (tự tạo lỗi rồi sửa) | 120 phút | |
| 67 | Production checklist review | 60 phút | `10-troubleshooting/production-checklist.md` |

**Bài Tập Debug Ngẫu Nhiên — Ngày 66:**
```bash
# Tự tạo các tình huống lỗi và debug:
# 1. Deploy với image không tồn tại
kubectl create deployment broken1 --image=nginx:nonexistent-tag

# 2. Deploy với request > capacity cluster
kubectl create deployment broken2 --image=nginx
kubectl patch deployment broken2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","resources":{"requests":{"cpu":"100","memory":"1000Gi"}}}]}}}}'

# 3. Misconfigure Service selector
kubectl create service clusterip broken-svc --tcp=80:80 --dry-run=client -o yaml \
  | kubectl set selector -f - 'app=nonexistent-label' --local -o yaml | kubectl apply -f -

# Debug từng case trong < 5 phút
```

---

### Tuần 10 (Ngày 68–74): Cloud Platforms và Advanced Topics

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 68 | EKS deep dive: IRSA, Fargate, add-on, node group | 120 phút | `11-cloud-platforms/eks.md` |
| 69 | GKE: Autopilot, Workload Identity | 90 phút | `11-cloud-platforms/gke.md` |
| 70 | AKS: Azure AD integration, Windows container | 60 phút | `11-cloud-platforms/aks.md` |
| 71 | Self-managed: kubeadm, k3s, etcd backup/restore | 90 phút | `11-cloud-platforms/self-managed.md` |
| 72 | Advanced: Custom Resource Definition (CRD — Định Nghĩa Tài Nguyên Tuỳ Chỉnh) và Operator overview | 60 phút | External reading |
| 73 | Advanced: Service Mesh với Istio — overview | 90 phút | Istio docs |
| 74 | Review cloud platforms + so sánh EKS/GKE/AKS | 60 phút | |

---

### Tuần 11 (Ngày 75–81): System Design với K8s

**Mục tiêu cuối tuần:**
- Thiết kế được hệ thống microservices trên K8s trong 45 phút
- Trình bày trade-off tự tin

**Lịch học:**

| Ngày | Chủ Đề | Thời Gian | Tài Nguyên |
|------|--------|-----------|-----------|
| 75 | Framework system design cho K8s | 60 phút | `system-design.md` phần đầu |
| 76 | Case Study 1: Multi-tier web app | 90 phút | `system-design.md` |
| 77 | Case Study 2: Microservices e-commerce | 120 phút | `system-design.md` |
| 78 | Case Study 3: Data pipeline | 90 phút | `system-design.md` |
| 79 | Case Study 4: Multi-region HA | 90 phút | `system-design.md` |
| 80 | Tự thiết kế: "Deploy một nền tảng streaming video" | 120 phút | |
| 81 | Review system design + điểm yếu | 60 phút | |

**Bài Tập System Design Tự Luyện:**
```
Đề 1: Thiết kế CI/CD platform nội bộ cho 50 developer team
  - 200 build jobs/ngày
  - Build time < 10 phút
  - Artifact storage
  - Environment: dev, staging, production

Đề 2: Thiết kế monitoring stack cho 100 microservices
  - 1 triệu metric time series
  - Log retention 30 ngày
  - Alert < 30 giây

Đề 3: Thiết kế K8s platform cho fintech startup
  - PCI DSS compliance
  - Zero-trust networking
  - Audit log đầy đủ
```

---

### Tuần 12 (Ngày 82–90): Mock Phỏng Vấn và Ôn Tập Cuối

**Lịch học:**

| Ngày | Hoạt Động | Mục Tiêu |
|------|----------|---------|
| 82 | Đọc lại `INTERVIEW_GUIDE.md` — ghi chú điểm yếu | Xác định gap |
| 83 | Mock phỏng vấn kỹ thuật (10 câu — kiến trúc + workload) | Luyện nói |
| 84 | Mock phỏng vấn kỹ thuật (10 câu — networking + storage + security) | Luyện nói |
| 85 | Mock phỏng vấn kỹ thuật (10 câu — scaling + monitoring + cicd) | Luyện nói |
| 86 | Luyện kể câu chuyện STAR (2–3 câu chuyện đầy đủ) | Storytelling |
| 87 | Mock system design phỏng vấn 45 phút | End-to-end |
| 88 | Ôn điểm yếu phát hiện trong mock | Gap fill |
| 89 | Mock phỏng vấn đầy đủ 1 tiếng (tự record hoặc nhờ bạn) | Final check |
| 90 | Review checklist cuối + nghỉ ngơi | Confidence |

---

## Hướng Dẫn Mock Phỏng Vấn

### Cách Tự Mock Hiệu Quả

```
1. Không nhìn tài liệu trong khi mock
2. Nói to ra thành lời — không chỉ nghĩ trong đầu
3. Đặt timer: phỏng vấn kỹ thuật thường 60–90 phút
4. Record nếu có thể — nghe lại để cải thiện
5. Sau mỗi câu, tự chấm: 1–5 điểm về độ chính xác và sự rõ ràng
```

### Các Dạng Câu Hỏi Cần Luyện

**Dạng 1: Khái niệm (30%)**
```
"Giải thích X là gì?"
"X và Y khác nhau thế nào?"
"Khi nào dùng X thay vì Y?"
```

**Dạng 2: Thực hành (40%)**
```
"Bạn sẽ cấu hình X thế nào?"
"Debug tình huống Y ra sao?"
"YAML manifest cho Z trông như thế nào?"
```

**Dạng 3: Kinh nghiệm (30%)**
```
"Kể về lần bạn xử lý sự cố..."
"Quyết định kiến trúc khó nhất bạn đã đưa ra?"
"Team của bạn tiếp cận X thế nào?"
```

---

## Công Cụ và Tài Nguyên Theo Tuần

### Công Cụ Cần Làm Quen

| Giai Đoạn | Công Cụ | Mục Đích |
|-----------|---------|---------|
| Tháng 1 | kubectl, kind/minikube | Thao tác cluster |
| Tháng 1 | k9s | Quan sát cluster interactively |
| Tháng 2 | Helm, ArgoCD | Deploy và GitOps |
| Tháng 2 | Prometheus, Grafana | Monitoring |
| Tháng 3 | kubectx/kubens | Multi-cluster |
| Tháng 3 | stern | Multi-pod log |

### Sách Theo Giai Đoạn

| Giai Đoạn | Sách | Mức Độ |
|-----------|------|--------|
| Tháng 1 | "The Kubernetes Book" — Nigel Poulton | Người mới |
| Tháng 2 | "Kubernetes in Action" — Marko Luksa | Trung cấp |
| Tháng 3 | "Production Kubernetes" — Josh Rosso | Nâng cao |

### Khóa Học Online (Tham Khảo Thêm)

| Tháng | Khóa | Nền Tảng |
|-------|------|---------|
| 1–2 | Kubernetes for Developers (LFD259) | Linux Foundation |
| 2–3 | Certified Kubernetes Administrator (CKA) | CNCF |
| 3 | Certified Kubernetes Security Specialist (CKS) | CNCF |

---

## Checklist Tiến Độ 90 Ngày

### Tháng 1 — Nền Tảng

- [ ] **Kiến Trúc:** Giải thích không cần tài liệu
- [ ] **Workload:** Deploy Deployment, StatefulSet, DaemonSet, Job
- [ ] **Networking:** Cấu hình Service, Ingress với TLS
- [ ] **Storage:** Tạo PVC, mount vào Pod
- [ ] **Config:** Mount ConfigMap và Secret theo nhiều cách
- [ ] **Bài tập hoàn thành:** 1, 2, 5, 8

### Tháng 2 — Vận Hành

- [ ] **RBAC:** Tạo user với quyền hạn chế, verify
- [ ] **HPA:** Scale up/down khi load thay đổi
- [ ] **Monitoring:** Prometheus + Grafana chạy trên cluster lab
- [ ] **Helm:** Deploy và upgrade ứng dụng bằng chart
- [ ] **GitOps:** ArgoCD sync ứng dụng từ Git repo
- [ ] **Bài tập hoàn thành:** 3, 4, 6, 7, 9, 11

### Tháng 3 — Nâng Cao

- [ ] **Debug:** Fix mọi trạng thái lỗi trong < 5 phút
- [ ] **System Design:** Thiết kế hệ thống trong 45 phút
- [ ] **STAR Stories:** Chuẩn bị 3 câu chuyện sự cố
- [ ] **Mock PV:** Trả lời tự tin 30 câu hỏi
- [ ] **Bài tập hoàn thành:** 10, 12

---

## Dấu Hiệu Sẵn Sàng Phỏng Vấn

Bạn đã sẵn sàng khi:

```
Kỹ thuật:
✅ Giải thích kiến trúc K8s đầy đủ trong 5 phút không nhìn tài liệu
✅ Debug CrashLoopBackOff, OOMKilled, Pending trong < 3 phút
✅ Viết YAML Deployment đầy đủ (probe, resource, affinity) từ memory
✅ Thiết kế microservices system trên K8s với trade-off rõ ràng

Kinh nghiệm:
✅ Có ≥ 2 câu chuyện sự cố STAR với số liệu cụ thể
✅ Có thể kể về 1 quyết định kiến trúc khó và tại sao chọn nó
✅ Biết ít nhất 1 cloud platform (EKS/GKE/AKS) đủ để thảo luận

Soft skills:
✅ Hỏi clarifying questions trước khi thiết kế hệ thống
✅ Nói "tôi không chắc nhưng tôi sẽ tiếp cận theo hướng X" thay vì im lặng
✅ Giải thích trade-off thay vì nói "cái này tốt hơn cái kia"
```

---

**Chúc bạn học tốt và phỏng vấn thành công!**

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
