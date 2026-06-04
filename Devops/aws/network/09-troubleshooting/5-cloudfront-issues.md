# ☁️ CloudFront Issues — Xử Lý Sự Cố CDN

> Hướng dẫn chẩn đoán các vấn đề phổ biến với CloudFront CDN (Content Delivery Network — Mạng Phân Phối Nội Dung): cache miss (Bỏ Lỡ Cache), origin errors (Lỗi Nguồn Gốc), SSL certificate issues, và các vấn đề performance.

---

## 📚 Mục Lục

1. [Kiến Trúc CloudFront — Điểm Debug](#kiến-trúc-cloudfront--điểm-debug)
2. [Cache Miss — Tại Sao Cache Không Hoạt Động?](#cache-miss--tại-sao-cache-không-hoạt-động)
3. [Origin Errors — 5xx Từ CloudFront](#origin-errors--5xx-từ-cloudfront)
4. [SSL/TLS Issues Với CloudFront](#ssltls-issues-với-cloudfront)
5. [Lỗi 403 Access Denied — S3 Origin](#lỗi-403-access-denied--s3-origin)
6. [Content Stale — Nội Dung Cũ Không Được Cập Nhật](#content-stale--nội-dung-cũ-không-được-cập-nhật)
7. [Performance Issues — Độ Trễ Cao](#performance-issues--độ-trễ-cao)
8. [Phân Tích CloudFront Logs](#phân-tích-cloudfront-logs)
9. [Checklist Nhanh](#checklist-nhanh)

---

## 🏗️ Kiến Trúc CloudFront — Điểm Debug

```
Client (Người Dùng)
   │
   ▼
CloudFront Edge Location (Điểm Biên — gần người dùng nhất)
   │
   ├── Cache HIT (Trúng Cache): Trả về nội dung từ cache, không gọi origin
   │
   └── Cache MISS (Bỏ Lỡ Cache): Forward request đến Regional Edge Cache
          │
          └── Cache MISS ở Regional Edge: Forward đến Origin
                 │
                 ├── S3 Bucket
                 ├── ALB (Application Load Balancer)
                 └── Custom HTTP Server
```

### Headers Quan Trọng Để Debug

| Header | Ý Nghĩa |
|--------|---------|
| `X-Cache: Hit from cloudfront` | Trúng cache tại edge |
| `X-Cache: Miss from cloudfront` | Bỏ lỡ cache, gọi về origin |
| `X-Cache: RefreshHit from cloudfront` | Cache được refresh |
| `X-Amz-Cf-Id` | CloudFront request ID để trace |
| `Age` | Số giây content đã ở trong cache |
| `Via: 1.1 <hash>.cloudfront.net` | Request đi qua CloudFront |

```bash
# Kiểm tra cache status
curl -I https://d1234abcde.cloudfront.net/image.jpg
# HTTP/2 200
# x-cache: Hit from cloudfront
# age: 3600
```

---

## 🟡 Cache Miss — Tại Sao Cache Không Hoạt Động?

### Nguyên Nhân Phổ Biến

```
1. Cache-Control: no-cache / no-store trong response của origin
2. Query string khác nhau tạo ra cache key khác
3. Cookie khác nhau tạo ra cache key khác
4. HTTP method không phải GET/HEAD (POST không được cache)
5. TTL = 0 vì Cache-Control: max-age=0
6. Behavior Forward All Headers → mỗi request đều unique
```

### Bước 1: Kiểm Tra Cache Policy (Chính Sách Cache)

```bash
# Xem cache behaviors của distribution
aws cloudfront get-distribution-config \
  --id <distribution-id> \
  --query 'DistributionConfig.CacheBehaviors'

# Xem cache policy
aws cloudfront get-cache-policy \
  --id <cache-policy-id>
```

**Kiểm tra cache policy có:**
- `DefaultTTL` > 0
- Headers, Cookies, Query Strings được include trong cache key có thực sự cần thiết không?

### Bước 2: Kiểm Tra Origin Response Headers

```bash
# Gọi trực tiếp origin, xem headers nó trả về
curl -I https://my-origin-alb.example.com/api/resource
```

**Nếu origin trả về:**
```
Cache-Control: no-store, no-cache, must-revalidate
```
→ CloudFront **không cache** dù config TTL cao đến đâu.

**Giải pháp:** Override với CloudFront Cache Policy:
```
MinTTL = 0
DefaultTTL = 86400
MaxTTL = 31536000
```
Và tắt "Respect Origin Cache-Control Headers" trong policy.

### Bước 3: Kiểm Tra Cache Key

Cache key mặc định = URL path + (headers/cookies/query strings được cấu hình trong cache policy).

**Ví dụ vấn đề:**
```
Request 1: /image.jpg?v=1&user=alice   → Cache key A
Request 2: /image.jpg?v=1&user=bob     → Cache key B (miss)
Request 3: /image.jpg?user=alice&v=1   → Cache key C (miss! thứ tự khác)
```

**Giải pháp:** Normalize query strings — chỉ include query strings thực sự cần thiết trong cache key.

### Bước 4: Kiểm Tra Cache Hit Ratio

```bash
# Xem CloudFront cache statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global \
  --start-time 2026-05-14T00:00:00 \
  --end-time 2026-05-14T23:59:59 \
  --period 3600 \
  --statistics Average
```

**Cache Hit Rate tốt:** > 80%

---

## 🔴 Origin Errors — 5xx Từ CloudFront

### Phân Biệt CloudFront 5xx vs Origin 5xx

| Error Code | Nguồn Gốc | Ý Nghĩa |
|-----------|-----------|---------|
| `502` | CloudFront | Origin trả về response không hợp lệ |
| `503` | CloudFront | Origin unavailable |
| `504` | CloudFront | Origin timeout (mặc định 30 giây) |
| `520` | CloudFront | Origin trả về unknown response |
| `521` | CloudFront | Origin từ chối connection |
| `522` | CloudFront | Connection timeout đến origin |
| `523` | CloudFront | Origin unreachable |

### Bước 1: Kiểm Tra Origin Trực Tiếp

```bash
# Bypass CloudFront, gọi trực tiếp origin
curl -v https://my-alb-origin.us-east-1.elb.amazonaws.com/api/resource \
  -H "Host: api.example.com"

# Với S3 origin
curl -v https://my-bucket.s3.amazonaws.com/path/to/file
```

**Nếu origin response bình thường** → Vấn đề ở CloudFront → origin connectivity.
**Nếu origin error** → Sửa origin trước.

### Bước 2: Kiểm Tra Origin Settings

```bash
# Xem origin config trong distribution
aws cloudfront get-distribution-config \
  --id <distribution-id> \
  --query 'DistributionConfig.Origins'
```

**Điểm kiểm tra:**
- `DomainName`: đúng domain của origin?
- `CustomOriginConfig.OriginProtocolPolicy`: HTTP-only, HTTPS-only, hay match-viewer?
- `CustomOriginConfig.HTTPSPort`: đúng port HTTPS?
- `CustomOriginConfig.OriginReadTimeout`: có đủ cao không? (mặc định 30s)
- `CustomOriginConfig.OriginKeepaliveTimeout`: có phù hợp không?

### Bước 3: Kiểm Tra Origin SSL Certificate

Nếu CloudFront connect đến origin qua HTTPS:

```bash
# Test SSL handshake đến origin
openssl s_client -connect <origin-domain>:443 -servername <origin-domain>
```

**Yêu cầu:** Certificate của origin phải:
- Được ký bởi CA (Certificate Authority — Tổ Chức Cấp Chứng Chỉ) tin cậy
- Match với domain name trong origin config
- Chưa expired

> CloudFront không chấp nhận self-signed certificate (Chứng Chỉ Tự Ký) cho HTTPS origins (trừ khi bật origin verification bypass — không khuyến nghị).

### Bước 4: Custom Error Pages

Khi origin down, hiển thị error page tùy chỉnh thay vì error mặc định:

```bash
# Cấu hình custom error response
aws cloudfront update-distribution \
  --id <dist-id> \
  --distribution-config file://dist-config.json

# Trong dist-config.json, thêm:
# "CustomErrorResponses": {
#   "Quantity": 1,
#   "Items": [{
#     "ErrorCode": 503,
#     "ResponsePagePath": "/maintenance.html",
#     "ResponseCode": "503",
#     "ErrorCachingMinTTL": 60
#   }]
# }
```

---

## 🔒 SSL/TLS Issues Với CloudFront

### CloudFront SSL Architecture

```
Client ←── SSL ──→ CloudFront Edge ←── SSL ──→ Origin

Có 2 SSL connections riêng biệt:
1. Client ↔ CloudFront: dùng certificate trong CloudFront
2. CloudFront ↔ Origin: dùng certificate của origin
```

### Lỗi SSL Client-to-CloudFront

**Triệu chứng:** Browser hiện "Your connection is not private" hoặc certificate warning.

**Kiểm tra:**
```bash
# Xem certificates gán vào distribution
aws cloudfront get-distribution-config \
  --id <dist-id> \
  --query 'DistributionConfig.ViewerCertificate'
```

**Yêu cầu:**
- `ACMCertificateArn`: Certificate phải ở **us-east-1** (region duy nhất CloudFront chấp nhận)
- Certificate cover đúng domain (hoặc wildcard)
- `MinimumProtocolVersion`: TLSv1.2_2021 (không dùng TLSv1 — đã lỗi thời)

```bash
# Kiểm tra certificate ở us-east-1
aws acm list-certificates \
  --region us-east-1 \
  --certificate-statuses ISSUED
```

### Lỗi Mixed Content (Nội Dung Hỗn Hợp)

**Triệu chứng:** HTTPS page load nhưng assets (JS, CSS, images) bị block vì dùng HTTP URL.

**Giải pháp:** Bật HTTPS redirect trong CloudFront behavior:
```
ViewerProtocolPolicy: redirect-to-https
```

---

## 🔐 Lỗi 403 Access Denied — S3 Origin

### Nguyên Nhân Phổ Biến 403

1. **S3 bucket policy không cho phép CloudFront OAC (Origin Access Control — Kiểm Soát Truy Cập Nguồn Gốc)**
2. **Dùng OAI cũ thay vì OAC mới**
3. **S3 Block Public Access bật nhưng policy chưa cấp quyền cho CloudFront**
4. **Object không tồn tại trong S3** (thực ra là 404 nhưng S3 trả về 403 để bảo mật)

### Bước 1: Kiểm Tra OAC Configuration

```bash
# Xem origin config có dùng OAC không
aws cloudfront get-distribution-config \
  --id <dist-id> \
  --query 'DistributionConfig.Origins.Items[].S3OriginConfig'

# Xem OAC details
aws cloudfront get-origin-access-control \
  --id <oac-id>
```

### Bước 2: Kiểm Tra S3 Bucket Policy

```bash
# Xem bucket policy
aws s3api get-bucket-policy \
  --bucket <bucket-name> \
  --query 'Policy' | python3 -m json.tool
```

**S3 Bucket Policy đúng cho OAC:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/<dist-id>"
      }
    }
  }]
}
```

### Bước 3: Test Trực Tiếp S3

```bash
# Thử access object từ S3 trực tiếp
aws s3 cp s3://<bucket-name>/path/to/file /tmp/test

# Kiểm tra Block Public Access settings
aws s3api get-public-access-block \
  --bucket <bucket-name>
```

---

## 🔄 Content Stale — Nội Dung Cũ Không Được Cập Nhật

### Tại Sao Nội Dung Bị Cũ?

CloudFront cache content dựa theo TTL (Time To Live):
```
TTL còn lại > 0: CloudFront phục vụ từ cache (không gọi origin)
TTL = 0:         CloudFront revalidate với origin
```

### Invalidation (Vô Hiệu Hóa Cache)

```bash
# Invalidate specific files
aws cloudfront create-invalidation \
  --distribution-id <dist-id> \
  --paths "/images/logo.png" "/css/style.css"

# Invalidate tất cả (tốn phí hơn sau 1000 paths/tháng đầu miễn phí)
aws cloudfront create-invalidation \
  --distribution-id <dist-id> \
  --paths "/*"

# Kiểm tra trạng thái invalidation
aws cloudfront get-invalidation \
  --distribution-id <dist-id> \
  --id <invalidation-id>
```

**Chi phí:** 1000 invalidation paths đầu tiên/tháng miễn phí. Sau đó $0.005/path.

### Chiến Lược Tránh Cần Invalidation

**Cache Busting với versioned filenames (Tên File Có Phiên Bản):**
```
/css/style.v2.css    (thay vì invalidate /css/style.css)
/js/app.20260514.js
/images/logo.v3.png
```

```html
<!-- HTML reference file mới → cache HIT cho file mới ngay lập tức -->
<link rel="stylesheet" href="/css/style.v2.css">
```

---

## ⚡ Performance Issues — Độ Trễ Cao

### Tại Sao CloudFront Chậm?

1. **Cache Miss Rate cao** → Mọi request đều đến origin
2. **Origin chậm** → Dù cache miss, origin response chậm ảnh hưởng đến user
3. **Price Class thấp** → Edge locations ít, user xa edge location hơn
4. **Compression không bật** → File size lớn hơn cần thiết
5. **HTTP/2 hoặc HTTP/3 chưa bật**

### Kiểm Tra Origin Latency

```bash
# Xem OriginLatency metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name OriginLatency \
  --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global \
  --start-time 2026-05-14T00:00:00 \
  --end-time 2026-05-14T23:59:59 \
  --period 3600 \
  --statistics p99
```

### Kiểm Tra Compression Settings

```bash
# Xem compress setting trong cache behavior
aws cloudfront get-distribution-config \
  --id <dist-id> \
  --query 'DistributionConfig.DefaultCacheBehavior.Compress'
```

**Bật compression** (`Compress: true`) giảm file size 60-90% cho text-based content (HTML, CSS, JS, JSON).

### Bật Real-Time Metrics

```bash
# Bật real-time metrics cho distribution
aws cloudfront create-monitoring-subscription \
  --distribution-id <dist-id> \
  --monitoring-subscription RealtimeMetricsSubscriptionConfig=RealtimeMetricsSubscriptionStatus=Enabled
```

---

## 📋 Phân Tích CloudFront Logs

### Bật Standard Logging (Nhật Ký Chuẩn)

```bash
# Bật CloudFront standard logging vào S3
aws cloudfront update-distribution \
  --id <dist-id> \
  --distribution-config file://dist-config-with-logging.json

# Logging config trong file:
# "Logging": {
#   "Enabled": true,
#   "IncludeCookies": false,
#   "Bucket": "my-cf-logs.s3.amazonaws.com",
#   "Prefix": "cloudfront-logs/"
# }
```

### Phân Tích Log Với Athena

```sql
-- Tạo Athena table cho CloudFront logs
CREATE EXTERNAL TABLE cf_logs (
  date DATE, time STRING, location STRING,
  bytes BIGINT, request_ip STRING, method STRING,
  host STRING, uri STRING, status INT,
  referrer STRING, user_agent STRING,
  query_string STRING, cookie STRING,
  result_type STRING, request_id STRING,
  host_header STRING, request_protocol STRING,
  bytes_sent BIGINT, processing_time FLOAT,
  forwarded_for STRING, ssl_protocol STRING,
  ssl_cipher STRING, response_result_type STRING,
  http_version STRING, fle_status STRING,
  fle_encrypted_fields STRING, c_port INT,
  time_to_first_byte FLOAT, x_edge_detailed_result_type STRING,
  sc_content_type STRING, sc_content_len BIGINT,
  sc_range_start BIGINT, sc_range_end BIGINT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION 's3://my-cf-logs/cloudfront-logs/';

-- Tìm cache miss theo path
SELECT uri, COUNT(*) as miss_count
FROM cf_logs
WHERE result_type = 'Miss'
GROUP BY uri
ORDER BY miss_count DESC
LIMIT 20;

-- Phân tích status codes
SELECT status, COUNT(*) as count
FROM cf_logs
GROUP BY status
ORDER BY count DESC;

-- P99 latency theo ngày
SELECT date,
       approx_percentile(processing_time, 0.99) as p99_latency,
       AVG(processing_time) as avg_latency
FROM cf_logs
GROUP BY date
ORDER BY date;
```

---

## ✅ Checklist Nhanh

### Cache Miss Rate Cao

```
□ Cache-Control headers từ origin có cho phép caching?
□ Cache policy có TTL > 0?
□ Cache key không quá granular (quá nhiều headers/cookies/query strings)?
□ Request method là GET hoặc HEAD (không phải POST)?
□ Không có "Authorization" header trong cache key (mặc định bypass cache)?
```

### Origin Errors (5xx)

```
□ Origin trực tiếp có trả lời không? (curl trực tiếp origin)
□ Origin domain name đúng trong CloudFront config?
□ Origin SSL certificate hợp lệ?
□ Origin read timeout có đủ cao không?
□ Security Group của origin có ALLOW CloudFront IPs không?
□ Origin có đủ capacity để xử lý traffic không?
```

### 403 Access Denied (S3 Origin)

```
□ S3 Bucket Policy có Allow CloudFront service principal?
□ Dùng OAC (không phải OAI cũ)?
□ Distribution ID trong condition đúng không?
□ S3 Block Public Access không chặn bucket policy?
□ Object thực sự tồn tại trong S3?
```

### SSL Issues

```
□ Certificate ở us-east-1?
□ Certificate trạng thái ISSUED?
□ Certificate cover đúng domain?
□ ViewerProtocolPolicy = redirect-to-https hoặc https-only?
□ MinimumProtocolVersion >= TLSv1.2?
```

### Content Stale

```
□ Đã tạo invalidation sau khi deploy mới?
□ Invalidation status là Completed?
□ Dùng versioned filenames để tránh cần invalidation?
□ Cache-Control headers từ origin có đúng không?
```

---

## 📚 Tài Liệu Liên Quan

- [../05-cdn-cloudfront/2-cache-behaviors.md](../05-cdn-cloudfront/2-cache-behaviors.md) — Cache Behaviors & Policies
- [../05-cdn-cloudfront/3-origin-access.md](../05-cdn-cloudfront/3-origin-access.md) — OAC, OAI, S3 Protection
- [../05-cdn-cloudfront/5-performance-cost.md](../05-cdn-cloudfront/5-performance-cost.md) — Performance Optimization

---

**Cập Nhật Lần Cuối:** 2026-05-14
