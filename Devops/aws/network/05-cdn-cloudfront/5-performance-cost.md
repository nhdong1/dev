# Performance & Cost Optimization — Tối Ưu Hiệu Suất và Chi Phí CloudFront

> Hiểu rõ Price Classes (Hạng Giá), cơ chế compression (nén), và các chiến lược optimization giúp bạn đạt được cache hit ratio cao, latency thấp, và chi phí hợp lý. Đây là kiến thức thiết yếu cho production CloudFront deployments.

---

## 📚 Mục Lục

1. [Mô Hình Định Giá CloudFront](#1-mô-hình-định-giá-cloudfront)
2. [Price Classes — Hạng Giá](#2-price-classes--hạng-giá)
3. [Compression — Nén Dữ Liệu](#3-compression--nén-dữ-liệu)
4. [HTTP/2 & HTTP/3 (QUIC)](#4-http2--http3-quic)
5. [Tối Ưu Cache Hit Ratio](#5-tối-ưu-cache-hit-ratio)
6. [CloudFront Access Logs & Metrics](#6-cloudfront-access-logs--metrics)
7. [Cost Optimization Strategies](#7-cost-optimization-strategies)
8. [Performance Checklist](#8-performance-checklist)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Mô Hình Định Giá CloudFront

### Các Thành Phần Chi Phí

```
1. Data Transfer Out (Truyền Dữ Liệu Ra)
   → Từ CloudFront đến Internet (đến người dùng)
   → Giá thay đổi theo region và lượng dùng

2. HTTP Requests
   → Tính phí mỗi request (GET, POST, v.v.)
   → Khác nhau theo HTTP method và region

3. Data Transfer to Origin (Dữ Liệu Đến Origin)
   → Từ CloudFront đến Origin Server khi cache miss
   → Thấp hơn Data Transfer Out

4. Lambda@Edge Invocations & Duration
   → Nếu dùng Lambda@Edge

5. CloudFront Functions Invocations
   → Nếu dùng CloudFront Functions

6. Invalidation Requests
   → Đầu tiên 1,000 paths/tháng: miễn phí
   → Sau đó: $0.005/path

7. Field-Level Encryption (Mã Hóa Cấp Trường)
   → Nếu bật tính năng này

8. Real-Time Logs
   → Nếu bật streaming logs đến Kinesis
```

### Ví Dụ Tính Chi Phí (US-East)

```
Data Transfer Out: 
  10TB đầu: $0.0085/GB
  40TB tiếp: $0.0080/GB
  100TB tiếp: $0.0060/GB
  ...

HTTP Requests (HTTPS):
  $0.01 per 10,000 requests

Ví dụ thực tế:
  Traffic: 100GB/ngày từ Singapore PoP
  Requests: 1 triệu/ngày
  
  Chi phí tháng:
    Data: 100GB × 30 × $0.012/GB (APAC) = $36
    Requests: 30M × $0.012/10,000 = $36
    Tổng: ~$72/tháng
```

### Free Tier CloudFront

```
Mỗi tháng (12 tháng đầu sau khi tạo account):
  - 1TB data transfer out
  - 10,000,000 HTTP/HTTPS requests
  - 2,000,000 CloudFront Functions invocations

→ Đủ cho website nhỏ/medium không tốn phí
```

---

## 2. Price Classes — Hạng Giá

**Price Class** (Hạng Giá) cho phép giới hạn Edge Locations mà CloudFront dùng — đánh đổi giữa coverage toàn cầu và chi phí.

### Ba Price Classes

#### PriceClass_All — Toàn Bộ Edge Locations

```
Bao gồm: Tất cả 600+ Edge Locations toàn cầu
  - Bắc Mỹ, Châu Âu, Châu Á, Nam Mỹ
  - Trung Đông, Châu Phi, Châu Đại Dương

Giá: Cao nhất (vì bao gồm các region đắt như Ấn Độ, Nam Mỹ)
Performance: Tốt nhất — người dùng luôn được serve từ edge gần nhất
Dùng khi: App có user base toàn cầu, performance là ưu tiên số 1
```

#### PriceClass_200 — Hầu Hết Regions

```
Bao gồm: US, Canada, Châu Âu, Israel, Nhật Bản, Australia
  + Hàn Quốc, Singapore, Hong Kong, Philippines, Bangladesh

Loại bỏ: Một số Edge Locations đắt tiền nhất ở:
  - Nam Mỹ (Brazil, Argentina, Chile...)
  - Trung Đông (Dubai, Cairo...)
  - Châu Phi (Cape Town, Nairobi...)

Giá: Trung bình — tiết kiệm ~20-30% so với PriceClass_All
Dùng khi: User chủ yếu ở US/EU/APAC — không cần cover Nam Mỹ/Châu Phi nhiều
```

#### PriceClass_100 — Rẻ Nhất

```
Bao gồm: Chỉ US, Canada, Châu Âu
  - Bắc Mỹ: ~30 PoPs
  - Châu Âu: ~25 PoPs

Loại bỏ: Châu Á, Nam Mỹ, Trung Đông, Châu Phi

Giá: Thấp nhất — tiết kiệm ~40-50% so với PriceClass_All
Performance: Tệ hơn cho user ở Châu Á (serve từ US/EU edge)
Dùng khi: App chỉ dành cho US/EU users, hoặc nội bộ công ty
```

### So Sánh Trực Quan

```
PriceClass_All:
  🌎 🌍 🌏 — Toàn cầu, mọi nơi
  💰💰💰 — Chi phí cao nhất

PriceClass_200:
  🌎 🌍 🌏* — Hầu hết, bỏ một số region đắt
  💰💰 — Chi phí trung bình

PriceClass_100:
  🌎 🌍 — Chỉ US + EU
  💰 — Chi phí thấp nhất
```

### Quyết Định Price Class

```
Phân tích traffic theo region:
  aws cloudfront get-distribution-metric-statistics
  (hoặc xem CloudWatch metrics theo region)

Nếu:
  90%+ traffic từ US/EU → PriceClass_100
  90%+ traffic từ US/EU/APAC → PriceClass_200
  Traffic đồng đều toàn cầu → PriceClass_All

Lưu ý: Người dùng ở region không có edge vẫn được serve
        nhưng từ edge xa hơn → latency cao hơn
```

---

## 3. Compression — Nén Dữ Liệu

### Compress Objects Automatically

CloudFront hỗ trợ hai thuật toán nén:

```
Gzip (GNU zip):
  - Hỗ trợ rộng rãi từ trước đến nay
  - Browser request: Accept-Encoding: gzip
  - Tỷ lệ nén: ~60-70% cho text content

Brotli:
  - Thuật toán mới hơn của Google
  - Nén tốt hơn Gzip ~20-26%
  - Browser request: Accept-Encoding: br
  - Hỗ trợ: 95%+ browsers hiện đại (Chrome, Firefox, Edge, Safari)
```

### Cơ Chế Compression Của CloudFront

```
Kịch bản 1: Origin gửi file chưa nén
  Browser: Accept-Encoding: br, gzip
  CloudFront: Nhận file chưa nén từ origin
  CloudFront: Nén bằng Brotli (ưu tiên hơn Gzip)
  CloudFront: Cache bản đã nén
  CloudFront → Browser: Content-Encoding: br

Kịch bản 2: Origin gửi file đã nén Gzip
  CloudFront: Lưu cache bản Gzip từ origin
  Browser hỗ trợ Brotli → CloudFront vẫn trả Gzip (không re-compress)
  Browser không hỗ trợ Gzip → CloudFront decompress → trả uncompressed

Kịch bản 3: Cache key và compression
  CloudFront tự động vary cache theo compression:
  - Cache entry riêng cho Gzip
  - Cache entry riêng cho Brotli
  - Cache entry riêng cho uncompressed
```

### Điều Kiện Để CloudFront Compress

```
1. "Compress objects automatically" = Yes trong Cache Behavior
2. File size: 1KB - 10MB (quá nhỏ/lớn không nén)
3. Content-Type phải là text format:
   ✅ text/html, text/css, text/javascript, application/javascript
   ✅ application/json, application/xml, text/xml
   ✅ application/rss+xml, application/atom+xml
   ✅ image/svg+xml (SVG là text)
   ❌ image/jpeg, image/png, image/webp (đã compressed)
   ❌ video/mp4, audio/mp3 (binary)
   ❌ application/zip, application/pdf
4. Response không có Content-Encoding header từ origin
```

### Impact Của Compression

```
Ví dụ thực tế:
  File: app.bundle.js (uncompressed: 500KB)
  After Gzip: 150KB (70% reduction)
  After Brotli: 120KB (76% reduction)

Lợi ích:
  - Bandwidth tiết kiệm: 75% → giảm bill CloudFront
  - Page load nhanh hơn: ~3x cho slow connections
  - Đặc biệt hiệu quả cho mobile users (cellular data)

Chi phí CloudFront tính trên compressed size:
  → 500KB uncompressed phát sinh bill $0.00425 (at $0.0085/GB)
  → 120KB Brotli phát sinh bill $0.00102
  → Tiết kiệm 76% bandwidth cost
```

---

## 4. HTTP/2 & HTTP/3 (QUIC)

### HTTP/2 — Multiplexing & Header Compression

```
HTTP/1.1 (cũ):
  Browser: gửi request tuần tự (hoặc 6 parallel connections)
  → Nhiều RTT (Round Trip Time — Thời Gian Khứ Hồi)
  → Head-of-line blocking

HTTP/2:
  Multiplexing: Nhiều requests trong 1 TCP connection
  Header Compression: HPACK algorithm giảm overhead
  Server Push: Server gửi resources trước khi browser request
  Binary Protocol: Hiệu quả hơn text-based HTTP/1.1
  
  → CloudFront hỗ trợ HTTP/2 mặc định (không cần cấu hình thêm)
  → Cải thiện performance đáng kể cho web apps nhiều small requests
```

### HTTP/3 / QUIC — Transport Layer Revolution

```
HTTP/2 vẫn dùng TCP:
  Khi có packet loss → toàn bộ stream bị block (TCP head-of-line blocking)

HTTP/3 dùng QUIC (UDP-based):
  Packet loss chỉ ảnh hưởng stream đó, không block streams khác
  0-RTT connection resumption (reconnect nhanh hơn)
  Built-in encryption (TLS 1.3)
  Tốt hơn trên mobile networks (thay đổi IP khi chuyển WiFi → 4G)

CloudFront:
  HTTP/3 hỗ trợ từ 2022
  Cần bật thủ công trong Distribution settings:
    HTTP Version: HTTP/2 and HTTP/3
```

### HTTP Version Setting

```bash
# Bật HTTP/3 trong Distribution
aws cloudfront update-distribution \
  --id EDFDVBD6EXAMPLE \
  --distribution-config '{
    "HttpVersion": "http2and3",
    ...
  }'
```

```hcl
# Terraform
resource "aws_cloudfront_distribution" "main" {
  http_version = "http2and3"  # Options: http1.1, http2, http2and3
  ...
}
```

---

## 5. Tối Ưu Cache Hit Ratio

**Cache Hit Ratio** (Tỷ Lệ Truy Cập Cache) = (Số requests từ cache) / (Tổng requests) × 100%

Mục tiêu: **>80% cache hit ratio** cho static assets, >50% cho mixed workloads.

### Nguyên Nhân Gây Cache Hit Ratio Thấp

```
1. TTL quá ngắn → cache expire nhanh, phải fetch lại
2. Cache key quá rộng:
   Include tất cả query strings → mỗi ?session=xxx tạo entry riêng
   Include cookies → nhiều users, nhiều entries

3. Nội dung thực sự dynamic (API responses, user-specific pages)
   → Không thể cache cao hơn

4. Invalidations quá thường xuyên → xóa cache sớm

5. Ít traffic → cache "lạnh" (cold cache), chưa có hit
```

### Strategies Tăng Cache Hit Ratio

#### Strategy 1: Loại Bỏ Query Parameters Không Cần Thiết Khỏi Cache Key

```
Vấn đề:
  /product?id=123&utm_source=google&_ga=2.xxxxx&session=abc
  Cache key: /product?id=123&utm_source=google&_ga=2.xxxxx&session=abc
  → Mỗi user, mỗi source có cache entry riêng → hit ratio thấp

Giải pháp — Custom Cache Policy:
  Chỉ include "id" trong cache key
  Forward "id" lên origin
  Loại bỏ: utm_*, _ga, session khỏi cache key

  Cache key: /product?id=123 (shared across all users)
  → Hit ratio tăng đáng kể
```

#### Strategy 2: Versioned Assets

```
Thay vì:
  /styles.css?v=1  và  /styles.css?v=2  (2 cache entries)
  Cache TTL: 5 phút (phải check thường xuyên)

Dùng:
  /styles.a3f4b2.css  và  /styles.c5d6e7.css
  Cache TTL: 1 năm (immutable)

  → Cùng file → hit liên tục trong 1 năm
  → File mới → URL mới → miss một lần → cache tiếp 1 năm
```

#### Strategy 3: Tách Cache Behaviors

```
Tối ưu từng loại nội dung khác nhau:

/static/*     → CachingOptimized (1 năm, no headers/cookies in key)
/api/public/* → Custom (60 giây, chỉ include "lang" query param)
/api/user/*   → CachingDisabled (per-user content)
/images/*     → Custom (30 ngày, no params in key)
/*            → Custom (5 phút, HTML files)
```

#### Strategy 4: Origin Shield (Tấm Khiên Origin)

**Origin Shield** là tầng cache trung gian bổ sung giữa Regional Edge Caches và Origin.

```
Không có Origin Shield:
  Nhiều Regional Edge Caches (Singapore, Tokyo, Sydney)
  → Tất cả miss → gửi request đến Origin cùng lúc
  → Origin nhận nhiều requests

Với Origin Shield (VD: Singapore):
  Regional Edge Caches → trước tiên check Origin Shield
  → Nếu hit tại Shield: không đến origin
  → Nếu miss tại Shield: chỉ 1 request đến origin (collapsed)

  → Giảm tải origin đáng kể (đặc biệt với cacheable content)
  → Cải thiện cache hit ratio overall
  → Chi phí thêm: ~$0.0075/10,000 requests (Origin Shield requests)
```

```hcl
# Terraform: Bật Origin Shield
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_s3_bucket.assets.bucket_domain_name
    origin_id   = "S3Origin"

    origin_shield {
      enabled              = true
      origin_shield_region = "ap-southeast-1"  # Region gần origin nhất
    }
  }
}
```

#### Strategy 5: Correct Cache-Control Headers Từ Origin

```
Origin nên trả về đúng headers:

Static assets (có hash):
  Cache-Control: public, max-age=31536000, immutable
  → CloudFront cache 1 năm, không check lại

HTML (cần fresh):
  Cache-Control: public, max-age=300, must-revalidate
  → Cache 5 phút, sau đó revalidate (conditional GET)
  ETag: "abc123" / Last-Modified: Thu, 01 Jan 2026 00:00:00 GMT
  → Nếu không thay đổi: 304 Not Modified (không re-transfer body)

API (user-specific):
  Cache-Control: private, no-store
  → CloudFront không cache (Viewer cache cũng không)
```

---

## 6. CloudFront Access Logs & Metrics

### Access Logs (Nhật Ký Truy Cập)

```bash
# Bật access logs → ghi vào S3 bucket
aws cloudfront update-distribution \
  --id EDFDVBD6EXAMPLE \
  --distribution-config '{
    "Logging": {
      "Enabled": true,
      "IncludeCookies": false,
      "Bucket": "my-logs.s3.amazonaws.com",
      "Prefix": "cloudfront-logs/"
    }
  }'
```

**Format Log Fields quan trọng:**

```
date          → Ngày (2026-05-14)
time          → Giờ UTC (12:34:56)
x-edge-location → Edge location code (SIN3-C1)
sc-bytes      → Bytes gửi đến client
c-ip          → IP của client
cs-method     → HTTP method (GET, POST)
cs-uri-stem   → URL path (/images/logo.png)
sc-status     → HTTP status (200, 304, 403)
x-edge-result-type → Hit, Miss, Error, LimitExceeded
x-edge-request-id  → Unique request ID
x-host-header → Host header từ client
cs-protocol   → https
cs-bytes      → Bytes nhận từ client
time-taken    → Thời gian xử lý (giây)
x-forwarded-for → Forwarded IP
ssl-protocol  → TLSv1.3
ssl-cipher    → TLS_AES_128_GCM_SHA256
x-edge-response-result-type → Hit, Miss, Error
cs-uri-query  → Query string
x-edge-detailed-result-type → Miss, Hit, RefreshHit, OriginShieldHit...
```

### Real-Time Logs

```
Access Logs: Delivered trong 1 giờ → cho analysis, không real-time
Real-Time Logs: Streaming đến Kinesis Data Streams → near real-time

Cấu hình:
  Sampling rate: 1-100% (100% = mọi request)
  Fields: Chọn subset fields cần thiết
  Kinesis stream: Gửi đến Kinesis → Firehose → S3/Elasticsearch

Use cases:
  - Dashboard real-time traffic
  - Anomaly detection (đột biến traffic, error spikes)
  - Security monitoring (IP patterns, attack detection)
```

### CloudWatch Metrics Quan Trọng

```
Requests: Tổng số requests đến CloudFront
BytesDownloaded: Data transfer đến viewer
BytesUploaded: Data upload từ viewer
4xxErrorRate: Tỷ lệ 4xx errors (client errors)
5xxErrorRate: Tỷ lệ 5xx errors (server/origin errors)
TotalErrorRate: 4xx + 5xx

CacheHitRate: Tỷ lệ cache hit (quan trọng nhất)
  → Monitor liên tục, alert khi < 50%

OriginLatency: Latency từ CloudFront đến Origin
  → Tăng bất thường → origin có vấn đề

Không có metric trực tiếp cho edge location latency
→ Dùng Real-Time Logs để phân tích time-taken per PoP
```

---

## 7. Cost Optimization Strategies

### 1. Chọn Price Class Phù Hợp

```
Phân tích: Xem CloudWatch → Requests by Region (qua Access Logs)
→ Nếu 95% traffic từ US/EU: dùng PriceClass_100
→ Tiết kiệm: 40-50% data transfer cost
```

### 2. Tối Đa Hóa Cache Hit Ratio

```
Mỗi 1% tăng cache hit ratio:
  → Giảm 1% requests đến origin (giảm EC2/S3 chi phí)
  → Giảm tỷ lệ data transfer origin → edge (rẻ hơn edge → internet)
  
Target: >85% cho static-heavy sites
```

### 3. Dùng CloudFront Thay Vì S3 Direct Transfer

```
S3 data transfer out: $0.09/GB (đồng nhất toàn cầu)
CloudFront data transfer out: $0.0085-0.02/GB (tùy region)

→ CloudFront thường RẺ HƠN cho large scale
→ Cộng thêm: CloudFront có cache → giảm S3 requests

S3 GET request: $0.0004/1,000 requests
CloudFront request: $0.01/10,000 requests (rẻ hơn 4x)

→ Với traffic lớn: CloudFront + S3 thường rẻ hơn S3 direct
```

### 4. Compression Để Giảm Bandwidth

```
Bật "Compress objects automatically":
→ Gzip/Brotli giảm size 60-76% cho text content
→ Giảm trực tiếp data transfer cost

Ước tính: Site với 5TB text content/tháng
  Không nén: 5TB × $0.0085/GB = $42.50
  Brotli (76% reduction): 1.2TB × $0.0085/GB = $10.20
  → Tiết kiệm: $32.30/tháng chỉ từ compression
```

### 5. Tối Ưu Invalidations

```
Đừng:
  - Invalidate "/*" mỗi lần deploy (tốn phí nếu > 1000 paths)
  - Invalidate files riêng lẻ khi có thể dùng versioning

Nên:
  - Dùng versioned filenames (hash trong URL) → không cần invalidate
  - Invalidate chỉ HTML files sau deploy (thường < 100 files)
  - 1,000 paths/tháng miễn phí → đủ cho hầu hết deployments
```

### 6. Giám Sát và Alert Chi Phí

```bash
# Tạo CloudWatch alert khi chi phí vượt ngưỡng
aws cloudwatch put-metric-alarm \
  --alarm-name "CloudFront-HighCost" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:123456789:billing-alerts"
```

---

## 8. Performance Checklist

### ✅ Cấu Hình Distribution

```
□ HTTP version: http2and3 (bật cả HTTP/2 và HTTP/3)
□ Compress objects: Yes
□ Default root object: index.html (nếu static site)
□ SSL minimum protocol: TLSv1.2_2021
□ Price Class: Phù hợp với user base

Cache Behaviors:
□ Static assets (/static/*): TTL 1 năm, CachingOptimized
□ HTML files (/*): TTL 5 phút, Custom
□ API (/api/*): CachingDisabled, AllViewer policy
□ Fonts (/fonts/*): TTL 1 năm, CORS headers
```

### ✅ Origin Optimization

```
□ S3 origin: Dùng S3 Regional Domain Name (không dùng global endpoint)
  → my-bucket.s3.us-east-1.amazonaws.com (đúng)
  → my-bucket.s3.amazonaws.com (có thể chậm hơn)

□ ALB origin: Connection timeout phù hợp (10-30s)
□ Origin Shield: Bật nếu traffic global + origin tập trung 1 region
□ Origin response timeout: Phù hợp với backend (không để quá ngắn)
□ Origin custom headers: Bảo vệ ALB/EC2 origin
```

### ✅ Cache Headers Từ Origin

```
□ Static assets: Cache-Control: public, max-age=31536000, immutable
□ HTML: Cache-Control: public, max-age=300
□ API public: Cache-Control: public, max-age=60
□ API private: Cache-Control: private, no-store
□ ETag / Last-Modified: Có để support conditional GET
```

### ✅ Security

```
□ OAC: Bật cho S3 origin (không dùng OAI)
□ S3 bucket: Block All Public Access
□ WAF: Attach WebACL nếu cần protection
□ Geo-restriction: Nếu có yêu cầu compliance
□ HTTPS only: Redirect HTTP → HTTPS
□ Response headers: Security headers policy
□ Signed URLs/Cookies: Nếu có premium/protected content
```

### ✅ Monitoring

```
□ Access logs: Bật → S3 (phân tích sau)
□ CloudWatch alarms:
    - 5xxErrorRate > 1%
    - CacheHitRate < 70%
    - OriginLatency > 2000ms
□ Real-time logs: Bật nếu cần monitoring real-time
□ Cost alerts: Budget alert cho CloudFront
```

---

## 9. Câu Hỏi Phỏng Vấn

### Q1: Làm thế nào để tối ưu cache hit ratio cho CloudFront?

**Trả lời mẫu:**
> "Tôi tiếp cận từ nhiều góc độ. Đầu tiên, thiết kế cache key đúng — loại bỏ query params như utm_source, session tokens không ảnh hưởng nội dung. Thứ hai, dùng versioned filenames (hash trong URL) cho CSS/JS để TTL có thể set 1 năm. Thứ ba, tách Cache Behaviors theo loại nội dung — static assets, HTML, API có TTL khác nhau. Thứ tư, cấu hình Cache-Control headers chính xác từ origin. Cuối cùng, xem xét Origin Shield nếu traffic toàn cầu nhưng origin tập trung 1 region."

### Q2: Price Class ảnh hưởng đến performance như thế nào?

**Trả lời:**
> "Price Class quyết định Edge Locations nào được dùng. PriceClass_100 chỉ dùng US và EU edges — người dùng ở Việt Nam sẽ được serve từ edge ở châu Âu hoặc Mỹ, thêm 100-200ms latency. PriceClass_All dùng edge Singapore/Tokyo cho user Việt Nam — latency có thể < 50ms. Quyết định phụ thuộc vào distribution địa lý của user base và budget. Tôi luôn phân tích traffic report trước khi chọn."

### Q3: Origin Shield là gì và khi nào dùng?

**Trả lời:**
> "Origin Shield là tầng cache bổ sung giữa Regional Edge Caches và Origin. Khi nhiều edges cùng miss cache và gửi request đến origin (gọi là 'cache stampede'), Origin Shield collapse chúng thành một request duy nhất. Đặc biệt hữu ích khi: origin ở một region nhưng traffic từ nhiều nơi trên thế giới, origin expensive để scale (EC2, database-backed APIs), hoặc nội dung ít thay đổi nhưng traffic burst. Chi phí thêm $0.0075/10,000 requests — nhỏ so với tiết kiệm ở origin."

### Q4: Compression trong CloudFront hoạt động như thế nào?

**Trả lời:**
> "CloudFront hỗ trợ Gzip và Brotli. Khi bật 'Compress objects automatically', CloudFront kiểm tra Accept-Encoding header từ browser. Nếu browser hỗ trợ Brotli (hầu hết browser hiện đại), CloudFront ưu tiên Brotli — nén tốt hơn Gzip ~20-26%. CloudFront compress tại edge nếu origin trả uncompressed, hoặc cache bản đã nén từ origin. Cache key được vary theo encoding — bản Gzip và Brotli là hai cache entries khác nhau. Với text content, compression giảm bandwidth 60-76%, trực tiếp giảm chi phí data transfer."

---

**Hoàn Thành Section:** [05-cdn-cloudfront/](./README.md) — CloudFront CDN

**Tiếp Theo Gợi Ý:**
- [06-connectivity/README.md](../06-connectivity/README.md) — Kết nối VPN, Direct Connect
- [07-advanced-networking/1-vpc-endpoints.md](../07-advanced-networking/1-vpc-endpoints.md) — VPC Endpoints & PrivateLink
