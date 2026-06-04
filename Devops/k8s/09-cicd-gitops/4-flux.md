# Flux — GitOps Operator cho Kubernetes

> Flux là bộ công cụ GitOps mã nguồn mở, CNCF graduated project. Flux hoạt động hoàn toàn qua Kubernetes controller pattern và Custom Resource, cung cấp đồng bộ tự động giữa Git repository và Kubernetes cluster.

## Mục Lục

1. [Kiến Trúc Flux v2](#kiến-trúc-flux-v2)
2. [Cài Đặt Flux (Bootstrap)](#cài-đặt-flux-bootstrap)
3. [Source Controller — Quản Lý Nguồn](#source-controller--quản-lý-nguồn)
4. [Kustomization — Đồng Bộ Manifest](#kustomization--đồng-bộ-manifest)
5. [HelmRelease — Triển Khai Helm Chart](#helmrelease--triển-khai-helm-chart)
6. [Image Automation — Cập Nhật Tag Tự Động](#image-automation--cập-nhật-tag-tự-động)
7. [Notification Controller](#notification-controller)
8. [Multi-Tenancy — Nhiều Team Trên Cùng Cluster](#multi-tenancy--nhiều-team-trên-cùng-cluster)
9. [So Sánh Flux vs ArgoCD](#so-sánh-flux-vs-argocd)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Flux v2

Flux v2 (còn gọi là GitOps Toolkit — Bộ Công Cụ GitOps) bao gồm các **controller** chạy riêng biệt, mỗi controller xử lý một loại Custom Resource cụ thể.

```
┌──────────────────────────────────────────────────────────────────────┐
│                   Flux Controllers (flux-system namespace)           │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Source Controller                              │    │
│  │  Theo dõi các nguồn: Git, Helm repo, OCI, S3 Bucket        │    │
│  │  GitRepository → HelmRepository → OCIRepository → Bucket   │    │
│  └──────────────────────┬──────────────────────────────────────┘    │
│                         │ artifact (tar.gz)                         │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │    Kustomize Controller   │    Helm Controller               │   │
│  │                           │                                  │   │
│  │  Reconcile Kustomization  │  Reconcile HelmRelease           │   │
│  │  → kubectl apply          │  → helm upgrade --install        │   │
│  └───────────┬───────────────┴──────────────┬───────────────────┘   │
│              │                              │                       │
│              └──────────────┬───────────────┘                       │
│                             │ apply resources                       │
│                             ▼                                       │
│                    Kubernetes Resources                             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │         Image Automation Controller                          │   │
│  │  Theo dõi image registry → cập nhật tag trong Git           │   │
│  │  ImageRepository → ImagePolicy → ImageUpdateAutomation      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │         Notification Controller                              │   │
│  │  Gửi thông báo đến Slack, Teams, GitHub, Webhook...         │   │
│  │  Provider → Alert                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Custom Resources Của Flux

```
flux-system namespace:
├── Source resources:
│   ├── GitRepository       → clone Git repo, tạo artifact
│   ├── HelmRepository      → index Helm chart repository
│   ├── OCIRepository       → pull OCI artifact (Helm chart hoặc raw)
│   └── Bucket              → sync từ S3/GCS bucket
│
├── Reconciliation resources:
│   ├── Kustomization       → apply Kustomize overlay hoặc plain YAML
│   └── HelmRelease         → install/upgrade Helm chart
│
├── Image resources:
│   ├── ImageRepository     → quét image registry tìm tag mới
│   ├── ImagePolicy         → filter tag theo rule (semver, regex)
│   └── ImageUpdateAutomation → commit tag mới vào Git
│
└── Notification resources:
    ├── Provider            → khai báo endpoint (Slack, GitHub...)
    └── Alert               → subscribe event và route đến Provider
```

---

## Cài Đặt Flux (Bootstrap)

**Bootstrap (Khởi Tạo)** là quá trình cài đặt Flux vào cluster VÀ đưa cấu hình Flux vào Git — sau đó Flux tự quản lý chính mình.

```bash
# Cài Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Kiểm tra prerequisites (yêu cầu trước)
flux check --pre

# Bootstrap với GitHub — Flux tạo repo (hoặc push vào repo có sẵn)
# và cài toàn bộ controller vào cluster
flux bootstrap github \
  --token-auth \
  --owner=my-org \
  --repository=my-cluster-config \
  --branch=main \
  --path=clusters/production \
  --personal

# Sau bootstrap, repo sẽ có cấu trúc:
# my-cluster-config/
# └── clusters/
#     └── production/
#         └── flux-system/
#             ├── gotk-components.yaml    ← Flux controller manifests
#             ├── gotk-sync.yaml          ← GitRepository + Kustomization cho flux-system
#             └── kustomization.yaml

# Kiểm tra trạng thái sau bootstrap
flux get all
kubectl get pods -n flux-system
```

---

## Source Controller — Quản Lý Nguồn

### GitRepository — Theo Dõi Git

```yaml
# clusters/production/flux-system/sources/my-app-repo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app-config
  namespace: flux-system
spec:
  interval: 1m            # poll Git mỗi 1 phút kiểm tra thay đổi
  url: https://github.com/my-org/my-app-config
  ref:
    branch: main          # theo dõi branch main

  # Xác thực private repo bằng SSH key hoặc HTTPS token
  secretRef:
    name: my-app-config-auth

  # Chỉ trigger khi các file này thay đổi (tối ưu hiệu suất)
  ignore: |
    # Bỏ qua markdown và các file không ảnh hưởng deploy
    *.md
    docs/
    tests/
---
# Secret cho HTTPS authentication
apiVersion: v1
kind: Secret
metadata:
  name: my-app-config-auth
  namespace: flux-system
type: Opaque
stringData:
  username: "git"
  password: "ghp_xxxxxxxxxxxxxxxxxxxx"   # GitHub Personal Access Token
```

```bash
# Kiểm tra trạng thái GitRepository
flux get sources git
# NAME               REVISION         SUSPENDED  READY  MESSAGE
# my-app-config      main/abc123def   False      True   stored artifact

# Trigger reconcile ngay lập tức (không đợi interval)
flux reconcile source git my-app-config
```

### HelmRepository — Helm Chart Source

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
---
# OCI Helm Repository (chuẩn mới)
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: my-org-charts
  namespace: flux-system
spec:
  type: oci
  interval: 30m
  url: oci://ghcr.io/my-org/charts
  secretRef:
    name: ghcr-auth
```

---

## Kustomization — Đồng Bộ Manifest

**Kustomization** (không nhầm với `kubectl kustomize`) là Custom Resource của Flux, xác định nguồn manifest cần apply và cách apply.

### Kustomization Cơ Bản

```yaml
# clusters/production/apps/my-app.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m             # reconcile mỗi 5 phút
  timeout: 2m              # timeout nếu apply quá lâu
  sourceRef:
    kind: GitRepository
    name: my-app-config    # tên GitRepository đã khai báo
  path: ./kustomize/overlays/production   # thư mục trong repo
  prune: true              # xóa resource không còn trong Git
  wait: true               # chờ resource healthy sau khi apply
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: production
```

### Kustomization Với Substitution (Thay Thế Biến)

```yaml
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app-config
  path: ./manifests
  prune: true
  postBuild:
    substitute:
      # Thay thế ${ENVIRONMENT} trong manifest bằng "production"
      ENVIRONMENT: production
      IMAGE_TAG: "v1.2.3"
      REPLICA_COUNT: "3"
    substituteFrom:
      # Lấy thêm giá trị từ Secret/ConfigMap
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
```

### Dependency Giữa Kustomization

```yaml
# Đảm bảo infrastructure cài xong trước khi app deploy
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app-config
  path: ./apps/my-app
  dependsOn:
    - name: infrastructure-controllers   # chờ Kustomization này healthy
    - name: cert-manager
```

---

## HelmRelease — Triển Khai Helm Chart

```yaml
# clusters/production/apps/my-app-helm.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: production
spec:
  interval: 15m          # reconcile mỗi 15 phút (helm upgrade nếu có drift)
  releaseName: my-app    # tên helm release (mặc định: metadata.name)
  targetNamespace: production

  chart:
    spec:
      chart: my-app                    # tên chart
      version: ">=1.0.0 <2.0.0"       # semver range
      sourceRef:
        kind: HelmRepository
        name: my-org-charts
        namespace: flux-system

  values:
    replicaCount: 3
    image:
      repository: my-registry.io/my-app
      tag: "v1.2.3"
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"

  # Lấy values từ Secret (cho credentials)
  valuesFrom:
    - kind: Secret
      name: my-app-secrets
      valuesKey: values.yaml
    - kind: ConfigMap
      name: my-app-config
      valuesKey: values-prod.yaml

  upgrade:
    remediation:
      remediateLastFailure: true   # nếu upgrade fail, tự rollback
      retries: 3
  install:
    remediation:
      retries: 3
```

```bash
# Kiểm tra trạng thái HelmRelease
flux get helmreleases -A
# NAMESPACE   NAME     REVISION  SUSPENDED  READY  MESSAGE
# production  my-app   1.2.3     False      True   Release reconciliation succeeded

# Force reconcile ngay
flux reconcile helmrelease my-app -n production

# Suspend (tạm dừng) reconcile để troubleshoot
flux suspend helmrelease my-app -n production
kubectl edit deployment my-app -n production  # sửa thủ công để debug
flux resume helmrelease my-app -n production
```

---

## Image Automation — Cập Nhật Tag Tự Động

Image Automation là tính năng đặc biệt của Flux: tự động phát hiện image tag mới trên registry và cập nhật commit lên Git.

```
CI Pipeline build image → push lên registry (ghcr.io/my-org/my-app:v1.2.4)
                                                        │
                    Flux Image Automation phát hiện tag mới
                                                        │
                    Cập nhật values.yaml trong Git:
                    image.tag: "v1.2.3" → "v1.2.4"
                                                        │
                    Flux Kustomization/HelmRelease nhận thấy Git thay đổi
                                                        │
                    Deploy version mới lên cluster ✓
```

### Cấu Hình Image Automation

```yaml
# Bước 1: ImageRepository — theo dõi registry
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  image: ghcr.io/my-org/my-app   # image path (không có tag)
  interval: 1m                   # quét registry mỗi 1 phút
  secretRef:
    name: ghcr-auth              # credentials để pull từ private registry
---
# Bước 2: ImagePolicy — lọc tag theo rule
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-app
  policy:
    semver:
      range: ">=1.0.0"           # chỉ accept semver >= 1.0.0, bỏ qua pre-release

    # Hoặc dùng alphabetical để lấy tag mới nhất theo thứ tự alphabet
    # alphabetical:
    #   order: asc

    # Hoặc dùng numerical cho tag dạng số
    # numerical:
    #   order: asc
---
# Bước 3: ImageUpdateAutomation — tự động commit vào Git
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: my-app-config
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@my-org.com
        name: FluxBot
      messageTemplate: |
        chore: update {{range .Updated.Images}}{{println .}}{{end}}
    push:
      branch: main
  update:
    path: ./clusters/production   # thư mục cần scan và update
    strategy: Setters             # dùng marker comment trong YAML
```

### Marker Comment Trong values.yaml

```yaml
# values.yaml — thêm marker comment để Flux biết field nào cần update
image:
  repository: ghcr.io/my-org/my-app
  tag: "v1.2.3" # {"$imagepolicy": "flux-system:my-app:tag"}
  # marker trên nói: field tag này được quản lý bởi ImagePolicy "my-app" trong namespace "flux-system"
  # khi có tag mới phù hợp policy, Flux tự commit update tag này
```

---

## Notification Controller

```yaml
# Provider — khai báo endpoint nhận thông báo
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack-devops
  namespace: flux-system
spec:
  type: slack
  channel: "#deployments"
  secretRef:
    name: slack-webhook-secret
---
# Alert — subscribe event và route đến Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: all-apps
  namespace: flux-system
spec:
  providerRef:
    name: slack-devops
  eventSeverity: info        # info hoặc error
  eventSources:
    - kind: HelmRelease      # nhận event từ tất cả HelmRelease
      namespace: "*"
      name: "*"
    - kind: Kustomization
      namespace: "*"
      name: "*"
  exclusionList:
    - ".*no artifact.*"      # bỏ qua message này để giảm noise
```

---

## Multi-Tenancy — Nhiều Team Trên Cùng Cluster

Flux hỗ trợ multi-tenancy bằng cách sử dụng **ServiceAccount** cho mỗi tenant (người thuê) và giới hạn quyền thông qua Kubernetes RBAC thông thường.

```yaml
# Mỗi team có Kustomization riêng, chạy với ServiceAccount riêng
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-backend-apps
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: team-backend-config
  path: ./apps
  prune: true
  serviceAccountName: team-backend-reconciler   # ServiceAccount với quyền hạn chế
  targetNamespace: team-backend                 # giới hạn namespace
---
# ServiceAccount với quyền chỉ trong namespace team-backend
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-backend-reconciler
  namespace: flux-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-backend-reconciler
  namespace: team-backend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: team-backend-reconciler
    namespace: flux-system
```

---

## So Sánh Flux vs ArgoCD

| Tiêu Chí | Flux v2 | ArgoCD |
|---|---|---|
| **UI** | Không có (CLI + kubectl) | Web UI đẹp, trực quan |
| **Philosophy (Triết lý)** | Kubernetes-native, mọi thứ là CRD | Application-centric, UI-first |
| **Helm support** | HelmRelease CRD riêng, mạnh | Source: Helm chart, render xong apply |
| **Kustomize** | Native controller | Hỗ trợ tốt |
| **Image automation** | Built-in (ImagePolicy, ImageUpdateAutomation) | Cần plugin riêng (argocd-image-updater) |
| **Multi-tenancy** | K8s RBAC + ServiceAccount, linh hoạt | AppProject + ArgoCD RBAC |
| **Bootstrap** | `flux bootstrap` — Flux quản lý chính mình | Install thủ công, config không tự-manage |
| **Drift detection** | Mỗi interval | Có webhook + interval |
| **Dependency** | `dependsOn` trong Kustomization | Sync waves + hooks |
| **Community** | Nhỏ hơn, nhưng growing | Lớn hơn, nhiều tài liệu hơn |
| **Phù hợp** | Platform team, GitOps-first, không cần UI | Team cần visual, mới học GitOps |

---

## Câu Hỏi Phỏng Vấn

**Flux Image Automation hoạt động thế nào? Lợi ích so với tự update thủ công?**

> Image Automation hoạt động theo 3 bước. (1) **ImageRepository** quét registry định kỳ (ví dụ mỗi phút) để phát hiện tag mới. (2) **ImagePolicy** lọc tag theo rule — ví dụ chỉ chấp nhận semver `>=1.0.0` để bỏ qua tag như `dev-xxx` hay `pr-123`. (3) **ImageUpdateAutomation** tự động tạo Git commit cập nhật tag trong file YAML đã đánh dấu bằng marker comment. Lợi ích so với update thủ công: loại bỏ bước CI phải có quyền ghi vào config repo, tách biệt hoàn toàn app repo và config repo, Git log có lịch sử tag change rõ ràng không lẫn vào commit code, và có thể cấu hình để chỉ update tag trên branch staging — không ảnh hưởng production cho đến khi review và merge.

**Kustomization của Flux khác với `kustomize` CLI thế nào?**

> Đây là điểm dễ nhầm lẫn. `kustomize` là tool CLI dùng để render Kubernetes manifest với layering/patching. **Kustomization của Flux** là một Custom Resource (tài nguyên tùy chỉnh) — một Kubernetes object khai báo: "lấy manifest từ source X, render bằng Kustomize, apply vào cluster, reconcile mỗi N phút". Flux Kustomization *có thể* (nhưng không bắt buộc) có `kustomization.yaml` trong thư mục đích — nếu không có, Flux apply tất cả YAML file trong thư mục đó như plain manifest. Nói cách khác: `kustomize` là tool render, còn Flux Kustomization là controller orchestrate việc "fetch → render → apply → monitor" liên tục.

**Làm thế nào để rollback khi Flux HelmRelease gây lỗi?**

> Flux HelmRelease có thể cấu hình `upgrade.remediation.remediateLastFailure: true` và `retries: 3` để tự động rollback nếu upgrade fail. Nếu cần rollback thủ công: (1) Suspend HelmRelease để Flux không override: `flux suspend helmrelease my-app -n production`; (2) Rollback bằng Helm CLI: `helm rollback my-app -n production`; (3) Update version trong Git về phiên bản cũ; (4) Resume: `flux resume helmrelease my-app -n production` và Flux reconcile về version trong Git. Cách clean nhất là revert commit trong Git (git revert) — đây là spirit của GitOps: Git là nguồn sự thật, rollback = revert Git commit.
