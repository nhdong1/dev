# 📝 Câu Hỏi Phỏng Vấn Phổ Biến — Mọi Cấp Độ

> Bộ câu hỏi nền tảng về React mà hầu hết mọi cuộc phỏng vấn đều hỏi, từ Junior đến Mid-level. Nắm chắc phần này là điều kiện cần để pass vòng kỹ thuật.

---

## 🗂️ Mục Lục

1. [JSX & Components](#jsx--components)
2. [Props & State](#props--state)
3. [Lifecycle & Side Effects](#lifecycle--side-effects)
4. [Lists & Rendering](#lists--rendering)
5. [Forms & Events](#forms--events)
6. [Component Architecture](#component-architecture)
7. [Bài Tập Live Coding](#bài-tập-live-coding)

---

## JSX & Components

---

### Câu 1: Functional Component và Class Component — Component Lớp khác nhau như thế nào?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

React hiện tại khuyến khích **Functional Components — Component Hàm** với Hooks thay vì Class Components — Component Lớp (được coi là legacy — cũ).

```jsx
// Class Component — Cách cũ (legacy)
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.handleClick = this.handleClick.bind(this); // Phải bind this
  }

  componentDidMount() {
    document.title = `Count: ${this.state.count}`;
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.count !== this.state.count) {
      document.title = `Count: ${this.state.count}`;
    }
  }

  componentWillUnmount() {
    // Cleanup
  }

  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        Count: {this.state.count}
      </button>
    );
  }
}

// Functional Component — Cách hiện đại
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Count: ${count}`;
    return () => { document.title = 'App'; }; // Cleanup
  }, [count]);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

| Đặc Điểm | Class Component | Functional Component |
|-----------|-----------------|----------------------|
| Cú pháp | Phức tạp hơn | Đơn giản hơn |
| this | Phải bind, dễ nhầm | Không có this |
| Lifecycle | componentDidMount/Update/Unmount | useEffect |
| Logic tái sử dụng | HOC, Render Props (cồng kềnh) | Custom Hooks (gọn gàng) |
| Performance | Slightly nặng hơn | Nhẹ hơn |
| React team khuyến nghị | ❌ (legacy) | ✅ (hiện đại) |

---

### Câu 2: Tại sao không được return nhiều root elements trong JSX?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

JSX được biên dịch thành `React.createElement()` — hàm này chỉ trả về một object. Không thể return hai objects cùng lúc.

```jsx
// ❌ Lỗi — hai root elements
function BadComponent() {
  return (
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  );
}

// ✅ Giải pháp 1: Bọc trong div
function Solution1() {
  return (
    <div>
      <h1>Tiêu đề</h1>
      <p>Nội dung</p>
    </div>
  );
}

// ✅ Giải pháp 2: Fragment — Mảnh (không tạo DOM node)
function Solution2() {
  return (
    <>
      <h1>Tiêu đề</h1>
      <p>Nội dung</p>
    </>
  );
}

// ✅ Giải pháp 3: React.Fragment (khi cần key)
function ListItems({ items }) {
  return items.map(item => (
    <React.Fragment key={item.id}>
      <dt>{item.term}</dt>
      <dd>{item.description}</dd>
    </React.Fragment>
  ));
}
```

**Fragment — Mảnh** là giải pháp tốt hơn `<div>` bọc ngoài vì không tạo thêm DOM node không cần thiết, giúp giữ cấu trúc HTML semantic.

---

### Câu 3: Component thuần (Pure Component) — Purity trong React là gì?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

React yêu cầu components là **pure functions — hàm thuần** — nghĩa là:

1. **Deterministic — Xác Định:** Cùng input (props, state) → luôn cùng output (JSX)
2. **No side effects — Không tác dụng phụ trong render:** Không thay đổi DOM, không fetch data, không ghi biến ngoài trong hàm render

```jsx
// ❌ Impure — không thuần: thay đổi biến ngoài trong render
let count = 0;
function Counter() {
  count++; // Side effect trong render — sai!
  return <div>{count}</div>;
}

// ❌ Impure: đọc random/Date trong render (non-deterministic)
function Greeting() {
  return <div>Xin chào lúc {new Date().toString()}</div>; // Mỗi render khác nhau
}

// ✅ Pure: cùng props → cùng output
function Greeting({ name, time }) {
  return <div>Xin chào {name}, lúc {time}</div>;
}

// ✅ Side effects thuộc về useEffect, event handlers — ngoài render
function DataFetcher({ userId }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setData);
  }, [userId]); // Fetch trong useEffect — đúng

  return <div>{data?.name}</div>;
}
```

**Tại sao quan trọng?** React 18+ với Strict Mode và Concurrent Features có thể gọi render function nhiều lần. Nếu component không pure, kết quả không thể đoán trước.

---

## Props & State

---

### Câu 4: Làm thế nào để truyền dữ liệu từ con lên cha (lifting state up — nâng state lên)?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

Dữ liệu trong React chảy một chiều từ cha xuống con. Để truyền dữ liệu ngược lại, cha truyền **callback function** xuống con qua props:

```jsx
// ❌ Sai — con không thể modify props trực tiếp
function Child({ value }) {
  value = 'new value'; // Không được!
}

// ✅ Đúng — Lifting State Up — Nâng State Lên
function Parent() {
  const [selectedItem, setSelectedItem] = useState(null);

  return (
    <div>
      <ChildList onSelect={setSelectedItem} />
      {selectedItem && <Detail item={selectedItem} />}
    </div>
  );
}

function ChildList({ onSelect }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onSelect(item)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

**Khi nào lifting state up không đủ?** Khi nhiều component ở các nhánh khác nhau của tree cần cùng dữ liệu → dùng Context API hoặc state management library.

---

### Câu 5: Default Props — Props Mặc Định và PropTypes — Kiểm Tra Kiểu Props là gì?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

```jsx
// Default Props với ES6 default parameters (cách hiện đại)
function Button({ label = 'Click me', variant = 'primary', onClick }) {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {label}
    </button>
  );
}

// PropTypes — Kiểm Tra Kiểu Runtime (JavaScript)
import PropTypes from 'prop-types';

Button.propTypes = {
  label: PropTypes.string,
  variant: PropTypes.oneOf(['primary', 'secondary', 'danger']),
  onClick: PropTypes.func.isRequired,
};

Button.defaultProps = { // Cách cũ, vẫn hoạt động
  label: 'Click me',
  variant: 'primary',
};
```

**TypeScript thay thế PropTypes (khuyến nghị):**
```tsx
interface ButtonProps {
  label?: string;           // Optional
  variant?: 'primary' | 'secondary' | 'danger';
  onClick: () => void;      // Required
}

function Button({ label = 'Click me', variant = 'primary', onClick }: ButtonProps) {
  return <button className={`btn btn-${variant}`} onClick={onClick}>{label}</button>;
}
```

**Lưu ý:** Trong TypeScript, PropTypes thường không cần thiết vì TypeScript kiểm tra tại compile time, PropTypes kiểm tra tại runtime.

---

### Câu 6: useState — cập nhật state bất đồng bộ như thế nào?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

State updates trong React là **asynchronous — bất đồng bộ** và **batched — được gộp lại**:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    // Kết quả: count tăng 1, không phải 3!
    // Ba setCount đều đọc cùng giá trị count cũ (stale closure)
    console.log(count); // Vẫn in giá trị cũ!
  };
}
```

**Fix với functional updater — Cập nhật theo hàm:**
```jsx
const handleClick = () => {
  setCount(prev => prev + 1); // Luôn dùng giá trị mới nhất
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  // Kết quả: count tăng 3 ✅
};
```

**Batching — Gộp Updates (React 18+):**
```jsx
// React 18 tự động batch tất cả updates, kể cả trong async code
async function handleSubmit() {
  setLoading(true);
  const data = await fetchData();
  setLoading(false);  // React 18: hai updates này được batch thành một render
  setData(data);       // React 17: async context → render riêng
}

// Nếu muốn force update riêng (hiếm khi cần):
import { flushSync } from 'react-dom';
flushSync(() => setLoading(true)); // Render ngay lập tức
```

---

## Lifecycle & Side Effects

---

### Câu 7: Mô phỏng componentDidMount, componentDidUpdate, componentWillUnmount với useEffect

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐⭐⭐

**Trả lời:**

```jsx
function MyComponent({ userId }) {
  const [data, setData] = useState(null);

  // componentDidMount — Chạy sau lần render đầu tiên
  useEffect(() => {
    console.log('Component mounted — đã mount');
  }, []); // Dependencies rỗng = chỉ chạy 1 lần

  // componentDidUpdate (khi userId thay đổi)
  useEffect(() => {
    fetchUser(userId).then(setData);
  }, [userId]); // Chạy lại khi userId thay đổi

  // componentWillUnmount — Cleanup khi unmount
  useEffect(() => {
    const subscription = subscribeToUser(userId);

    return () => {
      subscription.unsubscribe(); // Chạy khi unmount hoặc trước khi effect chạy lại
    };
  }, [userId]);

  return <div>{data?.name}</div>;
}
```

**Lưu ý thứ tự:**
```
Mount:    render() → DOM update → useEffect cleanup(nếu có) → useEffect
Update:   render() → DOM update → cleanup(effect cũ) → useEffect(mới)
Unmount:  cleanup(effect cuối)
```

---

### Câu 8: Tại sao không nên fetch data trực tiếp trong render function?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

```jsx
// ❌ Sai — fetch trong render tạo vòng lặp vô tận
function DataComponent() {
  const [data, setData] = useState(null);

  // Fetch trong render → setData → re-render → fetch lại → ...∞
  fetch('/api/data').then(r => r.json()).then(setData);

  return <div>{data?.name}</div>;
}

// ✅ Đúng — fetch trong useEffect
function DataComponent({ id }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false; // Tránh race condition

    setLoading(true);
    fetch(`/api/data/${id}`)
      .then(r => {
        if (!r.ok) throw new Error('Network error');
        return r.json();
      })
      .then(data => {
        if (!cancelled) setData(data);
      })
      .catch(err => {
        if (!cancelled) setError(err.message);
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => { cancelled = true; }; // Cleanup tránh setState sau unmount
  }, [id]);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <div>{data?.name}</div>;
}

// ✅✅ Tốt hơn — dùng TanStack Query
const { data, isLoading, error } = useQuery({
  queryKey: ['data', id],
  queryFn: () => fetch(`/api/data/${id}`).then(r => r.json()),
});
```

---

## Lists & Rendering

---

### Câu 9: Conditional Rendering — Render Có Điều Kiện các cách thực hiện

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

```jsx
function Dashboard({ user, isLoading, notifications }) {
  // 1. if/else statement
  if (isLoading) return <LoadingSpinner />;

  // 2. Ternary operator — Toán tử ba ngôi
  return (
    <div>
      {user ? <UserGreeting user={user} /> : <GuestGreeting />}

      {/* 3. Short-circuit evaluation — Đánh giá ngắn mạch (chú ý!) */}
      {notifications.length > 0 && <NotificationBadge count={notifications.length} />}

      {/* ❌ Gotcha: 0 sẽ render ra "0", không phải falsy */}
      {notifications.length && <Badge />} {/* Bug: render "0" khi length = 0! */}
      {/* ✅ Fix */}
      {notifications.length > 0 && <Badge />}

      {/* 4. nullish coalescing với tùy chọn */}
      {user?.role === 'admin' && <AdminPanel />}
    </div>
  );
}
```

---

### Câu 10: Khi nào dùng index làm key được? Khi nào không?

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐⭐

**Trả lời:**

**Dùng index làm key được khi** cả 3 điều kiện đều thỏa:
1. Danh sách **không thay đổi thứ tự**
2. Các items **không có stable ID**
3. Danh sách **không bị filter hoặc sort**

```jsx
// ✅ OK dùng index — static list, không reorder
function StaticLinks({ links }) {
  return links.map((link, index) => (
    <a key={index} href={link.url}>{link.text}</a>
  ));
}

// ❌ Không dùng index — dynamic list có thể sort/filter/delete
function TodoList({ todos }) {
  return todos.map((todo, index) => (
    <TodoItem
      key={index} // ❌ Bug khi delete item giữa danh sách
      todo={todo}
    />
  ));
}

// ✅ Dùng ID ổn định
function TodoList({ todos }) {
  return todos.map(todo => (
    <TodoItem key={todo.id} todo={todo} />
  ));
}
```

**Tại sao bug?** Khi xóa item index 1 từ `[A(0), B(1), C(2)]`:
- Còn lại `[A(0), C(1)]` — C giờ có key=1 (cũ là key=2)
- React nghĩ C là B (vì cùng key=1) → state nội bộ của C bị nhầm với state của B

---

## Forms & Events

---

### Câu 11: Xử lý form phức tạp — nhiều fields

**Cấp độ:** Junior/Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

```jsx
// Cách 1: State riêng cho mỗi field (đơn giản nhưng verbose)
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <form>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <input value={password} onChange={e => setPassword(e.target.value)} />
    </form>
  );
}

// Cách 2: Object state với một handler (gọn hơn)
function RegistrationForm() {
  const [form, setForm] = useState({
    name: '',
    email: '',
    password: '',
    confirmPassword: '',
  });

  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setForm(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value,
    }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    // Validate và submit
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" value={form.name} onChange={handleChange} />
      <input name="email" value={form.email} onChange={handleChange} />
      <input name="password" type="password" value={form.password} onChange={handleChange} />
    </form>
  );
}

// Cách 3: React Hook Form (khuyến nghị cho production)
// Uncontrolled với ref, performance tốt hơn nhiều
import { useForm } from 'react-hook-form';

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email', { required: true, pattern: /\S+@\S+/ })} />
      {errors.email && <span>Email không hợp lệ</span>}
    </form>
  );
}
```

---

### Câu 12: Event Handling — Xử Lý Sự Kiện trong React khác gì HTML thuần?

**Cấp độ:** Junior | **Tần suất:** ⭐⭐⭐

**Trả lời:**

| Đặc Điểm | HTML thuần | React |
|----------|------------|-------|
| Tên event | `onclick`, `onchange` (lowercase) | `onClick`, `onChange` (camelCase) |
| Giá trị | String function | Function reference |
| Ngăn default | `return false` | `event.preventDefault()` |
| Event object | Native browser event | SyntheticEvent — Sự Kiện Tổng Hợp |

```jsx
// HTML thuần
<button onclick="handleClick()">Click</button>

// React
<button onClick={handleClick}>Click</button>           // ✅
<button onClick={() => handleClick()}>Click</button>   // ✅ (anonymous function)
<button onClick={handleClick()}>Click</button>         // ❌ Gọi ngay khi render!

// Ngăn hành vi mặc định
function Form() {
  const handleSubmit = (event) => {
    event.preventDefault(); // Ngăn form submit và reload trang
    // Xử lý logic submit
  };

  return <form onSubmit={handleSubmit}>...</form>;
}

// SyntheticEvent — Sự Kiện Tổng Hợp
// React bọc native events để normalize cross-browser differences
// Trong React 17+, events không được pool nên có thể dùng async
const handleChange = (event) => {
  const value = event.target.value; // Trong React 17+ có thể dùng async
  console.log(value);
};
```

---

## Component Architecture

---

### Câu 13: Phân biệt Container Components và Presentational Components — Component Trình Bày

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

Pattern phân tách logic và UI — không bắt buộc nhưng giúp code dễ test và tái sử dụng:

```jsx
// Container Component — Chứa Logic
// Biết data từ đâu, không quan tâm UI render thế nào
function UserListContainer() {
  const { data: users, isLoading } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  if (isLoading) return <LoadingSpinner />;

  return <UserList users={users} />;
}

// Presentational Component — Component Trình Bày
// Không biết data từ đâu, chỉ render dựa trên props
function UserList({ users }) {
  return (
    <ul className="user-list">
      {users.map(user => (
        <li key={user.id} className="user-item">
          <img src={user.avatar} alt={user.name} />
          <span>{user.name}</span>
        </li>
      ))}
    </ul>
  );
}
```

**Lưu ý hiện đại:** Với Custom Hooks, không nhất thiết phải tách thành hai component riêng:

```jsx
// Hook chứa logic, component chứa UI
function UserList() {
  const { users, isLoading } = useUsers(); // Custom hook trích xuất logic

  if (isLoading) return <Spinner />;
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

---

### Câu 14: Component Composition — Thành Phần Kết Hợp vs Inheritance — Kế Thừa trong React

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

**Trả lời:**

React khuyến nghị **Composition over Inheritance** — Ưu Tiên Thành Phần Hơn Kế Thừa.

```jsx
// ❌ Inheritance — Không được khuyến nghị trong React
class FancyButton extends Button {
  // Khó hiểu, khó test, khó bảo trì
}

// ✅ Composition — Thành Phần
// 1. Children prop
function Dialog({ title, children }) {
  return (
    <div className="dialog">
      <h2>{title}</h2>
      <div className="dialog-body">{children}</div>
    </div>
  );
}

function ConfirmDialog({ onConfirm, onCancel }) {
  return (
    <Dialog title="Xác Nhận">
      <p>Bạn có chắc không?</p>
      <button onClick={onConfirm}>Đồng ý</button>
      <button onClick={onCancel}>Hủy</button>
    </Dialog>
  );
}

// 2. Render props hoặc named slots
function Layout({ header, sidebar, content }) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{content}</main>
    </div>
  );
}

<Layout
  header={<Navbar />}
  sidebar={<FilterPanel />}
  content={<ProductList />}
/>
```

---

## 🖥️ Bài Tập Live Coding

---

### Live Coding 1: Implement component Toggle với custom hook

```jsx
// Yêu cầu: Viết useToggle hook và dùng nó trong ToggleButton
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse };
}

function ToggleButton() {
  const { value: isOn, toggle } = useToggle(false);

  return (
    <button
      onClick={toggle}
      style={{ background: isOn ? 'green' : 'gray' }}
    >
      {isOn ? 'Bật' : 'Tắt'}
    </button>
  );
}
```

---

### Live Coding 2: Implement Accordion — Danh Sách Thu Gọn

```jsx
function Accordion({ items }) {
  const [openIndex, setOpenIndex] = useState(null);

  return (
    <div className="accordion">
      {items.map((item, index) => {
        const isOpen = openIndex === index;
        return (
          <div key={item.id} className="accordion-item">
            <button
              className="accordion-header"
              onClick={() => setOpenIndex(isOpen ? null : index)}
            >
              {item.title}
              <span>{isOpen ? '▲' : '▼'}</span>
            </button>
            {isOpen && (
              <div className="accordion-content">
                {item.content}
              </div>
            )}
          </div>
        );
      })}
    </div>
  );
}
```

---

### Live Coding 3: Implement Search với Debounce — Trì Hoãn Tìm Kiếm

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

function SearchBox({ onSearch }) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);

  useEffect(() => {
    if (debouncedQuery) {
      onSearch(debouncedQuery); // Chỉ gọi khi user dừng gõ 500ms
    }
  }, [debouncedQuery, onSearch]);

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      {query !== debouncedQuery && <span>Đang tìm...</span>}
    </div>
  );
}
```

---

### Live Coding 4: Implement Infinite Scroll — Cuộn Vô Hạn

```jsx
function useIntersectionObserver(callback, options) {
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) callback(); },
      options
    );

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, [callback, options]);

  return ref;
}

function InfiniteList() {
  const [page, setPage] = useState(1);
  const [items, setItems] = useState([]);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = useCallback(async () => {
    if (!hasMore) return;
    const newItems = await fetchItems(page);
    if (newItems.length === 0) {
      setHasMore(false);
    } else {
      setItems(prev => [...prev, ...newItems]);
      setPage(p => p + 1);
    }
  }, [page, hasMore]);

  const loaderRef = useIntersectionObserver(loadMore, { threshold: 0.1 });

  return (
    <div>
      {items.map(item => <ItemCard key={item.id} item={item} />)}
      {hasMore && <div ref={loaderRef}>Đang tải...</div>}
    </div>
  );
}
```

---

## ✅ Checklist Tự Đánh Giá — Junior/Mid

Bạn đã sẵn sàng khi có thể trả lời không cần notes:

**JSX & Components:**
- [ ] Giải thích JSX được compile thành gì
- [ ] Phân biệt Functional vs Class Component
- [ ] Biết khi nào dùng Fragment

**Props & State:**
- [ ] Giải thích data flow một chiều
- [ ] Lifting State Up — khi nào cần
- [ ] Functional updater trong useState — tại sao cần

**Lifecycle & Effects:**
- [ ] Map 3 lifecycle methods sang useEffect patterns
- [ ] Cleanup function — khi nào cần
- [ ] Stale closure cơ bản

**Lists & Rendering:**
- [ ] Tại sao cần key — index làm key khi nào OK
- [ ] Conditional rendering 3 cách
- [ ] Short-circuit với 0 — gotcha phổ biến

**Forms:**
- [ ] Controlled vs Uncontrolled — trade-offs
- [ ] Xử lý nhiều fields với một handler

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
