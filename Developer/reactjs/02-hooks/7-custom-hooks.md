# Custom Hooks — Tạo và Tái Sử Dụng Logic

> **Custom Hooks** (Hook Tùy Chỉnh) là hàm JavaScript bắt đầu bằng `use` và có thể gọi các hooks khác bên trong. Chúng cho phép **trích xuất stateful logic** (logic có trạng thái) ra khỏi component, tái sử dụng ở nhiều nơi mà không cần HOC hay Render Props. Đây là pattern quan trọng nhất để viết React code sạch, tái sử dụng được.

---

## 📌 Mục Lục

1. [Custom Hook Là Gì?](#1-custom-hook-là-gì)
2. [Custom Hooks Phổ Biến Từ Đầu](#2-custom-hooks-phổ-biến)
3. [Patterns Nâng Cao](#3-patterns-nâng-cao)
4. [Thư Viện Custom Hooks](#4-thư-viện)
5. [Nguyên Tắc Thiết Kế Custom Hook Tốt](#5-nguyên-tắc-thiết-kế)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Custom Hook Là Gì?

### Định Nghĩa

**Custom Hook** là hàm JavaScript thông thường với hai điều kiện:
1. **Tên bắt đầu bằng `use`** — quy ước bắt buộc để React và ESLint nhận biết
2. **Có thể gọi hooks bên trong** — `useState`, `useEffect`, và các hooks khác

```jsx
// ✅ Custom Hook hợp lệ
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);
  return { count, increment, decrement, reset };
}

// ❌ Không phải Custom Hook — tên không bắt đầu bằng use
function getCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue); // Lỗi nếu gọi trong hook!
}
```

### Tại Sao Cần Custom Hooks?

```jsx
// ❌ Trước Custom Hooks — Logic lặp lại ở nhiều component
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => { setUser(data); setLoading(false); })
      .catch(err => { setError(err); setLoading(false); });
  }, [userId]);

  // ...
}

function PostDetail({ postId }) {
  const [post, setPost] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(`/api/posts/${postId}`)
      .then(res => res.json())
      .then(data => { setPost(data); setLoading(false); })
      .catch(err => { setError(err); setLoading(false); });
  }, [postId]);

  // Cùng một logic! Lặp lại!
}
```

```jsx
// ✅ Sau Custom Hooks — Logic tập trung, tái sử dụng
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    const controller = new AbortController();

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => { setData(data); setLoading(false); })
      .catch(err => {
        if (err.name !== "AbortError") { setError(err); setLoading(false); }
      });

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Sử dụng lại ở bất kỳ đâu
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  // ...
}

function PostDetail({ postId }) {
  const { data: post, loading, error } = useFetch(`/api/posts/${postId}`);
  // ...
}
```

---

## 2. Custom Hooks Phổ Biến

### useLocalStorage — Đồng Bộ State Với localStorage

```jsx
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    if (typeof window === "undefined") return initialValue;
    try {
      const item = localStorage.getItem(key);
      return item !== null ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      if (typeof window !== "undefined") {
        localStorage.setItem(key, JSON.stringify(valueToStore));
      }
    } catch (error) {
      console.error(`useLocalStorage: Lỗi khi ghi key "${key}":`, error);
    }
  }, [key, storedValue]);

  const removeValue = useCallback(() => {
    localStorage.removeItem(key);
    setStoredValue(initialValue);
  }, [key, initialValue]);

  return [storedValue, setValue, removeValue];
}

// Sử dụng — y hệt useState nhưng persist qua sessions
function Settings() {
  const [theme, setTheme] = useLocalStorage("theme", "light");
  const [language, setLanguage] = useLocalStorage("language", "vi");

  return (
    <div>
      <select value={theme} onChange={e => setTheme(e.target.value)}>
        <option value="light">Sáng</option>
        <option value="dark">Tối</option>
      </select>
      <select value={language} onChange={e => setLanguage(e.target.value)}>
        <option value="vi">Tiếng Việt</option>
        <option value="en">English</option>
      </select>
    </div>
  );
}
```

### useDebounce — Trì Hoãn Giá Trị

```jsx
function useDebounce(value, delay = 300) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timeoutId = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timeoutId);
  }, [value, delay]);

  return debouncedValue;
}

// Sử dụng trong search
function SearchBar({ onSearch }) {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 400); // Chờ 400ms sau khi gõ xong

  useEffect(() => {
    if (debouncedQuery) {
      onSearch(debouncedQuery);
    }
  }, [debouncedQuery, onSearch]);

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Tìm kiếm..."
    />
  );
}
```

### useToggle — Quản Lý Boolean State

```jsx
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse, setValue };
}

// Sử dụng
function Modal() {
  const { value: isOpen, toggle, setFalse: close } = useToggle(false);

  return (
    <>
      <button onClick={toggle}>
        {isOpen ? "Đóng" : "Mở"} modal
      </button>
      {isOpen && (
        <div className="modal">
          <p>Nội dung modal</p>
          <button onClick={close}>Đóng</button>
        </div>
      )}
    </>
  );
}
```

### useMediaQuery — Responsive Logic Trong JavaScript

```jsx
function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => {
    if (typeof window === "undefined") return false;
    return window.matchMedia(query).matches;
  });

  useEffect(() => {
    const mediaQueryList = window.matchMedia(query);
    const listener = (event) => setMatches(event.matches);

    mediaQueryList.addEventListener("change", listener);
    return () => mediaQueryList.removeEventListener("change", listener);
  }, [query]);

  return matches;
}

// Hook convenience dựa trên useMediaQuery
function useBreakpoint() {
  const isMobile = useMediaQuery("(max-width: 767px)");
  const isTablet = useMediaQuery("(min-width: 768px) and (max-width: 1023px)");
  const isDesktop = useMediaQuery("(min-width: 1024px)");

  return { isMobile, isTablet, isDesktop };
}

function Navigation() {
  const { isMobile } = useBreakpoint();

  return isMobile ? <MobileNav /> : <DesktopNav />;
}
```

### useIntersectionObserver — Phát Hiện Phần Tử Trong Viewport

```jsx
function useIntersectionObserver(options = {}) {
  const [entry, setEntry] = useState(null);
  const elementRef = useRef(null);

  useEffect(() => {
    const element = elementRef.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => setEntry(entry),
      { threshold: 0.1, rootMargin: "0px", ...options }
    );

    observer.observe(element);
    return () => observer.disconnect();
  }, [options.threshold, options.rootMargin]);

  return { ref: elementRef, entry, isIntersecting: entry?.isIntersecting ?? false };
}

// Sử dụng cho lazy loading và animation on scroll
function AnimatedSection({ children }) {
  const { ref, isIntersecting } = useIntersectionObserver({ threshold: 0.2 });

  return (
    <div
      ref={ref}
      style={{
        opacity: isIntersecting ? 1 : 0,
        transform: isIntersecting ? "translateY(0)" : "translateY(30px)",
        transition: "opacity 0.6s, transform 0.6s",
      }}
    >
      {children}
    </div>
  );
}
```

### usePrevious — Lưu Giá Trị Trước Đó

```jsx
function usePrevious(value) {
  const ref = useRef(undefined);

  useEffect(() => {
    ref.current = value;
  }); // Không có deps → chạy sau mỗi render, lưu giá trị trước đó

  return ref.current;
}

// Sử dụng
function StockPrice({ price }) {
  const prevPrice = usePrevious(price);
  const change = prevPrice !== undefined ? price - prevPrice : 0;

  return (
    <div>
      <span>{price.toFixed(2)}</span>
      {change !== 0 && (
        <span style={{ color: change > 0 ? "green" : "red" }}>
          {change > 0 ? "▲" : "▼"} {Math.abs(change).toFixed(2)}
        </span>
      )}
    </div>
  );
}
```

---

## 3. Patterns Nâng Cao

### Pattern 1: Hook Trả Về Tuple vs Object

```jsx
// Tuple (Bộ Giá Trị) — khi chỉ có 2 giá trị, đặt tên linh hoạt
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  return [value, () => setValue(v => !v)]; // [value, toggle]
}
const [isOpen, toggleOpen] = useToggle(false);
const [isActive, toggleActive] = useToggle(true); // Đặt tên tự do

// Object — khi có nhiều hơn 2 giá trị, API rõ ràng hơn
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  return { data, loading, error, refetch: () => { /* ... */ } };
}
const { data, loading, error } = useFetch("/api/users"); // Destructure rõ ràng
```

### Pattern 2: Composing Hooks (Kết Hợp Nhiều Hooks)

```jsx
// Hook con — xử lý một concern (mối quan tâm) duy nhất
function useDataFetching(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let ignore = false;
    fetch(url)
      .then(res => res.json())
      .then(d => { if (!ignore) { setData(d); setLoading(false); } });
    return () => { ignore = true; };
  }, [url]);

  return { data, loading };
}

function usePagination(totalItems, itemsPerPage = 10) {
  const [page, setPage] = useState(1);
  const totalPages = Math.ceil(totalItems / itemsPerPage);

  return {
    page,
    totalPages,
    nextPage: () => setPage(p => Math.min(p + 1, totalPages)),
    prevPage: () => setPage(p => Math.max(p - 1, 1)),
    goToPage: setPage,
  };
}

// Hook tổng hợp — kết hợp nhiều concern
function usePaginatedData(baseUrl, itemsPerPage = 10) {
  const [total, setTotal] = useState(0);
  const { page, totalPages, nextPage, prevPage } = usePagination(total, itemsPerPage);

  const url = `${baseUrl}?page=${page}&limit=${itemsPerPage}`;
  const { data, loading } = useDataFetching(url);

  useEffect(() => {
    if (data?.total) setTotal(data.total);
  }, [data]);

  return {
    items: data?.items ?? [],
    loading,
    page,
    totalPages,
    nextPage,
    prevPage,
  };
}

// Sử dụng đơn giản
function UserList() {
  const { items, loading, page, totalPages, nextPage, prevPage } =
    usePaginatedData("/api/users", 20);

  if (loading) return <Spinner />;

  return (
    <div>
      <ul>{items.map(u => <li key={u.id}>{u.name}</li>)}</ul>
      <button onClick={prevPage} disabled={page === 1}>← Trước</button>
      <span>Trang {page}/{totalPages}</span>
      <button onClick={nextPage} disabled={page === totalPages}>Sau →</button>
    </div>
  );
}
```

### Pattern 3: Hook Factory (Nhà Máy Hook)

```jsx
// Tạo hook động dựa trên cấu hình
function createApiHook(endpoint) {
  return function useApiData(params) {
    const url = `${endpoint}?${new URLSearchParams(params)}`;
    return useFetch(url);
  };
}

// Tạo các hooks chuyên biệt
const useUsers = createApiHook("/api/users");
const usePosts = createApiHook("/api/posts");
const useProducts = createApiHook("/api/products");

// Sử dụng — mỗi hook có đúng một mục đích
function AdminDashboard() {
  const { data: users } = useUsers({ role: "admin", limit: 10 });
  const { data: posts } = usePosts({ status: "published" });
  return (/* ... */);
}
```

### Pattern 4: useReducer-Based Hook

```jsx
// Custom Hook quản lý state phức tạp với useReducer
function useCart() {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });

  const addItem = useCallback((product) => {
    dispatch({ type: "ADD_ITEM", payload: product });
  }, []);

  const removeItem = useCallback((productId) => {
    dispatch({ type: "REMOVE_ITEM", payload: { id: productId } });
  }, []);

  const updateQuantity = useCallback((productId, quantity) => {
    dispatch({ type: "UPDATE_QUANTITY", payload: { id: productId, quantity } });
  }, []);

  const clearCart = useCallback(() => {
    dispatch({ type: "CLEAR" });
  }, []);

  const itemCount = state.items.reduce((sum, item) => sum + item.quantity, 0);

  return {
    items: state.items,
    total: state.total,
    itemCount,
    addItem,
    removeItem,
    updateQuantity,
    clearCart,
  };
}
```

---

## 4. Thư Viện Custom Hooks

Trước khi tự viết, kiểm tra xem đã có solution sẵn chưa:

| Nhu Cầu | Thư Viện | Custom Hook |
| ------- | -------- | ----------- |
| Data fetching, caching | [TanStack Query](https://tanstack.com/query) | `useQuery`, `useMutation` |
| Form state & validation | [React Hook Form](https://react-hook-form.com/) | `useForm`, `useController` |
| Animations | [Framer Motion](https://www.framer.com/motion/) | `useAnimation`, `useMotionValue` |
| Clipboard | [usehooks-ts](https://usehooks-ts.com/) | `useClipboard` |
| Intersection Observer | [react-intersection-observer](https://github.com/thebuilder/react-intersection-observer) | `useInView` |
| Throttle/Debounce | [use-debounce](https://github.com/xnimorz/use-debounce) | `useDebounce` |
| Media queries | [react-responsive](https://github.com/contra/react-responsive) | `useMediaQuery` |
| Window size | [usehooks-ts](https://usehooks-ts.com/) | `useWindowSize` |
| Event listener | [usehooks-ts](https://usehooks-ts.com/) | `useEventListener` |

---

## 5. Nguyên Tắc Thiết Kế Custom Hook Tốt

### Nguyên Tắc 1: Single Responsibility (Trách Nhiệm Đơn)

```jsx
// ❌ Hook làm quá nhiều thứ
function useUserManagement() {
  // Fetch users
  // Filter users
  // Sort users
  // Handle pagination
  // Handle CRUD operations
  // Handle selection
}

// ✅ Mỗi hook một trách nhiệm
function useFetchUsers(params) { /* ... */ }
function useUserFilter(users) { /* ... */ }
function useUserSort(users) { /* ... */ }
function useUserSelection(users) { /* ... */ }
```

### Nguyên Tắc 2: API Rõ Ràng — Dễ Dùng Đúng, Khó Dùng Sai

```jsx
// ❌ API mơ hồ
function useData(arg1, arg2, arg3) { /* ... */ }

// ✅ API rõ ràng với named parameters
function useFetch(url, { method = "GET", headers = {}, skip = false } = {}) {
  // skip: boolean — bỏ qua fetch (dùng khi url chưa sẵn sàng)
}

const { data } = useFetch(`/api/users/${userId}`, {
  skip: !userId, // Chỉ fetch khi có userId
});
```

### Nguyên Tắc 3: Luôn Xử Lý Loading, Error, Empty States

```jsx
// ✅ Hook đầy đủ states
function useFetch(url) {
  const [state, setState] = useState({
    data: null,
    loading: true,
    error: null,
  });

  useEffect(() => {
    if (!url) {
      setState({ data: null, loading: false, error: null });
      return;
    }

    setState(prev => ({ ...prev, loading: true, error: null }));

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`${res.status} ${res.statusText}`);
        return res.json();
      })
      .then(data => setState({ data, loading: false, error: null }))
      .catch(error => setState({ data: null, loading: false, error }));
  }, [url]);

  return state; // { data, loading, error }
}
```

### Nguyên Tắc 4: Cho Phép Cleanup và Cancel

```jsx
// ✅ Hook có thể bị cancel
function useAsync(asyncFunction, dependencies = []) {
  const [state, setState] = useState({ loading: false, data: null, error: null });

  useEffect(() => {
    let cancelled = false;
    setState({ loading: true, data: null, error: null });

    asyncFunction()
      .then(data => {
        if (!cancelled) setState({ loading: false, data, error: null });
      })
      .catch(error => {
        if (!cancelled) setState({ loading: false, data: null, error });
      });

    return () => { cancelled = true; }; // Cleanup: cancel khi unmount/deps thay đổi
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, dependencies);

  return state;
}
```

### Nguyên Tắc 5: Document Bằng useDebugValue

```jsx
function useFetch(url) {
  const [state, setState] = useState({ data: null, loading: true, error: null });

  // Hiển thị trong React DevTools: "Fetching: /api/users" hoặc "Loaded: 25 items"
  useDebugValue(
    state,
    ({ loading, data, error }) => {
      if (loading) return `Đang tải: ${url}`;
      if (error) return `Lỗi: ${error.message}`;
      return `Đã tải: ${Array.isArray(data) ? `${data.length} items` : "1 item"}`;
    }
  );

  // ...
  return state;
}
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: Custom Hook khác gì với utility function (hàm tiện ích)?

**Trả lời:**
- **Utility function**: Hàm JavaScript thuần, không thể chứa hooks, không có state, không có lifecycle. Ví dụ: `formatDate()`, `calculateDiscount()`
- **Custom Hook**: Hàm có thể chứa `useState`, `useEffect` và các hooks khác. Mỗi component sử dụng Custom Hook sẽ có **isolated state** (state cô lập riêng). Hai component dùng cùng `useCounter()` sẽ có `count` riêng biệt, không chia sẻ.

### Q2: Custom Hook có chia sẻ state giữa các component không?

**Trả lời:** **Không** — Custom Hook chia sẻ **logic** (stateful logic), không chia sẻ **state**. Mỗi lần component gọi Custom Hook, Hook đó tạo ra một instance state riêng. Để chia sẻ state thực sự, cần Context hoặc external store (Zustand, Redux).

```jsx
function useCounter() {
  const [count, setCount] = useState(0); // Mỗi component có count riêng
  return { count, increment: () => setCount(c => c + 1) };
}

function ComponentA() {
  const { count } = useCounter(); // count = 0, riêng của A
}

function ComponentB() {
  const { count } = useCounter(); // count = 0, riêng của B — không liên quan A
}
```

### Q3: Khi nào nên tạo Custom Hook?

**Trả lời:** Tạo Custom Hook khi:
1. **Cùng một stateful logic xuất hiện ở 2+ component** — dấu hiệu rõ nhất để refactor
2. **Component quá phức tạp** — quá nhiều state/effect → tách ra để dễ đọc
3. **Muốn test logic độc lập** — Custom Hook là hàm thuần, test dễ hơn component
4. **Muốn chia sẻ với team hoặc publish** — đóng gói thành thư viện

**Không cần** Custom Hook khi logic đơn giản, chỉ dùng một lần.

### Q4: Làm thế nào để test Custom Hook?

**Trả lời:** Dùng `@testing-library/react` với `renderHook` và `act`:

```jsx
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

describe("useCounter", () => {
  it("khởi tạo với giá trị ban đầu", () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });

  it("increment tăng count lên 1", () => {
    const { result } = renderHook(() => useCounter(0));
    act(() => {
      result.current.increment();
    });
    expect(result.current.count).toBe(1);
  });
});
```

### Q5: Làm thế nào để tránh stale closure trong Custom Hook?

**Trả lời:** Stale Closure (Closure Lỗi Thời) trong Custom Hook xảy ra khi callback capture giá trị cũ. Giải pháp:
1. **Dùng functional update** `setState(prev => ...)` thay vì `setState(state + 1)`
2. **Thêm đầy đủ dependencies** vào useEffect
3. **Dùng useRef** để lưu callback luôn fresh:

```jsx
function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  // Luôn cập nhật ref với callback mới nhất
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Trước đó:** [6-react19-new-hooks.md](./6-react19-new-hooks.md) — React v19 Hooks Mới
- **Xem thêm:** [../03-state-management/](../03-state-management/) — Quản Lý State Toàn Cục
- **Xem thêm:** [../10-advanced-patterns/4-custom-hooks-patterns.md](../10-advanced-patterns/) — Custom Hooks Nâng Cao

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
