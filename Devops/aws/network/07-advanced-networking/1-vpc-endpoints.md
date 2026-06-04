# VPC Endpoints — Điểm Cuối VPC (Gateway & Interface)

> **Thuộc topic:** 07-advanced-networking | **Mức độ:** Nâng cao

---

## 1. Tại Sao Cần VPC Endpoints?

### Vấn đề khi không có VPC Endpoints

Trong kiến trúc mặc định, khi EC2 hoặc Lambda trong **private subnet** cần gọi AWS services (S3, DynamoDB, SSM, Secrets Manager…), traffic phải đi theo đường vòng:

```
EC2 (private subnet)
  → NAT Gateway (trong public subnet)
  → Internet Gateway
  → Internet công cộng (public internet)
  → AWS Service Endpoint (public)
```

**Vấn đề phát sinh:**

| Vấn đề | Chi tiết |
|--------|---------|
| **Bảo mật** | Traffic đi qua internet công cộng, có nguy cơ bị nghe lén hoặc MITM |
| **Chi phí** | NAT Gateway tính $0.045/GB data processed + $0.045/giờ |
| **Latency** | Thêm độ trễ do phải routing qua internet |
| **Compliance** | HIPAA, PCI-DSS, SOC 2 yêu cầu data không rời khỏi mạng riêng |
| **Độ phức tạp** | Phải duy trì NAT Gateway, quản lý Elastic IP |

### Giải pháp: VPC Endpoints

VPC Endpoint (Điểm Cuối VPC) là một **kết nối riêng tư** giữa VPC của bạn và AWS services được hỗ trợ, sử dụng **AWS PrivateLink** hoặc **route table** — không qua internet, không cần NAT Gateway, không cần Internet Gateway.

```
EC2 (private subnet)
  → VPC Endpoint (trong VPC)
  → AWS Private Network
  → AWS Service
```

---

## 2. Hai Loại VPC Endpoints

AWS cung cấp hai loại VPC Endpoints với cơ chế hoạt động khác nhau:

### 2.1. Gateway Endpoints — Điểm Cuối Cổng

**Định nghĩa:** Một entry trong **route table** của subnet, trỏ traffic đến AWS service qua AWS internal network thay vì Internet Gateway.

**Chỉ hỗ trợ 2 services:**
- **Amazon S3** (Simple Storage Service — Dịch Vụ Lưu Trữ Đơn Giản)
- **Amazon DynamoDB** (Cơ Sở Dữ Liệu NoSQL)

**Cách hoạt động:**

```
Route Table của subnet:
  10.0.0.0/16  → local
  0.0.0.0/0    → NAT Gateway (trước khi có endpoint)
  
  Sau khi thêm Gateway Endpoint:
  10.0.0.0/16  → local
  0.0.0.0/0    → NAT Gateway
  pl-xxxxx      → vpce-xxxxxxxx   ← Prefix List trỏ tới S3/DynamoDB
```

**Đặc điểm Gateway Endpoint:**
- **Miễn phí** — không tốn phí theo giờ hay theo GB
- Hoạt động ở cấp **route table** — không tạo ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi)
- **Không** hỗ trợ kết nối từ on-premises qua VPN/Direct Connect
- **Không** hỗ trợ kết nối từ VPC Peering peers hoặc Transit Gateway
- Chỉ có thể truy cập từ **trong VPC đó**
- Hỗ trợ **endpoint policy** để kiểm soát truy cập

**Diagram hoạt động:**

```
┌─────────────────────────────────────────────┐
│                  VPC                         │
│                                              │
│  ┌──────────────────┐                        │
│  │  Private Subnet  │                        │
│  │                  │                        │
│  │  ┌────────────┐  │    Route Table:        │
│  │  │  EC2 / λ   │  │    pl-xxxxx →         │
│  │  └─────┬──────┘  │    vpce-xxxxxxxx      │
│  │        │         │         │              │
│  └────────┼─────────┘         │              │
│           └───────────────────┘              │
│                    Gateway Endpoint          │
└─────────────────────────────────────────────┘
                    │
                    ▼ (AWS Internal Network)
             ┌──────────────┐
             │   S3 / DDB   │
             └──────────────┘
```

### 2.2. Interface Endpoints — Điểm Cuối Giao Diện

**Định nghĩa:** Một **ENI** (Elastic Network Interface — Giao Diện Mạng Đàn Hồi) được tạo trong subnet của bạn với **private IP address**. Traffic đến AWS services đi qua ENI này.

**Hỗ trợ hầu hết AWS services**, bao gồm:
- SSM (Systems Manager), Secrets Manager
- ECR (Elastic Container Registry), ECS
- SQS, SNS, Kinesis
- CloudWatch, CloudTrail, Config
- KMS (Key Management Service)
- API Gateway, Lambda
- Và hơn 100 services khác

**Cách hoạt động:**

```
EC2 → [DNS resolution] → Interface Endpoint ENI (private IP: 10.0.1.45)
   → AWS Private Network → AWS Service
```

**Đặc điểm Interface Endpoint:**
- **Có phí:** ~$0.01/giờ/AZ + $0.01/GB data processed
- Tạo ra **ENI thật** trong subnet với private IP
- Hỗ trợ kết nối từ **on-premises** qua VPN / Direct Connect
- Hỗ trợ kết nối từ **VPC Peering** và **Transit Gateway**
- Hỗ trợ **Private DNS** (thay thế public DNS của service)
- Hỗ trợ **endpoint policy**
- Nên triển khai ở **nhiều AZ** (Availability Zone — Vùng Khả Dụng) để HA

**Diagram hoạt động:**

```
┌─────────────────────────────────────────────────────┐
│                        VPC                           │
│                                                      │
│  ┌──────────────┐      ┌──────────────────────────┐  │
│  │  Private     │      │  Interface Endpoint ENI  │  │
│  │  Subnet A    │      │  IP: 10.0.1.45           │  │
│  │              │      │  (trong Subnet A)         │  │
│  │  ┌────────┐  │─────>│                          │  │
│  │  │  EC2   │  │      └────────────┬─────────────┘  │
│  │  └────────┘  │                   │                 │
│  └──────────────┘                   │                 │
│                                     │                 │
│  ┌──────────────┐      ┌────────────┴─────────────┐  │
│  │  Private     │      │  Interface Endpoint ENI  │  │
│  │  Subnet B    │      │  IP: 10.0.2.67           │  │
│  └──────────────┘      │  (trong Subnet B)         │  │
│                        └────────────┬─────────────┘  │
└─────────────────────────────────────┼────────────────┘
                                      │ AWS Private Network
                                      ▼
                               ┌─────────────┐
                               │  AWS Service │
                               │  (SSM, ECR…) │
                               └─────────────┘
```

---

## 3. So Sánh: Gateway vs Interface Endpoints

| Khía cạnh | Gateway Endpoint | Interface Endpoint |
|-----------|-----------------|-------------------|
| **Services hỗ trợ** | Chỉ S3 và DynamoDB | Hầu hết AWS services |
| **Cơ chế** | Route table entry | ENI với private IP |
| **Chi phí** | **Miễn phí** | ~$0.01/giờ/AZ + $0.01/GB |
| **On-premises access** | Không hỗ trợ | Hỗ trợ (qua VPN/DX) |
| **VPC Peering access** | Không hỗ trợ | Hỗ trợ |
| **Transit Gateway access** | Không hỗ trợ | Hỗ trợ |
| **Private DNS** | Không | Có (thay thế public DNS) |
| **Security Groups** | Không | Có thể gán SG cho ENI |
| **Multiple AZ** | Tự động (managed) | Phải tạo ENI ở từng AZ |
| **IP address** | Không có private IP | Có private IP trong subnet |

---

## 4. Endpoint Policies — Chính Sách Điểm Cuối

Endpoint Policy là document JSON dạng IAM Policy, gắn trực tiếp vào endpoint để kiểm soát **ai có thể làm gì** qua endpoint đó.

### Ví dụ: Endpoint Policy cho S3 — chỉ cho phép truy cập bucket cụ thể

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictToSpecificBucket",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-data-bucket",
        "arn:aws:s3:::my-company-data-bucket/*"
      ]
    },
    {
      "Sid": "AllowAWSManagedBuckets",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "*",
      "Resource": [
        "arn:aws:s3:::aws-ssm-*",
        "arn:aws:s3:::aws-windows-downloads-*",
        "arn:aws:s3:::amazon-ssm-*"
      ]
    }
  ]
}
```

**Lưu ý quan trọng:**
- Endpoint policy **không thay thế** IAM policies hay S3 bucket policies — tất cả đều phải cho phép (Allow) thì traffic mới đi qua
- Default endpoint policy là `Allow *` (cho phép tất cả)
- Endpoint policy chỉ kiểm soát traffic đi qua endpoint, không ảnh hưởng traffic qua đường khác

### Kết hợp S3 Bucket Policy với VPC Endpoint

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonVPCAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-secure-bucket",
        "arn:aws:s3:::my-secure-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0a1b2c3d4e5f67890"
        }
      }
    }
  ]
}
```

Bucket policy trên **từ chối tất cả** request không đến từ VPC endpoint cụ thể — đây là cách enforce "chỉ được truy cập S3 từ nội bộ VPC".

---

## 5. DNS Resolution cho Interface Endpoints — Private DNS

### DNS hoạt động như thế nào?

Khi tạo Interface Endpoint với **Private DNS enabled** (mặc định là bật):

1. AWS tự động tạo Route 53 Private Hosted Zone trong VPC
2. DNS record `ssm.ap-southeast-1.amazonaws.com` được resolve thành **private IP** của ENI
3. Ứng dụng không cần thay đổi gì — vẫn dùng endpoint URL cũ nhưng traffic đi qua private network

```
Không có Private DNS:
  Ứng dụng gọi: ssm.ap-southeast-1.amazonaws.com
  DNS resolve: 54.xxx.xxx.xxx (public IP của SSM)
  → Traffic ra internet

Có Private DNS:
  Ứng dụng gọi: ssm.ap-southeast-1.amazonaws.com
  DNS resolve: 10.0.1.45 (private IP của Interface Endpoint ENI)
  → Traffic ở trong VPC
```

### Điều kiện để Private DNS hoạt động

- VPC phải bật **DNS Hostnames** và **DNS Resolution**
- Có thể kiểm tra qua console: VPC → Actions → Edit VPC settings

```bash
# Bật DNS Hostnames cho VPC
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0123456789abcdef0 \
  --enable-dns-hostnames

# Bật DNS Resolution cho VPC  
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0123456789abcdef0 \
  --enable-dns-support
```

### Endpoint-specific DNS names

Ngoài private DNS, mỗi endpoint còn có **endpoint-specific DNS names**:

```
vpce-0a1b2c3d4e5f67890-xyz12345.ssm.ap-southeast-1.vpce.amazonaws.com
```

DNS này luôn resolve thành private IP, ngay cả khi Private DNS bị tắt.

---

## 6. Use Cases Thực Tế

### Use Case 1: EC2 gọi S3 không qua NAT Gateway

**Tình huống:** Hệ thống xử lý dữ liệu — EC2 trong private subnet cần đọc/ghi file S3 với throughput cao.

**Giải pháp:** Tạo **Gateway Endpoint** cho S3.

**Lợi ích:**
- Tiết kiệm hoàn toàn chi phí NAT Gateway cho S3 traffic
- Không cần NAT Gateway nếu S3 là dịch vụ duy nhất cần truy cập
- Throughput không bị giới hạn bởi NAT Gateway bandwidth

```bash
# Tính toán tiết kiệm:
# NAT Gateway: $0.045/GB × 10TB/tháng = $450/tháng
# Gateway Endpoint: $0
# → Tiết kiệm $450/tháng
```

### Use Case 2: Lambda trong VPC gọi AWS services

**Tình huống:** Lambda function trong VPC cần gọi Secrets Manager, SSM Parameter Store, SQS.

**Vấn đề nếu không có endpoint:** Lambda cần NAT Gateway để ra internet → thêm chi phí + latency.

**Giải pháp:** Tạo Interface Endpoints cho từng service.

```
Lambda (VPC) → Interface Endpoint ENI → AWS Service
              (không cần NAT Gateway)
```

### Use Case 3: ECS task pull image từ ECR

**Tình huống:** ECS Fargate task trong private subnet cần pull Docker image từ ECR (Elastic Container Registry — Kho Lưu Trữ Container Đàn Hồi).

**Cần các endpoints:**
- `com.amazonaws.region.ecr.api` — ECR API
- `com.amazonaws.region.ecr.dkr` — Docker Registry
- `com.amazonaws.region.s3` — S3 (ECR lưu layers trên S3)
- `com.amazonaws.region.logs` — CloudWatch Logs

---

## 7. Ví Dụ AWS CLI — Tạo VPC Endpoints

### Tạo Gateway Endpoint cho S3

```bash
# Tìm prefix list ID của S3
aws ec2 describe-prefix-lists \
  --filters "Name=prefix-list-name,Values=com.amazonaws.ap-southeast-1.s3"

# Tạo Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0a1b2c3d4e5f67890 rtb-0fedcba987654321 \
  --policy-document file://endpoint-policy.json

# Output:
# {
#   "VpcEndpoint": {
#     "VpcEndpointId": "vpce-0a1b2c3d4e5f67890",
#     "VpcEndpointType": "Gateway",
#     "State": "available",
#     ...
#   }
# }
```

### Tạo Interface Endpoint cho SSM

```bash
# Tạo Security Group cho endpoint
aws ec2 create-security-group \
  --group-name "ssm-endpoint-sg" \
  --description "SG for SSM Interface Endpoint" \
  --vpc-id vpc-0123456789abcdef0

# Cho phép HTTPS inbound từ VPC CIDR
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.0.0/16

# Tạo Interface Endpoint cho SSM
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --service-name com.amazonaws.ap-southeast-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-0a1b2c3d4e5f67890 subnet-0fedcba987654321 \
  --security-group-ids sg-0a1b2c3d4e5f67890 \
  --private-dns-enabled

# Kiểm tra endpoint
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=vpc-0123456789abcdef0"
```

---

## 8. Ví Dụ Terraform — Tạo VPC Endpoints

```hcl
# ========================================
# Gateway Endpoint cho S3
# ========================================
resource "aws_vpc_endpoint" "s3_gateway" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_a.id,
    aws_route_table.private_b.id,
  ]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource  = [
          "arn:aws:s3:::${var.data_bucket_name}",
          "arn:aws:s3:::${var.data_bucket_name}/*"
        ]
      }
    ]
  })

  tags = {
    Name = "s3-gateway-endpoint"
  }
}

# ========================================
# Security Group cho Interface Endpoints
# ========================================
resource "aws_security_group" "vpc_endpoints" {
  name        = "vpc-endpoints-sg"
  description = "Security group for VPC Interface Endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
    description = "Allow HTTPS from VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "vpc-endpoints-sg"
  }
}

# ========================================
# Interface Endpoint cho SSM
# ========================================
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ssm"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
  ]

  security_group_ids = [aws_security_group.vpc_endpoints.id]

  tags = {
    Name = "ssm-interface-endpoint"
  }
}

# ========================================
# Interface Endpoint cho Secrets Manager
# ========================================
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids         = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  tags = {
    Name = "secretsmanager-interface-endpoint"
  }
}

# ========================================
# Interface Endpoint cho ECR
# ========================================
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids         = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  tags = { Name = "ecr-api-endpoint" }
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids         = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  tags = { Name = "ecr-dkr-endpoint" }
}
```

---

## 9. Best Practices & Common Pitfalls

### Best Practices (Thực Hành Tốt Nhất)

**1. Luôn dùng Gateway Endpoints cho S3 và DynamoDB**
- Miễn phí, không có lý do gì không dùng
- Tạo cho tất cả route tables trong VPC

**2. Triển khai Interface Endpoints ở nhiều AZ**
- Tạo ENI ở ít nhất 2 AZ để tránh single point of failure (điểm lỗi duy nhất)
- Dùng Service Discovery thay vì hardcode endpoint IP

**3. Dùng endpoint-specific Security Group**
- Tạo SG riêng cho endpoints, chỉ cho phép HTTPS (443) từ VPC CIDR
- Không dùng SG quá permissive (rộng rãi)

**4. Bật Private DNS cho Interface Endpoints**
- Cho phép ứng dụng dùng endpoint URL thông thường mà không cần thay đổi code
- Yêu cầu VPC có DNS Hostnames và DNS Resolution được bật

**5. Viết Endpoint Policy có hạn chế**
- Không để default Allow All trong môi trường production
- Giới hạn actions và resources cần thiết

### Common Pitfalls (Lỗi Thường Gặp)

**Pitfall 1: Quên bật DNS Hostnames/Resolution trong VPC**
```
Triệu chứng: Private DNS không hoạt động, traffic vẫn ra internet
Giải pháp: Bật cả hai trong VPC settings
```

**Pitfall 2: Route table của subnet chưa được liên kết với Gateway Endpoint**
```
Triệu chứng: EC2 trong subnet không đi qua endpoint
Giải pháp: Kiểm tra route table của subnet có chứa prefix list route không
```

**Pitfall 3: Security Group của Interface Endpoint không cho phép traffic từ EC2**
```
Triệu chứng: Connection timeout khi gọi AWS service
Giải pháp: Thêm inbound rule HTTPS (443) từ EC2 SG hoặc VPC CIDR
```

**Pitfall 4: Thiếu endpoint cho ECR khi chạy ECS/EKS**
```
Triệu chứng: ECS task không pull được image, task ở trạng thái PENDING mãi
Giải pháp: Cần đủ 3 endpoints: ecr.api, ecr.dkr, s3 (cho image layers)
```

**Pitfall 5: Endpoint Policy quá hạn chế**
```
Triệu chứng: Ứng dụng bị denied dù IAM role đúng
Giải pháp: Nhớ rằng cả IAM policy VÀ endpoint policy đều phải Allow
```

**Pitfall 6: Gateway Endpoint không hoạt động với cross-VPC access**
```
Triệu chứng: On-premises hoặc peered VPC không thể dùng Gateway Endpoint
Giải pháp: Dùng Interface Endpoint thay thế — hỗ trợ cross-VPC access
```

---

## 10. Kiểm Tra Endpoint Hoạt Động

```bash
# Kiểm tra S3 Gateway Endpoint từ EC2
# (Chạy từ EC2 trong private subnet — không có NAT Gateway)
aws s3 ls s3://my-bucket --region ap-southeast-1

# Nếu có endpoint → thành công
# Nếu không có endpoint → timeout (vì không có đường ra internet)

# Kiểm tra SSM Interface Endpoint
aws ssm get-parameter --name "/myapp/db-password" --with-decryption

# Xem route cho S3 endpoint
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
  --query 'RouteTables[*].Routes[?GatewayId!=null]'

# Xem tất cả endpoints trong VPC
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
  --query 'VpcEndpoints[*].{ID:VpcEndpointId,Service:ServiceName,State:State}'
```

---

## 11. Chi Phí Tham Khảo (ap-southeast-1 — Singapore)

| Loại | Chi phí |
|------|---------|
| Gateway Endpoint (S3, DynamoDB) | **Miễn phí** |
| Interface Endpoint — giờ hoạt động | $0.013/giờ/endpoint/AZ |
| Interface Endpoint — data processed | $0.013/GB |
| NAT Gateway — giờ hoạt động | $0.059/giờ |
| NAT Gateway — data processed | $0.059/GB |

**Ví dụ tính toán tiết kiệm:**
```
Scenario: 1 AZ, 5 Interface Endpoints, xử lý 1TB data/tháng

Chi phí Interface Endpoints:
  - Giờ hoạt động: 5 × $0.013 × 24 × 30 = $46.8/tháng
  - Data: 1TB × $0.013 = $13/tháng
  - Tổng: ~$59.8/tháng

Chi phí NAT Gateway thay thế:
  - Giờ hoạt động: $0.059 × 24 × 30 = $42.5/tháng
  - Data: 1TB × $0.059 = $60.4/tháng
  - Tổng: ~$102.9/tháng

→ Tiết kiệm: ~$43/tháng (và không có traffic ra internet)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Sự khác biệt chính giữa Gateway Endpoint và Interface Endpoint là gì?

**Trả lời:**
- **Gateway Endpoint:** Chỉ hỗ trợ S3 và DynamoDB, hoạt động qua route table entry (prefix list), **miễn phí**, không tạo ENI, không hỗ trợ on-premises hay cross-VPC access.
- **Interface Endpoint:** Hỗ trợ hầu hết AWS services, tạo ENI với private IP trong subnet, **có phí** (~$0.013/giờ/AZ), hỗ trợ on-premises qua VPN/Direct Connect, hỗ trợ Private DNS.

**Cách chọn:** Với S3 và DynamoDB → luôn dùng Gateway Endpoint (miễn phí). Với các service khác → dùng Interface Endpoint.

---

### Q2: Endpoint Policy là gì? Nó khác gì với IAM Policy?

**Trả lời:**
Endpoint Policy là resource-based policy gắn vào VPC Endpoint, kiểm soát **traffic đi qua endpoint**. Nó **bổ sung** thêm lớp kiểm soát, không thay thế IAM Policy hay S3 Bucket Policy.

Nguyên tắc: **Tất cả ba lớp** (IAM Policy + Endpoint Policy + Resource Policy) đều phải Allow thì request mới thành công. Nếu bất kỳ lớp nào Deny → request bị từ chối.

Ứng dụng phổ biến: Tạo Endpoint Policy chỉ cho phép truy cập một số bucket S3 cụ thể, đồng thời dùng S3 Bucket Policy với điều kiện `aws:sourceVpce` để chặn truy cập từ ngoài VPC.

---

### Q3: Tại sao ECS Fargate task trong private subnet không pull được image từ ECR?

**Trả lời:**
ECS Fargate task cần 3 loại endpoints để pull image từ ECR mà không cần NAT Gateway:
1. `ecr.api` — để gọi ECR API (authentication, get download URL)
2. `ecr.dkr` — để giao tiếp Docker Registry protocol
3. `s3` — Gateway Endpoint — vì ECR lưu image layers trên S3

Ngoài ra, task cũng cần endpoint cho CloudWatch Logs để gửi logs. Nếu thiếu bất kỳ endpoint nào, task sẽ ở trạng thái `PENDING` hoặc fail khi pulling image.

---

### Q4: Private DNS của Interface Endpoint hoạt động như thế nào? Điều kiện để nó hoạt động?

**Trả lời:**
Khi bật Private DNS, AWS tạo Route 53 Private Hosted Zone trong VPC với record DNS của AWS service (ví dụ: `ssm.ap-southeast-1.amazonaws.com`) trỏ về private IP của ENI endpoint. Mọi DNS query từ trong VPC sẽ resolve ra private IP thay vì public IP.

Điều kiện:
1. VPC phải bật **enableDnsHostnames** = true
2. VPC phải bật **enableDnsSupport** = true
3. Tạo endpoint ở **đúng VPC** muốn dùng Private DNS

---

### Q5: Gateway Endpoint có hoạt động cho on-premises servers kết nối qua VPN không?

**Trả lời:**
**Không.** Gateway Endpoint là route table entry, chỉ có hiệu lực cho traffic xuất phát từ **trong VPC đó**. On-premises servers kết nối qua VPN/Direct Connect không thể sử dụng Gateway Endpoint.

Giải pháp: Dùng **Interface Endpoint** — ENI có IP thật trong subnet, on-premises server có thể route traffic đến đó qua VPN/Direct Connect. Sau đó từ ENI, traffic đi qua AWS private network đến S3/DynamoDB.

---

### Q6: Làm thế nào để enforce rằng S3 bucket chỉ được truy cập từ một VPC cụ thể?

**Trả lời:**
Dùng **S3 Bucket Policy** với điều kiện `aws:sourceVpce` hoặc `aws:sourceVpc`:

```json
{
  "Condition": {
    "StringNotEquals": {
      "aws:sourceVpce": "vpce-0a1b2c3d4e5f67890"
    }
  }
}
```

Điều này đảm bảo mọi request không xuất phát từ VPC Endpoint cụ thể đều bị **Deny**, kể cả request có IAM credentials hợp lệ. Kết hợp với Endpoint Policy để có lớp kiểm soát kép.

---

## Điều Hướng

- [← README.md](./README.md) — Tổng quan 07-advanced-networking
- [→ 2-privatelink.md](./2-privatelink.md) — AWS PrivateLink
- [3-global-accelerator.md](./3-global-accelerator.md) — Global Accelerator
- [4-elastic-ip-eni.md](./4-elastic-ip-eni.md) — Elastic IP & ENI
- [5-ipv6.md](./5-ipv6.md) — IPv6 & Dual-Stack
- [← 06-connectivity](../06-connectivity/) — VPN, Direct Connect, Transit Gateway
