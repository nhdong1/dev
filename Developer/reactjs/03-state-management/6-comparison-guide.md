# Hướng Dẫn Lựa Chọn State Management — Khi Nào Dùng Gì?

> Đây là hướng dẫn thực tế để chọn đúng giải pháp state management cho từng tình huống, dựa trên quy mô dự án, team, và yêu cầu kỹ thuật.

---

## 📌 Mục Lục

1. [Bản Đồ Quyết Định Nhanh](#1-bản-đồ-quyết-định)
2. [So Sánh Chi Tiết](#2-so-sánh-chi-tiết)
3. [Theo Quy Mô Dự Án](#3-theo-quy-mô-dự-án)
4. [Theo Loại State](#4-theo-loại-state)
5. [Kết Hợp Nhiều Giải Pháp](#5-kết-hợp-nhiều-giải-pháp)
6. [Những Sai Lầm Thường Gặp](#6-những-sai-lầm-thường-gặp)
7. [Câu Hỏi Phỏng Vấn Tổng Hợp](#7-câu-hỏi-phỏng-vấn)

---

## 1. Bản Đồ Quyết Định Nhanh

```
Bắt đầu:
  │
  ├─ Chỉ cần chia sẻ state giữa vài component gần nhau?
  │     └─ → useState + props lifting (nâng state lên cha)
  │
  ├─ State ít thay đổi, app nhỏ, không muốn thêm dependency?
  │     └─ → Context API + useReducer
  │
  ├─ Cần fetch dữ liệu từ API với caching, loading state?
  │     ├─ Đang dùng Redux → RTK Query
  │     └─ Không dùng Redux → TanStack Query (xem 05-data-fetching/)
  │
  ├─ Cần global state, app vừa–lớn, ít boilerplate?
  │     └─ → Zustand
  │
  ├─ App enterprise, nhiều team, cần DevTools mạnh, time-travel debugging?
  │     └─ → Redux Toolkit
  │
  └─ Cần fine-grained reactivity, Suspense integration, atomic model?
        └─ → Jotai
```

---

## 2. So Sánh Chi Tiết

### Bảng So Sánh Toàn Diện

| Tiêu Chí | Context API | Redux Toolkit | RTK Query | Zustand | Jotai |
| -------- | :---------: | :-----------: | :-------: | :-----: | :---: |
| **Bundle size** (kích thước gói) | 0 KB | ~11 KB | ~11 KB | ~1 KB | ~3 KB |
| **Boilerplate** (mã soạn sẵn) | Thấp | Trung bình | Thấp | Rất thấp | Thấp |
| **Learning curve** (độ khó học) | ⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐ |
| **DevTools** (công cụ phát triển) | ❌ | ✅✅✅ | ✅✅✅ | ✅✅ | ✅✅ |
| **TypeScript** | ✅✅ | ✅✅✅ | ✅✅✅ | ✅✅✅ | ✅✅✅ |
| **Server state** | ❌ | ❌ | ✅✅✅ | ❌ | ❌ |
| **Caching** | ❌ | ❌ | ✅✅✅ | ❌ | ❌ |
| **Persist** | Thủ công | Thủ công | ❌ | `persist` middleware | `atomWithStorage` |
| **Async** | Thủ công | `createAsyncThunk` | Tự động | Trực tiếp trong action | Async atoms |
| **Re-render control** | Thô | Selector | Selector | Selector | Fine-grained |
| **Suspense** | ❌ | ❌ | ✅ | ❌ | ✅✅✅ |
| **Phù hợp với** | Small app | Enterprise | Data fetching | Mọi quy mô | Fine-grained UI |

---

## 3. Theo Quy Mô Dự Án

### Small App — Ứng Dụng Nhỏ (1–3 developers, <10K users)

**Khuyến nghị: Context API + useReducer, hoặc Zustand**

```
Ví dụ: Blog cá nhân, landing page, tool nội bộ, POC (Proof of Concept — Bằng Chứng Khái Niệm)

Stack đề xuất:
- Local state:   useState, useReducer
- Global state:  Context API (theme, auth) hoặc Zustand
- Server state:  TanStack Query hoặc SWR (xem 05-data-fetching/)
```

Lý do: Tốc độ phát triển nhanh, ít overhead (chi phí phụ), dễ thay đổi sau.

### Medium App — Ứng Dụng Vừa (2–8 developers, 10K–100K users)

**Khuyến nghị: Zustand + TanStack Query**

```
Ví dụ: SaaS tool, dashboard, e-commerce

Stack đề xuất:
- Local state:   useState, useReducer
- Global state:  Zustand (cart, auth, UI state)
- Server state:  TanStack Query (API data)
- Forms:         React Hook Form
- Routing:       React Router v6
```

Lý do: Cân bằng tốt giữa đơn giản và mạnh mẽ. Dễ onboard (đào tạo) developer mới.

### Large App — Ứng Dụng Lớn (8+ developers, 100K+ users, enterprise)

**Khuyến nghị: Redux Toolkit + RTK Query**

```
Ví dụ: Banking app, ERP (Enterprise Resource Planning — Hệ Thống Hoạch Định Nguồn Lực Doanh Nghiệp), healthcare system

Stack đề xuất:
- Local state:   useState, useReducer
- Global state:  Redux Toolkit (predictable — có thể dự đoán được, debuggable — dễ debug)
- Server state:  RTK Query (integrated caching — caching tích hợp)
- Forms:         React Hook Form + Zod (validation — kiểm tra dữ liệu)
- Testing:       Jest + RTL + MSW (Mock Service Worker — Worker Giả Lập API)
```

Lý do: Predictability, powerful DevTools, large ecosystem, well-known patterns cho team lớn.

---

## 4. Theo Loại State

### UI State (Trạng Thái Giao Diện)

State chỉ liên quan đến giao diện, không cần server biết.

```
Ví dụ: Modal mở/đóng, tab active, sidebar collapse, loading indicator
```

```jsx
// ✅ Giải pháp tốt nhất: useState cục bộ
function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;
  return <div className="modal">{children}</div>;
}

// Nếu nhiều component cần biết modal có đang mở không:
// Zustand hoặc Context API là phù hợp
const useUIStore = create((set) => ({
  activeModal: null,
  openModal: (id) => set({ activeModal: id }),
  closeModal: () => set({ activeModal: null }),
}));
```

### Authentication State (Trạng Thái Xác Thực)

```jsx
// Context API phù hợp — thay đổi ít, cần ở nhiều nơi
const AuthContext = createContext(null);

// Hoặc Zustand với persist
const useAuthStore = create(
  persist(
    (set) => ({
      user: null,
      token: null,
      login: async (credentials) => { /* ... */ },
      logout: () => set({ user: null, token: null }),
    }),
    { name: "auth", partialize: (s) => ({ token: s.token }) }
  )
);
```

### Server State (Trạng Thái Máy Chủ)

```jsx
// TanStack Query hoặc RTK Query — KHÔNG dùng Redux/Zustand cho server state

// ✅ TanStack Query (không dùng Redux)
const { data: users, isLoading } = useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
});

// ✅ RTK Query (đang dùng Redux)
const { data: users, isLoading } = useGetUsersQuery();

// ❌ Chống chỉ định: Dùng Redux/Zustand để lưu server data thủ công
// → Không có caching, phải tự handle loading/error, dễ stale (cũ)
```

### Form State (Trạng Thái Biểu Mẫu)

```jsx
// React Hook Form (khuyến nghị nhất) — không liên quan đến state management libraries
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Email không hợp lệ"),
  password: z.string().min(8, "Mật khẩu tối thiểu 8 ký tự"),
});

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}
      {/* ... */}
    </form>
  );
}
```

### Shared Client State (Trạng Thái Client Chia Sẻ)

```jsx
// Zustand — phù hợp nhất cho client state phức tạp
const useCartStore = create(persist(
  (set, get) => ({
    items: [],
    addItem: (product) => { /* ... */ },
    removeItem: (id) => { /* ... */ },
  }),
  { name: "cart" }
));
```

---

## 5. Kết Hợp Nhiều Giải Pháp

Trong thực tế, hầu hết dự án đều **kết hợp** nhiều giải pháp:

### Stack Được Khuyến Nghị Cho 2026

```
Dự án Medium–Large:

┌────────────────────────────────────────────────────────────┐
│                      Application                           │
├────────────────────────────────────────────────────────────┤
│  Server State      │  TanStack Query hoặc RTK Query        │
│  (API data)        │  → fetch, cache, sync                 │
├────────────────────────────────────────────────────────────┤
│  Global Client     │  Zustand (app vừa) hoặc RTK (lớn)    │
│  State             │  → auth, cart, preferences            │
├────────────────────────────────────────────────────────────┤
│  Shared UI State   │  Context API                          │
│                    │  → theme, language, notifications     │
├────────────────────────────────────────────────────────────┤
│  Form State        │  React Hook Form + Zod                │
│                    │  → validation, submission             │
├────────────────────────────────────────────────────────────┤
│  Local Component   │  useState, useReducer                 │
│  State             │  → modal, input, toggle               │
└────────────────────────────────────────────────────────────┘
```

### Ví Dụ: E-commerce App

```javascript
// Cart — global client state → Zustand (persist vào localStorage)
const useCartStore = create(persist(...));

// Product list, user orders — server state → TanStack Query
const { data: products } = useQuery({ queryKey: ["products"], queryFn: fetchProducts });

// Theme, language — shared UI → Context API
const ThemeContext = createContext(null);

// Checkout form — form state → React Hook Form
const { register, handleSubmit } = useForm();

// Modal mở/đóng — local state → useState
const [isCheckoutOpen, setCheckoutOpen] = useState(false);
```

---

## 6. Những Sai Lầm Thường Gặp

### ❌ Sai Lầm 1: Đặt Server Data Vào Redux/Zustand Thủ Công

```javascript
// ❌ Antipattern — tự manage server state
const useProductStore = create((set) => ({
  products: [],
  isLoading: false,
  error: null,
  fetchProducts: async () => {
    set({ isLoading: true });
    // Phải tự xử lý caching, stale data, refetching...
  },
}));

// ✅ Dùng TanStack Query / RTK Query thay thế
const { data: products } = useQuery({ queryKey: ["products"], queryFn: fetchProducts });
// Tự động: caching, background refetch, stale-while-revalidate, ...
```

### ❌ Sai Lầm 2: Đặt Mọi State Vào Redux

```javascript
// ❌ Không cần thiết
const uiSlice = createSlice({
  name: "ui",
  initialState: { isModalOpen: false },
  reducers: { setModalOpen: (state, action) => { state.isModalOpen = action.payload; } },
});

// ✅ useState là đủ cho modal local
const [isModalOpen, setModalOpen] = useState(false);
```

### ❌ Sai Lầm 3: Dùng Context API Cho State Thay Đổi Thường Xuyên

```javascript
// ❌ Context cho real-time data → gây re-render hàng loạt
const WebSocketContext = createContext(null);
function WSProvider({ children }) {
  const [messages, setMessages] = useState([]);  // Cập nhật mỗi giây
  return <WebSocketContext.Provider value={messages}>{children}</WebSocketContext.Provider>;
}

// ✅ Zustand hoặc Jotai — fine-grained subscription
const useMessagesStore = create((set) => ({
  messages: [],
  addMessage: (msg) => set((s) => ({ messages: [...s.messages, msg] })),
}));
```

### ❌ Sai Lầm 4: Chọn Redux Cho App Nhỏ Vì Thấy Nhiều Công Ty Dùng

Redux phù hợp cho app lớn có nhiều developer. Với app nhỏ, overhead (chi phí setup, boilerplate) làm chậm tốc độ phát triển.

**Quy tắc thực tế:** Bắt đầu đơn giản (useState, sau đó Zustand), chuyển sang Redux khi thực sự cần predictability và DevTools mạnh.

---

## 7. Câu Hỏi Phỏng Vấn Tổng Hợp

### Câu 1: Nếu bắt đầu dự án mới, bạn chọn state management nào và tại sao?

**Trả lời mẫu (cho dự án medium-size SaaS):**

"Tôi sẽ dùng stack: **Zustand** cho global client state + **TanStack Query** cho server state. Lý do:
1. Zustand đủ mạnh cho 90% use cases, bundle nhỏ (~1KB), learning curve thấp, dễ onboard
2. TanStack Query xử lý server state tốt hơn bất kỳ giải pháp thủ công nào — caching, background refetch, optimistic updates
3. Không cần Redux trừ khi app thực sự phức tạp với nhiều team

Nếu là enterprise app với 10+ developer và cần audit trail (nhật ký kiểm toán) cho debugging, tôi sẽ xem xét Redux Toolkit + RTK Query."

### Câu 2: Server state khác client state như thế nào? Ảnh hưởng gì đến việc chọn thư viện?

**Trả lời:**

| | Server State | Client State |
|---|---|---|
| **Nguồn gốc** | Máy chủ | Client (trình duyệt) |
| **Tính bền vững** | Persist ở server | Có thể mất khi refresh |
| **Đồng bộ** | Có thể stale (cũ) | Luôn mới nhất |
| **Ownership** | Chia sẻ với server | Client sở hữu |

Vì server state có những thách thức riêng (caching, stale data, background sync), nên dùng **chuyên biệt tools** (TanStack Query, RTK Query) thay vì Redux/Zustand.

### Câu 3: Làm sao bạn xử lý authentication state trong React app?

**Trả lời:**

```
1. Lưu token trong httpOnly cookie (an toàn nhất) hoặc localStorage
2. Dùng Zustand persist hoặc Context API để lưu user object (sau khi verify token)
3. Tạo PrivateRoute/ProtectedRoute để redirect (chuyển hướng) nếu chưa đăng nhập
4. Refresh token tự động với interceptor (bộ chặn) trong axios/fetch wrapper
5. Logout: xóa token + clear state + redirect về login
```

### Câu 4: Khi nào bạn sẽ refactor từ Context API sang Zustand/Redux?

**Trả lời:**

Các dấu hiệu cần refactor:
- **Performance issues** (vấn đề hiệu năng): Nhiều component re-render không cần thiết dù đã optimize Context
- **Complex async logic**: Cần manage nhiều async operations với loading/error states riêng biệt
- **Debugging khó**: Không thể trace (theo dõi) state thay đổi
- **Team size tăng**: Nhiều developer cần convention (quy ước) rõ ràng hơn
- **State phức tạp**: Nhiều derived state, nhiều actions liên quan nhau

### Câu 5: Tại sao không nên dùng Redux cho mọi thứ?

**Trả lời:**

Redux phù hợp khi:
- State cần predictability cao (có thể dự đoán)
- Cần time-travel debugging
- Nhiều developer cần work together trên cùng state
- Cần middleware (logging, analytics)

Nhưng Redux không phù hợp cho:
- **Server state**: Dùng TanStack Query tốt hơn nhiều
- **Form state**: React Hook Form chuyên biệt hơn
- **Local UI state**: `useState` đơn giản hơn, không overhead
- **App nhỏ**: Chi phí setup không xứng đáng với lợi ích

---

## 📊 Tóm Tắt Cuối

```
Quyết định theo thứ tự ưu tiên:

1. LOCAL STATE (useState) — Ưu tiên đầu tiên
   └─ Chỉ 1 component cần? → useState

2. SHARED PARENT STATE (lifting state up — nâng state lên)
   └─ Vài component gần nhau cần? → props + lifting

3. GLOBAL CLIENT STATE
   ├─ Đơn giản, thay đổi ít → Context API
   └─ Phức tạp, thay đổi nhiều → Zustand (vừa) / Redux Toolkit (lớn)

4. SERVER STATE
   └─ Luôn dùng → TanStack Query hoặc RTK Query
      KHÔNG dùng Redux/Zustand cho server data

5. FORM STATE
   └─ React Hook Form + Zod

6. URL STATE
   └─ React Router (query params, route params)
```

---

## 🔗 Điều Hướng

- **Trước đó:** [5-jotai.md](./5-jotai.md) — Jotai
- **Quay lại:** [README.md](./README.md) — Tổng quan State Management
- **Tiếp theo trong lộ trình:** [04-routing/](../04-routing/) — Điều Hướng Trong React
- **Chỉ mục:** [INDEX.md](../INDEX.md)
