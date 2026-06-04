# 📋 Top 30 Câu Hỏi Phỏng Vấn React — Đáp Án Chi Tiết

> Bộ câu hỏi được chọn lọc từ thực tế phỏng vấn tại các công ty công nghệ, phủ đầy đủ từ Junior đến Senior React Developer.

---

## 🗂️ Mục Lục

1. [Fundamentals — Nền Tảng (Q1–Q10)](#fundamentals)
2. [Hooks (Q11–Q18)](#hooks)
3. [State & Data (Q19–Q22)](#state--data)
4. [Performance (Q23–Q26)](#performance)
5. [Advanced & Modern React (Q27–Q30)](#advanced--modern-react)

---

## 📘 Fundamentals — Nền Tảng {#fundamentals}

---

### Q1. Virtual DOM — DOM Ảo là gì? Tại sao React cần nó?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

Virtual DOM — DOM Ảo là một bản sao nhẹ (lightweight copy) của Real DOM — DOM Thật, được lưu trong bộ nhớ dưới dạng object JavaScript. React sử dụng nó để tối ưu quá trình cập nhật giao diện theo 3 bước:

```
1. Render — Kết Xuất:
   Khi state/props thay đổi, React tạo Virtual DOM mới.

2. Diffing — So Sánh:
   React so sánh Virtual DOM mới với Virtual DOM cũ
   (thuật toán này gọi là Reconciliation — Đối Chiếu).

3. Commit — Áp Dụng:
   Chỉ những phần thực sự thay đổi mới được cập nhật vào Real DOM.
```

**Tại sao cần?** Thao tác trực tiếp với Real DOM rất chậm. Virtual DOM cho phép React batch — gộp nhiều thao tác DOM lại thành một, tối thiểu hóa số lần reflow và repaint của trình duyệt.

**Trade-off:** Virtual DOM không phải lúc nào cũng nhanh hơn DOM manipulation thủ công. Với ứng dụng nhỏ, có thể không cần thiết. Svelte, Solid.js giải quyết khác: compile-time reactivity, không cần Virtual DOM.

---

### Q2. JSX là gì? Nó được biên dịch thành gì?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

JSX — JavaScript XML là cú pháp mở rộng (syntax extension) cho JavaScript, trông giống HTML nhưng thực ra là JavaScript. Babel — trình biên dịch JavaScript biên dịch JSX thành các lời gọi `React.createElement()`:

```jsx
// JSX — code bạn viết
const element = <h1 className="title">Xin chào</h1>;

// Được biên dịch thành (React 16 trở về trước)
const element = React.createElement(
  'h1',
  { className: 'title' },
  'Xin chào'
);

// React 17+ dùng JSX Transform mới (không cần import React)
import { jsx as _jsx } from 'react/jsx-runtime';
const element = _jsx('h1', { className: 'title', children: 'Xin chào' });
```

**Lưu ý:** JSX không phải HTML. Một số khác biệt quan trọng:
- `class` → `className`
- `for` → `htmlFor`
- Thuộc tính style nhận object: `style={{ color: 'red' }}`
- Tất cả thẻ phải đóng (self-closing): `<img />` không phải `<img>`

---

### Q3. Sự khác biệt giữa Props — Thuộc Tính và State — Trạng Thái?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

| Đặc Điểm | Props | State |
|-----------|-------|-------|
| Chủ sở hữu | Component cha truyền xuống | Component tự quản lý |
| Tính bất biến | Immutable — Bất biến (component con không sửa) | Mutable — Có thể thay đổi qua setter |
| Mục đích | Giao tiếp từ cha → con | Dữ liệu nội bộ, thay đổi theo thời gian |
| Trigger re-render | Khi cha re-render và truyền props mới | Khi gọi setState / setter |

```jsx
// Props — nhận từ ngoài, không sửa được
function Button({ label, onClick }) {
  return <button onClick={onClick}>{label}</button>;
}

// State — tự quản lý bên trong
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Nguyên tắc:** Dữ liệu chảy một chiều (one-way data flow) từ cha xuống con qua props. Con muốn "báo lên" cha thì dùng callback được truyền qua props.

---

### Q4. Tại sao cần `key` trong danh sách React?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

`key` là gợi ý (hint) để React xác định item nào trong danh sách đã thay đổi, được thêm vào, hay bị xóa — giúp Reconciliation — Thuật Toán Đối Chiếu hoạt động hiệu quả.

```jsx
// ❌ Sai — dùng index làm key (gây bug khi reorder)
{items.map((item, index) => (
  <ListItem key={index} item={item} />
))}

// ✅ Đúng — dùng ID ổn định
{items.map(item => (
  <ListItem key={item.id} item={item} />
))}
```

**Tại sao index sai?** Nếu list bị sort hoặc filter, index thay đổi nhưng data thì không → React nghĩ component A đã trở thành component B → state nội bộ bị lẫn lộn.

**Key phải:**
- Unique — Duy nhất trong cùng một danh sách (không cần unique toàn app)
- Stable — Ổn định (không thay đổi giữa các render)
- Predictable — Dự đoán được (không dùng `Math.random()`)

---

### Q5. Controlled vs Uncontrolled Components — Component Được Kiểm Soát vs Không Được Kiểm Soát là gì?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**Controlled Component — Component Được Kiểm Soát:** React là "source of truth — nguồn dữ liệu duy nhất" cho giá trị form. Mỗi thay đổi cập nhật state, mỗi state update cập nhật lại input.

```jsx
function ControlledForm() {
  const [value, setValue] = useState('');

  return (
    <input
      value={value}                        // React kiểm soát giá trị
      onChange={e => setValue(e.target.value)} // Mọi thay đổi đi qua setState
    />
  );
}
```

**Uncontrolled Component — Component Không Được Kiểm Soát:** DOM là source of truth. Dùng `ref` để đọc giá trị khi cần.

```jsx
function UncontrolledForm() {
  const inputRef = useRef(null);

  const handleSubmit = () => {
    console.log(inputRef.current.value); // Đọc khi submit
  };

  return <input ref={inputRef} defaultValue="mặc định" />;
}
```

| | Controlled | Uncontrolled |
|--|------------|--------------|
| Validation tức thời | ✅ Dễ dàng | ❌ Khó hơn |
| Ít code | ❌ Nhiều boilerplate | ✅ Đơn giản |
| Tích hợp thư viện | ✅ React Hook Form, Formik | ❌ Phức tạp |

---

### Q6. Reconciliation — Thuật Toán Đối Chiếu hoạt động như thế nào?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

Reconciliation — Thuật Toán Đối Chiếu là quá trình React so sánh Virtual DOM mới với Virtual DOM cũ để quyết định cập nhật gì vào Real DOM. React dùng thuật toán heuristic O(n) (thay vì O(n³) của tree diff thông thường) với hai giả định:

1. **Khác loại element → rebuild toàn bộ subtree:**
```jsx
// Từ <div> sang <span>: React unmount toàn bộ children của div,
// mount lại children mới cho span
```

2. **Key giúp nhận dạng item trong danh sách:**
```jsx
// Key ổn định → React biết item nào moved, nào added, nào removed
```

**React Fiber — Sợi Xử Lý (React 16+):** Kiến trúc Fiber cho phép React chia công việc Reconciliation thành các đơn vị nhỏ (units of work), có thể pause — tạm dừng, resume — tiếp tục, hoặc abort — hủy bỏ. Đây là nền tảng cho Concurrent Features — Tính Năng Đồng Thời.

---

### Q7. Prop Drilling — Truyền Props Qua Nhiều Tầng là gì? Cách giải quyết?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

Prop drilling xảy ra khi phải truyền props qua nhiều tầng component trung gian không thực sự cần dữ liệu đó:

```jsx
// ❌ Prop drilling: Button cần theme nhưng phải đi qua Page → Section
function App() {
  const theme = 'dark';
  return <Page theme={theme} />;
}
function Page({ theme }) {
  return <Section theme={theme} />;
}
function Section({ theme }) {
  return <Button theme={theme} />;
}
function Button({ theme }) {
  return <button className={theme}>Click</button>;
}
```

**Giải pháp:**

| Giải Pháp | Khi Dùng |
|-----------|----------|
| Context API | Dữ liệu global, ít thay đổi (theme, locale, user) |
| Redux Toolkit | State phức tạp, nhiều nơi đọc/ghi |
| Zustand | Dơn giản hơn Redux, global state nhẹ |
| Component composition | Tái cấu trúc component tree |

```jsx
// ✅ Context API giải quyết prop drilling
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page /> {/* Không cần truyền theme qua Page, Section */}
    </ThemeContext.Provider>
  );
}

function Button() {
  const theme = useContext(ThemeContext); // Đọc trực tiếp
  return <button className={theme}>Click</button>;
}
```

---

### Q8. Error Boundaries — Ranh Giới Lỗi là gì?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

Error Boundary — Ranh Giới Lỗi là class component bắt các JavaScript errors trong component tree con, log lỗi, và hiển thị UI fallback — giao diện dự phòng thay vì crash toàn bộ app.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    // Log lên error tracking (Sentry, Datadog...)
    logErrorToService(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return <h2>Có lỗi xảy ra. Vui lòng thử lại.</h2>;
    }
    return this.props.children;
  }
}

// Sử dụng
<ErrorBoundary>
  <ComponentCoThểBịLỗi />
</ErrorBoundary>
```

**Lưu ý quan trọng:** Error Boundaries KHÔNG bắt được:
- Lỗi trong event handlers (dùng try/catch thông thường)
- Lỗi trong async code (setTimeout, fetch...)
- Lỗi trong chính Error Boundary
- Lỗi trong Server-Side Rendering — SSR — Kết Xuất Phía Server

**React 19:** Có thể dùng `react-error-boundary` library hoặc hook mới để tránh dùng class component.

---

### Q9. React.memo là gì? Khác gì useMemo?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**React.memo:** HOC — Higher-Order Component — Component Bậc Cao ngăn một component re-render khi props không thay đổi (shallow comparison — so sánh nông).

```jsx
// Không có memo: Button re-render mỗi khi Parent re-render
const Button = ({ onClick, label }) => <button onClick={onClick}>{label}</button>;

// Có memo: Button chỉ re-render khi onClick hoặc label thay đổi
const Button = React.memo(({ onClick, label }) => (
  <button onClick={onClick}>{label}</button>
));
```

**useMemo:** Hook ghi nhớ kết quả tính toán, không phải component.

```jsx
// useMemo ghi nhớ giá trị
const sortedItems = useMemo(
  () => items.sort((a, b) => a.price - b.price),
  [items]
);
```

| | React.memo | useMemo |
|--|------------|---------|
| Áp dụng cho | Component | Giá trị / kết quả tính toán |
| Loại | HOC | Hook |
| Mục đích | Tránh re-render component | Tránh tính toán lại |

**Kết hợp:** React.memo + useCallback thường đi đôi với nhau:
```jsx
const handleClick = useCallback(() => { /* ... */ }, []);
// useCallback đảm bảo onClick reference ổn định → React.memo có tác dụng
<Button onClick={handleClick} label="Submit" />
```

---

### Q10. Strict Mode — Chế Độ Nghiêm Ngặt trong React làm gì?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

`React.StrictMode` là wrapper component giúp phát hiện vấn đề tiềm ẩn **chỉ trong môi trường development**:

```jsx
// Bật Strict Mode cho toàn app hoặc một phần
<React.StrictMode>
  <App />
</React.StrictMode>
```

**Strict Mode làm gì:**
1. **Double-invoking functions:** Gọi các function 2 lần (render, constructor, useState initializer...) để phát hiện side effects ngoài ý muốn
2. **Phát hiện deprecated APIs:** Cảnh báo dùng API cũ
3. **Phát hiện unexpected side effects:** useEffect cleanup issues
4. **React 18+:** Simulate mount → unmount → remount để kiểm tra component có cleanup đúng không

**Lưu ý:** Double invocation chỉ xảy ra trong development. Trong production, function chỉ chạy 1 lần.

---

## 🎣 Hooks {#hooks}

---

### Q11. useEffect dependency array hoạt động như thế nào?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

```jsx
// 1. Không có dependency array: chạy sau mỗi render
useEffect(() => { doSomething(); });

// 2. Mảng rỗng []: chạy 1 lần sau mount (componentDidMount)
useEffect(() => { fetchData(); }, []);

// 3. Có dependencies: chạy khi bất kỳ dependency nào thay đổi
useEffect(() => {
  fetchUser(userId);
}, [userId]); // Chạy lại khi userId thay đổi
```

**Cleanup function — Hàm Dọn Dẹp:**
```jsx
useEffect(() => {
  const subscription = subscribe(userId);
  
  // Cleanup: chạy trước khi effect chạy lại, và khi unmount
  return () => {
    subscription.unsubscribe();
  };
}, [userId]);
```

**Exhaustive deps — Danh sách dependencies đầy đủ:** ESLint rule `react-hooks/exhaustive-deps` cảnh báo khi thiếu dependency. Thường nên tuân theo rule này — nếu không cần một giá trị trong deps, hãy tái cấu trúc code.

---

### Q12. Stale Closure — Closure Cũ trong Hooks là gì?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

Stale closure xảy ra khi một function trong useEffect "bẫy" (capture) giá trị từ render cũ, không phản ánh giá trị hiện tại:

```jsx
// ❌ Bug: stale closure
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // count ở đây luôn là 0 (giá trị tại thời điểm mount)
      setCount(count + 1); // ❌ Stale!
    }, 1000);
    return () => clearInterval(id);
  }, []); // deps rỗng → closure bẫy count = 0

  return <div>{count}</div>;
}

// ✅ Fix 1: Functional updater (không cần đọc count)
setCount(prevCount => prevCount + 1);

// ✅ Fix 2: Thêm count vào deps (effect re-subscribe mỗi lần count thay đổi)
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);

// ✅ Fix 3: useRef để lưu giá trị mutable
const countRef = useRef(count);
countRef.current = count;
useEffect(() => {
  const id = setInterval(() => setCount(countRef.current + 1), 1000);
  return () => clearInterval(id);
}, []);
```

---

### Q13. Khi nào dùng useCallback? Khi nào không cần?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

`useCallback` ghi nhớ (memoize) một function, trả về cùng reference nếu dependencies không thay đổi.

**Dùng useCallback KHI:**
```jsx
// 1. Truyền function xuống component con được bọc bởi React.memo
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
<MemoizedButton onClick={handleClick} />

// 2. Function là dependency của useEffect khác
const fetchData = useCallback(async () => {
  const data = await api.get(url);
  setData(data);
}, [url]);

useEffect(() => {
  fetchData();
}, [fetchData]); // fetchData ổn định → effect không re-run vô lý
```

**KHÔNG cần useCallback KHI:**
```jsx
// Function chỉ dùng trong component hiện tại (không truyền xuống)
const handleChange = (e) => setValue(e.target.value); // Không cần memo

// Component con không được bọc bởi React.memo
<Button onClick={() => setCount(c => c + 1)} /> // Không hiệu quả nếu Button không memo
```

**Quy tắc:** useCallback chỉ có ý nghĩa khi component con dùng React.memo **hoặc** khi function là dep của hook khác.

---

### Q14. useReducer vs useState — Khi nào dùng gì?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

```jsx
// useState — đơn giản, giá trị độc lập
const [name, setName] = useState('');
const [age, setAge] = useState(0);

// useReducer — state phức tạp, nhiều fields liên quan, nhiều transition
const initialState = { name: '', age: 0, loading: false, error: null };

function reducer(state, action) {
  switch (action.type) {
    case 'SET_LOADING': return { ...state, loading: true, error: null };
    case 'SET_DATA':    return { ...state, ...action.payload, loading: false };
    case 'SET_ERROR':   return { ...state, error: action.error, loading: false };
    default:            return state;
  }
}

const [state, dispatch] = useReducer(reducer, initialState);
dispatch({ type: 'SET_LOADING' });
```

| Nên dùng useState | Nên dùng useReducer |
|-------------------|---------------------|
| Giá trị đơn giản (string, number, boolean) | State object phức tạp nhiều fields |
| State độc lập | Các state transitions liên quan nhau |
| Ít logic cập nhật | Logic cập nhật phức tạp |
| | Muốn dễ test reducer thuần |

---

### Q15. useRef dùng để làm gì ngoài tham chiếu DOM?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

`useRef` trả về object `{ current: value }` — giá trị **persist** qua các lần render nhưng **không trigger re-render** khi thay đổi. Khác `useState` ở điểm này.

**Use case 1: Tham chiếu DOM**
```jsx
const inputRef = useRef(null);
useEffect(() => { inputRef.current.focus(); }, []);
return <input ref={inputRef} />;
```

**Use case 2: Lưu giá trị mutable không cần re-render**
```jsx
const timerRef = useRef(null);

const startTimer = () => {
  timerRef.current = setInterval(() => tick(), 1000);
};
const stopTimer = () => {
  clearInterval(timerRef.current);
};
```

**Use case 3: Lưu giá trị previous**
```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value; // Lưu sau mỗi render
  });
  return ref.current; // Trả về giá trị từ render trước
}
```

**Use case 4: Tránh stale closure** (xem Q12)

---

### Q16. Rules of Hooks — Quy Tắc Hooks là gì? Tại sao cần tuân thủ?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**Quy tắc 1: Chỉ gọi Hooks ở top level — cấp cao nhất**
```jsx
// ❌ Sai — hook trong điều kiện
function Component({ isLoggedIn }) {
  if (isLoggedIn) {
    const [data, setData] = useState(null); // Hook trong if — sai!
  }
}

// ✅ Đúng — luôn gọi hook, dùng điều kiện bên trong
function Component({ isLoggedIn }) {
  const [data, setData] = useState(null); // Luôn gọi
  // Điều kiện ở bên trong logic
}
```

**Quy tắc 2: Chỉ gọi Hooks trong React functions**
- ✅ Functional components
- ✅ Custom hooks
- ❌ Regular JavaScript functions
- ❌ Class components

**Tại sao?** React theo dõi thứ tự gọi hooks để liên kết state đúng với hook. Nếu thứ tự thay đổi (do if/else), React sẽ nhầm lẫn state của hook này với hook khác.

```
Render 1:  useState(0) → hook[0], useEffect → hook[1], useState('') → hook[2]
Render 2 (nếu if false):  useState(0) → hook[0], useState('') → hook[1] ← Nhầm!
```

---

### Q17. Custom Hooks — Hook Tùy Chỉnh là gì? Viết ví dụ thực tế.

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

Custom Hook là JavaScript function bắt đầu bằng `use`, có thể gọi các built-in hooks khác bên trong. Dùng để **trích xuất và tái sử dụng stateful logic**.

```jsx
// Custom hook: useLocalStorage — Lưu trữ Cục Bộ
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
}

// Sử dụng
const [theme, setTheme] = useLocalStorage('theme', 'light');
```

```jsx
// Custom hook: useDebounce — Trì Hoãn
function useDebounce(value, delay = 300) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Sử dụng
const debouncedSearch = useDebounce(searchTerm, 500);
useEffect(() => { fetchResults(debouncedSearch); }, [debouncedSearch]);
```

---

### Q18. useLayoutEffect vs useEffect — Khác nhau như thế nào?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

| | useEffect | useLayoutEffect |
|--|-----------|-----------------|
| Thời điểm chạy | Sau khi browser đã paint | Sau DOM mutation, **trước** khi browser paint |
| Blocking — Chặn paint | Không | Có (đồng bộ) |
| Use case | Fetch data, subscriptions, timers | Đọc layout DOM, animation, tooltip positioning |

```jsx
// useLayoutEffect: đọc kích thước DOM trước khi user thấy UI nhảy
function Tooltip({ targetRef, children }) {
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const tooltipRef = useRef(null);

  useLayoutEffect(() => {
    // Đo lường DOM và set position TRƯỚC khi browser paint
    // → User không thấy tooltip bị "nhảy"
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({ top: rect.bottom, left: rect.left });
  }, [targetRef]);

  return (
    <div ref={tooltipRef} style={position}>
      {children}
    </div>
  );
}
```

**Quy tắc:** Mặc định dùng `useEffect`. Chỉ dùng `useLayoutEffect` khi cần đọc layout và cập nhật DOM trước khi user thấy — nếu không sẽ thấy "flash" — nhấp nháy.

---

## 🗃️ State & Data {#state--data}

---

### Q19. Context API — Khi nào phù hợp, khi nào không?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**Phù hợp với Context API:**
- Dữ liệu global, ít thay đổi: theme, language, current user, feature flags
- Tránh prop drilling qua 3+ tầng

**Không phù hợp:**
- State thay đổi thường xuyên (mỗi keystroke, animation) → gây re-render toàn bộ consumers
- Logic phức tạp, nhiều slice state → khó manage

```jsx
// ✅ Tách Context để tối ưu re-render
// Nếu gộp tất cả vào một context, mọi component consume sẽ re-render
// khi bất kỳ phần nào thay đổi

const ThemeContext = createContext(); // Ít thay đổi
const UserContext = createContext();  // Ít thay đổi
const CartContext = createContext();  // Có thể thay đổi nhiều hơn

// Tốt hơn: CartContext nên dùng Zustand hoặc Redux nếu phức tạp
```

**Tối ưu re-render với Context:**
```jsx
// Memo children để tránh re-render không cần thiết
const value = useMemo(() => ({ user, login, logout }), [user]);
<UserContext.Provider value={value}>
  {children}
</UserContext.Provider>
```

---

### Q20. Redux Toolkit vs Zustand — Khi nào chọn gì?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

| Tiêu Chí | Redux Toolkit (RTK) | Zustand |
|----------|---------------------|---------|
| Boilerplate | Vừa (ít hơn Redux thuần) | Rất ít |
| DevTools | ✅ Redux DevTools mạnh mẽ | ✅ DevTools hỗ trợ |
| Middleware | ✅ Phong phú (Thunk, Saga) | Middleware tự tạo |
| TypeScript | ✅ Rất tốt | ✅ Rất tốt |
| Learning curve | Vừa | Thấp |
| Phù hợp | App lớn, team lớn, cần audit trail | App nhỏ-vừa, prototype, cần nhanh |
| Data fetching | RTK Query tích hợp sẵn | Kết hợp TanStack Query |

```js
// Zustand — cực kỳ đơn giản
const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// RTK — chuẩn hóa, có nhiều built-in utilities
const counterSlice = createSlice({
  name: 'counter',
  initialState: { count: 0 },
  reducers: {
    increment: (state) => { state.count += 1; }, // Immer — tự deep clone
  },
});
```

**Chọn RTK khi:** Enterprise app, cần middleware phức tạp, team lớn cần conventions rõ ràng, đã dùng Redux.

**Chọn Zustand khi:** App vừa-nhỏ, muốn đơn giản, prototype nhanh.

---

### Q21. TanStack Query — React Query giải quyết vấn đề gì?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

TanStack Query quản lý **server state** — trạng thái từ server (khác client state). Nó giải quyết:

1. **Caching — Lưu Đệm:** Không fetch lại nếu data còn fresh — tươi
2. **Background refetching — Tự Động Làm Mới:** Tự cập nhật khi window focus, interval...
3. **Loading/Error states — Trạng Thái Tải/Lỗi:** Built-in, không cần tự quản lý
4. **Optimistic Updates — Cập Nhật Lạc Quan:** Cập nhật UI trước khi server confirm

```jsx
// Fetching data — Lấy dữ liệu
const { data, isLoading, error, refetch } = useQuery({
  queryKey: ['users', userId],    // Cache key — Khóa bộ đệm
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000,      // Data fresh trong 5 phút
  gcTime: 10 * 60 * 1000,        // Giữ trong cache 10 phút sau khi unused
});

// Mutation với optimistic update
const { mutate } = useMutation({
  mutationFn: updateUser,
  onMutate: async (newUser) => {
    await queryClient.cancelQueries(['users', userId]);
    const previous = queryClient.getQueryData(['users', userId]);
    queryClient.setQueryData(['users', userId], newUser); // Cập nhật lạc quan
    return { previous };
  },
  onError: (err, newUser, context) => {
    queryClient.setQueryData(['users', userId], context.previous); // Rollback
  },
  onSettled: () => queryClient.invalidateQueries(['users', userId]),
});
```

---

### Q22. Server State vs Client State — Khi nào dùng thư viện nào?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

| Loại State | Ví Dụ | Công Cụ Phù Hợp |
|------------|-------|-----------------|
| **Server State** — từ server | User data, product list, API responses | TanStack Query, SWR, RTK Query |
| **Client State** — của UI | Modal open/close, tab active, form draft | useState, Zustand, Redux |
| **URL State** — trong URL | Filter, pagination, search query | React Router searchParams |
| **Form State** — của form | Input values, validation errors | React Hook Form, Formik |

**Sai lầm phổ biến:** Dùng Redux để lưu server data → phải tự quản lý cache invalidation, loading state, error state. TanStack Query làm điều này tốt hơn nhiều.

---

## ⚡ Performance {#performance}

---

### Q23. Khi nào React.memo thực sự có ích? Khi nào gây hại?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**React.memo CÓ ÍCH khi:**
```jsx
// 1. Component render tốn kém + parent re-render thường xuyên
const ExpensiveChart = React.memo(({ data }) => {
  // Tính toán nặng bên trong
  return <Chart data={data} />;
});

// 2. Kết hợp với useCallback để giữ reference ổn định
const handleSort = useCallback((field) => { /* ... */ }, []);
<ExpensiveTable onSort={handleSort} data={items} />
```

**React.memo KHÔNG GIÚP (hoặc GÂY HẠI) khi:**
```jsx
// 1. Props chứa object/array literals (reference mới mỗi render)
<MemoComponent config={{ theme: 'dark' }} /> // ❌ Config mới mỗi render!
<MemoComponent items={[...products]} />       // ❌ Array mới mỗi render!

// 2. Component nhẹ — overhead của memo > lợi ích
const SimpleText = React.memo(({ text }) => <span>{text}</span>); // Không cần

// 3. Props thay đổi gần như mỗi render → memo luôn fail
```

**Quy trình đúng:** Profile trước → xác định bottleneck → thêm memo nếu cần. Đừng thêm memo mù quáng.

---

### Q24. Code Splitting — Tách Code với React.lazy là gì?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

Code splitting chia bundle thành các chunks nhỏ, chỉ tải khi cần — giảm initial load time — thời gian tải ban đầu.

```jsx
// ❌ Không split: tất cả vào một bundle lớn
import AdminDashboard from './AdminDashboard'; // Luôn tải dù user không vào admin

// ✅ Lazy loading — Tải Lười: chỉ tải khi cần
const AdminDashboard = React.lazy(() => import('./AdminDashboard'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/admin" element={<AdminDashboard />} />
      </Routes>
    </Suspense>
  );
}
```

**Chiến lược split:**
1. **Route-based splitting — Tách theo Route:** Mỗi route là một chunk → phổ biến nhất
2. **Component-based splitting:** Modal, drawer, tab content nặng
3. **Vendor splitting:** Thư viện lớn (chart.js, lodash) vào chunk riêng

---

### Q25. startTransition và useDeferredValue giải quyết vấn đề gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

Cả hai là Concurrent Features — Tính Năng Đồng Thời giúp React ưu tiên updates quan trọng hơn updates ít khẩn cấp.

**startTransition — Bắt Đầu Chuyển Tiếp:**
```jsx
import { startTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    setQuery(e.target.value); // Urgent — Khẩn cấp: cập nhật input ngay

    startTransition(() => {
      // Non-urgent — Không khẩn cấp: có thể trì hoãn nếu có input mới
      setResults(filterItems(e.target.value));
    });
  };
}
```

**useDeferredValue — Giá Trị Trì Hoãn:**
```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  // deferredQuery lag sau query → hiển thị kết quả cũ trong khi tính toán mới
  
  const results = useMemo(
    () => heavyFilter(deferredQuery),
    [deferredQuery]
  );

  return (
    <div style={{ opacity: query !== deferredQuery ? 0.5 : 1 }}>
      {results.map(item => <Item key={item.id} item={item} />)}
    </div>
  );
}
```

**Khác nhau:**
- `startTransition`: Bạn kiểm soát cái gì là "transition" — transition
- `useDeferredValue`: React tự quyết định khi nào update value

---

### Q26. Làm thế nào để debug và tối ưu hiệu năng React?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**Quy trình chuẩn: Measure → Identify → Fix → Verify**

```
Bước 1: MEASURE — ĐO
   React DevTools Profiler → Record → Interact → Stop
   Xem flame chart: bar dài = render chậm, màu đỏ/vàng = cần tối ưu

Bước 2: IDENTIFY — XÁC ĐỊNH
   - Component nào render lâu nhất?
   - Component nào render nhiều lần không cần thiết?
   - "Why did this render?" trong Profiler Settings

Bước 3: FIX — SỬA
   - Thêm React.memo nếu parent re-render làm component nặng re-render
   - Thêm useCallback/useMemo nếu reference không ổn định
   - Virtualize list nếu danh sách dài (react-window)
   - Code split nếu bundle quá lớn

Bước 4: VERIFY — XÁC NHẬN
   Profile lại sau khi sửa để đo lường cải thiện
```

**Tools:** React DevTools Profiler, Chrome Performance tab, Lighthouse, `why-did-you-render` library.

---

## 🚀 Advanced & Modern React {#advanced--modern-react}

---

### Q27. React Server Components — RSC — Component Phía Server là gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

RSC — React Server Components là components chạy hoàn toàn trên server, không có JavaScript gửi xuống client. Khác với SSR — Server-Side Rendering — Kết Xuất Phía Server:

| | RSC | SSR | Client Components |
|--|-----|-----|-------------------|
| Render ở đâu | Server only | Server + Hydrate client | Client only |
| JavaScript gửi client | Không | Có (hydration) | Có |
| Access server resources | ✅ (DB, file system) | ✅ | ❌ |
| useState, useEffect | ❌ | ✅ (sau hydrate) | ✅ |
| Event handlers | ❌ | ✅ | ✅ |

```jsx
// app/page.tsx (Next.js App Router) — Server Component mặc định
async function ProductPage({ params }) {
  // Truy cập DB trực tiếp — không qua API!
  const product = await db.product.findUnique({ where: { id: params.id } });

  return (
    <div>
      <h1>{product.name}</h1>
      {/* Client Component phải dùng 'use client' */}
      <AddToCartButton productId={product.id} />
    </div>
  );
}

// components/AddToCartButton.tsx
'use client'; // Directive đánh dấu là Client Component
function AddToCartButton({ productId }) {
  const [added, setAdded] = useState(false);
  return <button onClick={() => setAdded(true)}>Add to Cart</button>;
}
```

---

### Q28. Server Actions — Hành Động Phía Server trong React v19 là gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

Server Actions cho phép gọi server-side functions trực tiếp từ Client Components, không cần tạo API endpoint riêng.

```jsx
// actions.ts — Server Action
'use server';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Trực tiếp ghi DB — không cần REST API
  const user = await db.user.create({ data: { name, email } });
  revalidatePath('/users'); // Invalidate cache — Làm Mới Cache
  return { success: true, user };
}

// Dùng trong Client Component hoặc Server Component
function UserForm() {
  return (
    <form action={createUser}> {/* React tự handle form submission */}
      <input name="name" />
      <input name="email" />
      <button type="submit">Tạo User</button>
    </form>
  );
}

// Dùng với useActionState (React v19)
const [state, action, isPending] = useActionState(createUser, null);
```

---

### Q29. React Compiler — Trình Biên Dịch React là gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

React Compiler (tên dự án trước đây: React Forget) là trình biên dịch tự động thêm memoization — ghi nhớ vào code React tại compile time — thời gian biên dịch, thay vì yêu cầu developer tự thêm `useMemo`, `useCallback`, `React.memo`.

**Trước khi có Compiler:**
```jsx
// Developer phải tự thêm memoization
const MyComponent = React.memo(({ onClick, items }) => {
  const processed = useMemo(() => heavyProcess(items), [items]);
  const handler = useCallback(() => onClick(processed), [onClick, processed]);
  return <List items={processed} onSelect={handler} />;
});
```

**Với React Compiler:**
```jsx
// Viết code tự nhiên — Compiler tự thêm optimizations
function MyComponent({ onClick, items }) {
  const processed = heavyProcess(items);
  const handler = () => onClick(processed);
  return <List items={processed} onSelect={handler} />;
}
// Compiler tự phân tích dependencies và thêm memo phù hợp
```

**Điều kiện:** Code phải tuân thủ Rules of React (pure functions, immutable state). Compiler phát hiện violations và bỏ qua optimize những phần không an toàn.

---

### Q30. Micro-frontends — Vi Giao Diện là gì? Khi nào nên áp dụng?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

Micro-frontends áp dụng triết lý microservices — vi dịch vụ vào frontend: chia ứng dụng thành các phần nhỏ độc lập, mỗi phần có thể được phát triển và deploy riêng bởi các team khác nhau.

**Khi NÊN áp dụng:**
- Tổ chức lớn (10+ frontend developers)
- Nhiều team độc lập cần deploy riêng
- Codebase monolith frontend quá lớn, khó maintain

**Khi KHÔNG NÊN:**
- Team nhỏ (< 5 devs) — overhead không xứng đáng
- App đơn giản
- Cần consistency chặt chẽ giữa các phần

**Các chiến lược implement:**
```
1. Module Federation (Webpack 5):
   Mỗi micro-frontend là Webpack remote module,
   share dependencies qua host app.

2. iframes — Khung Nhúng:
   Đơn giản nhất, isolation tốt nhất, nhưng UX kém.

3. Web Components — Component Web:
   Framework-agnostic, dùng Custom Elements.

4. Single-SPA:
   Framework phổ biến cho micro-frontends.
```

**Trade-offs:**
- ✅ Team independence — Độc lập team, deploy riêng
- ✅ Technology diversity — Mỗi phần dùng tech khác nhau
- ❌ Phức tạp về shared state, authentication, routing
- ❌ Bundle duplication nếu không share dependencies
- ❌ Performance: nhiều bundles tải riêng biệt

---

## 📊 Bảng Tổng Hợp 30 Câu Hỏi

| # | Câu Hỏi | Cấp Độ | Tần Suất |
|---|---------|--------|----------|
| Q1 | Virtual DOM là gì? | Junior | ⭐⭐⭐⭐⭐ |
| Q2 | JSX được biên dịch thành gì? | Junior | ⭐⭐⭐⭐ |
| Q3 | Props vs State | Junior | ⭐⭐⭐⭐⭐ |
| Q4 | Tại sao cần key trong danh sách? | Junior | ⭐⭐⭐⭐⭐ |
| Q5 | Controlled vs Uncontrolled Components | Junior | ⭐⭐⭐⭐ |
| Q6 | Reconciliation hoạt động như thế nào? | Mid | ⭐⭐⭐⭐ |
| Q7 | Prop Drilling và cách giải quyết | Junior/Mid | ⭐⭐⭐⭐ |
| Q8 | Error Boundaries là gì? | Mid | ⭐⭐⭐ |
| Q9 | React.memo vs useMemo | Mid | ⭐⭐⭐⭐ |
| Q10 | React Strict Mode làm gì? | Mid | ⭐⭐⭐ |
| Q11 | useEffect dependency array | Junior/Mid | ⭐⭐⭐⭐⭐ |
| Q12 | Stale Closure trong Hooks | Mid/Senior | ⭐⭐⭐⭐ |
| Q13 | Khi nào dùng useCallback? | Mid | ⭐⭐⭐⭐ |
| Q14 | useReducer vs useState | Mid | ⭐⭐⭐ |
| Q15 | useRef ngoài tham chiếu DOM | Mid | ⭐⭐⭐ |
| Q16 | Rules of Hooks | Junior/Mid | ⭐⭐⭐⭐ |
| Q17 | Custom Hooks — ví dụ thực tế | Mid | ⭐⭐⭐⭐ |
| Q18 | useLayoutEffect vs useEffect | Mid/Senior | ⭐⭐⭐ |
| Q19 | Context API — khi nào phù hợp? | Mid | ⭐⭐⭐⭐ |
| Q20 | Redux Toolkit vs Zustand | Mid/Senior | ⭐⭐⭐⭐ |
| Q21 | TanStack Query giải quyết gì? | Mid | ⭐⭐⭐⭐ |
| Q22 | Server State vs Client State | Mid/Senior | ⭐⭐⭐ |
| Q23 | React.memo khi nào thực sự giúp ích? | Mid/Senior | ⭐⭐⭐⭐ |
| Q24 | Code Splitting với React.lazy | Mid | ⭐⭐⭐⭐ |
| Q25 | startTransition và useDeferredValue | Senior | ⭐⭐⭐ |
| Q26 | Debug và tối ưu hiệu năng | Senior | ⭐⭐⭐⭐ |
| Q27 | React Server Components | Senior | ⭐⭐⭐⭐⭐ |
| Q28 | Server Actions | Senior | ⭐⭐⭐⭐ |
| Q29 | React Compiler | Senior | ⭐⭐⭐ |
| Q30 | Micro-frontends | Senior | ⭐⭐⭐ |

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
