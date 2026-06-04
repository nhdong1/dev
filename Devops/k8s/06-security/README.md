# Security — Bảo Mật Cluster Kubernetes

> Tổng quan về bảo mật Kubernetes theo mô hình phòng thủ nhiều lớp (Defense in Depth): từ kiểm soát truy cập qua RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò), bảo mật Pod, bảo mật mạng, quét lỗ hổng image, đến mã hoá dữ liệu lưu trữ.

## Mục Lục

1. [Mô Hình Bảo Mật Kubernetes](#mô-hình-bảo-mật-kubernetes)
2. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
3. [Các Thành Phần Bảo Mật](#các-thành-phần-bảo-mật)
4. [4C Security Model](#4c-security-model)
5. [Ma Trận Mối Đe Doạ vs Biện Pháp](#ma-trận-mối-đe-doạ-vs-biện-pháp)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
7. [Checklist Bảo Mật Production](#checklist-bảo-mật-production)

---

## Mô Hình Bảo Mật Kubernetes

Bảo mật Kubernetes không phải một điểm đơn lẻ — đây là **nhiều lớp phòng thủ** xếp chồng nhau. Nếu một lớp bị vượt qua, các lớp còn lại vẫn giữ an toàn.

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloud / Infrastructure                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                      Cluster                          │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │                   Container                     │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │                   Code                    │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**4C Security Model** (Mô Hình Bảo Mật 4C):
- **Cloud:** Bảo mật hạ tầng cloud — IAM, VPC, firewall, node OS hardening
- **Cluster:** RBAC, NetworkPolicy, Pod Security, Audit Logging
- **Container:** Image scanning, non-root user, read-only filesystem, resource limit
- **Code:** Dependency scanning, SAST, secret management trong code

---

## Bản Đồ Quyết Định

```
Bạn cần bảo vệ điều gì?
│
├── Kiểm soát ai được làm gì trong cluster?
│   ├── User / CI/CD pipeline truy cập API Server
│   └── → RBAC (Role, ClusterRole, RoleBinding, ClusterRoleBinding)
│       └── Xem: 1-rbac.md
│
├── Pod / ứng dụng cần gọi Kubernetes API hoặc dịch vụ cloud?
│   ├── Pod gọi K8s API: kubectl, helm, operator
│   ├── Pod trên EKS cần truy cập AWS: S3, RDS, SQS
│   └── → ServiceAccount + IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho ServiceAccount)
│       └── Xem: 2-service-account.md
│
├── Hạn chế quyền hạn của bản thân Pod?
│   ├── Không cho chạy root, không cho mount host, không cho privileged
│   └── → Pod Security Admission (PSA — Kiểm Soát Bảo Mật Pod)
│       └── Xem: 3-pod-security.md
│
├── Kiểm soát lưu lượng mạng giữa các Pod?
│   ├── Service A không nên gọi Service B
│   ├── Cần mã hoá traffic nội bộ cluster
│   └── → NetworkPolicy + mTLS (mutual TLS — TLS Hai Chiều)
│       └── Xem: 4-network-security.md
│
├── Kiểm soát image nào được chạy trong cluster?
│   ├── Image có lỗ hổng CVE nghiêm trọng
│   ├── Image từ registry không tin cậy
│   └── → Image Scanning + OPA Gatekeeper (Open Policy Agent — Tác Nhân Chính Sách Mở)
│       └── Xem: 5-image-security.md
│
└── Bảo vệ dữ liệu nhạy cảm lưu trong cluster?
    ├── Secret lưu plaintext trong etcd
    ├── Backup etcd có thể bị đọc
    └── → Encryption at Rest (Mã Hoá Tại Nơi Lưu Trữ) + KMS
        └── Xem: 6-secrets-encryption.md
```

---

## Các Thành Phần Bảo Mật

### RBAC — Kiểm Soát Truy Cập Dựa Trên Vai Trò

**RBAC (Role-Based Access Control)** là cơ chế phân quyền chính của Kubernetes. Mọi thao tác với API Server đều phải qua RBAC — từ developer chạy `kubectl get pods` đến CI/CD pipeline deploy ứng dụng.

```
User / ServiceAccount
        │
        │  subject của
        ▼
   RoleBinding / ClusterRoleBinding
        │
        │  liên kết tới
        ▼
   Role / ClusterRole
        │
        │  định nghĩa quyền
        ▼
   apiGroups + resources + verbs
   (ví dụ: ["apps"] + ["deployments"] + ["get","list","update"])
```

**4 tài nguyên RBAC cốt lõi:**
- `Role` — quyền trong một namespace
- `ClusterRole` — quyền trên toàn cluster
- `RoleBinding` — gán Role cho user/group/SA trong một namespace
- `ClusterRoleBinding` — gán ClusterRole trên toàn cluster

Tham khảo chi tiết: [1-rbac.md](./1-rbac.md)

---

### ServiceAccount — Tài Khoản Dịch Vụ

**ServiceAccount** là danh tính (identity) Kubernetes cấp cho Pod — khác với user account dành cho con người. Mỗi Pod chạy dưới một ServiceAccount và có thể được cấp quyền RBAC để gọi Kubernetes API hoặc dịch vụ cloud.

```
Pod
 │  chạy dưới
 ▼
ServiceAccount
 │  được gắn RBAC Role
 ▼
Kubernetes API / AWS IAM (qua IRSA trên EKS)
```

**IRSA (IAM Roles for Service Accounts)** là cơ chế EKS cho phép Pod assume IAM role mà không cần lưu access key — token ServiceAccount được trao đổi lấy AWS credential tạm thời qua OIDC.

Tham khảo chi tiết: [2-service-account.md](./2-service-account.md)

---

### Pod Security Admission (PSA) — Kiểm Soát Bảo Mật Pod

**PSA (Pod Security Admission — Kiểm Soát Bảo Mật Pod)** là admission controller tích hợp sẵn (từ K8s 1.25+) kiểm tra Pod spec trước khi cho phép chạy. Thay thế PodSecurityPolicy (PSP) đã bị loại bỏ.

**3 cấp độ bảo mật:**
- `Privileged` — không giới hạn (chỉ dùng cho system namespace)
- `Baseline` — ngăn chặn cấu hình nguy hiểm phổ biến (khuyến nghị minimum)
- `Restricted` — tiêu chuẩn bảo mật cao nhất (khuyến nghị production workload)

**3 chế độ thực thi:**
- `enforce` — từ chối Pod vi phạm
- `audit` — ghi log vi phạm nhưng cho phép
- `warn` — hiện cảnh báo nhưng cho phép

Tham khảo chi tiết: [3-pod-security.md](./3-pod-security.md)

---

### Network Security — Bảo Mật Mạng

**Mặc định trong Kubernetes, mọi Pod đều có thể giao tiếp với mọi Pod khác** — không có phân vùng mạng. Để bảo mật, cần kết hợp:

- **NetworkPolicy** — tường lửa Layer 3/4, kiểm soát lưu lượng vào/ra theo IP và port
- **mTLS (mutual TLS — TLS Hai Chiều)** — mã hoá và xác thực lẫn nhau ở Layer 7, thường qua Service Mesh (Istio, Linkerd)
- **Ingress TLS** — mã hoá HTTPS từ client đến Ingress Controller

```
Internet → [Ingress TLS] → Ingress → [mTLS] → Service A → [NetworkPolicy] → Service B
```

Tham khảo chi tiết: [4-network-security.md](./4-network-security.md)

---

### Image Security — Bảo Mật Container Image

Container image là vector tấn công quan trọng — image chứa lỗ hổng CVE (Common Vulnerabilities and Exposures — Lỗ Hổng Bảo Mật Phổ Biến) có thể bị khai thác sau khi deploy.

**Hai lớp kiểm soát:**
1. **Scanning (Quét lỗ hổng):** Trivy, Snyk, Grype — phát hiện CVE trong image trước khi deploy
2. **Policy Enforcement (Thực Thi Chính Sách):** OPA Gatekeeper, Kyverno — từ chối image từ registry không tin cậy hoặc image chưa được scan

```
Build → [Scan Trivy] → Registry → [OPA Gatekeeper check] → Cluster
```

Tham khảo chi tiết: [5-image-security.md](./5-image-security.md)

---

### Secrets Encryption — Mã Hoá Bí Mật

**Mặc định, Secret trong Kubernetes chỉ được base64 encode — không phải encrypt**. Ai có quyền đọc etcd (backup, admin) đều xem được giá trị thật.

**Encryption at Rest (Mã Hoá Tại Nơi Lưu Trữ)** mã hoá dữ liệu Secret trước khi ghi vào etcd, dùng key từ:
- **Local key (aescbc, aesgcm)** — đơn giản nhưng key vẫn trên control plane
- **KMS (Key Management Service)** — AWS KMS, GCP KMS, HashiCorp Vault — bảo mật nhất

Tham khảo chi tiết: [6-secrets-encryption.md](./6-secrets-encryption.md)

---

## 4C Security Model

### Cloud — Lớp Hạ Tầng

```
Quan tâm đến:
- Node OS: cập nhật patch, CIS benchmark hardening
- IAM cloud: least privilege cho node role
- VPC/Network: không expose API Server ra internet nếu không cần
- etcd: backup mã hoá, không expose port 2379 ra ngoài
- Control plane access: private endpoint (EKS private cluster)
```

### Cluster — Lớp Kubernetes

```
Quan tâm đến:
- RBAC: principle of least privilege (nguyên tắc đặc quyền tối thiểu)
- Network Policy: default deny all, allow explicit
- Pod Security Admission: Restricted mode cho production
- Audit Logging: ghi lại mọi thao tác với API Server
- API Server flags: anonymous-auth=false, authorization-mode=RBAC
- Admission Controllers: NodeRestriction, AlwaysPullImages
```

### Container — Lớp Container

```
Quan tâm đến:
- Base image: dùng image nhỏ, ít lớp (distroless, alpine)
- Non-root user: runAsNonRoot: true, runAsUser: 1000
- Read-only filesystem: readOnlyRootFilesystem: true
- Drop capabilities: drop ALL, chỉ thêm capability cần thiết
- Resource limit: tránh DoS do resource starvation
- No privilege escalation: allowPrivilegeEscalation: false
```

### Code — Lớp Ứng Dụng

```
Quan tâm đến:
- Dependency scanning: kiểm tra thư viện có CVE không
- SAST (Static Application Security Testing — Kiểm Tra Bảo Mật Ứng Dụng Tĩnh)
- Không hardcode secret trong source code
- Input validation: ngăn injection attack
- TLS cho mọi kết nối ra ngoài
```

---

## Ma Trận Mối Đe Doạ vs Biện Pháp

| Mối Đe Doạ | Biện Pháp Đối Phó | File Tham Khảo |
| ---------- | ----------------- | -------------- |
| Developer có quá nhiều quyền | RBAC least privilege, namespace isolation | [1-rbac.md](./1-rbac.md) |
| Pod lấy credential AWS hardcode | IRSA, ServiceAccount annotation | [2-service-account.md](./2-service-account.md) |
| Container chạy root, leo thang đặc quyền | PSA Restricted, securityContext | [3-pod-security.md](./3-pod-security.md) |
| Pod A tấn công Pod B qua mạng nội bộ | NetworkPolicy default-deny, mTLS | [4-network-security.md](./4-network-security.md) |
| Image chứa CVE nghiêm trọng được deploy | Trivy CI scan + OPA Gatekeeper policy | [5-image-security.md](./5-image-security.md) |
| Backup etcd bị đọc, lộ Secret | Encryption at Rest với KMS | [6-secrets-encryption.md](./6-secrets-encryption.md) |
| Supply chain attack — image bị tamper | Image signing với Cosign + Sigstore | [5-image-security.md](./5-image-security.md) |
| Lateral movement qua compromised Pod | RBAC chặt cho SA, NetworkPolicy | [1-rbac.md](./1-rbac.md) + [4-network-security.md](./4-network-security.md) |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**Kubernetes bảo mật theo những lớp nào?**

> Kubernetes tuân theo mô hình **4C Defense in Depth**: Cloud (hạ tầng, node OS, IAM cloud), Cluster (RBAC, NetworkPolicy, PSA, Audit Logging), Container (non-root, read-only filesystem, drop capabilities), Code (dependency scan, không hardcode secret). Không có lớp đơn lẻ nào là đủ — cần kết hợp tất cả để đạt bảo mật production.

**RBAC hoạt động thế nào trong Kubernetes?**

> RBAC (Role-Based Access Control) kiểm soát mọi thao tác với API Server. Có 4 tài nguyên: `Role` (quyền trong namespace), `ClusterRole` (quyền trên toàn cluster), `RoleBinding` (gán Role cho subject trong namespace), `ClusterRoleBinding` (gán ClusterRole toàn cluster). Subject có thể là User, Group, hoặc ServiceAccount. Mỗi quyền định nghĩa bởi 3 chiều: `apiGroups` (nhóm API), `resources` (loại tài nguyên), `verbs` (hành động: get, list, create, update, delete, watch).

**Nguyên tắc Least Privilege trong Kubernetes nghĩa là gì?**

> Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu) — mỗi người dùng, ServiceAccount, hay tiến trình chỉ được cấp đúng quyền cần thiết để thực hiện nhiệm vụ, không hơn không kém. Trong K8s áp dụng: (1) Không cấp ClusterAdmin khi chỉ cần quyền một namespace; (2) Không cấp `list/watch` secret khi chỉ cần `get` một secret cụ thể; (3) ServiceAccount mặc định nên không có quyền gì — chỉ thêm khi thực sự cần; (4) Dùng `resourceNames` để giới hạn chính xác đến tên tài nguyên.

### Câu Hỏi Nâng Cao

**Làm thế nào để bảo mật cluster K8s cho môi trường production?**

> Production security checklist: (1) **RBAC:** least privilege, namespace isolation, không dùng `cluster-admin` cho CI/CD; (2) **Pod Security:** PSA Restricted hoặc Baseline cho mọi namespace production; (3) **Network:** NetworkPolicy default-deny, mTLS qua Istio/Linkerd, TLS ingress; (4) **Image:** scan Trivy trong CI, OPA Gatekeeper từ chối image chưa scan hoặc từ registry không tin cậy; (5) **Secrets:** Encryption at Rest với KMS, External Secrets Operator, không commit Secret lên Git; (6) **Audit:** bật Kubernetes Audit Log, forward vào SIEM; (7) **Node:** CIS Benchmark hardening, immutable node, auto-patching.

**Sự khác nhau giữa NetworkPolicy và mTLS?**

> **NetworkPolicy** hoạt động ở Layer 3/4 (IP và port) — kiểm soát Pod nào được kết nối đến Pod nào dựa trên label selector và CIDR. Nó không mã hoá traffic và không xác thực danh tính ứng dụng. **mTLS (mutual TLS)** hoạt động ở Layer 7 — mã hoá toàn bộ traffic và xác thực danh tính hai chiều (cả client và server). Thường được triển khai qua Service Mesh (Istio, Linkerd). Hai kỹ thuật bổ sung cho nhau: NetworkPolicy ngăn kết nối trái phép ở tầng network, mTLS mã hoá và xác thực tầng ứng dụng.

---

## Checklist Bảo Mật Production

### RBAC và Truy Cập

- [ ] Không gán `cluster-admin` cho user hoặc ServiceAccount thông thường
- [ ] Mỗi team chỉ có quyền trong namespace của mình — không có quyền cross-namespace
- [ ] CI/CD pipeline dùng ServiceAccount riêng với quyền tối thiểu (deploy namespace cụ thể)
- [ ] Audit định kỳ: `kubectl get clusterrolebindings,rolebindings -A` để phát hiện overpermission
- [ ] Disable ServiceAccount token auto-mounting cho Pod không cần gọi K8s API

### Pod Security

- [ ] Áp dụng PSA Restricted hoặc Baseline cho tất cả namespace production
- [ ] Tất cả container chạy `runAsNonRoot: true`
- [ ] Không có container chạy `privileged: true` trừ system daemonset
- [ ] Drop ALL linux capabilities, chỉ thêm lại capability cần thiết cụ thể
- [ ] `readOnlyRootFilesystem: true` cho mọi container

### Network Security

- [ ] Áp dụng `default-deny` NetworkPolicy cho mọi namespace
- [ ] Chỉ mở port cần thiết giữa các service — document rõ ràng
- [ ] TLS termination tại Ingress với certificate hợp lệ (Let's Encrypt hoặc internal CA)
- [ ] Xem xét triển khai Service Mesh cho mTLS nếu cluster xử lý dữ liệu nhạy cảm

### Image Security

- [ ] Scan tất cả image trong CI pipeline — fail build nếu có CVE nghiêm trọng (Critical/High)
- [ ] Chỉ cho phép pull image từ registry nội bộ đã được kiểm soát
- [ ] Không dùng tag `latest` — pin version cụ thể
- [ ] Dùng distroless hoặc alpine base image để giảm attack surface

### Secrets và Mã Hoá

- [ ] Bật Encryption at Rest cho etcd — ưu tiên dùng KMS (AWS KMS, GCP KMS)
- [ ] Không commit Secret plaintext lên Git — dùng Sealed Secrets hoặc External Secrets
- [ ] RBAC: giới hạn quyền `get/list` Secret đến ServiceAccount cụ thể
- [ ] Rotate Secret định kỳ — ưu tiên auto-rotation qua Vault hoặc External Secrets

### Monitoring và Audit

- [ ] Bật Kubernetes Audit Logging — ghi lại mọi thao tác với API Server
- [ ] Alert khi có ClusterRoleBinding mới được tạo
- [ ] Alert khi Secret bị truy cập ngoài giờ hoặc từ ServiceAccount bất thường
- [ ] Dùng Falco để detect runtime anomaly (hành vi bất thường lúc runtime)

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
| ---- | -------- |
| [1-rbac.md](./1-rbac.md) | Role, ClusterRole, RoleBinding, ClusterRoleBinding chi tiết |
| [2-service-account.md](./2-service-account.md) | ServiceAccount, token projection, IRSA trên EKS |
| [3-pod-security.md](./3-pod-security.md) | Pod Security Admission, securityContext, runAsNonRoot |
| [4-network-security.md](./4-network-security.md) | NetworkPolicy, mTLS, Ingress TLS |
| [5-image-security.md](./5-image-security.md) | Image scanning Trivy, OPA Gatekeeper, image signing |
| [6-secrets-encryption.md](./6-secrets-encryption.md) | Encryption at Rest etcd, KMS integration |
