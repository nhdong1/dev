# Ingress — Điều Phối Traffic HTTP/HTTPS Vào Cluster

> Ingress là tài nguyên Kubernetes khai báo quy tắc điều hướng HTTP/HTTPS từ bên ngoài vào các Service bên trong cluster. Ingress Controller là thành phần thực thi các quy tắc đó.

## Mục Lục

1. [Ingress Là Gì?](#ingress-là-gì)
2. [Ingress Controller — Thành Phần Bắt Buộc](#ingress-controller--thành-phần-bắt-buộc)
3. [Routing Rules — Quy Tắc Điều Hướng](#routing-rules--quy-tắc-điều-hướng)
4. [TLS và HTTPS](#tls-và-https)
5. [Annotation Phổ Biến](#annotation-phổ-biến)
6. [cert-manager — Quản Lý Chứng Chỉ Tự Động](#cert-manager--quản-lý-chứng-chỉ-tự-động)
7. [IngressClass — Chọn Controller](#ingressclass--chọn-controller)
8. [Các Ingress Controller Phổ Biến](#các-ingress-controller-phổ-biến)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Ingress Là Gì?

**Ingress** giải quyết bài toán: làm sao để một (hoặc vài) IP public phục vụ nhiều service HTTP khác nhau dựa trên domain và path.

### Không Có Ingress

```
service-A → LoadBalancer (IP: 1.2.3.4)  → $$$
service-B → LoadBalancer (IP: 1.2.3.5)  → $$$
service-C → LoadBalancer (IP: 1.2.3.6)  → $$$
```

### Có Ingress

```
Ingress Controller → LoadBalancer (IP: 1.2.3.4)  → $ (một load balancer duy nhất)
    ├── api.example.com      → service-A
    ├── web.example.com      → service-B
    └── web.example.com/api  → service-C
```

### Ingress Hoạt Động Ở Tầng L7

Ingress là **L7 proxy** (tầng ứng dụng) — có thể đọc nội dung HTTP header, path, host để đưa ra quyết định routing. Khác với Service LoadBalancer chỉ hoạt động ở tầng L4 (TCP/UDP).

---

## Ingress Controller — Thành Phần Bắt Buộc

**Ingress resource** chỉ là khai báo (YAML). Cần **Ingress Controller** — một Pod chạy trong cluster — để đọc khai báo và thực sự xử lý traffic.

> Kubernetes không cài sẵn Ingress Controller. Bạn phải tự cài.

### Cài NGINX Ingress Controller (Phổ Biến Nhất)

```bash
# Cài bằng Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Kiểm tra
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx ingress-nginx-controller
```

### Kiến Trúc NGINX Ingress Controller

```
Request HTTPS → NGINX Pod (Ingress Controller)
                    │
                    ├── đọc Ingress resources qua K8s API
                    ├── tạo nginx.conf tương ứng
                    └── proxy_pass đến Service backend
```

---

## Routing Rules — Quy Tắc Điều Hướng

### Path-Based Routing (Điều Hướng Theo Path)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix      # khớp /v1, /v1/users, /v1/orders...
            backend:
              service:
                name: api-v1-service
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 80
          - path: /
            pathType: Prefix      # catch-all — bắt tất cả còn lại
            backend:
              service:
                name: web-service
                port:
                  number: 3000
```

### Host-Based Routing (Điều Hướng Theo Domain)

```yaml
spec:
  rules:
    - host: api.example.com          # subdomain API
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: admin.example.com        # subdomain admin
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 8080
```

### PathType — Kiểu Khớp Path

| PathType | Ý Nghĩa | Ví Dụ Path | Khớp |
| -------- | ------- | ---------- | ---- |
| `Exact` | Khớp chính xác | `/api` | Chỉ `/api`, không khớp `/api/` |
| `Prefix` | Khớp prefix (tiền tố) | `/api` | `/api`, `/api/users`, `/api/v1/orders` |
| `ImplementationSpecific` | Phụ thuộc controller | Regex với NGINX | Khác nhau theo controller |

### Default Backend (Backend Dự Phòng)

```yaml
spec:
  defaultBackend:           # trả về khi không có rule nào khớp
    service:
      name: not-found-service
      port:
        number: 80
  rules:
    - host: api.example.com
      # ...
```

---

## TLS và HTTPS

### Tạo TLS Secret Thủ Công

```bash
# Tạo self-signed certificate (chứng chỉ tự ký) để test
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=api.example.com/O=MyOrg"

# Tạo Secret từ certificate
kubectl create secret tls api-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=production
```

### Cấu Hình TLS Trong Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress-tls
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
        - admin.example.com
      secretName: api-tls-secret    # Secret chứa cert và key
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

### HTTP → HTTPS Redirect

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"     # tự động redirect HTTP → HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

### TLS Termination — Chấm Dứt TLS Tại Ingress

```
Client → HTTPS → Ingress Controller (terminate TLS tại đây)
                       │
                       └── HTTP → Service → Pod
```

TLS được giải mã tại Ingress Controller. Traffic từ Ingress đến Pod là HTTP thông thường (trong cluster network). Nếu cần encrypt toàn bộ đường đi (end-to-end TLS), cần cấu hình thêm backend protocol HTTPS và certificate cho Pod.

---

## Annotation Phổ Biến

Ingress Controller mở rộng chức năng qua **annotation** (ghi chú) trên metadata.

### NGINX Ingress Controller

```yaml
metadata:
  annotations:
    # Rate limiting — giới hạn tốc độ request
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"

    # Timeout
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"

    # Kích thước body upload
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # CORS (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Nguồn Gốc Chéo)
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"

    # Rewrite path — viết lại đường dẫn
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # Auth — xác thực basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Protected Area"

    # Whitelist IP (cho phép IP cụ thể)
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.1.0/24"

    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

### Path Rewrite Ví Dụ

```yaml
# Request: /service-a/api/users
# Sau rewrite: /api/users (bỏ prefix /service-a)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - http:
        paths:
          - path: /service-a(/|$)(.*)    # capture group $2
            pathType: ImplementationSpecific
            backend:
              service:
                name: service-a
                port:
                  number: 80
```

---

## cert-manager — Quản Lý Chứng Chỉ Tự Động

**cert-manager** là Kubernetes operator tự động xin và gia hạn certificate TLS từ Let's Encrypt (hoặc CA khác).

### Cài cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Tạo ClusterIssuer (Nhà Cấp Chứng Chỉ)

```yaml
# Let's Encrypt Production Issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx    # hoặc ingressClassName: nginx
```

### Ingress Với cert-manager (Tự Động Lấy Certificate)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod    # trigger cert-manager
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert    # cert-manager tạo Secret này tự động
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

### Quy Trình cert-manager Lấy Certificate

```
1. cert-manager phát hiện Ingress có annotation cluster-issuer
2. Tạo Certificate resource và CertificateRequest
3. Giải ACME challenge (HTTP-01): tạo Pod tạm thời trả lời challenge từ Let's Encrypt
4. Let's Encrypt xác nhận domain ownership → cấp certificate
5. cert-manager lưu cert vào Secret (secretName trong spec.tls)
6. NGINX đọc Secret, serve HTTPS
7. cert-manager tự gia hạn khi certificate sắp hết hạn (30 ngày trước)
```

---

## IngressClass — Chọn Controller

Khi cluster có nhiều Ingress Controller (NGINX + Traefik chẳng hạn), dùng **IngressClass** để chỉ định controller nào xử lý Ingress resource nào.

### Tạo IngressClass

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # dùng khi Ingress không chỉ định class
spec:
  controller: k8s.io/ingress-nginx
```

### Sử Dụng Trong Ingress

```yaml
spec:
  ingressClassName: nginx    # chỉ định rõ controller
  rules:
    # ...
```

---

## Các Ingress Controller Phổ Biến

### NGINX Ingress Controller

| Đặc Điểm | Giá Trị |
| --------- | ------- |
| Dự án | kubernetes/ingress-nginx (cộng đồng) |
| Nền tảng | NGINX |
| Phổ biến | Rất cao |
| Tính năng | Rate limiting, auth, rewrite, CORS, custom snippets |
| Phù hợp | Hầu hết use case, on-premise và cloud |

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

### Traefik

| Đặc Điểm | Giá Trị |
| --------- | ------- |
| Nền tảng | Go, native cloud |
| Dashboard | Có UI web |
| Auto-discovery | Tự phát hiện service qua label |
| Phù hợp | Microservices, cần dashboard |

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm install traefik traefik/traefik -n traefik --create-namespace
```

### AWS Load Balancer Controller (ALB Ingress)

Dùng trên **EKS**, tạo AWS Application Load Balancer thật thay vì dùng NGINX.

| Đặc Điểm | Giá Trị |
| --------- | ------- |
| Chạy trên | Amazon EKS |
| Tạo | AWS ALB / NLB thật |
| Tích hợp | AWS WAF, ACM (chứng chỉ), Cognito |
| Chi phí | Theo chi phí ALB của AWS |

```yaml
# Annotation đặc trưng của AWS ALB Controller
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
```

### So Sánh Nhanh

| | NGINX | Traefik | AWS ALB |
| - | ----- | ------- | ------- |
| Phổ biến | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ (EKS) |
| Cấu hình | Annotation | Label/CRD | Annotation |
| TLS Auto | Với cert-manager | Native ACME | ACM tích hợp |
| Dashboard | Không | Có | AWS Console |
| Rate Limiting | Annotation | Middleware | WAF |
| On-Premise | ✅ | ✅ | ❌ |

---

## Câu Hỏi Phỏng Vấn

**Ingress và Service LoadBalancer khác nhau thế nào?**

> LoadBalancer hoạt động ở tầng L4 (TCP/UDP), mỗi Service cần một cloud LB riêng (tốn tiền). Ingress hoạt động ở tầng L7 (HTTP), một Ingress Controller duy nhất phục vụ nhiều service qua virtual hosting và path routing. Ingress cũng hỗ trợ TLS termination, header manipulation, rate limiting — những thứ LoadBalancer không làm được.

**Nếu Ingress Controller bị xoá, điều gì xảy ra với traffic?**

> Traffic sẽ bị mất hoàn toàn. Ingress resource vẫn tồn tại trong etcd nhưng không có gì xử lý nó. Đây là single point of failure nếu không có high availability. Trong production, chạy Ingress Controller với ít nhất 2 replica và PodDisruptionBudget để đảm bảo rolling update không gây downtime.

**Tại sao cần cert-manager thay vì tự quản lý certificate?**

> cert-manager tự động hoá toàn bộ vòng đời certificate: xin, validate domain, lưu vào Secret, và gia hạn trước khi hết hạn. Quản lý thủ công rất dễ quên gia hạn (certificate hết hạn = downtime) và tốn công trên cluster nhiều domain.

**Làm sao debug Ingress không hoạt động?**

```bash
# 1. Kiểm tra Ingress resource đã được tạo đúng
kubectl describe ingress my-ingress

# 2. Kiểm tra Ingress Controller có đang chạy không
kubectl get pods -n ingress-nginx

# 3. Xem log của Ingress Controller
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

# 4. Kiểm tra Service backend tồn tại và có Endpoint
kubectl get endpoints my-service

# 5. Test từ trong cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -H "Host: api.example.com" http://<ingress-controller-cluster-ip>/

# 6. Kiểm tra IngressClass đã đúng
kubectl get ingressclass
```
