# Incident Playbook — Runbook Xử Lý Sự Cố Kubernetes

> Runbook (sổ tay vận hành) với hướng dẫn từng bước để xử lý các sự cố Kubernetes phổ biến trong môi trường production. Mỗi tình huống có triệu chứng, chẩn đoán và các bước hành động cụ thể.

## Mục Lục

1. [Quy Trình Xử Lý Sự Cố Chuẩn](#quy-trình-xử-lý-sự-cố-chuẩn)
2. [Playbook 1: Dịch Vụ Không Phản Hồi (Service Down)](#playbook-1-dịch-vụ-không-phản-hồi)
3. [Playbook 2: Deployment Bị Stuck Khi Rollout](#playbook-2-deployment-bị-stuck-khi-rollout)
4. [Playbook 3: Cluster Hết Tài Nguyên](#playbook-3-cluster-hết-tài-nguyên)
5. [Playbook 4: Node NotReady Hàng Loạt](#playbook-4-node-notready-hàng-loạt)
6. [Playbook 5: Database Pod Crash](#playbook-5-database-pod-crash)
7. [Playbook 6: Certificate Hết Hạn](#playbook-6-certificate-hết-hạn)
8. [Playbook 7: etcd Sức Khoẻ Kém](#playbook-7-etcd-sức-khoẻ-kém)
9. [Playbook 8: Triển Khai Gây Tăng Error Rate](#playbook-8-triển-khai-gây-tăng-error-rate)
10. [Post-Mortem Template (Mẫu Phân Tích Sau Sự Cố)](#post-mortem-template)

---

## Quy Trình Xử Lý Sự Cố Chuẩn

### Mức Độ Nghiêm Trọng (Severity)

| Mức | Tên         | Định Nghĩa                                        | Thời Gian Phản Hồi |
| --- | ----------- | ------------------------------------------------- | ------------------ |
| P1  | Critical    | Dịch vụ hoàn toàn không hoạt động, ảnh hưởng 100% người dùng | < 5 phút   |
| P2  | High        | Chức năng quan trọng bị ảnh hưởng, > 25% người dùng | < 15 phút     |
| P3  | Medium      | Một số chức năng bị ảnh hưởng, < 25% người dùng   | < 1 giờ           |
| P4  | Low         | Vấn đề nhỏ, không ảnh hưởng trực tiếp đến người dùng | < 24 giờ       |

### Quy Trình 5 Bước

```
1. DETECT (Phát Hiện)   — Nhận alert, xác nhận sự cố đang xảy ra
2. TRIAGE (Phân Loại)   — Đánh giá severity, thông báo đội nhóm
3. DIAGNOSE (Chẩn Đoán) — Tìm root cause (nguyên nhân gốc rễ)
4. MITIGATE (Giảm Nhẹ)  — Khôi phục dịch vụ nhanh nhất có thể
5. RESOLVE (Giải Quyết) — Sửa nguyên nhân gốc, ngăn tái phát
```

### Nguyên Tắc Cơ Bản Khi Xử Lý Sự Cố

```
✅ Bình tĩnh — panic làm chậm mọi thứ
✅ Ghi lại tất cả — mọi lệnh bạn chạy, kết quả quan sát được
✅ Ưu tiên khôi phục dịch vụ trước, tìm nguyên nhân sau
✅ Thông báo stakeholder — không im lặng
✅ Một người chỉ huy — tránh nhiều người cùng thực hiện thay đổi
✅ Rollback nếu có thể — thay đổi ít nhất để đạt ổn định
✅ Cẩn thận với force delete, gracePeriod=0
```

---

## Playbook 1: Dịch Vụ Không Phản Hồi

### Triệu Chứng

- Alert: `service_up = 0` hoặc `HTTP 5xx rate > threshold`
- Người dùng báo cáo không truy cập được
- Ingress trả về 503 hoặc 502

### Bước 1: Xác Nhận Sự Cố

```bash
# Kiểm tra từ bên ngoài
curl -v https://api.example.com/health

# Kiểm tra từ bên trong cluster
kubectl run test --rm -it --image=curlimages/curl -- \
  curl http://<service-name>.<namespace>.svc.cluster.local/health

# Xem error rate hiện tại (nếu có Prometheus)
# rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

### Bước 2: Kiểm Tra Pod

```bash
kubectl get pods -n <namespace> -o wide
# Nếu có Pod không Running → đây là nguyên nhân

kubectl describe pod <failing-pod> -n <namespace>
kubectl logs <failing-pod> -n <namespace> --previous
```

### Bước 3: Kiểm Tra Endpoint

```bash
kubectl get endpoints <service-name> -n <namespace>
# Nếu <none> → không có Pod healthy nào

# Xem selector có match không
kubectl describe svc <service-name> -n <namespace>
```

### Bước 4: Cô Lập Vấn Đề

```bash
# Test trực tiếp một Pod (bypass Service)
kubectl port-forward pod/<pod-name> 8080:80 -n <namespace>
curl http://localhost:8080/health

# Nếu Pod OK → vấn đề ở Service/Ingress
# Nếu Pod không OK → vấn đề ở ứng dụng/config
```

### Bước 5: Khôi Phục Nhanh

```bash
# Option A: Rollback deployment
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace>

# Option B: Restart deployment
kubectl rollout restart deployment/<name> -n <namespace>

# Option C: Scale up thêm replica (nếu đang có ít replica)
kubectl scale deployment/<name> --replicas=5 -n <namespace>

# Option D: Chuyển traffic sang môi trường khác (Blue-Green)
# Cập nhật Service selector sang stable version
kubectl patch svc <name> -n <namespace> \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Bước 6: Xác Nhận Đã Khôi Phục

```bash
# Kiểm tra Pod healthy
kubectl get pods -n <namespace>

# Kiểm tra endpoint có traffic
kubectl get endpoints <service-name> -n <namespace>

# Test lại từ ngoài
curl -v https://api.example.com/health
```

---

## Playbook 2: Deployment Bị Stuck Khi Rollout

### Triệu Chứng

- `kubectl rollout status` không hoàn thành sau > 10 phút
- Pod mới ở trạng thái Pending hoặc CrashLoopBackOff
- Pod cũ vẫn chạy (chưa bị xóa)

### Chẩn Đoán

```bash
# Xem trạng thái rollout
kubectl rollout status deployment/<name> -n <namespace>
# Thường thấy: "Waiting for deployment 'xxx' rollout to finish..."

# Xem ReplicaSet history
kubectl get rs -n <namespace> | grep <deployment-name>

# Xem Pod của rollout mới
kubectl get pods -n <namespace> -l <selector>

# Xem lý do Pod mới chưa sẵn sàng
kubectl describe pod <new-pod> -n <namespace>
```

### Nguyên Nhân Phổ Biến

```bash
# 1. Pod mới CrashLoopBackOff → ứng dụng lỗi
kubectl logs <new-pod> --previous -n <namespace>

# 2. Pod mới Pending → không đủ tài nguyên
kubectl describe pod <new-pod> | grep -A5 "Events"

# 3. minReadySeconds quá cao
kubectl get deployment <name> -o jsonpath='{.spec.minReadySeconds}'

# 4. Readiness probe fail
kubectl describe pod <new-pod> | grep -A10 "Readiness"

# 5. maxSurge = 0 và maxUnavailable = 0 (không thể rollout)
kubectl get deployment <name> -o jsonpath='{.spec.strategy}'
```

### Hành Động

```bash
# Nếu ứng dụng mới lỗi → rollback ngay
kubectl rollout undo deployment/<name> -n <namespace>

# Xác nhận rollback thành công
kubectl rollout status deployment/<name> -n <namespace>
kubectl get pods -n <namespace>

# Nếu muốn pause rollout (để điều tra)
kubectl rollout pause deployment/<name> -n <namespace>
# ... điều tra ...
kubectl rollout resume deployment/<name> -n <namespace>
```

---

## Playbook 3: Cluster Hết Tài Nguyên

### Triệu Chứng

- Pod mới liên tục ở Pending
- Alert: `node_cpu_usage > 90%` hoặc `node_memory_usage > 90%`
- HPA đã max replica nhưng vẫn không đủ

### Chẩn Đoán

```bash
# Xem tài nguyên từng node
kubectl top nodes

# Xem pod Pending và lý do
kubectl get pods -A | grep Pending
kubectl describe pod <pending-pod> | grep -A10 "Events"

# Xem allocatable resource còn lại
kubectl describe nodes | grep -A5 "Allocated resources" | grep -v "^--"
```

### Hành Động Ngắn Hạn

```bash
# Tăng số lượng node (nếu có Cluster Autoscaler)
# Cluster Autoscaler sẽ tự thêm node khi có Pod Pending đủ lâu

# Thêm node thủ công (AWS)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name <asg-name> \
  --desired-capacity <new-capacity>

# Thêm node thủ công (GKE)
gcloud container clusters resize <cluster> --node-pool <pool> --num-nodes <n>

# Giảm tải khẩn cấp — xóa Pod không quan trọng
kubectl delete pod <batch-job-pod> -n <namespace>

# Tạm thời giảm replica của deployment ít quan trọng hơn
kubectl scale deployment/<low-priority> --replicas=1 -n <namespace>
```

### Hành Động Dài Hạn

```bash
# Kiểm tra Pod không có resource request (BestEffort — nguy hiểm)
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[0].resources == {}) | .metadata.name'

# Thiết lập ResourceQuota và LimitRange cho mọi namespace
# Xem production-checklist.md để biết thêm

# Cấu hình Cluster Autoscaler đúng
kubectl describe deployment cluster-autoscaler -n kube-system
```

---

## Playbook 4: Node NotReady Hàng Loạt

### Triệu Chứng

- Nhiều node (> 2) chuyển sang NotReady cùng lúc
- Nhiều Pod bị evict và stuck Terminating
- Alert: cluster-level availability giảm đột ngột

### Chẩn Đoán

```bash
# Xem trạng thái tất cả node
kubectl get nodes

# Xem events cluster-wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50

# Kiểm tra control plane
kubectl get pods -n kube-system
kubectl get componentstatuses  # Deprecated nhưng vẫn hữu ích
```

### Phân Loại Nguyên Nhân

```bash
# Kiểm tra nếu là vấn đề mạng (network partition)
# → Nhiều node mất kết nối cùng lúc, thường theo zone/AZ

# Kiểm tra nếu là vấn đề cloud provider
# → Xem AWS/GCP/Azure status page

# Kiểm tra nếu là vấn đề etcd
kubectl exec -it etcd-<master> -n kube-system -- \
  etcdctl endpoint health --cluster

# Kiểm tra API Server
kubectl get nodes  # Nếu lệnh này không chạy → API Server có vấn đề
curl -k https://<api-server>:6443/healthz
```

### Hành Động

```bash
# Nếu là vấn đề mạng trên node — SSH vào điều tra
ssh <node-ip>
systemctl status kubelet
journalctl -u kubelet -n 50

# Nếu disk full — dọn ngay
df -h && docker system prune -f

# Nếu node hoàn toàn không phục hồi — drain và replace
kubectl drain <node> --ignore-daemonsets --force
# Terminate node cũ, tạo node mới từ ASG

# Nếu API Server down — liên hệ team platform ngay
# Không có gì có thể làm mà không có API Server
```

---

## Playbook 5: Database Pod Crash

### Áp Dụng Cho

- PostgreSQL, MySQL, MongoDB, Redis chạy trong StatefulSet

### Triệu Chứng

- Database Pod CrashLoopBackOff hoặc OOMKilled
- Ứng dụng lỗi kết nối database
- `connection refused` hoặc `connection timeout`

### Chẩn Đoán — Thứ Tự Ưu Tiên

```bash
# 1. Xem log database (thường có thông tin rõ ràng)
kubectl logs <db-pod> -n <namespace> --previous | tail -100

# 2. Kiểm tra disk (database cần disk, dễ đầy)
kubectl exec -it <db-pod> -- df -h /data

# 3. Kiểm tra memory (OOMKilled?)
kubectl describe pod <db-pod> | grep -A5 "Last State"

# 4. Kiểm tra PVC còn tồn tại không
kubectl get pvc -n <namespace>
```

### Xử Lý PostgreSQL Crash

```bash
# Kiểm tra log PostgreSQL
kubectl logs <postgres-pod> -n <ns> | grep -E "FATAL|ERROR|PANIC"

# PostgreSQL crash thường do:
# - Disk full → kiểm tra df -h /var/lib/postgresql
# - Out of shared_buffers → kiểm tra config
# - Corrupt data (cần pg_resetwal — chỉ khi có backup!)

# Nếu disk full — giải phóng ngay
kubectl exec -it <postgres-pod> -- psql -c "VACUUM FULL;"
kubectl exec -it <postgres-pod> -- psql -c "SELECT pg_size_pretty(pg_database_size('mydb'));"
```

### Xử Lý Redis Crash

```bash
# Kiểm tra log Redis
kubectl logs <redis-pod> -n <ns> | grep -i "oom\|error\|warn" | tail -30

# Redis thường crash do maxmemory-policy không phù hợp
# Hoặc AOF persistence file bị corrupt

# Restart Redis (mất dữ liệu in-memory, persistence OK)
kubectl delete pod <redis-pod> -n <ns>  # StatefulSet tự tạo lại
```

---

## Playbook 6: Certificate Hết Hạn

### Triệu Chứng

- `x509: certificate has expired or is not yet valid`
- kubectl không kết nối được API Server
- Ingress trả về TLS error
- Các service không giao tiếp được nhau (mutual TLS)

### Loại Certificate Trong Kubernetes

```
1. Cluster certificates (do kubeadm quản lý):
   - API Server cert
   - etcd cert
   - kubelet client cert
   - Controller Manager cert
   - Scheduler cert
   Hạn: 1 năm (mặc định kubeadm)

2. Ingress TLS certificates:
   - Cert được mount vào Ingress (từ Secret)
   - Thường quản lý bởi cert-manager

3. Application-level certificates:
   - mTLS giữa các service (Istio, Linkerd tự quản lý)
```

### Kiểm Tra Certificate Sắp Hết Hạn

```bash
# Kiểm tra cluster certificates (kubeadm)
kubeadm certs check-expiration

# Ví dụ output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 May 10, 2027 09:00 UTC   364d
# apiserver                  May 10, 2027 09:00 UTC   364d
# apiserver-kubelet-client   May 10, 2027 09:00 UTC   364d

# Kiểm tra Ingress TLS cert
kubectl get secret -n <ns> | grep tls
kubectl get secret <tls-secret> -n <ns> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -enddate

# Kiểm tra cert-manager certificates
kubectl get certificate -A
kubectl describe certificate <name> -n <ns>
```

### Gia Hạn Certificate

```bash
# Gia hạn tất cả cluster certificates
kubeadm certs renew all

# Restart control plane sau khi renew
kubectl -n kube-system delete pod -l component=kube-apiserver
kubectl -n kube-system delete pod -l component=kube-controller-manager
kubectl -n kube-system delete pod -l component=kube-scheduler

# Gia hạn Ingress cert (cert-manager)
kubectl delete certificate <name> -n <ns>  # cert-manager tự tạo lại
# Hoặc
kubectl annotate certificate <name> -n <ns> cert-manager.io/issue-temporary-certificate="true"
```

---

## Playbook 7: etcd Sức Khoẻ Kém

### Triệu Chứng

- API Server chậm hoặc không phản hồi
- `etcd cluster is unavailable`
- Thao tác kubectl bị timeout

### Kiểm Tra etcd

```bash
# Xem Pod etcd
kubectl get pods -n kube-system | grep etcd

# Kiểm tra sức khoẻ etcd cluster
kubectl exec -it etcd-<master> -n kube-system -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster

# Kiểm tra disk etcd
kubectl exec -it etcd-<master> -n kube-system -- \
  etcdctl endpoint status --cluster -w table
# Xem DB SIZE — nếu > 2-4GB cần compact

# Xem leader election
kubectl exec -it etcd-<master> -n kube-system -- \
  etcdctl endpoint status --cluster | grep -i leader
```

### Compact và Defrag etcd

```bash
# Lấy revision hiện tại
REV=$(kubectl exec etcd-<master> -n kube-system -- \
  etcdctl endpoint status --write-out fields | grep Revision | awk '{print $3}')

# Compact (nén lịch sử cũ)
kubectl exec etcd-<master> -n kube-system -- \
  etcdctl compact $REV

# Defragment (giải phóng disk space)
kubectl exec etcd-<master> -n kube-system -- \
  etcdctl defrag --cluster
```

### etcd Disk Quota Exceeded

```bash
# Alarm sẽ khiến etcd từ chối write
# Kiểm tra alarm
etcdctl alarm list

# Sau khi compact và defrag, xóa alarm
etcdctl alarm disarm
```

---

## Playbook 8: Triển Khai Gây Tăng Error Rate

### Phát Hiện

```bash
# Alert: http_error_rate tăng sau khi deploy
# Xem deployment lịch sử
kubectl rollout history deployment/<name> -n <namespace>

# Xem diff giữa revision
kubectl rollout history deployment/<name> -n <namespace> --revision=5
kubectl rollout history deployment/<name> -n <namespace> --revision=6
```

### Quyết Định Nhanh: Rollback Hay Tiếp Tục

```bash
# Kiểm tra error rate thực tế
# Nếu error rate > 5% → rollback ngay không cần điều tra

# Rollback về revision trước
kubectl rollout undo deployment/<name> -n <namespace>

# Rollback về revision cụ thể
kubectl rollout undo deployment/<name> -n <namespace> --to-revision=5

# Theo dõi rollback
kubectl rollout status deployment/<name> -n <namespace>
watch kubectl get pods -n <namespace>
```

### Sau Khi Rollback — Điều Tra

```bash
# Xem log của version mới trước khi rollback
kubectl logs <pod-from-new-version> -n <namespace> --previous | tail -100

# Xem events xung quanh thời điểm deploy
kubectl get events -n <namespace> --sort-by=.lastTimestamp | \
  grep --after-context=2 --before-context=2 <deployment-name>

# So sánh config cũ và mới
kubectl diff -f new-deployment.yaml
```

---

## Post-Mortem Template

### Mục Đích

Post-mortem (phân tích sau sự cố) không phải để tìm người có lỗi mà để **tìm lỗi hệ thống** và **ngăn tái phát**.

### Template

```markdown
## Post-Mortem: [Tên Sự Cố] — [Ngày]

### Tóm Tắt

Mô tả ngắn gọn: sự cố gì, ảnh hưởng gì, kéo dài bao lâu.

### Thời Gian (Timeline)

| Thời Gian | Sự Kiện |
|-----------|---------|
| HH:MM     | Alert được kích hoạt |
| HH:MM     | Kỹ sư on-call nhận alert |
| HH:MM     | Xác định nguyên nhân |
| HH:MM     | Bắt đầu mitigation |
| HH:MM     | Dịch vụ phục hồi |
| HH:MM     | Incident closed |

### Ảnh Hưởng

- Thời gian downtime: X phút
- Người dùng bị ảnh hưởng: Y%
- Error rate peak: Z%
- Tác động kinh doanh: ...

### Nguyên Nhân Gốc Rễ (Root Cause)

Mô tả kỹ thuật về nguyên nhân thực sự.
Tránh dừng ở "lỗi của developer" — tìm lỗi hệ thống cho phép điều đó xảy ra.

### Yếu Tố Gây Ra (Contributing Factors)

1. Thiếu test coverage cho case X
2. Alert threshold quá cao (không phát hiện sớm)
3. Quy trình deploy không có canary

### Những Gì Đã Làm Tốt

- Phát hiện trong X phút
- Rollback thành công trong Y phút
- Thông báo stakeholder kịp thời

### Những Gì Cần Cải Thiện

- Thời gian phát hiện quá chậm
- Không có runbook → tốn nhiều thời gian điều tra
- Thiếu metric quan trọng

### Action Items (Việc Cần Làm)

| Hạng Mục | Người Phụ Trách | Hạn Chót | Trạng Thái |
|----------|-----------------|----------|------------|
| Thêm alert cho metric X | @engineer-a | 2026-05-17 | [ ] |
| Viết runbook cho scenario Y | @engineer-b | 2026-05-20 | [ ] |
| Thêm integration test cho case Z | @engineer-c | 2026-05-24 | [ ] |
```

---

## Checklist On-Call (Trực Vận Hành)

### Trước Ca Trực

```markdown
- [ ] Đọc handover notes từ ca trước
- [ ] Kiểm tra tất cả alert đang mở
- [ ] Xem deployment scheduled trong ca này
- [ ] Có đủ quyền access vào cluster, cloud console, Slack
- [ ] Biết số điện thoại escalation (leo thang) nếu cần
```

### Khi Nhận Alert

```markdown
- [ ] Ghi lại thời gian nhận alert
- [ ] Xác nhận alert đang thực (không phải false positive)
- [ ] Đánh giá severity (P1/P2/P3/P4)
- [ ] Thông báo vào channel #incidents nếu P1/P2
- [ ] Bắt đầu ghi chú (timeline, lệnh đã chạy, kết quả)
- [ ] Follow playbook tương ứng
```

### Sau Khi Giải Quyết

```markdown
- [ ] Xác nhận dịch vụ 100% recovered
- [ ] Thông báo stakeholder incident closed
- [ ] Ghi lại tóm tắt trong #incidents channel
- [ ] Schedule post-mortem meeting (nếu P1/P2)
- [ ] Tạo ticket cho action items
```

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
