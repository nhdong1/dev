# STAR Stories — Template Câu Chuyện Phỏng Vấn

> Hướng dẫn chuẩn bị câu chuyện behavioral interview theo mô hình STAR (Situation — Tình Huống, Task — Nhiệm Vụ, Action — Hành Động, Result — Kết Quả) cho Backend Node.js developer.

## Mục Lục

1. [STAR Framework](#star-framework)
2. [Các Loại Câu Hỏi Behavioral](#các-loại-câu-hỏi-behavioral)
3. [Template Stories](#template-stories)
4. [Câu Chuyện Mẫu Theo Chủ Đề](#câu-chuyện-mẫu-theo-chủ-đề)
5. [Cách Viết STAR Story Của Bạn](#cách-viết-star-story-của-bạn)
6. [Câu Hỏi Ngược Cho Interviewer](#câu-hỏi-ngược-cho-interviewer)

---

## STAR Framework

```
┌─────────────────────────────────────────────────────────────┐
│  S — SITUATION (15%)                                        │
│  Bối cảnh: team, project, timeline, constraints             │
├─────────────────────────────────────────────────────────────┤
│  T — TASK (15%)                                             │
│  Nhiệm vụ cụ thể của BẠN, không phải team                 │
├─────────────────────────────────────────────────────────────┤
│  A — ACTION (50%)                                           │
│  Chi tiết kỹ thuật bạn đã làm — đây là phần quan trọng    │
├─────────────────────────────────────────────────────────────┤
│  R — RESULT (20%)                                           │
│  Kết quả đo lường được + bài học                            │
└─────────────────────────────────────────────────────────────┘
```

**Thời gian trả lời:** 2–3 phút mỗi story. Practice để không quá dài.

**Nguyên tắc:** Dùng "I" (tôi), không "we" (chúng tôi) — interviewer muốn biết **đóng góp cá nhân** của bạn.

---

## Các Loại Câu Hỏi Behavioral

| Loại | Câu Hỏi Mẫu | Story Cần Chuẩn Bị |
| ---- | ----------- | ------------------ |
| **Technical challenge** | "Kể về bug khó nhất bạn từng fix" | Production incident |
| **Leadership** | "Kể khi bạn lead một initiative" | Tech migration, process improvement |
| **Conflict** | "Kể khi bạn disagree với đồng nghiệp" | Technical disagreement |
| **Failure** | "Kể về lần bạn fail" | Deployment mistake, missed deadline |
| **Achievement** | "Kể về thành tựu tự hào nhất" | Performance optimization, feature launch |
| **Pressure** | "Kể khi bạn làm việc dưới áp lực deadline" | Hotfix production |
| **Learning** | "Kể khi bạn phải học công nghệ mới nhanh" | New framework adoption |
| **Mentoring** | "Kể khi bạn giúp đồng nghiệp grow" | Code review, pair programming |

**Chuẩn bị tối thiểu:** 5 stories cover được nhiều loại câu hỏi (dùng lại story với góc nhìn khác).

---

## Template Stories

### Template 1: Production Incident

```markdown
## [Tên Story Ngắn — e.g., "API Latency Spike"]

**S — Situation:**
Tại [công ty], hệ thống [mô tả] phục vụ [số users/requests].
Vào [thời điểm], [monitoring tool] alert [metric] tăng từ [X] lên [Y].

**T — Task:**
Tôi là [role] responsible cho [service/component].
Nhiệm vụ: identify root cause và restore service trong [timeframe].

**A — Action:**
1. Tôi check [logs/metrics/traces] và phát hiện [symptom]
2. Tôi reproduce issue locally bằng [method]
3. Root cause: [technical explanation — e.g., N+1 query sau deploy mới]
4. Short-term fix: [action — e.g., rollback + add missing index]
5. Long-term fix: [action — e.g., add eager loading + integration test]

**R — Result:**
- Latency giảm từ [Y]ms về [X]ms trong [time]
- Thêm [monitoring/alert/test] để prevent recurrence
- Bài học: [what you learned]
```

---

### Template 2: Performance Optimization

```markdown
## [Tên Story — e.g., "Redis Caching Layer"]

**S — Situation:**
[Service] xử lý [X] requests/day, response time trung bình [Y]ms.
Product team report user complaints về [slow page/feature].
Database CPU ở [Z]% trong giờ cao điểm.

**T — Task:**
Tôi được giao optimize [specific endpoint/feature] để đạt [target latency].

**A — Action:**
1. Profile với [tool — clinic.js, EXPLAIN ANALYZE]
2. Phát hiện: [bottleneck — e.g., 3 queries per request, no index]
3. Implement: [solution — cache-aside Redis, TTL 5min, cache invalidation on write]
4. Load test với [k6/Artillery] trước khi deploy
5. Deploy với feature flag, monitor metrics

**R — Result:**
- P95 latency: [Y]ms → [X]ms (giảm [Z]%)
- Database load giảm [N]%
- Cache hit rate [M]% sau 1 tuần
```

---

### Template 3: Technical Disagreement

```markdown
## [Tên Story — e.g., "Monolith vs Microservices"]

**S — Situation:**
Team plan [initiative]. Đồng nghiệp [role] đề xuất [approach A].
Tôi believe [approach B] phù hợp hơn vì [reason].

**T — Task:**
Đưa ra quyết định kỹ thuật đúng cho project, maintain team harmony.

**A — Action:**
1. Tôi document pros/cons của cả 2 approaches
2. Tạo POC (Proof of Concept) nhỏ cho approach B — [timeframe]
3. Present data: [metrics — complexity, time to market, team capacity]
4. Team discussion, address concerns của đồng nghiệp
5. Compromise: [final decision — e.g., modular monolith trước, extract sau]

**R — Result:**
- Team aligned trên [decision]
- Delivered [feature] on time
- Relationship với đồng nghiệp vẫn tốt
- Bài học: data-driven discussion > opinion
```

---

### Template 4: Learning New Technology

```markdown
## [Tên Story — e.g., "Adopting NestJS"]

**S — Situation:**
Team quyết định migrate từ [old stack] sang [new stack] cho [reason].
Tôi chưa có kinh nghiệm với [technology].

**T — Task:**
Trở thành productive với [technology] trong [timeframe] và contribute vào migration.

**A — Action:**
1. Học fundamentals: [resources — docs, course, 1 tuần]
2. Build side project nhỏ để practice [specific features]
3. Pair programming với senior đã biết [technology]
4. Contribute first PR: [specific module/feature]
5. Document learnings cho team wiki

**R — Result:**
- Shipped [feature] với [technology] trong [timeframe]
- Team wiki có [N] articles từ learnings của tôi
- Giờ mentor junior members về [technology]
```

---

## Câu Chuyện Mẫu Theo Chủ Đề

### Story A: Memory Leak Production (Technical Challenge)

**S:** SaaS platform Node.js, 50k daily active users. Sau 2 tuần deploy feature mới, server restart mỗi 6 giờ do OOM (Out Of Memory).

**T:** Tôi là backend lead, cần tìm và fix memory leak trước khi ảnh hưởng users.

**A:**
- Phân tích heap snapshot với Chrome DevTools — phát hiện `EventEmitter` listeners không được remove
- Root cause: WebSocket connection handler add listener mỗi reconnect nhưng không cleanup on disconnect
- Fix: `socket.removeAllListeners()` trong disconnect handler + weak reference pattern
- Thêm memory monitoring alert (>80% heap) vào Prometheus

**R:** Server stable 30+ ngày không restart. Thêm lint rule detect listener leaks. Team adopt pattern cleanup trong code review checklist.

---

### Story B: JWT Migration (Technical Leadership)

**S:** Legacy session-based auth, cần migrate sang JWT cho mobile app launch trong 6 tuần.

**T:** Tôi design và lead implementation auth migration.

**A:**
- Design dual-auth period: support cả session và JWT simultaneously
- Implement access/refresh token flow với rotation
- Write migration guide và training session cho frontend team
- Comprehensive test suite: unit + integration + security tests
- Phased rollout: internal → beta users → full launch

**R:** Migration hoàn thành đúng deadline, zero downtime. Mobile app launch on time. Auth-related incidents giảm 40% so với session-based (do better token expiration handling).

---

### Story C: Failed Deployment (Failure/Learning)

**S:** Deploy payment feature Friday afternoon (vi phạm no-Friday-deploy policy).

**T:** Tôi là developer deploy, cần fix payment processing down.

**A:**
- Payment API return 500 — tôi check logs, phát hiện missing env var `STRIPE_WEBHOOK_SECRET` trên production
- Immediate rollback trong 10 phút
- Root cause: env var có trên staging nhưng chưa add vào production ConfigMap
- Fix: add env var, update deployment checklist, add startup validation

**R:** Downtime 10 phút, không mất transaction (idempotency keys). Team adopt mandatory env var validation at startup. Tôi không deploy Friday nữa.

---

## Cách Viết STAR Story Của Bạn

### Bước 1: Brainstorm (30 phút)

Liệt kê 10–15 experiences từ career:
- Bugs khó fix nhất
- Features tự hào nhất
- Lần conflict với teammate
- Lần miss deadline
- Lần học tech mới nhanh
- Lần giúp junior grow

### Bước 2: Chọn Top 5 (cover nhiều loại câu hỏi)

| # | Story | Covers |
| - | ----- | ------ |
| 1 | Production incident | Technical, pressure |
| 2 | Performance optimization | Technical, achievement |
| 3 | Team conflict | Conflict, leadership |
| 4 | Failed deployment | Failure, learning |
| 5 | New tech adoption | Learning, leadership |

### Bước 3: Viết Chi Tiết (mỗi story 1 trang)

- Số liệu cụ thể: latency ms, %, users, timeline
- Technical terms chính xác
- "I" statements
- Kết quả đo lường được

### Bước 4: Practice Aloud (3–5 lần mỗi story)

- Record audio, nghe lại
- Timing: 2–3 phút
- Mock interview với bạn bè

### Checklist Mỗi Story

- [ ] Có số liệu cụ thể (metrics, timeline)
- [ ] Action chiếm 50% thời gian
- [ ] Dùng "I" không phải "we"
- [ ] Có bài học (lesson learned)
- [ ] Trả lời được trong 2–3 phút
- [ ] Technical depth phù hợp level phỏng vấn

---

## Câu Hỏi Ngược Cho Interviewer

Chuẩn bị 3–5 câu hỏi thể hiện genuine interest:

### Về Team & Culture

- "Team hiện tại structure thế nào? Backend team bao nhiêu người?"
- "Code review process của team ra sao?"
- "Team handle on-call và production incidents thế nào?"

### Về Technical

- "Tech stack hiện tại và roadmap 6–12 tháng tới?"
- "Biggest technical challenge team đang face?"
- "Testing strategy — unit vs integration ratio?"
- "Deployment frequency và CI/CD pipeline?"

### Về Growth

- "Opportunities cho mentorship hoặc tech lead path?"
- "Conference/training budget cho engineers?"
- "Làm sao đo success cho role này trong 6 tháng đầu?"

### Tránh Hỏi

- Salary/benefits ở vòng technical (để HR round)
- "Công ty làm gì?" — nên research trước
- Câu hỏi có answer trên website

---

## Tài Liệu Tham Khảo

- [README.md](./README.md) — Tổng quan chuẩn bị phỏng vấn
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Câu hỏi kỹ thuật
- [7-90-day-study-plan.md](./7-90-day-study-plan.md) — Kế hoạch học dài hạn
