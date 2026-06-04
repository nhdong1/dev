# CNI Plugins — Nền Tảng Mạng Pod Trong Kubernetes

> CNI (Container Network Interface — Giao Diện Mạng Container) là tiêu chuẩn xác định cách plugin mạng cấp IP cho Pod và đảm bảo kết nối Pod-to-Pod trong cluster. Lựa chọn CNI ảnh hưởng đến bảo mật, hiệu năng, và tính năng mạng của cluster.

## Mục Lục

1. [CNI Là Gì?](#cni-là-gì)
2. [Mô Hình Mạng Kubernetes Yêu Cầu](#mô-hình-mạng-kubernetes-yêu-cầu)
3. [Flannel — Đơn Giản và Nhẹ](#flannel--đơn-giản-và-nhẹ)
4. [Calico — NetworkPolicy và BGP Routing](#calico--networkpolicy-và-bgp-routing)
5. [Cilium — eBPF và Observability Cao](#cilium--ebpf-và-observability-cao)
6. [Weave — Multi-Cloud và Mã Hoá](#weave--multi-cloud-và-mã-hoá)
7. [So Sánh Tổng Hợp](#so-sánh-tổng-hợp)
8. [Cách Chọn CNI Phù Hợp](#cách-chọn-cni-phù-hợp)
9. [Cài Đặt và Vận Hành](#cài-đặt-và-vận-hành)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CNI Là Gì?

**CNI (Container Network Interface)** là một đặc tả (specification) định nghĩa interface giữa container runtime và network plugin. Khi Kubernetes tạo một Pod:

```
1. kubelet yêu cầu container runtime tạo container
2. Container runtime gọi CNI plugin
3. CNI plugin:
   a. Cấp phát IP cho Pod từ IPAM (IP Address Management — Quản Lý Địa Chỉ IP)
   b. Tạo virtual network interface (giao diện mạng ảo) trong Pod
   c. Cấu hình routing (định tuyến) để Pod giao tiếp được với Pod khác
4. Pod có IP và có thể giao tiếp trong cluster
```

### CNI Plugin Không Phải Là

- **Không phải service discovery** (CoreDNS làm điều này)
- **Không phải load balancer** (kube-proxy làm điều này)
- **Không phải firewall ứng dụng** (NetworkPolicy là L3/L4, không phải L7)

### Các CNI Plugin Phổ Biến

| Plugin | Công Ty / Dự Án | Công Nghệ Core |
| ------ | --------------- | -------------- |
| **Flannel** | CoreOS / Flannel OSS | VXLAN / host-gw |
| **Calico** | Tigera | BGP / iptables |
| **Cilium** | Isovalent / CNCF | eBPF |
| **Weave** | Weaveworks | VXLAN + IPAM riêng |
| **Antrea** | VMware | Open vSwitch |
| **Canal** | Calico + Flannel | Kết hợp |

---

## Mô Hình Mạng Kubernetes Yêu Cầu

Bất kỳ CNI plugin nào cũng phải đảm bảo ba nguyên tắc:

1. **Pod-to-Pod không NAT**: Pod A (10.244.1.5) gọi Pod B (10.244.2.3) — B nhìn thấy IP nguồn là 10.244.1.5, không phải IP node
2. **Node-to-Pod không NAT**: Node giao tiếp với Pod qua IP thật của Pod
3. **Pod thấy IP của mình = IP mà người khác thấy**: Không có địa chỉ ẩn

### Overlay vs Underlay Network

**Overlay Network** (mạng chồng): Đóng gói gói tin Pod vào gói tin khác trước khi gửi qua mạng vật lý. Không yêu cầu cấu hình router.

```
Pod A gửi gói tin → CNI đóng gói (VXLAN/GENEVE) → gửi qua mạng vật lý → CNI bóc vỏ → Pod B nhận
```

**Underlay Network** (mạng nền): Dùng routing thật (BGP) trên switch/router vật lý. Hiệu năng cao hơn vì không có overhead đóng gói.

```
Pod A gửi gói tin → BGP routing (router vật lý biết route Pod network) → Pod B nhận
```

---

## Flannel — Đơn Giản và Nhẹ

**Flannel** là CNI plugin đơn giản nhất, phù hợp cho môi trường học tập và lab nhỏ.

### Đặc Điểm

- Cơ chế: VXLAN (mặc định), UDP tunnel, host-gw
- NetworkPolicy: **Không hỗ trợ** (cần kết hợp với Calico để có NetworkPolicy)
- Cài đặt: Rất đơn giản, ít cấu hình
- Resource: Thấp

### Cơ Chế VXLAN (Virtual eXtensible LAN — Mạng Cục Bộ Ảo Mở Rộng)

```
Node 1 (10.0.0.1)                    Node 2 (10.0.0.2)
Pod A (10.244.1.5)                   Pod B (10.244.2.3)
    │                                     │
    ▼ gói tin Pod A → Pod B               │
flannel encapsulate (đóng gói):           │
[IP header: 10.0.0.1 → 10.0.0.2]         │
[UDP port 8472]                           │
[VXLAN header: VNI=1]                     │
[Original: 10.244.1.5 → 10.244.2.3]      │
    │                                     │
    └───────────────────────────────────→ flannel decapsulate (bóc vỏ)
                                          Pod B nhận gói tin gốc
```

### Cài Đặt Flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Khi Nào Dùng Flannel

- Môi trường lab, học tập, development
- Cluster nhỏ (<50 node) không cần NetworkPolicy
- Cần cài đặt nhanh, ít overhead cấu hình
- On-premise đơn giản với kubeadm

---

## Calico — NetworkPolicy và BGP Routing

**Calico** là CNI plugin phổ biến nhất trong production, nổi tiếng với hỗ trợ NetworkPolicy mạnh mẽ và hiệu năng cao qua BGP routing.

### Đặc Điểm

| Tính Năng | Calico |
| --------- | ------ |
| NetworkPolicy | ✅ Kubernetes NetworkPolicy + Calico GlobalNetworkPolicy (mở rộng) |
| Routing | BGP (Border Gateway Protocol — Giao Thức Định Tuyến Biên Giới) hoặc VXLAN |
| Hiệu Năng | Rất tốt (BGP mode không có overhead đóng gói) |
| eBPF mode | ✅ Có (Calico eBPF data plane) |
| Độ Phức Tạp | Trung bình |
| Cloud Provider | EKS, AKS, GKE, on-premise |

### Hai Chế Độ Hoạt Động

**BGP Mode (Underlay):**
```
Pod network routes được phân phối qua BGP
Router vật lý biết cách route đến từng Pod CIDR
Không overhead đóng gói → latency thấp hơn
Yêu cầu: cơ sở hạ tầng mạng hỗ trợ BGP
```

**VXLAN Mode (Overlay):**
```
Tương tự Flannel VXLAN
Không yêu cầu BGP trên switch/router
Phù hợp khi không kiểm soát được cơ sở hạ tầng mạng (cloud)
```

### GlobalNetworkPolicy — Mở Rộng NetworkPolicy

Calico cung cấp **GlobalNetworkPolicy** (chính sách mạng toàn cục) áp dụng xuyên namespace:

```yaml
# Calico GlobalNetworkPolicy — Kubernetes NetworkPolicy chỉ áp dụng trong một namespace
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: block-external-egress
spec:
  selector: all()            # áp dụng cho TẤT CẢ Pod trong cluster
  types:
    - Egress
  egress:
    - action: Allow
      destination:
        nets:
          - 10.0.0.0/8      # cho phép traffic nội bộ
    - action: Deny           # chặn mọi egress ra ngoài
```

### Cài Đặt Calico

```bash
# Cài Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Cài Calico với cấu hình mặc định
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

# Kiểm tra
kubectl get pods -n calico-system
```

### Calicoctl — CLI Quản Lý Calico

```bash
# Xem routes đang dùng
calicoctl get nodes --output=wide

# Xem BGP peers
calicoctl get bgppeers

# Xem NetworkPolicy của Calico (bao gồm cả K8s NetworkPolicy)
calicoctl get networkpolicy --all-namespaces

# Debug policy cho một endpoint
calicoctl endpoint show --workload=production/api-pod-xxx --output=yaml
```

---

## Cilium — eBPF và Observability Cao

**Cilium** là CNI plugin thế hệ mới dùng **eBPF (extended Berkeley Packet Filter)** — công nghệ cho phép chạy code trong kernel Linux mà không cần kernel module, mang lại hiệu năng và observability (khả năng quan sát) vượt trội.

### Đặc Điểm

| Tính Năng | Cilium |
| --------- | ------ |
| NetworkPolicy | ✅ K8s NetworkPolicy + CiliumNetworkPolicy (L7 aware) |
| Routing | eBPF direct routing, VXLAN, Geneve |
| Hiệu Năng | Xuất sắc (bypass iptables hoàn toàn) |
| L7 Policy | ✅ HTTP path, gRPC method, Kafka topic |
| Observability | ✅ Hubble — network flow visualization |
| Service Mesh | ✅ Cilium Service Mesh (thay thế Istio cho một số use case) |
| Độ Phức Tạp | Cao |

### eBPF — Tại Sao Nhanh Hơn

**iptables (Flannel, Calico truyền thống):**
```
Packet → iptables chain → nhiều rule → quyết định
         O(n) với n = số Service × số rule mỗi Service
         Với 10.000 Services → rất chậm
```

**eBPF (Cilium):**
```
Packet → eBPF program trong kernel → hash lookup → quyết định
         O(1) bất kể số lượng Service
         Không cần copy packet lên user space
```

### CiliumNetworkPolicy — Policy Tầng L7

```yaml
# Chỉ cho phép GET /api/users, từ chối tất cả path khác
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: l7-http-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api/users"
              - method: "POST"
                path: "/api/users"
              # mọi request khác → bị từ chối tự động
```

### Hubble — Observability Cho Mạng

**Hubble** là thành phần của Cilium cung cấp khả năng quan sát toàn diện luồng mạng:

```bash
# Cài Hubble CLI
cilium hubble enable

# Xem luồng traffic real-time
hubble observe --namespace production --follow

# Lọc theo Pod cụ thể
hubble observe --pod api-pod-xxx --follow

# Kết quả mẫu:
# TIMESTAMP    SOURCE                DESTINATION           TYPE   VERDICT  SUMMARY
# 10:23:01     production/frontend   production/api-pod    HTTP   FORWARDED  GET /api/users 200
# 10:23:02     production/frontend   production/postgres   TCP    DROPPED  policy-deny
```

### Hubble UI — Giao Diện Đồ Hoạ

```bash
# Bật Hubble UI
cilium hubble enable --ui

# Port-forward để xem
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# Mở http://localhost:12000 — thấy service dependency map
```

### Cài Đặt Cilium

```bash
# Cài bằng Helm (khuyến nghị)
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true  # thay thế kube-proxy bằng eBPF

# Kiểm tra
cilium status
cilium connectivity test
```

---

## Weave — Multi-Cloud và Mã Hoá

**Weave** (Weave Net) nổi bật với khả năng mã hoá traffic giữa các node và hỗ trợ multi-cloud.

### Đặc Điểm

| Tính Năng | Weave |
| --------- | ----- |
| NetworkPolicy | ✅ Có |
| Mã Hoá | ✅ Sleeve mode với mã hoá NaCl |
| Multi-Cloud | ✅ Kết nối node trên nhiều cloud/network |
| Routing | VXLAN (fast data path) hoặc Sleeve (có mã hoá) |
| Hiệu Năng | Tốt (VXLAN), chậm hơn khi bật mã hoá |

> Lưu ý: Weaveworks (công ty tạo Weave) đã ngừng hoạt động vào 2024. Weave Net vẫn là OSS nhưng không còn được maintain tích cực. Cân nhắc dùng Calico hoặc Cilium cho dự án mới.

---

## So Sánh Tổng Hợp

| Tính Năng | Flannel | Calico | Cilium | Weave |
| --------- | ------- | ------ | ------ | ----- |
| **NetworkPolicy K8s** | ❌ | ✅ | ✅ | ✅ |
| **NetworkPolicy L7** | ❌ | ❌ | ✅ | ❌ |
| **Công Nghệ** | VXLAN | BGP/iptables | eBPF | VXLAN |
| **Hiệu Năng** | Tốt | Rất tốt | Xuất sắc | Tốt |
| **Độ Phức Tạp** | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Observability** | Thấp | Trung bình | Rất cao (Hubble) | Thấp |
| **Mã Hoá Traffic** | ❌ | WireGuard (thêm) | WireGuard (thêm) | ✅ (native) |
| **kube-proxy Replacement** | ❌ | ✅ (eBPF mode) | ✅ (eBPF mode) | ❌ |
| **BGP Support** | ❌ | ✅ | ✅ | ❌ |
| **Hỗ Trợ Trên Cloud** | Mọi nơi | Mọi nơi | Mọi nơi | Mọi nơi |
| **Phù Hợp Môi Trường** | Lab/Dev | Production | Production lớn | Multi-cloud |

---

## Cách Chọn CNI Phù Hợp

### Sơ Đồ Quyết Định

```
Bạn cần gì?
│
├── Lab, học tập, cluster đơn giản
│   └── Flannel (cài nhanh, không phức tạp)
│
├── Production cần NetworkPolicy, hiệu năng cao
│   ├── Đã có kinh nghiệm Kubernetes → Calico
│   └── Cần observability, L7 policy, cluster lớn → Cilium
│
├── On-premise với routers hỗ trợ BGP
│   └── Calico BGP mode (latency thấp nhất, không overhead)
│
├── Cloud managed (EKS, GKE, AKS)
│   ├── EKS → Calico hoặc Cilium hoặc AWS VPC CNI (native)
│   ├── GKE → Cilium (GKE Dataplane V2) hoặc Calico
│   └── AKS → Azure CNI hoặc Calico
│
└── Cần security cao, network flow visibility
    └── Cilium + Hubble
```

### Trên Managed Kubernetes

| Cloud | CNI Mặc Định | Khuyến Nghị |
| ----- | ------------ | ----------- |
| EKS | AWS VPC CNI | Calico (NetworkPolicy) hoặc Cilium |
| GKE | Kubenet | Cilium (GKE Dataplane V2) hoặc Calico |
| AKS | Azure CNI | Calico hoặc Cilium |

---

## Cài Đặt và Vận Hành

### Kiểm Tra CNI Đang Dùng

```bash
# Xem CNI plugin trên node
ls /etc/cni/net.d/

# Xem Pod của CNI plugin đang chạy
kubectl get pods -n kube-system | grep -E "calico|cilium|flannel|weave"

# Xem log CNI plugin
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50
```

### Kiểm Tra Pod Network Hoạt Động

```bash
# Tạo Pod trên node khác nhau để test cross-node connectivity
kubectl run pod-a --image=busybox --restart=Never -- sleep 3600
kubectl run pod-b --image=busybox --restart=Never -- sleep 3600

# Lấy IP của pod-b
kubectl get pod pod-b -o wide

# Từ pod-a, ping pod-b
kubectl exec pod-a -- ping -c 3 <pod-b-IP>
```

### Xem Route Table Của CNI

```bash
# Trên node, xem routes được tạo bởi CNI
ip route show

# Kết quả mẫu với Calico (BGP mode):
# 10.244.2.0/24 via 10.0.0.2 dev eth0 proto bird  ← route đến node 2's Pod CIDR
# 10.244.3.0/24 via 10.0.0.3 dev eth0 proto bird  ← route đến node 3's Pod CIDR
# 10.244.1.0/24 dev cali0 scope link               ← Pod CIDR trên node này
```

### Migrate CNI Plugin (Cẩn Thận)

> Migrate CNI plugin là thao tác **rủi ro cao** trong production. Thường cần rebuild cluster hoặc drain từng node và cài lại.

```
Quy trình an toàn:
1. Dựng cluster mới với CNI mới
2. Migrate workload sang cluster mới (blue-green cluster migration)
3. Không nên thay CNI trực tiếp trên cluster đang chạy production
```

---

## Câu Hỏi Phỏng Vấn

**CNI plugin làm gì chính xác khi một Pod được tạo?**

> Khi kubelet tạo một Pod, nó gọi CNI plugin qua CNI spec. Plugin thực hiện: (1) tạo network namespace cho Pod, (2) tạo virtual ethernet pair (veth pair) — một đầu trong Pod namespace, một đầu trên host, (3) gắn IP từ IPAM vào đầu trong Pod, (4) cấu hình routing để gói tin từ/đến IP này đi đúng đường, (5) cấu hình iptables/eBPF rules nếu cần.

**Tại sao Flannel không hỗ trợ NetworkPolicy?**

> Flannel chỉ đảm nhận việc cấp IP và kết nối Pod — không có thành phần nào đọc NetworkPolicy object từ API Server và enforce (thực thi) chúng ở tầng packet filtering. Calico và Cilium có agent (calico-node, cilium-agent) chạy trên mỗi node, theo dõi NetworkPolicy và tương ứng cấu hình iptables/eBPF rules.

**Sự khác biệt giữa CNI dùng iptables và eBPF?**

> iptables duyệt tuần tự qua danh sách rules — O(n) với n rules. Khi cluster có hàng nghìn Service và NetworkPolicy, số lượng iptables rules tăng rất nhanh, gây latency cao và CPU tốn kém. eBPF dùng hash map trong kernel — O(1) bất kể số lượng. eBPF cũng cho phép xử lý packet ngay tại điểm vào network stack, trước khi đi qua toàn bộ TCP/IP stack, nên nhanh hơn nhiều.

**Khi nào nên chọn Cilium thay Calico?**

> Chọn Cilium khi: (1) cần L7 NetworkPolicy (HTTP path, gRPC method), (2) cần network flow visibility để debug service dependencies (Hubble), (3) cluster lớn (>1000 node) với hiệu năng kube-proxy là bottleneck, (4) muốn thay thế kube-proxy bằng eBPF. Chọn Calico khi: đội đã quen, cần BGP integration với router on-premise, hoặc muốn giải pháp đơn giản hơn Cilium.
