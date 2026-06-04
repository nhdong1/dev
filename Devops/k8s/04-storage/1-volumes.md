# Volumes — Các Loại Volume Trong Kubernetes

> Giải thích chi tiết các loại volume ephemeral (tạm thời) trong Kubernetes: emptyDir, hostPath, configMap volume, secret volume — cách khai báo, hành vi, và khi nào nên dùng loại nào.

## Mục Lục

1. [Tổng Quan Volume](#tổng-quan-volume)
2. [emptyDir — Thư Mục Chia Sẻ Tạm Thời](#emptydir--thư-mục-chia-sẻ-tạm-thời)
3. [hostPath — Mount Thư Mục Từ Node](#hostpath--mount-thư-mục-từ-node)
4. [configMap Volume — Mount Cấu Hình Vào Filesystem](#configmap-volume--mount-cấu-hình-vào-filesystem)
5. [secret Volume — Mount Secret Vào Filesystem](#secret-volume--mount-secret-vào-filesystem)
6. [projected Volume — Gộp Nhiều Nguồn](#projected-volume--gộp-nhiều-nguồn)
7. [So Sánh Các Loại Volume](#so-sánh-các-loại-volume)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Volume

Trong Kubernetes, **Volume** là một đơn vị lưu trữ gắn vào Pod, độc lập với vòng đời của từng container. Volume được khai báo ở cấp Pod (`spec.volumes`) và container trong Pod chọn để mount vào (`spec.containers[].volumeMounts`).

```
Pod
├── spec.volumes[]          ← khai báo volume (nguồn dữ liệu)
│   ├── name: my-vol
│   └── emptyDir: {}
│
└── spec.containers[]
    └── volumeMounts[]      ← container chọn volume để mount
        ├── name: my-vol
        └── mountPath: /data
```

**Điểm quan trọng:**
- Nhiều container trong cùng Pod có thể mount cùng một Volume → chia sẻ dữ liệu
- Volume được tạo trước container và tồn tại trong suốt vòng đời Pod
- Init Container cũng có thể mount Volume để chuẩn bị dữ liệu trước khi container chính chạy

---

## emptyDir — Thư Mục Chia Sẻ Tạm Thời

**emptyDir** tạo ra một thư mục rỗng khi Pod được khởi tạo trên node. Thư mục này tồn tại trong suốt vòng đời Pod và bị **xoá khi Pod bị removed** khỏi node.

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Tạo khi | Pod được schedule lên node |
| Xoá khi | Pod bị xoá (terminated, evicted) |
| Container restart | **Dữ liệu vẫn còn** |
| Phạm vi | Chỉ trong một Pod |
| Backend mặc định | Disk của node |
| Backend tuỳ chọn | RAM (`medium: Memory`) |

### Manifest Cơ Bản

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-data-demo
spec:
  volumes:
    - name: cache-vol
      emptyDir: {}

    - name: ram-vol
      emptyDir:
        medium: Memory        # lưu trong RAM thay vì disk
        sizeLimit: 256Mi      # giới hạn dung lượng RAM

  containers:
    - name: producer
      image: busybox
      command: ["/bin/sh", "-c", "echo 'hello' > /data/msg.txt && sleep 3600"]
      volumeMounts:
        - name: cache-vol
          mountPath: /data

    - name: consumer
      image: busybox
      command: ["/bin/sh", "-c", "while true; do cat /shared/msg.txt; sleep 5; done"]
      volumeMounts:
        - name: cache-vol
          mountPath: /shared   # cùng volume, khác mountPath
```

### Khi Nào Dùng emptyDir

```
✅ Nên dùng:
- Chia sẻ file tạm giữa các container trong cùng Pod (sidecar pattern)
- Cache tạm thời không cần persist (compiled assets, temp files)
- Buffer dữ liệu giữa producer và consumer container
- Scratch space (không gian tạm) cho quá trình xử lý

❌ Không nên dùng:
- Dữ liệu cần tồn tại sau khi Pod chết (dùng PVC)
- Dữ liệu cần chia sẻ giữa nhiều Pod (dùng PVC với RWX)
- Cache lớn cần tối ưu IOPS (dùng local PV hoặc tmpfs có kiểm soát)
```

### emptyDir RAM (tmpfs — Temporary File System)

```yaml
volumes:
  - name: fast-cache
    emptyDir:
      medium: Memory
      sizeLimit: 128Mi
```

> Dùng `medium: Memory` khi cần tốc độ đọc/ghi cực cao (ML model loading, in-memory processing). Lưu ý: dữ liệu trong RAM mất khi container restart; dung lượng tính vào memory limit của container.

---

## hostPath — Mount Thư Mục Từ Node

**hostPath** mount một thư mục hoặc file từ **filesystem của Node** vào container. Dữ liệu tồn tại trên node ngay cả khi Pod chết, nhưng **không di chuyển theo Pod khi Pod được reschedule sang node khác**.

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Tồn tại khi Pod chết | ✅ Có (trên node cũ) |
| Di chuyển cùng Pod | ❌ Không |
| Rủi ro bảo mật | ⚠️ Cao — container có thể đọc file nhạy cảm của node |
| Use case chính | DaemonSet, log agent, system-level tools |

### Manifest

```yaml
volumes:
  - name: host-log
    hostPath:
      path: /var/log          # đường dẫn trên node
      type: Directory         # loại: Directory, File, Socket, CharDevice, BlockDevice

  - name: docker-socket
    hostPath:
      path: /var/run/docker.sock
      type: Socket            # mount Unix socket
```

### Các Giá Trị hostPath.type

| Type | Ý Nghĩa |
| ---- | ------- |
| `""` (rỗng) | Không kiểm tra — path phải tồn tại trước |
| `DirectoryOrCreate` | Tạo thư mục nếu chưa có |
| `Directory` | Phải là thư mục đang tồn tại |
| `FileOrCreate` | Tạo file nếu chưa có |
| `File` | Phải là file đang tồn tại |
| `Socket` | Phải là Unix domain socket |

### Khi Nào Dùng hostPath

```
✅ Hợp lệ:
- DaemonSet log agent cần đọc /var/log/containers/ của node
- DaemonSet monitor cần đọc /proc, /sys từ node
- Node-local storage cho tool hệ thống (không phải ứng dụng)

❌ Tuyệt đối không dùng cho:
- Ứng dụng business thông thường (dùng PVC)
- Container cần quyền ghi vào /etc, /root, /var/run của node
- Môi trường multi-node (Pod restart trên node khác = mất data)
```

> **Cảnh báo bảo mật:** hostPath có thể bị khai thác để container-escape (thoát container). Pod Security Admission (PSA — Kiểm Soát Bảo Mật Pod) ở profile `restricted` cấm hoàn toàn hostPath.

---

## configMap Volume — Mount Cấu Hình Vào Filesystem

**configMap volume** cho phép mount nội dung của ConfigMap vào filesystem container dưới dạng **file**. Mỗi key trong ConfigMap trở thành một file; value là nội dung file.

### Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    db.host=postgres-service
    db.port=5432
  logging.yaml: |
    level: info
    format: json
---
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  volumes:
    - name: config-vol
      configMap:
        name: app-config             # tên ConfigMap
        defaultMode: 0644            # permission của file (mặc định 0644)
        items:                       # chỉ mount key cụ thể (tuỳ chọn)
          - key: app.properties
            path: config/app.properties  # đường dẫn tương đối trong volume

  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app        # container đọc /etc/app/config/app.properties
          readOnly: true             # nên đặt readOnly cho cấu hình
```

### Cập Nhật ConfigMap Tự Động

Khi ConfigMap được cập nhật, Kubernetes tự động cập nhật file trong volume sau một khoảng thời gian (thường **60–120 giây**, phụ thuộc kubelet sync period). Tuy nhiên:

```
ConfigMap volume:  tự động cập nhật sau ~60–120s (eventual consistency)
Biến môi trường:   KHÔNG tự động cập nhật — phải restart Pod
```

> Ứng dụng cần tự implement hot-reload (đọc lại file khi thay đổi) hoặc dùng `inotify` để theo dõi thay đổi. Nếu ứng dụng đọc config một lần lúc khởi động, cập nhật ConfigMap không có tác dụng cho đến khi Pod restart.

### subPath — Mount Một File Cụ Thể

```yaml
volumeMounts:
  - name: config-vol
    mountPath: /etc/nginx/nginx.conf  # mount đúng vào đường dẫn file
    subPath: nginx.conf               # chỉ lấy key "nginx.conf" từ ConfigMap
    readOnly: true
```

> Với `subPath`, file không được cập nhật tự động khi ConfigMap thay đổi. Đây là hạn chế quan trọng cần biết.

---

## secret Volume — Mount Secret Vào Filesystem

**secret volume** tương tự configMap volume nhưng dành cho dữ liệu nhạy cảm. Kubernetes lưu Secret trong `tmpfs` (RAM) trên node — dữ liệu không được ghi ra disk.

### Manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=      # "postgres" base64-encoded
  password: c2VjcmV0MTIz      # "secret123" base64-encoded
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  volumes:
    - name: secret-vol
      secret:
        secretName: db-credentials   # tên Secret
        defaultMode: 0400            # chỉ owner đọc được (bảo mật hơn 0644)

  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets
          readOnly: true
```

Container thấy các file:
```
/etc/secrets/username   → "postgres"
/etc/secrets/password   → "secret123"
```

> Kubernetes tự động decode base64 — container đọc giá trị thật, không phải chuỗi base64.

### So Sánh: Mount Qua Volume vs Biến Môi Trường

| Tiêu Chí | Volume Mount | Environment Variable |
| -------- | ------------ | -------------------- |
| Cập nhật tự động | ✅ (sau ~60s) | ❌ (cần restart) |
| Lộ ra trong `kubectl describe pod` | ❌ Không | ⚠️ Tên biến lộ, giá trị ẩn |
| Lộ ra trong process environment | ❌ Không | ⚠️ Có — `/proc/PID/environ` |
| Bảo mật | ✅ Tốt hơn | ⚠️ Kém hơn |
| Dễ dùng cho ứng dụng legacy | ❌ Cần đọc file | ✅ Không cần thay đổi app |

**Khuyến nghị:** Dùng volume mount cho secret quan trọng (TLS cert, API key). Dùng biến môi trường cho cấu hình không nhạy cảm hoặc ứng dụng legacy.

---

## projected Volume — Gộp Nhiều Nguồn

**projected volume** (volume chiếu) gộp nhiều nguồn (ConfigMap, Secret, ServiceAccount token, downwardAPI) vào **một mountPath duy nhất** thay vì mount riêng lẻ.

```yaml
volumes:
  - name: combined
    projected:
      sources:
        - configMap:
            name: app-config
        - secret:
            name: db-credentials
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600   # token tự động rotate
            audience: api-server
```

> Đặc biệt hữu ích cho service account token với thời gian sống ngắn (bound service account token — thay thế cho token tĩnh cũ).

---

## So Sánh Các Loại Volume

| Volume Type | Tồn Tại Sau Pod Chết | Chia Sẻ Giữa Pod | Bảo Mật | Use Case Chính |
| ----------- | -------------------- | ---------------- | ------- | -------------- |
| **emptyDir** | ❌ | ❌ | ✅ | Chia sẻ tạm giữa container |
| **emptyDir (Memory)** | ❌ | ❌ | ✅ | Cache tốc độ cao |
| **hostPath** | ✅ (trên node) | ❌ | ⚠️ Thấp | DaemonSet, log agent |
| **configMap** | N/A (từ ConfigMap) | ✅ | ✅ | Mount file cấu hình |
| **secret** | N/A (từ Secret) | ✅ | ✅✅ | Mount credentials, cert |
| **projected** | N/A | ✅ | ✅✅ | Gộp nhiều nguồn |
| **PVC** | ✅ | Phụ thuộc mode | ✅ | Database, file storage |

---

## Câu Hỏi Phỏng Vấn

**emptyDir khác persistent volume ở điểm nào quan trọng nhất?**

> emptyDir bị xoá khi **Pod bị removed khỏi node** — bao gồm khi Pod crash quá nhiều lần (bị evict), node drain, hoặc deployment update xoá Pod cũ. PV/PVC tồn tại độc lập với vòng đời Pod; Pod mới mount lại đúng volume cũ. emptyDir phù hợp cho dữ liệu tạm; PVC phù hợp cho dữ liệu cần tồn tại lâu dài.

**Tại sao secret mount qua volume được coi là an toàn hơn biến môi trường?**

> Khi Secret mount qua volume, giá trị được lưu trong `tmpfs` (RAM) của node — không bao giờ ghi ra disk. Ứng dụng đọc file trực tiếp; secret không xuất hiện trong process environment (`/proc/PID/environ`). Ngược lại, biến môi trường có thể bị lộ qua `kubectl describe pod`, crash dump, hoặc log framework vô tình in ra. Ngoài ra, volume mount hỗ trợ cập nhật tự động khi Secret thay đổi — biến môi trường không.

**Khi ConfigMap thay đổi, container có nhận ngay không?**

> Không — với **volume mount**, kubelet polling khoảng 60–120 giây mới cập nhật file trong volume (có thể cấu hình qua `--sync-frequency`). Nếu ứng dụng không tự reload config, cần restart Pod. Với **subPath mount**, file không bao giờ được cập nhật tự động. Với **biến môi trường**, không bao giờ cập nhật tự động — phải restart Pod. Đây là lý do quan trọng để thiết kế ứng dụng hỗ trợ hot-reload config qua file.

**hostPath có những rủi ro bảo mật nào?**

> Container có thể: (1) đọc file nhạy cảm của node như `/etc/passwd`, `/var/run/docker.sock`; (2) ghi vào hệ thống file của node; (3) leo thang đặc quyền (privilege escalation) bằng cách mount `/` của node và chạy lệnh với quyền root. Đây là vector tấn công phổ biến trong container-escape. Pod Security Admission ở profile `restricted` cấm hostPath. Chỉ dùng hostPath cho DaemonSet system-level với image đáng tin cậy từ team infra.
