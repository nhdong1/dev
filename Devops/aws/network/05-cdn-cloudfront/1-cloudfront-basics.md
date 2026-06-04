# CloudFront Basics — Distributions, Edge Locations & Origins

> Tài liệu này đi sâu vào các khái niệm nền tảng của Amazon CloudFront: cách tạo Distribution (Phân Phối), hiểu Edge Location (Điểm Biên), cấu hình Origins (Nguồn Gốc), và cách CloudFront xử lý request từ người dùng.

---

## 📚 Mục Lục

1. [CloudFront Distribution](#1-cloudfront-distribution)
2. [Edge Locations & PoP](#2-edge-locations--pop)
3. [Origins — Các Loại Nguồn Gốc](#3-origins--các-loại-nguồn-gốc)
4. [Tạo Distribution Đầu Tiên](#4-tạo-distribution-đầu-tiên)
5. [SSL/TLS & Custom Domain](#5-ssltls--custom-domain)
6. [Geo-Restriction](#6-geo-restriction)
7. [Thực Hành Hands-On](#7-thực-hành-hands-on)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. CloudFront Distribution

### Distribution Là Gì?

**Distribution** (Phân Phối) là đơn vị cấu hình trung tâm của CloudFront. Khi bạn tạo một Distribution, AWS sẽ:

1. Cấp cho bạn một **domain name tự động**: `d1234abcdefgh.cloudfront.net`
2. Cấu hình routing trên toàn bộ mạng lưới Edge Locations
3. Áp dụng các quy tắc cache, bảo mật, và compression theo cấu hình

### Hai Loại Distribution (Lịch Sử)

> **Lưu ý 2024:** AWS đã hợp nhất — chỉ còn một loại Distribution chung. RTMP Distribution đã bị deprecated (ngừng hỗ trợ).

| Loại            | Trạng Thái    | Mô Tả                                                |
| --------------- | ------------- | ---------------------------------------------------- |
| **Web**         | ✅ Hiện tại   | HTTP/HTTPS — dùng cho mọi trường hợp hiện đại        |
| **RTMP**        | ❌ Deprecated | Real-Time Messaging Protocol — streaming cũ; dùng HLS thay thế |

### Anatomy of a Distribution (Cấu Trúc Distribution)

```yaml
Distribution:
  Domain: d1234abcdefgh.cloudfront.net
  Alternate Names (CNAMEs): [cdn.example.com]
  SSL Certificate: ACM Certificate ARN
  HTTP Version: HTTP/2, HTTP/3
  Price Class: PriceClass_All
  WAF: Optional WebACL ARN
  Logging: S3 bucket for access logs

  Origins:
    - S3 Bucket (my-assets.s3.amazonaws.com)
    - ALB (my-alb.elb.amazonaws.com)

  Cache Behaviors:
    - Default (*): → S3 Origin
    - /api/*: → ALB Origin (no cache)
    - /images/*: → S3 Origin (cache 24h)
```

---

## 2. Edge Locations & PoP

### Mạng Lưới CloudFront (2024)

```
Châu Á — Thái Bình Dương:
  Singapore, Tokyo, Osaka, Seoul, Mumbai, Sydney,
  Jakarta, Kuala Lumpur, Bangkok, Manila, Taipei...

Châu Âu:
  Frankfurt, London, Paris, Amsterdam, Stockholm,
  Madrid, Milan, Warsaw, Vienna, Dublin...

Bắc Mỹ:
  ~50+ điểm tại US, Canada, Mexico

Nam Mỹ:
  São Paulo, Buenos Aires, Bogotá, Santiago...

Trung Đông & Châu Phi:
  Dubai, Tel Aviv, Cape Town, Nairobi...

Tổng cộng: 600+ Edge Locations tại 90+ thành phố, 47+ quốc gia
```

### Hai Tầng Cache

```
┌────────────────────────────────────────────────────────┐
│  Tầng 1: Edge Location (PoP — Point of Presence)       │
│  • Gần người dùng nhất                                 │
│  • Cache dung lượng nhỏ, TTL ngắn hơn                  │
│  • Hàng trăm PoP toàn cầu                              │
└─────────────────────────┬──────────────────────────────┘
                          │ Cache miss
                          ▼
┌────────────────────────────────────────────────────────┐
│  Tầng 2: Regional Edge Cache (REC)                     │
│  • ~13 điểm, 1-2 per AWS Region                        │
│  • Cache dung lượng lớn hơn (TB range)                 │
│  • Giữ nội dung lâu hơn → giảm tải origin              │
└─────────────────────────┬──────────────────────────────┘
                          │ Vẫn miss
                          ▼
                    Origin Server
```

### Tại Sao Có Regional Edge Cache?

Bài toán: Hàng trăm PoP không thể tất cả cache mọi thứ. Nếu PoP Singapore miss, thay vì bay thẳng đến Origin ở US-East (150ms+), request đi đến Regional Edge Cache ở Singapore (~20ms) — cache hit rate tổng thể tăng đáng kể.

### Anycast Routing (Định Tuyến Anycast)

CloudFront dùng **Anycast** — nhiều Edge Location dùng cùng một dải IP. Khi người dùng query DNS, mạng internet tự động định tuyến đến Edge Location **địa lý gần nhất**, không cần Route 53 can thiệp thêm.

```
Người dùng HN → DNS d1234.cloudfront.net
             ← IP: 54.230.x.x (IP anycast của CloudFront)
             → TCP connect đến 54.230.x.x
             → Mạng internet tự route đến Singapore PoP
```

---

## 3. Origins — Các Loại Nguồn Gốc

### 3.1 S3 Bucket Origin

Phổ biến nhất — dùng cho static assets.

```
Cấu hình:
  Origin Domain: my-bucket.s3.us-east-1.amazonaws.com
  Origin Path: /static (optional prefix)
  Protocol: HTTPS only (khuyến nghị)
  Origin Access: OAC (Origin Access Control) — xem file 3-origin-access.md
```

**Best Practice:** Luôn dùng OAC — không bao giờ để S3 bucket public chỉ vì CloudFront.

### 3.2 ALB (Application Load Balancer) Origin

Dùng cho dynamic content, web applications.

```
Cấu hình:
  Origin Domain: my-alb.ap-southeast-1.elb.amazonaws.com
  Protocol: HTTPS only
  Custom Headers: X-CloudFront-Secret: <random-value>
                  (ALB chỉ accept request có header này)
  Timeouts:
    Connection: 10s (default)
    Response: 30s (default, max 60s)
```

**Bảo vệ ALB:** ALB security group chỉ cho phép traffic từ CloudFront IP ranges (`aws.json` prefix list).

### 3.3 Custom HTTP Origin

Bất kỳ server HTTP/HTTPS nào — on-premises, external APIs.

```
Cấu hình:
  Origin Domain: api.thirdparty.com
  Protocol: HTTPS only
  Minimum SSL: TLSv1.2
  Custom Port: 8443 (nếu không dùng 443)
```

**Hạn chế:** Không có OAC — bảo vệ bằng custom header bí mật hoặc IP allowlist.

### 3.4 Origin Groups — Failover Tự Động

**Origin Group** cho phép cấu hình failover tự động giữa hai origins:

```
Primary Origin: S3 Bucket us-east-1
Fallback Origin: S3 Bucket eu-west-1

Failover Khi: HTTP 5xx errors, connection timeout
→ CloudFront tự động retry với Fallback Origin
```

```
┌──────────────────────────────────────────┐
│           Origin Group                   │
│                                          │
│  Primary: s3-us-east-1                   │
│     │ Nếu trả về 500/503/504...          │
│     ▼                                    │
│  Failover: s3-eu-west-1                  │
└──────────────────────────────────────────┘
```

### 3.5 Multiple Origins — Cấu Hình Thực Tế

```
Distribution: cdn.example.com
│
├── Origin 1: S3 (static assets)
│   └── Cache Behavior: /static/* → cache 7 ngày
│
├── Origin 2: ALB (API)
│   └── Cache Behavior: /api/* → no cache, forward all headers
│
└── Origin 3: S3 (media files)
    └── Cache Behavior: /media/* → cache 30 ngày
```

---

## 4. Tạo Distribution Đầu Tiên

### Qua AWS Console — Step by Step

```
1. CloudFront Console → Create Distribution

2. Origin Settings:
   Origin domain: my-bucket.s3.amazonaws.com
   Origin access: Origin access control (OAC) → Create new
   Viewer protocol policy: Redirect HTTP to HTTPS

3. Default Cache Behavior:
   Allowed HTTP methods: GET, HEAD
   Cache policy: CachingOptimized (managed)
   Origin request policy: CORS-S3Origin (nếu cần CORS)
   Compress objects: Yes

4. Distribution Settings:
   Price class: Use all edge locations (best performance)
   Alternate domain name: cdn.example.com
   Custom SSL certificate: *.example.com (ACM)
   HTTP/2: Enabled
   HTTP/3: Enabled (QUIC)
   Default root object: index.html

5. → Create Distribution (deploy 5-10 phút)
```

### Qua AWS CLI

```bash
# Tạo distribution đơn giản với S3 origin
aws cloudfront create-distribution \
  --origin-domain-name my-bucket.s3.amazonaws.com \
  --default-root-object index.html

# Tạo distribution từ config file (khuyến nghị cho production)
aws cloudfront create-distribution \
  --distribution-config file://distribution-config.json
```

### Terraform (Infrastructure as Code)

```hcl
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  default_root_object = "index.html"
  http_version        = "http2and3"

  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3-${aws_s3_bucket.assets.bucket}"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  default_cache_behavior {
    target_origin_id       = "S3-${aws_s3_bucket.assets.bucket}"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true

    cache_policy_id = "658327ea-f89d-4fab-a63d-7e88639e58f6" # CachingOptimized
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cdn.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  price_class = "PriceClass_All"
}
```

---

## 5. SSL/TLS & Custom Domain

### Yêu Cầu ACM Certificate

> **Quan trọng:** CloudFront chỉ nhận ACM certificates từ **us-east-1 (N. Virginia)**. Dù bạn deploy ở bất kỳ đâu, cert phải tạo tại us-east-1.

```bash
# Tạo cert tại us-east-1
aws acm request-certificate \
  --domain-name cdn.example.com \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS \
  --region us-east-1
```

### SNI vs Dedicated IP

| Phương Thức                                | Chi Phí            | Compatibility                    |
| ------------------------------------------ | ------------------ | -------------------------------- |
| **SNI** (Server Name Indication — khuyến nghị) | Miễn phí           | Hỗ trợ 98%+ browsers hiện đại   |
| **Dedicated IP**                           | $600/tháng/distribution | Hỗ trợ IE6 và cũ hơn — rất hiếm |

### TLS Minimum Version

```
TLSv1.2_2021 (khuyến nghị):
  - Hỗ trợ TLS 1.2 và 1.3
  - Loại bỏ các cipher suite yếu
  - Tuân thủ PCI DSS 4.0, HIPAA

TLSv1.2_2019:
  - Tương tự nhưng cipher suite cũ hơn một chút

TLSv1 (không dùng):
  - Deprecated — dễ bị tấn công POODLE, BEAST
```

### Custom Domain Setup — Checklist

```
1. ✅ Tạo ACM cert tại us-east-1 (validated)
2. ✅ Thêm Alternate Domain Name trong Distribution settings
3. ✅ Chọn cert từ ACM
4. ✅ Tạo CNAME record trong Route 53:
      cdn.example.com → d1234abcd.cloudfront.net
5. ✅ Hoặc dùng Alias record (nếu domain ở Route 53):
      cdn.example.com ALIAS → d1234abcd.cloudfront.net
```

---

## 6. Geo-Restriction

### Geo-Restriction (Giới Hạn Truy Cập Theo Địa Lý)

Cho phép chặn hoặc chỉ cho phép người dùng từ các quốc gia cụ thể.

```
Whitelist (Danh Sách Trắng):
  Chỉ cho phép: US, CA, GB, AU
  → Tất cả quốc gia khác nhận HTTP 403

Blacklist (Danh Sách Đen):
  Chặn: CN, RU, KP
  → Các quốc gia còn lại được phép
```

**Cách hoạt động:** CloudFront dùng IP geolocation database để xác định quốc gia của người dùng → quyết định cho phép hoặc chặn.

**Hạn chế:** Không phân biệt chi tiết theo tỉnh/thành. Muốn chi tiết hơn → dùng Lambda@Edge.

```bash
# Cấu hình geo restriction qua CLI
aws cloudfront update-distribution \
  --id EDFDVBD6EXAMPLE \
  --distribution-config '{
    "Restrictions": {
      "GeoRestriction": {
        "RestrictionType": "blacklist",
        "Quantity": 2,
        "Items": ["CN", "RU"]
      }
    }
  }'
```

---

## 7. Thực Hành Hands-On

### Lab 1: Static Website với S3 + CloudFront

```bash
# Bước 1: Tạo S3 bucket (private)
aws s3 mb s3://my-cloudfront-lab-bucket

# Bước 2: Upload files
echo '<h1>Hello from CloudFront!</h1>' > index.html
aws s3 cp index.html s3://my-cloudfront-lab-bucket/

# Bước 3: Tạo CloudFront distribution (xem Terraform snippet ở trên)
# → Distribution sẽ deploy trong 5-10 phút

# Bước 4: Test
curl -I https://d1234abcd.cloudfront.net/
# Kiểm tra header: X-Cache: Hit from cloudfront / Miss from cloudfront
```

### Lab 2: Kiểm Tra Cache Headers

```bash
# Request lần đầu — Cache MISS
curl -I https://d1234abcd.cloudfront.net/index.html
# X-Cache: Miss from cloudfront
# Via: 1.1 d1234.cloudfront.net (CloudFront)
# Age: 0

# Request lần hai — Cache HIT
curl -I https://d1234abcd.cloudfront.net/index.html
# X-Cache: Hit from cloudfront
# Age: 45 (seconds since first cached)
```

### Lab 3: Invalidate Cache

```bash
# Invalidate một file cụ thể
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/index.html"

# Invalidate tất cả files (dùng thận trọng — tốn phí)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# Kiểm tra trạng thái invalidation
aws cloudfront list-invalidations \
  --distribution-id EDFDVBD6EXAMPLE
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: CloudFront hoạt động như thế nào ở mức high level?

**Trả lời mẫu:**
> "CloudFront là CDN global của AWS. Khi tôi cấu hình một Distribution, CloudFront nhận request tại Edge Location gần người dùng nhất. Nếu nội dung đã được cache, nó trả về ngay với độ trễ rất thấp. Nếu miss, nó forward đến Regional Edge Cache — tầng cache thứ hai. Nếu vẫn miss, mới fetch từ Origin Server. Luồng 2 tầng cache này giúp giảm tải origin và cải thiện latency đáng kể."

### Q2: Tại sao phải tạo ACM certificate ở us-east-1?

**Trả lời:**
> "CloudFront là global service — không gắn với một Region cụ thể. AWS đã chọn us-east-1 làm endpoint tập trung cho CloudFront. Để certificate được tất cả Edge Location thế giới sử dụng, nó phải được lưu tại us-east-1 và AWS tự động replicate ra toàn bộ mạng lưới edge."

### Q3: Sự khác biệt giữa Edge Location và Regional Edge Cache?

**Trả lời:**
> "Edge Location (PoP) là điểm gần người dùng nhất — có hàng trăm PoP toàn cầu, dung lượng nhỏ. Regional Edge Cache là tầng trung gian với ~13 điểm, dung lượng lớn hơn nhiều. Khi PoP miss, thay vì fetch từ Origin (tốn bandwidth và latency), nó kiểm tra Regional Cache trước. Điều này giúp cache hit rate tổng thể tăng đáng kể, đặc biệt cho nội dung ít phổ biến."

### Q4: Invalidation và TTL khác nhau như thế nào?

**Trả lời:**
> "TTL là thời gian CloudFront tự động hết hạn cache. Invalidation là thao tác chủ động bắt CloudFront xóa cache ngay lập tức, bất kể TTL. TTL dùng cho cache management bình thường; Invalidation dùng khi cần push nội dung khẩn cấp — ví dụ hotfix, fix nội dung sai. Chi phí: 1.000 paths invalidation đầu tiên mỗi tháng miễn phí, sau đó $0.005/path."

### Q5: Origin Group dùng khi nào?

**Trả lời:**
> "Origin Group dùng khi cần **failover tự động** giữa hai origins. Ví dụ: primary là S3 bucket ở us-east-1, fallback là S3 ở eu-west-1. Nếu primary trả về 5xx errors hoặc connection timeout, CloudFront tự động retry với failover origin — hoàn toàn transparent với người dùng. Phù hợp cho disaster recovery không cần Route 53 health check can thiệp."

---

**Tiếp Theo:** [2-cache-behaviors.md](./2-cache-behaviors.md) — Cache Policies, TTL, và Cache Key
