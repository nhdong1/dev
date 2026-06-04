# React.js & React Ecosystem — Chỉ Mục Toàn Diện

> Bản đồ điều hướng toàn bộ kiến thức React.js, React v19, và hệ sinh thái xung quanh

## 📁 Cấu Trúc Thư Mục

```
reactjs/
├── README.md                               [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/                        Nền tảng React
│   ├── README.md                           Tổng quan chủ đề nền tảng
│   ├── 1-jsx-components.md                 JSX, Functional Components, cú pháp cơ bản
│   ├── 2-props-state.md                    Props, State, dữ liệu một chiều
│   ├── 3-event-handling.md                 Xử lý sự kiện, Synthetic Events
│   ├── 4-conditional-list-rendering.md     Render có điều kiện, danh sách, key
│   └── 5-forms-controlled.md               Forms, Controlled vs Uncontrolled components
│
├── 02-hooks/                               React Hooks (built-in & custom)
│   ├── README.md                           Tổng quan về Hooks
│   ├── 1-usestate-usereducer.md            useState, useReducer — quản lý state
│   ├── 2-useeffect-uselayout.md            useEffect, useLayoutEffect — side effects
│   ├── 3-usecontext-useref.md              useContext, useRef — context và DOM refs
│   ├── 4-usememo-usecallback.md            useMemo, useCallback — memoization
│   ├── 5-advanced-hooks.md                 useId, useTransition, useDeferredValue, useDebugValue
│   ├── 6-react19-new-hooks.md              use(), useActionState(), useFormStatus(), useOptimistic()
│   └── 7-custom-hooks.md                   Custom Hooks — tạo và tái sử dụng logic
│
├── 03-state-management/                    Quản lý trạng thái toàn cục
│   ├── README.md                           So sánh các giải pháp state management
│   ├── 1-context-api.md                    Context API — giải pháp tích hợp sẵn
│   ├── 2-redux-toolkit.md                  Redux Toolkit (RTK) — chuẩn công nghiệp
│   ├── 3-rtk-query.md                      RTK Query — data fetching & caching
│   ├── 4-zustand.md                        Zustand — nhẹ, đơn giản, hiện đại
│   ├── 5-jotai.md                          Jotai — atomic state management
│   └── 6-comparison-guide.md               Khi nào dùng gì — hướng dẫn lựa chọn
│
├── 04-routing/                             Điều hướng trong React
│   ├── README.md                           Tổng quan routing strategies
│   ├── 1-react-router-basics.md            React Router v6 — cơ bản
│   ├── 2-nested-dynamic-routes.md          Nested routes, Dynamic params, Outlet
│   ├── 3-protected-routes.md               Protected Routes, Auth Guards
│   ├── 4-react-router-v7.md                React Router v7 — tính năng mới
│   └── 5-tanstack-router.md                TanStack Router — type-safe routing
│
├── 05-data-fetching/                       Lấy và đồng bộ dữ liệu
│   ├── README.md                           Chiến lược data fetching
│   ├── 1-tanstack-query-basics.md          TanStack Query — useQuery, useMutation
│   ├── 2-tanstack-query-advanced.md        Caching, Invalidation, Optimistic Updates
│   ├── 3-swr.md                            SWR — Stale-While-Revalidate
│   ├── 4-react-server-components.md        RSC — React Server Components
│   └── 5-server-actions.md                 Server Actions (React v19 & Next.js)
│
├── 06-performance/                         Tối ưu hiệu năng
│   ├── README.md                           Phương pháp tối ưu hiệu năng
│   ├── 1-memoization.md                    React.memo, useMemo, useCallback
│   ├── 2-code-splitting.md                 Code Splitting, React.lazy, dynamic import
│   ├── 3-virtualization.md                 Virtualize danh sách dài (react-window)
│   ├── 4-concurrent-features.md            Concurrent Mode, startTransition, Suspense
│   ├── 5-profiling.md                      React DevTools Profiler, bottleneck detection
│   └── 6-bundle-optimization.md            Bundle size, tree-shaking, lazy loading
│
├── 07-testing/                             Kiểm thử
│   ├── README.md                           Chiến lược kiểm thử React
│   ├── 1-jest-basics.md                    Jest — unit testing framework
│   ├── 2-react-testing-library.md          RTL — kiểm thử theo góc nhìn người dùng
│   ├── 3-mocking.md                        Mock API, modules, hooks
│   ├── 4-integration-testing.md            Integration tests với MSW (Mock Service Worker)
│   ├── 5-vitest.md                         Vitest — thay thế Jest cho Vite projects
│   └── 6-e2e-testing.md                    E2E với Playwright và Cypress
│
├── 08-styling/                             Tạo kiểu dáng UI
│   ├── README.md                           So sánh các giải pháp styling
│   ├── 1-css-modules.md                    CSS Modules — scope CSS theo component
│   ├── 2-tailwind-css.md                   Tailwind CSS — utility-first
│   ├── 3-styled-components.md              Styled Components — CSS-in-JS
│   ├── 4-emotion.md                        Emotion — CSS-in-JS hiệu năng cao
│   └── 5-design-systems.md                 shadcn/ui, Radix UI, Material UI
│
├── 09-ecosystem/                           Hệ sinh thái & công cụ
│   ├── README.md                           Tổng quan hệ sinh thái React
│   ├── 1-nextjs-app-router.md              Next.js App Router — Server/Client Components
│   ├── 2-nextjs-pages-router.md            Next.js Pages Router — SSR, SSG, ISR
│   ├── 3-remix.md                          Remix — web standards framework
│   ├── 4-vite-tooling.md                   Vite, ESBuild, Turbopack
│   ├── 5-typescript-react.md               TypeScript với React — typing components, hooks
│   └── 6-storybook-monorepo.md             Storybook, Turborepo, Nx
│
├── 10-advanced-patterns/                   Mẫu thiết kế nâng cao
│   ├── README.md                           Tổng quan Advanced Patterns
│   ├── 1-compound-components.md            Compound Components — API linh hoạt
│   ├── 2-render-props.md                   Render Props — chia sẻ logic
│   ├── 3-hoc.md                            Higher-Order Components (HOC)
│   ├── 4-custom-hooks-patterns.md          Custom Hooks nâng cao
│   ├── 5-portals-boundaries.md             Portals và Error Boundaries
│   └── 6-accessibility.md                  A11y — ARIA, keyboard nav, screen readers
│
└── 11-interview-prep/                      Chuẩn bị phỏng vấn
    ├── README.md                           Tổng quan ôn thi
    ├── INTERVIEW_GUIDE.md                  Top 30 câu hỏi + đáp án chi tiết
    ├── 1-common-questions.md               Câu hỏi phổ biến mọi cấp độ
    ├── 2-hooks-deep-dive.md                Câu hỏi sâu về Hooks
    ├── 3-performance-questions.md          Câu hỏi về hiệu năng (Senior level)
    ├── 4-system-design.md                  Thiết kế hệ thống Frontend
    ├── 5-react19-questions.md              Câu hỏi về React v19
    └── 6-90-day-study-plan.md              Kế hoạch học 90 ngày
```

---

## ✅ Trạng Thái Nội Dung

| Chủ Đề                             | Thư Mục / File                                | Trạng Thái | Mức Độ      |
| ---------------------------------- | --------------------------------------------- | ---------- | ----------- |
| **Tổng Quan & Lộ Trình**           | README.md                                     | ✅         | Toàn diện   |
| **Chỉ Mục**                        | INDEX.md                                      | ✅         | Toàn diện   |
| **Nền Tảng React**                 | 01-fundamentals/                              | ✅         | Hoàn thành  |
| **React Hooks**                    | 02-hooks/                                     | ✅         | Hoàn thành  |
| **State Management**               | 03-state-management/                          | ✅         | Hoàn thành  |
| **Routing**                        | 04-routing/                                   | ✅         | Hoàn thành  |
| **Data Fetching**                  | 05-data-fetching/                             | ✅         | Hoàn thành  |
| **Performance**                    | 06-performance/                               | ✅         | Hoàn thành  |
| **Testing**                        | 07-testing/                                   | ✅         | Hoàn thành  |
| **Styling**                        | 08-styling/                                   | ✅         | Hoàn thành  |
| **Ecosystem**                      | 09-ecosystem/                                 | ✅         | Hoàn thành  |
| **Advanced Patterns**              | 10-advanced-patterns/                         | ✅         | Hoàn thành  |
| **Interview Prep**                 | 11-interview-prep/                            | ✅         | Hoàn thành  |

---

## 🎯 Thứ Tự Tạo Nội Dung (Ưu Tiên)

### Ưu Tiên Cao — Core React Skills

- [x] `01-fundamentals/` — JSX, Components, Props, State, Events ✅ **Hoàn thành**
- [x] `02-hooks/` — Tất cả built-in hooks + React v19 hooks mới ✅ **Hoàn thành**
- [x] `03-state-management/` — Redux Toolkit, Zustand, Context API ✅ **Hoàn thành**
- [x] `05-data-fetching/` — TanStack Query, RSC, Server Actions ✅ **Hoàn thành**
- [x] `11-interview-prep/INTERVIEW_GUIDE.md` — Bộ câu hỏi phỏng vấn ✅ **Hoàn thành**

### Ưu Tiên Trung Bình — Ecosystem & Quality

- [x] `06-performance/` — Memoization, Code splitting, Concurrent ✅ **Hoàn thành**
- [x] `04-routing/` — React Router v6/v7, TanStack Router ✅ **Hoàn thành**
- [x] `09-ecosystem/` — Next.js App Router, TypeScript ✅ **Hoàn thành**
- [x] `07-testing/` — Jest, RTL, MSW, Vitest, Playwright, Cypress ✅ **Hoàn thành**

### Ưu Tiên Thấp — Reference & Advanced

- [x] `08-styling/` — CSS Modules, Tailwind, CSS-in-JS ✅ **Hoàn thành**
- [x] `10-advanced-patterns/` — Compound, HOC, Portals ✅ **Hoàn thành**
- [x] `11-interview-prep/` — Toàn bộ study plans ✅ **Hoàn thành**

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Dành Cho Tự Học

```
1. Bắt đầu với README.md
2. Chọn Learning Path phù hợp mức độ (Beginner/Intermediate/Advanced)
3. Học từng section theo thứ tự
4. Thực hành với mini-projects sau mỗi section
5. Build portfolio project cuối cùng
```

### Dành Cho Ôn Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào 02-hooks/ (thường bị hỏi nhiều nhất)
3. Ôn 03-state-management/ — Redux vs Zustand, Context API
4. Học React v19 features (differentiator cho senior roles)
5. Chuẩn bị system design với 11-interview-prep/4-system-design.md
6. Mock interview với đồng nghiệp
```

### Dành Cho Dự Án Thực Tế

```
Dùng làm reference:
- Chọn state management: 03-state-management/6-comparison-guide.md
- Setup testing: 07-testing/README.md
- Tối ưu hiệu năng: 06-performance/README.md
- Next.js patterns: 09-ecosystem/1-nextjs-app-router.md
- Accessibility: 10-advanced-patterns/6-accessibility.md
```

### Dành Cho System Design

```
1. Đọc 09-ecosystem/README.md để so sánh frameworks
2. Xem 03-state-management/6-comparison-guide.md cho kiến trúc state
3. Áp dụng 10-advanced-patterns/ cho component architecture
4. Tham khảo 06-performance/ cho scalability
```

---

## 📊 Thời Gian Học Ước Tính

| Chủ Đề                  | Thời Gian    | Độ Khó     | Ưu Tiên     |
| ----------------------- | ------------ | ---------- | ----------- |
| Fundamentals            | 10-15 giờ   | ⭐          | Bắt buộc   |
| Hooks                   | 15-20 giờ   | ⭐⭐        | Bắt buộc   |
| State Management        | 10-15 giờ   | ⭐⭐        | Bắt buộc   |
| Routing                 | 6-8 giờ     | ⭐⭐        | Bắt buộc   |
| Data Fetching           | 10-12 giờ   | ⭐⭐⭐      | Bắt buộc   |
| Performance             | 12-15 giờ   | ⭐⭐⭐      | Nên có      |
| Testing                 | 10-12 giờ   | ⭐⭐        | Nên có      |
| Ecosystem (Next.js)     | 15-20 giờ   | ⭐⭐⭐      | Nên có      |
| Styling                 | 6-8 giờ     | ⭐          | Nên có      |
| Advanced Patterns       | 10-12 giờ   | ⭐⭐⭐      | Tốt nếu có  |
| React v19 Features      | 5-8 giờ     | ⭐⭐        | Tốt nếu có  |

**Tổng cộng: 110-145 giờ để nắm vững React ecosystem toàn diện**

---

## 🎓 Mức Độ Kỹ Năng Được Hỗ Trợ

### Beginner — Mới Bắt Đầu (0-6 tháng)

- [ ] JSX và cú pháp React
- [ ] Functional Components
- [ ] Props và State cơ bản
- [ ] useState và useEffect
- [ ] Conditional rendering và Lists

**Thời gian để đạt mức này:** 1-2 tháng tự học tích cực

### Intermediate — Trung Cấp (6 tháng - 2 năm)

- [ ] Thành thạo tất cả built-in hooks
- [ ] Custom Hooks
- [ ] State management (Redux Toolkit hoặc Zustand)
- [ ] React Router v6 — nested, protected routes
- [ ] TanStack Query — server state
- [ ] Memoization đúng cách
- [ ] Testing với RTL

**Thời gian để đạt mức này:** 3-6 tháng với thực hành thường xuyên

### Advanced — Nâng Cao (2+ năm)

- [ ] React Server Components và Server Actions
- [ ] React v19 APIs mới
- [ ] Next.js App Router toàn diện
- [ ] Performance profiling và tối ưu
- [ ] Advanced component patterns
- [ ] Micro-frontends
- [ ] Accessibility (WCAG 2.1)

**Thời gian để đạt mức này:** Học liên tục, kinh nghiệm thực tế là chủ yếu

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                                | Vị Trí                                                                 |
| -------------------------------------- | ---------------------------------------------------------------------- |
| Tổng quan nhanh                        | [README.md](README.md)                                                 |
| Học JSX & Components                   | [01-fundamentals/](./01-fundamentals/)                                 |
| Hiểu sâu về Hooks                      | [02-hooks/](./02-hooks/)                                               |
| Redux Toolkit                          | [03-state-management/2-redux-toolkit.md](./03-state-management/)       |
| Zustand                                | [03-state-management/4-zustand.md](./03-state-management/)             |
| TanStack Query                         | [05-data-fetching/1-tanstack-query-basics.md](./05-data-fetching/)     |
| Next.js App Router                     | [09-ecosystem/1-nextjs-app-router.md](./09-ecosystem/)                 |
| React v19 Hooks mới                    | [02-hooks/6-react19-new-hooks.md](./02-hooks/)                         |
| Tối ưu hiệu năng                       | [06-performance/](./06-performance/)                                   |
| Câu hỏi phỏng vấn                      | [11-interview-prep/INTERVIEW_GUIDE.md](./11-interview-prep/)           |
| Kế hoạch học 90 ngày                   | [11-interview-prep/6-90-day-study-plan.md](./11-interview-prep/)       |

---

## 📈 Theo Dõi Tiến Độ Học

Sao chép và tự điền vào đây:

```markdown
## Tiến Độ React.js của [Tên bạn]

### Phase 1: Nền Tảng (Tuần 1-2)

- [ ] JSX và Components
- [ ] Props và State
- [ ] Event handling
- [ ] Conditional rendering và Lists
- [ ] Forms và Controlled Components

### Phase 2: Hooks (Tuần 3-4)

- [ ] useState và useReducer
- [ ] useEffect — cleanup và dependencies
- [ ] useContext và useRef
- [ ] useMemo và useCallback (khi nào cần)
- [ ] Custom Hooks đầu tiên
- [ ] React v19 hooks mới (use, useActionState)

### Phase 3: State & Data (Tuần 5-7)

- [ ] Redux Toolkit cơ bản
- [ ] RTK Query hoặc TanStack Query
- [ ] React Router v6
- [ ] Protected Routes

### Phase 4: Chất Lượng (Tuần 8-10)

- [ ] Testing với RTL
- [ ] Performance optimization (memo, lazy)
- [ ] Code splitting và Suspense
- [ ] TypeScript cơ bản với React

### Phase 5: Nâng Cao (Tuần 11-13)

- [ ] Next.js App Router
- [ ] React Server Components
- [ ] Server Actions
- [ ] Advanced patterns (Compound, HOC)
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích Virtual DOM và Reconciliation không cần notes
- [ ] Viết và debug custom hooks phức tạp
- [ ] Chọn đúng state management solution cho use case cụ thể
- [ ] Implement data fetching với caching và error handling

### ✅ Năng Lực Vận Hành

- [ ] Tối ưu hiệu năng bằng cách đo lường trước (profile first)
- [ ] Viết tests có ý nghĩa, không phải tests cho có
- [ ] Setup Next.js với App Router từ đầu
- [ ] Migrate code sang React v19 features khi phù hợp

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời 30 câu hỏi React tự tin
- [ ] Design kiến trúc frontend cho hệ thống lớn
- [ ] Giải thích trade-offs giữa các giải pháp
- [ ] Live code: implement features không cần docs

---

## 🚀 Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn learning path phù hợp
3. Setup môi trường phát triển (Vite hoặc Next.js)
4. Bắt đầu `01-fundamentals/`

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành `01-fundamentals/` và `02-hooks/`
2. Xây dựng Todo app áp dụng kiến thức
3. Bắt đầu `03-state-management/`
4. Thực hành với Redux Toolkit hoặc Zustand

### Trung Hạn (1 Tháng Tới)

1. Hoàn thành `03-05` (State, Routing, Data Fetching)
2. Build một CRUD app đầy đủ
3. Viết tests cho app
4. Bắt đầu học Next.js

### Dài Hạn (3 Tháng Tới)

1. Master Next.js App Router + React Server Components
2. Thành thạo React v19 features
3. Build portfolio project production-ready
4. Ôn tập phỏng vấn và mock interviews

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng cách xây dựng:** Đừng chỉ đọc — clone repos, phá vỡ chúng, sửa lại
2. **Đọc source code:** React DevTools và mã nguồn React dạy nhiều hơn bài viết
3. **Hiểu "tại sao":** Mỗi hook và pattern đều có lý do tồn tại — hiểu vấn đề nó giải quyết
4. **Sử dụng TypeScript sớm:** Bắt đầu với TypeScript từ đầu, không thêm sau
5. **Viết tests song song:** Test ngay khi code, không để cuối
6. **Theo dõi React blog:** Team React thông báo tính năng mới thường xuyên
7. **Tham gia cộng đồng:** Reactiflux Discord, React subreddit
8. **Build, ship, iterate:** Hoàn thiện > Hoàn hảo

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn thêm nội dung?

- [ ] Sửa nội dung không chính xác
- [ ] Thêm ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Cập nhật khi React ra phiên bản mới
- [ ] Thêm section chưa được bao phủ
- [ ] Cải thiện giải thích phức tạp

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phiên Bản:** 3.0 (README + INDEX + 01-fundamentals + 02-hooks + 03-state-management + 04-routing + 05-data-fetching + 06-performance + 07-testing + 08-styling + 09-ecosystem + 10-advanced-patterns + 11-interview-prep)
**React Version:** v19 (stable)
**Trạng Thái:** ✅ Toàn bộ knowledge base hoàn thành — bao gồm 11-interview-prep (README, INTERVIEW_GUIDE, 6 chuyên đề, kế hoạch 90 ngày)
