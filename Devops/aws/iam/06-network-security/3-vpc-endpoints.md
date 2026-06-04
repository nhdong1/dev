# VPC Endpoints — Kết Nối Riêng Tư đến Dịch Vụ AWS

> VPC Endpoint (Điểm Cuối VPC) cho phép kết nối tài nguyên trong VPC đến dịch vụ AWS mà không cần đi qua internet công cộng, tăng bảo mật và giảm chi phí data transfer.

## 📚 Mục Lục

1. [Tại Sao Cần VPC Endpoints?](#tại-sao-cần-vpc-endpoints)
2. [Hai Loại VPC Endpoint](#hai-loại-vpc-endpoint)
3. [Gateway Endpoint](#gateway-endpoint)
4. [Interface Endpoint](#interface-endpoint)
5. [Endpoint Policies](#endpoint-policies)
6. [So Sánh Chi Phí](#so-sánh-chi-phí)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần VPC Endpoints?

### Vấn Đề Khi Không Có VPC Endpoint

```
EC2 (private subnet) → NAT Gateway → Internet → S3

Rủi ro:
  ✗ Traffic đi qua internet công cộng
  ✗ Chi phí NAT Gateway ($0.045/GB data processing)
  ✗ Chi phí data transfer ra internet ($0.09/GB)
  ✗ Không thể giới hạn bucket S3 nào EC2 được truy cập
```

### Sau Khi Có VPC Endpoint

```
EC2 (private subnet) → VPC Endpoint → S3

Lợi ích:
  ✓ Traffic hoàn toàn trong mạng AWS, không qua internet
  ✓ Không cần NAT Gateway cho S3/DynamoDB
  ✓ Endpoint Policy kiểm soát bucket/table nào được phép
  ✓ Giảm chi phí đáng kể
  ✓ Tăng tốc độ (network path ngắn hơn)
```

---

## Hai Loại VPC Endpoint

| Loại | Dịch Vụ Hỗ Trợ | Chi Phí | Mechanism |
|---|---|---|---|
| **Gateway Endpoint** | S3, DynamoDB | Miễn phí | Route table entry |
| **Interface Endpoint** | 100+ dịch vụ AWS | Có phí | ENI với private IP |

---

## Gateway Endpoint

### Cơ Chế Hoạt Động

Gateway Endpoint tạo một entry trong **route table** của subnet, chuyển hướng traffic đến dịch vụ thông qua AWS network backbone thay vì internet.

```
Route Table của Private Subnet (trước):
  Destination: 0.0.0.0/0   → Target: nat-gateway-id
  Destination: 10.0.0.0/16 → Target: local

Route Table của Private Subnet (sau khi tạo Gateway Endpoint):
  Destination: 0.0.0.0/0   → Target: nat-gateway-id
  Destination: pl-63a5400a  → Target: vpce-s3-endpoint-id   ← S3 Prefix List
  Destination: 10.0.0.0/16 → Target: local
```

S3 Prefix List (pl-xxxxxx) chứa danh sách IP range của S3 trong region — AWS tự quản lý, tự cập nhật.

### Tạo Gateway Endpoint cho S3

```bash
# Tạo VPC Endpoint cho S3
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-private-app rtb-private-db

# Xem endpoint đã tạo
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=vpc-12345"
```

### Terraform: Gateway Endpoint

```terraform
# S3 Gateway Endpoint — miễn phí
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_app.id,
    aws_route_table.private_db.id,
  ]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = "*"
      Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
      Resource = [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ]
    }]
  })

  tags = { Name = "vpce-s3-prod" }
}

# DynamoDB Gateway Endpoint — miễn phí
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private_app.id]
}
```

---

## Interface Endpoint

### Cơ Chế Hoạt Động

Interface Endpoint tạo một **ENI** (Elastic Network Interface — Giao Diện Mạng Đàn Hồi) với private IP trong subnet của bạn. DNS resolution được cấu hình để resolve tên miền dịch vụ AWS về private IP này.

```
EC2 gọi API: secretsmanager.us-east-1.amazonaws.com

Không có Interface Endpoint:
  DNS → 52.95.x.x (public IP) → đi qua internet

Có Interface Endpoint (Private DNS enabled):
  DNS → 10.0.2.45 (ENI trong private subnet) → AWS internal network
```

### Dịch Vụ Hỗ Trợ Interface Endpoint

Hơn 100 dịch vụ AWS, bao gồm:

| Dịch Vụ | Service Name |
|---|---|
| Secrets Manager | com.amazonaws.REGION.secretsmanager |
| SSM (Systems Manager) | com.amazonaws.REGION.ssm |
| EC2 Messages | com.amazonaws.REGION.ec2messages |
| SSM Messages | com.amazonaws.REGION.ssmmessages |
| KMS | com.amazonaws.REGION.kms |
| STS | com.amazonaws.REGION.sts |
| CloudWatch Logs | com.amazonaws.REGION.logs |
| ECR API | com.amazonaws.REGION.ecr.api |
| ECR Docker | com.amazonaws.REGION.ecr.dkr |
| ECS | com.amazonaws.REGION.ecs |
| Lambda | com.amazonaws.REGION.lambda |

### Tạo Interface Endpoint cho Secrets Manager

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids subnet-private-a subnet-private-b \
  --security-group-ids sg-endpoint \
  --private-dns-enabled
```

### Security Group cho Interface Endpoint

Interface Endpoint cần Security Group cho phép traffic từ tài nguyên trong VPC:

```terraform
resource "aws_security_group" "vpc_endpoints" {
  name        = "sg-vpc-endpoints"
  description = "Security Group cho Interface VPC Endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]    # Từ trong VPC
    description = "HTTPS từ trong VPC đến AWS services"
  }
}
```

### Terraform: Nhiều Interface Endpoints

```terraform
# Danh sách endpoints cần cho môi trường không có internet
locals {
  interface_endpoints = {
    "ssm"             = "com.amazonaws.${var.region}.ssm"
    "ssmmessages"     = "com.amazonaws.${var.region}.ssmmessages"
    "ec2messages"     = "com.amazonaws.${var.region}.ec2messages"
    "secretsmanager"  = "com.amazonaws.${var.region}.secretsmanager"
    "kms"             = "com.amazonaws.${var.region}.kms"
    "logs"            = "com.amazonaws.${var.region}.logs"
    "ecr_api"         = "com.amazonaws.${var.region}.ecr.api"
    "ecr_dkr"         = "com.amazonaws.${var.region}.ecr.dkr"
  }
}

resource "aws_vpc_endpoint" "interface" {
  for_each = local.interface_endpoints

  vpc_id              = aws_vpc.main.id
  service_name        = each.value
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids         = aws_subnet.private_app[*].id
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  tags = { Name = "vpce-${each.key}" }
}
```

---

## Endpoint Policies

### Khái Niệm

Endpoint Policy (Chính Sách Điểm Cuối) là IAM resource policy gắn với VPC Endpoint, kiểm soát **dịch vụ/resource nào** được phép truy cập qua endpoint đó, độc lập với IAM policy của principal.

### Ví Dụ: Chỉ Cho Phép Truy Cập Bucket Cụ Thể

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::company-app-bucket",
        "arn:aws:s3:::company-app-bucket/*",
        "arn:aws:s3:::aws-ssm-us-east-1",
        "arn:aws:s3:::aws-ssm-us-east-1/*"
      ]
    }
  ]
}
```

### S3 Bucket Policy: Yêu Cầu Dùng VPC Endpoint

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::sensitive-bucket",
        "arn:aws:s3:::sensitive-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-1234567890abcdef0"
        }
      }
    }
  ]
}
```

Bucket policy này đảm bảo **chỉ** có thể truy cập bucket qua VPC Endpoint cụ thể — từ internet sẽ bị từ chối.

---

## So Sánh Chi Phí

### Gateway Endpoint (S3, DynamoDB)

```
Chi phí: MIỄN PHÍ
Tiết kiệm khi dùng so với NAT Gateway:
  - NAT Gateway processing: $0.045/GB
  - Data transfer out: $0.09/GB
  - Tổng: ~$0.135/GB

1 TB data/tháng qua NAT → $138/tháng
1 TB data/tháng qua S3 Endpoint → $0/tháng (endpoint) + $0.023/GB (S3 storage)
```

### Interface Endpoint

```
Chi phí:
  - $0.01/endpoint-AZ/giờ = ~$7.30/tháng/AZ
  - $0.01/GB data processed

Trung bình 3 AZ, 8 endpoints:
  8 × 3 × $7.30 = $175.20/tháng cố định

Cần so sánh với chi phí NAT Gateway cho workload cụ thể.
```

---

## Best Practices

1. **Luôn tạo Gateway Endpoint cho S3 và DynamoDB** — miễn phí, giảm chi phí NAT
2. **Tạo Interface Endpoints cho môi trường highly secure** — không cần NAT Gateway
3. **Bật Private DNS** — application không cần thay đổi code, DNS tự resolve về private IP
4. **Dùng Endpoint Policy** — hạn chế resource được phép truy cập qua endpoint
5. **Kết hợp với S3 Bucket Policy** — yêu cầu truy cập phải qua endpoint cụ thể
6. **Deploy endpoint ở nhiều AZ** — High Availability (Tính Sẵn Sàng Cao)

---

## Câu Hỏi Phỏng Vấn

### Q: Sự khác biệt giữa Gateway Endpoint và Interface Endpoint?

**Trả lời:**
- **Gateway Endpoint:** Chỉ cho S3 và DynamoDB; miễn phí; hoạt động bằng cách thêm entry vào route table; không tạo ENI; không hỗ trợ cross-region
- **Interface Endpoint:** Hỗ trợ 100+ dịch vụ; có phí; tạo ENI trong subnet với private IP; hỗ trợ Private DNS; có thể cross-region; cần Security Group

### Q: Tại sao cần VPC Endpoint khi đã có NAT Gateway?

**Trả lời:** VPC Endpoint giải quyết cả 3 vấn đề mà NAT Gateway không giải quyết được:
1. **Bảo mật:** Traffic không rời AWS network, không qua internet công cộng
2. **Chi phí:** Gateway Endpoint miễn phí; Interface Endpoint thường rẻ hơn NAT cho workload nặng
3. **Kiểm soát:** Endpoint Policy cho phép giới hạn chính xác dịch vụ nào được truy cập

### Q: Làm thế nào đảm bảo EC2 trong private subnet chỉ truy cập S3 bucket của công ty?

**Trả lời:** Kết hợp hai cơ chế:
1. **Endpoint Policy** trên VPC Endpoint: chỉ Allow action trên `arn:aws:s3:::company-bucket`
2. **S3 Bucket Policy** với condition `aws:SourceVpce`: Deny nếu không đến từ VPC Endpoint cụ thể

Cả hai lớp đảm bảo dữ liệu chỉ có thể truy cập từ trong VPC và chỉ bucket được phép.

---

## 🔗 Xem Thêm

- [4-privatelink.md](4-privatelink.md) — PrivateLink cho dịch vụ tùy chỉnh
- [1-security-groups.md](1-security-groups.md) — Security Group cho endpoint ENI
- [README.md](README.md) — Tổng quan Network Security

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
