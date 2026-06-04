# Service Types — Các Loại Service Kubernetes

> Service là tài nguyên Kubernetes tạo một địa chỉ ổn định (ClusterIP) đại diện cho một nhóm Pod có thể thay đổi. Có bốn loại Service với phạm vi phơi bày khác nhau.

## Mục Lục

1. [Service Là Gì?](#service-là-gì)
2. [ClusterIP — Giao Tiếp Nội Bộ](#clusterip--giao-tiếp-nội-bộ)
3. [NodePort — Phơi Bày Ra Node](#nodeport--phơi-bày-ra-node)
4. [LoadBalancer — Phơi Bày Ra Internet](#loadbalancer--phơi-bày-ra-internet)
5. [ExternalName — Alias DNS Ngoài Cluster](#externalname--alias-dns-ngoài-cluster)
6. [Headless Service — Không ClusterIP](#headless-service--không-clusterip)
7. [Selector và Endpoint](#selector-và-endpoint)
8. [So Sánh Và Khi Nào Dùng](#so-sánh-và-khi-nào-dùng)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Service Là Gì?

Pod trong Kubernetes là **ephemeral** (ngắn hạn) — có thể bị xoá và tạo lại bất cứ lúc nào với IP mới. **Service** giải quyết vấn đề này bằng cách cung cấp:

- **Địa chỉ IP ổn định** (ClusterIP) không thay đổi dù Pod thay đổi
- **Load balancing** (cân bằng tải) tự động giữa các Pod backend
- **Service discovery** (khám phá dịch vụ) qua DNS với CoreDNS

### Cơ Chế Hoạt Động

```
1. Service được tạo → API Server ghi vào etcd
2. kube-proxy trên mỗi node đọc Service và Endpoints
3. kube-proxy ghi iptables / IPVS rules
4. Khi Pod gửi request đến ClusterIP → kernel intercept
5. iptables / IPVS chọn Pod backend và forward gói tin
```

### Endpoint và EndpointSlice

Khi Service có selector, Kubernetes tự tạo **Endpoint** (điểm cuối) — danh sách IP:Port của các Pod khớp selector.

```bash
# Xem endpoints của một service
kubectl get endpoints my-service

# Kết quả mẫu:
# NAME         ENDPOINTS                           AGE
# my-service   10.244.1.5:8080,10.244.2.3:8080    5m
```

---

## ClusterIP — Giao Tiếp Nội Bộ

**ClusterIP** là loại Service mặc định. Tạo một IP ảo chỉ có thể truy cập **bên trong cluster**.

### Khi Nào Dùng

- Giao tiếp giữa các microservice trong cluster
- Backend service không cần phơi bày ra ngoài
- Database, cache (Redis, Postgres) chỉ dùng nội bộ

### Manifest Ví Dụ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: default
spec:
  type: ClusterIP        # có thể bỏ qua, đây là mặc định
  selector:
    app: api             # khớp với label của Pod
  ports:
    - name: http
      port: 80           # port của Service (ClusterIP:80)
      targetPort: 8080   # port thực của container
      protocol: TCP
```

### Truy Cập Từ Pod Khác

```bash
# Từ Pod trong cùng namespace
curl http://api-service/endpoint

# Từ Pod ở namespace khác
curl http://api-service.default.svc.cluster.local/endpoint

# Test từ Pod tạm thời
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://api-service.default.svc.cluster.local
```

---

## NodePort — Phơi Bày Ra Node

**NodePort** mở một port cố định trên **mọi node** trong cluster. Traffic đến `<AnyNodeIP>:<NodePort>` được forward vào Service.

### Khi Nào Dùng

- Môi trường dev/test không có cloud load balancer
- Cần truy cập nhanh từ bên ngoài mà không cần cài thêm gì
- On-premise cluster không có cloud provider

### Hạn Chế

- Port range hạn chế: `30000–32767`
- Client phải biết IP của node (không có single entry point — điểm vào duy nhất)
- Nếu node bị xoá, client phải cập nhật IP
- Không phù hợp production vì không có health check ở tầng node

### Manifest Ví Dụ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80           # ClusterIP port (dùng nội bộ)
      targetPort: 3000   # port container
      nodePort: 30080    # port trên node (30000–32767); bỏ qua để K8s tự chọn
```

### Luồng Traffic

```
Internet
    │
    ▼ port 30080
Node 1 (192.168.1.10:30080) ──┐
Node 2 (192.168.1.11:30080) ──┼──► kube-proxy ──► Pod A hoặc Pod B
Node 3 (192.168.1.12:30080) ──┘
```

> Traffic có thể đến node 1 nhưng Pod nằm trên node 2 — kube-proxy tự route qua internal network.

---

## LoadBalancer — Phơi Bày Ra Internet

**LoadBalancer** kế thừa NodePort và **yêu cầu cloud provider** tạo một load balancer thật bên ngoài cluster, cấp IP public/DNS.

### Khi Nào Dùng

- Production service cần phơi bày TCP/UDP ra internet
- Ứng dụng không phải HTTP (game server, WebSocket thuần, gRPC trực tiếp)
- Khi không muốn dùng Ingress (ví dụ: mỗi service cần IP public riêng)

### Hạn Chế

- Mỗi Service LoadBalancer tạo ra một cloud load balancer riêng → **tốn tiền**
- Chỉ hoạt động trên cloud có controller (EKS, GKE, AKS); không hoạt động trên bare-metal trừ khi có MetalLB
- Với HTTP/HTTPS thông thường, dùng Ingress tiết kiệm hơn

### Manifest Ví Dụ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # Network Load Balancer trên AWS
spec:
  type: LoadBalancer
  selector:
    app: game-server
  ports:
    - port: 7777
      targetPort: 7777
      protocol: UDP
```

### Kết Quả Sau Khi Tạo

```bash
kubectl get service game-server

# NAME          TYPE           CLUSTER-IP    EXTERNAL-IP        PORT(S)          AGE
# game-server   LoadBalancer   10.96.45.22   a1b2c3.elb.aws..   7777:31234/UDP   2m
```

### MetalLB — LoadBalancer Trên Bare-Metal

Nếu chạy cluster on-premise (không có cloud provider), dùng **MetalLB** để cung cấp LoadBalancer IP từ một pool IP nội bộ.

```yaml
# MetalLB IPAddressPool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.250  # range IP dành cho LoadBalancer
```

---

## ExternalName — Alias DNS Ngoài Cluster

**ExternalName** tạo một Service không có selector và không có ClusterIP. Thay vào đó, nó tạo một **CNAME** DNS trỏ đến tên miền bên ngoài.

### Khi Nào Dùng

- Kết nối đến database bên ngoài cluster (RDS, Cloud SQL)
- Kết nối đến API bên ngoài bằng tên thay vì IP
- Chuẩn bị migrate service từ ngoài vào trong cluster

### Manifest Ví Dụ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.us-east-1.rds.amazonaws.com  # CNAME target
```

### Cách Dùng

```python
# Code trong Pod dùng service name thay vì hostname thật
db_host = "external-db.production.svc.cluster.local"

# CoreDNS phân giải → CNAME → mydb.us-east-1.rds.amazonaws.com
# Khi migrate DB vào cluster, chỉ cần đổi Service, không cần đổi code
```

---

## Headless Service — Không ClusterIP

Headless Service khai báo `clusterIP: None`. CoreDNS trả về **tất cả IP của Pod** thay vì một ClusterIP duy nhất.

### Khi Nào Dùng

- StatefulSet: client cần kết nối trực tiếp đến Pod cụ thể (Pod-0, Pod-1)
- Database cluster với replication (PostgreSQL HA, MongoDB)
- Service discovery tùy chỉnh (gRPC load balancing phía client)

### Manifest Ví Dụ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None         # ← đây là điểm khác biệt
  selector:
    app: postgres
  ports:
    - port: 5432
```

### DNS Response So Sánh

```bash
# Service thông thường (ClusterIP)
nslookup postgres-service
→ 10.96.45.22  (ClusterIP duy nhất)

# Headless Service
nslookup postgres-headless
→ 10.244.1.5   (Pod 0 — postgres-0)
→ 10.244.2.3   (Pod 1 — postgres-1)
→ 10.244.3.7   (Pod 2 — postgres-2)
```

### Tên DNS Của StatefulSet Pod

Với Headless Service kết hợp StatefulSet, mỗi Pod có DNS riêng:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
postgres-0.postgres-headless.default.svc.cluster.local
postgres-1.postgres-headless.default.svc.cluster.local
```

---

## Selector và Endpoint

### Service Với Selector (Phổ Biến)

Service tự động tạo và cập nhật Endpoint khi Pod có label khớp selector.

```yaml
spec:
  selector:
    app: api      # khớp Pod có label app=api
    env: prod     # VÀ label env=prod
```

### Service Không Có Selector (Custom Endpoint)

Dùng khi muốn trỏ Service đến IP cố định (database on-premise, IP bên ngoài).

```yaml
# Service không có selector
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  ports:
    - port: 5432
---
# Endpoint thủ công
apiVersion: v1
kind: Endpoints
metadata:
  name: legacy-db  # phải trùng tên Service
subsets:
  - addresses:
      - ip: 10.10.0.100  # IP của server ngoài cluster
    ports:
      - port: 5432
```

---

## So Sánh Và Khi Nào Dùng

| Loại | Phạm Vi | Cloud LB | Dùng Khi |
| ---- | ------- | -------- | -------- |
| **ClusterIP** | Nội bộ cluster | Không | Microservice giao tiếp nội bộ |
| **NodePort** | Node IP + port | Không | Dev/test, on-premise không có LB |
| **LoadBalancer** | Internet | Có | TCP/UDP cần IP public, production |
| **ExternalName** | CNAME DNS | Không | Alias dịch vụ ngoài cluster |
| **Headless** | Nội bộ (IP trực tiếp) | Không | StatefulSet, gRPC client-side LB |

### Nguyên Tắc Chọn Service Type

```
HTTP/HTTPS ra internet → Ingress (+ Service ClusterIP backend)
TCP/UDP ra internet → Service LoadBalancer
Nội bộ cluster → Service ClusterIP
Dev/test nhanh → Service NodePort
StatefulSet / client-side LB → Headless Service
External DB/service → Service ExternalName
```

---

## Câu Hỏi Phỏng Vấn

**Tại sao không dùng LoadBalancer cho mọi thứ?**

> Mỗi Service LoadBalancer tạo một cloud load balancer riêng, tốn tiền (thường $20–50/tháng/LB trên AWS). Với HTTP/HTTPS, một Ingress Controller duy nhất có thể phục vụ hàng chục service, rẻ hơn nhiều. LoadBalancer phù hợp khi cần IP/port riêng biệt cho traffic không phải HTTP.

**Điều gì xảy ra khi tất cả Pod backend của một Service bị xoá?**

> Service vẫn tồn tại với ClusterIP, nhưng Endpoint trở thành rỗng. Request đến Service sẽ không được forward và kết nối bị từ chối (connection refused). Kubernetes không xoá Service tự động.

**Sự khác biệt giữa `port`, `targetPort`, và `nodePort`?**

> - `port`: port của Service (ClusterIP), client trong cluster dùng port này
> - `targetPort`: port thực của container trong Pod, traffic được forward đến đây
> - `nodePort`: port mở trên mỗi node (30000–32767), chỉ có ý nghĩa với NodePort/LoadBalancer

**sessionAffinity là gì và khi nào dùng?**

> `sessionAffinity: ClientIP` khiến kube-proxy luôn forward request từ cùng một client IP đến cùng một Pod (sticky session — phiên gắn kết). Dùng khi ứng dụng có session state phía server (WebSocket, upload lớn). Nhược điểm: phân tải không đều nếu ít client.
