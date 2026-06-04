# ⚛️ React.js & React Ecosystem — Lộ Trình Học Toàn Diện

> Hướng dẫn toàn diện về React.js và hệ sinh thái React, bao gồm React v19, từ nền tảng cơ bản đến kỹ thuật nâng cao, chuẩn bị phỏng vấn Frontend Engineer.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Phase 1: Nền Tảng — Fundamentals (Tuần 1-2)**

- [ ] JSX — JavaScript XML — Cú pháp mở rộng JavaScript
- [ ] Components — Thành Phần — Functional & Class Components
- [ ] Props — Properties — Truyền dữ liệu giữa các component
- [ ] State — Trạng Thái — Quản lý dữ liệu nội bộ
- [ ] Event Handling — Xử Lý Sự Kiện
- [ ] Conditional Rendering — Render có Điều Kiện
- [ ] Lists & Keys — Danh sách và Khóa

### **Phase 2: Hooks & Quản Lý Trạng Thái (Tuần 3-5)**

- [ ] useState — Hook quản lý state cục bộ
- [ ] useEffect — Hook xử lý side effects
- [ ] useContext — Hook sử dụng Context API
- [ ] useReducer — Hook quản lý state phức tạp
- [ ] useRef — Hook tham chiếu DOM và giá trị mutable
- [ ] useMemo & useCallback — Hook tối ưu hiệu năng
- [ ] Custom Hooks — Hook Tùy Chỉnh
- [ ] React v19 New Hooks: `use`, `useActionState`, `useFormStatus`, `useOptimistic`

### **Phase 3: Hệ Sinh Thái & Công Cụ (Tuần 6-9)**

- [ ] State Management — Quản Lý Trạng Thái Toàn Cục (Redux Toolkit, Zustand, Jotai)
- [ ] Routing — Điều Hướng (React Router v6/v7, TanStack Router)
- [ ] Data Fetching — Lấy Dữ Liệu (TanStack Query, SWR)
- [ ] React Server Components — RSC — Component Phía Server
- [ ] Server Actions — Hành Động Phía Server (React v19)
- [ ] Testing — Kiểm Thử (Jest, React Testing Library, Vitest)

### **Phase 4: Hiệu Năng & Nâng Cao (Tuần 10-13)**

- [ ] Performance Optimization — Tối Ưu Hiệu Năng
- [ ] Code Splitting — Tách Code (React.lazy, Suspense)
- [ ] Concurrent Features — Tính Năng Đồng Thời (React v18/v19)
- [ ] Advanced Patterns — Mẫu Nâng Cao (Compound, HOC, Render Props)
- [ ] Accessibility — A11y — Khả Năng Tiếp Cận
- [ ] Next.js & Meta-Frameworks — Framework Tổng Hợp

### **Phase 5: Chuyên Sâu (Tuần 14+)**

- [ ] Next.js App Router — Điều Hướng Dựa Trên Thư Mục
- [ ] TypeScript với React — Kiểu Dữ Liệu Tĩnh
- [ ] Micro-frontends — Vi Giao Diện
- [ ] React Native — Phát Triển Ứng Dụng Di Động
- [ ] Animation — Hoạt Ảnh (Framer Motion, React Spring)

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                              | Độ Ưu Tiên | Thời Gian  | Trạng Thái |
| ------------------------------------- | ---------- | ---------- | ---------- |
| **React Fundamentals**                | ⭐⭐⭐     | 2 tuần     | -          |
| **Hooks (Built-in & Custom)**         | ⭐⭐⭐     | 2 tuần     | -          |
| **State Management**                  | ⭐⭐⭐     | 2 tuần     | -          |
| **Data Fetching & Server State**      | ⭐⭐⭐     | 1.5 tuần   | -          |
| **Performance Optimization**          | ⭐⭐⭐     | 2 tuần     | -          |
| **Testing**                           | ⭐⭐⭐     | 1.5 tuần   | -          |
| **React v19 Features**                | ⭐⭐⭐     | 1 tuần     | -          |
| **Next.js / Meta-Frameworks**         | ⭐⭐⭐     | 2 tuần     | -          |
| **TypeScript với React**              | ⭐⭐       | 1 tuần     | -          |
| **Advanced Patterns**                 | ⭐⭐       | 1.5 tuần   | -          |
| **Styling Solutions**                 | ⭐⭐       | 1 tuần     | -          |
| **Accessibility (A11y)**              | ⭐⭐       | 0.5 tuần   | -          |

---

## 🗂️ Tổng Quan Chủ Đề

### 📁 **1. Fundamentals — Nền Tảng** (`01-fundamentals/`)

- **JSX** — JavaScript XML — Cú pháp kết hợp HTML trong JavaScript
- **Components** — Functional Components, Class Components (legacy)
- **Props & PropTypes** — Truyền và kiểm tra dữ liệu
- **State & Lifecycle** — Vòng đời component và quản lý state
- **Event System** — Hệ thống sự kiện tổng hợp (Synthetic Events)
- **Conditional & List Rendering** — Render có điều kiện và danh sách
- **Forms & Controlled Components** — Form và component được kiểm soát

### 📁 **2. Hooks** (`02-hooks/`)

- **useState** — Quản lý state cục bộ
- **useEffect** — Side effects, cleanup, dependency array
- **useContext** — Tiêu thụ Context không cần prop drilling
- **useReducer** — Quản lý state phức tạp theo pattern Flux
- **useRef** — Tham chiếu DOM, lưu giá trị mutable không trigger re-render
- **useMemo** — Ghi nhớ giá trị tính toán tốn kém
- **useCallback** — Ghi nhớ hàm để tránh re-render không cần thiết
- **useLayoutEffect** — Tương tự useEffect nhưng đồng bộ với DOM
- **useId, useTransition, useDeferredValue** — Hooks tối ưu UX
- **React v19: `use()`** — Hook đọc Promise và Context
- **React v19: `useActionState()`** — Quản lý state từ Form Actions
- **React v19: `useFormStatus()`** — Trạng thái submit của form cha
- **React v19: `useOptimistic()`** — Cập nhật UI lạc quan

### 📁 **3. State Management — Quản Lý Trạng Thái** (`03-state-management/`)

- **Context API** — Quản lý state toàn cục tích hợp sẵn
- **Redux Toolkit** — RTK — Bộ công cụ Redux chính thức
- **RTK Query** — Data fetching & caching tích hợp trong Redux
- **Zustand** — Thư viện state management nhẹ, đơn giản
- **Jotai** — Quản lý state theo nguyên tử (atomic)
- **Recoil** — Atomic state management từ Meta
- **So sánh & Lựa chọn** — Khi nào dùng gì

### 📁 **4. Routing — Điều Hướng** (`04-routing/`)

- **React Router v6/v7** — Thư viện điều hướng phổ biến nhất
- **Nested Routes** — Route lồng nhau
- **Dynamic Routes** — Route động với params
- **Protected Routes** — Route được bảo vệ (Authentication)
- **TanStack Router** — Thư viện routing type-safe hiện đại
- **File-based Routing** — Điều hướng dựa trên cấu trúc thư mục (Next.js, Remix)

### 📁 **5. Data Fetching — Lấy Dữ Liệu** (`05-data-fetching/`)

- **TanStack Query (React Query)** — Server state management, caching, synchronization
- **SWR** — Stale-While-Revalidate — Chiến lược lấy dữ liệu từ Vercel
- **React Server Components (RSC)** — Component chạy hoàn toàn trên server
- **Server Actions** — Hành động server tích hợp trong React v19
- **Suspense & Concurrent Fetching** — Lấy dữ liệu đồng thời
- **Optimistic Updates** — Cập nhật lạc quan với useOptimistic

### 📁 **6. Performance — Hiệu Năng** (`06-performance/`)

- **React.memo** — HOC — Higher-Order Component ngăn re-render không cần
- **useMemo & useCallback** — Memoization — Ghi nhớ để tối ưu
- **Code Splitting** — Tách code với React.lazy và dynamic import
- **Suspense** — Hiển thị trạng thái loading khai báo
- **Virtualization** — Ảo hóa danh sách (react-window, react-virtual)
- **Concurrent Mode** — Chế độ đồng thời với startTransition, useDeferredValue
- **Profiler** — React DevTools Profiler — Phân tích hiệu năng
- **Bundle Optimization** — Tối ưu kích thước bundle

### 📁 **7. Testing — Kiểm Thử** (`07-testing/`)

- **Jest** — Framework kiểm thử JavaScript phổ biến
- **React Testing Library (RTL)** — Kiểm thử theo góc nhìn người dùng
- **Vitest** — Framework kiểm thử nhanh hơn cho Vite projects
- **Mock & Stub** — Giả lập dependencies
- **Integration Testing** — Kiểm thử tích hợp
- **E2E Testing** — Kiểm thử đầu cuối với Playwright, Cypress
- **Component Testing** — Kiểm thử component đơn lẻ

### 📁 **8. Styling — Tạo Kiểu Dáng** (`08-styling/`)

- **CSS Modules** — Module CSS cô lập theo phạm vi
- **Tailwind CSS** — Framework CSS theo tiện ích (utility-first)
- **Styled Components** — CSS-in-JS với tagged template literals
- **Emotion** — Thư viện CSS-in-JS hiệu năng cao
- **CSS Variables** — Biến CSS cho theme động
- **Design Systems** — Hệ thống thiết kế (shadcn/ui, Radix UI, MUI)

### 📁 **9. Ecosystem — Hệ Sinh Thái** (`09-ecosystem/`)

- **Next.js** — Meta-framework phổ biến nhất (App Router, Pages Router)
- **Remix** — Framework tập trung vào Web Standards
- **Vite** — Build tool — Công cụ build siêu nhanh
- **Turbopack** — Trình đóng gói kế nhiệm Webpack (Rust-based)
- **TypeScript** — Kiểu dữ liệu tĩnh cho JavaScript
- **Storybook** — Phát triển và tài liệu hóa UI components độc lập
- **Monorepo** — Quản lý nhiều package trong một repo (Turborepo, Nx)

### 📁 **10. Advanced Patterns — Mẫu Nâng Cao** (`10-advanced-patterns/`)

- **Compound Components** — Component hợp thành (ví dụ: Select + Option)
- **Render Props** — Chia sẻ logic qua prop là hàm render
- **Higher-Order Components (HOC)** — Bọc component để thêm chức năng
- **Custom Hooks** — Trích xuất và tái sử dụng logic
- **Provider Pattern** — Mẫu cung cấp dữ liệu toàn cục
- **Observer Pattern** — Mẫu quan sát cho event handling
- **Portals** — Render component ra ngoài DOM hierarchy
- **Error Boundaries** — Ranh giới bắt lỗi trong component tree
- **Controlled vs Uncontrolled** — Chiến lược quản lý form

### 📁 **11. Interview Prep — Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 30 câu hỏi phỏng vấn React
- System Design với React — Thiết Kế Hệ Thống
- Các câu hỏi về hiệu năng và tối ưu
- Câu hỏi về React v19 và tính năng mới
- Kịch bản thực tế (STAR method)
- Live coding challenges — Thử thách code trực tiếp

---

## 🆕 React v19 — Tính Năng Mới Quan Trọng

### **Actions & Form Integration**

```
- Server Actions — Gọi hàm server trực tiếp từ client
- useActionState() — Quản lý state từ kết quả Action
- useFormStatus() — Đọc trạng thái pending của form cha
- useOptimistic() — Cập nhật UI lạc quan trước khi server xác nhận
```

### **Compiler (React Forget)**

```
- React Compiler — Tự động tối ưu memoization
- Không còn cần useMemo/useCallback thủ công trong nhiều trường hợp
- Compile-time optimization thay vì runtime
```

### **New APIs**

```
- use() Hook — Đọc Promise, Context trong render
- ref as prop — Không cần forwardRef nữa
- Context.Provider → Context trực tiếp
- Metadata trong JSX (<title>, <meta> trong component)
- Resource Preloading APIs (preload, preinit, prefetchDNS)
```

---

## 📊 Ma Trận Kỹ Năng

### Beginner — Mới Bắt Đầu (0-6 tháng)

- [ ] Hiểu JSX và cú pháp cơ bản
- [ ] Tạo Functional Components
- [ ] Sử dụng useState và useEffect
- [ ] Truyền Props giữa components
- [ ] Render danh sách với key
- [ ] Xử lý form cơ bản (controlled components)
- [ ] Sử dụng Context API đơn giản

### Intermediate — Trung Cấp (6 tháng - 2 năm)

- [ ] Thành thạo tất cả built-in hooks
- [ ] Viết Custom Hooks
- [ ] Redux Toolkit hoặc Zustand
- [ ] React Router v6 — Nested routes, protected routes
- [ ] TanStack Query — Caching, invalidation, optimistic updates
- [ ] React.memo, useMemo, useCallback đúng cách
- [ ] Code splitting với React.lazy + Suspense
- [ ] Viết tests với RTL
- [ ] TypeScript cơ bản với React

### Advanced — Nâng Cao (2+ năm)

- [ ] React Server Components (RSC) & Server Actions
- [ ] React v19 tính năng mới
- [ ] Next.js App Router toàn diện
- [ ] Performance profiling và tối ưu
- [ ] Advanced patterns (Compound, HOC, Render Props)
- [ ] Micro-frontends architecture
- [ ] Design Systems và Component Libraries
- [ ] Accessibility (WCAG 2.1)
- [ ] Animation nâng cao (Framer Motion)
- [ ] Monorepo setup (Turborepo)

---

## 🚀 Hướng Dẫn Bắt Đầu

### Bước 1: Thiết Lập Môi Trường

```bash
# Tạo project React với Vite (khuyến nghị)
npm create vite@latest my-react-app -- --template react-ts

# Hoặc dùng Next.js
npx create-next-app@latest my-next-app --typescript --tailwind --app

# Chạy development server
npm run dev
```

### Bước 2: Học Theo Từng Module

```
1. Đọc nội dung lý thuyết (30 phút)
2. Thực hành code theo ví dụ (30 phút)
3. Tự xây dựng mini-project áp dụng kiến thức (60 phút)
4. Review checklist và self-assessment (15 phút)
```

### Bước 3: Xây Dựng Portfolio Projects

```
Beginner:     Todo App, Weather App, Calculator
Intermediate: E-commerce UI, Dashboard, Blog với Next.js
Advanced:     Real-time Chat, Video Platform, Design System
```

### Bước 4: Chuẩn Bị Phỏng Vấn

```
Với mỗi topic, chuẩn bị:
- Giải thích concept rõ ràng (không dùng tài liệu)
- Viết code demo trên whiteboard/editor
- Kể về kinh nghiệm thực tế (STAR method)
- Biết trade-offs của từng giải pháp
```

---

## 🔗 Quick Links — Liên Kết Nhanh

| Nội Dung                   | Thư Mục                                                            | Ưu Tiên             |
| -------------------------- | ------------------------------------------------------------------ | ------------------- |
| Bắt đầu học React          | [01-fundamentals](./01-fundamentals/)                              | Bắt đầu tại đây     |
| Hiểu Hooks sâu             | [02-hooks](./02-hooks/)                                            | Quan trọng nhất      |
| Quản lý state              | [03-state-management](./03-state-management/)                      | Hay được hỏi        |
| Data fetching              | [05-data-fetching](./05-data-fetching/)                            | Thực tế cao          |
| Tối ưu hiệu năng           | [06-performance](./06-performance/)                                | Phỏng vấn senior    |
| Chuẩn bị phỏng vấn         | [11-interview-prep](./11-interview-prep/)                          | Trước phỏng vấn     |

---

## 📖 Tài Liệu Tham Khảo

### Đọc Bắt Buộc

- **React Official Docs** — [react.dev](https://react.dev) — Tài liệu chính thức mới nhất
- **"Fluent React"** by Tejas Kumar — Hiểu sâu cơ chế hoạt động React
- **"Learning React"** by Alex Banks & Eve Porcello — Nền tảng toàn diện
- **"React Design Patterns"** — Các mẫu thiết kế thực chiến

### Tài Liệu Chính Thức

- [React v19 Release Notes](https://react.dev/blog/2024/12/05/react-19)
- [React Router Docs](https://reactrouter.com/)
- [TanStack Query Docs](https://tanstack.com/query)
- [Next.js Documentation](https://nextjs.org/docs)
- [Redux Toolkit Docs](https://redux-toolkit.js.org/)
- [Zustand Docs](https://docs.pmnd.rs/zustand)

### Kênh Học Tập Chất Lượng

- React Core Team Blog — [react.dev/blog](https://react.dev/blog)
- Josh Comeau — [joshwcomeau.com](https://www.joshwcomeau.com/)
- Kent C. Dodds — [kentcdodds.com](https://kentcdodds.com/)
- Tanner Linsley (TanStack creator)
- ByteGrad, Theo Browne (t3.gg), Jack Herrington

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Thường Gặp Theo Chủ Đề

#### Fundamentals & Hooks

- [ ] Virtual DOM — DOM Ảo là gì và hoạt động như thế nào?
- [ ] Reconciliation — Thuật toán so sánh và cập nhật DOM
- [ ] Tại sao cần key trong danh sách?
- [ ] useEffect vs useLayoutEffect — Khi nào dùng gì?
- [ ] Giải thích Closure trong hooks (stale closures)
- [ ] Rules of Hooks — Tại sao không được gọi hook trong điều kiện?

#### State Management

- [ ] Khi nào dùng local state, khi nào dùng global state?
- [ ] Prop drilling — Vấn đề và giải pháp
- [ ] Redux vs Zustand — Trade-offs là gì?
- [ ] Tối ưu Context API để tránh re-render không cần thiết

#### Performance

- [ ] Khi nào React.memo thực sự giúp ích?
- [ ] Phân biệt useMemo và useCallback
- [ ] Code splitting — Tách code ở đâu trong ứng dụng?
- [ ] Concurrent Mode — startTransition làm gì?
- [ ] Sử dụng React Profiler để tìm bottleneck

#### React v19 & Modern React

- [ ] Server Components vs Client Components — Phân biệt và khi nào dùng?
- [ ] Server Actions hoạt động như thế nào?
- [ ] useOptimistic — Cơ chế Optimistic UI
- [ ] React Compiler — Tự động memoization

#### System Design

- [ ] Thiết kế kiến trúc Frontend cho ứng dụng lớn
- [ ] Chiến lược chia nhỏ component (component decomposition)
- [ ] Quản lý authentication trong SPA — Single Page Application
- [ ] Micro-frontends — Khi nào nên áp dụng?

Xem `11-interview-prep/` để có bộ Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước khi phỏng vấn hoặc nhận dự án mới:

- [ ] Có thể giải thích Virtual DOM và Reconciliation không cần note
- [ ] Viết Custom Hook phức tạp từ đầu
- [ ] Thiết kế cấu trúc state cho ứng dụng phức tạp
- [ ] Tối ưu component bị re-render quá nhiều
- [ ] Triển khai Protected Routes với authentication
- [ ] Implement infinite scroll với TanStack Query
- [ ] Viết unit test và integration test với RTL
- [ ] Migrate sang React v19 features (biết cái mới thay gì cũ)
- [ ] Giải thích trade-offs giữa các state management libraries
- [ ] Thiết kế component API theo Compound Components pattern

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn mức độ học (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Thiết lập môi trường thực hành với Vite hoặc Next.js
├─ 5️⃣  Hoàn thành bài tập thực hành từng topic
├─ 6️⃣  Xây dựng portfolio project
└─ 7️⃣  Ôn tập phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-04
**Phiên Bản:** 1.0
**React Version:** v19 (stable)
**Người Duy Trì:** Backend Interview Prep
