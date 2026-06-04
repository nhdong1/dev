# Kubernetes (K8s) Knowledge Base — Chỉ Mục Toàn Bộ Tài Liệu

> Chỉ mục tổng thể cho tất cả tài liệu Kubernetes, bao gồm trạng thái tạo file, thứ tự học và ước lượng thời gian.

## Cấu Trúc Thư Mục

```
Devops/k8s/
├── README.md                               [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                Chỉ mục đầy đủ (file này)
│
├── 01-architecture/
│   ├── README.md                           Tổng quan kiến trúc Kubernetes
│   ├── 1-control-plane.md                    API Server, etcd, Scheduler, Controller Manager
│   ├── 2-worker-node.md                      kubelet, kube-proxy, container runtime
│   ├── 3-kubernetes-objects.md               Pod, Deployment, Service, Namespace và các object khác
│   └── 4-request-flow.md                     Luồng xử lý từ kubectl apply đến Pod chạy
│
├── 02-workload/
│   ├── README.md                           Tổng quan quản lý workload
│   ├── 1-pod.md                              Pod lifecycle, multi-container, init container
│   ├── 2-deployment.md                       Rolling update, rollback, strategy
│   ├── 3-statefulset.md                      Workload có trạng thái, headless service
│   ├── 4-daemonset.md                        Chạy agent trên mọi node
│   ├── 5-job-cronjob.md                      Tác vụ một lần và định kỳ
│   └── 6-health-probes.md                    Liveness, Readiness, Startup Probe
│
├── 03-networking/
│   ├── README.md                           Tổng quan mạng Kubernetes
│   ├── 1-service-types.md                    ClusterIP, NodePort, LoadBalancer, ExternalName
│   ├── 2-ingress.md                          Ingress Controller, TLS, routing rules
│   ├── 3-network-policy.md                   Kiểm soát lưu lượng vào/ra giữa các Pod
│   ├── 4-dns-coredns.md                      DNS nội bộ cluster, service discovery
│   └── 5-cni-plugins.md                      Calico, Flannel, Cilium — so sánh và lựa chọn
│
├── 04-storage/
│   ├── README.md                           Tổng quan lưu trữ Kubernetes
│   ├── volumes.md                          emptyDir, hostPath, configMap volume, secret volume
│   ├── persistent-volume.md                PV, PVC, StorageClass, dynamic provisioning
│   ├── access-modes.md                     ReadWriteOnce, ReadOnlyMany, ReadWriteMany
│   ├── csi-drivers.md                      Container Storage Interface, phổ biến trên cloud
│   └── volume-backup.md                    Snapshot, backup và khôi phục dữ liệu
│
├── 05-config-secret/
│   ├── README.md                           Quản lý cấu hình và bí mật
│   ├── configmap.md                        Tạo, mount, cập nhật ConfigMap
│   ├── secret.md                           Tạo, mount Secret, mã hoá tại rest
│   ├── external-secrets.md                 External Secrets Operator, AWS Secrets Manager, Vault
│   └── sealed-secrets.md                   Mã hoá Secret lưu an toàn trên Git
│
├── 06-security/
│   ├── README.md                           Bảo mật cluster Kubernetes
│   ├── rbac.md                             Role, ClusterRole, RoleBinding, ClusterRoleBinding
│   ├── service-account.md                  ServiceAccount, token, IRSA trên EKS
│   ├── pod-security.md                     Pod Security Admission (PSA), runAsNonRoot, securityContext
│   ├── network-security.md                 NetworkPolicy, mTLS, Ingress TLS
│   ├── image-security.md                   Image scanning, policy với OPA Gatekeeper
│   └── secrets-encryption.md              Mã hoá etcd tại rest, quản lý khoá
│
├── 07-scaling/
│   ├── README.md                           Tổng quan tự động mở rộng
│   ├── hpa.md                              Horizontal Pod Autoscaler — mở rộng theo CPU/memory/custom
│   ├── vpa.md                              Vertical Pod Autoscaler — điều chỉnh resource request
│   ├── cluster-autoscaler.md               Tự động thêm/bớt node trong cluster
│   ├── keda.md                             KEDA — mở rộng dựa trên sự kiện (Kafka, SQS, cron...)
│   └── resource-management.md             Resource Request, Limit, Quota, LimitRange
│
├── 08-monitoring/
│   ├── README.md                           Giám sát và observability cho K8s
│   ├── prometheus-setup.md                 Cài đặt Prometheus, scrape config, ServiceMonitor
│   ├── grafana-dashboards.md               Dashboard K8s, cảnh báo, alert rule
│   ├── loki-logging.md                     Tổng hợp log với Loki và Promtail
│   ├── tracing.md                          Distributed tracing với Jaeger hoặc Tempo
│   ├── kube-state-metrics.md               Metric trạng thái object K8s
│   └── slo-alerting.md                     Định nghĩa SLO, SLI, cấu hình Alert Manager
│
├── 09-cicd-gitops/
│   ├── README.md                           CI/CD và GitOps trên Kubernetes
│   ├── helm-basics.md                      Helm chart, repository, install, upgrade, rollback
│   ├── helm-advanced.md                    Template, hook, test, chart dependency
│   ├── argocd.md                           GitOps với ArgoCD — ApplicationSet, sync policy
│   ├── flux.md                             GitOps với Flux — Kustomization, HelmRelease
│   ├── github-actions-k8s.md               Pipeline CI/CD với GitHub Actions deploy lên K8s
│   └── progressive-delivery.md             Canary deployment, Blue-Green với Argo Rollouts
│
├── 10-troubleshooting/
│   ├── README.md                           Hướng dẫn xử lý sự cố Kubernetes
│   ├── pod-errors.md                       CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled
│   ├── node-issues.md                      NotReady node, taint, drain, cordon
│   ├── networking-debug.md                 Service không kết nối, DNS fail, Ingress lỗi
│   ├── storage-issues.md                   PVC Pending, mount error, disk full
│   ├── performance-issues.md               Throttling, high latency, resource starvation
│   ├── incident-playbook.md                Runbook xử lý sự cố theo từng tình huống
│   └── production-checklist.md            Checklist trước khi đưa lên production
│
├── 11-cloud-platforms/
│   ├── README.md                           So sánh EKS, GKE, AKS và self-managed
│   ├── eks.md                              Amazon EKS — IRSA, Fargate, add-on, node group
│   ├── gke.md                              Google GKE — Autopilot, Workload Identity, GCS
│   ├── aks.md                              Azure AKS — Azure AD, node pool, Windows container
│   └── self-managed.md                     kubeadm, k3s, RKE2 — cài đặt và vận hành thủ công
│
└── 12-interview-prep/
    ├── README.md                            Tổng quan chuẩn bị phỏng vấn K8s
    ├── INTERVIEW_GUIDE.md                   Top 30 câu hỏi phỏng vấn K8s kèm đáp án
    ├── system-design.md                     Thiết kế hệ thống với Kubernetes
    ├── star-stories.md                      Câu chuyện sự cố thực tế theo phương pháp STAR
    ├── hands-on-scenarios.md               Bài tập thực hành tình huống thực chiến
    └── 90-day-study-plan.md               Kế hoạch học 90 ngày có lịch học chi tiết
```

---

## Trạng Thái Tạo Tài Liệu

| Chủ Đề                             | Thư Mục / File                   | Trạng Thái | Chất Lượng    |
| ---------------------------------- | -------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**           | README.md                        | ✅          | Toàn diện     |
| **Chỉ Mục Tài Liệu**              | INDEX.md                         | ✅          | Toàn diện     |
| **Kiến Trúc Kubernetes**           | 01-architecture/                 | ✅          | Toàn diện     |
| **Quản Lý Workload**               | 02-workload/                     | ✅          | Toàn diện     |
| **Networking**                     | 03-networking/                   | ✅          | Toàn diện     |
| **Storage**                        | 04-storage/                      | ✅          | Toàn diện     |
| **Config & Secret**                | 05-config-secret/                | ✅          | Toàn diện     |
| **Bảo Mật**                        | 06-security/                     | ✅          | Toàn diện     |
| **Auto Scaling**                   | 07-scaling/                      | ✅          | Toàn diện     |
| **Monitoring**                     | 08-monitoring/                   | ✅          | Toàn diện     |
| **CI/CD & GitOps**                 | 09-cicd-gitops/                  | ✅          | Toàn diện     |
| **Troubleshooting**                | 10-troubleshooting/              | ✅          | Toàn diện     |
| **Cloud Platforms**                | 11-cloud-platforms/              | ✅          | Toàn diện     |
| **Phỏng Vấn**                      | 12-interview-prep/               | ✅          | Toàn diện     |

---

## Thứ Tự Ưu Tiên Tạo Tài Liệu

### Ưu Tiên Cao (Kỹ năng cốt lõi — tạo trước)

- [x] `01-architecture/README.md` — Kiến trúc K8s, Control Plane, Worker Node
- [x] `02-workload/README.md` — Pod, Deployment, StatefulSet
- [x] `03-networking/README.md` — Service, Ingress, NetworkPolicy
- [x] `10-troubleshooting/README.md` — Debug Pod, Node, Network
- [x] `12-interview-prep/INTERVIEW_GUIDE.md` — Top 30 câu hỏi phỏng vấn

### Ưu Tiên Trung Bình (Kỹ năng vận hành)

- [x] `06-security/README.md` — RBAC, Pod Security, mTLS
- [x] `07-scaling/README.md` — HPA, VPA, Cluster Autoscaler, KEDA
- [x] `08-monitoring/README.md` — Prometheus, Grafana, Loki
- [x] `09-cicd-gitops/README.md` — Helm, ArgoCD, Flux
- [x] `04-storage/README.md` — PV, PVC, StorageClass

### Ưu Tiên Thấp (Tham khảo nâng cao)

- [x] `05-config-secret/README.md` — External Secrets, Sealed Secrets
- [x] `11-cloud-platforms/README.md` — EKS, GKE, AKS
- [x] `12-interview-prep/system-design.md` — System design với K8s
- [x] `12-interview-prep/90-day-study-plan.md` — Kế hoạch học chi tiết

---

## Cách Sử Dụng Knowledge Base Này

### Cho Việc Tự Học

```
1. Bắt đầu bằng README.md để nắm lộ trình tổng thể
2. Chọn cấp độ phù hợp (Người mới / Trung cấp / Nâng cao)
3. Học tuần tự từng section theo số thứ tự
4. Thực hành trên cluster lab sau mỗi module
5. Ghi chép lại những điểm khó và câu hỏi phát sinh
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 12-interview-prep/INTERVIEW_GUIDE.md
2. Ôn kỹ 01-architecture/ — luôn được hỏi
3. Ôn 02-workload/ và 03-networking/ — kỹ năng thực chiến
4. Ôn 06-security/ và RBAC — phỏng vấn senior hay hỏi
5. Chuẩn bị 2–3 câu chuyện sự cố theo phương pháp STAR
6. Luyện tập giải thích diagram kiến trúc
```

### Cho Công Việc Vận Hành

```
Dùng làm tài liệu tham chiếu:
- Sự cố xảy ra: Vào 10-troubleshooting/ để tìm runbook phù hợp
- Deploy ứng dụng: Theo 09-cicd-gitops/ và 02-workload/
- Cấu hình bảo mật: Xem 06-security/ checklist
- Thiết lập monitoring: Theo hướng dẫn trong 08-monitoring/
- Trước khi lên production: Kiểm tra 10-troubleshooting/production-checklist.md
```

### Cho System Design

```
1. Đọc 11-cloud-platforms/README.md để chọn nền tảng
2. Dùng 03-networking/ để thiết kế ingress và service mesh
3. Tham khảo 07-scaling/ để thiết kế auto scaling
4. Dùng 06-security/ để thiết kế bảo mật từ đầu
5. Kết hợp 08-monitoring/ để thiết kế observability stack
```

---

## Ước Lượng Thời Gian Học

| Phần Tài Liệu              | Thời Gian   | Độ Khó | Độ Ưu Tiên |
| -------------------------- | ----------- | ------ | ---------- |
| Kiến Trúc Cơ Bản           | 4–6 giờ     | ⭐      | Bắt buộc   |
| Workload Management        | 6–8 giờ     | ⭐⭐    | Bắt buộc   |
| Networking                 | 8–10 giờ    | ⭐⭐⭐  | Bắt buộc   |
| Storage                    | 4–6 giờ     | ⭐⭐    | Bắt buộc   |
| Config & Secret            | 3–4 giờ     | ⭐      | Bắt buộc   |
| Bảo Mật & RBAC             | 8–10 giờ    | ⭐⭐⭐  | Bắt buộc   |
| Auto Scaling               | 4–6 giờ     | ⭐⭐    | Nên có     |
| Monitoring & Observability | 6–8 giờ     | ⭐⭐    | Nên có     |
| CI/CD & GitOps             | 8–10 giờ    | ⭐⭐⭐  | Nên có     |
| Troubleshooting            | 6–8 giờ     | ⭐⭐    | Nên có     |
| Cloud Platforms            | 8–12 giờ    | ⭐⭐⭐  | Nên có     |
| Phỏng Vấn                  | 6–8 giờ     | ⭐⭐    | Trước PV   |

**Tổng cộng: 70–100 giờ để nắm vững Kubernetes từ cơ bản đến nâng cao**

---

## Cấp Độ Kỹ Năng Được Hỗ Trợ

### Người Mới (0–1 năm kinh nghiệm)

- [ ] Kiến trúc Control Plane và Worker Node
- [ ] Pod, Deployment, Service cơ bản
- [ ] kubectl thao tác cơ bản
- [ ] ConfigMap và Secret
- [ ] Cài ứng dụng đơn giản bằng Helm

**Thời gian để làm chủ:** 2–3 tháng

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Rolling update, canary, blue-green
- [ ] HPA, VPA và resource management
- [ ] RBAC và ServiceAccount
- [ ] Ingress và NetworkPolicy
- [ ] ArgoCD / Flux GitOps workflow
- [ ] Prometheus và Grafana

**Thời gian để làm chủ:** 2–3 tháng để nâng sâu

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Multi-cluster và federation
- [ ] Custom Operator và CRD
- [ ] Service Mesh (Istio / Linkerd)
- [ ] Bảo mật nâng cao (OPA Gatekeeper, CIS Benchmark)
- [ ] Capacity planning và cost optimization
- [ ] Incident command và post-mortem

**Thời gian để làm chủ:** Học liên tục

---

## Điều Hướng Nhanh

| Nhu Cầu                             | Vị Trí Tài Liệu                                                           |
| ----------------------------------- | ------------------------------------------------------------------------- |
| Tổng quan nhanh                     | [README.md](README.md)                                                    |
| Kiến trúc K8s                       | [01-architecture/README.md](01-architecture/README.md)                    |
| Triển khai ứng dụng                 | [02-workload/README.md](02-workload/README.md)                            |
| Cấu hình Service & Ingress          | [03-networking/README.md](03-networking/README.md)                        |
| Quản lý lưu trữ                     | [04-storage/README.md](04-storage/README.md)                              |
| Quản lý Secret                      | [05-config-secret/README.md](05-config-secret/README.md)                  |
| RBAC và bảo mật                     | [06-security/README.md](06-security/README.md)                            |
| Auto Scaling                        | [07-scaling/README.md](07-scaling/README.md)                              |
| Thiết lập Monitoring                | [08-monitoring/README.md](08-monitoring/README.md)                        |
| CI/CD và GitOps                     | [09-cicd-gitops/README.md](09-cicd-gitops/README.md)                      |
| Xử lý sự cố                         | [10-troubleshooting/README.md](10-troubleshooting/README.md)              |
| So sánh EKS / GKE / AKS            | [11-cloud-platforms/README.md](11-cloud-platforms/README.md)              |
| Câu hỏi phỏng vấn                   | [12-interview-prep/INTERVIEW_GUIDE.md](12-interview-prep/INTERVIEW_GUIDE.md) |

---

## Theo Dõi Tiến Độ Học Tập

Sao chép đoạn dưới và cập nhật theo tiến độ của bạn:

```markdown
## Tiến Độ Học Kubernetes

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] Kiến trúc Control Plane và Worker Node
- [ ] Pod, Deployment, Service
- [ ] kubectl cơ bản
- [ ] Namespace và resource isolation
- [ ] ConfigMap và Secret

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)

- [ ] Ingress và NetworkPolicy
- [ ] PV, PVC, StorageClass
- [ ] RBAC và ServiceAccount
- [ ] HPA và resource management
- [ ] Health Probe (Liveness, Readiness)

### Giai Đoạn 3: Vận Hành (Tuần 7–10)

- [ ] Helm chart cơ bản và nâng cao
- [ ] ArgoCD hoặc Flux GitOps
- [ ] Prometheus + Grafana + Loki
- [ ] Xử lý sự cố thực tế
- [ ] StatefulSet cho database

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Service Mesh với Istio
- [ ] Custom Operator và CRD
- [ ] Multi-cluster federation
- [ ] Bảo mật nâng cao
- [ ] Mock phỏng vấn
```

---

## Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### Năng Lực Nền Tảng

- [ ] Giải thích kiến trúc K8s không cần tài liệu
- [ ] Triển khai và quản lý ứng dụng trên K8s
- [ ] Debug Pod và service bằng kubectl thuần thục
- [ ] Thiết kế RBAC cho team nhiều người
- [ ] Cấu hình Ingress với TLS termination

### Năng Lực Vận Hành

- [ ] Xử lý sự cố CrashLoopBackOff, OOMKilled hệ thống
- [ ] Cấu hình HPA đúng cho production
- [ ] Triển khai zero-downtime deployment
- [ ] Thiết lập monitoring và alerting đầy đủ
- [ ] Thực hiện migrate ứng dụng giữa namespace hoặc cluster

### Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 30 câu hỏi K8s
- [ ] Kể 2–3 câu chuyện sự cố theo STAR
- [ ] Thiết kế hệ thống có xét đến K8s
- [ ] Thảo luận trade-off giữa các lựa chọn kỹ thuật
- [ ] Hiểu sâu ít nhất một nền tảng cloud (EKS / GKE / AKS)

---

## Mẹo Thực Hành

1. **Học qua phá vỡ:** Cố tình tạo sự cố (xoá service, kill Pod) rồi phục hồi — đây là cách học nhanh nhất
2. **Đọc YAML thực tế:** Xem manifest của các open source project nổi tiếng trên GitHub
3. **Theo dõi event:** `kubectl get events --sort-by=.lastTimestamp` thường tiết lộ nguyên nhân sự cố
4. **Dùng k9s hàng ngày:** Giao diện terminal của k9s giúp bạn quen với cluster nhanh hơn kubectl thuần
5. **Ghi lại runbook:** Mỗi sự cố bạn giải quyết, hãy viết runbook — đây là portfolio thực chiến
6. **Test trên nhiều loại cluster:** Thử cả kind, minikube và một managed cluster trên cloud
7. **Đọc release notes:** K8s ra phiên bản mới mỗi 4 tháng — theo dõi thay đổi API quan trọng
8. **Thực hành kubectl explain:** Dùng `kubectl explain <resource>.<field>` để hiểu field mà không cần lên docs

---

## Đóng Góp và Cập Nhật

Phát hiện lỗi hoặc muốn bổ sung nội dung?

- [ ] Sửa nội dung không chính xác về K8s API hoặc hành vi
- [ ] Thêm ví dụ YAML manifest thực tế
- [ ] Bổ sung câu hỏi phỏng vấn từ kinh nghiệm thực tế
- [ ] Cải thiện giải thích cho khái niệm phức tạp
- [ ] Thêm hướng dẫn cho phiên bản K8s mới

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 4.0 (12-interview-prep hoàn thành — toàn bộ knowledge base hoàn tất)
**Trạng Thái:** ✅ README.md hoàn thành | ✅ INDEX.md hoàn thành | ✅ 01-architecture hoàn thành | ✅ 02-workload hoàn thành | ✅ 03-networking hoàn thành | ✅ 04-storage hoàn thành | ✅ 05-config-secret hoàn thành | ✅ 06-security hoàn thành | ✅ 07-scaling hoàn thành | ✅ 08-monitoring hoàn thành | ✅ 09-cicd-gitops hoàn thành | ✅ 10-troubleshooting hoàn thành | ✅ 11-cloud-platforms hoàn thành | ✅ 12-interview-prep hoàn thành
