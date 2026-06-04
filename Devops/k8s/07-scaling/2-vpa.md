# VPA — Vertical Pod Autoscaler (Tự Động Điều Chỉnh Tài Nguyên Pod)

> Hướng dẫn chi tiết về VPA (Vertical Pod Autoscaler — Tự Động Điều Chỉnh Tài Nguyên Pod): kiến trúc 3 thành phần, 4 chế độ hoạt động, cấu hình YAML, kết hợp với HPA, và các pattern thực chiến để xác định resource request phù hợp.

## Mục Lục

1. [VPA Là Gì?](#vpa-là-gì)
2. [Kiến Trúc Ba Thành Phần](#kiến-trúc-ba-thành-phần)
3. [4 Chế Độ Hoạt Động](#4-chế-độ-hoạt-động)
4. [Cài Đặt VPA](#cài-đặt-vpa)
5. [Cấu Hình VPA](#cấu-hình-vpa)
6. [Đọc VPA Recommendation](#đọc-vpa-recommendation)
7. [VPA và HPA — Kết Hợp Như Thế Nào?](#vpa-và-hpa--kết-hợp-như-thế-nào)
8. [Debug và Giám Sát VPA](#debug-và-giám-sát-vpa)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## VPA Là Gì?

**VPA (Vertical Pod Autoscaler)** tự động điều chỉnh `resources.requests` và `resources.limits` của container dựa trên lịch sử tiêu thụ tài nguyên thực tế. Khác với HPA (thêm Pod), VPA làm cho **mỗi Pod khoẻ hơn** bằng cách cấp đúng lượng tài nguyên cần thiết.

**Hai vấn đề VPA giải quyết:**
1. **Over-provisioning (Cấp Quá Nhiều):** Khi request đặt quá cao → Cluster Autoscaler phải thêm node không cần thiết → lãng phí chi phí
2. **Under-provisioning (Cấp Quá Ít):** Khi request đặt quá thấp → CPU throttle liên tục hoặc OOMKilled → ảnh hưởng performance và stability

```
Không có VPA:                      Có VPA (mode Off — chỉ đề xuất):
───────────────                    ──────────────────────────────────
Dev đoán: cpu request 500m         VPA Recommender phân tích 14 ngày
Thực tế: ứng dụng dùng 200m  →    Khuyến nghị: cpu request 220m
         ← lãng phí 300m/Pod       limit: 440m
         ← 10 Pod lãng phí 3 core  → tiết kiệm ~60% tài nguyên CPU
```

---

## Kiến Trúc Ba Thành Phần

VPA gồm 3 thành phần cài đặt riêng (không tích hợp sẵn như HPA):

```
┌──────────────────────────────────────────────────────────┐
│                    VPA System                            │
│                                                          │
│  ┌──────────────────┐    ┌──────────────────────────┐   │
│  │ VPA Recommender  │    │      VPA Updater         │   │
│  │                  │    │                          │   │
│  │ - Theo dõi Pod   │    │ - Kiểm tra Pod có cần   │   │
│  │   usage liên tục │    │   cập nhật không        │   │
│  │ - Tính toán      │    │ - Evict Pod nếu request │   │
│  │   recommendation │    │   lệch nhiều so với     │   │
│  │ - Lưu vào VPA    │    │   recommendation        │   │
│  │   status         │    │                         │   │
│  └──────────────────┘    └──────────────────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │         VPA Admission Controller                 │   │
│  │                                                  │   │
│  │ - Webhook intercept khi Pod được tạo mới        │   │
│  │ - Thay thế resources theo VPA recommendation    │   │
│  │ - Áp dụng cho Pod mới (kể cả sau evict)         │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**Luồng cập nhật tài nguyên:**
```
Pod running với request cũ
     │  metrics được thu thập liên tục
     ▼
VPA Recommender tính recommendation
     │  recommendation lưu vào VPA object status
     ▼
VPA Updater phát hiện pod cần update (mode Auto/Recreate)
     │  evict Pod
     ▼
Pod bị evict → Deployment tạo Pod mới
     │  Pod mới đi qua Admission Controller
     ▼
Admission Controller inject resource mới theo recommendation
     ▼
Pod mới chạy với resource được tối ưu
```

---

## 4 Chế Độ Hoạt Động

```
updateMode: Off
│  Recommender chạy, Updater và Admission Controller không làm gì
│  Chỉ xem recommendation — không bao giờ tự thay đổi Pod
│  Dùng cho: học/khám phá, lấy số liệu để set request thủ công

updateMode: Initial
│  Admission Controller áp dụng recommendation khi Pod được tạo lần đầu
│  Không evict Pod đang chạy
│  Dùng cho: muốn Pod mới khởi động với request đúng, không muốn gián đoạn

updateMode: Recreate
│  Updater evict Pod khi recommendation thay đổi đáng kể
│  Admission Controller áp dụng recommendation cho Pod mới
│  Gây restart Pod — chỉ dùng khi ứng dụng chịu được restart

updateMode: Auto   (mặc định)
│  Hiện tại hoạt động như Recreate
│  Tương lai: K8s sẽ hỗ trợ in-place resize (không cần restart)
│  Dùng khi muốn VPA quản lý hoàn toàn tự động
```

> **Khuyến nghị thực tế:** Bắt đầu với `Off` để quan sát recommendation 1–2 tuần, sau đó điền request/limit vào manifest tĩnh dựa trên recommendation. Chỉ dùng `Auto/Recreate` khi workload thực sự biến động và bạn kiểm soát được restart.

---

## Cài Đặt VPA

VPA không cài sẵn — phải deploy thêm:

```bash
# Cài từ GitHub official repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Hoặc dùng Helm
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa \
  --namespace vpa-system \
  --create-namespace

# Kiểm tra VPA đang chạy
kubectl get pods -n vpa-system
# vpa-admission-controller-xxx   Running
# vpa-recommender-xxx            Running
# vpa-updater-xxx                Running
```

---

## Cấu Hình VPA

### VPA Mode Off — Chỉ Đề Xuất

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  updatePolicy:
    updateMode: "Off"         # chỉ đề xuất, không thay đổi Pod
  resourcePolicy:
    containerPolicies:
      - containerName: web-api
        minAllowed:
          cpu: "50m"          # recommendation không thấp hơn 50m CPU
          memory: "64Mi"
        maxAllowed:
          cpu: "2"            # recommendation không cao hơn 2 CPU core
          memory: "4Gi"
        controlledResources:
          - cpu
          - memory
```

### VPA Mode Auto — Tự Động Điều Chỉnh

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: batch-processor-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: batch-processor
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 2            # chỉ evict khi có ít nhất 2 replicas — tránh downtime
  resourcePolicy:
    containerPolicies:
      - containerName: batch-processor
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
```

### VPA Chỉ Điều Chỉnh Một Container

```yaml
resourcePolicy:
  containerPolicies:
    - containerName: main-app
      mode: Auto              # tự động điều chỉnh container này
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"

    - containerName: sidecar-proxy
      mode: Off               # không điều chỉnh sidecar — request đã được tối ưu
```

---

## Đọc VPA Recommendation

```bash
# Xem recommendation hiện tại
kubectl describe vpa web-api-vpa -n production

# Output (phần quan trọng):
# Status:
#   Conditions:
#     Type:         RecommendationProvided
#     Status:       True
#   Recommendation:
#     Container Recommendations:
#       Container Name:  web-api
#       Lower Bound:                  ← giới hạn dưới an toàn
#         Cpu:     50m
#         Memory:  262144k
#       Target:                       ← giá trị VPA khuyến nghị dùng
#         Cpu:     220m
#         Memory:  420Mi
#       Uncapped Target:              ← target trước khi áp dụng min/max
#         Cpu:     210m
#         Memory:  398Mi
#       Upper Bound:                  ← giới hạn trên (có thể vẫn ổn)
#         Cpu:     890m
#         Memory:  1Gi
```

**Giải thích recommendation:**
- `Target` — giá trị nên dùng cho `resources.requests`
- `Lower Bound` — giá trị thấp nhất vẫn ổn định
- `Upper Bound` — giá trị tối đa có thể cần trong peak load
- **Chiến lược thực tế:** Dùng `Target` cho `requests`, dùng `Upper Bound` cho `limits`

```yaml
# Sau khi xem VPA recommendation, cập nhật Deployment tĩnh:
resources:
  requests:
    cpu: "220m"          # = VPA Target
    memory: "420Mi"      # = VPA Target
  limits:
    cpu: "890m"          # = VPA Upper Bound
    memory: "1Gi"        # = VPA Upper Bound
```

---

## VPA và HPA — Kết Hợp Như Thế Nào?

### Xung Đột Mặc Định

Khi cả HPA và VPA cùng quản lý **cùng một metric (CPU hoặc memory)**:

```
VPA tăng CPU request của Pod lên 2x
   → HPA thấy CPU utilization giảm (vì denominator tăng)
   → HPA scale down Pod
   → CPU per Pod tăng lại
   → Vòng lặp vô hạn
```

### Cách Kết Hợp Đúng — 3 Pattern

**Pattern 1: HPA theo CPU, VPA mode Off (phổ biến nhất)**
```yaml
# VPA chỉ đề xuất, không tự áp dụng
# → Dev đọc recommendation và update manifest tĩnh
# → HPA scale theo CPU sẽ ổn định
updateMode: "Off"
```

**Pattern 2: HPA theo Custom Metric, VPA Auto**
```yaml
# HPA scale theo RPS hoặc queue depth (không phải CPU)
# → VPA điều chỉnh CPU/memory request mà không ảnh hưởng HPA
# Yêu cầu: Prometheus Adapter hoặc KEDA cho custom metric
```

**Pattern 3: VPA mode Initial — Set một lần rồi thôi**
```yaml
# VPA chỉ áp dụng khi Pod mới tạo, không evict Pod đang chạy
# HPA vẫn quản lý số lượng Pod bình thường
updateMode: "Initial"
```

---

## Debug và Giám Sát VPA

### Xem Trạng Thái VPA

```bash
# Danh sách VPA objects
kubectl get vpa -n production

# Output:
# NAME           MODE   CPU    MEM    PROVIDED   AGE
# web-api-vpa    Off    220m   420Mi  True       3d

# Chi tiết đầy đủ
kubectl describe vpa web-api-vpa -n production
```

### Kiểm Tra VPA Recommender Log

```bash
# Log của VPA Recommender — xem có lỗi thu thập metric không
kubectl logs -n vpa-system -l app=vpa-recommender --tail=100

# Log của VPA Updater — xem có evict Pod không
kubectl logs -n vpa-system -l app=vpa-updater --tail=100
```

### VPA Không Cung Cấp Recommendation

```bash
# Kiểm tra VPA status
kubectl describe vpa web-api-vpa -n production | grep -A 5 "Conditions:"

# Nguyên nhân phổ biến:
# 1. Pod chưa chạy đủ lâu (VPA cần ít nhất vài chục phút data)
# 2. metrics-server không hoạt động
# 3. targetRef sai (sai tên Deployment, sai namespace)
# 4. VPA Recommender gặp lỗi

# Kiểm tra events
kubectl events --for vpa/web-api-vpa -n production
```

---

## Câu Hỏi Phỏng Vấn

**VPA hoạt động như thế nào? Tại sao cần restart Pod để cập nhật resource?**

> VPA gồm 3 thành phần: **Recommender** (phân tích metric lịch sử, tính toán và lưu recommendation), **Updater** (evict Pod có resource lệch nhiều so với recommendation), **Admission Controller** (intercept việc tạo Pod mới, inject resource theo recommendation). Hiện tại K8s cần restart Pod vì `resources.requests` không thể thay đổi in-place trên container đang chạy — container runtime cần khởi động lại để áp dụng cgroup limit mới. Tính năng **in-place Pod resize** (KEP-1287) đang được phát triển để loại bỏ hạn chế này trong K8s tương lai.

**Tại sao không nên dùng VPA cùng HPA theo CPU/memory mặc định?**

> Xung đột xảy ra vì HPA tính CPU utilization = `actualUsage / requests`. Khi VPA tăng `requests` (ví dụ từ 200m lên 400m), tử số (actualUsage) giữ nguyên nhưng mẫu số tăng → utilization percentage giảm → HPA nghĩ CPU đang thấp → scale down → ít Pod hơn → actualUsage/Pod tăng → VPA giảm requests → HPA scale up... tạo ra vòng lặp không ổn định. Giải pháp: dùng VPA mode `Off` và điền giá trị thủ công, hoặc để HPA scale theo custom metric (RPS, queue depth) không liên quan đến CPU request.

**Dùng VPA mode nào trong production? Tại sao?**

> Thực tế production thường dùng **VPA mode Off** để quan sát recommendation, sau đó cập nhật manifest tĩnh trong code. Lý do: (1) Tránh Pod bị evict bất ngờ gây latency spike; (2) Thay đổi resource đi qua code review và CI/CD pipeline, không bị Kubernetes thay đổi ngoài tầm kiểm soát; (3) Dễ rollback khi có vấn đề; (4) Kết hợp an toàn với HPA. Mode `Auto/Recreate` phù hợp hơn cho môi trường staging/dev hoặc batch workload ít quan trọng về SLA.
