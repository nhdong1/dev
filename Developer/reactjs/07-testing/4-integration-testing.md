# 4 — Integration Testing với MSW (Kiểm Thử Tích Hợp)

> **Integration Testing** (Kiểm Thử Tích Hợp) kiểm tra nhiều units hoạt động cùng nhau — component + hooks + API calls + state management. **MSW** — Mock Service Worker (Trình Giả Lập Dịch Vụ Web) — là công cụ tốt nhất để giả lập API ở cấp độ mạng (network level), không phải ở cấp độ module, tạo ra tests gần với thực tế nhất.

---

## 🎯 Mục Tiêu

- [ ] Phân biệt **integration tests** (kiểm thử tích hợp) với unit tests
- [ ] Cài đặt và cấu hình **MSW** — Mock Service Worker
- [ ] Tạo **request handlers** (bộ xử lý yêu cầu) để giả lập REST API
- [ ] Test đầy đủ luồng: render → user interaction → API call → UI update
- [ ] Xử lý **error scenarios** (kịch bản lỗi): network error, 404, 500
- [ ] Dùng MSW trong cả browser (development) và Node.js (test)

---

## 🆚 Unit Test vs Integration Test

```
Unit Test:
  ┌─────────────────────────────────────┐
  │  Component A                        │
  │  (mọi dependency đều bị mock)       │
  └─────────────────────────────────────┘
  → Nhanh, cô lập, test logic nhỏ

Integration Test:
  ┌──────────┐  ┌──────────┐  ┌─────────────┐
  │Component │→ │  Hook    │→ │  MSW (fake  │
  │          │  │  (thật)  │  │   API)      │
  └──────────┘  └──────────┘  └─────────────┘
  → Chậm hơn, tự tin hơn, test luồng thực tế
```

---

## 📦 Cài Đặt MSW

```bash
npm install --save-dev msw

# Tạo service worker cho browser (development)
npx msw init public/ --save
```

### Cấu Trúc Thư Mục

```
src/
├── mocks/
│   ├── handlers.ts     ← Định nghĩa request handlers
│   ├── browser.ts      ← Setup MSW cho browser
│   └── server.ts       ← Setup MSW cho Node.js (tests)
└── ...
```

---

## 🛠️ Cấu Hình MSW

### handlers.ts — Định Nghĩa API Giả Lập

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

// Dữ liệu giả lập (fixture data)
const mockUsers = [
  { id: 1, name: 'Nguyen Van A', email: 'a@example.com', role: 'admin' },
  { id: 2, name: 'Tran Thi B',  email: 'b@example.com', role: 'user' },
];

let mockTodos = [
  { id: 1, title: 'Học MSW', completed: false, userId: 1 },
  { id: 2, title: 'Viết tests', completed: true, userId: 1 },
];

export const handlers = [
  // GET /api/users — Lấy danh sách users
  http.get('/api/users', () => {
    return HttpResponse.json(mockUsers);
  }),

  // GET /api/users/:id — Lấy user theo ID
  http.get('/api/users/:id', ({ params }) => {
    const { id } = params;
    const user = mockUsers.find(u => u.id === Number(id));
    
    if (!user) {
      return HttpResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    }
    
    return HttpResponse.json(user);
  }),

  // POST /api/todos — Tạo todo mới
  http.post('/api/todos', async ({ request }) => {
    const body = await request.json() as { title: string; userId: number };
    
    const newTodo = {
      id: mockTodos.length + 1,
      title: body.title,
      completed: false,
      userId: body.userId,
    };
    
    mockTodos.push(newTodo);
    
    return HttpResponse.json(newTodo, { status: 201 });
  }),

  // PATCH /api/todos/:id — Cập nhật todo
  http.patch('/api/todos/:id', async ({ params, request }) => {
    const { id } = params;
    const body = await request.json() as Partial<typeof mockTodos[0]>;
    
    const index = mockTodos.findIndex(t => t.id === Number(id));
    if (index === -1) {
      return HttpResponse.json({ error: 'Todo not found' }, { status: 404 });
    }
    
    mockTodos[index] = { ...mockTodos[index], ...body };
    return HttpResponse.json(mockTodos[index]);
  }),

  // DELETE /api/todos/:id — Xóa todo
  http.delete('/api/todos/:id', ({ params }) => {
    const { id } = params;
    mockTodos = mockTodos.filter(t => t.id !== Number(id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

### server.ts — Cấu Hình Cho Node.js (Tests)

```typescript
// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

// Tạo server với handlers mặc định
export const server = setupServer(...handlers);
```

### browser.ts — Cấu Hình Cho Browser (Development)

```typescript
// src/mocks/browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);
```

### Kích Hoạt MSW Trong Development

```typescript
// src/main.tsx
async function enableMocking() {
  if (process.env.NODE_ENV !== 'development') return;
  
  const { worker } = await import('./mocks/browser');
  return worker.start({
    onUnhandledRequest: 'warn', // Cảnh báo khi có request không được handle
  });
}

enableMocking().then(() => {
  ReactDOM.createRoot(document.getElementById('root')!).render(<App />);
});
```

### Setup Trong Jest

```typescript
// jest.setup.ts
import '@testing-library/jest-dom';
import { server } from './src/mocks/server';

// Khởi động server trước tất cả tests
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));

// Reset handlers sau mỗi test (tránh ảnh hưởng giữa các tests)
afterEach(() => server.resetHandlers());

// Tắt server sau tất cả tests
afterAll(() => server.close());
```

---

## 🧪 Viết Integration Tests

### Test Danh Sách Users

```typescript
// UserList.test.tsx
import { render, screen, within } from '@testing-library/react';
import { UserList } from './UserList';
import { renderWithProviders } from '../test-utils';

describe('UserList — Integration', () => {
  test('should fetch and display users from API', async () => {
    renderWithProviders(<UserList />);
    
    // Loading state
    expect(screen.getByRole('status')).toBeInTheDocument(); // Spinner
    
    // Đợi data load xong
    const userItems = await screen.findAllByRole('listitem');
    expect(userItems).toHaveLength(2);
    
    // Kiểm tra nội dung
    expect(screen.getByText('Nguyen Van A')).toBeInTheDocument();
    expect(screen.getByText('Tran Thi B')).toBeInTheDocument();
  });

  test('should show error state when API fails', async () => {
    // Override handler cho test này
    server.use(
      http.get('/api/users', () => {
        return HttpResponse.json(
          { error: 'Internal Server Error' },
          { status: 500 }
        );
      })
    );
    
    renderWithProviders(<UserList />);
    
    const errorMessage = await screen.findByRole('alert');
    expect(errorMessage).toHaveTextContent(/lỗi khi tải danh sách/i);
    
    // Nút thử lại
    expect(screen.getByRole('button', { name: /thử lại/i })).toBeInTheDocument();
  });

  test('should show empty state when no users', async () => {
    server.use(
      http.get('/api/users', () => {
        return HttpResponse.json([]);
      })
    );
    
    renderWithProviders(<UserList />);
    
    expect(await screen.findByText(/chưa có người dùng nào/i)).toBeInTheDocument();
  });
});
```

### Test CRUD — Todo App Đầy Đủ

```typescript
// TodoApp.test.tsx
import { render, screen, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { server } from '../mocks/server';
import { http, HttpResponse } from 'msw';
import { TodoApp } from './TodoApp';

describe('TodoApp — Integration', () => {
  const user = userEvent.setup();

  test('should load and display existing todos', async () => {
    render(<TodoApp userId={1} />);
    
    // Đợi todos load
    expect(await screen.findByText('Học MSW')).toBeInTheDocument();
    expect(screen.getByText('Viết tests')).toBeInTheDocument();
    
    // "Viết tests" đã completed → checkbox checked
    const writingTestsCheckbox = screen.getByRole('checkbox', {
      name: /viết tests/i,
    });
    expect(writingTestsCheckbox).toBeChecked();
  });

  test('should add new todo', async () => {
    render(<TodoApp userId={1} />);
    
    // Đợi load xong
    await screen.findByText('Học MSW');
    
    // Thêm todo mới
    const input = screen.getByRole('textbox', { name: /todo mới/i });
    await user.type(input, 'Đọc tài liệu MSW');
    await user.keyboard('{Enter}');
    
    // Kiểm tra todo mới xuất hiện
    expect(await screen.findByText('Đọc tài liệu MSW')).toBeInTheDocument();
    
    // Input đã được clear
    expect(input).toHaveValue('');
  });

  test('should toggle todo completion', async () => {
    render(<TodoApp userId={1} />);
    
    await screen.findByText('Học MSW');
    
    const learnMswCheckbox = screen.getByRole('checkbox', { name: /học msw/i });
    expect(learnMswCheckbox).not.toBeChecked();
    
    await user.click(learnMswCheckbox);
    
    // Optimistic update — UI cập nhật ngay
    expect(learnMswCheckbox).toBeChecked();
    
    // Sau khi API confirm
    await waitFor(() => {
      expect(learnMswCheckbox).toBeChecked();
    });
  });

  test('should delete todo', async () => {
    render(<TodoApp userId={1} />);
    
    await screen.findByText('Học MSW');
    
    // Tìm nút xóa trong row của todo
    const todoItem = screen.getByText('Học MSW').closest('li')!;
    const deleteButton = within(todoItem).getByRole('button', { name: /xóa/i });
    
    await user.click(deleteButton);
    
    // Xác nhận xóa (nếu có dialog)
    const confirmButton = screen.getByRole('button', { name: /xác nhận/i });
    await user.click(confirmButton);
    
    // Todo đã biến mất
    await waitFor(() => {
      expect(screen.queryByText('Học MSW')).not.toBeInTheDocument();
    });
  });

  test('should handle network error gracefully', async () => {
    server.use(
      http.post('/api/todos', () => {
        return HttpResponse.error(); // Network error (không phải HTTP error)
      })
    );
    
    render(<TodoApp userId={1} />);
    await screen.findByText('Học MSW');
    
    await user.type(screen.getByRole('textbox', { name: /todo mới/i }), 'Test todo');
    await user.keyboard('{Enter}');
    
    // Hiển thị thông báo lỗi
    expect(await screen.findByRole('alert')).toHaveTextContent(/lỗi kết nối/i);
  });
});
```

---

## 🔀 Handler Overrides — Ghi Đè Handler Cho Test Cụ Thể

```typescript
import { server } from '../mocks/server';
import { http, HttpResponse } from 'msw';

test('should handle 401 unauthorized', async () => {
  // Ghi đè handler chỉ cho test này
  // server.resetHandlers() trong afterEach sẽ reset về handlers gốc
  server.use(
    http.get('/api/profile', () => {
      return HttpResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    })
  );
  
  render(<ProfilePage />);
  
  // Nên redirect đến login
  await waitFor(() => {
    expect(window.location.pathname).toBe('/login');
  });
});

test('should handle slow network', async () => {
  server.use(
    http.get('/api/users', async () => {
      // Mô phỏng network chậm
      await new Promise(resolve => setTimeout(resolve, 2000));
      return HttpResponse.json([]);
    })
  );
  
  // Test với timeout cao hơn
  render(<UserList />);
  
  // Loading skeleton phải hiển thị trong thời gian chờ
  expect(screen.getAllByRole('status')).toHaveLength(3); // 3 skeleton loaders
});
```

---

## 🎯 REST vs GraphQL

### MSW Với GraphQL

```typescript
import { graphql, HttpResponse } from 'msw';

export const handlers = [
  // GraphQL query handler
  graphql.query('GetUser', ({ variables }) => {
    const { id } = variables;
    
    return HttpResponse.json({
      data: {
        user: {
          id,
          name: 'Nguyen Van A',
          email: 'a@example.com',
        },
      },
    });
  }),

  // GraphQL mutation handler
  graphql.mutation('CreateTodo', ({ variables }) => {
    const { title, userId } = variables;
    
    return HttpResponse.json({
      data: {
        createTodo: {
          id: Date.now(),
          title,
          userId,
          completed: false,
        },
      },
    });
  }),
];
```

---

## 🛠️ Test Utilities — Tiện Ích Test Chung

```typescript
// src/test-utils.tsx — Custom render với providers
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from '../contexts/AuthContext';

interface RenderWithProvidersOptions extends RenderOptions {
  initialUser?: User | null;
  route?: string;
}

export function renderWithProviders(
  ui: React.ReactElement,
  { initialUser = null, route = '/', ...options }: RenderWithProvidersOptions = {}
) {
  // Tạo QueryClient mới cho mỗi test (tránh cache chia sẻ)
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,          // Tắt retry để test fail nhanh
        gcTime: Infinity,      // Không xóa cache trong khi test
      },
    },
  });

  window.history.pushState({}, '', route);

  function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        <AuthProvider initialUser={initialUser}>
          <BrowserRouter>
            {children}
          </BrowserRouter>
        </AuthProvider>
      </QueryClientProvider>
    );
  }

  return render(ui, { wrapper: Wrapper, ...options });
}

// Re-export tất cả từ RTL để không cần import ở 2 nơi
export * from '@testing-library/react';
export { renderWithProviders as render };
```

---

## 📊 MSW vs jest.mock() — Khi Nào Dùng Cái Gì?

| Tiêu Chí | MSW | jest.mock() |
| -------- | --- | ----------- |
| **Độ thực tế** (realism) | Cao — intercept ở network level | Thấp — mock ở module level |
| **Dùng lại** (reusability) | Dùng được cả tests lẫn development | Chỉ trong tests |
| **Cấu hình** | Một lần, dùng cho toàn dự án | Cần setup mỗi file test |
| **Linh hoạt** | Override per-test | Override per-test |
| **HTTP library** | Độc lập (fetch, axios, ...) | Phụ thuộc vào library |
| **Khi dùng** | Integration tests, API-heavy tests | Unit tests, isolated module tests |

---

## 🔗 Điều Hướng

- **Trước đó:** [3-mocking.md](./3-mocking.md) — Mock API, modules, hooks
- **Tiếp theo:** [5-vitest.md](./5-vitest.md) — Vitest cho Vite projects
- **Trở lại:** [README.md](./README.md) — Tổng quan testing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
