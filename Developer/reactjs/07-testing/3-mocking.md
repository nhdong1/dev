# 3 — Mocking (Giả Lập Phụ Thuộc)

> **Mocking** (Giả Lập) là kỹ thuật thay thế các phụ thuộc thực (API calls, modules, timers, ...) bằng phiên bản giả lập có thể kiểm soát được. Mocking giúp test chạy nhanh hơn, độc lập hơn, và dự đoán được kết quả mà không cần kết nối thực đến server hay database.

---

## 🎯 Mục Tiêu

- [ ] Mock **functions** (hàm) với `jest.fn()` và `jest.spyOn()`
- [ ] Mock **modules** (mô-đun) với `jest.mock()`
- [ ] Mock **HTTP API calls** với `global.fetch` và axios
- [ ] Mock **custom hooks** — giả lập hook phức tạp
- [ ] Kiểm soát **timers** (bộ đếm thời gian) với `jest.useFakeTimers()`
- [ ] Mock **environment variables** (biến môi trường)
- [ ] Hiểu khi nào nên và không nên mock

---

## 🧩 jest.fn() — Hàm Giả Lập

### Tạo Mock Function

```typescript
// Tạo mock function rỗng
const mockFn = jest.fn();

// Mock function với giá trị trả về
const mockGetUser = jest.fn().mockReturnValue({ id: 1, name: 'An' });

// Mock function bất đồng bộ
const mockFetchUser = jest.fn().mockResolvedValue({ id: 1, name: 'An' });

// Mock function trả về lỗi
const mockFailFetch = jest.fn().mockRejectedValue(new Error('Network error'));

// Mock function với implementation
const mockAdd = jest.fn().mockImplementation((a, b) => a + b);
```

### Kiểm Tra Mock Function Đã Được Gọi

```typescript
test('should call onSubmit when form is submitted', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = jest.fn();
  
  render(<ContactForm onSubmit={mockOnSubmit} />);
  
  await user.type(screen.getByLabelText(/email/i), 'an@example.com');
  await user.click(screen.getByRole('button', { name: /gửi/i }));
  
  // Kiểm tra đã được gọi
  expect(mockOnSubmit).toHaveBeenCalled();
  expect(mockOnSubmit).toHaveBeenCalledTimes(1);
  
  // Kiểm tra được gọi với arguments đúng
  expect(mockOnSubmit).toHaveBeenCalledWith({
    email: 'an@example.com',
  });
  
  // Kiểm tra arguments của lần gọi cụ thể
  expect(mockOnSubmit.mock.calls[0][0]).toEqual({ email: 'an@example.com' });
  expect(mockOnSubmit.mock.results[0].value).toBe(undefined);
});
```

### Return Values Theo Lần Gọi

```typescript
const mockFetch = jest.fn()
  .mockResolvedValueOnce({ data: 'first call' })   // Lần 1
  .mockResolvedValueOnce({ data: 'second call' })  // Lần 2
  .mockRejectedValue(new Error('Error'));           // Lần 3 trở đi

await mockFetch(); // { data: 'first call' }
await mockFetch(); // { data: 'second call' }
await mockFetch(); // throw Error
```

### Reset và Clear Mocks

```typescript
beforeEach(() => {
  // Xóa mock calls và instances, giữ nguyên implementation
  jest.clearAllMocks();
  
  // Reset về trạng thái ban đầu (xóa cả implementation)
  // jest.resetAllMocks();
  
  // Restore lại implementation gốc (dùng với spyOn)
  // jest.restoreAllMocks();
});
```

---

## 🕵️ jest.spyOn() — Theo Dõi Hàm Thật

```typescript
// spyOn theo dõi function thật, vẫn gọi implementation gốc
const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

// Kiểm tra console.error được gọi
expect(consoleSpy).toHaveBeenCalledWith('Error message');

// Restore sau test
consoleSpy.mockRestore();
```

```typescript
// Spy trên methods của object
import { authService } from '../services/authService';

test('should call authService.login', async () => {
  const loginSpy = jest.spyOn(authService, 'login')
    .mockResolvedValue({ id: 1, name: 'An' });
  
  render(<LoginForm />);
  // ... interact with form
  
  expect(loginSpy).toHaveBeenCalledWith('an@example.com', 'password123');
  loginSpy.mockRestore();
});
```

---

## 📦 jest.mock() — Mock Toàn Bộ Module

### Mock Module Hoàn Toàn

```typescript
// Mock module — đặt TRƯỚC imports (Jest hoist tự động)
jest.mock('../services/userService');

import { userService } from '../services/userService';
// userService bây giờ là mock — tất cả methods là jest.fn()

// TypeScript: dùng jest.mocked() để có đúng types
const mockedUserService = jest.mocked(userService);

test('should fetch user', async () => {
  mockedUserService.getUser.mockResolvedValue({ id: 1, name: 'An' });
  
  render(<UserProfile userId="1" />);
  
  await screen.findByText('An');
  expect(mockedUserService.getUser).toHaveBeenCalledWith('1');
});
```

### Mock Một Phần Module (Partial Mock)

```typescript
// Mock chỉ một số exports, giữ nguyên phần còn lại
jest.mock('../utils/formatters', () => ({
  ...jest.requireActual('../utils/formatters'), // Giữ nguyên thật
  formatDate: jest.fn().mockReturnValue('01/01/2026'), // Chỉ mock cái này
}));

import { formatDate, formatCurrency } from '../utils/formatters';
// formatDate → mock
// formatCurrency → thật
```

### Mock Default Export

```typescript
// Khi module export default
jest.mock('../components/Chart', () => ({
  __esModule: true,
  default: jest.fn(() => <div data-testid="mock-chart" />),
}));

// Dùng trong test
import Chart from '../components/Chart';
// Chart bây giờ là mock component
```

### Factory Function Pattern

```typescript
// Khi cần mock phức tạp hơn
jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: () => jest.fn(),
  useParams: () => ({ id: '1' }),
  useLocation: () => ({ pathname: '/users/1', search: '' }),
}));
```

---

## 🌐 Mock HTTP — Giả Lập Gọi HTTP

### Mock global.fetch

```typescript
// Trong test file
beforeEach(() => {
  global.fetch = jest.fn();
});

afterEach(() => {
  jest.restoreAllMocks();
});

test('should display users from API', async () => {
  const mockUsers = [
    { id: 1, name: 'Nguyen Van A' },
    { id: 2, name: 'Tran Thi B' },
  ];

  (global.fetch as jest.Mock).mockResolvedValue({
    ok: true,
    json: () => Promise.resolve(mockUsers),
  });

  render(<UserList />);
  
  // Đợi danh sách load
  expect(await screen.findByText('Nguyen Van A')).toBeInTheDocument();
  expect(screen.getByText('Tran Thi B')).toBeInTheDocument();
});

test('should show error message when API fails', async () => {
  (global.fetch as jest.Mock).mockRejectedValue(new Error('Network error'));
  
  render(<UserList />);
  
  expect(await screen.findByRole('alert')).toHaveTextContent(/lỗi kết nối/i);
});
```

### Mock Axios

```typescript
// __mocks__/axios.ts (auto-mock)
import axios from 'axios';

jest.mock('axios');
const mockedAxios = jest.mocked(axios);

test('should fetch products', async () => {
  mockedAxios.get.mockResolvedValue({
    data: [{ id: 1, name: 'Sản phẩm A', price: 100000 }],
  });
  
  render(<ProductList />);
  
  expect(await screen.findByText('Sản phẩm A')).toBeInTheDocument();
  expect(mockedAxios.get).toHaveBeenCalledWith('/api/products');
});
```

---

## 🪝 Mock Custom Hooks

### Cách 1: Mock Module Chứa Hook

```typescript
// hooks/useAuth.ts
export function useAuth() {
  // ... logic thật với context, API calls
  return { user, login, logout, isLoading };
}

// Trong test
jest.mock('../hooks/useAuth');
import { useAuth } from '../hooks/useAuth';

const mockUseAuth = jest.mocked(useAuth);

test('should show user name when authenticated', () => {
  mockUseAuth.mockReturnValue({
    user: { id: 1, name: 'An', email: 'an@example.com' },
    login: jest.fn(),
    logout: jest.fn(),
    isLoading: false,
  });
  
  render(<Header />);
  
  expect(screen.getByText('An')).toBeInTheDocument();
  expect(screen.queryByText(/đăng nhập/i)).not.toBeInTheDocument();
});

test('should show login button when not authenticated', () => {
  mockUseAuth.mockReturnValue({
    user: null,
    login: jest.fn(),
    logout: jest.fn(),
    isLoading: false,
  });
  
  render(<Header />);
  
  expect(screen.getByRole('link', { name: /đăng nhập/i })).toBeInTheDocument();
});
```

### Cách 2: Dependency Injection — Truyền Hook Qua Props

```typescript
// Thiết kế component linh hoạt hơn cho testing
interface UserProfileProps {
  userId: string;
  // Cho phép inject hook — dễ test hơn
  useUserHook?: (id: string) => { user: User | null; isLoading: boolean };
}

function UserProfile({ userId, useUserHook = useUser }: UserProfileProps) {
  const { user, isLoading } = useUserHook(userId);
  // ...
}

// Trong test
test('should display user', () => {
  const mockUseUser = jest.fn().mockReturnValue({
    user: { id: '1', name: 'An' },
    isLoading: false,
  });
  
  render(<UserProfile userId="1" useUserHook={mockUseUser} />);
  
  expect(screen.getByText('An')).toBeInTheDocument();
});
```

---

## ⏰ Fake Timers — Kiểm Soát Thời Gian

### jest.useFakeTimers()

```typescript
// Debounce search — chỉ gọi API sau 300ms không gõ
function SearchInput({ onSearch }: { onSearch: (query: string) => void }) {
  const [value, setValue] = useState('');

  useEffect(() => {
    const timer = setTimeout(() => {
      if (value) onSearch(value);
    }, 300);
    
    return () => clearTimeout(timer);
  }, [value, onSearch]);

  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

test('should debounce search call', async () => {
  jest.useFakeTimers(); // Kiểm soát timers
  
  const onSearch = jest.fn();
  const user = userEvent.setup({ delay: null }); // Tắt delay trong userEvent
  
  render(<SearchInput onSearch={onSearch} />);
  
  await user.type(screen.getByRole('textbox'), 'react');
  
  // Chưa đủ 300ms → chưa gọi
  expect(onSearch).not.toHaveBeenCalled();
  
  // Tua thời gian 300ms
  jest.advanceTimersByTime(300);
  
  expect(onSearch).toHaveBeenCalledWith('react');
  expect(onSearch).toHaveBeenCalledTimes(1); // Chỉ gọi 1 lần (debounced)
  
  jest.useRealTimers(); // Restore timers thật
});
```

### Fake Timers Với Date

```typescript
test('should show correct date', () => {
  // Mock ngày hiện tại
  jest.setSystemTime(new Date('2026-06-04'));
  
  render(<DateDisplay />);
  
  expect(screen.getByText('04/06/2026')).toBeInTheDocument();
  
  jest.useRealTimers();
});
```

---

## 🌍 Mock Environment Variables — Biến Môi Trường

```typescript
// Trong test
const originalEnv = process.env;

beforeEach(() => {
  jest.resetModules();
  process.env = { ...originalEnv }; // Copy env gốc
});

afterAll(() => {
  process.env = originalEnv; // Restore
});

test('should use production API URL', () => {
  process.env.REACT_APP_API_URL = 'https://api.production.com';
  process.env.NODE_ENV = 'production';
  
  render(<App />);
  
  // Kiểm tra app sử dụng đúng URL
});
```

---

## 📁 __mocks__ Directory — Thư Mục Giả Lập Tự Động

```
src/
├── __mocks__/
│   ├── axios.ts              ← Auto-mock axios khi gọi jest.mock('axios')
│   ├── react-router-dom.ts   ← Auto-mock react-router-dom
│   └── fileMock.js           ← Mock file imports (CSS, images)
└── services/
    ├── __mocks__/
    │   └── userService.ts    ← Auto-mock cho ../services/userService
    └── userService.ts
```

```typescript
// src/__mocks__/fileMock.js — Dùng cho CSS/image imports
module.exports = 'test-file-stub';

// jest.config.ts
moduleNameMapper: {
  '\\.(css|less|scss|sass)$': '<rootDir>/src/__mocks__/fileMock.js',
  '\\.(jpg|jpeg|png|gif|svg)$': '<rootDir>/src/__mocks__/fileMock.js',
}
```

---

## ⚠️ Khi Nào Nên Và Không Nên Mock

### ✅ Nên Mock

```
✅ HTTP API calls — Không gọi server thật trong unit tests
✅ External services (email, payment, analytics) — Tránh side effects thật
✅ Browser APIs (localStorage, window.location) — JSDOM không hỗ trợ đầy đủ
✅ Timers (setTimeout, setInterval) — Tránh test chậm
✅ Modules nặng và chậm — giúp tests nhanh hơn
✅ Non-deterministic values — Date.now(), Math.random()
```

### ❌ Không Nên Mock

```
❌ Code của chính bạn — Mất đi giá trị kiểm thử thực sự
❌ React internals — Đừng mock useState, useEffect
❌ Quá nhiều thứ — Test mà mock mọi thứ = test không có ý nghĩa
❌ Database trong integration tests — Dùng test database thật hoặc MSW
```

---

## 🔗 Điều Hướng

- **Trước đó:** [2-react-testing-library.md](./2-react-testing-library.md) — RTL
- **Tiếp theo:** [4-integration-testing.md](./4-integration-testing.md) — Integration tests với MSW
- **Trở lại:** [README.md](./README.md) — Tổng quan testing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
