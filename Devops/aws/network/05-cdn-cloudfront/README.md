# CloudFront CDN — Tổng Quan Phân Phối Nội Dung Toàn Cầu

> Amazon CloudFront là CDN (Content Delivery Network — Mạng Phân Phối Nội Dung) toàn cầu của AWS, giúp phân phối nội dung tĩnh, động và streaming đến người dùng cuối với độ trễ thấp nhất thông qua hệ thống Edge Locations (Điểm Biên) rải rác trên khắp thế giới.

---

## 📚 Mục Lục

1. [CloudFront Là Gì?](#cloudfront-là-gì)
2. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
3. [Các Thành Phần Chính](#các-thành-phần-chính)
4. [Use Cases Phổ Biến](#use-cases-phổ-biến)
5. [Lộ Trình Học](#lộ-trình-học)
6. [Tài Liệu Trong Section Này](#tài-liệu-trong-section-này)

---

## CloudFront Là Gì?

**Amazon CloudFront** là dịch vụ CDN được quản lý hoàn toàn bởi AWS, hoạt động theo mô hình **phân phối nội dung theo mạng lưới edge**. Thay vì người dùng kết nối trực tiếp đến Origin Server (Máy Chủ Nguồn) đặt tại một Region AWS cụ thể, CloudFront phục vụ nội dung từ Edge Location gần nhất với người dùng.

### Tại Sao Cần CloudFront?

| Vấn Đề Không Có CloudFront                        | Giải Pháp Với CloudFront                               |
| ------------------------------------------------- | ------------------------------------------------------ |
| Người dùng ở Việt Nam tải file từ server US-East  | Phục vụ từ Edge Location Singapore/Tokyo               |
| Mỗi request đều chạm đến origin server            | Cache tại edge — phần lớn request không tốn tài nguyên |
| DDoS trực tiếp vào EC2/ALB                         | Shield + WAF tại edge, lọc trước khi đến origin        |
| S3 bucket phải public để serve file               | OAC bảo vệ bucket — chỉ CloudFront mới truy cập được  |
| Static asset phân phối chậm trên toàn cầu         | Cache hit ratio >90% — tốc độ tương đương local        |

---

## Kiến Trúc Tổng Thể

```
Người Dùng (Hà Nội)
        │
        │ DNS query → Route 53 trả về IP Edge Location gần nhất
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                  CLOUDFRONT EDGE LAYER                          │
│                                                                 │
│  Edge Location (Singapore)     Edge Location (Tokyo)           │
│  ┌─────────────────────┐       ┌─────────────────────┐         │
│  │  Cache (L1 — PoP)   │       │  Cache (L1 — PoP)   │         │
│  └──────────┬──────────┘       └─────────────────────┘         │
│             │ Cache miss                                        │
│             ▼                                                   │
│  Regional Edge Cache (L2)                                       │
│  ┌──────────────────────────────────┐                           │
│  │  Larger cache — regional scope   │                           │
│  └──────────────────┬───────────────┘                           │
└───────────────────────────────────────────────────────────────  │
                       │ Vẫn miss → fetch from origin             │
                       ▼
             ┌──────────────────────────────────┐
             │        ORIGIN (Nguồn Gốc)         │
             │  S3 Bucket / ALB / EC2 / Custom   │
             └──────────────────────────────────┘
```

### Luồng Request Điển Hình

```
1. Người dùng → DNS lookup cho domain (vd: cdn.example.com)
2. Route 53 → trả IP Edge Location gần nhất (Anycast routing)
3. Edge Location → kiểm tra cache:
   - Cache HIT → trả ngay (< 5ms)
   - Cache MISS → chuyển lên Regional Edge Cache
4. Regional Edge Cache → kiểm tra cache:
   - Cache HIT → trả về (< 20ms)
   - Cache MISS → fetch từ Origin Server
5. Origin → trả nội dung → cache tại edge → trả về người dùng
```

---

## Các Thành Phần Chính

### 1. Distribution (Phân Phối)

Entry point của CloudFront. Mỗi Distribution có:
- **Domain name** tự động: `d1234abcd.cloudfront.net`
- **Custom domain** (CNAME): `cdn.example.com`
- **SSL/TLS certificate** từ ACM (AWS Certificate Manager)
- **One or more Origins** và **Cache Behaviors**

### 2. Origins (Nguồn Gốc)

Nơi CloudFront lấy nội dung gốc khi cache miss:

| Loại Origin           | Ví Dụ                          | Ghi Chú                              |
| --------------------- | ------------------------------ | ------------------------------------ |
| **S3 Bucket**         | `my-bucket.s3.amazonaws.com`   | Cần OAC để bảo vệ bucket             |
| **ALB**               | `my-alb.ap-southeast-1.elb`   | Cho dynamic content                  |
| **EC2 Instance**      | IP hoặc DNS của EC2            | Ít dùng; không có HA tự động         |
| **Custom HTTP**       | Bất kỳ HTTP/HTTPS endpoint     | On-premises, external APIs           |
| **MediaStore/Package**| AWS Media Services             | Cho video streaming                  |

### 3. Cache Behaviors (Hành Vi Cache)

Quy tắc quyết định cách CloudFront xử lý từng loại request:
- Dựa trên **URL path pattern** (vd: `/api/*`, `/images/*`, `*.js`)
- Quy định **TTL** (Time To Live — Thời Gian Cache), **HTTP methods**, **compression**
- Chọn **Cache Policy** và **Origin Request Policy**

### 4. Edge Locations vs Regional Edge Caches

```
Edge Locations (PoP — Points of Presence):
- 600+ điểm trên toàn cầu (2024)
- Cache L1 — phục vụ trực tiếp người dùng
- Dung lượng nhỏ hơn, refresh nhanh hơn

Regional Edge Caches (REC):
- ~13 điểm, mỗi AWS Region có 1-2
- Cache L2 — trung gian giữa PoP và Origin
- Dung lượng lớn hơn — giảm tải origin đáng kể
```

---

## Use Cases Phổ Biến

### 1. Static Website / S3 + CloudFront

```
S3 (private bucket) ←─── OAC ──── CloudFront ←── Users
```
- Phân phối HTML, CSS, JS, images toàn cầu
- S3 bucket không cần public
- Tích hợp Lambda@Edge để customize response

### 2. Dynamic Web Application

```
Users → CloudFront → ALB → EC2 / ECS / Lambda
                  ↳ Cache static assets tại edge
                  ↳ Pass-through dynamic API calls
```
- Cache static assets (CSS, JS, images)
- Forward API requests đến backend
- WAF protection tại tầng CloudFront

### 3. Video Streaming

```
S3 / MediaPackage → CloudFront → Users (HLS/DASH)
```
- HTTP Live Streaming (HLS) và MPEG-DASH
- Price class tối ưu theo region

### 4. API Acceleration

```
Users → CloudFront → API Gateway / ALB
```
- Edge Caching cho GET responses
- Giảm latency cho global API consumers
- Shield Standard tự động chống DDoS

### 5. Software Distribution

```
S3 → CloudFront → Download clients toàn cầu
```
- File lớn: phân phối nhanh không phụ thuộc server
- Signed URLs/Cookies để kiểm soát truy cập

---

## Lộ Trình Học

### Beginner (Người Mới)

```
1. Hiểu CDN là gì và tại sao cần CloudFront
2. Tạo Distribution đầu tiên với S3 origin
3. Hiểu cache hit/miss và TTL
4. Thực hành bật CloudFront cho static website
```

**Đọc trước:** `1-cloudfront-basics.md`

### Intermediate (Trung Cấp)

```
1. Cấu hình Cache Behaviors theo URL pattern
2. Tạo Cache Policies và Origin Request Policies
3. Bảo vệ S3 bucket với OAC
4. Cấu hình custom domain với ACM SSL
5. Invalidate cache khi deploy
```

**Đọc tiếp:** `2-cache-behaviors.md`, `3-origin-access.md`

### Advanced (Nâng Cao)

```
1. Lambda@Edge cho request/response manipulation
2. CloudFront Functions cho lightweight logic
3. Geo-restriction (Giới Hạn Theo Địa Lý)
4. Tối ưu cache hit ratio
5. Cost optimization với Price Classes
```

**Đọc tiếp:** `4-lambda-edge.md`, `5-performance-cost.md`

---

## Tài Liệu Trong Section Này

| File                        | Nội Dung                                            | Cấp Độ       |
| --------------------------- | --------------------------------------------------- | ------------ |
| `README.md`                 | Tổng quan & kiến trúc CloudFront (file này)         | Beginner     |
| `1-cloudfront-basics.md`    | Distributions, Edge Locations, Origins, cấu hình cơ bản | Beginner  |
| `2-cache-behaviors.md`      | Cache Policies, TTL, Cache Key, Invalidation        | Intermediate |
| `3-origin-access.md`        | OAC, OAI, bảo vệ S3 bucket                         | Intermediate |
| `4-lambda-edge.md`          | Lambda@Edge & CloudFront Functions                  | Advanced     |
| `5-performance-cost.md`     | Price Classes, Compression, Cost Optimization       | Intermediate |

---

## So Sánh CloudFront vs Các Giải Pháp Khác

| Tiêu Chí               | CloudFront       | Global Accelerator | Không Dùng CDN    |
| ---------------------- | ---------------- | ------------------ | ----------------- |
| **Mục đích**           | Cache + Phân phối | Routing + TCP/UDP  | Origin trực tiếp  |
| **Giao thức**          | HTTP/HTTPS       | TCP, UDP, HTTP     | Tùy ứng dụng      |
| **Cache**              | ✅ Có            | ❌ Không           | ❌ Không           |
| **Edge compute**       | ✅ Lambda@Edge   | ❌ Không           | ❌ Không           |
| **Static content**     | ✅ Tuyệt vời     | ⚠️ Không tối ưu    | ❌ Chậm           |
| **UDP / Gaming**       | ❌ Không         | ✅ Hỗ trợ          | ✅ Hỗ trợ         |
| **Chi phí**            | Thấp (cache nhiều) | Cao hơn           | Thấp nhất ban đầu |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

1. **CloudFront hoạt động như thế nào?** — Mô tả luồng request: DNS → Edge → Regional Cache → Origin
2. **Sự khác biệt giữa OAC và OAI?** — OAC là thế hệ mới hơn, hỗ trợ SSE-KMS và POST methods
3. **Làm thế nào để invalidate cache?** — `aws cloudfront create-invalidation` hoặc Console
4. **Cache key là gì?** — Tập hợp headers/cookies/query strings quyết định cache entry
5. **Lambda@Edge khác CloudFront Functions ở điểm gì?** — Runtime, timeout, và nơi thực thi

---

**Tiếp Theo:** [1-cloudfront-basics.md](./1-cloudfront-basics.md) — Cấu hình Distribution và các khái niệm cơ bản
