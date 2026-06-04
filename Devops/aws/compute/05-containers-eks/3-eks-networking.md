# EKS Networking — VPC CNI, CoreDNS, kube-proxy & Ingress

> Networking là một trong những phần phức tạp nhất của EKS. Hiểu rõ cách Pod nhận IP, cách traffic đi từ internet vào Pod, và cách các Pod nói chuyện với nhau là kiến thức thiết yếu để vận hành cluster production.

---

## 📚 Mục Lục

1. [Kubernetes Networking Model — Mô Hình Mạng Kubernetes](#kubernetes-networking-model)
2. [Amazon VPC CNI — Plugin Mạng AWS](#amazon-vpc-cni)
3. [CoreDNS — DNS Nội Bộ Cluster](#coredns)
4. [kube-proxy — Service Networking](#kube-proxy)
5. [Service Types — Các Loại Dịch Vụ](#service-types)
6. [Ingress & AWS Load Balancer Controller](#ingress--aws-load-balancer-controller)
7. [Network Policies — Chính Sách Mạng](#network-policies)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kubernetes Networking Model

### 4 Nguyên Tắc Cốt Lõi

```
1. Mỗi Pod có IP riêng biệt và duy nhất trong cluster
2. Tất cả Pods có thể communicate với nhau mà không cần NAT
3. Tất cả Nodes có thể communicate với tất cả Pods mà không cần NAT
4. IP mà Pod tự thấy = IP mà các Pod khác thấy (no masquerading)
```

### Các Cấp Độ Networking Trong Kubernetes

```
Internet
    │
    ▼
LoadBalancer / Ingress (Lối vào cluster)
    │
    ▼
Service (ClusterIP / NodePort / LoadBalancer)
    │  kube-proxy implements Service VIP → Pod IP mapping
    ▼
Pod (IP từ VPC subnet — với Amazon VPC CNI)
    │
    ▼
Container (chia sẻ network namespace với Pod)
```

---

## Amazon VPC CNI

### Cơ Chế Hoạt Động

**VPC CNI (Container Network Interface — Giao Diện Mạng Container)** là plugin mạng mặc định của EKS. Điểm đặc biệt: mỗi Pod nhận **IP address thực từ VPC subnet**, không phải overlay network.

```
Node EC2 instance:
  Primary ENI (Elastic Network Interface — Giao Diện Mạng):
    - eth0: 10.0.1.10 (IP của node)
    - Secondary IPs: 10.0.1.11, 10.0.1.12 ... (gán cho Pods)

  Secondary ENI (khi cần nhiều Pod hơn):
    - eth1: 10.0.1.20 (ENI IP)
    - Secondary IPs: 10.0.1.21, 10.0.1.22 ...

  Pod A → IP 10.0.1.11 (VPC IP thực, route trực tiếp)
  Pod B → IP 10.0.1.12 (VPC IP thực, route trực tiếp)
```

### Lợi Ích So Với Overlay Network

| VPC CNI (EKS) | Overlay Network (Flannel/Calico VXLAN) |
|---|---|
| Pod IP là VPC IP thực | Pod IP từ overlay subnet riêng |
| Route trực tiếp, không có encapsulation | VXLAN encapsulation overhead ~5-10% |
| Security Group áp dụng được cho Pod | Security Group chỉ cho node level |
| AWS services (RDS, S3...) thấy Pod IP thực | AWS services thấy node IP |
| Đơn giản hơn để debug | Phức tạp hơn |

### Giới Hạn Số Pod Per Node

```
Số Pod tối đa per node = (Số ENI tối đa × (Số IP per ENI - 1)) + 2

Ví dụ: t3.medium
  - Max ENI: 3
  - Max IP per ENI: 6
  - Max Pods = (3 × (6-1)) + 2 = 17 Pods

Ví dụ: m5.xlarge
  - Max ENI: 4
  - Max IP per ENI: 15
  - Max Pods = (4 × (15-1)) + 2 = 58 Pods
```

### Prefix Delegation — Tăng Mật Độ Pod

```bash
# Bật Prefix Delegation để tăng đáng kể số Pod per node
# Thay vì gán IP individual, gán /28 prefix (16 IPs) cho ENI

kubectl set env daemonset aws-node \
  -n kube-system \
  ENABLE_PREFIX_DELEGATION=true

# Với Prefix Delegation trên m5.xlarge:
# Max ENI = 4, Max prefixes per ENI = 14
# Max Pods = (4 × 14 × 16) - overhead = ~800 Pods!
# (Thực tế giới hạn bởi kubelet: --max-pods=250 mặc định)
```

### Security Groups for Pods — Nhóm Bảo Mật Cho Pod

```yaml
# Tính năng: Gán Security Group trực tiếp cho Pod (không chỉ node)
# Yêu cầu: instance type hỗ trợ branch ENI (nitro instances)

# 1. Tạo SecurityGroupPolicy
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: backend-sg-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
  securityGroups:
    groupIds:
    - sg-0123456789abcdef0   # Security group chỉ cho backend Pods

# 2. Giờ chỉ Pods với label app=backend-api mới có SG này
# Không cần mở rộng security group cho toàn bộ node
```

---

## CoreDNS

### Vai Trò CoreDNS

**CoreDNS** là DNS server của cluster — cho phép Pods tìm kiếm Services bằng tên thay vì IP.

### DNS Naming Convention — Quy Tắc Đặt Tên DNS

```
Service DNS:
  <service-name>.<namespace>.svc.cluster.local
  Ví dụ: mysql.database.svc.cluster.local → 10.100.45.23 (ClusterIP)

  Short form (trong cùng namespace): mysql
  Cross-namespace: mysql.database

Pod DNS:
  <pod-ip-dashes>.<namespace>.pod.cluster.local
  Ví dụ: 10-0-1-11.default.pod.cluster.local
```

### DNS Lookup Flow

```
Pod muốn kết nối đến "mysql.database"
  │
  │ /etc/resolv.conf: search default.svc.cluster.local svc.cluster.local cluster.local
  │
  ▼
DNS query: mysql.database.default.svc.cluster.local → NXDOMAIN
DNS query: mysql.database.svc.cluster.local          → NXDOMAIN
DNS query: mysql.database.cluster.local              → 10.100.45.23 ✅
  │
  │ Pod kết nối đến 10.100.45.23 (Service ClusterIP)
  │ kube-proxy route đến Pod thực sự
  ▼
```

### CoreDNS Scaling

```yaml
# CoreDNS là bottleneck potential — scale nó theo cluster size
# Rule of thumb: 1 CoreDNS replica per 5,000 Pods, tối thiểu 2

# Bật autoscaling cho CoreDNS
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: coredns
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
EOF
```

### Troubleshooting DNS

```bash
# Test DNS resolution từ trong Pod
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never \
  -- nslookup kubernetes.default

# Xem CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Kiểm tra ConfigMap CoreDNS
kubectl get configmap coredns -n kube-system -o yaml
```

---

## kube-proxy

### Vai Trò kube-proxy

**kube-proxy** implement Service networking — chuyển đổi Service IP (ClusterIP) thành Pod IP thực.

### Cách kube-proxy Hoạt Động (iptables mode)

```
Service "my-app" ClusterIP: 10.100.45.23:80
  Backend Pods:
    - 10.0.1.11:8080
    - 10.0.1.12:8080
    - 10.0.1.13:8080

kube-proxy tạo iptables rules:
  DNAT: 10.100.45.23:80 → random(10.0.1.11:8080, 10.0.1.12:8080, 10.0.1.13:8080)
  
  Cụ thể:
    33% → 10.0.1.11:8080
    50% của 67% còn lại → 10.0.1.12:8080
    100% còn lại → 10.0.1.13:8080
    (statistical load balancing)
```

### iptables vs IPVS Mode

| Tính Năng | iptables (Mặc Định) | IPVS |
|---|---|---|
| **Performance** | O(n) rules — chậm với nhiều Services | O(1) hash table |
| **Load balancing** | Random | RR, LC, SH, DH, nhiều thuật toán |
| **Scale** | ~10,000 Services trở đi bắt đầu chậm | Tốt với hàng chục nghìn Services |
| **Setup** | Có sẵn | Cần enable kernel module |
| **Debug** | `iptables -L -t nat` | `ipvsadm -ln` |

---

## Service Types

### 1. ClusterIP (Mặc Định)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP       # Chỉ accessible từ trong cluster
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
# ClusterIP: 10.100.45.23 (tự động gán)
# Dùng khi: internal service-to-service communication
```

### 2. NodePort

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080     # Range: 30000-32767
# Accessible từ ngoài: <NodeIP>:30080
# Dùng khi: testing, không production (expose node IP ra ngoài)
```

### 3. LoadBalancer

```yaml
spec:
  type: LoadBalancer
  # Khi dùng AWS Load Balancer Controller:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
  ports:
  - port: 443
    targetPort: 8443
# AWS tự tạo NLB/CLB, gán DNS name
# Dùng khi: expose TCP/UDP service ra internet (không phải HTTP/HTTPS)
```

### 4. ExternalName

```yaml
spec:
  type: ExternalName
  externalName: my-rds.cluster-abc.us-east-1.rds.amazonaws.com
# Pod query "my-db.default" → trả về CNAME → RDS endpoint
# Dùng khi: map Kubernetes Service name đến external DNS
```

---

## Ingress & AWS Load Balancer Controller

### Ingress là gì?

**Ingress** là object Kubernetes quản lý HTTP/HTTPS routing từ bên ngoài vào Services bên trong cluster — tương tự reverse proxy/API gateway.

### AWS Load Balancer Controller

**AWS Load Balancer Controller (Bộ Điều Khiển Cân Bằng Tải AWS)** là add-on chuyển Kubernetes Ingress object thành AWS ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng).

```yaml
# Ví dụ Ingress với ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip    # Route trực tiếp đến Pod IP
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123:certificate/abc
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

### ALB target-type: ip vs instance

```
target-type: instance (Mặc Định Cũ)
  ALB → NodePort trên EC2 → kube-proxy → Pod
  Thêm 1 hop, mất thông tin IP gốc của client

target-type: ip (Khuyến Nghị)
  ALB → Pod IP trực tiếp (qua VPC routing)
  Ít hop hơn, Pod nhận được client IP thực
  Yêu cầu: VPC CNI (EKS mặc định đã dùng VPC CNI)
```

### IngressClass — Chọn Controller

```yaml
# Khi cluster có nhiều Ingress controllers
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
spec:
  controller: ingress.k8s.aws/alb

# Trong Ingress spec:
spec:
  ingressClassName: alb   # Chỉ định dùng ALB controller
```

---

## Network Policies — Chính Sách Mạng

### Tại Sao Cần Network Policy?

Mặc định trong Kubernetes, **mọi Pod đều communicate được với mọi Pod** (zero trust không áp dụng). Network Policy cho phép micro-segmentation (phân đoạn vi mô) — chỉ cho phép traffic cần thiết.

> **Lưu ý:** Amazon VPC CNI không hỗ trợ Network Policy natively. Cần thêm **Calico** hoặc dùng **VPC CNI với Network Policy add-on** (AWS ra mắt tháng 8/2023).

### Ví Dụ Network Policy Thực Tế

```yaml
# Chính sách cho namespace "backend": chỉ nhận traffic từ frontend và monitoring
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: api      # Áp dụng cho Pods có label tier=api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend    # Cho phép traffic từ namespace frontend
    ports:
    - port: 8080
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring  # Cho phép Prometheus scrape
    ports:
    - port: 9090
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database    # Chỉ cho phép gọi ra database namespace
    ports:
    - port: 5432
  - to:
    - namespaceSelector: {}   # Cho phép DNS
    ports:
    - port: 53
      protocol: UDP
```

### Default Deny Policy — Chính Sách Từ Chối Mặc Định

```yaml
# Best practice: Default deny all, sau đó whitelist những gì cần
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}      # Match tất cả Pods trong namespace
  policyTypes:
  - Ingress
  - Egress
# Không có ingress/egress rules → block tất cả traffic
```

### Bật AWS VPC CNI Network Policy Add-on

```bash
# AWS native solution (không cần Calico)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy":"true"}'
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao EKS dùng VPC CNI thay vì overlay network như Flannel?

**Trả lời:**
> VPC CNI cho phép Pod nhận IP thực từ VPC subnet, không qua overlay network. Lợi ích: (1) Performance tốt hơn — không có VXLAN encapsulation overhead; (2) Pod communicate trực tiếp với AWS services (RDS, ElastiCache) mà không qua NAT; (3) Security Groups có thể áp dụng trực tiếp cho Pod level (Security Groups for Pods); (4) Đơn giản hơn để debug — không có overlay subnet phức tạp. Nhược điểm: mỗi Pod tốn 1 VPC IP → cần VPC sizing đủ lớn.

### Câu 2: Khi Pod A gọi Service "mysql.database", traffic đi như thế nào?

**Trả lời:**
> (1) Pod A query DNS "mysql.database" → CoreDNS giải quyết thành ClusterIP 10.100.x.x; (2) Pod A gửi packet đến 10.100.x.x:3306; (3) iptables rules của kube-proxy (chạy trên cùng node) DNAT packet đến một trong các Pod IP của mysql (ví dụ 10.0.2.15:3306); (4) VPC routing gửi packet trực tiếp đến Pod IP đó (vì dùng VPC CNI, là VPC IP thực); (5) mysql Pod nhận packet và phản hồi.

### Câu 3: Phân biệt Ingress và Service type LoadBalancer trong Kubernetes?

**Trả lời:**
> Service type LoadBalancer tạo một NLB/CLB riêng cho mỗi Service — phù hợp TCP/UDP non-HTTP, nhưng tốn tiền nếu có nhiều Services. Ingress là layer 7 (HTTP/HTTPS) routing controller — một ALB duy nhất có thể route đến nhiều Services dựa trên hostname và path. Ví dụ: api.example.com/users → user-service, api.example.com/orders → order-service. Ingress hiệu quả hơn về chi phí và quản lý cho HTTP workloads.

### Câu 4: Tại sao cần Network Policy trong EKS? Mặc định có an toàn không?

**Trả lời:**
> Mặc định Kubernetes không có isolation giữa Pods — mọi Pod trong cluster đều communicate được với mọi Pod khác. Đây là zero-security model. Network Policy implement micro-segmentation: frontend chỉ nói chuyện với backend, backend chỉ gọi database. Best practice là default-deny cho từng namespace, sau đó whitelist traffic cần thiết. Điều này giảm blast radius khi có compromise — attacker vào được 1 Pod không thể lateral movement sang services khác.

---

**Tiếp theo:** [4-eks-storage.md](./4-eks-storage.md) — EBS CSI Driver, EFS CSI Driver, và StatefulSets.
