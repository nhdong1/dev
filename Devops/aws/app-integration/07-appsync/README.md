# AWS AppSync — Managed GraphQL API (API GraphQL Được Quản Lý)

> **AppSync** là dịch vụ GraphQL được quản lý hoàn toàn của AWS, cho phép xây dựng API linh hoạt, real-time (thời gian thực) và offline-capable (hỗ trợ ngoại tuyến) mà không cần tự quản lý hạ tầng.

---

## 📚 Mục Lục Module

| File | Nội Dung | Độ Khó |
|---|---|---|
| **README.md** (file này) | Tổng quan AppSync, kiến trúc, khi nào dùng | ⭐⭐ |
| [1-schema-resolvers.md](./1-schema-resolvers.md) | Schema, Resolver, Data Sources | ⭐⭐ |
| [2-real-time-subscriptions.md](./2-real-time-subscriptions.md) | Real-time Subscription qua WebSocket | ⭐⭐⭐ |
| [3-caching-strategy.md](./3-caching-strategy.md) | Caching Strategy — Chiến Lược Bộ Nhớ Đệm | ⭐⭐⭐ |

---

## 🎯 Tại Sao Cần Học AWS AppSync?

### Trong Phỏng Vấn

AppSync thường được hỏi trong các cuộc phỏng vấn backend/fullstack khi ứng viên làm việc với:

- **Mobile / Frontend** cần dữ liệu linh hoạt (tránh over-fetching / under-fetching)
- **Real-time features** như chat, notification, live dashboard
- **Offline-first applications** (ứng dụng ưu tiên ngoại tuyến)
- **Multi-data-source aggregation** (tổng hợp nhiều nguồn dữ liệu)

### Trong Thực Tế

AppSync rút ngắn đáng kể thời gian phát triển bằng cách tự động xử lý:
- WebSocket management (quản lý kết nối WebSocket)
- Subscription fan-out (phân phối đăng ký đến nhiều client)
- Data source integration (tích hợp nguồn dữ liệu)
- Auth & authorization (xác thực và phân quyền)

---

## 🏗️ Kiến Trúc Tổng Quan AppSync

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
│  Web App   Mobile App   IoT Device   Third-party Client             │
└────────┬───────────┬─────────────────┬──────────────────────────────┘
         │           │                 │
         │  HTTPS/WSS (WebSocket Secure)
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS APPSYNC                                     │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                  GraphQL Schema (Lược Đồ)                   │    │
│  │  type Query  { ... }                                        │    │
│  │  type Mutation { ... }                                      │    │
│  │  type Subscription { ... }                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                    Resolver (Bộ Giải Quyết)                         │
│           ┌──────────────────┼──────────────────┐                   │
│           ▼                  ▼                  ▼                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  VTL Template│  │  JS Resolver │  │ Direct Lambda│              │
│  │  (Velocity)  │  │  (JavaScript)│  │  Resolver    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     Auth Layer                               │   │
│  │  Cognito | API Key | IAM | OIDC | Lambda Authorizer          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              Server-Side Caching (Bộ Nhớ Đệm)               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              DATA SOURCES (Nguồn Dữ Liệu)
     ┌─────────────────────────┼─────────────────────────┐
     ▼                         ▼                         ▼
┌─────────┐            ┌──────────────┐          ┌──────────────┐
│DynamoDB │            │  AWS Lambda  │          │  Aurora/RDS  │
│         │            │              │          │  (via RDS    │
│         │            │              │          │  Data API)   │
└─────────┘            └──────────────┘          └──────────────┘
     ▼                         ▼                         ▼
┌─────────┐            ┌──────────────┐          ┌──────────────┐
│OpenSearch│           │  HTTP APIs   │          │  EventBridge │
│(Search) │            │  (Third-party│          │              │
└─────────┘            │   REST APIs) │          └──────────────┘
                       └──────────────┘
```

---

## 📖 GraphQL Cơ Bản — Nền Tảng Cần Biết

### Ba Loại Operation (Thao Tác) GraphQL

```graphql
# 1. Query — Đọc dữ liệu (Read — tương đương GET trong REST)
query GetUser($id: ID!) {
  getUser(id: $id) {
    id
    name
    email
    orders {
      id
      total
    }
  }
}

# 2. Mutation — Ghi dữ liệu (Write — tương đương POST/PUT/DELETE trong REST)
mutation CreateOrder($input: CreateOrderInput!) {
  createOrder(input: $input) {
    id
    status
    total
    createdAt
  }
}

# 3. Subscription — Lắng nghe sự kiện real-time qua WebSocket
subscription OnOrderStatusChanged($orderId: ID!) {
  onOrderStatusChanged(orderId: $orderId) {
    orderId
    newStatus
    updatedAt
  }
}
```

### So Sánh GraphQL vs REST

| Tiêu Chí | GraphQL | REST |
|---|---|---|
| **Cấu trúc dữ liệu** | Client quyết định cấu trúc nhận | Server quyết định cấu trúc trả về |
| **Over-fetching** (Lấy dư dữ liệu) | Không xảy ra | Thường gặp |
| **Under-fetching** (Lấy thiếu dữ liệu) | Không xảy ra | Dẫn đến N+1 requests |
| **Versioning** (Quản lý phiên bản) | Thêm field không breaking | Cần versioning URL |
| **Real-time** | Built-in Subscription | Cần SSE hoặc WebSocket riêng |
| **Introspection** (Tự mô tả) | Schema tự mô tả | Cần tài liệu riêng (Swagger) |
| **Caching phía client** | Phức tạp hơn | Đơn giản (HTTP cache) |
| **Learning curve** (Độ khó học) | Cao hơn | Thấp hơn |

---

## 🔌 Data Sources (Nguồn Dữ Liệu) Được Hỗ Trợ

| Data Source | Mô Tả | Use Case Phổ Biến |
|---|---|---|
| **Amazon DynamoDB** | NoSQL managed database | CRUD operations, user data |
| **AWS Lambda** | Custom business logic | Complex queries, transformations |
| **Amazon Aurora (RDS)** | Relational DB qua RDS Data API | SQL queries, relational data |
| **Amazon OpenSearch** | Full-text search engine | Search functionality |
| **HTTP Endpoint** | Bất kỳ REST API nào | Third-party integrations |
| **AWS EventBridge** | Event bus | Publish events từ mutations |
| **None** | Loại đặc biệt — dùng với Local Resolver | Mock data, pass-through |

---

## 🔐 Authentication (Xác Thực) & Authorization (Phân Quyền)

AppSync hỗ trợ **5 cơ chế xác thực**:

### 1. API Key
```
- Đơn giản nhất, không cần login
- Thích hợp: Public APIs, development/testing
- Hết hạn: tối đa 365 ngày
- Không nên dùng cho dữ liệu nhạy cảm
```

### 2. Amazon Cognito User Pools (Nhóm Người Dùng Cognito)
```
- Xác thực qua JWT (JSON Web Token) từ Cognito
- Thích hợp: Ứng dụng có user accounts
- Hỗ trợ: @auth directive trong schema
- Phổ biến nhất cho consumer apps
```

### 3. AWS IAM (Identity and Access Management)
```
- Dùng AWS Signature Version 4
- Thích hợp: Service-to-service (dịch vụ gọi dịch vụ)
- Phù hợp cho Lambda, EC2, ECS gọi AppSync
```

### 4. OpenID Connect — OIDC
```
- Tích hợp với third-party identity providers (nhà cung cấp danh tính)
- Ví dụ: Auth0, Okta, Google, Facebook
```

### 5. Lambda Authorizer (Bộ Ủy Quyền Lambda)
```
- Custom authorization logic hoàn toàn
- Lambda nhận request, trả về authorization decision
- Thích hợp: Logic phân quyền phức tạp, đặc thù
```

### Multiple Authentication Modes (Đa Chế Độ Xác Thực)

AppSync cho phép cấu hình **primary + additional auth modes**:

```graphql
type Post @aws_cognito_user_pools @aws_api_key {
  id: ID!
  title: String!
  content: String!
  # Trường này CHỈ dành cho Cognito users (authenticated)
  secretField: String @aws_cognito_user_pools
}
```

---

## ⚡ Khi Nào Dùng AppSync?

### ✅ Nên Dùng AppSync Khi

```
1. Mobile / Web app cần dữ liệu linh hoạt theo màn hình
   → Tránh over-fetching và under-fetching

2. Real-time features cần thiết yếu (chat, notification, live data)
   → Built-in WebSocket Subscriptions

3. Aggregating (tổng hợp) nhiều data sources trong một API
   → Resolver có thể kết hợp DynamoDB + Lambda + REST API

4. Offline-first mobile apps với Amplify DataStore
   → Tự động sync khi có kết nối trở lại

5. Rapid prototyping (prototype nhanh) với DynamoDB
   → AppSync có thể auto-generate resolvers từ schema
```

### ❌ Không Nên Dùng AppSync Khi

```
1. API đơn giản với cấu trúc cố định
   → REST API Gateway + Lambda đơn giản hơn

2. Yêu cầu file upload lớn
   → S3 Presigned URL phù hợp hơn

3. Streaming data liên tục (như log analytics)
   → Kinesis phù hợp hơn

4. Team không quen GraphQL
   → Learning curve cao, cần đầu tư đào tạo

5. Chỉ cần CRUD đơn giản với một bảng DynamoDB
   → API Gateway + DynamoDB trực tiếp đơn giản hơn
```

### So Sánh AppSync vs API Gateway

| Tiêu Chí | AWS AppSync | API Gateway + Lambda |
|---|---|---|
| **Protocol** | GraphQL over HTTP/WebSocket | REST/HTTP/WebSocket |
| **Real-time** | Built-in Subscriptions | Cần tự xây dựng |
| **Flexible queries** (Truy vấn linh hoạt) | Mạnh — client quyết định | Yếu — endpoint cố định |
| **Schema validation** (Kiểm tra lược đồ) | Tự động | Cần tự implement |
| **Caching** | Server-side caching tích hợp | ElastiCache riêng |
| **Complexity** (Độ phức tạp) | GraphQL learning curve | Thân quen với REST |
| **Pricing** (Giá) | Theo request + connection minutes | Theo request + Lambda |
| **Direct DB access** (Truy cập DB trực tiếp) | DynamoDB không cần Lambda | Cần Lambda |

---

## 💰 Pricing (Định Giá) Tóm Tắt

AppSync tính phí theo 3 chiều:

```
1. Query & Mutation Requests (Yêu Cầu Truy Vấn & Biến Đổi)
   → $4.00 / triệu requests (sau 250,000 miễn phí/tháng)

2. Subscription Connection Minutes (Phút Kết Nối Đăng Ký)
   → $0.08 / triệu connection minutes

3. Subscription Messages (Tin Nhắn Đăng Ký)
   → $2.00 / triệu messages delivered

4. Caching (Bộ Nhớ Đệm) — tùy instance size
   → Tính theo giờ, tương tự ElastiCache
```

**Miễn phí hàng tháng (Free Tier):**
- 250,000 query/mutation requests
- 250,000 real-time updates (subscription messages)
- 600,000 connection minutes

---

## 🔑 Các Khái Niệm Quan Trọng Cần Nhớ

| Khái Niệm | Định Nghĩa Ngắn |
|---|---|
| **Schema** (Lược Đồ) | Định nghĩa cấu trúc API — types, queries, mutations, subscriptions |
| **Resolver** (Bộ Giải Quyết) | Logic kết nối field với data source |
| **Data Source** (Nguồn Dữ Liệu) | DynamoDB, Lambda, RDS, OpenSearch, HTTP |
| **Pipeline Resolver** (Bộ Giải Quyết Đường Ống) | Chuỗi các function chạy tuần tự |
| **Subscription** (Đăng Ký) | Real-time listener qua WebSocket |
| **Directive** (Chỉ Thị) | Annotation trong schema như `@aws_auth`, `@deprecated` |
| **VTL** (Velocity Template Language) | Ngôn ngữ template dùng trong resolver |
| **JS Resolver** (Bộ Giải Quyết JavaScript) | Runtime mới — JavaScript thay thế VTL |

---

## 📚 Tài Liệu Liên Quan Trong Module Này

- **[1-schema-resolvers.md](./1-schema-resolvers.md)** — Đi sâu vào Schema định nghĩa, VTL và JS Resolver, Pipeline Resolver
- **[2-real-time-subscriptions.md](./2-real-time-subscriptions.md)** — WebSocket connection lifecycle, subscription filtering, fan-out
- **[3-caching-strategy.md](./3-caching-strategy.md)** — Server-side caching, TTL, cache invalidation

---

## 📎 Checklist Nhanh Trước Phỏng Vấn

- [ ] Giải thích GraphQL Query, Mutation, Subscription bằng ví dụ thực tế
- [ ] Phân biệt AppSync vs API Gateway — khi nào dùng cái nào
- [ ] Biết các data sources AppSync hỗ trợ (DynamoDB, Lambda, RDS...)
- [ ] Hiểu 5 auth modes của AppSync
- [ ] Giải thích cơ chế real-time subscription hoạt động như thế nào
- [ ] Biết caching hoạt động ở cấp độ nào trong AppSync
- [ ] Nắm pipeline resolver là gì và khi nào cần dùng

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Module:** 07-appsync | App Integration Series
