# DaemonSet — Chạy Agent Trên Mọi Node

> DaemonSet đảm bảo mỗi node (hoặc tập node được chọn) chạy đúng một bản sao của một Pod — lý tưởng cho các agent cần có mặt trên toàn bộ infrastructure như log collector, monitoring agent, network plugin.

## Mục Lục

1. [DaemonSet Là Gì?](#daemonset-là-gì)
2. [So Sánh Với Deployment](#so-sánh-với-deployment)
3. [Chọn Node Chạy DaemonSet](#chọn-node-chạy-daemonset)
4. [Update Strategy](#update-strategy)
5. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
6. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DaemonSet Là Gì?

**DaemonSet** là workload controller đảm bảo:
- Khi node **mới** được thêm vào cluster → Pod tự động được tạo trên node đó
- Khi node **bị xoá** → Pod cũng được thu hồi
- Mỗi node **phù hợp** chạy đúng **một** Pod

```
Cluster (3 nodes)
│
├── Node 1 ──→ DaemonSet Pod (fluentd-xk9p2)
├── Node 2 ──→ DaemonSet Pod (fluentd-r7m4n)
└── Node 3 ──→ DaemonSet Pod (fluentd-h2v8q)

Khi thêm Node 4:
└── Node 4 ──→ DaemonSet Pod (fluentd-p3q7s)  ← tự động tạo
```

### Trường Hợp Sử Dụng Thực Tế

| Loại Agent | Ví Dụ | Lý Do Dùng DaemonSet |
| ---------- | ------ | -------------------- |
| **Log collector** (thu thập log) | Fluentd, Filebeat, Fluent Bit | Cần thu log từ mọi node |
| **Monitoring agent** (agent giám sát) | Prometheus Node Exporter, Datadog Agent | Cần metric từ mọi node |
| **Network plugin** (plugin mạng) | Calico node, Flannel, Cilium | Cần cấu hình network trên mọi node |
| **Storage driver** (trình điều khiển lưu trữ) | GlusterFS, Ceph | Cần chạy trên mọi node có storage |
| **Security agent** (agent bảo mật) | Falco, Sysdig Agent | Cần monitor syscall trên mọi node |
| **Node-level proxy** | kube-proxy | Quản lý iptables trên mọi node |

---

## So Sánh Với Deployment

| Tiêu Chí | Deployment | DaemonSet |
| --------- | ---------- | --------- |
| Số lượng Pod | Khai báo cố định (`replicas: 3`) | Một Pod mỗi node phù hợp |
| Khi thêm node | Không thay đổi | Tự động tạo Pod trên node mới |
| Khi xoá node | Không thay đổi | Tự động xoá Pod trên node đó |
| Scheduling | Scheduler chọn node | DaemonSet controller bypass scheduler |
| Dùng cho | Ứng dụng stateless | Agent cần hiện diện trên mọi node |

**DaemonSet bypass scheduler:** Kubernetes scheduler bình thường chọn node dựa trên resource. DaemonSet controller đặt trực tiếp Pod vào đúng node, không qua quá trình scheduling thông thường. Kết quả là DaemonSet Pod có thể chạy ngay cả trên node bị taint `NoSchedule`.

---

## Chọn Node Chạy DaemonSet

Mặc định, DaemonSet chạy trên **tất cả** node. Có ba cách để giới hạn:

### 1. nodeSelector — Chọn Node Theo Label

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd          # chỉ chạy trên node có label disktype=ssd
        environment: production
```

```bash
# Gán label cho node
kubectl label node node-1 disktype=ssd
kubectl label node node-2 disktype=ssd
```

### 2. nodeAffinity — Chọn Node Theo Quy Tắc Phức Tạp

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values: ["linux"]       # chỉ chạy trên Linux node
                  - key: node-role.kubernetes.io/worker
                    operator: Exists        # chỉ chạy trên worker node, không chạy trên control plane
```

### 3. tolerations — Chạy Trên Node Có Taint

Node control plane thường có taint để ngăn Pod thông thường chạy trên đó. DaemonSet agent như kube-proxy cần chạy trên mọi node kể cả control plane — dùng tolerations để "tha thứ" taint.

```yaml
spec:
  template:
    spec:
      tolerations:
        # Cho phép chạy trên control plane node
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule

        # Cho phép chạy trên node đang bị NotReady
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 300   # Pod có 300 giây trước khi bị evict

        # Cho phép chạy trên node đang bị unreachable
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 300
```

---

## Update Strategy

### RollingUpdate (Mặc Định)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # tối đa 1 node không có DaemonSet Pod trong khi update
                          # có thể dùng số tuyệt đối hoặc phần trăm: "10%"
```

Kubernetes cập nhật Pod trên từng node theo thứ tự, đảm bảo không quá `maxUnavailable` node thiếu Pod cùng lúc.

### OnDelete

Pod chỉ được cập nhật khi bạn **xoá thủ công** Pod đó. Dùng khi cần kiểm soát chặt chẽ thời điểm agent trên từng node được cập nhật.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

---

## Ví Dụ Thực Tế

### Fluentd — Thu Thập Log Node

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging

spec:
  selector:
    matchLabels:
      name: fluentd

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1

  template:
    metadata:
      labels:
        name: fluentd

    spec:
      # Tolerate control plane taint để chạy trên mọi node
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule

      # Chỉ chạy trên Linux node
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values: ["linux"]

      # ServiceAccount để Fluentd có thể đọc Pod metadata qua Kubernetes API
      serviceAccountName: fluentd

      # Ưu tiên cao để không bị evict khi node thiếu tài nguyên
      priorityClassName: system-node-critical

      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: elasticsearch.logging.svc.cluster.local
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"

          resources:
            limits:
              memory: 200Mi
              cpu: 100m
            requests:
              memory: 200Mi
              cpu: 100m

          # Mount thư mục log của node vào container
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: dockercontainerlogpath
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config-volume
              mountPath: /fluentd/etc

      # Khi Fluentd dừng, cần thời gian flush buffer
      terminationGracePeriodSeconds: 30

      volumes:
        # Log hệ thống từ node
        - name: varlog
          hostPath:
            path: /var/log

        # Log của container từ Docker/containerd
        - name: dockercontainerlogpath
          hostPath:
            path: /var/lib/docker/containers

        - name: config-volume
          configMap:
            name: fluentd-config
```

### Node Exporter — Metric CPU/Memory/Disk

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring

spec:
  selector:
    matchLabels:
      app: node-exporter

  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"

    spec:
      hostNetwork: true    # dùng network của host để node-exporter đọc network metric đúng
      hostPID: true        # dùng PID namespace của host để đọc process metric
      hostIPC: true

      tolerations:
        - operator: Exists   # tolerate mọi taint — chạy trên mọi node

      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.rootfs=/host
          ports:
            - name: metrics
              containerPort: 9100
              hostPort: 9100    # bind vào port của host node

          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 180Mi

          volumeMounts:
            - name: root
              mountPath: /host
              readOnly: true
              mountPropagation: HostToContainer

      volumes:
        - name: root
          hostPath:
            path: /    # mount toàn bộ filesystem của host
```

---

## Câu Hỏi Phỏng Vấn

### Q: Tại sao DaemonSet không dùng `replicas`?

DaemonSet không có trường `replicas` vì số lượng Pod được xác định bởi số node phù hợp — Kubernetes tự tính. Khi số node thay đổi, số Pod tự động thay đổi tương ứng.

### Q: Làm sao để chạy DaemonSet trên control plane node?

Control plane node thường có taint `node-role.kubernetes.io/control-plane:NoSchedule`. Thêm toleration phù hợp vào DaemonSet spec:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

### Q: DaemonSet Pod có bị evict khi node thiếu resource không?

Có thể, nếu không đặt `priorityClassName`. Nên đặt `priorityClassName: system-node-critical` cho các DaemonSet quan trọng (monitoring, logging, networking) để chúng được ưu tiên giữ lại khi node thiếu tài nguyên.

### Q: Sự khác biệt giữa `hostNetwork: true` và dùng Service?

`hostNetwork: true` khiến Pod dùng mạng của node thay vì Pod network. Pod có cùng IP với node và có thể bind vào port của node. Dùng cho node-exporter (cần đọc network interface của host), kube-proxy (cần thao tác iptables của host). Không nên dùng cho ứng dụng thông thường vì gây xung đột port và kém bảo mật.

---

**Xem Tiếp:** [job-cronjob.md](./job-cronjob.md) — Job và CronJob cho Tác Vụ Batch
