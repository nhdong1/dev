# Networking — Mạng Kubernetes

> Tổng quan về mô hình mạng Kubernetes: cách Pod giao tiếp, Service phơi bày ứng dụng, Ingress điều hướng traffic, và NetworkPolicy kiểm soát luồng dữ liệu.

## Mục Lục

1. [Mô Hình Mạng Kubernetes](#mô-hình-mạng-kubernetes)
2. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
3. [Các Thành Phần Networking](#các-thành-phần-networking)
4. [Luồng Traffic Điển Hình](#luồng-traffic-điển-hình)
5. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
6. [Checklist Thực Chiến](#checklist-thực-chiến)

---

## Mô Hình Mạng Kubernetes

Kubernetes áp đặt một mô hình mạng **flat** (phẳng) với ba nguyên tắc bất biến:

| Nguyên Tắc | Ý Nghĩa |
| ---------- | ------- |
| **Pod-to-Pod không có NAT** | Mọi Pod giao tiếp trực tiếp với nhau qua IP thật, không cần NAT (Network Address Translation — Dịch Địa Chỉ Mạng) |
| **Node-to-Pod không có NAT** | Node giao tiếp với Pod qua IP thật của Pod |
| **Pod thấy IP của mình giống IP node thấy** | Không có IP ẩn, không có địa chỉ bị biến đổi |

### Lớp Mạng Trong Cluster

```
Internet
    │
    ▼
Load Balancer (bộ cân bằng tải bên ngoài)
    │
    ▼
Ingress Controller (điều phối traffic HTTP/HTTPS vào cluster)
    │
    ▼
Service (ClusterIP — địa chỉ IP ảo ổn định)
    │
    ▼  kube-proxy ghi iptables / IPVS rules
    ▼
Pod (IP riêng, giao tiếp qua CNI plugin)
```

### Pod Network vs Service Network vs Node Network

| Loại Mạng | Phạm Vi | Ví Dụ CIDR | Quản Lý Bởi |
| --------- | ------- | ---------- | ----------- |
| **Pod Network** (mạng Pod) | Toàn cluster | `10.244.0.0/16` | CNI plugin (Calico, Flannel…) |
| **Service Network** (mạng Service) | Toàn cluster | `10.96.0.0/12` | kube-proxy + kube-apiserver |
| **Node Network** (mạng node) | Hạ tầng vật lý / cloud | `192.168.1.0/24` | Cloud provider / admin |

---

## Bản Đồ Quyết Định

```
Bạn cần làm gì?
│
├── Giao tiếp nội bộ trong cluster (Pod ↔ Pod, Service ↔ Pod)
│   └── Service loại ClusterIP + DNS CoreDNS
│       └── Xem: service-types.md, dns-coredns.md
│
├── Phơi bày port từ Pod ra ngoài (cho dev/test, không phải production)
│   └── Service loại NodePort
│       └── Xem: service-types.md
│
├── Phơi bày ứng dụng ra internet qua HTTP/HTTPS (production)
│   ├── Một domain, nhiều path/service → Ingress
│   └── Nhiều domain, TLS termination → Ingress + cert-manager
│       └── Xem: ingress.md
│
├── Phơi bày ứng dụng TCP/UDP (không phải HTTP) ra ngoài
│   └── Service loại LoadBalancer
│       └── Xem: service-types.md
│
├── Kiểm soát Pod nào được nói chuyện với Pod nào
│   └── NetworkPolicy
│       └── Xem: network-policy.md
│
├── Tìm hiểu về DNS nội bộ cluster và service discovery
│   └── CoreDNS
│       └── Xem: dns-coredns.md
│
└── Chọn hoặc so sánh CNI plugin (Calico, Flannel, Cilium)
    └── Xem: cni-plugins.md
```

---

## Các Thành Phần Networking

### Service — Địa Chỉ Ổn Định Cho Pod

**Service** là một tài nguyên Kubernetes tạo ra một **virtual IP** (địa chỉ IP ảo) ổn định đại diện cho một nhóm Pod. Khi Pod chết và được tạo mới (IP mới), Service giữ địa chỉ không đổi.

```
Client ──► Service (ClusterIP: 10.96.0.1:80) ──► Pod A (10.244.1.5:8080)
                                              ──► Pod B (10.244.2.3:8080)
                                              ──► Pod C (10.244.3.7:8080)
```

**Bốn loại Service:**
- `ClusterIP` — chỉ truy cập được trong cluster (mặc định)
- `NodePort` — mở port trên mọi node, truy cập từ ngoài
- `LoadBalancer` — tạo cloud load balancer (phụ thuộc cloud provider)
- `ExternalName` — alias DNS cho dịch vụ bên ngoài cluster

Tham khảo chi tiết: [service-types.md](./service-types.md)

---

### Ingress — Cổng Vào HTTP/HTTPS

**Ingress** là tài nguyên Kubernetes khai báo quy tắc điều hướng HTTP/HTTPS từ bên ngoài vào các Service bên trong cluster. Cần có **Ingress Controller** (bộ điều khiển Ingress) để thực thi quy tắc.

```
Internet
    │  HTTPS :443
    ▼
Ingress Controller (NGINX / Traefik / AWS ALB)
    │
    ├── /api/* → service: api-service:8080
    ├── /web/* → service: web-service:3000
    └── *.static.example.com → service: cdn-proxy:80
```

**Ingress Controller phổ biến:**
- **NGINX Ingress Controller** — phổ biến nhất, linh hoạt cao
- **Traefik** — tự động phát hiện service, UI dashboard
- **AWS ALB Ingress Controller** — tích hợp Application Load Balancer trên EKS
- **GKE Ingress** — tích hợp Google Cloud Load Balancer trên GKE

Tham khảo chi tiết: [ingress.md](./ingress.md)

---

### NetworkPolicy — Tường Lửa Nội Bộ Cluster

**NetworkPolicy** (chính sách mạng) là tài nguyên Kubernetes kiểm soát lưu lượng mạng vào/ra của Pod ở tầng L3/L4. Mặc định, Kubernetes cho phép mọi Pod giao tiếp với nhau — NetworkPolicy thêm rào cản để cô lập.

```
Mặc định (không có NetworkPolicy):
Pod A ──► Pod B ──► Pod C   ✅ tất cả thông nhau

Sau khi áp dụng NetworkPolicy:
Pod A ──► Pod B   ✅ cho phép
Pod A ──► Pod C   ❌ từ chối
Pod B ──► Pod C   ✅ cho phép (theo rule riêng)
```

> NetworkPolicy chỉ có tác dụng nếu CNI plugin hỗ trợ (Calico, Cilium, Weave). Flannel **không** hỗ trợ NetworkPolicy.

Tham khảo chi tiết: [network-policy.md](./network-policy.md)

---

### CoreDNS — Bộ Phân Giải Tên Nội Bộ

**CoreDNS** là DNS server mặc định trong Kubernetes cluster. Nó cho phép Pod tìm kiếm Service bằng tên thay vì phải biết IP.

```
Pod gọi: http://api-service.default.svc.cluster.local
                │           │        │       │
                │           │        │       └── domain gốc cluster
                │           │        └── svc (service)
                │           └── default (namespace)
                └── tên service
```

**Dạng rút gọn trong cùng namespace:**

```
http://api-service          → CoreDNS tự thêm .default.svc.cluster.local
http://api-service.staging  → tìm service ở namespace "staging"
```

Tham khảo chi tiết: [dns-coredns.md](./dns-coredns.md)

---

### CNI Plugin — Nền Tảng Mạng Pod

**CNI (Container Network Interface — Giao Diện Mạng Container)** là tiêu chuẩn xác định cách plugin mạng cấp IP cho Pod và đảm bảo Pod-to-Pod connectivity.

| CNI Plugin | Điểm Mạnh | NetworkPolicy | Hiệu Năng |
| ---------- | --------- | ------------- | --------- |
| **Flannel** | Đơn giản, dễ cài | ❌ Không | Tốt |
| **Calico** | NetworkPolicy mạnh, BGP | ✅ Có | Rất tốt |
| **Cilium** | eBPF-based, observability cao | ✅ Có | Xuất sắc |
| **Weave** | Multi-cloud, encrypted | ✅ Có | Tốt |

Tham khảo chi tiết: [cni-plugins.md](./cni-plugins.md)

---

## Luồng Traffic Điển Hình

### Luồng 1: Request Từ Ngoài Vào Ứng Dụng

```
1. Client gửi request HTTPS tới api.example.com
2. DNS public phân giải → IP của Load Balancer (cloud)
3. Load Balancer chuyển tiếp → Ingress Controller Pod
4. Ingress Controller đọc rules → chọn Service phù hợp
5. kube-proxy (iptables/IPVS) chọn một Pod backend
6. Request đến Pod xử lý
7. Response trả về theo đường ngược lại
```

### Luồng 2: Giao Tiếp Service-to-Service Nội Bộ

```
1. Pod A gọi http://order-service/api/orders
2. CoreDNS phân giải "order-service" → ClusterIP 10.96.45.12
3. kube-proxy intercept traffic đến 10.96.45.12
4. iptables/IPVS chọn ngẫu nhiên một Pod của order-service
5. Gói tin được gửi trực tiếp đến Pod đích (không qua NAT)
```

### Luồng 3: Pod Gọi Ra Ngoài Internet

```
1. Pod gửi gói tin đến 8.8.8.8 (Google DNS)
2. CNI plugin route gói tin ra khỏi Pod network
3. Node thực hiện SNAT (Source NAT — Dịch Địa Chỉ Nguồn)
   đổi IP nguồn từ Pod IP → Node IP
4. Gói tin rời cluster qua network interface của node
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**ClusterIP, NodePort, LoadBalancer khác nhau thế nào?**

> - `ClusterIP`: IP ảo chỉ dùng trong cluster, không truy cập từ ngoài được
> - `NodePort`: mở một port cố định (30000–32767) trên mọi node, truy cập từ ngoài qua `<NodeIP>:<NodePort>`
> - `LoadBalancer`: yêu cầu cloud provider tạo load balancer thật, phơi bày ra internet qua IP public

**Ingress khác Service LoadBalancer thế nào?**

> - `LoadBalancer` hoạt động ở tầng TCP/UDP (L4), mỗi service cần một load balancer riêng (tốn tiền)
> - `Ingress` hoạt động ở tầng HTTP/HTTPS (L7), một Ingress Controller có thể phục vụ nhiều service, hỗ trợ path-based routing và TLS termination

**CoreDNS phân giải tên service ra sao?**

> Pod truy vấn CoreDNS (thường ở `10.96.0.10`). CoreDNS tìm trong internal DNS zone `cluster.local` → trả về ClusterIP của service. Pod dùng ClusterIP này để gửi request.

### Câu Hỏi Nâng Cao

**Tại sao cần NetworkPolicy nếu đã có RBAC?**

> RBAC (Role-Based Access Control) kiểm soát **ai có thể làm gì với Kubernetes API** (ví dụ: ai có thể deploy, xem Pod). NetworkPolicy kiểm soát **lưu lượng mạng tầng TCP/UDP giữa các Pod**. Hai cơ chế bảo mật ở hai tầng hoàn toàn khác nhau và bổ sung cho nhau.

**Headless Service là gì và khi nào dùng?**

> Headless Service được khai báo với `clusterIP: None`. Thay vì trả về một ClusterIP, CoreDNS trả về trực tiếp danh sách IP của các Pod. Dùng khi cần kết nối trực tiếp đến từng Pod cụ thể — điển hình cho StatefulSet (database cluster, Kafka) hoặc service discovery tùy chỉnh.

**kube-proxy dùng iptables hay IPVS?**

> Mặc định là `iptables`. Với cluster lớn (> 1000 Services), `IPVS` (IP Virtual Server — Máy Chủ IP Ảo) hiệu quả hơn vì dùng hash table O(1) thay vì duyệt tuyến tính iptables O(n). Cấu hình bằng `--proxy-mode=ipvs` trên kube-proxy.

---

## Checklist Thực Chiến

### Thiết Lập Networking Cơ Bản

- [ ] Chọn CNI plugin phù hợp (Calico cho NetworkPolicy, Cilium cho observability cao)
- [ ] Xác nhận Pod-to-Pod connectivity hoạt động (`kubectl exec` + `curl`)
- [ ] Kiểm tra CoreDNS hoạt động (`kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes`)
- [ ] Cấu hình Service ClusterIP cho mọi ứng dụng
- [ ] Đặt tên Service rõ ràng để DNS resolution dễ nhớ

### Phơi Bày Ứng Dụng Ra Ngoài

- [ ] Cài Ingress Controller (NGINX hoặc tương đương)
- [ ] Cấu hình TLS với cert-manager (Let's Encrypt hoặc certificate nội bộ)
- [ ] Đặt annotation rate-limiting trên Ingress
- [ ] Test Ingress routing với `curl -H "Host: domain.com"`
- [ ] Cấu hình health check endpoint cho Ingress

### Bảo Mật Mạng

- [ ] Áp dụng default-deny NetworkPolicy cho mọi namespace production
- [ ] Chỉ mở ingress/egress cụ thể theo nguyên tắc least privilege
- [ ] Kiểm tra NetworkPolicy với `kubectl exec` thử kết nối bị chặn
- [ ] Xem xét mTLS (mutual TLS) nếu dùng service mesh (Istio/Linkerd)
- [ ] Monitor network traffic anomaly với Cilium hoặc Falco

### Tối Ưu Hiệu Năng

- [ ] Cân nhắc IPVS thay iptables cho cluster > 1000 Services
- [ ] Dùng Endpoint Slices (thay EndpointsList cũ) — mặc định từ K8s 1.21+
- [ ] Đặt `sessionAffinity: ClientIP` nếu ứng dụng cần sticky session
- [ ] Monitor latency giữa các service bằng Prometheus + Grafana

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
| ---- | -------- |
| [service-types.md](./1-service-types.md) | ClusterIP, NodePort, LoadBalancer, ExternalName chi tiết |
| [ingress.md](./2-ingress.md) | Ingress Controller, TLS, routing rules |
| [network-policy.md](./3-network-policy.md) | Kiểm soát lưu lượng vào/ra giữa các Pod |
| [dns-coredns.md](./4-dns-coredns.md) | DNS nội bộ cluster, service discovery |
| [cni-plugins.md](./5-cni-plugins.md) | Calico, Flannel, Cilium — so sánh và lựa chọn |
