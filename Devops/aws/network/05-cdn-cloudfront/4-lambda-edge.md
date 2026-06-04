# Lambda@Edge & CloudFront Functions — Compute Tại Tầng Edge

> **Lambda@Edge** và **CloudFront Functions** cho phép chạy code tại các Edge Location của CloudFront — xử lý request/response mà không cần đến origin server. Đây là công cụ mạnh mẽ để cá nhân hóa nội dung, bảo mật, và tối ưu performance tại biên mạng (edge computing).

---

## 📚 Mục Lục

1. [Edge Computing Là Gì?](#1-edge-computing-là-gì)
2. [4 Event Triggers (Điểm Kích Hoạt)](#2-4-event-triggers-điểm-kích-hoạt)
3. [CloudFront Functions](#3-cloudfront-functions)
4. [Lambda@Edge](#4-lambdaedge)
5. [So Sánh: CloudFront Functions vs Lambda@Edge](#5-so-sánh-cloudfront-functions-vs-lambdaedge)
6. [Use Cases Thực Tế](#6-use-cases-thực-tế)
7. [Giới Hạn & Lưu Ý](#7-giới-hạn--lưu-ý)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Edge Computing Là Gì?

### Vấn Đề Của Server-Side Processing

```
Tình huống: Thêm security headers vào mọi response

Không có Edge Compute:
  User (Tokyo) → Edge Location (Tokyo) → Origin (US-East, 150ms) → add headers → User

Với Edge Compute:
  User (Tokyo) → Edge Location (Tokyo) → add headers ngay tại edge → User
                                         (< 1ms, không chạm origin)
```

### Edge Computing Trong CloudFront

```
CloudFront Edge Layer
  │
  ├── CloudFront Functions: Lightweight, ~1ms, tại mọi PoP
  │   └── Dùng cho: URL rewrites, header manipulation đơn giản
  │
  └── Lambda@Edge: Full Node.js/Python, tại Regional Edge Cache
      └── Dùng cho: Authentication, A/B testing, complex logic
```

---

## 2. 4 Event Triggers (Điểm Kích Hoạt)

CloudFront có 4 điểm trong vòng đời request nơi có thể chèn code:

```
User
  │
  │ 1. Viewer Request
  │    (Sau khi CloudFront nhận request từ user,
  │     TRƯỚC khi kiểm tra cache)
  ▼
CloudFront Cache
  │
  │ Cache HIT → trả thẳng về user (qua Viewer Response)
  │ Cache MISS → tiếp tục đến origin
  │
  │ 2. Origin Request
  │    (TRƯỚC khi CloudFront gửi request đến origin,
  │     chỉ khi cache miss)
  ▼
Origin (S3/ALB/EC2)
  │
  │ 3. Origin Response
  │    (SAU khi CloudFront nhận response từ origin,
  │     TRƯỚC khi cache và trả về)
  ▼
CloudFront Cache (lưu response)
  │
  │ 4. Viewer Response
  │    (SAU khi CloudFront chuẩn bị trả response về user,
  │     bất kể cache hit hay miss)
  ▼
User
```

### Khi Nào Dùng Event Nào?

| Event               | Tần Suất Chạy             | Use Case Chính                          |
| ------------------- | ------------------------- | --------------------------------------- |
| **Viewer Request**  | Mọi request (kể cả cache hit) | Auth check, URL normalization, A/B test |
| **Origin Request**  | Chỉ cache miss             | Fetch thêm data, modify request đến origin |
| **Origin Response** | Chỉ cache miss             | Thêm headers vào response trước khi cache |
| **Viewer Response** | Mọi request               | Security headers, logging, analytics    |

---

## 3. CloudFront Functions

### Đặc Điểm

```
Runtime: JavaScript (ECMAScript 5.1 — không có ES6+ full)
Giới hạn thời gian: 1ms (execution time)
Memory: 2MB
Package size: 10KB
Trigger: Viewer Request, Viewer Response ONLY
Giá: $0.10 per 1 triệu invocations
Scale: Hàng tỷ requests/giây
Deploy tại: Tất cả Edge Locations (PoP)
```

### Ví Dụ 1: URL Normalization (Chuẩn Hóa URL)

```javascript
// Redirect URL có trailing slash → không có trailing slash
// Mục đích: tránh duplicate cache entries
// /about/ và /about → cùng nội dung nhưng 2 cache entries khác nhau

function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // Thêm index.html cho thư mục (S3 static website)
    if (uri.endsWith('/')) {
        request.uri = uri + 'index.html';
    }
    // Loại bỏ trailing slash (trừ root /)
    else if (uri.endsWith('/') && uri.length > 1) {
        return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: { location: { value: uri.slice(0, -1) } }
        };
    }

    return request;
}
```

### Ví Dụ 2: Thêm Security Headers

```javascript
// Thêm các security headers vào mọi response
function handler(event) {
    var response = event.response;
    var headers = response.headers;

    headers['strict-transport-security'] = {
        value: 'max-age=31536000; includeSubDomains; preload'
    };
    headers['x-content-type-options'] = { value: 'nosniff' };
    headers['x-frame-options'] = { value: 'DENY' };
    headers['x-xss-protection'] = { value: '1; mode=block' };
    headers['referrer-policy'] = {
        value: 'strict-origin-when-cross-origin'
    };
    headers['permissions-policy'] = {
        value: 'camera=(), microphone=(), geolocation=()'
    };

    return response;
}
```

### Ví Dụ 3: Redirect Theo Country Header

```javascript
// CloudFront tự động thêm CloudFront-Viewer-Country header
function handler(event) {
    var request = event.request;
    var country = request.headers['cloudfront-viewer-country'];

    if (country && country.value === 'JP') {
        return {
            statusCode: 302,
            statusDescription: 'Found',
            headers: {
                location: { value: 'https://jp.example.com' + request.uri }
            }
        };
    }

    return request;
}
```

### Ví Dụ 4: API Key Validation Đơn Giản

```javascript
function handler(event) {
    var request = event.request;
    var apiKey = request.headers['x-api-key'];

    // Hardcoded key chỉ dùng cho demo
    // Production: dùng Lambda@Edge với Secrets Manager
    if (!apiKey || apiKey.value !== 'my-api-key-value') {
        return {
            statusCode: 401,
            statusDescription: 'Unauthorized',
            body: JSON.stringify({ error: 'Invalid API key' })
        };
    }

    return request;
}
```

---

## 4. Lambda@Edge

### Đặc Điểm

```
Runtime: Node.js 18.x, Python 3.11 (và các version mới hơn)
Giới hạn thời gian:
  - Viewer events: 5 giây
  - Origin events: 30 giây
Memory: 128MB - 10GB
Package size: 1MB (compressed) / 50MB (unzipped)
Trigger: Tất cả 4 events
Giá: $0.60 per 1 triệu requests + duration
Deploy tại: us-east-1 (AWS tự replicate ra các Regional Edge Caches)
VPC access: Không (stateless, không có private network access)
```

> **Lưu ý Deploy:** Lambda@Edge phải tạo tại **us-east-1**, tương tự ACM certificate cho CloudFront.

### Ví Dụ 1: JWT Authentication (Viewer Request)

```javascript
// Node.js — Kiểm tra JWT token trong cookie
const jwt = require('jsonwebtoken'); // Phải bundle vào deployment package

exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;

    // Lấy JWT từ cookie
    const cookieHeader = headers.cookie?.[0]?.value || '';
    const token = cookieHeader.split(';')
        .find(c => c.trim().startsWith('auth_token='))
        ?.split('=')[1];

    if (!token) {
        return buildUnauthorizedResponse();
    }

    try {
        // Verify JWT (public key phải bundle trong package)
        jwt.verify(token, process.env.JWT_PUBLIC_KEY);
        return request; // Allow request
    } catch (e) {
        return buildUnauthorizedResponse();
    }
};

function buildUnauthorizedResponse() {
    return {
        status: '401',
        statusDescription: 'Unauthorized',
        body: JSON.stringify({ error: 'Authentication required' }),
        headers: {
            'content-type': [{ key: 'Content-Type', value: 'application/json' }],
            'www-authenticate': [{ key: 'WWW-Authenticate', value: 'Bearer' }]
        }
    };
}
```

### Ví Dụ 2: A/B Testing (Origin Request)

```javascript
// Phân chia traffic: 80% → origin A, 20% → origin B
exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;

    // Kiểm tra cookie A/B group đã gán chưa
    const cookieHeader = headers.cookie?.[0]?.value || '';
    const existingGroup = cookieHeader.match(/ab_group=(a|b)/)?.[1];

    let group = existingGroup;
    if (!group) {
        // Phân ngẫu nhiên: 80% A, 20% B
        group = Math.random() < 0.8 ? 'a' : 'b';
    }

    // Thay đổi origin dựa trên group
    if (group === 'b') {
        headers['x-origin-override'] = [{
            key: 'X-Origin-Override',
            value: 'origin-b.example.com'
        }];
        // Hoặc thay đổi origin URL trực tiếp trong request
        request.origin.custom.domainName = 'origin-b.example.com';
    }

    return request;
};
```

### Ví Dụ 3: Dynamic Image Resizing (Origin Response)

```javascript
// Resize ảnh theo query param ?width=300
const Sharp = require('sharp'); // Phải bundle native module

exports.handler = async (event) => {
    const response = event.Records[0].cf.response;
    const request = event.Records[0].cf.request;

    // Chỉ resize images
    if (!response.headers['content-type']?.[0]?.value?.startsWith('image/')) {
        return response;
    }

    const params = new URLSearchParams(request.querystring);
    const targetWidth = parseInt(params.get('width') || '0');

    if (!targetWidth || targetWidth > 2000) {
        return response;
    }

    try {
        const imageBuffer = Buffer.from(response.body, response.bodyEncoding);
        const resized = await Sharp(imageBuffer)
            .resize(targetWidth)
            .toBuffer();

        response.body = resized.toString('base64');
        response.bodyEncoding = 'base64';
        response.headers['content-length'] = [{
            key: 'Content-Length',
            value: String(resized.length)
        }];
    } catch (e) {
        // Trả về ảnh gốc nếu resize lỗi
    }

    return response;
};
```

### Ví Dụ 4: Fetch Data Từ DynamoDB (Origin Request)

```javascript
// Thêm user-specific data vào request trước khi gửi đến origin
const { DynamoDBClient, GetItemCommand } = require('@aws-sdk/client-dynamodb');

const ddb = new DynamoDBClient({ region: 'us-east-1' });

exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;

    const userId = headers['x-user-id']?.[0]?.value;
    if (!userId) return request;

    try {
        const { Item } = await ddb.send(new GetItemCommand({
            TableName: 'UserPreferences',
            Key: { userId: { S: userId } }
        }));

        if (Item) {
            // Thêm preference vào header để origin server xử lý
            headers['x-user-tier'] = [{ key: 'X-User-Tier', value: Item.tier.S }];
            headers['x-user-region'] = [{ key: 'X-User-Region', value: Item.region.S }];
        }
    } catch (e) {
        // Log lỗi nhưng vẫn forward request
        console.error('DynamoDB error:', e.message);
    }

    return request;
};
```

---

## 5. So Sánh: CloudFront Functions vs Lambda@Edge

| Tiêu Chí                   | CloudFront Functions         | Lambda@Edge                      |
| -------------------------- | ---------------------------- | -------------------------------- |
| **Runtime**                | JavaScript (ES5.1 giới hạn)  | Node.js, Python (full)           |
| **Execution time**         | < 1ms                        | Viewer: 5s / Origin: 30s         |
| **Memory**                 | 2MB                          | 128MB - 10GB                     |
| **Package size**           | 10KB                         | 1MB (compressed)                 |
| **Events supported**       | Viewer Request/Response       | Tất cả 4 events                  |
| **Network access**         | ❌ Không                      | ❌ Không (không có VPC)           |
| **External calls**         | ❌ Không                      | ✅ Có (DynamoDB, S3, Secrets...)  |
| **Third-party packages**   | ❌ Không                      | ✅ Có (npm, pip)                  |
| **Giá**                    | $0.10/triệu requests         | $0.60/triệu + $0.00000625/GB-s   |
| **Deploy tại**             | Mọi PoP toàn cầu             | Regional Edge Caches             |
| **Concurrency limits**     | Rất cao (tự động scale)      | Theo Lambda concurrency limits   |
| **Cold start**             | Không có                     | Có (nhưng giảm thiểu tại edge)   |
| **Khi nào dùng**           | Simple transforms, <1ms      | Complex logic, external APIs     |

### Quyết Định Nên Dùng Cái Nào?

```
Dùng CloudFront Functions khi:
  ✅ URL rewrite/redirect đơn giản
  ✅ Thêm/xóa headers
  ✅ Normalize cache keys
  ✅ Simple request validation
  ✅ Cần chi phí thấp nhất
  ✅ Cần scale không giới hạn

Dùng Lambda@Edge khi:
  ✅ JWT/OAuth authentication
  ✅ A/B testing phức tạp
  ✅ Cần gọi external APIs (DynamoDB, Secrets Manager)
  ✅ Image transformation/resizing
  ✅ Complex business logic (> 1ms)
  ✅ Cần npm packages (JWT lib, Sharp, v.v.)
  ✅ Origin events (origin request/response)
```

---

## 6. Use Cases Thực Tế

### 6.1 SPA (Single Page Application) — React/Vue/Angular

```
Vấn đề: /about → S3 object không tồn tại → 403
Giải pháp: URL rewrite tất cả non-file paths → /index.html

CloudFront Function (Viewer Request):
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // Nếu không có file extension → SPA route → /index.html
    if (!uri.includes('.') || uri.endsWith('/')) {
        request.uri = '/index.html';
    }

    return request;
}
```

### 6.2 Multi-Language Redirect

```
CloudFront Function phát hiện Accept-Language header
→ Redirect đến /vi/, /en/, /ja/ tương ứng

function handler(event) {
    var request = event.request;
    var acceptLang = request.headers['accept-language']?.value || '';
    var uri = request.uri;

    // Chỉ redirect nếu ở root
    if (uri !== '/') return request;

    var lang = 'en'; // default
    if (acceptLang.includes('vi')) lang = 'vi';
    else if (acceptLang.includes('ja')) lang = 'ja';

    return {
        statusCode: 302,
        headers: { location: { value: '/' + lang + '/' } }
    };
}
```

### 6.3 Protected Premium Content

```
Flow:
  1. User login → Backend tạo JWT → Set cookie
  2. User request /premium/video.mp4
  3. Lambda@Edge (Viewer Request):
     - Đọc JWT cookie
     - Verify signature
     - Check subscription tier trong JWT claims
     - Allow: forward request
     - Deny: redirect đến /upgrade
```

### 6.4 Real-time Personalization

```
Lambda@Edge (Origin Request):
  1. Lấy user ID từ cookie
  2. Query DynamoDB (nearest region) → user preferences
  3. Thêm headers: X-User-Segment, X-Feature-Flags
  4. Forward đến origin với context đầy đủ

Origin Server nhận headers → render personalized content
```

### 6.5 Bot Detection

```
CloudFront Function (Viewer Request):
  1. Kiểm tra User-Agent known bot patterns
  2. Kiểm tra missing common browser headers
  3. Kiểm tra rate indicators trong headers

Lambda@Edge (Viewer Request — phức tạp hơn):
  1. Query WAF token từ request
  2. Gọi Fraud Detection API
  3. Block, challenge, hoặc allow
```

---

## 7. Giới Hạn & Lưu Ý

### Các Hạn Chế Quan Trọng

```
Lambda@Edge KHÔNG THỂ:
  ❌ Truy cập VPC resources (RDS, ElastiCache, private ALB)
  ❌ Dùng Environment Variables (không có cơ chế chuẩn)
  ❌ Chạy quá 5s (viewer) / 30s (origin)
  ❌ Truy cập Layers (phải bundle tất cả trong package)
  ❌ Có reserved concurrency
  ❌ Có EFS access
  ❌ Được deploy ngoài us-east-1

CloudFront Functions KHÔNG THỂ:
  ❌ Gọi external services
  ❌ Dùng npm packages
  ❌ Chạy quá 1ms (hard limit)
  ❌ Attach vào Origin Request/Response events
```

### Tips Tránh Lỗi

```
1. Lambda@Edge Package Size:
   Limit: 1MB compressed / 50MB unzipped
   → Bundle chỉ dependencies cần thiết
   → Native modules (Sharp): dùng Amazon Linux 2 build environment

2. Region cho Lambda@Edge:
   → PHẢI tạo tại us-east-1
   → Không thể deploy Lambda function ở region khác cho Lambda@Edge

3. Environment Variables:
   → Lambda@Edge không support env vars trực tiếp
   → Giải pháp: hardcode config, hoặc fetch từ SSM/Secrets Manager lúc init

4. Debugging:
   → CloudWatch Logs được tạo tại region của Edge Location
   → Phải kiểm tra log tại region mà request đi qua
   → Dùng structured logging (JSON) để dễ query với CloudWatch Insights

5. Pricing shock:
   → Lambda@Edge chạy tại NHIỀU regions → billing từ nhiều regions
   → Monitor chi phí cẩn thận — có thể tốn hơn Lambda thường
```

### Cold Start Và Performance

```
Lambda@Edge có cold start nhưng ít hơn Lambda thường vì:
  - AWS keep warm nhiều instances hơn tại edge
  - Traffic thường đủ lớn để giữ warm
  - Sử dụng Provisioned Concurrency nếu cần (tốn phí)

CloudFront Functions:
  - Không có cold start (V8 JavaScript engine luôn sẵn sàng)
  - Sub-millisecond latency
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: Lambda@Edge khác Lambda thông thường như thế nào?

**Trả lời mẫu:**
> "Lambda@Edge chạy tại Regional Edge Caches của CloudFront — gần người dùng hơn, không phải tại AWS Region cụ thể. Hạn chế: không có VPC access, không có environment variables native, phải deploy tại us-east-1, package nhỏ hơn (1MB vs 50MB). Lợi thế: latency thấp hơn vì gần người dùng, tự động replication ra các edge locations, và có thể modify CloudFront request/response lifecycle."

### Q2: CloudFront Functions vs Lambda@Edge — khi nào dùng cái nào?

**Trả lời:**
> "CloudFront Functions cho lightweight operations dưới 1ms: URL rewrites, thêm security headers, normalize cache keys. Chi phí rất thấp ($0.10/triệu) và không có cold start. Lambda@Edge khi cần runtime đầy đủ, external API calls (DynamoDB, Secrets Manager), hoặc complex business logic như JWT authentication, A/B testing với state tracking. Lambda@Edge tốn hơn nhưng mạnh hơn nhiều."

### Q3: Làm thế nào để debug Lambda@Edge functions?

**Trả lời:**
> "Lambda@Edge logs vào CloudWatch tại region của Edge Location xử lý request — không phải us-east-1 nơi function được deploy. Nếu request đi qua Singapore edge, log sẽ ở ap-southeast-1. Tôi thường dùng console.log với structured JSON, sau đó query logs bằng CloudWatch Logs Insights với filter theo request ID. Một trick hữu ích là thêm X-Debug-Region header vào response để biết request đi qua edge nào."

### Q4: Tại sao Lambda@Edge không thể truy cập VPC resources?

**Trả lời:**
> "Lambda@Edge chạy tại hàng chục Regional Edge Cache locations trên toàn cầu. Để access VPC, cần ENI (Elastic Network Interface) trong VPC — nhưng không thể tạo ENI ở tất cả edge locations, đặc biệt vì edges thuộc nhiều regions khác nhau. Kiến trúc stateless của edge compute không tương thích với VPC connectivity. Workaround: gọi VPC resources qua public API endpoint (ALB/API Gateway), hoặc dùng DynamoDB Global Tables cho stateful data."

---

**Tiếp Theo:** [5-performance-cost.md](./5-performance-cost.md) — Price Classes, Compression & Cost Optimization
