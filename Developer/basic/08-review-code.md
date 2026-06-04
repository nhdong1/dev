# Backend Code Review – Theory & Principles

## 1. Mục tiêu của code review

- Đảm bảo **đúng chức năng** (correctness).
- Giảm **bug** và **rủi ro production**.
- Nâng **chất lượng thiết kế, kiến trúc**.
- Tăng **readability, maintainability** để team dễ làm việc chung.
- Chia sẻ kiến thức, chuẩn hóa **coding standard** trong team.

---

## 2. Nguyên tắc chung khi review

- **Review logic, không review cá nhân** – tập trung vào code, không phán xét người viết.
- **Trade-off & context** – không áp tiêu chuẩn tuyệt đối, cân nhắc deadline, rủi ro, scope.
- **Small, frequent PR** – PR nhỏ dễ review, dễ rollback.
- **Consistency > Perfection** – ưu tiên nhất quán với codebase & convention hiện tại.
- **Ask, don’t tell** – dùng câu hỏi (“vì sao”, “có thể”) thay vì mệnh lệnh.

---

## 3. Correctness & Business Logic

Mục tiêu: code làm đúng requirement, xử lý đúng mọi luồng.

Checklist:
- Input/Output đúng như **spec**? Có cover happy path & edge cases?
- Có xử lý **error path**, case null/empty, timeout, network error?
- Có **idempotent** khi cần (retry, message re-delivery, API bị gọi lại)?
- Logic có phụ thuộc **timezone, locale, currency, rounding**… đã được xử lý rõ ràng?

---

## 4. Readability & Clean Code

Mục tiêu: người khác hiểu code nhanh, ít phải suy luận.

Checklist:
- Tên **variable, function, class** rõ nghĩa, domain-driven, tránh viết tắt mơ hồ.
- Hàm có **một trách nhiệm chính** (Single Responsibility), ngắn, ít nhánh lồng nhau.
- Tránh **magic number/string** – đưa vào constant, enum.
- Comment tập trung giải thích **“tại sao”**, không lặp lại “cái gì”.
- Code style theo chuẩn thống nhất (formatter, linter).

---

## 5. Maintainability & Modularity

Mục tiêu: dễ sửa, dễ mở rộng, ít ảnh hưởng chéo.

Checklist:
- Có tuân thủ **SOLID** (đặc biệt SRP, OCP, DIP)?
- Logic business tách khỏi **layer infrastructure** (DB, HTTP, message queue)?
- Tránh **duplicate code** – có thể extract thành helper/service chung?
- Boundary giữa module rõ ràng? Tránh **circular dependency**?
- Config (URL, key, feature flag) tách khỏi code, dùng **config/env**?

---

## 6. Architecture & Layering (Backend)

Mục tiêu: tuân thủ kiến trúc đã chọn (layered, hexagonal, clean architecture, microservice).

Checklist:
- Controller/API chỉ làm **orchestration**, không nhét full business logic.
- Service layer/Domain layer chứa **business rule**, không lẫn SQL/HTTP client trực tiếp (nếu theo clean architecture).
- Repository/DAO chỉ xử lý **truy cập dữ liệu**, không chứa business logic.
- API, DTO, domain model tách biệt khi cần; tránh **leak** entity DB ra public API.
- Service boundaries rõ ràng, giao tiếp qua **API/event** chuẩn.

---

## 7. Performance & Efficiency

Mục tiêu: tránh performance issue rõ ràng, đặc biệt trong backend high load.

Checklist:
- Có **N+1 query**? Có dùng eager/lazy loading hợp lý, join/batch?
- Có xử lý **pagination** thay vì trả hết dữ liệu?
- Các loop, transformation có thể tối ưu (avoid O(N²) không cần thiết)?
- Có kế hoạch **caching** hợp lý (in-memory, Redis), invalidation rõ ràng?
- Tài nguyên như connection, file, thread… được **đóng/giải phóng** đúng cách?

---

## 8. Security Principles

Mục tiêu: giảm lỗ hổng bảo mật phổ biến trong backend.

Checklist:
- Authorization:
  - Có check **permission** đúng chỗ, không chỉ dựa vào client?
  - Endpoint nào public, private, internal được phân tách rõ?
- Authentication:
  - Có validate token (JWT, session, OAuth) đúng chuẩn, hạn sử dụng?
- Input validation:
  - Validate input (type, range, format) trước khi xử lý, tránh **injection**.
- Data protection:
  - Không log **PII, secret, token, password**.
  - Dùng **hash (bcrypt/argon2)** cho password, không lưu plaintext.
  - Sử dụng HTTPS, secure cookie, CSRF protection (nếu applicable).
- Secrets:
  - Không commit **secret, key, password** vào repo; dùng secret manager/env.

---

## 9. Reliability, Fault Tolerance & Resilience

Mục tiêu: hệ thống ổn định, degrade gracefully khi có lỗi.

Checklist:
- Có **timeout** cho call ra ngoài (DB, HTTP, MQ), tránh treo thread.
- Có **retry with backoff** cho lỗi tạm thời; không retry vô hạn.
- Có **circuit breaker, bulkhead** (nếu cần) để cô lập lỗi dependency.
- Error handling:
  - Không swallow exception; log đủ context (correlation id, request id).
  - Phân biệt **client error (4xx)** vs **server error (5xx)**.
- Có **dead-letter queue** cho message không xử lý được?

---

## 10. Scalability & Distributed Systems

Mục tiêu: code phù hợp chạy trên nhiều instance, dễ scale.

Checklist:
- Stateless:
  - Service có **stateless** chưa? Session/state lưu ở store ngoài (DB/cache)?
- Idempotency & retry-safe:
  - API, consumer message có thể được gọi lại mà không gây **double processing**?
- Locking & contention:
  - Tránh lock global gây bottleneck; sử dụng **optimistic locking** khi phù hợp.
- Dependency:
  - Service có coupling chặt với service khác? Có thể dùng async/event để giảm coupling?

---

## 11. Database & Data Integrity

Mục tiêu: đảm bảo dữ liệu đúng, tránh corruption, race condition.

Checklist:
- Query sử dụng **index** hợp lý? Có nguy cơ full table scan?
- Transaction:
  - Có bao phủ đầy đủ các thao tác cần **atomic**?
  - Isolation level phù hợp (read committed, repeatable read, serializable)?
- Constraint:
  - Dùng **constraint (PK, FK, unique, check)** trên DB, không chỉ ở app.
- Migration:
  - Thay đổi schema có backward compatible (zero-downtime) không?
  - Có migration script, versioning rõ ràng?

---

## 12. Concurrency & Thread Safety

Mục tiêu: tránh race condition, deadlock, data corruption.

Checklist:
- Shared mutable state:
  - Có biến global, cache in-memory được **access từ nhiều thread**?
  - Đã dùng synchronization hoặc pattern bất biến/immmutable?
- Async/await, futures, promise:
  - Có handle mọi **error path** của async call?
  - Không block event loop (trong Node.js) hay thread pool bằng operation nặng?
- Locking:
  - Có thể gây **deadlock**? Lock order rõ ràng?
  - Đã cân nhắc **optimistic vs pessimistic locking**?

---

## 13. Testing & Testability

Mục tiêu: code dễ test, có đủ coverage tối thiểu.

Checklist:
- Có **unit test** cho business logic quan trọng?
- Integration test cho path chính (API, DB, MQ) khi cần?
- Code được thiết kế để dễ mock dependency (DI, interface, adapter)?
- Test cover:
  - Happy path, edge case, error path.
  - Idempotent behavior, retry, timeout.

---

## 14. Observability: Logging, Metrics, Tracing

Mục tiêu: dễ debug production, theo dõi health hệ thống.

Checklist:
- Logging:
  - Log structured (JSON), có level rõ ràng (debug/info/warn/error).
  - Không spam log, không log data nhạy cảm.
- Metrics:
  - Có metric cho throughput, latency, error rate, resource usage?
  - Đặt name/label nhất quán (Prometheus, etc.).
- Tracing:
  - Có **correlation id / trace id** đi xuyên suốt request?
  - Log/traces đủ context để recreate flow lỗi?

---

## 15. DevOps & Operational Concerns

Mục tiêu: code dễ deploy, rollback, vận hành.

Checklist:
- Config:
  - Dùng **12-factor**: config qua env/secret, không hardcode.
- Startup/Shutdown:
  - Graceful shutdown (hoàn thành request dang dở, đóng connection).
- Health check:
  - Có endpoint **liveness/readiness** cho load balancer/k8s?
- Backward compatibility:
  - Thay đổi API/schema có gây break client cũ? Có versioning/feature flag?

---

## 16. Communication khi review

- Comment phải:
  - **Rõ ràng, cụ thể**, đưa lý do và tài liệu tham khảo nếu có (link spec, doc).
  - Đề xuất **giải pháp** thay thế, không chỉ nói “chưa tốt”.
- Phân biệt:
  - **Blocking**: bug, security risk, breaking change.
  - **Non-blocking**: cải thiện style, tối ưu nhỏ, gợi ý refactor tương lai.

---
Dưới đây là “bộ khung” phổ biến nên áp dụng khi review code:

1.  **Clean Code / Readability**
    -   Dễ đọc, dễ hiểu, dễ đổi người maintain.
    -   Tên biến/hàm/class rõ nghĩa, hàm ngắn, một hàm làm một việc.
    -   Comment chỉ giải thích _tại sao_, không lặp lại _cái gì_.
2.  **SOLID (cho OOP)**
    -   **S**ingle Responsibility: mỗi class/module chỉ có _một lý do_ để thay đổi.
    -   **O**pen/Closed: dễ mở rộng, hạn chế sửa code cũ.
    -   **L**iskov Substitution: subclass thay thế được cho base class mà không phá logic.
    -   **I**nterface Segregation: interface nhỏ, chuyên biệt.
    -   **D**ependency Inversion: phụ thuộc abstraction, không phụ thuộc implementation cụ thể.
3.  **KISS, DRY, YAGNI**
    -   **KISS** (Keep It Simple, Stupid): ưu tiên giải pháp đơn giản, rõ ràng.
    -   **DRY** (Don’t Repeat Yourself): không lặp lại logic, trích ra hàm/chung module.
    -   **YAGNI** (You Aren’t Gonna Need It): không viết trước tính năng/chung chung khi chưa cần.
4.  **Design & Architecture**
    -   Tách biệt layer (API, service, repository, domain…).
    -   Tuân thủ pattern đã thống nhất (DDD, CQRS, hexagonal,… nếu có).
    -   Không “leak” logic domain xuống tầng dưới/ngoài.
5.  **Correctness & Robustness**
    -   Đúng yêu cầu business, handle edge cases.
    -   Kiểm tra null, validate input, xử lý lỗi, log đủ thông tin.
6.  **Performance & Complexity**
    -   Độ phức tạp hợp lý (không O(n³) khi có thể O(n log n)).
    -   Tránh query n+1, loop lồng nhau không cần thiết, allocation dư thừa.
7.  **Security & Safety**
    -   Tránh SQL injection, XSS, lộ secret (API key, password…).
    -   Check quyền truy cập (auth, authz), validate dữ liệu client gửi lên.
8.  **Testability**
    -   Code dễ test: ít phụ thuộc cứng, dùng DI, interface.
    -   Có unit/integration test cho logic quan trọng, happy + edge case.
9.  **Consistency & Convention**
    -   Tuân thủ coding convention, style guide, format chung của team.
    -   Cách đặt tên, cấu trúc folder, pattern thống nhất trong toàn project.

Khi review, có thể lần lượt “soi” theo trục:

-   **Đúng** (correctness)
-   **Rõ** (readability / maintainability)
-   **Gọn** (design / redundancy / simplicity)
-   **Nhanh & An toàn** (performance, security)