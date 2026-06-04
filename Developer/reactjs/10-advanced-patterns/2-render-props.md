# 2 — Render Props (Props Hàm Render)

> **Render Props** là pattern chia sẻ logic giữa các component bằng cách truyền một **hàm** qua props — hàm đó trả về JSX. Pattern này giải quyết vấn đề tái sử dụng logic có trạng thái trước khi React Hooks ra đời (React < 16.8), và vẫn hữu ích trong một số trường hợp cụ thể.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Giải thích Render Props pattern và khi nào nên dùng
- [ ] Implement component dùng render props và `children as function`
- [ ] Hiểu tại sao **Custom Hooks** thường thay thế Render Props
- [ ] Nhận biết Render Props trong codebase thực tế (React Router, Formik, ...)
- [ ] Biết các vấn đề hiệu năng và cách tránh

---

## 1. Vấn Đề Cần Giải Quyết

### Bài Toán: Chia Sẻ Logic Mouse Tracking

```tsx
// ❌ Không tái sử dụng được — mỗi component tự track mouse
function CatFollowsMouse() {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  return <img src="/cat.png" style={{ position: 'fixed', left: position.x, top: position.y }} />;
}

function DogFollowsMouse() {
  // Copy-paste logic mouse tracking — trùng lặp hoàn toàn
  const [position, setPosition] = useState({ x: 0, y: 0 });
  // ...
  return <img src="/dog.png" style={{ position: 'fixed', left: position.x, top: position.y }} />;
}
```

---

## 2. Render Props — Giải Pháp

### Cách Implement Cơ Bản

```tsx
interface Position {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  // render prop: hàm nhận position và trả về JSX
  render: (position: Position) => React.ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [position, setPosition] = useState<Position>({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  // Gọi hàm render với position hiện tại
  return <>{render(position)}</>;
}

// Sử dụng — linh hoạt, không cần biết cách render
function App() {
  return (
    <div>
      <MouseTracker render={({ x, y }) => (
        <img
          src="/cat.png"
          style={{ position: 'fixed', left: x, top: y }}
          alt="Con mèo"
        />
      )} />

      <MouseTracker render={({ x, y }) => (
        <p>Vị trí chuột: ({x}, {y})</p>
      )} />
    </div>
  );
}
```

---

## 3. Children As Function — Biến Thể Phổ Biến

Thay vì prop `render`, dùng `children` là một hàm — cú pháp tự nhiên hơn:

```tsx
interface MouseTrackerProps {
  children: (position: Position) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState<Position>({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  // children là hàm — gọi như function
  return <>{children(position)}</>;
}

// Sử dụng — đọc tự nhiên hơn
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <img
          src="/cat.png"
          style={{ position: 'fixed', left: x, top: y }}
          alt="Con mèo"
        />
      )}
    </MouseTracker>
  );
}
```

---

## 4. Ví Dụ Thực Tế — Toggle Component

```tsx
interface ToggleRenderProps {
  on: boolean;
  toggle: () => void;
  setOn: (value: boolean) => void;
}

interface ToggleProps {
  defaultOn?: boolean;
  onToggle?: (on: boolean) => void;
  children: (props: ToggleRenderProps) => React.ReactNode;
}

function Toggle({ defaultOn = false, onToggle, children }: ToggleProps) {
  const [on, setOnInternal] = useState(defaultOn);

  const setOn = (value: boolean) => {
    setOnInternal(value);
    onToggle?.(value);
  };

  const toggle = () => setOn(!on);

  return <>{children({ on, toggle, setOn })}</>;
}

// Sử dụng với nhiều UI khác nhau
function App() {
  return (
    <>
      {/* Dùng như switch */}
      <Toggle defaultOn={false} onToggle={(v) => console.log('Toggle:', v)}>
        {({ on, toggle }) => (
          <div>
            <button onClick={toggle}>
              {on ? '🔆 Tắt đèn' : '🔅 Bật đèn'}
            </button>
            <p>Đèn đang: {on ? 'BẬT' : 'TẮT'}</p>
          </div>
        )}
      </Toggle>

      {/* Dùng như modal trigger */}
      <Toggle>
        {({ on, toggle }) => (
          <>
            <button onClick={toggle}>Mở chi tiết</button>
            {on && (
              <div className="detail-panel">
                <p>Nội dung chi tiết...</p>
                <button onClick={toggle}>Đóng</button>
              </div>
            )}
          </>
        )}
      </Toggle>
    </>
  );
}
```

---

## 5. Ví Dụ Thực Tế — Data Fetcher

```tsx
interface FetchRenderProps<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

interface DataFetcherProps<T> {
  url: string;
  children: (props: FetchRenderProps<T>) => React.ReactNode;
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetch_ = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const json = await res.json();
      setData(json);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [url]);

  useEffect(() => { fetch_(); }, [fetch_]);

  return <>{children({ data, loading, error, refetch: fetch_ })}</>;
}

// Sử dụng
interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: number }) {
  return (
    <DataFetcher<User> url={`/api/users/${userId}`}>
      {({ data, loading, error, refetch }) => {
        if (loading) return <Spinner />;
        if (error) return (
          <div>
            <p>Lỗi: {error.message}</p>
            <button onClick={refetch}>Thử lại</button>
          </div>
        );
        if (!data) return null;

        return (
          <div>
            <h2>{data.name}</h2>
            <p>{data.email}</p>
          </div>
        );
      }}
    </DataFetcher>
  );
}
```

---

## 6. Render Props Trong Thực Tế — React Router, Formik

### React Router v5 (Legacy) — Route Render Props

```tsx
// React Router v5 dùng render props
<Route
  path="/users/:id"
  render={({ match, history }) => (
    <UserDetail
      userId={match.params.id}
      onBack={() => history.goBack()}
    />
  )}
/>
```

### Formik — Form Library

```tsx
import { Formik } from 'formik';

<Formik
  initialValues={{ email: '', password: '' }}
  onSubmit={(values) => handleLogin(values)}
>
  {({ values, errors, handleChange, handleSubmit, isSubmitting }) => (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        value={values.email}
        onChange={handleChange}
      />
      {errors.email && <span>{errors.email}</span>}
      <button disabled={isSubmitting} type="submit">
        {isSubmitting ? 'Đang đăng nhập...' : 'Đăng nhập'}
      </button>
    </form>
  )}
</Formik>
```

### Downshift — Accessible Combobox

```tsx
import Downshift from 'downshift';

<Downshift
  onChange={(item) => console.log('selected:', item)}
  itemToString={(item) => item?.label ?? ''}
>
  {({
    getInputProps,
    getItemProps,
    getMenuProps,
    isOpen,
    inputValue,
    highlightedIndex,
  }) => (
    <div>
      <input {...getInputProps({ placeholder: 'Tìm kiếm...' })} />
      {isOpen && (
        <ul {...getMenuProps()}>
          {items
            .filter((item) => item.label.includes(inputValue ?? ''))
            .map((item, index) => (
              <li
                key={item.value}
                {...getItemProps({ item, index })}
                style={{
                  backgroundColor: highlightedIndex === index ? '#eee' : 'white',
                }}
              >
                {item.label}
              </li>
            ))}
        </ul>
      )}
    </div>
  )}
</Downshift>
```

---

## 7. Vấn Đề Hiệu Năng Của Render Props

### Vấn Đề: Hàm Inline Tạo Mới Mỗi Render

```tsx
// ❌ Hàm render được tạo mới mỗi lần App re-render
function App() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <MouseTracker
        render={(pos) => <Cat position={pos} />}  // Hàm mới mỗi render!
      />
    </>
  );
}
```

### Giải Pháp: Tách Ra Hoặc Dùng useCallback

```tsx
// ✅ Cách 1: Tách hàm render ra ngoài component
const renderCat = (pos: Position) => <Cat position={pos} />;

function App() {
  return <MouseTracker render={renderCat} />;
}

// ✅ Cách 2: useCallback khi cần closure
function App() {
  const [name, setName] = useState('Mèo');

  const renderAnimal = useCallback(
    (pos: Position) => <Animal name={name} position={pos} />,
    [name]
  );

  return <MouseTracker render={renderAnimal} />;
}
```

---

## 8. Render Props vs Custom Hooks — So Sánh

| Tiêu Chí | Render Props | Custom Hooks |
| -------- | ------------ | ------------ |
| **Syntax** (Cú pháp) | JSX, verbose | Hook call, gọn hơn |
| **Composability** (Kết hợp) | Nested, callback hell | Dễ compose nhiều hooks |
| **TypeScript** | Phức tạp hơn | Tự nhiên, dễ infer |
| **Debug** | Nhiều wrapper trong DevTools | Ít wrapper hơn |
| **Use in class components** | ✅ Có | ❌ Không |
| **Render control** (Kiểm soát render) | ✅ Component quyết định | ❌ Caller quyết định |

### Cùng Logic — Hai Cách Viết

```tsx
// --- Render Props (cũ hơn) ---
function WithMousePosition({ render }: { render: (pos: Position) => ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  // ... event listener
  return <>{render(pos)}</>;
}

function App() {
  return (
    <WithMousePosition
      render={({ x, y }) => <p>X: {x}, Y: {y}</p>}
    />
  );
}

// --- Custom Hook (hiện đại) ---
function useMousePosition() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  // ... event listener
  return pos;
}

function App() {
  const { x, y } = useMousePosition();
  return <p>X: {x}, Y: {y}</p>;
}
```

### Khi Nào Vẫn Nên Dùng Render Props

```
✅ Khi cần render control — component con quyết định KỊCH BẢN render
✅ Khi làm việc với class components (không dùng hooks được)
✅ Khi cần library compatibility (Formik, Downshift, React Router v5)
✅ Khi cần truyền render context rõ ràng trong JSX tree

❌ Đừng dùng khi Custom Hook đủ → simpler, more composable
```

---

## 9. Anti-pattern: Callback Hell Với Render Props Lồng Nhau

```tsx
// ❌ Pyramid of doom — khó đọc, khó debug
function App() {
  return (
    <WithAuth>
      {({ user }) => (
        <WithPermissions userId={user.id}>
          {({ permissions }) => (
            <WithTheme>
              {({ theme }) => (
                <WithLocale>
                  {({ locale }) => (
                    <Dashboard
                      user={user}
                      permissions={permissions}
                      theme={theme}
                      locale={locale}
                    />
                  )}
                </WithLocale>
              )}
            </WithTheme>
          )}
        </WithPermissions>
      )}
    </WithAuth>
  );
}

// ✅ Refactor sang Custom Hooks
function Dashboard() {
  const { user } = useAuth();
  const { permissions } = usePermissions(user.id);
  const { theme } = useTheme();
  const { locale } = useLocale();

  return <DashboardUI user={user} permissions={permissions} theme={theme} locale={locale} />;
}
```

---

## 📝 Checklist Tự Đánh Giá

- [ ] Giải thích được Render Props pattern trong 1-2 câu
- [ ] Implement được cả `render prop` và `children as function`
- [ ] Biết cách tránh vấn đề hiệu năng với `useCallback`
- [ ] Quyết định được khi nào Render Props vs Custom Hooks
- [ ] Nhận ra pattern này trong Formik, React Router v5

---

## 🔗 Điều Hướng

- **Trước đó:** [1-compound-components.md](./1-compound-components.md) — Compound Components
- **Tiếp theo:** [3-hoc.md](./3-hoc.md) — Higher-Order Components
- **Chỉ mục:** [INDEX.md](../INDEX.md)
