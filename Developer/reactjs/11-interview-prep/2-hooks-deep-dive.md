# 🎣 Hooks Deep Dive — Câu Hỏi Chuyên Sâu Về Hooks

> Phần được hỏi nhiều nhất trong phỏng vấn React. Nắm vững Hooks là yếu tố phân biệt Junior với Mid/Senior Developer.

---

## 🗂️ Mục Lục

1. [Mental Model — Mô Hình Tư Duy Về Hooks](#mental-model)
2. [useState — Chuyên Sâu](#usestate-chuyên-sâu)
3. [useEffect — Chuyên Sâu](#useeffect-chuyên-sâu)
4. [useMemo & useCallback — Khi Nào Dùng](#usememo--usecallback)
5. [useRef — Ngoài DOM](#useref-ngoài-dom)
6. [useContext & useReducer](#usecontext--usereducer)
7. [Custom Hooks Nâng Cao](#custom-hooks-nâng-cao)
8. [Câu Hỏi Bẫy Thường Gặp](#câu-hỏi-bẫy-thường-gặp)

---

## Mental Model — Mô Hình Tư Duy Về Hooks {#mental-model}

### Hooks hoạt động dựa trên gì? Tại sao phải gọi theo thứ tự?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

React theo dõi hooks bằng **thứ tự gọi** — không phải tên. Mỗi component instance có một danh sách hooks riêng (fiber node). Mỗi lần gọi `useState`, `useEffect`... React lấy slot tiếp theo trong danh sách đó.

```
Component render lần 1:
  useState(0)      → hook[0]: { value: 0, setter: fn }
  useEffect(fn)    → hook[1]: { effect: fn, deps: [] }
  useState('')     → hook[2]: { value: '', setter: fn }

Component render lần 2 (state thay đổi):
  useState(0)      → hook[0]: { value: 1, setter: fn }  ← đọc state mới
  useEffect(fn)    → hook[1]: { effect: fn, deps: [] }
  useState('')     → hook[2]: { value: 'hello', setter: fn }
```

**Nếu vi phạm thứ tự (gọi hook trong điều kiện):**
```jsx
function Bad({ flag }) {
  const [a, setA] = useState(0);   // hook[0]
  if (flag) {
    const [b, setB] = useState(''); // hook[1] — chỉ đôi khi chạy!
  }
  const [c, setC] = useState(false); // hook[1] hoặc hook[2] tùy flag!
  // → React nhầm state c với state b → Bug khó debug
}
```

---

## useState — Chuyên Sâu {#usestate-chuyên-sâu}

### Câu hỏi: Lazy initialization — Khởi Tạo Lười trong useState là gì?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

```jsx
// ❌ initialState được tính toán mỗi lần render
// Dù useState chỉ dùng nó lần đầu tiên
function Component() {
  const [data, setData] = useState(parseExpensiveJSON(rawData)); // Chạy mỗi render!
}

// ✅ Lazy initialization — truyền hàm, chỉ gọi lần đầu
function Component() {
  const [data, setData] = useState(() => parseExpensiveJSON(rawData)); // Chạy 1 lần
}

// ✅ Thực tế: đọc localStorage
const [theme, setTheme] = useState(() => {
  // Chỉ đọc localStorage khi mount, không phải mỗi render
  return localStorage.getItem('theme') ?? 'light';
});
```

---

### Câu hỏi: Tại sao setState với cùng giá trị không trigger re-render?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

React dùng **Object.is()** để so sánh giá trị mới với cũ. Nếu bằng nhau → bail out — bỏ qua re-render.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(0); // count đã là 0 → Object.is(0, 0) = true → KHÔNG re-render
  };

  // Object reference:
  const [user, setUser] = useState({ name: 'Alice' });
  const handleUpdate = () => {
    setUser({ name: 'Alice' }); // Object mới, dù cùng nội dung → re-render!
    // Object.is({name:'Alice'}, {name:'Alice'}) = false (khác reference)
  };
}
```

**Implication — Hệ Quả:**
```jsx
// ❌ Mutate state trực tiếp → không re-render
const [items, setItems] = useState([]);
items.push('new item'); // Mutation! Reference không đổi → React không biết
setItems(items);        // Object.is(items, items) = true → bail out

// ✅ Luôn tạo object/array mới
setItems([...items, 'new item']); // Reference mới → re-render
setItems(prev => [...prev, 'new item']); // Functional updater — an toàn hơn
```

---

### Câu hỏi: useState vs useRef — khi nào dùng ref thay vì state?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

| | useState | useRef |
|--|----------|--------|
| Trigger re-render | ✅ Có | ❌ Không |
| Persist qua renders | ✅ Có | ✅ Có |
| Mutable | Qua setter | Trực tiếp `.current` |

```jsx
// Dùng useRef khi:
// 1. Cần lưu giá trị nhưng KHÔNG cần UI cập nhật khi nó thay đổi

function VideoPlayer() {
  const [isPlaying, setIsPlaying] = useState(false); // UI cần biết
  const videoRef = useRef(null);                     // DOM reference
  const playCountRef = useRef(0);                    // Track play count — không cần hiển thị

  const handlePlay = () => {
    playCountRef.current++;           // Tăng count không trigger re-render
    videoRef.current.play();          // DOM API
    setIsPlaying(true);               // Cập nhật UI
    analytics.track('play', { count: playCountRef.current }); // Dùng ref value
  };
}

// 2. Lưu timer ID, subscription, animation frame
function Timer() {
  const [seconds, setSeconds] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    intervalRef.current = setInterval(() => setSeconds(s => s + 1), 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current); // Dùng ref để dừng timer
  };
}
```

---

## useEffect — Chuyên Sâu {#useeffect-chuyên-sâu}

### Câu hỏi: Race condition — Điều Kiện Cạnh Tranh trong useEffect là gì? Cách fix?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

Race condition xảy ra khi nhiều async requests được gửi đi và response đến theo thứ tự không mong muốn:

```jsx
// ❌ Race condition
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => setUser(data)); // Không biết response nào đến sau!
  }, [userId]);

  // Scenario: userId đổi 1 → 2
  // Request 1 (user 1) gửi đi
  // Request 2 (user 2) gửi đi
  // Response 2 đến trước → setUser(user2)
  // Response 1 đến sau → setUser(user1) ← Hiển thị user SAI!
}

// ✅ Fix 1: Boolean flag
useEffect(() => {
  let cancelled = false;

  fetch(`/api/users/${userId}`)
    .then(r => r.json())
    .then(data => {
      if (!cancelled) setUser(data); // Chỉ update nếu effect còn "active"
    });

  return () => { cancelled = true; }; // Cleanup: mark as cancelled
}, [userId]);

// ✅ Fix 2: AbortController (chuẩn hơn)
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/users/${userId}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => setUser(data))
    .catch(err => {
      if (err.name !== 'AbortError') setError(err); // Bỏ qua lỗi do abort
    });

  return () => controller.abort(); // Hủy request khi cleanup
}, [userId]);

// ✅✅ Fix 3: TanStack Query tự xử lý race condition
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: ({ signal }) => fetch(`/api/users/${userId}`, { signal }).then(r => r.json()),
});
```

---

### Câu hỏi: Tại sao useEffect trong React 18 Strict Mode chạy 2 lần?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

React 18 Strict Mode trong development **mount → unmount → remount** để giả lập Concurrent Features — Tính Năng Đồng Thời. Mục đích: phát hiện effects không có cleanup đúng cách.

```jsx
// React 18 Strict Mode: mount → cleanup → mount lại
// Nếu effect của bạn chạy 2 lần và gây bug → thiếu cleanup!

// ❌ Effect không idempotent — không thể chạy 2 lần
useEffect(() => {
  fetchAndSaveToGlobalStore(); // Chạy 2 lần → duplicate data!
}, []);

// ✅ Effect có cleanup = idempotent
useEffect(() => {
  const subscription = subscribe(userId);

  return () => {
    subscription.unsubscribe(); // Cleanup → remount không bị duplicate
  };
}, [userId]);

// ✅ Suppress chỉ 1 lần (dùng ref trick — không khuyến nghị nhưng cần biết)
const isFirstMount = useRef(true);
useEffect(() => {
  if (isFirstMount.current) {
    isFirstMount.current = false;
    doOneTimeSetup();
  }
}, []);
```

---

### Câu hỏi: Object và Array trong dependency array — Vấn đề phổ biến

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

```jsx
// ❌ Vòng lặp vô tận — Infinite loop
function Component({ filters }) {
  // filters = { page: 1, size: 10 } — object mới mỗi lần parent render!
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchData(filters).then(setData);
  }, [filters]); // Mỗi render tạo filters mới → effect chạy mãi

  return <DataDisplay data={data} />;
}

// ✅ Fix 1: Truyền primitive values (số, string) thay vì object
function Component({ page, size }) {
  useEffect(() => {
    fetchData({ page, size }).then(setData);
  }, [page, size]); // Primitives: Object.is so sánh đúng
}

// ✅ Fix 2: Nếu phải dùng object, serialize nó
useEffect(() => {
  fetchData(filters).then(setData);
}, [JSON.stringify(filters)]); // Chỉ dùng khi filters có thể serialize được

// ✅ Fix 3: useDeepCompareEffect (custom hook)
function useDeepCompareEffect(callback, dependencies) {
  const currentDepsRef = useRef(dependencies);

  if (!deepEqual(currentDepsRef.current, dependencies)) {
    currentDepsRef.current = dependencies;
  }

  useEffect(callback, [currentDepsRef.current]);
}
```

---

## useMemo & useCallback — Khi Nào Dùng {#usememo--usecallback}

### Câu hỏi: Overhead của useMemo — Khi nào memoization có hại?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐

```jsx
// ❌ useMemo không cần thiết — tính toán rất đơn giản
const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// Nên viết thẳng:
const fullName = `${firstName} ${lastName}`;

// ❌ useMemo không cần thiết — tính toán chạy một lần
const config = useMemo(() => ({ theme: 'dark', lang: 'vi' }), []);
// Nên dùng: const config = { theme: 'dark', lang: 'vi' }; // Ngoài component

// ✅ useMemo cần thiết
// 1. Tính toán tốn kém
const sortedAndFilteredItems = useMemo(() => {
  return items
    .filter(item => item.price >= minPrice && item.price <= maxPrice)
    .sort((a, b) => b.rating - a.rating);
}, [items, minPrice, maxPrice]);

// 2. Kết quả được truyền xuống component con có React.memo
const chartData = useMemo(() => transformToChartFormat(rawData), [rawData]);
<ExpensiveChart data={chartData} />

// 3. Object được dùng trong dependency array
const queryParams = useMemo(() => ({
  search: debouncedSearch,
  filters: activeFilters,
  page,
}), [debouncedSearch, activeFilters, page]);

useEffect(() => {
  fetchResults(queryParams);
}, [queryParams]); // Ổn định reference
```

---

### Câu hỏi: useCallback — Khi nào thực sự cần?

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐⭐

```jsx
// ❌ useCallback không có tác dụng nếu component con không memo
function Parent() {
  const handleClick = useCallback(() => setCount(c => c + 1), []);
  return <Child onClick={handleClick} />; // Child không memo → vẫn re-render
}

// ✅ useCallback cần khi component con CÓ memo
const MemoizedChild = React.memo(({ onClick, label }) => (
  <button onClick={onClick}>{label}</button>
));

function Parent() {
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []); // Reference ổn định → MemoizedChild không re-render khi Parent re-render

  return <MemoizedChild onClick={handleClick} label="Increment" />;
}

// ✅ useCallback cần khi function là dependency của hook khác
const fetchData = useCallback(async (page) => {
  const response = await api.get('/data', { params: { page } });
  return response.data;
}, []); // Reference ổn định

useEffect(() => {
  fetchData(currentPage).then(setData);
}, [fetchData, currentPage]); // fetchData không thay đổi → effect chỉ chạy khi currentPage đổi
```

---

## useContext & useReducer {#usecontext--usereducer}

### Câu hỏi: useContext gây re-render toàn bộ như thế nào? Cách tối ưu?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

```jsx
// ❌ Mọi component consume context re-render khi BẤT KỲ phần nào của context value thay đổi
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [cart, setCart] = useState([]);

  // Mỗi lần cart thay đổi → UserProfile (chỉ cần user) cũng re-render!
  return (
    <AppContext.Provider value={{ user, setUser, theme, setTheme, cart, setCart }}>
      {children}
    </AppContext.Provider>
  );
}

// ✅ Tách Context theo domain
const UserContext = createContext();
const ThemeContext = createContext();
const CartContext = createContext();

// ✅ Tách giá trị và setter riêng
const UserStateContext = createContext();  // { user } — đọc
const UserDispatchContext = createContext(); // setUser — ghi

function UserProvider({ children }) {
  const [user, setUser] = useState(null);

  return (
    <UserStateContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserStateContext.Provider>
  );
}

// Component chỉ đọc user → không re-render khi setUser thay đổi
function UserProfile() {
  const user = useContext(UserStateContext);
  return <div>{user?.name}</div>;
}

// Component chỉ gọi setUser → không re-render khi user thay đổi
function LoginButton() {
  const setUser = useContext(UserDispatchContext); // setUser reference ổn định
  return <button onClick={() => setUser({ name: 'Alice' })}>Login</button>;
}
```

---

### Câu hỏi: Implement useReducer với middleware — Middleware pattern

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// Implement logger middleware cho useReducer
function useReducerWithLogger(reducer, initialState) {
  const reducerWithLogger = useCallback((state, action) => {
    const nextState = reducer(state, action);
    console.group(`Action: ${action.type}`);
    console.log('Prev state:', state);
    console.log('Action:', action);
    console.log('Next state:', nextState);
    console.groupEnd();
    return nextState;
  }, [reducer]);

  return useReducer(reducerWithLogger, initialState);
}

// Thực tế: quản lý form state phức tạp
const formReducer = (state, action) => {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: '' }, // Xóa error khi user sửa
      };
    case 'SET_ERROR':
      return { ...state, errors: { ...state.errors, [action.field]: action.error } };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true };
    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false, submitted: true };
    case 'SUBMIT_ERROR':
      return { ...state, isSubmitting: false, serverError: action.error };
    default:
      return state;
  }
};
```

---

## Custom Hooks Nâng Cao {#custom-hooks-nâng-cao}

### Câu hỏi: Implement useFetch hook toàn diện

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

```jsx
function useFetch(url, options = {}) {
  const [state, dispatch] = useReducer(
    (state, action) => {
      switch (action.type) {
        case 'FETCHING': return { ...state, loading: true, error: null };
        case 'SUCCESS':  return { data: action.data, loading: false, error: null };
        case 'ERROR':    return { ...state, loading: false, error: action.error };
        default:         return state;
      }
    },
    { data: null, loading: false, error: null }
  );

  const optionsRef = useRef(options); // Tránh re-fetch khi options thay đổi reference

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController();
    dispatch({ type: 'FETCHING' });

    fetch(url, { ...optionsRef.current, signal: controller.signal })
      .then(async response => {
        if (!response.ok) throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        const data = await response.json();
        dispatch({ type: 'SUCCESS', data });
      })
      .catch(error => {
        if (error.name !== 'AbortError') {
          dispatch({ type: 'ERROR', error: error.message });
        }
      });

    return () => controller.abort();
  }, [url]);

  return state;
}

// Sử dụng
function UserCard({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <Skeleton />;
  if (error) return <ErrorMessage message={error} />;
  return <div>{user?.name}</div>;
}
```

---

### Câu hỏi: Implement useEventListener hook

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

```jsx
function useEventListener(eventName, handler, element = window) {
  const savedHandler = useRef(handler);

  // Cập nhật ref khi handler thay đổi — tránh stale closure
  useLayoutEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const targetElement = element?.current ?? element;
    if (!targetElement?.addEventListener) return;

    const eventListener = (event) => savedHandler.current(event);
    targetElement.addEventListener(eventName, eventListener);

    return () => {
      targetElement.removeEventListener(eventName, eventListener);
    };
  }, [eventName, element]);
}

// Sử dụng
function KeyboardShortcut() {
  useEventListener('keydown', (event) => {
    if (event.ctrlKey && event.key === 's') {
      event.preventDefault();
      saveDraft();
    }
  });
}

function ClickOutside({ onClickOutside, containerRef }) {
  useEventListener('mousedown', (event) => {
    if (containerRef.current && !containerRef.current.contains(event.target)) {
      onClickOutside();
    }
  });
}
```

---

### Câu hỏi: Implement useAsyncState — quản lý async state tái sử dụng

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
function useAsync(asyncFunction, immediate = true) {
  const [status, setStatus] = useState('idle'); // idle | pending | success | error
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  const execute = useCallback(async (...args) => {
    setStatus('pending');
    setData(null);
    setError(null);

    try {
      const result = await asyncFunction(...args);
      setData(result);
      setStatus('success');
      return result;
    } catch (err) {
      setError(err);
      setStatus('error');
      throw err;
    }
  }, [asyncFunction]);

  useEffect(() => {
    if (immediate) execute();
  }, [execute, immediate]);

  return {
    execute,
    status,
    data,
    error,
    isIdle: status === 'idle',
    isPending: status === 'pending',
    isSuccess: status === 'success',
    isError: status === 'error',
  };
}

// Sử dụng
function UserDashboard({ userId }) {
  const fetchUser = useCallback(() => api.getUser(userId), [userId]);
  const { data: user, isPending, isError, error, execute: refetch } = useAsync(fetchUser);

  if (isPending) return <Spinner />;
  if (isError) return <ErrorBoundary message={error.message} onRetry={refetch} />;
  return <UserProfile user={user} />;
}
```

---

## Câu Hỏi Bẫy Thường Gặp {#câu-hỏi-bẫy-thường-gặp}

### Bẫy 1: useEffect với async function

```jsx
// ❌ Sai — useEffect không nên là async function
useEffect(async () => {
  const data = await fetchData(); // async/await OK
  setData(data);
  // Nhưng: useEffect async trả về Promise, React không xử lý Promise cleanup!
  // return async () => {} ← React nhận Promise, không thể cleanup đúng
}, []);

// ✅ Đúng — Tạo async function bên trong
useEffect(() => {
  async function loadData() {
    try {
      const data = await fetchData();
      setData(data);
    } catch (error) {
      setError(error);
    }
  }

  loadData();

  return () => { /* cleanup sync */ };
}, []);
```

---

### Bẫy 2: Setter function không cần trong dependencies

```jsx
// useState setters — KHÔNG cần thêm vào deps (ổn định reference)
const [count, setCount] = useState(0);

useEffect(() => {
  setCount(count + 1); // setCount KHÔNG cần trong deps
  // Nhưng count CẦN (stale closure)
}, [count]); // ✅ Chỉ cần count

// useDispatch từ Redux — tương tự, không cần thêm vào deps
const dispatch = useDispatch();
useEffect(() => {
  dispatch(fetchData()); // dispatch không cần deps
}, []); // ✅
```

---

### Bẫy 3: useCallback không cứu được nếu deps thay đổi

```jsx
// useCallback với deps thay đổi mỗi render = không có ích gì
function Parent({ onDataChange }) {
  const [items, setItems] = useState([]);

  // ❌ items thay đổi → handler thay đổi → child re-render
  const handleAdd = useCallback((item) => {
    setItems([...items, item]); // items trong closure
  }, [items]); // deps thay đổi khi items thay đổi

  // ✅ Dùng functional updater → không cần items trong deps
  const handleAdd = useCallback((item) => {
    setItems(prev => [...prev, item]); // Không đọc items
  }, []); // deps rỗng → reference ổn định

  return <Child onAdd={handleAdd} />;
}
```

---

### Bẫy 4: Không cleanup khi component unmount

```jsx
// ❌ setState sau khi component unmount
function DataFetcher({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchData(id).then(data => {
      setData(data); // ⚠️ Nếu component unmount trước khi fetch xong → warning
    });
  }, [id]);
  // "Warning: Can't perform a React state update on an unmounted component"
  // (React 17 warning, React 18 đã xử lý khác nhưng vẫn là bug logic)
}

// ✅ Cleanup đúng cách
useEffect(() => {
  let mounted = true;

  fetchData(id).then(data => {
    if (mounted) setData(data);
  });

  return () => { mounted = false; };
}, [id]);
```

---

### Bẫy 5: Đọc state trong useEffect trước khi re-render

```jsx
// ❌ Bug: đọc state cũ trong setTimeout
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setTimeout(() => {
      console.log(count); // In giá trị CŨ — stale closure!
    }, 1000);
  };
}

// ✅ Fix: dùng ref để lấy giá trị hiện tại
function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  countRef.current = count; // Luôn cập nhật ref mỗi render

  const handleClick = () => {
    setCount(count + 1);
    setTimeout(() => {
      console.log(countRef.current); // Luôn là giá trị mới nhất
    }, 1000);
  };
}
```

---

## ✅ Checklist Hooks — Mid/Senior Level

**useState:**
- [ ] Lazy initialization — khi nào dùng
- [ ] Functional updater — tại sao cần
- [ ] Immutability — không mutate state trực tiếp

**useEffect:**
- [ ] 3 patterns: no deps, [], [deps]
- [ ] Cleanup function — khi nào cần
- [ ] Race condition — cách fix với AbortController
- [ ] Strict Mode double-invoke — giải thích được

**useMemo & useCallback:**
- [ ] Phân biệt mục đích
- [ ] Khi nào có ích, khi nào gây hại
- [ ] Kết hợp với React.memo

**useRef:**
- [ ] DOM reference
- [ ] Mutable value không trigger re-render
- [ ] Stale closure workaround

**Custom Hooks:**
- [ ] Viết được useFetch, useDebounce, useLocalStorage không nhìn notes
- [ ] Biết khi nào nên extract ra custom hook

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
