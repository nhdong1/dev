# 🏗️ System Design Frontend — Thiết Kế Hệ Thống Frontend

> Phỏng vấn Senior thường có 1-2 câu system design. Mục tiêu không phải tìm "đáp án đúng" mà là đánh giá tư duy kiến trúc, khả năng phân tích trade-offs, và kinh nghiệm thực tế.

---

## 🗂️ Mục Lục

1. [Framework Trả Lời System Design](#framework)
2. [State Architecture — Kiến Trúc State](#state-architecture)
3. [Component Architecture — Kiến Trúc Component](#component-architecture)
4. [Data Fetching Strategy — Chiến Lược Lấy Dữ Liệu](#data-fetching)
5. [Authentication & Authorization — Xác Thực & Phân Quyền](#auth)
6. [Các Bài Toán System Design Thực Tế](#bài-toán-thực-tế)

---

## Framework Trả Lời System Design {#framework}

### Khung 5 bước cho mọi câu hỏi system design

```
1. CLARIFY — LÀM RÕ YÊU CẦU (2-3 phút)
   Đặt câu hỏi trước khi thiết kế:
   - Quy mô: bao nhiêu user? DAU (Daily Active Users — Người Dùng Hoạt Động Hàng Ngày)?
   - Target devices: mobile-first, desktop, cross-platform?
   - Performance requirements: offline support không? Real-time updates không?
   - Team size: bao nhiêu developer? Timeline?
   - Existing infrastructure: REST API hay GraphQL? Auth method?

2. HIGH-LEVEL DESIGN — THIẾT KẾ TỔNG QUAN (5 phút)
   Vẽ kiến trúc tổng thể:
   - Page/route structure
   - Component hierarchy sơ bộ
   - State management approach
   - Data flow

3. DEEP DIVE — ĐÀO SÂU CHI TIẾT (10-15 phút)
   Đào sâu từng phần quan trọng:
   - Component architecture
   - State management
   - API design / data fetching
   - Performance strategy
   - Authentication flow

4. TRADE-OFFS — ĐÁNH ĐỔI (3-5 phút)
   Nêu pros/cons của quyết định đưa ra:
   - Tại sao chọn Redux thay vì Zustand?
   - REST vs GraphQL cho use case này?
   - CSR vs SSR vs SSG?

5. FOLLOW-UP — CÂU HỎI TIẾP THEO (nếu có thời gian)
   Cải thiện:
   - Scalability: nếu user tăng 10x?
   - Testing strategy
   - Monitoring & error tracking
```

---

## State Architecture — Kiến Trúc State {#state-architecture}

### Câu hỏi: Thiết kế state management cho e-commerce app lớn

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

**Bước 1: Phân loại state:**

```
State Categories — Phân Loại State:

SERVER STATE (Dùng TanStack Query):
├── Products catalog — Danh Mục Sản Phẩm
├── User profile
├── Orders history
├── Reviews
└── Search results

CLIENT STATE (Dùng Zustand hoặc Redux Toolkit):
├── Shopping cart (có thể offline, cần persist)
├── UI state: modals, drawers, tabs
├── Filter/sort preferences
└── Checkout wizard state

URL STATE (Dùng React Router searchParams):
├── Search query
├── Category filter
├── Sort order
├── Pagination
└── Product detail (slug)

FORM STATE (Dùng React Hook Form):
├── Checkout form
├── Address form
└── Payment form
```

**Bước 2: Kiến trúc cụ thể:**

```jsx
// Zustand store cho cart
const useCartStore = create(
  persist( // Persist sang localStorage
    (set, get) => ({
      items: [],
      addItem: (product, quantity = 1) => set((state) => {
        const existing = state.items.find(item => item.id === product.id);
        if (existing) {
          return {
            items: state.items.map(item =>
              item.id === product.id
                ? { ...item, quantity: item.quantity + quantity }
                : item
            )
          };
        }
        return { items: [...state.items, { ...product, quantity }] };
      }),
      removeItem: (productId) => set((state) => ({
        items: state.items.filter(item => item.id !== productId)
      })),
      updateQuantity: (productId, quantity) => set((state) => ({
        items: state.items.map(item =>
          item.id === productId ? { ...item, quantity } : item
        )
      })),
      clearCart: () => set({ items: [] }),
      getTotalPrice: () => get().items.reduce(
        (total, item) => total + item.price * item.quantity, 0
      ),
    }),
    {
      name: 'cart-storage', // localStorage key
      partialize: (state) => ({ items: state.items }), // Chỉ persist items
    }
  )
);

// TanStack Query cho products
const useProducts = (filters) => useQuery({
  queryKey: ['products', filters],
  queryFn: () => productApi.getProducts(filters),
  staleTime: 5 * 60 * 1000, // 5 phút — product catalog ít thay đổi
});

// URL state cho filters
function useProductFilters() {
  const [searchParams, setSearchParams] = useSearchParams();

  const filters = {
    category: searchParams.get('category') ?? 'all',
    sort: searchParams.get('sort') ?? 'popular',
    page: Number(searchParams.get('page') ?? '1'),
    priceMin: Number(searchParams.get('priceMin') ?? '0'),
    priceMax: Number(searchParams.get('priceMax') ?? '999999'),
  };

  const setFilter = useCallback((key, value) => {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      if (value) { next.set(key, value); }
      else { next.delete(key); }
      if (key !== 'page') next.set('page', '1'); // Reset page khi filter thay đổi
      return next;
    });
  }, [setSearchParams]);

  return { filters, setFilter };
}
```

---

## Component Architecture — Kiến Trúc Component {#component-architecture}

### Câu hỏi: Thiết kế hệ thống component cho Design System — Hệ Thống Thiết Kế

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

**Nguyên tắc Atomic Design — Thiết Kế Nguyên Tử:**

```
Atoms — Nguyên Tử (primitive UI elements):
├── Button
├── Input
├── Icon
├── Badge
└── Avatar

Molecules — Phân Tử (kết hợp atoms):
├── SearchBar = Input + Icon + Button
├── UserCard = Avatar + Badge + Text
└── FormField = Label + Input + ErrorMessage

Organisms — Sinh Vật (complex UI sections):
├── Header = Logo + Navigation + UserMenu
├── ProductCard = Image + Title + Price + AddToCart
└── DataTable = SearchBar + Filters + Table + Pagination

Templates — Khuôn Mẫu (page layouts):
├── DashboardLayout = Sidebar + Header + Content
└── AuthLayout = Logo + Form + Footer

Pages — Trang:
├── ProductListPage
└── CheckoutPage
```

**Thiết kế API component linh hoạt:**

```tsx
// Button component với composition-friendly API
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  fullWidth?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  (
    { variant = 'primary', size = 'md', isLoading, leftIcon, rightIcon,
      fullWidth, className, children, disabled, ...props },
    ref
  ) => {
    return (
      <button
        ref={ref}
        className={cn(
          buttonVariants({ variant, size }),
          fullWidth && 'w-full',
          className
        )}
        disabled={disabled || isLoading}
        {...props}
      >
        {isLoading ? (
          <Spinner size={size} />
        ) : (
          <>
            {leftIcon && <span className="mr-2">{leftIcon}</span>}
            {children}
            {rightIcon && <span className="ml-2">{rightIcon}</span>}
          </>
        )}
      </button>
    );
  }
);

// Compound Components Pattern cho phức tạp hơn
// Select = Select.Root + Select.Trigger + Select.Options + Select.Option
const Select = {
  Root: SelectRoot,
  Trigger: SelectTrigger,
  Options: SelectOptions,
  Option: SelectOption,
};

// Dùng:
<Select.Root value={value} onChange={setValue}>
  <Select.Trigger placeholder="Chọn..." />
  <Select.Options>
    {options.map(opt => (
      <Select.Option key={opt.value} value={opt.value}>
        {opt.label}
      </Select.Option>
    ))}
  </Select.Options>
</Select.Root>
```

---

### Câu hỏi: Thiết kế Component tái sử dụng cho Data Table — Bảng Dữ Liệu

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```tsx
// Generic, flexible DataTable với TypeScript
interface Column<T> {
  key: keyof T | string;
  header: string;
  render?: (value: T[keyof T], row: T) => React.ReactNode;
  sortable?: boolean;
  width?: string;
}

interface DataTableProps<T> {
  data: T[];
  columns: Column<T>[];
  keyExtractor: (item: T) => string | number;
  isLoading?: boolean;
  error?: string | null;
  emptyMessage?: string;

  // Pagination — Phân Trang
  pagination?: {
    currentPage: number;
    totalPages: number;
    onPageChange: (page: number) => void;
  };

  // Sorting — Sắp Xếp
  sorting?: {
    sortBy: string;
    sortOrder: 'asc' | 'desc';
    onSort: (key: string) => void;
  };

  // Row actions
  onRowClick?: (item: T) => void;
  rowActions?: (item: T) => React.ReactNode;
}

function DataTable<T>({
  data, columns, keyExtractor,
  isLoading, error, emptyMessage = 'Không có dữ liệu',
  pagination, sorting, onRowClick, rowActions
}: DataTableProps<T>) {
  if (isLoading) return <TableSkeleton columns={columns.length} />;
  if (error) return <ErrorMessage message={error} />;
  if (data.length === 0) return <EmptyState message={emptyMessage} />;

  return (
    <div>
      <table className="data-table">
        <thead>
          <tr>
            {columns.map(col => (
              <th
                key={String(col.key)}
                style={{ width: col.width }}
                onClick={col.sortable ? () => sorting?.onSort(String(col.key)) : undefined}
                className={col.sortable ? 'sortable' : ''}
              >
                {col.header}
                {col.sortable && sorting?.sortBy === col.key && (
                  <SortIcon direction={sorting.sortOrder} />
                )}
              </th>
            ))}
            {rowActions && <th>Hành Động</th>}
          </tr>
        </thead>
        <tbody>
          {data.map(item => (
            <tr
              key={keyExtractor(item)}
              onClick={onRowClick ? () => onRowClick(item) : undefined}
              className={onRowClick ? 'clickable' : ''}
            >
              {columns.map(col => (
                <td key={String(col.key)}>
                  {col.render
                    ? col.render(item[col.key as keyof T], item)
                    : String(item[col.key as keyof T] ?? '')
                  }
                </td>
              ))}
              {rowActions && <td>{rowActions(item)}</td>}
            </tr>
          ))}
        </tbody>
      </table>

      {pagination && (
        <Pagination
          currentPage={pagination.currentPage}
          totalPages={pagination.totalPages}
          onPageChange={pagination.onPageChange}
        />
      )}
    </div>
  );
}

// Sử dụng — Type-safe, flexible
<DataTable
  data={users}
  keyExtractor={user => user.id}
  columns={[
    { key: 'name', header: 'Tên', sortable: true },
    { key: 'email', header: 'Email' },
    {
      key: 'role',
      header: 'Vai Trò',
      render: (role) => <RoleBadge role={role as string} />
    },
    {
      key: 'createdAt',
      header: 'Ngày Tạo',
      sortable: true,
      render: (date) => format(new Date(date as string), 'dd/MM/yyyy')
    },
  ]}
  sorting={{ sortBy, sortOrder, onSort: handleSort }}
  pagination={{ currentPage, totalPages, onPageChange: setPage }}
  rowActions={user => (
    <>
      <EditButton onClick={() => handleEdit(user)} />
      <DeleteButton onClick={() => handleDelete(user.id)} />
    </>
  )}
/>
```

---

## Data Fetching Strategy — Chiến Lược Lấy Dữ Liệu {#data-fetching}

### Câu hỏi: Khi nào dùng SSR, SSG, ISR, hay CSR?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐⭐

```
CSR (Client-Side Rendering — Kết Xuất Phía Client):
Render hoàn toàn trên browser sau khi tải JavaScript.

Dùng khi:
├── Dashboard sau khi login (không cần SEO)
├── Data cập nhật real-time
├── User-specific, không cache được
└── Internal tools

Nhược điểm: FCP chậm, SEO kém

---

SSR (Server-Side Rendering — Kết Xuất Phía Server):
Render HTML trên server mỗi request, trả về cho client.

Dùng khi:
├── Data thay đổi theo user (personalized)
├── SEO quan trọng + data fresh — tươi
├── Search results, product detail với inventory realtime
└── Data cần token/cookie của user

Nhược điểm: Server load cao, TTFB chậm hơn static

---

SSG (Static Site Generation — Tạo Trang Tĩnh):
Tạo HTML tại build time, serve từ CDN.

Dùng khi:
├── Blog posts, documentation
├── Marketing pages
├── Data ít thay đổi (thay đổi khi deploy)
└── Cần performance tối đa

Nhược điểm: Không có data real-time, cần rebuild khi data thay đổi

---

ISR (Incremental Static Regeneration — Tái Tạo Tĩnh Tăng Dần):
SSG + tự động rebuild theo interval hoặc on-demand.

Dùng khi:
├── E-commerce product pages (giá thay đổi thỉnh thoảng)
├── News articles (cập nhật theo giờ, không cần realtime)
├── Blog với nhiều posts (không muốn rebuild tất cả)
└── Data thay đổi định kỳ nhưng không realtime

Nhược điểm: Phức tạp hơn SSG, có thể serve stale data

---

RSC (React Server Components — Component Phía Server):
Components chạy trên server, zero JS client, có thể fetch data trực tiếp.

Dùng khi (Next.js App Router):
├── Static content + dynamic leaf nodes
├── Cần truy cập DB/file system trực tiếp
├── Giảm thiểu JS bundle size
└── Phức tạp data fetching

Kết hợp: RSC + Client Components cho interactive parts
```

---

### Câu hỏi: Thiết kế real-time features — Tính Năng Thời Gian Thực trong React

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// Chiến lược 1: WebSocket — Giao Thức Web Socket
// Dùng cho: Chat, live notifications, multiplayer, collaborative editing

function useChatWebSocket(chatId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [status, setStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectTimeoutRef = useRef<number>();

  const connect = useCallback(() => {
    const ws = new WebSocket(`wss://api.example.com/chat/${chatId}`);
    wsRef.current = ws;

    ws.onopen = () => setStatus('connected');
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setMessages(prev => [...prev, message]);
    };
    ws.onclose = () => {
      setStatus('disconnected');
      // Auto reconnect với exponential backoff — Tăng Dần Thời Gian Chờ
      reconnectTimeoutRef.current = window.setTimeout(connect, 3000);
    };
    ws.onerror = () => ws.close();
  }, [chatId]);

  useEffect(() => {
    connect();
    return () => {
      clearTimeout(reconnectTimeoutRef.current);
      wsRef.current?.close();
    };
  }, [connect]);

  const sendMessage = useCallback((content: string) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({ type: 'message', content }));
    }
  }, []);

  return { messages, status, sendMessage };
}

// Chiến lược 2: SSE — Server-Sent Events — Sự Kiện Do Server Gửi
// Dùng cho: Live feeds, notifications (one-way từ server → client)
// Đơn giản hơn WebSocket, tự động reconnect

function useLiveNotifications(userId: string) {
  const [notifications, setNotifications] = useState<Notification[]>([]);

  useEffect(() => {
    const eventSource = new EventSource(`/api/notifications/stream?userId=${userId}`);

    eventSource.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      setNotifications(prev => [notification, ...prev]);
    };

    eventSource.addEventListener('ping', () => {}); // Keep alive

    return () => eventSource.close();
  }, [userId]);

  return notifications;
}

// Chiến lược 3: Polling — Thăm Dò Định Kỳ (đơn giản nhất)
// Dùng khi: Không cần realtime thực sự, acceptable delay (30s-1min)

const { data } = useQuery({
  queryKey: ['dashboard-stats'],
  queryFn: fetchStats,
  refetchInterval: 30 * 1000, // Tự động refetch mỗi 30s
  refetchIntervalInBackground: false, // Dừng khi tab không active
});
```

---

## Authentication & Authorization — Xác Thực & Phân Quyền {#auth}

### Câu hỏi: Thiết kế authentication flow trong React SPA

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

```jsx
// Auth Context Architecture
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

interface AuthContextValue extends AuthState {
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
}

// Token storage strategy — Chiến Lược Lưu Token:
// localStorage: XSS vulnerable nhưng tồn tại sau page refresh
// sessionStorage: Mất khi đóng tab
// httpOnly cookie: XSS-safe, CSRF vulnerable (cần CSRF token)
// Memory only: Safest nhưng mất khi refresh

// Khuyến nghị: httpOnly cookie + CSRF token cho production

function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, initialState);

  // Silent token refresh — Làm Mới Token Ngầm
  useEffect(() => {
    const refreshInterval = setInterval(async () => {
      try {
        await refreshToken();
      } catch {
        dispatch({ type: 'LOGOUT' });
      }
    }, 14 * 60 * 1000); // Refresh mỗi 14 phút (token expire sau 15 phút)

    return () => clearInterval(refreshInterval);
  }, []);

  // Intercept 401 responses để refresh token
  useEffect(() => {
    const interceptor = api.interceptors.response.use(
      response => response,
      async error => {
        if (error.response?.status === 401 && !error.config._retry) {
          error.config._retry = true;
          await refreshToken();
          return api(error.config); // Retry request
        }
        return Promise.reject(error);
      }
    );
    return () => api.interceptors.response.eject(interceptor);
  }, []);
}

// Protected Route — Route Được Bảo Vệ
function PrivateRoute({ children, requiredRole }: { children: React.ReactNode, requiredRole?: string }) {
  const { isAuthenticated, isLoading, user } = useAuth();
  const location = useLocation();

  if (isLoading) return <FullPageSpinner />;

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/forbidden" replace />;
  }

  return <>{children}</>;
}

// Routing structure
<Routes>
  <Route path="/login" element={<LoginPage />} />
  <Route path="/register" element={<RegisterPage />} />

  <Route path="/dashboard" element={
    <PrivateRoute>
      <DashboardLayout />
    </PrivateRoute>
  }>
    <Route index element={<DashboardHome />} />
    <Route path="profile" element={<ProfilePage />} />
    <Route path="admin" element={
      <PrivateRoute requiredRole="admin">
        <AdminPage />
      </PrivateRoute>
    } />
  </Route>
</Routes>
```

---

## Các Bài Toán System Design Thực Tế {#bài-toán-thực-tế}

### Bài Toán 1: Thiết kế Social Media Feed — Newsfeed Mạng Xã Hội

**Yêu cầu:** Infinite scroll — Cuộn Vô Hạn, real-time updates, like/comment, media

```
HIGH-LEVEL DESIGN:

1. ROUTING:
   /feed → FeedPage (protected)
   /post/:id → PostDetailPage
   /profile/:username → ProfilePage

2. STATE MANAGEMENT:
   Server State (TanStack Query):
   ├── Feed posts (cursor-based pagination)
   ├── Post details + comments
   ├── User profiles
   └── Notifications

   Client State (Zustand):
   ├── Modal state (comment modal, image viewer)
   ├── Compose post state
   └── Notification preferences

3. DATA FETCHING:
   ├── Cursor-based pagination cho feed (không phải offset-based)
   │   useInfiniteQuery({ queryKey: ['feed'], ... })
   ├── Optimistic updates cho likes
   │   Tăng like count NGAY, rollback nếu API fail
   └── WebSocket cho real-time notifications

4. PERFORMANCE:
   ├── Virtualize feed (react-window hoặc @tanstack/react-virtual)
   ├── Image lazy loading + progressive loading
   ├── Code split: media upload, story viewer
   └── Prefetch post detail khi hover

5. OFFLINE SUPPORT (nếu yêu cầu):
   Service Worker + IndexedDB
   ├── Cache recent feed
   └── Queue likes/comments khi offline
```

---

### Bài Toán 2: Thiết kế Collaborative Document Editor — Trình Soạn Thảo Cộng Tác

**Yêu cầu:** Multiple users edit cùng lúc, real-time sync, version history

```
KEY CHALLENGES — Thách Thức Chính:

1. CONFLICT RESOLUTION — Giải Quyết Xung Đột:
   Dùng CRDT (Conflict-free Replicated Data Type — Kiểu Dữ Liệu Không Xung Đột)
   hoặc OT (Operational Transformation — Biến Đổi Hoạt Động)
   Library: Yjs (phổ biến nhất cho React)

2. REAL-TIME SYNC:
   WebSocket với fallback WebRTC (peer-to-peer)
   
   Awareness — Nhận Biết:
   ├── Show cursor positions của các user khác
   ├── Show user avatars trong document
   └── Highlight text đang được ai edit

3. OPTIMISTIC UPDATES:
   Local changes apply ngay, sync lên server async
   Conflict merge tự động với CRDT

4. VERSION HISTORY:
   Snapshot + incremental changes
   Cho phép view và restore previous versions

ARCHITECTURE:

Editor → Yjs Document → WebSocket Provider → Server
                      ↓
               IndexedDB (offline persistence)

React Component:
function CollaborativeEditor({ docId }) {
  const { ydoc, provider, status } = useYjsProvider(docId);
  const editor = useEditor({ extensions: [Collaboration.configure({ document: ydoc })] });

  return (
    <>
      <CollaboratorAvatars awareness={provider.awareness} />
      <EditorContent editor={editor} />
    </>
  );
}
```

---

### Bài Toán 3: Thiết kế Dashboard Analytics — Dashboard Phân Tích

**Yêu cầu:** Nhiều charts, real-time data, date range filter, export

```
COMPONENT ARCHITECTURE:

DashboardPage
├── DashboardHeader
│   ├── DateRangePicker — Bộ Chọn Khoảng Ngày
│   ├── RefreshButton
│   └── ExportButton
├── MetricsOverview (KPI cards)
│   ├── RevenueCard
│   ├── UserCard
│   └── ConversionCard
├── ChartsSection
│   ├── RevenueChart (Line chart, lazy loaded)
│   ├── UserGrowthChart (Bar chart, lazy loaded)
│   └── GeographyMap (lazy loaded, heavy library)
└── DataTable (với virtualization)

STATE MANAGEMENT:
URL State: dateRange, granularity (day/week/month)
Server State (TanStack Query):
  ├── Dashboard overview metrics
  ├── Revenue data (depends on dateRange)
  ├── User data
  └── Table data

PERFORMANCE:
├── Lazy load chart libraries (chart.js, recharts nặng)
├── Memoize expensive calculations
├── Use Web Workers cho data transformation phức tạp
└── Skeleton loading cho tất cả charts

REAL-TIME:
├── Auto-refresh mỗi 5 phút
├── Websocket cho live metrics (nếu yêu cầu)
└── Visual indicator khi data đang refresh
```

---

### Bài Toán 4: Thiết kế Micro-frontend Architecture — Kiến Trúc Vi Giao Diện

**Yêu cầu:** Nhiều team, independent deployment, shared design system

```
ARCHITECTURE DECISION:

Approach: Module Federation (Webpack 5)
├── Shell App (host): routing, auth, shared components
├── Team A: /products (React)
├── Team B: /checkout (React + Redux)
├── Team C: /account (React)
└── Shared: design-system, auth-utils

SHARED CONTRACTS — Hợp Đồng Chia Sẻ:
1. Design System: UI components, tokens, typography
2. Auth State: Current user, permissions
3. Event Bus: Cross-app communication (ít khi cần)
4. Router: URL structure conventions

MODULE FEDERATION CONFIG (Webpack 5):
// shell/webpack.config.js
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    products: 'products@http://products.example.com/remoteEntry.js',
    checkout: 'checkout@http://checkout.example.com/remoteEntry.js',
  },
  shared: {
    react: { singleton: true, requiredVersion: '^19.0.0' },
    'react-dom': { singleton: true },
    '@design-system/ui': { singleton: true },
  },
});

TRADE-OFFS:
✅ Team independence — Deploy riêng, tech stack riêng
✅ Incremental migration — Từng team migrate dần
❌ Bundle duplication nếu không share đúng
❌ Cross-app state phức tạp
❌ Performance: multiple HTTP requests
❌ Version management phức tạp
```

---

## ✅ Checklist System Design — Senior Level

**Process:**
- [ ] Luôn clarify requirements trước khi thiết kế
- [ ] Vẽ high-level diagram trước
- [ ] Nêu trade-offs của từng quyết định

**State Architecture:**
- [ ] Phân loại: server state / client state / URL state / form state
- [ ] Biết khi nào dùng TanStack Query vs Redux vs Zustand vs Context

**Component Design:**
- [ ] Compound Components cho complex UI
- [ ] Generic components với TypeScript generics
- [ ] Composition over inheritance

**Performance:**
- [ ] Biết khi nào cần SSR vs SSG vs ISR vs CSR
- [ ] Virtualization cho long lists
- [ ] Code splitting strategy

**Real-time:**
- [ ] WebSocket vs SSE vs Polling — khi nào dùng gì
- [ ] Optimistic updates với rollback

**Auth:**
- [ ] Token storage trade-offs
- [ ] Protected routes
- [ ] Permission-based rendering

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
