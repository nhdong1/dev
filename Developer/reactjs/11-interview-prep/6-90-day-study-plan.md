# 📅 Kế Hoạch Học React 90 Ngày — Từ Beginner Đến Job-Ready

> Lộ trình học có cấu trúc, thực tế, và đo lường được. Mỗi tuần có mục tiêu rõ ràng, project thực hành, và checklist kiểm tra tiến độ.

---

## 🎯 Cam Kết Học Tập

Trước khi bắt đầu, hãy tự đặt ra:

| Câu Hỏi | Câu Trả Lời Của Bạn |
|---------|---------------------|
| Tôi có bao nhiêu giờ/ngày để học? | ___ giờ/ngày |
| Mục tiêu của tôi là gì? | Junior / Mid / Senior |
| Timeline mong muốn có job? | ___ tháng |
| Tôi đã biết gì về JavaScript? | Beginner / Intermediate / Advanced |

**Khuyến nghị:** Tối thiểu **2 giờ/ngày** để hoàn thành 90 ngày plan này.

---

## 🗺️ Tổng Quan 90 Ngày

```
Tháng 1 (Ngày 1-30): Nền Tảng Vững Chắc
├── Tuần 1-2: React Fundamentals + Hooks cơ bản
├── Tuần 3-4: Hooks nâng cao + State Management
└── Milestone: Build Todo App hoàn chỉnh với TypeScript

Tháng 2 (Ngày 31-60): Ecosystem & Quality
├── Tuần 5-6: Routing + Data Fetching
├── Tuần 7-8: Testing + Performance
└── Milestone: Build E-commerce product list với full features

Tháng 3 (Ngày 61-90): Senior Skills & Job Prep
├── Tuần 9-10: Next.js + React v19
├── Tuần 11-12: System Design + Interview Prep
└── Milestone: Portfolio project production-ready + Mock interviews
```

---

## 📆 Tháng 1: Nền Tảng (Ngày 1-30)

---

### Tuần 1 (Ngày 1-7): React Fundamentals — Nền Tảng React

**Mục tiêu cuối tuần:** Render được UI phức tạp bằng React không cần nhìn tài liệu

#### Ngày 1-2: JSX & Components

**Lý thuyết (2 giờ):**
- [ ] Đọc: `01-fundamentals/1-jsx-components.md`
- [ ] Hiểu: JSX compile thành gì, sự khác biệt với HTML
- [ ] Thực hành: Viết 10 Functional Components từ đơn giản đến phức tạp

**Code (2 giờ):**
```jsx
// Bài tập 1: Profile Card component
function ProfileCard({ name, role, avatar, skills }) {
  return (/* ... */);
}

// Bài tập 2: Navigation Menu
function NavMenu({ items, activeItem }) {
  return (/* ... */);
}

// Bài tập 3: Product Card với badge
function ProductCard({ product, isFeatured }) {
  return (/* ... */);
}
```

#### Ngày 3-4: Props & State

**Lý thuyết (2 giờ):**
- [ ] Đọc: `01-fundamentals/2-props-state.md`
- [ ] Hiểu: Data flow một chiều, khi nào nâng state lên (lifting state up)

**Code (2 giờ):**
```jsx
// Bài tập: Counter với nhiều controls
function Counter() {
  // Implement: increment, decrement, reset, step size
}

// Bài tập: Parent-child communication
function ShoppingCart() {
  // Parent giữ cart state, truyền callbacks xuống CartItem
}
```

#### Ngày 5-7: Events, Conditional Rendering, Lists

**Lý thuyết (3 giờ):**
- [ ] Đọc: `01-fundamentals/3-event-handling.md`
- [ ] Đọc: `01-fundamentals/4-conditional-list-rendering.md`
- [ ] Đọc: `01-fundamentals/5-forms-controlled.md`

**Weekend Project — Dự Án Cuối Tuần (4 giờ):**
```
Mini Project: Notes App cơ bản
├── Thêm ghi chú (controlled form)
├── Hiển thị danh sách ghi chú
├── Xóa ghi chú
└── Filter ghi chú theo trạng thái
```

**Checklist Tuần 1:**
- [ ] Tạo Functional Component không nhìn docs
- [ ] Hiểu sự khác biệt của JSX vs HTML (className, style object...)
- [ ] Truyền và sử dụng props
- [ ] Dùng useState cho counter, toggle, list
- [ ] Render list với key đúng cách
- [ ] Handle events (onClick, onChange, onSubmit)

---

### Tuần 2 (Ngày 8-14): React Hooks — Cơ Bản

**Mục tiêu cuối tuần:** Thành thạo useState, useEffect, viết được custom hook đầu tiên

#### Ngày 8-10: useState chuyên sâu + useEffect

**Lý thuyết (3 giờ):**
- [ ] Đọc: `02-hooks/1-usestate-usereducer.md`
- [ ] Đọc: `02-hooks/2-useeffect-uselayout.md`
- [ ] Focus: Lazy initializer, functional updater, cleanup function, race condition

**Code (3 giờ):**
```jsx
// Bài tập 1: Implement usePrevious hook
function usePrevious(value) { /* ... */ }

// Bài tập 2: Fetch data với loading/error state và race condition fix
function useFetch(url) { /* ... */ }

// Bài tập 3: Window size tracker
function useWindowSize() { /* ... */ }
```

#### Ngày 11-12: useContext, useRef, useReducer

**Lý thuyết (2 giờ):**
- [ ] Đọc: `02-hooks/3-usecontext-useref.md`
- [ ] Hiểu: Khi nào useRef thay vì useState

**Code (2 giờ):**
```jsx
// Bài tập: Theme context với dark/light mode
const ThemeContext = createContext();

// Bài tập: useReducer cho form state phức tạp
const formReducer = (state, action) => { /* ... */ }

// Bài tập: Scroll to top button với ref
function ScrollToTop() { /* ... */ }
```

#### Ngày 13-14: useMemo, useCallback + Weekend Project

**Lý thuyết (2 giờ):**
- [ ] Đọc: `02-hooks/4-usememo-usecallback.md`
- [ ] Hiểu: Khi nào thực sự cần, overhead của memoization

**Weekend Project (5 giờ):**
```
Project: Weather App
├── Fetch weather data theo city (useEffect + fetch)
├── Debounce search input (custom hook)
├── Loading skeleton
├── Error handling
└── 5-day forecast (array render)
```

**Checklist Tuần 2:**
- [ ] useEffect: 3 patterns (no deps, [], [deps])
- [ ] Cleanup function — khi nào cần
- [ ] Race condition trong fetch — cách fix
- [ ] useRef: DOM ref và mutable value
- [ ] Context API: Provider và Consumer
- [ ] Viết được useDebounce hook từ đầu

---

### Tuần 3 (Ngày 15-21): State Management

**Mục tiêu cuối tuần:** Implement được Zustand store và Redux Toolkit slice

#### Ngày 15-17: Zustand

**Lý thuyết (2 giờ):**
- [ ] Đọc: `03-state-management/4-zustand.md`
- [ ] Đọc: `03-state-management/6-comparison-guide.md`

**Code (3 giờ):**
```javascript
// Bài tập: Shopping cart store
const useCartStore = create((set, get) => ({
  items: [],
  addItem: (product) => { /* ... */ },
  removeItem: (id) => { /* ... */ },
  getTotal: () => { /* ... */ },
}));

// Bài tập: User store với persist
const useUserStore = create(
  persist(/* ... */, { name: 'user-storage' })
);
```

#### Ngày 18-21: Redux Toolkit

**Lý thuyết (3 giờ):**
- [ ] Đọc: `03-state-management/2-redux-toolkit.md`
- [ ] Đọc: `03-state-management/3-rtk-query.md`

**Code (3 giờ):**
```javascript
// Bài tập: Todo list với Redux Toolkit
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => { /* ... */ },
    toggleTodo: (state, action) => { /* ... */ },
    deleteTodo: (state, action) => { /* ... */ },
  },
});

// Bài tập: RTK Query để fetch users
const usersApi = createApi({ /* ... */ });
```

**Checklist Tuần 3:**
- [ ] Tạo Zustand store với actions
- [ ] Persist Zustand store với localStorage
- [ ] Tạo Redux slice với Immer mutations
- [ ] createAsyncThunk cho async operations
- [ ] RTK Query: useQuery và useMutation cơ bản
- [ ] Biết khi nào chọn Zustand vs Redux

---

### Tuần 4 (Ngày 22-30): TypeScript + Tháng 1 Capstone Project

**Mục tiêu:** TypeScript cơ bản + build project hoàn chỉnh đầu tiên

#### Ngày 22-24: TypeScript với React

**Lý thuyết (3 giờ):**
- [ ] Đọc: `09-ecosystem/5-typescript-react.md`
- [ ] Focus: Component props typing, hooks typing, generics

**Code (2 giờ):**
```tsx
// Bài tập: Typing React components
interface ButtonProps {
  variant: 'primary' | 'secondary';
  onClick: () => void;
  children: React.ReactNode;
  disabled?: boolean;
}

// Generic component
function DataList<T extends { id: string | number }>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}) {
  return <ul>{items.map(item => <li key={item.id}>{renderItem(item)}</li>)}</ul>;
}
```

#### Ngày 25-30: CAPSTONE PROJECT THÁNG 1

```
Project: Todo App Hoàn Chỉnh (TypeScript)

Features:
├── CRUD todos (add, edit, delete, toggle)
├── Categories/tags cho todos
├── Filter & search
├── Drag & drop reorder (optional)
├── Persist to localStorage (Zustand persist)
├── Dark mode (Context API + localStorage)
└── Deadline + overdue detection

Tech Stack:
├── React + TypeScript
├── Zustand cho state
├── Vite
└── CSS Modules

Không dùng UI library — viết CSS tự

Thời gian: 20-25 giờ (5-6 ngày)
```

---

## 📆 Tháng 2: Ecosystem & Quality (Ngày 31-60)

---

### Tuần 5 (Ngày 31-37): Routing — Điều Hướng

**Mục tiêu:** Implement được routing phức tạp với protected routes

#### Ngày 31-33: React Router v6

**Lý thuyết (2 giờ):**
- [ ] Đọc: `04-routing/1-react-router-basics.md`
- [ ] Đọc: `04-routing/2-nested-dynamic-routes.md`

**Code (3 giờ):**
```jsx
// Bài tập: Multi-level nested routes
<Routes>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<DashboardHome />} />
    <Route path="users" element={<UserList />} />
    <Route path="users/:userId" element={<UserDetail />} />
    <Route path="settings/*" element={<Settings />} />
  </Route>
</Routes>

// Bài tập: URL state (filter/pagination trong URL)
function useTableState() {
  const [searchParams, setSearchParams] = useSearchParams();
  // ...
}
```

#### Ngày 34-37: Protected Routes + TanStack Router

**Lý thuyết (2 giờ):**
- [ ] Đọc: `04-routing/3-protected-routes.md`
- [ ] Đọc: `04-routing/5-tanstack-router.md`

**Checklist Tuần 5:**
- [ ] Nested routes với Outlet
- [ ] Dynamic routes với useParams
- [ ] Protected routes với Navigate
- [ ] URL state với useSearchParams
- [ ] useNavigate và programmatic navigation

---

### Tuần 6 (Ngày 38-44): Data Fetching — Lấy Dữ Liệu

**Mục tiêu:** Thành thạo TanStack Query, implement được infinite scroll

#### Ngày 38-40: TanStack Query cơ bản

**Lý thuyết (2 giờ):**
- [ ] Đọc: `05-data-fetching/1-tanstack-query-basics.md`

**Code (3 giờ):**
```javascript
// Bài tập: User list với useQuery
const { data, isLoading, error } = useQuery({
  queryKey: ['users', { page, search }],
  queryFn: () => fetchUsers({ page, search }),
  placeholderData: keepPreviousData, // Smooth pagination
});

// Bài tập: Create user với useMutation
const createUser = useMutation({
  mutationFn: (data) => api.post('/users', data),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
});
```

#### Ngày 41-44: TanStack Query nâng cao

**Lý thuyết (2 giờ):**
- [ ] Đọc: `05-data-fetching/2-tanstack-query-advanced.md`

**Code (3 giờ):**
```javascript
// Bài tập: Infinite scroll
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: ({ pageParam }) => fetchPosts({ cursor: pageParam }),
  getNextPageParam: (lastPage) => lastPage.nextCursor,
});

// Bài tập: Optimistic update cho like button
const toggleLike = useMutation({
  mutationFn: likePost,
  onMutate: async (postId) => {
    // Optimistic update
  },
  onError: (err, postId, context) => {
    // Rollback
  },
});
```

**Checklist Tuần 6:**
- [ ] useQuery với queryKey, queryFn, staleTime
- [ ] useMutation với invalidateQueries
- [ ] Infinite scroll với useInfiniteQuery
- [ ] Optimistic updates với rollback
- [ ] Background refetching, cache management

---

### Tuần 7 (Ngày 45-51): Testing — Kiểm Thử

**Mục tiêu:** Viết được unit tests và integration tests cho React components

#### Ngày 45-48: Jest + React Testing Library

**Lý thuyết (2 giờ):**
- [ ] Đọc: `07-testing/1-jest-basics.md`
- [ ] Đọc: `07-testing/2-react-testing-library.md`

**Code (4 giờ):**
```jsx
// Bài tập: Test Button component
describe('Button', () => {
  test('renders correctly', () => { /* ... */ });
  test('calls onClick when clicked', () => { /* ... */ });
  test('is disabled when disabled prop is true', () => { /* ... */ });
  test('shows loading state', () => { /* ... */ });
});

// Bài tập: Test form submit
test('submits form with correct data', async () => {
  const handleSubmit = jest.fn();
  render(<LoginForm onSubmit={handleSubmit} />);

  await userEvent.type(screen.getByLabelText('Email'), 'test@example.com');
  await userEvent.type(screen.getByLabelText('Password'), 'password123');
  await userEvent.click(screen.getByRole('button', { name: 'Đăng Nhập' }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123',
  });
});
```

#### Ngày 49-51: Mocking + MSW — Mock Service Worker

**Lý thuyết (2 giờ):**
- [ ] Đọc: `07-testing/3-mocking.md`
- [ ] Đọc: `07-testing/4-integration-testing.md`

**Checklist Tuần 7:**
- [ ] Test rendered output với getBy queries
- [ ] userEvent cho interactions
- [ ] Mock functions với jest.fn()
- [ ] MSW cho API mocking
- [ ] Testing async với waitFor

---

### Tuần 8 (Ngày 52-60): Performance + Tháng 2 Capstone

#### Ngày 52-54: Performance Optimization

**Lý thuyết (2 giờ):**
- [ ] Đọc: `06-performance/1-memoization.md`
- [ ] Đọc: `06-performance/2-code-splitting.md`
- [ ] Đọc: `06-performance/3-virtualization.md`

**Code (2 giờ):**
```jsx
// Bài tập: Virtualize 10,000 item list
// Bài tập: Code split heavy chart component
// Bài tập: Profile và fix a slow component
```

#### Ngày 55-60: CAPSTONE PROJECT THÁNG 2

```
Project: GitHub Explorer (E-commerce style app)

Features:
├── Search GitHub users/repos
├── TanStack Query cho data fetching
├── Infinite scroll cho search results
├── Protected routes (mock auth)
├── Sorting, filtering (URL state)
├── Loading skeletons
├── Error boundaries
├── Unit tests cho 3+ components
└── Integration test cho search flow

Tech Stack:
├── React + TypeScript
├── TanStack Query
├── React Router v6
├── Zustand (nếu cần global state)
└── Vitest + RTL

Thời gian: 25-30 giờ
```

---

## 📆 Tháng 3: Senior Skills & Job Prep (Ngày 61-90)

---

### Tuần 9 (Ngày 61-67): Next.js App Router

**Mục tiêu:** Build được full-stack app với Next.js, RSC, và Server Actions

#### Ngày 61-64: Next.js Fundamentals

**Lý thuyết (3 giờ):**
- [ ] Đọc: `09-ecosystem/1-nextjs-app-router.md`
- [ ] Focus: App Router, Server Components, Client Components

**Code (4 giờ):**
```
Dự án: Blog với Next.js App Router
├── app/
│   ├── layout.tsx (Root Layout)
│   ├── page.tsx (Home — Server Component)
│   ├── blog/
│   │   ├── page.tsx (Blog List — fetch từ API)
│   │   └── [slug]/page.tsx (Blog Detail — generateStaticParams)
│   └── contact/
│       └── page.tsx (Client Component với form)
```

#### Ngày 65-67: Server Actions + React v19

**Lý thuyết (2 giờ):**
- [ ] Đọc: `05-data-fetching/5-server-actions.md`
- [ ] Đọc: `11-interview-prep/5-react19-questions.md`

**Code (2 giờ):**
```tsx
// Bài tập: Form với Server Action + useActionState
'use server';
async function subscribeNewsletter(prevState, formData) {
  const email = formData.get('email');
  // Validate + save to DB
  return { success: true };
}

'use client';
function NewsletterForm() {
  const [state, action, isPending] = useActionState(subscribeNewsletter, null);
  return <form action={action}>{ /* ... */ }</form>;
}
```

**Checklist Tuần 9:**
- [ ] Hiểu App Router directory structure
- [ ] Server Component vs Client Component — khi nào dùng
- [ ] Server Actions với useActionState
- [ ] generateStaticParams cho SSG
- [ ] Metadata API

---

### Tuần 10 (Ngày 68-74): Advanced Patterns + React v19

**Mục tiêu:** Thành thạo advanced patterns — điểm cộng trong phỏng vấn Senior

#### Ngày 68-71: Advanced Component Patterns

**Lý thuyết (2 giờ):**
- [ ] Đọc: `10-advanced-patterns/1-compound-components.md`
- [ ] Đọc: `10-advanced-patterns/5-portals-boundaries.md`

**Code (3 giờ):**
```jsx
// Bài tập: Implement Tabs component với Compound Components
const Tabs = {
  Root: TabsRoot,
  List: TabsList,
  Tab: Tab,
  Panels: TabsPanels,
  Panel: TabsPanel,
};

// Dùng:
<Tabs.Root defaultValue="profile">
  <Tabs.List>
    <Tabs.Tab value="profile">Hồ Sơ</Tabs.Tab>
    <Tabs.Tab value="security">Bảo Mật</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panels>
    <Tabs.Panel value="profile"><ProfileForm /></Tabs.Panel>
    <Tabs.Panel value="security"><SecurityForm /></Tabs.Panel>
  </Tabs.Panels>
</Tabs.Root>
```

#### Ngày 72-74: Concurrent Features nâng cao

**Lý thuyết + Code (4 giờ):**
- [ ] Đọc: `06-performance/4-concurrent-features.md`
- [ ] Implement: Search với startTransition + useDeferredValue
- [ ] Implement: Streaming SSR example

---

### Tuần 11 (Ngày 75-81): Interview Preparation — Chuẩn Bị Phỏng Vấn

**Mục tiêu:** Tự tin trả lời mọi câu hỏi trong 11-interview-prep/

#### Ngày 75-77: Ôn lý thuyết + Flashcards

```
Ngày 75: Ôn INTERVIEW_GUIDE.md (30 câu hỏi)
Ngày 76: Ôn 2-hooks-deep-dive.md
Ngày 77: Ôn 3-performance-questions.md + 5-react19-questions.md
```

**Phương pháp:**
- Đọc câu hỏi → Che đáp án → Tự trả lời to → So sánh
- Dùng Anki hoặc Notion để tạo flashcards

#### Ngày 78-81: System Design Practice

```
Ngày 78: Thiết kế E-commerce frontend (30 phút whiteboard)
Ngày 79: Thiết kế Social Media Feed (30 phút)
Ngày 80: Thiết kế Real-time Collaboration Tool (30 phút)
Ngày 81: Review và refine answers
```

**Đọc:** `4-system-design.md`

---

### Tuần 12 (Ngày 82-90): Mock Interviews + Portfolio Polish

#### Ngày 82-84: Live Coding Practice

```
Mỗi ngày: 2 bài tập live coding (30 phút/bài, không nhìn docs)

Bài tập mẫu:
├── Implement useLocalStorage hook
├── Build Accordion component
├── Fix bug: stale closure trong setInterval
├── Implement infinite scroll
├── Viết test cho LoginForm
└── Optimize slow ProductList component
```

#### Ngày 85-87: Mock Interviews — Phỏng Vấn Thử

```
Hình thức: Phỏng vấn với bạn bè hoặc mentor
Mỗi session (60 phút):
├── 5 phút giới thiệu bản thân
├── 20 phút câu hỏi kỹ thuật (React fundamentals)
├── 20 phút live coding
├── 15 phút system design
└── Feedback và cải thiện
```

**Nếu không có người mock:** Record video tự phỏng vấn bản thân, xem lại và phê bình.

#### Ngày 88-90: FINAL PORTFOLIO PROJECT

```
Project: Full-Stack App với Next.js (1 tuần kết tinh)

Ý tưởng:
Option A: Task Management App (Trello-like)
  ├── Drag & drop boards/cards
  ├── Real-time collaboration (WebSocket)
  ├── Authentication (NextAuth.js)
  └── DB (Prisma + PostgreSQL/SQLite)

Option B: Blog Platform
  ├── MDX content với live preview
  ├── Authentication để write
  ├── Comments với Server Actions
  └── RSS feed

Option C: Job Board
  ├── Company + Job posting với CRUD
  ├── Advanced filtering (URL state)
  ├── Application tracking
  └── Email notifications (Resend)

Yêu cầu bắt buộc:
├── Production-ready code (không có console.log)
├── TypeScript strict mode
├── Unit tests cho core logic
├── README đầy đủ (setup, architecture)
├── Deploy lên Vercel
└── Code trên GitHub với commit history rõ ràng
```

---

## 📊 Milestone Tracking — Theo Dõi Tiến Độ

### Cuối Tháng 1

```
□ Giải thích Virtual DOM và Reconciliation không nhìn notes
□ Viết useState, useEffect, useContext từ đầu
□ Tạo custom hook (useDebounce, useFetch, useLocalStorage)
□ Hoàn thành Todo App với TypeScript + Zustand
□ Giải thích prop drilling và 3 cách giải quyết
```

### Cuối Tháng 2

```
□ Setup routing với nested routes và protected routes
□ Implement TanStack Query với caching và optimistic updates
□ Viết tests đạt coverage > 70% cho features chính
□ Code split app theo routes
□ Hoàn thành GitHub Explorer project + deploy
```

### Cuối Tháng 3 (Job Ready)

```
□ Build và deploy Next.js app với Server Components
□ Trả lời 30 câu INTERVIEW_GUIDE không nhìn notes
□ Thiết kế kiến trúc frontend cho medium-scale app (30 phút)
□ Live code implement useCallback, useMemo, custom hook (không nhìn docs)
□ Portfolio: 2+ projects trên GitHub với README tốt
□ LinkedIn profile updated với React projects
```

---

## 💡 Mẹo Học Hiệu Quả

### Phương Pháp Feynman cho React

```
1. Học concept → Giải thích cho "người mới" (bạn bè, ghi chú)
2. Nếu không giải thích được → Quay lại học lại
3. Đơn giản hóa ngôn ngữ → Không dùng jargon
4. Review → Học lại điểm yếu
```

**Ví dụ:** "Virtual DOM là gì?" → Giải thích cho người không biết React.

---

### Spaced Repetition — Lặp Lại Có Khoảng Cách

```
Ngày học: Học concept mới
Ngày +1: Review nhanh (15 phút)
Ngày +3: Review nhanh (10 phút)
Ngày +7: Review (20 phút)
Ngày +14: Quiz bản thân
Ngày +30: Ôn lại

Dùng: Anki, Notion, hoặc tự tạo flashcards
```

---

### Học Từ Code Thực Tế

```
1. Clone các open source React projects:
   ├── github.com/calcom/cal.com (Next.js, advanced patterns)
   ├── github.com/shadcn-ui/ui (Design system, Radix)
   └── github.com/vercel/commerce (E-commerce, Next.js)

2. Đọc React source code (không cần hiểu hết):
   ├── react/packages/react/src/ReactHooks.js
   └── react/packages/react-reconciler/

3. Theo dõi Twitter/X:
   ├── @acdlite (React team)
   ├── @sophiebits (React team)
   ├── @TkDodo (TanStack Query author)
   └── @t3dotgg (Theo Browne — Next.js content)
```

---

### Debug Như Senior Dev

```
Khi gặp bug:
1. Read the error message carefully — Đọc kỹ thông báo lỗi
2. Add console.log có chiến lược — không random
3. Check React DevTools → component tree, state, props
4. Isolate — Thu hẹp: bug trong component nào?
5. Search: GitHub Issues, Stack Overflow, Discord
6. Rubber duck debugging — Giải thích bug cho người khác (hoặc vịt cao su)
7. Take a break — Đôi khi nhìn lại sau 30 phút thấy ngay vấn đề
```

---

## 📚 Resources — Tài Nguyên Học Tập

### Bắt Buộc Đọc

| Tài Nguyên | Mô Tả | Thời Điểm |
|------------|--------|-----------|
| [react.dev](https://react.dev) | Official docs — tốt nhất hiện tại | Từ đầu |
| [Kent C. Dodds Blog](https://kentcdodds.com) | Testing, Hooks patterns | Tuần 2+ |
| [Josh Comeau Blog](https://joshwcomeau.com) | CSS, React animations | Tuần 4+ |
| [TkDodo's Blog](https://tkdodo.eu/blog) | TanStack Query patterns | Tuần 6 |

### Video

| Kênh | Loại Content | Mức Độ |
|------|-------------|--------|
| ByteGrad | React, Next.js projects | Beginner/Mid |
| Theo Browne (t3.gg) | Modern React ecosystem | Mid/Senior |
| Jack Herrington | Advanced patterns, RSC | Mid/Senior |
| Fireship | Quick overviews | Mọi cấp |

### Tools

```
Editor: VS Code với extensions:
├── ES7+ React Snippets
├── Prettier
├── ESLint
├── TypeScript Error Translator
└── GitLens

Browser:
├── React Developer Tools
└── Redux DevTools

Thực hành online (không cần setup):
├── CodeSandbox
├── StackBlitz
└── Playcode.io
```

---

## 🎯 Sau 90 Ngày — Bước Tiếp Theo

```
Bạn đã hoàn thành 90 ngày nếu:
✅ 2+ projects deploy trên Vercel/Netlify
✅ Trả lời được 30 câu INTERVIEW_GUIDE
✅ Tự tin với Hooks, TypeScript, TanStack Query
✅ Hiểu React Server Components và Server Actions
✅ Viết được tests cơ bản

Bước tiếp theo:
1. Apply cho vị trí Junior/Mid React Developer
2. Tiếp tục học: React Native (mobile), animation (Framer Motion)
3. Contribute to open source React projects
4. Xây dựng thương hiệu cá nhân: viết blog, tham gia cộng đồng
5. Network: Reactiflux Discord, meetups địa phương
```

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
**Thời Gian Hoàn Thành Ước Tính:** 90 ngày × 4 giờ/ngày = 360 giờ
