# Origin Access — OAC, OAI & Bảo Vệ S3 Bucket

> **OAC** (Origin Access Control — Kiểm Soát Truy Cập Nguồn Gốc) và **OAI** (Origin Access Identity — Danh Tính Truy Cập Nguồn Gốc) là cơ chế bảo vệ S3 bucket khỏi truy cập trực tiếp từ internet, chỉ cho phép CloudFront fetch nội dung. OAC là thế hệ mới hơn và là lựa chọn được AWS khuyến nghị hiện tại.

---

## 📚 Mục Lục

1. [Vấn Đề Cần Giải Quyết](#1-vấn-đề-cần-giải-quyết)
2. [OAI — Origin Access Identity (Thế Hệ Cũ)](#2-oai--origin-access-identity-thế-hệ-cũ)
3. [OAC — Origin Access Control (Thế Hệ Mới)](#3-oac--origin-access-control-thế-hệ-mới)
4. [So Sánh OAC vs OAI](#4-so-sánh-oac-vs-oai)
5. [Cấu Hình OAC Từng Bước](#5-cấu-hình-oac-từng-bước)
6. [Bảo Vệ Custom Origin (ALB/EC2)](#6-bảo-vệ-custom-origin-albec2)
7. [Signed URLs & Signed Cookies](#7-signed-urls--signed-cookies)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Cần Giải Quyết

### S3 Bucket Không OAC — Rủi Ro Bảo Mật

```
Tình huống sai:
  S3 Bucket: my-private-assets (phải để Public)
    ↓ Direct access (không qua CloudFront)
  Người dùng → https://my-private-assets.s3.amazonaws.com/secret.pdf
               ✅ Download thành công — BỎ QUA CloudFront hoàn toàn!

Hậu quả:
  - Bypass WAF rules đặt trên CloudFront
  - Bypass Signed URLs/Cookies
  - Bypass Geo-restriction
  - Lộ origin server URL thực tế
  - Tốn băng thông S3 trực tiếp (đắt hơn CloudFront)
```

### Mục Tiêu

```
S3 Bucket: my-private-assets (PRIVATE — không public)
  ↓ Chỉ CloudFront mới được phép access
CloudFront → (dùng OAC/OAI để authenticate) → S3

Kết quả:
  - Bucket private hoàn toàn
  - Mọi truy cập phải đi qua CloudFront
  - WAF, Signed URLs, Geo-restriction hoạt động hiệu quả
```

---

## 2. OAI — Origin Access Identity (Thế Hệ Cũ)

### OAI Là Gì?

**OAI** (Origin Access Identity — Danh Tính Truy Cập Nguồn Gốc) là một IAM identity đặc biệt đại diện cho CloudFront Distribution. S3 bucket policy cho phép OAI này đọc files.

### Cách OAI Hoạt Động

```
1. Tạo OAI: CloudFront → Lấy identity ARN
   (ví dụ: arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity EXXXXX)

2. Gán OAI cho Distribution → S3 Origin

3. Cập nhật S3 Bucket Policy:
   Principal: { "CanonicalUser": "<OAI S3CanonicalUserId>" }
   Action: s3:GetObject
   Resource: arn:aws:s3:::my-bucket/*

4. S3 Bucket: Block all public access ✅

5. Request flow:
   User → CloudFront → S3 (authenticated với OAI) ✅
   User → S3 (direct) → Access Denied 403 ✅
```

### S3 Bucket Policy Với OAI

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAI",
      "Effect": "Allow",
      "Principal": {
        "CanonicalUser": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-private-bucket/*"
    }
  ]
}
```

### Hạn Chế Của OAI

- **Không hỗ trợ SSE-KMS** (Server-Side Encryption với KMS — Mã Hóa Phía Server): OAI dùng legacy auth → không thể decrypt KMS-encrypted objects
- **Không hỗ trợ POST/PUT methods**: Chỉ đọc (GET/HEAD)
- **Một OAI cho một Distribution**: Không linh hoạt
- **Sắp bị deprecated**: AWS đã ngừng đề xuất OAI cho deployments mới

---

## 3. OAC — Origin Access Control (Thế Hệ Mới)

### OAC Là Gì?

**OAC** (Origin Access Control — Kiểm Soát Truy Cập Nguồn Gốc) là cơ chế xác thực thế hệ mới, dùng **AWS Signature Version 4 (SigV4)** để ký request từ CloudFront đến origin. Thay vì dùng IAM identity cố định như OAI, OAC sign từng request với credentials của CloudFront service.

### Cách OAC Hoạt Động

```
1. Tạo OAC trong CloudFront (cấu hình signing behavior)

2. Gán OAC cho S3 Origin trong Distribution

3. S3 Bucket Policy: cho phép CloudFront Service Principal:
   Principal: { "Service": "cloudfront.amazonaws.com" }
   Condition: StringEquals aws:SourceArn → Distribution ARN

4. Khi có request:
   User → CloudFront Edge
   CloudFront → Sign request với SigV4 (OAC)
   CloudFront → S3: GET /my-file.jpg
                    Authorization: AWS4-HMAC-SHA256 Credential=...
   S3: Verify signature → trả file

5. Request flow an toàn:
   User → CloudFront → S3 ✅
   User → S3 direct → Access Denied ✅
```

### Signing Behaviors (Hành Vi Ký)

```
Sign requests: Always (Luôn ký — khuyến nghị)
  → CloudFront luôn thêm SigV4 authorization header

Sign requests: Never (Không ký)
  → Giống không có OAC — dùng cho public bucket
  → Không có ý nghĩa bảo mật

Sign requests: No override (Không ghi đè)
  → Ký chỉ khi origin không có auth riêng
```

### S3 Bucket Policy Với OAC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-private-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

**Điểm mấu chốt:** Condition `AWS:SourceArn` ràng buộc policy chỉ cho phép Distribution cụ thể — không phải mọi CloudFront distribution của AWS.

### OAC Với SSE-KMS

```
S3 bucket dùng SSE-KMS encryption:

1. OAC sign request → S3 authenticate CloudFront ✅
2. S3 decrypt object dùng KMS → cần KMS permission cho CloudFront

KMS Key Policy (thêm):
{
  "Sid": "AllowCloudFrontServicePrincipal",
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
    }
  }
}

→ OAI không làm được điều này — đây là lý do OAC ra đời
```

---

## 4. So Sánh OAC vs OAI

| Tiêu Chí                         | OAI (Cũ)              | OAC (Mới — Khuyến Nghị)       |
| -------------------------------- | --------------------- | ------------------------------ |
| **Authentication method**        | Legacy S3 auth        | AWS Signature Version 4 (SigV4) |
| **SSE-KMS support**              | ❌ Không hỗ trợ        | ✅ Hỗ trợ đầy đủ               |
| **HTTP POST/PUT support**        | ❌ Chỉ GET/HEAD        | ✅ Hỗ trợ mọi methods          |
| **S3 bucket policy**             | CanonicalUser         | Service Principal + Condition  |
| **Một OAC nhiều distributions**  | ❌ 1:1                 | ✅ 1 OAC cho nhiều distributions |
| **Trạng thái AWS**               | Legacy (sắp deprecated) | ✅ Recommended                 |
| **Non-S3 origins**               | ❌ Chỉ S3              | ✅ S3, MediaStore, API Gateway  |
| **Dễ cấu hình**                  | Tương đương            | Tương đương                    |

**Quyết định:** Luôn dùng OAC cho deployments mới. Migrate OAI → OAC nếu cần SSE-KMS.

---

## 5. Cấu Hình OAC Từng Bước

### Bước 1: Tạo S3 Bucket (Private)

```bash
# Tạo bucket
aws s3 mb s3://my-private-assets --region us-east-1

# Block all public access (quan trọng!)
aws s3api put-public-access-block \
  --bucket my-private-assets \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,\
     BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### Bước 2: Tạo OAC

```bash
# Tạo OAC
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-s3-oac",
    "Description": "OAC for my-private-assets bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'

# Lưu lại OAC ID từ output
```

### Bước 3: Tạo/Update Distribution

```bash
# Trong distribution config, set OriginAccessControlId:
{
  "Origins": {
    "Items": [{
      "Id": "S3-my-private-assets",
      "DomainName": "my-private-assets.s3.us-east-1.amazonaws.com",
      "OriginAccessControlId": "E2QWRUHXXXXXXXX",  # OAC ID
      "S3OriginConfig": {
        "OriginAccessIdentity": ""  # Để trống khi dùng OAC
      }
    }]
  }
}
```

### Bước 4: Cập Nhật S3 Bucket Policy

```bash
# Tạo file policy.json
cat > policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-private-assets/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
EOF

# Áp dụng policy
aws s3api put-bucket-policy \
  --bucket my-private-assets \
  --policy file://policy.json
```

### Bước 5: Kiểm Tra

```bash
# Test qua CloudFront — phải hoạt động
curl -I https://d1234abcd.cloudfront.net/test.jpg
# → HTTP/2 200

# Test trực tiếp S3 — phải bị chặn
curl -I https://my-private-assets.s3.amazonaws.com/test.jpg
# → HTTP/1.1 403 Forbidden
```

### Terraform — OAC Full Setup

```hcl
# S3 Bucket (private)
resource "aws_s3_bucket" "assets" {
  bucket = "my-private-assets"
}

resource "aws_s3_bucket_public_access_block" "assets" {
  bucket                  = aws_s3_bucket.assets.id
  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}

# OAC
resource "aws_cloudfront_origin_access_control" "main" {
  name                              = "my-s3-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Distribution
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3Origin"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }
  # ... (các cấu hình khác)
}

# S3 Bucket Policy
resource "aws_s3_bucket_policy" "assets" {
  bucket = aws_s3_bucket.assets.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "AllowCloudFrontOAC"
      Effect = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action   = "s3:GetObject"
      Resource = "${aws_s3_bucket.assets.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
        }
      }
    }]
  })
}
```

---

## 6. Bảo Vệ Custom Origin (ALB/EC2)

S3 không phải origin duy nhất cần bảo vệ. ALB và EC2 cũng có thể bị bypass nếu người dùng biết DNS name trực tiếp.

### Bảo Vệ ALB Bằng Custom Header

```
Cách hoạt động:
  1. CloudFront thêm Custom Header bí mật vào mọi request đến origin:
     X-Origin-Verify: <random-secret-value-32-chars>

  2. ALB Listener Rule kiểm tra header:
     IF x-origin-verify = <secret> → forward to target group
     ELSE → Return 403

  3. Người dùng bypass CloudFront → không có header → 403
```

```bash
# Cấu hình trong Distribution Origin Settings:
{
  "CustomHeaders": {
    "Quantity": 1,
    "Items": [{
      "HeaderName": "X-Origin-Verify",
      "HeaderValue": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"  # Random, bí mật
    }]
  }
}
```

```
ALB Listener Rule (via Console hoặc Terraform):
  Condition: HTTP header x-origin-verify = a1b2c3d4...
  Action: Forward to target group

  Default action (khi không match): Return 403
```

**Bảo mật thêm:** Rotate secret định kỳ. Lưu trong AWS Secrets Manager.

### Bảo Vệ ALB Bằng Security Group

```
Security Group của ALB:
  Inbound Rule:
    Port: 443
    Source: CloudFront managed prefix list
    (com.amazonaws.global.cloudfront.origin-facing)

→ Chặn mọi traffic trực tiếp không từ CloudFront IP ranges
→ CloudFront IP ranges tự động update bởi AWS
```

```hcl
# Terraform: SG chỉ cho phép CloudFront
data "aws_ec2_managed_prefix_list" "cloudfront" {
  name = "com.amazonaws.global.cloudfront.origin-facing"
}

resource "aws_security_group_rule" "alb_cloudfront" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  prefix_list_ids   = [data.aws_ec2_managed_prefix_list.cloudfront.id]
  security_group_id = aws_security_group.alb.id
}
```

---

## 7. Signed URLs & Signed Cookies

Khi muốn kiểm soát truy cập **từng file** hoặc **nhóm file** theo user/thời gian.

### Signed URL (URL Đã Ký)

```
Dùng khi:
  - Restrict access đến từng file cụ thể
  - Link download có thời hạn
  - RTMP streaming (cũ)

Ví dụ URL:
https://d1234.cloudfront.net/video.mp4
  ?Policy=base64encoded...
  &Signature=sha1hash...
  &Key-Pair-Id=APKA...

Hết hạn sau: 15 phút (configurable)
Chỉ cho: 1 file (video.mp4)
```

### Signed Cookie (Cookie Đã Ký)

```
Dùng khi:
  - Restrict access đến nhiều files/thư mục
  - Streaming video với nhiều segments
  - User đã authenticate → access nhiều resources

Cookie được set:
  CloudFront-Policy: base64encoded...
  CloudFront-Signature: sha1hash...
  CloudFront-Key-Pair-Id: APKA...

Phạm vi: Tất cả files khớp với path pattern
```

### So Sánh Signed URL vs Signed Cookie

| Tiêu Chí              | Signed URL                    | Signed Cookie                        |
| --------------------- | ----------------------------- | ------------------------------------ |
| **Phạm vi**           | 1 file                        | Nhiều files/patterns                 |
| **Ứng dụng**          | Download link                 | Video streaming, premium content     |
| **URL visible?**      | Có — token trong URL          | Không — token trong cookie (ẩn hơn)  |
| **Mobile app**        | Dễ dùng                       | Phức tạp hơn (cookie management)     |
| **Streaming HLS**     | Khó (mỗi segment cần URL riêng) | Phù hợp (1 cookie cho tất cả)       |

### Cách Tạo Signed URL (Python)

```python
import boto3
from botocore.signers import CloudFrontSigner
import datetime
import rsa

def create_signed_url(url, key_id, private_key_path, expire_minutes=15):
    # Đọc private key
    with open(private_key_path, 'rb') as f:
        private_key = rsa.PrivateKey.load_pkcs1(f.read())

    def rsa_signer(message):
        return rsa.sign(message, private_key, 'SHA-1')

    expire_time = datetime.datetime.utcnow() + datetime.timedelta(minutes=expire_minutes)

    cloudfront_signer = CloudFrontSigner(key_id, rsa_signer)

    signed_url = cloudfront_signer.generate_presigned_url(
        url,
        date_less_than=expire_time
    )
    return signed_url

# Sử dụng
url = create_signed_url(
    url="https://d1234.cloudfront.net/premium/video.mp4",
    key_id="APKAXXXXXXXXXXX",
    private_key_path="/secrets/cloudfront-private-key.pem",
    expire_minutes=60
)
```

### Key Groups & Trusted Key Groups

```
Luồng setup Signed URLs:
  1. Tạo CloudFront Key Pair (RSA 2048-bit):
     - Public key: upload lên CloudFront
     - Private key: lưu an toàn (Secrets Manager)

  2. Tạo Key Group với public key

  3. Trong Cache Behavior:
     Restrict Viewer Access: Yes
     Trusted Key Groups: [my-key-group]

  4. Backend server: tạo Signed URL dùng private key

  5. User nhận Signed URL từ backend → access CloudFront
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: OAC khác OAI như thế nào và khi nào dùng OAC?

**Trả lời mẫu:**
> "OAI là cơ chế cũ dùng legacy S3 authentication — đại diện bởi một IAM canonical user. OAC là thế hệ mới dùng SigV4 để ký từng request, giống cách IAM role ký API calls. Sự khác biệt quan trọng nhất: OAC hỗ trợ SSE-KMS encrypted S3 buckets vì CloudFront có thể authenticate để decrypt; OAI thì không. AWS khuyến nghị dùng OAC cho mọi deployment mới."

### Q2: Làm thế nào để bảo vệ S3 bucket chỉ cho phép CloudFront truy cập?

**Trả lời:**
> "Tôi dùng OAC: Đầu tiên, bật Block All Public Access trên S3 bucket. Sau đó tạo OAC trong CloudFront và gán cho S3 origin của Distribution. Cuối cùng, cập nhật S3 bucket policy với Principal là cloudfront.amazonaws.com và Condition là AWS:SourceArn trỏ đến Distribution ARN cụ thể. Condition này quan trọng — nó đảm bảo chỉ Distribution của tôi được phép, không phải mọi CloudFront distribution của AWS."

### Q3: Khi nào dùng Signed URL vs Signed Cookie?

**Trả lời:**
> "Signed URL cho một file cụ thể — phù hợp với download link có thời hạn, chia sẻ file đơn lẻ. Signed Cookie cho nhiều files — phù hợp với video streaming HLS vì player cần access nhiều .ts segments và manifest files trong cùng session. Cookie cho phép dùng một bộ credentials cho toàn bộ content, trong khi Signed URL phải tạo riêng cho mỗi file."

### Q4: Nếu user biết S3 URL trực tiếp, họ có thể bypass CloudFront không?

**Trả lời:**
> "Nếu cấu hình đúng với OAC thì không. S3 bucket policy chỉ cho phép cloudfront.amazonaws.com với SourceArn là Distribution ARN. Request trực tiếp từ user đến S3 không có SigV4 signature của CloudFront → S3 trả 403 Access Denied. Đây là lý do tôi không bao giờ để S3 bucket public khi dùng CloudFront — nếu không, OAC trở nên vô nghĩa."

---

**Tiếp Theo:** [4-lambda-edge.md](./4-lambda-edge.md) — Lambda@Edge và CloudFront Functions
