# NetworkPolicy — Kiểm Soát Lưu Lượng Mạng Giữa Các Pod

> NetworkPolicy là tài nguyên Kubernetes kiểm soát lưu lượng mạng vào/ra của Pod ở tầng L3/L4. Mặc định, mọi Pod trong cluster có thể giao tiếp tự do — NetworkPolicy thêm rào cản để cô lập và bảo vệ workload.

## Mục Lục

1. [NetworkPolicy Là Gì?](#networkpolicy-là-gì)
2. [Mô Hình Mặc Định — Cho Phép Tất Cả](#mô-hình-mặc-định--cho-phép-tất-cả)
3. [Cấu Trúc NetworkPolicy](#cấu-trúc-networkpolicy)
4. [Ingress Policy — Kiểm Soát Lưu Lượng Vào](#ingress-policy--kiểm-soát-lưu-lượng-vào)
5. [Egress Policy — Kiểm Soát Lưu Lượng Ra](#egress-policy--kiểm-soát-lưu-lượng-ra)
6. [Selector — Chọn Pod Và Nguồn/Đích](#selector--chọn-pod-và-nguồnđích)
7. [Pattern Thực Chiến](#pattern-thực-chiến)
8. [Debug NetworkPolicy](#debug-networkpolicy)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## NetworkPolicy Là Gì?

**NetworkPolicy** là firewall (tường lửa) nội bộ của Kubernetes hoạt động ở tầng mạng (L3 — IP, L4 — TCP/UDP port). Nó xác định:

- Pod nào có thể **nhận** traffic từ đâu (ingress — lưu lượng vào)
- Pod nào có thể **gửi** traffic đến đâu (egress — lưu lượng ra)

> NetworkPolicy chỉ có tác dụng nếu **CNI plugin hỗ trợ** thực thi nó:
> - ✅ Calico, Cilium, Weave, Antrea — hỗ trợ NetworkPolicy
> - ❌ Flannel — **không** hỗ trợ NetworkPolicy (cần cài thêm Calico hoặc Cilium bên cạnh)

### NetworkPolicy Không Phải Là

- **Không phải L7 firewall** — không đọc HTTP path, header, body
- **Không phải authentication/authorization** — không thay thế RBAC hay JWT
- **Không có stateful inspection** — không kiểm tra nội dung gói tin
- **Không encrypt traffic** — muốn encrypt cần mTLS (service mesh)

---

## Mô Hình Mặc Định — Cho Phép Tất Cả

Khi **không có** NetworkPolicy nào áp dụng cho một Pod, Pod đó cho phép tất cả traffic vào và ra.

```
Namespace A                Namespace B
┌─────────┐               ┌─────────┐
│  Pod A  │ ←───────────→ │  Pod B  │   ✅ thông nhau
└─────────┘               └─────────┘

┌─────────┐               ┌─────────┐
│  Pod A  │ ←───────────→ │  Pod C  │   ✅ thông nhau
└─────────┘               └─────────┘
```

Đây là thiết lập nguy hiểm trong production: nếu một Pod bị tấn công, attacker có thể lateral move (di chuyển ngang) sang mọi Pod khác trong cluster.

---

## Cấu Trúc NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-network-policy
  namespace: production          # NetworkPolicy chỉ áp dụng trong namespace này
spec:
  podSelector:                   # chọn Pod nào bị ảnh hưởng bởi policy này
    matchLabels:
      app: api
  policyTypes:                   # loại traffic cần kiểm soát
    - Ingress                    # kiểm soát traffic VÀO Pod
    - Egress                     # kiểm soát traffic RA khỏi Pod
  ingress:                       # rules cho traffic vào (chỉ cần nếu Ingress trong policyTypes)
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:                        # rules cho traffic ra (chỉ cần nếu Egress trong policyTypes)
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

### Nguyên Tắc Quan Trọng

1. **Additive** (cộng dồn): nhiều NetworkPolicy áp dụng cho cùng một Pod thì **union** (hợp) các rules
2. **Whitelist model** (danh sách trắng): khi Pod bị ít nhất một NetworkPolicy chọn, chỉ traffic khớp rule mới được phép — còn lại bị từ chối
3. **Stateful**: nếu ingress từ A → B được phép, response từ B → A tự động được phép (không cần egress rule riêng)

---

## Ingress Policy — Kiểm Soát Lưu Lượng Vào

### Ví Dụ: Chỉ Cho Frontend Gọi API

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api            # áp dụng cho Pod có label app=api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # chỉ frontend được gọi vào
      ports:
        - protocol: TCP
          port: 8080
```

Kết quả:

```
frontend Pod → api Pod :8080    ✅ được phép
database Pod → api Pod :8080   ❌ bị từ chối
Ingress Controller → api Pod   ❌ bị từ chối (chưa có rule)
```

### Ví Dụ: Cho Phép Ingress Controller Và Frontend

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: frontend
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: ingress-nginx   # từ namespace ingress-nginx
        podSelector:
          matchLabels:
            app.kubernetes.io/name: ingress-nginx
    ports:
      - protocol: TCP
        port: 8080
```

---

## Egress Policy — Kiểm Soát Lưu Lượng Ra

### Ví Dụ: API Chỉ Được Gọi Database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:                        # cho phép gọi DNS (bắt buộc phải có!)
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

> **Lưu ý quan trọng:** Khi chặn Egress, **phải** thêm rule cho phép DNS (port 53 UDP/TCP đến kube-dns). Nếu không, Pod sẽ không phân giải được tên miền bất kỳ và bị lỗi.

### Cho Phép Gọi Ra Internet

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0        # cho phép mọi IP
          except:
            - 10.0.0.0/8         # trừ IP nội bộ cluster
            - 172.16.0.0/12
            - 192.168.0.0/16
```

---

## Selector — Chọn Pod Và Nguồn/Đích

### podSelector — Chọn Theo Label Pod

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
        env: production          # cần cả hai label
```

### namespaceSelector — Chọn Theo Namespace

```yaml
from:
  - namespaceSelector:
      matchLabels:
        environment: production  # label trên Namespace object
```

### Kết Hợp podSelector VÀ namespaceSelector (AND — Và)

```yaml
from:
  - namespaceSelector:           # một phần tử trong danh sách "from"
      matchLabels:
        environment: staging
    podSelector:                 # cùng phần tử → AND: namespace staging VÀ pod có label này
      matchLabels:
        app: test-runner
```

### Kết Hợp podSelector HOẶC namespaceSelector (OR — Hoặc)

```yaml
from:
  - namespaceSelector:           # phần tử 1
      matchLabels:
        environment: staging
  - podSelector:                 # phần tử 2 (riêng biệt → OR)
      matchLabels:
        app: monitoring
```

### ipBlock — Chọn Theo CIDR

```yaml
from:
  - ipBlock:
      cidr: 203.0.113.0/24      # range IP bên ngoài (ví dụ: văn phòng công ty)
      except:
        - 203.0.113.128/25      # trừ range con này
```

---

## Pattern Thực Chiến

### Pattern 1: Default Deny All (Chặn Tất Cả Mặc Định)

Áp dụng cho mọi namespace production. Sau đó mở từng kết nối cần thiết.

```yaml
# Chặn tất cả Ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}              # {} = áp dụng cho TẤT CẢ Pod trong namespace
  policyTypes:
    - Ingress
---
# Chặn tất cả Egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Pattern 2: Allow Same Namespace (Cho Phép Trong Cùng Namespace)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}      # Pod nào trong cùng namespace đều được
```

### Pattern 3: Database Isolation (Cô Lập Database)

```yaml
# Chỉ API service được kết nối vào database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 5432
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api         # postgres chỉ trả về cho api (replication traffic)
      ports:
        - protocol: TCP
          port: 5432
```

### Pattern 4: Monitoring Access (Cho Prometheus Scrape)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api                 # policy cho Pod được scrape
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9090           # metrics port
```

### Pattern 5: Multi-Tier Architecture (Kiến Trúc Nhiều Tầng)

```
Internet → Ingress Controller → Frontend (3000)
                                Frontend → API (8080)
                                API → Cache Redis (6379)
                                API → Database (5432)
```

```yaml
# Frontend chỉ nhận từ Ingress Controller
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
---
# API nhận từ Frontend, gọi ra Redis và Postgres
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: cache
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - protocol: TCP
          port: 5432
    - to:                     # DNS
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## Debug NetworkPolicy

### Kiểm Tra Policy Đang Áp Dụng Cho Pod

```bash
# Xem tất cả NetworkPolicy trong namespace
kubectl get networkpolicy -n production

# Xem chi tiết một policy
kubectl describe networkpolicy allow-frontend-to-api -n production

# Xem label của Pod để đối chiếu selector
kubectl get pods -n production --show-labels
```

### Test Kết Nối Từ Pod

```bash
# Tạo Pod test tạm thời trong namespace nguồn
kubectl run -n production -it --rm nettest \
  --image=nicolaka/netshoot \
  --restart=Never -- bash

# Trong netshoot, thử kết nối:
curl http://api-service:8080/health       # qua Service
curl http://10.244.1.5:8080/health        # qua Pod IP trực tiếp
nc -zv postgres-service 5432              # kiểm tra TCP port
nslookup api-service                      # kiểm tra DNS
```

### Kiểm Tra Với Cilium (Nếu Dùng)

```bash
# Cilium cung cấp CLI để debug NetworkPolicy
cilium policy trace \
  --src-k8s-pod production/frontend-pod \
  --dst-k8s-pod production/api-pod \
  --dport 8080

# Kết quả cho thấy policy nào match và quyết định allow/deny
```

---

## Câu Hỏi Phỏng Vấn

**Tại sao cần NetworkPolicy nếu đã có RBAC?**

> RBAC kiểm soát **quyền truy cập Kubernetes API** (ai có thể tạo Pod, xem Secret…). NetworkPolicy kiểm soát **lưu lượng mạng TCP/UDP giữa Pod** — hoàn toàn khác nhau. Một Pod bị compromise có thể không có RBAC permission nhưng vẫn mở kết nối mạng sang Pod khác nếu không có NetworkPolicy.

**Điều gì xảy ra khi hai NetworkPolicy cùng áp dụng cho một Pod?**

> Các rule được **union** (hợp): traffic được phép nếu khớp bất kỳ rule nào trong bất kỳ policy nào. Không có rule nào có thể "override deny" rule từ policy khác — chỉ có allow, không có explicit deny trong từng rule.

**Tại sao sau khi thêm Egress policy, ứng dụng bị lỗi DNS?**

> Khi áp dụng Egress policy, tất cả egress bị chặn mặc định, bao gồm cả DNS (port 53 UDP/TCP). Phải thêm rule egress cho phép kết nối đến CoreDNS (`kube-dns` trong namespace `kube-system`). Đây là lỗi phổ biến nhất khi dùng NetworkPolicy.

**NetworkPolicy có block traffic từ node đến Pod không?**

> Không. NetworkPolicy chỉ áp dụng cho traffic **Pod-to-Pod** và traffic **vào/ra khỏi Pod**. Traffic từ node host network đến Pod không bị ảnh hưởng bởi NetworkPolicy (đây là hành vi thiết kế, không phải bug). Nếu cần kiểm soát traffic từ node, cần dùng iptables trực tiếp hoặc các cơ chế khác.
