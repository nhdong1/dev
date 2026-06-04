# Networking Debug — Xử Lý Sự Cố Mạng Kubernetes

> Hướng dẫn chẩn đoán và khắc phục sự cố mạng trong Kubernetes: Service không kết nối được, DNS không phân giải, Ingress lỗi 5xx, và Pod không ping được nhau.

## Mục Lục

1. [Mô Hình Mạng Kubernetes — Cần Nắm Vững](#mô-hình-mạng-kubernetes)
2. [Công Cụ Debug Mạng](#công-cụ-debug-mạng)
3. [Service Không Kết Nối Được](#service-không-kết-nối-được)
4. [DNS Không Phân Giải (DNS Fail)](#dns-không-phân-giải)
5. [Ingress Lỗi 502 / 503 / 504](#ingress-lỗi)
6. [Pod Không Ping Được Pod Khác](#pod-không-ping-được-pod-khác)
7. [NetworkPolicy Chặn Lưu Lượng](#networkpolicy-chặn-lưu-lượng)
8. [Debug Nâng Cao](#debug-nâng-cao)

---

## Mô Hình Mạng Kubernetes

### Ba Nguyên Tắc Cơ Bản

```
1. Mỗi Pod có IP riêng (Pod IP) — không share IP với Pod khác
2. Pod trên cùng node hoặc khác node đều có thể giao tiếp trực tiếp (flat network)
3. NAT (Network Address Translation) không xảy ra khi Pod giao tiếp với nhau
```

### Luồng Kết Nối Điển Hình

```
Client bên ngoài
  → DNS phân giải domain
  → LoadBalancer (cloud LB hoặc NodePort)
  → Ingress Controller (NGINX, Traefik, ALB...)
  → Service (ClusterIP — Virtual IP)
  → kube-proxy chọn Endpoint ngẫu nhiên
  → Pod IP:ContainerPort
  → Ứng dụng trong Container
```

### DNS Naming Convention (Quy Tắc Đặt Tên DNS)

```bash
# Format đầy đủ
<service-name>.<namespace>.svc.<cluster-domain>

# Ví dụ (cluster domain mặc định là cluster.local)
my-service.my-namespace.svc.cluster.local

# Trong cùng namespace — chỉ cần tên service
my-service

# Khác namespace — cần thêm namespace
my-service.other-namespace
```

---

## Công Cụ Debug Mạng

### Pod Debug All-in-One

```bash
# nicolaka/netshoot — chứa đầy đủ công cụ debug mạng
kubectl run netshoot --rm -it \
  --image=nicolaka/netshoot \
  --namespace=<target-namespace> \
  -- bash

# Trong netshoot có sẵn: curl, dig, nslookup, ping, traceroute, tcpdump,
#   netstat, ss, iptables, wget, telnet, nc (netcat)

# Tạo netshoot trong namespace cần debug
kubectl run netshoot -n production --rm -it --image=nicolaka/netshoot -- bash
```

### Lệnh Debug Mạng Cốt Lõi

```bash
# Kiểm tra Service và Endpoints
kubectl get svc -n <ns>
kubectl get endpoints <svc-name> -n <ns>

# Kiểm tra DNS từ trong cluster
kubectl run dns-test --rm -it --image=busybox -- nslookup <service-name>.<ns>

# Kiểm tra kết nối HTTP
kubectl run curl-test --rm -it --image=curlimages/curl -- \
  curl -v http://<service-name>.<ns>.svc.cluster.local:<port>

# Xem iptables rules (chạy trên node)
iptables -t nat -L KUBE-SERVICES | grep <service-name>

# Xem IPVS rules (nếu kube-proxy dùng IPVS mode)
ipvsadm -Ln | grep <service-cluster-ip>
```

---

## Service Không Kết Nối Được

### Quy Trình Debug Service

```
Service unreachable?
│
├── 1. Service có tồn tại không?
├── 2. Endpoints có rỗng không?
├── 3. Selector có match với Pod label không?
├── 4. Port có đúng không?
├── 5. NetworkPolicy có chặn không?
└── 6. kube-proxy có chạy đúng không?
```

### Bước 1: Kiểm Tra Service Tồn Tại

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>

# Xem ClusterIP và port mapping
# Ví dụ output:
# Name:      my-service
# Namespace: default
# Type:      ClusterIP
# IP:        10.96.45.23        ← ClusterIP (virtual IP)
# Port:      http  80/TCP
# TargetPort: 8080/TCP          ← Port thực của container
# Endpoints:  10.244.1.5:8080,10.244.2.7:8080  ← Pod IPs
```

### Bước 2: Kiểm Tra Endpoints — Nguyên Nhân Phổ Biến Nhất

```bash
kubectl get endpoints <service-name> -n <namespace>

# Trường hợp bình thường:
# NAME         ENDPOINTS                       AGE
# my-service   10.244.1.5:8080,10.244.2.7:8080  5m

# Trường hợp bị lỗi (endpoints rỗng):
# NAME         ENDPOINTS   AGE
# my-service   <none>      5m
```

**Endpoints rỗng = selector không match với Pod nào.**

### Bước 3: So Sánh Selector Với Pod Label

```bash
# Xem selector của Service
kubectl get svc <name> -o jsonpath='{.spec.selector}'
# Ví dụ: {"app":"my-app","version":"v2"}

# Xem label của Pod
kubectl get pods -n <ns> --show-labels | grep my-app

# Nếu không match → sửa label Pod hoặc selector Service
kubectl label pod <pod-name> version=v2 -n <ns>
```

### Bước 4: Kiểm Tra Port Mapping

```bash
kubectl describe svc <name> -n <ns>
# Port: 80/TCP  ← Exposed port của Service
# TargetPort: 8080/TCP  ← Port container đang lắng nghe

kubectl describe pod <pod-name> | grep "Port:"
# Phải khớp với targetPort của Service
```

### Bước 5: Test Kết Nối Trực Tiếp

```bash
# Test từ bên trong cluster
kubectl run test --rm -it --image=curlimages/curl -n <ns> -- \
  curl -v http://<service-name>.<ns>.svc.cluster.local

# Test port-forward (bỏ qua Service, kết nối thẳng vào Pod)
kubectl port-forward pod/<pod-name> 8080:8080 -n <ns>
# Sau đó: curl http://localhost:8080

# Test kết nối trực tiếp đến Pod IP
kubectl get pod <name> -o jsonpath='{.status.podIP}'
kubectl run test --rm -it --image=curlimages/curl -- curl http://<pod-ip>:8080
```

### Lỗi Phổ Biến Và Cách Sửa

| Triệu Chứng                    | Nguyên Nhân               | Cách Sửa                              |
| ------------------------------ | ------------------------- | ------------------------------------- |
| `Endpoints: <none>`            | Selector không match      | Sửa label Pod hoặc selector Service   |
| Connection refused             | TargetPort sai            | Sửa targetPort khớp với containerPort |
| Connection timeout             | NetworkPolicy chặn        | Xem NetworkPolicy                     |
| `service not found`            | Namespace sai             | Thêm namespace vào DNS query          |
| Một số request fail, một số OK | Pod không healthy         | Kiểm tra readiness probe              |

---

## DNS Không Phân Giải

### Kiến Trúc DNS Trong Kubernetes

```
Pod gửi DNS query
  → kube-dns (ClusterIP 10.96.0.10 mặc định)
  → CoreDNS Pod (thường 2 replica trong kube-system)
  → Phân giải nội bộ (*.svc.cluster.local) hoặc forward ra ngoài
```

### Kiểm Tra CoreDNS

```bash
# CoreDNS có đang chạy không?
kubectl get pods -n kube-system | grep coredns

# Xem log CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Xem ConfigMap của CoreDNS
kubectl get configmap coredns -n kube-system -o yaml
```

### Debug DNS Từ Trong Pod

```bash
# Chạy Pod debug trong namespace cần test
kubectl run dns-test -n <namespace> --rm -it --image=busybox -- sh

# Trong Pod:
nslookup kubernetes.default.svc.cluster.local   # Test DNS cơ bản
nslookup <service-name>.<namespace>.svc.cluster.local
cat /etc/resolv.conf                              # Xem DNS config của Pod
nslookup google.com                               # Test DNS ra ngoài internet
```

### Nguyên Nhân Phổ Biến

#### 1. CoreDNS Pod Crash

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system <coredns-pod> --previous
```

**Cách sửa:** Restart CoreDNS.

```bash
kubectl rollout restart deployment/coredns -n kube-system
```

#### 2. /etc/resolv.conf Trong Pod Sai

```bash
# Kiểm tra từ trong Pod
cat /etc/resolv.conf
# Phải có dòng:
# nameserver 10.96.0.10       ← ClusterIP của kube-dns
# search default.svc.cluster.local svc.cluster.local cluster.local

# Kiểm tra ClusterIP của kube-dns
kubectl get svc kube-dns -n kube-system
```

#### 3. NetworkPolicy Chặn DNS Query

```bash
# DNS dùng UDP port 53
# Kiểm tra NetworkPolicy có cho phép egress đến port 53 không
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>
```

**Cách sửa:** Thêm egress rule cho DNS.

```yaml
egress:
  - ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

#### 4. ndots Configuration

```bash
# /etc/resolv.conf trong Pod thường có
options ndots:5

# Nghĩa là: tên miền < 5 dấu chấm sẽ được thử với search domain trước
# Ví dụ: "my-service" → thử "my-service.namespace.svc.cluster.local" trước
# Nếu ndots=5 mà dùng FQDN (fully qualified) thì thêm dấu chấm ở cuối
curl http://my-service.default.svc.cluster.local./   # FQDN với dấu chấm
```

---

## Ingress Lỗi

### Kiến Trúc Ingress

```
Client → DNS → LoadBalancer IP
  → Ingress Controller Pod (NGINX / Traefik / ALB)
  → Routing rules từ Ingress resource
  → Service (ClusterIP)
  → Pod
```

### Quy Trình Debug Ingress

```bash
# Bước 1: Kiểm tra Ingress resource
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>

# Bước 2: Kiểm tra Ingress Controller có chạy không
kubectl get pods -n ingress-nginx  # hoặc namespace của ingress controller
kubectl logs -n ingress-nginx <controller-pod> | tail -50

# Bước 3: Kiểm tra Service backend
kubectl get svc <backend-service> -n <namespace>
kubectl get endpoints <backend-service> -n <namespace>

# Bước 4: Test trực tiếp với curl, thêm header Host
curl -H "Host: myapp.example.com" http://<ingress-controller-ip>

# Bước 5: Kiểm tra certificate (nếu HTTPS)
echo | openssl s_client -connect myapp.example.com:443 -servername myapp.example.com 2>/dev/null | openssl x509 -noout -dates
```

### Lỗi Phổ Biến

#### 502 Bad Gateway

**Nguyên nhân:** Backend Pod không phản hồi hoặc bị crash.

```bash
# Kiểm tra backend Pod
kubectl get pods -n <ns> -l <selector>
kubectl logs <backend-pod> -n <ns>

# Kiểm tra Ingress Controller log
kubectl logs -n ingress-nginx <pod-name> | grep "502\|error"
```

#### 503 Service Unavailable

**Nguyên nhân:** Không có endpoint nào available (tất cả Pod unhealthy hoặc endpoint rỗng).

```bash
kubectl get endpoints <service> -n <ns>
# Nếu <none> → không có Pod nào healthy
```

#### 504 Gateway Timeout

**Nguyên nhân:** Backend xử lý quá lâu, vượt quá timeout của Ingress Controller.

```bash
# Tăng timeout cho NGINX Ingress
kubectl annotate ingress <name> -n <ns> \
  nginx.ingress.kubernetes.io/proxy-read-timeout="600" \
  nginx.ingress.kubernetes.io/proxy-send-timeout="600"
```

#### 404 Not Found

**Nguyên nhân:** Path rule không match hoặc pathType sai.

```bash
kubectl describe ingress <name> -n <ns>
# Xem Rules section:
# Host             Path  Backends
# myapp.example.com
#                  /api  my-service:80
# Kiểm tra pathType: Prefix / Exact / ImplementationSpecific
```

#### TLS Certificate Error

```bash
# Kiểm tra Secret chứa TLS cert
kubectl get secret <tls-secret> -n <ns>

# Giải mã và kiểm tra cert
kubectl get secret <tls-secret> -n <ns> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text

# Với cert-manager — kiểm tra Certificate object
kubectl get certificate -n <ns>
kubectl describe certificate <name> -n <ns>
```

### Cấu Hình Ingress Mẫu

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

---

## Pod Không Ping Được Pod Khác

### Quy Trình Debug Pod-to-Pod Connectivity

```bash
# Bước 1: Lấy IP của Pod đích
kubectl get pod <target-pod> -n <ns> -o jsonpath='{.status.podIP}'

# Bước 2: Từ Pod nguồn, thử ping và curl
kubectl exec -it <source-pod> -n <ns> -- sh
# Trong shell:
ping <target-pod-ip>
curl http://<target-pod-ip>:<port>
telnet <target-pod-ip> <port>

# Bước 3: Kiểm tra Pod trên cùng node hay khác node
kubectl get pods -o wide -n <ns>

# Bước 4: Kiểm tra CNI (Container Network Interface) plugin
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium|weave"
kubectl logs -n kube-system <cni-pod> | tail -30
```

### Nguyên Nhân Phổ Biến

#### 1. CNI Plugin Không Hoạt Động

```bash
# Kiểm tra CNI pods
kubectl get pods -n kube-system -l k8s-app=calico-node  # Calico
kubectl get pods -n kube-system -l app=flannel           # Flannel
kubectl get pods -n kube-system -l k8s-app=cilium        # Cilium

# Nếu CNI pod crash, xem log
kubectl logs -n kube-system <cni-pod> --previous
```

#### 2. NetworkPolicy Chặn

```bash
# Kiểm tra NetworkPolicy trong namespace
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>

# Nếu có NetworkPolicy, phải thêm explicit allow rule
# NetworkPolicy mặc định deny all khi được áp dụng
```

#### 3. iptables Rules Bị Corrupt

```bash
# Chạy trên node (cần quyền root)
iptables -L -n | grep -E "DROP|REJECT" | head -20

# Với Calico — kiểm tra Felix logs
kubectl logs -n kube-system <calico-node-pod> -c calico-node | grep -i "error\|warn"
```

---

## NetworkPolicy Chặn Lưu Lượng

### Nguyên Tắc Cơ Bản

```
- Mặc định: không có NetworkPolicy → tất cả traffic được phép
- Khi có NetworkPolicy: chỉ traffic khớp với rule mới được phép
- NetworkPolicy là additive (cộng dồn) — nhiều policy cùng áp dụng cho một Pod
- Dùng podSelector rỗng {} để áp dụng cho tất cả Pod trong namespace
```

### Debug NetworkPolicy

```bash
# Xem tất cả NetworkPolicy
kubectl get networkpolicy -A

# Xem chi tiết
kubectl describe networkpolicy <name> -n <namespace>

# Test kết nối để xác nhận
kubectl run test-source -n <source-ns> --rm -it --image=curlimages/curl -- \
  curl http://<target-service>.<target-ns>.svc.cluster.local
```

### NetworkPolicy Allow All Tạm Thời (Để Debug)

```yaml
# Tạm thời cho phép tất cả ingress và egress để kiểm tra
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-debug
  namespace: my-namespace
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - {}
  egress:
    - {}
```

### NetworkPolicy Phổ Biến — Deny All + Allow Cụ Thể

```yaml
# Bước 1: Deny tất cả ingress và egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Bước 2: Allow cụ thể
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres       # Áp dụng cho Pod postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api    # Chỉ cho phép từ Pod api
      ports:
        - protocol: TCP
          port: 5432
```

---

## Debug Nâng Cao

### Packet Capture (Bắt Gói Tin) Với tcpdump

```bash
# Cài tcpdump trên node (nếu chưa có)
# Hoặc dùng netshoot với --privileged

kubectl run tcpdump-debug \
  --image=nicolaka/netshoot \
  --privileged \
  --rm -it -- tcpdump -i eth0 -n port 80

# Bắt traffic đến Pod cụ thể
kubectl debug node/<node-name> -it --image=nicolaka/netshoot -- \
  tcpdump -i any host <pod-ip> -n
```

### Xem iptables Rules Liên Quan Đến Service

```bash
# SSH vào node
ssh <node-ip>

# Xem tất cả rules liên quan đến một Service ClusterIP
CLUSTER_IP=$(kubectl get svc <name> -n <ns> -o jsonpath='{.spec.clusterIP}')
iptables -t nat -L -n | grep $CLUSTER_IP
```

### Kiểm Tra kube-proxy

```bash
# kube-proxy có chạy không?
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Log kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy | tail -30

# Mode kube-proxy đang dùng (iptables hay ipvs)
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```

---

## Tóm Tắt Lệnh Debug Mạng

```bash
# === SERVICE ===
kubectl get svc -n <ns>
kubectl get endpoints <svc> -n <ns>
kubectl describe svc <svc> -n <ns>

# === DNS ===
kubectl run dns --rm -it --image=busybox -- nslookup <service>.<ns>.svc.cluster.local
kubectl logs -n kube-system -l k8s-app=kube-dns

# === INGRESS ===
kubectl describe ingress <name> -n <ns>
kubectl logs -n ingress-nginx <controller-pod> | grep -i error

# === CONNECTIVITY ===
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
# → curl, ping, dig, traceroute, tcpdump...

# === NETWORKPOLICY ===
kubectl get networkpolicy -n <ns>
kubectl describe networkpolicy <name> -n <ns>

# === PORT FORWARD (bypass Service để test trực tiếp Pod) ===
kubectl port-forward pod/<name> 8080:8080 -n <ns>
```

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
