# 2 — React Testing Library (RTL)

> **React Testing Library** — RTL (Thư Viện Kiểm Thử React) — là bộ công cụ kiểm thử React components theo triết lý "test behavior, not implementation" (kiểm thử hành vi, không phải chi tiết triển khai). RTL cung cấp các utilities để render component, truy vấn DOM, và mô phỏng tương tác người dùng.

---

## 🎯 Mục Tiêu

- [ ] Render component trong môi trường test với `render()`
- [ ] Truy vấn DOM bằng **queries** (truy vấn) theo thứ tự ưu tiên đúng: `getByRole`, `getByLabelText`, ...
- [ ] Mô phỏng tương tác người dùng với `userEvent` và `fireEvent`
- [ ] Kiểm thử **async behavior** (hành vi bất đồng bộ) với `waitFor`, `findBy`
- [ ] Test **custom hooks** với `renderHook`
- [ ] Sử dụng các **matchers từ jest-dom**: `toBeInTheDocument`, `toBeVisible`, ...

---

## 📦 Cài Đặt

```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom @testing-library/user-event

# Với TypeScript
npm install --save-dev @types/testing-library__jest-dom
```

```typescript
// jest.setup.ts — import jest-dom matchers toàn cục
import '@testing-library/jest-dom';
```

```typescript
// jest.config.ts
export default {
  setupFilesAfterFramework: ['<rootDir>/jest.setup.ts'],
};
```

---

## 🎨 Render — Hiển Thị Component

### `render()` — Hàm Cơ Bản

```typescript
import { render, screen } from '@testing-library/react';

function Greeting({ name }: { name: string }) {
  return <h1>Xin chào, {name}!</h1>;
}

test('should render greeting with name', () => {
  render(<Greeting name="An" />);
  
  // screen — đối tượng đại diện cho toàn bộ document
  expect(screen.getByText('Xin chào, An!')).toBeInTheDocument();
});
```

### Render Với Providers (Context, Router, ...)

```typescript
import { render } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';

// Custom render — bọc providers cần thiết
function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false }, // Tắt retry trong test
    },
  });

  return render(
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {ui}
      </BrowserRouter>
    </QueryClientProvider>
  );
}

// Dùng như render() bình thường
test('should show user profile', async () => {
  renderWithProviders(<UserProfile userId="1" />);
  await screen.findByText('Nguyen Van A');
});
```

---

## 🔍 Queries — Truy Vấn DOM

### Thứ Tự Ưu Tiên (Theo RTL Guidelines)

```
Ưu tiên cao nhất → Ưu tiên thấp nhất

1. getByRole          ← Accessible queries (truy vấn có khả năng tiếp cận)
2. getByLabelText     ← Form elements
3. getByPlaceholderText
4. getByText          ← Non-interactive elements
5. getByDisplayValue  ← Form với giá trị hiện tại
6. getByAltText       ← Images
7. getByTitle         ← title attribute
8. getByTestId        ← Chỉ dùng khi không còn lựa chọn nào khác
```

### Biến Thể Query (Prefix)

| Prefix | Không Tìm Thấy | Tìm Thấy Nhiều | Async | Khi Dùng |
| ------ | -------------- | -------------- | ----- | --------- |
| `getBy` | ❌ throw error | ❌ throw error | ❌ | Element chắc chắn có mặt |
| `queryBy` | ✅ null | ❌ throw error | ❌ | Kiểm tra element KHÔNG có mặt |
| `findBy` | ❌ throw error | ❌ throw error | ✅ | Element xuất hiện sau async operation |
| `getAllBy` | ❌ throw error | ✅ array | ❌ | Nhiều elements |
| `queryAllBy` | ✅ [] | ✅ array | ❌ | Kiểm tra list có thể rỗng |
| `findAllBy` | ❌ throw error | ✅ array | ✅ | Nhiều elements sau async |

### getByRole — Query Ưu Tiên Nhất

```typescript
// Role là ARIA role — vai trò trong cây accessibility
// Xem đầy đủ tại: https://www.w3.org/TR/html-aria/

render(
  <div>
    <button>Submit</button>
    <input type="text" aria-label="Search" />
    <h1>Page Title</h1>
    <nav>Navigation</nav>
    <img src="logo.png" alt="Logo" />
    <a href="/about">About</a>
    <ul><li>Item 1</li><li>Item 2</li></ul>
  </div>
);

screen.getByRole('button', { name: /submit/i });     // <button>
screen.getByRole('textbox', { name: /search/i });    // <input type="text">
screen.getByRole('heading', { level: 1 });           // <h1>
screen.getByRole('navigation');                       // <nav>
screen.getByRole('img', { name: /logo/i });          // <img alt="Logo">
screen.getByRole('link', { name: /about/i });        // <a>
screen.getAllByRole('listitem');                       // <li> elements
```

### getByLabelText — Form Elements

```typescript
render(
  <form>
    <label htmlFor="email">Email</label>
    <input id="email" type="email" />
    
    <label>
      Password
      <input type="password" />
    </label>
    
    <input type="text" aria-label="Search query" />
  </form>
);

screen.getByLabelText('Email');         // Input liên kết qua htmlFor
screen.getByLabelText('Password');      // Input bên trong label
screen.getByLabelText('Search query'); // Input với aria-label
```

### getByText vs queryByText

```typescript
function ConditionalMessage({ show }: { show: boolean }) {
  return <div>{show && <p>Thông báo quan trọng</p>}</div>;
}

// getByText — Dùng khi chắc chắn element CÓ mặt
test('shows message when show=true', () => {
  render(<ConditionalMessage show={true} />);
  expect(screen.getByText('Thông báo quan trọng')).toBeInTheDocument(); // ✅
});

// queryByText — Dùng khi kiểm tra element KHÔNG có mặt
test('hides message when show=false', () => {
  render(<ConditionalMessage show={false} />);
  expect(screen.queryByText('Thông báo quan trọng')).not.toBeInTheDocument(); // ✅
  // getByText sẽ throw error ở đây → không dùng được để test "not present"
});
```

### findBy — Async Query

```typescript
function LoadingUser({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetchUser(id).then(setUser);
  }, [id]);

  if (!user) return <p>Đang tải...</p>;
  return <p>{user.name}</p>;
}

test('should display user name after loading', async () => {
  render(<LoadingUser id="1" />);
  
  // Đợi async element xuất hiện (mặc định timeout 1000ms)
  const userName = await screen.findByText('Nguyen Van A');
  expect(userName).toBeInTheDocument();
});
```

### getByTestId — Phương Án Cuối Cùng

```typescript
// Chỉ dùng khi không thể dùng semantic queries
// Ưu điểm: ổn định, không bị ảnh hưởng bởi thay đổi text/role
// Nhược điểm: không phản ánh cách user tương tác

// Trong component
<div data-testid="product-card-{product.id}">
  {/* ... */}
</div>

// Trong test
const card = screen.getByTestId('product-card-1');
```

---

## 🖱️ User Events — Mô Phỏng Tương Tác Người Dùng

### userEvent vs fireEvent

```
userEvent (khuyến nghị):
  → Mô phỏng tương tác người dùng thực tế
  → Click → focus → hover → mousedown → mouseup → click
  → Gõ chữ → keydown → keypress → keyup → input → change
  → Phù hợp với cách browser thực sự xử lý events

fireEvent (ít dùng hơn):
  → Dispatch trực tiếp một DOM event
  → Nhanh hơn nhưng kém realistic
  → Dùng khi userEvent không hỗ trợ event cụ thể
```

### Setup userEvent

```typescript
import userEvent from '@testing-library/user-event';

// ✅ Cách đúng: setup userEvent một lần trước mỗi test
const user = userEvent.setup();

// Trong test
await user.click(button);
await user.type(input, 'Hello World');
```

### Click Events

```typescript
test('should increment counter on click', async () => {
  const user = userEvent.setup();
  
  render(<Counter initialCount={0} />);
  
  const button = screen.getByRole('button', { name: /tăng/i });
  
  await user.click(button);
  expect(screen.getByText('1')).toBeInTheDocument();
  
  await user.click(button);
  await user.click(button);
  expect(screen.getByText('3')).toBeInTheDocument();
});
```

### Typing Events — Gõ Chữ

```typescript
test('should update input value as user types', async () => {
  const user = userEvent.setup();
  const handleChange = jest.fn();
  
  render(<input type="text" onChange={handleChange} />);
  
  const input = screen.getByRole('textbox');
  
  await user.type(input, 'Hello');
  
  expect(input).toHaveValue('Hello');        // ✅ value đúng
  expect(handleChange).toHaveBeenCalledTimes(5); // Mỗi ký tự = 1 event
});

// Xóa nội dung và gõ lại
await user.clear(input);
await user.type(input, 'New content');

// Gõ với special keys (phím đặc biệt)
await user.type(input, 'Hello{Enter}'); // {Enter}, {Tab}, {Backspace}
```

### Keyboard Events — Phím Tắt

```typescript
test('should submit form on Enter', async () => {
  const user = userEvent.setup();
  const onSubmit = jest.fn();
  
  render(
    <form onSubmit={onSubmit}>
      <input type="text" />
    </form>
  );
  
  const input = screen.getByRole('textbox');
  await user.click(input);
  await user.keyboard('{Enter}');
  
  expect(onSubmit).toHaveBeenCalled();
});
```

### Select và Checkbox

```typescript
test('should select option and check checkbox', async () => {
  const user = userEvent.setup();
  
  render(
    <form>
      <select aria-label="Màu sắc">
        <option value="red">Đỏ</option>
        <option value="blue">Xanh</option>
      </select>
      <input type="checkbox" aria-label="Đồng ý điều khoản" />
    </form>
  );
  
  // Select
  await user.selectOptions(
    screen.getByRole('combobox', { name: /màu sắc/i }),
    'blue'
  );
  expect(screen.getByRole('combobox')).toHaveValue('blue');
  
  // Checkbox
  const checkbox = screen.getByRole('checkbox', { name: /đồng ý/i });
  expect(checkbox).not.toBeChecked();
  await user.click(checkbox);
  expect(checkbox).toBeChecked();
});
```

---

## ⏳ Async Testing — Kiểm Thử Bất Đồng Bộ

### waitFor — Đợi Điều Kiện Thỏa Mãn

```typescript
import { render, screen, waitFor } from '@testing-library/react';

test('should show success message after form submit', async () => {
  const user = userEvent.setup();
  render(<ContactForm />);
  
  await user.type(screen.getByLabelText(/email/i), 'an@example.com');
  await user.click(screen.getByRole('button', { name: /gửi/i }));
  
  // Đợi cho đến khi success message xuất hiện
  await waitFor(() => {
    expect(screen.getByText(/gửi thành công/i)).toBeInTheDocument();
  });
});

// waitFor với timeout tùy chỉnh
await waitFor(
  () => expect(screen.getByText('Done')).toBeInTheDocument(),
  { timeout: 3000, interval: 100 } // Kiểm tra mỗi 100ms, tối đa 3s
);
```

### waitForElementToBeRemoved

```typescript
test('should hide loading spinner after data loads', async () => {
  render(<DataTable />);
  
  // Đợi spinner biến mất
  await waitForElementToBeRemoved(() => screen.queryByRole('status'));
  
  // Sau đó kiểm tra data đã hiện
  expect(screen.getAllByRole('row')).toHaveLength(11); // header + 10 rows
});
```

---

## 🪝 renderHook — Test Custom Hooks

```typescript
import { renderHook, act } from '@testing-library/react';

function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  
  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  const reset = useCallback(() => setCount(initialValue), [initialValue]);
  
  return { count, increment, decrement, reset };
}

test('should increment counter', () => {
  const { result } = renderHook(() => useCounter(0));
  
  expect(result.current.count).toBe(0);
  
  // act — bọc tất cả actions gây ra state update
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});

test('should reset to initial value', () => {
  const { result } = renderHook(() => useCounter(10));
  
  act(() => {
    result.current.increment();
    result.current.increment();
  });
  
  expect(result.current.count).toBe(12);
  
  act(() => {
    result.current.reset();
  });
  
  expect(result.current.count).toBe(10);
});
```

### renderHook Với Provider

```typescript
test('useAuth hook should return user', async () => {
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <AuthProvider initialUser={{ name: 'An' }}>
      {children}
    </AuthProvider>
  );

  const { result } = renderHook(() => useAuth(), { wrapper });
  
  expect(result.current.user?.name).toBe('An');
});
```

---

## 🎭 Jest-DOM Matchers — Bộ So Khớp DOM

```typescript
// Sau khi import '@testing-library/jest-dom'

const element = screen.getByRole('button');

// Hiện diện trong DOM
expect(element).toBeInTheDocument();
expect(element).not.toBeInTheDocument();

// Hiển thị/ẩn
expect(element).toBeVisible();
expect(element).not.toBeVisible(); // display: none, visibility: hidden, opacity: 0

// Trạng thái form
expect(checkbox).toBeChecked();
expect(input).toBeDisabled();
expect(input).toBeEnabled();
expect(input).toBeRequired();
expect(input).toBeInvalid();
expect(input).toBeValid();

// Giá trị
expect(input).toHaveValue('Hello');
expect(select).toHaveValue('option-value');
expect(textarea).toHaveValue('');

// Text content
expect(element).toHaveTextContent('Xin chào');
expect(element).toHaveTextContent(/xin chào/i); // Regex

// Attributes
expect(link).toHaveAttribute('href', '/about');
expect(img).toHaveAttribute('alt', 'Logo');

// CSS class
expect(element).toHaveClass('active');
expect(element).toHaveClass('btn', 'btn-primary'); // Có cả hai class

// Style
expect(element).toHaveStyle({ display: 'none' });
expect(element).toHaveStyle('color: red');

// Focus
expect(input).toHaveFocus();
```

---

## 🧩 Ví Dụ Thực Tế — Login Form

```typescript
// LoginForm.tsx
function LoginForm({ onSuccess }: { onSuccess: (user: User) => void }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');
    
    try {
      const user = await authService.login(email, password);
      onSuccess(user);
    } catch {
      setError('Email hoặc mật khẩu không đúng');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={email} onChange={e => setEmail(e.target.value)} />
      
      <label htmlFor="password">Mật khẩu</label>
      <input id="password" type="password" value={password} onChange={e => setPassword(e.target.value)} />
      
      {error && <p role="alert">{error}</p>}
      
      <button type="submit" disabled={loading}>
        {loading ? 'Đang đăng nhập...' : 'Đăng nhập'}
      </button>
    </form>
  );
}
```

```typescript
// LoginForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { authService } from '../services/authService';
import { LoginForm } from './LoginForm';

jest.mock('../services/authService');
const mockAuthService = jest.mocked(authService);

describe('LoginForm', () => {
  const user = userEvent.setup();
  const onSuccess = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should render form fields correctly', () => {
    render(<LoginForm onSuccess={onSuccess} />);
    
    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/mật khẩu/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /đăng nhập/i })).toBeInTheDocument();
  });

  test('should call onSuccess with user data on successful login', async () => {
    const mockUser = { id: 1, name: 'An', email: 'an@example.com' };
    mockAuthService.login.mockResolvedValue(mockUser);
    
    render(<LoginForm onSuccess={onSuccess} />);
    
    await user.type(screen.getByLabelText(/email/i), 'an@example.com');
    await user.type(screen.getByLabelText(/mật khẩu/i), 'password123');
    await user.click(screen.getByRole('button', { name: /đăng nhập/i }));
    
    await waitFor(() => {
      expect(onSuccess).toHaveBeenCalledWith(mockUser);
    });
  });

  test('should show error message on failed login', async () => {
    mockAuthService.login.mockRejectedValue(new Error('Invalid credentials'));
    
    render(<LoginForm onSuccess={onSuccess} />);
    
    await user.type(screen.getByLabelText(/email/i), 'wrong@example.com');
    await user.type(screen.getByLabelText(/mật khẩu/i), 'wrongpassword');
    await user.click(screen.getByRole('button', { name: /đăng nhập/i }));
    
    expect(await screen.findByRole('alert')).toHaveTextContent(
      'Email hoặc mật khẩu không đúng'
    );
    expect(onSuccess).not.toHaveBeenCalled();
  });

  test('should disable button while loading', async () => {
    // Never resolve để giữ loading state
    mockAuthService.login.mockImplementation(() => new Promise(() => {}));
    
    render(<LoginForm onSuccess={onSuccess} />);
    
    await user.type(screen.getByLabelText(/email/i), 'an@example.com');
    await user.type(screen.getByLabelText(/mật khẩu/i), 'password123');
    await user.click(screen.getByRole('button', { name: /đăng nhập/i }));
    
    expect(screen.getByRole('button', { name: /đang đăng nhập/i })).toBeDisabled();
  });
});
```

---

## 🔗 Điều Hướng

- **Trước đó:** [1-jest-basics.md](./1-jest-basics.md) — Nền tảng Jest
- **Tiếp theo:** [3-mocking.md](./3-mocking.md) — Mock API, modules, hooks
- **Trở lại:** [README.md](./README.md) — Tổng quan testing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
