# ArgoCD — GitOps Continuous Delivery cho Kubernetes

> ArgoCD là công cụ GitOps continuous delivery (phân phối liên tục) mã nguồn mở, chạy trong Kubernetes cluster. ArgoCD tự động đồng bộ trạng thái cluster với trạng thái mong muốn được khai báo trong Git repository.

## Mục Lục

1. [Kiến Trúc ArgoCD](#kiến-trúc-argocd)
2. [Cài Đặt ArgoCD](#cài-đặt-argocd)
3. [Application CRD](#application-crd)
4. [Sync Policy — Chính Sách Đồng Bộ](#sync-policy--chính-sách-đồng-bộ)
5. [ApplicationSet — Quản Lý Nhiều Application](#applicationset--quản-lý-nhiều-application)
6. [App of Apps Pattern](#app-of-apps-pattern)
7. [RBAC Trong ArgoCD](#rbac-trong-argocd)
8. [Notifications — Thông Báo](#notifications--thông-báo)
9. [Multi-Cluster Management](#multi-cluster-management)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc ArgoCD

```
┌──────────────────────────────────────────────────────────────────────┐
│                        ArgoCD Architecture                           │
│                                                                      │
│  Git Repository                                                      │
│  (GitHub/GitLab/Bitbucket)                                           │
│         │                                                            │
│         │ poll / webhook                                             │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  ArgoCD Namespace                            │   │
│  │                                                              │   │
│  │  ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐  │   │
│  │  │  argocd-    │   │    Repo      │   │  Application     │  │   │
│  │  │  server     │   │   Server     │   │  Controller      │  │   │
│  │  │             │   │              │   │                  │  │   │
│  │  │ REST API    │   │ Fetch & render│  │ Reconcile loop   │  │   │
│  │  │ Web UI      │   │ Git manifest  │  │ Compare desired  │  │   │
│  │  │ gRPC (CLI)  │   │ Helm / Kust. │   │ vs live state    │  │   │
│  │  └──────┬──────┘   └──────┬───────┘   └────────┬─────────┘  │   │
│  │         │                 │                    │             │   │
│  │         └─────────────────┴────────────────────┘             │   │
│  │                           │                                  │   │
│  │              ┌────────────┴────────────┐                     │   │
│  │              │         Redis           │                     │   │
│  │              │  (cache manifest/state) │                     │   │
│  │              └────────────────────────┘                     │   │
│  │                                                              │   │
│  │  ┌──────────────┐                                           │   │
│  │  │     Dex      │  ← OIDC (OpenID Connect) identity provider│   │
│  │  │ (SSO bridge) │  ← GitHub, Google, LDAP, SAML...         │   │
│  │  └──────────────┘                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│                    ▼ sync resource                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              Target Kubernetes Cluster                       │   │
│  │  (có thể là cùng cluster ArgoCD đang chạy hoặc remote)      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Các Thành Phần Chính

| Thành Phần | Vai Trò |
|---|---|
| **argocd-server** | API server (REST + gRPC), Web UI, xác thực |
| **argocd-repo-server** | Clone Git repo, render manifest (Helm, Kustomize, plain YAML) |
| **argocd-application-controller** | Reconciliation loop — so sánh desired vs live state, thực hiện sync |
| **argocd-dex-server** | OIDC provider tích hợp cho SSO (Single Sign-On) |
| **argocd-redis** | Cache manifest đã render, giảm tải repo-server |
| **argocd-notifications** | (optional) Gửi thông báo khi sync thay đổi |

---

## Cài Đặt ArgoCD

```bash
# Tạo namespace
kubectl create namespace argocd

# Cài đặt bằng manifest chính thức
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Hoặc dùng Helm (khuyến nghị cho production)
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  -n argocd --create-namespace \
  -f argocd-values.yaml

# Lấy password admin ban đầu
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward UI để truy cập local
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Truy cập: https://localhost:8080
# User: admin  |  Password: (lấy từ lệnh trên)

# Đăng nhập CLI
argocd login localhost:8080 --username admin --password <password>
```

---

## Application CRD

**Application** là Custom Resource (tài nguyên tùy chỉnh) cốt lõi của ArgoCD. Mỗi Application định nghĩa: lấy manifest từ đâu (source — nguồn) và deploy vào đâu (destination — đích).

### Application Cơ Bản — Helm Chart

```yaml
# application-my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd           # Application luôn thuộc namespace argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # xóa K8s resource khi xóa Application
spec:
  project: production         # ArgoCD Project để phân quyền

  source:
    repoURL: https://github.com/my-org/my-app-config
    targetRevision: HEAD       # branch, tag hoặc commit SHA
    path: helm/my-app          # thư mục trong repo
    helm:
      valueFiles:
        - values-production.yaml
      parameters:              # override giá trị cụ thể (như --set)
        - name: image.tag
          value: "abc123def"

  destination:
    server: https://kubernetes.default.svc   # cluster hiện tại
    namespace: production

  syncPolicy:
    automated:
      prune: true              # xóa resource không còn trong Git
      selfHeal: true           # tự phục hồi nếu cluster bị sửa thủ công
    syncOptions:
      - CreateNamespace=true   # tự tạo namespace nếu chưa có
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5                 # thử lại tối đa 5 lần nếu sync thất bại
      backoff:
        duration: 5s
        factor: 2              # exponential backoff: 5s, 10s, 20s, 40s, 80s
        maxDuration: 3m
```

### Application — Kustomize

```yaml
spec:
  source:
    repoURL: https://github.com/my-org/my-app-config
    targetRevision: HEAD
    path: kustomize/overlays/production
    kustomize:
      images:
        - my-app=my-registry.io/my-app:v1.2.3   # override image tag
      namePrefix: prod-                           # thêm prefix vào tên resource
      nameSuffix: ""
      commonLabels:
        environment: production
```

### Application — Plain YAML (Directory)

```yaml
spec:
  source:
    repoURL: https://github.com/my-org/my-app-config
    targetRevision: main
    path: manifests/production
    directory:
      recurse: true            # đệ quy vào thư mục con
      jsonnet: {}              # hỗ trợ Jsonnet nếu cần
```

### Application Health Status (Trạng Thái Sức Khoẻ)

```
ArgoCD Application có hai dimension (chiều) trạng thái:

SYNC STATUS (Trạng Thái Đồng Bộ):
├── Synced    — cluster khớp với Git
├── OutOfSync — cluster khác với Git (có drift hoặc chưa apply)
└── Unknown   — không thể xác định

HEALTH STATUS (Trạng Thái Sức Khoẻ):
├── Healthy     — tất cả resource healthy
├── Progressing — đang trong quá trình rollout
├── Degraded    — có resource unhealthy
├── Suspended   — pause (tạm dừng) theo yêu cầu
├── Missing     — resource không tồn tại trong cluster
└── Unknown     — không thể xác định health
```

---

## Sync Policy — Chính Sách Đồng Bộ

### So Sánh Manual vs Automated Sync

```yaml
# MANUAL SYNC — Production safest practice
syncPolicy: {}                 # không set syncPolicy = manual

# Dùng khi:
# - Production environment cần human approval trước khi deploy
# - Change phức tạp cần review trước
# - Compliance yêu cầu 4-eyes principle (hai người cùng xem)

# CLI manual sync:
argocd app sync my-app-production
argocd app sync my-app-production --revision v1.2.3  # sync một revision cụ thể
argocd app sync my-app-production --dry-run          # xem diff mà không apply
```

```yaml
# AUTOMATED SYNC — Development / Staging
syncPolicy:
  automated:
    prune: true      # XÓA resource trong cluster nếu bị xóa khỏi Git
    selfHeal: true   # KHÔI PHỤC nếu cluster bị sửa ngoài Git
```

### Prune (Dọn Dẹp)

```yaml
# Prune: khi resource bị XÓA khỏi Git manifest, ArgoCD sẽ xóa nó khỏi cluster
# Đây là hành vi nguy hiểm nếu vô tình xóa file manifest → xóa production resource
# Khuyến nghị: chỉ bật prune ở dev/staging; production xem xét cẩn thận

syncPolicy:
  automated:
    prune: true               # bật auto prune
  syncOptions:
    - PrunePropagationPolicy=foreground   # chờ child resource xóa xong mới xóa parent
    - PruneLast=true          # prune sau khi apply tất cả resource khác (an toàn hơn)
```

### Sync Waves (Sóng Đồng Bộ)

```yaml
# Sync waves cho phép kiểm soát THỨ TỰ apply resource trong cùng một sync
# Wave nhỏ hơn chạy trước, wave lớn hơn chạy sau
# ArgoCD chờ wave n healthy trước khi bắt đầu wave n+1

# Ví dụ: Database (wave 0) → Migration Job (wave 1) → App (wave 2)

# namespace.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-2"   # tạo namespace trước tiên

# database.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # cài DB trước app

# migration-job.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"    # migrate DB trước khi chạy app

# deployment.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"    # app chạy sau khi migration xong
```

---

## ApplicationSet — Quản Lý Nhiều Application

**ApplicationSet** là CRD của ArgoCD tự động sinh ra nhiều Application từ một template và generator (bộ tạo). Thay vì viết tay 10 Application YAML gần giống nhau, viết một ApplicationSet là đủ.

### List Generator (Bộ Tạo Danh Sách)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-environments
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - environment: dev
            namespace: development
            values_file: values-dev.yaml
            cluster: https://dev-cluster.example.com
          - environment: staging
            namespace: staging
            values_file: values-staging.yaml
            cluster: https://staging-cluster.example.com
          - environment: prod
            namespace: production
            values_file: values-prod.yaml
            cluster: https://prod-cluster.example.com

  template:
    metadata:
      name: "my-app-{{environment}}"          # tên Application sinh ra
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/my-app-config
        targetRevision: HEAD
        path: helm/my-app
        helm:
          valueFiles:
            - "{{values_file}}"
      destination:
        server: "{{cluster}}"
        namespace: "{{namespace}}"
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
```

### Git Directory Generator (Tự Động Từ Thư Mục Git)

```yaml
# Tự động tạo Application cho mỗi thư mục trong apps/
# Khi thêm apps/new-service/ → ApplicationSet tự tạo Application mới
spec:
  generators:
    - git:
        repoURL: https://github.com/my-org/my-app-config
        revision: HEAD
        directories:
          - path: apps/*           # mỗi thư mục trong apps/ là một app

  template:
    metadata:
      name: "{{path.basename}}"   # tên thư mục = tên Application
    spec:
      source:
        path: "{{path}}"          # path đến thư mục
        repoURL: https://github.com/my-org/my-app-config
        targetRevision: HEAD
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
```

### Pull Request Generator (Tạo App Cho Mỗi PR)

```yaml
# Tự động tạo preview environment cho mỗi Pull Request
spec:
  generators:
    - pullRequest:
        github:
          owner: my-org
          repo: my-app
          tokenRef:
            secretName: github-token
            key: token
          labels:
            - preview              # chỉ PR có label "preview" mới tạo env

  template:
    metadata:
      name: "preview-pr-{{number}}"
    spec:
      source:
        repoURL: https://github.com/my-org/my-app-config
        targetRevision: HEAD
        path: helm/my-app
        helm:
          parameters:
            - name: image.tag
              value: "pr-{{number}}"
      destination:
        namespace: "preview-pr-{{number}}"
```

---

## App of Apps Pattern

**App of Apps** là pattern nơi một root Application quản lý tập hợp các Application khác. Khi deploy root app, nó tự động tạo tất cả app con.

```
Git Repository Structure (Cấu Trúc Git):
apps/
├── root-app.yaml          ← Application trỏ đến thư mục apps/
├── app-a.yaml             ← Application cho service A
├── app-b.yaml             ← Application cho service B
└── infrastructure/
    ├── ingress-nginx.yaml
    ├── cert-manager.yaml
    └── prometheus-stack.yaml
```

```yaml
# root-app.yaml — App trỏ đến thư mục chứa các App khác
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-cluster-config
    targetRevision: HEAD
    path: apps              # thư mục chứa tất cả Application YAML
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd       # Application resource tạo trong argocd namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# apps/app-a.yaml — Application con
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/app-a-config
    targetRevision: HEAD
    path: helm/app-a
    helm:
      valueFiles: [values-prod.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: app-a
```

---

## RBAC Trong ArgoCD

ArgoCD có hệ thống RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò) riêng, hoạt động song song với Kubernetes RBAC.

### ArgoCD Projects (Dự Án)

```yaml
# AppProject phân chia và giới hạn quyền theo team/product
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  description: "Project cho team backend"

  # Cho phép deploy từ repo nào
  sourceRepos:
    - "https://github.com/my-org/backend-*"
    - "https://github.com/my-org/shared-config"

  # Cho phép deploy lên cluster và namespace nào
  destinations:
    - namespace: backend-*     # wildcard — backend-dev, backend-prod...
      server: https://kubernetes.default.svc

  # Cấm tạo ClusterRole, ClusterRoleBinding (quá nhiều quyền)
  clusterResourceBlacklist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

  # Cho phép tạo resource loại nào trong namespace
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
```

### RBAC Policy (Chính Sách RBAC)

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly    # mặc định: chỉ đọc

  policy.csv: |
    # Role định nghĩa quyền
    p, role:developer, applications, sync, team-backend/*, allow
    p, role:developer, applications, get, team-backend/*, allow
    p, role:developer, logs, get, team-backend/*, allow

    p, role:devops-lead, applications, *, */*, allow
    p, role:devops-lead, clusters, get, *, allow

    # Gán role cho group (từ OIDC/LDAP)
    g, github-org:backend-team, role:developer
    g, github-org:devops-team, role:devops-lead
    g, admin@company.com, role:admin
```

---

## Notifications — Thông Báo

```yaml
# argocd-notifications-cm ConfigMap
# Gửi Slack khi Application sync fail hoặc thành công
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-sync-succeeded: |
    message: |
      Application {{.app.metadata.name}} đã sync thành công!
      Revision: {{.app.status.sync.revision}}
      Environment: {{.app.metadata.labels.environment}}
  template.app-sync-failed: |
    message: |
      ⚠️ Application {{.app.metadata.name}} sync THẤT BẠI!
      Error: {{.app.status.conditions[0].message}}
  trigger.on-sync-succeeded: |
    - when: app.status.sync.status == 'Synced'
      send: [app-sync-succeeded]
  trigger.on-sync-failed: |
    - when: app.status.sync.status == 'Unknown' or app.status.conditions[0].type == 'SyncError'
      send: [app-sync-failed]

# Annotation trên Application để subscribe notification:
# notifications.argoproj.io/subscribe.on-sync-succeeded.slack: general
# notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts
```

---

## Multi-Cluster Management

```bash
# Register (đăng ký) cluster ngoài vào ArgoCD
argocd cluster add <kubectl-context-name>
# Ví dụ:
argocd cluster add production-eks --name production-eks

# Xem danh sách cluster đã đăng ký
argocd cluster list

# Application deploy lên cluster ngoài
spec:
  destination:
    server: https://PROD-CLUSTER-ENDPOINT.eks.amazonaws.com
    namespace: production
```

---

## Câu Hỏi Phỏng Vấn

**Giải thích sự khác biệt giữa Sync Status và Health Status trong ArgoCD?**

> Đây là hai chiều độc lập nhau. **Sync Status** nói về sự đồng bộ giữa Git và cluster: `Synced` khi manifest trong cluster khớp với Git, `OutOfSync` khi có khác biệt (chưa apply thay đổi mới trong Git, hoặc cluster bị sửa thủ công). **Health Status** nói về trạng thái hoạt động của resource: `Healthy` khi tất cả Pod đang chạy và sẵn sàng, `Progressing` khi deployment đang rollout, `Degraded` khi có Pod bị crash. Có thể có trạng thái `Synced + Degraded` — cluster đã apply đúng manifest từ Git nhưng deployment bị crash (bug trong code mới). Hoặc `OutOfSync + Healthy` — đang chạy version cũ ổn định nhưng Git đã có version mới chưa apply.

**Tại sao ApplicationSet tốt hơn copy-paste Application YAML?**

> Copy-paste Application tạo ra DRY violation (lặp lại code) và dễ bị out-of-sync khi thay đổi template chung. Nếu cần thêm annotation mới cho compliance, phải sửa 20 Application YAML riêng lẻ. ApplicationSet với List Generator giải quyết bằng cách định nghĩa template một lần, thay đổi một chỗ áp dụng tất cả. Git Directory Generator còn tốt hơn: khi developer tạo thư mục mới trong `apps/` để deploy service mới, ApplicationSet tự động phát hiện và tạo Application — không cần DevOps can thiệp. Điều này thực hiện "paved road" (con đường đã được mở sẵn) — developer tự deploy mà không cần ticket cho DevOps.

**ArgoCD selfHeal và prune có rủi ro gì ở production?**

> Cả hai đều có thể gây mất dữ liệu nếu không cẩn thận. `selfHeal: true` nghĩa là bất kỳ `kubectl edit` hay `kubectl patch` thủ công nào cũng bị overwrite sau vài phút — debug tạm thời bằng cách tăng replica hay thay đổi config sẽ bị reset. Cần tắt selfHeal hoặc suspend sync khi cần troubleshoot trực tiếp. `prune: true` nghĩa là xóa file YAML khỏi Git = xóa resource khỏi production cluster — một `git rm` vô tình có thể xóa production database. Best practice: production dùng manual sync hoặc ít nhất bật confirmations cho prune. Staging và dev có thể dùng automated + selfHeal + prune.
