# Context API — Quản Lý Trạng Thái Tích Hợp Sẵn

> **Context API** (API Ngữ Cảnh) là giải pháp quản lý state toàn cục được tích hợp sẵn trong React, không cần cài thêm thư viện. Phù hợp cho các trạng thái chia sẻ như theme, ngôn ngữ, thông tin người dùng đăng nhập.

---

## 📌 Mục Lục

1. [Context API là gì?](#1-context-api-là-gì)
2. [Tạo và Sử Dụng Context](#2-tạo-và-sử-dụng-context)
3. [useContext Hook](#3-usecontext-hook)
4. [Context + useReducer — Pattern Nâng Cao](#4-context--usereducer)
5. [Tối Ưu Hiệu Năng Context](#5-tối-ưu-hiệu-năng-context)
6. [Ví Dụ Thực Tế — Theme & Auth](#6-ví-dụ-thực-tế)
7. [Hạn Chế Của Context API](#7-hạn-chế)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Context API Là Gì?

**Context API** cho phép truyền dữ liệu qua cây component mà không cần truyền props qua từng cấp trung gian — giải quyết vấn đề **prop drilling** (khoan truyền props).

### Vấn Đề Prop Drilling

```jsx
// ❌ Prop drilling — phải truyền user qua Layout và Sidebar dù chúng không dùng
function App() {
  const [user, setUser] = useState({ name: "Nguyễn An", role: "admin" });
  return <Layout user={user} />;
}

function Layout({ user }) {
  return <Sidebar user={user} />;  // Layout không dùng user nhưng vẫn phải truyền
}

function Sidebar({ user }) {
  return <UserAvatar user={user} />;  // Sidebar không dùng user nhưng vẫn phải truyền
}

function UserAvatar({ user }) {
  return <img src={user.avatar} alt={user.name} />;  // Chỉ đây mới dùng user
}
```

### Giải Pháp Với Context API

```jsx
// ✅ Context API — UserAvatar truy cập trực tiếp, không cần prop drilling
const UserContext = createContext(null);

function App() {
  const [user] = useState({ name: "Nguyễn An", role: "admin" });
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

function Layout() {
  return <Sidebar />;  // Không cần biết về user
}

function Sidebar() {
  return <UserAvatar />;  // Không cần biết về user
}

function UserAvatar() {
  const user = useContext(UserContext);  // Truy cập trực tiếp
  return <img src={user.avatar} alt={user.name} />;
}
```

---

## 2. Tạo Và Sử Dụng Context

### Các Bước Cơ Bản

```jsx
import { createContext, useContext, useState } from "react";

// Bước 1: Tạo Context với giá trị mặc định
const ThemeContext = createContext("light");

// Bước 2: Bọc cây component trong Provider (Nhà Cung Cấp)
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <MainContent />
    </ThemeContext.Provider>
  );
}

// Bước 3: Tiêu thụ Context trong component con bất kỳ
function ThemedButton() {
  const { theme, setTheme } = useContext(ThemeContext);

  return (
    <button
      style={{
        background: theme === "light" ? "#fff" : "#333",
        color: theme === "light" ? "#333" : "#fff",
      }}
      onClick={() => setTheme(theme === "light" ? "dark" : "light")}
    >
      Chuyển sang {theme === "light" ? "Dark" : "Light"} Mode
    </button>
  );
}
```

### Giá Trị Mặc Định Của createContext

```jsx
// Giá trị mặc định chỉ được dùng khi component KHÔNG có Provider bao bên ngoài
const UserContext = createContext({
  user: null,
  isLoggedIn: false,
  login: () => {},
  logout: () => {},
});

// Trường hợp có ích: testing — test component mà không cần Provider
function ProfilePage() {
  const { user } = useContext(UserContext);  // Dùng giá trị mặc định nếu không có Provider
  return <div>{user?.name ?? "Khách"}</div>;
}
```

---

## 3. useContext Hook

`useContext` là hook để đọc và subscribe (đăng ký nhận thay đổi) giá trị Context hiện tại.

```jsx
import { useContext } from "react";

function Component() {
  const value = useContext(SomeContext);
  // ...
}
```

### Khi Nào Component Re-render (Hiển Thị Lại)?

Component sẽ re-render **mỗi khi giá trị Context thay đổi**, kể cả khi nó chỉ dùng một phần nhỏ của giá trị đó.

```jsx
const AppContext = createContext(null);

function App() {
  const [count, setCount] = useState(0);
  const [user, setUser] = useState({ name: "An" });

  // ⚠️ Cả count và user được đóng gói vào 1 object mới mỗi lần render
  return (
    <AppContext.Provider value={{ count, setCount, user, setUser }}>
      <Counter />
      <UserProfile />
    </AppContext.Provider>
  );
}

function Counter() {
  const { count, setCount } = useContext(AppContext);
  // ⚠️ Re-render kể cả khi chỉ user thay đổi
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

---

## 4. Context + useReducer

Kết hợp Context với `useReducer` (hook giảm thiểu) tạo ra pattern quản lý state phức tạp mà không cần Redux.

```jsx
import { createContext, useContext, useReducer } from "react";

// --- Định nghĩa types và reducer ---

const initialState = {
  items: [],
  total: 0,
};

function cartReducer(state, action) {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find(i => i.id === action.payload.id);
      if (existing) {
        const items = state.items.map(i =>
          i.id === action.payload.id ? { ...i, qty: i.qty + 1 } : i
        );
        return { ...state, items, total: state.total + action.payload.price };
      }
      return {
        items: [...state.items, { ...action.payload, qty: 1 }],
        total: state.total + action.payload.price,
      };
    }
    case "REMOVE_ITEM": {
      const item = state.items.find(i => i.id === action.payload.id);
      return {
        items: state.items.filter(i => i.id !== action.payload.id),
        total: state.total - item.price * item.qty,
      };
    }
    case "CLEAR_CART":
      return initialState;
    default:
      return state;
  }
}

// --- Tạo Context ---

const CartContext = createContext(null);

// --- Provider component ---

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, initialState);

  // Bọc dispatch trong các action creators (hàm tạo hành động) để API rõ ràng hơn
  const addItem = (item) => dispatch({ type: "ADD_ITEM", payload: item });
  const removeItem = (id) => dispatch({ type: "REMOVE_ITEM", payload: { id } });
  const clearCart = () => dispatch({ type: "CLEAR_CART" });

  return (
    <CartContext.Provider value={{ ...state, addItem, removeItem, clearCart }}>
      {children}
    </CartContext.Provider>
  );
}

// --- Custom hook để tiêu thụ Context ---

export function useCart() {
  const context = useContext(CartContext);
  if (!context) {
    throw new Error("useCart phải được dùng bên trong CartProvider");
  }
  return context;
}

// --- Sử dụng ---

function ProductCard({ product }) {
  const { addItem } = useCart();
  return (
    <div>
      <h3>{product.name}</h3>
      <p>{product.price.toLocaleString("vi-VN")} VNĐ</p>
      <button onClick={() => addItem(product)}>Thêm vào giỏ</button>
    </div>
  );
}

function CartSummary() {
  const { items, total, removeItem, clearCart } = useCart();
  return (
    <div>
      <h2>Giỏ hàng ({items.length} sản phẩm)</h2>
      {items.map(item => (
        <div key={item.id}>
          <span>{item.name} x{item.qty}</span>
          <button onClick={() => removeItem(item.id)}>Xóa</button>
        </div>
      ))}
      <p>Tổng: {total.toLocaleString("vi-VN")} VNĐ</p>
      <button onClick={clearCart}>Xóa tất cả</button>
    </div>
  );
}
```

---

## 5. Tối Ưu Hiệu Năng Context

### Vấn Đề: Re-render Không Cần Thiết

```jsx
// ❌ Vấn đề: mọi thứ trong 1 Context — mọi component re-render khi bất kỳ thứ gì thay đổi
const AppContext = createContext(null);

function App() {
  const [theme, setTheme] = useState("light");
  const [user, setUser] = useState(null);
  const [notifications, setNotifications] = useState([]);

  return (
    <AppContext.Provider value={{ theme, setTheme, user, setUser, notifications, setNotifications }}>
      {/* Mọi component dùng AppContext đều re-render khi notifications thay đổi */}
      <Header />
      <Main />
    </AppContext.Provider>
  );
}
```

### Giải Pháp 1: Tách Context Theo Tần Suất Thay Đổi

```jsx
// ✅ Tách Context — mỗi phần thay đổi độc lập
const ThemeContext = createContext(null);        // ít thay đổi
const UserContext = createContext(null);          // thay đổi khi login/logout
const NotificationContext = createContext(null);  // thay đổi thường xuyên

function App() {
  const [theme, setTheme] = useState("light");
  const [user, setUser] = useState(null);
  const [notifications, setNotifications] = useState([]);

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <UserContext.Provider value={{ user, setUser }}>
        <NotificationContext.Provider value={{ notifications, setNotifications }}>
          <Header />
          <Main />
        </NotificationContext.Provider>
      </UserContext.Provider>
    </ThemeContext.Provider>
  );
}
```

### Giải Pháp 2: Tách State và Dispatch

Pattern phổ biến với useReducer — tách riêng giá trị state (ít re-render) và dispatch (không bao giờ thay đổi):

```jsx
const CountStateContext = createContext(null);
const CountDispatchContext = createContext(null);

function CounterProvider({ children }) {
  const [count, dispatch] = useReducer(reducer, 0);

  return (
    // dispatch không bao giờ thay đổi → component chỉ dùng dispatch không bị re-render khi count thay đổi
    <CountDispatchContext.Provider value={dispatch}>
      <CountStateContext.Provider value={count}>
        {children}
      </CountStateContext.Provider>
    </CountDispatchContext.Provider>
  );
}

// Component chỉ đọc state → re-render khi count thay đổi
function Counter() {
  const count = useContext(CountStateContext);
  return <span>{count}</span>;
}

// Component chỉ dispatch → KHÔNG re-render khi count thay đổi
function IncrementButton() {
  const dispatch = useContext(CountDispatchContext);
  return <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>;
}
```

### Giải Pháp 3: Dùng useMemo Cho Value

```jsx
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  // useMemo đảm bảo object value chỉ thay đổi khi theme thay đổi
  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}
```

---

## 6. Ví Dụ Thực Tế

### Theme Provider Hoàn Chỉnh

```jsx
import { createContext, useContext, useState, useMemo } from "react";

const ThemeContext = createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState(() => {
    // Đọc theme đã lưu từ localStorage (lưu trữ trình duyệt)
    return localStorage.getItem("theme") ?? "light";
  });

  const toggleTheme = () => {
    setTheme(prev => {
      const next = prev === "light" ? "dark" : "light";
      localStorage.setItem("theme", next);
      return next;
    });
  };

  const value = useMemo(() => ({ theme, toggleTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      <div data-theme={theme} className={`app theme-${theme}`}>
        {children}
      </div>
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error("useTheme phải được dùng trong ThemeProvider");
  return context;
}
```

### Auth Context (Ngữ Cảnh Xác Thực)

```jsx
import { createContext, useContext, useState, useEffect } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  // Kiểm tra session (phiên đăng nhập) khi app khởi động
  useEffect(() => {
    const token = localStorage.getItem("token");
    if (token) {
      fetchCurrentUser(token)
        .then(setUser)
        .catch(() => localStorage.removeItem("token"))
        .finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, []);

  const login = async (email, password) => {
    const { user, token } = await loginApi(email, password);
    localStorage.setItem("token", token);
    setUser(user);
  };

  const logout = () => {
    localStorage.removeItem("token");
    setUser(null);
  };

  if (isLoading) return <div>Đang tải...</div>;

  return (
    <AuthContext.Provider value={{ user, login, logout, isLoggedIn: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth phải được dùng trong AuthProvider");
  return ctx;
};

// Sử dụng
function Navbar() {
  const { user, logout, isLoggedIn } = useAuth();

  return (
    <nav>
      {isLoggedIn ? (
        <>
          <span>Xin chào, {user.name}</span>
          <button onClick={logout}>Đăng xuất</button>
        </>
      ) : (
        <a href="/login">Đăng nhập</a>
      )}
    </nav>
  );
}
```

---

## 7. Hạn Chế

| Hạn Chế | Giải Thích | Giải Pháp |
| ------- | ---------- | --------- |
| **Không có DevTools** | Khó debug state thay đổi theo thời gian | Dùng Redux Toolkit nếu cần debug phức tạp |
| **Re-render rộng** | Mọi consumer re-render khi value thay đổi | Tách Context, dùng useMemo |
| **Không phù hợp server state** | Không có cơ chế caching, fetching | Dùng TanStack Query hoặc RTK Query |
| **Không có middleware** | Không thể intercept (chặn) actions như Redux | Cần xử lý thủ công hoặc dùng Redux |
| **Hiệu năng kém với update thường xuyên** | Ví dụ: real-time data, counter tốc độ cao | Zustand hoặc Jotai phù hợp hơn |

### Khi Nào NÊN Dùng Context API

- ✅ Theme, ngôn ngữ (i18n — Internationalization — Quốc Tế Hóa)
- ✅ Thông tin user đã đăng nhập
- ✅ App nhỏ hoặc feature đơn giản
- ✅ Khi không muốn cài thêm thư viện
- ✅ Kết hợp với useReducer cho state phức tạp vừa phải

### Khi Nào KHÔNG NÊN Dùng Context API

- ❌ State thay đổi rất thường xuyên (mỗi giây, real-time)
- ❌ Cần time-travel debugging (debug theo thời gian)
- ❌ App lớn, nhiều team, cần predictability cao
- ❌ Server state (dữ liệu từ API) — dùng TanStack Query

---

## 8. Câu Hỏi Phỏng Vấn

### Câu 1: Context API khác gì so với prop drilling?

**Trả lời:**
- **Prop drilling:** Truyền dữ liệu từ component cha xuống con nhiều cấp, các component trung gian phải nhận và truyền tiếp props dù chúng không dùng.
- **Context API:** Tạo một "kho chứa" dữ liệu trung tâm, bất kỳ component con nào cũng có thể đọc trực tiếp mà không cần component cha truyền qua.

### Câu 2: Khi nào Context API gây re-render không cần thiết và cách tránh?

**Trả lời:**
Context API gây re-render mỗi khi giá trị trong Provider thay đổi, kể cả khi component chỉ dùng một phần nhỏ của giá trị đó.

Cách tránh:
1. Tách Context thành nhiều Context nhỏ hơn theo tần suất thay đổi
2. Tách state và dispatch thành 2 Context riêng
3. Dùng `useMemo` để bọc value object

### Câu 3: Context API có thể thay thế Redux không?

**Trả lời:**
Có thể trong một số trường hợp, nhưng không hoàn toàn:
- **Thay thế được:** App nhỏ–vừa, state ít thay đổi, không cần DevTools phức tạp
- **Không thay thế được:** App lớn cần time-travel debugging, middleware, complex async logic, hoặc cần DevTools mạnh
- Redux Toolkit hiện đại không còn boilerplate nhiều như Redux cũ — khoảng cách với Context + useReducer đã thu hẹp nhiều

### Câu 4: Tại sao nên tạo custom hook (hook tùy chỉnh) cho Context?

**Trả lời:**

```jsx
// ❌ Không có custom hook — component phải biết về SomeContext
function Component() {
  const value = useContext(SomeContext);
  if (!value) throw new Error("...");
  return <div>{value.data}</div>;
}

// ✅ Custom hook — ẩn chi tiết triển khai, có error handling (xử lý lỗi)
export function useSomeContext() {
  const context = useContext(SomeContext);
  if (!context) throw new Error("useSomeContext phải dùng trong SomeProvider");
  return context;
}

function Component() {
  const { data } = useSomeContext();  // Đơn giản hơn, kiểm tra null đã được xử lý
  return <div>{data}</div>;
}
```

Lợi ích: API sạch hơn, kiểm tra null ở một chỗ, dễ đổi implementation sau này.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-redux-toolkit.md](./2-redux-toolkit.md) — Redux Toolkit
- **Quay lại:** [README.md](./README.md) — Tổng quan State Management
- **Chỉ mục:** [INDEX.md](../INDEX.md)
