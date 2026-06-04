# Multi-AZ Design — Thiết Kế Đa Vùng Khả Dụng

> Multi-AZ (Multi Availability Zone — Đa Vùng Khả Dụng) là nền tảng của mọi kiến trúc High Availability trên AWS. Thay vì đặt tất cả tài nguyên trong một data center, bạn phân tán chúng qua nhiều AZ độc lập để chống SPOF (Single Point of Failure — Điểm Lỗi Duy Nhất).

## 📚 Mục Lục

1. [AZ & Region — Nền Tảng Vật Lý](#az--region)
2. [Multi-AZ cho EC2 + ELB](#multi-az-ec2-elb)
3. [Multi-AZ cho ECS & EKS](#multi-az-ecs-eks)
4. [Multi-AZ cho Database & Cache](#multi-az-database)
5. [Multi-Region Patterns](#multi-region-patterns)
6. [Failover với Route 53](#failover-route-53)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏢 AZ & Region — Nền Tảng Vật Lý {#az--region}

### Cấu Trúc Vật Lý AWS

```
AWS Global Infrastructure
├── Region: ap-southeast-1 (Singapore)
│   ├── AZ: ap-southeast-1a  ─── Data center cluster A
│   ├── AZ: ap-southeast-1b  ─── Data center cluster B
│   └── AZ: ap-southeast-1c  ─── Data center cluster C
│
├── Region: us-east-1 (N. Virginia)
│   ├── AZ: us-east-1a
│   ├── AZ: us-east-1b
│   ├── AZ: us-east-1c
│   ├── AZ: us-east-1d
│   ├── AZ: us-east-1e
│   └── AZ: us-east-1f
│
└── Edge Locations (CloudFront, Route 53)
    ├── Ho Chi Minh City
    ├── Hanoi
    └── 400+ locations globally
```

### Đặc Điểm AZ

| Thuộc Tính              | Chi Tiết                                                    |
| ----------------------- | ----------------------------------------------------------- |
| Khoảng cách vật lý      | ≥100km giữa các AZ trong cùng Region                       |
| Độ trễ giữa các AZ      | <2ms (thường 0.5–1ms)                                      |
| Kết nối                 | Cáp quang riêng, băng thông cao, mã hóa                    |
| Nguồn điện              | Độc lập, nguồn dự phòng UPS + generator riêng              |
| Kết nối internet        | Mỗi AZ có ISP (Internet Service Provider) riêng            |
| Tên AZ                  | Tên AZ (1a, 1b...) ánh xạ đến data center vật lý khác nhau theo account |

> **Lưu ý quan trọng:** `us-east-1a` trong account của bạn có thể là data center khác với `us-east-1a` trong account của người khác. AWS ngẫu nhiên hóa tên để phân phối tải đều.

### Khi Nào Một AZ Bị Ảnh Hưởng?

- Sự cố nguồn điện quy mô lớn
- Thiên tai (lũ lụt, động đất)
- Sự cố mạng của ISP cấp khu vực
- Lỗi phần cứng quy mô data center
- Lỗi AWS managed service tại AZ đó

---

## ⚖️ Multi-AZ cho EC2 + ELB {#multi-az-ec2-elb}

### Kiến Trúc Cơ Bản

```
                     Internet
                         │
              ┌──────────▼──────────┐
              │  Application Load   │
              │  Balancer (ALB)     │
              │  (span 3 AZ)        │
              └────┬──────────┬─────┘
                   │          │
        ┌──────────▼──┐  ┌────▼──────────┐
        │    AZ-1a    │  │    AZ-1b      │
        │  ┌────────┐ │  │  ┌────────┐  │
        │  │ EC2 #1 │ │  │  │ EC2 #3 │  │
        │  └────────┘ │  │  └────────┘  │
        │  ┌────────┐ │  │  ┌────────┐  │
        │  │ EC2 #2 │ │  │  │ EC2 #4 │  │
        │  └────────┘ │  │  └────────┘  │
        └─────────────┘  └─────────────┘
              ASG min=2, max=8, desired=4
```

### Auto Scaling Group Multi-AZ

```bash
# Tạo ASG span nhiều AZ
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-ha-asg \
  --launch-template LaunchTemplateId=lt-1234567890abcdef0 \
  --min-size 2 \
  --max-size 8 \
  --desired-capacity 4 \
  --vpc-zone-identifier "subnet-aaa111,subnet-bbb222,subnet-ccc333" \
  # Mỗi subnet thuộc một AZ khác nhau
  --health-check-type ELB \
  --health-check-grace-period 300
```

**Quy tắc thiết kế ASG Multi-AZ:**

| Quy Tắc                       | Lý Do                                                              |
| ----------------------------- | ------------------------------------------------------------------ |
| `min-size ≥ số AZ`            | Đảm bảo luôn có instance ở mỗi AZ                                 |
| `desired` chia đều cho số AZ  | Phân phối đều tải                                                  |
| Subnet mỗi AZ khác nhau       | ASG tự phân phối instances theo AZ                                 |
| `health-check-type = ELB`     | ASG dùng ELB health check để thay thế instance lỗi                |

### Cross-Zone Load Balancing (Cân Bằng Tải Liên Vùng)

```
Không có Cross-Zone LB:            Có Cross-Zone LB:
                                   
ALB Node AZ-1a ──→ EC2 AZ-1a      ALB Node AZ-1a ──→ EC2 AZ-1a (50%)
                                                   ──→ EC2 AZ-1b (50%)
ALB Node AZ-1b ──→ EC2 AZ-1b      ALB Node AZ-1b ──→ EC2 AZ-1a (50%)
                                                   ──→ EC2 AZ-1b (50%)

Vấn đề: Nếu AZ-1a có 2 instances     Kết quả: Tải phân phối đều bất kể
và AZ-1b có 8 instances → mất cân    số instances mỗi AZ
bằng tải
```

**ALB** bật Cross-Zone LB mặc định (miễn phí).
**NLB** (Network Load Balancer — Cân Bằng Tải Mạng) và **CLB** tính phí cross-AZ data transfer.

### Subnet Design cho Multi-AZ

```
VPC: 10.0.0.0/16
├── Public Subnets (cho ALB, NAT Gateway)
│   ├── AZ-1a: 10.0.1.0/24  (subnet-pub-1a)
│   ├── AZ-1b: 10.0.2.0/24  (subnet-pub-1b)
│   └── AZ-1c: 10.0.3.0/24  (subnet-pub-1c)
│
├── Private Subnets (cho EC2, ECS, EKS)
│   ├── AZ-1a: 10.0.10.0/24 (subnet-priv-1a)
│   ├── AZ-1b: 10.0.20.0/24 (subnet-priv-1b)
│   └── AZ-1c: 10.0.30.0/24 (subnet-priv-1c)
│
└── Database Subnets (cho RDS, ElastiCache)
    ├── AZ-1a: 10.0.100.0/24 (subnet-db-1a)
    ├── AZ-1b: 10.0.101.0/24 (subnet-db-1b)
    └── AZ-1c: 10.0.102.0/24 (subnet-db-1c)
```

---

## 🐳 Multi-AZ cho ECS & EKS {#multi-az-ecs-eks}

### ECS Service Multi-AZ

ECS tự động phân phối task qua nhiều AZ khi cấu hình đúng:

```json
{
  "serviceName": "my-ha-service",
  "taskDefinition": "my-app:5",
  "desiredCount": 6,
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": [
        "subnet-priv-1a",
        "subnet-priv-1b",
        "subnet-priv-1c"
      ],
      "securityGroups": ["sg-app"],
      "assignPublicIp": "DISABLED"
    }
  },
  "placementStrategies": [
    {
      "type": "spread",
      "field": "attribute:ecs.availability-zone"
    },
    {
      "type": "spread",
      "field": "instanceId"
    }
  ],
  "loadBalancers": [...]
}
```

**Placement Strategy (Chiến Lược Vị Trí):**

| Strategy   | Field                             | Kết Quả                                       |
| ---------- | --------------------------------- | --------------------------------------------- |
| `spread`   | `attribute:ecs.availability-zone` | Phân phối đều qua AZ (HA cơ bản)             |
| `spread`   | `instanceId`                      | Phân phối qua nhiều instance trong AZ         |
| `binpack`  | `cpu` / `memory`                  | Tận dụng tối đa tài nguyên (tiết kiệm chi phí)|
| `random`   | —                                 | Ngẫu nhiên, không khuyến khích cho production |

### EKS Node Group Multi-AZ

```yaml
# Managed Node Group trải qua nhiều AZ
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: ap-southeast-1

managedNodeGroups:
  - name: ng-ha
    instanceType: m5.large
    minSize: 3
    maxSize: 9
    desiredCapacity: 6
    availabilityZones:
      - ap-southeast-1a
      - ap-southeast-1b
      - ap-southeast-1c
```

**Pod Anti-Affinity (Quy Tắc Chống Gần Cận Pod) để phân tán pods:**

```yaml
# Đảm bảo pods không nằm cùng AZ
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - my-app
        topologyKey: topology.kubernetes.io/zone
```

**PodDisruptionBudget (Ngân Sách Gián Đoạn Pod — PDB):**

```yaml
# Đảm bảo luôn có ít nhất 2 pods healthy khi drain node
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2          # Hoặc dùng maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

---

## 🗄️ Multi-AZ cho Database & Cache {#multi-az-database}

### RDS Multi-AZ

```
Primary DB (AZ-1a)
        │
        │ Synchronous replication
        │ (đồng bộ liên tục, mọi write đều được confirm)
        ▼
Standby DB (AZ-1b)  ←── Không phục vụ read traffic (chỉ standby)

Khi Primary lỗi:
1. AWS phát hiện lỗi qua health check (~1-2 phút)
2. DNS endpoint tự cập nhật trỏ sang Standby
3. Standby promote thành Primary mới
4. Tổng downtime: 60-120 giây (RTO ~2 phút)
```

```bash
# Bật Multi-AZ cho RDS
aws rds create-db-instance \
  --db-instance-identifier my-ha-db \
  --db-instance-class db.r6g.large \
  --engine postgres \
  --multi-az           # Bật Multi-AZ standby
  --allocated-storage 100
```

**Lưu ý:** RDS Multi-AZ Standby không dùng để đọc. Nếu muốn đọc từ replica → dùng **RDS Read Replica** (bất đồng bộ).

### ElastiCache Multi-AZ

**Redis Cluster Mode Enabled (Chế Độ Cluster Redis):**

```
Shard 1 (Primary AZ-1a) ──→ Replica (AZ-1b)
Shard 2 (Primary AZ-1b) ──→ Replica (AZ-1c)
Shard 3 (Primary AZ-1c) ──→ Replica (AZ-1a)

Nếu Primary Shard 1 lỗi:
→ Replica ở AZ-1b tự động promoted
→ Downtime ~60 giây (failover tự động)
```

---

## 🌍 Multi-Region Patterns {#multi-region-patterns}

### Active-Passive (Chủ Động - Thụ Động)

```
Region Chính (ap-southeast-1 / Singapore)
├── Toàn bộ traffic sản xuất
├── RDS Primary
└── EC2 / ECS production workload

                ↕ Replication (sao chép dữ liệu)

Region Dự Phòng (ap-east-1 / Hong Kong)
├── Chờ standby, không nhận traffic
├── RDS Read Replica (sẵn sàng promote)
└── EC2 AMI / ECS images sẵn có

Khi Region Chính sập:
1. Chuyển DNS (Route 53 failover)
2. Promote RDS Read Replica thành Primary
3. Scale up resources ở Region Dự Phòng
4. RTO: 15-60 phút | RPO: <5 phút (tùy replication lag)
```

### Active-Active (Chủ Động - Chủ Động)

```
Region 1 (ap-southeast-1)          Region 2 (us-west-2)
├── 50% traffic                     ├── 50% traffic
├── RDS Primary                     ├── DynamoDB Global Tables
└── EC2 / ECS                       └── EC2 / ECS

Route 53 Latency Routing:
- User ở Asia → ap-southeast-1
- User ở US → us-west-2
- Nếu một region sập → 100% traffic sang region còn lại

RTO: <60 giây | RPO: ~0 (DynamoDB Global Tables đồng bộ real-time)
```

**Phù Hợp Cho Active-Active:**
- DynamoDB Global Tables (Bảng Toàn Cầu DynamoDB)
- S3 Cross-Region Replication
- Stateless applications (ứng dụng không lưu trạng thái)
- Lambda (triển khai riêng mỗi region)

---

## 🗺️ Failover với Route 53 {#failover-route-53}

### Routing Policies (Chính Sách Định Tuyến)

| Policy               | Khi Dùng                                                    |
| -------------------- | ----------------------------------------------------------- |
| **Failover**         | Active-Passive: tự chuyển sang secondary khi primary lỗi   |
| **Latency**          | Active-Active: định tuyến đến region có độ trễ thấp nhất   |
| **Geolocation**      | Định tuyến theo vị trí địa lý của người dùng               |
| **Weighted**         | Phân bổ % traffic (dùng cho canary releases)                |
| **Health Check**     | Kết hợp với mọi policy để tự động loại bỏ endpoint lỗi     |

### Cấu Hình Route 53 Failover

```bash
# 1. Tạo health check cho primary endpoint
aws route53 create-health-check \
  --caller-reference unique-ref-1 \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "primary.example.com",
    "Port": 443,
    "RequestInterval": 10,
    "FailureThreshold": 3
  }'

# 2. Tạo DNS record PRIMARY
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "HealthCheckId": "abc-health-check-id",
        "AliasTarget": {
          "DNSName": "primary-alb.ap-southeast-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Route 53 Health Check Đặc Điểm

- Kiểm tra từ **8-15 health checkers** ở nhiều vị trí khác nhau
- Endpoint được đánh dấu **unhealthy** khi >18% health checkers báo lỗi
- **Calculated Health Checks**: Tổng hợp nhiều health checks con
- **CloudWatch Alarm Health Checks**: Dùng metric để đánh giá sức khỏe

---

## 🏆 Best Practices Multi-AZ

### Checklist Thiết Kế

```
✅ Triển khai ít nhất 2 AZ (tốt hơn là 3)
✅ ASG subnet bao gồm cả 3 AZ
✅ ALB span tất cả AZ trong Region
✅ RDS Multi-AZ bật cho mọi database production
✅ ElastiCache Redis với Multi-AZ + auto-failover
✅ Không dùng Elastic IP tĩnh (gây single AZ dependency)
✅ Test failover định kỳ (ít nhất mỗi quý)
✅ Route 53 health checks kết hợp với DNS failover
✅ Dùng EFS (Elastic File System) thay NFS nếu cần shared storage Multi-AZ
```

### Anti-Patterns (Mẫu Phản Diện — Những Điều Không Nên Làm)

| Anti-Pattern                                  | Vấn Đề                                     | Giải Pháp                          |
| --------------------------------------------- | ------------------------------------------ | ---------------------------------- |
| Hardcode một AZ trong Launch Template         | Instance luôn ở một AZ                     | Để ASG quyết định AZ               |
| Dùng instance store cho dữ liệu quan trọng    | Mất dữ liệu khi instance bị thay thế       | Dùng EBS hoặc S3                   |
| ELB chỉ cấu hình một AZ                       | Mất LB khi AZ đó lỗi                       | Span ALB qua ≥2 AZ                 |
| Single RDS instance không Multi-AZ            | Database SPOF                              | Bật Multi-AZ cho RDS production    |
| Không test failover                           | Failover có thể không hoạt động khi cần   | Chaos Engineering định kỳ          |

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Multi-AZ và Multi-Region?**
> Multi-AZ: Phân tán trong một Region, bảo vệ chống lỗi AZ đơn, độ trễ thấp (<2ms), tự động failover. Multi-Region: Phân tán qua nhiều Region, bảo vệ chống thảm họa quy mô Region, độ trễ cao hơn (>50ms), failover chậm hơn và phức tạp hơn.

**Q: RDS Multi-AZ hoạt động thế nào?**
> RDS tạo Standby replica ở AZ khác, sao chép đồng bộ (mọi write được confirm ở cả Primary và Standby). Khi Primary lỗi, AWS tự cập nhật DNS endpoint sang Standby (~60-120 giây). Standby không phục vụ read traffic — muốn read scaling thì dùng Read Replica.

**Q: Tại sao Cross-Zone Load Balancing quan trọng?**
> Không có Cross-Zone LB, traffic chỉ đi đến instances trong cùng AZ với ALB node đang nhận request. Nếu AZ-1a có 2 instances và AZ-1b có 8 instances, traffic phân phối không đều. Cross-Zone LB đảm bảo mỗi instance nhận lượng traffic bằng nhau bất kể AZ nào.

**Q: Làm sao đảm bảo ECS tasks phân phối đều qua AZ?**
> Dùng placement strategy `spread` với field `attribute:ecs.availability-zone`. ECS sẽ cố gắng đặt tasks đều qua các AZ.

---

**Tiếp Theo:** [2-scaling-strategies.md](./2-scaling-strategies.md) — Chiến Lược Co Giãn Tự Động
