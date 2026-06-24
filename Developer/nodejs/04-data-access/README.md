# Truy Cập Dữ Liệu — Tổng Quan

> Chủ đề tích hợp database (cơ sở dữ liệu) với Node.js: PostgreSQL driver, ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng–Quan Hệ), ODM (Object-Document Mapper — Ánh Xạ Đối Tượng–Tài Liệu), Redis caching, transactions (giao dịch), và tối ưu query.

## Mục Lục

1. [Tại Sao Data Layer Quan Trọng](#tại-sao-data-layer-quan-trọng)
2. [SQL vs NoSQL — Khi Nào Chọn Gì](#sql-vs-nosql--khi-nào-chọn-gì)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [So Sánh ORM/ODM Phổ Biến](#so-sánh-ormodm-phổ-biến)
5. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
6. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
7. [Bài Tập Thực Hành](#bài-tập-thực-hành)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Data Layer Quan Trọng

Hầu hết Backend API cần **persistent storage (lưu trữ bền vững)**. Data layer quyết định:

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **Connection Pool (Bể Kết Nối)** | Mỗi DB connection tốn tài nguyên — pool tái sử dụng connection hiệu quả |
| **Parameterized Queries (Truy Vấn Tham Số Hóa)** | Ngăn SQL Injection — bắt buộc trong production |
| **ORM/ODM** | Type-safe queries, migrations, relations — tăng productivity |
| **Transactions (ACID)** | Đảm bảo data consistency khi nhiều thao tác liên quan |
| **Caching (Redis)** | Giảm load DB, cải thiện latency cho read-heavy apps |
| **N+1 Problem** | Anti-pattern phổ biến gây API chậm — hay được hỏi phỏng vấn |

---

## SQL vs NoSQL — Khi Nào Chọn Gì

| Tiêu Chí | PostgreSQL (SQL) | MongoDB (NoSQL) |
| -------- | ---------------- | --------------- |
| **Data Model** | Relational, schema cố định | Document-based, schema linh hoạt |
| **ACID Transactions** | Native, mạnh | Hỗ trợ từ v4.0+ (multi-document) |
| **Joins (Kết Nối Bảng)** | Native SQL JOIN | `$lookup` aggregation hoặc embed |
| **Scaling** | Vertical + read replicas | Horizontal sharding dễ hơn |
| **Use Case** | E-commerce, finance, reporting | CMS, IoT logs, rapid prototyping |
| **Node.js Ecosystem** | Prisma, TypeORM, Sequelize, `pg` | Mongoose, native driver |

**Khuyến nghị:** Bắt đầu với **PostgreSQL + Prisma** (phổ biến nhất trong phỏng vấn Node.js), sau đó học **MongoDB + Mongoose** và **Redis** cho caching.

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA ACCESS LAYER                              │
│                                                                 │
│  HTTP Request ──► Controller ──► Service Layer                  │
│                                      │                          │
│                    ┌─────────────────┼─────────────────┐        │
│                    ▼                 ▼                 ▼        │
│              ┌──────────┐      ┌──────────┐      ┌──────────┐  │
│              │  ORM/ODM │      │  Redis   │      │ Raw SQL  │  │
│              │ (Prisma, │      │  Cache   │      │  (pg)    │  │
│              │ TypeORM) │      │          │      │          │  │
│              └────┬─────┘      └────┬─────┘      └────┬─────┘  │
│                   │                 │                 │         │
│                   ▼                 ▼                 ▼         │
│              ┌──────────┐      ┌──────────┐      ┌──────────┐  │
│              │PostgreSQL│      │  Redis   │      │PostgreSQL│  │
│              │ / MongoDB│      │  Server  │      │          │  │
│              └──────────┘      └──────────┘      └──────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Repository Pattern (Mẫu Kho Lưu Trữ):** Tách data access khỏi business logic — service gọi repository, repository gọi ORM/driver.

---

## So Sánh ORM/ODM Phổ Biến

| Tiêu Chí | Prisma | TypeORM | Sequelize | Mongoose |
| -------- | ------ | ------- | --------- | -------- |
| **Database** | SQL (PG, MySQL, SQLite) | SQL | SQL | MongoDB |
| **TypeScript** | First-class, generated types | Native decorators | Hỗ trợ qua types | Hỗ trợ qua types |
| **Schema** | `schema.prisma` DSL | Entity classes | Model definitions | Schema definitions |
| **Migrations** | `prisma migrate` | CLI migrations | `sequelize-cli` | Manual hoặc plugins |
| **Query Style** | Fluent API + raw SQL | Query Builder + Repository | Sequelize API | Mongoose API |
| **Learning Curve** | Thấp | Trung bình | Trung bình | Thấp |
| **NestJS Integration** | `@prisma/client` module | `@nestjs/typeorm` | `@nestjs/sequelize` | `@nestjs/mongoose` |
| **Phù Hợp** | Greenfield TS projects | Enterprise, NestJS | Legacy Node.js | MongoDB projects |

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 10–12 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-postgresql-pg.md](./1-postgresql-pg.md) | `pg` driver, connection pool, parameterized queries | 1.5 giờ |
| 2 | [2-prisma-orm.md](./2-prisma-orm.md) | Schema, migrations, Prisma Client, relations | 2 giờ |
| 3 | [3-typeorm.md](./3-typeorm.md) | Entity, repository, query builder, relations | 1.5 giờ |
| 4 | [4-sequelize.md](./4-sequelize.md) | Models, associations, hooks, migrations | 1 giờ |
| 5 | [5-mongodb-mongoose.md](./5-mongodb-mongoose.md) | Document schema, aggregation, indexing | 1.5 giờ |
| 6 | [6-redis-caching.md](./6-redis-caching.md) | ioredis, cache patterns, pub/sub, sessions | 1.5 giờ |
| 7 | [7-transactions.md](./7-transactions.md) | ACID transactions, optimistic locking | 1 giờ |
| 8 | [8-n-plus-one.md](./8-n-plus-one.md) | Eager loading, DataLoader, query optimization | 1.5 giờ |

**Thứ tự học khuyến nghị:** 1 → 2 → 7 → 8 → 6 → 5 → 3 → 4. Nắm `pg` và Prisma trước; TypeORM/Sequelize tham khảo khi làm việc với codebase legacy hoặc NestJS.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-postgresql-pg.md](./1-postgresql-pg.md) | `Pool`, `Client`, parameterized queries, connection lifecycle |
| [2-prisma-orm.md](./2-prisma-orm.md) | `schema.prisma`, `prisma migrate`, relations, `$transaction` |
| [3-typeorm.md](./3-typeorm.md) | `@Entity`, Repository, QueryBuilder, eager/lazy loading |
| [4-sequelize.md](./4-sequelize.md) | `define()`, associations, hooks, `sequelize.sync()` |
| [5-mongodb-mongoose.md](./5-mongodb-mongoose.md) | Schema, Model, aggregation pipeline, compound indexes |
| [6-redis-caching.md](./6-redis-caching.md) | Cache-aside, TTL, pub/sub, session store với `connect-redis` |
| [7-transactions.md](./7-transactions.md) | ACID, isolation levels, optimistic vs pessimistic locking |
| [8-n-plus-one.md](./8-n-plus-one.md) | N+1 detection, `include`/`join`, DataLoader batching |

---

## Bài Tập Thực Hành

### Lab 1: PostgreSQL với `pg` (1.5 giờ)

```bash
docker run -d --name pg-lab -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:16
mkdir pg-lab && cd pg-lab && npm init -y && npm install pg
# Tạo bảng users, posts; CRUD với parameterized queries và Pool
```

### Lab 2: Prisma Full Stack (2 giờ)

```bash
npx prisma init
# Định nghĩa User + Post với relation one-to-many
npx prisma migrate dev --name init
# CRUD API Express với Prisma Client
```

### Lab 3: Redis Cache Layer (1 giờ)

```bash
docker run -d --name redis-lab -p 6379:6379 redis:7
npm install ioredis
# Implement cache-aside cho GET /users/:id
```

### Lab 4: Fix N+1 Problem (1 giờ)

```javascript
// Bắt đầu với code gây N+1, đo số queries
// Refactor với Prisma include hoặc DataLoader
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Connection pool là gì? Size bao nhiêu là đủ?

**Gợi ý trả lời:** Pool duy trì tập connection sẵn sàng, tránh overhead tạo connection mới mỗi request. Size phụ thuộc DB `max_connections`, số Node.js instances, và workload. Rule of thumb: `(core_count * 2) + effective_spindle_count` cho PG; với nhiều app instances, chia `max_connections` cho số instances.

### Câu 2: SQL Injection — prevent thế nào trong Node.js?

**Gợi ý trả lời:** Luôn dùng parameterized queries (`$1`, `$2` với `pg`) hoặc ORM prepared statements. Không concatenate user input vào SQL string. Validate input ở application layer (Zod/Joi) trước khi query.

### Câu 3: Prisma vs TypeORM — trade-offs?

**Gợi ý trả lời:** Prisma: type-safe generated client, migrations tốt, DX cao, ít magic. TypeORM: decorators quen thuộc với Java/Spring devs, active record + data mapper, tích hợp NestJS sâu. Prisma phù hợp greenfield; TypeORM phù hợp enterprise NestJS.

### Câu 4: N+1 problem là gì? Cách fix?

**Gợi ý trả lời:** Load N parent records, rồi loop query children → 1 + N queries. Fix: eager loading (`include`/`join`), batch query với `IN` clause, hoặc DataLoader pattern cho GraphQL.

### Câu 5: Khi nào dùng Redis thay vì query DB trực tiếp?

**Gợi ý trả lời:** Read-heavy data ít thay đổi (product catalog, user profile), session storage, rate limiting counters, pub/sub real-time. Không cache data cần strong consistency mà không có invalidation strategy.

---

**Xem tiếp:** [1-postgresql-pg.md](./1-postgresql-pg.md) — bắt đầu với PostgreSQL driver cơ bản.
