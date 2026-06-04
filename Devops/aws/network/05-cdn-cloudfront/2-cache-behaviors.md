# Cache Behaviors — Chính Sách Cache, TTL & Cache Key

> **Cache Behavior** (Hành Vi Cache) là quy tắc trung tâm quyết định CloudFront xử lý từng loại request như thế nào: có cache hay không, cache bao lâu, cần forward gì lên origin, và áp dụng compression nào. Hiểu rõ Cache Behaviors là chìa khóa để tối ưu performance và giảm chi phí CloudFront.

---

## 📚 Mục Lục

1. [Cache Behavior Là Gì?](#1-cache-behavior-là-gì)
2. [Cache Policies — Chính Sách Cache](#2-cache-policies--chính-sách-cache)
3. [Origin Request Policies](#3-origin-request-policies)
4. [Cache Key — Khóa Cache](#4-cache-key--khóa-cache)
5. [TTL — Time To Live](#5-ttl--time-to-live)
6. [Cache Invalidation](#6-cache-invalidation)
7. [Response Headers Policies](#7-response-headers-policies)
8. [Thiết Kế Cache Behaviors Thực Tế](#8-thiết-kế-cache-behaviors-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Cache Behavior Là Gì?

**Cache Behavior** là tập hợp cấu hình được áp dụng khi request URL khớp với một **path pattern** (mẫu đường dẫn). Một Distribution có:
- Một **Default Cache Behavior** (`*`) — áp dụng khi không có rule nào khớp
- Nhiều **Cache Behaviors bổ sung** — matching theo order từ trên xuống dưới

### Thứ Tự Matching

```
Request: GET /api/users?page=2

Cache Behaviors (theo thứ tự ưu tiên):
  1. /api/*        ← MATCH! → dùng rule này
  2. /images/*.jpg
  3. /static/*
  4. *             ← Default (không dùng vì đã match ở bước 1)
```

### Cấu Hình Trong Một Cache Behavior

```
Path Pattern: /images/*

Viewer Protocol Policy:
  - Redirect HTTP to HTTPS ✅
  - HTTPS only
  - HTTP and HTTPS

Allowed HTTP Methods:
  - GET, HEAD (chỉ đọc — cho static)
  - GET, HEAD, OPTIONS (cần pre-flight CORS)
  - GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE (API)

Restrict Viewer Access: No | Yes (Signed URLs/Cookies)

Cache Policy: CachingOptimized
Origin Request Policy: CORS-S3Origin

Compress Objects Automatically: Yes ✅

Lambda@Edge / CloudFront Functions: Optional
```

---

## 2. Cache Policies — Chính Sách Cache

**Cache Policy** quyết định **những gì tạo thành cache key** và **TTL defaults**. Đây là phần quan trọng nhất của Cache Behavior.

### Managed Cache Policies (AWS Cung Cấp Sẵn)

#### 2.1 CachingOptimized (Khuyến Nghị Cho Static)

```
ID: 658327ea-f89d-4fab-a63d-7e88639e58f6
TTL: Min=1s, Default=86400s (24h), Max=31536000s (1 năm)
Cache Key:
  - Headers: none
  - Cookies: none
  - Query Strings: none
Dùng cho: S3 static assets, images, CSS, JS
```

**Lý do:** Loại trừ mọi thứ khỏi cache key → nhiều người dùng share cùng cache entry → hit rate cao nhất.

#### 2.2 CachingDisabled (Không Cache)

```
ID: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
TTL: 0 (không cache)
Dùng cho: API calls, dynamic content, authenticated requests
```

#### 2.3 CachingOptimizedForUncompressedObjects

```
Giống CachingOptimized nhưng không apply compression
Dùng cho: Files đã được nén sẵn (ZIP, JPEG, MP4...)
```

#### 2.4 Elemental-MediaPackage

```
Dùng cho: AWS Elemental MediaPackage (video streaming)
TTL ngắn — phù hợp live streaming segments
```

#### 2.5 UseOriginCacheControlHeaders (Dùng Header Từ Origin)

```
TTL: Lấy từ Cache-Control / Expires header của origin
Cache Key: Lấy từ origin headers
Dùng khi: Origin server tự quản lý TTL chính xác
```

### Custom Cache Policy — Tạo Riêng

Khi managed policies không phù hợp, tạo custom policy:

```json
{
  "Name": "MyAPIPolicy",
  "DefaultTTL": 0,
  "MaxTTL": 1,
  "MinTTL": 0,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": {
      "HeaderBehavior": "whitelist",
      "Headers": ["Authorization", "Accept-Language"]
    },
    "CookiesConfig": {
      "CookieBehavior": "none"
    },
    "QueryStringsConfig": {
      "QueryStringBehavior": "whitelist",
      "QueryStrings": ["version", "lang"]
    }
  }
}
```

---

## 3. Origin Request Policies

**Origin Request Policy** quyết định **những gì được forward lên origin** khi cache miss — độc lập với cache key.

> **Phân biệt quan trọng:**
> - **Cache Policy**: Quyết định cache key (những gì làm "khác nhau" giữa các request)
> - **Origin Request Policy**: Quyết định những gì được gửi lên origin (headers, cookies, query strings)
>
> Bạn có thể cache theo ít tiêu chí nhưng vẫn forward nhiều thông tin lên origin.

### Managed Origin Request Policies

#### CORS-S3Origin

```
Dùng cho: S3 bucket với CORS enabled
Forward headers: Origin, Access-Control-Request-Headers,
                 Access-Control-Request-Method
```

#### CORS-CustomOrigin

```
Dùng cho: Custom origin với CORS
Forward headers: Giống CORS-S3Origin + Referer, Accept
```

#### AllViewer

```
Forward: Tất cả headers, cookies, query strings từ viewer
Dùng cho: Dynamic content cần tất cả context
```

#### UserAgentRefererHeaders

```
Forward headers: User-Agent, Referer
Dùng cho: Analytics, A/B testing tại origin
```

#### AllViewerAndCloudFrontHeaders-2022-06

```
Forward: Tất cả viewer headers + CloudFront headers bổ sung
CloudFront headers bổ sung: CloudFront-Viewer-Country, CloudFront-Viewer-City,
                            CloudFront-Is-Mobile-Viewer, v.v.
```

---

## 4. Cache Key — Khóa Cache

**Cache Key** là định danh duy nhất của một cache entry. Hai request có cùng cache key → share cùng cached response.

### Thành Phần Của Cache Key

```
Default cache key (tối thiểu):
  scheme://host/path
  ↓
  https://d1234.cloudfront.net/images/logo.png

Cache key mở rộng (tuỳ cấu hình):
  scheme://host/path?query_strings + headers + cookies
```

### Ví Dụ Thực Tế

```
Cache Policy: include query string "lang"

Request 1: GET /page.html?lang=vi&session=abc123
  Cache Key: https://example.cloudfront.net/page.html?lang=vi

Request 2: GET /page.html?lang=en&session=xyz789
  Cache Key: https://example.cloudfront.net/page.html?lang=en

→ Hai cache entries khác nhau (đúng — nội dung khác ngôn ngữ)
→ "session" bị loại khỏi cache key → không ảnh hưởng cache
→ "session" vẫn được forward lên origin nếu cấu hình trong Origin Request Policy
```

### Cache Key Tệ vs Tốt

```
Tệ — Cache key quá rộng:
  Include tất cả query strings
  Request: /api/data?user=123&token=abc&_=timestamp
  → Mỗi request có cache key khác nhau → hit rate = 0%

Tệ — Cache key quá hẹp:
  Không include "Accept-Language"
  Request EN/VI/JP đều share cùng cache entry
  → Người dùng JP nhận nội dung tiếng Anh

Tốt — Cache key vừa đủ:
  Include: "Accept-Language", "lang" query string
  Exclude: "session_token", "_timestamp", "utm_*"
  → Cache theo ngôn ngữ, ignore session và tracking params
```

---

## 5. TTL — Time To Live

**TTL** (Time To Live — Thời Gian Sống) là khoảng thời gian CloudFront giữ một cache entry trước khi check lại origin.

### Ba Giá Trị TTL

```
MinTTL (TTL Tối Thiểu):
  - CloudFront cache ÍT NHẤT bao lâu
  - Ghi đè nếu origin header thấp hơn
  - Default: 0 (cho phép origin kiểm soát)

DefaultTTL (TTL Mặc Định):
  - Dùng khi origin không trả về Cache-Control hoặc Expires
  - Default: 86400 (24 giờ)

MaxTTL (TTL Tối Đa):
  - CloudFront cache NHIỀU NHẤT bao lâu
  - Ghi đè nếu origin header cao hơn
  - Default: 31536000 (1 năm)
```

### Ưu Tiên TTL

```
Thứ tự quyết định TTL thực tế:

1. Cache-Control: max-age=3600 từ origin
   → TTL = max(MinTTL, min(3600, MaxTTL))

2. Cache-Control: no-cache, no-store, private
   → TTL = 0 (không cache, bất kể MinTTL)
   → Trừ khi MinTTL > 0 → MinTTL override

3. Expires: Thu, 01 Jan 2026 00:00:00 GMT
   → TTL tính từ thời điểm hiện tại đến Expires
   → Cũng bị ràng buộc bởi Min/MaxTTL

4. Không có header nào → DefaultTTL áp dụng
```

### Cache-Control Headers Quan Trọng

```http
# Cache lâu dài — cho static assets có versioning
Cache-Control: public, max-age=31536000, immutable

# Cache ngắn, revalidate với server
Cache-Control: public, max-age=3600, must-revalidate

# Không cache (API responses)
Cache-Control: no-cache, no-store, must-revalidate

# Cache nhưng revalidate (conditional GET)
Cache-Control: no-cache (buộc revalidate mỗi lần)
ETag: "abc123"
```

### Chiến Lược TTL Theo Loại Nội Dung

| Loại Nội Dung              | TTL Khuyến Nghị | Ghi Chú                              |
| -------------------------- | --------------- | ------------------------------------ |
| Images (có hash trong tên) | 1 năm           | `logo.a3f4b2.png` — immutable        |
| CSS/JS (có hash)           | 1 năm           | `app.7d8e9f.js` — immutable          |
| HTML                       | 5-60 phút       | Thường xuyên thay đổi                |
| API GET responses          | 0-60 giây       | Tùy freshness requirement            |
| Video/Audio                | 7-30 ngày       | Nội dung lớn, ít thay đổi            |
| Fonts                      | 1 năm           | Rất ít thay đổi                      |
| robots.txt / sitemap.xml   | 24 giờ          | Thay đổi định kỳ                     |

---

## 6. Cache Invalidation

**Invalidation** (Vô Hiệu Cache) là thao tác chủ động xóa cache tại tất cả Edge Locations, bất kể TTL còn lại.

### Khi Nào Cần Invalidate?

```
✅ Cần invalidate:
  - Bug fix khẩn cấp trên static file
  - Thay đổi nội dung HTML/CSS/JS không có versioning
  - Cập nhật security policy (robots.txt, sitemap)
  - Rollback deployment

❌ Không cần invalidate (thay thế tốt hơn):
  - File đã có hash trong tên → tên file khác → cache key khác
  - Nội dung API dynamic → dùng no-cache header
  - Nội dung TTL ngắn → chờ expire tự nhiên
```

### Cách Thực Hiện Invalidation

```bash
# Invalidate một file cụ thể
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/index.html"

# Invalidate nhiều files
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/css/main.css" "/js/app.js" "/index.html"

# Invalidate tất cả (dùng thận trọng)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# Kiểm tra trạng thái
aws cloudfront get-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --id I3UN6WX5RRO2AG
```

### Chi Phí Invalidation

```
1.000 paths đầu tiên mỗi tháng: Miễn phí
Sau đó: $0.005 per path

"/images/*" đếm là 1 path (wildcard)
"/*" đếm là 1 path
"/css/main.css" đếm là 1 path

→ Luôn dùng wildcard khi invalidate nhiều files cùng thư mục
```

### Versioned Files — Tránh Invalidation

```
Tệ — không có versioning:
  styles.css → phải invalidate thủ công sau mỗi deploy

Tốt — có hash versioning:
  styles.a3f4b2c1.css → tên file thay đổi → cache key mới
  → Không cần invalidate
  → Old files tự expire theo TTL
  → Rollback dễ dàng (URL cũ vẫn còn trong cache)
```

---

## 7. Response Headers Policies

**Response Headers Policy** thêm hoặc chỉnh sửa HTTP headers trong response gửi về viewer — không ảnh hưởng origin.

### Managed Response Headers Policies

#### SecurityHeadersPolicy

```
Thêm các security headers quan trọng:
  Strict-Transport-Security: max-age=31536000; includeSubDomains
  X-Content-Type-Options: nosniff
  X-Frame-Options: SAMEORIGIN
  X-XSS-Protection: 1; mode=block
  Referrer-Policy: strict-origin-when-cross-origin
```

#### CORS-With-Preflight

```
Thêm CORS headers:
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET, HEAD, OPTIONS
  Access-Control-Allow-Headers: *
  Access-Control-Max-Age: 600
```

#### SimpleCORS

```
Access-Control-Allow-Origin: *
(Đơn giản nhất — chỉ dùng khi không cần preflight)
```

### Custom Response Headers

```json
{
  "Name": "MySecurityHeaders",
  "SecurityHeadersConfig": {
    "StrictTransportSecurity": {
      "Override": true,
      "AccessControlMaxAgeSec": 31536000,
      "IncludeSubdomains": true,
      "Preload": true
    },
    "ContentTypeOptions": {
      "Override": true
    },
    "FrameOptions": {
      "FrameOption": "DENY",
      "Override": true
    }
  },
  "CustomHeadersConfig": {
    "Items": [
      {
        "Header": "X-Custom-App-Version",
        "Value": "2.1.0",
        "Override": false
      }
    ]
  }
}
```

---

## 8. Thiết Kế Cache Behaviors Thực Tế

### Ví Dụ: SPA (Single Page Application) + API

```
Distribution: app.example.com

Cache Behavior 1: /api/*
  Origin: ALB (backend)
  Cache Policy: CachingDisabled
  Origin Request Policy: AllViewer (forward Authorization header)
  Viewer Protocol: HTTPS only
  Methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE

Cache Behavior 2: /static/*
  Origin: S3 (assets)
  Cache Policy: CachingOptimized (TTL 1 năm)
  Origin Request Policy: CORS-S3Origin
  Viewer Protocol: Redirect HTTP to HTTPS
  Methods: GET, HEAD

Cache Behavior 3: /media/*
  Origin: S3 (media files)
  Cache Policy: Custom (TTL 7 ngày)
  Viewer Protocol: HTTPS only
  Methods: GET, HEAD
  Compress: No (files đã compressed)

Default Cache Behavior: /*
  Origin: S3 (HTML files)
  Cache Policy: Custom (TTL 5 phút — HTML thay đổi thường)
  Origin Request Policy: CORS-S3Origin
  Viewer Protocol: Redirect HTTP to HTTPS
  Methods: GET, HEAD
  Default root: index.html
```

### Ví Dụ: E-commerce — Phức Tạp

```
/product-images/*  → S3, Cache 30 ngày
/api/products/*    → ALB, Cache 60 giây (product listing)
/api/cart/*        → ALB, No cache (real-time)
/api/auth/*        → ALB, No cache (security)
/checkout/*        → ALB, No cache (session-based)
/static/*          → S3, Cache 1 năm (versioned assets)
/*                 → ALB, Cache 5 phút (HTML pages)
```

### Quyết Định: Cache Hay Không?

```
Should I cache this?

├── Nội dung thay đổi mỗi request (timestamp, user-specific)?
│   └── → KHÔNG cache (hoặc cache với Vary: user-id)
│
├── Nội dung giống nhau cho nhiều users?
│   ├── Thay đổi thường xuyên (< 1 phút)?
│   │   └── → Cache ngắn (10-60 giây) hoặc không cache
│   ├── Thay đổi định kỳ (giờ/ngày)?
│   │   └── → Cache trung bình (1-24 giờ) + invalidate khi deploy
│   └── Hiếm thay đổi?
│       └── → Cache dài + versioning trong filename
│
└── Có authentication? (Authorization header)
    └── → Không cache hoặc cache theo Authorization key
```

---

## 9. Câu Hỏi Phỏng Vấn

### Q1: Cache key là gì và tại sao nó quan trọng?

**Trả lời mẫu:**
> "Cache key là định danh duy nhất của một cached response — thường là URL tối thiểu, có thể mở rộng với query strings, headers, hoặc cookies. Nếu cache key quá rộng, bao gồm session tokens hay timestamps, mỗi request có key khác nhau → không bao giờ có cache hit. Nếu quá hẹp, người dùng khác ngôn ngữ nhận nội dung sai. Thiết kế cache key cân bằng là nghệ thuật: bao gồm những gì thực sự làm khác nhau nội dung, loại trừ những gì chỉ là tracking hay session data."

### Q2: Sự khác biệt giữa Cache Policy và Origin Request Policy?

**Trả lời:**
> "Cache Policy quyết định cache key và TTL — nó kiểm soát khi nào CloudFront serve từ cache. Origin Request Policy quyết định những gì được gửi lên origin khi có cache miss — nó kiểm soát những gì origin nhìn thấy. Ví dụ: tôi có thể không include Authorization header trong cache key (không cache theo user) nhưng vẫn forward nó lên origin để authenticate request."

### Q3: Khi nào dùng Invalidation vs versioned filenames?

**Trả lời:**
> "Versioned filenames là best practice vì: không tốn phí, ngay lập tức hiệu quả (URL mới = cache entry mới), và hỗ trợ rollback tốt. Invalidation phù hợp khi không kiểm soát được tên file — ví dụ HTML files cần update gấp, robots.txt, hoặc emergency hotfix. Trong CI/CD pipeline hiện đại, tôi thường dùng content hash trong filename cho JS/CSS, và invalidation chỉ cho HTML và các files cố định."

### Q4: Làm thế nào để cache API responses mà vẫn an toàn?

**Trả lời:**
> "Với public API responses (cùng nội dung cho mọi user), tôi cache với TTL ngắn — 30-60 giây. Với user-specific responses, tôi không cache hoặc include Authorization header trong cache key (nhưng mỗi token tạo entry riêng → cache ít hiệu quả). Best practice là thiết kế API sao cho public endpoints và user-specific endpoints tách biệt rõ ràng theo URL path."

---

**Tiếp Theo:** [3-origin-access.md](./3-origin-access.md) — OAC, OAI và bảo vệ S3 bucket
