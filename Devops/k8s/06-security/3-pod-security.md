# Pod Security — Bảo Mật Cấu Hình Pod

> Hướng dẫn về Pod Security Admission (PSA — Kiểm Soát Bảo Mật Pod), securityContext (ngữ cảnh bảo mật), và các thiết lập bảo mật container: runAsNonRoot, readOnlyRootFilesystem, drop capabilities, và seccomp profile.

## Mục Lục

1. [Tại Sao Cần Pod Security?](#tại-sao-cần-pod-security)
2. [Pod Security Admission (PSA)](#pod-security-admission-psa)
3. [securityContext — Ngữ Cảnh Bảo Mật](#securitycontext--ngữ-cảnh-bảo-mật)
4. [Capability Linux — Đặc Quyền Linux](#capability-linux--đặc-quyền-linux)
5. [Seccomp Profile — Lọc System Call](#seccomp-profile--lọc-system-call)
6. [AppArmor Profile](#apparmor-profile)
7. [Pod Security Context Toàn Diện](#pod-security-context-toàn-diện)
8. [OPA Gatekeeper Cho Policy Nâng Cao](#opa-gatekeeper-cho-policy-nâng-cao)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Pod Security?

Mặc định, container trong Kubernetes có thể:
- Chạy với user `root` (uid 0) — nếu container bị compromise, attacker có root trong container
- Leo thang đặc quyền (`setuid`) lên root ngay cả khi khởi động không phải root
- Mount host filesystem, host network, host PID namespace — thoát khỏi container
- Dùng Linux capability như `NET_ADMIN`, `SYS_ADMIN` — thao tác mạng, kernel module

```
Container bị compromise (không có security hardening)
        │
        ▼
Attacker có root trong container
        │
        ├── Mount hostPath → đọc file nhạy cảm trên node
        ├── Dùng SYS_ADMIN → mount thiết bị, thao tác namespace
        ├── Dùng NET_ADMIN → sniff traffic, thay đổi routing
        └── Privileged mode → thao tác trực tiếp kernel — container escape
```

**Pod Security** ngăn chặn các vector này bằng cách hạn chế Pod spec.

---

## Pod Security Admission (PSA)

**PSA (Pod Security Admission)** là admission controller tích hợp sẵn từ K8s 1.25 — thay thế PodSecurityPolicy (PSP) đã bị loại bỏ. PSA kiểm tra Pod spec khi tạo/update.

### 3 Cấp Độ Bảo Mật (Security Level)

```
Privileged ← Baseline ← Restricted
(ít giới hạn)           (nhiều giới hạn)
```

| Cấp Độ | Mô Tả | Dùng Cho |
| ------ | ------ | -------- |
| `privileged` | Không giới hạn — cho phép mọi thứ | kube-system, CNI plugins, CSI drivers |
| `baseline` | Chặn cấu hình nguy hiểm phổ biến — cho phép mặc định | Production namespace minimum |
| `restricted` | Tiêu chuẩn bảo mật cao nhất | Production workload, tài chính, y tế |

### 3 Chế Độ Thực Thi (Enforcement Mode)

| Chế Độ | Hành Động Khi Vi Phạm | Dùng Khi Nào |
| ------ | --------------------- | ------------ |
| `enforce` | Từ chối tạo Pod | Chắc chắn workload tương thích |
| `audit` | Cho phép nhưng ghi vào audit log | Giai đoạn migration |
| `warn` | Cho phép nhưng hiện warning | Thử nghiệm, onboarding |

### Áp Dụng PSA Qua Label Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce: từ chối Pod vi phạm
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # Audit: ghi log vi phạm
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest

    # Warn: hiện cảnh báo
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

```bash
# Áp dụng PSA cho namespace hiện có
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Kiểm tra label namespace
kubectl get namespace production --show-labels

# Dry-run: kiểm tra Pod nào sẽ bị từ chối
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --dry-run=server

# Kiểm tra Pod spec có pass PSA level không
kubectl apply --dry-run=server -f pod.yaml -n production
```

### So Sánh Baseline vs Restricted

**Baseline** ngăn chặn:
- `hostProcess: true`, `hostNetwork: true`, `hostPID: true`, `hostIPC: true`
- `privileged: true`
- `allowPrivilegeEscalation: true` (nếu container chạy uid 0)
- Một số `capabilities` nguy hiểm: `NET_RAW`, `SYS_ADMIN`, v.v.
- `hostPath` volume
- `appArmor` override không an toàn

**Restricted** bổ sung thêm (ngoài Baseline):
- `runAsNonRoot: true` — **bắt buộc**
- `allowPrivilegeEscalation: false` — **bắt buộc**
- `seccompProfile.type: RuntimeDefault` hoặc `Localhost` — **bắt buộc**
- Chỉ cho phép `capabilities: drop: ["ALL"]` — **bắt buộc**
- Volume types giới hạn: chỉ ConfigMap, Secret, emptyDir, projected, downwardAPI, PVC

---

## securityContext — Ngữ Cảnh Bảo Mật

`securityContext` có thể khai báo ở hai cấp:
- **Pod level** (`spec.securityContext`) — áp dụng cho mọi container trong Pod
- **Container level** (`spec.containers[].securityContext`) — override Pod level

### Pod-Level securityContext

```yaml
spec:
  securityContext:
    runAsNonRoot: true          # Pod phải chạy non-root user
    runAsUser: 1000             # UID của tất cả container (nếu image không chỉ định)
    runAsGroup: 3000            # GID chính
    fsGroup: 2000               # GID cho volume — file trong volume thuộc group này
    fsGroupChangePolicy: "OnRootMismatch"  # chỉ chown khi cần — nhanh hơn Always
    supplementalGroups: [4000]  # GID phụ bổ sung
    sysctls:                    # kernel parameter (chỉ safe sysctls)
      - name: net.core.somaxconn
        value: "1024"
    seccompProfile:
      type: RuntimeDefault       # dùng seccomp profile mặc định của container runtime
```

### Container-Level securityContext

```yaml
spec:
  containers:
    - name: app
      image: myapp:v1
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        allowPrivilegeEscalation: false    # QUAN TRỌNG: không cho leo thang quyền
        readOnlyRootFilesystem: true       # filesystem chỉ đọc
        privileged: false                  # không privileged
        capabilities:
          drop:
            - ALL                          # bỏ tất cả Linux capabilities
          add:
            - NET_BIND_SERVICE             # chỉ thêm lại cái cần thiết (bind port < 1024)
        seccompProfile:
          type: RuntimeDefault
```

### Ví Dụ Thực Tế — Web Application An Toàn

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-webapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-webapp
  template:
    metadata:
      labels:
        app: secure-webapp
    spec:
      # Pod level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      # Không mount SA token nếu không cần K8s API
      automountServiceAccountToken: false

      containers:
        - name: webapp
          image: myapp:v1.2.3              # pin version cụ thể, không dùng latest
          ports:
            - containerPort: 8080

          # Container level security
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true    # filesystem chỉ đọc
            capabilities:
              drop:
                - ALL

          # Nếu cần ghi file, mount volume riêng
          volumeMounts:
            - name: tmp-dir
              mountPath: /tmp               # thư mục /tmp để app ghi tạm thời
            - name: cache-dir
              mountPath: /app/cache

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

      volumes:
        - name: tmp-dir
          emptyDir: {}                      # tmp writable volume
        - name: cache-dir
          emptyDir: {}
```

---

## Capability Linux — Đặc Quyền Linux

Linux capabilities (đặc quyền Linux) là cơ chế chia nhỏ quyền root thành các đặc quyền nhỏ hơn. Container không cần toàn bộ quyền root — chỉ cần một số capability cụ thể.

### Các Capability Phổ Biến

| Capability | Mô Tả | Rủi Ro |
| ---------- | ------ | ------ |
| `NET_BIND_SERVICE` | Bind port < 1024 | Thấp — thường cần cho web server |
| `NET_ADMIN` | Cấu hình network interface, routing, firewall | **Cao** — thao tác mạng tự do |
| `NET_RAW` | Raw socket, packet sniffing | **Cao** — nghe traffic |
| `SYS_ADMIN` | Mount, namespace, cgroup, nhiều syscall admin | **Rất cao** — gần như root |
| `SYS_PTRACE` | Debug tiến trình khác | **Cao** — đọc memory tiến trình khác |
| `CAP_CHOWN` | Thay đổi file ownership | Trung bình |
| `AUDIT_WRITE` | Ghi vào kernel audit log | Thấp |

### Nguyên Tắc Áp Dụng

```yaml
# Best practice: drop ALL, thêm lại cái thực sự cần
securityContext:
  capabilities:
    drop:
      - ALL           # bỏ tất cả — bắt đầu từ điểm 0
    add:
      - NET_BIND_SERVICE   # chỉ thêm nếu app cần bind port 80/443

# KHÔNG BAO GIỜ làm:
securityContext:
  capabilities:
    add:
      - SYS_ADMIN   # gần như toàn quyền root — tránh tuyệt đối
```

### Kiểm Tra Capability Trong Container

```bash
# Xem capabilities hiện tại trong container
kubectl exec -it myapp-pod -- capsh --print

# Hoặc đọc /proc/self/status
kubectl exec -it myapp-pod -- cat /proc/self/status | grep -i cap
# CapPrm, CapEff, CapBnd, CapAmb — mỗi bit là một capability
```

---

## Seccomp Profile — Lọc System Call

**Seccomp (Secure Computing Mode — Chế Độ Tính Toán An Toàn)** lọc system call (lời gọi hệ thống) mà container được phép thực hiện. Container bình thường cần ~300 trong số ~400+ syscall. Seccomp chặn syscall không cần thiết.

### 3 Loại Seccomp Profile

```yaml
securityContext:
  seccompProfile:
    # Loại 1: RuntimeDefault — dùng profile mặc định của container runtime (containerd/crio)
    type: RuntimeDefault

    # Loại 2: Localhost — dùng profile JSON tự định nghĩa trên node
    type: Localhost
    localhostProfile: profiles/custom-profile.json

    # Loại 3: Unconfined — không lọc syscall (mặc định, không an toàn)
    type: Unconfined
```

### Custom Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "read", "write", "open", "close", "stat", "fstat",
        "poll", "lseek", "mmap", "mprotect", "munmap",
        "execve", "exit", "kill", "getpid", "getuid",
        "socket", "connect", "accept", "sendto", "recvfrom",
        "bind", "listen", "epoll_wait", "epoll_ctl", "clone",
        "futex", "nanosleep", "clock_gettime", "exit_group",
        "openat", "newfstatat", "pread64", "pwrite64"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

## AppArmor Profile

**AppArmor** là Linux Security Module (Mô-đun Bảo Mật Linux) hạn chế hành vi của tiến trình — kiểm soát file, network, capability theo profile.

```yaml
metadata:
  annotations:
    # Áp dụng AppArmor profile cho container
    container.apparmor.security.beta.kubernetes.io/webapp: runtime/default
    # Hoặc dùng profile tuỳ chỉnh đã load trên node
    container.apparmor.security.beta.kubernetes.io/webapp: localhost/my-custom-profile
```

```bash
# Kiểm tra AppArmor status trên node
aa-status

# Load profile tuỳ chỉnh
apparmor_parser -r /etc/apparmor.d/k8s-custom-profile
```

---

## Pod Security Context Toàn Diện

Ví dụ Pod đạt chuẩn **PSA Restricted**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: production
spec:
  # Pod-level security context
  securityContext:
    runAsNonRoot: true                    # bắt buộc với PSA Restricted
    runAsUser: 10000                      # non-root UID
    runAsGroup: 10000
    fsGroup: 10000
    seccompProfile:
      type: RuntimeDefault               # bắt buộc với PSA Restricted

  automountServiceAccountToken: false   # không cần K8s API

  containers:
    - name: app
      image: gcr.io/distroless/java:17  # distroless image — không có shell, ít CVE
      args: ["--config=/app/config.yaml"]

      # Container-level security context
      securityContext:
        allowPrivilegeEscalation: false  # bắt buộc với PSA Restricted
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL                        # bắt buộc với PSA Restricted

      # Resource limits — tránh DoS
      resources:
        requests:
          cpu: "100m"
          memory: "256Mi"
        limits:
          cpu: "1000m"
          memory: "512Mi"

      # Volume mounts cho writeable directories
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: app-config
          mountPath: /app/config.yaml
          subPath: config.yaml
          readOnly: true

  volumes:
    - name: tmp
      emptyDir:
        medium: Memory      # lưu trong RAM thay vì disk — nhanh hơn, an toàn hơn
        sizeLimit: 64Mi
    - name: app-config
      configMap:
        name: app-config
```

### Kiểm Tra PSA Compliance

```bash
# Kiểm tra trước khi apply
kubectl apply --dry-run=server -f pod.yaml

# Kết quả nếu vi phạm PSA Restricted:
# Warning: would violate PodSecurity "restricted:latest":
#   allowPrivilegeEscalation != false
#   unrestricted capabilities
#   runAsNonRoot != true
#   seccompProfile not set

# Audit namespace để tìm Pod vi phạm
kubectl get events -n production \
  --field-selector reason=FailedCreate | grep "violates PodSecurity"

# Kiểm tra securityContext của Pod đang chạy
kubectl get pod myapp-pod -o jsonpath='{.spec.containers[*].securityContext}'
```

---

## OPA Gatekeeper Cho Policy Nâng Cao

**OPA Gatekeeper (Open Policy Agent — Tác Nhân Chính Sách Mở)** cung cấp policy engine linh hoạt hơn PSA — dùng ngôn ngữ Rego để viết policy tuỳ chỉnh.

### Ví Dụ: Constraint Bắt Buộc Resource Limits

```yaml
# ConstraintTemplate — định nghĩa loại constraint
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireresourcelimits
spec:
  crd:
    spec:
      names:
        kind: RequireResourceLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requireresourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' phải có CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' phải có memory limit", [container.name])
        }

---
# Constraint — áp dụng ConstraintTemplate
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResourceLimits
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production", "staging"]
    excludedNamespaces: ["kube-system"]
```

### Ví Dụ: Từ Chối Container Chạy Root

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: noroot
spec:
  crd:
    spec:
      names:
        kind: NoRoot
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package noroot

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' phải có runAsNonRoot: true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.runAsUser == 0
          msg := sprintf("Container '%v' không được chạy với UID 0 (root)", [container.name])
        }
```

---

## Câu Hỏi Phỏng Vấn

**PSA (Pod Security Admission) là gì? Khác PodSecurityPolicy như thế nào?**

> PSA là admission controller tích hợp sẵn trong K8s 1.25+ kiểm tra Pod spec khi tạo. PSA có 3 cấp độ (Privileged/Baseline/Restricted) và 3 chế độ (enforce/audit/warn), áp dụng qua label namespace. So với PSP (đã bị xoá từ K8s 1.25): PSA đơn giản hơn nhiều — không cần tạo tài nguyên PSP hay cấu hình RBAC riêng, chỉ cần label namespace là xong. Nhược điểm PSA: ít linh hoạt hơn PSP — chỉ có 3 cấp cứng nhắc. Để policy tinh tế hơn cần OPA Gatekeeper hoặc Kyverno.

**`allowPrivilegeEscalation: false` có ý nghĩa gì?**

> Ngăn tiến trình trong container leo thang quyền lên cao hơn quyền cha. Cụ thể là tắt bit `setuid`/`setgid` — không cho phép thực thi file với quyền owner file (ví dụ `/bin/sudo` không thể chạy với uid 0 nếu process là uid 1000). Đây là thiết lập **cực kỳ quan trọng**: ngay cả khi container chạy với non-root user, nếu `allowPrivilegeEscalation: true` thì attacker có thể dùng setuid binary để leo thang về root.

**`readOnlyRootFilesystem: true` ảnh hưởng thế nào đến ứng dụng?**

> Filesystem gốc của container chỉ đọc — không ghi được vào `/`, `/app`, `/var/log`, v.v. Ứng dụng cần ghi file phải dùng volume riêng (emptyDir, PVC). Lợi ích bảo mật: attacker compromise container không thể chỉnh sửa binary, thêm script, hay sửa cấu hình ứng dụng. Cách áp dụng thực tế: mount `emptyDir` cho `/tmp`, `/var/log`, `/app/cache` — những thư mục ứng dụng cần ghi. Nhiều framework hiện đại hỗ trợ cấu hình log ra stdout thay vì file, giảm nhu cầu volume.

**Linux capabilities là gì? Tại sao cần `drop: ALL` trong production?**

> Linux capabilities chia quyền root thành ~40 đặc quyền nhỏ. Container mặc định có một tập capabilities cho phép, bao gồm một số cái nguy hiểm như `NET_RAW` (có thể sniff network) và `AUDIT_WRITE`. `drop: ALL` bỏ hết mọi capability — container chạy với đặc quyền thấp nhất, gần như user thường. Sau đó chỉ `add` lại cái thực sự cần, ví dụ `NET_BIND_SERVICE` cho web server cần bind port 80. Nguyên tắc: least privilege ở cấp độ kernel.
