# useContext & useRef — Context và DOM Refs

> `useContext` cho phép component đọc và đăng ký nhận dữ liệu từ **Context** (Ngữ Cảnh) — giải pháp tránh **prop drilling** (truyền prop qua nhiều tầng). `useRef` tạo ra một tham chiếu có thể thay đổi mà **không kích hoạt re-render** — dùng để truy cập DOM elements hoặc lưu giá trị mutable (có thể thay đổi).

---

## 📌 Mục Lục

1. [useContext — Tiêu Thụ Context](#1-usecontext)
2. [Tạo và Cung Cấp Context](#2-tạo-và-cung-cấp-context)
3. [useContext Nâng Cao](#3-usecontext-nâng-cao)
4. [useRef — DOM Refs và Mutable Values](#4-useref)
5. [Các Pattern Phổ Biến Với useRef](#5-các-pattern-với-useref)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. useContext — Tiêu Thụ Context

### Vấn Đề Prop Drilling

**Prop drilling** (khoan prop qua nhiều tầng) xảy ra khi cần truyền dữ liệu từ component cha xuống component cháu chắt, phải đi qua nhiều tầng trung gian không cần dùng dữ liệu đó:

```jsx
// ❌ Prop Drilling — theme phải đi qua 3 tầng
function App() {
  const [theme, setTheme] = useState("light");
  return <Layout theme={theme} setTheme={setTheme} />;
}

function Layout({ theme, setTheme }) {
  // Layout không dùng theme, chỉ truyền xuống
  return <Sidebar theme={theme} setTheme={setTheme} />;
}

function Sidebar({ theme, setTheme }) {
  // Sidebar cũng không dùng, chỉ truyền tiếp
  return <ThemeToggle theme={theme} setTheme={setTheme} />;
}

function ThemeToggle({ theme, setTheme }) {
  // Đây mới là nơi cần dùng theme!
  return (
    <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
      Chế độ hiện tại: {theme}
    </button>
  );
}
```

### Giải Pháp — Context + useContext

```jsx
import { createContext, useContext, useState } from "react";

// Bước 1: Tạo Context
const ThemeContext = createContext("light"); // "light" là default value

// Bước 2: Cung cấp Context bằng Provider
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

// Bước 3: Tiêu thụ Context bằng useContext — ở bất kỳ tầng nào
function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);

  return (
    <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
      Chế độ hiện tại: {theme}
    </button>
  );
}

// Layout và Sidebar không cần nhận theme qua props nữa
function Layout() {
  return <Sidebar />;
}

function Sidebar() {
  return <ThemeToggle />;
}
```

---

## 2. Tạo Và Cung Cấp Context

### Cấu Trúc Context Đầy Đủ

Đây là pattern chuẩn để tạo và sử dụng Context:

```jsx
// auth-context.jsx — file riêng cho context
import { createContext, useContext, useState, useCallback } from "react";

// 1. Định nghĩa kiểu dữ liệu của context (với TypeScript)
// interface AuthContextType {
//   user: User | null;
//   login: (credentials: Credentials) => Promise<void>;
//   logout: () => void;
//   isLoading: boolean;
// }

// 2. Tạo context với default value = null (sẽ throw nếu dùng ngoài Provider)
const AuthContext = createContext(null);

// 3. Tạo Provider Component — component bao bọc
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(false);

  const login = useCallback(async ({ email, password }) => {
    setIsLoading(true);
    try {
      const response = await fetch("/api/auth/login", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, password }),
      });
      const userData = await response.json();
      setUser(userData);
    } finally {
      setIsLoading(false);
    }
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    localStorage.removeItem("token");
  }, []);

  const value = { user, login, logout, isLoading };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// 4. Custom Hook để tiêu thụ context — luôn tạo hook riêng
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth phải được dùng bên trong AuthProvider");
  }
  return context;
}
```

```jsx
// main.jsx — Bọc ứng dụng trong Provider
import { AuthProvider } from "./auth-context";

function App() {
  return (
    <AuthProvider>
      <Router>
        <Routes />
      </Router>
    </AuthProvider>
  );
}

// profile.jsx — Sử dụng ở bất kỳ đâu
function UserProfile() {
  const { user, logout } = useAuth(); // Đơn giản, không cần props

  if (!user) return <p>Chưa đăng nhập</p>;

  return (
    <div>
      <p>Xin chào, {user.name}!</p>
      <button onClick={logout}>Đăng xuất</button>
    </div>
  );
}
```

### Multiple Contexts (Nhiều Context)

```jsx
// Tổ chức nhiều provider — tránh "Provider Hell" bằng cách tạo AppProviders
function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <NotificationProvider>
          {children}
        </NotificationProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <Router />
    </AppProviders>
  );
}
```

---

## 3. useContext Nâng Cao

### Tối Ưu Re-render Với Context

Khi context value thay đổi, **tất cả** component đang tiêu thụ context đó đều re-render. Đây là vấn đề hiệu năng quan trọng:

```jsx
// ❌ Vấn đề: Mọi consumer re-render khi user HOẶC theme thay đổi
const AppContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState("light");

  return (
    <AppContext.Provider value={{ user, setUser, theme, setTheme }}>
      {children}
    </AppContext.Provider>
  );
}
```

```jsx
// ✅ Giải pháp: Tách thành nhiều context theo domain (lĩnh vực)
const UserContext = createContext(null);
const ThemeContext = createContext(null);

function AppProvider({ children }) {
  return (
    <UserProvider>
      <ThemeProvider>
        {children}
      </ThemeProvider>
    </UserProvider>
  );
}

// Component chỉ dùng theme → chỉ re-render khi theme thay đổi
function ThemeToggle() {
  const { theme } = useContext(ThemeContext);
  return <button>{theme}</button>;
}

// Component chỉ dùng user → chỉ re-render khi user thay đổi
function UserGreeting() {
  const { user } = useContext(UserContext);
  return <p>Xin chào {user?.name}</p>;
}
```

### Context Với useReducer — Pattern Mạnh Mẽ

Kết hợp `useContext` + `useReducer` để tạo global state management (quản lý trạng thái toàn cục) đơn giản:

```jsx
import { createContext, useContext, useReducer } from "react";

// Tách context thành State và Dispatch riêng biệt
const CartStateContext = createContext(null);
const CartDispatchContext = createContext(null);

function cartReducer(state, action) {
  switch (action.type) {
    case "ADD":
      return { ...state, items: [...state.items, action.payload], count: state.count + 1 };
    case "REMOVE":
      return { ...state, items: state.items.filter(i => i.id !== action.payload), count: state.count - 1 };
    case "CLEAR":
      return { items: [], count: 0 };
    default:
      return state;
  }
}

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], count: 0 });

  return (
    <CartStateContext.Provider value={state}>
      <CartDispatchContext.Provider value={dispatch}>
        {children}
      </CartDispatchContext.Provider>
    </CartStateContext.Provider>
  );
}

// Hook cho state — component chỉ dùng state không bị re-render do dispatch thay đổi
export const useCartState = () => useContext(CartStateContext);

// Hook cho dispatch — hàm dispatch không bao giờ thay đổi
export const useCartDispatch = () => useContext(CartDispatchContext);
```

---

## 4. useRef — DOM Refs và Mutable Values

### Cú Pháp

```jsx
const ref = useRef(initialValue);
// ref.current — giá trị hiện tại (có thể đọc và ghi)
// Thay đổi ref.current KHÔNG kích hoạt re-render
```

### Hai Công Dụng Chính của useRef

#### Công Dụng 1: Tham Chiếu DOM Elements

```jsx
import { useRef, useEffect } from "react";

function FocusInput() {
  const inputRef = useRef(null);

  function handleFocus() {
    // Truy cập DOM element trực tiếp
    inputRef.current.focus();
  }

  return (
    <div>
      {/* ref attribute kết nối với DOM element */}
      <input ref={inputRef} type="text" placeholder="Nhập văn bản..." />
      <button onClick={handleFocus}>Focus vào input</button>
    </div>
  );
}
```

```jsx
// Ví dụ thực tế — Video Player
function VideoPlayer({ src }) {
  const videoRef = useRef(null);
  const [isPlaying, setIsPlaying] = useState(false);

  function togglePlay() {
    if (isPlaying) {
      videoRef.current.pause();
    } else {
      videoRef.current.play();
    }
    setIsPlaying(!isPlaying);
  }

  return (
    <div>
      <video ref={videoRef} src={src} />
      <button onClick={togglePlay}>
        {isPlaying ? "⏸ Tạm dừng" : "▶ Phát"}
      </button>
    </div>
  );
}
```

#### Công Dụng 2: Lưu Mutable Value Không Gây Re-render

```jsx
// So sánh: useState vs useRef
function Timer() {
  const [count, setCount] = useState(0);    // Thay đổi → re-render
  const intervalRef = useRef(null);         // Thay đổi → KHÔNG re-render

  function startTimer() {
    intervalRef.current = setInterval(() => {
      setCount(prev => prev + 1);
    }, 1000);
  }

  function stopTimer() {
    clearInterval(intervalRef.current); // Truy cập timer ID đã lưu
    intervalRef.current = null;
  }

  useEffect(() => {
    return () => clearInterval(intervalRef.current); // Cleanup khi unmount
  }, []);

  return (
    <div>
      <p>Giây đã trôi qua: {count}</p>
      <button onClick={startTimer}>Bắt đầu</button>
      <button onClick={stopTimer}>Dừng</button>
    </div>
  );
}
```

---

## 5. Các Pattern Phổ Biến Với useRef

### Pattern 1: Lưu Previous Value (Giá Trị Trước Đó)

```jsx
function usePreviousValue(value) {
  const prevRef = useRef(undefined);

  useEffect(() => {
    prevRef.current = value; // Cập nhật sau mỗi render
  });

  return prevRef.current; // Trả về giá trị từ render trước
}

function PriceDisplay({ price }) {
  const prevPrice = usePreviousValue(price);

  const trend = prevPrice === undefined ? "—"
    : price > prevPrice ? "📈 Tăng"
    : price < prevPrice ? "📉 Giảm"
    : "Không đổi";

  return (
    <div>
      <p>Giá hiện tại: {price.toLocaleString()}đ</p>
      <p>Giá trước: {prevPrice?.toLocaleString() ?? "—"}đ</p>
      <p>Xu hướng: {trend}</p>
    </div>
  );
}
```

### Pattern 2: forwardRef — Chuyển Tiếp Ref

Khi cần truyền ref từ component cha vào DOM element bên trong component con:

```jsx
import { forwardRef, useRef } from "react";

// forwardRef — Bộ Chuyển Tiếp Ref — cho phép component con nhận ref từ cha
const CustomInput = forwardRef(function CustomInput({ label, ...props }, ref) {
  return (
    <div className="input-wrapper">
      <label>{label}</label>
      <input ref={ref} {...props} />
    </div>
  );
});

// Component cha dùng ref để điều khiển input bên trong CustomInput
function LoginForm() {
  const emailRef = useRef(null);
  const passwordRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    console.log("Email:", emailRef.current.value);
    console.log("Password:", passwordRef.current.value);
    emailRef.current.focus(); // Điều khiển DOM trực tiếp
  }

  return (
    <form onSubmit={handleSubmit}>
      <CustomInput ref={emailRef} label="Email" type="email" />
      <CustomInput ref={passwordRef} label="Mật khẩu" type="password" />
      <button type="submit">Đăng nhập</button>
    </form>
  );
}
```

> **Lưu ý React v19:** Trong React v19, `ref` có thể được truyền như prop thông thường mà **không cần** `forwardRef` nữa!
>
> ```jsx
> // React v19: ref là prop thông thường
> function CustomInput({ label, ref, ...props }) {
>   return (
>     <div>
>       <label>{label}</label>
>       <input ref={ref} {...props} />
>     </div>
>   );
> }
> ```

### Pattern 3: useImperativeHandle — Kiểm Soát API Của Ref

Cho phép tùy chỉnh những gì component cha có thể làm với ref của component con:

```jsx
import { forwardRef, useRef, useImperativeHandle } from "react";

const MediaPlayer = forwardRef(function MediaPlayer({ src }, ref) {
  const videoRef = useRef(null);

  // Chỉ expose những method cần thiết, không expose toàn bộ DOM
  useImperativeHandle(ref, () => ({
    play() { videoRef.current.play(); },
    pause() { videoRef.current.pause(); },
    seek(time) { videoRef.current.currentTime = time; },
    // videoRef.current.remove() — KHÔNG expose method nguy hiểm này
  }), []);

  return <video ref={videoRef} src={src} />;
});

// Cha chỉ có thể gọi play, pause, seek — không có gì khác
function PlayerPage() {
  const playerRef = useRef(null);

  return (
    <div>
      <MediaPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current.play()}>Phát</button>
      <button onClick={() => playerRef.current.pause()}>Dừng</button>
      <button onClick={() => playerRef.current.seek(60)}>Đến giây 60</button>
    </div>
  );
}
```

### Pattern 4: Tránh Stale Closures Trong Callbacks

```jsx
function SearchComponent({ onSearch }) {
  const [query, setQuery] = useState("");

  // Lưu onSearch trong ref để tránh dependency trong useEffect
  const onSearchRef = useRef(onSearch);
  useEffect(() => {
    onSearchRef.current = onSearch;
  }, [onSearch]);

  useEffect(() => {
    const timeoutId = setTimeout(() => {
      // Luôn dùng phiên bản mới nhất của onSearch
      onSearchRef.current(query);
    }, 400);
    return () => clearTimeout(timeoutId);
  }, [query]); // Không cần onSearch trong deps nhờ ref

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Tìm kiếm..."
    />
  );
}
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: Khi nào dùng Context thay vì prop drilling? Khi nào không nên dùng Context?

**Trả lời:**

**Nên dùng Context khi:**
- Dữ liệu cần chia sẻ trên nhiều tầng (theme, locale, user auth, shopping cart)
- Nhiều component ở nhiều nơi cần cùng dữ liệu
- Muốn tránh prop drilling qua 3+ tầng

**Không nên dùng Context khi:**
- Chỉ cần truyền xuống 1-2 tầng — prop đơn giản hơn
- Dữ liệu thay đổi rất thường xuyên với tần suất cao → dùng Zustand/Redux
- Cần performance cao — mỗi lần context thay đổi, tất cả consumers re-render

### Q2: Làm thế nào để tránh re-render không cần thiết khi dùng Context?

**Trả lời:**
1. **Tách context theo domain** — Không dùng một context cho tất cả
2. **Tách State context và Dispatch context** — Component chỉ đọc dispatch không re-render khi state thay đổi
3. **Memoize context value** — `useMemo(() => ({ user, login }), [user, login])` — tránh tạo object mới mỗi render
4. **Dùng `React.memo`** cho components tiêu thụ context nếu cần
5. **Cân nhắc dùng Zustand hoặc Jotai** khi context gây quá nhiều re-render

### Q3: useRef vs useState — khi nào dùng cái nào?

**Trả lời:**

| Tình huống | Dùng |
| ---------- | ---- |
| Giá trị ảnh hưởng đến UI cần hiển thị | `useState` |
| Giá trị KHÔNG cần hiển thị trên UI | `useRef` |
| ID của timer, interval | `useRef` |
| Tham chiếu DOM element | `useRef` |
| Previous value không cần render | `useRef` |
| Counter, text, boolean điều khiển UI | `useState` |

**Quy tắc:** Nếu thay đổi giá trị cần component re-render → `useState`. Nếu không cần re-render → `useRef`.

### Q4: ref.current có phải là state không? Tại sao thay đổi nó không gây re-render?

**Trả lời:** Không. `useRef` trả về một plain JavaScript object `{ current: value }`. React không theo dõi (track) `ref.current` trong rendering cycle. Khi bạn viết `ref.current = newValue`, đó là mutation trực tiếp — React không hay biết. Ngược lại, `useState` thông báo cho React biết cần re-render bằng cách schedule (lên lịch) một update qua React's reconciler.

### Q5: forwardRef là gì và React v19 thay đổi gì liên quan?

**Trả lời:** `forwardRef` là HOC (Higher-Order Component) cho phép component nhận `ref` từ parent và "chuyển tiếp" nó xuống DOM element hoặc component con bên trong. Cần thiết vì `ref` không phải prop thông thường — React xử lý nó đặc biệt. **Từ React v19**, `ref` được truyền như prop thông thường, không cần `forwardRef` nữa, giảm boilerplate đáng kể.

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Trước đó:** [2-useeffect-uselayout.md](./2-useeffect-uselayout.md) — useEffect và useLayoutEffect
- **Tiếp theo:** [4-usememo-usecallback.md](./4-usememo-usecallback.md) — useMemo và useCallback

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
