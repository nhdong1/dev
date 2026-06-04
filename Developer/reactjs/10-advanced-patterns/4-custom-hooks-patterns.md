# 4 — Custom Hooks Nâng Cao (Advanced Custom Hook Patterns)

> **Custom Hooks nâng cao** đi sâu hơn so với việc chỉ gói state đơn giản — đây là nghệ thuật thiết kế **hook API** trực quan, compose được, có thể tái sử dụng cho nhiều ngữ cảnh. Các pattern như **state machines**, **factory hooks**, **hook composition** là đặc trưng của senior React developer.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Thiết kế Custom Hook API trực quan, dễ sử dụng
- [ ] Implement **State Machine** (Máy Trạng Thái) với `useReducer`
- [ ] Xây dựng **Factory Hooks** (Hook Nhà Máy) tạo hook từ config
- [ ] Compose nhiều hook thành hook phức tạp hơn
- [ ] Viết hook có **controlled/uncontrolled** mode
- [ ] Implement các utility hooks thường dùng: `useDebounce`, `usePrevious`, `useLocalStorage`

---

## 1. Nguyên Tắc Thiết Kế Hook API Tốt

### Nguyên Tắc 1: Return Object, Không Phải Array (Cho Nhiều Giá Trị)

```tsx
// ❌ Array — phải nhớ thứ tự, không đặt tên lại được
const [user, loading, error, refetch] = useUser(id);

// ✅ Object — tên rõ ràng, lấy chỉ cái cần
const { user, isLoading, error, refetch } = useUser(id);

// Người dùng có thể rename khi destructure
const { user: currentUser, isLoading: isFetchingUser } = useUser(id);
```

### Nguyên Tắc 2: Nhất Quán Về Tên

```tsx
// Quy ước đặt tên nhất quán
interface UseQueryResult<T> {
  data: T | null;         // Dữ liệu
  isLoading: boolean;     // Đang tải lần đầu
  isFetching: boolean;    // Đang tải lại (background)
  isError: boolean;       // Có lỗi
  error: Error | null;    // Chi tiết lỗi
  refetch: () => void;    // Tải lại
}
```

### Nguyên Tắc 3: Controlled/Uncontrolled Mode

```tsx
interface UseDisclosureOptions {
  defaultOpen?: boolean;  // Uncontrolled mode
  open?: boolean;         // Controlled mode
  onOpenChange?: (open: boolean) => void;
}

function useDisclosure({
  defaultOpen = false,
  open: controlledOpen,
  onOpenChange,
}: UseDisclosureOptions = {}) {
  const [internalOpen, setInternalOpen] = useState(defaultOpen);

  // Nếu có controlled value → dùng nó, không thì dùng internal
  const isOpen = controlledOpen !== undefined ? controlledOpen : internalOpen;

  const setOpen = useCallback((nextOpen: boolean) => {
    if (controlledOpen === undefined) {
      setInternalOpen(nextOpen);
    }
    onOpenChange?.(nextOpen);
  }, [controlledOpen, onOpenChange]);

  return {
    isOpen,
    open: () => setOpen(true),
    close: () => setOpen(false),
    toggle: () => setOpen(!isOpen),
  };
}

// Uncontrolled
const dialog = useDisclosure({ defaultOpen: false });

// Controlled
const [open, setOpen] = useState(false);
const dialog = useDisclosure({ open, onOpenChange: setOpen });
```

---

## 2. State Machine Pattern Với useReducer

### Vấn Đề Với Boolean States Nhiều Chiều

```tsx
// ❌ Trạng thái không hợp lệ có thể xảy ra
const [isLoading, setIsLoading] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [isError, setIsError] = useState(false);

// Bug: có thể vô tình set cả loading=true VÀ error=true cùng lúc
setIsLoading(true);
setIsError(true); // Trạng thái không hợp lệ!
```

### Giải Pháp: State Machine

```tsx
// State Machine — Máy Trạng Thái — loại bỏ trạng thái không hợp lệ
type AsyncState<T, E = Error> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: E };

type AsyncAction<T, E = Error> =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: T }
  | { type: 'FETCH_ERROR'; error: E }
  | { type: 'RESET' };

function asyncReducer<T, E = Error>(
  state: AsyncState<T, E>,
  action: AsyncAction<T, E>
): AsyncState<T, E> {
  switch (action.type) {
    case 'FETCH_START':
      return { status: 'loading' };
    case 'FETCH_SUCCESS':
      return { status: 'success', data: action.payload };
    case 'FETCH_ERROR':
      return { status: 'error', error: action.error };
    case 'RESET':
      return { status: 'idle' };
    default:
      return state;
  }
}

function useAsync<T>(asyncFn: () => Promise<T>) {
  const [state, dispatch] = useReducer(asyncReducer<T>, { status: 'idle' });

  const run = useCallback(async () => {
    dispatch({ type: 'FETCH_START' });
    try {
      const data = await asyncFn();
      dispatch({ type: 'FETCH_SUCCESS', payload: data });
    } catch (error) {
      dispatch({ type: 'FETCH_ERROR', error: error as Error });
    }
  }, [asyncFn]);

  const reset = useCallback(() => dispatch({ type: 'RESET' }), []);

  return { ...state, run, reset };
}

// Sử dụng — không bao giờ có trạng thái không hợp lệ
function UserProfile({ userId }: { userId: string }) {
  const fetchUser = useCallback(() => fetchUserById(userId), [userId]);
  const { status, data, error, run } = useAsync(fetchUser);

  useEffect(() => { run(); }, [run]);

  // Type narrowing — TypeScript biết chính xác kiểu data trong mỗi case
  if (status === 'idle') return <button onClick={run}>Tải dữ liệu</button>;
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <ErrorMessage message={error.message} onRetry={run} />;
  // status === 'success' → data: T (không phải T | undefined)
  return <UserCard user={data} />;
}
```

---

## 3. Finite State Machine — FSM (Máy Trạng Thái Hữu Hạn) Nâng Cao

```tsx
// Ví dụ: Traffic Light FSM — Đèn giao thông
type TrafficLightState = 'red' | 'yellow' | 'green';

const transitions: Record<TrafficLightState, TrafficLightState> = {
  red: 'green',
  green: 'yellow',
  yellow: 'red',
};

const durations: Record<TrafficLightState, number> = {
  red: 5000,
  yellow: 1000,
  green: 4000,
};

function useTrafficLight(initialState: TrafficLightState = 'red') {
  const [state, setState] = useState<TrafficLightState>(initialState);

  useEffect(() => {
    const timer = setTimeout(() => {
      setState((current) => transitions[current]);
    }, durations[state]);

    return () => clearTimeout(timer);
  }, [state]);

  return { state, skip: () => setState(transitions[state]) };
}

// Ví dụ thực tế hơn: Checkout flow FSM
type CheckoutState =
  | 'cart'
  | 'address'
  | 'payment'
  | 'review'
  | 'processing'
  | 'success'
  | 'error';

type CheckoutEvent =
  | 'PROCEED'
  | 'BACK'
  | 'SUBMIT'
  | 'PAYMENT_SUCCESS'
  | 'PAYMENT_FAILURE'
  | 'RETRY';

const checkoutMachine: Record<CheckoutState, Partial<Record<CheckoutEvent, CheckoutState>>> = {
  cart:       { PROCEED: 'address' },
  address:    { PROCEED: 'payment', BACK: 'cart' },
  payment:    { PROCEED: 'review', BACK: 'address' },
  review:     { SUBMIT: 'processing', BACK: 'payment' },
  processing: { PAYMENT_SUCCESS: 'success', PAYMENT_FAILURE: 'error' },
  success:    {},
  error:      { RETRY: 'payment' },
};

function useCheckout() {
  const [state, setState] = useState<CheckoutState>('cart');

  const send = useCallback((event: CheckoutEvent) => {
    setState((current) => {
      const next = checkoutMachine[current][event];
      return next ?? current; // Không có transition → ở lại state hiện tại
    });
  }, []);

  return { state, send };
}
```

---

## 4. Factory Hooks — Tạo Hook Từ Config

```tsx
// Factory: nhận config, trả về hook được cấu hình sẵn
function createLocalStorageHook<T>(key: string, defaultValue: T) {
  return function useLocalStorage(): [T, (value: T | ((prev: T) => T)) => void] {
    const [storedValue, setStoredValue] = useState<T>(() => {
      try {
        const item = window.localStorage.getItem(key);
        return item ? (JSON.parse(item) as T) : defaultValue;
      } catch {
        return defaultValue;
      }
    });

    const setValue = useCallback((value: T | ((prev: T) => T)) => {
      try {
        const valueToStore =
          value instanceof Function ? value(storedValue) : value;
        setStoredValue(valueToStore);
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      } catch (error) {
        console.error('useLocalStorage error:', error);
      }
    }, [storedValue]);

    return [storedValue, setValue];
  };
}

// Tạo hooks cụ thể từ factory
const useThemePreference = createLocalStorageHook<'light' | 'dark'>('theme', 'light');
const useLanguagePreference = createLocalStorageHook<string>('lang', 'vi');
const useCartItems = createLocalStorageHook<CartItem[]>('cart', []);

// Sử dụng như hook thông thường
function ThemeToggle() {
  const [theme, setTheme] = useThemePreference();
  return (
    <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
      {theme === 'light' ? '🌙 Tối' : '☀️ Sáng'}
    </button>
  );
}
```

---

## 5. Hook Composition — Kết Hợp Hooks

```tsx
// Hooks đơn lẻ — building blocks
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  return {
    count,
    increment: () => setCount(c => c + 1),
    decrement: () => setCount(c => c - 1),
    reset: () => setCount(initialValue),
  };
}

function useLocalStorage<T>(key: string, defaultValue: T) {
  // ... implementation
}

// Compose: hook mới kết hợp nhiều hooks
function usePersistedCounter(key: string, initialValue = 0) {
  const [storedValue, setStoredValue] = useLocalStorage(key, initialValue);
  const counter = useCounter(storedValue);

  // Sync storage khi count thay đổi
  useEffect(() => {
    setStoredValue(counter.count);
  }, [counter.count]);

  return counter;
}

// Ví dụ phức tạp hơn: useDataTable
function useSorting<T>(data: T[], key: keyof T) {
  const [sortConfig, setSortConfig] = useState<{
    key: keyof T;
    direction: 'asc' | 'desc';
  } | null>(null);

  const sortedData = useMemo(() => {
    if (!sortConfig) return data;
    return [...data].sort((a, b) => {
      const aVal = a[sortConfig.key];
      const bVal = b[sortConfig.key];
      if (aVal < bVal) return sortConfig.direction === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortConfig.direction === 'asc' ? 1 : -1;
      return 0;
    });
  }, [data, sortConfig]);

  return { sortedData, sortConfig, setSortConfig };
}

function usePagination<T>(data: T[], pageSize: number) {
  const [page, setPage] = useState(1);
  const totalPages = Math.ceil(data.length / pageSize);

  const paginatedData = useMemo(
    () => data.slice((page - 1) * pageSize, page * pageSize),
    [data, page, pageSize]
  );

  return { paginatedData, page, setPage, totalPages };
}

function useSearch<T>(data: T[], searchKey: keyof T) {
  const [query, setQuery] = useState('');

  const filteredData = useMemo(
    () =>
      query
        ? data.filter((item) =>
            String(item[searchKey]).toLowerCase().includes(query.toLowerCase())
          )
        : data,
    [data, query, searchKey]
  );

  return { filteredData, query, setQuery };
}

// Compose tất cả thành useDataTable
function useDataTable<T extends Record<string, unknown>>(
  initialData: T[],
  options: { searchKey: keyof T; pageSize?: number }
) {
  const { filteredData, query, setQuery } = useSearch(initialData, options.searchKey);
  const { sortedData, sortConfig, setSortConfig } = useSorting(filteredData, options.searchKey);
  const { paginatedData, page, setPage, totalPages } = usePagination(
    sortedData,
    options.pageSize ?? 10
  );

  // Reset về trang 1 khi search thay đổi
  useEffect(() => { setPage(1); }, [query]);

  return {
    data: paginatedData,
    search: { query, setQuery },
    sort: { config: sortConfig, setConfig: setSortConfig },
    pagination: { page, setPage, totalPages },
  };
}
```

---

## 6. Utility Hooks Thường Dùng

### useDebounce — Trì Hoãn Thực Thi

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Sử dụng trong search input
function SearchBar() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  // Chỉ gọi API khi debouncedQuery thay đổi (sau 300ms không gõ)
  const { data } = useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn: () => searchAPI(debouncedQuery),
    enabled: debouncedQuery.length > 2,
  });

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

### usePrevious — Giá Trị Trước Đó

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  // Trả về giá trị từ render trước (không phải render hiện tại)
  return ref.current;
}

// Sử dụng để animate khi giá trị thay đổi
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Hiện tại: {count}</p>
      <p>Trước đó: {prevCount ?? 'N/A'}</p>
      <p>
        {prevCount !== undefined && count > prevCount ? '📈 Tăng' : '📉 Giảm'}
      </p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={() => setCount(c => c - 1)}>-</button>
    </div>
  );
}
```

### useEventListener — Event Listener An Toàn

```tsx
function useEventListener<
  K extends keyof WindowEventMap,
  T extends EventTarget = Window
>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element?: T | null,
  options?: AddEventListenerOptions
) {
  // Dùng ref để tránh stale closure
  const savedHandler = useRef(handler);

  useLayoutEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const target = element ?? window;
    if (!target?.addEventListener) return;

    const eventListener = (event: Event) =>
      savedHandler.current(event as WindowEventMap[K]);

    target.addEventListener(eventName, eventListener, options);
    return () => target.removeEventListener(eventName, eventListener, options);
  }, [eventName, element, options]);
}

// Sử dụng
function KeyboardShortcutDemo() {
  useEventListener('keydown', (e) => {
    if (e.ctrlKey && e.key === 's') {
      e.preventDefault();
      handleSave();
    }
  });

  return <div>Nhấn Ctrl+S để lưu</div>;
}
```

### useIntersectionObserver — Lazy Loading và Infinite Scroll

```tsx
interface IntersectionObserverOptions extends IntersectionObserverInit {
  freezeOnceVisible?: boolean;
}

function useIntersectionObserver(
  elementRef: React.RefObject<Element>,
  options: IntersectionObserverOptions = {}
): IntersectionObserverEntry | undefined {
  const { threshold = 0, root = null, rootMargin = '0%', freezeOnceVisible = false } = options;
  const [entry, setEntry] = useState<IntersectionObserverEntry>();

  const frozen = entry?.isIntersecting && freezeOnceVisible;

  useEffect(() => {
    const element = elementRef?.current;
    if (!element || frozen) return;

    const observer = new IntersectionObserver(
      ([entry]) => setEntry(entry),
      { threshold, root, rootMargin }
    );

    observer.observe(element);
    return () => observer.disconnect();
  }, [elementRef, threshold, root, rootMargin, frozen]);

  return entry;
}

// Sử dụng cho lazy image loading
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const entry = useIntersectionObserver(ref, { freezeOnceVisible: true });
  const isVisible = !!entry?.isIntersecting;

  return (
    <div ref={ref}>
      {isVisible ? <img src={src} alt={alt} /> : <div className="placeholder" />}
    </div>
  );
}
```

### useMediaQuery — Responsive Logic Trong JS

```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(
    () => window.matchMedia(query).matches
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    setMatches(mediaQuery.matches);

    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Tạo breakpoint hooks từ factory
const createBreakpointHook = (query: string) => () => useMediaQuery(query);

const useIsMobile = createBreakpointHook('(max-width: 768px)');
const useIsTablet = createBreakpointHook('(min-width: 769px) and (max-width: 1024px)');
const useIsDesktop = createBreakpointHook('(min-width: 1025px)');
const useIsDarkMode = createBreakpointHook('(prefers-color-scheme: dark)');

// Sử dụng
function ResponsiveNav() {
  const isMobile = useIsMobile();
  return isMobile ? <HamburgerMenu /> : <DesktopNav />;
}
```

---

## 7. useReducer + useContext — Global State Pattern

```tsx
// Pattern thay thế Redux nhẹ hơn với hooks
interface AppState {
  theme: 'light' | 'dark';
  language: string;
  notifications: Notification[];
}

type AppAction =
  | { type: 'SET_THEME'; payload: 'light' | 'dark' }
  | { type: 'SET_LANGUAGE'; payload: string }
  | { type: 'ADD_NOTIFICATION'; payload: Notification }
  | { type: 'REMOVE_NOTIFICATION'; payload: string };

function appReducer(state: AppState, action: AppAction): AppState {
  switch (action.type) {
    case 'SET_THEME': return { ...state, theme: action.payload };
    case 'SET_LANGUAGE': return { ...state, language: action.payload };
    case 'ADD_NOTIFICATION':
      return { ...state, notifications: [...state.notifications, action.payload] };
    case 'REMOVE_NOTIFICATION':
      return {
        ...state,
        notifications: state.notifications.filter((n) => n.id !== action.payload),
      };
    default: return state;
  }
}

const AppStateContext = createContext<AppState | null>(null);
const AppDispatchContext = createContext<React.Dispatch<AppAction> | null>(null);

// Tách context để tránh re-render không cần thiết
function AppProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(appReducer, {
    theme: 'light',
    language: 'vi',
    notifications: [],
  });

  return (
    <AppStateContext.Provider value={state}>
      <AppDispatchContext.Provider value={dispatch}>
        {children}
      </AppDispatchContext.Provider>
    </AppStateContext.Provider>
  );
}

// Custom hooks để tiêu thụ
function useAppState() {
  const ctx = useContext(AppStateContext);
  if (!ctx) throw new Error('Cần bọc trong AppProvider');
  return ctx;
}

function useAppDispatch() {
  const ctx = useContext(AppDispatchContext);
  if (!ctx) throw new Error('Cần bọc trong AppProvider');
  return ctx;
}

// Selector hooks — chỉ re-render khi phần dữ liệu cần thay đổi
function useTheme() {
  const { theme } = useAppState();
  const dispatch = useAppDispatch();

  const toggleTheme = useCallback(
    () => dispatch({ type: 'SET_THEME', payload: theme === 'light' ? 'dark' : 'light' }),
    [theme, dispatch]
  );

  return { theme, toggleTheme };
}
```

---

## 📝 Checklist Tự Đánh Giá

- [ ] Thiết kế hook API với controlled/uncontrolled mode
- [ ] Implement State Machine với `useReducer` để loại bỏ trạng thái không hợp lệ
- [ ] Tạo Factory Hook đơn giản
- [ ] Compose nhiều hooks nhỏ thành hook phức tạp hơn
- [ ] Implement `useDebounce`, `usePrevious`, `useMediaQuery` từ đầu

---

## 🔗 Điều Hướng

- **Trước đó:** [3-hoc.md](./3-hoc.md) — Higher-Order Components
- **Tiếp theo:** [5-portals-boundaries.md](./5-portals-boundaries.md) — Portals và Error Boundaries
- **Chỉ mục:** [INDEX.md](../INDEX.md)
