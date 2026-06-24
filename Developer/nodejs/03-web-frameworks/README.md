# Web Frameworks & REST API — Tổng Quan

> Chủ đề xây dựng HTTP API (Application Programming Interface — Giao Diện Lập Trình Ứng Dụng) với Node.js: so sánh Express, Fastify, NestJS; thiết kế REST API chuẩn; validation (xác thực dữ liệu đầu vào); và API documentation (tài liệu API).

## Mục Lục

1. [Tại Sao Web Framework Quan Trọng](#tại-sao-web-framework-quan-trọng)
2. [So Sánh Express vs Fastify vs NestJS](#so-sánh-express-vs-fastify-vs-nestjs)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Web Framework Quan Trọng

Node.js có module `http` built-in, nhưng production API cần **routing (định tuyến)**, **middleware chain (chuỗi phần mềm trung gian)**, **error handling (xử lý lỗi)**, **validation**, và **security headers**. Framework giải quyết các concern này để bạn tập trung vào business logic.

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **Express.js** | Framework phổ biến nhất, được hỏi nhiều nhất trong phỏng vấn |
| **Middleware** | Hiểu request lifecycle — auth, logging, parsing đều là middleware |
| **REST API Design** | Thiết kế resource-oriented API dễ maintain và scale |
| **Validation** | Ngăn invalid data vào database — defense layer đầu tiên |
| **Error Handling** | Centralized handler — không leak stack trace ra client |
| **API Documentation** | OpenAPI/Swagger — contract giữa backend và frontend |

---

## So Sánh Express vs Fastify vs NestJS

| Tiêu Chí | Express | Fastify | NestJS |
| -------- | ------- | ------- | ------ |
| **Philosophy (Triết Lý)** | Minimal, unopinionated | Performance-first, schema-driven | Opinionated, enterprise architecture |
| **Learning Curve** | Thấp | Trung bình | Cao (cần TypeScript + decorators) |
| **Performance** | ~15k req/s (benchmark) | ~30–40k req/s | ~10–15k req/s (overhead DI) |
| **TypeScript** | Hỗ trợ qua `@types` | First-class | Native, bắt buộc |
| **Validation** | Thư viện ngoài (Zod, Joi) | JSON Schema tích hợp | class-validator + pipes |
| **DI (Dependency Injection — Tiêm Phụ Thuộc)** | Không có sẵn | Không có sẵn | Built-in container |
| **Ecosystem** | Rất lớn | Đang phát triển | Angular-inspired, growing |
| **Phù Hợp** | MVP, startup, legacy | High-throughput API | Enterprise, team lớn |

**Khuyến nghị học:** Bắt đầu với **Express** (nền tảng), sau đó học **REST API design** và **validation** (áp dụng cho mọi framework), cuối cùng **Fastify** hoặc **NestJS** tùy hướng career.

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP REQUEST LIFECYCLE                         │
│                                                                 │
│  Client ──► Load Balancer ──► Node.js Server                    │
│                                    │                            │
│                                    ▼                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Middleware Chain (Express/Fastify)          │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ │   │
│  │  │ Request  │ │  Auth    │ │ Validate │ │  Route    │ │   │
│  │  │ ID/Log   │ │ Middleware│ │ (Zod)   │ │ Handler   │ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └─────┬─────┘ │   │
│  └─────────────────────────────────────────────────┼───────┘   │
│                                                    ▼            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Business Logic / Service Layer              │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Data Access (ORM, Redis, External API)      │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         Error Handler Middleware (4-arg handler)         │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               ▼                                 │
│                         HTTP Response                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 10–12 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-express-basics.md](./1-express-basics.md) | Middleware, routing, request/response | 2 giờ |
| 2 | [2-express-advanced.md](./2-express-advanced.md) | Error handling, validation, file upload | 1.5 giờ |
| 3 | [3-fastify.md](./3-fastify.md) | Schema validation, plugins, performance | 1.5 giờ |
| 4 | [4-nestjs.md](./4-nestjs.md) | Modules, DI, guards, interceptors, pipes | 2 giờ |
| 5 | [5-rest-api-design.md](./5-rest-api-design.md) | Resource naming, HTTP methods, versioning | 1.5 giờ |
| 6 | [6-request-validation.md](./6-request-validation.md) | Zod, Joi, class-validator | 1.5 giờ |
| 7 | [7-api-documentation.md](./7-api-documentation.md) | Swagger/OpenAPI, Scalar, versioning | 1.5 giờ |

**Thứ tự học khuyến nghị:** 1 → 2 → 5 → 6 → 7 → 3 → 4. REST API design và validation áp dụng cho mọi framework; Fastify và NestJS học sau khi nắm Express.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-express-basics.md](./1-express-basics.md) | App setup, middleware chain, Router, req/res objects |
| [2-express-advanced.md](./2-express-advanced.md) | Centralized error handler, multer upload, async wrapper |
| [3-fastify.md](./3-fastify.md) | JSON Schema, plugin encapsulation, hooks lifecycle |
| [4-nestjs.md](./4-nestjs.md) | Module/Controller/Provider, Guards, Interceptors, Pipes |
| [5-rest-api-design.md](./5-rest-api-design.md) | RESTful conventions, status codes, pagination, HATEOAS |
| [6-request-validation.md](./6-request-validation.md) | Zod schema, Joi, class-validator với DTO |
| [7-api-documentation.md](./7-api-documentation.md) | OpenAPI spec, Swagger UI, Scalar, API versioning |

---

## Bài Tập Thực Hành

### Lab 1: CRUD API với Express (2 giờ)

```bash
mkdir express-crud && cd express-crud
npm init -y
npm install express zod
# Tạo REST API cho resource "books": GET, POST, PUT, DELETE
# Middleware: request logging, JSON parser, error handler
```

### Lab 2: Validation Layer (1 giờ)

```javascript
// Thêm Zod validation cho POST /books
// Trả 400 Bad Request với chi tiết lỗi field-level
```

### Lab 3: So Sánh Express vs Fastify (1 giờ)

```bash
# Implement cùng endpoint /api/health trên cả hai
# Benchmark với autocannon: npx autocannon -c 100 -d 10 http://localhost:3000/health
```

### Lab 4: OpenAPI Documentation (1 giờ)

```bash
npm install @scalar/express-api-reference swagger-jsdoc
# Generate OpenAPI spec từ JSDoc comments hoặc Zod schemas
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Middleware trong Express hoạt động thế nào?

**Gợi ý trả lời:** Middleware là function `(req, res, next) => {}` được gọi tuần tự theo thứ tự đăng ký. Mỗi middleware có thể modify `req`/`res`, kết thúc response, hoặc gọi `next()` để chuyển sang middleware tiếp theo. Error-handling middleware có signature 4 tham số `(err, req, res, next)`.

### Câu 2: Express vs Fastify — khi nào chọn gì?

**Gợi ý trả lời:** Express khi cần ecosystem lớn, team quen thuộc, prototype nhanh. Fastify khi cần throughput cao, schema validation built-in, TypeScript-first. Trade-off: Express đơn giản hơn nhưng chậm hơn; Fastify nhanh hơn nhưng learning curve cao hơn một chút.

### Câu 3: REST vs GraphQL — trade-offs?

**Gợi ý trả lời:** REST dùng fixed endpoints, cache-friendly (HTTP caching), đơn giản. GraphQL cho client query chính xác fields cần, tránh over/under-fetching, nhưng phức tạp hơn (N+1, caching khó). REST phù hợp CRUD API; GraphQL phù hợp mobile apps với nhiều view khác nhau.

### Câu 4: Làm sao thiết kế error response chuẩn?

**Gợi ý trả lời:** Dùng consistent format: `{ error: { code, message, details } }`. Map exception sang HTTP status code phù hợp. Không expose stack trace trong production. Dùng custom error class (AppError) với `statusCode` và `isOperational`.

### Câu 5: API versioning strategies?

**Gợi ý trả lời:** URL path (`/v1/users`) — rõ ràng, dễ cache. Header (`Accept: application/vnd.api+json;version=1`) — URL sạch. Query param (`?version=1`) — đơn giản nhưng ít dùng. Khuyến nghị URL versioning cho public API.

---

**Xem tiếp:** [1-express-basics.md](./1-express-basics.md) — bắt đầu với Express.js fundamentals.
