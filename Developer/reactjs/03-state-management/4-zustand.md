# Zustand — State Management Nhẹ, Đơn Giản, Hiện Đại

> **Zustand** (tiếng Đức nghĩa là "trạng thái") là thư viện state management (quản lý trạng thái) nhỏ gọn (~1KB), không có boilerplate, không cần Provider bọc. Đây là lựa chọn ngày càng phổ biến, đặc biệt cho các dự án không muốn sự phức tạp của Redux.

---

## 📌 Mục Lục

1. [Zustand Là Gì? Tại Sao Dùng?](#1-zustand-là-gì)
2. [Tạo Store Đơn Giản](#2-tạo-store)
3. [Patterns Phổ Biến](#3-patterns-phổ-biến)
4. [Middleware — Persist, DevTools, Immer](#4-middleware)
5. [Slice Pattern Cho Store Lớn](#5-slice-pattern)
6. [Ví Dụ Thực Tế — Shopping Cart](#6-ví-dụ-thực-tế)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Zustand Là Gì?

### Điểm Nổi Bật

- **Không cần Provider** — store là module JavaScript thông thường
- **Minimal API** (API tối giản) — chỉ cần `create()` để tạo store
- **Không có action types, reducers, selectors phức tạp** — chỉ là hàm
- **Tích hợp tốt với React và ngoài React** — có thể dùng trong vanilla JS
- **Bundle size cực nhỏ** — ~1KB gzipped

### So Sánh Nhanh

```jsx
// Redux Toolkit — cần boilerplate
const counterSlice = createSlice({...});
export const { increment } = counterSlice.actions;
export default counterSlice.reducer;
// configureStore, Provider, useSelector, useDispatch...

// Zustand — đơn giản hơn nhiều
const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));
// Dùng ngay: const { count, increment } = useCounterStore();
```

---

## 2. Tạo Store

```bash
npm install zustand
```

### Store Đơn Giản Nhất

```javascript
// src/store/counterStore.js
import { create } from "zustand";

const useCounterStore = create((set) => ({
  // State
  count: 0,

  // Actions (hành động) — hàm cập nhật state
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  incrementBy: (amount) => set((state) => ({ count: state.count + amount })),
  reset: () => set({ count: 0 }),
}));

export default useCounterStore;
```

### Sử Dụng Trong Component

```jsx
import useCounterStore from "../store/counterStore";

function Counter() {
  // Lấy toàn bộ store — component re-render khi bất kỳ giá trị nào thay đổi
  const { count, increment, decrement, reset } = useCounterStore();

  return (
    <div>
      <p>Đếm: {count}</p>
      <button onClick={decrement}>−</button>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Đặt lại</button>
    </div>
  );
}
```

### Chọn Có Chọn Lọc — Tránh Re-render Thừa

```jsx
// ✅ Chỉ đăng ký (subscribe) giá trị cần thiết
function CountDisplay() {
  // Component chỉ re-render khi count thay đổi, KHÔNG re-render khi actions thay đổi
  const count = useCounterStore((state) => state.count);
  return <span>{count}</span>;
}

function CounterButtons() {
  // Component chỉ lấy actions — KHÔNG bao giờ re-render (actions không thay đổi)
  const { increment, decrement } = useCounterStore((state) => ({
    increment: state.increment,
    decrement: state.decrement,
  }));

  return (
    <>
      <button onClick={decrement}>−</button>
      <button onClick={increment}>+</button>
    </>
  );
}
```

### Đọc State Ngoài Component (Không Dùng Hook)

```javascript
// Zustand cho phép đọc/ghi state bên ngoài React component
const count = useCounterStore.getState().count;
useCounterStore.setState({ count: 10 });

// Hữu ích trong: event handlers, utilities, tests
```

---

## 3. Patterns Phổ Biến

### Async Actions (Hành Động Bất Đồng Bộ)

```javascript
// src/store/userStore.js
import { create } from "zustand";

const useUserStore = create((set, get) => ({
  users: [],
  isLoading: false,
  error: null,

  // Async action trực tiếp trong store
  fetchUsers: async () => {
    set({ isLoading: true, error: null });
    try {
      const response = await fetch("/api/users");
      const users = await response.json();
      set({ users, isLoading: false });
    } catch (error) {
      set({ error: error.message, isLoading: false });
    }
  },

  createUser: async (userData) => {
    try {
      const response = await fetch("/api/users", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(userData),
      });
      const newUser = await response.json();
      // get() — đọc state hiện tại trong action
      set({ users: [newUser, ...get().users] });
      return newUser;
    } catch (error) {
      set({ error: error.message });
      throw error;
    }
  },

  deleteUser: async (id) => {
    // Optimistic update — xóa khỏi UI trước khi server xác nhận
    const previousUsers = get().users;
    set({ users: previousUsers.filter(u => u.id !== id) });

    try {
      await fetch(`/api/users/${id}`, { method: "DELETE" });
    } catch (error) {
      // Rollback nếu lỗi
      set({ users: previousUsers, error: error.message });
    }
  },
}));
```

### Computed Values (Giá Trị Tính Toán)

```javascript
// src/store/cartStore.js
import { create } from "zustand";

const useCartStore = create((set, get) => ({
  items: [],

  addItem: (product) => {
    const items = get().items;
    const existing = items.find(i => i.id === product.id);
    if (existing) {
      set({
        items: items.map(i =>
          i.id === product.id ? { ...i, qty: i.qty + 1 } : i
        ),
      });
    } else {
      set({ items: [...items, { ...product, qty: 1 }] });
    }
  },

  removeItem: (id) =>
    set({ items: get().items.filter(i => i.id !== id) }),

  updateQty: (id, qty) => {
    if (qty <= 0) {
      get().removeItem(id);
      return;
    }
    set({
      items: get().items.map(i => i.id === id ? { ...i, qty } : i),
    });
  },

  clearCart: () => set({ items: [] }),

  // Computed values — tính toán từ state
  get totalItems() {
    return get().items.reduce((sum, i) => sum + i.qty, 0);
  },

  get totalPrice() {
    return get().items.reduce((sum, i) => sum + i.price * i.qty, 0);
  },
}));
```

---

## 4. Middleware

### Persist Middleware — Lưu State Vào LocalStorage

```javascript
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: "light",
      language: "vi",
      fontSize: 14,

      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      setFontSize: (fontSize) => set({ fontSize }),
    }),
    {
      name: "app-settings",  // Key trong localStorage
      storage: createJSONStorage(() => localStorage),
      // Chỉ persist một số fields
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
      }),
    }
  )
);
```

### DevTools Middleware — Tích Hợp Redux DevTools

```javascript
import { create } from "zustand";
import { devtools } from "zustand/middleware";

const useCounterStore = create(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 }), false, "increment"),
      decrement: () => set((state) => ({ count: state.count - 1 }), false, "decrement"),
    }),
    { name: "Counter Store" }  // Tên hiển thị trong DevTools
  )
);
```

### Immer Middleware — Viết State Mutations Trực Quan

```javascript
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

const useTodosStore = create(
  immer((set) => ({
    todos: [],

    // Có thể viết "mutating" code như Redux Toolkit
    addTodo: (text) =>
      set((state) => {
        state.todos.push({ id: Date.now(), text, done: false });
      }),

    toggleTodo: (id) =>
      set((state) => {
        const todo = state.todos.find(t => t.id === id);
        if (todo) todo.done = !todo.done;
      }),

    deleteTodo: (id) =>
      set((state) => {
        state.todos = state.todos.filter(t => t.id !== id);
      }),
  }))
);
```

### Kết Hợp Nhiều Middleware

```javascript
import { create } from "zustand";
import { persist, devtools, immer } from "zustand/middleware";
import { immer as immerMiddleware } from "zustand/middleware/immer";

const useStore = create(
  devtools(
    persist(
      immerMiddleware((set) => ({
        // state và actions ở đây
      })),
      { name: "my-store" }
    ),
    { name: "My App Store" }
  )
);
```

---

## 5. Slice Pattern Cho Store Lớn

Khi app lớn, tách store thành các slice nhỏ:

```javascript
// src/store/slices/authSlice.js
const createAuthSlice = (set, get) => ({
  user: null,
  token: null,
  isAuthenticated: false,

  login: async (credentials) => {
    const { user, token } = await loginApi(credentials);
    set({ user, token, isAuthenticated: true });
    localStorage.setItem("token", token);
  },

  logout: () => {
    set({ user: null, token: null, isAuthenticated: false });
    localStorage.removeItem("token");
  },
});

// src/store/slices/uiSlice.js
const createUiSlice = (set) => ({
  isSidebarOpen: false,
  activeModal: null,

  toggleSidebar: () => set((state) => ({ isSidebarOpen: !state.isSidebarOpen })),
  openModal: (modalId) => set({ activeModal: modalId }),
  closeModal: () => set({ activeModal: null }),
});

// src/store/index.js
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";
import { createAuthSlice } from "./slices/authSlice";
import { createUiSlice } from "./slices/uiSlice";

const useStore = create(
  devtools(
    persist(
      (...args) => ({
        ...createAuthSlice(...args),
        ...createUiSlice(...args),
      }),
      {
        name: "app-store",
        partialize: (state) => ({ user: state.user, token: state.token }),
      }
    )
  )
);

// Các custom hooks cho từng slice
export const useAuth = () => useStore((state) => ({
  user: state.user,
  isAuthenticated: state.isAuthenticated,
  login: state.login,
  logout: state.logout,
}));

export const useUI = () => useStore((state) => ({
  isSidebarOpen: state.isSidebarOpen,
  activeModal: state.activeModal,
  toggleSidebar: state.toggleSidebar,
  openModal: state.openModal,
  closeModal: state.closeModal,
}));
```

---

## 6. Ví Dụ Thực Tế — Shopping Cart

```jsx
// src/store/cartStore.js
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useCartStore = create(
  persist(
    (set, get) => ({
      items: [],

      addItem: (product) => {
        const items = get().items;
        const existing = items.find(i => i.id === product.id);
        if (existing) {
          set({
            items: items.map(i =>
              i.id === product.id ? { ...i, qty: i.qty + 1 } : i
            ),
          });
        } else {
          set({ items: [...items, { ...product, qty: 1 }] });
        }
      },

      removeItem: (id) => set({ items: get().items.filter(i => i.id !== id) }),

      updateQty: (id, qty) => {
        if (qty < 1) { get().removeItem(id); return; }
        set({ items: get().items.map(i => i.id === id ? { ...i, qty } : i) });
      },

      clearCart: () => set({ items: [] }),
    }),
    { name: "shopping-cart" }
  )
);

// Selectors — tính toán từ state
export const selectCartTotal = (state) =>
  state.items.reduce((sum, i) => sum + i.price * i.qty, 0);

export const selectItemCount = (state) =>
  state.items.reduce((sum, i) => sum + i.qty, 0);


// src/components/CartIcon.jsx — chỉ hiển thị số lượng
function CartIcon() {
  const itemCount = useCartStore(selectItemCount);  // Chỉ re-render khi số lượng thay đổi
  return (
    <div>
      🛒 <span>{itemCount}</span>
    </div>
  );
}

// src/components/CartTotal.jsx — chỉ hiển thị tổng tiền
function CartTotal() {
  const total = useCartStore(selectCartTotal);
  return <p>Tổng: {total.toLocaleString("vi-VN")} VNĐ</p>;
}

// src/components/ProductCard.jsx
function ProductCard({ product }) {
  const addItem = useCartStore((state) => state.addItem);  // Không bao giờ re-render

  return (
    <div>
      <h3>{product.name}</h3>
      <p>{product.price.toLocaleString("vi-VN")} VNĐ</p>
      <button onClick={() => addItem(product)}>Thêm vào giỏ</button>
    </div>
  );
}

// src/components/CartPage.jsx
function CartPage() {
  const { items, updateQty, removeItem, clearCart } = useCartStore();
  const total = useCartStore(selectCartTotal);

  if (items.length === 0) return <p>Giỏ hàng trống</p>;

  return (
    <div>
      {items.map(item => (
        <div key={item.id}>
          <span>{item.name}</span>
          <input
            type="number"
            value={item.qty}
            min={1}
            onChange={(e) => updateQty(item.id, Number(e.target.value))}
          />
          <span>{(item.price * item.qty).toLocaleString("vi-VN")} VNĐ</span>
          <button onClick={() => removeItem(item.id)}>Xóa</button>
        </div>
      ))}
      <p>Tổng: {total.toLocaleString("vi-VN")} VNĐ</p>
      <button onClick={clearCart}>Xóa giỏ hàng</button>
    </div>
  );
}
```

---

## 7. Câu Hỏi Phỏng Vấn

### Câu 1: Zustand khác Redux như thế nào?

**Trả lời:**

| Khía Cạnh | Redux Toolkit | Zustand |
| --------- | ------------- | ------- |
| Setup | Provider, configureStore, slice | Chỉ `create()` |
| Boilerplate | Trung bình (RTK giảm) | Rất ít |
| Learning curve | Cao hơn | Thấp |
| Bundle size | ~11KB | ~1KB |
| DevTools | Xuất sắc (time-travel) | Tốt (qua middleware) |
| Phù hợp với | App enterprise, team lớn | App mọi quy mô |
| Cộng đồng | Lớn, nhiều tài liệu | Đang lớn nhanh |

### Câu 2: Làm thế nào để tránh re-render không cần thiết trong Zustand?

**Trả lời:**

Dùng **selector function** để chỉ subscribe giá trị cần thiết:

```jsx
// ❌ Subscribe toàn bộ store — re-render khi bất kỳ thứ gì thay đổi
const store = useMyStore();

// ✅ Subscribe chỉ count — re-render khi count thay đổi
const count = useMyStore((state) => state.count);

// ✅ Subscribe object với shallow comparison — re-render khi một trong các giá trị thay đổi
import { useShallow } from "zustand/react/shallow";
const { count, name } = useMyStore(useShallow((state) => ({
  count: state.count,
  name: state.name,
})));
```

### Câu 3: Khi nào dùng Zustand thay vì Context API?

**Trả lời:**
- **Dùng Zustand khi:**
  - State thay đổi thường xuyên (ví dụ: real-time updates, animation)
  - Cần truy cập state ngoài React component (utilities, event handlers)
  - Muốn DevTools để debug
  - Cần persist state vào localStorage đơn giản

- **Dùng Context API khi:**
  - App nhỏ, state ít thay đổi (theme, language)
  - Không muốn thêm dependency
  - Team không quen với Zustand

---

## 🔗 Điều Hướng

- **Tiếp theo:** [5-jotai.md](./5-jotai.md) — Jotai
- **Trước đó:** [3-rtk-query.md](./3-rtk-query.md) — RTK Query
- **Quay lại:** [README.md](./README.md) — Tổng quan State Management
