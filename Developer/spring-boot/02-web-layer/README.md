# 02 — Tầng Web — REST API

> Module này bao gồm toàn bộ kiến thức xây dựng tầng Web (Web Layer) trong Spring Boot — từ thiết kế REST Controller,
> xử lý Request/Response, quản lý ngoại lệ, đến tài liệu hóa API với OpenAPI/Swagger.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Xây dựng **REST API** (Giao Diện Lập Trình Ứng Dụng REST) đầy đủ HTTP methods với `@RestController`
- [ ] Thiết kế **DTO** (Data Transfer Object — Đối Tượng Truyền Dữ Liệu) và áp dụng **Bean Validation** (Xác Thực Dữ Liệu)
- [ ] Implement **Global Exception Handling** (Xử Lý Ngoại Lệ Toàn Cục) với `@ControllerAdvice` và chuẩn RFC 7807
- [ ] Phân biệt và sử dụng đúng **Filter** (Bộ Lọc) và **Interceptor** (Bộ Chặn)
- [ ] Tạo tài liệu API tự động với **springdoc-openapi** và **Swagger UI**
- [ ] Áp dụng các best practices (thực tiễn tốt nhất) khi thiết kế REST API

---

## 🗂️ Danh Sách Bài Học

| File | Chủ Đề | Thời Gian | Độ Khó |
|------|--------|-----------|--------|
| [1-rest-controllers.md](1-rest-controllers.md) | @RestController, routing, HTTP methods, response codes | 90 phút | ⭐⭐ |
| [2-request-response.md](2-request-response.md) | DTO, Bean Validation, MapStruct, Jackson serialization | 90 phút | ⭐⭐ |
| [3-exception-handling.md](3-exception-handling.md) | @ControllerAdvice, ProblemDetail (RFC 7807) | 60 phút | ⭐⭐ |
| [4-filters-interceptors.md](4-filters-interceptors.md) | OncePerRequestFilter, HandlerInterceptor, thứ tự thực thi | 60 phút | ⭐⭐ |
| [5-openapi-swagger.md](5-openapi-swagger.md) | springdoc-openapi, Swagger UI, annotation tài liệu | 60 phút | ⭐ |

**Tổng thời gian ước tính: 6–8 giờ**

---

## 🔁 Thứ Tự Học Khuyến Nghị

```
1-rest-controllers.md
      ↓
2-request-response.md      ← Học cùng với controllers
      ↓
3-exception-handling.md    ← Bắt buộc trước khi build API thực
      ↓
4-filters-interceptors.md
      ↓
5-openapi-swagger.md       ← Học sau cùng để tài liệu hóa API đã build
```

---

## 🧠 Kiến Trúc Tầng Web

```
                    HTTP Request (Yêu Cầu HTTP)
                           │
                    ┌──────▼──────┐
                    │   Filter    │  ← Servlet Filter (javax/jakarta)
                    │  (Bộ Lọc)  │     Chạy trước DispatcherServlet
                    └──────┬──────┘
                           │
                    ┌──────▼──────────────────────────────────────┐
                    │           DispatcherServlet                  │
                    │        (Servlet Điều Phối Trung Tâm)         │
                    │                                              │
                    │  ┌────────────┐    ┌────────────────────┐   │
                    │  │Interceptor │    │ HandlerMapping      │   │
                    │  │(Bộ Chặn)   │    │ (Ánh Xạ Handler)   │   │
                    │  └──────┬─────┘    └────────┬───────────┘   │
                    │         │                   │               │
                    │  ┌──────▼───────────────────▼───────────┐   │
                    │  │            @RestController            │   │
                    │  │  @GetMapping / @PostMapping / ...     │   │
                    │  │                                       │   │
                    │  │  ┌──────────┐   ┌────────────────┐   │   │
                    │  │  │  @Valid  │   │  @RequestBody  │   │   │
                    │  │  │  Bean    │   │  @PathVariable │   │   │
                    │  │  │Validation│   │  @RequestParam │   │   │
                    │  │  └──────────┘   └────────────────┘   │   │
                    │  └───────────────────────────────────────┘   │
                    │                                              │
                    │  ┌───────────────────────────────────────┐   │
                    │  │      @ControllerAdvice                │   │
                    │  │   (Xử Lý Ngoại Lệ Toàn Cục)          │   │
                    │  └───────────────────────────────────────┘   │
                    └──────────────────────────────────────────────┘
                           │
                    HTTP Response (Phản Hồi HTTP)
```

---

## 🔑 Khái Niệm Cốt Lõi

### REST — Representational State Transfer (Chuyển Đổi Trạng Thái Đại Diện)

REST là kiến trúc thiết kế API dựa trên giao thức HTTP, với các nguyên tắc:

| Nguyên Tắc | Ý Nghĩa |
|------------|---------|
| **Stateless** (Không Trạng Thái) | Mỗi request chứa đủ thông tin, server không lưu session |
| **Resource-based** (Dựa Trên Tài Nguyên) | URL đại diện cho tài nguyên (`/users`, `/orders`) |
| **Uniform Interface** (Giao Diện Thống Nhất) | HTTP methods có ý nghĩa chuẩn (GET, POST, PUT, DELETE) |
| **Client-Server** | Tách biệt client và server |
| **Cacheable** (Có Thể Cache) | Response có thể được cache |

### HTTP Methods & Ý Nghĩa

| Method | Ý Nghĩa | Idempotent (Bất Biến) | Safe (An Toàn) |
|--------|---------|-----------------------|----------------|
| `GET` | Đọc tài nguyên | ✅ | ✅ |
| `POST` | Tạo tài nguyên mới | ❌ | ❌ |
| `PUT` | Thay thế toàn bộ tài nguyên | ✅ | ❌ |
| `PATCH` | Cập nhật một phần tài nguyên | ✅ | ❌ |
| `DELETE` | Xóa tài nguyên | ✅ | ❌ |

> **Idempotent** (Bất Biến): Gọi nhiều lần cho kết quả giống lần đầu.  
> **Safe** (An Toàn): Không thay đổi trạng thái server.

### HTTP Status Codes (Mã Trạng Thái HTTP) Phổ Biến

| Nhóm | Mã | Ý Nghĩa |
|------|----|---------|
| 2xx — Thành Công | 200 OK | Request thành công |
| | 201 Created | Tạo tài nguyên thành công |
| | 204 No Content | Thành công, không có body |
| 3xx — Chuyển Hướng | 301 Moved Permanently | Đã chuyển URL vĩnh viễn |
| 4xx — Lỗi Client | 400 Bad Request | Request không hợp lệ |
| | 401 Unauthorized | Chưa xác thực |
| | 403 Forbidden | Không có quyền |
| | 404 Not Found | Không tìm thấy tài nguyên |
| | 409 Conflict | Xung đột (ví dụ: email đã tồn tại) |
| | 422 Unprocessable Entity | Dữ liệu không hợp lệ về mặt nghiệp vụ |
| 5xx — Lỗi Server | 500 Internal Server Error | Lỗi server không mong muốn |
| | 503 Service Unavailable | Server tạm thời không khả dụng |

---

## 📐 Cấu Trúc Request-Response Chuẩn

### Request Body (Thân Request) — DTO vào

```json
POST /api/v1/users
Content-Type: application/json

{
  "email": "user@example.com",
  "fullName": "Nguyễn Văn A",
  "password": "SecurePass123!"
}
```

### Response Body (Thân Response) — DTO ra (thành công)

```json
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "uuid-123",
  "email": "user@example.com",
  "fullName": "Nguyễn Văn A",
  "createdAt": "2026-06-02T10:00:00Z"
}
```

### Response Body — Lỗi (theo RFC 7807 ProblemDetail)

```json
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Email không hợp lệ",
  "instance": "/api/v1/users",
  "errors": [
    { "field": "email", "message": "phải là địa chỉ email hợp lệ" }
  ]
}
```

---

## ✅ Checklist Trước Khi Sang Module Tiếp Theo

- [ ] Xây dựng được CRUD REST API hoàn chỉnh
- [ ] DTO validation với `@Valid` và custom constraints
- [ ] Global exception handler trả về response nhất quán
- [ ] Hiểu khi nào dùng Filter vs Interceptor
- [ ] Tạo được Swagger UI chạy được từ code
- [ ] Trả lời được: ProblemDetail RFC 7807 là gì và tại sao dùng?

---

## 🔗 Liên Kết Tiếp Theo

Sau khi hoàn thành module này:

- → [03-data-access/](../03-data-access/) — Kết nối Controller với Database qua JPA
- → [04-security/](../04-security/) — Bảo vệ REST API với Spring Security + JWT

---

**Cập Nhật Lần Cuối:** 2026-06-02
