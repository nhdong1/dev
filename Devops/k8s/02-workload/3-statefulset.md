# StatefulSet — Workload Có Trạng Thái

> StatefulSet quản lý Pod với danh tính mạng ổn định, lưu trữ bền vững và thứ tự triển khai được đảm bảo — dành cho các ứng dụng có trạng thái như database, message queue, distributed cache.

## Mục Lục

1. [StatefulSet Là Gì?](#statefulset-là-gì)
2. [StatefulSet vs Deployment](#statefulset-vs-deployment)
3. [Headless Service — Dịch Vụ Không Có ClusterIP](#headless-service--dịch-vụ-không-có-clusterip)
4. [VolumeClaimTemplate — Template Yêu Cầu Lưu Trữ](#volumeclaimtemplate--template-yêu-cầu-lưu-trữ)
5. [Thứ Tự Tạo và Xoá Pod](#thứ-tự-tạo-và-xoá-pod)
6. [Update Strategy](#update-strategy)
7. [Ví Dụ Thực Tế: PostgreSQL HA](#ví-dụ-thực-tế-postgresql-ha)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## StatefulSet Là Gì?

**StatefulSet** là workload controller cho ứng dụng cần:

1. **Stable network identity** (danh tính mạng ổn định): tên hostname không đổi dù Pod restart
2. **Stable persistent storage** (lưu trữ bền vững ổn định): mỗi Pod có PVC riêng, không bị xoá khi Pod restart
3. **Ordered deployment and scaling** (triển khai và mở rộng theo thứ tự): Pod-0 phải Ready trước khi tạo Pod-1
4. **Ordered rolling updates** (cập nhật cuốn theo thứ tự): cập nhật từ Pod có index lớn nhất về 0

**Ứng dụng phù hợp:**
- Database: MySQL, PostgreSQL, MongoDB
- Distributed cache (bộ nhớ đệm phân tán): Redis Cluster, Memcached
- Message queue (hàng đợi tin nhắn): Kafka, RabbitMQ
- Coordination service: Zookeeper, etcd (self-hosted)
- Search engine: Elasticsearch
- Bất kỳ ứng dụng nào cần leader election (bầu chọn leader)

---

## StatefulSet vs Deployment

| Đặc Điểm | Deployment | StatefulSet |
| --------- | ---------- | ----------- |
| Tên Pod | Ngẫu nhiên: `app-7d4b8-xk9p2` | Theo thứ tự: `app-0`, `app-1` |
| Hostname | Ngẫu nhiên, không ổn định | Ổn định: `app-0.svc.namespace.svc.cluster.local` |
| Thứ tự tạo Pod | Tuỳ ý | Tuần tự: 0 → 1 → 2 |
| Thứ tự xoá Pod | Tuỳ ý | Ngược lại: 2 → 1 → 0 |
| Lưu trữ | Dùng chung hoặc không có | Mỗi Pod có PVC riêng |
| Headless Service | Không cần | Bắt buộc |
| Dùng cho | Stateless app | Stateful app (database, MQ) |

---

## Headless Service — Dịch Vụ Không Có ClusterIP

StatefulSet **bắt buộc** có một **Headless Service** (dịch vụ không có ClusterIP — `clusterIP: None`) để cung cấp DNS cho từng Pod.

### Tại Sao Cần Headless Service?

Service thông thường tạo một virtual IP (VIP) và load balance traffic đến các Pod. Headless Service thay vào đó trả về DNS record trực tiếp cho từng Pod, cho phép client giao tiếp với Pod cụ thể — điều cần thiết khi:
- Replica cần biết địa chỉ của primary/replica khác
- Client cần kết nối đến đúng một Pod (ví dụ: database read replica)
- Application-level load balancing thay vì network-level

### DNS Record Được Tạo

Với StatefulSet tên `postgres` trong namespace `default`, headless service tên `postgres-headless`:

```
# DNS cho từng Pod
postgres-0.postgres-headless.default.svc.cluster.local
postgres-1.postgres-headless.default.svc.cluster.local
postgres-2.postgres-headless.default.svc.cluster.local

# DNS cho service (trả về A records của tất cả Pod)
postgres-headless.default.svc.cluster.local
```

### Manifest Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: default
  labels:
    app: postgres
spec:
  clusterIP: None         # đây là điểm khác biệt — không có ClusterIP
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

---

## VolumeClaimTemplate — Template Yêu Cầu Lưu Trữ

`volumeClaimTemplates` (template yêu cầu lưu trữ) tự động tạo PVC riêng cho mỗi Pod khi Pod được tạo.

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 10Gi
```

Kubernetes sẽ tạo:
- `data-postgres-0` (PVC cho postgres-0)
- `data-postgres-1` (PVC cho postgres-1)
- `data-postgres-2` (PVC cho postgres-2)

**Quan trọng:** Khi xoá StatefulSet, các PVC **không bị xoá tự động**. Phải xoá PVC thủ công. Đây là hành vi cố ý để bảo vệ dữ liệu.

```bash
# Xoá StatefulSet nhưng giữ PVC (hành vi mặc định)
kubectl delete statefulset postgres

# Xoá PVC thủ công sau khi đã backup dữ liệu
kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2
```

---

## Thứ Tự Tạo và Xoá Pod

### Tạo Pod (Scale Up)

Pod được tạo **tuần tự** từ index 0: Pod-0 phải ở trạng thái `Running and Ready` trước khi Pod-1 bắt đầu tạo.

```
Tạo:  postgres-0 → (chờ Ready) → postgres-1 → (chờ Ready) → postgres-2
```

**Tại sao quan trọng?** Database cluster cần primary (postgres-0) khởi động và sẵn sàng trước khi replica (postgres-1, postgres-2) kết nối vào.

### Xoá Pod (Scale Down)

Pod được xoá **ngược lại**: từ index cao nhất về 0. Pod-2 phải kết thúc trước khi Pod-1 bị xoá.

```
Xoá:  postgres-2 → (chờ Terminated) → postgres-1 → (chờ Terminated)
```

### podManagementPolicy — Chính Sách Quản Lý Pod

```yaml
spec:
  podManagementPolicy: OrderedReady   # mặc định — tuần tự
  # hoặc
  podManagementPolicy: Parallel       # tạo/xoá tất cả Pod cùng lúc, dùng khi ứng dụng không cần thứ tự
```

`Parallel` phù hợp khi ứng dụng không yêu cầu thứ tự (ví dụ: stateless app đóng gói trong StatefulSet vì cần tên hostname ổn định).

---

## Update Strategy

### RollingUpdate (Mặc Định)

Cập nhật từ Pod có index **cao nhất** về **thấp nhất** (ngược với khi tạo).

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0   # cập nhật tất cả Pod từ index >= 0
```

**Partition** (phân vùng) cho phép **canary update** (cập nhật thử nghiệm):

```yaml
rollingUpdate:
  partition: 2   # chỉ cập nhật Pod có index >= 2 (tức là chỉ postgres-2)
                 # postgres-0 và postgres-1 giữ nguyên phiên bản cũ
```

Dùng để kiểm tra phiên bản mới trên một Pod trước khi rollout toàn bộ.

### OnDelete

Pod chỉ được cập nhật khi bạn **xoá thủ công** Pod đó. Phù hợp khi cần kiểm soát hoàn toàn thời điểm update.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

---

## Ví Dụ Thực Tế: PostgreSQL HA

```yaml
# ─── Headless Service ─────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: database
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432

---
# ─── Service cho read/write (trỏ đến primary) ─────────
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: database
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432

---
# ─── StatefulSet ──────────────────────────────────────
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database

spec:
  serviceName: postgres-headless   # tên Headless Service — bắt buộc
  replicas: 3
  podManagementPolicy: OrderedReady

  selector:
    matchLabels:
      app: postgres

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0

  template:
    metadata:
      labels:
        app: postgres

    spec:
      terminationGracePeriodSeconds: 60   # PostgreSQL cần thời gian flush write-ahead log

      initContainers:
        # Init container xác định vai trò (primary hay replica) dựa trên index
        - name: init-role
          image: postgres:15
          command:
            - bash
            - -c
            - |
              ORDINAL=$(hostname | awk -F'-' '{print $NF}')
              if [ "$ORDINAL" = "0" ]; then
                echo "primary" > /data/role
              else
                echo "replica" > /data/role
              fi
          volumeMounts:
            - name: data
              mountPath: /data

      containers:
        - name: postgres
          image: postgres:15
          ports:
            - name: postgres
              containerPort: 5432

          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /data/pgdata
            # Truyền hostname để script biết đây là Pod thứ mấy
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name

          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"

          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - $(POSTGRES_USER)
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - $(POSTGRES_USER)
            initialDelaySeconds: 5
            periodSeconds: 10

          volumeMounts:
            - name: data
              mountPath: /data

  # Tự động tạo PVC cho mỗi Pod
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 20Gi
```

### DNS Resolution Sau Khi Tạo

```bash
# Từ bất kỳ Pod nào trong cluster
nslookup postgres-0.postgres-headless.database.svc.cluster.local
# → 10.244.1.10

nslookup postgres-1.postgres-headless.database.svc.cluster.local
# → 10.244.2.15

# Kết nối đến primary
psql -h postgres-0.postgres-headless.database.svc.cluster.local -U myuser myapp
```

---

## Câu Hỏi Phỏng Vấn

### Q: Tại sao không dùng Deployment cho database?

Deployment không đảm bảo danh tính Pod ổn định — mỗi lần restart Pod có tên và IP mới. Database cluster (như MySQL replication) cần biết địa chỉ của primary và replica để cấu hình replication. StatefulSet cung cấp DNS ổn định (`mysql-0.mysql-headless`) không thay đổi dù Pod restart.

Ngoài ra, Deployment không có cơ chế tạo PVC riêng cho từng Pod — nếu dùng shared PVC thì nhiều Pod database ghi vào cùng lưu trữ sẽ gây corruption. StatefulSet tạo PVC độc lập cho mỗi Pod.

### Q: Điều gì xảy ra với PVC khi xoá StatefulSet?

PVC **không bị xoá** — đây là thiết kế cố ý để bảo vệ dữ liệu. Sau khi xoá StatefulSet, các PVC và dữ liệu vẫn tồn tại. Nếu tạo lại StatefulSet với cùng tên, các Pod mới sẽ tự động mount lại PVC cũ và phục hồi dữ liệu.

### Q: Headless Service là gì và tại sao StatefulSet cần nó?

Headless Service là Service với `clusterIP: None` — không tạo virtual IP mà trả về DNS A record trực tiếp của từng Pod. StatefulSet cần Headless Service để:
1. Cấp DNS stable hostname cho từng Pod (`pod-0.service.namespace.svc.cluster.local`)
2. Cho phép pod-to-pod communication (ví dụ: replica kết nối đến primary theo hostname)
3. Cho phép client kết nối đến Pod cụ thể thay vì được load balance ngẫu nhiên

### Q: podManagementPolicy: Parallel dùng khi nào?

Khi ứng dụng không cần thứ tự khởi động. Ví dụ: ứng dụng cần tên hostname ổn định (để URL cố định) nhưng không phụ thuộc lẫn nhau trong quá trình khởi động. `Parallel` giảm thời gian deploy khi scale up từ 0 lên nhiều replica.

---

**Xem Tiếp:** [daemonset.md](./daemonset.md) — Chạy Agent Trên Mọi Node
