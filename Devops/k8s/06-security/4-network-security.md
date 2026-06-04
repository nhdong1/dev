# Network Security — Bảo Mật Mạng Kubernetes

> Hướng dẫn chi tiết về bảo mật mạng Kubernetes: NetworkPolicy (Chính Sách Mạng) kiểm soát lưu lượng Layer 3/4, mTLS (mutual TLS — TLS Hai Chiều) mã hoá và xác thực Layer 7 qua Service Mesh, và Ingress TLS bảo mật kết nối từ client vào cluster.

## Mục Lục

1. [Mô Hình Mạng Mặc Định](#mô-hình-mạng-mặc-định)
2. [NetworkPolicy — Tường Lửa Pod](#networkpolicy--tường-lửa-pod)
3. [Pattern NetworkPolicy Thực Chiến](#pattern-networkpolicy-thực-chiến)
4. [mTLS — Mã Hoá Và Xác Thực Hai Chiều](#mtls--mã-hoá-và-xác-thực-hai-chiều)
5. [Ingress TLS — HTTPS Từ Client](#ingress-tls--https-từ-client)
6. [So Sánh NetworkPolicy vs mTLS](#so-sánh-networkpolicy-vs-mtls)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Mô Hình Mạng Mặc Định

**Mặc định trong Kubernetes: mọi Pod đều có thể kết nối đến mọi Pod khác** — không có firewall, không có phân vùng. Đây là flat network model (mô hình mạng phẳng) cho phép mọi workload giao tiếp tự do.

```
Namespace A                     Namespace B
┌─────────────────────────┐    ┌─────────────────────────┐
│  frontend-pod           │    │  database-pod           │
│  backend-pod    ←──── tất cả kết nối đều được ────→    │
│  cache-pod              │    │  internal-api-pod       │
└─────────────────────────┘    └─────────────────────────┘
```

**Vấn đề bảo mật:**
- `frontend-pod` có thể kết nối thẳng đến `database-pod` — không qua backend
- Pod bị compromise trong namespace A có thể tấn công mọi service trong namespace B
- Không có micro-segmentation (phân vùng vi mô) giữa các tier

---

## NetworkPolicy — Tường Lửa Pod

**NetworkPolicy** là tài nguyên Kubernetes định nghĩa rule cho phép/chặn lưu lượng mạng đến/đi từ Pod. Hoạt động ở **Layer 3/4 (IP và port)**.

> **Lưu ý quan trọng:** NetworkPolicy chỉ hoạt động khi CNI plugin hỗ trợ — Calico, Cilium, Weave Net có hỗ trợ. Flannel **không hỗ trợ** NetworkPolicy.

### Cấu Trúc NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  # Chọn Pod nào policy này áp dụng
  podSelector:
    matchLabels:
      app: backend

  # Loại traffic cần kiểm soát
  policyTypes:
    - Ingress    # traffic vào Pod
    - Egress     # traffic ra từ Pod

  # Rules cho traffic vào
  ingress:
    - from:
        # Nguồn 1: Pod có label app=frontend trong cùng namespace
        - podSelector:
            matchLabels:
              app: frontend
        # Nguồn 2: Pod từ namespace có label env=production
        - namespaceSelector:
            matchLabels:
              env: production
          podSelector:           # kết hợp: Pod từ namespace production VÀ có label app=api
            matchLabels:
              app: api
        # Nguồn 3: IP cụ thể
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.1.0.0/16      # ngoại trừ dải này
      ports:
        - protocol: TCP
          port: 8080

  # Rules cho traffic ra
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Cho phép DNS resolution
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### NetworkPolicy: AND vs OR Logic

```yaml
# OR logic — from là mảng: nguồn 1 HOẶC nguồn 2
ingress:
  - from:
      - podSelector:         # nguồn 1: Pod có label app=frontend
          matchLabels:
            app: frontend
      - namespaceSelector:   # nguồn 2 (OR): Pod từ namespace có label team=ops
          matchLabels:
            team: ops

# AND logic — podSelector VÀ namespaceSelector trong cùng một item
ingress:
  - from:
      - podSelector:         # Pod có label app=frontend
          matchLabels:
            app: frontend
        namespaceSelector:   # VÀ từ namespace có label env=production
          matchLabels:
            env: production
```

---

## Pattern NetworkPolicy Thực Chiến

### Pattern 1: Default Deny All (Từ Chối Mọi Thứ Mặc Định)

```yaml
# Áp dụng ngay khi tạo namespace — sau đó mở từng port cần thiết
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # áp dụng cho MỌI Pod trong namespace
  policyTypes:
    - Ingress
    - Egress
  # Không có ingress hay egress rules = từ chối tất cả
```

### Pattern 2: Cho Phép DNS (Bắt Buộc Sau Default Deny)

```yaml
# Sau khi default-deny, DNS cần được mở lại cho tất cả Pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Pattern 3: 3-Tier Application (Frontend → Backend → Database)

```yaml
# --- NetworkPolicy cho Frontend ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 0.0.0.0/0    # nhận traffic từ internet (qua Ingress)
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 8080
    - to: []
      ports:
        - protocol: UDP
          port: 53

---
# --- NetworkPolicy cho Backend ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend    # chỉ nhận từ frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database    # chỉ kết nối đến database
      ports:
        - protocol: TCP
          port: 5432
    - to: []
      ports:
        - protocol: UDP
          port: 53

---
# --- NetworkPolicy cho Database ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend    # chỉ nhận từ backend
      ports:
        - protocol: TCP
          port: 5432
  egress:
    - to: []
      ports:
        - protocol: UDP
          port: 53           # chỉ cần DNS
```

### Pattern 4: Cho Phép Monitoring Scrape

```yaml
# Prometheus cần scrape metrics từ mọi namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}    # áp dụng cho mọi Pod
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090    # metrics port
        - protocol: TCP
          port: 8080    # hoặc app port nếu có /metrics endpoint
```

### Pattern 5: Cô Lập Cross-Namespace

```yaml
# Chặn mọi traffic giữa namespace — chỉ cho phép trong cùng namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}    # chỉ Pod trong cùng namespace (không có namespaceSelector)
```

### Kiểm Tra NetworkPolicy

```bash
# Liệt kê NetworkPolicy trong namespace
kubectl get networkpolicies -n production

# Xem chi tiết
kubectl describe networkpolicy backend-policy -n production

# Test kết nối từ Pod này đến Pod khác
kubectl exec -it frontend-pod -n production -- \
  nc -zv database-pod 5432  # kiểm tra TCP port

kubectl exec -it frontend-pod -n production -- \
  curl -m 5 http://backend-service:8080/health

# Dùng netshoot container để debug network
kubectl run test-pod --image=nicolaka/netshoot --rm -it -- \
  ncat -zv database-service.production.svc.cluster.local 5432
```

---

## mTLS — Mã Hoá Và Xác Thực Hai Chiều

**mTLS (mutual TLS — TLS Hai Chiều)** mã hoá toàn bộ traffic giữa các service và xác thực danh tính cả hai phía (client và server), không chỉ server như TLS thông thường.

### TLS vs mTLS

```
TLS thông thường (HTTPS):
  Client ──── xác thực server (certificate) ──→ Server
  Client ←── kết nối mã hoá ─────────────────── Server
  (Server không biết client là ai)

mTLS (mutual TLS):
  Client ←── xác thực lẫn nhau (cả hai cần certificate) ──→ Server
  Client ←── kết nối mã hoá ─────────────────────────────── Server
  (Cả hai bên đều xác minh danh tính nhau)
```

### Triển Khai mTLS Qua Service Mesh

Việc tự quản lý certificate mTLS cho hàng chục service là cực kỳ phức tạp. **Service Mesh** (Istio, Linkerd) tự động hoá toàn bộ:

```
Không có Service Mesh:
  Service A ──── plaintext HTTP ──→ Service B

Với Istio (Automatic mTLS):
  Service A → [Envoy sidecar] ──── mTLS ──→ [Envoy sidecar] → Service B
              (tự quản lý cert)              (tự quản lý cert)
```

### Istio — Automatic mTLS

#### Cài Đặt Istio

```bash
# Cài Istio với istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Cài Istio vào cluster với profile production
istioctl install --set profile=production -y

# Verify cài đặt
istioctl verify-install
kubectl get pods -n istio-system
```

#### Bật Automatic mTLS Cho Namespace

```bash
# Bật sidecar injection cho namespace
kubectl label namespace production istio-injection=enabled

# Restart Deployment để inject sidecar
kubectl rollout restart deployment -n production
```

#### Cấu Hình mTLS STRICT Mode

```yaml
# PeerAuthentication — bắt buộc mTLS cho toàn namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT      # bắt buộc mTLS — từ chối plaintext

---
# Hoặc cho toàn cluster
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

#### AuthorizationPolicy — Kiểm Soát Truy Cập Layer 7

```yaml
# Chỉ cho phép frontend gọi backend qua HTTP GET /api/*
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:                    # danh tính xác thực qua mTLS
              - "cluster.local/ns/production/sa/frontend-sa"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

#### Verify mTLS

```bash
# Kiểm tra mTLS status
istioctl authn tls-check myapp-pod.production

# Xem traffic trong mesh
kubectl exec -it myapp-pod -c istio-proxy -- \
  pilot-agent request GET stats | grep ssl

# Kiểm tra certificate của sidecar
istioctl proxy-config secret myapp-pod.production
```

### Linkerd — Simpler mTLS

```bash
# Cài Linkerd (nhẹ hơn Istio, dễ dùng hơn)
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -
linkerd check

# Inject vào namespace
kubectl annotate namespace production linkerd.io/inject=enabled

# Kiểm tra mTLS
linkerd viz tap deployment/backend -n production
linkerd viz edges deployment -n production
```

---

## Ingress TLS — HTTPS Từ Client

### TLS Termination Tại Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # Redirect HTTP → HTTPS
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls           # Secret type: kubernetes.io/tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

### Cert-Manager — Quản Lý Certificate Tự Động

**cert-manager** tự động cấp và gia hạn TLS certificate từ Let's Encrypt hoặc CA nội bộ.

```bash
# Cài cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager
```

```yaml
# ClusterIssuer — Let's Encrypt production
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx

---
# Certificate — yêu cầu cert cho domain
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: production
spec:
  secretName: myapp-tls                   # cert-manager tạo Secret này
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
    - www.myapp.example.com
  duration: 2160h                          # 90 ngày
  renewBefore: 360h                        # gia hạn trước 15 ngày khi hết hạn
```

```yaml
# Ingress với cert-manager annotation — tự động tạo Certificate
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # annotation này trigger cert-manager
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls               # cert-manager tự tạo Secret này
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

```bash
# Kiểm tra certificate status
kubectl get certificate -n production
kubectl describe certificate myapp-tls -n production

# Kiểm tra certificate secret
kubectl get secret myapp-tls -n production
openssl x509 -in <(kubectl get secret myapp-tls -n production \
  -o jsonpath='{.data.tls\.crt}' | base64 -d) -noout -dates
```

### TLS Passthrough — Không Decrypt Tại Ingress

```yaml
# Traffic HTTPS đi thẳng đến backend, Ingress không decrypt
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: passthrough-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
    - host: secure-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-app    # backend tự xử lý TLS
                port:
                  number: 443
```

---

## So Sánh NetworkPolicy vs mTLS

| Tiêu Chí | NetworkPolicy | mTLS (Service Mesh) |
| -------- | ------------- | ------------------- |
| **Layer** | L3/L4 (IP, port) | L7 (application) |
| **Kiểm soát** | Allowed/Denied connection | Authenticated + Encrypted |
| **Xác thực danh tính** | Không (chỉ IP/label) | ✅ Có (certificate-based) |
| **Mã hoá traffic** | Không | ✅ Có (TLS) |
| **Overhead** | Rất thấp (kernel iptables/eBPF) | Có overhead (sidecar proxy) |
| **Phụ thuộc** | CNI plugin hỗ trợ | Service mesh (Istio/Linkerd) |
| **Độ phức tạp** | Thấp | Cao (nhiều component) |
| **Quan sát được** | Không (network drop) | ✅ Có (telemetry, tracing) |
| **Phù hợp** | Micro-segmentation cơ bản | Zero-trust network, compliance |

**Kết luận:** Dùng cả hai — NetworkPolicy là lớp nền bảo vệ, mTLS là lớp xác thực và mã hoá. Không phải hoặc-hoặc.

---

## Câu Hỏi Phỏng Vấn

**Tại sao cần NetworkPolicy? Kubernetes không có firewall mặc định à?**

> Đúng vậy — Kubernetes dùng **flat network model**: mọi Pod đều có thể kết nối đến mọi Pod khác mặc định. Không có phân vùng mạng giữa namespace, không có tier separation. Đây là lựa chọn thiết kế ưu tiên đơn giản và linh hoạt, nhưng không an toàn cho production. NetworkPolicy là cách khai báo micro-segmentation: chỉ cho phép kết nối cần thiết, từ chối mọi thứ còn lại (default-deny).

**NetworkPolicy `podSelector: {}` có nghĩa gì?**

> `podSelector: {}` nghĩa là **chọn tất cả Pod** trong namespace — không lọc theo label. Kết hợp với `policyTypes: [Ingress, Egress]` và không có rules, đây là **default-deny-all** policy: từ chối mọi traffic vào và ra từ tất cả Pod trong namespace. Đây là pattern bảo mật tốt nhất: tạo default-deny trước, sau đó mở từng kết nối cụ thể cần thiết.

**mTLS khác TLS thông thường như thế nào?**

> TLS thông thường: client xác minh server certificate (client biết nó đang nói chuyện với server hợp lệ), server không xác minh client. mTLS (mutual): cả hai bên xác minh certificate lẫn nhau — server biết chắc chắn client là service đã được phép, không phải attacker bất kỳ. Trong microservices, mTLS đảm bảo: (1) Traffic mã hoá end-to-end; (2) Service A biết chắc nó đang gọi Service B thật, không phải service giả; (3) Service B biết chắc request đến từ Service A hợp lệ, không phải attacker compromise mạng.

**Istio và Linkerd khác nhau thế nào?**

> Cả hai đều là Service Mesh cung cấp mTLS tự động, traffic management, observability. Sự khác nhau chính: **Istio** dùng Envoy proxy làm sidecar — tính năng rất phong phú (circuit breaker, rate limiting, JWT auth, WebAssembly extension) nhưng phức tạp và tốn tài nguyên hơn. **Linkerd** dùng proxy nhỏ viết bằng Rust — đơn giản hơn, nhẹ hơn, dễ vận hành hơn, nhưng ít tính năng nâng cao hơn. Với cluster nhỏ/vừa cần mTLS đơn giản: Linkerd. Với cluster lớn cần full traffic policy, AuthorizationPolicy, external auth: Istio.
