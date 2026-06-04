# ⭐ Template Câu Chuyện STAR — Phỏng Vấn Kỹ Thuật

> STAR là framework trả lời câu hỏi hành vi (behavioral questions) và câu hỏi kinh nghiệm thực tế. Tài liệu này cung cấp templates sẵn cho các tình huống kỹ thuật phổ biến nhất, kèm hướng dẫn tùy chỉnh theo kinh nghiệm của bạn.

---

## 🎯 Cấu Trúc STAR

```
S — Situation (Tình Huống): Bối cảnh — dự án gì, team bao nhiêu người, thời gian nào
T — Task (Nhiệm Vụ):       Bạn cần giải quyết vấn đề gì, yêu cầu là gì
A — Action (Hành Động):    Bạn đã làm gì cụ thể, lý do chọn approach này
R — Result (Kết Quả):      Impact đo lường được, bài học rút ra
```

### Quy Tắc Vàng

- **Dùng "Tôi" thay vì "Chúng tôi"** — interviewer muốn biết BẠN làm gì
- **Kết quả phải có số** — "cải thiện 40%" tốt hơn "cải thiện đáng kể"
- **Thời gian trả lời:** 2–4 phút mỗi câu chuyện
- **Chuẩn bị 5–7 câu chuyện** để cover nhiều loại câu hỏi

---

## 📖 Template 1: Giải Quyết Performance Problem (Vấn Đề Hiệu Năng)

**Dùng để trả lời các câu hỏi về:** optimization, debugging, performance tuning

### Template Cấu Trúc

**Situation:**
"Trong dự án [TÊN DỰ ÁN], team tôi xây dựng [MÔ TẢ HỆ THỐNG]. Khoảng [THỜI GIAN], chúng tôi nhận được phản ánh từ users/monitoring rằng [MÔ TẢ TRIỆU CHỨNG]."

**Task:**
"Tôi được giao nhiệm vụ điều tra và giải quyết vấn đề này trong vòng [THỜI HẠN]. Mục tiêu cụ thể là [METRIC MỤC TIÊU]."

**Action:**
"Tôi bắt đầu bằng việc:
1. Thu thập dữ liệu — [CÔNG CỤ ĐÃ DÙNG: logs, profiler, APM]
2. Phát hiện root cause — [MÔ TẢ PHÁT HIỆN]
3. Đề xuất và implement giải pháp — [MÔ TẢ GIẢI PHÁP CỤ THỂ]
4. Đo lường kết quả sau khi implement"

**Result:**
"Kết quả là [METRIC CỤ THỂ]. Ngoài ra, tôi rút ra bài học [BÀI HỌC] và áp dụng [PREVENTION MEASURE — BIỆN PHÁP PHÒNG NGỪA] để tránh tái diễn."

---

### Câu Chuyện Mẫu: Fix N+1 Problem

**Situation:**
"Trong dự án e-commerce với Spring Boot và PostgreSQL, team 5 người. Sau khi traffic tăng lên 10,000 users/ngày, tôi nhận thấy endpoint GET /api/orders bắt đầu có latency (độ trễ) tăng từ 200ms lên 3 giây, CPU database spike lên 90%."

**Task:**
"Tôi được giao diagnose (chẩn đoán) và fix vấn đề performance của endpoint này trong 2 ngày, với target là đưa latency xuống dưới 300ms."

**Action:**
"Tôi bắt đầu bằng cách bật `show-sql` và `generate_statistics` trong Hibernate để quan sát SQL queries thực tế. Phát hiện ra với 100 orders, hệ thống đang thực thi 201 queries — 1 query lấy orders, 100 queries lấy customer info (customers thông tin), và 100 queries lấy order items. Đây là N+1 Problem điển hình.

Tôi đã:
1. Thay `findAll()` bằng JPQL với `JOIN FETCH`: `SELECT o FROM Order o JOIN FETCH o.customer JOIN FETCH o.items WHERE...`
2. Tuy nhiên JOIN FETCH với 2 collections gây CartesianProduct (tích Đề-các) — số rows tăng gấp bội. Tôi phân tách thành 2 queries riêng cho customer và items
3. Bật Hibernate BatchSize cho collections: `@BatchSize(size = 20)`
4. Thêm database indexes trên `customer_id` và `order_id`"

**Result:**
"Sau khi deploy, latency giảm từ 3 giây xuống còn 85ms (giảm 97%). Số queries từ 201 xuống còn 3. CPU database giảm từ 90% xuống 15%. Tôi cũng viết thêm một performance test với Gatling (công cụ kiểm thử tải) để làm regression baseline, và added documentation về N+1 patterns cho team."

---

## 📖 Template 2: Thiết Kế & Implement Feature Phức Tạp

**Dùng để trả lời:** "Mô tả dự án phức tạp nhất bạn từng làm", "Cách bạn thiết kế hệ thống X"

### Câu Chuyện Mẫu: Implement JWT Authentication

**Situation:**
"Ứng dụng SaaS (Software as a Service) mà team tôi đang phát triển ban đầu dùng session-based authentication với single server. Khi scale lên 3 servers, users bị log out ngẫu nhiên vì session không được share giữa các servers."

**Task:**
"Tôi được assign implement JWT-based stateless authentication để giải quyết session sharing problem, đồng thời phải backward compatible — users đang login không bị force logout."

**Action:**
"Tôi thiết kế giải pháp theo các bước:

**Bước 1 — Nghiên cứu và lên plan:**
Đọc RFC 7519 (JWT spec) và so sánh JWT với session + Redis. Chọn JWT vì team chưa có Redis infrastructure, và stateless authentication phù hợp hơn với microservices roadmap.

**Bước 2 — Implement core:**
- Tạo `JwtService` để sign/verify tokens dùng HS256 algorithm với secret key từ environment variable
- Implement Access Token (15 phút TTL) + Refresh Token (7 ngày, lưu DB)
- Viết `JwtAuthenticationFilter extends OncePerRequestFilter` để verify token mỗi request
- Implement Refresh Token Rotation (xoay vòng Refresh Token) — mỗi lần refresh, token cũ bị xóa

**Bước 3 — Security hardening:**
- Lưu revoked tokens trong blacklist Redis với TTL bằng token expiry
- Rate limiting endpoint `/api/auth/login` với Bucket4j
- Kiểm tra security với OWASP ZAP scanner

**Bước 4 — Migration:**
Deploy dưới feature flag (cờ tính năng) — chạy song song cả 2 systems. Sau 1 tuần không có incident, tắt session system."

**Result:**
"Migration hoàn thành không có downtime, 0 users bị ảnh hưởng. Vấn đề session sharing được giải quyết hoàn toàn. Bonus: response time giảm 30ms vì không cần query session store mỗi request. Tôi cũng viết Internal Tech Talk (buổi chia sẻ kỹ thuật nội bộ) về JWT security cho cả team."

---

## 📖 Template 3: Xử Lý Production Incident (Sự Cố Sản Xuất)

**Dùng để trả lời:** "Kể về lần bạn phải handle bug production", "Cách bạn xử lý khi hệ thống down"

### Template Cấu Trúc

**Situation:**
"Vào [THỜI GIAN], tôi nhận được alert (cảnh báo) từ monitoring rằng [MÔ TẢ SỰ CỐ]. Hệ thống ảnh hưởng [SỐ USERS / DỊCH VỤ NÀO]. Đây là lúc [CONTEXT — ví dụ: peak traffic, sau deployment mới, không rõ lý do]."

**Task:**
"Là [VỊ TRÍ], tôi [ĐƠNMÌNH / CÙNG TEAM] chịu trách nhiệm restore service nhanh nhất có thể và root cause analysis."

**Action:**
"Tôi follow Incident Response Playbook (quy trình ứng phó sự cố):
1. Assess impact — xác định phạm vi ảnh hưởng
2. Communicate — thông báo cho stakeholders
3. Contain — giới hạn damage (rollback nếu cần)
4. Investigate — root cause analysis
5. Fix & verify — deploy fix + confirm resolved"

**Result:**
"Downtime tổng cộng [X phút]. Post-mortem (phân tích sau sự cố) hoàn thành trong 24 giờ. Action items: [CÁC IMPROVEMENT ĐÃ THỰC HIỆN]."

---

### Câu Chuyện Mẫu: Database Connection Pool Exhaustion (Kiệt Sức Bể Kết Nối)

**Situation:**
"Một tối thứ 6, tôi nhận alert: 500 Internal Server Error rate tăng từ 0.1% lên 40% trong vòng 10 phút. Logs hiển thị: `HikariPool-1 - Connection is not available, request timed out after 30000ms`. Đây là production với ~5,000 concurrent users."

**Task:**
"Với tư cách là on-call engineer (kỹ sư trực), tôi cần restore service trong vòng 15 phút (SLA — Service Level Agreement — Cam Kết Mức Dịch Vụ) và tìm root cause."

**Action:**
"**Containment (Giới Hạn Thiệt Hại) — 5 phút đầu:**
- Check `/actuator/metrics/hikaricp.connections` — active connections đang ở mức 20/20 (maxPoolSize)
- Restart 1 trong 3 app servers để free connections → error rate giảm còn 20%
- Tăng `maximumPoolSize` từ 20 lên 30 tạm thời qua Spring Cloud Config

**Investigation (Điều Tra) — trong khi service ổn định hơn:**
- Query `pg_stat_activity` trên PostgreSQL → thấy nhiều queries đang IDLE IN TRANSACTION từ 30+ phút
- Trace lại code → phát hiện một API endpoint mới deploy chiều hôm đó có `@Transactional` nhưng không bao giờ commit vì gọi external API (bên ngoài) bên trong transaction

**Fix:**
- Rollback deployment của endpoint đó → error rate về 0
- Fix code: tách external API call ra ngoài @Transactional boundary
- Add timeout cho transaction: `@Transactional(timeout = 30)`

**Prevention (Phòng Ngừa):**
- Thêm `leak-detection-threshold: 60000` vào HikariCP config
- Thêm rule CodeReview: không được gọi external service trong @Transactional
- Thêm alert cho connection utilization > 80%"

**Result:**
"Total downtime: 8 phút. Sau fix, không tái diễn trong 6 tháng. Post-mortem được chia sẻ với cả engineering team. Tôi còn viết linting rule tự động detect `@Transactional` + external HTTP calls để prevent tương lai."

---

## 📖 Template 4: Cải Thiện Technical Debt (Nợ Kỹ Thuật)

**Dùng để trả lời:** "Kể về lần bạn refactor code", "Cách bạn cải thiện codebase legacy"

### Câu Chuyện Mẫu: Migrate từ Monolith sang Microservices

**Situation:**
"Hệ thống inventory management có lịch sử 5 năm, ~80K lines of code trong 1 Spring Boot monolith. Thời gian build 25 phút, deploy mất 45 phút downtime. Team 12 engineers thường xuyên conflict code khi develop parallel features."

**Task:**
"Tôi được giao lead migration module Inventory sang microservice độc lập — phải không phá vỡ existing functionality, không có extended downtime, hoàn thành trong Q3."

**Action:**
"Tôi áp dụng Strangler Fig Pattern (Kiểu Mẫu Cây Siết):

**Bước 1 — Identify Bounded Context:**
Domain analysis cho thấy Inventory có clear bounded context — chỉ phụ thuộc vào Product IDs từ domain khác, không share database tables với module khác.

**Bước 2 — Create new microservice:**
New Spring Boot application với database riêng (PostgreSQL separate schema), expose REST API + Kafka events.

**Bước 3 — Data migration:**
Script migrate data từ monolith DB sang new DB. Run cả 2 DB song song (dual-write) trong 2 tuần để verify consistency.

**Bước 4 — Traffic migration:**
Deploy API Gateway (Spring Cloud Gateway). Route `/api/inventory/**` → new service. Giữ code cũ trong monolith nhưng disable — rollback trong 5 phút nếu cần.

**Bước 5 — Cleanup:**
Sau 1 tháng stable, xóa code cũ trong monolith."

**Result:**
"Module inventory được tách thành công. Build time module mới: 4 phút (giảm từ 25). Deploy: không có downtime với rolling update. Team inventory deploy độc lập 8 lần/ngày thay vì 2 lần/tuần trước đây. Pattern này sau đó được áp dụng cho 3 module khác."

---

## 📖 Template 5: Leadership & Mentoring (Dẫn Dắt & Hướng Dẫn)

**Dùng để trả lời:** "Kể về lần bạn dạy/hướng dẫn người khác", "Cách bạn influence team không có authority"

### Câu Chuyện Mẫu: Introduce Testing Culture (Giới Thiệu Văn Hóa Kiểm Thử)

**Situation:**
"Team 8 engineers, codebase không có automated tests. Mỗi release đều có regression bugs. Cứ sau mỗi deployment là có ít nhất 1 hotfix (vá lỗi khẩn cấp) trong vòng 24 giờ. Không ai muốn viết tests vì cảm thấy chậm."

**Task:**
"Tôi muốn introduce automated testing nhưng không có quyền mandate (yêu cầu bắt buộc) — chỉ là senior developer trong team."

**Action:**
"Thay vì áp đặt, tôi chọn approach từ từ:

**Tháng 1 — Build trust:**
- Tự mình thêm tests cho module mình đang làm
- Show concrete numbers: trước test = 3 bugs/sprint; sau test = 0 bugs/sprint

**Tháng 2 — Share knowledge:**
- Tổ chức workshop 'Testing with JUnit 5 & Mockito in 30 minutes'
- Pair programming (lập trình đôi) với từng teammate để họ quen

**Tháng 3 — Make it easy:**
- Setup JaCoCo với report trên CI/CD pipeline
- Tạo test templates và conventions trong wiki
- Start code review comments nhẹ nhàng về test coverage

**Tháng 4 — Propose team agreement:**
- Đề xuất: 'Chúng ta có thể thử enforce 60% coverage cho code mới không? Không phải code cũ'
- Được team đồng ý sau khi thấy kết quả tháng trước"

**Result:**
"Sau 6 tháng, coverage tăng từ 0% lên 72%. Regression bugs giảm 80% (từ 12/sprint xuống 2-3/sprint). Deployment confidence tăng — team bắt đầu deploy vào thứ 6 thay vì trước đây tránh deploy cuối tuần vì sợ. Team lead ghi nhận initiative trong performance review của tôi."

---

## 🛠️ Hướng Dẫn Cá Nhân Hóa Stories

### Bước 1: Liệt Kê Tình Huống Từ Kinh Nghiệm

Điền vào bảng này từ kinh nghiệm thực tế của bạn:

| Loại Tình Huống | Tên Dự Án / Thời Gian | Kết Quả Đo Được |
| --------------- | --------------------- | ---------------- |
| Performance fix | | |
| Bug production | | |
| Feature phức tạp | | |
| Architecture decision | | |
| Conflict resolution | | |
| Mentoring / teaching | | |
| Failure / lesson learned | | |

### Bước 2: Bổ Sung Số Liệu

Với mỗi câu chuyện, cố gắng có ít nhất 2 con số cụ thể:

```
Latency: X ms → Y ms (giảm Z%)
Error rate: X% → Y%
Build time: X phút → Y phút
Test coverage: X% → Y%
Deployment frequency: X/tháng → Y/tháng
Team size: N engineers
Timeline: M tháng/tuần/ngày
Business impact: N users, $X revenue affected
```

### Bước 3: Kiểm Tra Câu Chuyện

Trả lời YES cho tất cả câu hỏi sau:

- [ ] Tôi nói "Tôi" chủ yếu, không phải "chúng tôi"?
- [ ] Situation đủ ngắn gọn (30 giây)?
- [ ] Action mô tả cụ thể tôi đã làm gì, tại sao?
- [ ] Result có ít nhất 1 con số đo lường được?
- [ ] Câu chuyện kéo dài 2–4 phút khi đọc to?
- [ ] Có mention bài học rút ra / điều sẽ làm khác?

---

## ⚠️ Anti-Patterns Cần Tránh

### ❌ Chỉ kể thành công

```
❌ "Tôi luôn làm đúng và dự án thành công"
✅ "Lúc đầu tôi chọn approach A, kết quả không tốt vì X,
   sau đó tôi pivot sang B và học được Y"
```

### ❌ Không có specific details

```
❌ "Tôi fix một performance issue lớn"
✅ "Tôi fix N+1 problem trong JPA, giảm latency từ 3s xuống 85ms,
   số SQL queries giảm từ 201 xuống 3"
```

### ❌ Blame team/management

```
❌ "Deadline quá ngắn nên code không tốt, đó là lỗi manager"
✅ "Với deadline ngắn, tôi ưu tiên X và Y, chấp nhận technical debt Z,
   và sau đó lên plan để trả nợ kỹ thuật trong sprint tiếp theo"
```

### ❌ Câu chuyện quá dài

```
❌ 10 phút kể chi tiết từng dòng code
✅ 2–3 phút, high-level + highlight 1–2 technical decisions quan trọng
```

---

## 📋 5 Câu Chuyện STAR Cần Chuẩn Bị

Chuẩn bị sẵn 5 câu chuyện này trước khi phỏng vấn:

| # | Loại | Câu Hỏi Thường Gặp |
|---|------|--------------------|
| 1 | **Technical challenge** — vấn đề kỹ thuật khó nhất | "Tell me about a challenging technical problem" |
| 2 | **Failure / mistake** — sai lầm và bài học | "Tell me about a time you failed" |
| 3 | **Leadership without authority** — dẫn dắt không có quyền | "Tell me about a time you influenced others" |
| 4 | **Conflict** — mâu thuẫn kỹ thuật với đồng nghiệp | "Tell me about a disagreement with a teammate" |
| 5 | **Initiative** — tự chủ cải thiện | "Tell me about a time you went above and beyond" |

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
