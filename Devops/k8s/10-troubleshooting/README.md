# Troubleshooting Kubernetes — Hướng Dẫn Xử Lý Sự Cố

> Hướng dẫn toàn diện cách chẩn đoán và khắc phục các sự cố phổ biến nhất trên Kubernetes (K8s), từ Pod lỗi đến Node gặp vấn đề, mạng không thông và lưu trữ hỏng.

## Mục Lục

1. [Triết Lý Debug Kubernetes](#triết-lý-debug-kubernetes)
2. [Bộ Công Cụ Debug Cốt Lõi](#bộ-công-cụ-debug-cốt-lõi)
3. [Sơ Đồ Phân Loại Sự Cố](#sơ-đồ-phân-loại-sự-cố)
4. [Danh Sách File Trong Module](#danh-sách-file-trong-module)
5. [Quy Trình Debug Tổng Quát](#quy-trình-debug-tổng-quát)
6. [Lệnh Kubectl Debug Thiết Yếu](#lệnh-kubectl-debug-thiết-yếu)
7. [Ma Trận Triệu Chứng — Nguyên Nhân](#ma-trận-triệu-chứng--nguyên-nhân)
8. [Câu Hỏi Phỏng Vấn Về Troubleshooting](#câu-hỏi-phỏng-vấn-về-troubleshooting)

---

## Triết Lý Debug Kubernetes

Kubernetes có tính **declarative** (khai báo) — bạn mô tả trạng thái mong muốn, K8s cố gắng đạt tới trạng thái đó. Khi có sự cố, câu hỏi cần đặt ra là:

```
1. Trạng thái mong muốn (desired state) là gì?
2. Trạng thái hiện tại (actual state) là gì?
3. Khoảng cách giữa hai trạng thái đó là gì?
4. Tại sao K8s không thể thu hẹp khoảng cách đó?
```

### Ba Lớp Cần Kiểm Tra

```
┌─────────────────────────────────────────┐
│  Lớp Ứng Dụng (Application Layer)      │  ← Logs, health probe, config
├─────────────────────────────────────────┤
│  Lớp Kubernetes (Platform Layer)        │  ← Events, resource, scheduling
├─────────────────────────────────────────┤
│  Lớp Hạ Tầng (Infrastructure Layer)    │  ← Node, network, storage, DNS
└─────────────────────────────────────────┘
```

---

## Bộ Công Cụ Debug Cốt Lõi

### kubectl — Công Cụ Dòng Lệnh Chính

```bash
# Xem trạng thái tổng quát của cluster
kubectl get nodes
kubectl get pods -A                        # -A = --all-namespaces (tất cả namespace)
kubectl get events --sort-by=.lastTimestamp

# Mô tả chi tiết một resource (bao gồm Events)
kubectl describe pod <pod-name> -n <namespace>
kubectl describe node <node-name>

# Xem log của Pod
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous  # log của lần chạy trước
kubectl logs <pod-name> -n <namespace> -c <container-name>  # multi-container

# Chạy lệnh trong container đang chạy
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Chạy Pod debug tạm thời
kubectl run debug-pod --image=nicolaka/netshoot --rm -it -- /bin/bash
```

### k9s — Giao Diện TUI (Terminal User Interface — Giao Diện Người Dùng Terminal)

```bash
k9s                          # Mở giao diện tương tác
k9s -n <namespace>           # Mở trong namespace cụ thể
# Phím tắt trong k9s:
# :pod     — xem danh sách Pod
# :node    — xem danh sách Node
# l        — xem logs của Pod được chọn
# d        — describe resource được chọn
# s        — shell vào container
```

### Các Công Cụ Bổ Trợ

| Công Cụ       | Mục Đích                                                   |
| ------------- | ---------------------------------------------------------- |
| `stern`       | Xem log nhiều Pod cùng lúc theo label                      |
| `kubectx`     | Chuyển đổi nhanh giữa các cluster context                 |
| `kubens`      | Chuyển đổi nhanh giữa các namespace                       |
| `kube-score`  | Phân tích manifest theo best practice                     |
| `popeye`      | Quét toàn cluster tìm vấn đề cấu hình                    |
| `netshoot`    | Pod chứa các công cụ debug mạng (curl, dig, nslookup...) |

---

## Sơ Đồ Phân Loại Sự Cố

```
Sự Cố Kubernetes
│
├── Pod có vấn đề?
│   ├── CrashLoopBackOff   → pod-errors.md
│   ├── ImagePullBackOff   → pod-errors.md
│   ├── Pending            → pod-errors.md
│   ├── OOMKilled          → pod-errors.md / performance-issues.md
│   └── Terminating mãi    → pod-errors.md
│
├── Node có vấn đề?
│   ├── NotReady           → node-issues.md
│   ├── MemoryPressure     → node-issues.md
│   ├── DiskPressure       → node-issues.md
│   └── Taint/Toleration   → node-issues.md
│
├── Mạng không thông?
│   ├── Service unreachable → networking-debug.md
│   ├── DNS fail           → networking-debug.md
│   ├── Ingress 502/504    → networking-debug.md
│   └── Pod không ping nhau → networking-debug.md
│
├── Storage lỗi?
│   ├── PVC Pending        → storage-issues.md
│   ├── Mount error        → storage-issues.md
│   └── Disk full          → storage-issues.md
│
└── Hiệu năng kém?
    ├── CPU throttling     → performance-issues.md
    ├── High latency       → performance-issues.md
    └── OOM thường xuyên  → performance-issues.md
```

---

## Danh Sách File Trong Module

| File                      | Nội Dung                                          | Độ Khó |
| ------------------------- | ------------------------------------------------- | ------ |
| `1-pod-errors.md`           | CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled | ⭐⭐    |
| `2-node-issues.md`          | NotReady node, taint, drain, cordon               | ⭐⭐    |
| `3-networking-debug.md`     | Service không kết nối, DNS fail, Ingress lỗi      | ⭐⭐⭐  |
| `4-storage-issues.md`       | PVC Pending, mount error, disk full               | ⭐⭐    |
| `5-performance-issues.md`   | Throttling, high latency, resource starvation     | ⭐⭐⭐  |
| `6-incident-playbook.md`    | Runbook xử lý sự cố theo từng tình huống          | ⭐⭐⭐  |
| `7-production-checklist.md` | Checklist trước khi đưa lên production            | ⭐⭐    |

---

## Quy Trình Debug Tổng Quát

### Bước 1: Thu Thập Thông Tin (Information Gathering)

```bash
# Kiểm tra tổng quan
kubectl get pods -n <namespace> -o wide

# Xem events gần nhất (events là nguồn thông tin quan trọng nhất)
kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -30

# Mô tả Pod để xem chi tiết
kubectl describe pod <pod-name> -n <namespace>
```

### Bước 2: Kiểm Tra Log

```bash
# Log container hiện tại
kubectl logs <pod-name> -n <namespace> --tail=100

# Log của lần crash trước (nếu Pod đã restart)
kubectl logs <pod-name> -n <namespace> --previous --tail=100

# Theo dõi log realtime
kubectl logs -f <pod-name> -n <namespace>
```

### Bước 3: Kiểm Tra Resource và Config

```bash
# Xem resource được assign cho Pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'

# Kiểm tra ConfigMap và Secret mà Pod dùng
kubectl describe pod <pod-name> | grep -A5 "Volumes\|Env"

# Xem node mà Pod đang chạy và tài nguyên còn lại
kubectl describe node <node-name> | grep -A5 "Allocated resources"
```

### Bước 4: Kiểm Tra Mạng

```bash
# Kiểm tra Service có endpoint không
kubectl get endpoints <service-name> -n <namespace>

# Kiểm tra DNS từ bên trong cluster
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>.<namespace>.svc.cluster.local

# Kiểm tra kết nối tới Service
kubectl run curl-test --image=curlimages/curl --rm -it -- curl http://<service-name>.<namespace>.svc.cluster.local
```

### Bước 5: Kiểm Tra Storage

```bash
# Xem trạng thái PVC
kubectl get pvc -n <namespace>

# Xem chi tiết PV được bind
kubectl describe pvc <pvc-name> -n <namespace>
```

---

## Lệnh Kubectl Debug Thiết Yếu

### Lệnh Phổ Biến Nhất

```bash
# === TRẠNG THÁI ===
kubectl get pods -n <ns> -o wide --show-labels
kubectl get all -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp

# === CHI TIẾT ===
kubectl describe pod <name> -n <ns>
kubectl describe node <name>
kubectl describe service <name> -n <ns>

# === LOG ===
kubectl logs <name> -n <ns>
kubectl logs <name> -n <ns> --previous
kubectl logs <name> -n <ns> -c <container>
kubectl logs -l app=<label> -n <ns> --all-containers  # nhiều Pod

# === SHELL ===
kubectl exec -it <name> -n <ns> -- sh
kubectl exec -it <name> -n <ns> -c <container> -- bash

# === DEBUG POD ===
kubectl debug <pod-name> -it --image=busybox --copy-to=debug-pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# === RESOURCE ===
kubectl top pods -n <ns>
kubectl top nodes

# === XEM RAW YAML ===
kubectl get pod <name> -n <ns> -o yaml
kubectl get pod <name> -n <ns> -o json | jq '.status'

# === PORT FORWARD (kiểm tra trực tiếp) ===
kubectl port-forward pod/<name> 8080:80 -n <ns>
kubectl port-forward svc/<name> 8080:80 -n <ns>
```

### Lệnh Phân Tích Nhanh

```bash
# Đếm số Pod theo trạng thái
kubectl get pods -A | grep -c Running
kubectl get pods -A | grep -v Running | grep -v Completed

# Tìm Pod bị restart nhiều lần
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount' | tail -10

# Tìm Pod đang dùng nhiều tài nguyên nhất
kubectl top pods -A --sort-by=cpu | head -10
kubectl top pods -A --sort-by=memory | head -10

# Xem tất cả container trong Pod
kubectl get pod <name> -o jsonpath='{.spec.containers[*].name}'
```

---

## Ma Trận Triệu Chứng — Nguyên Nhân

| Triệu Chứng                 | Nguyên Nhân Hay Gặp                                  | File Tham Khảo           |
| --------------------------- | ---------------------------------------------------- | ------------------------ |
| CrashLoopBackOff            | Ứng dụng lỗi, config sai, port bị chiếm             | `pod-errors.md`          |
| ImagePullBackOff            | Tên image sai, thiếu credential registry             | `pod-errors.md`          |
| Pod Pending                 | Không đủ tài nguyên, node không phù hợp, PVC chưa bound | `pod-errors.md`      |
| OOMKilled                   | Memory limit quá thấp, memory leak                   | `pod-errors.md`          |
| Node NotReady               | kubelet chết, đĩa đầy, mạng node hỏng               | `node-issues.md`         |
| Service unreachable         | Selector sai, endpoint rỗng, NetworkPolicy chặn      | `networking-debug.md`    |
| DNS không resolve           | CoreDNS crash, search domain sai, Pod isolation      | `networking-debug.md`    |
| Ingress 502/504             | Backend Pod chưa sẵn sàng, timeout, upstream lỗi    | `networking-debug.md`    |
| PVC Pending                 | StorageClass không tồn tại, quota hết, provisioner lỗi | `storage-issues.md`   |
| CPU throttling              | CPU limit quá thấp so với nhu cầu thực               | `performance-issues.md`  |
| High latency                | Resource starvation, network congestion, GC pause    | `performance-issues.md`  |

---

## Câu Hỏi Phỏng Vấn Về Troubleshooting

### Câu Hỏi Cơ Bản

**Q: Pod của bạn ở trạng thái CrashLoopBackOff — bạn làm gì đầu tiên?**

> **A:** Kiểm tra theo thứ tự: (1) `kubectl describe pod` để xem Events — thường tiết lộ nguyên nhân ngay. (2) `kubectl logs --previous` để xem log của lần crash trước đó. (3) Xem Exit Code: 137 = OOMKilled, 1 = lỗi ứng dụng, 2 = configuration error.

**Q: Service không kết nối được — bạn debug thế nào?**

> **A:** Kiểm tra theo mô hình từ trong ra ngoài: (1) Kiểm tra Pod có running không. (2) `kubectl get endpoints <svc>` — endpoint rỗng nghĩa là selector không match. (3) Thử `curl` từ trong cluster. (4) Kiểm tra NetworkPolicy có chặn không. (5) Kiểm tra port trong Service spec có khớp với containerPort không.

**Q: Làm thế nào để debug vấn đề mạng trong Kubernetes?**

> **A:** Dùng Pod netshoot (`kubectl run debug --image=nicolaka/netshoot --rm -it`), sau đó kiểm tra DNS (`nslookup`), kết nối (`curl`, `telnet`), route (`traceroute`), và iptables rules trên node.

### Câu Hỏi Nâng Cao

**Q: Một ứng dụng bị chậm đột ngột trong giờ cao điểm — nguyên nhân có thể là gì trong K8s?**

> **A:** CPU throttling (limit quá thấp), HPA chưa kịp scale, memory pressure gây swap, node-level noisy neighbor, network congestion, hoặc backend dependency (database, external API) chậm.

**Q: Bạn phân biệt liveness probe fail và readiness probe fail như thế nào?**

> **A:** Liveness probe fail → K8s restart container (tăng restartCount). Readiness probe fail → K8s xóa Pod khỏi Endpoints nhưng không restart. Xem `kubectl describe pod` để biết probe nào fail.

---

## Tài Nguyên Tham Khảo

- [Kubernetes Debug Pod](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Kubernetes Debug Service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [nicolaka/netshoot](https://github.com/nicolaka/netshoot) — Swiss Army Knife for Network Troubleshooting

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
