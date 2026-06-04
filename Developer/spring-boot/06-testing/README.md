# 06 — Kiểm Thử (Testing)

> Module này bao gồm toàn bộ chiến lược kiểm thử trong Spring Boot —
> từ Unit Test (Kiểm Thử Đơn Vị) với JUnit 5 & Mockito, Integration Test (Kiểm Thử Tích Hợp)
> với Testcontainers, kiểm thử từng tầng riêng biệt, đến đo lường độ phủ code với JaCoCo.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Viết **Unit Test** (Kiểm Thử Đơn Vị) với **JUnit 5** — `@Test`, `@ParameterizedTest`, `@ExtendWith`, các assertions nâng cao
- [ ] Sử dụng **Mockito** — `@Mock`, `@InjectMocks`, `verify()`, `ArgumentCaptor` (Bộ Chụp Đối Số) để cô lập phụ thuộc
- [ ] Viết **Integration Test** (Kiểm Thử Tích Hợp) với `@SpringBootTest` và **Testcontainers** (DB/Kafka thực trong Docker)
- [ ] Kiểm thử tầng Web với **`@WebMvcTest`** và **`MockMvc`** (Bộ Giả Lập MVC)
- [ ] Kiểm thử tầng Data với **`@DataJpaTest`** và cơ sở dữ liệu in-memory **H2**
- [ ] Đo lường và duy trì **Code Coverage** (Độ Phủ Code) với **JaCoCo** và mutation testing

---

## 🗂️ Danh Sách Bài Học

| File | Chủ Đề | Thời Gian | Độ Khó |
|------|--------|-----------|--------|
| [1-unit-testing.md](1-unit-testing.md) | JUnit 5 — @Test, @ParameterizedTest, assertions, lifecycle, extensions | 90 phút | ⭐⭐ |
| [2-mocking.md](2-mocking.md) | Mockito — @Mock, @InjectMocks, stubbing, verify, ArgumentCaptor | 90 phút | ⭐⭐ |
| [3-integration-testing.md](3-integration-testing.md) | @SpringBootTest, Testcontainers, @DirtiesContext, test slices | 120 phút | ⭐⭐⭐ |
| [4-web-layer-testing.md](4-web-layer-testing.md) | @WebMvcTest, MockMvc, @MockBean, security trong test | 90 phút | ⭐⭐ |
| [5-data-layer-testing.md](5-data-layer-testing.md) | @DataJpaTest, H2, Flyway test, custom queries | 90 phút | ⭐⭐ |
| [6-test-coverage.md](6-test-coverage.md) | JaCoCo, mutation testing (PITest), coverage thresholds, CI integration | 60 phút | ⭐⭐ |

**Tổng thời gian ước tính: 6–8 giờ**

---

## 🔁 Thứ Tự Học Khuyến Nghị

```
1-unit-testing.md            ← Nền tảng — học trước tiên
      ↓
2-mocking.md                 ← Cô lập phụ thuộc với Mockito
      ↓
4-web-layer-testing.md       ← Kiểm thử Controller với MockMvc
      ↓
5-data-layer-testing.md      ← Kiểm thử Repository với @DataJpaTest
      ↓
3-integration-testing.md     ← Kiểm thử toàn bộ với Testcontainers
      ↓
6-test-coverage.md           ← Đo lường và cải thiện coverage
```

---

## 🧠 Bức Tranh Tổng Thể — Testing Pyramid (Kim Tự Tháp Kiểm Thử)

```
           ▲
          /E2E\          ← End-to-End Test (Kiểm Thử Đầu Cuối)
         /─────\            Ít nhất, chậm nhất, tốn kém nhất
        / Integ \        ← Integration Test (Kiểm Thử Tích Hợp)
       /─────────\          @SpringBootTest, Testcontainers
      /  Unit     \      ← Unit Test (Kiểm Thử Đơn Vị)
     /─────────────\        JUnit 5 + Mockito
    ▼               ▼    ← Nhiều nhất, nhanh nhất, rẻ nhất
```

### Spring Test Slices (Lát Cắt Kiểm Thử Theo Tầng)

```
┌─────────────────────────────────────────────────────────────┐
│                     @SpringBootTest                         │
│              (Toàn bộ Application Context)                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │   @WebMvcTest          @DataJpaTest   @DataRedisTest   │ │
│  │  (Web Layer only)    (JPA Layer only)                  │ │
│  │   Controller           Repository                      │ │
│  │   Filter               Entity                          │ │
│  │   ControllerAdvice     @Query                          │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### Unit Test vs Integration Test vs E2E Test

| Đặc Điểm | Unit Test | Integration Test | E2E Test |
|-----------|-----------|------------------|----------|
| **Phạm vi** | 1 class/method | Nhiều component | Toàn bộ hệ thống |
| **Tốc độ** | Milli giây | Giây | Phút |
| **Phụ thuộc** | Mock hoàn toàn | Một phần mock | Không mock |
| **Chi phí** | Rất thấp | Trung bình | Cao |
| **Tỷ lệ khuyến nghị** | 70% | 20% | 10% |
| **Tool** | JUnit 5 + Mockito | @SpringBootTest + Testcontainers | Selenium, Playwright |

### Test Doubles (Bản Sao Kiểm Thử) — Phân Loại

| Loại | Mô Tả | Mockito |
|------|--------|---------|
| **Dummy** | Truyền vào nhưng không dùng | `null` hoặc `mock()` |
| **Stub** | Trả về giá trị cố định | `when(...).thenReturn(...)` |
| **Mock** | Xác minh tương tác | `verify(mock).method()` |
| **Spy** | Wrap object thực, override một phần | `@Spy` / `spy(realObject)` |
| **Fake** | Implement đơn giản hóa | H2 thay PostgreSQL |

---

## ⚡ Khi Nào Dùng Gì?

```
Kiểm thử logic nghiệp vụ trong Service?
  └─► Unit Test với JUnit 5 + Mockito — nhanh, cô lập hoàn toàn

Kiểm thử Controller endpoint (request/response, validation)?
  └─► @WebMvcTest + MockMvc — không cần Spring Security đầy đủ

Kiểm thử Repository với custom @Query?
  └─► @DataJpaTest với H2 — nhẹ, chỉ load JPA context

Kiểm thử toàn bộ flow từ API đến Database?
  └─► @SpringBootTest + Testcontainers (PostgreSQL thực)

Kiểm thử với Kafka thực?
  └─► Testcontainers KafkaContainer trong @SpringBootTest

Đo độ phủ code và enforce thresholds?
  └─► JaCoCo với Maven/Gradle plugin
```

---

## ⚠️ Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
|-----|-------------|----------|
| Test chậm do `@SpringBootTest` | Load toàn bộ context cho mọi test | Dùng test slices hoặc chia sẻ context |
| `@MockBean` làm dirty context | Mỗi `@MockBean` tạo context mới | Gom tất cả `@MockBean` vào base class |
| Testcontainers khởi động lại liên tục | Mỗi test class tạo container mới | Dùng `@Container` + `static` field |
| H2 không tương thích với PostgreSQL | Dùng SQL đặc thù PostgreSQL | Chuyển sang Testcontainers PostgreSQL |
| Test order dependency | Test phụ thuộc vào thứ tự chạy | Dùng `@DirtiesContext` hoặc cleanup `@AfterEach` |
| `@Transactional` trong test rollback | Transaction rollback sau mỗi test | Cố ý — đây là hành vi đúng, không cần sửa |

---

## 📊 Cấu Trúc Project Test Chuẩn

```
src/
├── main/java/com/example/
│   ├── controller/
│   │   └── OrderController.java
│   ├── service/
│   │   └── OrderService.java
│   └── repository/
│       └── OrderRepository.java
│
└── test/java/com/example/
    ├── unit/                          ← Unit Tests
    │   ├── service/
    │   │   └── OrderServiceTest.java  ← @ExtendWith(MockitoExtension)
    │   └── util/
    │       └── PriceCalculatorTest.java
    │
    ├── web/                           ← Web Layer Tests
    │   └── controller/
    │       └── OrderControllerTest.java  ← @WebMvcTest
    │
    ├── data/                          ← Data Layer Tests
    │   └── repository/
    │       └── OrderRepositoryTest.java  ← @DataJpaTest
    │
    └── integration/                   ← Integration Tests
        └── OrderFlowIT.java           ← @SpringBootTest + Testcontainers
```

---

## 📚 Tài Liệu Tham Khảo

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito Documentation](https://site.mockito.org/)
- [Testcontainers for Java](https://java.testcontainers.org/)
- [Spring Boot Testing Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)
- [JaCoCo](https://www.jacoco.org/jacoco/trunk/doc/)

---

## 🎯 Câu Hỏi Phỏng Vấn Quan Trọng

- [ ] Sự khác biệt giữa `@Mock` và `@MockBean` trong Spring?
- [ ] Tại sao nên dùng Constructor Injection thay vì Field Injection cho dễ test?
- [ ] `@SpringBootTest` khác `@WebMvcTest` và `@DataJpaTest` như thế nào?
- [ ] Testcontainers giải quyết vấn đề gì so với H2 in-memory?
- [ ] `@DirtiesContext` là gì? Khi nào nên dùng và khi nào không nên?
- [ ] Mutation Testing (Kiểm Thử Đột Biến) là gì? Tại sao code coverage 100% vẫn có thể thiếu chất lượng?
- [ ] Cách test Spring Security — `@WithMockUser` dùng thế nào?
- [ ] Giải thích chiến lược "Test Pyramid" — tỷ lệ unit/integration/e2e lý tưởng?
