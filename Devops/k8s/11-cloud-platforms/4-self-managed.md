# Self-Managed Kubernetes — Tự Quản Lý Cluster

> Hướng dẫn toàn diện về cài đặt và vận hành Kubernetes tự quản lý: kubeadm, k3s, RKE2 và Kubespray — từ on-premise, bare-metal đến môi trường air-gapped và edge computing.

## Mục Lục

1. [Khi Nào Dùng Self-Managed](#khi-nào-dùng-self-managed)
2. [So Sánh Các Phân Phối](#so-sánh-các-phân-phối)
3. [kubeadm — Cài Đặt Chuẩn Kubernetes](#kubeadm--cài-đặt-chuẩn-kubernetes)
4. [k3s — Kubernetes Nhẹ Cho Edge và Lab](#k3s--kubernetes-nhẹ-cho-edge-và-lab)
5. [RKE2 — Kubernetes Tập Trung Bảo Mật](#rke2--kubernetes-tập-trung-bảo-mật)
6. [Kubespray — Cài Đặt Bằng Ansible](#kubespray--cài-đặt-bằng-ansible)
7. [etcd — Quản Lý Và Backup](#etcd--quản-lý-và-backup)
8. [Upgrade Self-Managed Cluster](#upgrade-self-managed-cluster)
9. [Networking Self-Managed](#networking-self-managed)
10. [Storage Self-Managed](#storage-self-managed)
11. [High Availability (HA) — Tính Sẵn Sàng Cao](#high-availability-ha--tính-sẵn-sàng-cao)
12. [Monitoring và Logging](#monitoring-và-logging)
13. [Bảo Mật Self-Managed](#bảo-mật-self-managed)
14. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khi Nào Dùng Self-Managed

### Lý Do Chọn Self-Managed

```
✅ On-premise / Bare-metal:
   - Không có cloud provider hoặc không muốn dùng cloud
   - Phần cứng vật lý riêng, data center riêng

✅ Air-gapped Environment (Môi Trường Cách Ly Mạng):
   - Hệ thống không có kết nối Internet (chính phủ, quốc phòng, tài chính)
   - Yêu cầu kiểm soát hoàn toàn về image và binary

✅ Tối Ưu Chi Phí:
   - Tự vận hành rẻ hơn nếu đội đủ năng lực
   - Không trả phí managed service
   - Tận dụng phần cứng đã có sẵn

✅ Toàn Quyền Kiểm Soát:
   - Cần cấu hình đặc biệt không có trên managed service
   - Yêu cầu compliance đặc thù (cần audit từng component)

✅ Edge Computing / IoT:
   - Tài nguyên hạn chế (Raspberry Pi, mini PC)
   - Cần Kubernetes nhẹ: k3s phù hợp
```

### Trách Nhiệm Khi Tự Quản Lý

```
Bạn chịu trách nhiệm TẤT CẢ:

Control Plane:
  - etcd backup và restore
  - API Server HA và load balancing
  - Certificate rotation (chứng chỉ hết hạn thường gây sự cố lớn)
  - Upgrade an toàn (từng phiên bản một)

Worker Node:
  - OS patching và security update
  - kubelet, kube-proxy, container runtime upgrade
  - Node replacement khi phần cứng hỏng

Networking:
  - CNI plugin cài đặt và upgrade
  - Load Balancer nếu không có cloud LB

Storage:
  - Storage solution (Rook-Ceph, Longhorn, NFS...)
  - Backup và disaster recovery
```

---

## So Sánh Các Phân Phối

| Tiêu Chí                    | kubeadm           | k3s               | RKE2              | Kubespray         |
| --------------------------- | ----------------- | ----------------- | ----------------- | ----------------- |
| **Nhà phát triển**          | Kubernetes SIG    | Rancher (SUSE)    | Rancher (SUSE)    | Community         |
| **Mục đích chính**          | Production chính thức | Edge, IoT, lab | Security-focused  | Tự động hoá       |
| **Yêu cầu tài nguyên**      | Cao               | Rất thấp          | Trung bình        | Cao               |
| **Cài đặt**                 | Thủ công          | 1 lệnh            | Đơn giản          | Ansible playbook  |
| **Container Runtime**       | Bất kỳ            | containerd        | containerd        | Bất kỳ            |
| **etcd**                    | Bên ngoài         | SQLite / etcd     | etcd              | Bên ngoài         |
| **Air-gapped**              | Có (phức tạp)     | Có (đơn giản)     | Có (tốt nhất)     | Có (phức tạp)     |
| **CIS Benchmark**           | Không tích hợp    | Không tích hợp    | Tích hợp sẵn      | Không tích hợp    |
| **Phù hợp**                 | Học K8s sâu       | Edge, dev, lab    | Enterprise secure | Nhiều node lớn    |

---

## kubeadm — Cài Đặt Chuẩn Kubernetes

**kubeadm** là công cụ chính thức của K8s upstream để bootstrap cluster.

### Yêu Cầu Hệ Thống

```
Control Plane Node:
  - CPU: tối thiểu 2 vCPU
  - RAM: tối thiểu 2 GB
  - Disk: tối thiểu 50 GB
  - OS: Ubuntu 22.04 / 20.04, CentOS 8, RHEL 8/9

Worker Node:
  - CPU: tối thiểu 1 vCPU
  - RAM: tối thiểu 1 GB
  - Disk: tối thiểu 30 GB

Mạng:
  - Swap phải tắt (kubelet không hoạt động khi swap bật)
  - Port cần mở: 6443 (API Server), 2379-2380 (etcd), 10250 (kubelet)
  - IP duy nhất cho mỗi node
```

### Bước 1: Chuẩn Bị Tất Cả Node

```bash
# Tắt swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Bật kernel module cần thiết
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Cấu hình sysctl
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

# Cài containerd (container runtime — thời gian chạy container)
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Bước 2: Cài kubeadm, kubelet, kubectl

```bash
# Thêm Kubernetes apt repository
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Giữ nguyên phiên bản — không tự động upgrade
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable kubelet
```

### Bước 3: Khởi Tạo Control Plane

```bash
# Khởi tạo cluster (chạy trên Control Plane node)
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \    # CIDR cho Pod IP (Calico dùng dải này)
  --control-plane-endpoint=k8s-api.example.com \  # DNS/IP của load balancer (cho HA)
  --upload-certs                           # Upload cert để add Control Plane node sau

# Output sẽ có lệnh join — LƯU LẠI
# Worker node join:
#   kubeadm join k8s-api.example.com:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
# Control Plane join:
#   kubeadm join k8s-api.example.com:6443 --token <TOKEN> ... --control-plane --certificate-key <KEY>

# Cấu hình kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Bước 4: Cài CNI Plugin (Calico)

```bash
# Cài Calico (CNI plugin — plugin mạng container)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 192.168.0.0/16     # Phải khớp với --pod-network-cidr
        encapsulation: VXLANCrossSubnet
EOF

# Chờ Calico ready
kubectl get pods -n calico-system -w
```

### Bước 5: Join Worker Node

```bash
# Chạy trên mỗi worker node (lệnh lấy từ output của kubeadm init)
sudo kubeadm join k8s-api.example.com:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>

# Kiểm tra từ control plane
kubectl get nodes
```

### kubeadm — Lệnh Thường Dùng

```bash
# Tạo token mới (token cũ expire sau 24h)
kubeadm token create --print-join-command

# Xem certificate sắp hết hạn
kubeadm certs check-expiration

# Rotate certificate thủ công
kubeadm certs renew all

# Reset node (xoá K8s khỏi node)
sudo kubeadm reset
```

---

## k3s — Kubernetes Nhẹ Cho Edge và Lab

**k3s** là phân phối K8s nhẹ của Rancher/SUSE — đóng gói toàn bộ K8s trong một binary ~70 MB. Phù hợp edge computing, IoT, Raspberry Pi, môi trường tài nguyên thấp, và lab cá nhân.

### Điểm Khác Biệt Của k3s

```
Đã loại bỏ / thay thế:
  - etcd → SQLite (mặc định, single node), hoặc etcd (HA)
  - Cloud provider code → loại bỏ để giảm kích thước
  - Alpha feature → loại bỏ

Tích hợp sẵn (không cần cài thêm):
  - Traefik Ingress Controller
  - CoreDNS
  - Flannel (CNI plugin)
  - Local Path Provisioner (storage)
  - ServiceLB (LoadBalancer dùng hostPort)
```

### Cài k3s Server (Control Plane)

```bash
# Cài k3s Server (1 lệnh duy nhất)
curl -sfL https://get.k3s.io | sh -

# Hoặc chỉ định phiên bản và options
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.0+k3s1 sh -s - server \
  --cluster-cidr=10.42.0.0/16 \
  --service-cidr=10.43.0.0/16 \
  --disable=traefik \              # Tắt Traefik nếu dùng NGINX
  --write-kubeconfig-mode=644

# Lấy kubeconfig
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Lấy node token để join worker
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Cài k3s Agent (Worker Node)

```bash
# K3S_URL và K3S_TOKEN lấy từ server
curl -sfL https://get.k3s.io | K3S_URL=https://k3s-server:6443 \
  K3S_TOKEN=<NODE_TOKEN> \
  sh -

# Kiểm tra từ server
kubectl get nodes
```

### k3s HA với Embedded etcd

```bash
# Server đầu tiên — khởi tạo HA cluster
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --tls-san k3s-api.example.com    # DNS/IP cho API Server

# Server thứ 2 và 3 — join cluster
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://k3s-server-1:6443 \
  --token <CLUSTER_TOKEN> \
  --tls-san k3s-api.example.com
```

### k3s Air-gapped

```bash
# Tải binary và image về máy có Internet
wget https://github.com/k3s-io/k3s/releases/download/v1.30.0+k3s1/k3s
wget https://github.com/k3s-io/k3s/releases/download/v1.30.0+k3s1/k3s-airgap-images-amd64.tar

# Copy sang máy air-gapped
scp k3s user@air-gapped-server:~/
scp k3s-airgap-images-amd64.tar user@air-gapped-server:~/

# Trên máy air-gapped
sudo mkdir -p /var/lib/rancher/k3s/agent/images/
sudo cp k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
sudo chmod +x k3s
sudo mv k3s /usr/local/bin/

# Cài k3s offline
curl -sfL https://get.k3s.io > install.sh
INSTALL_K3S_SKIP_DOWNLOAD=true sh install.sh
```

### k3s vs minikube vs kind

| Tiêu Chí             | k3s                      | minikube                  | kind                      |
| -------------------- | ------------------------ | ------------------------- | ------------------------- |
| **Mục đích**         | Production edge, lab     | Dev local, học K8s        | CI/CD testing, dev        |
| **Multi-node**       | Có (production-like)     | Có (hạn chế)              | Có (docker containers)    |
| **Tài nguyên**       | Rất thấp (~512 MB RAM)   | Trung bình                | Thấp (Docker containers)  |
| **Tốc độ khởi động** | ~30 giây                 | 2–5 phút                  | ~1 phút                   |
| **Air-gapped**       | Hỗ trợ tốt               | Hạn chế                   | Hạn chế                   |

---

## RKE2 — Kubernetes Tập Trung Bảo Mật

**RKE2 (Rancher Kubernetes Engine 2 — Công Cụ Kubernetes Rancher 2)** là phân phối K8s của Rancher/SUSE, thiết kế từ đầu để đáp ứng tiêu chuẩn bảo mật cao: CIS Kubernetes Benchmark (CIS — Center for Internet Security — Trung Tâm Bảo Mật Internet), FIPS 140-2.

### Đặc Điểm Nổi Bật

```
✅ CIS Kubernetes Benchmark: hardened sẵn theo tiêu chuẩn CIS
✅ FIPS 140-2 compliant: phù hợp chính phủ, quốc phòng
✅ etcd tích hợp — không cần cài riêng
✅ Helm controller tích hợp — deploy workload qua HelmChart CRD
✅ NetworkPolicy bật sẵn với Canal (Calico + Flannel)
✅ Pod Security: RestrictedPod profile mặc định

Phù hợp:
  - Hệ thống yêu cầu compliance cao (FedRAMP, FISMA, PCI-DSS)
  - Chính phủ, tài chính, y tế
  - Bất kỳ nơi nào cần audit CIS Benchmark
```

### Cài RKE2 Server

```bash
# Tải và cài RKE2
curl -sfL https://get.rke2.io | sh -

# Tạo cấu hình
sudo mkdir -p /etc/rancher/rke2
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
tls-san:
  - rke2-api.example.com
  - 10.0.0.10
cni: calico                    # Calico thay thế Canal mặc định
disable:
  - rke2-ingress-nginx         # Tắt nếu muốn cài Ingress riêng
EOF

# Khởi động RKE2 server
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

# Lấy kubeconfig
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

# Lấy token để join agent
sudo cat /var/lib/rancher/rke2/server/node-token
```

### Cài RKE2 Agent (Worker Node)

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
server: https://rke2-api.example.com:9345
token: <NODE_TOKEN>
EOF

sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```

### RKE2 CIS Hardening

```bash
# Kiểm tra CIS Benchmark với kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```

---

## Kubespray — Cài Đặt Bằng Ansible

**Kubespray** là Ansible playbook để cài K8s trên nhiều node — phù hợp khi cần tự động hoá cài đặt cluster lớn.

### Khi Nào Dùng Kubespray

```
✅ Cần tự động hoá: cài K8s trên 10–100 server
✅ Đội đã quen Ansible
✅ Cần customization cao: chọn CNI, container runtime, etc.
✅ On-premise hoặc VM không phải cloud provider nào
```

### Cài Kubespray

```bash
# Clone repo
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

# Cài dependencies
pip3 install -r requirements.txt

# Tạo inventory từ template
cp -rfp inventory/sample inventory/my-cluster

# Cấu hình inventory
cat > inventory/my-cluster/hosts.yaml <<EOF
all:
  hosts:
    node1:
      ansible_host: 10.0.0.1
      ip: 10.0.0.1
    node2:
      ansible_host: 10.0.0.2
      ip: 10.0.0.2
    node3:
      ansible_host: 10.0.0.3
      ip: 10.0.0.3
  children:
    kube_control_plane:
      hosts:
        node1:
    kube_node:
      hosts:
        node2:
        node3:
    etcd:
      hosts:
        node1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
EOF

# Cấu hình cluster
cat > inventory/my-cluster/group_vars/k8s_cluster/k8s-cluster.yml <<EOF
kube_version: v1.30.0
kube_network_plugin: calico
container_manager: containerd
EOF

# Chạy playbook cài đặt
ansible-playbook -i inventory/my-cluster/hosts.yaml \
  --become \
  --become-user=root \
  cluster.yml
```

---

## etcd — Quản Lý Và Backup

**etcd** là distributed key-value store — lưu toàn bộ trạng thái cluster K8s. Mất etcd = mất cluster.

### Backup etcd

```bash
# Backup etcd snapshot (chạy trên etcd node)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Kiểm tra snapshot hợp lệ
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --write-out=table

# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | a0f...   | 12345    |       1234 |     5.5 MB |
# +----------+----------+------------+------------+
```

### Tự Động Backup Với CronJob

```bash
# Script backup chạy hàng ngày
cat > /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash
BACKUP_DIR=/backup/etcd
DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE=$BACKUP_DIR/etcd-snapshot-$DATE.db

mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl snapshot save $SNAPSHOT_FILE \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Giữ lại 7 ngày backup gần nhất
find $BACKUP_DIR -name "*.db" -mtime +7 -delete

echo "Backup completed: $SNAPSHOT_FILE"
EOF

chmod +x /usr/local/bin/etcd-backup.sh

# Cron job chạy 3h sáng mỗi ngày
echo "0 3 * * * root /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1" \
  >> /etc/cron.d/etcd-backup
```

### Restore etcd Từ Backup

```bash
# CẢNH BÁO: Quá trình restore phải dừng API Server trước

# Bước 1: Dừng Static Pod API Server
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Bước 2: Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-20260510.db \
  --data-dir=/var/lib/etcd-restored \
  --name=etcd-0 \
  --initial-cluster=etcd-0=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Bước 3: Replace thư mục etcd cũ
sudo mv /var/lib/etcd /var/lib/etcd-old
sudo mv /var/lib/etcd-restored /var/lib/etcd

# Bước 4: Khởi động lại etcd
sudo systemctl restart etcd

# Bước 5: Khởi động lại API Server
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Bước 6: Kiểm tra
kubectl get nodes
```

---

## Upgrade Self-Managed Cluster

### Nguyên Tắc Upgrade

```
1. Không được bỏ qua phiên bản — phải upgrade tuần tự (1.28 → 1.29 → 1.30)
2. Upgrade Control Plane TRƯỚC, Worker Node SAU
3. Backup etcd TRƯỚC khi upgrade
4. Test trên staging cluster trước production
5. Đọc release notes — kiểm tra deprecated API
6. Upgrade từng node — không upgrade tất cả cùng lúc
```

### Upgrade với kubeadm

```bash
# Bước 1: Backup etcd (bắt buộc)
/usr/local/bin/etcd-backup.sh

# Bước 2: Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.30.0-1.1
sudo apt-mark hold kubeadm

# Bước 3: Kiểm tra upgrade plan
sudo kubeadm upgrade plan

# Bước 4: Thực hiện upgrade Control Plane
sudo kubeadm upgrade apply v1.30.0

# Bước 5: Upgrade kubelet và kubectl trên Control Plane node
kubectl drain <control-plane-node> --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon <control-plane-node>

# Bước 6: Upgrade từng Worker Node
# (Thực hiện trên từng worker một)
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# Trên worker node:
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=1.30.0-1.1 kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
sudo apt-mark hold kubeadm kubelet kubectl
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Từ control plane:
kubectl uncordon <worker-node>
```

---

## Networking Self-Managed

### Chọn CNI Plugin

| Plugin       | Điểm Mạnh                                   | Khi Nào Dùng                       |
| ------------ | ------------------------------------------- | ---------------------------------- |
| **Calico**   | NetworkPolicy mạnh, BGP routing, hiệu năng  | Production, cần NetworkPolicy phức |
| **Flannel**  | Đơn giản, nhẹ, dễ cài                       | Lab, môi trường đơn giản           |
| **Cilium**   | eBPF-based, observability, L7 policy        | Hiệu năng cao, cần L7 visibility   |
| **Canal**    | Kết hợp Calico policy + Flannel networking  | Muốn NetworkPolicy mà vẫn đơn giản|
| **Weave**    | Mesh networking, mã hoá tự động             | Multi-cloud, mã hoá traffic Pod    |

### Cài Calico

```bash
# Cài Calico Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# Cài Calico với cấu hình
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 192.168.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
EOF
```

### MetalLB — Load Balancer Cho On-Premise

Khi không có cloud provider, Service type LoadBalancer không hoạt động. **MetalLB** cung cấp LoadBalancer implementation cho bare-metal.

```bash
# Cài MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

# Cấu hình IP pool cho LoadBalancer (IP range phải trong LAN của bạn)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.250    # Dải IP dành riêng cho LoadBalancer
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - production-pool
EOF

# Giờ Service type: LoadBalancer sẽ nhận External IP từ pool trên
```

---

## Storage Self-Managed

### Longhorn — Distributed Block Storage

**Longhorn** (của Rancher) là distributed persistent storage cho K8s — dễ cài, có UI, hỗ trợ snapshot và backup.

```bash
# Yêu cầu: open-iscsi trên tất cả node
sudo apt-get install -y open-iscsi

# Cài Longhorn
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.0/deploy/longhorn.yaml

# Truy cập Longhorn UI
kubectl port-forward svc/longhorn-frontend -n longhorn-system 8080:80
```

```yaml
# StorageClass dùng Longhorn
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"           # 3 bản sao dữ liệu
  staleReplicaTimeout: "2880"
  fromBackup: ""
  diskSelector: ""
  nodeSelector: ""
allowVolumeExpansion: true
reclaimPolicy: Retain
```

### Rook-Ceph — Enterprise Storage

**Rook-Ceph** cung cấp block, file, và object storage trên K8s — phù hợp production lớn.

```bash
# Cài Rook Operator
kubectl create -f https://raw.githubusercontent.com/rook/rook/v1.14.0/deploy/examples/crds.yaml
kubectl create -f https://raw.githubusercontent.com/rook/rook/v1.14.0/deploy/examples/common.yaml
kubectl create -f https://raw.githubusercontent.com/rook/rook/v1.14.0/deploy/examples/operator.yaml

# Tạo Ceph Cluster
kubectl create -f https://raw.githubusercontent.com/rook/rook/v1.14.0/deploy/examples/cluster.yaml
```

### NFS — Storage Đơn Giản Nhất

```bash
# Cài NFS server (trên node storage)
sudo apt-get install -y nfs-kernel-server
echo "/data/nfs *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
sudo exportfs -ra

# Cài NFS CSI Driver trên K8s
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=10.0.0.5 \
  --set nfs.path=/data/nfs
```

---

## High Availability (HA) — Tính Sẵn Sàng Cao

### Kiến Trúc HA Control Plane

```
                    ┌─────────────────────┐
                    │  Load Balancer      │
                    │  (HAProxy / keepalived)│
                    │  VIP: 10.0.0.100    │
                    └──────────┬──────────┘
                               │ :6443
           ┌───────────────────┼───────────────────┐
           ↓                   ↓                   ↓
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Control Plane 1  │  │ Control Plane 2  │  │ Control Plane 3  │
│ 10.0.0.1         │  │ 10.0.0.2         │  │ 10.0.0.3         │
│ API Server       │  │ API Server       │  │ API Server       │
│ etcd member      │  │ etcd member      │  │ etcd member      │
└──────────────────┘  └──────────────────┘  └──────────────────┘
           │                   │                   │
           └───────────────────┴───────────────────┘
                      etcd cluster (3 member)
```

### HAProxy + keepalived Cho VIP

```
# Cài trên tất cả Control Plane node
sudo apt-get install -y haproxy keepalived
```

```
# /etc/haproxy/haproxy.cfg
frontend kubernetes-apiserver
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-apiserver

backend kubernetes-apiserver
    mode tcp
    option tcp-check
    balance roundrobin
    server cp1 10.0.0.1:6443 check
    server cp2 10.0.0.2:6443 check
    server cp3 10.0.0.3:6443 check
```

```
# /etc/keepalived/keepalived.conf (trên primary node)
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        10.0.0.100/24              # VIP — Virtual IP — IP ảo
    }
}
```

### Kiểm Tra HA

```bash
# Tắt 1 Control Plane node — cluster vẫn phải hoạt động
ssh cp1 "sudo systemctl stop kube-apiserver"

# Kiểm tra từ client
kubectl get nodes              # Phải vẫn hoạt động qua VIP → cp2 hoặc cp3
```

---

## Monitoring và Logging

### Prometheus Stack (kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=admin123
```

### Logging Stack (Loki + Promtail)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki-stack grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=longhorn \
  --set loki.persistence.size=20Gi \
  --set promtail.enabled=true
```

---

## Bảo Mật Self-Managed

### Certificate Management — Quản Lý Chứng Chỉ

Certificate K8s mặc định expire sau **1 năm** — đây là nguồn gốc của nhiều sự cố production.

```bash
# Kiểm tra tất cả certificate sắp hết hạn
kubeadm certs check-expiration

# Output ví dụ:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 May 10, 2027 10:00 UTC   364d
# apiserver                  May 10, 2027 10:00 UTC   364d
# etcd-ca                    May 08, 2036 10:00 UTC   9y

# Renew tất cả certificate (renew trước khi expire 30 ngày)
sudo kubeadm certs renew all

# Restart Control Plane sau khi renew
sudo kill -s SIGHUP $(pidof kube-apiserver)
sudo kill -s SIGHUP $(pidof kube-controller-manager)
sudo kill -s SIGHUP $(pidof kube-scheduler)
```

### CIS Benchmark Check

```bash
# Chạy kube-bench kiểm tra CIS Benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# Hoặc chạy trực tiếp trên node
sudo docker run --rm --pid=host \
  -v /etc:/etc:ro \
  -v /var/lib:/var/lib:ro \
  -t docker.io/aquasec/kube-bench:latest \
  --version 1.30
```

### Pod Security Admission

```yaml
# Namespace production: chỉ cho phép Restricted pod
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khi nào bạn chọn self-managed thay vì managed Kubernetes?**

> Tôi chọn self-managed khi: (1) On-premise hoặc bare-metal không có cloud provider. (2) Môi trường air-gapped không có Internet — không thể dùng EKS/GKE/AKS. (3) Yêu cầu compliance đặc thù như FIPS 140-2 hay FedRAMP (RKE2 phù hợp). (4) Team đủ năng lực và muốn kiểm soát toàn bộ stack. Với dự án mới có đủ ngân sách, tôi thường khuyến nghị managed service vì lợi ích về SLA và giảm overhead vận hành lớn hơn chi phí bỏ ra.

**Q: etcd quan trọng thế nào? Backup và restore thế nào?**

> etcd là "bộ não" của Kubernetes — lưu toàn bộ trạng thái cluster: Pod, Deployment, ConfigMap, Secret, v.v. Mất etcd nghĩa là mất toàn bộ cluster state — không thể recover workload. Backup bằng `etcdctl snapshot save` ít nhất 1 lần/ngày, lưu nhiều nơi (local + remote S3 hoặc NFS). Restore cần dừng API Server trước, restore data, restart etcd và API Server. Kiểm tra backup hợp lệ bằng `etcdctl snapshot status`.

**Q: Sự khác biệt giữa kubeadm, k3s và RKE2?**

> kubeadm là tool chính thức từ K8s upstream — phù hợp học sâu và production on-premise chuẩn, nhưng cài đặt nhiều bước thủ công. k3s là K8s nhẹ của Rancher — cài 1 lệnh, thay thế etcd bằng SQLite, bỏ code thừa, phù hợp edge, IoT, Raspberry Pi. RKE2 cũng từ Rancher nhưng focus bảo mật — tích hợp sẵn CIS Benchmark, FIPS 140-2, phù hợp môi trường yêu cầu compliance cao. Ba phân phối đều chạy K8s tương thích upstream — khác biệt chủ yếu ở mục tiêu và tính năng đặc thù.

**Q: Làm thế nào để cấp LoadBalancer trên bare-metal không có cloud provider?**

> Dùng MetalLB — nó implement LoadBalancer spec của K8s cho bare-metal. MetalLB có hai mode: L2 mode (dùng ARP để announce IP — đơn giản, không cần cấu hình router) và BGP mode (peering với router để announce prefix — hiệu năng cao hơn, phù hợp datacenter). Cấu hình IP pool trong dải IP LAN của bạn, MetalLB sẽ gán IP từ pool đó cho Service type LoadBalancer, các client trong LAN có thể truy cập trực tiếp.

**Q: Certificate K8s expire thì điều gì xảy ra? Xử lý thế nào?**

> Khi certificate hết hạn, API Server từ chối tất cả request — cluster sẽ không thể quản lý được, `kubectl` trả lỗi TLS. Xử lý: (1) SSH trực tiếp vào Control Plane node. (2) Chạy `kubeadm certs renew all`. (3) Restart các component: kube-apiserver, kube-controller-manager, kube-scheduler. (4) Cập nhật kubeconfig: `kubeadm init phase kubeconfig admin`. Phòng ngừa: dùng cron job chạy `kubeadm certs check-expiration` hàng tháng, alert khi certificate sắp hết hạn trong 30 ngày, rotate mỗi năm trước khi expire.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
