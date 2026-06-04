# AWS PrivateLink — Kết Nối Dịch Vụ Riêng Tư

> **Thuộc topic:** 07-advanced-networking | **Mức độ:** Nâng cao

---

## 1. PrivateLink Là Gì?

**AWS PrivateLink** là công nghệ cho phép bạn **expose một dịch vụ** (service) từ VPC của mình (Provider VPC) cho các VPC khác hoặc account AWS khác (Consumer VPC) tiêu dùng thông qua **private IP address** — không đi qua internet, không cần VPC Peering toàn phần, không expose toàn bộ mạng VPC.

### Vấn đề PrivateLink giải quyết

**Bài toán:** Công ty A có microservice xử lý thanh toán trong VPC-A. Công ty B (hoặc team B trong cùng tổ chức) cần gọi API đó từ VPC-B.

**Các phương án và hạn chế:**

| Phương án | Hạn chế |
|-----------|---------|
| Public Internet | Không an toàn, latency cao, không phù hợp sensitive data |
| VPC Peering | Expose **toàn bộ** VPC-A cho VPC-B; không hoạt động nếu CIDR bị overlap |
| Transit Gateway | Cần quản lý routing phức tạp, expensive, expose nhiều hơn cần thiết |
| **AWS PrivateLink** | Chỉ expose **dịch vụ cụ thể** qua private IP, không cần routing toàn VPC |

### Nguyên tắc cốt lõi

```
Provider VPC                        Consumer VPC
┌─────────────────┐                ┌─────────────────┐
│                 │                │                 │
│  Service        │                │  EC2 / Lambda   │
│  (bất kỳ app)  │                │  (consumer)     │
│       │         │                │       │         │
│       ▼         │                │       │         │
│  Network Load   │                │  Interface      │
│  Balancer (NLB) │◄───PrivateLink─┤  Endpoint (ENI) │
│       │         │                │  10.1.2.45      │
│  Endpoint       │                │                 │
│  Service        │                │                 │
└─────────────────┘                └─────────────────┘
```

**Đặc điểm quan trọng:**
- Consumer chỉ thấy một **private IP** trong VPC của họ
- CIDR của hai VPC **có thể overlap** — không quan trọng
- Không cần route propagation, không cần peering connection
- Consumer **không thể** initiate traffic ra ngoài dịch vụ được expose

---

## 2. Kiến Trúc Chi Tiết

### Phía Provider (Nhà Cung Cấp Dịch Vụ)

**Bước 1:** Triển khai service (EC2, ECS, EKS, Lambda…)

**Bước 2:** Đặt service sau một **Network Load Balancer (NLB)**
- NLB bắt buộc — ALB không hỗ trợ làm endpoint service
- NLB có thể ở internal (không cần internet-facing)

**Bước 3:** Tạo **VPC Endpoint Service** (Dịch Vụ Điểm Cuối VPC) từ NLB
- AWS cấp một **service name** dạng: `com.amazonaws.vpce.ap-southeast-1.vpce-svc-0a1b2c3d4e5f67890`
- Cấu hình acceptance policy: manual (thủ công) hoặc automatic (tự động)
- Whitelist (cho phép) các AWS account ID được phép tạo endpoint đến service này

### Phía Consumer (Người Dùng Dịch Vụ)

**Bước 1:** Tìm service name của provider (qua console, API, hoặc provider thông báo)

**Bước 2:** Tạo **Interface Endpoint** (VPC Endpoint loại Interface) trỏ đến service name đó
- AWS tạo ENI trong subnet của consumer với private IP
- ENI này là "entry point" để gọi service của provider

**Bước 3:** (Nếu provider cần manual acceptance) Chờ provider chấp thuận request

**Bước 4:** Sau khi accepted, consumer gọi service qua private IP của ENI

---

## 3. Endpoint Services — Dịch Vụ Điểm Cuối

### Tạo Endpoint Service từ NLB

```bash
# Bước 1: Tạo NLB cho service
aws elbv2 create-load-balancer \
  --name my-service-nlb \
  --type network \
  --scheme internal \
  --subnets subnet-0a1b2c3d4e5f67890 subnet-0fedcba987654321

# Bước 2: Tạo Endpoint Service từ NLB
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:ap-southeast-1:123456789012:loadbalancer/net/my-service-nlb/abcdef1234567890 \
  --acceptance-required \
  --no-private-dns-name \
  --tag-specifications 'ResourceType=vpc-endpoint-service,Tags=[{Key=Name,Value=my-payment-service}]'

# Output:
# {
#   "ServiceConfiguration": {
#     "ServiceType": [{"ServiceType": "Interface"}],
#     "ServiceId": "vpce-svc-0a1b2c3d4e5f67890",
#     "ServiceName": "com.amazonaws.vpce.ap-southeast-1.vpce-svc-0a1b2c3d4e5f67890",
#     "ServiceState": "Available",
#     "AcceptanceRequired": true
#   }
# }

# Bước 3: Thêm permission cho account consumer
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --add-allowed-principals arn:aws:iam::987654321098:root
```

### Kiểm tra và quản lý connections

```bash
# Xem các connection requests đang pending
aws ec2 describe-vpc-endpoint-connections \
  --filters "Name=vpc-endpoint-service-id,Values=vpce-svc-0a1b2c3d4e5f67890" \
  --query 'VpcEndpointConnections[?VpcEndpointState==`pendingAcceptance`]'

# Accept một endpoint connection
aws ec2 accept-vpc-endpoint-connections \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --vpc-endpoint-ids vpce-0fedcba987654321

# Reject một endpoint connection
aws ec2 reject-vpc-endpoint-connections \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --vpc-endpoint-ids vpce-badactor1234567
```

---

## 4. Acceptance — Chấp Nhận Kết Nối

### Manual Acceptance (Chấp Nhận Thủ Công)

- Provider phải **review và chấp thuận** từng request tạo endpoint từ consumer
- Phù hợp khi: Provider cần kiểm soát chặt ai được kết nối
- Ví dụ: SaaS provider muốn onboard từng khách hàng thủ công
- Consumer endpoint ở trạng thái `pendingAcceptance` cho đến khi provider accept

### Automatic Acceptance (Chấp Nhận Tự Động)

- Mọi request từ account được whitelist đều được **tự động chấp thuận**
- Phù hợp khi: Internal services trong cùng tổ chức, không cần review thủ công
- Dùng khi đã trust tất cả account trong whitelist

```bash
# Chuyển từ manual sang automatic acceptance
aws ec2 modify-vpc-endpoint-service-configuration \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --no-acceptance-required   # automatic

# Chuyển lại manual
aws ec2 modify-vpc-endpoint-service-configuration \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --acceptance-required      # manual
```

---

## 5. So Sánh: PrivateLink vs VPC Peering vs Transit Gateway

| Khía cạnh | VPC Peering | Transit Gateway | PrivateLink |
|-----------|------------|----------------|-------------|
| **Expose scope** | Toàn bộ VPC | Toàn bộ VPC/mạng | Chỉ dịch vụ cụ thể |
| **Overlapping CIDR** | Không hỗ trợ | Không hỗ trợ | Hỗ trợ |
| **Transitive routing** | Không | Có | N/A |
| **Cross-account** | Có (phức tạp) | Có (RAM) | Có (đơn giản) |
| **Cross-region** | Có | Có | Có (từ 2021) |
| **Chi phí** | $0.01/GB | $0.05/GB + $0.05/giờ | $0.01/giờ + $0.01/GB |
| **Setup phức tạp** | Thấp | Cao | Trung bình |
| **Security** | Trung bình | Trung bình | Cao (chỉ expose service) |
| **Scalability** | Giới hạn (125 peering/VPC) | Cao | Cao |
| **Giao thức** | Mọi giao thức | Mọi giao thức | TCP |

**Khi nào dùng cái gì:**
- **VPC Peering:** Hai VPC cần giao tiếp đầy đủ, CIDR không overlap, số lượng ít
- **Transit Gateway:** Hub-and-spoke cho nhiều VPC, kết nối on-premises, routing phức tạp
- **PrivateLink:** Expose một service cụ thể, SaaS pattern, overlapping CIDR, security cao

---

## 6. AWS-Managed PrivateLink Services

AWS cung cấp sẵn hàng trăm Interface Endpoints cho các dịch vụ của họ — đây cũng là PrivateLink ở phía backend, chỉ là AWS đóng vai provider.

**Danh sách phổ biến:**

| Service | Endpoint service name |
|---------|----------------------|
| SSM | com.amazonaws.region.ssm |
| Secrets Manager | com.amazonaws.region.secretsmanager |
| ECR API | com.amazonaws.region.ecr.api |
| ECR Docker | com.amazonaws.region.ecr.dkr |
| CloudWatch Logs | com.amazonaws.region.logs |
| SQS | com.amazonaws.region.sqs |
| SNS | com.amazonaws.region.sns |
| KMS | com.amazonaws.region.kms |
| API Gateway | com.amazonaws.region.execute-api |
| Lambda | com.amazonaws.region.lambda |
| Kinesis Streams | com.amazonaws.region.kinesis-streams |

```bash
# Xem tất cả services có sẵn trong region
aws ec2 describe-vpc-endpoint-services \
  --query 'ServiceDetails[*].{Name:ServiceName,Type:ServiceType[0].ServiceType}' \
  --output table
```

---

## 7. Cross-Account và Cross-Region PrivateLink

### Cross-Account (Giữa Các Tài Khoản)

```
Account A (Provider)               Account B (Consumer)
┌────────────────────┐             ┌────────────────────┐
│  NLB + Service     │             │  Interface         │
│  vpce-svc-xxxxx    │◄────────────│  Endpoint          │
│  Whitelist:        │             │  (ENI private IP)  │
│  - arn:aws:iam::   │             │                    │
│    AccountB:root   │             │                    │
└────────────────────┘             └────────────────────┘
```

**Bước thực hiện:**
1. Account A tạo Endpoint Service, whitelist Account B ARN
2. Account B tạo Interface Endpoint trỏ đến service name của Account A
3. Account A accept connection (hoặc auto-accept)
4. Account B gọi service qua private IP của ENI

### Cross-Region PrivateLink

Từ 2021, AWS hỗ trợ cross-region PrivateLink:

```
Region A (Provider)         AWS Backbone          Region B (Consumer)
┌──────────────────┐       ──────────────        ┌──────────────────┐
│  NLB + Endpoint  │──────►AWS Private Net◄───────│  Interface       │
│  Service         │       ────────────────       │  Endpoint        │
└──────────────────┘                              └──────────────────┘
```

**Lưu ý cross-region:**
- Tốn thêm **data transfer costs** qua region (tương tự inter-region transfer)
- Độ trễ cao hơn so với same-region
- Vẫn đi qua AWS backbone — không qua internet công cộng

```bash
# Tạo Interface Endpoint cross-region
# (Consumer ở ap-northeast-1, Provider ở ap-southeast-1)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --service-name com.amazonaws.vpce.ap-southeast-1.vpce-svc-0a1b2c3d4e5f67890 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-0a1b2c3d4e5f67890 \
  --security-group-ids sg-0a1b2c3d4e5f67890 \
  --region ap-northeast-1   # Region của consumer
```

---

## 8. DNS và Routing cho PrivateLink

### DNS Resolution cho Custom Endpoint Services

Không giống AWS-managed services, custom endpoint services **không có Private DNS tự động**. Consumer cần cấu hình DNS thủ công hoặc dùng endpoint-specific DNS.

**Endpoint-specific DNS name** (tự động có sẵn):
```
vpce-0a1b2c3d4e5f67890-xyz12345.vpce-svc-0a1b2c3d4e5f67890.ap-southeast-1.vpce.amazonaws.com
```

**Cấu hình Private DNS tùy chỉnh:**

1. Provider đăng ký **Private DNS name** cho endpoint service:

```bash
# Provider: yêu cầu Private DNS name cho service
aws ec2 modify-vpc-endpoint-service-configuration \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --private-dns-name "payment-api.internal.example.com"
```

2. AWS sẽ cấp TXT record để verify domain ownership. Provider phải thêm TXT record này vào DNS công cộng.

3. Sau khi verified, consumer bật Private DNS khi tạo endpoint, `payment-api.internal.example.com` resolve thành private IP của ENI.

### Routing không cần thay đổi

Một lợi thế của PrivateLink: **không cần thay đổi route tables**. Traffic routing được xử lý tự động thông qua ENI trong subnet của consumer. Consumer chỉ cần kết nối đến private IP của ENI.

---

## 9. Security — Bảo Mật PrivateLink

### NLB Security Groups (từ 2023)

Từ tháng 8/2023, NLB hỗ trợ Security Groups. Provider có thể gán SG cho NLB để kiểm soát traffic:

```hcl
resource "aws_security_group" "nlb_sg" {
  name   = "nlb-privatelink-sg"
  vpc_id = aws_vpc.provider.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # NLB chấp nhận từ PrivateLink
    description = "HTTPS from PrivateLink"
  }
}
```

### Endpoint Service Permissions

Provider kiểm soát **ai được phép tạo endpoint** đến service:

```bash
# Whitelist theo account
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --add-allowed-principals \
    arn:aws:iam::111111111111:root \
    arn:aws:iam::222222222222:root

# Whitelist theo IAM role cụ thể
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0a1b2c3d4e5f67890 \
  --add-allowed-principals \
    arn:aws:iam::111111111111:role/EndpointConsumerRole
```

### Consumer Security Group

Consumer gán Security Group cho Interface Endpoint ENI. Chỉ traffic từ các resources được allow mới đến được ENI:

```hcl
resource "aws_security_group" "consumer_endpoint_sg" {
  name   = "consumer-endpoint-sg"
  vpc_id = aws_vpc.consumer.id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]
    description     = "Allow from app servers only"
  }
}
```

### PrivateLink không expose nguồn gốc IP của consumer

Một đặc điểm bảo mật quan trọng: Provider nhìn thấy IP nguồn là **NLB private IP**, không phải IP thực của consumer. Nếu cần biết IP consumer (ví dụ để logging), cần bật **proxy protocol** trên NLB.

---

## 10. Terraform Example — Tạo PrivateLink Đầy Đủ

```hcl
# ============================================================
# PROVIDER SIDE — Phía nhà cung cấp dịch vụ
# ============================================================

# VPC của provider
resource "aws_vpc" "provider" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "provider-vpc" }
}

resource "aws_subnet" "provider_a" {
  vpc_id            = aws_vpc.provider.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-southeast-1a"
  tags              = { Name = "provider-subnet-a" }
}

resource "aws_subnet" "provider_b" {
  vpc_id            = aws_vpc.provider.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "ap-southeast-1b"
  tags              = { Name = "provider-subnet-b" }
}

# Network Load Balancer (NLB) — bắt buộc cho PrivateLink
resource "aws_lb" "provider_nlb" {
  name               = "payment-service-nlb"
  load_balancer_type = "network"
  internal           = true   # Internal NLB
  subnets            = [aws_subnet.provider_a.id, aws_subnet.provider_b.id]

  tags = { Name = "payment-service-nlb" }
}

# Target Group cho NLB
resource "aws_lb_target_group" "payment_service" {
  name        = "payment-service-tg"
  port        = 8443
  protocol    = "TCP"
  vpc_id      = aws_vpc.provider.id
  target_type = "ip"

  health_check {
    protocol = "TCP"
    port     = 8443
  }
}

# NLB Listener
resource "aws_lb_listener" "payment_https" {
  load_balancer_arn = aws_lb.provider_nlb.arn
  port              = 443
  protocol          = "TLS"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.payment_service.arn
  }
}

# VPC Endpoint Service — đây là "sản phẩm" được expose
resource "aws_vpc_endpoint_service" "payment_service" {
  network_load_balancer_arns = [aws_lb.provider_nlb.arn]
  acceptance_required        = true   # Yêu cầu manual acceptance

  # Whitelist account consumers
  allowed_principals = [
    "arn:aws:iam::${var.consumer_account_id}:root"
  ]

  tags = {
    Name = "payment-service-endpoint-service"
  }
}

# Output service name để share với consumers
output "endpoint_service_name" {
  value = aws_vpc_endpoint_service.payment_service.service_name
  description = "Share this with consumers to create their endpoint"
}

# ============================================================
# CONSUMER SIDE — Phía người dùng dịch vụ
# (Thường ở account / VPC riêng)
# ============================================================

# VPC của consumer (có thể overlap CIDR với provider — không sao)
resource "aws_vpc" "consumer" {
  cidr_block = "10.0.0.0/16"   # Overlap với provider — OK với PrivateLink!
  tags       = { Name = "consumer-vpc" }
}

resource "aws_subnet" "consumer_a" {
  vpc_id            = aws_vpc.consumer.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-southeast-1a"
}

# Security Group cho Interface Endpoint
resource "aws_security_group" "consumer_endpoint" {
  name   = "privatelink-consumer-sg"
  vpc_id = aws_vpc.consumer.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.consumer.cidr_block]
  }
}

# Interface Endpoint trỏ đến service của provider
resource "aws_vpc_endpoint" "payment_consumer" {
  vpc_id              = aws_vpc.consumer.id
  service_name        = var.payment_service_name  # Nhận từ provider output
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = false   # Custom service không có Private DNS tự động

  subnet_ids         = [aws_subnet.consumer_a.id]
  security_group_ids = [aws_security_group.consumer_endpoint.id]

  tags = { Name = "payment-service-consumer-endpoint" }
}

# Route 53 Private Hosted Zone để consumer có DNS đẹp
resource "aws_route53_zone" "consumer_private" {
  name = "internal.example.com"

  vpc {
    vpc_id = aws_vpc.consumer.id
  }
}

resource "aws_route53_record" "payment_api" {
  zone_id = aws_route53_zone.consumer_private.zone_id
  name    = "payment-api.internal.example.com"
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.payment_consumer.dns_entry[0].dns_name
    zone_id                = aws_vpc_endpoint.payment_consumer.dns_entry[0].hosted_zone_id
    evaluate_target_health = true
  }
}
```

---

## 11. Use Case Thực Tế: SaaS Provider Expose API qua PrivateLink

### Scenario: Công ty Fintech (FinPayCo) cung cấp Payment API cho khách hàng (BankApp)

```
BankApp VPC (Consumer)              FinPayCo VPC (Provider)
┌────────────────────────┐          ┌────────────────────────┐
│                        │          │                        │
│  App Server            │          │  Payment Service       │
│  (gọi payment API)     │          │  EC2 Cluster           │
│       │                │          │       │                │
│       │ private IP      │          │       ▼                │
│       ▼                │          │  Internal NLB          │
│  Interface Endpoint    │          │  (nlb-xxxxxx)          │
│  ENI: 192.168.1.50     │◄─────────│       │                │
│  (payment-api.bank.co) │PrivateLink│  Endpoint Service     │
│                        │          │  (vpce-svc-xxxxx)      │
└────────────────────────┘          │                        │
                                    │  Whitelist:            │
                                    │  - BankApp Account ID  │
                                    └────────────────────────┘
```

**Flow hoạt động:**
1. App Server của BankApp gọi `https://payment-api.bank.co/v1/charge`
2. DNS resolve `payment-api.bank.co` → `192.168.1.50` (private IP của ENI)
3. Traffic đến ENI trong BankApp VPC
4. AWS PrivateLink route traffic qua AWS backbone đến NLB của FinPayCo
5. NLB forward đến Payment Service EC2

**Lợi ích:**
- BankApp không biết IP thật của FinPayCo service
- FinPayCo không expose bất kỳ phần nào khác của VPC
- Toàn bộ traffic đi qua AWS private network
- FinPayCo có thể audit ai đang kết nối qua Connection Notifications (SNS)

### Connection Notifications — Thông Báo Kết Nối

```bash
# Provider set up SNS notification khi có endpoint connect/disconnect
aws ec2 create-vpc-endpoint-connection-notification \
  --connection-notification-arn arn:aws:sns:ap-southeast-1:123456789012:endpoint-connections \
  --connection-events Connect Disconnect Accept Reject \
  --service-id vpce-svc-0a1b2c3d4e5f67890
```

---

## 12. Monitoring PrivateLink

```bash
# CloudWatch metrics cho Endpoint Service
aws cloudwatch get-metric-statistics \
  --namespace AWS/PrivateLinkEndpoints \
  --metric-name ActiveConnections \
  --dimensions Name=VpcEndpointServiceName,Value=com.amazonaws.vpce.ap-southeast-1.vpce-svc-xxx \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Sum

# Metrics quan trọng:
# - ActiveConnections: số connection đang active
# - BytesProcessed: lượng data qua endpoint
# - PacketsDropped: packets bị drop (health check fail?)
```

---

## 13. Troubleshooting (Xử Lý Sự Cố)

| Triệu chứng | Nguyên nhân | Giải pháp |
|------------|-------------|-----------|
| Endpoint ở trạng thái `pendingAcceptance` | Provider chưa accept | Provider cần accept qua console hoặc CLI |
| Connection timeout từ consumer | SG của endpoint không cho phép traffic | Thêm inbound rule HTTPS từ consumer CIDR |
| `Name resolution failed` | Private DNS chưa cấu hình | Thêm Route 53 record cho endpoint DNS |
| `rejected` state | Provider reject connection | Kiểm tra whitelist, liên hệ provider |
| High latency bất thường | Cross-region endpoint | Xem xét deploy service gần consumer hơn |
| NLB health check fail | Service không healthy | Kiểm tra NLB target group health |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Sự khác biệt giữa PrivateLink và VPC Peering là gì? Khi nào chọn cái nào?

**Trả lời:**
**VPC Peering** kết nối hai VPC với nhau hoàn toàn — mọi resource trong VPC-A có thể giao tiếp với mọi resource trong VPC-B (tùy Security Group). Không hỗ trợ overlapping CIDR, không transitive. Phù hợp khi hai VPC cần giao tiếp đa dạng giữa nhiều services.

**PrivateLink** chỉ expose **một dịch vụ cụ thể** từ Provider VPC. Consumer không thể truy cập bất kỳ resource nào khác. Hỗ trợ overlapping CIDR. Phù hợp cho SaaS pattern, microservice isolation, cross-account service sharing.

**Chọn:** Cần expose toàn bộ VPC → VPC Peering. Chỉ cần expose một service → PrivateLink.

---

### Q2: Tại sao PrivateLink yêu cầu NLB mà không hỗ trợ ALB?

**Trả lời:**
PrivateLink hoạt động ở **Layer 4 (TCP/UDP)** — nó forward raw TCP connections từ consumer đến NLB của provider. ALB hoạt động ở Layer 7 (HTTP/HTTPS) và terminate connection, làm thay đổi packet header theo cách không tương thích với PrivateLink's connection proxying mechanism.

NLB là **connection-level proxy** (pass-through), phù hợp với PrivateLink's architecture. Ngoài ra, NLB có khả năng xử lý hàng triệu connections/giây với ultra-low latency, phù hợp với yêu cầu performance của PrivateLink.

---

### Q3: Hai VPC có overlapping CIDR có thể dùng PrivateLink không?

**Trả lời:**
**Có.** Đây là một trong những ưu điểm lớn nhất của PrivateLink so với VPC Peering. Vì traffic không đi qua route table của provider VPC mà đi qua ENI trong consumer VPC, AWS routing không cần phân biệt CIDR. Consumer chỉ cần biết private IP của ENI trong VPC **của họ** để kết nối.

---

### Q4: Làm thế nào để provider biết consumer nào đang kết nối đến service?

**Trả lời:**
Provider có nhiều cách:
1. **AWS Console/CLI:** Xem danh sách `VpcEndpointConnections` với trạng thái và account ID của consumer
2. **Connection Notifications:** Cấu hình SNS topic để nhận notification khi có connect/disconnect/accept/reject event
3. **NLB Access Logs:** Enable access logs trên NLB để xem IP source (dù IP là NLB private IP, có thể trace qua VPC Flow Logs)
4. **CloudWatch Metrics:** Theo dõi `ActiveConnections` để biết số kết nối đang hoạt động

---

### Q5: PrivateLink có hỗ trợ UDP không?

**Trả lời:**
Từ 2022, AWS PrivateLink hỗ trợ cả **TCP và UDP** thông qua Gateway Load Balancer (GWLB) endpoints. Với GWLB Endpoint Service, có thể forward UDP traffic (ví dụ: DNS, syslog) qua PrivateLink. Tuy nhiên, NLB-based PrivateLink chỉ hỗ trợ TCP và TLS.

---

### Q6: Consumer có thể initiate connections ngược lại từ provider về consumer không?

**Trả lời:**
**Không.** PrivateLink là **unidirectional** — chỉ consumer mới có thể initiate connection đến provider. Provider không thể kết nối ngược về consumer VPC. Đây là một tính năng bảo mật — consumer expose ENI nhưng provider không có khả năng scan hay probe consumer VPC.

Nếu cần bi-directional communication, mỗi bên cần tạo endpoint service riêng và bên kia tạo consumer endpoint tương ứng.

---

## Điều Hướng

- [← 1-vpc-endpoints.md](./1-vpc-endpoints.md) — VPC Endpoints (Gateway & Interface)
- [→ 3-global-accelerator.md](./3-global-accelerator.md) — Global Accelerator
- [4-elastic-ip-eni.md](./4-elastic-ip-eni.md) — Elastic IP & ENI
- [5-ipv6.md](./5-ipv6.md) — IPv6 & Dual-Stack
- [README.md](./README.md) — Tổng quan 07-advanced-networking
- [← 06-connectivity](../06-connectivity/) — VPN, Direct Connect, Transit Gateway
