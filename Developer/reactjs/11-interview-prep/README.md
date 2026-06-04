# 🎯 Interview Prep — Chuẩn Bị Phỏng Vấn React.js

> Bộ tài liệu toàn diện giúp bạn tự tin vượt qua mọi vòng phỏng vấn React — từ Junior đến Senior Frontend Engineer.

---

## 📋 Nội Dung Trong Thư Mục Này

| File | Mô Tả | Cấp Độ |
|------|--------|--------|
| `INTERVIEW_GUIDE.md` | Top 30 câu hỏi phỏng vấn + đáp án chi tiết | Mọi cấp độ |
| `1-common-questions.md` | Câu hỏi phổ biến nhất — JSX, Components, Props, State | Junior |
| `2-hooks-deep-dive.md` | Câu hỏi chuyên sâu về Hooks — closures, deps, custom hooks | Mid–Senior |
| `3-performance-questions.md` | Câu hỏi về tối ưu hiệu năng — memoization, profiling | Senior |
| `4-system-design.md` | Thiết kế hệ thống Frontend — architecture, scalability | Senior |
| `5-react19-questions.md` | Câu hỏi về React v19 — Server Actions, Compiler, new APIs | Senior |
| `6-90-day-study-plan.md` | Kế hoạch học có lộ trình rõ ràng trong 90 ngày | Mọi cấp độ |

---

## 🗓️ Chiến Lược Ôn Thi Theo Thời Gian

### Còn 1 Tuần — Sprint Ngắn

```
Ngày 1-2: INTERVIEW_GUIDE.md — đọc và ghi nhớ 30 câu hỏi cốt lõi
Ngày 3:   1-common-questions.md — ôn lại nền tảng
Ngày 4:   2-hooks-deep-dive.md — tập trung vào hooks (hay hỏi nhất)
Ngày 5:   3-performance-questions.md — câu hỏi senior
Ngày 6:   5-react19-questions.md — điểm cộng khi biết React v19
Ngày 7:   Mock interview với đồng nghiệp / tự nói to các câu trả lời
```

### Còn 2 Tuần — Chuẩn Bị Đầy Đủ

```
Tuần 1: Đọc toàn bộ 6 file + thực hành code
Tuần 2: Ôn lại điểm yếu + mock interviews + system design
```

### Còn 1 Tháng — Học Bài Bản

```
Tuần 1: Ôn fundamentals + hooks
Tuần 2: State management + data fetching
Tuần 3: Performance + React v19 + system design
Tuần 4: Mock interviews + live coding practice
```

---

## 🏆 Các Dạng Câu Hỏi Phỏng Vấn React

### 1. Câu Hỏi Khái Niệm — Conceptual Questions

> Interviewer kiểm tra bạn **hiểu** React hoạt động như thế nào.

**Ví dụ:**
- "Virtual DOM — DOM Ảo là gì? Khác gì Real DOM — DOM Thật?"
- "Reconciliation — Thuật toán đối chiếu hoạt động như thế nào?"
- "Tại sao cần key trong danh sách?"

**Cách trả lời tốt:** Giải thích ngắn gọn → đưa ví dụ cụ thể → nêu trade-offs nếu có.

---

### 2. Câu Hỏi Code — Coding Questions

> Interviewer muốn thấy bạn **viết** React thực tế.

**Ví dụ:**
- "Implement một custom hook `useFetch` để fetch data"
- "Viết một component Counter với debounce"
- "Fix bug này: tại sao state không update?"

**Cách trả lời tốt:** Suy nghĩ to → viết code từng bước → test edge cases.

---

### 3. Câu Hỏi Tình Huống — Scenario Questions

> Interviewer kiểm tra **kinh nghiệm thực tế** và khả năng giải quyết vấn đề.

**Ví dụ:**
- "Ứng dụng của bạn render chậm, bạn debug và tối ưu như thế nào?"
- "Khi nào bạn chọn Redux thay vì Context API?"
- "Bạn sẽ architect state management cho một e-commerce app như thế nào?"

**Cách trả lời tốt:** Dùng STAR method (Situation — Tình huống, Task — Nhiệm vụ, Action — Hành động, Result — Kết quả).

---

### 4. Câu Hỏi System Design — Thiết Kế Hệ Thống

> Dành cho Senior — kiểm tra tư duy **kiến trúc** tổng thể.

**Ví dụ:**
- "Thiết kế kiến trúc frontend cho một social media platform"
- "Bạn sẽ implement real-time notifications như thế nào?"
- "Chiến lược code splitting cho một SPA — Single Page Application lớn"

**Cách trả lời tốt:** Hỏi clarifying questions → vẽ high-level diagram → đào sâu từng phần → nêu trade-offs.

---

## 📊 Trọng Số Câu Hỏi Theo Cấp Độ

### Junior React Developer (0-2 năm)

| Chủ Đề | Tần Suất Xuất Hiện |
|--------|-------------------|
| JSX, Components, Props | ████████ 80% |
| useState, useEffect | ████████ 80% |
| Event handling, Forms | ██████ 60% |
| Conditional rendering, Lists | ██████ 60% |
| Context API cơ bản | ████ 40% |
| React Router cơ bản | ████ 40% |

### Mid-Level React Developer (2-4 năm)

| Chủ Đề | Tần Suất Xuất Hiện |
|--------|-------------------|
| Custom Hooks | ████████ 80% |
| useMemo, useCallback | ████████ 80% |
| State management (Redux/Zustand) | ██████ 70% |
| TanStack Query / SWR | ██████ 60% |
| Performance optimization | ██████ 60% |
| Testing với RTL | ████ 50% |
| Code splitting, lazy loading | ████ 50% |

### Senior React Developer (4+ năm)

| Chủ Đề | Tần Suất Xuất Hiện |
|--------|-------------------|
| React Server Components | ████████ 80% |
| System design Frontend | ████████ 80% |
| React v19 features | ██████ 70% |
| Performance profiling | ██████ 70% |
| Architecture patterns | ██████ 60% |
| Micro-frontends | ████ 50% |
| Accessibility (WCAG 2.1) | ████ 40% |

---

## 🧠 Framework Trả Lời Câu Hỏi Kỹ Thuật

### Công Thức C-E-T (Concept → Example → Trade-off)

```
1. CONCEPT — Khái niệm:
   "X là...[định nghĩa ngắn gọn]"

2. EXAMPLE — Ví dụ:
   "Ví dụ thực tế: [code snippet hoặc use case]"

3. TRADE-OFF — Đánh đổi:
   "Lưu ý/Khi không nên dùng: [edge cases, alternatives]"
```

**Ví dụ áp dụng — câu hỏi "useMemo là gì?":**

```
1. CONCEPT:
   useMemo là hook ghi nhớ (memoize) kết quả tính toán tốn kém,
   chỉ tính lại khi dependencies thay đổi.

2. EXAMPLE:
   const sortedList = useMemo(
     () => items.sort((a, b) => a.price - b.price),
     [items]
   );
   // Chỉ sort lại khi `items` thay đổi, không sort mỗi render.

3. TRADE-OFF:
   Không lạm dụng — bản thân useMemo có chi phí (overhead).
   Chỉ dùng khi tính toán thực sự nặng hoặc kết quả truyền
   xuống component con dưới React.memo.
```

---

## ⚡ Những Lỗi Hay Gặp Trong Phỏng Vấn

### Lỗi Về Kiến Thức

| Hiểu Sai | Đúng |
|-----------|------|
| "useEffect chạy sau mỗi render" | useEffect chạy sau mỗi render **có dependencies thay đổi** |
| "useMemo luôn tốt hơn không dùng" | useMemo có overhead, chỉ dùng khi cần thiết |
| "Context API thay thế Redux" | Context dành cho low-frequency updates; Redux cho complex state |
| "key chỉ cần unique trong cả app" | key chỉ cần unique **trong cùng một danh sách** |
| "React.memo ngăn mọi re-render" | React.memo chỉ ngăn re-render do **parent re-render**, không ngăn re-render do state/context nội bộ |

### Lỗi Về Cách Trả Lời

- ❌ Trả lời quá ngắn: "useMemo là để tối ưu" → không đủ
- ❌ Không đưa ví dụ cụ thể
- ❌ Không hỏi clarifying questions trước khi thiết kế
- ❌ Tự ti khi không biết câu trả lời — hãy nói thẳng và đề xuất hướng tìm hiểu
- ✅ Nói to quá trình suy nghĩ (think aloud)
- ✅ Đặt câu hỏi ngược để hiểu context
- ✅ Nêu trade-offs thay vì câu trả lời một chiều

---

## 🔑 Chủ Đề "Must Know" Trước Mọi Phỏng Vấn React

### Core — Bắt Buộc Mọi Level

- [ ] Virtual DOM — DOM Ảo và Reconciliation — Thuật Toán Đối Chiếu
- [ ] Component lifecycle qua Hooks (mount, update, unmount)
- [ ] useState — stale state, functional updater
- [ ] useEffect — dependency array, cleanup function
- [ ] Closures trong Hooks — Vấn đề stale closure
- [ ] Props vs State — khi nào dùng gì
- [ ] Controlled vs Uncontrolled components

### Mid+ — Trung Cấp Trở Lên

- [ ] useMemo vs useCallback — phân biệt rõ ràng
- [ ] React.memo — shallow comparison
- [ ] Custom Hooks — trích xuất logic
- [ ] Context API — re-render behavior
- [ ] Code splitting — React.lazy + Suspense
- [ ] Error Boundaries — Ranh Giới Lỗi

### Senior — Phải Biết

- [ ] React Server Components — RSC — Component Phía Server
- [ ] Server Actions — Hành Động Phía Server
- [ ] React Compiler — Trình Biên Dịch React tự động optimize
- [ ] Concurrent Features — Tính Năng Đồng Thời (startTransition, useDeferredValue)
- [ ] Micro-frontends — Kiến trúc vi giao diện
- [ ] Performance profiling workflow

---

## 📝 Template Chuẩn Bị Cho Mỗi Chủ Đề

Với mỗi topic, hãy chuẩn bị đủ 4 phần:

```markdown
## [Tên Topic]

### 1. Định Nghĩa (30 giây)
[Giải thích ngắn gọn, không dùng tài liệu]

### 2. Ví Dụ Code (2-3 phút)
[Code snippet ngắn, dễ viết trên whiteboard]

### 3. Khi Nào Dùng / Không Dùng
[Use cases + anti-patterns]

### 4. Liên Kết Với Chủ Đề Khác
[Context, alternatives, trade-offs]
```

---

## 🚀 Bắt Đầu Từ Đâu?

```
Bước 1: Đọc INTERVIEW_GUIDE.md để có bức tranh toàn cảnh
Bước 2: Ôn từng file theo thứ tự 1 → 2 → 3 → 4 → 5
Bước 3: Thực hành live coding ít nhất 30 phút mỗi ngày
Bước 4: Xem 6-90-day-study-plan.md nếu cần lộ trình dài hạn
```

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
