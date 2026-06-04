# RBAC — Kiểm Soát Truy Cập Dựa Trên Vai Trò

> Hướng dẫn chi tiết về RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò) trong Kubernetes: Role, ClusterRole, RoleBinding, ClusterRoleBinding, và các pattern phân quyền thực chiến.

## Mục Lục

1. [RBAC Là Gì?](#rbac-là-gì)
2. [4 Tài Nguyên RBAC](#4-tài-nguyên-rbac)
3. [Verbs và API Groups](#verbs-và-api-groups)
4. [Role và ClusterRole](#role-và-clusterrole)
5. [RoleBinding và ClusterRoleBinding](#rolebinding-và-clusterrolebinding)
6. [Aggregated ClusterRole](#aggregated-clusterrole)
7. [Pattern Phân Quyền Thực Chiến](#pattern-phân-quyền-thực-chiến)
8. [Debug và Kiểm Tra Quyền](#debug-và-kiểm-tra-quyền)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## RBAC Là Gì?

**RBAC (Role-Based Access Control)** là hệ thống phân quyền chính của Kubernetes. Mọi request đến API Server đều phải qua 3 bước:

```
Request đến API Server
        │
        ▼
1. Authentication (Xác Thực) — bạn là ai?
        │  TLS certificate, Bearer token, OIDC token
        ▼
2. Authorization (Phân Quyền) — bạn được làm gì?
        │  RBAC kiểm tra Role/ClusterRole của subject
        ▼
3. Admission Control (Kiểm Soát Tiếp Nhận) — request có hợp lệ không?
        │  Webhook, PSA, ResourceQuota
        ▼
API Server xử lý request
```

**RBAC bật mặc định từ K8s 1.8+.** Không cần cấu hình thêm để sử dụng.

---

## 4 Tài Nguyên RBAC

```
Subject (Chủ Thể)          Binding (Ràng Buộc)       Role (Vai Trò)
─────────────────        ─────────────────────        ────────────────
User                  ←── RoleBinding         ───→  Role (namespace)
Group                 ←── ClusterRoleBinding  ───→  ClusterRole (global)
ServiceAccount
```

| Tài Nguyên | Phạm Vi | Mô Tả |
| ---------- | ------- | ------ |
| `Role` | Namespace | Định nghĩa quyền trong một namespace cụ thể |
| `ClusterRole` | Cluster-wide | Định nghĩa quyền trên toàn cluster hoặc tài nguyên non-namespaced |
| `RoleBinding` | Namespace | Gán Role hoặc ClusterRole cho subject trong một namespace |
| `ClusterRoleBinding` | Cluster-wide | Gán ClusterRole cho subject trên toàn cluster |

> **Kết hợp quan trọng:** `RoleBinding` có thể gán `ClusterRole` vào namespace — dùng ClusterRole như template, áp dụng trong namespace giới hạn.

---

## Verbs và API Groups

### Verbs — Hành Động

| Verb | Mô Tả | HTTP Method tương đương |
| ---- | ------ | ----------------------- |
| `get` | Đọc một resource cụ thể | GET /resource/{name} |
| `list` | Liệt kê tất cả resource | GET /resource |
| `watch` | Theo dõi thay đổi realtime | GET /resource?watch=true |
| `create` | Tạo resource mới | POST /resource |
| `update` | Cập nhật toàn bộ resource | PUT /resource/{name} |
| `patch` | Cập nhật một phần resource | PATCH /resource/{name} |
| `delete` | Xoá resource | DELETE /resource/{name} |
| `deletecollection` | Xoá nhiều resource | DELETE /resource |
| `exec` | Chạy lệnh trong Pod | POST /pods/{name}/exec |
| `portforward` | Chuyển tiếp port | POST /pods/{name}/portforward |

### API Groups — Nhóm API

```bash
# Core group (nhóm lõi) — apiGroup: ""
resources: pods, services, secrets, configmaps, nodes, namespaces, persistentvolumes, ...

# apps group
apiGroup: "apps"
resources: deployments, replicasets, statefulsets, daemonsets, ...

# batch group
apiGroup: "batch"
resources: jobs, cronjobs, ...

# networking group
apiGroup: "networking.k8s.io"
resources: ingresses, networkpolicies, ...

# rbac group
apiGroup: "rbac.authorization.k8s.io"
resources: roles, clusterroles, rolebindings, clusterrolebindings, ...

# autoscaling group
apiGroup: "autoscaling"
resources: horizontalpodautoscalers, ...
```

---

## Role và ClusterRole

### Role — Quyền Trong Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production          # chỉ áp dụng trong namespace này
rules:
  - apiGroups: [""]              # core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["app-secret", "db-credentials"]  # chỉ secret cụ thể
    verbs: ["get"]               # chỉ get, không list
```

### ClusterRole — Quyền Toàn Cluster

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]

  # Non-namespaced resource — chỉ ClusterRole mới định nghĩa được
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]

  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list"]
```

### ClusterRole Cho Developer (Chỉ Đọc)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-readonly
rules:
  - apiGroups: ["", "apps", "batch", "autoscaling", "networking.k8s.io"]
    resources:
      - pods
      - pods/log
      - pods/status
      - deployments
      - replicasets
      - statefulsets
      - daemonsets
      - jobs
      - cronjobs
      - services
      - ingresses
      - horizontalpodautoscalers
      - configmaps
    verbs: ["get", "list", "watch"]
  # Không có quyền với secrets — developer không cần đọc secret
```

### ClusterRole Tích Hợp Sẵn

Kubernetes cung cấp một số ClusterRole mặc định:

```bash
# Xem tất cả ClusterRole tích hợp sẵn
kubectl get clusterroles | grep -v "^system:"

# Các role quan trọng:
cluster-admin   # toàn quyền trên mọi thứ — dùng cực kỳ cẩn thận
admin           # admin trong namespace — không có quyền tạo namespace và ResourceQuota
edit            # tạo/sửa/xoá hầu hết resource — không có quyền với Role/RoleBinding
view            # chỉ đọc hầu hết resource — không thấy Secret

# Xem chi tiết một ClusterRole
kubectl describe clusterrole view
```

---

## RoleBinding và ClusterRoleBinding

### RoleBinding — Gán Role Trong Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: john-pod-reader
  namespace: production         # binding chỉ có hiệu lực trong namespace này
subjects:
  # User (người dùng cụ thể)
  - kind: User
    name: john@example.com      # phải khớp với CN trong certificate
    apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role                    # hoặc ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Gán Cho Group — Team

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-edit
  namespace: backend
subjects:
  - kind: Group
    name: backend-team          # group từ OIDC provider (Keycloak, Okta, Google)
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit                    # dùng ClusterRole tích hợp sẵn
  apiGroup: rbac.authorization.k8s.io
```

### Gán Cho ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-deploy-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-deployer        # tên ServiceAccount
    namespace: cicd             # namespace của ServiceAccount (có thể khác namespace binding)
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding — Gán Quyền Toàn Cluster

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-admin
subjects:
  - kind: Group
    name: platform-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin           # toàn quyền — dùng cho platform/SRE team
  apiGroup: rbac.authorization.k8s.io
```

> **Cảnh báo:** `cluster-admin` cho phép làm **bất kỳ điều gì** trong cluster — kể cả xoá namespace, xoá node, đọc mọi secret. Chỉ cấp cho người thực sự cần (platform admin, break-glass user).

---

## Aggregated ClusterRole

**Aggregated ClusterRole** cho phép kết hợp nhiều ClusterRole thành một — khi thêm CRD mới (Custom Resource Definition), có thể tự động kế thừa vào role tổng hợp mà không cần sửa role gốc.

```yaml
# Role tổng hợp — tự động thu thập rules từ role có label phù hợp
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-full-access
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
rules: []   # rules được fill tự động

---
# Role cho Prometheus CRD — sẽ được kế thừa vào monitoring-full-access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-rules-access
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"   # label kích hoạt aggregation
rules:
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["prometheusrules", "servicemonitors"]
    verbs: ["get", "list", "watch", "create", "update"]
```

---

## Pattern Phân Quyền Thực Chiến

### Pattern 1: Namespace Isolation Cho Đa Team

```yaml
# Mỗi team chỉ có quyền trong namespace của mình
---
# Backend team — namespace: backend
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-access
  namespace: backend
subjects:
  - kind: Group
    name: backend-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit              # create/update/delete nhưng không có Role/RoleBinding
  apiGroup: rbac.authorization.k8s.io

---
# Frontend team — namespace: frontend
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-access
  namespace: frontend
subjects:
  - kind: Group
    name: frontend-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

### Pattern 2: CI/CD Pipeline Có Quyền Deploy

```yaml
---
# ServiceAccount cho GitHub Actions / ArgoCD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-actions-deployer
  namespace: cicd

---
# Role chỉ deploy trong namespace production
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "create", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-deploy
  namespace: production
subjects:
  - kind: ServiceAccount
    name: github-actions-deployer
    namespace: cicd
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### Pattern 3: Read-Only Cho Monitoring

```yaml
---
# Prometheus cần đọc metrics từ mọi namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
  apiGroup: rbac.authorization.k8s.io
```

### Pattern 4: Break-Glass Access (Truy Cập Khẩn Cấp)

```yaml
# Tài khoản cluster-admin chỉ dùng khi sự cố nghiêm trọng
# Không gán thường xuyên — dùng Just-in-Time access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: break-glass-admin
  annotations:
    # Ghi chú: binding này chỉ active khi sự cố, xoá sau khi xử lý xong
    reason: "P0 incident 2026-05-10 — database migration failure"
    expires: "2026-05-11T00:00:00Z"
subjects:
  - kind: User
    name: oncall@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## Debug và Kiểm Tra Quyền

### Kiểm Tra Quyền Của Mình

```bash
# Mình có thể làm gì trong namespace production?
kubectl auth can-i --list -n production

# Kiểm tra quyền cụ thể
kubectl auth can-i get pods -n production
kubectl auth can-i delete deployments -n production
kubectl auth can-i create secrets -n production

# Kiểm tra với tư cách user khác (admin check)
kubectl auth can-i get secrets -n production --as john@example.com
kubectl auth can-i delete pods -n kube-system --as system:serviceaccount:monitoring:prometheus
```

### Xem Tất Cả Binding

```bash
# Liệt kê tất cả RoleBinding trong cluster
kubectl get rolebindings -A

# Liệt kê tất cả ClusterRoleBinding
kubectl get clusterrolebindings

# Tìm user/SA có quyền gì trên một namespace
kubectl get rolebindings -n production -o wide

# Tìm tất cả binding liên quan đến một subject
kubectl get rolebindings,clusterrolebindings -A \
  -o jsonpath='{range .items[?(@.subjects[*].name=="john@example.com")]}{.metadata.name}{"\n"}{end}'
```

### Xem Chi Tiết Role

```bash
# Chi tiết ClusterRole
kubectl describe clusterrole cluster-admin
kubectl describe clusterrole edit
kubectl describe clusterrole view

# Chi tiết Role trong namespace
kubectl describe role pod-reader -n production

# Export YAML để review
kubectl get clusterrole edit -o yaml
```

### Audit: Phát Hiện Overpermission

```bash
# Tìm tất cả binding có cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | {name: .metadata.name, subjects: .subjects}'

# Tìm ServiceAccount có quyền truy cập secrets
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.rules[]?.resources[]? == "secrets")'

# Dùng kubectl-who-can plugin (cần cài thêm)
kubectl who-can list secrets
kubectl who-can delete pods -n production
```

---

## Câu Hỏi Phỏng Vấn

**Role và ClusterRole khác nhau thế nào? Khi nào dùng cái nào?**

> `Role` chỉ có hiệu lực trong một namespace — phù hợp khi muốn cấp quyền giới hạn trong namespace cụ thể, ví dụ developer team có quyền deploy trong namespace của team mình. `ClusterRole` có hiệu lực trên toàn cluster và có thể định nghĩa quyền cho tài nguyên non-namespaced (nodes, persistentvolumes, namespaces). Dùng `ClusterRole` cho: hệ thống cần xem toàn bộ cluster (Prometheus), quyền quản lý node (kubelet), hoặc muốn tái sử dụng role definition qua nhiều namespace (tạo ClusterRole rồi bind bằng RoleBinding trong từng namespace riêng lẻ).

**Tại sao RoleBinding có thể liên kết ClusterRole? Ý nghĩa gì?**

> Đây là pattern rất hữu ích: tạo `ClusterRole` như một template định nghĩa quyền, rồi dùng `RoleBinding` để áp dụng trong namespace giới hạn. Ví dụ: có ClusterRole `developer` cho phép `get/list/create deployment` — mỗi team tạo RoleBinding trong namespace của mình liên kết với ClusterRole này. Lợi ích: (1) Định nghĩa quyền một lần, dùng nhiều namespace; (2) Thay đổi ClusterRole tự động áp dụng cho mọi namespace đang dùng nó; (3) Không cần tạo Role duplicate trong mỗi namespace.

**Làm thế nào để debug khi user gặp lỗi "Forbidden"?**

> Quy trình debug: (1) `kubectl auth can-i <verb> <resource> --as <user>` để kiểm tra quyền cụ thể; (2) `kubectl auth can-i --list --as <user> -n <namespace>` xem toàn bộ quyền; (3) `kubectl get rolebindings,clusterrolebindings -A -o wide` tìm binding của user đó; (4) Kiểm tra group membership nếu user được gán quyền qua group; (5) Kiểm tra namespace — có thể role được tạo ở namespace khác. Thường lỗi "Forbidden" do thiếu binding, binding sai namespace, hoặc tên subject không khớp (case-sensitive).

**Nguyên tắc Least Privilege áp dụng thế nào trong RBAC?**

> Áp dụng cụ thể: (1) Không dùng `cluster-admin` cho user thường — cấp đúng Role cần thiết; (2) Tránh `list/watch` secret toàn namespace — dùng `get` với `resourceNames` cụ thể; (3) ServiceAccount mặc định không cần quyền gì — `automountServiceAccountToken: false` nếu không gọi K8s API; (4) Audit định kỳ dùng `kubectl auth can-i --list` và loại bỏ quyền không còn cần; (5) Phân biệt read (get, list, watch) và write (create, update, delete) — dev thường chỉ cần read trong production, write trong staging.
