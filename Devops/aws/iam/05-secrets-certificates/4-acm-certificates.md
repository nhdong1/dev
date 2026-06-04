# AWS Certificate Manager (ACM) — Vòng Đời Chứng Chỉ TLS

> **ACM** (AWS Certificate Manager — Trình Quản Lý Chứng Chỉ AWS) là dịch vụ cấp phát, triển khai và gia hạn tự động chứng chỉ SSL/TLS (Secure Sockets Layer/Transport Layer Security — Lớp Bảo Mật Socket/Bảo Mật Tầng Vận Chuyển) cho các dịch vụ AWS.

---

## 📚 Mục Lục

1. [Tổng Quan SSL/TLS](#tổng-quan-ssltls)
2. [ACM Là Gì?](#acm-là-gì)
3. [Public vs Private Certificate](#public-vs-private-certificate)
4. [Xác Thực Domain (Domain Validation)](#xác-thực-domain-domain-validation)
5. [Triển Khai Chứng Chỉ](#triển-khai-chứng-chỉ)
6. [Auto-Renewal — Gia Hạn Tự Động](#auto-renewal--gia-hạn-tự-động)
7. [Certificate Transparency — Minh Bạch Chứng Chỉ](#certificate-transparency--minh-bạch-chứng-chỉ)
8. [Giám Sát Và Cảnh Báo](#giám-sát-và-cảnh-báo)
9. [Multi-Region Deployment](#multi-region-deployment)
10. [Bảo Mật Và Best Practices](#bảo-mật-và-best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan SSL/TLS

### TLS Là Gì?

**TLS** (Transport Layer Security — Bảo Mật Tầng Vận Chuyển) là giao thức mã hóa đảm bảo:
- **Confidentiality** (Bảo Mật): dữ liệu được mã hóa, không ai nghe lén được
- **Integrity** (Toàn Vẹn): dữ liệu không bị thay đổi trong quá trình truyền
- **Authentication** (Xác Thực): xác minh server là ai nó tự nhận

### TLS Handshake — Quá Trình Bắt Tay TLS

```
Client (Browser)                    Server (AWS ALB)
      │                                   │
      │─── ClientHello ──────────────────►│
      │    (TLS version, cipher suites)   │
      │                                   │
      │◄── ServerHello ──────────────────-│
      │    (chọn cipher suite)             │
      │                                   │
      │◄── Certificate ──────────────────-│
      │    (cert chứa public key)         │
      │                                   │
      │◄── ServerHelloDone ──────────────-│
      │                                   │
      │ [Client verify certificate]        │
      │   - Kiểm tra CA signature          │
      │   - Kiểm tra domain match          │
      │   - Kiểm tra expiry date           │
      │                                   │
      │─── ClientKeyExchange ────────────►│
      │    (mã hóa premaster secret       │
      │     bằng server public key)       │
      │                                   │
      │─── ChangeCipherSpec ─────────────►│
      │─── Finished ─────────────────────►│
      │◄── ChangeCipherSpec ──────────────│
      │◄── Finished ──────────────────────│
      │                                   │
      │══════ Encrypted Application Data ═│
```

### X.509 Certificate Structure (Cấu Trúc Chứng Chỉ X.509)

```
Certificate:
├── Subject (Chủ Thể): CN=*.example.com, O=Example Corp
├── Issuer (Tổ Chức Phát Hành): CN=Amazon RSA 2048 M01
├── Validity (Hiệu Lực):
│   ├── Not Before: 2026-01-01
│   └── Not After:  2027-01-01
├── Public Key: RSA 2048-bit hoặc EC P-256
├── Subject Alternative Names (SANs — Tên Thay Thế Chủ Thể):
│   ├── DNS: example.com
│   ├── DNS: *.example.com
│   └── DNS: api.example.com
├── Key Usage (Mục Đích Sử Dụng Khóa): Digital Signature, Key Encipherment
└── Extensions:
    ├── Basic Constraints: CA=FALSE
    └── Authority Info Access: OCSP URL, CA Issuers URL
```

---

## ACM Là Gì?

### Vấn Đề ACM Giải Quyết

```
Trước ACM (quản lý cert thủ công):
├── Mua cert từ CA (Certificate Authority) có phí ($100-$1000/năm)
├── Tạo CSR (Certificate Signing Request) thủ công
├── Track ngày hết hạn trong spreadsheet
├── Upload cert lên từng load balancer thủ công
├── Lo lắng cert hết hạn làm production down (lỗi phổ biến!)
└── Khó audit ai đang dùng cert nào

Sau ACM:
├── Cert miễn phí (cho dịch vụ AWS tích hợp)
├── Gia hạn tự động trước 60 ngày hết hạn
├── Deploy cert bằng console/API/Terraform
├── CloudWatch alert khi cert sắp hết hạn
└── Không bao giờ lo hết hạn cert nữa
```

### ACM Không Làm Được Gì?

- ❌ Không export private key (bảo mật — key không rời AWS)
- ❌ Không dùng trực tiếp trên EC2 (chỉ qua Load Balancer/CloudFront/API GW)
- ❌ Không dùng cho on-premises servers
- ❌ Không cấp code signing certificates
- ✅ Dùng được: ALB, NLB, CloudFront, API Gateway, Elastic Beanstalk, AppSync, App Runner

---

## Public vs Private Certificate

### Public Certificate (Chứng Chỉ Công Khai)

```
Đặc Điểm:
├── Miễn phí hoàn toàn
├── Được ký bởi Amazon Trust Services (được browsers tin cậy mặc định)
├── Chỉ dùng được với AWS managed services
├── Không thể export private key
└── Gia hạn tự động

Use Cases:
├── HTTPS cho website công khai
├── API Gateway TLS termination
└── CloudFront distribution
```

### Private Certificate (Chứng Chỉ Riêng Tư)

```
Đặc Điểm:
├── Cần ACM Private CA (CA nội bộ — xem file 5-acm-private-ca.md)
├── Có phí ($0.75/cert/tháng)
├── Có thể export private key (dùng cho EC2, on-prem, containers)
├── Dùng cho internal services, mTLS (mutual TLS — TLS Hai Chiều)
└── Gia hạn có thể tự động hoặc thủ công

Use Cases:
├── Internal microservices communication (mTLS)
├── EC2 instances cần cert riêng
├── On-premises servers trong hybrid cloud
└── Client authentication certificates
```

---

## Xác Thực Domain (Domain Validation)

Trước khi ACM cấp cert, bạn phải chứng minh sở hữu domain.

### Phương Pháp 1: DNS Validation (Khuyến Nghị)

```
Quy Trình:
1. Request certificate cho example.com
2. ACM cung cấp một CNAME record để add vào DNS:
   _acme-challenge.example.com → xyz123.acm-validations.aws.
3. Bạn add record vào Route 53 (hoặc DNS provider khác)
4. ACM tự động kiểm tra DNS record
5. Certificate được cấp (thường trong vài phút)

Ưu Điểm:
├── Gia hạn tự động (ACM kiểm tra DNS record mỗi lần gia hạn)
├── Không cần truy cập email
└── Có thể wildcard cert (*.example.com)

Trong Route 53 (tự động):
ACM tích hợp trực tiếp với Route 53 — click "Create record in Route 53"
là xong, không cần copy-paste thủ công
```

```bash
# Yêu cầu cert và xem DNS validation record
aws acm request-certificate \
  --domain-name "example.com" \
  --validation-method DNS \
  --subject-alternative-names "*.example.com" "api.example.com" \
  --region ap-southeast-1

# Output có CertificateArn, dùng để xem validation record
aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:ap-southeast-1:123456789012:certificate/abc-123" \
  --query "Certificate.DomainValidationOptions"

# Kết quả có ResourceRecord.Name và ResourceRecord.Value để add vào DNS
```

```bash
# Tự động add vào Route 53
HOSTED_ZONE_ID="Z1234567890ABC"
CERT_ARN="arn:aws:acm:ap-southeast-1:123456789012:certificate/abc-123"

# Lấy validation record
VALIDATION=$(aws acm describe-certificate \
  --certificate-arn "$CERT_ARN" \
  --query "Certificate.DomainValidationOptions[0].ResourceRecord")

NAME=$(echo $VALIDATION | jq -r '.Name')
VALUE=$(echo $VALIDATION | jq -r '.Value')

# Add CNAME vào Route 53
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"$NAME\",
        \"Type\": \"CNAME\",
        \"TTL\": 300,
        \"ResourceRecords\": [{\"Value\": \"$VALUE\"}]
      }
    }]
  }"
```

### Phương Pháp 2: Email Validation

```
Quy Trình:
1. Request certificate cho example.com
2. ACM gửi email xác nhận đến:
   - admin@example.com
   - administrator@example.com
   - hostmaster@example.com
   - postmaster@example.com
   - webmaster@example.com
3. Người nhận click link trong email để xác nhận
4. Certificate được cấp

Nhược Điểm:
├── Phải có email server cho domain
├── Gia hạn cần xác nhận email lại (không hoàn toàn tự động)
├── Không hỗ trợ wildcard cert
└── Không thể dùng nếu không kiểm soát email domain
```

### Terraform — Tự Động Hóa Hoàn Toàn

```hcl
# Yêu cầu cert
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com", "api.example.com"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true  # Zero-downtime cert rotation
  }

  tags = {
    Environment = "prod"
  }
}

# Tự động add DNS validation record vào Route 53
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}

# Đợi cert được validate
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

---

## Triển Khai Chứng Chỉ

### Application Load Balancer (ALB)

```bash
# Gắn cert vào ALB HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn "arn:aws:elasticloadbalancing:ap-southeast-1:123:loadbalancer/app/my-alb/xyz" \
  --protocol HTTPS \
  --port 443 \
  --ssl-policy "ELBSecurityPolicy-TLS13-1-2-2021-06" \
  --certificates CertificateArn="arn:aws:acm:ap-southeast-1:123:certificate/abc" \
  --default-actions Type=forward,TargetGroupArn="arn:..."
```

```bash
# Thêm cert bổ sung cho cùng listener (SNI — Server Name Indication)
# SNI cho phép một listener phục vụ nhiều domain với cert khác nhau
aws elbv2 add-listener-certificates \
  --listener-arn "arn:aws:elasticloadbalancing:...:listener/app/..." \
  --certificates CertificateArn="arn:aws:acm:...:certificate/def"
```

### CloudFront Distribution

```json
{
  "ViewerCertificate": {
    "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc-123",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  }
}
```

**Lưu ý quan trọng:** CloudFront chỉ dùng cert từ **us-east-1** (N. Virginia). Dù origin ở Singapore, cert phải ở us-east-1.

### API Gateway

```bash
# Tạo custom domain cho API Gateway
aws apigateway create-domain-name \
  --domain-name "api.example.com" \
  --regional-certificate-arn "arn:aws:acm:ap-southeast-1:123:certificate/abc" \
  --endpoint-configuration types=REGIONAL \
  --security-policy TLS_1_2
```

### Terraform — ALB + ACM Complete Setup

```hcl
# ALB với HTTPS listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# HTTP → HTTPS redirect
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

## Auto-Renewal — Gia Hạn Tự Động

### Cơ Chế Gia Hạn

```
Timeline của ACM Certificate:
│
├── Day 0: Certificate được cấp (hiệu lực 13 tháng)
│
├── Day 300 (60 ngày trước hết hạn):
│   └── ACM bắt đầu quá trình gia hạn tự động
│   └── Kiểm tra DNS CNAME record còn tồn tại không
│
├── Day 330 (30 ngày trước hết hạn):
│   └── CloudWatch metric DaysToExpiry = 30
│   └── ACM gửi email cảnh báo
│
├── Day 380 (nếu DNS validation còn nguyên):
│   └── ACM tự động cấp cert mới
│   └── Load balancer tự động dùng cert mới
│   └── Không cần manual intervention
│
└── Day 395: Cert cũ hết hạn
    └── Cert mới đã active từ trước — không có downtime
```

### Điều Kiện Gia Hạn Tự Động Thành Công

```
DNS Validation:
✅ CNAME record vẫn còn trong DNS
✅ Domain vẫn accessible
✅ Certificate đang được dùng trên AWS resource
✅ Không có DNS propagation issues

Email Validation:
❌ Không tự động — cần click email mỗi lần gia hạn
```

### Kiểm Tra Trạng Thái Gia Hạn

```bash
# Kiểm tra renewal status
aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:ap-southeast-1:123:certificate/abc" \
  --query "Certificate.{Status:Status,RenewalSummary:RenewalSummary,NotAfter:NotAfter}"
```

```json
{
  "Status": "ISSUED",
  "NotAfter": "2027-06-15T12:00:00Z",
  "RenewalSummary": {
    "RenewalStatus": "SUCCESS",
    "UpdatedAt": "2027-04-15T08:00:00Z",
    "DomainValidationOptions": [
      {
        "DomainName": "example.com",
        "ValidationStatus": "SUCCESS"
      }
    ]
  }
}
```

### Trạng Thái Gia Hạn (Renewal Status)

| Status | Ý Nghĩa | Hành Động |
|---|---|---|
| `PENDING_AUTO_RENEWAL` | Đang chờ kiểm tra | Bình thường |
| `PENDING_VALIDATION` | Cần xác nhận DNS/email | Kiểm tra DNS record |
| `SUCCESS` | Gia hạn thành công | Không cần làm gì |
| `FAILED` | Thất bại | Kiểm tra DNS record, tạo cert mới |

---

## Certificate Transparency — Minh Bạch Chứng Chỉ

**Certificate Transparency (CT — Minh Bạch Chứng Chỉ)** là yêu cầu bắt buộc: mọi public certificate phải được ghi vào public CT logs (nhật ký công khai). Điều này cho phép phát hiện cert giả mạo.

```
Lợi Ích của CT:
├── Bạn có thể monitor (giám sát) ai cấp cert cho domain của bạn
├── Phát hiện cert được cấp mà không có sự đồng ý
├── Bắt buộc từ tháng 4/2018 trên tất cả browsers
└── Google, Cloudflare, DigiCert đều có CT log server

Công Cụ Monitor CT:
├── crt.sh — tìm kiếm cert theo domain
├── Facebook Certificate Transparency Monitoring
└── AWS Certificate Manager tự động log lên CT
```

```bash
# Kiểm tra CT logging status của cert
aws acm describe-certificate \
  --certificate-arn "arn:..." \
  --query "Certificate.Options.CertificateTransparencyLoggingPreference"

# Nếu cần, tắt CT logging (không khuyến nghị cho public cert)
aws acm update-certificate-options \
  --certificate-arn "arn:..." \
  --options CertificateTransparencyLoggingPreference=DISABLED
```

---

## Giám Sát Và Cảnh Báo

### CloudWatch Metrics

| Metric | Namespace | Ý Nghĩa |
|---|---|---|
| `DaysToExpiry` | `AWS/CertificateManager` | Số ngày còn lại trước hết hạn |

### Alarm Cho Certificate Sắp Hết Hạn

```bash
# Cảnh báo khi cert còn 45 ngày
aws cloudwatch put-metric-alarm \
  --alarm-name "ACM-Certificate-Expiry-45days" \
  --alarm-description "ACM cert sắp hết hạn trong 45 ngày" \
  --metric-name DaysToExpiry \
  --namespace AWS/CertificateManager \
  --dimensions Name=CertificateArn,Value="arn:aws:acm:ap-southeast-1:123:certificate/abc" \
  --statistic Minimum \
  --period 86400 \
  --threshold 45 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:ap-southeast-1:123:security-alerts" \
  --treat-missing-data missing
```

### EventBridge Rule Cho ACM Events

```json
{
  "source": ["aws.acm"],
  "detail-type": ["ACM Certificate Approaching Expiration"],
  "detail": {
    "DaysToExpiry": [{"numeric": ["<=", 30]}]
  }
}
```

ACM tự động gửi event khi:
- Certificate sắp hết hạn (30, 45, 60 ngày)
- Gia hạn thất bại
- Certificate bị xóa

### AWS Config Rule

```
Managed Rule: acm-certificate-expiration-check
Mô Tả: Kiểm tra cert sẽ hết hạn trong N ngày
Parameter: daysToExpiration = 30
Action: Tạo Config finding, có thể trigger SNS
```

---

## Multi-Region Deployment

### Vấn Đề: ACM Cert Bị Region-Lock

Mỗi ACM cert chỉ có thể dùng ở region nó được tạo. Ngoại lệ: **CloudFront chỉ dùng cert từ us-east-1**.

```
Ứng Dụng Multi-Region:

ap-southeast-1 (Singapore):
└── ALB ap-southeast-1
    └── ACM cert ap-southeast-1: *.example.com

ap-northeast-1 (Tokyo):
└── ALB ap-northeast-1
    └── ACM cert ap-northeast-1: *.example.com  ← Cert khác nhau, domain giống nhau

us-east-1 (N. Virginia):
└── CloudFront Distribution
    └── ACM cert us-east-1: *.example.com  ← BẮT BUỘC ở us-east-1
```

### Terraform — Multi-Region Certificate

```hcl
# Provider cho từng region
provider "aws" {
  alias  = "ap-southeast-1"
  region = "ap-southeast-1"
}

provider "aws" {
  alias  = "us-east-1"
  region = "us-east-1"
}

# Cert cho ALB Singapore
resource "aws_acm_certificate" "singapore" {
  provider          = aws.ap-southeast-1
  domain_name       = "*.example.com"
  validation_method = "DNS"
}

# Cert cho CloudFront (BẮT BUỘC us-east-1)
resource "aws_acm_certificate" "cloudfront" {
  provider          = aws.us-east-1
  domain_name       = "*.example.com"
  validation_method = "DNS"
}
```

---

## TLS Policy — Phiên Bản TLS Được Hỗ Trợ

### SSL Policy Cho ALB

| Policy | TLS Versions | Cipher Suites | Khuyến Nghị |
|---|---|---|---|
| `ELBSecurityPolicy-TLS13-1-3-2021-06` | TLS 1.3 only | TLS 1.3 ciphers | Bảo mật cao nhất |
| `ELBSecurityPolicy-TLS13-1-2-2021-06` | TLS 1.2, 1.3 | Tốt | **Khuyến nghị** |
| `ELBSecurityPolicy-2016-08` | TLS 1.0-1.3 | Đầy đủ | Legacy, không dùng |

```bash
# Cập nhật TLS policy cho listener
aws elbv2 modify-listener \
  --listener-arn "arn:..." \
  --ssl-policy "ELBSecurityPolicy-TLS13-1-2-2021-06"
```

### Tại Sao Không Dùng TLS 1.0/1.1?

- **TLS 1.0** (1999): Có nhiều lỗ hổng (BEAST, POODLE), PCI-DSS 3.2 cấm từ 2018
- **TLS 1.1** (2006): Bị deprecated (lỗi thời) bởi RFC 8996 năm 2021
- **TLS 1.2** (2008): Vẫn an toàn với cipher suite đúng
- **TLS 1.3** (2018): Nhanh hơn (1-RTT), bảo mật hơn, loại bỏ cipher suite yếu

---

## Bảo Mật Và Best Practices

### Certificate Pinning — Gắn Chứng Chỉ (Không Khuyến Nghị)

```
Certificate Pinning: ứng dụng mobile chỉ tin chứng chỉ cụ thể

Vấn Đề Với ACM + Cert Pinning:
- ACM tự động thay cert khi gia hạn
- Cert mới có public key khác
- App mobile đang dùng cert cũ → từ chối kết nối
- Phải release app update để update pin

Khuyến Nghị: Dùng public key pinning (pin CA public key, không pin leaf cert)
Hoặc: Không dùng cert pinning với ACM
```

### Wildcard vs Multi-SAN Certificate

```
Wildcard Certificate (*.example.com):
✅ Một cert cho tất cả subdomains
✅ Dễ quản lý
❌ Không cover example.com (chỉ *.example.com)
❌ Không cover sub.sub.example.com (chỉ một cấp)
❌ Nếu cert bị compromise → tất cả subdomains bị ảnh hưởng

Multi-SAN Certificate:
✅ Chỉ định chính xác domains được phép
✅ An toàn hơn (scope hẹp hơn)
✅ Cover cả example.com và sub.example.com
❌ Phải liệt kê từng domain

ACM Best Practice:
- Dùng cả *.example.com VÀ example.com trong SAN
- Tạo separate cert cho critical domains nếu cần isolation
```

### IAM Permission Cho ACM

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ViewCertificates",
      "Effect": "Allow",
      "Action": [
        "acm:ListCertificates",
        "acm:DescribeCertificate",
        "acm:GetCertificate"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ManageCertificates",
      "Effect": "Allow",
      "Action": [
        "acm:RequestCertificate",
        "acm:DeleteCertificate",
        "acm:AddTagsToCertificate"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["ap-southeast-1", "us-east-1"]
        }
      }
    }
  ]
}
```

---

## Câu Hỏi Phỏng Vấn

**Q1: ACM certificate gia hạn tự động như thế nào? Cần điều kiện gì?**

> ACM bắt đầu gia hạn khoảng 60 ngày trước hết hạn. Với DNS validation, ACM kiểm tra xem CNAME validation record vẫn còn trong DNS không — nếu có, cert tự động gia hạn không cần can thiệp. Với email validation, người dùng phải click link trong email mỗi lần gia hạn. Đây là lý do DNS validation được khuyến nghị cho production.

**Q2: Tại sao CloudFront bắt buộc dùng cert từ us-east-1?**

> CloudFront là global service với edge locations (điểm hiện diện) trên khắp thế giới, nhưng control plane (mặt phẳng kiểm soát) ở us-east-1. Khi bạn associate cert, CloudFront phân phối cert đó ra toàn bộ edge network từ us-east-1. Cert ở region khác không thể được phân phối theo cách này.

**Q3: Wildcard cert có những hạn chế gì?**

> Wildcard `*.example.com` chỉ cover một cấp subdomain — không cover `example.com` (phải add vào SAN) và không cover `api.sub.example.com` (hai cấp). Ngoài ra, wildcard cert rủi ro hơn: nếu bị compromise (bị đánh cắp), tất cả subdomains bị ảnh hưởng. ACM cho phép kết hợp: request cert với SAN cả `example.com` và `*.example.com`.

**Q4: Làm sao giám sát cert sắp hết hạn ở scale lớn (100+ certs)?**

> Dùng AWS Config Rule `acm-certificate-expiration-check` với `daysToExpiration=45` — tự động báo cáo tất cả cert sắp hết hạn trong toàn bộ account/organization. Kết hợp với EventBridge để nhận event `ACM Certificate Approaching Expiration` và gửi alert qua SNS. CloudWatch metric `DaysToExpiry` cho phép tạo alarm per-certificate.

**Q5: Tại sao không thể dùng ACM cert trực tiếp trên EC2?**

> ACM không cho export private key — đây là tính năng bảo mật cố ý. Private key của public cert được lưu trong AWS HSM (Hardware Security Module) và không bao giờ rời khỏi AWS infrastructure. Để dùng TLS trên EC2, bạn cần: (1) Đặt EC2 sau ALB và terminate TLS ở ALB, hoặc (2) Dùng ACM Private CA để issue cert và export nó (ACM Private CA cho phép export).

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
