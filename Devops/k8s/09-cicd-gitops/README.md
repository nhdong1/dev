# CI/CD & GitOps — Tích Hợp Liên Tục và Triển Khai Liên Tục trên Kubernetes

> Tổng quan về CI/CD (Continuous Integration/Continuous Delivery — Tích Hợp Liên Tục/Triển Khai Liên Tục) và GitOps trên Kubernetes: từ Helm chart packaging, pipeline GitHub Actions, đến GitOps với ArgoCD và Flux, kết hợp Progressive Delivery với Argo Rollouts trong môi trường production.

## Mục Lục

1. [CI/CD Truyền Thống vs GitOps](#cicd-truyền-thống-vs-gitops)
2. [Kiến Trúc Tổng Thể Pipeline](#kiến-trúc-tổng-thể-pipeline)
3. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
4. [Các Thành Phần Chính](#các-thành-phần-chính)
5. [So Sánh Công Cụ GitOps](#so-sánh-công-cụ-gitops)
6. [Ma Trận Tình Huống vs Giải Pháp](#ma-trận-tình-huống-vs-giải-pháp)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
8. [Checklist Production CI/CD](#checklist-production-cicd)

---

## CI/CD Truyền Thống vs GitOps

### CI/CD Truyền Thống (Push-Based — Đẩy Trực Tiếp)

Trong mô hình truyền thống, pipeline CI/CD **đẩy** thay đổi trực tiếp vào cluster bằng cách chạy `kubectl apply` hoặc `helm upgrade` trong pipeline. Cluster là **bị động** — chờ pipeline đến triển khai.

```
Developer
    │ git push
    ▼
Git Repository (GitHub, GitLab)
    │ trigger
    ▼
CI/CD Pipeline (GitHub Actions, Jenkins, GitLab CI)
    │ 1. Build & test code
    │ 2. Build Docker image → push to registry
    │ 3. kubectl apply / helm upgrade
    ▼
Kubernetes Cluster ← pipeline CẦN quyền write vào cluster
```

**Vấn đề với mô hình truyền thống:**
- Pipeline cần credentials (thông tin xác thực) mạnh để truy cập cluster → rủi ro bảo mật
- Cluster drift (trạng thái thực tế khác với code) nếu ai đó chạy kubectl thủ công
- Khó audit (kiểm toán) xem ai đã deploy gì, lúc nào
- Rollback phức tạp — phải chạy lại pipeline với commit cũ

### GitOps (Pull-Based — Kéo Từ Git)

GitOps là phương pháp vận hành nơi **Git là nguồn sự thật duy nhất** (single source of truth — SSOT) cho trạng thái mong muốn của hệ thống. Thay vì pipeline đẩy vào cluster, một **operator** trong cluster liên tục **kéo** từ Git và tự động đồng bộ trạng thái.

```
Developer
    │ git push / open PR
    ▼
Git Repository ← NGUỒN SỰ THẬT DUY NHẤT
    │ operator poll/webhook
    ▼
GitOps Operator (ArgoCD / Flux) — CHẠY TRONG CLUSTER
    │ 1. Phát hiện sự khác biệt (drift detection)
    │ 2. Tự động reconcile (đồng bộ) hoặc chờ approval
    ▼
Kubernetes Cluster ← operator chỉ cần quyền TRONG cluster
```

**Lợi ích của GitOps:**
- **Bảo mật hơn:** Pipeline không cần credentials vào cluster; chỉ operator nội bộ có quyền
- **Auditability (Khả Năng Kiểm Toán):** Mọi thay đổi là một Git commit — có lịch sử, có người review, có timestamp
- **Tự phục hồi (Self-Healing):** Nếu ai đó sửa trực tiếp cluster, operator sẽ phát hiện drift và khôi phục về trạng thái trong Git
- **Rollback dễ dàng:** `git revert` một commit là rollback deployment — không cần chạy lại pipeline
- **Multi-environment dễ quản lý:** Mỗi branch/folder là một environment (môi trường)

### So Sánh Hai Mô Hình

| Tiêu Chí | CI/CD Truyền Thống (Push) | GitOps (Pull) |
|---|---|---|
| **Nguồn sự thật** | Pipeline script + manual kubectl | Git repository |
| **Bảo mật** | Pipeline cần cluster credentials | Credentials ở trong cluster |
| **Drift detection** | Không có tự động | Tự động, liên tục |
| **Rollback** | Phải chạy lại pipeline | `git revert` là đủ |
| **Audit trail** | Pipeline log, ít chi tiết | Git log đầy đủ |
| **Đồng bộ môi trường** | Dễ bị out-of-sync | Luôn đồng bộ với Git |
| **Learning curve** | Thấp hơn, quen thuộc | Cần học GitOps operator |
| **Phù hợp** | Team nhỏ, hệ thống đơn giản | Production, multi-team |

---

## Kiến Trúc Tổng Thể Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GIAI ĐOẠN CI (Continuous Integration)                │
│                     Tích Hợp Liên Tục — chạy trên mỗi PR/commit         │
│                                                                         │
│  Developer  →  git push  →  GitHub/GitLab                              │
│                                  │                                      │
│                          ┌───────▼────────┐                            │
│                          │  CI Pipeline   │                            │
│                          │                │                            │
│                          │ 1. Lint & Test │  ← unit test, integration  │
│                          │ 2. SAST Scan   │  ← Snyk, Semgrep           │
│                          │ 3. Build Image │  ← docker build            │
│                          │ 4. Image Scan  │  ← Trivy, Snyk             │
│                          │ 5. Push Image  │  ← registry.example.com    │
│                          │ 6. Update tag  │  ← update values.yaml      │
│                          └───────┬────────┘                            │
└──────────────────────────────────┼──────────────────────────────────────┘
                                   │ git commit (tag mới)
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              GIAI ĐOẠN CD (Continuous Delivery / Deployment)            │
│                  Triển Khai Liên Tục — GitOps operator                  │
│                                                                         │
│  Git Repository (config repo)                                           │
│  ├── environments/                                                      │
│  │   ├── dev/      ← auto-sync, deploy ngay                            │
│  │   ├── staging/  ← auto-sync sau dev pass                            │
│  │   └── prod/     ← manual approval + progressive delivery            │
│  └── apps/                                                              │
│      ├── app-a/values-dev.yaml                                         │
│      └── app-a/values-prod.yaml                                        │
│                    │                                                    │
│                    │ poll every 3 minutes (ArgoCD/Flux)                 │
│                    ▼                                                    │
│          ┌─────────────────┐     ┌─────────────────┐                  │
│          │  DEV Cluster    │     │  PROD Cluster   │                  │
│          │                 │     │                 │                  │
│          │  ArgoCD/Flux    │     │  ArgoCD/Flux    │                  │
│          │  auto-sync ✓    │     │  manual sync ✓  │                  │
│          └─────────────────┘     └────────┬────────┘                  │
│                                           │                            │
│                                  Progressive Delivery                  │
│                                  (Argo Rollouts)                       │
│                                  Canary 10% → 50% → 100%              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Bản Đồ Quyết Định

```
Cần triển khai ứng dụng lên Kubernetes — chọn công cụ nào?
│
├── Cần đóng gói ứng dụng để tái sử dụng và chia sẻ?
│   └── → Dùng HELM
│       ├── Ứng dụng đơn giản, không cần versioning → helm install trực tiếp
│       ├── Cần share với team khác → tạo Helm chart và đẩy lên chart repo
│       └── Cần cấu hình khác nhau cho dev/prod → values.yaml riêng mỗi env
│
├── Muốn áp dụng GitOps — cluster tự đồng bộ với Git?
│   ├── Đang dùng ArgoCD UI, cần visualization đẹp, team nhỏ?
│   │   └── → Dùng ARGOCD
│   │       ├── App cần manual approval trước khi sync → sync policy: manual
│   │       ├── Nhiều app tương tự cần quản lý → ApplicationSet
│   │       └── App phụ thuộc nhau → App of Apps pattern
│   │
│   └── Thích CLI-first, muốn tự động hoàn toàn, dùng nhiều Helm?
│       └── → Dùng FLUX
│           ├── Tự động update image khi có tag mới → Image Automation
│           ├── Dùng Helm chart → HelmRelease + HelmRepository
│           └── Dùng Kustomize → Kustomization resource
│
├── Cần CI pipeline build và test trước khi deploy?
│   └── → Dùng GITHUB ACTIONS (hoặc GitLab CI, Tekton)
│       ├── Đơn giản, ít infra → GitHub Actions hosted runner
│       ├── Cần runner tự quản lý trong cluster → self-hosted runner
│       └── Secret cần inject vào pipeline → GitHub Secrets + OIDC
│
└── Cần kiểm soát rủi ro khi deploy lên production?
    └── → Progressive Delivery với ARGO ROLLOUTS
        ├── Chỉ muốn giảm rủi ro dần → Canary Deployment (10% → 100%)
        ├── Muốn switch nhanh nếu lỗi → Blue-Green Deployment
        └── Cần auto-rollback dựa trên metric → Analysis Template + PromQL
```

---

## Các Thành Phần Chính

### Helm — Quản Lý Package Kubernetes

**Helm** là trình quản lý package (package manager — trình quản lý gói) cho Kubernetes, tương tự như `apt` cho Ubuntu hay `npm` cho Node.js. Helm giúp đóng gói, cấu hình và triển khai ứng dụng Kubernetes theo cách có thể tái sử dụng và versioned (quản lý phiên bản).

```
Helm Chart Structure (Cấu Trúc Helm Chart):
my-app/
├── Chart.yaml          ← metadata: tên, phiên bản, mô tả
├── values.yaml         ← giá trị mặc định cho template
├── values-prod.yaml    ← giá trị override cho production
├── templates/
│   ├── deployment.yaml ← Deployment template với {{ .Values.xxx }}
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    ← helper template, hàm dùng chung
│   └── NOTES.txt       ← hướng dẫn sau khi install
├── charts/             ← sub-chart (chart phụ thuộc)
└── .helmignore         ← file/folder bỏ qua khi đóng gói
```

Tham khảo chi tiết: [helm-basics.md](./helm-basics.md) và [helm-advanced.md](./helm-advanced.md)

---

### ArgoCD — GitOps Continuous Delivery

**ArgoCD** là công cụ GitOps continuous delivery (phân phối liên tục) mã nguồn mở, chạy trong cluster Kubernetes. ArgoCD liên tục so sánh trạng thái mong muốn (desired state — từ Git) với trạng thái thực tế (live state — từ cluster) và đồng bộ khi có sự khác biệt.

```
ArgoCD Architecture (Kiến Trúc ArgoCD):
Git Repository
      │ poll / webhook
      ▼
ArgoCD Server (API + UI)
      │
      ├── Application Controller — reconcile loop, sync logic
      ├── Repo Server — fetch và render manifest từ Git
      ├── Redis — cache manifest
      └── Dex — OIDC identity provider tích hợp
```

Tham khảo chi tiết: [argocd.md](./argocd.md)

---

### Flux — GitOps Operator

**Flux** là bộ công cụ GitOps mã nguồn mở, là CNCF (Cloud Native Computing Foundation — Tổ Chức Điện Toán Đám Mây Native) graduated project. Flux được xây dựng trên Kubernetes controller pattern và hoạt động hoàn toàn qua Custom Resource (tài nguyên tùy chỉnh).

```
Flux Controllers (Bộ Điều Khiển Flux):
├── Source Controller       → quản lý GitRepository, HelmRepository, OCIRepository
├── Kustomize Controller    → reconcile Kustomization resource
├── Helm Controller         → reconcile HelmRelease resource
├── Notification Controller → gửi alert khi reconcile thành công/thất bại
└── Image Automation        → tự động cập nhật tag image trong Git
```

Tham khảo chi tiết: [flux.md](./flux.md)

---

### GitHub Actions — CI Pipeline

**GitHub Actions** là nền tảng CI/CD tích hợp sẵn trong GitHub, sử dụng YAML workflow để tự động hóa build, test và deploy. Đây là lựa chọn phổ biến nhất hiện nay vì zero-setup với code đã trên GitHub.

```
GitHub Actions Workflow (Luồng Công Việc):
.github/workflows/
├── ci.yaml         ← chạy khi có PR: lint, test, build, scan
├── cd-dev.yaml     ← chạy khi merge vào main: deploy lên dev
└── cd-prod.yaml    ← chạy khi tag release: deploy lên prod
```

Tham khảo chi tiết: [github-actions-k8s.md](./github-actions-k8s.md)

---

### Progressive Delivery — Triển Khai Dần Dần

**Progressive Delivery (Triển Khai Dần Dần)** là phương pháp giảm rủi ro khi deploy bằng cách đưa version mới ra dần dần thay vì tất cả cùng lúc. Công cụ phổ biến nhất trong K8s ecosystem là **Argo Rollouts**.

```
Canary Deployment (Triển Khai Canary):
Version cũ: ████████████████████ 100% traffic
      │
      │ deploy version mới
      ▼
Giai đoạn 1: ████████████████░░░░ 80% cũ | 20% mới
      │ metric OK?
      ▼
Giai đoạn 2: ████████░░░░░░░░░░░░ 40% cũ | 60% mới
      │ metric OK?
      ▼
Giai đoạn 3: ░░░░░░░░░░░░░░░░░░░░   0% cũ | 100% mới ✓
      │ metric FAIL?
      └──────────────────── auto rollback về version cũ ✗
```

Tham khảo chi tiết: [progressive-delivery.md](./progressive-delivery.md)

---

## So Sánh Công Cụ GitOps

| Tiêu Chí | ArgoCD | Flux v2 |
|---|---|---|
| **Giao diện** | Web UI đẹp, trực quan | CLI-first, không có UI mặc định |
| **Cài đặt** | Một namespace `argocd` | Nhiều controller namespace `flux-system` |
| **Cơ chế** | Application CRD | GitRepository + Kustomization/HelmRelease CRD |
| **Helm support** | Tốt (Helm chart source) | Rất tốt (HelmRelease CRD riêng) |
| **Kustomize** | Hỗ trợ tốt | Native, mạnh hơn |
| **Image Automation** | Argocd-image-updater (riêng) | Tích hợp sẵn Image Automation |
| **RBAC** | ArgoCD RBAC riêng | Dùng Kubernetes RBAC |
| **Multi-cluster** | Tốt (register cluster) | Tốt (multi-tenant) |
| **Notification** | Tích hợp | Notification Controller riêng |
| **CNCF Status** | Graduated | Graduated |
| **Phù hợp** | Team cần UI, mới bắt đầu GitOps | Team K8s-native, thích GitOps thuần túy |

> **Lựa chọn thực tế:** Nhiều tổ chức dùng ArgoCD vì UI trực quan giúp team dễ onboard. Flux phổ biến trong môi trường platform engineering (kỹ thuật nền tảng) nơi tất cả cấu hình là code. Cả hai đều là lựa chọn tốt — quyết định dựa trên sở thích team và yêu cầu cụ thể.

---

## Ma Trận Tình Huống vs Giải Pháp

| Tình Huống | Dấu Hiệu | Công Cụ | Giải Pháp | File |
|---|---|---|---|---|
| Đóng gói app K8s để share | Nhiều manifest YAML lặp lại | Helm | Tạo Helm chart với values.yaml | [helm-basics.md](./helm-basics.md) |
| Cần template phức tạp trong chart | Logic điều kiện, vòng lặp | Helm Advanced | Go template, helper function | [helm-advanced.md](./helm-advanced.md) |
| Cluster drift — ai đó sửa thủ công | `kubectl get` khác với Git | ArgoCD / Flux | GitOps với auto-sync + drift detection | [argocd.md](./argocd.md) |
| Deploy nhiều app tương tự | Copy-paste ApplicationSet | ArgoCD ApplicationSet | ApplicationSet với generator | [argocd.md](./argocd.md) |
| Tự động update image khi build mới | Phải sửa values.yaml thủ công | Flux Image Automation | ImagePolicy + ImageUpdateAutomation | [flux.md](./flux.md) |
| CI pipeline build và test code | Cần test trước khi deploy | GitHub Actions | CI workflow với test + scan | [github-actions-k8s.md](./github-actions-k8s.md) |
| Deploy prod quá rủi ro | Sợ bug ảnh hưởng toàn bộ user | Argo Rollouts | Canary deployment với analysis | [progressive-delivery.md](./progressive-delivery.md) |
| Rollback nhanh khi lỗi | Deployment mới gây tăng error rate | Argo Rollouts | Auto-rollback từ AnalysisTemplate | [progressive-delivery.md](./progressive-delivery.md) |
| Deploy zero-downtime | Restart gây downtime ngắn | Helm + Rolling Update / Argo Rollouts | Blue-Green deployment | [progressive-delivery.md](./progressive-delivery.md) |
| Secret trong pipeline CI | Credentials hardcode trong YAML | GitHub Actions OIDC | OIDC federation, no long-lived secrets | [github-actions-k8s.md](./github-actions-k8s.md) |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**GitOps là gì và tại sao lại ưu việt hơn CI/CD truyền thống?**

> **GitOps** là phương pháp vận hành sử dụng Git làm nguồn sự thật duy nhất (single source of truth) cho cả infrastructure (hạ tầng) và application configuration (cấu hình ứng dụng). Mọi thay đổi đều thực hiện qua Git commit và pull request, sau đó được operator trong cluster tự động đồng bộ. Ưu việt hơn CI/CD truyền thống ở ba điểm: (1) **Bảo mật:** Pipeline không cần cluster credentials vì operator chạy trong cluster tự pull về; (2) **Drift detection:** Operator liên tục kiểm tra và tự động khôi phục nếu cluster bị sửa ngoài Git; (3) **Audit trail:** Mọi thay đổi có Git history với reviewer, timestamp, và diff rõ ràng — không phải dựa vào pipeline log.

**Helm giải quyết vấn đề gì mà plain kubectl apply không làm được?**

> `kubectl apply` đơn giản áp dụng YAML manifest tĩnh. Helm giải quyết 4 vấn đề: (1) **Templating (Mẫu hóa):** Thay vì copy YAML cho dev/staging/prod, dùng một template với `values.yaml` khác nhau cho mỗi môi trường; (2) **Versioning (Quản lý phiên bản):** Helm lưu history release, cho phép rollback về bất kỳ revision nào bằng `helm rollback`; (3) **Dependency management (Quản lý phụ thuộc):** Chart có thể khai báo phụ thuộc vào chart khác (ví dụ: app phụ thuộc vào postgresql chart); (4) **Lifecycle hooks (Móc vòng đời):** Chạy Job trước/sau install/upgrade để migrate database, kiểm tra health...

**ArgoCD tự phục hồi (self-healing) hoạt động thế nào?**

> Khi `syncPolicy.automated.selfHeal: true`, ArgoCD chạy reconciliation loop liên tục (mặc định 3 phút một lần). Mỗi lần loop chạy, ArgoCD so sánh desired state (trạng thái mong muốn từ Git manifest) với live state (trạng thái thực tế trong cluster). Nếu phát hiện sự khác biệt (drift) — ví dụ ai đó chạy `kubectl scale deployment --replicas=5` trong khi Git có `replicas: 3` — ArgoCD sẽ tự động chạy sync để đưa về trạng thái trong Git. Điều này đảm bảo Git luôn là nguồn sự thật, ngay cả khi có thay đổi ngoài quy trình.

### Câu Hỏi Nâng Cao

**Giải thích sự khác biệt giữa ArgoCD App of Apps pattern và ApplicationSet?**

> Cả hai đều giải quyết vấn đề quản lý nhiều Application trong ArgoCD, nhưng theo cách khác nhau. **App of Apps** là pattern nơi một "root" Application quản lý một tập hợp các Application khác — root app point đến một thư mục chứa nhiều Application manifest. Khi deploy root app, nó tạo ra tất cả app con. Phù hợp khi số lượng app cố định, cần kiểm soát thứ tự tạo. **ApplicationSet** là CRD (Custom Resource Definition) của ArgoCD tự động sinh ra Application từ template và generator (List, Git directory, Cluster, Pull Request...). Ví dụ: một ApplicationSet với Git directory generator tự động tạo Application mới khi có folder mới trong repo — không cần sửa ApplicationSet. Phù hợp khi số lượng app thay đổi động, pattern lặp lại nhiều.

**Canary deployment trong Kubernetes thuần (không dùng Argo Rollouts) có hạn chế gì?**

> Kubernetes thuần hỗ trợ canary cơ bản bằng cách chạy hai Deployment với label khác nhau và điều chỉnh tỉ lệ replica. Ví dụ: `stable: 9 replicas` và `canary: 1 replica` = ~10% traffic đến canary. Hạn chế: (1) **Granularity thô:** Để 1% traffic phải có 99 stable + 1 canary replica — rất tốn tài nguyên; (2) **Không có metric-based analysis:** Phải tự viết script kiểm tra metric và rollback; (3) **Không có header-based routing:** Không thể route chỉ traffic có header `X-Canary: true`; (4) **Quản lý thủ công:** Phải tự increment từng bước và monitor. Argo Rollouts giải quyết tất cả hạn chế này bằng cách tích hợp với service mesh hoặc nginx ingress để kiểm soát weight chính xác, tự động analyze metric, và tự động promote/rollback.

---

## Checklist Production CI/CD

### Helm Chart

- [ ] `Chart.yaml` có version và appVersion rõ ràng, theo Semantic Versioning (quản lý phiên bản ngữ nghĩa)
- [ ] `values.yaml` có default hợp lý, có comment giải thích mỗi value quan trọng
- [ ] `helm lint` không có warning hay error
- [ ] `helm template` kiểm tra được output YAML đúng
- [ ] `helm test` đã viết test để verify deployment thành công
- [ ] Resource request và limit được define trong chart (không để trống)
- [ ] readinessProbe và livenessProbe có trong Deployment template

### CI Pipeline

- [ ] Pipeline chạy unit test và integration test trước khi build image
- [ ] Image được scan tìm vulnerability (lỗ hổng bảo mật) trước khi push (Trivy, Snyk)
- [ ] Image tag là commit SHA hoặc semantic version — không dùng `latest`
- [ ] Pipeline dùng OIDC để xác thực với cloud provider — không có long-lived credential trong secret
- [ ] Branch protection rule yêu cầu pipeline pass trước khi merge
- [ ] Secrets được lưu trong GitHub Secrets / Vault, không trong code

### ArgoCD / Flux

- [ ] Application đã cấu hình `automated.selfHeal: true` cho dev/staging
- [ ] Production dùng manual sync hoặc có approval workflow
- [ ] Health check (kiểm tra sức khoẻ) được cấu hình cho custom resource
- [ ] Notification đã cấu hình gửi đến Slack khi sync thất bại
- [ ] RBAC đã setup — developer không có permission sync prod trực tiếp
- [ ] Secrets không commit lên Git — dùng Sealed Secrets hoặc External Secrets

### Progressive Delivery

- [ ] Argo Rollouts đã cài đặt nếu dùng canary/blue-green
- [ ] AnalysisTemplate đã định nghĩa metric để tự động promote/rollback
- [ ] Rollout strategy được test kỹ trên staging trước khi áp dụng prod
- [ ] Monitoring dashboard theo dõi version distribution trong quá trình canary
- [ ] Timeout cho mỗi step canary được set (không để stuck vô thời hạn)

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
|---|---|
| [helm-basics.md](./1-helm-basics.md) | Helm chart cơ bản, repository, install, upgrade, rollback |
| [helm-advanced.md](./2-helm-advanced.md) | Go template, hook, test, dependency, OCI registry |
| [argocd.md](./3-argocd.md) | ArgoCD architecture, Application CRD, ApplicationSet, sync policy |
| [flux.md](./4-flux.md) | Flux controllers, GitRepository, Kustomization, HelmRelease, Image Automation |
| [github-actions-k8s.md](./5-github-actions-k8s.md) | GitHub Actions workflow, build/push image, deploy lên K8s, OIDC |
| [progressive-delivery.md](./6-progressive-delivery.md) | Argo Rollouts, Canary, Blue-Green, AnalysisTemplate, auto-rollback |
