# Pod Errors — Xử Lý Lỗi Pod Kubernetes

> Hướng dẫn chẩn đoán và khắc phục các lỗi Pod phổ biến nhất: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled, và Terminating.

## Mục Lục

1. [CrashLoopBackOff](#crashloopbackoff)
2. [ImagePullBackOff / ErrImagePull](#imagepullbackoff--errimagepull)
3. [Pod Pending](#pod-pending)
4. [OOMKilled (Out of Memory Killed — Bị Hủy Do Hết Bộ Nhớ)](#oomkilled)
5. [Pod Terminating Mãi](#pod-terminating-mãi)
6. [CreateContainerConfigError](#createcontainerconfigerror)
7. [RunContainerError](#runcontainererror)
8. [Exit Code Reference](#exit-code-reference)

---

## CrashLoopBackOff

### Định Nghĩa

CrashLoopBackOff là trạng thái K8s báo hiệu rằng container **liên tục crash và được restart**, khoảng thời gian chờ giữa các lần restart tăng dần theo cấp số nhân (10s → 20s → 40s → 80s → 160s → 300s tối đa).

### Triệu Chứng

```bash
NAME         READY   STATUS             RESTARTS   AGE
my-app-pod   0/1     CrashLoopBackOff   5          10m
```

### Nguyên Nhân và Cách Xử Lý

#### 1. Ứng Dụng Crash Ngay Sau Khi Khởi Động

```bash
# Xem log của lần crash gần nhất
kubectl logs <pod-name> -n <namespace> --previous

# Xem log realtime khi Pod vừa start
kubectl logs -f <pod-name> -n <namespace>
```

**Dấu hiệu:** Log có error message rõ ràng, exit code = 1.

**Cách sửa:** Sửa lỗi trong code hoặc cấu hình theo thông báo trong log.

#### 2. Liveness Probe (Kiểm Tra Sức Sống) Cấu Hình Sai

```bash
kubectl describe pod <pod-name> | grep -A10 "Liveness"
```

**Dấu hiệu:** Events có dòng `Liveness probe failed`. Container bị restart dù ứng dụng bình thường.

**Cách sửa:** Tăng `initialDelaySeconds` và `timeoutSeconds`, điều chỉnh endpoint probe cho đúng.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30    # Tăng nếu app khởi động chậm
  periodSeconds: 10
  timeoutSeconds: 5          # Tăng nếu health endpoint phản hồi chậm
  failureThreshold: 3
```

#### 3. Thiếu ConfigMap hoặc Secret

```bash
kubectl describe pod <pod-name> | grep -A5 "Error\|Warning"
# Thường thấy: "secret 'my-secret' not found" hoặc "configmap 'my-config' not found"
```

**Cách sửa:** Tạo ConfigMap/Secret còn thiếu.

```bash
kubectl get configmap <name> -n <namespace>
kubectl get secret <name> -n <namespace>
```

#### 4. Entrypoint / Command Sai

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].command}'
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].args}'
```

**Dấu hiệu:** Exit code = 126 (không có quyền thực thi) hoặc 127 (không tìm thấy file).

**Cách sửa:** Kiểm tra lại `command` và `args` trong manifest.

#### 5. OOMKilled Dẫn Đến CrashLoopBackOff

```bash
kubectl describe pod <pod-name> | grep -i "OOMKilled\|OOM\|memory"
```

**Dấu hiệu:** `Last State: Terminated, Reason: OOMKilled`.

**Cách sửa:** Tăng memory limit hoặc sửa memory leak — xem [phần OOMKilled](#oomkilled).

### Quy Trình Debug CrashLoopBackOff

```bash
# Bước 1: Kiểm tra trạng thái và restart count
kubectl get pod <name> -n <ns>

# Bước 2: Xem Events (thường có nguyên nhân rõ ràng)
kubectl describe pod <name> -n <ns>

# Bước 3: Xem log của lần crash trước
kubectl logs <name> -n <ns> --previous

# Bước 4: Xem exit code
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Bước 5: Debug bằng cách override command (nếu container crash quá nhanh)
kubectl run debug-copy --image=<your-image> --command -- sleep infinity
kubectl exec -it debug-copy -- sh
```

---

## ImagePullBackOff / ErrImagePull

### Định Nghĩa

Pod không thể tải image từ container registry — K8s sẽ thử lại nhiều lần, dẫn đến trạng thái `ImagePullBackOff`.

### Triệu Chứng

```bash
NAME         READY   STATUS             RESTARTS   AGE
my-app-pod   0/1     ImagePullBackOff   0          2m
```

### Nguyên Nhân và Cách Xử Lý

#### 1. Tên Image hoặc Tag Sai

```bash
kubectl describe pod <name> | grep "image\|Image"
# Thường thấy: "Failed to pull image 'myapp:lates'" (lỗi typo)
```

**Cách sửa:** Kiểm tra tên image và tag chính xác.

```yaml
# Sai
image: myapp:lates  # typo

# Đúng
image: myapp:latest
```

#### 2. Image Không Tồn Tại Trong Registry

```bash
# Kiểm tra image có tồn tại không
docker pull <image-name>:<tag>

# Liệt kê các tag có sẵn (với Docker Hub)
curl https://registry.hub.docker.com/v2/repositories/<user>/<repo>/tags/?page_size=10
```

#### 3. Private Registry Thiếu Credential (Thông Tin Xác Thực)

```bash
kubectl describe pod <name> | grep "Error\|Failed"
# Thấy: "Failed to pull image: 401 Unauthorized"
```

**Cách sửa:** Tạo `imagePullSecret` và gắn vào Pod.

```bash
# Tạo Secret chứa credential registry
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>
```

```yaml
# Gắn vào Pod
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: private-registry.example.com/myapp:1.0
```

#### 4. Rate Limit của Docker Hub

Docker Hub giới hạn số lần pull: 100 pulls/6h (anonymous), 200 pulls/6h (free account).

**Dấu hiệu:** `Too Many Requests. Please see https://docs.docker.com/docker-hub/download-rate-limit/`

**Cách sửa:**
- Dùng authenticated pull với Docker Hub account
- Dùng mirror registry nội bộ (Harbor, ECR, GCR)
- Cache image trên registry nội bộ

#### 5. Network Node Không Kết Nối Được Registry

```bash
# Kiểm tra trên node
curl -v https://registry-1.docker.io/v2/
# Hoặc kiểm tra từ Pod debug
kubectl run test --image=busybox --rm -it -- wget -O- https://registry-1.docker.io/v2/
```

### Quy Trình Debug ImagePullBackOff

```bash
# Bước 1: Xem thông báo lỗi chi tiết
kubectl describe pod <name> -n <ns> | grep -A5 "Failed\|Error"

# Bước 2: Xác nhận image tồn tại
docker manifest inspect <image>:<tag>

# Bước 3: Kiểm tra imagePullSecrets
kubectl get pod <name> -o jsonpath='{.spec.imagePullSecrets}'

# Bước 4: Test pull trực tiếp trên node (nếu có quyền)
ssh <node-ip>
docker pull <image>:<tag>
```

---

## Pod Pending

### Định Nghĩa

Pod ở trạng thái `Pending` nghĩa là đã được chấp nhận bởi K8s cluster nhưng **chưa được lên lịch chạy trên node nào** hoặc đang chờ tài nguyên.

### Triệu Chứng

```bash
NAME         READY   STATUS    RESTARTS   AGE
my-app-pod   0/1     Pending   0          5m
```

### Nguyên Nhân và Cách Xử Lý

#### 1. Không Đủ Tài Nguyên CPU/Memory Trên Cluster

```bash
# Kiểm tra tài nguyên còn lại trên từng node
kubectl describe nodes | grep -A5 "Allocated resources"

# Xem pod yêu cầu bao nhiêu resource
kubectl get pod <name> -o jsonpath='{.spec.containers[*].resources}'
```

**Dấu hiệu:** Events có `0/3 nodes are available: 3 Insufficient cpu.`

**Cách sửa:**
- Thêm node vào cluster (hoặc bật Cluster Autoscaler)
- Giảm resource request của Pod
- Xóa Pod không dùng để giải phóng tài nguyên

#### 2. nodeSelector hoặc Node Affinity Không Phù Hợp

```bash
kubectl describe pod <name> | grep -A10 "Node-Selectors\|Tolerations\|Affinity"
kubectl get nodes --show-labels | grep <required-label>
```

**Dấu hiệu:** `0/3 nodes are available: 3 node(s) didn't match node selector.`

**Cách sửa:** Kiểm tra lại `nodeSelector` hoặc `nodeAffinity` trong Pod spec, hoặc gắn label lên node phù hợp.

```bash
kubectl label node <node-name> disktype=ssd
```

#### 3. Taint và Toleration Không Khớp

```bash
# Xem taint trên các node
kubectl describe nodes | grep Taints

# Xem toleration của Pod
kubectl get pod <name> -o jsonpath='{.spec.tolerations}'
```

**Dấu hiệu:** `0/3 nodes are available: 3 node(s) had untolerated taint.`

**Cách sửa:** Thêm `tolerations` phù hợp vào Pod spec.

#### 4. PVC Chưa Được Bound (Gắn Kết)

```bash
kubectl get pvc -n <namespace>
# Nếu thấy STATUS = Pending → PVC chưa được bind với PV

kubectl describe pvc <pvc-name> -n <namespace>
```

**Cách sửa:** Xem thêm `storage-issues.md`.

#### 5. Thiếu Node (Cluster Rỗng hoặc Tất Cả Node NotReady)

```bash
kubectl get nodes
```

**Cách sửa:** Thêm node hoặc sửa node NotReady — xem `node-issues.md`.

### Quy Trình Debug Pod Pending

```bash
# Bước 1: Luôn bắt đầu từ Events
kubectl describe pod <name> -n <ns> | grep -A20 "Events:"

# Bước 2: Kiểm tra tài nguyên node
kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory'

# Bước 3: Kiểm tra pod resource request
kubectl get pod <name> -o yaml | grep -A5 resources

# Bước 4: Kiểm tra PVC
kubectl get pvc -n <ns>

# Bước 5: Chạy scheduler simulator (nếu cần)
kubectl describe pod <name> | grep "Insufficient\|didn't match\|Taints"
```

---

## OOMKilled

### Định Nghĩa

OOMKilled (Out Of Memory Killed) xảy ra khi container **vượt quá memory limit** — kernel Linux sẽ kill process đó. Container sẽ restart nếu có restart policy (mặc định là Always).

### Triệu Chứng

```bash
NAME         READY   STATUS      RESTARTS   AGE
my-app-pod   0/1     OOMKilled   3          15m

# Hoặc trong describe:
# Last State: Terminated   Reason: OOMKilled   Exit Code: 137
```

### Nguyên Nhân và Cách Xử Lý

#### 1. Memory Limit Quá Thấp So Với Nhu Cầu Thực

```bash
# Xem limit hiện tại
kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources.limits.memory}'

# Xem thực tế đang dùng bao nhiêu
kubectl top pod <name> -n <namespace>

# Xem lịch sử (nếu có Prometheus)
# metric: container_memory_working_set_bytes
```

**Cách sửa:** Tăng memory limit dựa trên usage thực tế.

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"   # Tăng lên dựa trên peak usage
```

**Lưu ý:** Tăng limit không phải là giải pháp lâu dài nếu ứng dụng bị memory leak.

#### 2. Memory Leak Trong Ứng Dụng

**Dấu hiệu:** Memory tăng dần theo thời gian, không giải phóng.

**Cách xử lý:**
- Profile ứng dụng để tìm nguồn leak (Go pprof, Java heap dump, Node.js heapdump)
- Dùng VPA (Vertical Pod Autoscaler) để tự động điều chỉnh trong lúc đang debug
- Thiết lập alert khi memory > 80% limit

#### 3. JVM Heap Size Không Được Cấu Hình

Java Virtual Machine (JVM — Máy Ảo Java) mặc định sử dụng 1/4 tổng RAM của hệ thống cho heap, có thể vượt quá container limit.

**Cách sửa:**

```yaml
env:
  - name: JAVA_OPTS
    value: "-Xms256m -Xmx400m"   # Giới hạn heap size rõ ràng
```

Hoặc dùng `-XX:MaxRAMPercentage=75.0` để JVM tự tính dựa trên container limit.

### Quy Trình Debug OOMKilled

```bash
# Bước 1: Xác nhận OOMKilled
kubectl describe pod <name> | grep -A5 "Last State"

# Bước 2: Xem exit code (137 = SIGKILL from OOM)
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Bước 3: Xem memory usage trước khi bị kill
kubectl top pod <name> -n <ns> --containers

# Bước 4: Xem log trước khi OOMKilled
kubectl logs <name> --previous | tail -50

# Bước 5: Check node-level OOM event
kubectl describe node <node-name> | grep -i "OOM\|memory"
```

---

## Pod Terminating Mãi

### Định Nghĩa

Pod ở trạng thái `Terminating` quá lâu (thường > 30 giây) do không hoàn thành graceful shutdown (tắt êm ái) trong `terminationGracePeriodSeconds`.

### Nguyên Nhân và Cách Xử Lý

#### 1. PreStop Hook hoặc Graceful Shutdown Quá Chậm

```bash
kubectl get pod <name> -o jsonpath='{.spec.terminationGracePeriodSeconds}'
kubectl describe pod <name> | grep -A5 "PreStop"
```

**Cách sửa:** Tăng `terminationGracePeriodSeconds` hoặc tối ưu logic shutdown.

#### 2. Finalizer Chưa Được Xóa

```bash
kubectl get pod <name> -o jsonpath='{.metadata.finalizers}'
```

**Cách sửa:** Xóa finalizer nếu resource stuck.

```bash
kubectl patch pod <name> -p '{"metadata":{"finalizers":null}}'
```

#### 3. Force Delete (Xóa Bắt Buộc) — Dùng Cẩn Thận

```bash
# Chỉ dùng khi các cách khác không hiệu quả
kubectl delete pod <name> --grace-period=0 --force
```

**Lưu ý:** Force delete có thể gây ra split-brain (hai node cùng nghĩ mình đang run Pod đó) với StatefulSet.

---

## CreateContainerConfigError

### Nguyên Nhân Phổ Biến

```bash
kubectl describe pod <name> | grep -A10 "Error"
```

- Secret hoặc ConfigMap được mount không tồn tại
- Key trong Secret/ConfigMap không tồn tại
- Biến môi trường tham chiếu đến resource không hợp lệ

```yaml
# Lỗi: secret key không tồn tại
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password   # key này phải tồn tại trong secret
        optional: false # false = bắt buộc phải có
```

---

## RunContainerError

### Nguyên Nhân Phổ Biến

- Entrypoint không tồn tại trong image
- Permission denied khi chạy executable
- Security context (ngữ cảnh bảo mật) quá hạn chế

```bash
kubectl describe pod <name> | grep -A5 "RunContainerError"
```

---

## Exit Code Reference

| Exit Code | Tên              | Ý Nghĩa                                                        |
| --------- | ---------------- | -------------------------------------------------------------- |
| 0         | Success          | Container thoát bình thường                                    |
| 1         | Error            | Lỗi chung — xem log để biết chi tiết                          |
| 2         | Misuse           | Dùng sai shell builtin hoặc script lỗi                        |
| 126       | Permission Denied | Không có quyền thực thi file                                 |
| 127       | Not Found        | Command/file không tồn tại trong container                    |
| 128+N     | Fatal Signal     | Process bị kill bởi signal N                                  |
| 137       | SIGKILL          | OOMKilled (128+9) — bị kernel kill do hết memory              |
| 143       | SIGTERM          | Graceful termination (128+15) — K8s yêu cầu dừng             |

### Cách Đọc Exit Code

```bash
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'
# hoặc lần trước
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

---

## Tóm Tắt Nhanh

| Lỗi                      | Lệnh Debug Đầu Tiên                                            |
| ------------------------- | --------------------------------------------------------------- |
| CrashLoopBackOff          | `kubectl logs <pod> --previous`                                 |
| ImagePullBackOff          | `kubectl describe pod <pod>` → xem Events                      |
| Pending                   | `kubectl describe pod <pod>` → xem Events (resource/node)      |
| OOMKilled                 | `kubectl top pod <pod>` và `kubectl describe pod` → Last State |
| Terminating               | `kubectl get pod -o yaml` → xem finalizers                     |
| CreateContainerConfigError | `kubectl describe pod` → kiểm tra Secret/ConfigMap            |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
