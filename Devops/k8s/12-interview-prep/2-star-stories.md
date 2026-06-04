# Câu Chuyện Sự Cố K8s theo Phương Pháp STAR

> STAR = Situation (Tình Huống) — Task (Nhiệm Vụ) — Action (Hành Động) — Result (Kết Quả). Đây là cấu trúc chuẩn để kể câu chuyện kỹ thuật trong phỏng vấn senior/mid-level.

## Mục Lục

1. [Cách Sử Dụng Tài Liệu Này](#cách-sử-dụng-tài-liệu-này)
2. [Sự Cố 1: CrashLoopBackOff Hàng Loạt Sau Deployment](#sự-cố-1-crashloopbackoff-hàng-loạt-sau-deployment)
3. [Sự Cố 2: OOMKilled Gây Gián Đoạn Payment Service](#sự-cố-2-oomkilled-gây-gián-đoạn-payment-service)
4. [Sự Cố 3: Network Partition Giữa Microservices](#sự-cố-3-network-partition-giữa-microservices)
5. [Sự Cố 4: etcd Full Gây Cluster Read-only](#sự-cố-4-etcd-full-gây-cluster-read-only)
6. [Sự Cố 5: HPA Không Scale Gây Quá Tải](#sự-cố-5-hpa-không-scale-gây-quá-tải)
7. [Sự Cố 6: Node NotReady Gây Mất 30% Capacity](#sự-cố-6-node-notready-gây-mất-30-capacity)
8. [Template Tự Viết Câu Chuyện](#template-tự-viết-câu-chuyện)

---

## Cách Sử Dụng Tài Liệu Này

### Nguyên Tắc Kể Câu Chuyện Hiệu Quả

```
1. Mở đầu bằng bối cảnh (2–3 câu): hệ thống là gì, scale thế nào, team ra sao
2. Nêu vấn đề cụ thể (không nói chung chung): "latency tăng lên 800ms" thay vì "hệ thống chậm"
3. Mô tả hành động tuần tự: bạn làm gì trước, làm gì sau, tại sao chọn cách đó
4. Kết quả phải có con số đo lường: MTTR (Mean Time To Recovery), % cải thiện, số user bị ảnh hưởng
5. Kết thúc bằng bài học: bạn thay đổi quy trình/kiến trúc gì để không tái diễn?
```

### Câu Hỏi Follow-up Thường Gặp

```
"Bạn phát hiện sự cố đó như thế nào?" → Monitoring alert hay user báo?
"Tại sao không làm X thay vì Y?"      → Trade-off bạn đã cân nhắc?
"Bạn đã ngăn ngừa tái diễn thế nào?" → Post-mortem action items?
"Team phản ứng ra sao?"               → Communication và leadership?
```

---

## Sự Cố 1: CrashLoopBackOff Hàng Loạt Sau Deployment

### Bối Cảnh

Hệ thống API Gateway cho platform thương mại điện tử, xử lý ~2.000 request/giây. Cluster EKS với 8 node t3.xlarge, team DevOps 3 người.

### Situation (Tình Huống)

```
Thứ Sáu 22:30. Team vừa deploy phiên bản v3.2.0 của API Gateway lên production
bằng ArgoCD. Ngay sau khi sync hoàn tất, Grafana bắt đầu fire alert:
Pod restart rate tăng vọt. Trong 10 phút, tất cả 8 Pod API Gateway đều ở
trạng thái CrashLoopBackOff.

Tác động:
- 100% request đến API Gateway thất bại
- Khoảng 15.000 user không thể truy cập platform
- Đây là giờ cao điểm của Đông Nam Á (10:30 PM Singapore time)
```

### Task (Nhiệm Vụ)

```
- Khôi phục dịch vụ trong thời gian ngắn nhất có thể
- Xác định nguyên nhân gốc rễ (Root Cause Analysis — RCA)
- Ngăn không để điều này xảy ra lại
```

### Action (Hành Động)

```
Bước 1 — Rollback ngay lập tức (T+5 phút):
kubectl rollout undo deployment/api-gateway
→ Sau 3 phút, Pod trở lại Running với v3.1.9

Bước 2 — Thu thập evidence:
kubectl logs api-gateway-xyz --previous
→ Tìm thấy: "panic: runtime error: invalid memory address"
→ Exit code: 139 (Segmentation fault)

Bước 3 — Phân tích diff code v3.1.9 → v3.2.0:
git diff v3.1.9 v3.2.0 -- config/
→ Phát hiện: biến môi trường DATABASE_POOL_SIZE bị đổi tên thành
  DB_MAX_CONNECTIONS nhưng ConfigMap trong K8s chưa được cập nhật

Bước 4 — Reproduce trên staging:
kubectl apply -f v3.2.0-manifest.yaml --namespace staging
→ Confirm lỗi: Pod crash ngay khi start vì không tìm thấy env var

Bước 5 — Fix và deploy lại:
kubectl patch configmap api-gateway-config \
  --patch '{"data":{"DB_MAX_CONNECTIONS":"20"}}'
kubectl rollout restart deployment/api-gateway
→ v3.2.0 deploy thành công sau 7 phút

Bước 6 — Post-mortem và prevention:
- Thêm smoke test trong CI pipeline kiểm tra required env vars
- Thêm initContainer kiểm tra env var trước khi container chính chạy
- Yêu cầu checklist ConfigMap/Secret khi đổi tên env var trong code review
```

### Result (Kết Quả)

```
- MTTR (Mean Time To Recovery): 23 phút (từ alert đến 100% Pod healthy)
- Rollback hoàn tất trong 5 phút — PodDisruptionBudget giúp duy trì 2 Pod
  v3.1.9 trong khi rollout diễn ra
- Số user bị ảnh hưởng: ~15.000, thời gian impact: ~8 phút (trước khi
  rollback hoàn tất)
- Không có data loss vì API Gateway là stateless
- Action items từ post-mortem: 4 items, hoàn thành trong sprint tiếp theo

Bài học:
- Không bao giờ đổi tên env var mà không deploy ConfigMap trước
- Staged rollout (deploy ConfigMap → verify → deploy app) thay vì một lần
- Smoke test sau mỗi deployment là bắt buộc, không phải optional
```

---

## Sự Cố 2: OOMKilled Gây Gián Đoạn Payment Service

### Bối Cảnh

Fintech startup, payment service xử lý giao dịch thẻ tín dụng. SLA 99.95%, đang chạy trên GKE với 3 Pod payment-service, memory limit 512Mi.

### Situation (Tình Huống)

```
Ngày cuối tháng — thời điểm lương về và thanh toán hóa đơn tăng vọt.
11:45 AM: Alert cháy — payment-service có 2/3 Pod restart.
11:50 AM: Pod thứ 3 bị OOMKilled.
Trong 3 phút, 0/3 Pod available — 100% payment transaction fail.
Ảnh hưởng: ~500 giao dịch thất bại, doanh thu mất ~150 triệu VNĐ.
```

### Task (Nhiệm Vụ)

```
- Restore payment service ngay lập tức
- Tìm nguyên nhân memory leak hoặc memory spike
- Đảm bảo không xảy ra lại vào cuối tháng tiếp theo
```

### Action (Hành Động)

```
Bước 1 — Emergency scale + tăng limit tạm thời:
kubectl set resources deployment/payment-service \
  --limits=memory=1Gi
→ Pod khởi động lại với limit mới, service restore trong 4 phút

Bước 2 — Phân tích Grafana:
- Memory usage tăng từ 300Mi lên 512Mi trong vòng 15 phút
- Pattern: spike đúng lúc concurrent user tăng 3x (từ 100 lên 300)

Bước 3 — Profiling Java service:
kubectl exec -it payment-service-xyz -- jcmd 1 VM.native_memory
→ Phát hiện: mỗi request tạo một new connection pool thay vì dùng connection pool chung
→ 300 concurrent user × 5 connections = 1.500 DB connections, mỗi connection ~200KB

Bước 4 — Fix root cause:
- Sửa code: khởi tạo connection pool một lần khi startup (Singleton pattern)
- Giảm max pool size từ 20 xuống 5 (đủ cho throughput hiện tại)
- Thêm JVM heap tuning: -Xmx400m -Xms200m

Bước 5 — Thiết lập đúng resource:
kubectl set resources deployment/payment-service \
  --requests=memory=300Mi \
  --limits=memory=600Mi
(Không để limit = request để tránh mọi burst nhỏ đều OOMKill)

Bước 6 — Cấu hình VPA để tự động tuning:
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Off"  # Chỉ recommend, không tự apply (an toàn hơn)
```

### Result (Kết Quả)

```
- MTTR: 7 phút
- Root cause: connection pool bug gây memory grow với O(n) concurrent users
- Sau fix: memory ổn định ở 280–320Mi dù concurrent user tăng 5x
- Cuối tháng tiếp theo: không có incident
- Cải tiến quy trình: thêm load test với traffic pattern cuối tháng vào CI pipeline
- Monitoring: thêm alert trên JVM heap usage và DB connection count

Bài học:
- Memory limit không nên bằng request (buffer 50–100% cho burst)
- Load test phải reflect traffic pattern thực tế, bao gồm peak patterns
- VPA recommendation giúp calibrate resource settings theo data thực
```

---

## Sự Cố 3: Network Partition Giữa Microservices

### Bối Cảnh

Platform logistics, 15 microservices trên EKS. Team vừa migrate từ Flannel sang Calico để có NetworkPolicy support. Môi trường production.

### Situation (Tình Huống)

```
Sau khi migration CNI plugin từ Flannel sang Calico hoàn tất,
order-service không thể kết nối database-service.
Tất cả API call liên quan đến order đều trả về 500 error.
User không thể tạo đơn hàng mới.
Các service khác không bị ảnh hưởng — chỉ order-service bị cô lập.
```

### Task (Nhiệm Vụ)

```
- Khôi phục kết nối giữa order-service và database-service
- Hiểu tại sao Calico block traffic này
- Review toàn bộ NetworkPolicy trước khi kết thúc ngày
```

### Action (Hành Động)

```
Bước 1 — Verify connectivity:
kubectl exec -it order-service-pod -- nc -zv db-service 5432
→ Connection timed out

kubectl exec -it order-service-pod -- nslookup db-service
→ DNS resolve OK → vấn đề là network, không phải DNS

Bước 2 — Kiểm tra NetworkPolicy:
kubectl get networkpolicy -n production
→ Tìm thấy NetworkPolicy "default-deny-all" được áp dụng khi Calico install

kubectl describe networkpolicy default-deny-all -n production
→ Deny tất cả ingress và egress trong namespace

Bước 3 — Kiểm tra NetworkPolicy cho database-service:
kubectl get networkpolicy -n production -l component=database
→ Không có NetworkPolicy allow cho order-service!

Nguyên nhân:
- Với Flannel: không có NetworkPolicy enforcement → tất cả traffic thông
- Sau khi switch sang Calico: NetworkPolicy được enforce
- Team đã setup "default-deny" nhưng quên allow order-service → database

Bước 4 — Fix ngay lập tức:
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: order-service
    ports:
    - protocol: TCP
      port: 5432
EOF
→ Connectivity restore trong 30 giây

Bước 5 — Audit toàn bộ NetworkPolicy:
# Kiểm tra mọi service có policy allow cần thiết không
for svc in $(kubectl get svc -n production -o name); do
  echo "=== $svc ==="
  kubectl get networkpolicy -n production -l "app=$(basename $svc)"
done

→ Tìm thêm 3 service bị thiếu NetworkPolicy → fix hết trước khi EOD

Bước 6 — Cải thiện quy trình migration:
- Tạo checklist CNI migration: kiểm tra connectivity test cho mọi service pair
- Thêm bước "dry-run NetworkPolicy" trong staging trước khi production
- Dùng Cilium network policy editor để visualize traffic flow
```

### Result (Kết Quả)

```
- Downtime: 28 phút (từ khi phát hiện đến restore)
- Không có data loss vì database không bị affected
- Phát hiện thêm 3 service có NetworkPolicy misconfiguration → fix trước khi incident
- Tạo document "CNI Migration Checklist" — 12 bước kiểm tra

Bài học:
- Khi thay đổi networking layer, LUÔN test connectivity matrix giữa mọi service pair
- Default-deny policy là đúng về bảo mật, nhưng cần verify allow rules đầy đủ trước khi apply
- Staging environment phải mirror production network policy 100%
```

---

## Sự Cố 4: etcd Full Gây Cluster Read-only

### Bối Cảnh

Self-managed K8s cluster với kubeadm, 3 control plane node. Platform cho SaaS application phục vụ 200 tenant. Môi trường on-premise.

### Situation (Tình Huống)

```
Thứ Hai 9:00 AM. Tất cả kubectl command trả về lỗi:
"etcdserver: mvcc: database space exceeded"
Không thể tạo Pod mới, không thể scale Deployment.
Các Pod đang chạy vẫn hoạt động bình thường.
Nhưng không thể deploy hotfix, không thể restart Pod bị lỗi.
```

### Task (Nhiệm Vụ)

```
- Restore etcd về trạng thái writable ngay lập tức
- Tìm nguyên nhân etcd full (quota mặc định là 2GB)
- Thiết lập monitoring để cảnh báo trước khi xảy ra lần sau
```

### Action (Hành Động)

```
Bước 1 — Kiểm tra etcd status:
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  --write-out=table
→ DB SIZE: 2.0 GB / 2.0 GB (100% đầy)

Bước 2 — Tăng quota tạm thời để restore operability:
# Edit etcd static pod manifest
vi /etc/kubernetes/manifests/etcd.yaml
# Thêm flag: --quota-backend-bytes=4294967296  (4GB)
→ Sau 30 giây etcd restart, cluster writable trở lại

Bước 3 — Defragment etcd để giải phóng space:
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=... --cert=... --key=...
→ DB SIZE giảm từ 2.0 GB xuống 380 MB

Bước 4 — Tìm nguyên nhân gốc rễ:
# Đếm số object theo loại
kubectl get all --all-namespaces | wc -l         → 45.000 objects (bất thường)
kubectl get pods --all-namespaces | wc -l         → 38.000 pods!

kubectl get jobs --all-namespaces | wc -l         → 35.000 completed jobs!
→ CronJob mỗi 5 phút, chạy 6 tháng, successfulJobsHistoryLimit = 100
→ Nhưng bug: successfulJobsHistoryLimit không hoạt động đúng với K8s 1.22 cụ thể

Bước 5 — Cleanup completed jobs:
kubectl delete jobs --all-namespaces \
  --field-selector=status.completionTime\!= \
  --dry-run=client | wc -l
→ 34.500 jobs sẽ bị xoá

kubectl delete jobs --all-namespaces \
  --field-selector=status.completionTime\!=
→ Xoá 34.500 completed jobs, etcd giảm xuống 150 MB

Bước 6 — Patch CronJob:
kubectl patch cronjob my-cronjob \
  -p '{"spec":{"successfulJobsHistoryLimit":3,"failedJobsHistoryLimit":3}}'

Bước 7 — Thiết lập alert:
# Prometheus alert rule
- alert: EtcdDatabaseSpaceExceeded
  expr: etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes > 0.75
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "etcd database space > 75%"
```

### Result (Kết Quả)

```
- Cluster restore trong 8 phút sau khi phát hiện
- Nguyên nhân: 34.500 completed Job object tích lũy 6 tháng
- etcd giảm từ 2GB xuống 150MB sau cleanup
- Thiết lập Prometheus alert khi etcd > 75% — cảnh báo 3 tuần trước khi full
- Thêm bước kiểm tra etcd size vào runbook vận hành hàng tuần
- Upgrade K8s 1.22 lên 1.24 để fix bug successfulJobsHistoryLimit

Bài học:
- Backup etcd quan trọng nhưng monitoring etcd size quan trọng không kém
- CronJob với successfulJobsHistoryLimit = 0 hoặc nhỏ là best practice
- Defragment etcd định kỳ (tháng) là vận hành cần thiết
```

---

## Sự Cố 5: HPA Không Scale Gây Quá Tải

### Bối Cảnh

Ride-hailing app, backend service xử lý booking request. HPA đã cấu hình scale từ 3 đến 30 Pod. Black Friday campaign với traffic tăng đột ngột.

### Situation (Tình Huống)

```
Chiến dịch khuyến mãi bắt đầu 8:00 AM. Traffic tăng 20x trong 5 phút.
Dự kiến HPA sẽ scale từ 3 lên ~25 Pod.
Thực tế: HPA chỉ scale lên 5 Pod, CPU trung bình 95%, p99 latency 8 giây.
User complain không thể đặt xe, app timeout.
```

### Task (Nhiệm Vụ)

```
- Scale đủ Pod ngay để handle traffic
- Tìm lý do HPA không scale như kỳ vọng
```

### Action (Hành Động)

```
Bước 1 — Manual scale để restore:
kubectl scale deployment/booking-service --replicas=25
→ Sau 3 phút, Pod tăng lên 25, latency giảm xuống 200ms

Bước 2 — Điều tra HPA không scale:
kubectl describe hpa booking-hpa
→ Events: "FailedGetResourceMetric: unable to get metrics for resource cpu"
→ "failed to get cpu utilization: unable to get metrics"

kubectl get pods -n kube-system | grep metrics-server
→ metrics-server Pod đang ở trạng thái Running nhưng...

kubectl logs metrics-server-xxx -n kube-system
→ Error: "x509: certificate signed by unknown authority"
→ metrics-server không thể scrape kubelet vì certificate issue!

Bước 3 — Fix metrics-server:
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-",
    "value":"--kubelet-insecure-tls"}]'
→ Đây là workaround tạm thời, cần fix certificate đúng đắn sau

kubectl top nodes  → OK, metrics hoạt động
kubectl get hpa    → CURRENT metric hiển thị đúng

Bước 4 — Test lại HPA:
kubectl scale deployment/booking-service --replicas=3
→ Simulate load, HPA scale đúng lên 20+ Pod trong 3 phút

Bước 5 — Fix certificate đúng đắn (hôm sau):
# Regenerate kubelet certificate với proper SAN
# Cấu hình metrics-server dùng certificate đúng thay vì --insecure-tls

Bước 6 — Cải tiến quy trình:
- Thêm smoke test HPA sau mỗi cluster upgrade
- Alert khi HPA ở trạng thái "unable to get metrics" > 2 phút
- Pre-scale cho campaign đã biết trước: scale manual trước 30 phút
```

### Result (Kết Quả)

```
- Downtime-equivalent: 15 phút (latency > 5 giây → user experience xấu)
- Manual scale restore latency về 200ms trong 3 phút
- Root cause: metrics-server certificate expired sau K8s upgrade 2 tuần trước
- Thiệt hại ước tính: giảm 10% booking trong 15 phút
- Action: Certificate rotation automation, HPA smoke test post-upgrade

Bài học:
- HPA là critical path — nên có alert riêng khi HPA không lấy được metric
- Campaign traffic cần pre-scale thủ công dù có HPA (HPA có lag 15–60 giây)
- Certificate expiry cần được track tập trung (cert-manager, external monitoring)
```

---

## Sự Cố 6: Node NotReady Gây Mất 30% Capacity

### Bối Cảnh

Production K8s cluster trên AWS EKS với 10 node. Ứng dụng microservices với tổng 80 Pod trên cluster.

### Situation (Tình Huống)

```
Sáng thứ Ba, 3 trong số 10 node chuyển sang trạng thái NotReady đồng thời.
~24 Pod không được reschedule do không đủ capacity.
Service degradation: một số API call trả về 503 do thiếu Pod.
```

### Action (Hành Động) — Rút Gọn

```
Bước 1 — Xác nhận node status:
kubectl get nodes
→ 3 node status NotReady, Age ~20 phút

Bước 2 — Điều tra:
kubectl describe node <unhealthy-node>
→ Conditions: DiskPressure=True
→ kubelet logs: "disk usage 95%, evicting low priority pods"
→ /var/log và /var/lib/docker đầy

Nguyên nhân: Container log rotation bị disable sau kernel update tháng trước

Bước 3 — Emergency fix:
# SSH vào node (cho phép bởi AWS SSM)
# Xoá old container layers và logs
docker system prune -f
journalctl --vacuum-size=500M
→ Disk usage giảm xuống 45%, node trở lại Ready sau 5 phút

Bước 4 — Remediation:
- Cấu hình logrotate cho containerd
- Tăng EBS volume từ 50GB lên 100GB cho mọi node
- Thêm alert Prometheus khi disk usage node > 70%
- Bật --eviction-hard=nodefs.available<10% (kubelet tự evict trước khi full)
```

### Result (Kết Quả)

```
- MTTR: 35 phút (phát hiện + diagnose + fix)
- 3 node restore, Pod reschedule hoàn tất trong 10 phút
- Không mất data (Pod chạy stateless)
- Thiết lập disk monitoring: alert ở 70%, hard limit ở 85%
- Log rotation policy áp dụng qua DaemonSet cho mọi node hiện tại và mới
```

---

## Template Tự Viết Câu Chuyện

Dùng template này để chuẩn bị câu chuyện từ kinh nghiệm thực tế của bạn:

```markdown
## Sự Cố: [Tên Ngắn Gọn]

### Bối Cảnh
- Hệ thống là gì? (loại app, scale)
- Team bao nhiêu người? Role của bạn?
- Đang ở giai đoạn nào? (normal operation / deployment / migration)

### Situation (Tình Huống)
- Khi nào xảy ra? (thời điểm, ngày)
- Biểu hiện cụ thể là gì? (metric, error message, user complaint)
- Tác động cụ thể: bao nhiêu user / % traffic / doanh thu?
- Bạn biết về sự cố qua đâu? (alert, user báo, tự phát hiện)

### Task (Nhiệm Vụ)
- Ưu tiên ngay lập tức là gì? (restore vs investigate)
- Bạn chịu trách nhiệm phần nào?
- Ai cần notify? (manager, customer success, user)

### Action (Hành Động)
Liệt kê từng bước theo thứ tự thời gian:
1. [Bước 1] — lệnh/hành động cụ thể + lý do chọn cách này
2. [Bước 2] — ...
3. [Bước 3] — ...

Highlight:
- Quyết định khó bạn phải đưa ra
- Nơi bạn sai và sửa lại
- Sự hợp tác với người khác

### Result (Kết Quả)
- MTTR là bao lâu?
- Tác động cuối cùng (đã giảm được bao nhiêu so với ban đầu)
- Root cause là gì?
- Action items từ post-mortem: bao nhiêu, hoàn thành chưa?

### Bài Học
- Bạn thay đổi gì về kỹ thuật?
- Bạn thay đổi gì về quy trình / monitoring?
- Điều gì bạn làm khác nếu gặp lại?
```

### Mẹo Điều Chỉnh Cho Từng Cấp Độ

**Junior Engineer:**
- Tập trung vào: tôi học được gì từ sự cố này
- Không cần phải là "người hùng" — kể chuyện bạn support senior fix là OK
- Nhấn mạnh: tôi đã theo dõi và hiểu quy trình debug ra sao

**Mid-level Engineer:**
- Tập trung vào: tôi chủ động debug và tìm ra root cause
- Nêu các tool sử dụng cụ thể (kubectl, Grafana, Jaeger, ...)
- Nhấn mạnh: action items sau sự cố và bạn đã implement gì

**Senior Engineer:**
- Tập trung vào: tôi dẫn dắt incident response, communicate với stakeholder
- Nêu quyết định kiến trúc/thiết kế phòng ngừa dài hạn
- Nhấn mạnh: culture improvement — tôi giúp team học từ sự cố thế nào

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
