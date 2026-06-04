# Config & Secret — Quản Lý Cấu Hình và Bí Mật Kubernetes

> Tổng quan về cách Kubernetes quản lý cấu hình ứng dụng qua ConfigMap (Bản Đồ Cấu Hình) và dữ liệu nhạy cảm qua Secret (Bí Mật), cùng các giải pháp nâng cao như External Secrets Operator (Nhà Vận Hành Bí Mật Bên Ngoài) và Sealed Secrets (Bí Mật Được Niêm Phong).

## Mục Lục

1. [Vấn Đề Cần Giải Quyết](#vấn-đề-cần-giải-quyết)
2. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
3. [Các Thành Phần](#các-thành-phần)
4. [So Sánh ConfigMap vs Secret](#so-sánh-configmap-vs-secret)
5. [Luồng Tiêm Cấu Hình Vào Pod](#luồng-tiêm-cấu-hình-vào-pod)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
7. [Checklist Thực Chiến](#checklist-thực-chiến)

---

## Vấn Đề Cần Giải Quyết

Ứng dụng cần cấu hình và thông tin nhạy cảm để chạy — database URL, API key, certificate. Hardcode (viết cứng) những thứ này vào image Docker là anti-pattern vì:

- **Bảo mật:** Secret lộ trong image history và registry
- **Tính linh hoạt:** Không thể chạy cùng image ở dev/staging/production với cấu hình khác nhau
- **Rotation (xoay vòng):** Phải rebuild image mỗi khi đổi password

Kubernetes giải quyết điều này qua **ConfigMap** và **Secret** — tách cấu hình ra khỏi image, tiêm vào container lúc runtime (thời gian chạy).

```
Image Docker (bất biến)
  + ConfigMap (cấu hình phi nhạy cảm)    →  Pod đang chạy (có đầy đủ config)
  + Secret (dữ liệu nhạy cảm)
```

---

## Bản Đồ Quyết Định

```
Bạn cần lưu loại dữ liệu nào?
│
├── Dữ liệu phi nhạy cảm (không cần bảo mật đặc biệt)?
│   ├── Cấu hình ứng dụng: URL, port, feature flag, log level...
│   └── → ConfigMap
│       └── Xem: configmap.md
│
├── Dữ liệu nhạy cảm (cần bảo vệ, không được lộ)?
│   ├── Password, API key, token, certificate, SSH key...
│   ├── → Secret (lưu trong etcd, mã hoá base64)
│   │   └── Xem: secret.md
│   │
│   ├── Muốn lưu Secret an toàn trên Git (GitOps workflow)?
│   │   └── → Sealed Secrets (mã hoá bất đối xứng, chỉ cluster giải mã được)
│   │       └── Xem: sealed-secrets.md
│   │
│   └── Muốn đồng bộ Secret từ hệ thống bên ngoài?
│       ├── AWS Secrets Manager / Parameter Store
│       ├── HashiCorp Vault
│       ├── GCP Secret Manager / Azure Key Vault
│       └── → External Secrets Operator
│           └── Xem: external-secrets.md
│
└── Cần cả hai loại?
    └── Tạo cả ConfigMap và Secret, mount vào Pod theo nhu cầu
```

---

## Các Thành Phần

### ConfigMap — Bản Đồ Cấu Hình

**ConfigMap** lưu trữ dữ liệu cấu hình phi nhạy cảm dưới dạng key-value hoặc nội dung file. Phù hợp cho biến môi trường, file cấu hình (application.yml, nginx.conf), feature flag.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "postgres.internal"
  nginx.conf: |
    server {
      listen 80;
      location / { proxy_pass http://app:8080; }
    }
```

**Cách tiêm vào Pod:**
- Biến môi trường: `envFrom.configMapRef` hoặc `env[].valueFrom.configMapKeyRef`
- Volume mount: Mount từng key thành file trong container

Tham khảo chi tiết: [configmap.md](./1-configmap.md)

---

### Secret — Bí Mật

**Secret** giống ConfigMap nhưng dành cho dữ liệu nhạy cảm. Kubernetes lưu Secret trong etcd và mã hoá base64 (không phải encryption — chỉ encoding). Để mã hoá thật sự, cần bật **Encryption at Rest** (Mã Hoá Tại Nơi Lưu Trữ).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: cG9zdGdyZXM=     # base64("postgres")
  password: c3VwZXJzZWNyZXQ= # base64("supersecret")
```

**Các loại Secret phổ biến:**
- `Opaque` — dữ liệu tuỳ ý (mặc định)
- `kubernetes.io/dockerconfigjson` — thông tin xác thực image registry
- `kubernetes.io/tls` — certificate và private key cho TLS
- `kubernetes.io/service-account-token` — token ServiceAccount

Tham khảo chi tiết: [secret.md](./2-secret.md)

---

### External Secrets Operator — Nhà Vận Hành Bí Mật Bên Ngoài

**External Secrets Operator (ESO)** là Kubernetes operator tự động đồng bộ Secret từ hệ thống quản lý bí mật bên ngoài vào cluster. Thay vì lưu Secret trong cluster, bạn lưu ở nguồn tin cậy tập trung và ESO kéo về.

```
AWS Secrets Manager ──────────────────────────────────────────┐
HashiCorp Vault      →  External Secrets Operator  →  Secret  │
GCP Secret Manager  ──────────────────────────────────────────┘
```

**Hỗ trợ provider phổ biến:**
- AWS Secrets Manager và Parameter Store (Kho Tham Số)
- HashiCorp Vault
- GCP Secret Manager
- Azure Key Vault (Kho Khoá Azure)
- 1Password, Doppler, Infisical

Tham khảo chi tiết: [external-secrets.md](./3-external-secrets.md)

---

### Sealed Secrets — Bí Mật Được Niêm Phong

**Sealed Secrets** giải quyết bài toán GitOps: làm thế nào để commit Secret vào Git mà không lộ nội dung?

Kubeseal (công cụ CLI) mã hoá Secret bằng public key của cluster — chỉ cluster đó mới giải mã được bằng private key. File `SealedSecret` an toàn để lưu trên Git.

```
Secret (nhạy cảm)  →  kubeseal mã hoá  →  SealedSecret (an toàn trên Git)
                                               ↓
                                    Sealed Secrets Controller giải mã
                                               ↓
                                    Secret (trong cluster)
```

Tham khảo chi tiết: [sealed-secrets.md](./4-sealed-secrets.md)

---

## So Sánh ConfigMap vs Secret

| Tiêu Chí | ConfigMap | Secret |
| -------- | --------- | ------ |
| **Mục đích** | Cấu hình phi nhạy cảm | Dữ liệu nhạy cảm |
| **Lưu trữ trong etcd** | Plaintext (văn bản thường) | base64 encoded (không encrypt mặc định) |
| **Encryption at Rest** | Không áp dụng | Có thể bật (khuyến nghị) |
| **Hiển thị trong `kubectl get`** | Hiện giá trị | Ẩn giá trị (base64) |
| **Mount vào Pod** | Env var hoặc volume | Env var hoặc volume |
| **Dung lượng tối đa** | 1 MiB | 1 MiB |
| **Immutable (bất biến)** | Hỗ trợ (`immutable: true`) | Hỗ trợ (`immutable: true`) |
| **RBAC control** | Phân quyền theo namespace | Phân quyền chặt hơn (nên restrict) |

> **Quan trọng:** Secret mặc định chỉ base64 — bất kỳ ai có quyền `get secret` đều đọc được giá trị. Cần cấu hình RBAC chặt và bật Encryption at Rest để bảo vệ thật sự.

---

## So Sánh Giải Pháp Secret Nâng Cao

| Tiêu Chí | Secret K8s Thuần | External Secrets | Sealed Secrets |
| -------- | ---------------- | ---------------- | -------------- |
| **Lưu trên Git** | ❌ Không an toàn | ✅ Không cần lưu Secret | ✅ SealedSecret an toàn |
| **Nguồn gốc secret** | Tạo thủ công | Hệ thống bên ngoài (Vault, AWS) | Tạo local rồi seal |
| **Auto rotation** | ❌ Thủ công | ✅ ESO tự đồng bộ | ❌ Cần re-seal thủ công |
| **Phụ thuộc bên ngoài** | Không | Cần provider bên ngoài | Không (chỉ cần controller) |
| **Độ phức tạp** | Thấp | Trung bình | Thấp |
| **Phù hợp với GitOps** | Kém | Tốt | Rất tốt |
| **Audit log** | Qua K8s audit | Qua provider (Vault, AWS) | Qua K8s audit |

---

## Luồng Tiêm Cấu Hình Vào Pod

### Qua Biến Môi Trường (Environment Variable)

```
ConfigMap / Secret
       │
       │  envFrom / env[].valueFrom
       ▼
Pod spec được tạo với biến môi trường đã khai báo
       │
       ▼
Container khởi động — thấy biến qua process.env / os.environ / System.getenv
```

**Ưu điểm:** Đơn giản, ngôn ngữ nào cũng đọc được biến môi trường.  
**Nhược điểm:** Biến môi trường **không tự cập nhật** khi ConfigMap/Secret thay đổi — Pod phải restart.

### Qua Volume Mount

```
ConfigMap / Secret
       │
       │  volumes[].configMap / volumes[].secret
       ▼
Kubernetes mount dữ liệu thành file trong container
       │
       ▼
Container đọc file tại mountPath (ví dụ /etc/config/app.yml)
```

**Ưu điểm:** File **tự động cập nhật** (sau 1–2 phút) khi ConfigMap/Secret thay đổi — không cần restart Pod (với điều kiện ứng dụng hot-reload được).  
**Nhược điểm:** Ứng dụng phải tự watch và reload file khi thay đổi.

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**ConfigMap và Secret khác nhau thế nào?**

> **ConfigMap** lưu cấu hình phi nhạy cảm — không được mã hoá đặc biệt, mọi người có quyền `get configmap` đều đọc được. **Secret** lưu dữ liệu nhạy cảm — được base64 encode (không phải encrypt), nhưng Kubernetes xử lý Secret khác: không in ra khi `kubectl describe`, có thể bật Encryption at Rest trong etcd, RBAC nên restrict chặt hơn. Về cấu trúc và cách mount vào Pod, hai loại giống nhau.

**Secret có thực sự an toàn không?**

> Secret mặc định **không an toàn tuyệt đối** — base64 chỉ là encoding, không phải encryption. Bất kỳ ai có quyền `kubectl get secret -o yaml` đều đọc được giá trị gốc. Để bảo mật thật sự cần: (1) Bật **Encryption at Rest** trong etcd bằng EncryptionConfiguration; (2) Cấu hình RBAC chặt chẽ — chỉ ServiceAccount cần thiết mới được `get/list` secret; (3) Dùng External Secrets với Vault hoặc AWS Secrets Manager để Secret không bao giờ lưu plaintext trong etcd.

**Khi nào nên dùng External Secrets thay vì Secret thông thường?**

> Dùng External Secrets khi: (1) Tổ chức đã có hệ thống quản lý secret tập trung (Vault, AWS Secrets Manager); (2) Cần **auto rotation** (xoay vòng tự động) — ESO đồng bộ liên tục, khi Vault đổi secret, Pod tự nhận giá trị mới; (3) Cần **audit trail** (nhật ký kiểm tra) chi tiết cho compliance; (4) Nhiều cluster cùng dùng chung bộ secret. Với project nhỏ, Secret K8s thuần kết hợp Sealed Secrets thường đủ dùng.

### Câu Hỏi Nâng Cao

**Khi ConfigMap thay đổi, Pod có nhận được giá trị mới không?**

> Phụ thuộc cách mount: nếu mount qua **volume**, kubelet sẽ cập nhật file trong container sau khoảng 1–2 phút (dựa vào sync period của kubelet). Tuy nhiên ứng dụng cần tự reload — nhiều app dùng inotify hoặc vòng lặp kiểm tra để hot-reload. Nếu mount qua **biến môi trường (env var)**, giá trị **không bao giờ thay đổi** trong container đang chạy — phải restart Pod. Đây là lý do quan trọng khi chọn cách mount.

**immutable ConfigMap / Secret là gì và khi nào nên dùng?**

> Khi đặt `immutable: true`, Kubernetes từ chối mọi thay đổi nội dung sau khi tạo. Lợi ích: (1) Bảo vệ khỏi thay đổi vô tình gây ảnh hưởng tới Pod đang chạy; (2) **Cải thiện hiệu năng cluster đáng kể** — kubelet không cần watch ConfigMap immutable, giảm tải cho API Server; hữu ích khi cluster có hàng nghìn Pod. Nếu cần thay đổi, phải xoá và tạo lại với tên mới, sau đó cập nhật Pod reference. Pattern phổ biến: đặt tên kèm version — `app-config-v2`.

---

## Checklist Thực Chiến

### Thiết Lập ConfigMap

- [ ] Tách cấu hình theo môi trường (dev/staging/prod) bằng namespace hoặc tên khác nhau
- [ ] Không lưu dữ liệu nhạy cảm (password, key) trong ConfigMap
- [ ] Đặt `immutable: true` cho ConfigMap production ít thay đổi để giảm tải API Server
- [ ] Dùng `--from-file` để tạo ConfigMap từ file cấu hình thực tế (nginx.conf, application.yml)
- [ ] Test hot-reload: cập nhật ConfigMap và kiểm tra ứng dụng nhận giá trị mới (nếu dùng volume)

### Thiết Lập Secret

- [ ] Bật **Encryption at Rest** (Mã Hoá Tại Nơi Lưu Trữ) cho etcd — đặc biệt trong môi trường production
- [ ] Cấu hình RBAC: chỉ cho ServiceAccount và user cần thiết quyền `get/list` Secret
- [ ] Không commit Secret plaintext lên Git — dùng Sealed Secrets hoặc External Secrets
- [ ] Rotate (xoay vòng) Secret định kỳ — ít nhất 90 ngày với credential quan trọng
- [ ] Dùng `type: kubernetes.io/tls` cho TLS certificate thay vì `Opaque` — để tooling nhận dạng đúng

### Thiết Lập External Secrets (Cho Production)

- [ ] Chọn provider phù hợp: AWS Secrets Manager nếu dùng EKS, Vault nếu multi-cloud
- [ ] Cấu hình IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho ServiceAccount) để ESO truy cập AWS mà không cần access key
- [ ] Đặt `refreshInterval` hợp lý (15m–1h) — quá thấp tăng tải provider, quá cao chậm rotation
- [ ] Monitor ExternalSecret status: `kubectl get externalsecret` và cấu hình alert khi sync fail
- [ ] Test rotation: cập nhật secret trên provider, xác nhận ESO đồng bộ vào cluster

### Thiết Lập Sealed Secrets (Cho GitOps)

- [ ] Backup private key của Sealed Secrets Controller — mất key là mất khả năng giải mã toàn bộ
- [ ] Khai báo `--scope cluster-wide` hay `namespace` tường minh khi seal để tránh nhầm
- [ ] Thêm Sealed Secrets Controller vào cluster bootstrap (trước khi apply bất kỳ SealedSecret nào)
- [ ] Không seal Secret với public key của cluster khác — mỗi cluster có key riêng
- [ ] Kiểm tra `SealedSecret` status sau khi apply: `kubectl get sealedsecret` và xem condition

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
| ---- | -------- |
| [configmap.md](./1-configmap.md) | Tạo, mount, cập nhật ConfigMap chi tiết |
| [secret.md](./2-secret.md) | Tạo, mount Secret, các loại Secret, Encryption at Rest |
| [external-secrets.md](./3-external-secrets.md) | External Secrets Operator, AWS, Vault |
| [sealed-secrets.md](./4-sealed-secrets.md) | Sealed Secrets cho GitOps workflow |
