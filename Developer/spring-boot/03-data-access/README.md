# 03 — Truy Cập Dữ Liệu (Data Access)

> Module này bao gồm toàn bộ kiến thức về tầng dữ liệu trong Spring Boot — từ Spring Data JPA, ánh xạ quan hệ ORM,
> quản lý giao dịch, giải quyết N+1 Problem, caching với Redis, đến tích hợp MongoDB.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Sử dụng **Spring Data JPA** — `JpaRepository`, JPQL, Criteria API, Derived Query Methods
- [ ] Ánh xạ quan hệ ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng Quan Hệ) — `@OneToMany`, `@ManyToMany`, Cascade
- [ ] Hiểu và áp dụng đúng **`@Transactional`** — Propagation (Lan Truyền) & Isolation Levels (Mức Cô Lập)
- [ ] Phát hiện và giải quyết **N+1 Problem** — `@EntityGraph`, `JOIN FETCH`, DTO Projection
- [ ] Tích hợp **Redis** làm cache layer với `@Cacheable`, `RedisTemplate`
- [ ] Dùng **Spring Data MongoDB** — `MongoRepository`, Aggregation Pipeline

---

## 🗂️ Danh Sách Bài Học

| File | Chủ Đề | Thời Gian | Độ Khó |
|------|--------|-----------|--------|
| [1-spring-data-jpa.md](1-spring-data-jpa.md) | JpaRepository, @Entity, JPQL, Criteria API, Pagination | 120 phút | ⭐⭐ |
| [2-orm-mapping.md](2-orm-mapping.md) | Relationships, @OneToMany, CascadeType, FetchType, Inheritance | 90 phút | ⭐⭐⭐ |
| [3-transactions.md](3-transactions.md) | @Transactional, Propagation, Isolation Levels, Rollback | 90 phút | ⭐⭐⭐ |
| [4-n-plus-one-problem.md](4-n-plus-one-problem.md) | Lazy/Eager loading, @EntityGraph, JOIN FETCH, BatchSize | 90 phút | ⭐⭐⭐ |
| [5-spring-data-redis.md](5-spring-data-redis.md) | RedisTemplate, @Cacheable, RedisRepository, TTL, Pub/Sub | 60 phút | ⭐⭐ |
| [6-mongodb-integration.md](6-mongodb-integration.md) | MongoRepository, MongoTemplate, Aggregation Pipeline | 60 phút | ⭐⭐ |

**Tổng thời gian ước tính: 8–10 giờ**

---

## 🔁 Thứ Tự Học Khuyến Nghị

```
1-spring-data-jpa.md           ← Nền tảng — học trước tiên
      ↓
2-orm-mapping.md               ← Xây dựng trên JPA — entity relationships
      ↓
3-transactions.md              ← Bắt buộc trước khi build service layer
      ↓
4-n-plus-one-problem.md        ← Tối ưu sau khi có cơ bản
      ↓
5-spring-data-redis.md         ← Caching layer bổ sung
      ↓
6-mongodb-integration.md       ← NoSQL — học sau cùng
```

---

## 🧠 Kiến Trúc Tầng Dữ Liệu

```
                    Service Layer (Tầng Nghiệp Vụ)
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       JPA Repository  RedisCache   MongoRepository
              │            │            │
              ▼            ▼            ▼
         PostgreSQL      Redis        MongoDB
          (SQL DB)     (In-Memory)   (Document DB)
```

### Spring Data — Cây Phả Hệ Repository

```
Repository<T, ID>              ← Interface gốc (rỗng, chỉ đánh dấu)
    └── CrudRepository<T, ID>  ← CRUD cơ bản: save, findById, delete
         └── PagingAndSortingRepository<T, ID>  ← Thêm phân trang & sắp xếp
              └── JpaRepository<T, ID>          ← JPA-specific: flush, batch
```

---

## 🗄️ Stack Công Nghệ

| Công Nghệ | Vai Trò | Dependency (spring-boot-starter-*) |
|-----------|---------|--------------------------------------|
| **Hibernate** | JPA Provider (Nhà Cung Cấp JPA) — ORM engine | `data-jpa` |
| **Spring Data JPA** | Repository abstraction (Trừu Tượng Repository) | `data-jpa` |
| **HikariCP** | Connection Pool (Bể Kết Nối) — mặc định trong Boot | `data-jpa` |
| **Flyway / Liquibase** | Database Migration (Di Chuyển Schema DB) | `flyway` / `liquibase` |
| **Redis** | In-memory data store — cache & session | `data-redis` |
| **MongoDB** | Document database (Cơ Sở Dữ Liệu Tài Liệu) | `data-mongodb` |

---

## ⚡ Câu Hỏi Phỏng Vấn Thường Gặp

### Cấp Độ Junior

1. `JpaRepository` khác gì `CrudRepository`?
2. `@Entity` và `@Table` dùng để làm gì?
3. Derived Query Method là gì? Ví dụ `findByEmail`?
4. `LAZY` và `EAGER` loading khác nhau thế nào?

### Cấp Độ Mid-Level

1. **N+1 Problem** là gì và cách phát hiện?
2. `@Transactional` hoạt động thế nào thông qua proxy?
3. Các mức **Propagation** của `@Transactional` — khi nào dùng `REQUIRES_NEW`?
4. **Isolation Level** `REPEATABLE_READ` khác `READ_COMMITTED` thế nào?
5. Khi nào dùng `@EntityGraph` thay vì `JOIN FETCH`?

### Cấp Độ Senior

1. Cách thiết kế bidirectional relationship (quan hệ hai chiều) tránh vòng lặp vô hạn?
2. Optimistic Locking (Khóa Lạc Quan) vs Pessimistic Locking (Khóa Bi Quan) — khi nào dùng?
3. Caching strategy: Local Cache vs Distributed Cache — trade-offs?
4. Cách handle distributed transaction (giao dịch phân tán) trong microservices?

---

## 🔗 Liên Kết Với Các Module Khác

| Module | Liên Quan |
|--------|-----------|
| `02-web-layer/` | Controller gọi Service → Repository |
| `04-security/` | `@PreAuthorize` kết hợp với data access |
| `06-testing/` | `@DataJpaTest` — test Repository layer |
| `07-performance/` | Caching, Connection Pool, Query Optimization |

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
