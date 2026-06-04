# 01 — Nền Tảng Java & Spring

> Module này xây dựng nền móng vững chắc trước khi đi vào các tính năng nâng cao của Spring Boot.
> Hiểu rõ các khái niệm ở đây sẽ giúp bạn học mọi module còn lại nhanh hơn và sâu hơn nhiều.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Giải thích **IoC** (Inversion of Control — Đảo Ngược Quyền Kiểm Soát) và **DI** (Dependency Injection — Tiêm Phụ Thuộc) mà không cần nhìn tài liệu
- [ ] Phân biệt `ApplicationContext` và `BeanFactory`
- [ ] Mô tả **Bean Lifecycle** (Vòng Đời Bean) từ khởi tạo đến hủy
- [ ] Sử dụng đúng **Bean Scopes** (Phạm Vi Bean) — Singleton, Prototype, Request, Session
- [ ] Cấu hình ứng dụng bằng `@Value`, `@ConfigurationProperties` và **Profiles** (Môi Trường)
- [ ] Hiểu cơ chế **Auto-configuration** (Tự Động Cấu Hình) của Spring Boot
- [ ] Sử dụng các tính năng Java 17+ như Records, Sealed Classes, Pattern Matching

---

## 🗂️ Danh Sách Bài Học

| File | Chủ Đề | Thời Gian | Độ Khó |
|------|--------|-----------|--------|
| [1-java-core.md](1-java-core.md) | Java 17+ — Records, Sealed Classes, Streams, Virtual Threads | 90 phút | ⭐⭐ |
| [2-spring-core.md](2-spring-core.md) | Spring Core — IoC Container, DI, ApplicationContext | 90 phút | ⭐⭐ |
| [3-spring-boot-basics.md](3-spring-boot-basics.md) | Spring Boot — Auto-configuration, Starters, @SpringBootApplication | 60 phút | ⭐ |
| [4-bean-lifecycle.md](4-bean-lifecycle.md) | Bean Lifecycle & Scopes — Singleton, Prototype | 60 phút | ⭐⭐ |
| [5-configuration.md](5-configuration.md) | Cấu Hình — @Value, @ConfigurationProperties, Profiles | 60 phút | ⭐⭐ |

**Tổng thời gian ước tính: 4–6 giờ**

---

## 🔁 Thứ Tự Học Khuyến Nghị

```
1-java-core.md
      ↓
2-spring-core.md       ← Quan trọng nhất — đọc kỹ
      ↓
3-spring-boot-basics.md
      ↓
4-bean-lifecycle.md
      ↓
5-configuration.md
```

> **Lưu ý:** Nếu bạn đã có kinh nghiệm Java, có thể bắt đầu từ `2-spring-core.md`.

---

## 🧠 Khái Niệm Cốt Lõi

### IoC — Inversion of Control (Đảo Ngược Quyền Kiểm Soát)

Thay vì code của bạn tự tạo và quản lý các đối tượng phụ thuộc, Spring Container sẽ làm điều đó thay bạn.

```
Không có IoC:                    Có IoC (Spring):
┌─────────────┐                  ┌─────────────┐
│  OrderService│                  │Spring IoC   │
│  new UserRepo│  ←── tự tạo     │  Container  │
│  new EmailSvc│                  │             │
└─────────────┘                  └──────┬──────┘
                                        │ inject
                                 ┌──────▼──────┐
                                 │ OrderService │
                                 │ (nhận vào)  │
                                 └─────────────┘
```

### DI — Dependency Injection (Tiêm Phụ Thuộc)

DI là cơ chế Spring dùng để thực hiện IoC — Spring **tiêm** (inject) các dependency vào object thay vì object tự tạo.

**Ba dạng DI trong Spring:**
- **Constructor Injection** (Tiêm qua Constructor) — khuyến nghị
- **Setter Injection** (Tiêm qua Setter) — dùng cho optional dependency
- **Field Injection** (Tiêm qua Field) — tiện nhưng khó test, không khuyến khích

### Bean

Bất kỳ object nào được Spring IoC Container quản lý đều được gọi là **Bean**.
Spring tạo, cấu hình, và quản lý toàn bộ vòng đời của Bean.

---

## 📐 Sơ Đồ Kiến Trúc Spring Boot

```
┌──────────────────────────────────────────────────────────┐
│                    Spring Boot Application                │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │          Spring IoC Container (ApplicationContext) │  │
│  │                                                    │  │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────────┐   │  │
│  │   │Controller│  │ Service  │  │  Repository  │   │  │
│  │   │  (Bean)  │→ │  (Bean)  │→ │    (Bean)    │   │  │
│  │   └──────────┘  └──────────┘  └──────────────┘   │  │
│  │                                                    │  │
│  │   ┌──────────────────────────────────────────┐    │  │
│  │   │  Auto-Configuration + Starters            │    │  │
│  │   │  (DataSource, Security, Web, Cache...)    │    │  │
│  │   └──────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   JVM    │  │   JDK    │  │  Library │               │
│  └──────────┘  └──────────┘  └──────────┘               │
└──────────────────────────────────────────────────────────┘
```

---

## ✅ Checklist Trước Khi Sang Module Tiếp Theo

Trả lời được các câu hỏi sau thì đã sẵn sàng:

- [ ] IoC là gì? Tại sao cần IoC?
- [ ] Sự khác biệt giữa `@Component`, `@Service`, `@Repository`?
- [ ] `@Autowired` Constructor vs Field Injection — khi nào dùng cái nào?
- [ ] Auto-configuration hoạt động thế nào? `@Conditional` là gì?
- [ ] Bean Singleton vs Prototype — khác nhau gì?
- [ ] `@Value` vs `@ConfigurationProperties` — khi nào dùng cái nào?
- [ ] Spring Profiles dùng để làm gì?

---

## 🔗 Liên Kết Tiếp Theo

Sau khi hoàn thành module này:

- → [02-web-layer/](../02-web-layer/) — Xây dựng REST API
- → [03-data-access/](../03-data-access/) — Spring Data JPA

---

**Cập Nhật Lần Cuối:** 2026-06-02
