# Placement Groups — Nhóm Vị Trí Vật Lý

> **Placement Groups** cho phép kiểm soát vị trí vật lý của EC2 instances trong data center AWS, giúp tối ưu hiệu năng mạng hoặc tăng khả năng chịu lỗi phần cứng.

## 📚 Mục Lục

1. [Tại Sao Cần Placement Groups?](#tại-sao-cần-placement-groups)
2. [Cluster Placement Group](#cluster-placement-group)
3. [Spread Placement Group](#spread-placement-group)
4. [Partition Placement Group](#partition-placement-group)
5. [So Sánh Ba Loại](#so-sánh-ba-loại)
6. [Giới Hạn và Lưu Ý](#giới-hạn-và-lưu-ý)
7. [Thực Hành CLI](#thực-hành-cli)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Placement Groups?

AWS data center có kiến trúc nhiều lớp: Region → AZ → Data Center → Rack → Host. Mặc định, instances được phân bổ ngẫu nhiên trên các host để cân bằng tải phần cứng. **Placement Group** cho phép override hành vi này:

```
Mặc định (không có Placement Group):
  Instance A → Rack 15, Host 3
  Instance B → Rack 47, Host 8
  Instance C → Rack 2,  Host 12
  
  Ưu điểm: Ít bị ảnh hưởng nếu một host/rack lỗi
  Nhược điểm: Network latency cao hơn giữa instances

Với Placement Group → kiểm soát chính xác hơn (xem từng loại bên dưới)
```

---

## Cluster Placement Group

### Mô Hình

```
┌─────────────────────────────────────────┐
│         Cluster Placement Group         │
│         (Cùng một AZ, gần nhau)        │
│                                         │
│  ┌──── Rack ────────────────────────┐   │
│  │                                  │   │
│  │  [Instance A] ←──10Gbps──► [Instance B] │
│  │       ↕                    ↕     │   │
│  │  [Instance C] ←──10Gbps──► [Instance D] │
│  │                                  │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Đặc Điểm

```
✅ Instances đặt gần nhau về mặt vật lý (cùng rack hoặc racks liền kề)
✅ Network bandwidth cao nhất: 10 Gbps giữa instances (thay vì 5 Gbps mặc định)
✅ Latency cực thấp: single-digit microsecond
✅ Enhanced Networking bắt buộc để đạt hiệu năng tối đa
✅ Hỗ trợ: HPC (High-Performance Computing), ML training, Big Data tính toán nhanh
⚠️ Chỉ trong một AZ duy nhất — không span multiple AZ
⚠️ Nếu rack/host lỗi, nhiều instances bị ảnh hưởng (single point of failure)
⚠️ Capacity có thể hạn chế — đôi khi không thể launch thêm instance mới
```

### Khi Nào Dùng Cluster?

```
Use cases:
  ✅ HPC — High-Performance Computing (Tính Toán Hiệu Năng Cao)
     - CFD (Computational Fluid Dynamics — Mô Phỏng Động Học Chất Lỏng)
     - Molecular dynamics simulation
     
  ✅ Machine Learning / Deep Learning Training
     - Distributed training với GPU instances (p3, p4d)
     - All-reduce operations cần băng thông cao
     
  ✅ Big Data Processing
     - Apache Spark jobs cần shuffle nhiều data
     - Hadoop HDFS replication trong cluster
     
  ✅ Low-Latency Applications
     - Financial trading systems
     - Real-time game servers với synchronous state

Không dùng Cluster khi:
  ❌ Cần high availability (toàn bộ cluster có thể bị ảnh hưởng bởi một lỗi)
  ❌ Instances cần chạy ở nhiều AZ
  ❌ Workload không nhạy cảm với network latency
```

---

## Spread Placement Group

### Mô Hình

```
┌──── AZ-1a ────────────────────────────────────────────┐
│                                                         │
│  Rack 1      Rack 2      Rack 3      Rack 4      Rack 5 │
│  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐ │
│  │[Inst]│   │[Inst]│   │[Inst]│   │[Inst]│   │[Inst]│ │
│  │  A   │   │  B   │   │  C   │   │  D   │   │  E   │ │
│  └──────┘   └──────┘   └──────┘   └──────┘   └──────┘ │
│                                                         │
│  ← Mỗi instance trên một rack vật lý riêng biệt →     │
└─────────────────────────────────────────────────────────┘
```

### Đặc Điểm

```
✅ Mỗi instance trên một rack phần cứng riêng biệt
✅ Rack riêng có nguồn điện và switch mạng riêng
✅ Tối đa 7 instances mỗi AZ (giới hạn quan trọng!)
✅ Có thể span nhiều AZ trong cùng region
✅ Isolation tốt nhất — hardware failure chỉ ảnh hưởng 1 instance
⚠️ Giới hạn 7 instances/AZ — không phù hợp cho large cluster
⚠️ Latency cao hơn Cluster (vì instances ở rack xa nhau)
```

### Khi Nào Dùng Spread?

```
Use cases:
  ✅ Số ít instances quan trọng nhưng cần HA tối đa
     - 3 ZooKeeper nodes (quorum cluster)
     - 3 Primary/Replica database nodes
     - 5 Consul/etcd cluster nodes
     
  ✅ Critical instances không được chết cùng nhau
     - Domain controllers
     - License servers
     - Message broker primaries (ActiveMQ, RabbitMQ)
     
  ✅ Compliance requirement
     - "No two instances trên cùng physical hardware"

Không dùng Spread khi:
  ❌ Cần > 7 instances/AZ
  ❌ Large-scale distributed systems (dùng Partition thay thế)
  ❌ Cần bandwidth cao giữa instances (dùng Cluster)
```

---

## Partition Placement Group

### Mô Hình

```
┌──── AZ-1a ────────────────────────────────────────────────┐
│                                                             │
│  Partition 1        Partition 2        Partition 3          │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│  │  Rack A      │   │  Rack C      │   │  Rack E      │    │
│  │  [Node 1]    │   │  [Node 4]    │   │  [Node 7]    │    │
│  │  [Node 2]    │   │  [Node 5]    │   │  [Node 8]    │    │
│  │  Rack B      │   │  Rack D      │   │  Rack F      │    │
│  │  [Node 3]    │   │  [Node 6]    │   │  [Node 9]    │    │
│  └──────────────┘   └──────────────┘   └──────────────┘    │
│                                                             │
│  ← Partition 1,2,3 KHÔNG CHIA SẺ phần cứng với nhau →    │
└─────────────────────────────────────────────────────────────┘
```

### Đặc Điểm

```
✅ Tối đa 7 partitions mỗi AZ
✅ Mỗi partition là tập hợp racks riêng biệt
✅ Instances trong cùng partition gần nhau (network tốt)
✅ Partitions khác nhau không share rack/hardware
✅ Không giới hạn số instances (hàng trăm instances/partition)
✅ Metadata về partition ID có trong IMDS
✅ Có thể span nhiều AZ
⚠️ Không đảm bảo từng instance trên rack riêng (khác Spread)
```

### Partition Metadata

```bash
# Từ bên trong instance, xem partition ID
curl -s http://169.254.169.254/latest/meta-data/placement/partition-number
# → 1 (hoặc 2, 3... tùy partition)

# Ứng dụng có thể dùng partition ID để đưa ra quyết định
# Ví dụ: Kafka broker biết mình ở partition nào để tránh replicate với broker cùng partition
```

### Khi Nào Dùng Partition?

```
Use cases:
  ✅ Large distributed systems cần fault isolation
     - Apache Kafka (hàng chục broker nodes)
     - Apache Cassandra (hàng chục nodes)
     - Apache HBase, HDFS NameNode/DataNode
     - Elasticsearch large clusters
     
  ✅ Big Data frameworks
     - Hadoop với nhiều DataNode
     - Spark cluster lớn
     
Logic: Nếu Partition 1 lỗi, Partition 2 và 3 vẫn hoạt động
       → Kafka có thể serve từ replicas ở partition khác

Không dùng Partition khi:
  ❌ Nhỏ, < 10 instances (dùng Spread)
  ❌ Cần bandwidth cực cao giữa mọi node (dùng Cluster)
```

---

## So Sánh Ba Loại

```
┌────────────────┬───────────────┬───────────────┬───────────────┐
│ Tiêu Chí       │ Cluster       │ Spread        │ Partition     │
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Vị trí         │ Gần nhau      │ Mỗi instance  │ Nhóm instances│
│                │ (cùng rack    │ trên rack riêng│ trên racks    │
│                │ hoặc gần)     │               │ riêng/partition│
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Multi-AZ       │ ❌ Không      │ ✅ Có         │ ✅ Có         │
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Giới hạn       │ Không cố định │ 7 instances   │ 7 partitions  │
│ instances      │ (hàng chục)   │ mỗi AZ        │ mỗi AZ, mỗi  │
│                │               │               │ partition vô hạn│
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Network        │ Tốt nhất      │ Trung bình    │ Tốt (trong    │
│ performance    │ (10Gbps+)     │               │ partition)    │
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Fault          │ Thấp (cùng   │ Cao nhất      │ Trung bình-Cao│
│ isolation      │ rack)         │ (1 host/rack  │ (partition     │
│                │               │ riêng mỗi)    │ riêng nhau)   │
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Use case       │ HPC, ML train │ Critical small │ Large distrib.│
│                │ Big Data spark│ clusters, HA  │ systems       │
├────────────────┼───────────────┼───────────────┼───────────────┤
│ Ví dụ          │ GPU training  │ 3-node ZK,    │ 30-node Kafka │
│                │ cluster       │ 5-node etcd   │ 100-node HDFS │
└────────────────┴───────────────┴───────────────┴───────────────┘
```

### Bảng Quyết Định Nhanh

```
Câu hỏi đặt ra:
  "Workload có cần network bandwidth cực cao và latency thấp?"
  → Có → CLUSTER
  
  "Workload có ít instances quan trọng cần isolation tối đa?"
  → Có, ≤ 7 instances/AZ → SPREAD
  
  "Workload là distributed system lớn với nhiều nodes?"
  → Có → PARTITION
  
  "Không có yêu cầu đặc biệt gì?"
  → Không cần Placement Group
```

---

## Giới Hạn và Lưu Ý

### Giới Hạn Kỹ Thuật

```
Cluster Placement Group:
  - Chỉ một AZ
  - Khuyến nghị dùng cùng instance type cho toàn cluster
  - "Insufficient capacity error" phổ biến khi thêm instances sau
    → Giải pháp: Launch tất cả instances cùng lúc

Spread Placement Group:
  - Tối đa 7 instances mỗi AZ mỗi Placement Group
  - Dedicated Instances không hỗ trợ

Partition Placement Group:
  - Tối đa 7 partitions mỗi AZ
  - Dedicated Host không hỗ trợ
```

### Capacity Reservation và Placement Groups

```bash
# Tạo Capacity Reservation trong Placement Group
# Đảm bảo có capacity sẵn sàng khi cần

aws ec2 create-capacity-reservation \
  --instance-type c5n.18xlarge \
  --instance-platform Linux/UNIX \
  --availability-zone us-east-1a \
  --instance-count 10 \
  --placement-group-arn arn:aws:ec2:us-east-1:123456789:placement-group/my-hpc-cluster
```

### Không Thể Merge Placement Groups

```
Nếu đã có hai placement groups riêng biệt → không thể merge lại.
Phải terminate instances và launch lại trong một placement group.
```

---

## Thực Hành CLI

### Tạo Placement Groups

```bash
# Tạo Cluster Placement Group
aws ec2 create-placement-group \
  --group-name "hpc-cluster-pg" \
  --strategy cluster

# Tạo Spread Placement Group
aws ec2 create-placement-group \
  --group-name "critical-nodes-spread-pg" \
  --strategy spread

# Tạo Partition Placement Group với 3 partitions
aws ec2 create-placement-group \
  --group-name "kafka-cluster-partition-pg" \
  --strategy partition \
  --partition-count 3

# Liệt kê placement groups
aws ec2 describe-placement-groups
```

### Launch Instances vào Placement Group

```bash
# Launch vào Cluster Placement Group
aws ec2 run-instances \
  --image-id ami-0abc123 \
  --instance-type c5n.18xlarge \
  --count 10 \
  --placement "GroupName=hpc-cluster-pg" \
  --key-name my-key

# Launch vào Partition 2 của Partition Group
aws ec2 run-instances \
  --image-id ami-0abc123 \
  --instance-type m5.xlarge \
  --count 5 \
  --placement "GroupName=kafka-cluster-partition-pg,PartitionNumber=2" \
  --key-name my-key

# Launch vào Spread Placement Group
aws ec2 run-instances \
  --image-id ami-0abc123 \
  --instance-type m5.large \
  --count 3 \
  --placement "GroupName=critical-nodes-spread-pg" \
  --key-name my-key
```

### Thêm Instance Vào Placement Group Đang Tồn Tại

```bash
# Instance phải ở trạng thái stopped
aws ec2 stop-instances --instance-ids i-0abc123

# Modify placement (chỉ Cluster và Partition hỗ trợ modify)
aws ec2 modify-instance-placement \
  --instance-id i-0abc123 \
  --group-name "hpc-cluster-pg"

aws ec2 start-instances --instance-ids i-0abc123
```

### Xóa Placement Group

```bash
# Phải terminate tất cả instances trước
aws ec2 terminate-instances --instance-ids i-0abc123 i-0def456

# Sau đó xóa placement group
aws ec2 delete-placement-group --group-name "hpc-cluster-pg"
```

---

## Câu Hỏi Phỏng Vấn

**Q: Bạn đang setup Kafka cluster 30 nodes. Loại Placement Group nào phù hợp?**
> **Partition Placement Group** với 3 partitions, mỗi partition 10 nodes. Kafka natively hỗ trợ rack-awareness — có thể cấu hình để replicate partition sang broker ở các Placement Group partition khác nhau. Khi một partition lỗi (rack lỗi), các partitions còn lại vẫn hoạt động. Spread không phù hợp vì giới hạn 7 instances/AZ, Cluster không phù hợp vì không có fault isolation tốt.

**Q: Cluster Placement Group có thể span nhiều AZ không?**
> Không. Cluster Placement Group chỉ ở trong một AZ duy nhất. Đây là trade-off cơ bản: để đạt được network bandwidth cực cao và latency thấp, các instances phải ở gần nhau về vật lý — điều này không thể thực hiện cross-AZ. Nếu AZ đó có sự cố, toàn bộ cluster bị ảnh hưởng.

**Q: "Insufficient capacity error" khi thêm instance vào Cluster Placement Group là gì? Cách giải quyết?**
> Lỗi này xảy ra khi AWS không đủ capacity trên hardware gần cluster của bạn. Nguyên nhân: instances trong Cluster PG nằm trên hardware gần nhau, và phần cứng đó đã đầy. Giải pháp: (1) Terminate toàn bộ instances và launch lại cùng lúc với số lượng mong muốn — AWS sẽ cấp phát hardware đủ lớn. (2) Dùng Capacity Reservation trước khi launch. (3) Dùng instance type khác nếu capacity hiện tại cạn kiệt.

**Q: Tại sao Spread Placement Group chỉ cho 7 instances mỗi AZ?**
> Vì AWS đảm bảo mỗi instance ở trên rack vật lý riêng biệt, và AZ thường được tổ chức thành 7 "isolation zones" với hardware độc lập (nguồn điện, network riêng). Giới hạn 7 này phản ánh số "independent hardware fault domains" AWS đảm bảo trong một AZ. Nếu cần nhiều hơn 7 instances với fault isolation → dùng Partition Placement Group.

---

## Liên Kết

- [← User Data & Metadata](./4-user-data-metadata.md)
- [← README.md — EC2 Fundamentals Overview](./README.md)
- [Tiếp theo: Auto Scaling →](../02-auto-scaling/README.md)
- [AWS Placement Groups Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)

---

**Cập Nhật:** 2026-05-14 | **Trạng Thái:** ✅ Hoàn thành
