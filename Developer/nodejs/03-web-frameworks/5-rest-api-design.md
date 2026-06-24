# REST API Design — Resource Naming, HTTP Methods và Versioning

> Thiết kế RESTful API (Representational State Transfer — Chuyển Trạng Thái Biểu Diễn) chuẩn giúp API dễ hiểu, maintain, và scale. Đây là kỹ năng được đánh giá cao trong phỏng vấn Backend.

## Mục Lục

1. [REST Principles](#rest-principles)
2. [Resource Naming Conventions](#resource-naming-conventions)
3. [HTTP Methods và Idempotency](#http-methods-và-idempotency)
4. [HTTP Status Codes](#http-status-codes)
5. [Request và Response Format](#request-và-response-format)
6. [Pagination, Filtering, Sorting](#pagination-filtering-sorting)
7. [API Versioning](#api-versioning)
8. [HATEOAS và Hypermedia](#hateoas-và-hypermedia)
9. [Anti-Patterns](#anti-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## REST Principles

REST là architectural style (phong cách kiến trúc), không phải protocol:

| Nguyên Tắc | Mô Tả |
| ---------- | ----- |
| **Client-Server** | Tách biệt concerns — UI và data storage độc lập |
| **Stateless (Phi Trạng Thái)** | Mỗi request chứa đủ thông tin, server không lưu client state |
| **Cacheable (Có Thể Cache)** | Response phải đánh dấu có thể cache hay không |
| **Uniform Interface** | Resource identification (URI), manipulation qua representations |
| **Layered System** | Client không biết kết nối trực tiếp hay qua intermediary |
| **Code on Demand** (optional) | Server gửi executable code (hiếm dùng) |

---

## Resource Naming Conventions

### Dùng Nouns (Danh Từ), Không Dùng Verbs

```
✅ GET    /users
✅ GET    /users/123
✅ POST   /users
✅ PUT    /users/123
✅ DELETE /users/123

❌ GET    /getUsers
❌ POST   /createUser
❌ DELETE /deleteUser/123
```

### Plural Nouns (Số Nhiều)

```
✅ /users, /orders, /products
❌ /user, /order (trừ khi team convention khác — nhất quán là quan trọng)
```

### Nested Resources (Tài Nguyên Lồng Nhau)

```
✅ GET /users/123/orders          — orders của user 123
✅ GET /users/123/orders/456      — order cụ thể
✅ POST /users/123/orders         — tạo order cho user 123

⚠️ Tối đa 2–3 levels nesting — tránh URL quá dài
✅ GET /orders/456/items          — thay vì /users/123/orders/456/items
```

### Actions Không Phải CRUD

Dùng sub-resource hoặc POST với action name:

```
POST /orders/123/cancel          — cancel order
POST /users/123/activate         — activate user
POST /auth/login                 — authentication (exception cho verbs)
POST /auth/refresh               — refresh token
```

---

## HTTP Methods và Idempotency

| Method | Mục Đích | Idempotent | Safe (An Toàn) | Request Body |
| ------ | -------- | ---------- | -------------- | ------------ |
| **GET** | Đọc | Có | Có | Không |
| **POST** | Tạo mới | Không | Không | Có |
| **PUT** | Thay thế toàn bộ | Có | Không | Có |
| **PATCH** | Cập nhật một phần | Không* | Không | Có |
| **DELETE** | Xóa | Có | Không | Hiếm |
| **HEAD** | Metadata only | Có | Có | Không |
| **OPTIONS** | Supported methods | Có | Có | Không |

*PATCH idempotent khi implementation đúng (cùng patch → cùng kết quả)

### PUT vs PATCH

```javascript
// PUT — thay thế toàn bộ resource
PUT /users/123
{ "name": "John", "email": "john@example.com", "age": 30 }
// Phải gửi đầy đủ fields

// PATCH — cập nhật một phần (JSON Merge Patch hoặc JSON Patch)
PATCH /users/123
{ "name": "John Updated" }
// Chỉ gửi fields cần thay đổi
```

---

## HTTP Status Codes

### 2xx Success

| Code | Tên | Khi Dùng |
| ---- | --- | -------- |
| **200** | OK | GET, PUT, PATCH thành công |
| **201** | Created | POST tạo resource mới |
| **204** | No Content | DELETE thành công, không body |

### 4xx Client Error

| Code | Tên | Khi Dùng |
| ---- | --- | -------- |
| **400** | Bad Request | Validation fail, malformed request |
| **401** | Unauthorized | Chưa authenticate |
| **403** | Forbidden | Đã auth nhưng không có quyền |
| **404** | Not Found | Resource không tồn tại |
| **409** | Conflict | Duplicate, version conflict |
| **422** | Unprocessable Entity | Semantic validation fail |
| **429** | Too Many Requests | Rate limit exceeded |

### 5xx Server Error

| Code | Tên | Khi Dùng |
| ---- | --- | -------- |
| **500** | Internal Server Error | Bug, unexpected error |
| **502** | Bad Gateway | Upstream service fail |
| **503** | Service Unavailable | Maintenance, overload |
| **504** | Gateway Timeout | Upstream timeout |

**Quy tắc:** Dùng status code chính xác — không trả 200 với `{ error: true }` trong body.

---

## Request và Response Format

### Consistent Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "field": "email", "message": "Invalid email format" }
    ],
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Success Response Patterns

```json
// Single resource
{
  "data": {
    "id": "123",
    "name": "John",
    "email": "john@example.com"
  }
}

// Collection với pagination
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  },
  "links": {
    "self": "/users?page=1&limit=20",
    "next": "/users?page=2&limit=20",
    "prev": null
  }
}
```

### Content Negotiation

```
Accept: application/json
Accept: application/xml          — ít dùng
Content-Type: application/json
```

---

## Pagination, Filtering, Sorting

### Offset Pagination (Phân Trang Theo Vị Trí)

```
GET /users?page=2&limit=20
GET /users?offset=20&limit=20
```

**Nhược điểm:** Chậm với offset lớn (`OFFSET 100000`), inconsistent khi data thay đổi giữa các page.

### Cursor Pagination (Phân Trang Theo Con Trỏ)

```
GET /users?cursor=eyJpZCI6MTIzfQ&limit=20

Response:
{
  "data": [...],
  "meta": { "nextCursor": "eyJpZCI6MTQzfQ", "hasMore": true }
}
```

**Ưu điểm:** Stable với data thay đổi, hiệu năng tốt với index.

### Filtering và Sorting

```
GET /users?status=active&role=admin
GET /users?created_after=2024-01-01
GET /users?sort=-created_at,name     — descending created_at, ascending name
GET /users?fields=id,name,email      — sparse fieldsets (chỉ trả fields cần)
```

---

## API Versioning

| Strategy | Ví Dụ | Ưu/Nhược |
| -------- | ----- | -------- |
| **URL Path** | `/api/v1/users` | Rõ ràng, dễ test, khuyến nghị cho public API |
| **Header** | `Accept: application/vnd.myapi.v1+json` | URL sạch, khó discover |
| **Query Param** | `/users?version=1` | Đơn giản, ít professional |

```javascript
// Express versioning
const v1Router = require('./routes/v1');
const v2Router = require('./routes/v2');

app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);
```

**Deprecation strategy:**

```
Response headers:
Deprecation: true
Sunset: Sat, 01 Jan 2027 00:00:00 GMT
Link: </api/v2/users>; rel="successor-version"
```

---

## HATEOAS và Hypermedia

HATEOAS (Hypermedia As The Engine Of Application State) — response chứa links đến related actions:

```json
{
  "data": {
    "id": "123",
    "status": "pending",
    "amount": 99.99
  },
  "links": {
    "self": { "href": "/orders/123" },
    "cancel": { "href": "/orders/123/cancel", "method": "POST" },
    "pay": { "href": "/orders/123/pay", "method": "POST" }
  }
}
```

Thực tế: HATEOAS đầy đủ ít dùng; `links` cho pagination phổ biến hơn.

---

## Anti-Patterns

| Anti-Pattern | Vấn Đề | Giải Pháp |
| ------------ | ------ | --------- |
| Verbs trong URL | Không RESTful | Dùng HTTP methods |
| 200 với error body | Client khó handle | Dùng 4xx/5xx status |
| Trả quá nhiều data | Over-fetching | Pagination, field filtering |
| Không versioning | Breaking changes gây pain | Version từ đầu |
| Inconsistent naming | Confusion | Document convention |
| Session state trên server | Không stateless | JWT hoặc stateless tokens |
| `GET` với side effects | Vi phạm HTTP semantics | Dùng POST cho actions |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Thiết kế REST API cho e-commerce?

**Gợi ý:** Resources: `/products`, `/categories`, `/cart`, `/orders`, `/users`. Cart là resource riêng hoặc sub-resource của user. Checkout: `POST /orders` từ cart. Payment: `POST /orders/:id/pay`. Status transitions qua PATCH hoặc action endpoints.

### Câu 2: REST vs RPC-style API?

**Trả lời:** REST: resources + HTTP verbs, cacheable, stateless. RPC (Remote Procedure Call): actions/endpoints như `/calculateShipping`, `/sendEmail` — flexible nhưng ít chuẩn hóa. gRPC là RPC framework phổ biến.

### Câu 3: Khi nào dùng 401 vs 403?

**Trả lời:** 401 Unauthorized: client chưa authenticate hoặc credentials invalid — cần login. 403 Forbidden: đã authenticate nhưng không có permission — không retry với cùng credentials.

### Câu 4: Offset vs cursor pagination?

**Trả lời:** Offset đơn giản, phù hợp admin UI với page numbers. Cursor phù hợp infinite scroll, real-time feeds, large datasets — O(1) lookup vs O(n) skip với offset lớn.

---

**Xem tiếp:** [6-request-validation.md](./6-request-validation.md) — Zod, Joi, class-validator.
