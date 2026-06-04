# AWS PrivateLink — Kết Nối Dịch Vụ Qua Mạng Nội Bộ

> AWS PrivateLink là công nghệ cho phép truy cập dịch vụ AWS, dịch vụ của bên thứ ba, hoặc dịch vụ nội bộ của bạn qua mạng riêng tư AWS mà không cần internet gateway, NAT, hay VPN.

## 📚 Mục Lục

1. [Khái Niệm PrivateLink](#khái-niệm-privatelink)
2. [Kiến Trúc Provider-Consumer](#kiến-trúc-provider-consumer)
3. [VPC Peering vs PrivateLink](#vpc-peering-vs-privatelink)
4. [Thiết Lập PrivateLink](#thiết-lập-privatelink)
5. [Use Cases Thực Tế](#use-cases-thực-tế)
6. [Best Practices](#best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm PrivateLink

### PrivateLink là gì?

PrivateLink tạo kết nối private one-way (một chiều) từ **consumer VPC** đến **provider VPC** thông qua Interface VPC Endpoint. Traffic không bao giờ rời mạng AWS.

```
Consumer VPC                          Provider VPC
┌──────────────────┐                 ┌──────────────────┐
│                  │                 │                  │
│  EC2 App Server  │                 │  NLB             │
│       │          │                 │  ↓               │
│  Interface VPC   │◄──PrivateLink──►│  Service Targets │
│  Endpoint (ENI)  │                 │  (EC2/ECS/Lambda)│
│  10.0.1.55       │                 │                  │
└──────────────────┘                 └──────────────────┘
```

### Thành Phần Chính

**Phía Provider (Nhà Cung Cấp Dịch Vụ):**
- **NLB** (Network Load Balancer — Cân Bằng Tải Mạng) làm entry point
- **VPC Endpoint Service** — expose NLB qua PrivateLink

**Phía Consumer (Người Dùng Dịch Vụ):**
- **Interface VPC Endpoint** — ENI trong subnet với private IP
- **Private DNS** — tên miền tùy chỉnh resolve về IP của endpoint

### Đặc Điểm Quan Trọng

| Đặc Điểm | Mô Tả |
|---|---|
| **Định hướng** | One-way: consumer gọi đến provider, không ngược lại |
| **Overlapping IP** | Cho phép dù 2 VPC có CIDR trùng nhau |
| **Cross-account** | Consumer và Provider có thể ở account khác nhau |
| **Cross-region** | Không hỗ trợ natively; cần VPC Peering kết hợp |
| **Scalability** | Tự động scale theo NLB, không giới hạn số consumer |

---

## Kiến Trúc Provider-Consumer

### Mô Hình Hub-and-Spoke với PrivateLink

```
Shared Services Account (Hub)
┌──────────────────────────────────────┐
│  VPC: 10.0.0.0/16                    │
│                                       │
│  ┌─────────────────────────────┐     │
│  │  Authentication Service     │     │
│  │  (ECS Fargate)             │     │
│  └──────────────┬──────────────┘     │
│                 │                     │
│  ┌──────────────▼──────────────┐     │
│  │  NLB (Internal)             │     │
│  └──────────────┬──────────────┘     │
│                 │                     │
│  ┌──────────────▼──────────────┐     │
│  │  VPC Endpoint Service       │     │
│  │  (com.amazonaws.vpce.xxx)   │     │
│  └─────────────────────────────┘     │
└──────────────────────────────────────┘
              ▲          ▲
    PrivateLink│          │PrivateLink
              │          │
┌─────────────┴──┐  ┌────┴────────────┐
│ Team A Account │  │ Team B Account  │
│ VPC: 10.1.0/16 │  │ VPC: 10.1.0/16 │ ← CIDR có thể trùng!
│  Interface VPC │  │  Interface VPC  │
│  Endpoint      │  │  Endpoint       │
└────────────────┘  └─────────────────┘
```

---

## VPC Peering vs PrivateLink

| Tiêu Chí | VPC Peering | PrivateLink |
|---|---|---|
| **Kết nối** | Bidirectional (hai chiều) | Unidirectional (một chiều) |
| **Overlapping CIDR** | ❌ Không cho phép | ✅ Cho phép |
| **Transitive routing** | ❌ Không hỗ trợ | N/A |
| **Expose cụ thể** | Toàn bộ VPC | Chỉ service qua NLB |
| **Scale** | Phức tạp khi nhiều VPC | Tự động qua NLB |
| **Chi phí** | Chỉ data transfer | Có phí endpoint + data |
| **Cross-account** | ✅ | ✅ |
| **Use case** | VPC cần giao tiếp 2 chiều | Expose service an toàn |

**Quy tắc chọn:**
- Dùng **VPC Peering** khi hai VPC cần giao tiếp hai chiều, CIDR không trùng
- Dùng **PrivateLink** khi muốn expose một dịch vụ cụ thể cho nhiều consumer, bảo mật cao hơn, CIDR có thể trùng

---

## Thiết Lập PrivateLink

### Bước 1: Phía Provider — Tạo VPC Endpoint Service

```bash
# Tạo NLB trước (đây là target của PrivateLink)
aws elbv2 create-load-balancer \
  --name nlb-auth-service \
  --type network \
  --scheme internal \
  --subnets subnet-private-a subnet-private-b

# Tạo VPC Endpoint Service từ NLB
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:...:loadbalancer/net/nlb-auth-service/... \
  --acceptance-required \       # Consumer phải được chấp thuận trước
  --no-private-dns-name-configuration

# Lấy service name để chia sẻ với consumer
aws ec2 describe-vpc-endpoint-service-configurations
# → ServiceName: com.amazonaws.vpce.us-east-1.vpce-svc-0abc123
```

### Bước 2: Phía Provider — Cho Phép Account Consumer

```bash
# Cho phép account cụ thể tạo endpoint đến service
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0abc123 \
  --add-allowed-principals arn:aws:iam::123456789012:root

# Hoặc cho phép toàn bộ AWS (không khuyến nghị cho service nội bộ)
# --add-allowed-principals "*"
```

### Bước 3: Phía Consumer — Tạo Interface Endpoint

```bash
# Tạo Interface VPC Endpoint đến service của provider
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-consumer \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-0abc123 \
  --subnet-ids subnet-app-a subnet-app-b \
  --security-group-ids sg-endpoint \
  --private-dns-enabled
```

### Bước 4: Phía Provider — Chấp Thuận Request

```bash
# Xem pending connections
aws ec2 describe-vpc-endpoint-connections \
  --filters "Name=vpc-endpoint-service-id,Values=vpce-svc-0abc123"

# Chấp thuận kết nối
aws ec2 accept-vpc-endpoint-connections \
  --service-id vpce-svc-0abc123 \
  --vpc-endpoint-ids vpce-consumer-endpoint-id
```

### Terraform: Provider Side

```terraform
# NLB cho service
resource "aws_lb" "service_nlb" {
  name               = "nlb-auth-service"
  load_balancer_type = "network"
  internal           = true
  subnets            = aws_subnet.private[*].id
}

# VPC Endpoint Service
resource "aws_vpc_endpoint_service" "auth" {
  acceptance_required        = true
  network_load_balancer_arns = [aws_lb.service_nlb.arn]

  allowed_principals = [
    "arn:aws:iam::${var.consumer_account_id}:root"
  ]

  tags = { Name = "auth-service-endpoint" }
}

# Output service name để chia sẻ với consumer
output "endpoint_service_name" {
  value = aws_vpc_endpoint_service.auth.service_name
}
```

### Terraform: Consumer Side

```terraform
# Interface Endpoint kết nối đến provider service
resource "aws_vpc_endpoint" "auth_service" {
  vpc_id              = aws_vpc.consumer.id
  service_name        = var.auth_service_endpoint_name  # Từ provider output
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids         = aws_subnet.app[*].id
  security_group_ids = [aws_security_group.endpoint.id]

  tags = { Name = "vpce-auth-service" }
}
```

---

## Use Cases Thực Tế

### 1. SaaS Platform — Expose API cho Khách Hàng

```
SaaS Provider VPC
  NLB → API Gateway (ECS) → VPC Endpoint Service
      ↓
  Khách hàng A VPC: Interface Endpoint → gọi API qua private
  Khách hàng B VPC: Interface Endpoint → gọi API qua private
```

Lợi ích: Khách hàng không cần mở firewall cho IP public; API không expose ra internet.

### 2. Shared Services — Centralized Auth, Logging

```
Shared Services VPC (Account: security-tools)
  Authentication Service (Keycloak/OAuth2)
  Centralized Log Aggregation
  Secrets Vault
    → VPC Endpoint Service
        ↓
  Dev Account VPC → Interface Endpoint
  Staging Account VPC → Interface Endpoint
  Production Account VPC → Interface Endpoint
```

### 3. Data Platform — Secure Data Access

```
Data Platform VPC
  Apache Kafka (MSK) → VPC Endpoint Service
    ↓
  Analytics Account → Interface Endpoint → consume events
  ML Platform Account → Interface Endpoint → feature store
```

---

## Best Practices

1. **Luôn bật Acceptance Required** — xét duyệt thủ công mỗi consumer request
2. **Dùng Private DNS** — consumer không cần biết IP của endpoint
3. **Restrict Allowed Principals** — chỉ cho phép account cụ thể, không dùng `"*"`
4. **Monitor với CloudWatch** — metric `ActiveConnections`, `NewConnections` của NLB
5. **Tag rõ ràng** — ghi rõ owner, purpose, consumer accounts
6. **Đặt NLB ở nhiều AZ** — tránh single point of failure

---

## Câu Hỏi Phỏng Vấn

### Q: PrivateLink khác VPC Peering ở điểm gì?

**Trả lời:** Điểm khác biệt chính:
- **PrivateLink** expose một service cụ thể (qua NLB); consumer chỉ thấy service đó, không thấy toàn bộ VPC provider. Cho phép CIDR trùng. Kết nối một chiều.
- **VPC Peering** kết nối toàn bộ 2 VPC hai chiều; mọi resource trong VPC có thể giao tiếp; không cho phép CIDR trùng.

Dùng PrivateLink khi muốn expose service an toàn (principle of least privilege); dùng VPC Peering khi 2 team cần full connectivity.

### Q: PrivateLink có hỗ trợ cross-region không?

**Trả lời:** Không natively. PrivateLink hoạt động trong cùng region. Để cross-region, cần dùng VPC Peering (hoặc Transit Gateway) giữa 2 region, rồi trong mỗi region dùng PrivateLink. AWS hiện đang thêm hỗ trợ cross-region cho một số dịch vụ managed, nhưng custom VPC Endpoint Service vẫn cần cùng region.

### Q: Khi nào nên dùng PrivateLink thay vì VPC Endpoint Service thông thường?

**Trả lời:** PrivateLink là nền tảng kỹ thuật của Interface VPC Endpoint — chúng là cùng một thứ về mặt kỹ thuật. "VPC Endpoint Service" là thuật ngữ phía provider; "Interface VPC Endpoint" là phía consumer; PrivateLink là tên gọi chung cho công nghệ này.

---

## 🔗 Xem Thêm

- [3-vpc-endpoints.md](3-vpc-endpoints.md) — Gateway và Interface Endpoints
- [README.md](README.md) — Tổng quan Network Security
- [1-security-groups.md](1-security-groups.md) — Security Group cho endpoint ENI

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
