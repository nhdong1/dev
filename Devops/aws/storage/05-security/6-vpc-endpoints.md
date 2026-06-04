# VPC Endpoints cho S3 và EFS

> VPC Endpoint — Điểm Cuối VPC: kết nối riêng tư giữa VPC — Virtual Private Cloud và AWS services mà không cần internet gateway, NAT device, VPN, hay Direct Connect.

## 📚 Mục Lục

1. [Tổng Quan VPC Endpoints](#1-tổng-quan-vpc-endpoints)
2. [Gateway Endpoint cho S3](#2-gateway-endpoint-cho-s3)
3. [Interface Endpoint cho S3](#3-interface-endpoint-cho-s3)
4. [Interface Endpoint cho EFS](#4-interface-endpoint-cho-efs)
5. [VPC Endpoint Policy](#5-vpc-endpoint-policy)
6. [So Sánh Gateway vs Interface](#6-so-sánh-gateway-vs-interface)
7. [Thiết Kế Kiến Trúc](#7-thiết-kế-kiến-trúc)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan VPC Endpoints

### Vấn Đề VPC Endpoints Giải Quyết

```
Không có VPC Endpoint:
┌─────────────────────────────────────────────────────────┐
│ VPC                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐ │
│  │ EC2      │───▶│ NAT Gateway  │───▶│ Internet GW   │ │
│  │ Private  │    │ ($0.045/hr   │    └───────────────┘ │
│  │ Subnet   │    │ + data fee)  │           │           │
│  └──────────┘    └──────────────┘           │           │
└───────────────────────────────────────────── │ ─────────┘
                                              ▼
                                         Internet (Public)
                                              │
                                              ▼
                                           S3 / EFS
Nhược điểm:
- Chi phí NAT Gateway cao ($0.045/GB data)
- Traffic qua internet — rủi ro bảo mật
- Phụ thuộc internet availability

Với VPC Endpoint:
┌─────────────────────────────────────────────────────────┐
│ VPC                                                     │
│  ┌──────────┐                                           │
│  │ EC2      │──── VPC Endpoint ────▶ S3 / EFS           │
│  │ Private  │    (Private link,                         │
│  │ Subnet   │     AWS network)                          │
│  └──────────┘                                           │
└─────────────────────────────────────────────────────────┘
Lợi ích:
✅ Không qua internet — bảo mật hơn
✅ Gateway Endpoint: miễn phí
✅ Latency thấp hơn (trong AWS network)
✅ Không cần NAT Gateway cho S3/DynamoDB traffic
```

### Hai Loại VPC Endpoints

| Loại | Cơ Chế | Hỗ Trợ Dịch Vụ | Chi Phí |
|------|--------|----------------|---------|
| **Gateway Endpoint** | Route table entry | S3, DynamoDB | Miễn phí |
| **Interface Endpoint** (PrivateLink) | ENI trong subnet | 100+ dịch vụ AWS | $0.01/AZ/hr + data |

---

## 2. Gateway Endpoint cho S3

### Cách Hoạt Động

```
EC2 (Private Subnet)
     │
     │ s3.amazonaws.com → route table
     ▼
Route Table:
┌────────────────────────────────────────────────────────┐
│ Destination        │ Target                             │
├────────────────────┼────────────────────────────────────┤
│ 10.0.0.0/8         │ local                              │
│ 0.0.0.0/0          │ nat-gateway-xxx (internet)         │
│ pl-xxxxxxxx (S3)   │ vpce-xxxxxxxxx (VPC Endpoint) ← Mới│
└────────────────────┴────────────────────────────────────┘
     │
     │ Traffic đến S3 đi theo route endpoint
     ▼
S3 (AWS internal network — không qua internet)
```

**Prefix List** — Danh sách tiền tố: tập hợp CIDR ranges của AWS service (ví dụ S3 IPs) được quản lý tự động bởi AWS.

### Tạo Gateway Endpoint

```bash
# 1. Lấy service name
aws ec2 describe-vpc-endpoint-services \
  --filters Name=service-type,Values=Gateway \
  --query 'ServiceDetails[?contains(ServiceName,`s3`)].ServiceName'

# 2. Lấy route table IDs cần update
ROUTE_TABLE_IDS=$(aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-xxxxxxxxx \
  --query 'RouteTables[*].RouteTableId' \
  --output text)

# 3. Tạo Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxxx \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $ROUTE_TABLE_IDS

# 4. Kiểm tra endpoint đã tạo
aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=vpc-xxxxxxxxx \
            Name=service-name,Values=com.amazonaws.ap-southeast-1.s3
```

### Kiểm Tra Hoạt Động Từ EC2

```bash
# SSH vào EC2 private instance
# Thử truy cập S3 — phải thành công dù không có NAT Gateway
aws s3 ls s3://my-bucket

# Xem route table để confirm prefix list
aws ec2 describe-route-tables \
  --route-table-ids rtb-xxxxxxxxx \
  --query 'RouteTables[*].Routes[?GatewayId!=null]'
```

---

## 3. Interface Endpoint cho S3

### Cách Hoạt Động (PrivateLink)

```
VPC (ap-southeast-1)
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  EC2 (Private Subnet A)      EC2 (Private Subnet B)         │
│       │                            │                        │
│       └──────────────┬─────────────┘                        │
│                      │                                      │
│             Interface Endpoints (ENIs)                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Subnet A: ENI 10.0.1.100 ← vpce-xxx                  │   │
│  │ Subnet B: ENI 10.0.2.100 ← vpce-xxx                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                      │                                      │
└──────────────────────┼──────────────────────────────────────┘
                       │ AWS PrivateLink (không qua internet)
                       ▼
                    S3 API
```

### DNS Resolution với Interface Endpoint

```
Private DNS enabled (khuyến nghị):
s3.ap-southeast-1.amazonaws.com → 10.0.1.100 (ENI IP)

Private DNS disabled:
s3.ap-southeast-1.amazonaws.com → 52.x.x.x (public IP)
bucket.vpce-xxx.s3.ap-southeast-1.vpce.amazonaws.com → 10.0.1.100
```

### Tạo Interface Endpoint cho S3

```bash
# Tạo Interface Endpoint cho S3 (hỗ trợ private DNS)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxxx \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-aaaaaaaa subnet-bbbbbbbb \
  --security-group-ids sg-xxxxxxxxx \
  --private-dns-enabled

# Security Group cho endpoint (cho phép HTTPS từ VPC)
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxx \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.0.0/8
```

---

## 4. Interface Endpoint cho EFS

### EFS Không Hỗ Trợ Gateway Endpoint

```
EFS mount targets sử dụng ENI trong subnet — đã là private by design.
Không cần VPC Endpoint cho EFS vì:
- Mount target đã là private IP trong VPC
- NFS traffic không rời khỏi VPC

EFS cần VPC Endpoint cho:
- elasticfilesystem API calls (CreateFileSystem, DescribeFileSystems, v.v.)
- Không phải cho NFS data transfer
```

### Tạo Interface Endpoint cho EFS API

```bash
# Endpoint cho EFS management API
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxxx \
  --service-name com.amazonaws.ap-southeast-1.elasticfilesystem \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-aaaaaaaa \
  --security-group-ids sg-xxxxxxxxx \
  --private-dns-enabled
```

---

## 5. VPC Endpoint Policy

### Policy Kiểm Soát Quyền Qua Endpoint

VPC Endpoint Policy áp dụng **bổ sung** lên IAM policy và bucket policy. Tất cả phải Allow thì mới được truy cập.

### Policy Chỉ Cho Phép Bucket Của Tổ Chức

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrganizationBucketsOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::org-bucket-*",
        "arn:aws:s3:::org-bucket-*/*"
      ]
    },
    {
      "Sid": "AllowListAllBuckets",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    }
  ]
}
```

### Policy Ngăn Data Exfiltration — Rò Rỉ Dữ Liệu

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PreventDataExfiltration",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "s3:ResourceAccount": [
            "123456789012",
            "234567890123"
          ]
        }
      }
    }
  ]
}
```

`s3:ResourceAccount` — chỉ cho phép truy cập bucket thuộc về các account trong danh sách, ngăn upload sang bucket của attacker bên ngoài tổ chức.

### Policy Kết Hợp Với Bucket Policy

```json
// VPC Endpoint Policy (áp dụng cho mọi request qua endpoint)
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::company-data",
      "arn:aws:s3:::company-data/*"
    ]
  }]
}

// Bucket Policy (bổ sung thêm điều kiện)
{
  "Statement": [{
    "Sid": "AllowFromVPCEndpointOnly",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::company-data", "arn:aws:s3:::company-data/*"],
    "Condition": {
      "StringNotEquals": {
        "aws:SourceVpce": "vpce-xxxxxxxxxxxxxxxxx"
      }
    }
  }]
}
// Kết quả: chỉ request từ đúng VPC endpoint VÀ có quyền IAM mới được phép
```

---

## 6. So Sánh Gateway vs Interface

| Tiêu Chí | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|----------|-----------------|--------------------------------|
| **Dịch vụ hỗ trợ** | S3, DynamoDB | 100+ dịch vụ AWS |
| **Cơ chế** | Route table entry | ENI trong subnet |
| **Chi phí** | **Miễn phí** | $0.01/AZ/hr + $0.01/GB |
| **DNS** | Không thay đổi DNS | Private DNS có thể dùng |
| **Cross-region** | Không | Không |
| **On-premises** | Không | **Có** (qua Direct Connect/VPN) |
| **Security Group** | Không | **Có** — granular control |
| **HA — High Availability** | Tự động | Cần tạo ở nhiều AZ |
| **Latency** | Thấp | Thấp (tương đương) |
| **Khả năng scale** | Tự động | Tự động |

### Khi Nào Dùng Gateway Endpoint

```
✅ Truy cập S3 hoặc DynamoDB từ EC2 trong cùng region
✅ Muốn tiết kiệm chi phí NAT Gateway
✅ Không cần on-premises access qua endpoint
✅ Đơn giản — chỉ cần update route table

→ Dùng Gateway Endpoint cho S3
```

### Khi Nào Dùng Interface Endpoint

```
✅ Cần truy cập S3 từ on-premises qua Direct Connect
✅ Cần private DNS resolution
✅ Cần Security Group để kiểm soát ai được dùng endpoint
✅ Dịch vụ không hỗ trợ Gateway Endpoint (EFS, KMS, ECR, v.v.)

→ Dùng Interface Endpoint cho EFS API, KMS, và các dịch vụ khác
```

---

## 7. Thiết Kế Kiến Trúc

### Kiến Trúc Multi-Layer Security

```
┌──────────────────────────────────────────────────────────────┐
│ VPC (10.0.0.0/16)                                            │
│                                                              │
│  Private Subnet A (10.0.1.0/24)                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ EC2 App Server                                         │  │
│  │  IAM Role: AppRole                                     │  │
│  │  ↓ chỉ được làm s3:GetObject và s3:PutObject           │  │
│  └──────────────────────────┬───────────────────────────┘  │
│                              │                               │
│  Route Table:                │                               │
│  S3 → Gateway Endpoint       │                               │
│                              ▼                               │
│              ┌───────────────────────────┐                  │
│              │ VPC Gateway Endpoint       │                  │
│              │ Policy: chỉ allow          │                  │
│              │ company account buckets    │                  │
│              └────────────┬──────────────┘                  │
└───────────────────────────┼──────────────────────────────────┘
                            │ AWS Private Network
                            ▼
                    ┌──────────────────┐
                    │ S3 Bucket        │
                    │ Bucket Policy:   │
                    │ - Deny non-VPCE  │
                    │ - Deny HTTP      │
                    │ - SSE-KMS only   │
                    └──────────────────┘
```

### Terraform — Infrastructure as Code

```hcl
# Gateway Endpoint cho S3
resource "aws_vpc_endpoint" "s3_gateway" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = "*"
      Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
      Resource  = [
        "arn:aws:s3:::${var.bucket_name}",
        "arn:aws:s3:::${var.bucket_name}/*"
      ]
    }]
  })

  tags = {
    Name = "s3-gateway-endpoint"
  }
}

# Interface Endpoint cho KMS (dùng với SSE-KMS)
resource "aws_vpc_endpoint" "kms" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.kms"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.endpoint_sg.id]
  private_dns_enabled = true

  tags = {
    Name = "kms-interface-endpoint"
  }
}
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Tại sao nên tạo VPC Endpoint cho cả KMS khi dùng SSE-KMS với S3?**

A: Khi EC2 trong private subnet truy cập S3 với SSE-KMS, S3 gọi KMS để decrypt data key. Nếu không có KMS VPC Endpoint, KMS call phải đi qua NAT Gateway ra internet — tốn phí và kém an toàn. Tạo KMS Interface Endpoint giữ toàn bộ luồng bảo mật trong AWS private network: EC2 → S3 Endpoint → S3 → KMS Endpoint → KMS.

---

**Q: Khác biệt giữa `aws:SourceVpc` và `aws:SourceVpce` trong bucket policy?**

A: `aws:SourceVpc` kiểm tra request đến từ VPC nào — phù hợp khi muốn cho phép toàn bộ VPC bất kể endpoint nào. `aws:SourceVpce` kiểm tra request đến qua VPC Endpoint cụ thể nào — phù hợp khi muốn kiểm soát chính xác hơn, ví dụ chỉ endpoint được tạo cho môi trường production. Dùng `aws:SourceVpce` khi cần audit granular và đảm bảo traffic đi qua endpoint đúng.

---

**Q: Một tổ chức muốn đảm bảo on-premises systems truy cập S3 mà không qua internet. Giải pháp nào?**

A: Dùng S3 Interface Endpoint kết hợp Direct Connect hoặc VPN: Direct Connect/VPN kết nối on-premises với VPC, Interface Endpoint trong VPC cho phép traffic S3 đi từ on-premises → Direct Connect → VPC → Interface Endpoint → S3 mà không cần qua public internet. Gateway Endpoint không hỗ trợ kịch bản này vì chỉ hoạt động cho traffic khởi nguồn từ trong VPC. Chi phí thêm: Interface Endpoint + Direct Connect, nhưng bảo mật và compliance tốt hơn nhiều.

---

**Cập Nhật:** 2026-05-16
