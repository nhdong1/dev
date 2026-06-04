# Bài Tập Thực Hành Kubernetes — Tình Huống Thực Chiến

> 12 bài tập tình huống từ đơn giản đến phức tạp, mô phỏng các vấn đề thực tế gặp phải khi vận hành Kubernetes. Mỗi bài có đề bài, gợi ý, và lời giải.

## Yêu Cầu Môi Trường

```bash
# Dùng kind (Kubernetes IN Docker) để tạo cluster lab
kind create cluster --name k8s-lab --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Cài metrics-server (cần cho HPA bài tập)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

---

## Bài 1: Deploy Ứng Dụng 3-tier (Cấp Độ: Cơ Bản)

### Đề Bài

Deploy một web application gồm 3 tier:
- **Frontend:** nginx serving static files, 2 replicas
- **Backend:** Python Flask API, 3 replicas
- **Database:** PostgreSQL, 1 replica với persistent storage

Yêu cầu:
- Frontend có thể gọi Backend qua URL `http://backend-service:5000`
- Backend có thể kết nối PostgreSQL qua `postgresql-service:5432`
- Expose Frontend ra ngoài qua NodePort
- Database credentials lưu trong Secret

### Gợi Ý

```
1. Tạo Namespace riêng (vd: three-tier)
2. Tạo Secret cho database credentials
3. Deploy PostgreSQL với StatefulSet + PVC
4. Deploy Backend Deployment + ClusterIP Service
5. Deploy Frontend Deployment + NodePort Service
6. Test end-to-end từ NodePort
```

### Lời Giải

```yaml
# 1. Namespace
kubectl create namespace three-tier

# 2. Secret cho PostgreSQL
kubectl create secret generic postgres-credentials \
  --namespace=three-tier \
  --from-literal=POSTGRES_USER=appuser \
  --from-literal=POSTGRES_PASSWORD=S3cr3tP@ss \
  --from-literal=POSTGRES_DB=appdb

# 3. PostgreSQL StatefulSet
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: three-tier
spec:
  serviceName: postgresql-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        envFrom:
        - secretRef:
            name: postgres-credentials
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 5Gi

---
# PostgreSQL Service (headless cho StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
  namespace: three-tier
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: three-tier
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: python:3.11-slim
        command: ["python", "-m", "http.server", "5000"]
        ports:
        - containerPort: 5000
        env:
        - name: DB_HOST
          value: postgresql-service
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PASSWORD

---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: three-tier
spec:
  selector:
    app: backend
  ports:
  - port: 5000
    targetPort: 5000
```

---

## Bài 2: Debug Pod CrashLoopBackOff (Cấp Độ: Cơ Bản)

### Đề Bài

Apply manifest sau và debug để Pod chạy được:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        env:
        - name: REQUIRED_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: api-key
        - name: CONFIG_VALUE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: setting
```

**Các lỗi cần tìm:**
1. Secret `app-secret` chưa tồn tại
2. ConfigMap `app-config` chưa tồn tại
3. Có thêm lỗi ẩn khác trong cấu hình

### Lời Giải

```bash
# Kiểm tra trạng thái
kubectl get pods
kubectl describe pod broken-app-xxx
# Events sẽ cho thấy: "secret app-secret not found"

# Fix 1: Tạo Secret
kubectl create secret generic app-secret \
  --from-literal=api-key=my-secret-key-123

# Fix 2: Tạo ConfigMap
kubectl create configmap app-config \
  --from-literal=setting=production

# Sau khi apply, Pod vẫn crash? → Kiểm tra logs
kubectl logs broken-app-xxx
# nginx sẽ chạy bình thường

# Lỗi ẩn: không có readinessProbe
# Nếu image không start ngay → traffic gửi vào Pod chưa ready
```

---

## Bài 3: Cấu Hình HPA và Kiểm Tra Scale (Cấp Độ: Trung Bình)

### Đề Bài

1. Deploy một web server với resource limits
2. Cấu hình HPA scale từ 2 đến 10 Pod khi CPU > 60%
3. Generate load để trigger HPA
4. Quan sát HPA scale up và scale down

### Lời Giải

```bash
# 1. Deploy PHP server với resource limits
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m        # HPA cần request để tính %
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF

# 2. Tạo HPA
kubectl autoscale deployment php-apache \
  --cpu-percent=60 \
  --min=2 \
  --max=10

# 3. Kiểm tra HPA
kubectl get hpa php-apache --watch

# 4. Generate load (terminal khác)
kubectl run -i --tty load-generator --rm \
  --image=busybox --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

# 5. Quan sát scale up
kubectl get hpa --watch
# TARGETS: 250%/60% → HPA tính cần: ceil(2 × 250/60) = 9 Pod

# 6. Dừng load generator và quan sát scale down
# Ctrl+C trong terminal load generator
# HPA sẽ scale down sau 5 phút (stabilization window)
```

---

## Bài 4: RBAC — Tạo User Có Quyền Hạn Chế (Cấp Độ: Trung Bình)

### Đề Bài

Tạo một user `developer-john` với quyền:
- Chỉ được GET, LIST, WATCH pods và deployments trong namespace `dev`
- Không được CREATE, UPDATE, DELETE bất cứ resource nào
- Không được truy cập namespace khác (ví dụ: `production`)

Verify bằng cách kiểm tra quyền.

### Lời Giải

```bash
# 1. Tạo namespace dev
kubectl create namespace dev

# 2. Tạo certificate cho user (trong môi trường thực dùng cert-manager hoặc OIDC)
# Trong lab, ta dùng kubectl config để mô phỏng
openssl genrsa -out john.key 2048
openssl req -new -key john.key -out john.csr \
  -subj "/CN=developer-john/O=dev-team"

# Sign bằng cluster CA
openssl x509 -req -in john.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out john.crt -days 365

# 3. Tạo Role
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-deployment-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EOF

# 4. Tạo RoleBinding
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: john-pod-reader
  namespace: dev
subjects:
- kind: User
  name: developer-john
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-deployment-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# 5. Verify quyền
kubectl auth can-i get pods -n dev --as=developer-john
# → yes

kubectl auth can-i delete pods -n dev --as=developer-john
# → no

kubectl auth can-i get pods -n production --as=developer-john
# → no

# 6. Kiểm tra toàn bộ
kubectl auth can-i --list -n dev --as=developer-john
```

---

## Bài 5: Rolling Update và Rollback (Cấp Độ: Trung Bình)

### Đề Bài

1. Deploy nginx:1.24 với 4 replicas
2. Annotate Deployment với change-cause
3. Update lên nginx:1.25 — quan sát rolling update
4. Update lên `nginx:broken-image-tag` — Pod sẽ fail
5. Rollback về phiên bản hoạt động

### Lời Giải

```bash
# 1. Deploy ban đầu
kubectl create deployment web \
  --image=nginx:1.24 \
  --replicas=4

kubectl annotate deployment/web \
  kubernetes.io/change-cause="Initial deploy nginx 1.24"

# 2. Cấu hình rolling update safe
kubectl patch deployment web -p '{
  "spec": {
    "strategy": {
      "rollingUpdate": {
        "maxSurge": 1,
        "maxUnavailable": 0
      }
    }
  }
}'

# 3. Update lên nginx:1.25
kubectl set image deployment/web nginx=nginx:1.25
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade to nginx 1.25"

# Quan sát rolling update
kubectl rollout status deployment/web

# 4. Update lên image lỗi
kubectl set image deployment/web nginx=nginx:definitely-broken-tag
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Wrong image tag - will fail"

# Quan sát Pod fail
kubectl get pods --watch
# ImagePullBackOff trên các Pod mới

# 5. Kiểm tra lịch sử
kubectl rollout history deployment/web
# REVISION  CHANGE-CAUSE
# 1         Initial deploy nginx 1.24
# 2         Upgrade to nginx 1.25
# 3         Wrong image tag - will fail

# 6. Rollback về revision 2 (nginx:1.25)
kubectl rollout undo deployment/web --to-revision=2

# Verify
kubectl get pods
kubectl describe deployment web | grep Image
```

---

## Bài 6: NetworkPolicy — Cô Lập Traffic (Cấp Độ: Trung Bình)

### Đề Bài

Trong namespace `secure-app`, có 3 tier:
- `frontend` Pods
- `backend` Pods
- `database` Pods

Yêu cầu:
- Frontend chỉ được gọi Backend (port 8080)
- Backend chỉ được kết nối Database (port 5432)
- Database không nhận traffic từ Frontend
- Tất cả Pods có thể gọi DNS (UDP 53)

### Lời Giải

```bash
kubectl create namespace secure-app

# Deploy test Pods
kubectl run frontend --image=nginx -n secure-app --labels="tier=frontend"
kubectl run backend --image=nginx -n secure-app --labels="tier=backend"
kubectl run database --image=nginx -n secure-app --labels="tier=database"

# Tạo NetworkPolicy
kubectl apply -f - <<EOF
# Deny all ingress và egress mặc định
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Frontend: được gọi ra backend và DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 8080
  - ports:
    - port: 53
      protocol: UDP
---
# Backend: nhận từ frontend, gọi ra database và DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: secure-app
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
          tier: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  - ports:
    - port: 53
      protocol: UDP
---
# Database: chỉ nhận từ backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 5432
EOF

# Test: frontend có thể gọi backend
kubectl exec -n secure-app frontend -- curl -m 3 <backend-ip>:8080
# → OK

# Test: frontend không thể gọi database
kubectl exec -n secure-app frontend -- curl -m 3 <database-ip>:5432
# → Connection timed out
```

---

## Bài 7: StatefulSet với Persistent Storage (Cấp Độ: Nâng Cao)

### Đề Bài

Deploy một Redis cluster với StatefulSet:
- 3 replica (1 master, 2 slave)
- Mỗi Pod có PVC riêng 1Gi
- Headless Service cho stable DNS
- Verify mỗi Pod có identity riêng và data persistence

### Lời Giải

```yaml
# Headless Service cho stable DNS
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None      # Headless
  selector:
    app: redis
  ports:
  - port: 6379
    name: redis
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        command:
        - redis-server
        - /data/redis.conf
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 3
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi
```

```bash
# Verify identity
kubectl get pods -l app=redis
# redis-0, redis-1, redis-2 — thứ tự ổn định

# Verify DNS
kubectl exec -it redis-0 -- nslookup redis-0.redis-headless
# → Trả về IP ổn định của redis-0

# Verify persistence
kubectl exec -it redis-0 -- redis-cli SET mykey "hello"
kubectl delete pod redis-0
# Sau khi Pod restart với PVC cũ:
kubectl exec -it redis-0 -- redis-cli GET mykey
# → "hello" vẫn còn!

# Verify PVC riêng
kubectl get pvc
# redis-data-redis-0, redis-data-redis-1, redis-data-redis-2
```

---

## Bài 8: ConfigMap và Secret Management (Cấp Độ: Cơ Bản)

### Đề Bài

1. Tạo ConfigMap từ file config.properties
2. Tạo Secret với base64 encoding
3. Mount ConfigMap như file vào `/etc/config/`
4. Mount Secret như env var
5. Cập nhật ConfigMap và verify Pod nhận giá trị mới (mà không restart)

### Lời Giải

```bash
# 1. Tạo ConfigMap từ file
cat > config.properties <<EOF
app.name=MyApp
app.version=2.1.0
app.environment=production
log.level=INFO
EOF

kubectl create configmap app-config \
  --from-file=config.properties

# 2. Tạo Secret
kubectl create secret generic app-secrets \
  --from-literal=db-password="super-secret-pass" \
  --from-literal=api-token="tok_prod_xyz789"

# 3. Pod sử dụng cả hai
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: config-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do cat /etc/config/config.properties; sleep 30; done"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
    - name: API_TOKEN
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-token
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
EOF

# 4. Verify
kubectl exec config-test -- cat /etc/config/config.properties
kubectl exec config-test -- env | grep -E "DB_PASSWORD|API_TOKEN"

# 5. Update ConfigMap (live reload sau ~60 giây, không restart)
kubectl patch configmap app-config \
  --patch '{"data":{"config.properties":"app.name=MyApp\napp.version=2.2.0\nlog.level=DEBUG\n"}}'

# Sau 60 giây:
kubectl exec config-test -- cat /etc/config/config.properties
# → version đã đổi thành 2.2.0

# NOTE: Env var từ Secret/ConfigMap KHÔNG cập nhật live — cần restart Pod
```

---

## Bài 9: Ingress với Multiple Hosts và TLS (Cấp Độ: Nâng Cao)

### Đề Bài

1. Cài NGINX Ingress Controller
2. Deploy 2 service: `web-service` và `api-service`
3. Cấu hình Ingress:
   - `web.local` → web-service
   - `api.local/v1` → api-service
4. Thêm TLS cho `api.local`
5. Test bằng curl với Host header

### Lời Giải

```bash
# 1. Cài NGINX Ingress Controller (cho kind cluster)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 2. Deploy 2 service
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=API Response v1"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 5678
EOF

# 3. Tạo self-signed certificate cho api.local
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout api.key -out api.crt \
  -subj "/CN=api.local/O=test"

kubectl create secret tls api-tls \
  --key=api.key \
  --cert=api.crt

# 4. Tạo Ingress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.local
    secretName: api-tls
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 5678
EOF

# 5. Test
INGRESS_IP=$(kubectl get svc ingress-nginx-controller \
  -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -H "Host: web.local" http://$INGRESS_IP
curl -k -H "Host: api.local" https://$INGRESS_IP/v1
```

---

## Bài 10: Debug Node Pressure (Cấp Độ: Nâng Cao)

### Đề Bài

Simulate một node bị memory pressure và quan sát:
1. Pod bị evict theo thứ tự QoS class (BestEffort → Burstable → Guaranteed)
2. Cách drain node để maintenance
3. Cách cordon node để ngăn schedule mới

### Lời Giải

```bash
# 1. Deploy Pods với các QoS class khác nhau

# BestEffort (không có request/limit) → bị evict đầu tiên
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-besteffort
spec:
  containers:
  - name: app
    image: nginx
    # Không có resources block → BestEffort
EOF

# Burstable (có request nhưng request ≠ limit)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-burstable
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: 64Mi
      limits:
        memory: 128Mi
EOF

# Guaranteed (request = limit)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-guaranteed
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: 64Mi
        cpu: 100m
      limits:
        memory: 64Mi
        cpu: 100m
EOF

# 2. Kiểm tra QoS class
kubectl get pod pod-besteffort -o jsonpath='{.status.qosClass}'   # BestEffort
kubectl get pod pod-burstable -o jsonpath='{.status.qosClass}'    # Burstable
kubectl get pod pod-guaranteed -o jsonpath='{.status.qosClass}'   # Guaranteed

# 3. Cordon node (ngăn schedule Pod mới, Pod hiện tại vẫn chạy)
kubectl cordon <node-name>

kubectl get nodes
# STATUS: Ready,SchedulingDisabled

# 4. Drain node (evict tất cả Pod, chuẩn bị maintenance)
kubectl drain <node-name> \
  --ignore-daemonsets \    # Bỏ qua DaemonSet Pod (không thể evict)
  --delete-emptydir-data \ # Xoá Pod dùng emptyDir volume
  --force

# 5. Simulate maintenance xong, uncordon
kubectl uncordon <node-name>

# Kiểm tra Pod đã được reschedule
kubectl get pods -o wide
```

---

## Bài 11: Helm Chart Cơ Bản (Cấp Độ: Trung Bình)

### Đề Bài

1. Tạo Helm chart cho web application
2. Chart phải có values cho: image tag, replica count, service type
3. Deploy với `values-dev.yaml` và `values-prod.yaml` khác nhau
4. Thực hiện upgrade và rollback

### Lời Giải

```bash
# 1. Tạo chart skeleton
helm create webapp
cd webapp

# 2. Chỉnh values.yaml
cat > values.yaml <<EOF
replicaCount: 2

image:
  repository: nginx
  tag: "1.24"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
EOF

# 3. Tạo values cho các môi trường
cat > values-dev.yaml <<EOF
replicaCount: 1
image:
  tag: "1.24-dev"
service:
  type: NodePort
EOF

cat > values-prod.yaml <<EOF
replicaCount: 5
image:
  tag: "1.25"
service:
  type: LoadBalancer
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi
EOF

# 4. Deploy dev environment
helm install webapp-dev ./webapp \
  --namespace dev \
  --create-namespace \
  --values values-dev.yaml

# 5. Deploy prod environment
helm install webapp-prod ./webapp \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml

# 6. Upgrade production
helm upgrade webapp-prod ./webapp \
  --namespace production \
  --values values-prod.yaml \
  --set image.tag="1.25.1"

# 7. Kiểm tra history
helm history webapp-prod -n production

# 8. Rollback
helm rollback webapp-prod 1 -n production
```

---

## Bài 12: Multi-container Pod Patterns (Cấp Độ: Nâng Cao)

### Đề Bài

Implement Sidecar pattern: một app container ghi log vào file, một log-shipper sidecar đọc và forward log đó.

Implement Init Container pattern: app chỉ chạy sau khi database ready.

### Lời Giải

```yaml
# Sidecar Pattern: App + Log Shipper
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}        # Shared giữa 2 container

  containers:
  # Container chính: ghi log vào file
  - name: app
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - while true; do
        echo "$(date) INFO: Processing request" >> /var/log/app.log;
        sleep 5;
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  # Sidecar: đọc và ship log
  - name: log-shipper
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - tail -f /var/log/app.log | while read line; do
        echo "[SHIPPED] $line";
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

---
# Init Container Pattern
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  # Init 1: Chờ database ready
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c',
      'until nc -z postgres-service 5432;
       do echo "waiting for database..."; sleep 3;
       done;
       echo "Database is ready!"']

  # Init 2: Chạy migration
  - name: run-migration
    image: busybox
    command: ['sh', '-c', 'echo "Running database migration..."']

  # Main container chỉ chạy sau cả 2 init container thành công
  containers:
  - name: web-app
    image: nginx
    ports:
    - containerPort: 80
```

```bash
# Test Sidecar
kubectl apply -f sidecar-demo.yaml
kubectl logs sidecar-demo -c log-shipper --follow
# → Thấy log từ app container được forward realtime

# Test Init Container
kubectl apply -f init-demo.yaml
kubectl describe pod init-demo
# Events cho thấy: init container chạy trước, main container chạy sau
```

---

## Checklist Hoàn Thành Bài Tập

| Bài | Chủ Đề | Cấp Độ | Hoàn Thành |
|-----|--------|--------|-----------|
| 1 | Deploy 3-tier app | Cơ bản | [ ] |
| 2 | Debug CrashLoopBackOff | Cơ bản | [ ] |
| 3 | HPA và load testing | Trung bình | [ ] |
| 4 | RBAC và user permissions | Trung bình | [ ] |
| 5 | Rolling update và rollback | Trung bình | [ ] |
| 6 | NetworkPolicy | Trung bình | [ ] |
| 7 | StatefulSet + persistent storage | Nâng cao | [ ] |
| 8 | ConfigMap và Secret management | Cơ bản | [ ] |
| 9 | Ingress với TLS | Nâng cao | [ ] |
| 10 | Node pressure và eviction | Nâng cao | [ ] |
| 11 | Helm chart | Trung bình | [ ] |
| 12 | Multi-container patterns | Nâng cao | [ ] |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
