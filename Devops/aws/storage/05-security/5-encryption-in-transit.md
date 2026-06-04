# Mã Hóa In-Transit — TLS/HTTPS, VPC Endpoints

> Encryption in-transit — Mã hóa khi truyền tải: bảo vệ dữ liệu trong quá trình di chuyển qua mạng, ngăn chặn MITM — Man-in-the-Middle attack và nghe lén.

## 📚 Mục Lục

1. [Tại Sao Cần Mã Hóa In-Transit](#1-tại-sao-cần-mã-hóa-in-transit)
2. [TLS cho S3](#2-tls-cho-s3)
3. [Enforce HTTPS qua Bucket Policy](#3-enforce-https-qua-bucket-policy)
4. [VPC Endpoints cho S3](#4-vpc-endpoints-cho-s3)
5. [Mã Hóa In-Transit cho EBS](#5-mã-hóa-in-transit-cho-ebs)
6. [Mã Hóa In-Transit cho EFS](#6-mã-hóa-in-transit-cho-efs)
7. [Best Practices](#7-best-practices)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần Mã Hóa In-Transit

### Các Mối Đe Dọa Khi Truyền Không Mã Hóa

```
Không có TLS:
Client ──── HTTP ────▶ S3
               ▲
        Attacker đứng giữa:
        - Đọc được nội dung
        - Thay đổi dữ liệu
        - Lấy credentials
        - Replay attacks

Có TLS (HTTPS):
Client ──── HTTPS/TLS ────▶ S3
          Dữ liệu được mã hóa,
          certificate xác thực danh tính,
          không thể giả mạo hay nghe lén
```

### Compliance Yêu Cầu Mã Hóa In-Transit

| Standard | Yêu Cầu |
|----------|---------|
| **PCI-DSS** — Payment Card Industry | TLS 1.2+ bắt buộc cho cardholder data |
| **HIPAA** — Health Insurance Portability | Mã hóa PHI — Protected Health Information khi truyền |
| **SOC 2** — Service Organization Control | Controls về encryption in-transit |
| **ISO 27001** | Data in transit protection |
| **GDPR** — General Data Protection Regulation | Appropriate safeguards for personal data |

---

## 2. TLS cho S3

### TLS — Transport Layer Security — Bảo Mật Lớp Truyền Tải

#### S3 Hỗ Trợ TLS

```
Phiên bản TLS S3 hỗ trợ:
✅ TLS 1.2 — khuyến nghị tối thiểu
✅ TLS 1.3 — hiệu suất tốt hơn, bảo mật cao hơn
❌ TLS 1.0 — đã deprecated
❌ TLS 1.1 — đã deprecated
❌ SSL — hoàn toàn không an toàn
```

#### Kiểm Tra TLS Version

```bash
# Kiểm tra S3 bucket hỗ trợ TLS nào
curl -v --tlsv1.2 https://s3.amazonaws.com/ 2>&1 | grep "SSL connection"

# Kiểm tra với openssl
openssl s_client -connect s3.amazonaws.com:443 -tls1_2
openssl s_client -connect s3.amazonaws.com:443 -tls1_3

# Thử với TLS 1.1 (sẽ thất bại với S3)
openssl s_client -connect s3.amazonaws.com:443 -tls1_1
```

### AWS SDK Tự Động Dùng HTTPS

```python
import boto3

# Mặc định dùng HTTPS — không cần cấu hình
s3 = boto3.client('s3')

# Kiểm tra endpoint
print(s3.meta.endpoint_url)  # https://s3.amazonaws.com

# KHÔNG làm điều này (tắt SSL verification):
import urllib3
urllib3.disable_warnings()
s3 = boto3.client('s3', verify=False)  # ❌ KHÔNG BAO GIỜ
```

---

## 3. Enforce HTTPS qua Bucket Policy

### Bucket Policy Cơ Bản — Từ Chối HTTP

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceTLSRequestsOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": false
        }
      }
    }
  ]
}
```

**Giải thích:** `aws:SecureTransport` là `false` khi request dùng HTTP thay vì HTTPS. Statement này Deny mọi request HTTP.

### Enforce TLS 1.2 Tối Thiểu

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOldTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "NumericLessThan": {
          "s3:TlsVersion": 1.2
        }
      }
    }
  ]
}
```

### Kết Hợp Cả Hai Điều Kiện

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::secure-bucket",
        "arn:aws:s3:::secure-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": false
        }
      }
    },
    {
      "Sid": "RequireTLS12",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::secure-bucket",
        "arn:aws:s3:::secure-bucket/*"
      ],
      "Condition": {
        "NumericLessThan": {
          "s3:TlsVersion": 1.2
        }
      }
    }
  ]
}
```

### Kiểm Tra Policy Hoạt Động

```bash
# Thử truy cập qua HTTP (phải thất bại)
curl http://my-bucket.s3.amazonaws.com/test-file

# Thử truy cập qua HTTPS (phải thành công)
curl https://my-bucket.s3.amazonaws.com/test-file

# AWS CLI luôn dùng HTTPS
aws s3 cp test-file s3://my-bucket/
```

---

## 4. VPC Endpoints cho S3

### Lợi Ích VPC Endpoints

```
Không có VPC Endpoint:
EC2 ──── Internet Gateway ──── Internet ──── S3
         (lưu lượng ra ngoài, có phí egress, qua public internet)

Với VPC Gateway Endpoint:
EC2 ──── VPC Route Table ──── S3 Endpoint ──── S3
         (lưu lượng trong mạng AWS, không qua internet, FREE)
```

**VPC Endpoint — Điểm Cuối VPC**: kết nối trực tiếp giữa VPC và AWS service, không cần internet gateway hay NAT gateway.

### Gateway Endpoint cho S3

```bash
# Tạo Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxxx \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --route-table-ids rtb-xxxxxxxxx

# Kiểm tra endpoint
aws ec2 describe-vpc-endpoints \
  --filters Name=service-name,Values=com.amazonaws.ap-southeast-1.s3
```

### Bucket Policy Giới Hạn Chỉ Cho VPC Endpoint

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessFromVPCEndpointOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::internal-bucket",
        "arn:aws:s3:::internal-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-xxxxxxxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

### VPC Endpoint Policy — Giới Hạn Truy Cập Bucket Từ Endpoint

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictBucketAccess",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::allowed-bucket",
        "arn:aws:s3:::allowed-bucket/*"
      ]
    },
    {
      "Sid": "DenyOtherBuckets",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "NotResource": [
        "arn:aws:s3:::allowed-bucket",
        "arn:aws:s3:::allowed-bucket/*"
      ]
    }
  ]
}
```

**Dùng VPC Endpoint Policy để:** ngăn exfiltration — rò rỉ dữ liệu sang S3 bucket bên ngoài tổ chức qua EC2.

---

## 5. Mã Hóa In-Transit cho EBS

### EBS Encryption In-Transit

```
EC2 Instance                           EBS Volume
┌────────────┐                         ┌──────────────┐
│ Application│                         │ Data on disk │
│            │── NVMe/iSCSI over ─────▶│ (encrypted)  │
│            │   network (encrypted)   │              │
└────────────┘                         └──────────────┘

Dữ liệu giữa EC2 và EBS được mã hóa tự động
khi EBS encryption bật — không cần cấu hình thêm.
```

**Đặc điểm:**
- Tự động khi bật EBS encryption
- Dùng AES-256
- Không cần cấu hình TLS riêng
- Áp dụng cho cả snapshots transfer

### Kiểm Tra Trạng Thái

```bash
# Xem chi tiết volume (encrypted = true)
aws ec2 describe-volumes \
  --volume-ids vol-xxxxxxxxx \
  --query 'Volumes[*].{ID:VolumeId,Encrypted:Encrypted,KmsKeyId:KmsKeyId}'
```

---

## 6. Mã Hóa In-Transit cho EFS

### EFS Encryption In-Transit với TLS

```
EC2 / ECS Container
┌──────────────────────┐
│  Application         │
│       │              │
│  NFS Client          │
│       │              │
│  amazon-efs-utils    │ ← Tự động setup stunnel
│  stunnel (TLS)       │
└──────────────────────┘
         │
         │ TLS 1.2/1.3
         ▼
┌──────────────────────┐
│  EFS Mount Target    │
│  (TLS termination)   │
└──────────────────────┘
```

### Cách Mount EFS với TLS

```bash
# Cài đặt amazon-efs-utils
sudo yum install -y amazon-efs-utils

# Mount với TLS (khuyến nghị)
sudo mount -t efs -o tls fs-xxxxxxxx:/ /mnt/efs

# Mount với TLS và IAM authentication
sudo mount -t efs -o tls,iam fs-xxxxxxxx:/ /mnt/efs

# Mount với TLS và Access Point
sudo mount -t efs \
  -o tls,accesspoint=fsap-xxxxxxxx \
  fs-xxxxxxxx:/ /mnt/efs

# Thêm vào /etc/fstab cho auto-mount
fs-xxxxxxxx:/ /mnt/efs efs _netdev,tls 0 0
```

### EFS File System Policy — Bắt Buộc TLS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceTLSForEFS",
      "Effect": "Deny",
      "Principal": {"AWS": "*"},
      "Action": "*",
      "Resource": "arn:aws:elasticfilesystem:ap-southeast-1:123456789012:file-system/fs-xxxxxxxx",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": false
        }
      }
    }
  ]
}
```

---

## 7. Best Practices

### Checklist Mã Hóa In-Transit

```
S3:
□ Bucket policy có Deny với aws:SecureTransport: false
□ Bucket policy enforce TLS 1.2 tối thiểu
□ VPC Endpoint tạo cho EC2 cùng VPC truy cập S3
□ AWS Config rule: s3-bucket-ssl-requests-only

EBS:
□ EBS encryption bật (tự động mã hóa in-transit)
□ EBS encryption by default bật ở account level

EFS:
□ Mount options có -o tls
□ EFS file system policy deny non-TLS
□ amazon-efs-utils cài đặt trên tất cả client
```

### AWS Config Rules Tự Động Kiểm Tra

```bash
# Rule kiểm tra S3 có enforce TLS không
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-ssl-requests-only",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_SSL_REQUESTS_ONLY"
    }
  }'
```

### Kiểm Tra Toàn Bộ Bucket

```bash
# Script kiểm tra tất cả bucket có enforce TLS chưa
for bucket in $(aws s3 ls | awk '{print $3}'); do
  policy=$(aws s3api get-bucket-policy --bucket "$bucket" \
    --query Policy --output text 2>/dev/null)
  
  if echo "$policy" | grep -q "SecureTransport"; then
    echo "✅ $bucket — TLS enforced"
  else
    echo "❌ $bucket — TLS NOT enforced"
  fi
done
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: `aws:SecureTransport` condition key hoạt động chính xác như thế nào?**

A: `aws:SecureTransport` là global condition key trả về boolean. Khi client gửi request qua HTTPS, giá trị là `true`. Khi gửi qua HTTP, giá trị là `false`. Trong Deny statement với `"Bool": {"aws:SecureTransport": false}`, AWS từ chối mọi request HTTP. Lưu ý quan trọng: phải đặt trong Deny statement không phải Allow, vì implicit deny đã áp dụng và ta cần explicit deny để ghi đè mọi Allow khác.

---

**Q: Khác biệt giữa VPC Endpoint Gateway và Interface cho S3?**

A: Gateway Endpoint miễn phí, hoạt động qua route table entries, chỉ hỗ trợ S3 và DynamoDB, không hỗ trợ DNS resolution riêng. Interface Endpoint — còn gọi là PrivateLink dùng ENI — Elastic Network Interface trong subnet, có IP private riêng, tốn phí ($0.01/AZ/hour + data processing), hỗ trợ nhiều dịch vụ hơn, và hỗ trợ DNS private. Với S3, Gateway Endpoint là lựa chọn tốt hơn vì miễn phí và đủ dùng cho hầu hết trường hợp.

---

**Q: Tại sao cần VPC Endpoint nếu traffic đã được mã hóa bằng TLS?**

A: TLS bảo vệ nội dung khỏi bị đọc, nhưng không ngăn traffic đi qua internet công cộng. VPC Endpoint giữ traffic hoàn toàn trong mạng AWS: không phí egress data transfer, giảm latency, không phụ thuộc internet gateway hay NAT gateway, và có thể dùng VPC Endpoint Policy để kiểm soát bucket nào được phép truy cập — ngăn data exfiltration sang bucket bên ngoài tổ chức.

---

**Cập Nhật:** 2026-05-16
