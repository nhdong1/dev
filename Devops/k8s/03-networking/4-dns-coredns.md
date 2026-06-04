# DNS và CoreDNS — Khám Phá Dịch Vụ Nội Bộ Cluster

> CoreDNS là DNS server mặc định trong Kubernetes, cho phép Pod tìm kiếm Service và Pod khác bằng tên thay vì địa chỉ IP. Đây là nền tảng của service discovery (khám phá dịch vụ) trong cluster.

## Mục Lục

1. [Tại Sao Cần DNS Trong Kubernetes?](#tại-sao-cần-dns-trong-kubernetes)
2. [CoreDNS — DNS Server Mặc Định](#coredns--dns-server-mặc-định)
3. [Cấu Trúc Tên DNS Trong Cluster](#cấu-trúc-tên-dns-trong-cluster)
4. [DNS Cho Service](#dns-cho-service)
5. [DNS Cho Pod](#dns-cho-pod)
6. [Search Domain và NDOTS](#search-domain-và-ndots)
7. [Cấu Hình CoreDNS](#cấu-hình-coredns)
8. [Custom DNS và External DNS](#custom-dns-và-external-dns)
9. [Debug DNS](#debug-dns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần DNS Trong Kubernetes?

IP của Pod thay đổi liên tục (khi Pod restart, deploy mới, scale). Service ClusterIP ổn định hơn nhưng vẫn khó nhớ và khó cấu hình hardcode (cấu hình cứng) trong code.

**DNS giải quyết:** Thay vì code `http://10.96.45.22`, bạn code `http://api-service` — CoreDNS phân giải tên thành ClusterIP tại runtime (thời điểm chạy).

```python
# Không tốt — hardcode IP, thay đổi khi redeploy
DB_HOST = "10.96.45.22"

# Tốt — dùng Service name, CoreDNS phân giải
DB_HOST = "postgres-service"
DB_HOST = "postgres-service.production.svc.cluster.local"  # đầy đủ namespace
```

---

## CoreDNS — DNS Server Mặc Định

**CoreDNS** thay thế kube-dns từ Kubernetes 1.13. Nó chạy như một Deployment trong namespace `kube-system`.

### Kiến Trúc CoreDNS

```
Pod muốn phân giải "api-service"
    │
    ▼ query DNS (UDP/TCP port 53)
CoreDNS Pod (ClusterIP: 10.96.0.10)
    │
    ├── Tìm trong zone "cluster.local" → trả ClusterIP của Service
    │
    └── Không tìm thấy → forward (chuyển tiếp) đến DNS upstream
        (thường là DNS của node hoặc 8.8.8.8)
```

### Xem CoreDNS Đang Chạy

```bash
# Xem CoreDNS Pod
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Xem Service của CoreDNS
kubectl get service -n kube-system kube-dns

# Kết quả mẫu:
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   30d
```

### DNS Server Của Pod

Mỗi Pod được cấu hình tự động để dùng CoreDNS:

```bash
# Xem cấu hình DNS trong Pod
cat /etc/resolv.conf

# Kết quả điển hình:
# nameserver 10.96.0.10           ← IP của CoreDNS Service
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

---

## Cấu Trúc Tên DNS Trong Cluster

### Fully Qualified Domain Name (FQDN — Tên Miền Đầy Đủ)

```
<service-name>.<namespace>.svc.<cluster-domain>
     │              │        │         │
     │              │        │         └── thường là "cluster.local"
     │              │        └── "svc" — chỉ định đây là Service
     │              └── namespace chứa Service
     └── tên của Service object
```

**Ví dụ:**

```
api-service.production.svc.cluster.local
postgres.staging.svc.cluster.local
redis-master.default.svc.cluster.local
```

### Dạng Rút Gọn — Search Domain Tự Động Thêm

Do cấu hình `search` trong `/etc/resolv.conf`, bạn có thể dùng tên ngắn hơn khi trong cùng namespace:

| Tên Dùng | CoreDNS Thử Lần Lượt | Kết Quả |
| -------- | -------------------- | ------- |
| `api-service` | `api-service.default.svc.cluster.local` | Tìm thấy ✅ |
| `api-service.production` | `api-service.production.svc.cluster.local` | Tìm thấy ✅ |
| `api-service.production.svc` | `api-service.production.svc.cluster.local` | Tìm thấy ✅ |
| `google.com` | `google.com.default.svc.cluster.local` → không thấy → forward ra ngoài | Tìm thấy ✅ |

---

## DNS Cho Service

### ClusterIP Service

CoreDNS tạo record A (hoặc AAAA cho IPv6) trỏ đến ClusterIP của Service.

```
api-service.default.svc.cluster.local → A record → 10.96.45.22
```

### Headless Service (clusterIP: None)

CoreDNS tạo nhiều A record — một record cho mỗi Pod backend.

```
postgres-headless.default.svc.cluster.local → A record → 10.244.1.5 (Pod 0)
                                            → A record → 10.244.2.3 (Pod 1)
                                            → A record → 10.244.3.7 (Pod 2)
```

### SRV Record (Service Record — Bản Ghi Dịch Vụ)

CoreDNS cũng tạo SRV record chứa thông tin port và protocol:

```
_http._tcp.api-service.default.svc.cluster.local
    → SRV record → priority=0, weight=100, port=8080, target=api-service.default.svc.cluster.local
```

Dùng SRV record trong service discovery của một số framework (gRPC, Consul-compatible clients).

### ExternalName Service

CoreDNS tạo CNAME record:

```
external-db.default.svc.cluster.local → CNAME → mydb.us-east-1.rds.amazonaws.com
```

---

## DNS Cho Pod

### Pod IP Dưới Dạng DNS

CoreDNS cũng tạo DNS cho từng Pod, dùng IP đổi dấu chấm thành gạch ngang:

```
10-244-1-5.default.pod.cluster.local → A record → 10.244.1.5
```

Hiếm dùng trực tiếp, nhưng hữu ích trong StatefulSet với Headless Service.

### Pod Hostname Trong StatefulSet

StatefulSet với Headless Service tạo DNS tên ổn định cho từng Pod:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

postgres-0.postgres-headless.default.svc.cluster.local → 10.244.1.5
postgres-1.postgres-headless.default.svc.cluster.local → 10.244.2.3
postgres-2.postgres-headless.default.svc.cluster.local → 10.244.3.7
```

Khi Pod restart, hostname và DNS giữ nguyên — chỉ IP thay đổi. Đây là lý do StatefulSet phù hợp với database cluster cần định danh ổn định.

---

## Search Domain và NDOTS

### NDOTS — Ngưỡng Dấu Chấm

`options ndots:5` trong `/etc/resolv.conf` quyết định khi nào CoreDNS thử search domain trước.

**Quy tắc:** Nếu tên DNS có ít hơn 5 dấu chấm → thử search domain trước; ngược lại → thử tên đầy đủ trước.

```
Tên: "api-service"          (0 dấu chấm) → thử search domain trước
     api-service.default.svc.cluster.local  ← thử 1
     Tìm thấy → trả kết quả

Tên: "api.external.com"     (2 dấu chấm) → thử search domain trước
     api.external.com.default.svc.cluster.local  ← thử 1 (không thấy)
     api.external.com.svc.cluster.local           ← thử 2 (không thấy)
     api.external.com.cluster.local               ← thử 3 (không thấy)
     api.external.com.                            ← thử 4 (thêm dấu chấm = FQDN)
     Tìm thấy → trả kết quả
```

### Vấn Đề Với NDOTS=5

Mỗi lần gọi DNS bên ngoài (ví dụ `api.stripe.com`), Pod thử ít nhất 4 DNS query trước khi thành công. Trong hệ thống nhiều request/giây, điều này tăng latency DNS đáng kể.

**Giải pháp 1:** Dùng FQDN với dấu chấm ở cuối

```python
# Thêm dấu chấm cuối → được coi là FQDN, bỏ qua search domain
api_host = "api.stripe.com."
```

**Giải pháp 2:** Tùy chỉnh ndots cho Pod

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"    # giảm xuống còn 2 dấu chấm
```

---

## Cấu Hình CoreDNS

CoreDNS được cấu hình qua **ConfigMap** tên `coredns` trong namespace `kube-system`.

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

### Corefile — File Cấu Hình CoreDNS

```
.:53 {
    errors                        # log lỗi
    health {
        lameduck 5s               # grace period khi shutdown
    }
    ready                         # endpoint /ready cho readiness probe
    kubernetes cluster.local in-addr.arpa ip6.arpa {  # xử lý DNS trong cluster
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153              # expose metrics cho Prometheus
    forward . /etc/resolv.conf {  # forward query không tìm thấy ra DNS upstream
        max_concurrent 1000
    }
    cache 30                      # cache response 30 giây
    loop                          # phát hiện vòng lặp DNS
    reload                        # tự reload config khi ConfigMap thay đổi
    loadbalance                   # cân bằng tải giữa các DNS record A
}
```

### Thêm Stub Zone (Phân Giải Domain Riêng)

Nếu cần phân giải domain nội bộ công ty (ví dụ `internal.corp`) qua DNS server riêng:

```
internal.corp:53 {
    errors
    cache 30
    forward . 10.10.0.53    # forward sang DNS server nội bộ công ty
}
.:53 {
    # cấu hình mặc định...
}
```

### Tăng Resource Cho CoreDNS

```yaml
# Trong Deployment của CoreDNS
resources:
  requests:
    memory: "70Mi"
    cpu: "100m"
  limits:
    memory: "170Mi"
    cpu: "500m"
```

---

## Custom DNS và External DNS

### Cấu Hình DNS Tùy Chỉnh Cho Pod

```yaml
spec:
  dnsPolicy: None              # tắt DNS mặc định của cluster
  dnsConfig:
    nameservers:
      - 1.1.1.1                # Cloudflare DNS
      - 8.8.8.8                # Google DNS
    searches:
      - my-namespace.svc.cluster.local
      - svc.cluster.local
    options:
      - name: ndots
        value: "5"
      - name: timeout
        value: "1"
```

### DNS Policy Options

| dnsPolicy | Ý Nghĩa |
| --------- | ------- |
| `ClusterFirst` (mặc định) | Thử CoreDNS trước, sau đó DNS upstream |
| `ClusterFirstWithHostNet` | Như ClusterFirst nhưng cho Pod dùng hostNetwork |
| `Default` | Dùng DNS của node (không dùng CoreDNS) |
| `None` | Tùy chỉnh hoàn toàn qua `dnsConfig` |

### External DNS — Tự Động Tạo DNS Record Bên Ngoài

**ExternalDNS** là add-on tự động tạo DNS record trên cloud (Route53, Google Cloud DNS) khi bạn tạo Service LoadBalancer hoặc Ingress.

```yaml
# Ingress với annotation ExternalDNS
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.example.com
    external-dns.alpha.kubernetes.io/ttl: "300"
```

ExternalDNS đọc annotation → gọi API Route53 → tạo record A trỏ đến IP của LoadBalancer.

---

## Debug DNS

### Kiểm Tra DNS Cơ Bản

```bash
# Chạy Pod debug với tools DNS
kubectl run -it --rm dns-debug --image=infoblox/dnstools --restart=Never -- bash

# Trong container:
nslookup kubernetes                          # kiểm tra DNS cơ bản
nslookup api-service                         # service cùng namespace
nslookup api-service.production              # service khác namespace
nslookup api-service.production.svc.cluster.local  # FQDN đầy đủ
```

### Kiểm Tra Với BusyBox

```bash
kubectl run -it --rm busybox --image=busybox:1.28 --restart=Never -- nslookup kubernetes
```

### Xem Log CoreDNS

```bash
# Xem log CoreDNS để debug query
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Theo dõi real-time
kubectl logs -n kube-system -l k8s-app=kube-dns -f
```

### Bật Log Chi Tiết Tạm Thời

Thêm plugin `log` vào Corefile để log mọi DNS query (cẩn thận: rất nhiều log):

```
.:53 {
    log          # thêm dòng này
    errors
    # ... phần còn lại
}
```

### Xem Metrics CoreDNS

```bash
# Port-forward CoreDNS metrics
kubectl port-forward -n kube-system service/kube-dns 9153:9153

# Curl metrics
curl http://localhost:9153/metrics | grep coredns_dns_requests_total
```

### Checklist Debug DNS

```
1. Pod có thể ping CoreDNS IP?
   kubectl exec -n <ns> <pod> -- ping 10.96.0.10

2. DNS query đến CoreDNS có hoạt động?
   kubectl exec -n <ns> <pod> -- nslookup kubernetes.default

3. Service có tồn tại và có Endpoint?
   kubectl get service <svc> -n <ns>
   kubectl get endpoints <svc> -n <ns>

4. CoreDNS Pod có đang chạy?
   kubectl get pods -n kube-system -l k8s-app=kube-dns

5. CoreDNS có đủ resource (không OOMKilled)?
   kubectl describe pod -n kube-system <coredns-pod>

6. Corefile có lỗi cú pháp không?
   kubectl logs -n kube-system <coredns-pod>
```

---

## Câu Hỏi Phỏng Vấn

**CoreDNS phân giải `api-service` như thế nào từ trong Pod?**

> 1. Pod query DNS tới CoreDNS IP (`10.96.0.10:53`)
> 2. CoreDNS nhận query `api-service`
> 3. Vì có `search default.svc.cluster.local` trong resolv.conf, CoreDNS thử `api-service.default.svc.cluster.local`
> 4. CoreDNS tìm trong zone `cluster.local` → tìm thấy Service → trả về ClusterIP
> 5. Pod dùng ClusterIP này để kết nối

**Tại sao nên dùng FQDN khi gọi service từ namespace khác?**

> Dùng `api-service` chỉ khớp service trong cùng namespace (search domain default). Để gọi sang namespace khác phải dùng ít nhất `api-service.production` hoặc FQDN đầy đủ `api-service.production.svc.cluster.local`. Dùng FQDN rõ ràng hơn, tránh nhầm lẫn khi nhiều namespace có service cùng tên.

**CoreDNS có thể gây bottleneck (điểm nghẽn) không?**

> Có. Nếu cluster lớn và nhiều request DNS, CoreDNS có thể bị quá tải. Giải pháp: (1) scale up số replica CoreDNS, (2) tăng resource (CPU/memory), (3) dùng `NodeLocal DNSCache` — chạy DNS cache trên mỗi node để giảm request đến CoreDNS trung tâm, (4) giảm ndots để tránh query thừa.

**NodeLocal DNSCache là gì?**

> NodeLocal DNSCache là DaemonSet chạy một DNS cache agent trên mỗi node. Pod query DNS đến local cache (link-local IP `169.254.20.10`) thay vì CoreDNS Pod. Giảm latency DNS (local cache vs Pod-to-Pod), giảm tải CoreDNS, và giảm số lượng UDP connection mở (vấn đề với conntrack table khi nhiều DNS query đồng thời).
